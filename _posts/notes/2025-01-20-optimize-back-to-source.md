---
layout: post
title: '直播回源优化'
subtitle: 
date: 2025-01-20
author: Mr Chen
cover: '/assets/img/shan.jpg'
categories: Notes
tags: 
- Life
---

> 该文档主要考虑的是针对需要回源时秒开的优化。

# 现状

&emsp;&emsp;当前 streamd server 在收到拉流请求时，若需要回源，在回源后 streamd 下发给 Player 数据时使用的是帧级别的下发，特别是在接收首个关键帧时，只有接收到完整的视频帧以后，方才开始下发给 Player，这样无形中增加了一定的数据接收+组帧的耗时，通过当前 superset 的数据分析，在需要回源的 case 中， streamd 等待接收首个视频关键帧的时长几乎全部超过 100ms 以上，这便为该次播放请求增加了固定时间的延迟，优化空间明显。

# 优化内容

&emsp;&emsp;针对当前数据转发的模式，将帧级别的转发模式优化为 Slice 级别的转发模式。

## 流程图

![数据转发流程图](/assets/img/blog/streamd-optimize.png)


## 优点

- 加快秒开速度，减少回源阶段 Block 等待时间，初步估计对回源 case 的秒开优化在百毫秒以上。
- Streamd 下发数据时基于 Slice 进行下发，数据发送更加平滑，减少 Frame 级别的下发带来的网络拥塞，有益于卡顿率的降低。

## 缺点

&emsp;&emsp;Streamd 服务在各协议的封装性上需要结构性的调整，数据流程改动较大，需要考虑不同 Protocol/Muxer/Demuxer 之间的兼容性，可能难以实现所有 Protocol/Muxer/Demuxer 之间的分片转发，但依然有很大的优化场景需求。
最理想的情况为：源站提供 FLV 的数据拉流协议，而 Streamd 也是用 FLV 的回源拉流协议，Streamd 在接收端最多需要处理 FLV Header Tag 的部分信息，不阻塞 Slice 的下发。

## FLV 格式

[Reference URL](https://blog.ibaoger.com/2017/06/04/flv-file-format/)


![FLV format](/assets/img/blog/flv-format.png)
