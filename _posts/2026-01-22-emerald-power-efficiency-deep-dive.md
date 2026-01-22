---
title: "SNIA Emerald™ Power Efficiency Measurement Specification 深度解读"
date: 2026-01-22
tags:
  - SNIA Emerald
  - Power Efficiency
  - SSD
  - Storage
  - Firmware
  - NAND
  - Controller
categories:
  - Storage
  - SSD
  - Standards
description: "从标准设计动机到工程落地，系统解读 SNIA Emerald™ 存储功耗效率测试规范及其对 FW、NAND 与控制器架构的影响。"
---

# SNIA Emerald™ Power Efficiency Measurement Specification  
## 从标准动机到工程落地的完整技术解读

**适用读者**
- 存储解决方案架构团队  
- SSD 控制器 / FW 研发  
- NAND 技术与产品规划  
- 数据中心系统工程  

---

## 一、为什么会有 SNIA Emerald™

随着数据中心规模持续扩大，功耗已经不再是单一器件的“成本问题”，  
而是系统级约束、长期 TCO 与可持续发展的核心变量。

传统存储评估方式长期存在以下问题：

- 关注峰值性能（Peak IOPS / 带宽），忽略稳态行为  
- 功耗测量方法不统一，结果不可复现  
- 测试负载由测试方自由选择，容易“为结果服务”  

在这一背景下，SNIA 发布了 Emerald™ Power Efficiency Measurement Specification。  
该标准的目标并不是“测试低功耗”，而是建立一个 **以效率（Work per Watt）为核心的工程化评估体系**。

**Emerald 试图回答的问题是：**  
在真实数据中心负载与稳态条件下，  
谁能用更少的能量，持续完成更多有效工作？

---

## 二、Emerald 的核心设计理念

### 2.1 效率，而不是单纯的功耗或性能

Emerald 不直接比较：
- 谁的功耗更低  
- 谁的 IOPS 或带宽更高  

而是统一采用以下逻辑：

**功耗效率 = 有效工作量 / 平均功耗**

其中：
- 有效工作量：IOPS 或 MiB/s（稳态平均值）  
- 平均功耗：稳态平均功耗（W）

这意味着：
- 通过显著增加功耗换取性能提升，在 Emerald 中往往是负收益  
- 真正优秀的设计，应实现性能增长快于功耗增长  

---

### 2.2 稳态（Steady State）是硬约束

Emerald 明确拒绝以下测试方式：

- Fresh-Out-of-Box（FOB）峰值成绩  
- 短时间 burst 行为  
- 依赖缓存掩盖真实能耗的测试结果  

标准通过以下机制强制进入稳态：

- 至少 150 分钟的稳态判定过程  
- 多轮（Round）趋势分析  
- 要求性能与功耗在时间维度上保持稳定  

**在 Emerald 中，时间本身就是放大器。**

---

## 三、测试方法层面的关键差异

### 3.1 TOIO 自优化，而不是固定 QD / Queue

传统 benchmark 通常由测试人员人为指定：
- Queue Depth  
- 线程数  

这使得测试结果容易被“针对性优化”。

Emerald 引入 TOIO（Total Outstanding IO）机制：

- 在 warm-up 阶段扫描不同负载强度  
- 自动选择满足以下条件的工作点：
  - 平均响应时间（ART）小于 20 ms  
  - 吞吐量（IOPS）最大  

**本质上，Emerald 在 SLA 约束下寻找系统的自然最优工作区间。**

---

### 3.2 Complex Workload：真实世界的近似模型

Complex Workload 是 Emerald 的核心负载，具备以下特征：

- 多种 IO 大小  
- 读写混合  
- 冷热数据共存  
- 访问倾斜（skew）  

它同时施压于：
- Cache 命中与失效  
- FTL 映射稳定性  
- GC 与前台 IO 的竞争  
- 后台维护任务的功耗行为  

**Complex 是 NAND、FTL、控制器调度与功耗管理的综合考试。**

---

### 3.3 为什么随机采用 8 KiB，而不是 4 KiB

- 4 KiB 已被 DRAM、SLC Cache、队列路径高度优化  
- 难以暴露 NAND 与 FTL 的真实能耗差异  

8 KiB 更容易：
- 跨映射与页边界  
- 触发 partial page、RMW 与 GC  
- 进入真实稳态写能耗区间  

---

### 3.4 为什么顺序采用 256 KiB，而不是 128 KiB

256 KiB 顺序 IO：
- 可完全展开 NAND 并行度  
- 避免停留在控制器内部 buffer 区  
- 更贴近真实数据中心顺序流  

Emerald 实际关注的是：
**吞吐增长速度是否快于功耗增长速度。**

---

## 四、功耗效率指标体系的工程含义

### 4.1 Ready Idle（TB/W）

- 用户容量 ÷ 就绪空闲平均功耗  
- 大容量 SSD 具备结构性优势  
- 前提是后台维护足够安静  

---

### 4.2 Active（IOPS/W、MiB/s/W）

- 随机负载：IOPS / W  
- 顺序负载：MiB/s / W  

不奖励线性功耗换性能，  
强调长期稳态效率。

---

## 五、对 FW / NAND / 控制器的直接影响

### FW
- 稳态友好的 GC / WL  
- 后台任务可中断、可延迟  
- Idle 快速降功耗  

### NAND
- 单位 bit 能耗成为核心变量  
- 高密度 NAND 在 Ready Idle 上更具优势  

### 控制器
- 能效感知调度  
- 并行度与功耗平衡  
- 热 / 功耗 / 延迟协同管理  

---

## 六、总结

SNIA Emerald™ 不是偏向某一类产品的标准，  
而是一个系统性放大真实架构差异的工程规范。

**能在 Emerald 下表现优秀，意味着产品在真实数据中心环境中具备长期竞争力。**

---

*本文用于内部技术解读与工程参考。*
