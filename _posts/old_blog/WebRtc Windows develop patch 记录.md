---
title: WebRtc Windows develop patch 记录
tags: WebRTC windows
categories: RTC
abbrlink: 4152150583
date: 2018-09-12 06:13:22
---

# 编译

## 1、WebRtc 源码升级 m63->m68 

升级过程中主要遇到两种编译问题：一是 webrtc 本身在同步、编译过程中有问题，另外就是在使用 `webrtc.lib` 时有报错；
- gclient sync  一直报：下载失败，或其他失败

<!--more-->

> 这是因为 python 的缓冲中还留有部分旧版本的下载地址信息，导致下载失败，或者下载后版本不对应，删除当前用户目录下的 `.vpython_cipd_cache` 和 `.vpython-root`，然后再执行 `gclient sync ` 即可;

- SDK 编译链接时报无 `builtin_audio_decoder_factory` 相关符号

> 目前出现此问题的原因未知，不过 builtin_audio_decoder_factory 符号已编译，但 audio.lib（SDK 依赖 webrtc.lib, webrtc.lib 依赖 audio.lib）确没有把 builtin_audio_decoder_factory 符号包含进去，通过修改 `audio/BUILD.gn` 脚本，在其 line：58 添加一行依赖即可：`../api/audio_codecs:builtin_audio_decoder_factory`

# WebRtc 内部 Bug

## Adm 创建失败，导致 crash

>由于 webrtc 内部会对所有设备进行一下测试，有的设备可能测试是失败（但还是可以正常采集），测试失败的话，webrtc  默认会返回 false，导致创建的 adm 为 null，而其内部却没有对这种情况做兼容，导致 crash，修复方法：在 `modules/audio_device/win/audio_device_core_win.cc` 的 line：352 添加一行：`ok = 0;` ,即忽略测试失败的情况；

