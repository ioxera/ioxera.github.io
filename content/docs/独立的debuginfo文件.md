---
layout: post
title: 独立的debuginfo文件
date: 2021-08-24 16:10:33
tags:
  - debufinfo
toc: true
---

## 前言
在使用gcc编译源码时可以使用`-g`选项在二进制文件中保留调试信息，把二进制文件和调试信息合并在同一个文件中带来的一个显著问题是文件或库的SIZE增大，对于不关心调试的普通用户来说，调试信息是不必要的，且在系统安装和使用过程中显著增加了安装时间及磁盘空间占用。

为了解决这个问题，可以把调试信息从二进制文件中剥离出来保存在单独的文件中，某些linux发行版（redhat系列,debian系列等）将这些debuginfo文件打包成独立软件包。

## 生成debuginfo文件
GNU`binutils`工具包包含一个`objcopy`实用程序，可以使用以下命令生成单独的可执行/调试信息文件对，这些命令从`a.out`可执行文件中删除了调试信息，并将其放入`a.out.deubug`文件中。

```text
# objcopy --only-keep-debug a.out a.out.debug
# strip -g a.out
```
在生成debuginfo独立文件以后，可以使用如下方式之一或同时使用，创建二进制文件和debuginfo文件的关联：

 - debug link方式在二进制文件中添加了一个`.gnu_debuglink`节，此节必须包含删除了前导目录的debuginfo文件名称和一个四字节的CRC校验和
 
```text
# objcopy --add-gnu-debuglink=a.out.debug a.out
# objdump -s -j .gnu_debuglink ./a.out      
./a.out：     文件格式 elf64-x86-64
Contents of section .gnu_debuglink:
0000 612e6f75 742e6465 62756700 5f2e5ac3  a.out.debug._.Z.
```
 - 使用`ld--Build ID`或GCC对应的`gcc -Wl，--build-ID`将Build ID嵌入到可执行文件中的`.note.gnu.build-id`节中。然后按照下文的搜索规则将debuginfo文件复制到指定路径下，并按build-id进行重命名

```text
# readelf -n ./a.out
...
Displaying notes found in: .note.gnu.build-id
所有者            Data size   Description
GNU                  0x00000014       NT_GNU_BUILD_ID (unique build ID bitstring)
    Build ID: 5806c3389c3265d80ae315eba818ac8d2dc61a06
...
```

gdb支持两种指定debuginfo文件的方法：

 - 可执行文件包含一个`debug link`来指定debuginfo文件的名称，独立的debuginfo文件的名称通常为`executable.debug`，`debug link`还为debuginfo文件指定了32位CRC校验和，用于验证可执行文件和调试文件是否来自同一构建。
 - 可执行文件包含一个`build ID`，一个唯一的位序列，也存在于相应的调试信息文件中。调试信息文件的名称不是由构建 ID 明确指定的，但可以从构建 ID 计算。

