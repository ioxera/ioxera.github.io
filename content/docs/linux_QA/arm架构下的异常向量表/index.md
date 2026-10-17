---
title: arm架构下的异常向量表
---

Linux 内核在 ARM（32 位）与 ARM64（AArch64）下异常向量表的定义与安装流程完全不同，下面分开讲清楚 **定义位置**、**安装过程**、**关键文件与寄存器**。

---

## 一、ARM 32-bit（arch/arm）

### 1. 异常向量表定义在哪里？

**核心文件：**

- `arch/arm/kernel/entry-armv.S`

**向量表段与符号：**

```asm
.section .vectors, "ax", %progbits  @ 专门的向量段
.L__vectors_start:
	W(b)	vector_rst		@ 复位
	W(b)	vector_und		@ 未定义指令
	W(ldr)	pc, .L__vectors_start + 0x1000  @ SWI
	W(b)	vector_pabt		@ 预取中止
	W(b)	vector_dabt		@ 数据中止
	W(b)	vector_addrexcptn	@ 地址异常
	W(b)	vector_irq		@ IRQ中断
	W(b)	vector_fiq		@ FIQ中断
.L__vectors_end:
```

- 链接脚本指定位置：`arch/arm/kernel/vmlinux.lds.S`

```c
__vectors_start = .;
.vectors 0xffff0000 : AT(__vectors_start) {
	*(.vectors)
}
__vectors_end = .;
```

**虚拟地址：**

- Linux 32 位 ARM 固定使用 **高端向量 0xffff0000**（CP15 c1 V 位 = 1）

### 2. ARM 32 异常向量表如何安装（启动流程）

**阶段 1：汇编早期（head.S）**

- 设置 CP15 c1 寄存器，将 V 位（bit13）置 1 → 启用 **0xffff0000 高端向量**

**阶段 2：C 语言初始化（setup_arch）**

调用链：
```
start_kernel
  ↳ setup_arch
    ↳ devicemaps_init
      ↳ early_alloc 分配一页内存
      ↳ early_trap_init(vectors)  // 复制向量表+stub到新页
      ↳ create_mapping 把物理页映射到虚拟 0xffff0000
```

**关键函数：`early_trap_init`**

- 把 `.vectors` 段的向量表 + 异常处理 stub 复制到刚分配的物理页
- 确保运行时向量表在 **0xffff0000** 可执行

---

## 二、ARM64 / AArch64（arch/arm64）

### 1. 异常向量表定义在哪里？

**核心文件：**

- `arch/arm64/kernel/entry.S`

**向量表定义：**
```
.align 11                  @ 2KB对齐 (2^11)
SYM_CODE_START(vectors)    @ 符号名：vectors
	kernel_ventry	1, sync_invalid	// EL1t, SP0 同步
	kernel_ventry	1, irq_invalid		// EL1t IRQ
	kernel_ventry	1, fiq_invalid		// EL1t FIQ
	kernel_ventry	1, error_invalid	// EL1t SError

	kernel_ventry	1, sync			    // EL1h, SP_EL1 同步
	kernel_ventry	1, irq			    // EL1h IRQ（常用）
	kernel_ventry	1, fiq_invalid		// EL1h FIQ
	kernel_ventry	1, error_invalid	// EL1h SError
    ...
SYM_CODE_END(vectors)
```

- 共 **4 组 ×4 项 = 16 项**，每项 128 字节（0x80 对齐）
- 基地址由 **VBAR_EL1** 寄存器控制

### 2. ARM64 异常向量表如何安装

**完全在汇编阶段完成（head.S），关键代码（__primary_switched）：**
```
adr_l	x8, vectors			@ 取vectors符号的虚拟地址
msr	vbar_el1, x8			@ 写入VBAR_EL1寄存器
isb							@ 同步指令流水线
```

**特点：**

- 不需要复制到其他页，**直接用内核镜像里的 vectors 地址**
- 不需要映射 0xffff0000
- 启动时就设置好，进入 C 语言前已生效

---

## 三、32 位 ARM vs ARM64 对比（速查）

|项目|ARM 32-bit|ARM64 (AArch64)|
|---|---|---|
|**定义文件**|`arch/arm/kernel/entry-armv.S`|`arch/arm64/kernel/entry.S`|
|**向量符号**|`__vectors_start`|`vectors`|
|**运行虚拟地址**|`0xffff0000`|由 VBAR_EL1 指向内核 `vectors` 符号|
|**安装时机**|启动中期（C 函数 devicemaps_init）|启动早期（汇编 head.S）|
|**安装方式**|分配物理页 → 复制向量表 → 映射到 0xffff0000|直接把 `vectors` 地址写入 VBAR_EL1|
|**控制寄存器**|CP15 c1, V 位 (bit13)|VBAR_EL1（异常基地址寄存器）|
|**对齐要求**|32 字节|2KB（align 11）|

