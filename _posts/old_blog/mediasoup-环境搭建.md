---
title: mediasoup 公网环境部署
categories: RTC
abbrlink: 3784820804
date: 2019-04-01 23:42:10
tags:
---

最近以来一直基于 mediasoup 开发 rtc 相关产品，一直想基于 mediasoup 搭建自己的一套基础的 rtc 环境，用于自己练习 golang、rust 的开发、学习和测试，内网下搭建 mediasoup 比较简单，参照官网的文档一步步来很容易就能搞定，不过如果要在公网下搭建一套环境，相对来说略微麻烦些，需要云主机、nginx 配置、https 配置，通过这篇文章记录上述环境搭建的关键步骤。

<!--more-->

> 以下操作均在 Ubuntu 18.04 Server 系统上操作。

# 准备云主机

公网环境的搭建还是需要一台拥有公网 IP 的主机的，当然如果你能薅公司的羊毛就更好了 。。本人的是阿里云主机 Ubuntu 系统 16.04 upgrade to 18.04，双11 时买的最便宜的机型，同时安装 ssh、 git、nodejs、npm，开启远程登录。

# 域名准备

通过浏览器打开音视频设备因为有安全方面的限制，不能通过 IP 进行访问，必须通过 https://domain 的 url 打开，所以我们还需要准备一个域名并解析到我们自己的公网 IP 地址，域名也可以通过阿里云进行购买，因为不需要 seo，所以选一个最便宜的后缀即可，我选择了 `gobert.top`，第一年只有 9 元，后面还需要域名备案，否则域名将被重定向到指定地址，所以域名需要提前准备。

> 首次域名备案相对来说比较麻烦，需要准备居住证（来沪外来人员）等证件，各种审核需要耗时两三天吧。

# 安装 mediasoup

```
λ ssh gobert@47.100.110.xxx
gobert@47.100.110.xxx's password:
Welcome to Ubuntu 18.04.1 LTS (GNU/Linux 4.15.0-38-generic x86_64)

Last login: Mon Apr  1 17:28:34 2019 from 116.236.177.xxx
$ mkdir develop
$ cd develop
$ git clone git@github.com:versatica/mediasoup-demo.git
```

后续 npm 的安装需参照 `https://github.com/versatica/mediasoup-demo/` 文档进行。

# nginx 配置

> 这一步默认域名购买、解析、备案已完成。

## 在线安装 nginx

```
$ sudo apt-get install nginx
```

## 部署 mediasoup 到 nginx

> 将 mediasoup 中的 server 目录拷贝到 nginx 根目录下。

```
$ sudo mkdir -p /var/www/mediasoup
$ sudo cp -r medissoup-demo/server/ /var/www/mediasoup/
```

配置 nginx 解析到以上目录，编辑 `/etc/nginx/sites-available/default` 配置文件，将 server 根路径下的 root 属性修改为 `/var/www/mediasoup/server/public;`,即：

```
        # 以上省略
        #root /var/www/html;
        root /var/www/mediasoup/server/public;

        # Add index.php to the list if you are using PHP
        index index.html index.htm index.nginx-debian.html;
        # 以下省略
```

## 配置 https 服务

通过浏览器打开设备需要 https 安全 url，nginx 必须开启 https 服务，通过 letsencrypt 进行 https 的自动化配置，省去了自己很多的麻烦，不用再去申请证书、审核、验证部署等繁琐的操作了，通过 `https://certbot.eff.org/lets-encrypt/ubuntubionic-nginx` 下的命令按步执行即可。

![rtc::Thread UML 类图](/images/letsencrypt.png)

> letsencrpt 安装的 ssl 证书只有三个月的有效期，所以为了防止证书过期，建议添加系统定时任务，定期执行脚本命令更新证书：

```
$ su -
$ certbot renew --dry-run
```

正常情况下，你的 nginx 已经开启了 https 服务, ssl 证书及秘钥存放位置记录在 `/etc/nginx/sites-available/default` 配置文件中，打开 `/etc/nginx/sites-available/default` 文件，发现 letsencrpty 自动在脚本末尾增加了一项 server 配置，将 `root` 路径配置为以上 `/var/www/mediasoup/server/public;` 目录，同时记录下其 ssl_certificate 目录，后面 nginx 配置中需要指定：

```
    listen [::]:443 ssl ipv6only=on; # managed by Certbot
    listen 443 ssl; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/www.gobert.top/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/www.gobert.top/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot
```

## 配置 nginx 开启 gzip

