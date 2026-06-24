---
layout: post
title: '如何识别网络延迟突变是稳定性阶跃还是瞬时抖动，优化 BWE 稳定性'
subtitle: ''
date: 2025-11-01
author: Mr Chen
# cover: '/assets/img/shan.jpg'
#cover_author: 'rogov'
#cover_author_link: 'https://unsplash.com/@rogovca'
categories: Protocol
tags: 
- Protocol
---


> BBR 依赖 `min_rtt` 估算 BDP/CWND，但 `min_rtt` 只会被更小的 RTT 刷新，或等 10s 过期后进入 `PROBE_RTT` 重探。当链路传播时延发生持久阶跃（如路由切换、弱网追加延迟）时，旧 `min_rtt` 会长期偏低，导致 BDP 低估、CWND 被压、吞吐饿死；阶跃过渡期还可能因伪丢包触发 recovery 窗口进一步限流。该算法模块在 ACK 路径上用双轨道 EWMA + 6σ/持续性/稳定性判据，区分「稳定 RTT 阶跃」与「瞬时抖动」。确认 **上跳** 后立即抬高 `min_rtt` 并跳过常规过期逻辑；**下跳** 时放开 recovery 限制，由主干自然下调。


## 1. 解决的问题

BBR 的一切都建立在 `**min_rtt`（链路往返最低延迟）** 之上：

- `BDP = min_rtt × BandwidthEst`，目标拥塞窗口 `target_cwnd = gain × BDP`。
- `min_rtt` 只在两种情况下被刷新：采样到了**更小** 的 RTT，或 `**MinRttExpiry = 10s` 过期** 后进入 `PROBE_RTT` 重新探测。

这套机制在「**路径的真实传播时延发生持久阶跃**」时，也就是突然出现 rtt 持久性的变大，会失效：


| 场景                       | 现象       | 主干 BBR 的问题                                                                                          |
| ------------------------ | -------- | --------------------------------------------------------------------------------------------------- |
| 路由切换 / 移动网络切换基站（handoff） | RTT 整体上跳 | **旧的** `min_rtt` **已不可达，但在 10s 过期前一直被沿用 →** `BDP` **被低估 → CWND 偏小 → 吞吐被饿死**, bwe 整体会经历快速降低然后再逐步升高恢复 |
| 路径整体变短（如切回低延迟链路）         | RTT 整体下跳 | 主干靠「取更小值」能跟上，但下跳期间常伴随重排序/乱序丢包，恢复窗口会误压 CWND                                                          |

如果没有该算法介入，那么当发生 delay 阶跃时，传输协议内部指标 CWND、BWE 等关键性输出指标变化如下图，先速降然后再逐步爬升:


![无 Delay 阶跃检测算法的传输协议走势](/assets/img/blog/delay-spike/without_delay_spike_detect.png)


`延迟突变检测算法` 的职责：**把「真实、稳定的 RTT 水平阶跃」与「瞬时抖动 / 离群点」区分开**。一旦确认是稳定阶跃：

- **上跳（`SpikeUp`）**：立刻把 `min_rtt` 抬到阶跃后的平滑水平，相当于不等 10s 过期、也不付出 `PROBE_RTT` 吞吐凹陷的代价，就让 BBR 认识到「BDP 变大了」。
- **下跳（`SpikeDown`）**：交给主干「取更小值」自然跟进，检测器只负责放开恢复窗口限制。
- 两种情况都置位 `DisableRecoveryCwnd`，避免阶跃过渡期里的伪丢包通过恢复窗口把有效 CWND 压死。

> 总结：它是 BBR `min_rtt` 的一条 **快速上修通道 + 误判保护**，专门针对「传播延迟的突然变化」这一 BBR 处理不了的弱网形态。

---

## 2. 架构与接入点


![架构图](/assets/img/blog/delay-spike/architecture.png)



**关键交互过程**：

1. `当检测到 SpikeUp` ：直接绕过常规的 `min_rtt` 过期与下修逻辑，本轮 `min_rtt` 被强制设为阶跃后的平滑值，且 `min_rtt_expired` 视为 `false`（不触发 `PROBE_RTT`）。
2. `如果检测到 SpikeDown`： 不 `return`，继续走常规逻辑，`更小的 min rtt` 会自然把 `min_rtt 更新`。

---

## 3. 数据结构（状态变量）

构造函数初值见 `delay_spike_detector.h:12`。


