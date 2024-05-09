---
layout: post
title: 使用busybox构建最小rootfs
date: 2021-08-10 12:09:47
tags:
  - qemu
  - rootfs
toc: true
---

## 前言
在进行linux kernel漏洞分析过程中需要对kernel进行调试，虽然可借助virtualbox\vmware等虚拟机软件运行某个发行版，但对发行版更换kernel并不方便，某些发行版并不提供debuginfo或带有kgdb补丁的调试版kernel，且存在发行版指定libc库和不同版本kernel的兼容问题。

使用qemu调试可以灵活更换kernel版本及rootfs，且qemu自带的gdb stub可以调试无kgdb补丁的kernel二进制文件，使用`Ramdisk`可以避免kernel对不同文件系统的兼容性问题，具有极高的灵活性。本文主要构建一个可以兼容多种kernel版本的最小rootfs。

## 环境准备
CentOS: 7.9  
glibc: 2.17  
gcc: 4.8.5  
busybox: 1.32.1  

在低版本kernel上运行高版glibc会提示`KERNEL TOO OLD`错误，选择一个合适版本的glibc可以提高对不同kernel版本的兼容性，免去多次制作rootfs的麻烦。经过测试发现2.17版本的glibc对2.6.32～5.10版本的kernel具有良好的兼容性，

## 编译busybox
安装编译套件及依赖

```text
# yum groupinstall "Development Tools"
# yum install ncurses-devel
# yum install hmaccalc zlib-devel binutils-devel elfutils-libelf-devel
# yum install glibc_static
```
配置及编译busybox

```text
$ make defconfig
$ make menuconfig
  /* select Busybox Settings
     then General Configuration
       then exec prefers applets
     then Build Options
       then Build Busybox as a static binary
     exit back to top level menu
     select Linux System Utilities
       then mdev
     select Shells
       then Standalone shell
     exit back to top level menu
   exit and save new configuration
  */
$ make
$ make install
```
`make defconfig`配置选项提供了最大通用兼容性。  
`standalone shell mode`使busybox的内置命令shell，直接运行busybox的内部小程序，而不查找该名称的外部程序。  
`Build Busybox as a static binary`由于只需要busybox提供的功能，因此在此处使用静态链接库，省去设置glibc环境的麻烦。  
`make install`会将编译好的程序安装到源码根目录的`_install`文件夹内。  

## 配置ROOTFS
创建目录结构

```text
$ cd _install
$ mkdir -pv {bin,sbin,etc/init.d,proc,sys,usr/{bin,sbin},dev,tmp}
```
创建`inittab`，init进程是kernel加载完成后启动的第一个进程，其配置文件`/etc/inittab`指定在不同的情况下执行不同的应用程序。busybox inittab文件格式参考[官方说明](https://github.com/brgl/busybox/blob/master/examples/inittab)

```text
$ cat > etc/inittab << "EOF"
::sysinit:/etc/init.d/rcS
::askfirst:-/bin/ash
::ctrlaltdel:/sbin/reboot
::shutdown:/sbin/swapoff -a
::shutdown:/bin/umount -a -r
::restart:/sbin/init
EOF
```
创建rcS文件并赋予可执行权限,在rcS文件中挂载设备并创建`console`和`null`设备文件，然后运行mdev动态生成设备节点。  
`mount -a`负责挂载`/etc/fstab`文件中指定的挂载选项  

```text
$ cat > etc/init.d/rcS << "EOF"
#!/bin/sh

mount -a
mknod console c 5 1
mknod null c 1 3
mkdir /dev/pts
mount -t devpts devpts /dev/pts
echo /sbin/mdev > /proc/sys/kernel/hotplug
mdev -s
EOF
$ chmod 744 /etc/init.d/rcS
```
创建`/etc/fstab`文件，写入挂载配置选项

```text
$ cat > etc/fstab << "EOF"
#device		mount-point		type	options		dump	fsck	order
proc		/proc			proc	defaults	0		0
tmpfs		/tmp			tmpfs	defaults	0		0
sysfs		/sys			sysfs	defaults	0		0
tmpfs		/dev			tmpfs	defaults	0		0
EOF
```
创建`/etc/passwd`和`/etc/group`文件，添加一个`test`普通用户以进行测试

```text
$ mkdir -pv {root,home/test}
$ cat > etc/passwd << "EOF"
root:x:0:0:root:/root:/bin/sh
bin:x:1:1:bin:/dev/null:/bin/false
daemon:x:6:6:Daemon User:/dev/null:/bin/false
messagebus:x:18:18:D-Bus Message Daemon User:/run/dbus:/bin/false
uuidd:x:80:80:UUID Generation Daemon User:/dev/null:/bin/false
nobody:x:99:99:Unprivileged User:/dev/null:/bin/false
test:x:1000:1000:test:/home/test:/bin/sh
EOF
$ cat > etc/group << "EOF"
root:x:0:
bin:x:1:daemon
sys:x:2:
tty:x:5:
daemon:x:6:
floppy:x:7:
disk:x:8:
lp:x:9:
dialout:x:10:
audio:x:11:
video:x:12:
utmp:x:13:
usb:x:14:
cdrom:x:15:
adm:x:16:
messagebus:x:18:
input:x:24:
kvm:x:61:
uuidd:x:80:
wheel:x:97:
nogroup:x:99:
users:x:999:
test:x:1000:
EOF
```
完成以上文件创建之后，一个带有init系统的最小文件系统就完成了。

## 打包测试
创建完rootfs以后需要打包成initrd Ram disk可以加载的格式

```text
$ cd _install
$ find . | cpio -o -H newc > ../min_rootfs.cpio
```
创建qemu启动脚本，在参数中指定`initrd`参数和kernel启动参数`rdinit`

```text
$ cat > start.sh << "EOF"
#!/bin/sh
stty intr ^]
qemu-system-x86_64 \
-m 100M \
-kernel ./kernel_2.6.32/bzImage \
-initrd ./min_rootfs.cpio \
-append "root=/dev/ram rw rdinit=/sbin/init console=ttyS0 loglevel=3 oops=panic" \
-smp cores=1,threads=2 \
-cpu kvm64,+smep,+smap \
-net user -net nic \
-device e1000 \
-nographic \
-s \
EOF
```
进入qemu系统以后，可以使用halt命令关闭系统，然后键入`Ctrl+a x`通过qemu monitor退出qemu。
