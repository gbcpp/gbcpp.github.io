---
title: Android AEC 无效的解决方案
tags: Android AEC 16KHz
categories: RTC
abbrlink: 1837193558
date: 2019-01-23 10:12:21
---



Android 端回声消除模块在某些机型上面需要使用 16kHz 的采样率输入，否则将会产生回声问题，为此设计此特殊机型下发服务，在 Android 端启动时查询其是否在此特殊名单中，是则强制为 16kHz 采样率，否则使用默认的 48/44.1 kHz 采样率。

因为只有部分机型才会出现此类问题，并且会不断产生新的机型出现，如果在代码中写 HardCode，那么产品的发布将会很被动，因为随时可能会有新的问题机型产生，为此开发一套特殊机型下发的服务，在 APP 启动时向此服务发起请求，校验此机型是否需要配置为 16KHz，然后再启动 RTC 进行音视频通话。此服务通过 Go 进行开发。

GitHub 地址为 ：https://github.com/zzlc/rtc-aecm-android

<!--more-->

## 启动服务

启动命令：

```
go run main.go -addr="127.0.0.1:6688" -prefix="/v1/aecm/" -user="root" -password="123456"
```

> 实际部署后根据实际目标名称进行启动，但启动参数不变，可通过 -h 查看帮助。

## Android 端查询接口

API 查询接口提供两种，一是查询指定 `Model` 是否存在于此白名单列表中，二是查询所有的白名单列表；

### 1、查询指定 `Model`

> 这里及以后均假设 SEV_ADDR 即为 服务器地址。

```
Request Url:
SEV_ADDR/v1/aecm
Method: OPTIONS

Request Headers:
model=xxx

Result:
200 ok
```


> 通过返回 http status 判断 xxx 是否在白名单中，200 为是，否则为 否。

### 2、查询所有白名单

```
Request Url:
SEV_ADDR/v1/aecm
Method: QUERY

Result:
200 ok
Content type: text/json
body: json string
```

> 如果返回 200 ok，则说明查询成功，可通过 content body 的 json string 进行解析，格式如下：

```json
{
    "7": {
        "author": "gobert",
        "brand": "postman_brand",
        "insertTime": "2019-01-10 15:19:37",
        "model": "redmi_note4",
        "osVersion": "Android 6.2",
        "packageName": "com.org.xxx",
        "sdkVersion": "0.0.0"
    },
    "8": {
        "author": "gobert",
        "brand": "postman_brand",
        "insertTime": "2019-01-10 15:19:39",
        "model": "redmi_note4",
        "osVersion": "Android 6.2",
        "packageName": "com.org.xxx",
        "sdkVersion": "0.0.0"
    }
}
```


## 添加新机型

通过以下 API 即可添加

```
Request Url:
SEV_ADDR/v1/aecm
Method: ADD

Request Headers:
osVersion=Android 6.2
brand=postman_brand
model=xxx
sdkVersion=1.0.0
packageName=com.org.xxx
author=gobert

Result:
200 ok
```


## 删除已有机型

```
Request Url:
SEV_ADDR/v1/aecm
Method:DELETE

Request Headers:
model=xxx

Result:
200 ok
```

