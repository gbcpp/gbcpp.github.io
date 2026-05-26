---
layout: post
title: 'Probe Bandwidth'
subtitle: '带宽探测策略设计与实现'
date: 2025-10-30
author: Mr Chen
# cover: '/assets/img/shan.jpg'
#cover_author: 'rogov'
#cover_author_link: 'https://unsplash.com/@rogovca'
categories: Protocol
tags: 
- Protocol
mermaid: true
---

> 带宽探测通过主动制造可控的数据突发，去探测真实的链路带宽上限，来弥补被动拥塞控制**业务发多快、就只能看见多快**的盲区—，这是实时音视频以及移动网络场景里，尽量把码率拉满、又不至于把链路打爆的前提。

## 为什么需要主动探测

拥塞控制会根据 ACK 和丢包反馈调节发送速率，但它天然偏向跟着业务走,业务层发得慢，观测到的带宽就低；业务发得快，才更容易看到上限。

然而业务方普遍的偏保守（例如在弱网下主动降低码率甚至暂停发送），系统可能长期低估可用带宽。主动探测的价值在于：**在合适时机注入一小段可控的突发数据**，用样本反推链路能力，再把结果反馈给上层做码率决策，最终获得真实的码率上限。

## 1. 整体架构图

```mermaid
flowchart TB
    UP["上层业务 / 拥塞控制 / Pacer"]

    subgraph COORD["协调层"]
        direction TB
        AGG1["最大目标速率聚合<br/>(用于告知上层做 padding)"]
        AGG2["估计器窗长聚合<br/>(取所有会话最长间隔 × 倍数)"]
        BWE["全局带宽估计器<br/>滑窗最大值"]
    end

    subgraph SESS["探测会话集合 (可多实例)"]
        direction LR
        S1["会话 #1<br/>独立节奏 + 本地滑窗"]
        S2["会话 #2"]
        SN["会话 #N"]
    end

    subgraph CQ["待发 Cluster 队列<br/>(每个会话各持一份)"]
        Q1["Cluster Queue"]
    end

    UP -- 创建/销毁/订阅 --> COORD
    COORD -- OnPacketSent|Timer --> SESS
    S1 --> Q1
    S2 -.-> Q1
    SN -.-> Q1
    Q1 -- 发包工单 --> UP
    SESS -- 样本 --> BWE
    BWE -- 当前估计 --> UP
    AGG1 -- 通知 --> UP
```

![整体架构图](/assets/img/blog/probe-bandwidth/01-architecture.png)

协调层 ProbeManager 聚合多个探测会话，统一维护全局带宽估计，并把最大目标速率、探测窗口时长等参数回传给上层；每个会话独立维护自己的探测节奏、本地滑窗和待发的 Cluster 队列。

---

## 2. 单个探测会话状态机（核心状态机）

```mermaid
stateDiagram
    direction TB
    [*] --> Disabled : 初始 (未配置目标带宽)

    Disabled --> EnabledIdle : 设置探测目标带宽 (max > 0)
    EnabledIdle --> Disabled : 清除目标 / 达会话上限

    EnabledIdle --> InterProbing : 定时器到点\n且闸门放行

    state InterProbing {
        direction TB
        [*] --> IntraSending

        IntraSending --> InterPause : cluster 发完\n(满包数 & 满字节)\n或 1s 发送超时

        InterPause --> IntraSending : 继续升档\n(按指数缩放生成新 cluster)

        InterPause --> InterPause : 收到 ACK\n更新样本与本地滑窗
    }

    InterProbing --> EnabledIdle : 本轮结束\n会话未到上限
    InterProbing --> Disabled : 本轮结束\n会话达上限

    note right of Disabled
      未启用：不接受定时器触发
      不响应任何 ACK / 发包事件
    end note

    note right of InterProbing
      一轮 inter-probe 内会按指数走若干档
      默认 4 档，每档一个 cluster
    end note
```

![单个探测会话状态机](/assets/img/blog/probe-bandwidth/02-session-state.png)

- **Disabled**：未启用，不响应定时器与 ACK。
- **EnabledIdle**：已设目标，等待下一轮触发。
- **InterProbing**：一轮内默认约 4 档，每档对应一个 cluster；ACK 在暂停态更新样本与本地滑窗。

