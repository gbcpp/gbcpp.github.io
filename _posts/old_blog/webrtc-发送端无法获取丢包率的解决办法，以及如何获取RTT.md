---
title: WebRTC 发送端无法获取丢包率的解决办法，以及如何获取 rtt
categories: RTC
tags: WebRTC RTT 丢包率
abbrlink: 21d9b983
date: 2019-05-07 14:49:21
---

WebRTC 自 M68 版本推出新的重载方法 `GetStats` 以后，使用新的接口便再也获取不到发送端的丢包率了（包括所有的 Native 端和 Web 端），原因是在 `void OnStatsDelivered(const rtc::scoped_refptr<const RTCStatsReport>& report)` 回调中返回的结构体 `RTCStatsReport` 中没有 `packets_lost` 的定义，只有 `packets_sent`、`frames_encoded` 等信息，默认只能获取到发送的码率、帧率，而得不到丢包率、rtt 等判断网络质量的关键信息。不过 WebRTC 底层其实已经将丢包信息、rtt 等关键信息实时统计上来了，只是 `PeerConnectionInterface` 在获取 Statistics 时，没有将相关信息打包进 `RTCStatsReport` 结构体中，那么解决办法就是：在 `RTCStatsReport` 结构体中新增定义 `packets_lost`、`rtt_ms` 字段，并为其赋值即可，下面为 Native 源码中需要修改的相关代码。

<!--more-->

## 增加 `RTCOutboundRTPStreamStats` 导出结构体字段定义

数据发送方数据的统计记录在 Outbound 的结构中，即：`RTCOutboundRTPStreamStats`，为其在末尾新增两个字段：

> 文件为： api/stats/rtcstats_objects.h 

```c++
class RTCOutboundRTPStreamStats final : public RTCRTPStreamStats {
 public:
  WEBRTC_RTCSTATS_DECL();

  RTCOutboundRTPStreamStats(const std::string& id, int64_t timestamp_us);
  RTCOutboundRTPStreamStats(std::string&& id, int64_t timestamp_us);
  RTCOutboundRTPStreamStats(const RTCOutboundRTPStreamStats& other);
  ~RTCOutboundRTPStreamStats() override;

  RTCStatsMember<uint32_t> packets_sent;
  RTCStatsMember<uint64_t> bytes_sent;
  // TODO(hbos): Collect and populate this value. https://bugs.webrtc.org/7066
  RTCStatsMember<double> target_bitrate;
  RTCStatsMember<uint32_t> frames_encoded;
    // 新增字段
  RTCStatsMember<int32_t> packets_lost;
  RTCStatsMember<int32_t> rtt_ms;
};
```

## 初始化新字段

> 文件为：stats/rtcstats_objects.cc

- 构造函数的初始化列表增加 `RTCStatsMember` 键值对的初始化

```c++
RTCOutboundRTPStreamStats::RTCOutboundRTPStreamStats(
    std::string&& id, int64_t timestamp_us)
    : RTCRTPStreamStats(std::move(id), timestamp_us),
      packets_sent("packetsSent"),
      bytes_sent("bytesSent"),
      target_bitrate("targetBitrate"),
      frames_encoded("framesEncoded"),
      packets_lost("packetsLost"),
      rtt_ms("rttMs") {
}
```

- 拷贝构造函数初始化

```c++
RTCOutboundRTPStreamStats::RTCOutboundRTPStreamStats(
    const RTCOutboundRTPStreamStats& other)
    : RTCRTPStreamStats(other),
      packets_sent(other.packets_sent),
      bytes_sent(other.bytes_sent),
      target_bitrate(other.target_bitrate),
      frames_encoded(other.frames_encoded),
      packets_lost(other.packets_lost),
      rtt_ms(other.rtt_ms) {
}
```

- `outbound-rtp` 中增加 Json 导出字段

> 如果不增加，在上层 ToJoin 时是没有 `packetLost` 和 `rttMs` 字段的。

```c++
// clang-format off
WEBRTC_RTCSTATS_IMPL(
    RTCOutboundRTPStreamStats, RTCRTPStreamStats, "outbound-rtp",
    &packets_sent,
    &bytes_sent,
    &target_bitrate,
    &frames_encoded,
    &packets_lost,
    &rtt_ms);
// clang-format on
```

## 为新字段赋值

```c++
// Provides the media independent counters (both audio and video).
void SetOutboundRTPStreamStatsFromMediaSenderInfo(
    const cricket::MediaSenderInfo& media_sender_info,
    RTCOutboundRTPStreamStats* outbound_stats) {
  RTC_DCHECK(outbound_stats);
  outbound_stats->ssrc = media_sender_info.ssrc();
  // TODO(hbos): Support the remote case. https://crbug.com/657856
  outbound_stats->is_remote = false;
  outbound_stats->packets_sent =
      static_cast<uint32_t>(media_sender_info.packets_sent);
  outbound_stats->bytes_sent =
      static_cast<uint64_t>(media_sender_info.bytes_sent);
  
  // 为新字段赋值
  outbound_stats->packets_lost =
      static_cast<int32_t>(media_sender_info.packets_lost);
  outbound_stats->rtt_ms =
      static_cast<int32_t>(media_sender_info.rtt_ms);
}
```

## 验证

通过 `PeerConnectionInterface` 的 `GetStats` 获取到的 `RTCStatsReport` 结构体 转换为 Json 字符串后，可以发现 `outbound-rtp` 节点中已经有了以上我们新增的 `packetsLost` 和 `rttMs` 两个字段。

```json
[
    ..... 其它节点请忽略，这里只展示 outbound-rtp
    {
        "type": "outbound-rtp",
        "id": "RTCOutboundRTPAudioStream_856432637",
        "timestamp": 1557220118008000,
        "ssrc": 856432637,
        "isRemote": false,
        "mediaType": "audio",
        "trackId": "RTCMediaStreamTrack_sender_2",
        "transportId": "RTCTransport_audio_1",
        "codecId": "RTCCodec_audio_Outbound_111",
        "packetsSent": 484,
        "bytesSent": 81000,
        "packetsLost": 0,
        "rttMs": 13
    },
    {
        "type": "outbound-rtp",
        "id": "RTCOutboundRTPVideoStream_4038037569",
        "timestamp": 1557220118008000,
        "ssrc": 4038037569,
        "isRemote": false,
        "mediaType": "video",
        "trackId": "RTCMediaStreamTrack_sender_1",
        "transportId": "RTCTransport_audio_1",
        "codecId": "RTCCodec_video_Outbound_100",
        "firCount": 0,
        "pliCount": 1,
        "nackCount": 0,
        "qpSum": 2883,
        "packetsSent": 559,
        "bytesSent": 579220,
        "framesEncoded": 145,
        "packetsLost": 0,
        "rttMs": 13
    }
]
```

解析 Json 字符串获取 `packetsLost` 前后两次的差值，除以时间差便可以获取到此时间段内的丢包率了。

