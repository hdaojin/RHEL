## 一、实现过程  

客户端通过网卡PXE启动 --> 连接到DHCP服务器 --> 获得IP地址 --> 客户端从TFTP服务器下载pxelinux.0,根据配置文件（default）下载指定的vmlinuz,initrd --> 启动系统内核，加载初始化内存文件系统（根据加载内核参数是否有ks, 来决定自(手)动安装）--> 初始化完成,启动安装程序 （images/install.img）--> 到指定的位置（NFS｜FTP｜HTTP服务器上）下载软件包进行安装 --> 如果有ks_pre脚本, 则执行安装前的脚本 --> 安装过程 --> 如果有ks_post脚本, 执行安装后的脚本 --> 安装结束 --> 重启系统。



## 二、配置步骤

1. 最小化安装配置RHEL8系统;
2. 安装配置DHCP；
3. 安装配置TFTP;
4. 安装配置FTP或NFS或HTTP（ubuntu/Debian的安装源必须放在HTTP服务器下）；
5. 要实现自动安装，需配置KickStart； （可选）
6. 要实现自动分配主机名，需要配置DNS (或使用ks脚本)。（可选）



## 三、具体实现

### 第一步：配置服务器静态IP。

---

RHEL8默认使用NetworkManager管理网络连接，这是一个Gnome环境的网络管理工具。最小化安装的系统并不会安装NetworkManager服务程序，所以在命令行中对ifcfg-eth0做如下修改，并重启网络服务。

```bash
[root@365linux ~]# vim /etc/sysconfig/network-scripts/ifcfg-eth0
DEVICE="eth0"
HWADDR="00:0C:29:AD:FE:AC"
NM_CONTROLLED="no"
ONBOOT="yes"
BOOTPROTO="static"
IPADDR="192.168.122.200"
NETMASK="255.255.255.0"
GATEWAY="192.168.122.254"

[root@365linux ~]# service network restart
```



### 第二歩：安装所需的服务和软件包，通过DNF、YUM和RPM的安装方式都可以，这里以DNF为例。

---

1. 安装dhcp服务器端
2. 安装tftp服务器
3. 安装ftp服务器
4. 安装syslinux (可选, Simple kernel loader which boots from fat，pxe,iso,ext)
5. 安装网络引导目录相关文件system-config-netboot（可选，安装后会自动建立网络安装所需的引导文件。经测试RHEL6包括CentOS6已经移除了system-config-netboot软件包，红帽建议使用Cobbler和Red Hat Satellite，当然后者是要付费的。）



安装使用的命令如下：

```bash
[root@365linux ~]# dnf install -y dhcp-server
[root@365linux ~]# dnf install -y tftp-server
[root@365linux ~]# dnf install -y vsftpd    
[root@365linux ~]# dnf install -y syslinux
```



### 第三歩：配置相关的服务

---

- DHCP的配置

```bash
[root@365linux ~]# cat /usr/share/doc/dhcp-server/dhcpd.conf.example >>/etc/dhcp/dhcpd.conf
[root@365linux ~]# vim /etc/dhcp/dhcpd.conf
ddns-update-style none;
option domain-name "365linux.com";     //搜索域；
option domain-name-servers 192.168.122.200, 202.96.128.86;   //指定DNS服务器地址；
default-lease-time 6000;              //租约时间；
max-lease-time 72000;
log-facility local7;             //日志方式；
allow booting;
allow bootp;

subnet 192.168.122.0 netmask 255.255.255.0 {     //网段，要与dhcpd监听的网卡处在同一网段；
  range  192.168.122.1 192.168.122.253;    //分配IP地址范围；
  option routers 192.168.122.254;     //指定客户端路由；
  next-server 192.168.122.200;       //网络引导服务器的IP。
  filename "pxelinux.0"; //pxe启动引导文件，放置在tftp的根目录下，使用相对路径；
}
```

> 或者写在group里面单独申明：
> \#group {
> \#        next-server 192.168.122.200;
> \#        host tftpclient {
> \#        filename "pxelinux.0";
> \#        }
> \#}