| 成员                    | 含义                                              |
| --------------------- | ----------------------------------------------- |
| `short_term_srtt`     | **基线轨道** 平滑 RTT（正常网络的短期 SRTT）                   |
| `rtt_variance`        | **基线轨道** 平滑方差；`-1` = 未初始化（bootstrap 哨兵）         |
| `spiked_srtt`         | **spike 轨道** 平滑 RTT；`-1` = 本次候选尚未开始累积           |
| `spiked_rtt_variance` | **spike 轨道** 平滑方差                               |
| `spiked_rtt_count`    | spike 轨道累积的样本数（仅在越过 50ms 等待期后才累加）               |
| `outside_point_count` | 候选挂起期间出现的「正常样本」计数，用于误报回滚                        |
| `first_spiked_time`   | 首个可疑样本的时刻；是否已初始化代表「是否处于候选挂起态」                   |
| `delay_spike_states`  | 最近一次确认的阶跃方向（`Normal` / `SpikeUp` / `SpikeDown`） |


> **两条轨道（dual-track）** 是核心设计：基线轨道始终跟踪「当前认定的正常水平」，spike 轨道是「候选新水平」的试探性平滑。只有 spike 轨道被确认后，才会 **整体替换** 基线轨道。

平滑均采用 RFC6298 风格的 EWMA（位移实现）：

- SRTT：`srtt = (7·srtt + crtt) >> 3`，即 `α = 1/8` 取新值。
- 方差：`var = (3·var + dispersion) >> 2`，即 `β = 1/4` 取新值。
- 注意 `dispersion = (crtt − srtt)²` 是 **平方偏差**，因此 `rtt_variance` 是平滑后的 **方差**（均方偏差），`sqrt(rtt_variance)` 即标准差 σ。这一点是后面「6σ」判据成立的前提。

---

## 4. 参数与阈值

> 图片同表格，有时候表格渲染可能是乱码，可以查看图片。

![](/assets/img/blog/delay-spike/table-4.png)

| 常量                                      | 值            | 单位  | 作用 / 数学含义                                                                      |
| --------------------------------------- | ------------ | --- | ------------------------------------------------------------------------------ |
| `MinimalRttDispersionThreshold`        | `2500`       | ms² | 越界判据的方差下限。`2500 = 50²`，即 **偏差至少 50ms** 才算越界（防止低 RTT、低抖动链路上 6σ 太小被误触）           |
| 越界倍数（硬编码 `36`）                          | `36`         | —   | `36 = 6²`。`dispersion > 36·var ⟺                                               |
| `StableDelaySpikeWaitingTime`          | `50`         | ms  | 首个可疑点后的「等待期」，期间只标记不累积，剔除阶跃边沿的中间过渡点                                             |
| `MinimalDelaySpikeObservationDuration` | `200`        | ms  | 确认所需的 **最短** 观测时长（持续性下限）                                                       |
| `MaximalDelaySpikeObservationDuration` | `1200`       | ms  | 确认的 **最长** 观测时长；超过仍未收敛则放弃（认定不是干净阶跃）                                            |
| `MaximalRttSpikeThreshold`             | `1200`       | ms  | 上跳幅度上限。`spiked_srtt < short_term_srtt + 1200` 才认作水平阶跃；超过视为瞬时巨抖/中断，不抬 `min_rtt` |
| spike 样本数门限（硬编码 `10`）                   | `10`         | 个   | spike 轨道需累积 ≥10 个样本                                                            |
| 误报回滚门限（硬编码 `3`）                         | `3`          | 个   | 候选挂起期间累计 3 个正常样本即回滚                                                            |
| 上跳稳定性系数（硬编码 `50`）                       | `50`         | —   | `var·50 < srtt² ⟺ σ/srtt < 1/√50 ≈ 14.1%`（变异系数上限，**严**）                        |
| 下跳稳定性系数（硬编码 `10`）                       | `10`         | —   | `var·10 < srtt² ⟺ σ/srtt < 1/√10 ≈ 31.6%`（变异系数上限，**宽**）                        |
| EWMA 系数                                 | `1/8`, `1/4` | —   | SRTT 取 1/8 新值，方差取 1/4 新值（RFC6298 风格）                                           |


### 4.1 为什么上跳严、下跳宽？

- **上跳** 会把 `min_rtt` 抬高 → 直接放大 BDP 与 CWND。误判代价大（可能引入 bufferbloat），故要求 spike 轨道变异系数 < 14.1%（很稳）且幅度 < 1200ms（有上限）。
- **下跳** 只是放开恢复窗口钳制，真正的下修仍由主干「取更小值」把关，误判代价低，故放宽到 31.6% 以求 **快速反应**。

---

## 5. 运行状态机

> 这里描述的是检测器的 **运行态**（操作语义），与对外输出的 `States` 三值枚举不同——后者只是确认那一刻的方向标签。


![状态转换图](/assets/img/blog/delay-spike/state-diagram.png)



状态语义对照：


| 运行态            | 判定条件                     | 关键动作                                                               |
| -------------- | ------------------------ | ------------------------------------------------------------------ |
| `Bootstrap`    | `rtt_variance < 0`       | 用首样本直接初始化基线（`rtt_variance = dispersion`, `short_term_srtt = crtt`） |
| `Baseline`     | `first_spiked_time` 未初始化 | 每来一个正常样本就 EWMA 更新基线                                                |
| `SpikePending` | `first_spiked_time` 已初始化 | 等待期（<50ms）只标记；越过 50ms 后向 spike 轨道累积 EWMA                           |
| `Confirmed`    | 见 §6 确认判据                | 提升基线、输出方向、`ResetSpikeRecord`、返回 `true`                             |



## 6. 逐样本处理流程

入口 `MaybeXXXHappened(now, crtt)`，`crtt` 为最新 RTT（ms）。

![逐样本处理流程图](/assets/img/blog/delay-spike/sequeue.png)



### 6.1 越界判据

```cpp
rtt_dispersion > max(36 * rtt_variance, MinimalRttDispersionThreshold)
```

`|crtt − short_term_srtt| > max(6σ, 50ms)`，即同时满足「偏离基线 >6 个标准差」**且**「绝对偏离 >50ms」。

### 6.2 spike 轨道累积

仅在 `now > first_spiked_time + 50ms`（越过等待期）后执行：

- 首个累积样本：`spiked_srtt = crtt`，`spiked_rtt_variance = 0`。
- 后续：`spiked_srtt = (7·spiked_srtt + crtt) >> 3`；`spiked_rtt_variance = (3·spiked_rtt_variance + (crtt − spiked_srtt)²) >> 2`。
- `spiked_rtt_count++`。

### 6.3 确认判据

需同时满足 **持续性** 与 **稳定性 + 方向**：

```
now > first_spiked_time + 200ms                          // 持续 ≥200ms
AND spiked_rtt_count >= 10                                // ≥10 个 spike 样本
AND (
      // (a) 上跳：严稳定 + 高于基线 + 幅度不超 1200ms
      spiked_rtt_variance * 50 < spiked_srtt²
        && spiked_srtt > short_term_srtt
        && spiked_srtt < short_term_srtt + 1200
   OR
      // (b) 下跳：宽稳定 + 不高于基线
      spiked_rtt_variance * 10 < spiked_srtt²
        && spiked_srtt <= short_term_srtt
    )
