# Lab 1: Booting a PC

## Part1:PC Bootstrap

### Getting Started with x86 assembly

首先我们需要熟悉x86的汇编语言，在官网中为我们提供了很阅读资料(https://pdos.csail.mit.edu/6.828/2017/reference.html)

当然也可以在Lab的网页(https://pdos.csail.mit.edu/6.828/2017/labs/lab1/)中找打具体的References

#### **Exercise 1.	**熟悉汇编语言

 Familiarize yourself with the assembly language materials available on [the 6.828 reference page](https://pdos.csail.mit.edu/6.828/2018/reference.html). You don't have to read them now, but you'll almost certainly want to refer to some of this material when reading and writing x86 assembly.

We do recommend reading the section "The Syntax" in [Brennan's Guide to Inline Assembly](http://www.delorie.com/djgpp/doc/brennan/brennan_att_inline_djgpp.html). It gives a good (and quite brief) description of the AT&T assembly syntax we'll be using with the GNU assembler in JOS.

### Simulating the x86

#### Building a a simple kernel

我们不会在真实的物理PC机上进行实验，相替代的我们会使用一个叫做qemu的软件来模拟一个完整的PC

> 它能够简化debug，在qemu上debug会比在真实的硅质芯片上插入break point更容易，qemu本质上就是使用软件来模拟计算机硬件会做的事情

qemu自己本省提供的debug功能十分的有限，所以我们这里会使用GNU debugger(GDB)来一步一步进入JOS的启动过程

首先跳转到我们从github上clone下来的文件，调用make指令：

```shell
moocos@moocos-VirtualBox:~/6.828/lab$ make
+ as kern/entry.S
+ cc kern/entrypgdir.c
+ cc kern/init.c
+ cc kern/console.c
+ cc kern/monitor.c
+ cc kern/printf.c
+ cc kern/kdebug.c
+ cc lib/printfmt.c
+ cc lib/readline.c
+ cc lib/string.c
+ ld obj/kern/kernel
+ as boot/boot.S
+ cc -Os boot/main.c
+ ld boot/boot
boot block is 380 bytes (max 510)
+ mk obj/kern/kernel.img
```

会出现如下自动生成的命令。

这里我们会构建最小的6.828 boot loader以及kernel（我们会在后面的课程中一点一点丰富他）

> 如果我们在这里出现了错误，类似如下的代码
>
> ```shell
> obj/kern/printfmt.o: In function `printnum':
> lib/printfmt.c:41: undefined reference to `__udivdi3'
> lib/printfmt.c:49: undefined reference to `__umoddi3'
> make: *** [obj/kern/kernel] Error 1
> ```
>
> 这说明我们没有没有安装32-bit gcc multilib
>
> 终端中输入如下命令完成安装
>
> ```shell
> sudo apt-get install gcc-multilib
> ```

#### Run a kernel on qemu

现在我们已经准备好了一些简单的文件例如

`obj/kern/kernel.img`：他是模拟PC的一个虚拟硬盘，它有着一个**boot loader**(obj/boot/boot) 以及一个 **kernel**(obj/kernel)

接下来我们可以使用两个命令来启动qemu：

- make qemu：会同时打开VGA和Serial
  - 打印出来的这些行会同时出现在普通的我们**运行make qumu的shell窗口**以及QEMU的显示窗口，这是因为课程组为了测试以及打分的目的，他们设置JOS的kernel来不仅会在**VGA(Video Graphics Array)**也就是QUMU窗口输出，也会在模拟的PC串口(Serial Port)哪里QEMU会把他的输出到自己的标准输出。
  - JOSkernel会把从键盘以及串口中读取数据，所以你可以既在VGA中也可以在终端中输入命令
- make qemu-nox：只打开Serial
  - 使用这个命令我们可以使用serial console而**不打开VGA**，这样使用`SSH`来运行的十分方便

```shell
moocos@moocos-VirtualBox:~/6.828/lab$ make qemu-nox
sed "s/localhost:1234/localhost:26000/" < .gdbinit.tmpl > .gdbinit
***
*** Use Ctrl-a x to exit qemu
***
qemu-system-i386 -nographic -drive file=obj/kern/kernel.img,index=0,media=disk,format=raw -serial mon:stdio -gdb tcp::26000 -D qemu.log
6828 decimal is XXX octal!
entering test_backtrace 5
entering test_backtrace 4
entering test_backtrace 3
entering test_backtrace 2
entering test_backtrace 1
entering test_backtrace 0
leaving test_backtrace 0
leaving test_backtrace 1
leaving test_backtrace 2
leaving test_backtrace 3
leaving test_backtrace 4
leaving test_backtrace 5
Welcome to the JOS kernel monitor!
Type 'help' for a list of commands.
K>
```

我们目前只有两个命令可给kernel monitor中输入，help和kerninfo

```shell
K> help
help - display this list of commands
kerninfo - display information about the kernel
K> kerninfo
Special kernel symbols:
  entry  f010000c (virt)  0010000c (phys)
  etext  f0101a75 (virt)  00101a75 (phys)
  edata  f0112300 (virt)  00112300 (phys)
  end    f0112960 (virt)  00112960 (phys)
Kernel executable memory footprint: 75KB
K>
```

help命令是确定的，kerninfo我们会稍后讨论。尽管十分简单，但是我们需要注意到，kernel monitor”直接“运行在我们模拟的PC”虚拟硬件“上的。所以我们可以直接将(obj/kern/kernel.img)的内容覆盖在一个真正的硬盘上的前几个区域，也就是存入一个真正的PC上。运行机器后看是否和我们在QEMU上显示的一样。如果你的硬盘上又十分重要的信息，不建议这么做因为覆盖原有的重要信息

### The PC's Physical Address Space

PC物理内存如下：

```
+------------------+  <- 0xFFFFFFFF (4GB)
|      32-bit      |
|  memory mapped   |
|     devices      |
|                  |
/\/\/\/\/\/\/\/\/\/\

/\/\/\/\/\/\/\/\/\/\
|                  |
|      Unused      |
|                  |
+------------------+  <- depends on amount of RAM
|                  |
|                  |
| Extended Memory  |
|                  |
|                  |
+------------------+  <- 0x00100000 (1MB)
|     BIOS ROM     |
+------------------+  <- 0x000F0000 (960KB)
|  16-bit devices, |
|  expansion ROMs  |
+------------------+  <- 0x000C0000 (768KB)
|   VGA Display    |
+------------------+  <- 0x000A0000 (640KB)
|                  |
|    Low Memory    |
|                  |
+------------------+  <- 0x00000000
```

- 0x00000000-0x000A0000(640KB)  -- 用户可用的内存空间

  - Low Memory -- 世界上地一台电脑打在Intel 8088 16位处理器，仅仅有**1MB(20位地址线)**的物理内存。早期电脑地址从0x00000000到0x0000FFFF而不是到0xFFFFFFFF.标志着`Low Memory`的640KB是早期PC能够使用的**RAM（Random Access Memory)**物理内存空间。

- 0x000A0000-0x000FFFFF(384KB)  --  为特殊作用的`硬件预留的内存空间`:例如**显示缓存(display buffer)**，以及一些**非易失性(non-volatile)内存**

  - 最重要的部分就是**BIOS(Basic Input Output System)**  地址从0x000F0000 - 0x000FFFFF。**(64KB)**

  - 在早期BIOS是存储在ROM中的，但是现代的计算机中BIOS通常存储在可升级的(updatable)的闪存(flash memory)中

  - **BIOS负责计算机系统的【初始化】**，例如**激活显卡以及确定已安装内存的大小**

  - 初始化完成后，BIOS负责从正确的地方(软盘-floppy disk,硬盘，CD_ROM)等等装载OS，然后将控制权给OS

    

尽管现在的内存已经非常大了，但是PC架构师为了**向后兼容性(backward capacompatibility)**仍然保留了原始的排布，将1M的内存给溜了出来。因此现代计算机在物理内存中从0x000A0000到0x00100000存在一个空闲地带(hole)，他将RAM分割为”低“或者”传统内存“ **（conventional memory）**（前640KB）和 扩展内存**(extended memory)**两个部分

除此之外，PC32位物理地址空间的顶端的一些地址空间,在所有物理RAM之上，被预留给`32位PCI设备`

最近的x86处理器可以支持不止4GB的物理RAM，所以地址可扩充到0xFFFFFFFF之后。这种情况下BIOS必须拿牌留出第二个hole出来，在32可编址空间，给这些32位的设备来映射。

### The ROM BIOS

> 这部分我们会使用QEMU的来debug确定电脑如何启动

首先我们需要开启两个terminal，都带lab目录中。**一个运行**make qemu-gdb(或者make qemu-nox-gdb（SSH连接的话）)，这会启动qemu但是qemu会停止直到执行第一条指令并且等待GDB的调试连接。第二个运行make gdb.我们会看到这样的输出：

```shell
moocos@moocos-VirtualBox:~/6.828/lab$ make gdb
gdb -n -x .gdbinit
...

The target architecture is assumed to be i8086
[f000:fff0]    0xffff0:	ljmp   $0xf000,$0xe05b
0x0000fff0 in ?? ()
+ symbol-file obj/kern/kernel
(gdb)
```

接下来我们来看GDB的输出

```assembly
[f000:fff0]    0xffff0:	ljmp   $0xf000,$0xe05b
```

这是GDB第一条将要执行的反汇编代码，我们应该从端代码中理解如下知识

- 电脑执行的第一条指令在物理地址的`0x000ffff0`，也就是BIOS在PC物理地址中的比较顶层的地方（BIOS结尾的前16个字节）

  ```
  +------------------+  <- 0x00100000 (1MB)
  |     BIOS ROM     |
  +------------------+  <- 0x000F0000 (960KB)
  ```

- PC开始执行：**CS = 0xf000**以及**IP = 0xfff0**

  - 底层是如何计算的呢？将CS*16(左移一位) + IP就可以得到我们的执行地址0xffff0（参考<汇编语言中的1.2.6节>）

> CS类似于selector(本质上是段的基地址)，IP就是offset

- 第一个将会被执行的指令时一个jmp指令,他会跳转到片段地址CS = 0xf000以及IP = 0xe05b

我们前面可以看到BIOS是确定的位于物理地址的0x000f0000-0x000fffff这一段地址。这样的设计总是可以**确保在开机执行别的程序或系统前优先让BIOS获得计算机的控制权**。

这样做是十分重要的，因为在开机前RAM中还没有装载任何程序，处理器什么也不能做。

QEMU得到自己的BIOS，也就是他处理器模拟的物理地址空间。一旦处理器重置后，处理器就会进入**实模式**，并且将CS寄存器设置为0xf000，PI设置为0xfff0。这样就会从开始执行CS:IP这对指向的地址。如何计算这对地址？

> 这里先简单介绍一下地址的计算方法，在实模式下，指令中出现的地址都是采用
>
> 　　　　**(段基址：段内偏移)**
>
> 的形式的。其中这两个字段的值，通常是存放寄存器中。其中段基址必须放在【段寄存器】中，包括CS(代码段), DS(数据段), SS(堆栈段), ES(扩展段)。不同的段寄存器存放的是你程序不同的段的起始地址。

$ *physical address* = 16 * *segment* + *offset*. $

```
 16 * 0xf000 + 0xfff0   # in hex multiplication by 16 is
   = 0xf0000 + 0xfff0     # easy--just append a 0.
   = 0xffff0 
```

0xffff0是BIOS结尾的前16个字节，。所以他的跳转指令跳转到更前面的位置就十分理所当然了。那么这16字节可以完成什么呢？

#### **Exercise 2.	使用GDB了解BIOS在计算机启动时做了什么** 

> Use GDB's si (Step Instruction) command to trace into the ROM BIOS for a few more instructions, and try to guess what it might be doing. You might want to look at [Phil Storrs I/O Ports Description](http://web.archive.org/web/20040404164813/members.iweb.net.au/~pstorr/pcbook/book2/book2.htm), as well as other materials on the [6.828 reference materials page](https://pdos.csail.mit.edu/6.828/2017/reference.html). No need to figure out all the details - just the general idea of what the BIOS is doing first.

```shell
#执行跳转指令
[f000:fff0]    0xffff0:	ljmp   $0xf000,$0xe05b	#跳转到0xfe05b这个地址
[f000:e05b]    0xfe05b:	cmpl   $0x0,%cs:0x65a4	#将cs:0x65a4这个地址的内容和0x0这个地址作比较。设置eflags中的ZF位
[f000:e062]    0xfe062:	jne    0xfd2b9					#如果不相等的话发生跳转，显然这里的CS段中0x65a4寄存器的值为0x0，至于为什么要比较这个地址我也不知道，但是就是【检查】这个地址是否有被赋值

		#为了证明，我们使用x命令来查看0xf65a4也就是cs:0x65a4地址所指向的内容
(gdb) x/x 0xf65a4
0xf65a4:	0x00000000
		#我们可以看到0xf65a4的地址就是0，所以不会发生跳转

#一系列初始化
[f000:e066]    0xfe066:	xor    %ax,%ax					#使用异或运算将ax寄存器清零
[f000:e068]    0xfe068:	mov    %ax,%ss					#将ax的值赋值给ss的值
[f000:e06a]    0xfe06a:	mov    $0x7000,%esp			#将0x7000这个值赋值给esp堆栈寄存器
[f000:e070]    0xfe070:	mov    $0xf3c4f,%edx		#对edx寄存器进行初始化
		#这里我们调用info registers命令可以看到对应的寄存器值
    (gdb) info registers
    eax            0x0	0
    ecx            0x0	0
    edx            0xf3c4f	998479
    ebx            0x0	0
    esp            0x7000	0x7000
    ebp            0x0	0x0
    esi            0x0	0
    edi            0x0	0
    eip            0xe076	0xe076
    eflags         0x46	[ PF ZF ]
    cs             0xf000	61440
    ss             0x0	0
    ds             0x0	0
    es             0x0	0
    fs             0x0	0
    gs             0x0	0
#绝对跳转
[f000:e076]    0xfe076:	jmp    0xfd12a			#跳转到0xfd12a这个地址
[f000:d12a]    0xfd12a:	mov    %eax,%ecx		#完成跳转，并且初始化寄存器

[f000:d12d]    0xfd12d:	cli									#这条指令的作用是关闭中断指令
																						#cli 中断标志置0，使IF=0
[f000:d12e]    0xfd12e:	cld									#STD以及CLD指令分别用于设置以及清除DF标志,所以这里是清除了字符串的自动递减，使得字符串自动递增，栈向下生长
		#设置DF标志使得串指令自动递减（从高地址向低地址方向处理字符串），清除该标志则使得串指令自动递增。
#这三条指令是用来关闭NMI中断的。
[f000:d12f]    0xfd12f:	mov    $0x8f,%eax
[f000:d135]    0xfd135:	out    %al,$0x70
[f000:d137]    0xfd137:	in     $0x71,%al
    #in %al, PortAddress    向端口地址为PortAddress的端口写入al寄存器的值
    #out PortAddres,%al     把端口地址为PortAddress的端口中的值读入寄存器al
    #查询清单，我们知道了，0x70端口和0x71端口是用于控制系统中一个叫做CMOS的设备，这个设备是一个低功耗的存储设备，它可以用于在计算机关闭时存储一些信息，它是由独立的电池供电的。
    #这个CMOS中可以控制跟PC相关的多个功能，其中最重要的就是时钟设备（Real Time Clock）的 ，它还可以控制是否响应不可屏蔽中断NMI(Non-Maskable Interrupt)。
    #操作CMOS存储器中的内容需要两个端口，一个是【0x70】另一个就是【0x71】。其中0x70可以叫做索引寄存器，这个8位寄存器的最高位是【不可屏蔽中断(NMI)使能位】。如果你把这个位置1，则NMI不会被响应。低7位用于指定CMOS存储器中的存储单元地址，所以如果你想访问第1号存储单元，并且在访问时，我要使能NMI，那么你就应该向端口0x70里面送入0b10000001 = 0x81。
    
#这三步操作又是在控制端口，此时被控制的端口号为0x92，
[f000:d139]    0xfd139:	in     $0x92,%al
[f000:d13b]    0xfd13b:	or     $0x2,%al
[f000:d13d]    0xfd13d:	out    %al,$0x92
#lidt指令：加载中断向量表寄存器(IDTR)。这个指令会把从地址0xf6ab8起始的后面6个字节的数据读入到中断向量表寄存器(IDTR)中。中断是操作系统中非常重要的一部分，有了中断操作系统才能真正实现进程。每一种中断都有自己对应的中断处理程序，那么这个中断的处理程序的首地址就叫做这个中断的中断向量。中断向量表自然是存放所有中断向量的表了。
[f000:d13f]    0xfd13f:	lidtw  %cs:0x66c0
```

> cmpl会将两个操作数相减，但不会保存，计算结果会改变eflags寄存器中的标志位，如果结果为0，则eflags中的ZF位为1



> je是一个条件跳转指令，他会检查eflags中的ZF是否为1，如果ZF为1则发生跳转如果不为1则不跳转
>
> jne判断是都不相等，如果不相等的话就跳转。



> 两个新的指令out, in。这两个操作是用于操作IO端口的。这种IO端口的操作我们后面会经常接触到，这里大致说下。
>
> 　 CPU与外部设备通讯时，通常是通过访问，修改设备控制器中的寄存器来实现的。那么这些位于【设备控制器】当中的【寄存器】也叫做**IO端口**。为了方便管理，80x86CPU采用【IO端口单独编址】的方式，即所有设备的端口都被命名到一个IO端口地址空间中。这个空间是【独立于内存地址空间】的。所以必须采用和访问内存的指令【不一样的指令】来访问端口。
>
> 标准规定端口操作必须要用【al寄存器作为缓冲】





## Part 2: The Boot Loader

​	硬盘或者软盘被划分为【**512字节**】一个区域，他的名字叫做**扇区(sector)**。一个扇区是磁盘传输的**最小粒度**：每次读写操作都需要一个或者多个扇区的大小。

​	如果一个磁盘是可启动的，那么他的第一个扇区就是**启动扇区(boot sector)**，哪里存在着**boot loade**r的代码

​	当BIOS找到可启动的硬盘或者软盘，他就将512字节的启动扇区载入到内存的物理地址的**0x7c00~0x7dff**，然后调用jmp指令跳转到0x7c00地址将控制权交给boot loader。

​	对于PC来说，这个地址是确定并且标准化的

> 现代使用BIOS从CS-ROM启动会稍微复杂一点，他的sector大小是2048，而不是512。BIOS在交换控制权前，会装载更大的字节到内存中

​	本实验就使用传统的硬件启动机制。也就是我们说的之转载512字节。

### 理解boot loader代码

> 代码存放在boot目录中，它包含一个汇编文件boot.S，以及一个C文件main.c

#### boot.S

<img src="../pic/Screen Shot 2021-09-13 at 3.03.26 PM.png" alt="Screen Shot 2021-09-13 at 3.03.26 PM" style="zoom:50%;" />

我们可以看到CS段的选择符是0x8 = 1000（引索号为1） DS段的选择符是0x10 = 0001 0000（引索号为2）

```assembly
#include <inc/mmu.h>

# Start the CPU: switch to 32-bit protected mode, jump into C.
# The BIOS loads this code from the first sector of the hard disk into
# memory at physical address 0x7c00 and starts executing in real mode
# with %cs=0 %ip=7c00.

.set PROT_MODE_CSEG, 0x8         # kernel code segment selector
.set PROT_MODE_DSEG, 0x10        # kernel data segment selector
.set CR0_PE_ON,      0x1         # protected mode enable flag

.globl start
start:
  .code16                     # Assemble for 16-bit mode
  cli                         # Disable interrupts
  cld                         # String operations increment

  # Set up the important data segment registers (DS, ES, SS).
  # 初始化DS、EX、SS段，将这些段设置为0
  xorw    %ax,%ax             # Segment number zero
  movw    %ax,%ds             # -> Data Segment
  movw    %ax,%es             # -> Extra Segment
  movw    %ax,%ss             # -> Stack Segment

  # Enable A20:下面这两段代码都适用于打开A20们
  #   For backwards compatibility with the earliest PCs, physical
  #   address line 20 is tied low, so that addresses higher than
  #   1MB wrap around to zero by default.  This code undoes this.
  
  #AT keyboard controller output port bit definitions	AT uses 60-64
	#	  bit 7 =    keyboard data output
	#	  bit 6 =    keyboard clock output
	#	  bit 5 = 0  input buffer full
	#	  bit 4 = 0  output buffer empty
	#	  bit 3 =    reserved (see note)
	#	  bit 2 =    reserved (see note)
	#	  bit 1 =    gate A20
	#	  bit 0 =    system reset
	
	#打开A20
seta20.1:
  #从地址为0x64(Keyboard Controller)的地址中读取一个字节数据到al寄存器
  inb     $0x64,%al               # Wait for not busy ， 直到键盘不忙了再继续执行命令
  #testb S2,S1 基于S1&S2，测试字节，与关系。
  #我们查表可知，bit 1 = 1 input buffer full (output 60/64 has data)
  testb   $0x2,%al
  
  #jnz 非零，跳转条件：~SF。判断键盘控制器的Input buffer是否满了，如果他满了没有就会跳转，直到键盘的输入缓存没有满
  jnz     seta20.1

  movb    $0xd1,%al               # 0xd1 = 1101 0001 -> port 0x64
  outb    %al,$0x64

seta20.2:
  inb     $0x64,%al               # Wait for not busy
  testb   $0x2,%al
  jnz     seta20.2

  movb    $0xdf,%al               # 0xdf = 1101 1111 -> port 0x60
  #0060	r	KeyBoard or KB controller data output buffer (via PPI on XT)
  outb    %al,$0x60

  # Switch from real to protected mode, using a bootstrap GDT(Global Descriptor Table)
  # and segment translation that makes virtual addresses 
  # identical to their physical addresses, so that the 
  # effective memory map does not change during the switch.
  
  # lgdt(load global descriptor table)
  
  # GDT是全局描述符表，GDTR是全局描述符表寄存器
  #进入保护模式
  #将CR0_PR_ON使能
  lgdt    gdtdesc #将GDT的位置加载到GDTR中
  #CPU使用GDTR寄存器来保存我们【GDT在内存中的位置和GDT的长度】。lgdt gdtdesc将源操作数的值（存储在【gdtdesc地址】中）加载到全局描述符表寄存器中。
  movl    %cr0, %eax
  orl     $CR0_PE_ON, %eax #将CR0_PR_ON使能，CR0的第0位是保护允许位PE(Protedted Enable)，用于启动保护模式，如果PE位置1，则保护模式启动，如果PE=0，则在实模式下运行。
  movl    %eax, %cr0
  
  # Jump to next instruction, but in 32-bit code segment.
  # Switches processor into 32-bit mode.
  ljmp    $PROT_MODE_CSEG, $protcseg

  .code32                     # Assemble for 32-bit mode
  #.code32 in boot.S tells assembler to insert 0x66
protcseg:
  # Set up the protected-mode data segment registers
  # 将数据选择符给所有的段
  movw    $PROT_MODE_DSEG, %ax    # Our data segment selector
  movw    %ax, %ds                # -> DS: Data Segment
  movw    %ax, %es                # -> ES: Extra Segment
  movw    %ax, %fs                # -> FS
  movw    %ax, %gs                # -> GS
  movw    %ax, %ss                # -> SS: Stack Segment
  
  # Set up the stack pointer and call into C.
  #	将start的地址复制到堆栈指针中，并且调用C函数
  movl    $start, %esp
  call bootmain

  # If bootmain returns (it shouldn't), loop.
spin:
  jmp spin

# 设置Bootstrap GDT
.p2align 2                                # force 4 byte alignment，强制4字节对齐
gdt:#声明了三个段描述符
  SEG_NULL				# null seg,(偏移量为0，共64位)全为0
  SEG(STA_X|STA_R, 0x0, 0xffffffff)	# code seg,(偏移量为1，共64位)是CS段
  SEG(STA_W, 0x0, 0xffffffff)	        # data seg,(偏移量为2，共64位)是DS段
  #所以，这就和前面的将CS设置为0x8(偏移量为1)，DS、SS设置为0x10(偏移量为2)相对应。
	#至于第一个段描述符为何全为0，这是Intel文档规定的，在此不做深究。
gdtdesc:
  .word   0x17                            # sizeof(gdt) - 1
  .long   gdt                             # address gdt
```



```c
//mmu.h中的定义
#define SEG_NULL						\
	.word 0, 0;						\
	.byte 0, 0, 0, 0
#define SEG(type,base,lim)					\
	.word (((lim) >> 12) & 0xffff), ((base) & 0xffff);	\
	.byte (((base) >> 16) & 0xff), (0x90 | (type)),		\
		(0xC0 | (((lim) >> 28) & 0xf)), (((base) >> 24) & 0xff)
```



##### 全局描述附表(GDT)

当进入保护模式之后，和一个段有关的信息需要使用【8个字节】来进行描述。这8个字节就被称为段**描述符（Segment Descriptor**）。所有的描述符在内存中连续存放，从而构成**描述符表**。

最主要的描述符是**全局描述符表（Global Descriptor Table，GDT）**。在进入保护模式之前，必须**先定义全局描述符表**。

理论上，全局描述符表可以位于内存中的任意地址处，不必对齐。但是在进入**保护模式(只能够使用1MB)之前我们就要设置好GDT**，所以**最开始GDT必须位于前1MB内存**中。为简单起见，我们将GDT放在紧挨着主引导程序之后。因为主引导程序大小为512字节，起始物理地址是0x7C00，所以GDT的地址为0x7C00 + 512，即**物理地址0x00007E00**处，如下图所示：

<img src="../pic/Screen Shot 2021-09-13 at 1.42.35 PM.png" alt="Screen Shot 2021-09-13 at 1.42.35 PM" style="zoom:50%;" />

##### 段描述符

> <img src="../pic/Screen Shot 2021-09-13 at 1.43.20 PM.png" alt="Screen Shot 2021-09-13 at 1.43.20 PM" style="zoom:50%;" />



由上图可知，每一个段描述符大小是8个字节，64位。上图中**下面是低4字节**的内容，**上面是高4字节**的内容。

描述符中指定了**32位的段起始线性地址，以及20位的段边界**。如果未开启分页功能，那么线性地址就是物理地址。

对于20位的段界限来说，如果是向上扩展的代码段或数据段，它决定了偏移量的最大值；如果是向下扩展的堆栈段，段界限决定了偏移量的最小值。





- **G代表段的粒度**。若G=0，则表示段界限以字节为单位，此时20位的段界限可以表示从1字节到1MB的范围；若G=1，则表示段界限以4KB为单位，此时20位的段界限可以表示从4KB到4GB的范围
- **S位用于表示描述符的类型**。当S=0时，表示这是一个系统段；当S=1时，表示这是一个代码段或数据段
- **DPL位表示描述符的特权级（Descriptor Privilege Level，DPL）。**这两位用于指定对应段的特权级，即0、1、2、3。其中0级为最高特权级别，3级为最低特权级别。刚进入保护模式时执行的代码具有最高的特权级别。通常0级用于操作系统的代码，3级用于普通用户程序的代码。有些CPU指令只能由0特权级的程序来执行
- **P表示段存在位（Segment Present）。**P位用于指示描述符所对应的段是否存在。当P=0时，表示段位于硬盘中；当P=1时，表示段位于内存中
- **D/B位表示默认操作设大小。**当为0时表示是16位的，当为1时表示是32位的
- **L位是64位代码段标志，**为1表示是64位
- **TYPE字段共有4位，**用于指示描述符的子类型，具体含义如下图所示：
  

<img src="../pic/Screen Shot 2021-09-13 at 1.48.23 PM.png" alt="Screen Shot 2021-09-13 at 1.48.23 PM" style="zoom:50%;" />

​	CPU使用GDTR寄存器来保存我们GDT在内存中的位置和GDT的长度。`lgdt gdtdesc`将源操作数的值（存储在gdtdesc地址中）加载到全局描述符表寄存器中。

##### 总结

- 打开A20 gate让计算机突破1MB的寻址限制
- 在内存中建立GDT表，并高速CPU GDT的位置
- 初始化控制寄存器，进入保护模式，
- 按照保护模式集训运行

#### main.c

- 磁盘排布

这里我们看到的main.c以及boot.S就是我们说的bootloader，他们两个的代码应该放在磁盘的第一个扇区。在此之上的第二个扇区就会存放kernel的代码。kernel的格式必须是ELF格式。

- 启动步骤

当CPU启动，他会加载BIOS到内存中，进行一些基本的设备检查、初始化。接着讲**boot loader装载到0x7c00**这个地址，并跳转到这个地址开始执行bootloader程序。

bootloader首先会执行**boot.S的初始化一些段寄存器，设置GDT，开启分页机制，程序来进入保护模式**。接着调用C程序bootmain()。他会读取kernel，并跳转到他开始执行kernel

```c
#include <inc/x86.h>
#include <inc/elf.h>

#define SECTSIZE	512
#define ELFHDR		((struct Elf *) 0x10000) // scratch space

void readsect(void*, uint32_t);
void readseg(uint32_t, uint32_t, uint32_t);

void
bootmain(void)
{
	struct Proghdr *ph, *eph;

	// read 1st page off disk，8个扇区(4K = PGSIZE)的内容(ELF可执行文件)至物理内存0x100000(1MB)处
	readseg((uint32_t) ELFHDR, SECTSIZE*8, 0);

	// is this a valid ELF?判断是否是合法的ELF格式的文件
	if (ELFHDR->e_magic != ELF_MAGIC)
		goto bad;

	// load each program segment (ignores ph flags)，然后根据ELF文件里面的相关信息将相关的程序代码复制到相应的物理内存处。
	ph = (struct Proghdr *) ((uint8_t *) ELFHDR + ELFHDR->e_phoff);
	eph = ph + ELFHDR->e_phnum;
	for (; ph < eph; ph++)
		// p_pa is the load address of this segment (as well
		// as the physical address)
		readseg(ph->p_pa, ph->p_memsz, ph->p_offset);

	// call the entry point from the ELF header，调用内核入口函数，进入到内核中（((void (*)(void)) (ELFHDR->e_entry))() ）。
	// note: does not return!
	((void (*)(void)) (ELFHDR->e_entry))();

bad:
	outw(0x8A00, 0x8A00);
	outw(0x8A00, 0x8E00);
	while (1)
		/* do nothing */;
}

// Read 'count' bytes at 'offset' from kernel into physical address 'pa'.
// Might copy more than asked
void
readseg(uint32_t pa, uint32_t count, uint32_t offset)
{
	uint32_t end_pa;

	end_pa = pa + count;

	// round down to sector boundary
	pa &= ~(SECTSIZE - 1);

	// translate from bytes to sectors, and kernel starts at sector 1
	offset = (offset / SECTSIZE) + 1;

	// If this is too slow, we could read lots of sectors at a time.
	// We'd write more to memory than asked, but it doesn't matter --
	// we load in increasing order.
	while (pa < end_pa) {
		// Since we haven't enabled paging yet and we're using
		// an identity segment mapping (see boot.S), we can
		// use physical addresses directly.  This won't be the
		// case once JOS enables the MMU.
		readsect((uint8_t*) pa, offset);
		pa += SECTSIZE;
		offset++;
	}
}

void
waitdisk(void)
{
	// wait for disk reaady
	while ((inb(0x1F7) & 0xC0) != 0x40)
		/* do nothing */;
}

void
readsect(void *dst, uint32_t offset)
{
	// wait for disk to be ready
	waitdisk();

	outb(0x1F2, 1);		// count = 1
	outb(0x1F3, offset);
	outb(0x1F4, offset >> 8);
	outb(0x1F5, offset >> 16);
	outb(0x1F6, (offset >> 24) | 0xE0);
	outb(0x1F7, 0x20);	// cmd 0x20 - read sectors

	// wait for disk to be ready
	waitdisk();

	// read a sector
	insl(0x1F0, dst, SECTSIZE/4);
}

```



###### 语法难点分析

- `ph = (struct Proghdr *) ((uint8_t *) ELFHDR + ELFHDR->e_phoff);`
   首先将ELFHDR转为 uint8_t 型指针，做加法的时候按照 byte 加，获得程序头表的起始位置，再将这个位置转为 Proghdr 型指针 ph。

- `((void (*)(void)) (ELFHDR->e_entry))();`
   将`ELFHDR->e_entry`转为一个无参数，无返回值的函数指针，并执行该函数。

##### readsec()

```C
void
readsect(void *dst, uint32_t offset)
{
	// wait for disk to be ready
	waitdisk();

	outb(0x1F2, 1);		// count = 1
	outb(0x1F3, offset);
	outb(0x1F4, offset >> 8);
	outb(0x1F5, offset >> 16);
	outb(0x1F6, (offset >> 24) | 0xE0);
	outb(0x1F7, 0x20);	// cmd 0x20 - read sectors

	// wait for disk to be ready
	waitdisk();

	// read a sector
	insl(0x1F0, dst, SECTSIZE/4);
}
```

> 我们查看原码可以看出这个函数主要做了三件事，等待磁盘(waitdist)、输出扇区数目和地址信息到端口(out)、读取扇区信息(insl)

```assembly
// waitdisk:
 7c6a:   55                      push   %ebp
 7c6b:   ba f7 01 00 00          mov    $0x1f7,%edx
 7c70:   89 e5                   mov    %esp,%ebp
 7c72:   ec                      in     (%dx),%al
 7c73:   83 e0 c0                and    $0xffffffc0,%eax
 7c76:   3c 40                   cmp    $0x40,%al
 7c78:   75 f8                   jne    7c72 <waitdisk+0x8>
```



- At what point does the processor start executing 32-bit code? What exactly causes the switch from 16- to 32-bit mode?
  - 在.code32这段代码之后，开始正式执行32位的代码，也就是当汇编代码完成了**关闭中断、数组递增设置，堆栈初始化为0，使能A20、32位保护模式**之后。
  - 我认为是`lgdt    gdtdesc,movl    %cr0, %eax,orl     $CR0_PE_ON, %eax,movl    %eax, %cr0`这几条指令设置了cr0的第一位也就是保护模式使能位，使得他切换到32位的保护模式，因为他使能了CR0的允许启动保护模式
- What is the *last* instruction of the boot loader executed, and what is the *first* instruction of the kernel it just loaded?
  - 通过GDB我们可以看到boot loader调用的最后一条指令是`=> 0x7d61:	call   *0x10018`对应main.c中的`((void (*)(void)) (ELFHDR->e_entry))();`也就是进入kernel的入口函数。接着使用si我们可以看到内核加载的第一条指令是`=> 0x10000c:	movw   $0x1234,0x472`
- *Where* is the first instruction of the kernel?
  - kernel的第一条指令位于/kern/entry.S
  - 地址是`0x10000c`文件中
- How does the boot loader decide how many sectors it must read in order to fetch the entire kernel from disk? Where does it find this information?
  - 在boot loader加载了elf的文件头信息到内存之后，根据`ph = (struct Proghdr *) ((uint8_t *) ELFHDR + ELFHDR->e_phoff)`; 和`ph = (struct Proghdr *) ((uint8_t *) ELFHDR + ELFHDR->e_phoff)`;从ELF文件头获得了需要加载的大小
  - 根据对 main.c 的分析，显然是通过 ELF 文件头获取所有 program header table，每个 program header table 记录了三个重要信息用以描述段 (segment)：p_pa (物理内存地址)，p_memsz (所占内存大小)，p_offset (相对文件的偏移地址)。根据这三个信息，对每个段，从 p_offset 开始，读取 p_memsz 个 byte 的内容（需要根据扇区(sector)大小对齐），放入 p_pa 开始的内存中。

```cpp
void
waitdisk(void)
{
    7c6a:	55                   	push   %ebp

static inline uint8_t
inb(int port)
{
	uint8_t data;
	asm volatile("inb %w1,%0" : "=a" (data) : "d" (port));
    7c6b:	ba f7 01 00 00       	mov    $0x1f7,%edx
    7c70:	89 e5                	mov    %esp,%ebp
    7c72:	ec                   	in     (%dx),%al
	// wait for disk reaady
	while ((inb(0x1F7) & 0xC0) != 0x40)
    7c73:	83 e0 c0             	and    $0xffffffc0,%eax
    7c76:	3c 40                	cmp    $0x40,%al
    7c78:	75 f8                	jne    7c72 <waitdisk+0x8>
		/* do nothing */;
}
```

### Loading the kernel

#### **Exercise 4.	理解C指针** 

 Read about programming with pointers in C. The best reference for the C language is *The C Programming Language* by Brian Kernighan and Dennis Ritchie (known as 'K&R'). We recommend that students purchase this book (here is an [Amazon Link](http://www.amazon.com/C-Programming-Language-2nd/dp/0131103628/sr=8-1/qid=1157812738/ref=pd_bbs_1/104-1502762-1803102?ie=UTF8&s=books)) or find one of [MIT's 7 copies](http://library.mit.edu/F/AI9Y4SJ2L5ELEE2TAQUAAR44XV5RTTQHE47P9MKP5GQDLR9A8X-10422?func=item-global&doc_library=MIT01&doc_number=000355242&year=&volume=&sub_library=).

Read 5.1 (Pointers and Addresses) through 5.5 (Character Pointers and Functions) in K&R. Then download the code for [pointers.c](https://pdos.csail.mit.edu/6.828/2018/labs/lab1/pointers.c), run it, and make sure you understand where all of the printed values come from. In particular, make sure you understand where the pointer addresses in printed lines 1 and 6 come from, how all the values in printed lines 2 through 4 get there, and why the values printed in line 5 are seemingly corrupted.

```c
#include <stdio.h>
#include <stdlib.h>

void
f(void)
{
    int a[4];
    //a是一个int类型的数组，
    //1.a的值是指向这个【数组内存的地址【
    //2.&a是a存放的内存地址
    //3.*a是a指针指向内存地址中的内容
    int *b = malloc(16);
    int *c;
    int i;
    //下面这一行将会输出的代码是三个指针指向的不同的地址
    printf("1: a = %p, b = %p, c = %p\n", a, b, c);
    //将c指向和a同一个地址
    c = a;
    for (i = 0; i < 4; i++)
	a[i] = 100 + i;
    c[0] = 200;
    printf("2: a[0] = %d, a[1] = %d, a[2] = %d, a[3] = %d\n",
	   a[0], a[1], a[2], a[3]);

    // 下面主要是替换指针指向内存代表的地址中的内容，较为简单的三种形式
    // 按照如下模版都是一个意思：
    //c[1]=*(c+1)=1[c]
    c[1] = 300;
    *(c + 2) = 301;
    3[c] = 302;
    printf("3: a[0] = %d, a[1] = %d, a[2] = %d, a[3] = %d\n",
	   a[0], a[1], a[2], a[3]);

    c = c + 1;
    *c = 400;
    printf("4: a[0] = %d, a[1] = %d, a[2] = %d, a[3] = %d\n",
	   a[0], a[1], a[2], a[3]);

    c = (int *) ((char *) c + 1);
    
    *c = 500;
    printf("5: a[0] = %d, a[1] = %d, a[2] = %d, a[3] = %d\n",
	   a[0], a[1], a[2], a[3]);

    b = (int *) a + 1;
    c = (int *) ((char *) a + 1);
    printf("6: a = %p, b = %p, c = %p\n", a, b, c);
}

int
main(int ac, char **av)
{
    f();
    return 0;
}
```

> 需要注意linux的数据表示方法是小端法，所以在发生赋值时会有一点奇怪

为了能够充分理解/boot/main.c，我们应该直到ELF文件是什么。

当我们编译连接一个C程序时，编译器会将.c原码转换为.o的目标文件，这个目标文件包含着硬件所需要的二进制格式。接着连接器会将编译的目标文件转换为一个唯一的二进制映像(binary image)。他就是ELF format，Executable and Linkable Format

在这个实验中，我们可以将ELF的可执行文件理解为一个包含**装载信息的header**，紧接着是**几个程序段**（每一段都是位于固定空间连续的代码块或者数据）。boot loader并不会修改这些程序和数据，boot loader只负责将这些数据加载到内存并执行他们

<img src="../pic/Screen Shot 2021-09-14 at 12.36.07 AM.png" alt="Screen Shot 2021-09-14 at 12.36.07 AM" style="zoom:50%;" />

ELF格式的文件由一个**固定长度的头部**开始，紧随其后的是变长的Program header table，它显示了不同被装载的程序段。ELF头部被定义在inc/elf.h文件中。程序段中，我们感兴趣的是

- `.text`: 程序可执行的指令
- `.rodata`: 只读数据，例如一些字符串
- `.data`: 数据段有着初始化数据，例如全局数据`'int a = 5;'`

当链接器计算程序的内存布局时，他会为没有初始化的全局变量预留空间`int x;`内存中紧跟在.data之后的.bss段需要**取消初始化全局变量初始化为0.**因此我们并不需要在ELF文件中为.bss存储内容；instead，我们会让**linker记录.bss段的地址和大小**。.bss段中的loader和程序本身必须设置为0.

如果我们要查看kernel的所有部分的名称、大小、和链接地址:`objdump -h obj/kern/kernel`

这个命令的作用是将可执行文件或者目标文件反序列化

> -h 
>
> --section-headers 
>
> --headers 
>
> 显示目标文件各个section的头部摘要信息。

输出结果如下：

```shell
moocos@moocos-VirtualBox:~/6.828/lab$ objdump -h obj/kern/kernel

obj/kern/kernel:     file format elf32-i386

Sections:
Idx Name          Size      VMA       LMA       File off  Algn
  0 .text         00001917  f0100000  00100000  00001000  2**4
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
  1 .rodata       00000714  f0101920  00101920  00002920  2**5
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  2 .stab         00003889  f0102034  00102034  00003034  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  3 .stabstr      000018af  f01058bd  001058bd  000068bd  2**0
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  4 .data         0000a300  f0108000  00108000  00009000  2**12
                  CONTENTS, ALLOC, LOAD, DATA
  5 .bss          00000644  f0112300  00112300  00013300  2**5
                  ALLOC
  6 .comment      0000002b  00000000  00000000  00013300  2**0
                  CONTENTS, READONLY
```

> 这其中有许多的调试信息，但是并没有加载到内存中

我们把注意力放在`.text段`的`VMA`（link address）和`LMA`(load address)。

- **一个模块的装载地址是他将会被装载到内存中的具体的物理地址**
- **一个模块的链接地址这个模块在内存中将会被执行的虚拟地址。**linker以不同的方式编码link address为二进制代码。例如当一段代码需要一个全局变量的地址，但是如果这个全局变量并没有被链接到，那么这段二进制代码将会没有作用。（当然现在可以生成` position independent`的代码，不包含绝对地址，这在现代共享库中广泛使用，但是有一定的开销，所以我们不会在实验中使用）

**典型的**，一个程序的link and load的地址是相同的，例如我们查看**/obj/boot/boot.out**的反汇编信息，就可以看到他的.text段的装载和运行地址是一致的都是`0x00007c00`，因为此时还没有启动分页机制



```shell
moocos@moocos-VirtualBox:~/6.828/lab$ objdump -h obj/boot/boot.out

obj/boot/boot.out:     file format elf32-i386

Sections:
Idx Name          Size      VMA       LMA       File off  Algn
  0 .text         0000017c  00007c00  00007c00  00000074  2**2
                  CONTENTS, ALLOC, LOAD, CODE
  1 .eh_frame     000000b0  00007d7c  00007d7c  000001f0  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  2 .stab         000007b0  00000000  00000000  000002a0  2**2
                  CONTENTS, READONLY, DEBUGGING
  3 .stabstr      00000846  00000000  00000000  00000a50  2**0
                  CONTENTS, READONLY, DEBUGGING
  4 .comment      0000002b  00000000  00000000  00001296  2**0
                  CONTENTS, READONLY
```

boot loader会使用**ELF 程序的头来决定如何装载这些部分**。这个程序的**头部会明确ELF的哪些部分会装在进入到内存中，以及应该占用的目标地址**。我们可以使用`objdump -x obj/kern/kernel`来查看

> -x 显示所可用的头信息，包括符号表、重定位入口。-x 等价于-a -f -h -r -t 同时指定。

```
moocos@moocos-VirtualBox:~/6.828/lab$ objdump -x obj/kern/kernel

obj/kern/kernel:     file format elf32-i386
obj/kern/kernel
architecture: i386, flags 0x00000112:
EXEC_P, HAS_SYMS, D_PAGED
start address 0x0010000c

Program Header:
    LOAD off    0x00001000 vaddr 0xf0100000 paddr 0x00100000 align 2**12
         filesz 0x0000716c memsz 0x0000716c flags r-x
    LOAD off    0x00009000 vaddr 0xf0108000 paddr 0x00108000 align 2**12
         filesz 0x0000a300 memsz 0x0000a944 flags rw-
   STACK off    0x00000000 vaddr 0x00000000 paddr 0x00000000 align 2**4
         filesz 0x00000000 memsz 0x00000000 flags rwx

Sections:
略
SYMBOL TABLE:
略

```

我们查看Program Header下面的显示信息，ELF中需要被装载的代码都被标记了`LOAD`。接着每个头部的信息也被提供了例如虚拟地址`vaddr`物理地址`paddr`被装载区域的大小`memsz,filrsz`

> 根据Part1，我们可以看到操作系统真正的代码是被装载到BIOS之上的，而bootloader是被装载到low memroty这一段代码

接下来我们返回到/boot/main.c中，`ph->p_pa`就是程序头部包含段的物理地址。

BIOS将boot loader装载到0x7c00，boot loader也会在这里执行，所以0x7c00既是装载地址也是链接地址。具体的实现是将`-Ttext 0x7c00`选项传递给linker放在`/boot/Makefrag`文件中

**Exercise 5.** 

> Trace through the first few instructions of the boot loader again and identify the first instruction that would "break" or otherwise do the wrong thing if you were to get the boot loader's link address wrong. Then change the link address in `boot/Makefrag` to something wrong, run make clean, recompile the lab with make, and trace into the boot loader again to see what happens. Don't forget to change the link address back and make clean again afterward!

这里我们将/boot/Makefrag文档中的装载地址修改为了0x7c20，接下来使用gdb调试

```shell
(gdb) b *0x7c2d
Breakpoint 1 at 0x7c2d
(gdb) c
Continuing.
[   0:7c2d] => 0x7c2d:	ljmp   $0x8,$0x7c52

Breakpoint 1, 0x00007c2d in ?? ()
(gdb) si
[f000:e05b]    0xfe05b:	cmpl   $0x0,%cs:0x65a4
0x0000e05b in ?? ()
(gdb) info registers
eax            0x0	0
ecx            0x0	0
edx            0x663	1635
ebx            0x0	0
esp            0x0	0x0
ebp            0x0	0x0
esi            0x0	0
edi            0x0	0
eip            0xe05b	0xe05b
eflags         0x2	[ ]
cs             0xf000	61440
ss             0x0	0
ds             0x0	0
es             0x0	0
fs             0x0	0
gs             0x0	0
(gdb) si
[f000:e062]    0xfe062:	jne    0xfd2b9
0x0000e062 in ?? ()
(gdb) si
[f000:d2b9]    0xfd2b9:	cli
0x0000d2b9 in ?? ()
(gdb)
```

经过多次调试我们会发现真正出问题的代码发生在0x7c2e的这条跳转指令`ljmp   $0x8,$0x7c52`,跳转到这个指令后，程序会将`$0x0,%cs:0x65a4`这个地址和0比较，如果不等于零，我们会看到又重新回到了bootloader的最开始代码`cli`并且多次`continue`都会回到这个指令。所以我们可以推断是这里除了问题。

> 正确的装载地址

```shell
(gdb) b *0x7c00
Breakpoint 1 at 0x7c00
(gdb) b *0x7c2d
Breakpoint 2 at 0x7c2d
(gdb) c
Continuing.
[   0:7c00] => 0x7c00:	cli

Breakpoint 1, 0x00007c00 in ?? ()
(gdb) c
Continuing.
[   0:7c2d] => 0x7c2d:	ljmp   $0x8,$0x7c32

Breakpoint 2, 0x00007c2d in ?? ()
(gdb) si
The target architecture is assumed to be i386
=> 0x7c32:	mov    $0x10,%ax
0x00007c32 in ?? ()
(gdb) si
=> 0x7c36:	mov    %eax,%ds
```

对比`ljmp   $0x8,$0x7c32`(right)，正好这也是进入保护模式的代码。, `ljmp   $0x8,$0x7c52`(wrong)，不能进入保护模式

结合我们设置的boot loader装在地址`0x7c00`(right), `0x7c20`(wrong)。

我们就可以看到在发生跳转指令的时候如果错误的装在地址多偏移了0x20，也就是我们在连接的时候错误设置的。我们查看错误的GDB信息他跳转作物，某种机制让他进行判断`cmpl   $0x0,%cs:0x65a4`并从新进入bootloader的代码尝试重新进入保护模式。

> 根本原因：BIOS会将boot loader装载到0x7c00但是如果我们错误的将boot loader链接，那么就会出现我们刚刚看到的无法加载内核的情况

和bootloader不同，kernel的装载地址和链接地址不同，kernel会告诉boot loader装载kernel到比较低的地址，但是又希望kernel能够执行在比较高的地址

除了相关段的信息，ELF的头也会告诉另一个很重要的信息，那么就是`e_entry`。他是装载程序的**入口地址(entry address)**。这个区域包含着**程序中入口的链接地址**,也就是**程序在内存中将会被执行的地址**

我们可以使用`objdump -f file`指令来查看这个入口地址

```shell
moocos@moocos-VirtualBox:~/6.828/lab$ objdump -f obj/kern/kernel

obj/kern/kernel:     file format elf32-i386
architecture: i386, flags 0x00000112:
EXEC_P, HAS_SYMS, D_PAGED
start address 0x0010000c
```

我们可以看到入口地址为0x0010000c

**Exercise 6.** 

> We can examine memory using GDB's x command. The [GDB manual](https://sourceware.org/gdb/current/onlinedocs/gdb/Memory.html) has full details, but for now, it is enough to know that the command x/*N*x *ADDR* prints *`N`* words of memory at *`ADDR`*. (Note that both '`x`'s in the command are lowercase.) *Warning*: The size of a word is not a universal standard. In GNU assembly, a word is two bytes (the 'w' in xorw, which stands for word, means 2 bytes).
>
> Reset the machine (exit QEMU/GDB and start them again). Examine the 8 words of memory at 0x00100000 at the point the BIOS enters the boot loader, and then again at the point the boot loader enters the kernel. Why are they different? What is there at the second breakpoint? (You do not really need to use QEMU to answer this question. Just think.)

我们利用上面的相关信息可以了解到kernel最终是被装载到物理地址0x00100000，所以我们可以使用下面的指令来查看是否正确。

```shell
(gdb) b *0x7c00
Breakpoint 1 at 0x7c00
(gdb) c
Continuing.
#设置断点在boot loader的起始地址0x7c00,使用x/8i 0x100000查看kernel是否被装在
[   0:7c00] => 0x7c00:	cli
(gdb) x/8x 0x00100000#kernel并没有被装载，没有指令
0x100000:	0x00000000	0x00000000	0x00000000	0x00000000
0x100010:	0x00000000	0x00000000	0x00000000	0x00000000
(gdb) b *0x0010000c
Breakpoint 2 at 0x10000c#装载后
(gdb) c
Continuing.
The target architecture is assumed to be i386
=> 0x10000c:	movw   $0x1234,0x472
#有指令存在
Breakpoint 2, 0x0010000c in ?? ()
(gdb) x/8x 0x0010000c
0x10000c:	0x7205c766	0x34000004	0x0000b812	0x220f0011
0x10001c:	0xc0200fd8	0x0100010d	0xc0220f80	0x10002fb8
(gdb)
```

## Part 3: The Kernel

### Using virtual memory to work around position dependence

和boot loader 不同，kernel的装载和链接地址是十分复杂的，我们可以在kern/kernel.ld中看到

```c
SECTIONS
{
	/* Link the kernel at this address: "." means the current address */
	. = 0xF0100000;//link address

	/* AT(...) gives the load address of this section, which tells
	   the boot loader where to load the kernel in physical memory */
	.text : AT(0x100000) {//load address
		*(.text .stub .text.* .gnu.linkonce.t.*)
	}
```

操作系统的**内核通常会被链接和运行在比较高的地址空间`0xf0100000`**，目的就是为了**预留出比较低的处理器的虚拟地址空间让用户程序运行**

但是许多机器的内存可能不够大，**也就是说在`0xf0100000`这个地址可能并不存在具体对应的物理单元**，所以我们不能指望着把kernel的代码存在`0xf0100000`。所以我们这里会使用处理的内存管理的硬件(mmu)将虚拟地址`0xf0100000`（**链接地址**，也就是我们希望kernel运行的地址）映射到物理地址`0x00100000`(**装在地址**，bootloader将kernel真实存放的位置，**在BIOS的正上方**)

这样的话我们就可以让kernel的虚拟地址在十分高的位置，便以留给用户程序空间足够的虚拟地址空间，但是他将会被装载到PC1MB的物理地址(正好在BIOS之上)。但是这种方式需要PC的内存至少需要一些MB来让`0x00100000`这个地址有效。

> 为什么我们JOS只能使用256MB的内存？
>
> 因为在lab2中，我们会将虚拟地址`0xf0000000`~`0xffffffff`地址映射到物理地址`0x00000000`~`0x0fffffff`。我们知道32为的寻址空间为4GB，这里我们只使用了`0x00000000`~`0x0fffffff`所以4GB/16 = 256MB

现阶段我们仅仅映射4MB的内存，对于现在来说已经足够了。我们实现这个方式使用手写的、静态初始化的**页目录(page directory)**和**页表(page table)**代码(`kern/entrypgdir.c`)。对于我们现阶段，我们不必要理解它的原理，只需要知道它的功能。

在`kern/entry.S`设置`CR0_PG` flag之前，操作系统直接使用物理地址(严格地说应该是线性地址，但是boot.S文件设置了从线性地址到物理地址的映射，并且我们不会改变它)一点我们将`CR0_PG`使能,那么内存指向的就是虚拟地址(通过虚拟内存硬件转换到物理内存)

`entry_pgdir`会将虚拟地址`0xf0000000` through` 0xf0400000` to物理地址`0x00000000` through`0x00400000`(4MB),同样也会把虚拟地址`0x00000000` through `0x00400000` to 物理地址映射到 `0x00000000` through` 0x00400000`。

> 为什么还需要把虚拟地址的0~4MB这段区间映射到物理地址的0~4MB呢？，我们可以在entry.S文件中找到答案。他对于entry.S中的一些指令有用但是之后会使用到

```assembly
# '_start' specifies the ELF entry point.  Since we haven't set up
# virtual memory when the bootloader enters this code, we need the
# bootloader to jump to the *physical* address of the entry point.
.globl		_start
_start = RELOC(entry)#将代码跳转到入口地址的物理地址

.globl entry
```

由于我们并没有设置中断处理机制，所以任何不属于这两个范围的虚拟地址将会造成硬件异常，他会是QEMU停止或者不断重启。

**Exercise 7.** 

> Use QEMU and GDB to trace into the JOS kernel and stop at the `movl %eax, %cr0`. Examine memory at 0x00100000 and at 0xf0100000. Now, single step over that instruction using the stepi GDB command. Again, examine memory at 0x00100000 and at 0xf0100000. Make sure you understand what just happened.
>
> What is the first instruction *after* the new mapping is established that would fail to work properly if the mapping weren't in place? Comment out the `movl %eax, %cr0` in `kern/entry.S`, trace into it, and see if you were right.

> kern/kernel.asm

```assembly
	# Load the physical address of entry_pgdir into cr3.  entry_pgdir
	# is defined in entrypgdir.c.
	movl	$(RELOC(entry_pgdir)), %eax
f0100015:	b8 00 00 11 00       	mov    $0x110000,%eax
	movl	%eax, %cr3
f010001a:	0f 22 d8             	mov    %eax,%cr3
	# Turn on paging.
	movl	%cr0, %eax
f010001d:	0f 20 c0             	mov    %cr0,%eax
	orl	$(CR0_PE|CR0_PG|CR0_WP), %eax
f0100020:	0d 01 00 01 80       	or     $0x80010001,%eax
	movl	%eax, %cr0
f0100025:	0f 22 c0             	mov    %eax,%cr0
```

上面这段代码的前两个指令适用于设置CR3寄存器吗，作用是设置页目录的基地址(入口地址)是多少，而entry_pgdir可以在entrypgdir.c这个文件中找到

> CR3是页目录基址寄存器，保存页目录表的物理地址，页目录表总是放在以4K字节为单位的存储器边界上，因此，它的地址的低12位总为0，不起作用，即使写上内容，也不会被理会。

这里我们通过查看`kenel.asm`的反汇编码可以看到他其实和`entry.S`的代码是一样的。所以我们可以看到在进入内核后操作系统会做什么操作。而上面这段代码中就是`or     $0x80010001,%eax`**启动分页机制**。这里启动分页机制依赖于CPU的`CR0`寄存器,具体参考x86的CR0寄存器各个位的作用，这里我们只列出具体设置的位。 

`$0x80010001`相当于设置了CR0寄存器的0为16位以及31位。

> 参考80386 mamual 4.1.3 Control Register
>
> CR0是系统内的控制寄存器之一。控制寄存器是一些特殊的寄存器，它们可以控制CPU的一些重要特性。
> CR0的第0位是保护允许位PE(Protedted Enable)，用于启动保护模式，如果PE位置1，则保护模式启动，如果PE=0，则在实模式下运行。
> CR0的第16位是写保护未即WP位(486系列之后)，只要将这一位置0就可以禁用写保护，置1则可将其恢复。
> CR0的第31位是分页允许位PG(Paging Enable)，当设置该位时即开启了分页机制；当复位时则禁止分页机制，此时所有线性地址等同于物理地址。在开启这个标志之前必须已经或者同时开启PE标志。即若要启用分页机制，那么PE和PG标志都要置位。

参考上面GDB的显示信息我们可以看到`0x00100000`和`0xf0100000`中指向的内容是相同的。并且映射后的第一条指令时`mov    $0xf010002f,%eax`、`jmp    *%eax`

```assembly
=> 0x10002a:	jmp    *%eax
0x0010002a in ?? ()
(gdb) si
=> 0xf010002c <relocated>:	add    %al,(%eax)
relocated () at kern/entry.S:74
74		movl	$0x0,%ebp			# nuke frame pointer
(gdb) si
Remote connection closed
```

如果将`movl	%eax, %cr0`这个代码注释掉，没有映射机制，那么`jmp    *%eax`这个指令就会造成kernel崩溃，因为此时的`0xf010002f`地址中还没有完成对应的映射。GDB调试信息如下：

```assembly
=> 0x10002a:	jmp    *%eax
0x0010002a in ?? ()
(gdb) si
=> 0xf010002c <relocated>:	add    %al,(%eax)
relocated () at kern/entry.S:74
74		movl	$0x0,%ebp			# nuke frame pointer
(gdb) si
Remote connection closed
```

> QEMU的报错
>
> qemu: fatal: Trying to execute code outside RAM or ROM at 0xf010002c说明没有了映射机制。

### Formatted Printing to the Console

阅读 `kern/printf.c`, `lib/printfmt.c`, `kern/console.c`原码, 理解他们之间的关系。

`kern/console.c`

这个文件中主要是一些和硬件相关的设置。

第一部分是串口的的相关函数，处理数据、中断、输出文字、初始化

第二部分是并行端口输出的相关设置

第三部分是VGA/CGA的相关设置，初始化、输出（颜色输出）等等

第四部分是键盘的输入代码，其中存在一些map，用于获取键盘中输入的代码

第五部分就是console的一些代码，这里就会管理console的输入缓存，并且当中断时从键盘或者端口获取信息

第六部分是比较高层的I/O端口，用于readline以及cprintf

`lib/printfmt.c`

主要就是用于格式化输出的一些函数

`kern/printf.c`

三个打印的函数

**Exercise 8.**

>  We have omitted a small fragment of code - the code necessary to print octal numbers using patterns of the form "%o". Find and fill in this code fragment.

1. Explain the interface between `printf.c` and `console.c`. Specifically, what function does `console.c` export? How is this function used by `printf.c`?

   printf.c中使用了console.c中的`cputchar()`接口

   ```c
   //printf.c
   static void
   putch(int ch, int *cnt)
   {
   	cputchar(ch);
   	*cnt++;
   }
   //console.c
   void
   cputchar(int c)
   {
   	cons_putc(c);
   }
   static void
   cons_putc(int c)
   {
   	serial_putc(c);
   	lpt_putc(c);
   	cga_putc(c);
   }
   static void
   serial_putc(int c)
   {
   	int i;
   
   	for (i = 0;
   	     !(inb(COM1 + COM_LSR) & COM_LSR_TXRDY) && i < 12800;
          //判断line status register的转台是否可读，如果不可读就停止
   	     i++)
   		delay();
   
   	outb(COM1 + COM_TX, c);
     //如果可读的话，就将硬件中的缓存读取到c
     //COM1：计算机的端口用于使能外围设备，例如鼠标
   }
   ```

   

2. > Explain the following from
   >
   > console.c:
   >

   ```c
   1      if (crt_pos >= CRT_SIZE) {
   2              int i;
   3              memmove(crt_buf, crt_buf + CRT_COLS, (CRT_SIZE - CRT_COLS) * sizeof(uint16_t));
   4              for (i = CRT_SIZE - CRT_COLS; i < CRT_SIZE; i++)
   5                      crt_buf[i] = 0x0700 | ' ';
   6              crt_pos -= CRT_COLS;
   7      }
   ```

   判断crt现在的位置是否超过了crt的容量。使用memmove函数(类似memcpoy但是更安全)实现删除最上面一行的代码。最后一行的缓存清空，腾出可以继续让用户输入的空间。

3. > For the following questions you might wish to consult the notes for Lecture 2. These notes cover GCC's calling convention on the x86.
   >
   > Trace the execution of the following code step-by-step:
   >
   
   ```c
   int x = 1, y = 3, z = 4;
   cprintf("x %d, y %x, z %d\n", x, y, z);
   ```
   
   - In the call to `cprintf()`, to what does `fmt` point? To what does `ap` point?
     - 通过GDB的调制我们知道fmtz指针指向的是字符串`"x %d, y %x, z %d\n"`的地址，即`0xf0101c2e`
     - ap会指向第二个参数`0xf010ff04`
   
   **补充1: 如何通过ebp获取参数和局部变量**
   
   |        存储内容         |    内存地址     |
   | :---------------------: | :-------------: |
   |       第 n 个参数       | ebp + 4 * (n+1) |
   |        返回地址         |     ebp + 4     |
   | 上一级函数 ebp (旧 ebp) |       ebp       |
   |     第 m 个局部变量     |   ebp - 4 * m   |
   
   > 源代码对应的汇编函数
   
   ```c
   prologue：
   	 |0xf010079f <mon_backtrace>      push   %bp                                                  │
      │0xf01007a0 <mon_backtrace+1>    mov    %sp,%bp                                              │
   body：
     	//开辟存放【变量】的栈空间,依次存放z,y,x这三个变量到栈中，注意栈中内存的东西可能并没有清楚，所以会有一些奇奇怪怪的东西                                        
      │0xf01007a2 <mon_backtrace+3>    sub    $0x18,%esp                                           │
     >│0xf01007a5 <mon_backtrace+6>    movl   $0x4,0xc(%esp)                                       │
      │0xf01007ad <mon_backtrace+14>   movl   $0x3,0x8(%esp)                                       │
      │0xf01007b5 <mon_backtrace+22>   movl   $0x1,0x4(%esp)                                       │
     	//这个具体存的什么我也不知道,经过刚刚的查看直到这个地址是【字符串的地址】，也就是我们第一个参数的地址
      │0xf01007bd <mon_backtrace+30>   movl   $0xf0101c2e,(%esp)                                  │
     	//调用cprintf()进入到对应函数
      │0xf01007c4 <mon_backtrace+37>   call   0xf0100958 <cprintf>                                 │
      │0xf01007c9 <mon_backtrace+42>   mov    $0x0,%eax                                            │
   epilogue：
      │0xf01007ce <mon_backtrace+47>   leave                                                       │
      │0xf01007cf <mon_backtrace+48>   ret
   ```

> cprintf对应的汇编代码

```c
prologue： 
		| 0xf0100958 <cprintf>            push   %ebp                                                 │
   │0xf0100959 <cprintf+1>          mov    %esp,%ebp                                            │
body：
  //这一段代码是实际上是在实现va_start(ap,fmt)
  	//开辟栈空间
   │0xf010095b <cprintf+3>          sub    $0x18,%esp                                           │
  	//lea： lead effective address将addr的值存放到某个寄存器中
  	//根据GDB显示0xc(%ebp)地址是0xf010ff04，对应的值是0x00000001，也就是我们上一个函数中的第一个参数的地址
   │0xf010095e <cprintf+6>          lea    0xc(%ebp),%eax                                       │
  	//将第一个参数x的【地址】保存在栈
   │0xf0100961 <cprintf+9>          mov    %eax,0x4(%esp)                                       │
  	//caller的栈顶保存在保存在callee的栈顶
   │0xf0100965 <cprintf+13>         mov    0x8(%ebp),%eax                                       │
   │0xf0100968 <cprintf+16>         mov    %eax,(%esp)                                          │
  
  
   │0xf010096b <cprintf+19>         call   0xf0100925 <vcprintf>                                │
opelogue：
   │0xf0100970 <cprintf+24>         leave                                                       │
   │0xf0100971 <cprintf+25>         ret
```

**没有被初始化的变量并不会压入栈中，初始化了的数据会被压入栈中**

> vcprintf()对应的汇编代码

```c
prologue
	 |0xf0100925 <vcprintf>           push   %ebp                                                 │
   │0xf0100926 <vcprintf+1>         mov    %esp,%ebp                                            │
body
  	//开辟十个栈空间
   │0xf0100928 <vcprintf+3>         sub    $0x28,%esp                                           │
  	//初始化栈中
  	//局部变量int cnt = 0,对战中还有两个局部变量分别是vprintfmt的入口地址和cnt的地址
   │0xf010092b <vcprintf+6>         movl   $0x0,-0xc(%ebp)                                      │
  	//将caller的第二个参数也就是ap指向的第二个参数的地址存放在callee的第四个参数的位置(具体表示是使用栈来表示，栈顶是一个函数的第一个参数)
   │0xf0100932 <vcprintf+13>        mov    0xc(%ebp),%eax                                       │
   │0xf0100935 <vcprintf+16>        mov    %eax,0xc(%esp)                                       │
  	//将caller的第一个参数也就是fmt的地址存放在callee的第三个参数的位置
   │0xf0100939 <vcprintf+20>        mov    0x8(%ebp),%eax                                       │
   │0xf010093c <vcprintf+23>        mov    %eax,0x8(%esp)                                       │
  	//将第二个局部变量放到callee的第二个参数为止，cnt地址
   │0xf0100940 <vcprintf+27>        lea    -0xc(%ebp),%eax                                      │
   │0xf0100943 <vcprintf+30>        mov    %eax,0x4(%esp)                                       │
  	//将putch()的入口地址放到第一个参数
  >│0xf0100947 <vcprintf+34>        movl   $0xf0100912,(%esp)                                   │
   │0xf010094e <vcprintf+41>        call   0xf0100d9e <vprintfmt>                               │
   │0xf0100953 <vcprintf+46>        mov    -0xc(%ebp),%eax                                      │
epilogue
   │0xf0100956 <vcprintf+49>        leave                                                       │
   │0xf0100957 <vcprintf+50>        ret
```



- > List (in order of execution) each call to `cons_putc`, `va_arg`, and `vcprintf`. For `cons_putc`, list its argument as well. For `va_arg`, list what `ap` points to before and after the call. For `vcprintf` list the values of its two arguments.
  
  - 调用顺序`cprintf(const char *fmt, ...) ->  cnt = vcprintf(fmt, ap) -> vprintfmt((void*)putch, &cnt, fmt, ap) -> putch(ch, putdat) -> cputchar(ch) -> cons_putc(c) -> serial_putc(c)      lpt_putc(c);                                                 cga_putc(c);`
    - `cprintf(const char *fmt, ...)`：用户调用接口
    - `vcprintf(fmt, ap)`:获得变量的可变参数链
    - `vprintfmt((void*)putch, &cnt, fmt, ap) `：处理字符串
    - `putch(ch, putdat)`：调用接口
    - `cputchar(ch)`命令行比较高级的接口
    - `cons_putc(c)`:调用更底层的输出端口选择`serial_putc(c);lpt_putc(c);cga_putc(c);`,
      - `ch`则是从fmt中取出的每一个字符
  - 调用`va_arg()`后ap会指向传入参数的后一个参数的地址
  - fmt在调用完成后会空，ap为空，因为调用了`va_end()`函数

#### 函数调用栈的使用规则

一个函数由一个栈来维护，而这个栈由两个寄存器`ebp`以及`esp`来表明。栈底地址由`ebp`来存储，栈顶地址由`esp`来存储。

这两个指针所包含的范围叫做**栈帧(stack frame)**。

当一个函数caller调用另一个函数callee时，caller的栈顶会存储`eip`的地址，然后跳转到callee的函数地址。开始执行callee的代码前，编译器会首先将caller的栈底指针`ebp`压入栈，将`ebp`和`esp`指向同一个围着，接着使用`sub指令`开一一端栈空间。

根据我们现在观察到的。栈的底部通常会存放一些这个函数局部变量，**而栈的顶部则会存放一些下一个会被调用函数的参数列表的值**。并且可以通过`ebp`指针来访问上个函数中传入该函数的值。

1. > Run the following code.
   >
   > ```
   >     unsigned int i = 0x00646c72;
   >     cprintf("H%x Wo%s", 57616, &i);
   > ```
   >
   > What is the output? Explain how this output is arrived at in the step-by-step manner of the previous exercise.
   >
   > The output depends on that fact that the x86 is little-endian. If the x86 were instead big-endian what would you set `i` to in order to yield the same output? Would you need to change `57616` to a different value?

> #### output : He110 World

如果是大端法的话我会将i设置为`0x726c6400`，我不会修改`57616`这个值，因为整数的输出中和地址没有关系，它是使用%来一点一点输出整数的。

> 4. In the following code, what is going to be printed after
>
> ```
> 'y='
> ```
>
> ? (note: the answer is not a specific value.) Why does this happen?

因为va所指向的参数没有被指定，所以在调用`va_arg()`后ap会指向不明的地址导致输出错误

```
    cprintf("x=%d y=%d", 3);
```

5. Let's say that GCC changed its calling convention so that it pushed arguments on the stack in declaration order, so that the last argument is pushed last. How would you have to change `cprintf` or its interface so that it would still be possible to pass it a variable number of arguments?

### The Stack

在最后一个练习中我们会学习C语言如何使用栈在x86的平台上，在这个过程中我们会写一个新的monitor函数，打印stack的追踪情况，列出`IP(Instruction Potinter)`指向的具体的值

**Exercise 9.	了解栈的初始化** 

> Determine where the kernel initializes its stack, and exactly where in memory its stack is located. How does the kernel reserve space for its stack? And at which "end" of this reserved area is the stack pointer initialized to point to?

我们查看obj/kern/entry.asm文件可以看到在启动分页机制以后，代码会设置ip跳转到重定位后的应该执行的下一条代码。如下

```c
f010002f <relocated>:
relocated:
	# Clear the frame pointer register (EBP)
	# so that once we get into debugging C code,
	# stack backtraces will be terminated properly.
	movl	$0x0,%ebp			# nuke frame pointer
f010002f:	bd 00 00 00 00       	mov    $0x0,%ebp
	# Set the stack pointer
	movl	$(bootstacktop),%esp
f0100034:	bc 00 00 11 f0       	mov    $0xf0110000,%esp
	# now to C code
	call	i386_init
f0100039:	e8 5f 00 00 00       	call   f010009d <i386_init>
```

这些代码就在初始化我们之后将会使用到的堆栈信息。首先，他将我们的`栈帧指针%ebp`设置为`0x0`以便我们在之后的debug过程中栈会正确的终止。接着将我们的栈顶指针指向`$(bootstacktop):$0xf0110000`这个位置。这`0xf0110000`这个地址就是我们栈在我们虚拟内存中存在的地方。

kernel使用`sub`指令来开辟栈空间，例如进入我们kernel运行的第一个程序`i386_init()`这个函数他开辟栈空间就是使用`sub	$0x18,%esp`这里就位栈开辟了6个栈空间。

%esp会指向比较低的地址。

#### **Exercise 10.	熟悉汇编对栈的管理**

>  To become familiar with the C calling conventions on the x86, find the address of the `test_backtrace` function in `obj/kern/kernel.asm`, set a breakpoint there, and examine what happens each time it gets called after the kernel starts. How many 32-bit words does each recursive nesting level of `test_backtrace` push on the stack, and what are those words?
>
> Note that, for this exercise to work properly, you should be using the patched version of QEMU available on the [tools](https://pdos.csail.mit.edu/6.828/2018/tools.html) page or on Athena. Otherwise, you'll have to manually translate all breakpoint and memory addresses to linear addresses.

```c
	test_backtrace(5);
f01000de:	c7 04 24 05 00 00 00 	movl   $0x5,(%esp)
f01000e5:	e8 56 ff ff ff       	call   f0100040 <test_backtrace>
```

当调用一次test_backtrace()函数后，kernel会首先将上一个`%ebp`的指针压入栈中，然后设置新的栈帧指针`%ebp`，将他指向`栈顶%esp`。接着由GDB可知kernel继续为栈开辟了5个字的栈空间(`sub    $0x14,%esp`)，接着讲上一个函数中使用了的寄存器`%ebx`的值压入栈中。所以这个栈帧中总共有7字的空间。我们在GDB中使用x命令来查看这些值

```c
(gdb) x/7 没有备注的都是没有被初始化的值
0xf010ffc0:     0xf0101900(字符串地址)      0x00000005(上个函数中传入的值)      0x00000000      0x00010094
0xf010ffd0:     0x00010094      0x00010094(%ebx中的值)      0xf010fff8(上一个栈帧地址) 
```

```ruby
 +--------------------------------------------------------------+
 | next instruc |    str‘s addr  |    this x     |  don't know  |  
 +--------------+----------------+---------------+--------------+
 |	don't know  |  	 don't know  |    last ebx   |  last ebp    |   
 +------ -------------------------------------u------------------+
```

#### **Exercise 11.	自己动手实现backtrace**	 

> Implement the backtrace function as specified above. Use the same format as in the example, since otherwise the grading script will be confused. When you think you have it working right, run make grade to see if its output conforms to what our grading script expects, and fix it if it doesn't. *After* you have handed in your Lab 1 code, you are welcome to change the output format of the backtrace function any way you like.

根据了解了上一个联系中栈的一些调用习惯。我们了解到我们可以使用`%ebp`指针来一条一条的追溯上一个函数调用的是什么。

在`x86.c`的文件中提供了获取寄存器的一些函数，我们可以调用`read_ebp()`函数来读取ebp中的值。然后利用指针来访问上一个栈中的相关信息。

具体代码如下：

```c
int 
mon_backtrace(int argc, char **argv, struct Trapframe *tf)
{
	uint32_t ebp = read_ebp();
	uint32_t **ebp_ptr;
	cprintf("Stack backtrace:\n");
	while(ebp != 0) {
	ebp_ptr = (uint32_t *)ebp;
	cprintf("ebp %08x  eip %08x  args %08x %08x %08x %08x\n",ebp ,*(ebp + 1), *(ebp + 1), *(ebp + 3), *(ebp + 4), *(ebp + 5), *(ebp + 6));
	ebp = *ebp_pt;
  }
}
```

#### **Exercise 12.	使用stab完善backtrace函数** 

> Modify your stack backtrace function to display, for each `eip`, the function name, source file name, and line number corresponding to that `eip`.
>
> In `debuginfo_eip`, where do `__STAB_*` come from? This question has a long answer; to help you to discover the answer, here are some things you might want to do:
>
> - look in the file `kern/kernel.ld` for `__STAB_*`
> - run objdump -h obj/kern/kernel
> - run objdump -G obj/kern/kernel
> - run gcc -pipe -nostdinc -O2 -fno-builtin -I. -MD -Wall -Wno-format -DJOS_KERNEL -gstabs -c -S kern/init.c, and look at init.s.
> - see if the bootloader loads the symbol table in memory as part of loading the kernel binary
>
> Complete the implementation of `debuginfo_eip` by inserting the call to `stab_binsearch` to find the line number for an address.
>
> Add a `backtrace` command to the kernel monitor, and extend your implementation of `mon_backtrace` to call `debuginfo_eip` and print a line for each stack frame of the form:
>
> ```
> K> backtrace
> Stack backtrace:
>   ebp f010ff78  eip f01008ae  args 00000001 f010ff8c 00000000 f0110580 00000000
>          kern/monitor.c:143: monitor+106
>   ebp f010ffd8  eip f0100193  args 00000000 00001aac 00000660 00000000 00000000
>          kern/init.c:49: i386_init+59
>   ebp f010fff8  eip f010003d  args 00000000 00000000 0000ffff 10cf9a00 0000ffff
>          kern/entry.S:70: <unknown>+0
> K> 
> ```
>
> Each line gives the file name and line within that file of the stack frame's `eip`, followed by the name of the function and the offset of the `eip` from the first instruction of the function (e.g., `monitor+106` means the return `eip` is 106 bytes past the beginning of `monitor`).
>
> Be sure to print the file and function names on a separate line, to avoid confusing the grading script.
>
> Tip: printf format strings provide an easy, albeit obscure, way to print non-null-terminated strings like those in STABS tables. `printf("%.*s", length, string)` prints at most `length` characters of `string`. Take a look at the printf man page to find out why this works.
>
> You may find that some functions are missing from the backtrace. For example, you will probably see a call to `monitor()` but not to `runcmd()`. This is because the compiler in-lines some function calls. Other optimizations may cause you to see unexpected line numbers. If you get rid of the `-O2` from `GNUMakefile`, the backtraces may make more sense (but your kernel will run more slowly).

##### 了解stab

在开始练习12之前我们首先需要了解什么是**stab**（symbol table）符号表

stab实际上就是一种为debugger提供调试信息的一种格式。当我们在编译一个文件时，加入-g选项，在汇编文件中就会添加一些调试信息。

这些调试信息包含了原文件的很多特点，例如行号，种类、变量范围、函数名称、变量。

一共有三种不同的符号表运算符分别是.stabs(string),.stabn(number),.stbd(dot)，每一种符号表的大致格式如下：

```
.stabs "string",type,other,desc,value
.stabn type,other,desc,value
.stabd type,other,desc
.stabx "string",value,type,sdb-type
```

kernel中提供了一个函数`stab_binsearch(stabs, region_left, region_right, type, addr)`

某一些stab的种类是根据指令地址递增排序的。如果我们给出一个指令的地址addr，那么这个函数就能够找到一个某一个type的唯一包含这个地址的stab的入口。他是从[*region_left, *region_right]这个地址中开始搜索的。因此，为了找到一个N个stab的集合，我们需要将

`left = 0`

`right = N - 1`

搜索的过程中**修改**左右两个指针`region_left`, `region_right`来包含addr。`*region_left`指向包含这个地址的stab。`*region_right`指向下一个stab的前一个指针。

如果`*region_left > *region_right `那么这个addr就不包含任何匹配的stab.。



> 使用**objdump -h obj/kern/kernel**查看.stab以及.stabstr的虚拟、物理地址

```
2 .stab         000038c5  f0102078  00102078  00003078  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
3 .stabstr      000018e8  f010593d  0010593d  0000693d  2**0
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
```

汇编器会提供两种不同的部分.stab(包含一组固定长度的结构体，每一个stab都有一个结构体，以及一个section name), .stabstr(包含着被.stab中stab引用变长的字符串)

.stab和.stabs在虚拟地址空间紧跟在.text段之后，他们是一段连续的地址空间，可以根据引索来访问。这里JOS提供了一种搜索某个指令在哪个stab段引索的方法。`stab_binsearch(stab,region_left,region_right,type,addr)`函数。调用这个函数就能够搜索到某一种`type`的的stab包含addr的做引索和下一个不同地址的由引索。找到这个引索就可以获取stab中对应的某一些调试信息。

首先我们要做的就是补全`kern/debug.c`中寻找行的函数。阅读源码整个`debug_info()`函数的思想是首先在stab中寻找文件段、再搜索函数段最后搜索行段，并且在每次搜索的过程中给`info`添加相应信息。根据前面的一些函数我们在这里不难写出这样的函数。

```c
stab_binsearch(stabs, &lline, &rline, N_SLINE, addr);
    if (lline <= rline) {
        info->eip_line = stabs[lline].n_desc;
    } else {
        return -1;
    }
```

- 这里我们可以在inc/stab.h中找到和N_SLINE相关的描述
- 获取行号使用的格式 stabs[lline].n_desc，这个可以参考之前的获取函数地址等内容的代码。

> 这里需要注意判断的条件lline <= rline。因为lline = rline的话也可能可以搜索到对应的行号。

此后是添加命令，在 kern/monitor.c 的第27行：

```cpp
static struct Command commands[] = {
    { "help", "Display this list of commands", mon_help },
    { "kerninfo", "Display information about the kernel", mon_kerninfo },
    { "backtrace", "Display backtrace info", mon_backtrace },
};
```

最后是添加 backtrace 的输出信息，将 kern/monitor.c 的 mon_backtrace 函数改为：

```cpp
int
mon_backtrace(int argc, char **argv, struct Trapframe *tf)
{
    uint32_t ebp, *ptr_ebp;
    struct Eipdebuginfo info;
    ebp = read_ebp();
    cprintf("Stack backtrace:\n");
    while (ebp != 0) {
        ptr_ebp = (uint32_t *)ebp;
        cprintf("\tebp %x  eip %x  args %08x %08x %08x %08x %08x\n", ebp, ptr_ebp[1], ptr_ebp[2], ptr_ebp[3], ptr_ebp[4], ptr_ebp[5], ptr_ebp[6]);
        if (debuginfo_eip(ptr_ebp[1], &info) == 0) {
            uint32_t fn_offset = ptr_ebp[1] - info.eip_fn_addr;
            cprintf("\t\t%s:%d: %.*s+%d\n", info.eip_file, info.eip_line,info.eip_fn_namelen,  info.eip_fn_name, fn_offset);
        }
        ebp = *ptr_ebp;
    }
    return 0;
}
```

## Summary

### Part1

#### 总结

BIOS做的其实就是**检查、初始化一些底层设备**（时钟、GDTR寄存器、设置中断向量表）

但是作为PC启动的第一段程序，他最重要的作用就是将操作系统的boot loader从**磁盘导入内存，再将控制权交给操作系统**

BIOS在运行的最后会去**检测可以从当前系统的哪个设备中找到操作系统**，通常来说是我们的磁盘。

当BIOS确定了操作系统的位置，那么它就会把这个磁盘的第一个扇区，通常把它叫做启动区（boot sector）先加载到内存中，这个启动区中包括一个非常重要的程序--【**boot loader**】它会负责完成整个操作系统从磁盘导入内存的工作，以及一些其他的非常重要的配置工作。最后操作系统才会开始运行。



PC启动顺序：BIOS -> boot loader -> OS



> 当BIOS运行时，他会设置中断向量表，初始化一系列设备。初始化完成BIOS知道的一些设备后，他会去寻找一些bootable的设备（磁盘、软盘、CD_ROM）。最终到找到一个可启动的磁盘。BIOS会读取他的boot leader然后将控制权交个他。

### Part2

## 附录2. ELF详细介绍

- **ELF executable**
   可看作包含加载信息的文件头 (header) 以及一些程序段 (program section)。每个程序段是相邻的代码块或数据块，需要被加载到内存的特定位置。boot loader 不更改代码或数据，只是加载到内存并且执行。
- **ELF binary**
   以一个定长 ELF header 开头，然后是变长的 program header，包含了所有需要加载的程序段。
- **program section**
   只关注三个会用到的section。
  - .text
     程序的可执行指令。
  - .rodata
     只读数据。例如 C 编译器产生的 ASCII 字符串常量。
  - .data
     保存程序的初始数据。例如某个有初始值的全局变量 `int x = 5;`。



```ruby
~/OS/lab/obj/kern$ objdump -h kernel
kernel:     file format elf32-i386
Sections:
Idx Name          Size      VMA       LMA       File off  Algn
  0 .text         00001871  f0100000  00100000  00001000  2**4
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
  1 .rodata       00000714  f0101880  00101880  00002880  2**5
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  2 .stab         000038d1  f0101f94  00101f94  00002f94  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  3 .stabstr      000018bb  f0105865  00105865  00006865  2**0
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  4 .data         0000a300  f0108000  00108000  00009000  2**12
                  CONTENTS, ALLOC, LOAD, DATA
  5 .bss          00000644  f0112300  00112300  00013300  2**5
                  ALLOC
  6 .comment      00000034  00000000  00000000  00013300  2**0
                  CONTENTS, READONLY
```

重点关注的是 .text 部分的 VMA (link address) 和 LMA (load address)。link address 是开始执行该 section 的内存地址。而 load address 则顾名思义，是加载该 section 的内存地址。一般而言这两者是相同的。
 boot loader 利用 ELF program header 来决定如何加载 section，而 program header 指定应该读取 ELF 对象的哪个部分进内存，以及应该放在哪里。