- TFTP的配置

```bash
[root@365linux ~]# systemctl enable --now tftp.service
```

- vsftpd的配置

使用默认的匿名访问权限即可。

```bash
anonymous_enable=YES
```



### 第四歩：创建tftp目录下引导文件

---

在RHEL5的版本中，如果安装了system-config-netboot，那么在/tftpboot的目录下自动生成一个linux-install文件夹，并且将所有的引导文件统一放到这个目录下面，以便需要提供多个系统安装时方便管理。



在RHEL8版本里没有该软件包，我们只能手动一步步建立需要的目录跟文件了。

创建PXE工作的根目录，在DHCP服务器中定义过，保持目录名一致。

```bash
[root@365linux ~]# cd /var/lib/tftpboot/
```



复制PXE网络引导程序到工作目录。该程序有syslinux软件包提供，理论上光盘中的isolinux/isolinux.bin也可以。

```bash
[root@365linux ~]# cp /usr/share/syslinux/pxelinux.0 /var/lib/tftpboot/
```



挂载光盘(或者iso镜像文件)到系统目录，我们需要几个系统安装引导的文件。

```bash
[root@365linux ~]# mount -o loop /dev/cdrom /mnt
```



从光盘中复制启动引导配置菜单到工作目录的下的pxelinux.cfg目录下。稍后根据要引导安装的系统修改菜单的内容。

```bash
[root@365linux ~]# mkdir /var/lib/tftpboot/pxelinux.cfg
[root@365linux ~]# cp /mnt/isolinux/isolinux.cfg /var/lib/tftpboot/pxelinux.cfg/default
```



一个字符终端界面的背景图，可以写一些提示上去。5版本主要靠这个文件实现都系统引导的提示。rhel8的系统可以不需要。

```bash
[root@365linux ~]# cp /mnt/isolinux/boot.msg /var/lib/tftpboot/
```



如果你想使用RHEL8版本支持的的图形启动菜单的话，需要这些文件。

```bash
[root@365linux ~]# cp /mnt/isolinux/vesamenu.c32  /var/lib/tftpboot/
[root@365linux ~]# cp /mnt/isolinux/ldlinux.c32  /var/lib/tftpboot/
[root@365linux ~]# cp /mnt/isolinux/libcom32.c32  /var/lib/tftpboot/
[root@365linux ~]# cp /mnt/isolinux/libutil.c32  /var/lib/tftpboot/
```



为启动菜单创建一个背景图，我这里是定制的，大小为640x480像素的jpg图片，你可以使用安装光盘中自带的（或者syslinux提供的 cp  /usr/share/doc/syslinux/sample/syslinux_splash.jpg     /var/lib/tftpboot/）。

```bash
[root@365linux ~]# cp /root/365linux_splash.jpg /var/lib/tftpboot/
```



### 第五歩：提取准备安装的系统的安装引导内核文件

---

从RHEL6的光盘中提取PXE专用内核vmlinuz和初始化内存磁盘镜像initrd.img到安装工作目录下的RHEL8子目录。 

```
[root@365linux ~]# mkdir /var/lib/tftpboot/rhel8
[root@365linux ~]# umount /mnt
[root@365linux ~]# mount -o loop /systemiso/rhel8/rhel-server-8.0-i386-dvd.iso  /mnt
(如果有cdrom，从cdrom挂载目录中复制)
[root@365linux mnt]# cd /mnt/images/pxeboot/
[root@365linux isolinux]# cp vmlinuz initrd.img /var/lib/tftpboot/rhel8/
```

 

### 第六步：针对性的修改default文件，形成菜单。

---

