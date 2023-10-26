---
title: Windows 10 安装 .NetFrameWork 3.5
categories: OPC
abbrlink: 3530668087
date: 2018-02-26 14:05:19
tags:
---

# Windows 10 下安装 .NetFrameWork 3.5

某些软件依赖.NetFrameWork 3.5,但是通过系统自带的 `启动或关闭 Windows 功能`的界面接口去添加 .NetFrameWork 3.5 会失败，即使先删除了 .NetFrameWork 4.7 也是会失败，比如会报如下一些错误：
~~~
0x800F0906
0x800F081F
0x800F0907
0x800F0922
...
~~~
现总结如下安装方法;
<!--more-->

## 下载 .NetFrameWork 3.5 离线安装包

[下载地址](https://d11.baidupcs.com/file/f035714091c087774ca76a254018be04?bkt=p3-000014508cb2b9cb0151cee0f9ed3c6b0021&xcode=7cbd07a2b4f6f74a3477739bc86162007cbccbd369afad2e710b2321bcd8d2e5e4b423467a61eb46&fid=2098421940-250528-207166886389709&time=1519614793&sign=FDTAXGERLQBHSKa-DCb740ccc5511e5e8fedcff06b081203-kspSbRcHERbGt4GGM8ncJm%2B0M%2F8%3D&to=d11&size=72329390&sta_dx=72329390&sta_cs=70095&sta_ft=cab&sta_ct=7&sta_mt=7&fm2=MH%2CYangquan%2CAnywhere%2C%2Cshanghai%2Cct&vuk=282335&iv=0&newver=1&newfm=1&secfm=1&flow_ver=3&pkey=000014508cb2b9cb0151cee0f9ed3c6b0021&sl=79364174&expires=8h&rt=sh&r=512254481&mlogid=1309641164889007677&vbdid=2429812664&fin=NetFx3.cab&fn=NetFx3.cab&rtype=1&dp-logid=1309641164889007677&dp-callid=0.1.1&hps=1&tsl=100&csl=100&csign=azJdY%2B10Z49b0LbASthGmxMFU9c%3D&so=0&ut=6&uter=4&serv=0&uc=1415919733&ic=69126377&ti=f8fdaa4589ff2f184c4630f76c465917b1fcaab7023e5652&by=themis)

下载完成后，将此文件 `NetFx3.cab` 不要解压直接拷贝到 `C:/Windows` 目录下;

## 命令安装

以管理员权限运行 cmd 命令，执行以下命令直至 100% 成功即可：
~~~
dism.exe /online /enable-feature /featurename:netfx3 /Source:L:\sources\sxs
~~~

## 安装完成

如果不出问题的话是可以安装成功的，且不用重启，这时打开控制面板中 `启动或关闭 Windows 功能` 的界面，可以看到 `.Net FrameWork 3.5` 已经安装成功了，这时勾选 `.Net FrameWork 4.7` 并确定，以免其它已安装的依赖软件不能正常运行。