```

满足后再分流：

- `now > first_spiked_time + 1200ms` → 收敛太慢，`ResetSpikeRecord()`，`return false`（**不**认作干净阶跃）。
- 否则 → 确认：
  - `delay_spike_states = (spiked_srtt > short_term_srtt) ? SpikeUp : SpikeDown`；
  - **提升基线**：`rtt_variance = spiked_rtt_variance`，`short_term_srtt = spiked_srtt`；
  - `ResetSpikeRecord()`，`return true`。

> 有效确认窗口是 `(200ms, 1200ms]`。太快（<200ms）不算持续阶跃；太慢（>1200ms）不算干净阶跃。

---

## 7. 典型时序（上跳被确认）

![时序图](/assets/img/blog/delay-spike/sequeue-diagrm.png)


下跳（`SpikeDown`）时序基本相同，区别在确认后 **不** 设 `min_rtt`、**不** 提前 `return`，随后由主干常规逻辑用 `sample_min_rtt < min_rtt` 下调 `min_rtt`。

---

## 8. 误报防护机制


| 机制                 | 阈值                                      | 防的是什么                       |
| ------------------ | --------------------------------------- | --------------------------- |
| 6σ + 50ms 双门限      | `36·var` 且 `2500`                       | 把日常抖动挡在越界判定外                |
| 50ms 等待期           | `StableDelaySpikeWaitingTime`          | 剔除阶跃边沿的过渡样本，避免污染 spike 轨道均值 |
| ≥200ms 持续 + ≥10 样本 | `200ms` / `10`                          | 要求阶跃「站得住」，过滤短脉冲             |
| 变异系数稳定性            | `50` / `10`                             | 要求新水平本身平稳，过滤还在剧烈波动的伪阶跃      |
| 上跳幅度封顶             | `1200ms`                                | 避免把瞬时巨抖/中断当成 min_rtt 阶跃     |
| ≤1200ms 收敛窗        | `MaximalDelaySpikeObservationDuration` | 收敛太慢则放弃，不视作干净阶跃             |
| 3 个正常样本回滚          | `outside_point_count >= 3`              | 候选期间网络恢复正常即撤销候选             |


---

## 9. 设计观察与注意点

1. **整型方差 + `pow` 截断**：`int rtt_dispersion = pow(...)` 用 `double` 计算后截断为 `int`；在常见 RTT 范围（偏差 ≤1200ms → 1.44e6）内不会溢出 `int`，但属隐式窄化。
2. **依赖 ACK 采样密度**：等待期后需在 (200ms, 1200ms] 内攒够 10 个样本，低速率/稀疏 ACK 的链路可能始终攒不够而无法触发——这是有意的保守取舍（实时高包率场景才是目标）。

