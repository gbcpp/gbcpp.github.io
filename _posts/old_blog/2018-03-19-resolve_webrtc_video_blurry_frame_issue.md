---
layout: post
title: 解决 WebRTC 花屏问题记录
subtitle: '实现 RTCDataChannel 类似于裸 UDP 的传输效率'
date: 2018-03-19
author: Mr Chen
# cover: '/assets/img/shan.jpg'
#cover_author: 'rogov'
#cover_author_link: 'https://unsplash.com/@rogovca'
categories: Notes
tags: 
- WebRTC
---

>现象：WebRTC native 层经常打印 `Failed to unprotect SRTP packet, err=9`，订阅端会出现部分绿屏、花屏的情况，严重影响视频通话体验。

<!--more-->

# 情景描述

当前基于 WebRTC 开发自己的音视频实时通话的产品，服务器选择了 licode，在多端 H5、Android、iOS、C++ Native 端均有一定的概率会出现绿屏、花屏的现象，特别是在弱网下出现的概率更高；

在出现绿屏、花屏时，native log 也会经常输出的 `Failed to unprotect SRTP packet, err=9` log，花屏时，或者完全无法播放（有流量，但无画面），此 `Failed to unprotect SRTP packet, err=9` log 出现的概率更高；

当前已排除不同订阅终端在同一房间内跨运营商的网络问题，但依然会存在此类问题；

RTP/RTCP

- SR（Sender Report） 发送端报告

- RR（Receiver Report）接收端报告

- REMB (Receiver Estimated Maximum Bitrate)

 接收端最大接收码率估测，接收端会估计本地接收的最大带宽能力，并通过RTCP REMB 消息返回给对端，这样对端可以调整自己的发送端码率，达到动态调整带宽的目的，具体消息类型通过rtcp的属性 `
Unique identifier` 来区分是 `REMG` 消息，此部分是四个字节，

REMG 信令协商：webrtc里面的协商格式是：`a=rtcp-fb:100 goog-remb，100 是codec payload`; 这里goog 实际上是 google 实现了自己版本的remb。

- GCC 码率估计算法（接收端），用来生成 `REMG` 包

- TMMBR（Temporal Max Media Bitrate Request），表示临时最大码率请求。表明接收端当前带宽受限，告诉发送端控制码率。

- TMMBN（Temporal Max Media Bitrate Notification）临时最大码率通知。

- SDES 源描述，主要功能是作为会话成员有关标识信息的载体，如用户名、邮件地址、电话号码等，此外还具有向会话成员传达会话控制信息的功能。

- BYE　通知离开，主要功能是指示某一个或者几个源不再有效，即通知会话中的其他成员自己将退出会话。

- APP　由应用程序自己定义，解决了RTCP的扩展性问题，并且为协议的实现者提供了很大的灵活性。

**关键帧请求**

- PLI（Picture Loss Indication）

- SLI（Slice Loss Indication）

发送方接收到接收方反馈的PLI或SLI需要重新让编码器生成关键帧并发送给接收端。

- FIR（Full Intra Request）

这里面 Intra 的含义是图像内编码，不需要其他图像信息即可解码；Inter 指图像间编码，解码需要参考帧。所以 Intra Frame 其实就是指 I 帧，Inter Frame指 P 帧或 B 帧。

>那么为什么在 PLI 和 SLI 之外还需要一个 FIR 呢？
>原因是使用场景不同，FIR 更多是在一个中心化的 Video Conference 中，新的参与者加入，就需要发送一个 FIR，其他的参与者给他发送一个关键帧这样才能解码，而 PLI 和 SLI 的含义更多是在发生丢包或解码错误时使用。

**重传请求**

- RTX／NACK／RPSI

这个重传跟关键帧请求的区别是它可以要求任意帧进行重传;

- RED（redundant packet）冗余包，配合 FEC 使用。

- FEC 前向纠错算法，经 FEC 算法计算后，通过 RED 生成冗余包。

- SRTP 详解 [参考 ULR](http://blog.csdn.net/ljinddlj/article/details/3912747)



