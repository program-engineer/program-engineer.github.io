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

ngrok使用go语言开发，源代码分为客户端与服务器端。ngrok官方代码最新版为1.7，作者似乎已经完成了ngrok 2.0版本，但不知为何迟迟不放出最新代码。因此这里我们就以ngrok 1.7版本作为原理分析的基础。

ngrok实现了一个tcp之上的端到端的tunnel，两端的程序在ngrok实现的Tunnel内透明的进行数据交互。

![img](/img/in-post/ngrok-dsnas-let-encrypted/20180205084943.png)

ngrok分为client端(ngrok)和服务端(ngrokd)，实际使用中的部署如下：

![img](/img/in-post/ngrok-dsnas-let-encrypted/20180205085035.png)

内网服务程序可以与ngrok client部署在同一主机，也可以部署在内网可达的其他主机上。ngrok和ngrokd会为建立与public client间的专用通道（tunnel）。

更详细的内容请参考：
1、[网友分享 http://tonybai.com/2015/05/14/ngrok-source-intro/](http://tonybai.com/2015/05/14/ngrok-source-intro/)
2、[ngrok开发者指导 https://github.com/inconshreveable/ngrok/blob/master/docs/DEVELOPMENT.md](https://github.com/inconshreveable/ngrok/blob/master/docs/DEVELOPMENT.md)

想让ngrok与ngrokd顺利建立通信，我们还得制作数字证书,可以是使用OpenSSL自签发，同样也可以使用Let's Encrypt（不了解的请自行Google），下面会有涉及。
通过阅读前面两个链接的内容，应该大体对ngrok有了一定得了解，但是如果你自己操作一遍会发现在服务端运行帮助命令会看到：

![img](/img/in-post/ngrok-dsnas-let-encrypted/20180207162236.png)

从图中我们知道它只能设置http、https这两个对外通道，虽然可以自定义端口，但是我们要的可能是单纯的tcp连接，不需要http访问。（例如在连接黑群辉时）
*这个需要在客户端配置文件中进行设置。*

我们再看一下客户端的帮助命令：

![img](/img/in-post/ngrok-dsnas-let-encrypted/20180207162905.png)

其中有个-config就是我们要配置的文件，这样可以极大的扩展我们使用ngrok的范围。

关于如何配置这个文件，下文会有说明。

ngrokd在4443端口默认建立通道：

![img](/img/in-post/ngrok-dsnas-let-encrypted/20180205085058.png)

ngrok的基本原理就是这样，详细了解需要仔细阅读上面两个链接。

## ngrok配置

### ngrok服务端在vps上的配置

ngrok的配置大部分集中在服务器端，客户端等到配置完之后，直接在服务器端make生成后下载到客户端即可运行。运行编译ngrok程序需要的环境为git+go语言环境。我配置的环境为vps（centos7 内核版本4.14.15-1.el7.elrepo.x86_64，并开启了bbr加速）

#### 首先先装好一些编译需要使用的工具

    yum -y install zlib-devel openssl-devel perl hg cpio expat-devel gettext-devel curl curl-devel 
    perl-ExtUtils-MakeMaker hg wget gcc gcc-c++

安装git

    wget https://www.kernel.org/pub/software/scm/git/git-2.9.5.tar.gz
    tar zxvf git-2.9.5.tar.gz 
    cd git-2.9.5
    ./configure --prefix=/usr/local/git
    make
    make install
    ln -s /usr/local/git/bin/* /usr/bin/

然后安装go语言环境（这里使用的事google的镜像，国内的镜像也有不少）

    wget https://dl.google.com/go/go1.9.3.linux-amd64.tar.gz
    tar -C /usr/local -xzf go1.9.3.linux-amd64.tar.gz
    ln -s /usr/local/go/bin/* /usr/bin

安装完go程序后需要把go编译程序导入export中

    export PATH=$PATH:/usr/local/go/bin

当然，写入.bash_profile更好，重启不会丢失。没有重启的情况下需要执行命令使之生效。

    source .bash_profile 

下载ngrok程序

    cd /usr/local/
    git clone https://github.com/inconshreveable/ngrok.git
    cd ngrok/

如果前面的配置都比较顺利的话，配置到这里ngrok的基本环境就有了。接下来要做的就是编译出ngrok的服务器端ngrokd与客户端成ngrok。

生成ngrokd之前需要先配置自签名证书，使传输加密，这里也可以使用Let's Encrypt，在这里先使用自签名介绍，后面再谈另一种方法。

    openssl genrsa -out rootCA.key 2048 
    openssl req -x509 -new -nodes -key rootCA.key -subj "/CN=DOMAIN" -days 5000 -out rootCA.pem
    openssl genrsa -out server.key 2048
    openssl req -new -key server.key -subj "/CN=DOMAIN" -out server.csr 
    openssl x509 -req -in server.csr -CA rootCA.pem -CAkey rootCA.key CAcreateserial -out server.crt -days 5000

然后复制相关证书。

    yes|cp rootCA.pem ../assets/client/tls/ngrokroot.crt
    yes|cp server.crt ../assets/server/tls/snakeoil.crt 
    yes|cp server.key ../assets/server/tls/snakeoil.key

准备好证书后就可以编译ngrokd和ngrok程序了。

//指定环境变量位64位linux版本

    GOOS=linux GOARCH=amd64 #如果是32位系统，这里 GOARCH=386
    make release-server

启动ngrokd

    ./ngrokd -tlsKey="/usr/local/ngrok/assets/server/tls/snakeoil.key" -tlsCrt="/usr/local/ngrok/assets/server/tls/snakeoil.crt" -domain="你的域名"  -httpAddr=":80" -httpsAddr=":443" -tunnelAddr=":4443"

*-httpAddr默认是80
-httpsAddr默认是443
-tunnelAddr默认是4443*
这些端口号你都可以修改成别的，但是要和客户端的tunnel端口号对应。

当启动之后服务端开始监听以上这几个端口，等待客户端来连接。

**如果打开了防火墙，记得放开这几个端口号的限制。**

    ACCEPT     tcp  --  anywhere             anywhere             state NEW tcp dpt:http
    ACCEPT     tcp  --  anywhere             anywhere             state NEW tcp dpt:https
    ACCEPT     tcp  --  anywhere             anywhere             state NEW tcp dpt:pharos




### ngrok客户端在Linux中的配置

cd到ngrok目录

    //指定环境变量位64位linux版本，如果是32位系统，这里 GOARCH=386
    GOOS=linux GOARCH=amd64 
    make release-client

需要注意的是，我这里的环境是Linux，所以编译了Linux版本。客户端也编译的Linux版本，如果你的客户端是其他类型，可以在这里制定对应版本进行编译。

例如：

    GOOS=windows GOARCH=amd64 make release-client
    //同理，这里的amd64是64位系统，32位改成386

    GOOS=darwin GOARCH=amd64
    // mac osx，64位，对应的 GOOS 和 GOARCH 是这样的

编译完成后就可以将编译的ngrok文件复制到你想运行的客户端上即可。ngrok文件存在于/bin目录下。


### ngrok客户端配置到黑群晖

ngrok客户端配置到黑群晖系统内是比较简单的，只需要将在ngrok服务器上生成的客户端复制过来即可（注意黑群晖是Linux系统，所以要生成Linux客户端），然后再配置参数文件启动就可以了（bash文件 ngrok.sh）。

首先ssh登录黑群晖（没有启用ssh的请先在web页面控制面板内启用），并进入root账户下。

    admin@DiskStation:/$ 
    admin@DiskStation:/$ sudo -i
    Password: （此处密码为admin的密码）
    root@DiskStation:~# 

建立ngrok文件夹，将ngrok客户端scp过来。并建立配置文件ngrok.cfg。

    root@DiskStation:~/ngrok# ll
    total 10860
    drwxrwxrwx 2 root root     4096 Feb  4 16:36 .
    drwx------ 5 root root     4096 Feb  5 00:18 ..
    -rwxr-xr-x 1 root root 11105628 Feb  3 08:08 ngrok
    -rwxrwxrwx 1 root root      160 Feb  4 16:36 ngrok.cfg


在这里特别注意的是ngrok.cfg文件内不允许有空格，不允许使用tab缩进，否则会报错。

在ngrok.cf内配置HTTP，HTTPS，tcp都可以。一定要理解这里。如果配置http/https，加入的二级域名必须对应服务器端，否则要使用泛解析渔民，在购买域名页内设置。tcp协议只需要在客户端配置文件中设置即可，需要使用remote_port参数，当启动后，服务端会自动进行监听。


    server_addr: $Domain:4443
    trust_host_root_certs: false
    tunnels:
    csd:
       remote_port: 6690
       proto:
         tcp: 6690
    ds:
       proto:
       http: 5000

配置好相应文件之后，启动服务。

    setsid /root/ngrok/ngrok -config=/root/ngrok/ngrok.cfg start-all

setsid ： 将服务在后台启动。


### ngrok服务器配置Let's Encrypt



## 保持稳定运行

## 易出错位置

1） 客户端ngrok.cfg中server_addr后的值必须严格与-domain以及证书中的DOMAIN相同，否则Server端就会出现如下错误日志：

[03/13/15 09:55:46] [INFO] [tun:15dd7522] New connection from 54.149.100.42:38252
[03/13/15 09:55:46] [DEBG] [tun:15dd7522] Waiting to read message
[03/13/15 09:55:46] [WARN] [tun:15dd7522] Failed to read message: remote error: bad certificate
[03/13/15 09:55:46] [DEBG] [tun:15dd7522] Closing


