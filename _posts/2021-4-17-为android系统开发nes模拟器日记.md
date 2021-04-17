---
layout: "post"
title: 为Android系统开发NES模拟器（一）
author: LJR
category: 模拟器
tags:
    - arch
---

最近为了满足毕业学分要求不得不修移动应用开发（Android）这门课，实在不喜欢任何和前后端还有应用开发（各种管理系统？？？）有关的东西，又因为是跨专业修低年级的课，也不认识能组队的人。恰好之前逛Rust社区有看到用Rust写NES模拟器的[教程](https://bugzmanov.github.io/nes_ebook/index.html)。当时很想写但是没有时间。这次虽然没有时间但是得硬着头皮挤时间写（XD），遂尝试参考该教程用Kotlin在安卓系统上实现NES模拟器当作大作业。这一系列主要记录开发过程中迷思和笔记：

## 1. NES架构

和现代处理器和相比，NES平台/红白机的架构相当简陋:

```c
----------------------------
NES Games(assembly language)
----------------------------
    Machine language
----------------------------
  Computer Architecture
----------------------------
   ALU  |  Memory elements
----------------------------
    sequential logic
----------------------------
       circuit
----------------------------
```

从模拟硬件组件的角度，NES主要有下面这些硬件组件：

+ **CPU**: NES用的CPU是8-bit的6502芯片（APPLE II，世界上第一台个人电脑）的修改版本 2A03。
+ **PPU**(*Picture Processing Unit*): 在屏幕上渲染当前的游戏，2C02芯片。
+ **RAM**: CPU和PPU各有一个2kB大小的RAM，但是RAM和CPU/PPU仍然是母板上独立的两个芯片
+ **APU**(*Audio Processing Unit*): 2A03芯片的一部分，能够生成5信道的声音
+ **Catridge**: ROM卡带，主要有两种
  + Character ROM(*CHR ROM*): 存储游戏的视频图像数据
  + Program ROM(*PRG ROM*): 存储游戏代码的CPU指令
+ **Gamepads**：游戏柄，从玩家获取输入

这些组件中最重要的两个是PPU和CPU，这两个芯片都有自己的内部RAM内存，游戏一般被存储在卡带里的ROM芯片中，当卡带被插入系统，CPU就能够从中读取数据。

NES内的处理器通过内存映射IO和其它组件（PPU和输入设备）通信。

真实硬件板子的图片：

![nes架构](https://i.loli.net/2021/04/06/dTL3xlCEBWVubXo.png)

### 1.1. Central Processing Unit(CPU)

NES系统使用的CPU是6502的改版2A03，和6502的不同之处在于

+ 没有10进制的BCD模式
+ 能够处理声音，因此处理器同时作为pAPU（pseudo-Audio Processing Unit）和CPU。

除此之外，2A03使用的指令集和6502的相同，均为小端架构。

#### 1.1.1. 内存映射

**处理器**架构图：

![processor-diagram.png](https://i.loli.net/2021/04/14/uV7WtOnb81NscEk.png)

内存地址空间被划分为3部分：

+ 卡带中的ROM
+ CPU的RAM
+ IO寄存器

地址总线被用于提供地址（16位），控制总线被用于判断IO请求是读还是写。ROM是只读的，通过MMC访问。IO寄存器被用于和系统其它组件通信，PPU和控制设备。

16位地址总线支持从0x0到0xFFFF的64kB内存。下图是内存映射

![mem-map.png](https://i.loli.net/2021/04/14/6q8bKZ4XSdwvUn2.png)

+ 0页：代表了从0x0到0xFF的地址，是内存中的第一个页
+ 0x0\~0x7FF的内存位置被镜像了3次至0x800\~0x1FFF
  + 对0x0的写，同时也会被写入0x800，0x1000和0x1800
+ 内存映射IO寄存器位于0x2000~0x401F
  + 0x2000~0x2007每8个字节被镜像到0x2008~0x3FFF
+ SRAM（WRAM）是静态RAM（通过电池供电，存放游戏的状态，例如：分数、排名等WoW），地址被用于访问卡带中的可写RAM
+ 0x8000之上是分配给PRG-ROM卡带的地址
  + 只有1个16kBbank的PRG-ROM将会被装载到0x8000和0xc000处
  + 有两个bank的，1个被装到0x8000，另一个被装到0xc000
  + 超过两个bank的，会根据地址决定将哪个bank装入内存
    + MMC: **Memory mapper controler**
+ 从实现的角度
  + CPU自己的RAM只有2k，占据了0\~0x2000的空间
  + 0x2000~0x4020被重映射到PPU，APU，GamePads以及其它硬件组件
  + 0x4020~0x6000的空间是历史包袱，不同版本用途不一样，不管他
  + 0x6000~0x8000为RAM卡带保留，有的游戏需要，也不管他
  + 0x8000~0x10000映射到卡带的Program ROM（PRG ROM）空间

#### 1.1.2. 寄存器

下面是访问寄存器还有RAM需要的时钟周期

|操作|时钟周期|
|---|---|
|寄存器访问|2|
|RAM前255字节|3|
|255字节外的空间|4-7|

寄存器如下。虽然有通用意图寄存器，但是这些寄存器都是匿名的，即如果要使用，则必须使用具体的指令。（想到编写程序的时候的命名抽象）

+ **特殊用途寄存器**
  + PC: 16位的程序计数器
  + SP: 内存地址空间的0x0100~0x1FF用于栈（。。。好吧只有256字节，XD因为是8位机，寄存器只有8位），不存在栈溢出检测，栈指针会从0x00wrap around到0xFF。
  + P(Processor Status): 8位的寄存器，包含被上一次操作修改的7个状态标志（例如：上一次操作结果若是0，则Z标志被置1，否则置0）
+ **通用用途寄存器**
  + Accumulator(`A`)：存储运算/内存访问操作的结果。也可以作为某些操作的输入参数。
  + 索引寄存器X(`X`)：某个寻址模式下的偏移，也可用做辅助存储，例如用于temp值，counter
  + 索引寄存器Y(`Y`)：和X相同

|标志|名称|作用|
|---|---|---|
|N|negative|正负|
|V|overflow|借位/进位|
|B|break|`brk`指令|
|D|decimal|置位后，算术操作为BCD`09+01=10`，清除则二为补码|
|I|禁中断interrupt disable|置位，则禁中断|
|Z|zero|结果为0|
|C|carry|进位|

NES中未使用标志位D。

![status.png](https://i.loli.net/2021/04/14/WNOHj7glS59oUh2.png)

#### 1.1.3. 中断

NES有三种不同类型的中断，NMI，IRQ和reset。当中断发生时，跳转的**地址**被存放在0xFFFA~0xFFFF的向量表种。中断发生时，处理器执行下面的操作：

1. 识别中断请求
2. 完成当前指令的执行
3. 将程序计数器和状态寄存器压入栈
4. 设置禁中断
5. 从向量表中装载中断处理例程的地址到pc
6. 执行中断处理例程
7. 在执行中断返回RTI（Return From Interrupt）指令时，从栈中弹出pc和状态寄存器的值，恢复上下文
8. 恢复程序的执行

##### 1.1.3.1. IRQ中断

maskable中断。当中断禁位标志被设置时，它们可被处理器忽视。IRQ可以通过BRK指令被软件出啊发。当IRQ发生时，系统跳转到0xFFFE~0xFFFF给出的地址。

##### 1.1.3.2. NMI中断

non-maskable中断，当V-Blank发生在每一帧结束时由PPU生成。NMI发生时系统跳转到0xFFFA~oxFFFB中的地址处。处理见下图。

![nmi.png](https://i.loli.net/2021/04/14/x3CL6RQ5BTiXVb8.png)

##### 1.1.3.3. 复位中断

当系统启动或者用户按下复位按钮后，复位中断触发。系统跳转到0xFFFC~0xFFFD中的位置。

系统优先级为

1. 复位请求
2. NMI
3. IRQ

中断延迟为7个时钟周期。

### 1.2. 11种寻址模式

#### 1.2.1. 0页寻址

这种寻址模式使用单个操作数，作为指向零页（0x0000-0x00FF）中某个地址的指针，因此操作数只需要1个字节，指令更短，因此比使用两个操作数的指令执行得也更快。

例如：`AND $12`，其中的12指的就是地址0x0012。

#### 1.2.2. 索引0页寻址

索引0页寻址使用单个操作数并加上某个索引寄存器的值。存在两种形式的零页索引：

+ 零页，X：增加X寄存器的值。例如，`AND $12, X`
+ 零页，Y：增加Y寄存器的值，只能被用于`LDX`和`STX`。例如，`LDX $12, Y`

寻址模式通过opcode编码，因而总共也为2字节。时钟周期会长一些（长1或2个时钟周期）。

在执行加法时，遵循wraparound。因此数据的地址总是在零页中。

#### 1.2.3. 绝对寻址

数据地址由两个操作数给出，least significant byte优先。例如，`AND $1234`。

#### 1.2.4. 索引绝对寻址

索引绝对寻址使用两个操作数，构成16位的地址，least significant byte优先，同时加上一个索引寄存器的值。

#### 1.2.5. 间接寻址

间接寻址使用两个操作数，构成16位的地址A，A中给出目标地址的least significant byte，A+1给出目标地址的most significant byte。

例如，如果操作数是bb和cc，且ccbb包括了xx，ccbb+1包括了yy，则真正的目标地址是yyxx。

在6502上，只有JMP指令使用了这种模式，例如`JMP (1234)`。对于该指令，pc会被设置为yyxx，并从该位置开始执行。

#### 1.2.6. 隐式寻址（implied addressing）

很多指令不需要内存中的操作数。例如`CLD`，`NOP`

#### 1.2.7. 累加器寻址

累加器寻址仅操作累加器中的内容。只有移位指令使用该寻址模式。

#### 1.2.8. 立即数寻址

立即数寻址直接使用指令中提供的立即数常量。立即数前用`#`指示，例如`AND #$12`

#### 1.2.9. 相对寻址

相对寻址用于分支指令，这种寻址模式在某种情况满足时改变pc的值。若条件满足，单个操作数（1字节）会被加给pc，因而允许pc在-128到127的相对变化。

#### 1.2.10. 索引间接寻址

也被称为pre-indexed寻址，使用单个字节作为操作数，首先给出零页中的地址A + X和A + X + 1。这两个地址中的数构成最终的目标地址。

例如，若操作数是0xbb，0x00bb + x中是xx而0x00bb + x + 1是yy，则数据在yyxx中。例如`AND ($12, X)`

#### 1.2.11. 间接索引寻址

也被称为post-indexed寻址，使用单个字节作为操作数，首先给出零页中的地址A和A+1。将该16位地址中存放的数据加上某个索引寄存器后得到目标地址。

例如，若操作数是bb，00bb是xx，00bb + 1是yy，则数据能够在在**yyxx + Y**中找到。例如`AND ($12),Y`

这两个寻址模式的区别在于对操作数使用索引寄存器还是对操作数给出的值使用索引寄存器。

### 1.3. 指令

6502有56种不同的指令，在不同的寻址模式下有不同的变种，总共151个操作码（opcodes）。取决于寻址模式，指令可能为1，2或3字节长。第一个字节是opcode，剩余字节是操作数。

6502是累加器架构，意味着所有指令操作最多包括单个寄存器和某个内存单元，且寄存器都是匿名的，也就是无法直接通过名字控制寄存器，指令助记符的编码给出了具体使用的寄存器。下面来看下6502的指令种类吧（就不瓦特把指令集抄了一遍。。。原地址见[这里](#3-参考))

+ **读取/存储指令**：一般是从内存/立即数装载到寄存器A/X/Y
+ **栈操作**: 对A/X/Y的出入栈操作，标志位P的出入栈，栈指针传送到X/从X设置
+ **寄存器/内存递增、递减**IN(A/X/Y),DE(A/X/Y),INC/DEC
+ **移位**:
  + ASL：算术左，高位移入进位标志
  + LSR：逻辑右，低位移入进位标志
  + ROL：rotate循环左移
  + ROR: rotate循环右移
+ **逻辑操作**：
  + 与/或/异或AND, ORA, EOR
  + 比较BIT/CMP/CPX/CPY
  + 原子指令？测试和清除TRB/TSB
  + 内存位清除和设置RMB/SMB
+ **加减**：ADC, SBC
+ **分支跳转**：
  +  JMP: Stack<-PC, PC<-Address
  +  RTS: PC<-Stack
  +  RTI: P<-Stack, PC<-Stack
  +  BRA: PC = PC + offset
  +  BEQ/BNE: if Z=1/0, PC+=offset
  +  BCC/BCS: if C=0/1, PC+=offset
  +  BVC/BVS: if V=0/1, PC+=offset
  +  BMI/BPL: if N=1/0, PC += offset
  +  BBR/BBS: branch on bit reset/set
+ **处理器状态**：
  + 清除标志位：CLC/CLD/CLI/CLV
  + 设置标志位：SEC/SED/SEI
+ **传送指令**：X/Y/累加器到累加器/X/Y
  + TAX/Y, TXA/Y
+ **特殊**
  + NOP
  + 同步中断BRK

## 2. 参考

+ [nes_ebook](https://bugzmanov.github.io/nes_ebook/index.html)
+ [6502CPU以及NES游戏机系统](http://49.212.183.201/6502/6502_report.htm)
+ 这个文档写得很好！！介绍架构就应该这么介绍！！[Nintendo Entertainment System Documentation](http://nesdev.com/NESDoc.pdf)