---

## 四、总结

- **ARM 32**：向量表定义在 `entry-armv.S`，启动中期复制并映射到固定虚拟地址 **0xffff0000**。
- **ARM64**：向量表定义在 `entry.S`，启动早期直接把符号地址写入 **VBAR_EL1** 寄存器。

# Questions
## 1. 为什么arm32需要复制向量表，不直接使用内核镜像中的向量表？
ARM32 必须把向量表从内核镜像里**复制**出来、再**重新映射到 0xffff0000**，核心原因是：**硬件强制固定地址 + 早期 MMU 映射限制 + 安全与权限隔离**，三者叠加导致不能直接用镜像里的 `.vectors` 段。
### 一、最根本：ARM 硬件只认两个 “固定地址”

ARMv7-A 架构规定：

- 异常发生时，CPU **强制跳转到两个硬编码地址之一**：
    
    - **低端向量**：`0x00000000`
    - **高端向量**：`0xffff0000`
    
- 由 CP15_C1 寄存器的 **V 位（bit13）** 选择：`V=1 → 0xffff0000`

Linux 选 **0xffff0000（高端向量）**，因为：

- `0x00000000` 属于用户空间（0～3G），容易被空指针踩、不安全
- 内核空间在 3G～4G，`0xffff0000` 在内核高区，更安全

**问题**：

内核镜像里的 `.vectors` 段，**链接地址虽然写的是 0xffff0000**，但：

- 内核实际被加载到 **物理内存某块连续区域**（如 `0x30008000` 开始）
- 早期页表只做了 **线性映射**：`虚拟 = 物理 + 0xC0000000`
- 所以镜像里 `.vectors` 的 **运行时虚拟地址 ≠ 0xffff0000**

CPU 只认 `0xffff0000`，不认内核镜像里的实际地址 → **必须把向量表搬到 0xffff0000 能访问的物理页，并建立映射**。

---

### 二、内核启动：早期 MMU 映射不包含 0xffff0000

### 1. 内核刚启动时的映射（head.S）

- 只映射：
    
    - 内核代码 / 数据（`0xC0000000+` 线性映射）
    - 必要设备区域
    
- **完全没有映射 0xffff0000** 这个虚拟地址
- 此时如果直接用镜像里的向量表：
    
    - CPU 跳 `0xffff0000` → **缺页 / 访问违例** → 系统直接挂
    

### 2. 必须动态分配 + 复制 + 重新映射

流程（`devicemaps_init → early_trap_init`）：

1. **分配 2 个物理页（8KB）**
    
    - 一页放向量表（`vectors`）
    - 一页放异常处理 stub（`stubs`）
    
2. **从内核镜像复制**
    
    - 从 `__vectors_start ~ __vectors_end` 复制到新物理页
    - 同时复制 `__stubs_start ~ __stubs_end`
    
3. **建立映射**
    
    - 虚拟地址：`0xffff0000`
    - 物理地址：刚分配的页
    - 权限：**只读、内核态 - only、用户不可访问**
    
4. **设置 CP15 V=1**
    
    - 告诉 CPU：异常去 `0xffff0000`
    

---

### 三、链接脚本 “骗” 了你：.vectors 只是 “链接地址”

看 `arch/arm/kernel/vmlinux.lds.S`：

plaintext

```
__vectors_start = .;
.vectors 0xffff0000 : AT(__vectors_start) {
	*(.vectors)
}
```

- `0xffff0000` 是 **链接时虚拟地址（VMA）**
- `AT(__vectors_start)` 表示：**加载地址（LMA）跟在内核镜像后面**

意思是：

- 编译时：符号地址按 `0xffff0000` 算
- 运行时：**实际物理地址在内核镜像末尾**，不在 0xffff0000 对应的物理位置

**结论**：

镜像里的 `.vectors` 是 **“假地址”**，CPU 不认 → **必须复制重定位**。

---

### 四、安全与权限：必须单独一页、严格保护

1. **向量表必须只读**
    
    - 复制到新页后，设为 **AP = 只读、用户不可访问**
    - 防止被意外 / 恶意改写，避免系统崩溃
    
