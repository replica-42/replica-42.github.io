---
title: "从零开始搭建个简单的 NAS"
date: 2024-03-04T14:33:11+08:00
description: "废物再利用"
categories:
  - "计算机日常"
---

## 写在前面的话

总之是对[这篇](https://replica-42.github.io/2024/02/2024-february-memories/)的伏笔回收，且更多适用于手头有闲设备不想吃灰的情况。真正有长久稳定 NAS 需求的还是建议买成品/用 3.5 寸 CMR 机械盘自组。本文假定读者对计算机相关的概念有基本的了解，有基本的计算机使用能力。

本文不包含逐步指引，原因在于手头唯一一台树莓派工作得很正常，我并不想为了写篇博客就给它重做个系统再来一遍。过程中任何疑问请善用提供的外链及搜索引擎。

未来或许有扩展成详细教程的可能性（还请不要太期待）。

## 事前准备

最基本的事前准备包括：

* 能安装主流 Linux 发行版的设备，x86/ARM 皆可。性能大于等于树莓派 4b 就行（更弱的我也没试过）
* 确保能让你的 NAS 和消费 NAS 内容的设备处于同一局域网下，一般来说有有台路由器就行
* 视需求而定的存储设备，SSD/HDD 皆可，SD卡/U盘不建议

## 简单流程

装一个主流的 Linux 发行版，不知道选什么的话装 [Ubuntu](https://ubuntu.com/) 就好了。一般来说性能羸弱的话不建议装桌面，用 Server 版就好，性能足够的话按喜欢的来，装 Windows 也没所谓。以下流程按照 Server 版叙述。挂载你的存储设备，如果存储设备本身就是系统盘的话略过该步。选定一个文件夹作为共享的根目录。

安装 samba，配置其共享上文选定的目录。配置的教程网上一大堆，找不到合适的教程的话也可以参考[这篇](https://wiki.samba.org/index.php/Setting_up_Samba_as_a_Standalone_Server)，需要注意的是 smb 的用户与 Linux 的用户并不等同，验证时使用的是前者。以下是一个配置文件的例子：

```
[global]
    log file = /var/log/samba/%m
    log level = 1
    server role = standalone server

[demo]
    # This share requires authentication to access
    path = /srv/samba/demo/
    read only = no
    inherit permissions = yes
```

在 Windows 的资源管理器中选择“映射网络驱动器”，以上文的配置文件为例，服务器地址为 `\\server\demo`，其中 server 是服务器的地址。此时已经可以像访问本地存储一样访问共享出来的存储了。

iOS 系统如果是观看视频的话可以用 nPlayer 或 VLC，二者均支持 smb 协议。

## 功能扩展

一般来说 7x24 常开的设备挂个 BT 总归是不亏的。以上节的配置为基础，我的习惯是在共享的根目录下建立三个子目录，分别用于：

* 监控 torrent 文件自动触发下载
* 存放下载完成后做种的数据
* 存放其他非 BT 的数据

命令行环境下推荐使用 [rtorrent](https://github.com/rakshasa/rtorrent/wiki) 作为 BT 客户端，具体的安装与使用方法见 Wiki，一般来说需要配合 tmux/screen 这样的程序使用，确保启动的 rtorrent 进程不会因为 ssh 会话断开而噶掉。需要额外关注的是通过配置文件实现的目录监听功能，具体可见 [wiki](https://github.com/rakshasa/rtorrent/wiki/TORRENT-Watch-directories)。除此之外 [rtorrent 并不支持 UPnP](https://github.com/rakshasa/rtorrent/issues/261)，因此需要路由器配置端口转发。不想手动配的也可以尝试 [transmission](https://github.com/transmission/transmission)。桌面环境推荐使用 [qBittorrent](https://www.qbittorrent.org/)。

如果有同步文件的需求，[seafile](https://www.seafile.com/en/home/) 的社区版 server 和 [resilio sync](https://www.resilio.com/individuals/) 都可以试试。