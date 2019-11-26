---
layout: hexo
title: 装window、ubuntu双系统
date: 2019-04-24 09:09:12
tags:
- 装系统
- 配置
- ubuntu

categories:
- 配置
---

## 装window10、ubuntu16.04双系统
周末趁空装了个双系统，记录记录过程吧。
### 装windows10

 1. 首先下载好win10的系统镜像ISO文件，由于我不咋用win10就装了家庭版
 链接: http://pan.baidu.com/s/1sj3JNRJ 密码: z49r
 
 
 2. 准备好空的U盘，准备做系统启动盘。
 下载安装好UltraISO，插入U盘。
 点击打开，选择ISO文件
 点击启动 - 写入硬盘映像
 写入方式选择的是USB-HDD，USB-HDD+，一般默认就好
 在点击写入，就等着他默默写好就好了
![ULtraISO刻录系统启动盘](https://img-blog.csdnimg.cn/20190422141325858.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMyNjczNDUz,size_16,color_FFFFFF,t_70)

 3. 制作好的系统启动U盘插入要装系统的电脑。开启电脑，一直按 F2或在F12等（这个键根据电脑确定，可以查查，但一般就是这个），进入电脑的Bios设置。
 选择usb storage device，放到最前面，表示系统启动优先从USB开始。点击apply，再点exit。

 4. 之后电脑自动重启，然后进入windows10的安装。
 默认简体中文，下一步
 哪种类型的安装：选择自定义，以前windows的东西会变成windows.old
 输入产品密钥那里跳过。
 你想将windows安装在哪？ 选择分区，选择之前C盘所在分区位置。我这选择的是分区1，476G的盘。
后面就等着自己装就好了。

 5. 装完后注意，系统会重新启动。此时要拔掉U盘。产品密钥那个后面可以去找破解工具破解。暂时不管，然后设置用户密码进入就好。

 
### 装ubuntu16.04

 1. 同理下载好U盘，将ubuntu的系统镜像刻录到U盘里。
 2. 设置好bios优先从U盘启动。
 3. preparing to install Ubuntu: 这里可以选择第二项（Erase disk and install Ubuntu 单独装个Ubuntu系统）或者something else（我这装双系统，本来电脑里分区比较多，因此要选择之前从window划分出来的空闲空间）
 4. 挂载分区到根路径 / ,如果空间足够大，就只挂载这个，剩下的Ubuntu自己会分。如果不够，可以单独跟/home , /boot那些单独分。
 ![挂载](https://img-blog.csdnimg.cn/2019042409082143.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMyNjczNDUz,size_16,color_FFFFFF,t_70)
 5. 继续时区，创建用户，后面就会重启了。重启的时候，注意拔掉U盘。

