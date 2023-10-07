---
title: webrtc on linux alsa audio device blocking
tags: WebRTC Linux alsa
categories: RTC
abbrlink: a2661658
date: 2019-04-16 11:56:05
---

基于 webrtc 开发嵌入式 linux 实时音视频，其中音频 alsa 模块存在一个不小的 Bug，在较多的声卡上存在播放失败并阻塞的严重问题，问题如下：
存在大概率的声卡打开失败，即 `snd_pcm_prepare` 失败的概率，在 源码中 webrtc 会尝试通过 `ErrorRecovery` 恢复声卡至正确的状态，但是执行到 `snd_pcm_recover` 输出 `underrun occurred` 错误信息后便阻塞了。

<!--more-->

原因在于 `snd_pcm_recover` 源码中会调动 `snd_pcm_prepare` 方法尝试恢复声卡状态，但是 `snd_pcm_prepare` 会首先获取 `snd_pcm_t` 中的线程锁 `pthread_mutex_t`，而 webrtc 在 `AudioDeviceLinuxALSA::StartPlayout()` 方法中是先启动播放线程，然后再调用 `snd_pcm_prepare` 准备好声卡设备，导致线程有一定概率启动速度早于父线程中对 `snd_pcm_prepare` 的调用，最终播放线程中在执行 `snd_pcm_avail_update` 失败，触发 `ErrorRecovery` 方法，内部有线程锁，导致线程死锁，整个 APP Blocking；

修复方式比较简单，修改 `audio_device_alsa_linux.cc` 文件中的 `StartPlayout` 方法如下：

~~~cpp
int32_t AudioDeviceLinuxALSA::StartPlayout() {
  if (!_playIsInitialized) {
    return -1;
  }

  if (_playing) {
    return 0;
  }

  _playing = true;

  _playoutFramesLeft = 0;
  if (!_playoutBuffer)
    _playoutBuffer = new int8_t[_playoutBufferSizeIn10MS];
  if (!_playoutBuffer) {
    RTC_LOG(LS_ERROR) << "failed to alloc playout buf";
    _playing = false;
    return -1;
  }

  int errVal = LATE(snd_pcm_prepare)(_handlePlayout);
  if (errVal < 0) {
    RTC_LOG(LS_ERROR) << "playout snd_pcm_prepare failed ("
                      << LATE(snd_strerror)(errVal) << ")\n";
    // just log error
    // if snd_pcm_open fails will return -1
  }
  // 将线程的启动放到 snd_pcm_prepare 之后再启动
  // PLAYOUT
  _ptrThreadPlay.reset(new rtc::PlatformThread(
      PlayThreadFunc, this, "webrtc_audio_module_play_thread"));
  _ptrThreadPlay->Start();
  _ptrThreadPlay->SetPriority(rtc::kRealtimePriority);

  return 0;
}
~~~

> **StartRecording 最好也调整下调用顺序。**