---

## 3. 启动会话前的闸门判断

```mermaid
flowchart TD
    A([定时器到点]) --> B{在拥塞控制慢启动期?}
    B -- 是 --> X([跳过本轮])
    B -- 否 --> C{近期标记为受限?}
    C -- 是 --> X
    C -- 否 --> D{拥塞控制允许探测?}
    D -- 否 --> X
    D -- 是 --> E{当前业务稳态带宽<br/>已 ≥ 目标上限?}
    E -- 是 --> X
    E -- 否 --> F([启动一轮 Inter-Probe])
    X --> Y([安排下次时间])
```

![启动会话前的闸门判断](/assets/img/blog/probe-bandwidth/03-gate.png)

不是每个定时器到点都会探测。需同时满足：非慢启动、非近期受限、拥塞控制允许、且当前稳态带宽尚未达到目标上限。

---

## 4. 单档 Intra-Probe 推进策略

```mermaid
flowchart TD
    A([一档 cluster 发完<br/>样本陆续到齐]) --> B{样本带宽 ≥<br/>目标上限 × 90%?}
    B -- 是 --> S([本轮成功结束])

    B -- 否 --> C{本轮已发完<br/>所有预设档位?}
    C -- 是 --> S

    C -- 否 --> D{样本带宽 ≥<br/>当前档目标 × 70%?}
    D -- 否 --> E([不再升档<br/>本轮提前结束])
    D -- 是 --> F([按指数缩放<br/>进入下一档])

    F --> G([生成新 cluster<br/>回到 IntraSending])
    S --> H([记录估计 → 调度下一轮])
    E --> H
```

![单档 Intra-Probe 推进决策](/assets/img/blog/probe-bandwidth/04-intra-probe.png)

一档 cluster 发完且样本到齐后：达到目标上限 90% 则成功结束；已走完预设档位则结束；当前档不足 70% 则提前结束；否则按指数缩放进入下一档。

---

## 5. Cluster 队列的子状态机

```mermaid
stateDiagram
    direction LR
    [*] --> Inactive

    Inactive --> Active : 队列非空\n激活发送\n→ 通知上层"开始探测"

    Active --> Inactive : 满包数 & 满字节\n或 1 秒发送超时\n→ 通知上层"停止探测"

    Active --> Inactive : 整体清理\n(会话重置)

    note right of Active
      每次发包累积：
        包数 +1, 字节 += size
      命中任一条件即出队
    end note
```

![Cluster 队列的子状态机](/assets/img/blog/probe-bandwidth/05-cluster-queue.png)

队列在「非激活 / 激活」间切换：有 cluster 待发则激活并通知上层开始探测；满包数且满字节，或 1 秒发送超时，则回到非激活并通知停止。

---

## 6. Cluster 完整生命周期

```mermaid
stateDiagram-v2
    direction TB
    [*] --> Created : 会话决定本档目标速率\n生成 cluster

    Created --> Queued : 入待发队列

    Queued --> Sending : 队列激活\n上层开始按目标速率发包

    Sending --> Done : 包数 & 字节均达标
    Sending --> Aborted : 1 秒内未发完\n(pacer/业务在节流)

    Done --> Aggregating : 进入估计器\n累积 ACK / 接收时间戳

    Aggregating --> Settled : 样本足够\n产出有效带宽估计

    Aggregating --> Expired : 长时间无足够样本\n(超过聚合过期阈值)

    Aborted --> [*]
    Settled --> [*]
    Expired --> [*] : 触发兜底结算\n以当前最大样本作为结果
```

![Cluster 完整生命周期](/assets/img/blog/probe-bandwidth/06-cluster-lifecycle.png)

单个 cluster 从创建、入队、发送，到完成或中止，再进入估计器的聚合与结算；样本不足过久则过期，并以当前最大样本兜底。

---

## 7. 带宽估计

