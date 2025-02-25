---
layout: post
title: 'HTTP3/QUIC 在 个别 iPhone 下卡顿问题排查记录'
subtitle: 
date: 2025-02-24
author: Mr Chen
cover: '/assets/img/shan.jpg'
categories: Notes
tags: 
- Protocol
---


## 问题描述

&emsp;&emsp;直播的边缘服务器开启 HTTP3 以后，出现 iPhone 的部分机型明显卡顿的问题，而 HTTP2 确不会，明显不符合预期。正常应该是使用了 QUIC 的 HTTP3 的传输效果优于 TCP 才对，这才是大部分机型的表现。


## 排查过程

### 协议对比

在指定的 iPhone 设备上通过后台控制、确认播放端使用的 HTTP 版本，明确在正常的网络环境下使用 TCP 的播放效果要明显优于基于 UDP 的 HTTP3，明显问题现象。

### MTU 问题

通过打开 quic-go 中的 debug log，看到播放端在出现卡顿时，出现大概率的 `lost packet`，怀疑是 quic-go 的 MTU 探测问题，探测到错误的 max packet size 导致发送了过大的数据包触发的 IP 分片导致的丢失。

&emsp;&emsp;。。。。。。&emsp;&emsp; 此处省略一万字，debug 排查 guic-go 中的 MTU 探测模块代码逻辑简单、清晰，没有代码问题。



那么如果将 quic-go 中的 MTU 探测关闭，让其一直固定使用 initial 的 max packet size：1252，会有问题吗？

通过 quic-go 的配置 `DisablePathMTUDiscovery` 将其置为 `true`, 其默认为 `false`，进行测试验证后，发现卡顿现象消失了，可以明确问题就是 MTU 的问题了，链路上发送了超过实际 MTU 大小的数据，触发的 IP 分片导致大量的丢包，从而出现的卡顿。

### 抓包验证

依然开启 quic-go 的 MTU-Discorver 功能: `DisablePathMTUDiscovery` 将其置为 `false`。

- Server 测抓包

Server 测通过 `tcpdump` 抓包即可，命令： `tcpdump -i any -s 1500 udp src port 4443 -v` 抓取自己从 HTTP3 端口 4443 发送方向的数据包，重点是查看数据包的大小是否有在本地发送时出现切包的情况。

- iPhone 设备测抓包

通过 `xcode` 工具集中的 `rvictl` 结合 `wireshark` 进行抓包。

首先 Mac 上需要安装 `xcode` 开发工具，将 iPhone 连接 Mac 后，通过 xcode 查看该 iPhone 设备的 Identifier ID，在 `xcode` 的 `window` 菜单 ==`》Device` 选项 进行查看：

![获取 iPhone ID](/assets/img/blog/xcode-identifier.png)

**启动虚拟网卡：**

Terminal 中命令启动：`rvictl -s 00008130-00012C163EC2001C `，如果失败，会有 `FAILED` 的输出提示。

**启动 WireShark：**

需要 `root `权限，从 Terminal 中进行启动： `sudo wireshark`，选择网卡 **rvi** 网卡进行抓取，可以添加过滤条件，如：`udp.port == 4443`

- 复现抓包

手机测通过 HTTP3 拉流复现后，分别查看 Server 和 iPhone 测的抓包结果进行分析：

下图可以看到server 在发送 1400+ 的数据包的时候，是未分片且带有 `DF` flag 的，**server 发送的所有数据包均是如此**：

![Server 抓包截图（未切片）](/assets/img/blog/server-quic-capture0.png)

iPhone 测抓包可以看到首先收到的 1400+ 的 数据包是带有 `DF` flag 的，同时也没有进行切片，如下图：

![iPhone 抓包1（首先无分片）](/assets/img/blog/client-quic-capture.png)

但是在后面很快就出现了 IP 分片，且去除了 DF flag 的数据包，iPhone 测只能接收到一半的数据包：

![iPhone 抓包2（被分片）](/assets/img/blog/client-quic-capture2.png)

> 注意：该问题在仅在我司的 Wifi 网络环境下存在，4G/5G 网络条件下不存在。

## 结论

&emsp;&emsp; 通过上述抓包可以看到是因为链路中某个路由设备对 MTU 大于 1400 的数据包进行了强行切片，分成了 2 片进行发送，这里存在两个问题：

1、该路由设备放行了前面一小部分大于 1400 的数据包，接收端接收并 Ack 给了发送端，导致发送端的 Mtu 探测模块爬升到了更高的值，但是后面该路由设备又对 1400+ 的数据包改变了策略，不允许通过，而 quic-go 中的 mtu 探测模块又没有向下探测 mtu 的能力，导致业务遇到该场景必死无疑。

2、发送端发送的数据包已经明确设置了 `DF` 即 `Don't fragment` 的 flag，路由设备在收到超过自己 MTU 设备时应该直接丢掉，而非先进行放行，然后强行忽略 DF flag 进行切片转发，让接收端产生误判。

&emsp;&emsp; 对比抓取其它使用 HTTP3/QUIC 协议的 APP 数据包，比如油管的短视频和直播，可以看到其并未向上探测 MTU，使用的是固定的 1250 这样的一个 MTU 大小。

&emsp;&emsp; 使用 1300 以内的 MTU 大小是网络兼容性最好选择，但是会牺牲一定的服务器网络数据包处理的性能，像 Google 这样的 APP 亦是如此。如果需要追求极致性能，还有一个解决方案，那就是为 MTU 探测模块增加向下探测的能力，比如这种场景，已经探测到了 1400 的大小，但是后续路由器开启切片导致大量的丢包，协议内部开始向下探测 MTU 大小，或者重新从 1000/1200 开始探测即可。

## 后续

&emsp;&emsp; 无







