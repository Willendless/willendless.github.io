---
layout: "post"
category: 操作系统
author: LJR
title: 《Linux Inside》kernel booting process (二)
mathjax: true
tags:
    - Linux
---

> Base and Bound.

这部分会包括下面内容

+ 什么是保护模式
+ 如何从实模式转换到保护模式
+ 堆和控制台的初始化
+ 内存检测、CPU验证和键盘初始化

## 1. 术语

+ Long mode: 64位程序能够运行的x86-64架构。
+ protected mode: 32位地址总线，支持分页和分段的x86-64架构。
+ `movsl`指令：以四字节为单位将操作数2寄存器指向的内存地址中数据移动到操作数1寄存器指向的内存地址。

## 2. 保护模式

在进入long mode之前，cpu需要首先切换到保护模式。

保护模式具有32位地址总线，和实模式类似，内存地址分段访问。每个段的基地址和大小由*Segment Descriptor*给出，存放于被称为GDT表(*Global Descriptor Table*)的数据结构中。

GDT表存放于内存中，该地址并不固定，由`GDTR`寄存器给出。使用如下指令`lgdt gdt`将表装载到内存。

`GDTR`是一个48位的寄存器，由两部分组成：

+ 16bit：全局描述符表(global descriptor table)的大小
+ 32bit：全局描述符表的地址

GDT中64位的段描述符结构如下

```
63         56         51   48    45           39        32 
------------------------------------------------------------
|             | |B| |A|       | |   | |0|E|W|A|            |
| BASE 31:24  |G|/|L|V| LIMIT |P|DPL|S|  TYPE | BASE 23:16 |
|             | |D| |L| 19:16 | |   | |1|C|R|A|            |
------------------------------------------------------------

 31                         16 15                         0 
------------------------------------------------------------
|                             |                            |
|        BASE 15:0            |       LIMIT 15:0           |
|                             |                            |
------------------------------------------------------------
```

### 2.1. Limit域

+ 20位的Limit域由bits0-15和bits48-51组成。它定义了`段的长度 - 1`，配合着第55位`G`（Granularity）有如下组合
    + 若`G`为0且段限制(segment limit即limit域)为0，则段大小为1字节
    + 若`G`为1且段限制为0，则段大小为4096字节
    + 若`G`为0且段限制为0xfffff，则段大小为1MB
    + 若`G`为1且段限制为0xfffff，则段大小为4GB

这意味着

+ 如果G为0，则limit被解读为1字节，因此最大段大小为1MB
+ 如果G为1，则limit被解读为4096B=4K=1页，因此最大段大小为4G。

事实上，`Limit`的值由`G`由左移12位得到。

### 2.2. Base域

32位的Base域由bits 16-31， 32-39和56-63组成。定义了段的起始地址。

### 2.3. Type/Attribute域

Type/Attribute域由bits 40-44组成。定义了段的类型以及如何访问段。

+ bit 44的`S`flag表明了描述符的类型。若`S`为0，则该段为系统段，若`S`为1，则为代码段或数据段（栈段为可读可写的数据段）
+ bit 43的`Ex`属性表明了是代码段还是数据段。若为0则是数据段，否则是代码段。

段的类型如下

```
--------------------------------------------------------------------------------------
|           Type Field        | Descriptor Type | Description                        |
|-----------------------------|-----------------|------------------------------------|
| Decimal                     |                 |                                    |
|             0    E    W   A |                 |                                    |
| 0           0    0    0   0 | Data            | Read-Only                          |
| 1           0    0    0   1 | Data            | Read-Only, accessed                |
| 2           0    0    1   0 | Data            | Read/Write                         |
| 3           0    0    1   1 | Data            | Read/Write, accessed               |
| 4           0    1    0   0 | Data            | Read-Only, expand-down             |
| 5           0    1    0   1 | Data            | Read-Only, expand-down, accessed   |
| 6           0    1    1   0 | Data            | Read/Write, expand-down            |
| 7           0    1    1   1 | Data            | Read/Write, expand-down, accessed  |
|                  C    R   A |                 |                                    |
| 8           1    0    0   0 | Code            | Execute-Only                       |
| 9           1    0    0   1 | Code            | Execute-Only, accessed             |
| 10          1    0    1   0 | Code            | Execute/Read                       |
| 11          1    0    1   1 | Code            | Execute/Read, accessed             |
| 12          1    1    0   0 | Code            | Execute-Only, conforming           |
| 14          1    1    0   1 | Code            | Execute-Only, conforming, accessed |
| 13          1    1    1   0 | Code            | Execute/Read, conforming           |
| 15          1    1    1   1 | Code            | Execute/Read, conforming, accessed |
--------------------------------------------------------------------------------------
```

如上，可以注意到第一位(bit 43)为0表示*data*段，为1表示*code*段。接下来的三位（40，41，42）为`EWA`(Expansion Writable Accessible)或者`CRA`(Conforming Readable Accessible)。

