---
title: ArchLinux 安装
date: 2016-05-16 14:37:06
categories:
  - Linux
tags:
  - Linux
  - ArchLinux
---

## 上路
### 初识Arch

寒假闲的作死，非要在装了linux的机子上重装一遍windows，作死搞废GRUB引导，在[张大师的博客](https://blog.yoitsu.moe/)引导下初识Arch，配置了一天脑子都快炸了，最后在临睡觉前搞好了。
前两天没地方呆，各处实验室用不了，闲的无聊，无奈之下，装了个[virtualbox](https://www.virtualbox.org/)虚拟机，奋斗三天，装崩数次，最终装上，虽说用了不超过一小时，又雪崩了吧，好歹成功一回。
Arch并不太懂，官方介绍献上[Arch Linux (简体中文)](https://wiki.archlinux.org/index.php/Arch_Linux_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87))，当然英文好的还是建议看英文的（像我的这种英文渣......），英文文档也献上了[Arch Linux](https://wiki.archlinux.org/index.php/Arch_Linux)

<!-- more -->

## 安装
初次安装在虚拟机，主要怕把原来的系统搞废了，资料搞丢了。
### 镜像源下载
这个也不多说了Arch官网下一个啦，越新越好的了。
### 镜像烧录
虚拟机安装就不要考虑烧录U盘的问题了（貌似简单了不少）。不过要说一句虚拟机的配置，理论上，Arch Linux能在任何内存空间不小于 256MB 的i686兼容机上运行。最基本的base组中包含的包将占用约 800MB 存储空间。
### 安装开始
小白兔照着官方文档装第一次八成是失败，而且装的脑袋都要炸了。本人英语水平有限所以只能看中文的了[Beginners' guide (简体中文)](https://wiki.archlinux.org/index.php/Beginners%27_guide_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87))，英语好的还是建议看英文的[Beginners' guide](https://wiki.archlinux.org/index.php/Beginners%27_guide)。
下面就说正题了，安装：
把镜像挂载到虚拟机上，点击绿色的向右按钮就算是开机了。开机默认进入Arch的镜像安装界面，![Arch安装选择界面](/uploads/Arch安装选择界面.JPG)选择第一个就好了。
#### 连接到因特网
连接网络，这个就不要管了，虚拟机外面上上网了就不用担心了，所以此步骤跳过。
#### 更新系统时间
敲一遍命令就好了
``` zsh
# timedatectl set-ntp true
```
无返回。
然后就是识别设备，分区，格式化分区
#### 识别设备
首先要确定系统安装的目标设备，下面命令会显示所有连接到系统的设备和分区状况：
``` zsh
# lsblk
```
最少会有两个硬盘，一个是镜像盘，另一个是虚拟机虚拟的硬盘，硬盘大小看你虚拟机虚拟的大小啦。
#### 创建新分区表
虚拟机没有使用，自然没有分区了，所以一定要分区的（x按照虚拟机的情况去改就好了）：
``` zsh
# parted /dev/sdx
```
为 BIOS 系统创建 MBR/msdos 分区表（这个不用想了，virtualbox虚拟机只支持msdos分区，所以直接敲命令就好了）：
``` zsh
(parted) mklabel msdos
```
#### 用 parted 进行分区
因为是虚拟机，而且linux要求把可执行文件放在home文件下，所以只分了一个区：
``` zsh
(parted) mkpart primary ext4 1M 100%
```
退出parted
``` zsh
(parted) quit
```
#### 格式化文件系统
先查看所有分区：
``` zsh
# lsblk /dev/sdx
```
建议用 ext4 文件系统格式化分区（就一个分区，不要想了sda1就行了）：
``` zsh
# mkfs.ext4 /dev/sdx1
```
然后就是挂载
``` zsh
# mount /dev/sdx1 /mnt
```
#### 安装
选择安装镜像
编辑文件 /etc/pacman.d/mirrorlist
官方不知道为何使用中科大的镜像源，163的速度比较快，但是据说费流量，中科大的貌似最近学校限制出口速度，所以选择哪个镜像源自己看自家网络情况定了。
``` zsh
# nano /etc/pacman.d/mirrorlist
---------------------------------------
Server = http://mirrors.ustc.edu.cn/archlinux/$repo/os/$arch
```
留下上面那行，剩下全部删了就好了。
这个留的是中科大的，163的看见里面有163的就是163的镜像站了。
更改镜像列表后请务必使用命令强制刷新，命令如下：
``` zsh
# pacman -Syy
```
当年因为这个细节崩了数次，说出来都是泪啊
#### 安装基本软件包
没啥好说的了，直接上命令：
``` zsh
# pacstrap -i /mnt base base-devel
```
#### 配置
##### fstab
用以下命令生成 fstab. 之所以用 UUID 是因为它们能唯一且独立地标识（具体是嘛我也不懂，照着输嘛）：
``` zsh
# genfstab -U -p /mnt > /mnt/etc/fstab
```
*强烈建议* 在执行完以上命令后，后检查一下生成的 /mnt/etc/fstab 文件是否正确。若在运行 genfstab 或是之后发生错误，后续修改请直接手动编辑 fstab 文件。装了几次，没遇上过问题（官网上说的跟我能看懂似的）。
##### chroot
说实话这个是嘛我也不知道，总之不照着输后头就是错（对了，前面务必别忘了挂载，之前干了不知道多少回这个事情了）
``` zsh
# arch-chroot /mnt /bin/bash
```
##### Locale
本地化的程序与库若要本地化文本，都依赖 Locale, 后者明确规定地域、货币、时区日期的格式、字符排列方式和其他本地化标准等等。在下面两个文件设置：locale.gen 与 locale.conf.
/etc/locale.gen是一个仅包含注释文档的文本文件。指定您需要的本地化类型，只需移除对应行前面的注释符号（＃）即可，建议选择帶UTF-8的项：
``` bash
# nano /etc/locale.gen
---------------------------------------
en_US.UTF-8 UTF-8
zh_CN.UTF-8 UTF-8
zh_TW.UTF-8 UTF-8
```
接着执行locale-gen以生成locale讯息：
``` bash
# locale-gen
```
创建 locale.conf 并提交您的本地化选项：
``` bash
# echo LANG=en_US.UTF-8 > /etc/locale.conf
```
#### 时间
可用的时区全集中在 /usr/share/Zone/SubZone 目录里了。选择时区：
``` bash
# tzselect
```
将 /etc/localtime 软链接到 /usr/share/zoneinfo/Zone/SubZone，国内不知道什么情况，没有北京，只有上海，所以软链接到上海就好：
``` bash
# ln -s /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
```
建议设置时间标准为 UTC，并调整 时间漂移：
``` bash
# hwclock --systohc --utc
```
#### 设置 Root 密码
用 passwd 设置一个 root 密码，root密码就是最高用户权限啦：
``` bash
# passwd
```
#### BIOS/MBR
下面介绍在 MBR 系统上使用 Grub。安装 grub 和 os-prober 包，并执行 grub-install. 须根据实际分区自行调整 /dev/sda, 切勿在块设备后附加数字，比如 /dev/sda1 就不对。
``` bash
# pacman -S grub os-prober
# grub-install /dev/sda
```
下面命令会自动生成配置文件
``` bash
# grub-mkconfig -o /boot/grub/grub.cfg
```
详情参考[GRUB (简体中文)](https://wiki.archlinux.org/index.php/GRUB_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87))官方文档 。
#### 配置网络
该过程与#建立网络连接基本一致，只不过该配置在新系统每次开机时都会自动生效。
##### 主机名
设置个您喜欢的主机名，例如：
``` bash
# echo myhostname > /etc/hostname
```
myhostname替换为你喜欢的主机名，不是用户名。
##### 有线网络
如果您只用单一且固定的有线网络连接，启动 dhcpcd 服务，interface 是您的网络接口名：
``` bash
# systemctl enable dhcpcd@interface.service
```
#### 卸载分区并重启系统
设置 root 密码（前面干过一遍了，不知为何官方文档又来一遍，反正我是照做了，差不了几分钟，最后崩了重来更费事）：
``` bash
# passwd
```
离开 chroot 环境：
``` bash
# exit
```
重启计算机：
``` zsh
# reboot
```
虚拟机里面重启电脑默认进入虚拟光盘，所以啦，建议“关机”后“退出”光盘，再开机；不过鄙人一般先重启摁F12，选硬盘启动就好了。
### 安装之后
安装之后不要以为就万事大吉了，现在进入系统还是黑底白字的命令行，就像这样，只不过还没有登录![安装好后的Arch](/uploads/安装好后的Arch.JPG)
而且只有一个root用户，接下来看这篇文章[General recommendations (简体中文)](https://wiki.archlinux.org/index.php/General_recommendations_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87))
现在心烦意乱的，剩下的以后再写吧。