```mermaid
flowchart TB
    P([每个 ACK / 接收事件]) --> SPLIT{该包属于<br/>哪个 cluster?}
    SPLIT --> AC["定位该 cluster<br/>的聚合状态"]

    AC --> T1["发送列车<br/>(按发送时间锚)"]
    AC --> T2{有效接收时间?}
    T2 -- 有 --> T3["接收列车<br/>(按接收时间锚)"]
    T2 -- 无 --> T4["补偿器<br/>累计 bytes/cnt"]
    T4 -.下次有效时合并.-> T3
    AC --> T5["ACK 列车<br/>(按 ACK 时间锚)"]

    T1 --> V{三条列车<br/>是否同时有效?<br/>≥5包 & ≥90%字节<br/>& 间隔 ∈ 1ms~1s}
    T3 --> V
    T5 --> V

    V -- 否 --> Z([样本无效<br/>丢弃])
    V -- 是 --> R["对每条列车计算速率:<br/>剔除首尾包<br/>用'次新'时间作锚"]

    R --> M["样本 = min(<br/>  发送列车速率,<br/>  接收/ACK 列车速率<br/>)"]

    M --> W["写入全局滑窗最大值滤波器<br/>(窗长由协调者动态设)"]
    W --> OUT([对外输出当前估计])
```

![带宽估计](/assets/img/blog/probe-bandwidth/07-bandwidth-estimate.png)

每个 ACK/接收事件按 cluster 归入聚合状态，在发送、接收、ACK 三条「列车」上分别建样本；三条同时有效时取各列车速率的最小值，再写入全局滑窗最大值滤波器。

---

## 8. 定时器主循环（驱动一切的心跳）

```mermaid
flowchart TD
    T([周期定时器触发]) --> A[协调者遍历所有会话]

    A --> B{会话已禁用?}
    B -- 是 --> NEXT[下一个会话]
    B -- 否 --> C[清理过期 cluster<br/>必要时兜底结算本轮]

    C --> D{到达<br/>下一轮起始时间?}
    D -- 是 --> E[执行闸门判断<br/>见 图 3]
    E --> F[启动本轮<br/>生成首档 cluster]
    D -- 否 --> G

    F --> G{到达<br/>下一档时间?}
    G -- 是 --> H[激活下一档 cluster]
    G -- 否 --> NEXT
    H --> NEXT

    NEXT --> I{所有会话遍历完?}
    I -- 否 --> A
    I -- 是 --> J[聚合最大目标速率<br/>聚合估计器窗长]
    J --> K[估计器自身定时维护<br/>滑窗向下衰减]
    K --> END([本次心跳结束])
```

![定时器主循环](/assets/img/blog/probe-bandwidth/08-timer-loop.png)

周期心跳驱动一切：遍历会话、清理过期 cluster、按节奏启动新一轮或激活下一档，最后聚合全局参数并维护估计器滑窗衰减。

---

## 9. 三类回调的总览（时序角度）

```mermaid
sequenceDiagram
    autonumber
    participant UP as 上层业务
    participant C as 协调者
    participant S as 探测会话
    participant Q as Cluster 队列
    participant E as 估计器

    Note over C,S: 心跳：节奏驱动
    UP->>C: 周期心跳
    C->>S: 派发心跳
    S->>S: 通过闸门 → 启动一轮
    S->>Q: 生成首档 cluster
    Q-->>S: 进入激活态
    S-->>C: 当前目标速率上调
    C-->>UP: 通知开始 padding 到 X

    Note over UP,Q: 发包：业务侧驱动
    UP->>C: 业务包发出事件
    C->>S: 派发至各会话
    S->>Q: 累计包数/字节
    alt 达标或超时
        Q-->>S: cluster 出队
        S-->>C: 本档目标速率清零/下调
        C-->>UP: 通知调整 padding 目标
    end

    Note over UP,E: 反馈：对端 ACK 驱动
    UP->>C: ACK / 接收时间事件
    C->>S: 派发至处于评估期的会话
    S->>E: 提交样本输入
    E-->>S: 返回当前样本速率
    S->>S: 更新本地滑窗 → 判定升档/结束
    alt 本轮结束
        S-->>C: 估计已更新
        C-->>UP: 通知最新带宽估计
    end
```

![三类回调的总览](/assets/img/blog/probe-bandwidth/09-callbacks.png)

从时序上讲，探测由三类事件串联：**心跳**定节奏、**发包**执行 cluster、**ACK** 回填样本并驱动升档或结束。
