# SNIA Emerald™ Power Efficiency Measurement Specification  
## 从标准动机到工程落地的完整技术解读

**适用读者**  
- 存储解决方案架构团队  
- SSD 控制器 / FW 研发  
- NAND 技术与产品规划  
- 数据中心系统工程  

---

## 一、为什么会有 SNIA Emerald™？

随着数据中心规模持续扩大，功耗已经不再是单一器件的“成本问题”，  
而是**系统级约束、长期 TCO 与可持续发展的核心变量**。

传统存储评估方式长期存在以下问题：

- 关注峰值性能（Peak IOPS / BW），忽略稳态行为  
- 功耗测量方法不统一，结果不可复现  
- 测试负载由测试方自由选择，容易“为结果服务”  

在这一背景下，:contentReference[oaicite:0]{index=0}（SNIA）发布  
**Emerald™ Power Efficiency Measurement Specification**，其目标并不是“测试低功耗”，  
而是建立一个 **以效率（Work / Watt）为核心的、工程可复现的行业标准**。

> Emerald 试图回答的问题是：  
> **在真实数据中心负载与稳态条件下，  
谁能用更少的能量，持续完成更多有效工作？**

---

## 二、Emerald 的核心设计理念

### 2.1 效率，而不是功耗或性能

Emerald 并不直接比较：
- 谁的功耗更低  
- 谁的 IOPS 更高  

而是统一转换为：

\[
\text{Power Efficiency} = \frac{\text{Work}}{\text{Watt}}
\]

其中：
- Work = IOPS 或 MiB/s（稳态平均值）  
- Watt = 稳态平均功耗  

这意味着：
- 牺牲大量功耗换取性能提升，在 Emerald 中是负收益  
- 真正优秀的设计，应实现 **性能增长快于功耗增长**

---

### 2.2 稳态（Steady State）是硬约束

Emerald 明确拒绝：
- FOB（Fresh-Out-of-Box）峰值  
- 短时间 burst  
- 缓存“幻觉成绩”  

并通过：
- ≥150 分钟稳态判定  
- 5 × 30min Round 趋势分析  

强制将以下行为暴露出来：
- GC / WL 是否平滑  
- 后台任务是否周期性抖动  
- 热与功耗调度是否稳定  

**在 Emerald 中，时间是放大器。**

---

## 三、测试方法层面的关键差异

### 3.1 TOIO 自优化，而不是固定 QD / Queue

**传统 benchmark：**
- 人为指定 QD / 线程数  
- 极易针对固定点位做优化  

**Emerald：**
- 在 warm-up 阶段扫描负载强度  
- 自动选择：
  - ART < 20 ms  
  - 条件下 IOPS 最大的 TOIO  

其本质是：

> **在 SLA 约束下，寻找系统“自然最优工作点”。**

**工程影响：**
- 延迟 cliff 会直接拉低 TOIO  
- GC / cache 抖动会影响最终工作点  
- 无法通过人为参数“规避架构问题”

---

### 3.2 Complex Workload：真实世界的近似模型

Complex 并非简单混合负载，而是：

- 多 IO size  
- 读写混合  
- 冷热数据共存  
- 访问倾斜（skew）  

它同时施压：
- Cache 命中与失效  
- FTL 映射稳定性  
- GC 与前台 IO 的竞争  
- 后台维护任务的功耗行为  

在 Emerald 中：
- Steady State 来源于 Complex  
- Active Efficiency 权重也高度依赖 Complex  

> **Complex 是 NAND + FTL + 控制器调度 + 功耗管理的综合考试。**

---

### 3.3 为什么随机采用 8 KiB，而不是 4 KiB？

选择 8 KiB 是刻意规避 4 KiB 的“过度工程化”：

- 4 KiB 已被 DRAM / SLC / Queue path 极致优化  
- 难以区分 NAND、FTL、GC 的真实能耗差异  

8 KiB 的工程意义在于：
- 更容易跨映射与页边界  
- 更快触发 partial page / RMW / GC  
- 更早进入真实稳态写能耗区间  

**结论：**
> 8 KiB Random 是 Emerald 用来“逼 NAND 与 FTL 出场”的尺寸选择。

---

### 3.4 为什么顺序采用 256 KiB，而不是 128 KiB？

256 KiB 的目标不是“顺序”，而是 **系统级吞吐能效**：

- 能完全展开 NAND 并行度（channel / die / plane）  
- 避免停留在 controller-friendly buffer 区  
- 更贴近真实数据中心顺序流（AI / analytics / scan）  

从能效角度看：
> Emerald 关心的是  
> **MiB/s 增长是否快于 Watt 增长**，  
256 KiB 更容易拉开架构差异。

---

## 四、功耗效率指标体系的工程含义

### 4.1 就绪空闲功耗效率（Ready Idle）——TB/W

\[
\text{Efficiency}_{ReadyIdle} =
\frac{\text{User Capacity}}{\text{Average Idle Power}}
\]

特点：
- 容量直接进入分子  
- 后台行为全部体现在分母  

工程含义：
- 大容量（尤其 QLC）具备结构性优势  
- 但前提是：
  - refresh / scrub 低频、可延迟  
  - 后台维护“足够安静”

---

### 4.2 活跃功耗效率（Active）——IOPS/W、MiB/s/W

- Random：IOPS / W  
- Sequential：MiB/s / W  

Emerald 明确：
- 不奖励线性功耗换性能  
- 更偏好：
  - 次线性功耗增长  
  - 或性能线性增长

---

## 五、为什么行业要“使用 Emerald”

从标准动机看，Emerald 的价值体现在三层：

1. **对研发**  
   - 提供工程真实反馈，而不是营销成绩  

2. **对客户与采购**  
   - 提供 Active + Idle 的完整能效视图  

3. **对行业与监管**  
   - Emerald 已被 ENERGY STAR® 等能效规范采纳  
   - 成为合规与认证的重要基础方法  

> Emerald 是连接 **工程真实世界** 与 **行业评价体系** 的桥梁。

---

## 六、对 FW / NAND / 控制器的直接影响

### 6.1 FW：不存在“测试模式”

- Steady State 友好：
  - GC / WL 连续、低振幅  
- 后台任务：
  - 可中断、可延迟、可合并  
- Power state：
  - Idle 快速下探  
  - Active 不频繁抖动  

---

### 6.2 NAND：从“容量介质”变为“能效变量”

- 单位 bit 的 program / read 能耗直接影响成绩  
- 高密度 NAND（QLC）在 Ready Idle 上具备天然优势  
- 前提是：
  - refresh 能耗可控  
  - 后台维护节奏平滑  

---

### 6.3 控制器：能效感知成为核心能力

- energy-aware scheduling  
- 并行度与功耗的动态平衡  
- 热 / 功耗 / 延迟协同管理  

Emerald 实际筛选的是：

> **是否具备长期、稳定、数据中心级的功耗调度能力。**

---

## 七、总结

SNIA Emerald™ 并不是一个“对某一类产品友好”的标准，  
而是一个**系统性放大真实架构差异的工程标准**。

它正在推动行业从：
- 峰值性能导向  
转向：
- 稳态效率导向  

> **能在 Emerald 下表现优秀，  
意味着产品在真实数据中心环境中具备长期竞争力。**

---

*本文基于 SNIA Emerald™ 官方资料与标准规范整理，用于内部技术解读与工程参考。*
