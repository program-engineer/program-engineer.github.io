---
layout:     post
title:      "ngrok服务器搭建"
subtitle:   "内附结合黑群晖、Let's Encrypt配置"
date:       2018-02-03 12:00:00
author:     "multithinking"
header-img: "img/post-bg-2015.jpg"
tags:
    - ngrok
---


## 前言

什么是ngrok？简单来说就是一种代理，它使处于内网中的服务能够在没有固定IP的情况下通过公网ngrok服务转发而访问，在ngrok服务器与内网主机之间建立一个通道传输数据。所以说ngrok也是一个tunnel工具，实现nat内网穿透。当然，它还有重要的功能就是支持对隧道中数据的introspection（内省），支持可视化的观察隧道内数据，并replay（重放）相关请求（诸如http请求）。

ngrok的作用还是比较大的，特别是在一些特殊的情况下，而自己又不想购买相关服务，当然前提是自己必须有公网中的vps，有固定的对外IP地址。如果搭建自己的家庭云，需要将网内服务开放出去，可以做到随时随地的访问，但是自己是家用宽带，又不具备固定外网地址，建立通道通过vps访问是个不错的选择，如果再结合https做一下安全，整体是个不错的方案。

看到网络上很多使用ngrok的情况是需要进行微信开发，做了映射之后，可以做到80端口的访问，因为微信只认80端口访问，这样做就可以不需要每做一点修改都要上传进行验证了，只修改本地文件即可，确实方便了许多。

在网络上查了很多资料，终于配置成功了，回想自己配置的过程发现网上的教程还是不够细致，无法让人清晰理解，写下此文，一是帮助需要的人，二是便于自己以后回顾。（附引用了一些网络的图片和描述，在此致谢！）

## ngrok原理

ngrok使用go语言开发，源代码分为客户端与服务器端。
ngrok官方代码最新版为1.7，作者似乎已经完成了ngrok 2.0版本，但不知为何迟迟不放出最新代码。因此这里我们就以ngrok 1.7版本作为原理分析的基础。

ngrok实现了一个tcp之上的端到端的tunnel，两端的程序在ngrok实现的Tunnel内透明的进行数据交互。

![img](/img/in-post/ngrok-dsnas-let-encrypted/20180205084943.png)

## ngrok配置

### ngrok服务端在vps上的配置

### ngrok客户端在Linux中的配置

### ngrok客户端配置到黑群晖



## 稳定运行

## 易出错位置

1） 客户端ngrok.cfg中server_addr后的值必须严格与-domain以及证书中的DOMAIN相同，否则Server端就会出现如下错误日志：

[03/13/15 09:55:46] [INFO] [tun:15dd7522] New connection from 54.149.100.42:38252
[03/13/15 09:55:46] [DEBG] [tun:15dd7522] Waiting to read message
[03/13/15 09:55:46] [WARN] [tun:15dd7522] Failed to read message: remote error: bad certificate
[03/13/15 09:55:46] [DEBG] [tun:15dd7522] Closing


