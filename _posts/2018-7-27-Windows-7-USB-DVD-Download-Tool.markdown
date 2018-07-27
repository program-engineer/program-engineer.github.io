---
layout:     post
title:      "Windows 7 USB DVD Download Tool制作U盘与DVD系统启动盘"
subtitle:   "文中附带出现U盘无法拷贝文件的情况"
date:       2018-02-12 12:00:00
author:     "multithinking"
header-img: "img/post-bg-2015.jpg"
header-mask:  0.3
catalog:      true
multilingual: false
tags:
    - 技术杂货铺
---




## 工具说明

Windows 7 USB DVD Download Tool是微软官方提供的系统启动盘制作工具。可以从官网下载，也可以从其他地方下载。

第三方下载路径：http://www.xiazaiba.com/html/37587.html

下载后安装即可。


## 使用步骤

打开Windows 7 USB DVD Download Tool，制作U盘Win7安装盘

![](https://images0.cnblogs.com/blog2015/562136/201504/131433093547894.png)

![](https://images0.cnblogs.com/blog2015/562136/201504/131435122763975.png)

![](https://images0.cnblogs.com/blog2015/562136/201504/131437265269360.png)

![](https://images0.cnblogs.com/blog2015/562136/201504/131440109328066.png)

等待读条完毕，弹出U盘准备使用。

## 出现U盘无法拷贝文件的情况

如果出现U盘无法拷贝文件的情况，可以先检查ISO文件是否损坏，如果没有损坏，很有可能是u盘的格式不对，可以使用下面的方式调整。

将U盘插入本地计算机上，如下很明显磁盘1是U盘。

    DISKPART> list disk

     磁盘 ###  状态           大小     可用     Dyn  Gpt
     --------  -------------  -------  -------  ---  ---
     磁盘 0    联机              931 GB  1024 KB        *
     磁盘 1    联机               14 GB      0 B

    DISKPART> select disk  1

    磁盘 1 现在是所选磁盘。

    DISKPART> clean

    DiskPart 成功地清除了磁盘。

    DISKPART> create partition primary

    DiskPart 成功地创建了指定分区。

    DISKPART> select partition 1

    分区 1 现在是所选分区。

    DISKPART> active

    DiskPart 将当前分区标为活动。

    DISKPART> format quick fs=fat32

      100 百分比已完成

    成功格式化该卷。

    DISKPART> assign

    DiskPart 成功地分配了驱动器号或装载点。

    DISKPART> list disk

    磁盘 ###  状态           大小     可用     Dyn  Gpt
     --------  -------------  -------  -------  ---  ---
     磁盘 0    联机              931 GB  1024 KB        *
    * 磁盘 1    联机               14 GB      0 B

    DISKPART>


这样就可以了。注意不要输成硬盘的编号。

