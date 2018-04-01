# Archlinux安裝指南（uefi+gpt）

摘要：文章介绍了基于uefi+gpt模式下archlinux基本系统和图形界面的安装、一些基本的系统优化。

[TOC]


-------------------

##1.系统安装前的一些准备
   首先在[Archlinux官网](https://www.archlinux.org/download/)下载镜像文件,然后刻录到U盘或光盘上。在我的机器上刻录arch镜像文件到U盘后启动不了，因为时间问题我没做过多的探索，所以后面直接刻录到光盘，用光驱启动成功（我用的刻录工具是软碟通）。
       加载完成进入系统后，首先用parted分区工具对硬盘进行分区。以下是用pared工具分区的示例：
   分区情况：
   硬盘分区                       挂载点                         文件系统
   /dev/sda1                       /boot/efi                          vfat
  /dev/sda2                        /boot                                ext2
  /dev/sda3                         swap
   /dev/sda4                      /                                        btrfs
   /dev/sda5                     /var                                    btrfs
   /dev/sda6                     /usr                                     btrfs
   /dev/sda7                    /home                                  btrfs
 首先在终端里输入命令parted进入分区界面
 

```
#parted /dev/sda  //若对其他硬盘分区，把sda改为sdb或sdc等
```
然后建立gpt分区表，**注意：**这步会擦除硬盘上的所有数据！如果此硬盘装有其他系统，或者要装双系统的同学，则可跳过建立GPT分区表和ESP分区这两部。一般装有win10系统的电脑已经建有ESP分区，可以直接使用。

```
#(parted)mklabel gpt   //(parted)前缀表示已进入parted分区工具界面，可进行分区操作。

```
建立ESP分区，用于挂载efi分区（uefi模式必须要有efi分区）

```
# (parted)mkpart primary 2048s  512M  
```
建立其他分区（分区数量因人而异，也可以只划分swap分区和/分区）

```
#(parted)mkpart primary 512  812M  //boot分区大小为300M
#(parted)mkpart primary 812  663440M //swap分区，大小为4G， //若机器内存>=4G并且不需要休眠功能的话则可以不需要交换分区.
#(parted)mkpart primary 663440 673680M //根分区,大小为10G
#(parted)mkpart primary 673680 689040M //var分区，大小为15G
#(parted)mkpart primary 689040 709520M  //usr分区，大小为20G
#(parted)mkpart primary 709520  -1    //home分区，大小为硬盘的剩余大小。
```
**注意:**分区时注意4k对齐

设定ESP分区标志为boot

```
#(parted)set 1 boot on  
```
查看

```
#(parted)q
```
退出parted

```
#(parted)q
```
查看硬盘的分区情况

```
#lsblk -l
```
接下来格式化硬盘分区
ESP分区格式化成fat32文件系统

```
#mkfs.vfat -F32 /dev/sda1     当询问是否覆盖文件的时候，选yes
```
/boot分区格式化成ext2文件系统

```
#mkfs.ext2 /dev/sda2
```
创建交换分区

```
#mkswap /dev/sda3     //若机器内存>=4G并且不需要休眠功能的话则可以不需要交换分区.
```

其他分区格式化成btrfs,我个人比较喜欢这个文件系统，当然比较大众的文件系统是ext4

```
#mkfs.btrfs -f /dev/sda4   //-f参数是强制格式化，如果硬盘上有数据的话要加参数-f才能格式化硬盘分区。
#mkfs.btrfs -f /dev/sda5 
#mkf.btrfs -f /dev/sda6
#mkfs.btrfs -f /dev/sda7
```
若使用ext4,则

```
#mkfs.ext4 -t /dev/sda4     //其他分区依次类推

```

 然后挂载分区
 1.先挂载根分区到/mnt
 

```
#mount /dev/sda4 /mnt  //sda4为根分区,然后在sda4分区上建立所需的系统文件夹
#cd /mnt
#mkdir -v boot usr var home
//建立完成后检查一下/mnt下是否都有这几个文件夹
```
2.挂载/dev/sda2到/mnt/boot

```
#mount /dev/sda2 /mnt/boot
```
3.然后在/mnt/boot里创建efi挂载点

```
#mkdir -v /mnt/boot/efi
#mount /dev/sda1 /mnt/boot/efi  //挂载/dev/sda1到/mnt/boot/efi
```
4.挂载/dev/sda5 到/mnt/var

```
#mount /dev/sda5 /mnt/var
```
5.挂载/dev/sda6到/mnt/usr

```
#mount /dev/sda6 /mnt/usr
```
6.挂载/dev/sda7到/mnt/home

```
#mount /dev/sda7 /mnt/home
```
7.激活swap分区

```
#swapon /dev/sda3    //若机器内存>=4G并且不需要休眠功能的话则可以不需要交换分区.
```
至此，安装前的准备已完成。
##2.基础系统安装
1.编辑/etc/pacman.d/mirrorlist文件

```
#nano /etc/pacman.d/mirrorlist
```
只保留中国的源，顺便留一个美国的源和一个台湾的源，可以使用ctl+k去掉不需要的源，最后，用ctl+k(剪切）、ctl+u(粘贴)把剩下的源排顺序，建议把ustc和163的源放在最上面，因为这两个源速度真的很快。弄好后ctl+o保存变更，最后ctl+x退出编辑器。
2.联网，没有网路用arch不现实

```
#wifi-menu   //选择无线网路；若是有线网路，arch已经自动启动dhcpcd来联网
```
3.安装基本系统

```
pacstrap -i /mnt base base-devel    //安静的等待几分钟.....
```
4.生成fstab文件

```
#genfstab -U -p /mnt >> /mnt/etc/fstab
```
5.确认是否生成无误

```
#nano /mnt/etc/fstab
```
fstab文件看上去是像这样的，以我的为例

```
#
# /etc/fstab: static file system information
#
# <file system> <dir>   <type>  <options>       <dump>  <pass>
# /dev/sda9
UUID=1603cd25-707d-4d04-a542-e612b8821010       /               btrfs           rw,noatime,ssd,space_cache,subvolid=5,subvol=/  0 0

# /dev/sda2
UUID=1336841b-4b6a-4af4-a8e6-46747a2bb15e       /boot           ext2            rw,noatime,block_validity,barrier,user_xattr,acl,stripe=4       0 2

# /dev/sda1
UUID=E401-83AC          /boot/efi       vfat            rw,noatime,fmask=0022,dmask=0022,codepage=437,iocharset=iso8859-1,shortname=mixed,errors=remo$

................
//建议把relatime改成noatime（我的第二个挂载参数)


```
6.chroot到新安装的系统

```
#arch-chroot /mnt
```
##3.系统安装后的一些配置
7.时区与编码(编辑locale.gen文件，去掉en_US.UTF-8,zh_CN.UTF-8,zh_CN.GBK，zh_TW.前面的#

```
#vi /etc/locale.gen  
```
然后重建编码表

```
#locale-gen
```
8.设置时区

```
#tzselect

#ln -S /usr/share/zoneinfo/Asia/Shanghai /etc/localtime  

#hwclock --localtime  //同步本地时钟

```
10.设置主机名

```
#echo YourHostName > /etc/hostname  
```
11.设置语言

```
# echo LANG=en_US.UTF-8 > /etc/locale.conf
```

12.**重要的一步**，因为本次安装系统时，/usr分区是独立分出去的，所以必须要**修改/etc/mkinitcpio.conf文件**，否则安装完重启后不能进入新系统。

```
#nano /etc/mkinitcpio.conf  
```
请按如下说明修改之。

```
第7行  把 MODULEA="" 修改为  MODULES="ahci btrfs"  //这项属于一般的优化问题，不改也可以。
第52行 HOOKS="base udev autodetect modconf block filesystems keyboard fsck" 在末尾加入usr shutdown 
HOOKS="base udev autodetect modconf block filesystems keyboard fsck usr shutdown"
第60行 把COMPRESSION="xz"前面的#号去掉，xz的压缩效率更高。
```
别急，等下一步安装btrfs-progs时会自动更新.img文件！
14.安装btrfs-progs(因为本次安装使用btrfs文件系统，所以要安装这个包；对于ext4文件系统可以不安装这些包,;其他文件系统，比如nilfs2,需要相应的安装nilfs-utils包）

```
#pacman -S btrfs-progs
```
13.安装一些必要软件

```
pacman -S dialog wpa_supplicant  //安装联网软件，否则重启进入新     系统后不能连上wifi
pacman -S grub efibootmgr os-prober(双系统需要) 
//然后安装引导管理器
#grub-install --efi-directory=/boot/efi --bootloader-id=grub
#grub-mkconfig -o /boot/grub/grub.cfg
```
15.添加用户

```
#useradd -m -g users -G wheel -s /bin/bash 用户名
#passwd 用户名    //为用户设置密码
```
##4.安装图形服务（xorg）
16.安装图形驱动
```
#pacman -S xorg-server xorg-xinit 
```
17.安装触控板驱动

```
#pacman -S xf86-input-synaptics 
```
18.安装显卡驱动(根据机器要求安装）

```
#pacman -S xf86-video-intel  //intel集成显卡驱动
#pacman -S xf86-video-amdgpu  //amd集成显卡驱动
#pacman -S xf86-video-ati     //ati显卡驱动
//可以使用pacman -Ss xf86-video 查询是否有自己需要的显卡驱动
```
##4.安装图形服务（wayland）
若使用wayland图形服务
```
#pacman -S wayland  weston
```
19.安装字体

```
#pacman -S ttf-dejavu wqy-microhei  
```
##5.安装桌面(gnome)
20.安装gnome桌面环境
**注意:**新的gnome默认使用wayland,请注意下面关于wayland的配置。
```
#pacman -S gnome
```
21.安装登录管理器gdm

```
#pacman -S gdm
#systemctl enable gdm.service //开机自动启动gdm
```
##6.系统安装完成后的配置
22.安装fcitx输入法

```
pacman -S fcitx fcitx-im fcitx-configtool
```
对于xorg，要配置.xinitrc文件，没有这个文件时要自己创建。

```
$nano ~/.xinitrc
添加以下内容
export GTK_IM_MODULE=fcitx
export QT_IM_MODULE=fcitx
export XMODIFIERS="@im=fcitx"
```
若是wayland,则在/etc/environment里配置（此教程示例安装gnome3，它默认使用wayland,所以要配置/etc/environment，否则不能使用拼音输入法。）

```
#nano /etc/environment
在末尾添加以下内容
export GTK_IM_MODULE=fcitx
export QT_IM_MODULE=fcitx
export XMODIFIERS="@im=fcitx"
```
23.添加archlinuxcn源
在 /etc/pacman.conf 文件末尾添加两行：

```
[archlinuxcn]
Server = https://mirrors.ustc.edu.cn/archlinuxcn/$arch
```
然后安装 archlinuxcn-keyring 包以导入 GPG key

```
#pacman -S archlinuxcn-keyring
```
24.安装yaourt

```
#pacman -S yaourt   //archlinux是社区用户软件仓库。
```
25.安装sudo

```
pacman -S sudo   //为普通用户配置sudo权限
```
然后配置/etc/sudoers,找到

```
## User privilege specification
##
root ALL=(ALL) ALL    //在第79行
用户名 ALL=(ALL) ALL  //在第80行添加这行，即可为普通用户添加sudo权限
```
26.安装搜拼音输入法（前提是安装了yaourt或添加了archlinuxcn源)

```
$yaourt -S fcitx-sogoupinyin   或
#pacman -S fcitx-sogoupinyin
```
27.安装networkmanager

```
#pacman -S networkmanager
#systemctl enable NetworkManager.service //开机启动网络管理程序，然而你没看错，这两个字母确实是大写。
```
28.为root设置密码，否则root密码默认为空，非常危险。

```
#passwd
```
29.退出chroot环境，重启电脑进入刚安装好的arch

```
#exit    //退出chroot
#reboot  //重启电脑
```

到此，已基本差不多完成archlinux系统的安装与设置


##7.优化及注意事项
30.安装tlp电源管理软件

```
#pacman -S tlp tlp-rdw  //其他电脑
#pacman -S tlp tlp-rdw tp-smapi-dkms acpi-call-dkms //thinkpad电脑使用此项
```
tlp的配置文件在/etc/default/tlp内，可以根据自己的需求配置，对于笔记本在意续航时间的话可以配置成节能型，里面的参数设置已有详细说明。
设置tlp开机启动

```
#systemctl enable tlp.service
#systemctl enable tlp-sleep.service
```
31.对于双显卡电脑禁用Nvidia显卡
   首先安装bbswitch
   

```
pacman -S bbswitch
```
然后在/etc/modules-load.d下新建bbswitch.conf，并修改为如下内容。

```
bbswitch
```
三，在/etc/modprobe.d/下新建bbswitch.conf，并修改为如下内容。

```
options bbswitch load_state=0 
```
最后，在/etc/modprobe.d/下新建nouveau_blacklist.conf，并修改为如下内容。

```
blacklist nouveau  
blacklist nvidiafb  
```
对于有NVIDIA显卡的电脑，若想使用双显卡，可以用bumblebee来设置，由于在archlinux平台上鄙人没有使用过bumblebee来实现双显卡切换，只是简单禁用而已，所以此问题请参阅[Bumblebee ArchWiki](https://wiki.archlinux.org/index.php/Bumblebee_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87))
32.安装一些常用软件

```
yaourt -S netease-cloud-music //安装网易云音乐
pacman -S wps-office      //安装金山办公软件
pacman -S zip unzip rar unrar p7zip  //压缩与解压缩软件
pacman -S file-roller   //归档管理器
pacman -S firefox      //安装火狐浏览器
pacman -S google-chrome  //安装谷歌浏览器
```
更多问题请移步[Arch Wiki](https://wiki.archlinux.org/index.php/Main_page_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87))。享受探索给你带来的乐趣吧！欢迎加入Arch社区！





