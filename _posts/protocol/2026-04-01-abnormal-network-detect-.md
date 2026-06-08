---
layout: post
title: '基于 Ack 反馈实现的网络下行链路异常检测'
subtitle: 'RUT 基于 Ack 反馈包检测网络下行链路是否异常'
date: 2025-11-01
author: Mr Chen
# cover: '/assets/img/shan.jpg'
#cover_author: 'rogov'
#cover_author_link: 'https://unsplash.com/@rogovca'
categories: Protocol
tags: 
- Protocol
---


## 背景与问题

RUT 是基于 UDP 实现的同时支持可靠和不可靠差异化 Qos 的传输协议，发送端的拥塞控制、重传决策都强依赖于 **ACK 的到达节奏**。在真实弱网中，反馈链路（incoming | downlink，承载对端发回的 ACK）经常出现两种典型的异常状态：

1. **ACK 卡顿（ACK stuck）**：反馈链路被瞬时阻塞，ACK 长时间不到。发送端误以为数据丢了，触发不必要的超时重传，或者在静默期（quiescence）Timer 偏长，醒得太晚。
2. **ACK 突发（ACK burst）**：链路阻塞解除后，被排队积压的 ACK 在极短时间内集中涌入。这是 "反馈链路异常" 的特征之——：数据本身可能没丢，只是 ACK 在回来的路上被攒了一波。

简单的 "基于 RTT 阈值" 判断无法区分 "前向链路差" 还是 "反馈链路差"。本算法的目标是：**仅凭 ACK 到达的时间分布与数量分布，识别出反馈链路的异常，并为 RTO 重传策略提供更精准的 ACK 超时时间点。**

## 设计目标

| 目标 | 说明 |
| --- | --- |
| 链路方向判定 | 当判定为 ACK 突发时，向上层报告 `Direction::kIncoming` 异常 |
| ACK 卡顿超时 | 给出一个动态的 ACK 等待截止时间，供 `重传决策模块 RetransmitManager` 使用 |
| 零额外探测 | 完全复用已有的 `OnPacketSent` / `OnAckFrame` 上层驱动事件，无需主动探测包 |
| 极低开销 | 仅维护几个标记位 + 一个三态状态机；开关用位域，仅需 1 个字节开销 |

两个能力可独立开关：
- `enable_link_detection_`：链路方向检测（burst 突发判定）
- `enable_ack_stuck_detection_`：ACK 卡顿检测

## 核心状态与数据结构

>EWMA: 指数加权移动平均算法。


```cpp
enum State : uint8_t { STATE_NORMAL, STATE_ACK_SPARSE, STATE_ACK_BURST };
```


| 字段 | 含义 |
| --- | --- |
| `avg_ack_time_diff` | 相邻两次 ACK 事件时间间隔的 EWMA（0.7 旧 + 0.3 新） |
| `avg_ack_pkt_num` | 每 50ms 窗口内 ACK 包数量的 EWMA |
| `ack_pkt_counter` | 当前 50ms 窗口已累计的 ACK 包数 |
| `last_ack_event_time` | 上一次收到 ACK 反馈包的时间 |
| `ack_burst_start_time` | 进入异常态的起始时间（用于 500ms 后状态的清除） |
| `cal_ack_num_update_time` | 窗口统计的对齐时间戳 |


**位域技巧**：两个开关与 `enable_anything` 共用一个 union。热路径入口只需 `if (!enable_anything) return;` 一次判断即可同时短路两个特性，避免分别取两个 bool 进行判断。

```cpp
union {
    struct {
        uint8_t enable_link_detection      : 1;
        uint8_t enable_ack_stuck_detection : 1;
        uint8_t unused                     : 6;
    };
    uint8_t enable_anything;
};
```

## 关键算法实现

### 稀疏判定 `MaybeBurstAckComing`

判断是否 "ACK 突然变得稀疏 Sparse"，这是 Burst 突发到来的征兆，需同时满足以下条件：

