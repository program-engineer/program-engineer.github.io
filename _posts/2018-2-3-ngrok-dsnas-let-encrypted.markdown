---
layout:     post
title:      "ngrok服务器搭建"
subtitle:   "内附结合黑群晖外网访问、Let's Encrypt配置HTTPS"
date:       2018-02-03 12:00:00
author:     "multithinking"
header-img: "img/post-bg-2015.jpg"
header-mask:  0.3
catalog:      true
multilingual: false
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

### 1.ngrok服务端在vps上的配置

ngrok的配置大部分集中在服务器端，客户端等到配置完之后，直接在服务器端make生成后下载到客户端即可运行。运行编译ngrok程序需要的环境为git+go语言环境。我配置的环境为vps（centos7 内核版本4.14.15-1.el7.elrepo.x86_64，并开启了bbr加速）

**首先先装好一些编译需要使用的工具**

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

**注意**，如果不适用http或者https，可以留空，则disable。

当启动之后服务端开始监听以上这几个端口，等待客户端来连接。

**如果打开了防火墙，记得放开这几个端口号的限制。**

    ACCEPT     tcp  --  anywhere             anywhere             state NEW tcp dpt:http
    ACCEPT     tcp  --  anywhere             anywhere             state NEW tcp dpt:https
    ACCEPT     tcp  --  anywhere             anywhere             state NEW tcp dpt:pharos




### 2.ngrok客户端在Linux中的配置

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


### 3.ngrok客户端配置到黑群晖

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
    dsfile:
       remote_port: 5000
       proto:
         tcp: 5000

配置好相应文件之后(这个没有配置http访问，只配置了cloud station drive与安卓手机客户端DS file的访问)，启动服务。

    setsid /root/ngrok/ngrok -config=/root/ngrok/ngrok.cfg start-all

setsid ： 将服务在后台启动。

注意：设置之前要先了解群辉系统各服务所监听的端口。
参考这个链接 [http://www.sysdj.com/l/yunwei/nas/2016/1205/25.html](http://www.sysdj.com/l/yunwei/nas/2016/1205/25.html)


### 4.ngrok服务器配置Let's Encrypt

https基于RSA非对称加密算法，客户端利用公钥加密数据（准确来说是会话key），服务端利用对应的私钥来解密；由于公钥的公开性，无法保证正确性，所以引入了第三方权威机构CA来签发数字证书，数字证书中包含服务端的公钥并和私钥一起保存在服务端，客户端必须从服务端获取数字证书，然后从中取出公钥。然而，CA给站点签发数字证书通常都是收费的， 所以，支持HTTPS的站点，需要三个东西：数字证书、私钥和money。为了少花点钱，推荐使用Let's Encrypt，因为它是一个免费的CA。

      Let's Encrypt是国外一个公共的免费SSL项目，由 Linux 基金会托管，它的来头不小，由Mozilla、思科、Akamai、IdenTrust和EFF等组织发起，目的就是向网站自动签发和管理免费证书，以便加速互联网由HTTP过渡到HTTPS，目前Facebook等大公司开始加入赞助行列。Let's Encrypt已经得了 IdenTrust 的交叉签名，这意味着其证书现在已经可以被Mozilla、Google、Microsoft和Apple等主流的浏览器所信任，你只需要在Web 服务器证书链中配置交叉签名，浏览器客户端会自动处理好其它的一切，Let's Encrypt安装简单，未来大规模采用可能性非常大。

一般来说，如果http服务器要配置Let's Encrypt，需要有80端口，因为Let's Encrypt会使用80端口每三个月自动去CA验证服务器证书。也可以说Let's Encrypt证书并不是终生证书，是有期限的，还在三个月验证完可以继续使用。

https证书验证有两种方式，一种是80端口，如果80端口被封的话，一般也可以采用TXT记录的方式，但是这种方法比较麻烦，需要每三个月去更新一下TXT记录，无奈的情况下，才会使用此种方式。因为我的vps在国外，所以不存在80端口被封的情况。默认系统为centos7。并且推荐使用Certbot 安装。

官网有一些比较详细的说明可以参考一下[https://certbot.eff.org/#centosrhel7-apache](https://certbot.eff.org/#centosrhel7-apache)


如果要使用acme安装，可以参考如下：

    #Install online
    Check this project: https://github.com/Neilpang/get.acme.sh
    curl https://get.acme.sh | sh

    git clone https://github.com/Neilpang/acme.sh.git
    cd ./acme.sh
    ./acme.sh --install

You don't have to be root then, although it is recommended.

![img](/img/in-post/ngrok-dsnas-let-encrypted/20180207162906.png)

执行以下命令时请先将httpd服务关闭。

    systemctl stop httpd.service


使用命令获取证书，使用dns的方式。（本地80端口无法使用的情况下）

    ./acme.sh --issue --dns -d XXX.com

会生成TXT value值，将这个值进行解析。

等待解析成功后。。。
运行命令：

    ./acme.sh --renew -d xxx.com

将生产的证书加载到Apache服务。最后重启服务。

客户端cfg文件里，使用hostname+https的方式启动客户端（hostname就是你证书的域名）
然后在客户端第二行设置如下参数：

    trust_host_root_certs: true

确认服务端的启动参数-domain以及客户端cfg文件中的server_addr和证书的域名是同一个，否则会报错误证书的错误。（可以在客户端加上参数-log=log.txt查看日志）

如果你申请的是免费的证书，可能crt文件不带中间商和根证书,这时需要你去网站上把所有证书合在一起。


## 保持稳定运行

配置完以上内容，大部分服务已经可以使用，如何保持稳定运行是非常重要的。通过运行一段时间的观察，ngrok服务还是比较稳定的。如果偶尔断电，可以将bash加入到自动开机启动内。

当然，如果不放心在运行过程出现断开的的情况，可以考虑使用crontab。
在黑群晖里，设置一个计划任务，将服务启动命令，写入其中，这样就可以了。

## 易出错位置

1） 客户端ngrok.cfg中server_addr后的值必须严格与-domain以及证书中的DOMAIN相同，否则Server端就会出现如下错误日志：

[03/13/15 09:55:46] [INFO] [tun:15dd7522] New connection from 54.149.100.42:38252
[03/13/15 09:55:46] [DEBG] [tun:15dd7522] Waiting to read message
[03/13/15 09:55:46] [WARN] [tun:15dd7522] Failed to read message: remote error: bad certificate
[03/13/15 09:55:46] [DEBG] [tun:15dd7522] Closing


