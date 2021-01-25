---
layout: "post"
author: LJR
category: 操作系统
title: 《Linux Inside》kernel booting process (一)
mathjax: true
tags:
  - Linux
---

## 1. 术语

+ CS:IP
  + 8086: CPU reset后CS寄存器值为0xFFFF，IP寄存器值为0，（左移16位加IP）即0xFFFF0，1MB往下16字节的位置
  + 80286：CPU reset后CS寄存器值为0xF000，IP寄存器的值为0xFFF0，（左移16位加IP）即0xFFFF0
  + 80386:CPU reset后CS寄存器值为0xF000，CS base寄存器为0xFFFF0000，IP的值为0xFFF0，（base和ip相加）即0xFFFFFFF0，4GB往下16字节的位置
+ A20线
+ MBR分区布局(master boot record)
+ qemu文本模式显示：`qemu_system_x86_64 bootloader -curses`
  + 文本模式下的退出`Esc + 2`
  + gui界面的退出：`Ctrl + a + x`
+ coreboot
+ UEFI + GPT和BIOS + MBR

下面是AT&T语法和intel语法的一些不同之处

```
* 寄存器命名原则
    AT&T: %eax                      Intel: eax
* 源/目的操作数顺序 
    AT&T: movl %eax, %ebx           Intel: mov ebx, eax
* 常数/立即数的格式　
    AT&T: movl $_value, %ebx        Intel: mov eax, _value
  把value的地址放入eax寄存器
    AT&T: movl $0xd00d, %ebx        Intel: mov ebx, 0xd00d
* 操作数长度标识 
    AT&T: movw %ax, %bx             Intel: mov bx, ax
* 寻址方式 
    AT&T:   immed32(basepointer, indexpointer, indexscale)
    Intel:  [basepointer + indexpointer × indexscale + imm32)
```

### 1.1. 一些指令

+ `cld`指令：将flag寄存器中方向标志位DF清零。
+ `rep; stosl`: `stosl`将`eax`中四字节数据存储到`edi`指向的内存中，并将`edi`加4.`rep`重复执行后面的指令，每次执行`ecx`减1，直到`ecx`为0.

## 2. 你好，世界

`1. press power button -> 2. motherboard sends signal to power supply device -> 3. motherboard receives power good signal -> 4. start CPU -> 5. CPU resets all leftover data in registers and sets predefined values for each`

对于80386及之后的处理器架构，其初始化的cs和ip寄存器如下

```
IP  0xfff0
CS selector 0xf000
CS base 0xffff0000
```

处理器从实模式开始工作，实模式被所有x86系列处理器所支持。8086处理器具有20位地址总线，但是寄存器只有16位，为了让16位的寄存器能索引20位的地址空间，采用对内存分段的方式，将内存地址分为段地址和偏移地址两部分。物理地址计算方式如下：

$$物理地址\; =\; Segment\; Selector\; *\; 16\; +\; Offset$$

> cs寄存器(code segment): 包含两部分，segment selector和隐藏的base address（通常为segment selector的值乘以16）。base address初始化为0xffff0000，cs初始化为0xf000，处理器会一直使用base address直到cs发生改变。  
> ip寄存器(instruction pointer): 初始化为0xfff0

因此，starting地址为$0xffff0000 + 0xfff0 = 0xfffffff0$，这个值位于4GB下方16字节处，被称为复位向量(reset vector)。其中包含了一条`jmp`指令跳转到BIOS的入口点(entry point)。 

之后BIOS会找到可启动设备。当尝试从磁盘启动时，BIOS会尝试寻找第一个boot扇区，在以MBR作为分区布局的磁盘中第一个扇区的前446字节为启动扇区。第一个扇区的最后两个字节`0x55`和`0xaa`表明该设备是可启动的。

一个启动扇区的示例

```nasm
;
; Note: this example is written in Intel Assembly syntax
;
[BITS 16] ; 16位实模式

boot:
    mov al, '!'
    mov ah, 0x0e
    mov bh, 0x00
    mov bl, 0x07

    int 0x10
    jmp $ ; $表示当前地址，即死循环

times 510-($-$$) db 0 ; 重复510次，用0填满510字节空间

db 0x55 ; 魔数
db 0xaa ; 魔数
```

如上，BIOS将MBR中内容load到内存0x7c00后，BIOS将控制权交给MBR，接着启动扇区的代码会在16位实模式执行。代码调用0x10中断会打印`!`。同时填充100字节的0以及最后的魔数0x55和0xaa。

实模式下的内存映射:

```
0x00000000 - 0x000003FF - Real Mode Interrupt Vector Table
0x00000400 - 0x000004FF - BIOS Data Area
0x00000500 - 0x00007BFF - Unused
0x00007C00 - 0x00007DFF - Our Bootloader
0x00007E00 - 0x0009FFFF - Unused
0x000A0000 - 0x000BFFFF - Video RAM (VRAM) Memory
0x000B0000 - 0x000B7777 - Monochrome Video Memory
0x000B8000 - 0x000BFFFF - Color Video Memory
0x000C0000 - 0x000C7FFF - Video ROM BIOS
0x000C8000 - 0x000EFFFF - BIOS Shadow Area
0x000F0000 - 0x000FFFFF - System BIOS
```

