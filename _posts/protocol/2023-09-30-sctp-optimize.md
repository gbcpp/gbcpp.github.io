---
layout: post
title: 'RtcDataChannel(sctp) protocol optimize'
subtitle: '实现 RTCDataChannel 类似于裸 UDP 的传输效率'
date: 2023-09-30
author: Mr Chen
# cover: '/assets/img/shan.jpg'
#cover_author: 'rogov'
#cover_author_link: 'https://unsplash.com/@rogovca'
tags: 
- 协议
---

# DataChannel 实现
DataChannel 的实现有两种，webrtc 目前使用自行基于 C++ 开发的 dcsctp，mediasoup 和 libdatachannel 基于 usrsctp，usrsctp 为开源 sctp 协议实现，均遵守 rfc 4960 标准。
RFC：https://tools.ietf.org/html/rfc4960

# 能力概述
DataChannel (SCTP）支持可靠传输、不可靠传输两种模式，其中不可靠传输有两种方式：一种基于配置包最大生命周期进行配置，另一种是基于最大重传次数进行配置。

如若使用可靠传输只需将Client 端的 reliable 配置为 true 即可，Server 的 usrsctp 中的 `spa.sendv_prinfo.pr_policy` 配置为  `SCTP_PR_SCTP_NONE` 即可；

如若需要使用不可靠传输模式，则分以下两种方式：基于生存时间和重传次数，二者只能选其一：
基于最大生命周期 `maxRetransTime` 为数据报在 sctp 队列中最大存活时间，该参数可以保证发送队列可以快速的情况，尽量降低后续包发送失败的概率，但是存在在网络拥塞的情况下，因 CC 控制导致数据报尚未发送便被清理的问题，即数据包没有经过一次发送便因超时而被清理了。

基于最大重传次数 `maxRetrans` 为数据报在发送队列中最多重传的次数，可保证数据至少在网络中发送一次，但如若出现了网络丢包，不会进行重传；注意：在 usrsctp 中，此参数至少要配置为 1 才能保证至少发送一次，与 client 不同。

# 性能
两者默认都较弱，不论是其可靠传输，还是不可靠传输，抗弱网能力均较弱，但测试下来 usrsctp 比 dcsctp 要强一些，不过 usrsctp 基于 c 开发，代码缩写、简写严重，难以阅读理解，而 dcsctp 基于 C++ 实现，相对要容易理解一些；

# 弱网性能
在可靠传输模式下，30% 的丢包即为上限，整体还不如 tcp，具体数据暂不列了，更高的丢包率测试无法进行下去。

## 原因分析

### 发送窗口

sctp 在拥塞控制算法上与 tcp 类似，分为慢启动、拥塞避免、快速恢复 和快速重传几个阶段。
在出现弱网，即出现 丢包 或者 重传超时的情况时，发送窗口 cwnd 和 慢启动阈值 ssthresh 减半，降低过快，同时其 cwnd 最低值过低，首先从这一点进行入手分析。
以下 Url 是 RFC 中对 CWnd 和 SStreash 的配置策略：
	https://datatracker.ietf.org/doc/html/rfc4960#section-6.3.3
	https://datatracker.ietf.org/doc/html/rfc4960#section-7.2.3
原文如下：
```
7.2.3.  Congestion Control

   Upon detection of packet losses from SACK (see Section 7.2.4), an
   endpoint should do the following:

      ssthresh = max(cwnd/2, 4*MTU)
      cwnd = ssthresh
      partial_bytes_acked = 0

   Basically, a packet loss causes cwnd to be cut in half.

   When the T3-rtx timer expires on an address, SCTP should perform slow
   start by:

      ssthresh = max(cwnd/2, 4*MTU)
      cwnd = 1*MTU

   and ensure that no more than one SCTP packet will be in flight for
   that address until the endpoint receives acknowledgement for
   successful delivery of data to that address.
```
大致意思如下：
在检测到丢包时，更新 sstresh 和 cwnd 机制如下：
对原 ssthresh 减半，但最大为 4 个 MTU 大小，这个是致命的，同时将 cwnd 更新为 ssthresh；
当 T3-rtx 重传定时器超时时：
对原 ssthresh 减半，但最大为 4 个 MTU 大小，但对 cwnd 更新为仅 1 个 MTU。
通过上述两种机制，可以看到，在高丢包率的弱网环境中，很容易进入到上述两种 case 中，结合 flighting 的数据大小，导致 sctp 协议可发送数据量奇小 （cwnd 过小），同时因为重传包的优先级高于新生成的数据包，导致协议唯一的一个发送窗口一直在重传旧包，新包很难获取发送机会，这就导致了明显的队头阻塞现象。
解决方案：目标只是为了不使 CWnd 的大小降低的过快，即在检测到丢包和 T3-rtx 超时时，不要立即对 ssthresh 减半，而是更加平滑的去降低，并且提高最小值的上限，通过这两种策略的优化，可以保证 sctp 协议有很高的带宽抢占能力，因为发送窗口越大，带宽抢占能力就越高。

### 清空队列

通过上述策略优化发送窗口后，可以大幅度提高 sctp 在弱网（双向各30%丢包率）下的带宽传输能力，但是在更极端的网损（40%丢包，甚至更高）场景下，依然会出现队头阻塞现象，发送端不再主动发送新插入的数据，而是优先发送旧包（未收到 Ack），在实际应用场景中，我们希望不论是否收到对端的确认，都不应该阻塞新包的发送，和旧包的清理，以保证发送队列有一个快速的清理速度，且保证所有的数据均有机会被快速的发送到网络中去。
如果双方均为开源的实现，我们可以随意修改 sctp，可快速解决上述问题，但是当一端为浏览器时，我们只能考虑修改 Server 侧的协议实现，以期可以间接的影响到对端（浏览器），让对端（浏览器）最为发送端时，可以快速的清空发送队列，避免队头阻塞，答案就在 SACK 包中，首先看下 SACK 包的协议定义：
https://datatracker.ietf.org/doc/html/rfc4960#section-3.3.4

```
3.3.4.  Selective Acknowledgement (SACK) (3)

   This chunk is sent to the peer endpoint to acknowledge received DATA
   chunks and to inform the peer endpoint of gaps in the received
   subsequences of DATA chunks as represented by their TSNs.

   The SACK MUST contain the Cumulative TSN Ack, Advertised Receiver
   Window Credit (a_rwnd), Number of Gap Ack Blocks, and Number of
   Duplicate TSNs fields.

   By definition, the value of the Cumulative TSN Ack parameter is the
   last TSN received before a break in the sequence of received TSNs
   occurs; the next TSN value following this one has not yet been
   received at the endpoint sending the SACK.  This parameter therefore
   acknowledges receipt of all TSNs less than or equal to its value.

   The handling of a_rwnd by the receiver of the SACK is discussed in
   detail in Section 6.2.1.

   The SACK also contains zero or more Gap Ack Blocks.  Each Gap Ack
   Block acknowledges a subsequence of TSNs received following a break
   in the sequence of received TSNs.  By definition, all TSNs
   acknowledged by Gap Ack Blocks are greater than the value of the
   Cumulative TSN Ack.


        0                   1                   2                   3
        0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |   Type = 3    |Chunk  Flags   |      Chunk Length             |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |                      Cumulative TSN Ack                       |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |          Advertised Receiver Window Credit (a_rwnd)           |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       | Number of Gap Ack Blocks = N  |  Number of Duplicate TSNs = X |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |  Gap Ack Block #1 Start       |   Gap Ack Block #1 End        |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       /                                                               /
       \                              ...                              \
       /                                                               /
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |   Gap Ack Block #N Start      |  Gap Ack Block #N End         |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |                       Duplicate TSN 1                         |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       /                                                               /
       \                              ...                              \
       /                                                               /
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |                       Duplicate TSN X                         |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

   Chunk Flags: 8 bits

      Set to all '0's on transmit and ignored on receipt.

   Cumulative TSN Ack: 32 bits (unsigned integer)

      This parameter contains the TSN of the last DATA chunk received in
      sequence before a gap.  In the case where no DATA chunk has been
      received, this value is set to the peer's Initial TSN minus one.

   Advertised Receiver Window Credit (a_rwnd): 32 bits (unsigned
   integer)

      This field indicates the updated receive buffer space in bytes of
      the sender of this SACK; see Section 6.2.1 for details.

   Number of Gap Ack Blocks: 16 bits (unsigned integer)

      Indicates the number of Gap Ack Blocks included in this SACK.

   

   Number of Duplicate TSNs: 16 bit

      This field contains the number of duplicate TSNs the endpoint has
      received.  Each duplicate TSN is listed following the Gap Ack
      Block list.

   Gap Ack Blocks:

      These fields contain the Gap Ack Blocks.  They are repeated for
      each Gap Ack Block up to the number of Gap Ack Blocks defined in
      the Number of Gap Ack Blocks field.  All DATA chunks with TSNs
      greater than or equal to (Cumulative TSN Ack + Gap Ack Block
      Start) and less than or equal to (Cumulative TSN Ack + Gap Ack
      Block End) of each Gap Ack Block are assumed to have been received
      correctly.

   Gap Ack Block Start: 16 bits (unsigned integer)

      Indicates the Start offset TSN for this Gap Ack Block.  To
      calculate the actual TSN number the Cumulative TSN Ack is added to
      this offset number.  This calculated TSN identifies the first TSN
      in this Gap Ack Block that has been received.

   Gap Ack Block End: 16 bits (unsigned integer)

      Indicates the End offset TSN for this Gap Ack Block.  To calculate
      the actual TSN number, the Cumulative TSN Ack is added to this
      offset number.  This calculated TSN identifies the TSN of the last
      DATA chunk received in this Gap Ack Block.
```

每一个 SACK 包中，均包括一个字段：`Cumulative TSN`，该字段标识接收端已确认（通知给 Application）的最大包序号（TSN），通过此字段告诉发送端，不要再发送小于此 TSN 的包，并清空所有队列中，小于等于此 TSN 的数据包，其它的字段均不是最重要的，可以自行决定是否调整。
只有当发送端不断的接收到接收端发来的 SACK 包后，通过其携带的强制前移的 `Cumulative TSN` 来清理本地的发送队列，可以保证自己的发送队列一直处于一个较小的 Cache，这样，新插入的数据包便可以被快速的发送出去，避免了因旧包未被 Ack 而导致的队头阻塞问题，更加的贴近 UDP 的行为。

Ok，那么回溯下这个问题的上一步：为什么发送端无法快速的清空自己的发送队列的呢？我们看下 RFC 的定义便知，https://datatracker.ietf.org/doc/html/rfc4960#section-6.1
原文如下：

```
6.1.  Transmission of DATA Chunks

   This document is specified as if there is a single retransmission
   timer per destination transport address, but implementations MAY have
   a retransmission timer for each DATA chunk.

   The following general rules MUST be applied by the data sender for
   transmission and/or retransmission of outbound DATA chunks:

   A) At any given time, the data sender MUST NOT transmit new data to
      any destination transport address if its peer's rwnd indicates
      that the peer has no buffer space (i.e., rwnd is 0; see Section
      6.2.1).  However, regardless of the value of rwnd (including if it
      is 0), the data sender can always have one DATA chunk in flight to
      the receiver if allowed by cwnd (see rule B, below).  This rule
      allows the sender to probe for a change in rwnd that the sender
      missed due to the SACK's having been lost in transit from the data
      receiver to the data sender.

      When the receiver's advertised window is zero, this probe is
      called a zero window probe.  Note that a zero window probe SHOULD
      only be sent when all outstanding DATA chunks have been
      cumulatively acknowledged and no DATA chunks are in flight.  Zero
      window probing MUST be supported.

      If the sender continues to receive new packets from the receiver
      while doing zero window probing, the unacknowledged window probes
      should not increment the error counter for the association or any
      destination transport address.  This is because the receiver MAY
      keep its window closed for an indefinite time.  Refer to Section
      6.2 on the receiver behavior when it advertises a zero window.
      The sender SHOULD send the first zero window probe after 1 RTO
      when it detects that the receiver has closed its window and SHOULD
      increase the probe interval exponentially afterwards.  Also note
      that the cwnd SHOULD be adjusted according to Section 7.2.1.  Zero
      window probing does not affect the calculation of cwnd.

      The sender MUST also have an algorithm for sending new DATA chunks
      to avoid silly window syndrome (SWS) as described in [RFC0813].
      The algorithm can be similar to the one described in Section
      4.2.3.4 of [RFC1122].

      However, regardless of the value of rwnd (including if it is 0),
      the data sender can always have one DATA chunk in flight to the
      receiver if allowed by cwnd (see rule B below).  This rule allows
      the sender to probe for a change in rwnd that the sender missed
      due to the SACK having been lost in transit from the data receiver
      to the data sender.

   B) At any given time, the sender MUST NOT transmit new data to a
      given transport address if it has cwnd or more bytes of data
      outstanding to that transport address.

   C) When the time comes for the sender to transmit, before sending new
      DATA chunks, the sender MUST first transmit any outstanding DATA
      chunks that are marked for retransmission (limited by the current
      cwnd).

   D) When the time comes for the sender to transmit new DATA chunks,
      the protocol parameter Max.Burst SHOULD be used to limit the
      number of packets sent.  The limit MAY be applied by adjusting
      cwnd as follows:

      if((flightsize + Max.Burst*MTU) < cwnd) cwnd = flightsize +
      Max.Burst*MTU

      Or it MAY be applied by strictly limiting the number of packets
      emitted by the output routine.

   E) Then, the sender can send out as many new DATA chunks as rule A
      and rule B allow.

   Multiple DATA chunks committed for transmission MAY be bundled in a
   single packet.  Furthermore, DATA chunks being retransmitted MAY be
   bundled with new DATA chunks, as long as the resulting packet size
   does not exceed the path MTU.  A ULP may request that no bundling is
   performed, but this should only turn off any delays that an SCTP
   implementation may be using to increase bundling efficiency.  It does
   not in itself stop all bundling from occurring (i.e., in case of
   congestion or retransmission).

   Before an endpoint transmits a DATA chunk, if any received DATA
   chunks have not been acknowledged (e.g., due to delayed ack), the
   sender should create a SACK and bundle it with the outbound DATA
   chunk, as long as the size of the final SCTP packet does not exceed
   the current MTU.  See Section 6.2.

   IMPLEMENTATION NOTE: When the window is full (i.e., transmission is
   disallowed by rule A and/or rule B), the sender MAY still accept send
   requests from its upper layer, but MUST transmit no more DATA chunks
   until some or all of the outstanding DATA chunks are acknowledged and
   transmission is allowed by rule A and rule B again.

   Whenever a transmission or retransmission is made to any address, if
   the T3-rtx timer of that address is not currently running, the sender
   MUST start that timer.  If the timer for that address is already
   running, the sender MUST restart the timer if the earliest (i.e.,
   lowest TSN) outstanding DATA chunk sent to that address is being
   retransmitted.  Otherwise, the data sender MUST NOT restart the
   timer.

   When starting or restarting the T3-rtx timer, the timer value must be
   adjusted according to the timer rules defined in Sections 6.3.2 and
   6.3.3.

   Note: The data sender SHOULD NOT use a TSN that is more than 2**31 -
   1 above the beginning TSN of the current send window.
```

根据上述定义，再结合上面 CWND 的更新策略，可以推测出，在高丢包的弱网情况下，Sender 很容易 Trigger 因连续未收到 Receiver 的 Ack，同时自己的 Inflighting 数据量较多，导致自己可发送的数据量非常小（最小为 1 个 MTU），甚至发送 0 Bytes 的探测包代替，从而出现严重的对头阻塞问题。

> 但是按照上述方案通过接收端强制偏移 TSN，以达到发送端快速清空发送队列的目标时，DataChannel/SCTP 的可靠传输能力便丧失了。


### SACK Chunk 源码

首先整理下 sctp 协议内接收端组装sack 的数据结构 （https://tools.ietf.org/html/rfc4960#section-3.3.4），在 dcsctp 的代码中比较易懂，定义如下：

```c++
class SackChunk : public Chunk, public TLVTrait<SackChunkConfig> {
 public:
  static constexpr int kType = SackChunkConfig::kType;

  // 可以看出包序号使用的是 uint16_t，只需两个字节  
  struct GapAckBlock {
    GapAckBlock(uint16_t start, uint16_t end) : start(start), end(end) {}

    uint16_t start;
    uint16_t end;

    bool operator==(const GapAckBlock& other) const {
      return start == other.start && end == other.end;
    }
  };

  SackChunk(TSN cumulative_tsn_ack,
            uint32_t a_rwnd,
            std::vector<GapAckBlock> gap_ack_blocks,
            std::set<TSN> duplicate_tsns)
      : cumulative_tsn_ack_(cumulative_tsn_ack),
        a_rwnd_(a_rwnd),
        gap_ack_blocks_(std::move(gap_ack_blocks)),
        duplicate_tsns_(std::move(duplicate_tsns)) {}
  static absl::optional<SackChunk> Parse(rtc::ArrayView<const uint8_t> data);

  void SerializeTo(std::vector<uint8_t>& out) const override;
  std::string ToString() const override;
  TSN cumulative_tsn_ack() const { return cumulative_tsn_ack_; }
  uint32_t a_rwnd() const { return a_rwnd_; }
  rtc::ArrayView<const GapAckBlock> gap_ack_blocks() const {
    return gap_ack_blocks_;
  }
  const std::set<TSN>& duplicate_tsns() const { return duplicate_tsns_; }

 private:
  static constexpr size_t kGapAckBlockSize = 4;
  static constexpr size_t kDupTsnBlockSize = 4;

  const TSN cumulative_tsn_ack_;             // 最大已确认包序号，TSN 即为内部传输块的序号
  const uint32_t a_rwnd_;                    // 接收窗口的大小
  std::vector<GapAckBlock> gap_ack_blocks_;  // GAP 包序号范围，一次聚合多个
  std::set<TSN> duplicate_tsns_;             // 收到的已确认的包序号
};
```

这里，可能有些疑问，每次打包的 SACK 中，其 gap_ack_blocks_ 和 duplicate_tsns_ 是否有最大个数/长度限制，因为它们两个毕竟是要全部一个个的序列化到网络包中的，答案是：有的。
两者 gap_ack_blocks_ 和 duplicate_tsns_ 单次限制上限为 20 个，kMaxGapAckBlocksReported 和 kMazhimaxDuplicateTsnReported 均定义为 constexpr 值：20，声明在代码文件：data_tracker.h中。

### SACK 发包时机

在 dcsctp 的实现中，sack 的发送时机是动态的。在没有数据包丢失时，每秒钟发送一次，当监测到丢包时，针对每个数据包发送一次 sack；当不直接发送 SACKS 时，将使用 timer 控制 sack 的延迟发送，比如 min(RTO/2，200ms)。

发送端接收到对端回应的 SACK 后，判断如果其 cumulative tsn 比本地记录的最大的 last cumulative tsn 要大（即没有回退），则重启 ts_rtx_ 定时器。


# 遗留问题

## RTT 计算

有明显问题，如下两种：
在 dcsctp 中看到 RTT 的计算是有漏洞的，其在 sack 中没有 gap_ack_blocks 时，通过 cumulative tsn 计算其 rtt，存在两种风险:
- 这个 sack 整体是有延迟的，其计算 rtt 时，是直接通过 sendts - current ts，明显没有考虑 delay ack time；
- 其只有在没有 gap_ack_blocks 时，即没有出现丢包时才统计其 RTT，这样在丢包严重时，可能会出现长时间无法更新 rtt，或者在 rtt 和 loss 突然增大时，出现长时间无法更新 rtt 的情况，一直以旧的 rtt 为准。