2. **内核镜像本身是可写的（至少启动时）**
    
    - 直接用镜像里的向量表：**无法单独设权限**
    - 要么整段内核不可写（影响调试 / 动态修改），要么向量表可写（不安全）
    
3. **kuser helpers 共存**
    
    - 0xffff0000 页还放 **用户态辅助函数**（如 `__kuser_cmpxchg`）
    - 必须和内核镜像隔离、单独映射、单独权限
    

---

### 五、对比 ARM64：为什么它不用复制？

ARM64 没有 “固定地址” 限制：

- 向量表基地址由 **VBAR_EL1 寄存器** 任意设置
- 只要把 `vectors` 符号的虚拟地址写入 VBAR_EL1 即可
- 不需要固定地址、不需要复制、不需要额外映射

ARM32 是 **老架构历史包袱**：硬件强制固定地址 + 早期 MMU 设计限制。

## 2. arm32架构早期直接把内核镜像中的向量表映射到0xffff0000不就行了嘛，为什么还要重新分配然后复制一次？
ARM32 历史架构 + Linux 设计共同决定的，缺一不可：

### 1. **向量表不能和内核镜像放在同一段物理内存里**

ARM32 异常向量表有一个**硬性架构约束**：

> **异常向量表必须放在 16KB 对齐的物理地址上**
> 
> 并且必须是**一段独立、连续、可被 MMU 单独设置权限**的内存

而你的内核镜像：

- 是**一大段连续物理内存**（text + data + rodata + vectors 混在一起）
- **物理地址不可能刚好 16KB 对齐**
- 更不可能把镜像里**一小段 32/64 字节**单独拆出来映射

MMU 映射最小单位是**页（4KB）**，你不可能：

- 映射内核镜像整体到 `0xC0000000`
- 同时又把**同一段物理内存**的**前 32 字节**单独映射到 `0xffff0000`

**硬件不支持同一段物理内存映射到两个不同虚拟地址，还只截取一小段。**

所以：

**必须把向量表搬到一个全新的、独立的物理页**，才能单独映射到 0xffff0000。

---

### 2. **权限控制要求：向量表必须是只读 + 不可缓冲，内核镜像不能**

ARM 异常向量表要求：

- **必须设为 Strongly Ordered / Device 内存**（不可缓存）
- **必须只读**
- **用户态不可执行**

但内核镜像：

- 是正常的 **Cached + Buffered** 内存
- text 段是只读，但 data/bss 是可写
- 不能把整个内核镜像改成 Device 内存（会卡死）

**解决方案只有一个：**

把向量表复制到**单独一页**，然后对这一页设置：

- AP 权限：内核只读
- 内存类型：Device/Strongly-ordered
- 不缓冲、不缓存

**你不可能对内核镜像里的一小段单独改内存类型和权限。**

---

### 3. **向量表后面紧跟着要放异常处理 stub（必须紧挨着）**

ARM32 异常处理有个特殊结构：

plaintext

```
0xffff0000    向量表
0xffff0020    保留
0xffff0100    异常处理 stub 代码（必须紧跟在向量表后）
```

stub 代码会直接用：

asm

```
ldr pc, [pc, #-0x1000]
```

这种硬编码偏移跳转。

这意味着：

**向量表 + stub 必须放在同一个 4KB/16KB 物理页内，连续排布。**

但 stub 代码**并不在链接脚本的 .vectors 段里**，而是分散在 entry-armv.S。

你不可能把内核里分散的代码直接映射成连续的 0xffff0000 区域。

**必须复制到新页，才能把 vectors + stubs 拼成连续一块。**

---

### 4. 总结

**ARM32 不能直接映射内核镜像里的向量表，因为：**

1. **MMU 不能只映射一小段，必须整页映射**
2. **向量表需要独立的内存属性与权限，不能和内核镜像共用**
3. **向量表 + stub 必须连续存放，镜像里不连续**
4. **物理地址无法保证 16KB 对齐**

所以 Linux 只能：

- 新分配一页
- 复制向量表
- 复制 stub
- 单独映射到 0xffff0000
- 单独设置内存类型与权限

**这不是设计多余，是 ARM32 架构决定的。**
对比 ARM64 ：

- 向量表地址**任意**（写 VBAR_EL1）
- 不需要固定 0xffff0000
- 不需要单独页
- 不需要复制
- 不需要重新映射

**所以 ARM64 直接用镜像里的 vectors，根本不用复制。**
ARM32 完全是**老旧架构遗留限制**。