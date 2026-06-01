---
layout: post
title: '私有协议带宽探测的设计与实现'
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

> 带宽探测通过主动制造可控的数据突发，去探测真实的链路带宽上限，来弥补被动拥塞控制**业务发多快、就只能看见多快**的不足，这在实时音视频以及移动网络场景里，做到尽量把码率拉满、又不至于把链路打爆的前提。

## 为什么需要主动探测

拥塞控制会根据 ACK 和丢包反馈调节发送速率，但它天然偏向跟着业务走,业务层发得慢，观测到的带宽就低；业务发得快，才更容易看到上限。

然而业务方普遍的偏保守（例如在弱网下主动降低码率甚至暂停发送），系统可能长期低估可用带宽。主动探测的价值在于：**在合适时机注入一小段可控的突发数据**，用样本反推链路能力，再把结果反馈给上层做码率决策，最终获得真实的码率上限。

## 整体架构图


ProbeManager 聚合多个探测会话，统一维护全局带宽估计，并把最大目标速率、探测窗口时长等参数回传给上层；每个会话独立维护自己的探测节奏、本地滑窗和待发的 Cluster 队列。


![整体架构图](/assets/img/blog/probe-bandwidth/01-architecture.jpeg)

## 时序图


![整体时序图](/assets/img/blog/probe-bandwidth/09-callbacks.jpeg)


## 单个探测会话状态机


![单个探测会话状态机](/assets/img/blog/probe-bandwidth/02-session-state.jpeg)

- **Disabled**：未启用带宽探测，不响应定时器与 ACK。
- **EnabledIdle**：启用带宽探测，等待下一轮触发。
- **InterProbing**：一轮内默认约 4 档（可配），每档对应一个 cluster；ACK(包括 random loss) 在暂停态更新样本与本地滑窗。

---

## 启动探测前提条件

启动一轮新的探测需同时满足多组条件，如：
 - 当前 CC 状态是否处于慢启动，且 Path 配置是否允许在该阶段进行探测。
 - 当前带宽非近期受限。
 - 拥塞控制状态允许，如 BBR 需要在 Probe_BW 和 STARTUP 阶段才允许开启探测。
 - 且当前稳态带宽尚未达到目标上限。


![启动会话前的状态判断](/assets/img/blog/probe-bandwidth/03-gate.jpeg)


## 单档 Intra-Probe 推进策略

一档 cluster 发完且样本到齐后：达到目标上限 90% （包数或者字节数满足）则成功结束；已走完预设档位则结束；若当前档不足 70% 则提前结束；否则按指数缩放进入下一档。


![单档 Intra-Probe 推进决策](/assets/img/blog/probe-bandwidth/04-intra-probe.jpeg)


## Cluster 队列(按档位进行探测)的子状态机


队列在「非激活 / 激活」间切换：有 cluster 待发则激活并通知上层开始探测；满包数且满字节，或 1 秒发送超时，则回到非激活并通知停止。


![Cluster 队列的子状态机](/assets/img/blog/probe-bandwidth/05-cluster-queue.jpeg)


## Cluster 完整生命周期


单个 cluster 从创建、入队、发送，到完成或中止，再进入估计器的聚合与结算；样本不足过久则过期，并以当前最大样本兜底。


![Cluster 完整生命周期](/assets/img/blog/probe-bandwidth/06-cluster-lifecycle.png)


## 带宽估计


每个 ACK/接收事件按 cluster 归入聚合状态，在发送、接收、ACK 三个记录队列上分别建立样本；三条同时有效时取各记录队列速率的最小值，再写入全局滑窗最大值滤波器。


![带宽估计](/assets/img/blog/probe-bandwidth/07-bandwidth-estimate.jpeg)


## 定时器主循环


周期心跳驱动一切：遍历会话、清理过期 cluster、按节奏启动新一轮或激活下一档，最后聚合全局参数并维护估计器滑窗衰减。


![定时器主循环](/assets/img/blog/probe-bandwidth/08-timer-loop.jpeg)


## 三类回调总览


从时序上讲，探测由三类事件串联：**心跳**定节奏、**发包**执行 cluster、**ACK** 回填样本并驱动升档或结束。


![三类回调的总览](/assets/img/blog/probe-bandwidth/09-callbacks.jpeg)