---
layout: post
title: ZYBO開發板：linux作業系統移植
date: 2018-04-09
author: huang
header-img: img/green.jpg
catalog: true
tags: zybo, linux
keywords: zybo,linux,xilinx,zynq
---


摘要
文章主要介紹zybo board上linux作业系统移植过程。分别介绍了开发环境搭建、U-boot编译、linux内核编译、busybox制作等流程及注意事项。文章使用的开发板是zynq 7000系列的zybo board（开发板照片在文章后面）。Vivado版本是2015.1.主机系统是Debian 9.



-------------------

## 1.开发环境搭建
 工欲善其事必先利其器，做开发前搭建好编译环境是重要的一步，这些步骤大体上都相同，然而对于不同的系统平台、硬件平台，环境的搭建也会有些差别。虽然网上大多推荐使用ubuntu LTS作为开发环境，但是我还是倾向使用Debian系统,在Debian 9系统上，要安装一些依赖包，否则无法进行下面的工作，因为在我的机器上我遇到了这些问题，至于其他的机器是什么情况，我就无法得知了。
 首先安装以下软件包：
 ``` 
 apt install gcc-multilib libmpfr-dev bc
 ```
 在我的机器上，如果不安装gcc-multilib的话进行CROSS_COMPILE时会提示找不到相关的编译器，比如说找不到arm-linux-gnueabihf-gcc.没有安装bc时在编译uImage时会报错。
 ```
 /bin/sh: 1: bc: not found
 ```
如果机器上没有装u-boot-tools的话也要记得先安装。
```
apt install u-boot-tools
```
推荐手动添加路径,在主目录下：
```
nano ～/.profile
```
在里面添加:
```
export PATH=$PATH:/opt/Xilinx/SDK/2017.4/gnu/aarch32/lin/gcc-arm-linux-gnueabi/bin
export CROSS_COMPILE=arm-linux-gnueabihf-
export ARCH=arm 
 ```

 然后执行source指令初始化家目錄下的.profile檔案，使上面添加的交叉編譯器的PATH環境變數生效。
``` 
source ~/.profile
```

