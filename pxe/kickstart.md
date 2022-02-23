# 02 KICKSTART

在PXE（或其他安装方式）中，可以使用ks.cfg的安装配置文件实现系统的全自动安装。
ks.cfg文件中配置记录的安装Linux系统需要用户交互配置的安装选项。安装程序预加载该文件，并依据该文件来进行安装。

## 一、使用流程：

1. 创建ks.cfg配置文件；
2. 将ks.cfg文件放置到客户机可读取（下载）的位置；
3. 在pxelinux.cfg/default引导配置文件中通过内核参数加载ks.cfg；
4. ks.cfg将指导安装程序自动安装。



## 二、具体实现：

### 1. 创建ks.cfg配置文件（）

#### 方式一:

>  注意！在rhel8中已不支持此方式创建ks.cfg配置文件！！直接采用`方式2`.

##### 1.1. 在一个有图形界面的系统上安装system-config-kickstart的软件

```bash
[root@365linux ~]# yum install system-config-kickstart
```



##### 1.2. 通过系统菜单-->系统工具-->kickstart

在图形界面下操作，以下只列出了需要更改的选项：

- 选时区
- 使用utc时钟
- 给根口令加密
- 安装后重新引导系统
- 在文本模式中执行安装
- 执行新安装
- FTP 192.168.122.200  /rhel8.5_x64
- 清除主引导记录
- 初始化磁盘标签
- 布局：
  /boot  ext4  500
  交换	使用推荐的交换区大小
  /        ext4   使用磁盘上全部未用空间
- 添加网络设备  eth0 DHCP
- 防火墙  启用  ssh
- 取消安装图形环境
- 软件包选择：基本系统 －－> 基本

完成后：文件 保存 ks.cfg



##### 1.3. 生成的文件如下：

```bash
[demo@365linux ~]$ vim ks.cfg 
#platform=x86, AMD64, 或 Intel EM64T
#version=DEVEL

# Firewall configuration
firewall --enabled --ssh

# Install OS instead of upgrade
install

# Use network installation
url --url="ftp://192.168.122.200/rhel8.5_x64"

# Root password
rootpw --iscrypted $1$tKomAMfM$J4Nn8qsaJ.wrv.Ftp7BG40

# System authorization information
auth  --useshadow  --passalgo=sha512

# Use text mode install
text

# System keyboard
keyboard us

# System language
lang en_US.UTF-8

# SELinux configuration
selinux --enforcing

# Do not configure the X Window System
skipx

# Installation logging level
logging --level=info

# Reboot after installation
reboot

# System timezone
timezone --isUtc Asia/Shanghai

# Network information
network  --bootproto=dhcp --device=eth0 --onboot=on

# System bootloader configuration
bootloader --location=mbr

# Clear the Master Boot Record
zerombr

# Partition clearing information
clearpart --all --initlabel

# Disk partitioning information
part /boot --fstype="ext4" --size=500
part swap --fstype="swap" --recommended
part / --fstype="ext4" --grow --size=1

%packages
@base
%end
```



#### 方式二:

```bash
cp anaconda-ks.cfg /var/ftp/ks_rhel8.5_x64/mini_ks.cfg
[root@pxe pxelinux.cfg]# chmod 644 /var/ftp/ks_rhel8.5_x64/mini_ks.cfg

[root@pxe ~]# vim /var/ftp/ks_rhel8.5_x64/mini_ks.cfg
#version=DEVEL
install
url --url="ftp://192.168.122.200/rhel8.5_x64"
lang en_US.UTF-8
keyboard us
network --onboot yes --device eth0 --bootproto dhcp --noipv6
rootpw  --iscrypted $6$rWAc0Ovhonazi.59$EqOvRrHbPH8dljgsaHDMCMnC7uZfC6.uEP4bUt/JSlMh.34J41/tSs07ukUzRq1yNS30vt.gFg6doN.Tvu6sX0
firewall --service=ssh
authconfig --enableshadow --passalgo=sha512
selinux --enforcing
timezone --utc Asia/Shanghai
reboot
bootloader --location=mbr --driveorder=vda --append="crashkernel=auto rhgb quiet"

# The following is the partition information you requested
# Note that any partitions you deleted are not expressed
# here so unless you clear all partitions first, this is
# not guaranteed to work

zerombr
clearpart --all --drives=vda --initlabel

part /boot --fstype=ext4  --size=500
part  pv.01 --grow --size=1
volgroup vg01 pv.01
logvol  swap --name=lv_swap  --vgname=vg01 --grow --size=1024 --maxsize=2048
logvol  /   --fstype=ext4  --name=lv_root --vgname=vg01 --grow --size=1 

%packages --nobase
@core
%end
```



### 2. 将ks.cfg文件放置到客户机可读取（下载）的位置

```bash
[demo@365linux ~]$ scp ks.cfg root@192.168.122.200:/var/ftp/ks_rhel8.5_x64
[root@localhost ~]# ll /var/ftp/ks_rhel8.5_x64/
总用量 4
-rw-r--r--. 1 root root 1075 12月 12 00:03 ks.cfg
```



### 3. 在pxelinux.cfg/default引导配置文件中通过内核参数加载ks.cfg

```bash
[root@localhost ~]# vim /var/lib/tftpboot/linux-install/pxelinux.cfg/default 
# 在合适的位置添加自动安装的菜单：
label linux auto
  menu label ^Auto Install rhel6.5_x64 system
  kernel rhel6.5_x64/vmlinuz
  append initrd=rhel6.5_x64/initrd.img inst.ks=ftp://192.168.122.200/ks_rhel6.5_x64/ks.cfg inst.repo=ftp://192.168.122.200/source/rhel8
```



### 4. 找客户机测试， ks.cfg将指导安装程序自动安装

  略……



## 三、其他

\-  -   - －－－－－－－－－－－－－分区推荐的写法：－－－－－－－－－－－－－－－－－

```bash
part /boot --fstype=ext4 --size=500
part pv.253002 --grow --size=1
volgroup VolGroup --pesize=4096 pv.253002
logvol swap  --fstype=swap  --name=lv_swap --vgname=VolGroup --recommended
logvol / --fstype=ext4 --name=lv_root --vgname=VolGroup --grow --size=1
```

－－－－－－－－－－－－－－－－－－－－－－－－－－－

```bash
[root@pxe ~]# vim /var/lib/tftpboot/linux-install/pxelinux.cfg/default
label linux auto
  menu label ^Auto Install rhel8.5_x64 system
  kernel rhel8.5_x64/vmlinuz
  append initrd=rhel8.5_x64/initrd.img ks=ftp://192.168.122.200/ks_rhel8.5_x64/mini_ks.cfg
```