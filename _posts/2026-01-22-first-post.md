---
layout: post
title: "AI PC / 边缘盒子中的 NVMe：从 TTFT 到 KV Cache Offload 的存储设计与观测"
date: 2026-01-22
categories: [Storage, AI, NVMe, Edge]
tags: [AI PC, KV Cache, Tail Latency, NVMe Queue, TTFT]
---

> 目标读者：AI PC/边缘盒子平台架构、NVMe SSD/FW、系统性能工程团队  
> 阅读时间：10–12 分钟  
> 本文给出端侧推理（AI PC / Edge Box）对 NVMe 的关键需求分解，并提供可复用的观测基线与工程检查清单。

## 1. 背景与问题定义

在 AI PC 与边缘盒子上，推理性能往往不只由 NPU/GPU/CPU 决定；当 **权重/激活加载**、**KV Cache 驻留与溢出**、以及 **小块随机读的尾延迟**进入关键路径后，NVMe 子系统（SSD + PCIe + OS I/O 栈）的设计会直接决定：

- **TTFT（Time to First Token）**：首 token 输出时间
- **Tokens/s**：稳定段吞吐以及热稳态下的衰减斜率
- **P99/P999 尾延迟**：对 Decode 阶段的“卡顿/抖动”影响尤为显著
- **能耗与热设计**：持续负载下的性能保持能力

本文用三张示意图固定三件事：**架构、数据路径、观测方法**，便于你把它扩展为系列文章或内部设计评审材料。

---

## 2. 系统架构概览（图 1）

下图展示 AI PC / Edge Box 推理栈的典型结构：加速器（NPU/GPU/CPU）+ DRAM + NVMe SSD + OS I/O 栈。

![图1：AI PC / Edge Box 推理栈与 NVMe 数据路径]({{ site.baseurl }}/assets/images/2026-01-22-first-post/fig1-architecture.png)

**解读要点：**
- 推理关键路径不仅是算子执行，还包括 I/O 栈：页缓存、块层、NVMe 队列、IRQ/轮询、CPU 亲和性与 NUMA
- 当 DRAM 不足或采用分层策略时（例如 KV Cache offload、权重分层/按需加载），NVMe 的 **尾延迟**与 **QDepth 适配**会成为决定性因素
- 建议明确：平均时延不能代表体验，必须跟踪 **P95/P99/P999** 与热稳态下的变化

---

## 3. 数据流与访问形态（图 2）

把推理分为两个宏阶段更容易对齐 NVMe 策略：**Prefill（更接近顺序/大块）** 与 **Decode（小块随机 + 高频）**。KV Cache Offload 介于两者之间，通常对尾延迟最敏感。

![图2：Prefill/Decode 访问形态与 KV Cache Offload]({{ site.baseurl }}/assets/images/2026-01-22-first-post/fig2-dataflow.png)

**建议你在评估时至少回答：**
1. Prefill：权重文件是否可做连续化布局（减少碎片），读粒度是否能提升（例如 128KB/256KB 级别）
2. Decode：小块访问集中在 4K/8K/16K 还是更大？并发度（QDepth）典型是多少？
3. KV Cache：是按 token 追加（append）还是冷热分层？是否需要压缩/量化？写入是否可批量化？
4. 是否存在“写放大 → GC 抖动 → 读尾延迟尖刺 → tokens/s 波动”的链式放大？

---

## 4. 性能观测与基线（图 3）

建议把观测分三层：**应用层（TTFT、tokens/s）**、**系统层（CPU/IRQ/温度/功耗）**、**存储层（lat/IOPS/带宽/队列）**。只有三层联动，才能把“模型变慢”定位为算力瓶颈还是 I/O 尾延迟问题。

![图3：端到端观测基线：TTFT / Tokens/s / NVMe Latency]({{ site.baseurl }}/assets/images/2026-01-22-first-post/fig3-performance.png)

**可复用的“基线表”（建议每次评测都填）：**
- TTFT：平均 / P95 / P99
- Tokens/s：稳定段均值 + 热稳态下降斜率
- NVMe 读：4K/16K 随机读 P99（以及 P999），顺序读带宽
- NVMe 写：KV 或日志写的写放大、GC 触发频率（如可从 SMART/厂商日志侧推）
- 系统侧：CPU softirq 比例、IRQ 亲和性、NPU/GPU 利用率、温度/功耗曲线

---

## 5. 设计建议（工程可落地清单）

### 5.1 文件布局与预取（Prefill 侧）
- 权重文件尽量连续化（减少碎片），必要时离线重排或采用 pack/容器格式
- Prefill 尽量使用更大块读取粒度，避免被 4K I/O 切碎
- 合理设置 readahead 与页缓存策略，避免过度预取引发内存压力与抖动

### 5.2 NVMe 队列与中断策略（Decode 侧）
- 关注队列数与队列深度：并发不足会压低吞吐；并发过高会放大抖动与争用
- 根据平台选择轮询/中断模式，并检查 CPU 亲和性（避免关键线程与 I/O 中断争用同一核）
- 多 NUMA/多 PCIe Root Complex 场景下，确保 I/O 路径的 NUMA 亲和性

### 5.3 KV Cache Offload 的取舍
- Offload 可以换取 DRAM 空间，但把系统体验绑定到 NVMe 的尾延迟
- 优先手段：冷热分层 + 压缩/量化 + 批量化写入（减少随机写与 GC 干扰）
- 建议设定“可接受的 P99 读时延阈值”，并以 tokens/s 波动作为体验判据

---

## 6. 结论与下一步

本文固定了三件事：**架构、数据路径、观测方法**。下一篇建议聚焦其中一条主线：

- 选题 A：KV Cache Offload 到 NVMe 的队列设计与尾延迟预算
- 选题 B：AI PC 平台上 NVMe 轮询/中断与 CPU 亲和性对 tokens/s 的影响
- 选题 C：热稳态下 SSD 性能保持与推理吞吐衰减的关联模型

