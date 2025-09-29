---
layout: post
title: 'RUT protocol design'
subtitle: 'RUT 协议设计'
date: 2025-09-21
author: Mr Chen
# cover: '/assets/img/shan.jpg'
#cover_author: 'rogov'
#cover_author_link: 'https://unsplash.com/@rogovca'
categories: Protocol
tags: 
- Protocol
- RUT
---

## 设计思想

整体参考 quic，将协议格式在传输上进行分层，首先是 Packet 层，每次收发都是一个 Packet，比如用于握手建连的 InitialPacket 和建连后的 DataPacket。
在 Packet 内，是第二层，即 Frame 层，比如包括承载应用层数据的 StreamFrame，以及协议内 CC 用于实时反馈的 FeedbackFrame 和 pingpong frame 等。

## 设计目标

基于 UDP 实现一个更加高效和通用的数据传输协议，既能满足在 RTC 场景下的数据低延迟传输（尽量到达），也能满足在直播、文件传输、IM 等场景下的可靠传输，在一个链接内通过创建不同的 stream，实现同时支持 可靠 和 不可靠传输，实现 stream 的优先级控制，保证高优先级数据的优先传输。

在协议内部整体通过自定义 Congestion contri algorithm 实现更优的数据传输速率的控制，在每条 stream 内使用不同的发送控制策略，以及 Fec 算法，以适配各种不同的应用场景。


## 协议设计