## 2.U-boot编译
首先下载DigilentInc的 U-boot 版本，地址在[这里](https://github.com/Xilinx/u-boot-xlnx)，当然用git命令更方便
```
git clone https://github.com/Xilinx/u-boot-xlnx.git
```
 然后进入u-boot-xlnx-master目录，编译时使用CROSS_COMPILE选择交叉编译器。
```
CROSS_COMPILE=arm-linux-gnueabihf- make zynq_zybo_config
CROSS_COMPILE=arm-linux-gnueabihf- make -j2
```
编译完成后，在u-boot-xlnx-master根目录下生成一个u-boot文档，这就是开发板需要用到的可执行文件。由于xilinx只能识别.elf后缀名的可执行文件，所以要把u-boot该名为u-boot.elf
```
cp -v u-boot u-boot.elf
```

## 2.Linux kernel编译
接下来编译linux kernel.同样在github上下载DigilentInc的linux kernel版本，下载地址在[这里](https://github.com/Xilinx/linux-xlnx),使用git命令更方便

```
git clone https://github.com/Xilinx/linux-xlnx.git
```
下载完成后进入linux-xlnx-master目录编译。

```
ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- make xilinx_zynq_defconfig
ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- make -j2
ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- make UIMAGE_LOADADDR=0x8000 uImage
```
完成后会生成我们需要的uImage,此文件在arch/arm/boot/uImage
我还试了另一种制作uImage的方法，同样也能工作;所以如果第一种方法不行时可以考虑用第二种方法。

```
mkimage -n 'arm-linux' -A arm -O linux -T kernel -C none -a 0x30008000 -e 0x30008040 -d zImage uImage

#地址可以根据实际情况.
#mkimage命令在u-boot-xlnx-master/tools/下面执行，如果提示找不到命令可以切换到此目录下面操作。
```
编译设备树文件前先编辑一下zynq-zybo.dts文件：

1）如果不使用nfs根目录系统时，在bootargs改为：
```
chosen {
		bootargs = "console=ttyPS0,115200 root=/dev/ram earlyprintk earlycon";
		stdout-path = "serial0:115200n8";
	};
```
2）改完后执行(注意命令的执行要在linux-xlnx-master目录下操作)：
```
CROSS_COMPILE=arm-linux-gnueabihf- make zynq-zybo.dtb
```
zynq-zybo.dtb文件在arch/arm/boot/dts/zynq-zybo.dtb;此外需要将zynq-zybo.dtb重命名为devicetree.dtb

```
cp -v zynq-zybo.dtb devicetree.dtb
```
## 3.BusyBox制作
Busybox在单一的可执行文件中提供了精简的Unix工具集，可运行于多款POSIX环境的操作系统，例如Linux,包括Android、Hurd、FreeBSD等等。BusyBox可以被自定义化以提供一个超过两百种功能的子集。它可以提供多数详列在单一UNIX规范里的功能，以及许多用户会想在Linux系统上看到的功能。BusyBox使用ash。在 BusyBox的网站上可以找到所有功能的列表。------摘自维基百科
下载BusyBox的源代码：

git clone git://git.busybox.net/busybox -b 1_28_stable
在这里选择1_28_stable较新的分支，当然，也可以下载其他分支。接下来进行BusyBox的配置，跟配置linux kernel时差不多。这里先使用defconfig,使用menuconfig配置

```
ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- make defconfig
ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- make menuconfig
```
当然，如果使用menuconfig配置出现错误时可能是你的电脑系统没有装libncurses5-dev库，对于Debian系统可以使用下面的命令来安装。

```
apt-get install libncurses5-dev
```


设定完成后开始进行编译

```
ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- make -j2
```
然后使用make install命令把编译出来的BusyBox安装到默认的_install文件夹里。

```
ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- make install
```
进入_install文件夹里建立一些缺失的文件夹，文件夹的建立遵循FHS标准。

```
cd _install
mkdir -p dev proc sys root etc/init.d home mnt opt var lib tmp
```
这看起来和一个完整的linux文件夹很像吧。当然，可以根据具体项目需要建立这些文件夹，这里建立的文件夹只是一个参考。
接下来创建 etc/init.d/rcS 作为系统的启动脚本: nano etc/init.d/rcS

```
#在里面添加以下内容

#Begin /etc/init.d/rcS

#!/bin/sh
PATH=/bin:/sbin:/usr/bin:/usr/sbin
runlevel=S
mount -a
mount -t proc none /proc
mount -t sysfs none /sys
mkdir /dev/pts
mount -t devpts devpts /dev/pts
#執行此句之前要先在上面掛載/proc /sys,否則會出現錯誤
echo /sbin/mdev > /proc/sys/kernel/hotplug
/sbin/mdev -s
#把localhost切換成你想要的主機名
/bin/hostname localhost
export PATH runlevel

#End /etc/init.d/rcS
```
然后为rcS添加可执行权限

```
chmod +x etc/init.d/rcS
```
再建立 etc/inittab 文件

```
nano etc/inittab
#在里面添加以下内容
#Begin /etc/inittab

#!/bin/sh
#Init script
::sysinit:/etc/init.d/rcS
#Start shell on the serial ports
::respawn:/sbin/getty -L ttyPS0 115200 vt100
#What to do when restarting the init process
::restart:/sbin/init
#What to do before rebooting
::shutdown:/bin/umount -a -r

#End /etc/inittab
```
添加/etc/profile文件

```
nano /etc/profile
	#Begin /etc/profile

	#Ash profile
	#vi:syntax=sh
	#No core files by default
	#ulimit -S -c 0 > /dev/null 2>&1
	USER="`id -un`"
	LOGNAME=$USER
	PS1='[\u@\h \W]#'
	PATH=$PATH
	HOSTNAME=`/bin/hostname`
	echo "Processing /etc/profile..."
	echo "Done"
	export PATH USER LOGNAME PS1

	#End /etc/profile
```

设定一个 etc/passwd 文件，这样可以免去输入root的密码

```
nano etc/passwd
#在里面添加以下内容

#Begin /etc/passwd

root::0:0:root:/root:/bin/sh

#End /etc/passwd
```

建立 /init 并软链接到 /sbin/init ，这非常重要，可以避免 Linux Kernel 开机时找不到 rootfs 的 init。

```
ln -s sbin/init init
```
如果启动开发板后出现以下信息
```
getty: can't open '/dev/ttySAC0': No such file or directory
getty: can't open '/dev/ttySAC0': No such file or directory
getty: can't open '/dev/ttySAC0': No such file or directory
getty: can't open '/dev/ttySAC0': No such file or directory
getty: can't open '/dev/ttySAC0': No such file or directory
getty: can't open '/dev/ttySAC0': No such file or directory
getty: can't open '/dev/ttySAC0': No such file or directory
```
要在busybox/_install/dev/下建立一些特殊的文件，这需要用root权限执行
```
mknod console c 5 1
mknod ttyS0 c 204 64
mknod ttySAC0 c 204 64
mknod ttyPS0 c 204 64
```
然后重启开发板，如果不用nfs网络根文件系统时要重新打包成uramdisk.image.gz再使用。

从交叉编译工具链复制一些文件到相应目录下，不然用户自己移植的一些软件无法运行。

```
cp -v /opt/Xilinx/SDK/2017.4/gnu/aarch32/lin/gcc-arm-linux-gnueabi/arm-linux-gnueabihf/libc/lib/libc-2.3.2.so _install/lib
cp -v /opt/Xilinx/SDK/2017.4/gnu/aarch32/lin/gcc-arm-linux-gnueabi/arm-linux-gnueabihf/libc/lib/libc.* _install/lib
cp -v /opt/Xilinx/SDK/2017.4/gnu/aarch32/lin/gcc-arm-linux-gnueabi/arm-linux-gnueabihf/libc/lib/libm-2.3.2.so _install/lib
cp -v /opt/Xilinx/SDK/2017.4/gnu/aarch32/lin/gcc-arm-linux-gnueabi/arm-linux-gnueabihf/libc/lib/libm.* _install/lib
cp -v /opt/Xilinx/SDK/2017.4/gnu/aarch32/lin/gcc-arm-linux-gnueabi/arm-linux-gnueabihf/libc/lib/ld-* _install/lib
cp -v /opt/Xilinx/SDK/2017.4/gnu/aarch32/lin/gcc-arm-linux-gnueabi/arm-linux-gnueabihf/libc/lib/libthread_db* _install/lib
```

把编译出来的rootfs打包成cpio格式

```
find . |  cpio -H newc -o | gzip -9 > ../uramdisk.cpio.gz
```
之后再通过 mkimage 將 uramdisk.cpio.gz 文件转成 uboot 使用的 uramdisk.image.gz

```
 mkimage -A arm -T ramdisk -C gzip -d ../uramdisk.cpio.gz ../uramdisk.image.gz
```
![这里写图片描述](https://github.com/huangwenshan1999/huangwenshan1999.github.io/raw/master/post_img/vivado/8.png)
这个生成的uramdisk.image.gz是我们需要的文件，再制作BOOT.bin文件。


##5.制作BOOT.bin文件
**注意**：这部分我是在windows 10平台下用vivado2015.1做的.
1.这里假设PL部分已经做好。**提示**：zybo的板级描述文件可以在[这里](https://github.com/ucb-bar/fpga-zynq/blob/master/zybo/src/xml/ZYBO_zynq_def.xml) 下载，建立工程后导入xml文件。
2.启动SDK![](https://github.com/huangwenshan1999/huangwenshan1999.github.io/raw/master/post_img/vivado/1.png) 


3.在SDK的file选单选择New->Application Project.![ ](https://github.com/huangwenshan1999/huangwenshan1999.github.io/raw/master/post_img/vivado/2.png)


4.在会话框中，Project name可以随便填,OS Platform选择standalone,Hardware Platform默认是my_wrapper_hw_platform_0,根据实际选择对应的Hardware Platform,并点击next进入下一步。![](https://github.com/huangwenshan1999/huangwenshan1999.github.io/raw/master/post_img/vivado3.png) 


5.在Templates单里选择Zynq FSBL,之后点击Finish。![](https://github.com/huangwenshan1999/huangwenshan1999.github.io/raw/master/post_img/vivado/4.png) 


6.在SDK左侧栏目中，右击刚刚建立的FSBL项目，在弹出的选单中选择Buid Project.![](https://github.com/huangwenshan1999/huangwenshan1999.github.io/raw/master/post_img/vivado/5.png) 


7,在SDK顶部工具栏点击Create Zynq Boot Image。![](https://github.com/huangwenshan1999/huangwenshan1999.github.io/raw/master/post_img/vivado/6.png) 



8.在Create Zynq Boot Image对话框中，点击Add添加上面制作好的u-boot.elf文件，添加后点击OK.![](https://github.com/huangwenshan1999/huangwenshan1999.github.io/raw/master/post_img/vivado/7.png) 


9.最后点击Create Image, 至此BOOT.bin制作完成。

附：使用命令建立BOOT.bin更加方便，首先要编写一个.bif文件，这里暂且命名boot.bif，并且假设它在/home/huang/xilinx了目录下。
```
the_ROM_image:
{
	[bootloader]/home/huang/xilinx/workspace/zybo/zybo.sdk/FSBL/bootimage/FSBL.elf   #只能有一個bootloader，並且只能是FSBL.elf文檔
	/home/huang/xilinx/workspace/zybo/zybo.sdk/design_1_wrapper_hw_platform_0/design_1_wrapper.bit
	/home/huang/xilinx/dev/u-boot-Digilent-Dev/u-boou.elf
}
```

然后执行bootgen指令，bootgen指令在/opt/Xilinx/SDK/2015.4/bin/下面。
```
bootgen -image boot.bif -w on -o i BOOT.bin
```

之后把BOOT.bin,uramdisk.image.gz,devicetree.dtb,uImage复制到sd卡的第一个分区里。
![要复制进SD卡第一分区的文件](https://github.com/huangwenshan1999/huangwenshan1999.github.io/raw/master/post_img/zynq/4.png)

## 5.启动系统
最后把复制好文件的SD卡插入开发板卡槽，通电后检查是否正常。开发板上电后如果上面的Done指示灯亮，如下图，证明上面制作的文件是可以启动的：
![开发板上电状态](https://github.com/huangwenshan1999/huangwenshan1999.github.io/raw/master/post_img/zynq/1.png)

注意：我的zybo开发板设置使用USB1,也有些使用USB0，可以使用命令

```
ls /dev/ttyUSB   
```
按Tab键查看有多少个USB口，然后选择合适的。比如我的机器是ttyUSB1
![显示的USB串口](https://github.com/huangwenshan1999/huangwenshan1999.github.io/raw/master/post_img/zynq/2.png)

然后在终端输入：
```

sudo minicom -D /dev/ttyUSB1

```
启动成功后大概是这样的：

![开发板成功启动linux的画面](https://github.com/huangwenshan1999/huangwenshan1999.github.io/raw/master/post_img/zynq/3.png)

## 6.结语
 文章到此已经讲解完本次的linux系统移值过程，包括开发环境搭建，内核编译，uboot编译，BusyBox制作，相信读者自己动手，参考教程走完一遍流程的话会对所学过的知识有更进一步的了解。我认为学以致用是比较好的学习方式。由于这是我第一次接触zynq系列的开发板，所以也有很多不完善的地方，如果有文章有错误或读者有疑问的地方，欢迎留言.

## 7.引用
[1].[zybo board 開發記錄: 執行 Linux 作業系統](https://coldnew.github.io/d9dfdd56/)
[2].维基百科.[BusyBox](https://zh.wikipedia.org/zh-cn/BusyBox)
