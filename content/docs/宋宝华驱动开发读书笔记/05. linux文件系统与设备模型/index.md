---
title: 05. linux文件系统与设备模型
---

# 文件系统与设备驱动之间的关系
![Pasted image 20251106114222](assets/Pasted%20image%2020251106114222.png)

- VFS与文件系统之间的接口是file_operations结构体
- 对于块设备驱动，上层有文件系统抽象层，块设备驱动看不到file_operations。块设备可以通过文件系统的接口进行文件访问，file_operations的实现在文件系统内。也可以直接通过设备文件访问裸设备(例如`dd if=/dev/sdb1 of=sdb1.img`)，此时通过内核实现的def_blk_fops（fs/block_dev.c)这个file_operations结构进行访问，不走文件系统。
- 对于字符设备，由于没有文件系统的抽象，所以file_operations接口需要由设备驱动提供。

# file结构体和inode结构体
file结构体代表一个打开的文件，系统中每个打开的文件在内核空间都有一个关联的struct file。它由内核在打开文件时创建，并传递给在文件上进行操作的任何函数。在文件的所有实例都关闭后，内核释放这个数据结构。在内核和驱动源代码中，struct file的指针通常被命名为file或filp（即file pointer）。

VFS inode结构体包含文件访问权限、属主、组、大小、生成时间、访问时间、最后修改时间等信息，对于设备文件，内部包含设备结构体指针，设备号，及设备文件的其他属性信息。它是Linux管理文件系统的最基本单位，也是文件系统连接任何子目录、文件的桥梁。

# sysfs文件系统与linux设备模型
linux2.6引入了sysfs虚拟文件系统，sysfs把连接在系统上的设备和总线组织成为一个分级的文件系统，它们可以由用户空间存取，向用户空间导出内核数据结构以及它们的属性，展示设备驱动模型中各组件的层次关系。
```sh
/sys/bus/     # 总线驱动目录,内部的设备目录链接到/sys/devices/中
/sys/device/  # 所有设备目录
/sys/class/   # 具有共同特性的设备，内部的设备目录链接到/sys/device/中
```

在linux内核中，分别使用bus_type、device_driver和device来描述总线、驱动和设备，这3个结构体定义于include/linux/device.h中。device_driver和device通常被包含在具体的驱动和设备结构体中作为其成员。

device_driver和device分别表示驱动和设备，而这两者都必须依附于一种总线，因此都包含struct bus_type指针。在Linux内核中，设备和驱动是分开注册的，通过bus_type的match回调函数将两者匹配在一起，一旦匹配成功，device_driver中的probe回调函数就被执行。

总线、驱动和设备最终都会落实为sysfs中的1个目录，它们可看做是kobject的派生类，kobject可看作是所有总线、设备和驱动的抽象基类。1个kobject对应sysfs中的1个目录，总线、设备和驱动中的各个attribute(attribute、bus_attribute、driver_attribute和device_attribute)则直接落实为sysfs中的1个文件，attribute会伴随着show和store这两个回调函数，分别用于读写该attribute对应的sysfs文件。
# devfs与udev
devfs用于设备驱动程序自主管理设备文件，文件名称由驱动程序指定，应用层无法自由命名。由linux2.4内核引入，于linux2.6内核中被删除，现由应用层的udev取代它。

udev工作在应用层，实现了设备文件管理机制与策略的分离。udev依靠接收内核发送的热插拔事件工作，热插拔时设备信息由内核通过netlink套接字发送uevent事件到应用层，udev接收uevent并根据设备信息匹配策略，在/dev下创建对应的设备文件。对于非热插拔设备，在sysfs文件系统下的设备文件夹中都有一个uevent属性文件(例如`/sys/module/psmouse/uevent`)，向此文件中写入`add`可令内核重新发送uevent信息。
```sh
# 查看实时捕获的uevent信息
udevadm monitor --kernel --udev --property
# 查看具体设备的可用uevent信息
udevadm info -a -p /sys/devices/platform/serial8250/tty/ttyS0
# 查看/dev下的设备文件对应/sys/下的哪个设备
udevadm info -q path -n /dev/<NAME>
```

udev规则在/etc/udev/rules.d/目录下配置，分为匹配部分和赋值部分，匹配关键字包括：ACTION（行为）、KERNEL（匹配内核设备名）、BUS（匹配总线类型）、SUBSYSTEM（匹配子系统名）、ATTR（属性）等，赋值关键字包括：NAME（创建的设备文件名）、SYMLINK（符号创建链接名）、OWNER（设置设备的所有者）、GROUP（设置设备的组）、IMPORT（调用外部程序）、MODE（节点访问权限）等。如下规则根据匹配部分创建/dev/eth1设备文件：
```sh
SUBSYSTEM=="net", ACTION=="add", DRIVERS==" *", ATTR{address}=="08:00:27:35:be:ff", ATTR{dev_id}=="0x0", ATTR{type}=="1", KERNEL=="eth*", NAME="eth1"
```