> 我们通过 [该工具 protocol](https://github.com/luismartingarcia/protocol) 来设计协议的具体格式。

## Packet

Packet 主要分为两种，一种为在握手建连时的 InitialPacket，也可以认为是 HandshakePacket，在建链成功以后，便不再有此类数据包，后续全部使用 DataPacket，
用于传输应用层的数据，以及协议内的反馈包。


### Packet Common Header

Packet Common Header 是所有 Packet 的通用包头，包括 InitialPacket 和 DataPacket。
具体格式如下：

```
./protocol "type:1,connection_id:1,path_id:1,zero_rtt:1,cid_existence:1,reserved:3,pkt_no:24,connection_id:64,...:32"
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|t|c|p|z|c|rese.|                     pkt_no                    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                         connection_id                         +
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                              ...                              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

type：packet type,标识此包为 InitialPacket 还是 DataPacket，这里我们做如此约定：0: InitialPacket，1: DataPacket。
connection_id: 此包的包头中是否存在 connection_id 字段, connection_id 是实现连接迁移功能的作用字段，但并不是每个包都必须要带。
path_id: 在此 DataPacket 的 header 中是否存在 Patch Id 字段，用于开启 MultiPath 时标识数据属于哪一条 Path。
zero_rtt: 标识是否为 0-RTT 的数据包，即在 0-RTT 的建链场景中，用于在三次握手尚未完成时，一端已经开始发送数据，以此 flag 告知对端该数据包合法，需先缓存，在三次握手完成后，不再携带此 flag。
cid_existence: 该 Connection 在后续的连接中是否会使用 conenction id。
reserved: 保留位，用于以后能力扩展。
pkt_no：包序号，每次发送均递增。
payload：各种 frames 的集合。
```

### InitialPacket
InitialPacket 用于初始化连接, 握手类似与 TCP 的3次握手，在在一次RTT中完成。

如上约定，InitialPacket 的 Packet Common Header 中的 type 为 0，在 Packet Common Header 之后，才是该数据包，InitialPacket 的具体格式设计如下：

```
./protocol "version:7,sync:1,ack:1,rst:1,early_data:1,tags:1,reserved:10,acked_pkt_no:32,early_data_len:16,early_data:112,tag_count:8,tag_len:16,tag_data:40,tag_len:16,tag_data:58"
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|   version   |s|a|r|e|t|      reserved     |                   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                acked_pkt_no               |   early_data_len  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           |                                                   |
+-+-+-+-+-+-+                                                   +
|                                                               |
+                                                               +
|                           early_data                          |
+                                           +-+-+-+-+-+-+-+-+-+-+
|                                           |   tag_count   |   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          tag_len          |                                   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+               +-+-+-+-+-+-+-+-+-+-+
|                  tag_data                 |      tag_len      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           |                                                   |
+-+-+-+-+-+-+                                                   +
|                            tag_data                           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

version：the corresponding protocol version, curently we use 1；
s：SYN；
a：ACK；
r：RST；
e：标识在该包中是否包含 early_data 数据段，如果包含则在后续的 data 中进行解析。
t：标识在该包中是否包含自定义的 tags，如果包含则在后续的 data 中进行解析。
```

### DataPacket
DataPacket encapsulates all kinds of Frames. Each DataPacket might contains one or more Frames.
DataPacket 封装了所有类型的 Frames，且每个 DataPacket 可以包含多个/种 Frames（至少一个 Frame）。

如上约定，DataPacket 的 Packet Common Header 中的 type 为 1，在 Packet Common Header 之后，才是该数据包，DataPacket 的具体格式设计如下：

```
./protocol "largest_acked_packet_num:24,path_id:4,frame_count:4,type:5,frame_length:11,frame_data:80,...:128,type:5,frame_length:11,frame_data:80"
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|            largest_acked_packet_num           |path_id|frame_.|
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|   type  |     frame_length    |                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+                               +
|                                                               |
+                           frame_data                          +
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                                                               +
|                                                               |
+                              ...                              +
|                                                               |
+                                                               +
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|   type  |     frame_length    |                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+                               +
|                                                               |
+                           frame_data                          +
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

largest_acked_packet_num：本端从远端接收到的最大的 Ack Packet number。
path_id: 当 Pcketet Common Header 中标识 path id 为 true 后，此字段有效，否则无效。
frame_count：在该包的 payload 中种打包了多少个 Frame。
```

## Frame

帧（Frame）是用来在连接建立后传输不同类型的消息的，比如应用层需要传输的数据，协议内部的统计、反馈数据等。

### Frame Common Header

Frame Common Header 是每个数据帧的统一包头。


```
./protocol "type:5,frame_length:11,...:48"

 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|   type  |     frame_length    |                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+                               +
|                            ...                                |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
type: 帧类型
frame_length：帧长度
```

暂时定义如下几种 Frame 类型：

```c++
enum FrameType : uint8_t {
  kStreamFrame = 0,
  kAckFrame = 1,
  kPingFrame = 2,
  kCloseFrame = 3,
  kCongestionFeedbackFrame = 4,
  kControlFrame = 5,
  kPaddingFrame = 6,
  kPathEventFrame = 7,
  kMaxFrameType = 32
};
```

### StreamFrame
StreamFrame 用于承载传输应用层数据.

```
./protocol "stream_id:16,option:1,metadata:1,push:1,reserved:5,writer_flags:8,opt_len:10,opt:40,meta_len:10,meta:48,payload:116"
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           stream_id           |o|m|p| reserved|  writer_flags |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|      opt_len      |                    opt                    |
+-+-+-+-+-+-+-+-+-+-+               +-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                   |      meta_len     |       |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+       +
|                              meta                             |
+                       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       |                                       |
+-+-+-+-+-+-+-+-+-+-+-+-+                                       +
|                                                               |
+                                                               +
|                            payload                            |
+                                                               +
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

stream_id: 流 Id，整个 connection 内唯一。
option：标识是否携带有 option 数据，如果为 true，则后续 "opt_len" & "opt" 字段均存在，用于传递 stream 内的自定义数据，用于协议内。
metadata：标识是否携带有 metadata 数据，如果为 true，则后续 "meta_len" & "meta" 字段有效，用于传递应用层为该 stream 指定的自定义数据。
push: 标识该 frame 是否应该单独发送出去，而非与其它 Frame 组合发送。
writer_flags: stream 使用的 writer 类型， stream 内部实现传输模式。
reserved: no used
```

### AckFrame

AckFrame 用于在接收端对接收的 DataPacket 进行反馈。

```
./protocol "ts:1,ack_delay:8,largest_observed_packet_number:24,largest_observed_recv_time:32,range_length:8,range_count:8,next_range_gap1:8,range_length1:8,...:48,next_range_gapN:8,range_lengthN:8,ts_count:8,delta_number1:8,delta_ms1:8,...:72,delta_numberN:8,delta_msN:8"
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|t|   ack_delay   |        largest_observed_packet_number       |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| |                  largest_observed_recv_time                 |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| |  range_length |  range_count  |next_range_gap1|range_length1|
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| |                             ...                             |
+-+                               +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                 |next_range_gapN|range_lengthN|
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| |    ts_count   | delta_number1 |   delta_ms1   |             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+             +
|                                                               |
+                                                               +
|                              ...                              |
+ +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| | delta_numberN |   delta_msN   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+


ts：表示数据中稍后是否会有数据包接收时间戳。
ack_delay：Acking largest_observed_packet_number 这个包序号的延迟时间，单位 ms。
largest_observed_packet_number：接收到最大的数据包序号。
largest_observed_recv_time：接收到 |largest_observed_packet_number| 包的时间。
range_count：the count of ranges;
next_range_gap：the gap in packet number from last range to current range;
range_length：the length of current range;
ts_count：后续数据中包接收时间戳的数量。
delta_number：从|largest_observed_packet_number|到当前包序号的差值。
delta_ms：从|largest_observed_packet_number|收时间到当前包接收时间的差值。
```

### CloseFrame

CloseFrame用于关闭特定的流或连接。

```
./protocol "stream_id:16,error_code:16,detail:64"
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           stream_id           |           error_code          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                             detail                            +
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

stream_id: 要关闭的流的ID，或使用0xFFFF来关闭连接。
error_code：当前关闭操作的错误码
detail：当前关闭操作的详细原因。
```

### CongestionFeedbackFrame

CongestionFeedbackFrame用于向对端报告一些网络统计信息。

```
./protocol "reserved:9,lost_ratio:7,delta_from_begin_ms:32,estimated_bandwidth_kbps:32,total_sent_kbps:32,outgoing_stream_count:16,jitter95:16,queueing_time:16,stream_id1:16,sent_kbps1:16,...:48,stream_idN:16,sent_kbpsN:16"
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|     reserved    |  lost_ratio |      delta_from_begin_ms      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                               |    estimated_bandwidth_kbps   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                               |        total_sent_kbps        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                               |     outgoing_stream_count     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|            jitter95           |         queueing_time         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           stream_id1          |           sent_kbps1          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                              ...                              |
+                               +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                               |           stream_idN          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           sent_kbpsN          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

reserved: 保留位。
lost_ratio: 本地统计的丢包率。
delta_from_begin_ms: 发送此包时的链接时间。
estimated_bandwidth_kbps: 自行探测到的带宽，需要高于 16bits，否则65Mbps的带宽会被截断。
total_sent_kbps: 本链接共发送的数据量。
outgoing_stream_count: 发送方向创建的流的数量。
jitter95: 本端统计的 jitter95 指标。
queueing_time: 本端发送队列中的数据带以当前 bwe 需要多久时间发送结束。
stream_id1 和 sent_kbps1 用于标识每路流各自的发送速率。
```


### PingFrame
```
./protocol "kPing:5,frame_length:11"
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  kPing  |     frame_length    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

### ControlFrame
```
./protocol "type:8,frame_id:16,stream_id:16,payload:56"
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|      type     |            frame_id           |   stream_id   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|               |                                               |
+-+-+-+-+-+-+-+-+                                               +
|                            payload                            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

type: 表示此控制帧的类型。
frame_id: 表示此帧的ID，每个控制帧都有唯一的ID。
stream_id: 表示此控制帧的流ID。
payload: 表示此控制帧的载荷，长度包含在frame_length中。
```

其中control frame 中的  |type| 暂先定义如下类型:

```c++
enum class ControlFrameType : uint8_t {
  kOptionFrame = 0, // 主要用作在传输过程中，对对端做远程的控制

  kMax = 256,
};
```