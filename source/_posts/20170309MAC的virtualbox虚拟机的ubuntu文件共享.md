---
title: mac与virtualbox虚拟机的ubuntu文件共享
date: 2017-03-09 16:58:50
tags:
- 配置
- ubuntu
categories:
- 配置
---

1、坑是啥？

我在进行hadoop的配置的时候，需要从mac共享一个文件到ubuntu访问，并且也查了网上很多共享的方法。都试过了，先设置共享文件，路径，永久保存。然后在终端敲命令：sudo mount -t vboxsf share /home/wangxue/share （前一个是新建的文件夹，后一个是mac的文件夹）。

坑就是“文件系统错误，共享文件有超级坏块。。。”我去，我搜了半天居然没解决，还是同学帮忙搞定了

2、办法

要卸载并重新安装 virtualbox-guest-utils virtualbox-guest-additions-iso，linux源有这个。我旧的出问题了。

贴一下笔记：

客户、宿主机共享目录

设置 -> 共享文件夹 -> 固定分配 -> 添加一个本地文件夹，并设置自动挂载或不挂载

安装后有下面的文件

/usr/share/virtualbox/VBoxGuestAdditions.iso

之后再进行mount -t vboxsf share /home/wangxue/share

virtualbox的ubuntu是关闭剪贴板共享，共享剪贴板需要选择 虚拟机 -> 设置 -> 常规 -> 高级 -> 共享剪贴板 -> 双向

好了，晚安！