- `avg_ack_time_diff` 即 Ack 平均到达间隔时间的平均值，已初始化；
- `ack_diff > kAckDiffMinGap`（50ms）；
- `rtt_diverge > kRttDivergeThreshold(60ms)`（`rtt_diverge` 为 `latest_rtt - smoothed_rtt`，满足该条件时，说明网络队列在增长）；
- 且本次间隔显著超过历史均值，满足以下任一：
  - `ack_diff > avg + 3*50ms`（绝对超出）
  - `ack_diff > avg*10 且 > 2*50ms`（相对暴增）
  - `ack_diff > avg*9 且 cwnd 余量 < 1000`（窗口快打满时更敏感）

### 状态迁移

```
NORMAL ──(检测到稀疏)──► ACK_SPARSE ──(随后涌入大量 ACK)──► ACK_BURST
   ▲                                                          │
   └──────────────(异常起点超过 500ms 老化｜过期｜清除)───────────────────┘
```

- **NORMAL → ACK_SPARSE**：`MaybeBurstAckComing` 命中时，刷新 `ack_burst_start_time`。
- **ACK_SPARSE → ACK_BURST**：在 500ms 窗口内，ACK 数量相对均值暴涨——`ack_pkt_counter > avg*2 且 avg+7 < counter`，或 `avg+5 < counter 且 bytes_in_flight < cwnd/5`（在途数据已大量被确认）。此时确认是积压 ACK 的集中释放。
- **任意态 → NORMAL**：`ack_burst_start_time` 未初始化，或距异常起点已超过 **500ms**（复位）。

只有处于 `STATE_ACK_BURST` 状态且开启链路检测时，`GetAbnormalLink()` 才返回 `kIncoming`。

### 均值更新策略

- `avg_ack_time_diff` 仅在 **NORMAL 态** 且 `ack_diff > 0` 时用 EWMA 更新，避免异常态污染基线。
- `avg_ack_pkt_num` 在 `UpdateAckPacketNum` 中每超过 50ms 结算一次窗口，同样**只在 NORMAL 态更新**，结算后计数清零。首次直接取值，之后 EWMA。

### ACK 卡顿超时 `GetAckStuckTimeout`

为重传管理器计算出"再等多久还没收到 ACK 包就算当前链路卡住了"，**如果重传管理器的该定时器超时未收到 Ack，那么将通过上层设置 Pacer 开启一段时间的非受限模式,即在未来的 1 秒内可以立即发送任何数据，不受 Pacer Bucket 的限制，直到接收到新的 Ack 反馈包以后将此状态消除**。

```
max( now + min(avg + 3*50ms, max(avg*9, 50ms)),
     expected_next_ack_time + 60ms )
```

`bytes_in_flight == 0` 或基线未建立时返回 0（不启用）。下界由历史 ACK 节奏决定，并保证不早于"期望下次 ACK 时间 + RTT 抖动容忍"。

## 时序图

下面的时序图展示一次"反馈链路阻塞 → 恢复"过程中，各组件的交互与状态迁移。

![时序图](/assets/img/blog/abnormal_dectect_1.jpeg)


## 状态机图

![状态转换图](/assets/img/blog/abnormal_dectect_2.jpeg)

## 设计权衡与局限

- **只测反馈方向**：当前 `GetAbnormalLink` 只产出 `kIncoming`，因为 "ACK 突发" 是反馈链路的特征信号；前向链路异常由丢包/拥塞算法另行判断。
- **基线纯净性**：均值统计严格只在 NORMAL 态更新，保证异常期间不会自我污染，但代价是异常持续时基线被 "冻结"。
- **500ms 老化**：在保证及时复位与避免抖动误判之间取的折中值；超长抖动场景可能反复进出异常态，对于超大延迟 + jitter 的场景不友好。
- **无主动探测**：完全被动，无额外流量发送，但检测灵敏度受发送速率影响——发包过少时样本不足，检测的及时性和准确性受限。