```bash
[root@365linux ~]# vim /var/lib/tftpboot/pxelinux.cfg/default

default vesamenu.c32    //默认使用图形菜单，注意这个文件的相对路径；
\#prompt 1   //不使用图形菜单，而是用boot.msg定义的菜单时，要启用该项；
timeout 600 //等待时间60秒

\#display boot.msg  //文本背景模式，我注释掉了；

menu background 365linux_splash.jpg  //图形背景模式；
menu title Welcome to 365linux installation system!  //菜单标题；
menu color border 0 #ffffffff #00000000  //颜色定义；
menu color sel 7 #ffffffff #ff000000
menu color title 0 #ffffffff #00000000
menu color tabmsg 0 #ffffffff #00000000
menu color unsel 0 #ffffffff #00000000
menu color hotsel 0 #ff000000 #ffffffff
menu color hotkey 7 #ffffffff #ff000000
menu color scrollbar 0 #ffffffff #00000000

label rhel8  //菜单选项，这里有两个，安装RHEL8和从本地启动。
  menu label ^Install or upgrade RHEL8 X86
  kernel rhel8/vmlinuz   //注意相对路径;
  append initrd=rhel8/initrd.img inst.repo=ftp://192.168.122.200/source/rhel8 quiet
label local
  menu label Boot from ^local drive
  menu default    //60秒后默认加载的选项。
  localboot 0xffff   //启动本地系统。
```

>还可以加上其他系统版本的启动，或救援模式，兼容视频驱动模式，内存测试等。参考光盘原文件。



### 第七步：把客户机要安装的系统的光盘镜像解开后复制到ftp的目录。

---

```bash
[root@365linux ~]# mount -o loop /systemiso/rhel6/rhel-server-6.0-i386-dvd.iso  /mnt
如果之前已经挂载了则不用挂载。(如果有cdrom，从cdrom挂载目录中复制)
[root@365linux ~]# mkdir -p /var/ftp/source/rhel8
[root@365linux ~]# cp -R　/mnt/*   /var/ftp/source/rhel8
```



### 第八步：启动所有服务

---

```bash
[root@365linux ~]# systemctl restart dhcpd
[root@365linux ~]# systemctl restart tftp.socket
[root@365linux ~]# systemctl restart vsftpd

[root@365linux ~]# chkconfig dhcpd on
[root@365linux ~]# chkconfig tftp on
[root@365linux ~]# chkconfig vsftpd on
```



### 第九步：防火墙策略

```bash
[root@pxe-server ~]# iptables -I INPUT  -m state  --state NEW  -p udp --dport 67  -j ACCEPT
[root@pxe-server ~]# iptables -I INPUT  -m state  --state NEW  -p udp --dport 69  -j ACCEPT
[root@pxe-server ~]# iptables -I INPUT  -m state  --state NEW  -p tcp --dport 21  -j ACCEPT
[root@pxe-server ~]# modprobe   nf_conntrack_ftp
[root@pxe-server ~]# modprobe   nf_nat_ftp
[root@pxe-server ~]# vim  /etc/sysconfig/iptables-config
IPTABLES_MODULES="nf_conntrack_ftp nf_nat_ftp"
[root@pxe-server ~]# service iptables save

PS: 如果不进行相关的安全设置，可简单的关闭iptables和selinux。
PS: 如果在kvm虚拟机（并使用nat网络）中搭建pxe服务器测试， 客户机（同一虚拟网段的虚拟机）有可能先从虚拟机服务libvirt提供的dhcp获取IP， 而不是从你的pxe服务器获得。可以设置在物理机上的防火墙规则来屏蔽libvirt的dhcp。
[root@zhenji ~]# iptables -I INPUT -p udp --dport 67 -j REJECT --reject-with icmp-host-unreachable
```



## 四、客户机网络引导安装RHEL8

图一：客户端机器选择从网络启动。

图二：选择菜单的界面，当然我的背景是我为公司定制过的。

图三: 内核引导过程。

图四：安装方式选择URL。

图五：填写FTP服务器的URL（IP地址和镜像解压目录）,比如本例中是：ftp://192.168.122.200/source/rhel8/。

图六：开始安装了……



> fedora EPEL 的项目中有一个cobbler项目，用于创建kickstart的自动安装，红帽推荐。