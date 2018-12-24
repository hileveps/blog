---
layout: post
title: "PXE+DHCP+TFTP+vsftpd+KickStart配置无人值守安装Linux"
subtitle: '能用代码解决的，就别动手！'
date: "2018-12-20 19:43:17"
author: dairugang
cover: '/images/2018-12-20-pxe-ftp-kickstart-auto-install-linux/cover.jpg'
tags: Linux
---

尝试通过PXE，FTP，KickStart等来配置服务器，实现无人值守安装Linux。

<!-- more -->

## PXE介绍
PXE(preboot execute environment，预启动执行环境)是由Intel公司开发的最新技术，
工作于Client/Server的网络模式，支持工作站通过网络从远端服务器下载映像，
并由此支持通过网络启动操作系统，在启动过程中，终端要求服务器分配IP地址，
再用TFTP（trivial file transfer protocol）或MTFTP(multicast trivial file transfer protocol)协议
下载一个启动软件包到本机内存中执行，由这个启动软件包完成终端（客户端）基本软件设置，
从而引导预先安装在服务器中的终端操作系统。
PXE可以引导多种操作系统，如：Windows95/98/2000/windows2003/windows2008/winXP/win7/win8,linux系列系统等。

来自：[ 百度百科 ](https://baike.baidu.com/item/PXE)

## KickStart介绍

KickStart是一种无人职守安装方式。KickStart的工作原理是通过记录典型的安装过程中所需人工干预填写的各种参数，
并生成一个名为ks.cfg的文件；在其后的安装过程中（不只局限于生成KickStart安装文件的机器）当出现要求填写参数的情况时，
安装程序会首先去查找KickStart生成的文件，当找到合适的参数时，就采用找到的参数，当没有找到合适的参数时，
才需要安装者手工干预。这样，如果KickStart文件涵盖了安装过程中出现的所有需要填写的参数时，
安装者完全可以只告诉安装程序从何处取ks.cfg文件，然后去忙自己的事情。
等安装完毕，安装程序会根据ks.cfg中设置的重启选项来重启系统，并结束安装。

以上内容来自：[https://www.cnblogs.com/cloudos/p/8143929.html]()

---- 
## 基本原理

### 流程图

![PXE请求流程](/images/2018-12-20-pxe-ftp-kickstart-auto-install-linux/pxe_flow.jpg "PXE请求流程")

![PXE请求流程2](/images/2018-12-20-pxe-ftp-kickstart-auto-install-linux/pxe_flow_ks.jpg "PXE请求流程2")

上图来源：[这里](https://www.cnblogs.com/cloudos/p/8143929.html)



### 关键步骤说明

1. 客户机接上网线后，从网口启动
1. 通过udp寻找dhcp服务
1. dhcp服务器给客户机分配ip，并告知他在那里请求 `pxelinux.0`
1. 客户机请求pxelinux.0文件并执行
1. 客户机请求配置文件pxelinux.cfg(主要是里面的default文件)
1. 客户机请求vmlinuz等文件
1. 启动Linux内核开始安装
1. 请求应答ks文件
1. 安装操作系统（包括从ftp中下载rpm包等）

---- 
## 具体步骤

以下开始是具体步骤,作业环境为：

CentOS Linux release 7.5.1804 (Gnome桌面)  
Linux内核：3.10.0-862.el7.x86_64

## 配置DHCP服务

DHCP服务的作用就是为局域网的客户机分配ip地址，并且告诉他们，获取引导文件的地址。

- 安装dhcp服务 

```bash
yum install dhcp
```

- 设置配置文件

```conf
# cat /etc/dhcp/dhcpd.conf

#
# DHCP Server Configuration file.
#   see /usr/share/doc/dhcp*/dhcpd.conf.example
#   see dhcpd.conf(5) man page
#
option client-system-arch code 93 = unsigned integer 16;

allow booting;
allow bootp;

ddns-update-style none;

subnet 192.168.10.0 netmask 255.255.255.0 {
        range 192.168.10.100 192.168.10.200;
        option subnet-mask 255.255.255.0;
        default-lease-time 21600;
        max-lease-time 43200;
        class "pxeclients" {
		match if substring (option vendor-class-identifier, 0, 9) = "PXEClient";
	        next-server 192.168.10.1;
		if option client-system-arch = 00:07 or option client-system-arch = 00:09 {
       			filename "BOOTX64.efi";
		} else {
       			filename "pxelinux.0";
		}
	}
}

```

- 将dhcpd服务设置为开机启动

```bash
systemctl restart dhcpd
systemctl enable dhcpd
```

## 配置TFTP服务

TFTP是一种基于UDP协议的简单文件传输协议，所以不需要进行用户认证即可获取所需的文件资源。
在这里，可以为客户端主机提供引导和驱动文件。

当客户端主机有了基本的驱动程序之后，可以通过vsftpd服务进行完整的光盘镜像文件的传输。

- 安装tftp服务器

```bash
yum install tftp-server
```

- 配置文件

```config
# cat /etc/xinetd.d/tftp

# default: off
# description: The tftp server serves files using the trivial file transfer \
#	protocol.  The tftp protocol is often used to boot diskless \
#	workstations, download configuration files to network-aware printers, \
#	and to start the installation process for some operating systems.
service tftp
{
	socket_type		= dgram
	protocol		= udp
	wait			= yes
	user			= root
	server			= /usr/sbin/in.tftpd
	server_args		= -s /tftpboot
	disable			= no
	per_source		= 11
	cps			= 100 2
	flags			= IPv4
}
```

- 服务启动以及开机启动

```shell
systemctl restart xinetd
systemctl enable xinetd
```

- TFTP默认使用UDP协议，端口69，如果开启防火墙的话，那么需要将端口加入防火墙规则中。

```shell
firewall-cmd --permanent --add-port=69/udp
firewall-cmd --reload
```

## 配置SYSLinux服务

SYSLinux是一个提供引导加载的服务程序。与其说是服务程序，不如说我们更需要里面的引导文件。

- 安装

```shell
yum install syslinux
```

- 将`pxelinux.0`拷贝到tftp的目录中

```shell
cp /usr/share/syslinux/pxelinux.0 /tftpboot
```

- 将提前下载准备的iso安装文件挂载

```shell
mkdir /mnt/linux_iso
mount -o loop CentOS-7-x86_64-DVD-1804.iso /mnt/linux_iso
```

- 拷贝安装光盘的以下文件到tftp根目录

```shell
cp /mnt/linux_iso/images/pxeboot/{vmlinuz,initrd.img} /tftpboot
cp /mnt/linux_iso/images/pxeboot/{vesamenu.c32,boot.msg} /tftpboot
```

- 创建`pxelinux.cfg`目录，虽然该目录有后缀，但依然是个目录。然后拷贝安装光盘中的开机选项菜单到此目录中，
并命名为`default`

```shell
mkdir /tftpboot/pxelinux.cfg
cp /mnt/linux_iso/isolinux/isolinux.cfg /tftpboot/pxelinux.cfg/default
```

- 修改default文件，主要是修改默认的选项菜单，以及光盘镜像安装方式的修改，并指定好光盘镜像的获取网址以及KickStart应答文件
的获取路径

```shell
# vi /tftpboot/pxelinux.cfg/default


default linux
# default vesamenu.c32
timeout 600

display boot.msg

# Clear the screen when exiting the menu, instead of leaving the menu displayed.
# For vesamenu, this means the graphical background is still displayed without
# the menu itself for as long as the screen remains in graphics mode.
menu clear
menu background splash.png
menu title CentOS 7
menu vshift 8
menu rows 18
menu margin 8
#menu hidden
menu helpmsgrow 15
menu tabmsgrow 13

# Border Area
menu color border * #00000000 #00000000 none

# Selected item
menu color sel 0 #ffffffff #00000000 none

# Title bar
menu color title 0 #ff7ba3d0 #00000000 none

# Press [Tab] message
menu color tabmsg 0 #ff3a6496 #00000000 none

# Unselected menu item
menu color unsel 0 #84b8ffff #00000000 none

# Selected hotkey
menu color hotsel 0 #84b8ffff #00000000 none

# Unselected hotkey
menu color hotkey 0 #ffffffff #00000000 none

# Help text
menu color help 0 #ffffffff #00000000 none

# A scrollbar of some type? Not sure.
menu color scrollbar 0 #ffffffff #ff355594 none

# Timeout msg
menu color timeout 0 #ffffffff #00000000 none
menu color timeout_msg 0 #ffffffff #00000000 none

# Command prompt text
menu color cmdmark 0 #84b8ffff #00000000 none
menu color cmdline 0 #ffffffff #00000000 none

# Do not display the actual menu unless the user presses a key. All that is displayed is a timeout message.

menu tabmsg Press Tab for full configuration options on menu items.

menu separator # insert an empty line
menu separator # insert an empty line

label linux
  menu label ^Install CentOS 7
  kernel vmlinuz
  append initrd=initrd.img ip=dhcp inst.stage2=ftp://192.168.10.1 ks=ftp://192.168.10.1/pub/ks.cfg quiet
  # append initrd=initrd.img inst.stage2=hd:LABEL=CentOS\x207\x20x86_64 quiet

label check
  menu label Test this ^media & install CentOS 7
  menu default
  kernel vmlinuz
  append initrd=initrd.img inst.stage2=hd:LABEL=CentOS\x207\x20x86_64 rd.live.check quiet

menu separator # insert an empty line

# utilities submenu
menu begin ^Troubleshooting
  menu title Troubleshooting

label vesa
  menu indent count 5
  menu label Install CentOS 7 in ^basic graphics mode
  text help
	Try this option out if you're having trouble installing
	CentOS 7.
  endtext
  kernel vmlinuz
  append initrd=initrd.img inst.stage2=hd:LABEL=CentOS\x207\x20x86_64 xdriver=vesa nomodeset quiet

label rescue
  menu indent count 5
  menu label ^Rescue a CentOS system
  text help
	If the system will not boot, this lets you access files
	and edit config files to try to get it booting again.
  endtext
  kernel vmlinuz
  append initrd=initrd.img inst.stage2=hd:LABEL=CentOS\x207\x20x86_64 rescue quiet

label memtest
  menu label Run a ^memory test
  text help
	If your system is having issues, a problem with your
	system's memory may be the cause. Use this utility to
	see if the memory is working correctly.
  endtext
  kernel memtest

menu separator # insert an empty line

label local
  menu label Boot from ^local drive
  localboot 0xffff

menu separator # insert an empty line
menu separator # insert an empty line

label returntomain
  menu label Return to ^main menu
  menu exit

menu end
```


## 配置vsftpd服务

当前构建的这套系统中，安装光盘镜像是通过ftp协议传输的，所以使用vsftpd服务。
也可以使用httpd等服务来提供web访问的方式，只要能确保客户端主机可以访问到即可。
如果使用其他的方式，一定要注意上文提到的`default`文件中的相应的路径需要修改。

- 安装

```shell
yum install vsftpd

# 配置文件路径(没有什么需要修改，保持默认即可)
/etc/vsftpd/vsftpd.conf
```

- 启动服务及开机自启

```shell
systemctl restart vsftpd
systemctl enable vsftpd
```

- 将安装光盘镜像中的所有文件copy到ftp的目录下

```shell
cp -r /mnt/linux_iso/* /var/ftp
```

- 防火墙设置

```shell
firewall-cmd --permanent --add-service=ftp
firewall-cmd --reload

setsebool -P ftpd_connect_all_unreserved=on
```

> 一定要注意ftp目录的权限问题，我在挂在了新的硬盘后，没有检查文件夹的权限，导致客户端读取不到响应的仓库数据。

## 创建KickStart应答文件

需要注意的是，修改ks文件中的url的部分

```shell
# /var/ftp/pub/ks.cfg

#platform=x86, AMD64, 或 Intel EM64T
#version=DEVEL
# Install OS instead of upgrade
install
# X Window System configuration information
xconfig  --startxonboot
# Keyboard layouts
# old format: keyboard us
# new format:
keyboard --vckeymap=cn --xlayouts='cn'
# Root password
# password: 123456
# rootpw --iscrypted $1$tczqctJg$tQbT39eH4XwhvmKwh9tLz/
rootpw --iscrypted $1$zY8JPQGF$/0SuPWCYhf3QbRQRe/nv90
# System language
lang zh_CN.UTF-8
# user --groups=wheel --name=hileveps --password=$6$NeWwSpeU8PkXgi7r$j8JHJbaBMIdmBwO4cg0P.NLPeqg3K7qvwqXVZKP3Eb8dFeGygDubn1U/7C2JvxLqLoWcb//NDi.GqPEDVMvXC/ --iscrypted --gecos="hileveps"
user --groups=wheel --name=hileveps --password=$6$dGXjqr9RxmqEstqx$S9DYIuHLZaK3pEXYLyNINrj9FS62B2FW83iEOpWDvc3WioZ8jQeUlc.Q1BWjDdeh6cxQqrMQg2ElCKMgXqStw0 --iscrypted --gecos="hileveps"
# System authorization information
auth  --useshadow  --passalgo=sha512
# Use CDROM installation media
# cdrom
url --url=ftp://192.168.10.1
# Use graphical install
graphical
# Run the Setup Agent on first boot
firstboot --enable
# SELinux configuration
selinux --enforcing

# repo
# repo --name=hileveps_repo --baseurl=ftp://192.168.10.1/hileveps_repo
repo --name=epel  --baseurl=ftp://192.168.10.1/hileveps_epel/epel

# System services
services --enabled="chronyd"
ignoredisk --only-use=sda
# Firewall configuration
firewall --disabled
# Network information
network  --bootproto=dhcp --device=enp2s0
network  --bootproto=dhcp --device=None
# Halt after installation
halt
# System timezone
timezone Asia/Shanghai
# System bootloader configuration
bootloader --append=" crashkernel=auto" --location=mbr --boot-drive=sda
autopart --type=lvm
# Partition clearing information
# @^minimal
clearpart --all --initlabel --drives=sda

%packages
@X Window System
@Xfce
@core
chrony
kexec-tools
wqy-zenhei-fonts


%end
```

- KickStart文件也可以通过GUI程序`system-config-kickstart`来生成

- 如何生成rootpw以及user用的password

使用pwkickstart可以生成ks文件中用到密码的部分

```shell
# 安装
yum install pwkickstart
```

使用示例

```shell
[root@localhost pxelinux.cfg]# pwkickstart
Password:

# md5
rootpw --iscrypted $1$rPSD7fnP$p3ln1FwvOcyR5PoFLZXIu.
# sha256
rootpw --iscrypted $5$MjKxxhitee9cowpB$HPjnUIN8Acu/4ePafaXle56q/TVPrTAUII0Fu9z9mV7
# sha512
rootpw --iscrypted $6$rEUb8jcWXctlkZtj$yq3c4xqJGgFQiGm8njV.bXTEdzWXL8vNYWqGkn9jvVlWnMp8QV9/7Ze8MItFgv84lfrj0/LJARmBCrbM7ilws/
```

> 注意$1$，$5$，$6$，因为添加用户时的shadow需要和这里的配置对应, ks.cfg中的`auth  --useshadow  --passalgo=sha512`

## 自建yum repo仓库

这个需求是因为如果我们需要安装一些下载的iso镜像中没有的软件包的时候，我们可以在ks文件中配置repo，
然后就可以安装额外的包了。

而在这次的实践中，由于我们需要epel仓库来安装xfce桌面，所以我克隆了整个epel的仓库。

- 克隆epel仓库

```shell
reposync -g -l -d -m --repoid=epel --newest-only --download-metadata --download_path=/var/ftp/epel_repo
```

> 特别注意的是，仓库比较占用硬盘资源，最好保证有足够的空间。
我自己实践的过程中，好不容易下载了10几个G的资源，最后硬盘空间不够，只好重新加了硬盘后重新下载。

- 克隆完成后需要重新生成comps(如果不重新生成，客户机安装的时候会报找不到Xfce组)

```shell
cd /var/ftp/epel_repo/epel
createrepo -g comps.xml ./
```

- ks文件中repo的配置

```shell
repo --name=epel  --baseurl=ftp://192.168.10.1/hileveps_epel/epel
```

> 这里的路径配置非常重要,一定要注意name和后面url的路径

## 步骤结束

到这里PXE无人值守的服务器配置就算完成了，之后将客户机和服务器连到一个局域网，
然后从网卡启动，既可以实现自动安装。

需要注意的是，由于我们的服务器提供了dhcp的服务，所以要么使用集线器进行局域网连接，
如果使用路由器的话，需要关闭路由器的dhcp服务（未测试）。

---- 

## 遇到问题及Tips

### 挂载ntfs格式的u盘

由于copy下载的iso镜像文件时，iso文件大小超过4G，而fat32格式的u盘无法copy，所以使用了ntfs格式的u盘。
但是Linux默认不支持挂载ntfs的u盘，所以需要使用ntfs-3g安装包。

```shell
yum install ntfs-3g
mount -t ntfs-3g /dev/sdb4 /mnt
```

### 关闭selinux

安装完ftp服务后，如果selinux没有进行特殊的配置的话，其他客户机设备无法通过ftp获取文件。由于是在公司内部使用，
所以简单的方式就是关闭selinux。

```shell
# 临时关闭selinux
# 获取当前selinux状态
getenforce

# 设置selinux状态为自由
setenforce 0

# 彻底关闭selinux
vim /etc/sysconfig/selinux
# 修改前
SELINUX=enforcing
# 修改后
SELINUX=disabled
```

### 查看tftp等的下载log

```shell
tail -f /var/log/xferlog
```

## 参考资料

- [PXE+Kickstart 全自动安装部署CentOS7.4](https://www.cnblogs.com/cloudos/p/8143929.html)
- 图书《Linux就该这么学》(作者:刘遄)  第19章节:使用PXE+KickStart无人值守安装服务
- [kickstart说明（中文）](https://blog.csdn.net/jiajiren11/article/details/79882254)
- [kickstart说明（中文）2](https://fedoraproject.org/wiki/Anaconda/Kickstart/zh-cn)
- [ KickStart语法介绍（英文） ](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/installation_guide/sect-kickstart-syntax)
- [ On generating kickstart passwords ](https://lukas.zapletalovi.com/2018/02/on-generating-kickstart-passwords.html)
- [ 用yum将安装包及其依赖包下载到本地的方法 ](https://blog.csdn.net/GX_1_11_real/article/details/80694556)
- [ CentOS 7 PXE+Kickstart+TFTP+VSFTP+BIOS+UEFI ](https://www.cnblogs.com/IMxY/p/8955411.html)
- [ 基于linux下的rpm仓库搭建以及kickstart ]( https://blog.csdn.net/aaaaaab_/article/details/80166206 )
- [ PXE+kickstart无人值守安装CentOS 6 ]( https://www.cnblogs.com/f-ck-need-u/p/6442024.html )
- [ How to Setup Local HTTP Yum Repository on CentOS 7 ]( https://www.tecmint.com/setup-local-http-yum-repository-on-centos-7/ )

( The End )