根据提供debuginfo文件的不同方式，gdb具有不同的搜索路径：

 - 对于“debug link”方法，GDB在可执行文件的目录中查找命名文件，然后在名为.debug的目录的子目录中查找，最后在每个全局调试目录下的子目录中查找，该子目录的名称与可执行文件绝对文件名的前导目录相同
 - 对于“build ID”方法，GDB在每个全局调试目录的.build ID子目录中查找名为nn/nnnnnn.debug的文件，其中nn是build ID位字符串的前2个十六进制字符，nnnnnnnn是位字符串的其余部分(实际生成ID字符串为32个或更多十六进制字符）

对于一个具体的二进制文件`a.out`，GDB的搜索路径如下：

 1. \<global debug directory>/.build-id/nn/nnnn...nnnn.debug
 2. \<the path of a.out>/a.out.debug
 3. \<the path of a.out>/.debug/a.out.debug
 4. \<global debug directory>/\<the patch of a.out>/a.out.debug

`<global debug directory>`默认为`/usr/lib/debug/`。可以在gdb中通过`show debug-file-directory`及`set debug-file-directory DIRECTORIES`命令分别查看和设置此目录的值。

## debug文件中的信息
gcc默认会采用[DWARF4](http://dwarfstd.org/doc/DWARF4.pdf)格式来保存调试信息，可以通过`readelf -w a.out.debug`来查看文件中的信息。

```text
# readelf -w ./a.out.debug                
...
.debug_info 节的内容:
  编译单元 @ 偏移 0x0:
   长度：        0xc4 (32-bit)
   版本：        5
   Unit Type:     DW_UT_compile (1)
   缩写偏移量：    0x0
   指针大小：    8
 <0><c>：缩写编号：3 (DW_TAG_compile_unit)
    <d>   DW_AT_producer    : (indirect string, offset: 0x2f): GNU C17 11.1.0 -mtune=generic -march=x86-64 -g -O0
    <11>   DW_AT_language    : 29	(C11)
    <12>   DW_AT_name        : (indirect line string, offset: 0x13): test.c
    <16>   DW_AT_comp_dir    : (indirect line string, offset: 0x0): /home/ioxera/Codes
    <1a>   DW_AT_low_pc      : 0x401795
    <22>   DW_AT_high_pc     : 0xb
    <2a>   DW_AT_stmt_list   : 0x0
 <1><2e>：缩写编号：1 (DW_TAG_base_type)
    <2f>   DW_AT_byte_size   : 8
    <30>   DW_AT_encoding    : 7	(unsigned)
    <31>   DW_AT_name        : (indirect string, offset: 0x0): long unsigned int
 <1><35>：缩写编号：1 (DW_TAG_base_type)
    <36>   DW_AT_byte_size   : 4
    <37>   DW_AT_encoding    : 7	(unsigned)
    <38>   DW_AT_name        : (indirect string, offset: 0x5): unsigned int
 <1><3c>：缩写编号：4 (DW_TAG_pointer_type)
    <3d>   DW_AT_byte_size   : 8
...
```

DWARF内部通过DIE（debugging information entry)形成一颗调用树，DWARF支持C/C++/JAVA等各种语言调试信息的表达且并不依赖ELF文件结构。

## centos中的debuginfo软件包
在centos中使用gdb调试依赖glibc的二进制文件时，需要安装`glibc-debuginfo`软件包，在使用systemtap对内核进行探测之前需要安装`kernel-debuginfo`软件包，否则会报告找不到调试符号的错误信息。

```text
# rpm -ql glibc-debuginfo
/usr/lib/debug
/usr/lib/debug/.build-id
/usr/lib/debug/.build-id/00
/usr/lib/debug/.build-id/00/3018d58fff1bc866f691a02b3d9269be42e5cd
/usr/lib/debug/.build-id/00/3018d58fff1bc866f691a02b3d9269be42e5cd.debug
/usr/lib/debug/.build-id/01
/usr/lib/debug/.build-id/01/07534bb658cedceb7d549d779458ef53178fbe
/usr/lib/debug/.build-id/01/07534bb658cedceb7d549d779458ef53178fbe.debug
...
/usr/lib/debug/usr
/usr/lib/debug/usr/lib64
/usr/lib/debug/usr/lib64/audit
/usr/lib/debug/usr/lib64/audit/sotruss-lib.so.debug
/usr/lib/debug/usr/lib64/gconv
/usr/lib/debug/usr/lib64/gconv/ANSI_X3.110.so.debug
/usr/lib/debug/usr/lib64/gconv/ARMSCII-8.so.debug
...
/usr/src/debug
/usr/src/debug/glibc-2.17-c758a686
/usr/src/debug/glibc-2.17-c758a686/argp
/usr/src/debug/glibc-2.17-c758a686/argp/argp-ba.c
/usr/src/debug/glibc-2.17-c758a686/argp/argp-eexst.c
/usr/src/debug/glibc-2.17-c758a686/argp/argp-fmtstream.c
/usr/src/debug/glibc-2.17-c758a686/argp/argp-fmtstream.h
...
```
使用`rpm -ql glibc-debuginfo`查看软件包中的文件发现主要有三类文件：
 - 存放在`/usr/lib/debug/.build-id`目录下的`nn/nnn...nnn.debug`文件，文件名为hash值。
 - 存放在`/usr/lib/debug/`下的其他`.debug`文件，文件名为库文件名`+.debug`后缀。
 - 存放在`/usr/src/debug/`下的源代码文件。

这些文件和我们上面介绍的debuginfo文件内容基本一致，可以按照上述方式查看文件内容。

## debuginfo在内核漏洞分析中的应用
在内核漏洞分析中，需要对内核进行调试，并分析目标结构体在二进制文件中的偏移。但是发行版提供的内核通常剥离了debuginfo信息，这时就可以从发行版的`kernel-debuginfo`软件包中获取调试信息。

```text
# rpm -ql kernel-debuginfo
...
/usr/lib/debug/lib/modules/3.10.0-1160.31.1.el7.x86_64/vdso
/usr/lib/debug/lib/modules/3.10.0-1160.31.1.el7.x86_64/vdso/vdso.so.debug
/usr/lib/debug/lib/modules/3.10.0-1160.31.1.el7.x86_64/vdso/vdso32-int80.so.debug
/usr/lib/debug/lib/modules/3.10.0-1160.31.1.el7.x86_64/vdso/vdso32-syscall.so.debug
/usr/lib/debug/lib/modules/3.10.0-1160.31.1.el7.x86_64/vdso/vdso32-sysenter.so.debug
/usr/lib/debug/lib/modules/3.10.0-1160.31.1.el7.x86_64/vmlinux
...
# file /usr/lib/debug/lib/modules/3.10.0-1160.31.1.el7.x86_64/vmlinux
/usr/lib/debug/lib/modules/3.10.0-1160.31.1.el7.x86_64/vmlinux: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, BuildID[sha1]=145340cd0f97d560527fa7222afae8c87699e72b, not stripped
```
`kernel-debuginfo`中包含各个模块的debuginfo文件及ELF结构的`vmlinux`文件，此`vmlinux`文件可用于在gdb调试内核时提供符号信息及确定结构体偏移。

```text
# pahole -C cred /usr/lib/debug/lib/modules/3.10.0-1160.31.1.el7.x86_64/vmlinux                                                 
struct cred {
	atomic_t                   usage;                /*     0     4 */
	kuid_t                     uid;                  /*     4     4 */
	kgid_t                     gid;                  /*     8     4 */
	kuid_t                     suid;                 /*    12     4 */
	kgid_t                     sgid;                 /*    16     4 */
	kuid_t                     euid;                 /*    20     4 */
	kgid_t                     egid;                 /*    24     4 */
	kuid_t                     fsuid;                /*    28     4 */
	kgid_t                     fsgid;                /*    32     4 */
	unsigned int               securebits;           /*    36     4 */
	kernel_cap_t               cap_inheritable;      /*    40     8 */
	kernel_cap_t               cap_permitted;        /*    48     8 */
	kernel_cap_t               cap_effective;        /*    56     8 */
	/* --- cacheline 1 boundary (64 bytes) --- */
	kernel_cap_t               cap_bset;             /*    64     8 */
	unsigned char              jit_keyring;          /*    72     1 */

	/* XXX 7 bytes hole, try to pack */

	struct key *               session_keyring;      /*    80     8 */
	struct key *               process_keyring;      /*    88     8 */
	struct key *               thread_keyring;       /*    96     8 */
	struct key *               request_key_auth;     /*   104     8 */
	void *                     security;             /*   112     8 */
	struct user_struct *       user;                 /*   120     8 */
	/* --- cacheline 2 boundary (128 bytes) --- */
	struct user_namespace *    user_ns;              /*   128     8 */
	struct group_info *        group_info;           /*   136     8 */
	struct callback_head       rcu;                  /*   144    16 */
	kernel_cap_t               cap_ambient;          /*   160     8 */

	/* size: 168, cachelines: 3, members: 25 */
	/* sum members: 161, holes: 1, sum holes: 7 */
	/* last cacheline: 40 bytes */
};
```
使用`pahole`工具可以解析调试信息并快速确定二进制文件中目标结构体成员及偏移，后两列分别给出了成员变量在结构体中的偏移及成员变量的大小。

## 参考文献
1. [Debugging Information in Separate Files](https://sourceware.org/gdb/onlinedocs/gdb/Separate-Debug-Files.html)
2. [DWARF4](http://dwarfstd.org/doc/DWARF4.pdf)
