---
layout: post
title: '基于 HTTP/3 的 WebTransport '
tags: WebTransport
author: Mr Chen
date: 2024-02-01
categories: Protocol
tag:
- Protocol
---

> 本文主要翻译自https://datatracker.ietf.org/doc/html/draft-vvv-webtransport-http3 ，如有错误请指正。


# 摘要

WebTransport 是一个协议框架，它强迫客户端受 web 安全模型去与远程服务器进行安全的多路传输；
WebTransport 是一个受 Web 安全模型约束的多路复用安全传输协议，本文将描述一个 WebTransport 协议基于 HTTP3 并提供单向流、双向流和数据报，并全部复用同一个相同的 HTTP3 链接；

<!--more-->

# 现状

这是一份 Internet 标准跟踪文档。
本文是  Internet Engineering Task Force（IETF）的产品，它代表了 IETF 社区已达成的共识，它已经经历了公开的审阅并已被 Internet Engineering Steering Group (IESG) 批准出版；更多信息可以参考 [Section 2 of RFC 7841](!https://datatracker.ietf.org/doc/html/rfc7841#section-2)。

# 介绍

HTTP3 是在 QUIC [QUIC-TRANSPORT] 之上可以通过 QUIC 链接多路复用 HTTP 请求的协议。本文档定义了一种通过 HTTP3 实现 WebTransport 协议的要求，以实现多路复用 non-HTTP 数据的机制。多个 WebTransport 实例可以同时复用在同一个 HTTP3 链接上进行常规的 HTTP 数据传输。

# 协议概述

WebTransport Server 通常由一对权限值和录制值来标识  (defined in [RFC3986] Sections 3.2 and 3.3   correspondingly)。
当一个 HTTP3 连接后，Client 和 Server 端都必须发送一个 `SETTINGS_ENABLE_WEBTRANSPORT` 配置以表明它们都支持基于 HTTP3 的 WebTransport。WebTransport 会话由 Client 在给定的 HTTP3 连接内发起，即由 Client 发送一个扩展的 CONNECT 请求，如果 Server 接受该请求，则一个 WebTransport session 便建立了；生成的 stream 将被进一步称为  _CONNECT stream_，其 stream ID 用于唯一标识当前给定链接内的 WebTransport session。在给定的 WebTransport session 上建立的 CONNECT stream 的 ID 将被进一步称为 _Session ID_。

会话建立后，双方便可以使用以下机制交换数据：

 - Client 可以创建一个由特殊的不限制长度的 HTTP3 frame 组成的双向流（bidirectional stream），然后转移该流的所有权给 WebTransport；
 - Server 也可以创建一个 bidirectional stream，因为 HTTP3 没有为 Server 端创建 bidirectional stream 定义任何语义（言外之意：不禁止便是允许）；
 - Client 和 Server 都可以创建单向流类型（unidirectional stream）；
 - 可以通过 QUIC DATAGRAM 帧发送数据包 datagram；[QUIC-DATAGRAM](!https://datatracker.ietf.org/doc/html/draft-vvv-webtransport-http3#ref-QUIC-DATAGRAM)

当创建的 CONNECT stream 关闭时，WebTransport 会话也即终止。

# 会话建立

## 建立支持传输的 HTTP3 链接

Client 和 Server 必须均在它们的设置框架中发送一个为 `1` 的值。终端不能使用任何 WebTransport 的任何功能，除非它们已经协商各种参数。
如果协商了 `SETTINGS_ENABLE_WEBTRANSPORT` 参数，则在协商 HTTP3 中 QUIC DATAGRAMS 时，必须按照 [HTTP3-DATAGRAM](!https://datatracker.ietf.org/doc/html/draft-vvv-webtransport-http3#ref-HTTP3-DATAGRAM) 进行协商；否则在协商 WebTransport 时如果没有 QUIC DATAGRAM 扩展，将导致 H3_SETTINGS_ERROR。
HTTP3 要求 Client 的 `initial_max_bidi_streams` 传输参数设置为 `0`。当前的实现是在协商时强制执行这个规定；因此，在 client 发起 bidirectional streams 时，需发送一个非零的 `MAX_STREAMS` 。

## HTTP3 扩展方法 CONNECT

RFC8441 在第 4 节中定义了一个扩展方法 `CONNECT`，可以通过 `SETTINGS_ENABLE_CONNECT_PROTOCOL` 参数激活，该参数是为 HTTP2 定义的，但是该协议不会在 HTTP3 中为 `CONNECT` 创建更多的扩展含义；因为 `SETTINGS_ENABLE_WEBTRANSPORT` 配置已经代表了终端支持扩展的 `CONNECT`。

## 创建会话

由于 WebTransport 会话是通过 HTTP3 建立的，所以它们使用 `https` URI 规则 [RFC 7230](!https://datatracker.ietf.org/doc/html/rfc7230) 。
为了创建一个 WebTransport 会话，client 可以发送一个 HTTP CONNNECT 请求，其 `:protocol` 伪-头域 [RFC8441](!https://datatracker.ietf.org/doc/html/rfc8441) 字段必须设置为 `webtransport`，`:scheme` 字段必须设置为 `https`, 同时 `:authorith`和 `:path` 必须配置，这些字段标识这是一个 WebTransport 服务；一个 `Origin` header 必须包含在这个请求中。

在收到一个 `:protocol` 字段设置为 `webtransport` 的 CONNECT 扩展请求时，HTTP3 服务器可以检查自己是否包含该请求中指定的 `:authorith` 和 `:path` 配置的 WebTransport 服务，如果没有，它应该返回状态码 404；如果有，它可以通过回复状态码 200 标识接受会话请求。WebTransport server 必须验证 `Origin` header 以确保指定 origin 是可以被访问的。

从客户端的角度看，一个 WebTransport 会话建立的标识是 client 收到了 server 的 200 响应；
从服务端的角度来看，一旦发送了 200 响应，就算是建了会话，两端都不能在会话建立之前打开任何 streams 或者发送任何的 datagrams。

**基于 HTTP3 的 WebTransport 不支持 0-RTT。**

## 限制同时会话的数量

从流控的角度来看，WebTransport 会话数量针对stream 的流控就像常规的 HTTP 请求一样（通过 HTTP CONNECT 请求建立连接）。本协议没有引入新的，单独的流控机制，也不将 HTTP 请求与  WebTransport 会话分离，如果服务器需要限制接收请求的速率，它可以使用其它的机制：

 - `HTTP_REQUEST_REJECTED` error code 定义在 [HTTP3](!https://datatracker.ietf.org/doc/html/draft-vvv-webtransport-http3#ref-HTTP3) 标识未处理的请求的 HTTP3 堆栈；

 - HTTP 状态码 429 表示达到了服务器的速率控制，请求被拒绝 [RFC6585](!https://datatracker.ietf.org/doc/html/rfc6585)；与上一个方法不一样，它直接将这个信号通知给发起的请求的应用程序；

# WebTransport 功能列表

WebTransport over HTTP3 提供了以下功能特性：

 - 单向流 unidirectional streams 
 - 双向流 bidirectional streams
 - 数据包 datagrams

以上功能均可由任意一端发起。
Session ID 可用于解复用 Streams 和 Datagrams 以判断属于不同的 WebTransport session，在网络中，session IDs 使用的是 QUIC 可变长度整数方案编码的  [QUIC-TRANSPORT](!https://datatracker.ietf.org/doc/html/draft-vvv-webtransport-http3#ref-QUIC-TRANSPORT)。

## Unidirectional streams

WebTransport 连接一旦建立，两端均可以打开 Unidirectional streams。HTTP3 unidirectional stream type 应为 0x54，整个包体结构依次是 stream type [0x54]、Session ID、已编码的可变整数类型的长度信息、用户指定的流数据，如下：

**Unidirectional WebTransport stream format：**
~~~
     0                   1                   2                   3
     0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |                           0x54 (i)                          ...
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |                        Session ID (i)                       ...
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |                         Stream Body                         ...
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~

## 客户端发起的 Birectional Streams

WebTransport 客户端可以通过打开一个 HTTP3 双向流并发送一个帧类型为 `WEBTRANSPORT_STREAM` (type=0x41) 的 HTTP3 frame 以初始化一个 WebTransport 的双向流。帧的格式为：frame type、Session ID、已编码可变数据长度、用户数据；
数据帧应该持续到流结束。

**WEBTRANSPORT_STREAM frame format:**
~~~
     0                   1                   2                   3
     0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |                           0x41 (i)                          ...
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |                        Session ID (i)                       ...
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |                         Stream Body                         ...
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~

## 服务端发起的 Bidirectional Streams

WebTransport 服务端可以通过打开一个 HTTP3 双向流初始化 Bidirectional Stream。请注意：由于 HTTP3 没有为服务器启动定义任何的 Bidirectional Strream 的相关语义，本协议中此类流适用于所有的 HTTP3 连接，前提是 `SETTINGS_ENABLE_WEBTRANSPORT` 选项已经协商过了。这些流的格式如下：

***Server-initiated bidirectional stream format：**
~~~
     0                   1                   2                   3
     0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |                        Session ID (i)                       ...
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |                         Stream Body                         ...
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~

## 数据包 Datagrams

Datagrams 可以发送   [QUIC-DATAGRAM](!https://datatracker.ietf.org/doc/html/draft-vvv-webtransport-http3#ref-QUIC-DATAGRAM) and [HTTP3-DATAGRAM](!https://datatracker.ietf.org/doc/html/draft-vvv-webtransport-http3#ref-HTTP3-DATAGRAM) 定义的 DATAGRAM frame，前提是 `SETTINGS_ENABLE_WEBTRANSPORT` 已协商，流标识符（Flow Identifier）被设置为 session ID，格式如下：

 **Datagram format：** 
 ~~~
  0                   1                   2                   3
     0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |                        Session ID (i)                       ...
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |                        Datagram Body                        ...
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 ~~~

在 QUIC 中，一个 Datagram 帧最多可以跨越一个数据包中，因为于此，应用程序必须知道可以发送的最大1数据报大小；然而当代理数据报时，逐跳的 MTU 可能并不相同。

# 会话终止

任何一方都可以通过关闭起初通过发送 CONNECT 请求建立起来的流来关闭当前  WebTransport 会话；在得知会话正在终止时，断点必须停止发送新的 datagrams 并且重置所有与此 session 相关联的 sterams。

# 安全注意事项

WebTransport over HTTP3 满足所有对 WebTransport 协议施加的所有安全要求，由于 Client 有可能不被信任，所以提供了一个client-server 通信的安全框架。
WebTransport over HTTP3 需要基于 QUIC 传输参数，这样可以通过 HTTP3 server 的支持来避免潜在的协议混淆带来的攻击；它还需要使用 Origin header，为服务器提供拒绝访问非源自于 Web 客户端的不被信任的 origin。

就像通过 HTTP3 传输的 HTTP 流量一样，WebTransport 流量汇集不同的来源到同一个链接中，不同的源代表不同的信任域，意味着需要在同一个链接上应对不同的潜在的攻击，一种潜在的攻击是耗尽所有资源，因为所有的传输共享拥塞控制和流量控制，单个客户端大量使用这些资源可能会导致其它传输被终止。所以用户代理应该实施一个公平的方案，以确保链接内的每个传输可以获得可控的合理资源份额，这适用于数据发送和创建新流操作。










