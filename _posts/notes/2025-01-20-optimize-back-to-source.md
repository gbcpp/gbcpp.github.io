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

# SRS 现状

&emsp;&emsp;那么问题来了， srs 是否已经考虑到这一点的优化空间呢？

&emsp;&emsp;对比查看 srs 是否使用的这种透明转发模式，还是帧转发模式？
这里记录的为使用 HTTP-FLV 回源协议的代码流程。

srs 需要回源时建议优先使用 Edge 模式，回源时，内部首先创建一个 `SrsEdgeFlvUpstream` 实例，用于向源站发起连接，并用于实时的接收、处理数据，代码文件：`srs_app_edge.cpp`。

通过 `connect` ==> `do_connect` 发起向源站的连接请求，判定 http response status code 非 404、非 302 后，开始接收、处理媒体数据。

事件轮询在 `SrsEdgeIngester::ingest()` 中，同步模式不断循环执行 `SrsEdgeFlvUpstream::recv_message()` 接收完整的数据，而后通过 `SrsEdgeIngester::process_publish_message` 将数据流转到后序处理的模块，比如 Source、Consumer 等，但在这之前重点是确认在 recv_message 中是否会 block 等待完整的一个 tag data。

```cpp
srs_error_t SrsEdgeFlvUpstream::recv_message(SrsCommonMessage** pmsg)
{
    srs_error_t err = srs_success;

    char type;
    int32_t size;
    uint32_t time;
    if ((err = decoder_->read_tag_header(&type, &size, &time)) != srs_success) {
        return srs_error_wrap(err, "read tag header");
    }

    char* data = new char[size];
    if ((err = decoder_->read_tag_data(data, size)) != srs_success) {
        srs_freepa(data);
        return srs_error_wrap(err, "read tag data");
    }

    char pps[4];
    if ((err = decoder_->read_previous_tag_size(pps)) != srs_success) {
        return srs_error_wrap(err, "read pts");
    }

    int stream_id = 1;
    SrsCommonMessage* msg = NULL;
    if ((err = srs_rtmp_create_msg(type, time, data, size, stream_id, &msg)) != srs_success) {
        return srs_error_wrap(err, "create message");
    }

    *pmsg = msg;

    return err;
}
```

可以看到首先读取 tag_header，然后从header 中获取 tag size 开始读取指定size 的tag data，`SrsFlvDecoder::read_tag_data()` ==>> `SrsHttpFileReader:read()`

可以看到在 `SrsHttpFileReader::read(void* buf, size_t count, ssize_t* pnread)` 中其会持续的尝试去接收指定 size 大小的数据，直到接收完整，或者 io 报错，源码如下：

```cpp
srs_error_t SrsHttpFileReader::read(void* buf, size_t count, ssize_t* pnread)
{
    srs_error_t err = srs_success;
    
    if (http->eof()) {
        return srs_error_new(ERROR_HTTP_REQUEST_EOF, "EOF");
    }
    
    int total_read = 0;
    while (total_read < (int)count) {
        ssize_t nread = 0;
        if ((err = http->read((char*)buf + total_read, (int)(count - total_read), &nread)) != srs_success) {
            return srs_error_wrap(err, "read");
        }
        
        if (nread == 0) {
            err = srs_error_new(ERROR_HTTP_REQUEST_EOF, "EOF");
            break;
        }
        
        srs_assert(nread);
        total_read += (int)nread;
    }
    
    if (pnread) {
        *pnread = total_read;
    }
    
    return err;
}
```

**结论：** 当前 srs 使用的是帧级别的转发模式，没有实现分片的透明转发。


