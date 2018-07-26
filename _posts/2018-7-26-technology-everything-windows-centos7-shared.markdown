---
layout:     post
title:      "centos7如何访问windows下的共享文件夹"
subtitle:   ""
date:       2018-02-12 12:00:00
author:     "multithinking"
header-img: "img/post-bg-2015.jpg"
header-mask:  0.3
catalog:      true
multilingual: false
tags:
    - 技术杂货铺
---


## 设置centos7

在安装好的centos7中安装ntfs-3g软件包。

    [root@helpsite ~]# yum list |grep ntfs
    Repodata is over 2 weeks old. Install yum-cron? Or run: yum makecache fast

    ntfs-3g.x86_64                           2:2017.3.23-1.el7              epel    
    ntfs-3g-devel.x86_64                     2:2017.3.23-1.el7              epel    
    ntfsprogs.x86_64                         2:2017.3.23-1.el7              epel    
    [root@helpsite ~]# 
    [root@helpsite ~]# yum install ntfs-3g

## 设置windows共享文件夹


在windows下，建立文件夹，设置共享，共享如果有用户权限，设置好读写权限。


## 共享命令


    [root@helpsite opt]# mkdir 19
    [root@helpsite ~]# mount -t cifs -o username=administrator,password=high_go@hg123  //192.168.0.19/share  /opt/19/


验证：

    [root@helpsite ~]# cd /opt/19/
    [root@helpsite 19]# ll
    total 34188
     -rwxr-xr-x 1 root root 35007212 Jul 25 06:44 openvpn-as-2.5.2-CentOS7.x86_64.rp