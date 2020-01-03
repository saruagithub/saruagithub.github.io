---
title: ubuntu16配置GPU深度学习环境、CUDA、cuNDD等
date: 2019-11-06 11:15:21
tags:
- 配置
- 深度学习
categories:
- 深度学习
---
### 1、准备
 1. 请先看好各种软件的版本对应要求，这仨一定要对应好。
	 	 [Tensorflow不同版本要求与CUDA及CUDNN版本对应关系](https://blog.csdn.net/omodao1/article/details/83241074)
 2. 知道要下哪些版本了，就预先做好各种软件下载工作。
     首先下载好英伟达的驱动 [NVIDIA驱动下载](https://www.nvidia.cn/Download/index.aspx?lang=cn)
     注意！！！下载好跟自己显卡对应的驱动。显卡的产品类型、系列那些如果之前已经装好了驱动，则可以通过命令 nvidia-smi查询到。没有装刚买来就自己查。
 ![我的显卡驱动](https://img-blog.csdnimg.cn/20190519153242367.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMyNjczNDUz,size_1,color_FFFFFF,t_70)
    即使你的机器之前已经装过驱动，那也最好重新装一遍驱动，因为那个CUDA一定要对应起来。不然后面有坑！
    
    下载CUDA，链接 [cuda-toolkit-archive](https://developer.nvidia.com/cuda-toolkit-archive)
    ![下载CUDA9.0版本](https://img-blog.csdnimg.cn/20190519154540803.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMyNjczNDUz,size_1,color_FFFFFF,t_70)
请注意这里一定要选择下载runfilw文件，不是deb！，不然会覆盖之前的显卡驱动带来问题。
![对应操作系统下载CUDA](https://img-blog.csdnimg.cn/20190519154709423.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMyNjczNDUz,size_1,color_FFFFFF,t_70)
最后下载cuDNN，[cuDNN下载地址](https://developer.nvidia.com/rdp/cudnn-archive)，我下的7.0.5版本
![cuDNN下载](https://img-blog.csdnimg.cn/20190519160003336.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMyNjczNDUz,size_1,color_FFFFFF,t_70)
### 2、安装驱动
#### 2.1、正常装驱动。
按ctrl+alt+f2（有的是f1）进入字符界面命令行，先删除以前的驱动：
```
sudo apt-get purge nvidia*
sudo apt-get autoremove
```
禁止自带的nouveau nvidia驱动：
```
# 打开配置文件
sudo vim /etc/modprobe.d/blacklist-nouveau.conf
```
添加以下内容：
```
blacklist nouveau
options nouveau modeset=0
```
再更新一下：
```
sudo update-initramfs -u
```
最后需要进行重启。查看下Nouveau是否已经禁止，无输出则为成功：
```
lsmod | grep nouveau
```
按ctrl+alt+f2，接着关闭图形化界面：
```
sudo service lightdm stop
```
然后准备开始装驱动了。
```
sudo sh NVIDIA-Linux-x86_64-XXX.run  –-no-opengl-files
```
然后重新打开图形界面：
```
sudo service lightdm start
```
再ctrl+alt+f7进入图形界面，再测试下驱动是否装好：
```
nvidia-smi
```
安装完成后，重启:
```
sudo reboot
```
在命令行通过nvidia-smi还可以查看到驱动的话就没有问题了，以上皆为顺利的过程。


#### 2.2、意外情况
当然我装的时候是遇到了个大坑的。我看到之前机器上装好了驱动就没管，然后开始装后面的CUDA，结果下的CUDA又是deb的包，导致安装中覆盖了之前的驱动，然后ubuntu打开正确输入密码也无法进入桌面了。

##### 2.2.1 安装libelf-dev
于是我又修复，倒回到2.1开始，清理驱动，重装。中间在执行sudo sh NVIDIA-Linux-x86_64-XXX.run  –-no-opengl-files的时候还遇到了build出错，如图：
![驱动编译出错](https://img-blog.csdnimg.cn/20190519162933827.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMyNjczNDUz,size_1,color_FFFFFF,t_70)
打开他提示的nvidia-installer.log看，里面提示了很多
![问题提示](https://img-blog.csdnimg.cn/20190519163249938.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMyNjczNDUz,size_1,color_FFFFFF,t_70)
这里还挺好的提示了请安装libelf-dev这种信息，于是我又去下载 [libelf-dex安装包](https://pkgs.org/download/libelf-dev)。本来我只下了1那个，然后输入命令安装：
![libelf的版本](https://img-blog.csdnimg.cn/20190519163856594.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMyNjczNDUz,size_1,color_FFFFFF,t_70)
```
sudo dpkg -i libelf-dev_0.165-3ubuntu1_amd64.deb
```
很无情的又报了个错，提示amd64 system is ....ubuntu1.1，于是我又下了2那个更新包，再dpkg安装。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190519164151594.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMyNjczNDUz,size_1,color_FFFFFF,t_70)
终于顺利给装上了，没有报错了。

##### 2.2.2 gcc和g++版本问题
前面的装好了，我又准备执行sudo sh NVIDIA-Linux-x86_64-XXX.run  –-no-opengl-files 来着，然而还有问题，又通过命令查看log信息，sudo vim nvidia-installer.log。
![不识别Command line](https://img-blog.csdnimg.cn/20190519164725139.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMyNjczNDUz,size_1,color_FFFFFF,t_70)
这个问题就是由于gcc和g++版本太低编译不过导致的，因为我看之前有个教程是将这个版本降低了方便CUDA编译来着。但其实我这是CUDA9.0，CUDA9要求GCC版本是5.x或者6.x，其他版本不可以，需要自己进行配置。我之前就是5.5的版本，就不该降级。好的现在再根据那篇博文给换回来。
[ubuntu 16.04 LTS 降级安装gcc 4.8
](https://blog.csdn.net/u011784994/article/details/80080938)

##### 2.2.3 装好驱动
在sh NVIDIA-Linux-x86_64-XXX.run安装就可以了，哎哟喂真是不容易啊。。。
然后我再重启，输入密码，终于可以进入桌面了呀，感动到哭。。。

### 3、安装CUDA

 1. 安装CUDA
打开终端，执行命令，运行run文件：
```
sudo sh cuda_9.0.176_384.81_linux.run
```
注意提示，前面是一些法律信息啥的，enter过去就好。到后面提示是否安装图像驱动的时候，一定选择no ！！！
![no Driver](https://img-blog.csdnimg.cn/20190519165519471.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMyNjczNDUz,size_4,color_FFFFFF,t_70)
后面的一些提示选择y就行。出现下图，就表示安装完成。
![CUDA安装](https://img-blog.csdnimg.cn/20190519165702507.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMyNjczNDUz,size_1,color_FFFFFF,t_70)
如果出现其他问题，可能是某些依赖库没装好，反正我是没遇到。可以试试安装依赖，然后重启再试试。
```
sudo apt-get install libprotobuf-dev libleveldb-dev libsnappy-dev libopencv-dev libhdf5-serial-dev protobuf-compiler
sudo apt-get install --no-install-recommends libboost-all-dev
sudo apt-get install libopenblas-dev liblapack-dev libatlas-base-dev
sudo apt-get install libgflags-dev libgoogle-glog-dev liblmdb-dev
```

2. 配置环境变量
```
sudo gedit ~/.bashrc
```
打开文件后在最后写入：
```
export PATH=/usr/local/cuda-9.0/bin${PATH:+:${PATH}}  
export LD_LIBRARY_PATH=/usr/local/cuda-9.0/lib64${LD_LIBRARY_PATH:+:${LD_LIBRARY_PATH}}
```
然后点save后关闭在source一下生效：
```
source ~/.bashrc
```

3. 测试一下CUDA是否安装成功
```
# 第一步，进入例子文件
cd /usr/local/cuda-8.0/samples/1_Utilities/deviceQuery
# 第二步，执行make命令
sudo make
# 第三步
./deviceQuery
```
有提示GPU信息，就表示可以了。

### 4、安装cuDNN
安装命令：
```
sudo dpkg -i libcudnn7_7.0.5.15-1+cuda9.0_amd64.deb
sudo dpkg -i libcudnn7-dev_7.0.5.11-1+cuda9.0_amd64.deb
sudo dpkg -i libcudnn7-doc_7.0.5.11-1+cuda9.0_amd64.deb
```
安装完以后需要进行测试是否安装成功，出现了“Test passed! ”，这几步我都没啥问题：
```
cp -r /usr/src/cudnn_samples_v7/ $HOME
cd $HOME/cudnn_samples_v7/mnistCUDNN
make clean && make
./mnistCUDNN
```

### 5、安装TensorFlow-gpu
卸载以前的TensorFlow，我的python环境是3.6
```
pip3 uninstall tensorflow
```
然后重新装gpu版本就可以，注意我要用的是TensorFlow-gpu1.7版本，这个跟前面的都是对应的！

```
pip3 -i https://pypi.tuna.tsinghua.edu.cn/simple/ install tensorflow-gpu==1.7.0
```
跑程序的时候，自动就调用了gpu进行计算，学习起来快了6、7倍，真的是开心啊~

### 6、总结

 1. 最关键的问题就是软件各个版本要对应好
 2. 注意先装驱动再CUDA再cuDNN，总之就是驱动要先搞好，不然就会有我那种意外。
 3. CUDA一定下载runfile文件。


### 7、reference
[Ubuntu18.04深度学习GPU环境配置](https://blog.csdn.net/weixin_41863685/article/details/80303963)
我进不了桌面，也连不了网，所以都是自己拿另外的电脑下了U盘弄过去的。
[ubuntu中使用终端查看U盘里的内容](https://blog.csdn.net/hhhhh89/article/details/54311161)
[ubuntu 16.04 LTS 降级安装gcc 4.8](https://blog.csdn.net/u011784994/article/details/80080938)
[Tensorflow不同版本要求与CUDA及CUDNN版本对应关系](https://blog.csdn.net/omodao1/article/details/83241074)
最后感谢各个外援~
