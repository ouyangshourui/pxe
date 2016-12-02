# pxe 安装centos7.2 制作过程（hp x86 server）
 
       本文档是为了使用pxe 在hp x86 server上面安装 centos7.2，hp server 需要uefi pxe安装方式，本文需要做作efi引导文件。下面是具体步骤：
## 1、关闭selinux

```
setenforce 0
sed -i '/^SELINUX=/ s/=.*/=disabled/ ' /etc/selinux/config
```

## 2、安装GRUB2+DHCP+TFTP+HTTP
```
yum install grub2-efi-modules tftp-server xinetd dhcp dhcp-devel dhcp-common httpd vsftpd
```

## 3. 配置tftp服务
```
sed -i '/disable/ s/yes/no/' /etc/xinetd.d/tftp
```

## 4. 配置DHCP
```
#
# DHCP Server Configuration file.
# see /usr/share/doc/dhcp*/dhcpd.conf.example
# see dhcpd.conf(5) man page
ddns-update-style interim;
allow booting;
allow bootp;
ignore client-updates;
set vendorclass = option vendor-class-identifier;
option architecture-type code 93 = unsigned integer 16;
option pxe-system-type code 93 = unsigned integer 16;
subnet 10.214.128.0 netmask 255.255.255.0 {
range dynamic-bootp 10.214.128.10 10.214.128.50; # 分配地址范围
option routers 10.214.128.1; #路由选择
option subnet-mask 255.255.255.0; 
default-lease-time 21600; 
max-lease-time 43200;
next-server 10.214.128.49; #dhcp server ip 地址
filename "bootx64.efi"; #安装引导文件，步骤7会说明制作过程

#网卡mac 和ip地址绑定
host 48{
hardware ethernet 14:02:ec:73:fc:70; 
fixed-address 10.214.128.48;
}
 ```
后续如果修改该文件，需要重启dhcp服务。
## 5. 挂载ISO
```
mkdir -p /var/ftp/pub
mount /opt/CentOS-7-x86_64-Everything-1511.iso /var/ftp/pub -o loop
``` 
## 6．配置ftp
```
cp /var/ftp/pub/EFI/BOOT/grub.cfg  /var/lib/tftpboot/
cp /var/ftp/pub/images/pxeboot/{vmlinuz,initrd.img}  /var/lib/tftpboot/
```

## 7. 配置pxe启动引导文件
```
yum install -y grub2-efi-modules 
制作efi引导文件
 cd /var/lib/tftpboot/
  grub2-mkstandalone -d /usr/lib/grub/x86_64-efi/ -O x86_64-efi --modules="tftp net efinet linux part_gpt efifwsetup" -o bootx64.efi
  ```

## 8、制作kickstart 文件

 
      如果需要安装的系统的版本(特别是跨版本)之前没有被安装过，最好先用ISO安装一遍或者使用kickstart 图形界面制作kickstart文件；因为跨版本的kickstart的配置会有很大的变动。安装完成之后的" /root/anaconda-ks.cfg"文件可以作为kickstart的模板，然后在它的基础上修改成符合网络安装的kickstart模板。
修改kickstart模板文件如下：
如果有如下行，删除它:
```
# Use CDROM installation media
cdrom
# Use graphical install
graphical
# X Window System configuration information
xconfig  --startxonboot
```
下面是模版ks.conf 模版
```
#platform=x86, AMD64, or Intel EM64T
#version=DEVEL
# Install OS instead of upgrade
install
# Keyboard layouts
keyboard 'us'
# System timezone
timezone Africa/Abidjan
# Use network installation
url --url="ftp://10.214.128.49/pub"
# Root password
rootpw --plaintext 123456
# System language
lang en_US
# Firewall configuration
firewall --disabled
# System authorization information
auth --useshadow --passalgo=sha512
firstboot --disable
# SELinux configuration
selinux --disabled
# Network information 选择需要启动的网卡
network --bootproto=dhcp --device=eno49 --onboot=on --ipv6=auto
network --hostname=localhost.localdomain
# Reboot after installation
reboot
# System bootloader configuration
bootloader --location=mbr
# Clear the Master Boot Record
zerombr
# Partition clearing information
clearpart --all --initlabel
# Disk partitioning information
#part / --fstype="ext4" --size=204800
#/boot/efi必须是efi 格式，用户根据自己的需要定制分区。
part /boot/efi --fstype=efi --size=200 --fsoptions="defaults,uid=0,gid=0,umask=0077,shortname=winnt"
part /boot --fstype=ext4 --size=500
part swap --recommended
part pv.01 --grow --size=200
volgroup wdvg pv.01
logvol / --fstype=ext4 --vgname=wdvg --grow --size=20480 --name=root
%packages
@base
%end
```
将该文件拷贝到/var/ftp
```
cp ks.conf  /var/ftp/88.ks
```
## 9. 配置grub.conf使之能网络安装
grub.cfg模版
```
set default="1"
function load_video {
insmod efi_gop
insmod efi_uga
insmod video_bochs
insmod video_cirrus
insmod all_video
}
load_video
set gfxpayload=keep
insmod gzio
insmod part_gpt
insmod ext2
set timeout=60
### END /etc/grub.d/00_header ###
search --no-floppy --set=root -l 'CentOS 7 x86_64'
### BEGIN /etc/grub.d/10_linux ###
menuentry 'Install CentOS 7.2 wanda' --class fedora --class gnu-linux --class gnu --class os {
linuxefi (tftp)/vmlinuz bootdev=bootif inst.lang=en_US inst.text inst.ks=ftp://10.214.128.49/88.ks inst.repo=ftp://10.214.128.49/pub ip=dhcp
initrdefi (tftp)/initrd.img
}
```

## 10. 启动服务
```
systemctl enable httpd.service
systemctl enable xinetd.service
systemctl enable dhcpd.service
systemctl restart httpd.service
systemctl restart xinetd.service
systemctl restart dhcpd.service
systemctl disable firewalld.service
systemctl stop firewalld.service
```
## 11、批量安装hp服务器
 
进入ilo   －> 发送F12 进入 网络安装模式 －> 选择需要安装的系统。
 
 
 
参看：
https://github.com/openSUSE/kiwi/wiki/Setup-PXE-boot-with-EFI-using-grub2
https://www.ibm.com/developerworks/community/blogs/a2674a1d-a968-4f17-998f-b8b38497c9f7?sortby=0&maxresults=10&lang=zh
CentOS7 kickstart安装官方文档：https://access.redhat.com/documentation/zh-CN/Red_Hat_Enterprise_Linux/7/html/Installation_Guide/sect-kickstart-syntax.html#sect-kickstart-commands
GRUB2配置文件: http://www.jinbuguo.com/linux/grub.cfg.html
通过UEFI启动pxe安装CentOS 7 : http://blog.chinaunix.net/uid-22621471-id-4980582.html

