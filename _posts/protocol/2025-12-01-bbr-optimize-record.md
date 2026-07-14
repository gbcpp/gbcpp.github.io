---
layout: post
title: 'BBR 优化重点项'
subtitle: '实时数据传输拥塞控制算法基于 BBR 优化重点方向'
date: 2025-12-01
author: Mr Chen
# cover: '/assets/img/shan.jpg'
#cover_author: 'rogov'
#cover_author_link: 'https://unsplash.com/@rogovca'
categories: Protocol
tags: 
- Protocol
mermaid: true
---

## BBR 状态机

BBR 通过四个阶段动态探测带宽与最小 RTT，在不同阶段间循环切换：

```mermaid
flowchart LR
    STARTUP --> DRAIN
    DRAIN --> PROBE_BW
    PROBE_BW --> PROBE_RTT
    PROBE_RTT --> PROBE_BW
```