可以注意到实模式可用内存只有1MB，这是因为8086地址总线只有20位，即最多支持1MB地址空间。

对于复位向量，其被保存在ROM中而非RAM中，进而被映射到地址空间中。

`0xFFFE_0000 - 0xFFFF_FFFF: 128 kilobyte ROM mapped into address space`

## 3. bootloader: GRUB 2

bootloader的实现需要遵守boot protocol。

BIOS将控制权交给启动扇区的代码，并从[boot.img](http://git.savannah.gnu.org/gitweb/?p=grub.git;a=blob;f=grub-core/boot/i386/pc/boot.S;hb=HEAD)处开始执行。其代码非常简单，仅仅是跳转到GRUB 2的core image的位置。core image从[diskboot.img](http://git.savannah.gnu.org/gitweb/?p=grub.git;a=blob;f=grub-core/boot/i386/pc/diskboot.S;hb=HEAD)开始，其通常存储在第一个扇区之后。这里的代码将GRUB 2的内核和文件系统驱动装载到内存中。之后，执行`grub_main`函数。

`grub_main`函数初始化终端、获取模块的基地址、设置根设备、装载并解析grub配置文件、装载模块等。`grub_main`最终会进入normal mode，`grub_normal_execute`函数会完成最终的准备阶段并给出能够选择的操作系统列表。当选择之后，`grub_menu_execute_entry`函数会运行，并执行`boot`命令启动被选择的操作系统。

bootloader必须读取并写入内核设置部分(setup header)的某些域。这些域的位置由linker script给出，即位于内核设置代码的`0x01f1`偏移处。如下：

```asm
    .globl hdr
hdr:
    setup_sects: .byte 0
    root_flags:  .word ROOT_RDONLY
    syssize:     .long 0
    ram_size:    .word 0
    vid_mode:    .word SVGA_MODE
    root_dev:    .word 0
    boot_flag:   .word 0xAA55
```

这些值由命令行给出或者在booting过程中计算得到。

如boot protocol中，内核装载到内存之后的地址空间映射如下

```
         | Protected-mode kernel  |
100000   +------------------------+
         | I/O memory hole        |
0A0000   +------------------------+
         | Reserved for BIOS      | Leave as much as possible unused
         ~                        ~
         | Command line           | (Can also be below the X+10000 mark)
X+10000  +------------------------+
         | Stack/heap             | For use by the kernel real-mode code.
X+08000  +------------------------+
         | Kernel setup           | The kernel real-mode code.
         | Kernel boot sector     | The kernel legacy boot sector.
       X +------------------------+
         | Boot loader            | <- Boot sector entry point 0x7C00
001000   +------------------------+
         | Reserved for MBR/BIOS  |
000800   +------------------------+
         | Typically used by MBR  |
000600   +------------------------+
         | BIOS use only          |
000000   +------------------------+
```

bootloader将控制权交给内核时，从

$$X + sizeof(KernelBootSector) + 1$$

启动。X是kernel boot sector被装载的地址。

## 4. The Beginning of the Kernel Setup Stage

目前，内核还没有运行。内核配置阶段（setup part）需要配置解压缩器和内存管理。之后会解压缩实际的内核并跳转过去。setup part从[arch/x86/boot/header.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S)的`_start`符号开始执行。

```asm
  .globl _start
_start:
  .byte 0xeb
  .byte start_of_setup-1f
1:
  //
  // rest of the header
  //
# End of setup header #####################################################

	.section ".entrytext", "ax"
start_of_setup:
# Force %es = %ds
	movw	%ds, %ax
	movw	%ax, %es
	cld
```

0xeb表示`jmp`指令，是一条相对跳转指令。`start_of_setup - 1f`表示`start_of_setup`label和本地label`1:`地址相减的结果。该指令位于内核实模式偏移的`0x200`处，即第一个512字节之后。根据kernel boot protocol，需要保证

```
segment = grub_linux_real_target >> 4;
state.gs = state.fs = state.es = state.ds = state.ss = segment;
state.cs = segment + 0x20;
```

若内核被装载到物理地址`0x10000`处，则

```
gs = fs = es = ds = ss = 0x1000 ; 其它段选择器
cs = 0x1020 ; 左移4位后，就指向第一条的jmp指令
```

在跳转到`start_of_setup`后，内核需要：

+ 设置所有段寄存器的值，保证它们相同
+ 设置栈
+ 设置bss
+ 跳转到[arch/x86/boot/main.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/main.c)中的c代码处。

## 5. Aligning the Segment Registers

首先保证`es`和`ds`相同

```assembly
movw    %ds, %ax # %ax = %ds
movw    %ax, %es # %es = %ax
cld
```

同时也需要把cs寄存器的值设置为和ds寄存器相同，

```assembly
  pushw   %ds
  pushw   $6f
  lretw 
  # 1. ret从栈中将返回地址pop到EIP寄存器中
  # 2. l前缀表示`far return`，因此指令首先从栈中pop值到EIP寄存器，然后pop第二个值到CS寄存器。
  # 3. w后缀表示实模式下操作数宽度为16位
6:
```

这里的代码将`ds`的值和label`6:`的地址依次压入栈，然后执行`lretw`指令。当`lretw`被调用，它将会把`ip`寄存器的地址设置为`6:`的地址，同时把`cs`的值设置为`ds`的值

## 6. Stack Setup

setup代码主要是为了实模式下的c语言环境做准备。下一步是验证并设置`ss`寄存器的值

```assembly
movw    %ss, %dx # %dx = %ss
cmpw    %ax, %dx # %dx - %ax
movw    %sp, %dx # %dx = %sp
je      2f
```

此时`ax`的值为`ds`的值。比较`ax`和`ss`可能有三种可能结果

+ `ss`和`ax`相等为有效的值`0x1000`
+ `ss`无效且`CAN_USE_HEAP`被置位
+ `ss`无效且`CAN_USE_HEAP`没有被置位

当`ss`为`0x1000`有效时会跳转到`2:`如下：

```assembly
2:  # Now %dx should point to the end of our stack space 
    andw    $~3, %dx      # %dx &= 1100b 四字节对齐
    jnz     3f            # if %dx != 0 goto 3
    movw    $0xfffc, %dx  # %dx = 0xfffc
3:  movw    %ax, %ss      # %ss = %ax
    movzwl  %dx, %esp     # %esp = %dx
    sti
```

此时`dx`中是`sp`的值。首先将`dx`四字节对齐(末两位置0)，然后检查是否为0。若为0，则把`dx`置为`0xfffc`（64KB段中最后一个四字节对齐的位置）。若非0，则继续使用bootloader给出的`sp`的值。接着，设置`ss`为`0x1000`，设置`esp`为`dx`的值。此时内存空间如下图，底部是setup代码，

![stack-1](https://github.com/0xAX/linux-insides/raw/master/Booting/images/stack1.png)

当`ss`!=`ds`时，处理代码如下

```assembly
    # Invalid %ss, make up a new stack
    xmovw	$_end, %dx
    xtestb	$CAN_USE_HEAP, loadflags
    jz	1f
    movw	heap_end_ptr, %dx
1:  addw	$STACK_SIZE, %dx
    jnc	2f
    xorw	%dx, %dx  # Prevent wraparound
```

首先将`_end`地址的值放入`dx`中，然后使用`testb`测试`loadflags`掩码，`loadflags`定义如下：

```
#define LOADED_HIGH     (1<<0)
#define QUIET_FLAG      (1<<5)
#define KEEP_SEGMENTS   (1<<6)
#define CAN_USE_HEAP    (1<<7)
```

如果`CAN_USE_HEAP`置位了，则将`dx`设置为`heap_end_ptr`(_end+STACK_SIZE-512)，并加上`STACK_SIZE`(最小的栈大小，1024字节)。这之后，如果`dx`结果没有溢出，就跳转到`2:`处执行。(说明`dx`此时已经指向栈顶)

如果`CAN_USE_HEAP`没有置位，则使用一个最小栈：从`_end`到`_end + STACK_SIZE`。如下：

![stack-2](https://0xax.gitbooks.io/linux-insides/content/Booting/images/minimal_stack.png)

## 7. BSS Setup

接着比较setup_sig和魔数(`0x5a5aaa55`)。

```assembly
6:
# Check signature at end of setup
	cmpl	$0x5a5aaa55, setup_sig
	jne	setup_bad
```

若魔数能够成功匹配，则表明段寄存器和栈设置成功。因此在跳转到`main`函数前只需设置BSS段。

BSS段用于存储静态未初始化/初始化为0的变量。Linux确保BSS段初始化为0。如下，

```assembly
# Zero the bss
	movw	$__bss_start, %di
	movw	$_end+3, %cx
	xorl	%eax, %eax
	subw	%di, %cx
	shrw	$2, %cx # cx >> 2
	rep; stosl
```

首先将`__bss_start`地址移动到`di`中。接着`_end + 3`(4字节对齐)被放入到`cx`中。清零`eax`。并将bss节的大小`cx - di`放入`cx`中。之后`cx`被除以4（右移两位）。再重复使用`stosl`指令，即将`eax`的值(=0)存入`di`指向的地址中，每次自动将`di`加4，`cx`减1，直到`cx`为0。


## 8. Jump to main

设置好栈和BSS后，就需要跳转到`main()`函数，如下代码所示

```asm
calll main
```

main函数位于[arch/x86/boot/main.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/main.c)中。

## 9. 小结

总结一下，上电启动阶段主要涉及三个东西

+ BIOS: 复位向量给出的第一条指令（`jmp`）跳转到BIOS。
+ bootloader(GRUB 2): 启动扇区为boot.img，之后跟着的是core.img。其会遵守kernel boot protocol，将内核装载到内存（一般为0x10000）处。
+ 内核实模式代码: GRUB会将控制权交给内核实模式代码，其会设置各个段寄存器的值、初始化堆、栈、BSS段等，共占用64KB空间(包括代码以及boot sector)。boot sector最开始两个字节为`0x4d`和`0x5a`，即`MZ`表示MS-DOS header。