+ E(bit 42)为0表示expand up段。
+ W(bit 41)为1表示允许写访问，若为0表示只读。对于数据段，读访问总是被允许的。
+ A(bit 40)控制着段是否能被处理器访问。
+ C(bit 42)为代码段的确认位。若为1表示该段的代码能够在低优先级（例如：用户态）被执行。若为0则表示能够在相同优先级被执行。
+ R(bit 41)控制着代码段的读访问。若为1表示段能够被读。代码段的写访问不能被保证。

### 2.4. 其它

+ DPL(bits 45-46)定义了段的优先级。可以为0~3，0表示最高优先级。
+ P(bit 47)表示段是否在内存中。若P为0，则段会被表示成*invalid*同时处理器会拒绝从段中读取。
+ AVL(bit 52)保留位。被Linux忽略。
+ L(bit 53)表明代码段是否包含原生64位代码。如果置1，则表示代码段在64位模式中执行。
+ D/B(bit 54)表示操作数大小即16/32位。

### 段选择子

和实模式类型，段寄存器包含了段选择子。然而保护模式下的每个段选择描述符都对应有一个16位的段选择子如下

```
 15             3 2  1     0
-----------------------------
|      Index     | TI | RPL |
-----------------------------
```

+ **Index**存储着GDT表中描述符的索引
+ *TI*(Table Indicator)表明描述符的位置。0表示描述符在GDT表中。0表示描述符在LDT(Local Descriptor Table)中。
+ *RPL*包含着请求者的优先级。

每一个段寄存器都由可见和隐藏两个部分组成

+ 可见：段选择符(segment selector)
+ 隐藏：段描述符(base，limit，attributes & flags)

为在保护模式下获取物理地址

+ 将segment selector装载到某个段寄存器
+ CPU在`GDT address + Index`位置找到段描述符，并装载到段寄存器的隐藏部分(hidden part)
+ 若分页被禁用，则段的线性地址/物理地址由如下公式给出：`Base address + Offset`

![保护模式物理地址](https://0xax.gitbooks.io/linux-insides/content/Booting/images/linear_address.png)

实模式转换成保护模式的算法如下

+ 禁中断
+ 使用`lgdt`指令给出并装载GDT
+ 设置CR0(Control Register)寄存器的PE(Protection Enable)位
+ 跳转到保护模式代码

## 3. 拷贝启动参数到"zeropage"

[arch/x86/boot/main.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/main.c)中首先调用的函数是`copy_boot_params(void)`。它将kernel setup header拷贝到`boot_params`结构。

该函数主要执行两个操作：

+ 从[header.S]()将`hdr`拷贝到`boot_params`结构的`setup_header`域。
+ 若内核依据旧的命令行协议装载到内存则更新指向内核命令行的指针。

[copy.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/copy.S)文件中`memcpy`文件的实现如下

```assembly
#include <linux/linkage.h>
	.code16
	.text

GLOBAL(memcpy)
	pushw	%si
	pushw	%di
	movw	%ax, %di
	movw	%dx, %si
	pushw	%cx
	shrw	$2, %cx
	rep; movsl
	popw	%cx
	andw	$3, %cx
	rep; movsb
	popw	%di
	popw	%si
	retl
ENDPROC(memcpy)
```

`arch/x86/Makefile`中的`REALMODE_CFLAGS`表明构建系统在实模式下使用`-mregparm=3`参数，即函数从`ax`,`dx`,`cx`中获取前三个参数。

`memcpy`的调用为`memcpy(&boot_params.hdr, &hdr, sizeof hdr);`。`memcpy`将`boot_params.hdr`的地址放入`di`，`hdr`地址放入`si`，`sizeof hdr`放入栈。之后将`cx`右移两位，并重复从`si`对应`&hdr`位置拷贝4字节到`di`对应的`&boot_params.hdr`位置。之后恢复`%cx`的值，将其模4(`&0b11`)，再拷贝剩余最多3字节。最后恢复`di`和`si`的值。

`GLOBAL(name)`在[arch/x86/include/asm/linkage.h](https://github.com/torvalds/linux/blob/v4.16/arch/x86/include/asm/linkage.h)中定义，`ENDPROC`在[include/linux/linkage.h](https://github.com/torvalds/linux/blob/v4.16/include/linux/linkage.h)中定义。如下

```assembly
#define GLOBAL(name)	\
	.globl name;	\
	name:
```

```assembly
#ifndef END
#define END(name) \
	.size name, .-name
#endif

#ifndef ENDPROC
#define ENDPROC(name) \
	.type name, @function ASM_NL \
	END(name)
#endif
```

## 4. 控制台初始化

下一步`main`调用的是[arch/x86/boot/early_serial_console.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/early_serial_console.c)中定义的`console_init`函数。



## 5. 堆初始化

## 6. CPU校验

## 7. 内存检测

## 8. 键盘初始化

## 9. 参数查询

## 10. 小结