mediasoup 中 `mediasoup-demo-app.js` 有 12MB 大小，如果 server 带宽比较低的话，用户首次拉取会比较慢，由于 nginx 默认配置是不开启 gzip 压缩的，所以需要我们手动开启 gzip 压缩，修改 nginx 的配置文件（全站配置）`/etc/nginx/nginx.conf`，将以下内容的注释全部取消：

```
        gzip on;

        gzip_vary on;
        gzip_proxied any;
        gzip_comp_level 6;
        gzip_buffers 16 8k;
        gzip_http_version 1.1;
        gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;
```

然后重启 nginx 服务即可：

```
sudo /etc/init.d/nginx reload
```


## 配置 mediasoup server

- 配置自定义脚本：

```
$ copy config.example.js config.js
```

打开 config.js 文件，在 tls 路径中，将上面记录的 letsencrypt 证书路径拷贝到相应的 item 中，同时修改 `rtcAnnouncedIPv4` 为云主机公网 IP 地址，完整的 config.js 的文件为

```
module.exports =
{
        // Listening hostname for `gulp live|open`.
        domain : 'localhost',
        tls    :
        {
                cert : `/etc/letsencrypt/live/www.gobert.top/fullchain.pem`, // letsencrypt 证书
                key  : `/etc/letsencrypt/live/www.gobert.top/privkey.pem`    // letsencrypt 秘钥
        },
        mediasoup :
        {
                // mediasoup Server settings.
                logLevel : 'warn',
                logTags  :
                [
                        'info',
                        'ice',
                        'dtls',
                        'rtp',
                        'srtp',
                        'rtcp',
                        // 'rbe',
                        // 'rtx'
                ],
                numWorkers       : null, // Use number of CPUs.
                rtcIPv4          : true,
                rtcIPv6          : true,
                rtcAnnouncedIPv4 : '47.100.110.192', // 云主机公网地址
                rtcAnnouncedIPv6 : null,  // 根据需要，自主填写
                rtcMinPort       : 40000, // udp 最小端口，需要云主机安全组中配置
                rtcMaxPort       : 49999, // dup 最大端口，需要云主机安全组中配置
                // mediasoup Room codecs.
                mediaCodecs      :
                [
                        {
                                kind       : 'audio',
                                name       : 'opus',
                                mimeType   : 'audio/opus',
                                clockRate  : 48000,
                                channels   : 2,
                                parameters :
                                {
                                        useinbandfec : 1
                                }
                        },
                        {
                                kind       : 'video',
                                name       : 'VP8',
                                mimeType   : 'video/VP8',
                                clockRate  : 90000,
                                parameters :
                                {
                                        'x-google-start-bitrate': 1500
                                }
                        },
                        {
                                kind       : 'video',
                                name       : 'h264',
                                mimeType   : 'video/h264',
                                clockRate  : 90000,
                                parameters :
                                {
                                        'packetization-mode'      : 1,
                                        'profile-level-id'        : '42e01f',
                                        'level-asymmetry-allowed' : 1
                                }
                        }
                ],
                // mediasoup per Peer max sending bitrate (in bps).
                maxBitrate : 500000
        }
};
```
- 修改 Server 监听端口

> 此处修改 server 的监听端口其实没什么意义，可以忽略。

编辑 `server.js` 配置文件，将默认端口 `httpsServer.listen(3443, '0.0.0.0', () =>` 替换为 `httpsServer.listen(5678, '0.0.0.0', () =>`。


- 配置 Client 访问 Server WSS 地址

打开 ` public/mediasoup-demo-app.js` 将 `getProtooUrl` 方法中的 URL 地址端口改为自定义的 `5678` 端口：

> 如果上一步中没有修改 server 的监听端口的话，这一步也可忽略。

```
function getProtooUrl(peerName, roomId, forceH264) {
  var hostname = window.location.hostname;
  var url = "wss://".concat(hostname, ":5678/?peerName=").concat(peerName, "&roomId=").concat(roomId);
  if (forceH264) url = "".concat(url, "&forceH264=true");
  return url;
}
```

## 启动 mediasoup server

```
$ cd /var/www/mediasoup/server
$ sudo DEBUG="*mediasoup* *ERROR* *WARN*" INTERACTIVE="true" node server.js
```

## 重启 nginx

```
$ sudo nginx -s reload
```
# 在线体验

> 此主机带宽比较低，初次打开比较慢，需耐心等待加载完成，后面缓冲后速度会变化，音视频延迟不受影响。

在两个终端浏览器（支持 Chrome、Firefox、Safari）中均打开 `https://www.gobert.top/?roomId=123456` 即可体验，RoomId 可自定义，效果图：

![连麦效果图](/images/mediasoup-brower.png)



