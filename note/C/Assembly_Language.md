# 1 Instruction

## 1.1 Number System

## 1.2 Computer Organazation

### 1.2.1 Memory

计算机内存中的基本单位是byte，每个byte都有自己的地址

### 1.2.2 The CPU

每种CPU有自己的机器语言

时钟用于控制指令的同步

### 1.2.3 The 80x86 family of CPUs

- 8088，8086
  - 只有实模式（real mode）。在这种模式下，任何程序都可以访问任何地址，即使是别的程序的地址，这种方法会十分的不安全，调试也会变得十分麻烦
  - 程序分段存储，每段不能超多64KB
- 80286
  - 新增16位的保护模式，它能够确保程序不会访问到别的地方
  - 程序分段，每段不能超过64KB
- 80386
  - 扩展许多寄存器为32bit的。新增两个寄存器FS，GS
  - 新增32位的保护博士
  - 程序分段，但是每段可达4GB

### 1.2.4  8086 16-bits register

#### general purpose register -- AX,BX,CX,DX

每个寄存器可被分为两个八位寄存器AX->(AH,AL)high，low

- 作用：数据移动，算数指令

#### index register --  SI,DI

他们不能被分为两个8位寄存器

- 作用：他们主要用作指针，当然也可以用作通用寄存器能做的数据移动以及算数运算

#### BP，SP

指向**机器语言栈(Machine Language Stack)**

#### Segment Register  --  CS,DS,SS,ES

解释了那段程序位于程序的那一部分中

- CS:Code Segment
- DS:Data Segment
- SS:Stack Segment
- ES:Extra Segment

#### Instruction Pointer  --  IP(PC)

和CS寄存器一起使用用于跟踪**下一条将被CPU执行的指令的地址**

#### FLAGS

用于存储**上一条指令的重要结果信息**

### 1.2.5 8086 32-bits register

为了向后兼容性，使用EAX来代表32位的寄存器E代表Extended. **AX是EAX的低16位寄存器，不能够直接访问高16位寄存器**

**段寄存器任然是16位**，新增了两个段寄存器  --  FS，GS。他们没有具体的意义，他们就是临时的段寄存器（就像ES）一样



word这个词的一种定义就是数据寄存器（data register）的大小

### 1.2.6 Real Mode

实模式中内存限制在1MB中(0x00000~0xFFFFF)很显然，这么一个地址需要20位。然是16位寄存器是不能够存储20位的。所以Intel使用了**两个16位寄存器来决定一个地址**

> - 为什么限制在1MB？
>
> 实模式是早期CPU，比如8088处理器的工作模式，这类处理器由于【只有20为地址线】【16为数据线】所以它们只能访问1MB的内存空间。
>
> 【16位数据总线】：也就是说明，总共能够表示的数据由2^16=64KB，也就是一个页的大小

- selector（0~15）：**必须存储在段寄存器(Segment Register)中**
- offset（16~31）

> 公式：16*selector  +  offset

##### 分段的缺点

- 一个单独的selector能够引索的地址只有64K
- 每个字节没有唯一的地址，

### 1.2.7 16-bits Protected Mode

在**80286中的16位保护模式**的selector和在实模式中的解释**完全不同**。

【实模式】中selector作用是段的基地址，直接指向具体的某个物理地址

而在【保护模式】中，它的作用是指向【descriptor table】描述附表的一个引索。

两种模式下，程序都被分为不同的段。实模式中这些段都具有固定的位置，selector就指向了不同段的开始地址。而在保护模式中，这些段并不是固定的物理地址空间，事实上他们可能并不在内存中。

**保护模式**使用了一种叫做**”虚拟内存“ -- virtual memory**的技术。

> 虚拟地址（逻辑地址）：段选择符  +  段内偏移

#### 分段机制

<img src="../pic/Screen Shot 2021-09-12 at 10.29.32 PM.png" alt="Screen Shot 2021-09-12 at 10.29.32 PM" style="zoom:50%;" />

这张图就显示了**逻辑地址（段选择符 + 段内偏移地址）**到线性地址的转换，如果没有分页管理的话，这个线性地址就是我们最终的物理地址

- 段选择符16位：不直接指向段，而是指向描述附表的某一个具体地址

段选择符结构：

<img src="../pic/Screen Shot 2021-09-12 at 10.33.09 PM.png" alt="Screen Shot 2021-09-12 at 10.33.09 PM" style="zoom:50%;" />

​	TI用来指明全局描述符表GDT还是局部描述符表LDT，RPL表示请求特权级。

​	索引值为13位，所以从这里看出，在保护模式下最多可以表示2^13=8192个段描述符。

​	TI又分GDT和LDT（如图3所示），所以一共可以表示8192*2=16384个段描述符，每个段描述符可以指定一个具体的段信息，所以一共可以表示16384个段。	

​	段内偏移地址为16位值，所以一个段最大可达1MB，这样16384*1MB，这就是所谓的最大寻址能力，也即逻辑地址/虚拟地址

- 基本思想：将正在用的代码与数据放在内存中，其他暂时没有使用的代码和数据存放在磁盘中，直到他们再次被使用。在保护模式中，这些段**随着需要而在内存和磁盘中来回移动**。当一个被使用的段被从磁盘移动到内存中时，他和上次地址可能并不相同，所有这一切的地址转换都是由OS隐形地完成的，而程序并不需要因为虚拟内存而修改不同的地址。

保护模式中，每个段都被赋予了一个**入口(entry)**存放在**描述表(descriptor table)**中，这个entry中存放着所有需要的信息

- 是否在内存中？如果在，存放在哪里？是否可access？

**段的入口地址的引索就是selector的值，存放在段寄存器中**

唯一的问题就在于他的段大小限制在64K

### 1.2.8 32-bits Protected Mode

和32位的区别

- 大小变大了
- 每个段能够被分为更小的单元叫做页(Pages) --  4K。而现在的虚拟内存就作用在页之上，而不是段上面了。这也就意味着只有段的一部分位于内存中。

#### 分页机制

> 分页机制将物理内存分为很多和固定大小的页面，大小为4KB，页面的0~4KB的地址由【线性地址】低12位表示  --  offset。
>
> 线性地址的高10位用于指定页目录中的位置，也就是说可以选择1024个目录项，每个目录项的大小为4字节(32位)所以页目录为4KB。
>
> 每个目录项中的高20位用以查更页表在物理内存中的页面（其余位用于表示状态），每个页表含1024个页表项，每个页表项也是四字节，这样一页表也是1024 * 4B = 4KB。
>
> 所以一个页目录可以查找1024个页表，每个页表为4KB，所以总共可以查找的页面大小为1024 * 4KB = 4MB大。
>
> 最后每个表表项的高20位用以定位物理地址空间中的某个页基地址，此地址再加上线性地址空间的偏移值就是最后物理内存空间单元。目录项和页表项结构如图8所示。

### 1.2.9 Interrupts

计算机硬件提供了一种处理需要相应外部事件的机制叫做”中断“

中断由一种叫做**中断处理器(interrupt handler)**的东西处理。中端处理器是一种处理中断的流程(routine)。计算机中的每一种中断被赋予了一个独一无二的整数值

物理内存的开头存在着**中断向量表(table of interrupt vector)**存放着中断处理器的地址。而每种中断对应的那个**数字本质上就是中断向量表的引索**

中断分为**外中断**和**内终端**

- 外中断由CPU外部引起，例如各种外设
- 内部中断在CPU内部引起，无论是**错误**（error -> trap）还是**中断指令**(interrupt instruction -> software interrupt)

DOS使用这些中断来实现API

## 1.3 Assembly Language

### 1.3.1 Machine Language

每条机器语言中的指令都有自己独一无二的操作码(operation code)他的大小不一定。opcode通常在指令的最开头。许多指令也会包含数据和地址。他十分难以理解，所以我们有汇编语言

### 1.3.2 Assembly Language

汇编语言以文本存储。每一条汇编语言都对应唯一的一条机器指令。

- 汇编器(Assembler)：将汇编语言转换为机器语言
- 编译器(Compiler)：将高级语言转换为汇编语言

### 1.3.3 Instruction operands

机器指令代码会有不同的数量和种类的操作符。

但是可以整体分为以下几种

- register:直接指向CPU寄存器
- memory:指向在内存中的数据
- immediate:在大妈中列出的固定的值，他不会在数据段，而是在指令自己本省
- implied:不会显示出现。例如加一指令

### 1.3.4 Basic Instruction

#### MOV

```assembly
mov dest, src
mov eax, 3	;store 3 into EAX register(3 is immediate operand)
mov bx, ax	;store the value of AX into the BX register
```

两个都不是内存中的操作数

【注意】：不同大小的寄存器不能相互赋值：例如将AX的值不能够赋给BL

#### ADD,SUB,INC,DEC

```assembly
add eax, 4	;eax = eax + 4
add al, ah	;al = al + ah

sub bx, 10	;bx = bx - 10
sub ebx, edi;ebx = ebx - edi

inc ecx	;ecx++
dec dl	;dl--
```

### 1.3.5 Directives

指令汇编器的某种东西

他们用作告诉汇编器做某些事，或者告诉汇编器某些事

他们不会被翻译为机器语言，他的作用是

- 定义常量
- 定义将数据存入内存
- 将内存分为段
- 分情况包含源码
- 包含其他文件

和他C语言的#十分类似，它使用的是%

#### The equ directive

"equ"运算符用于定义符号，符号被命名为常量在以后的汇编语言中使用

```assembly
symbol equ value
```

symbol之后不能被重新定义

#### The % directive

```assembly
%define SIZE 100
	mov eax, SIZE
```

宏定义能够被重新定义

并且比常量更简单

#### Data directive

data directive用于数据段，用于为内存定义空间。

- 仅仅为数据定义空间  --  “RESX”

  | Unit        | Letter |
  | ----------- | ------ |
  | byte        | B      |
  | word        | W      |
  | double word | D      |
  | quad word   | Q      |
  | ten bytes   | T      |

  X能够用上面的Letter替换，来决定对象将会被存储的空间大小

- 定义空间并初始化值  --  “DX”

```assembly
L1	db	0	;L1标志的字节初始化为0
L2	resb	1;1 uninialized byte
L3	db	"A";byte initialized to ASCII code for A(65)
L4	db	0,1,2,3;defines 4 bytes
L5	db "w","o","r","d";0定义了C string "word"
L6 db 'word';和上面的效果一样

;对于很大一片空间可以使用TIMES directives

L7 times 100 db 0;相当于“db 0 ”100次
L8 resw 100;预留空间100words
L9 dd 100h;

;我们可以使用标志来引用代码中的数据
mov al,[L1];将L1的字节复制到AL
mov eax,L1;EAX = 在L1这个字节的地址
mov al,[L9];将L9的第一个字节赋值给al寄存器，这说明了汇编器的重要特性就是他并不会跟踪标志所指向的数据种类大小
```

他们是连续存储的，也就是说L2会紧跟着L1存储的空间之后

如果**不检查的话会使得指针使用错误**。这样的话汇编器会比C有更多的错误

```assembly
mov [L6],1;store 1 at L6
mov dword [L6], 1；
```

第一个指令并没有指明操作符的大小，所以我们需要加一个大小指定符号来确定

- DWORD
- BYTE
- QWORD
- TWORD

## 1.4 Creating a Program

### 1.4.1 First Program

```assembly
;file: first.asm
;This program asks for two integers as input and 
;prints out thier sum
;
;To create executable using djgpp:
;nasm -f coff first.asm
;gcc -o first first.o driver.c asm_io.o

%include "asm_io.inc"
;
;initialized data is put in the .data segment
;
segment .data
;
;these labels reger to strings used for output
;
prompt1 db  "Enter a number: ", 0       ;don't forger null terminator
prompt2 db  "Enter another number: ", 0
outmsg1 db  "You entered ", 0
outmsg2 db  " and ", 0
outmsg3 db  ", the sum of these is ", 0

;
;uninitialized data is put the .bss segment
;
segment .bss;block started by symbol
;
;These labels refer to double words used to store the inputs
;
input1  resd  1
input2  resd  1

;
;code is put in the .text segment
;
segment .text ;code segment
        global  _asm_main
_asm_main:
  enter 0,0             ;setup routine
  pusha
  mov eax, prompt1      ;print out prompt
  call print_string

  call read_int         ;read integer
  mov [input1], eax     ;store into input1

  mov eax, prompte2     ;print out prompt
  call print_string

  call read_int         ;read integer
  mov [intput2], eax    ;store into input2
  
  mov eax, [input1]     ;eax = dword at input1
  add eax, [input2]     ;eax += dword at input2
  mov ebx, eax          ;ebx = eax

  dump_regs 1           ;print out register values
  dump_mem  2,outmsg1,1 ;print out memory
;
;next print out result message as series of steps
  mov eax, outmsg1
  call print_string     ;print out first message
  mov eax, [input1]
  call print)string     ;print out input1
  mov eax, outmsg2
  call print_string
  mov eax, [input2]
  call print_string
  mov eax, outmsg3
  call print_string
  mov eax, ebx
  call print_int
  call print_nl; print new line

  popa
  mov eax, 0;return back to C
  leave
  ret

```

- **.data**段用于存储初始化了的数据
- 没有被初始化的数据应该放在.bss段（block started by the symbols）
- .text段用于存放指令
  - 需要注意主函数前有“_",这是C语言调用的传统，当C和汇编交互式，这个十分重要
  - C的符号会被加上_被C编译器
- global指示符用于告诉汇编器_asm_amin这个函数是全局的f

如果想在linux下运行，我们需要将37，38行的下划线删除，因为linux使用的是ELF（executable and linkable format）文件格式

### 1.4.2 Compiler dependencies

我们在此程序中主要使用nasm汇编器来汇编汇编代码生成机器语言。

但是因为不同的机器对应不同的机器语言，所以我们需要在汇编是加入不同的参数来产生不同机器可以识别的机器语言。因为我实在Ubantu上运行的，所以按照书上写的我使用的命令是：

```shell
first.asm
	nasm -f elf first.asm
asm_io.asm
	nasm -f elf -d ELF_TYPE first.asm
duiver.c
	gcc -c driver.c
first
	gcc -o first first.o driver.o asm_io.o
```

但是如果这么编译会出现出现错误

```shell

/usr/bin/ld: i386:x86-64 architecture of input file `driver.o' is incompatible with i386 output
/usr/bin/ld: driver.o: file class ELFCLASS64 incompatible with ELFCLASS32
/usr/bin/ld: final link failed: File in wrong format
collect2: error: ld returned 1 exit status

/usr/bin/ld: i386 architecture of input file `asm_io.o' is incompatible with i386:x86-64 output
/usr/bin/ld: i386 architecture of input file `driver.o' is incompatible with i386:x86-64 output
/usr/bin/ld: i386 architecture of input file `first.o' is incompatible with i386:x86-64 output
collect2: error: ld returned 1 exit status
```

在网上查找原因后发现，是因为UBANTU系统默认的是64位，所以我们高版本的GCC生成的是64位机器代码，而nasm只能够生成32位的机器代码。所以在连接时会发生输入输出不匹配的问题。这里的解决办法就是在命令行中加入-m32参数，让GCC生成与汇编器匹配的32位机器指令。

还可能会出现如下的错误：

```shell
asm_io.o: In function `read_int':
asm_io.asm:(.text+0x10): undefined reference to `_scanf'
asm_io.o: In function `print_int':
asm_io.asm:(.text+0x2a): undefined reference to `_printf'
asm_io.o: In function `print_string':
asm_io.asm:(.text+0x41): undefined reference to `_printf'
asm_io.o: In function `read_char':
asm_io.asm:(.text+0x52): undefined reference to `_getchar'
asm_io.o: In function `print_char':
asm_io.asm:(.text+0x68): undefined reference to `_putchar'
asm_io.o: In function `print_nl':
asm_io.asm:(.text+0x7a): undefined reference to `_putchar'
asm_io.o: In function `push_of':
asm_io.asm:(.text+0x149): undefined reference to `_printf'
asm_io.o: In function `sub_dump_stack':
asm_io.asm:(.text+0x16c): undefined reference to `_printf'
asm_io.o: In function `stack_line_loop':
asm_io.asm:(.text+0x198): undefined reference to `_printf'
asm_io.o: In function `sub_dump_mem':
asm_io.asm:(.text+0x1bf): undefined reference to `_printf'
asm_io.o: In function `mem_outer_loop':
asm_io.asm:(.text+0x1d8): undefined reference to `_printf'
asm_io.o:asm_io.asm:(.text+0x1ed): more undefined references to `_printf' follow
collect2: error: ld returned 1 exit status
```

出现这个问题的原因是asm_io.o没有找到对应的函数，在windows系统中汇编语言中引用的C程序需要加上下划线如下：_printf。但是在linux中不能加入下划线，所以在汇编是需要加入-d -ELF 选项来编译生成适合linux的机器语言

最终指令如下：

```shell
moocos@moocos-VirtualBox:~/linux-ex$ nasm -f elf -d ELF_TYPE asm_io.asm
moocos@moocos-VirtualBox:~/linux-ex$ nasm -f elf first.asm
moocos@moocos-VirtualBox:~/linux-ex$ gcc -c driver.c -m32
moocos@moocos-VirtualBox:~/linux-ex$ gcc -o first first.o driver.o asm_io.o -m32
moocos@moocos-VirtualBox:~/linux-ex$
```

### 1.4.6 Understanding an assembly listing file

To produce a listing file, with the hex codes output from NASM displayed on the left of the original sources, use the `-l` option to give a listing file name, for example:

```
nasm -f elf myfile.asm -l myfile.lst
```

在汇编时加入-l指令就可以生成展开文件，我们就可以看到汇编器为我们生成的一写代码。部分代码如下：

```assembly
113 0000002D A1[00000000]                    mov     eax, [input1]     ; eax = dword at input1
114 00000032 0305[04000000]                  add     eax, [input2]     ; eax += dword at input2
115 00000038 89C3                            mov     ebx, eax          ; ebx = eax
```

第一行是行号，第二行是数据再某一段的偏移量，第三行诗十六进制存入的机器语言的值。

例如113行标签Input1的值在连接之前我们都不能够确定，所以汇编器默认给他赋值为0.但是汇编器任然能够计算出mov指令的机器语言之也就是我们看到的A1。所以在[]之间的数值不能够计算就暂时设置为0（因为input1实在bss段的开头部分，但是并不意味着最终他就在bss段的开头）

例如像115行这样的代码，汇编器能够计算出对应命令的机器语言，就直接显示为89C3.

#### Big and Little Endian Representation

如果我们仔细观察，我们会发现，标签Input2的位移明明是00000004但是在内存中却写的是04000000。这是因为不同的处理器存储数据的顺序不同。大端法是自然存储的，例如一个两字大小的数据00000004，在大端法的存储下就是00 00 00 04，而在intel这种小端法的存放下他的存放就是04 00 00 00.

> 两个字也就是4个字节，一个字节需要两位16进制的树来表示，所以有8位。

- 在网络与不同计算机通信时需要注意这个问题
- 当从内存中读取多个字节但是写回单个字节是，需要注意大小端

对于数组没有区别，但是对于数组的单个元素是由区别的。

## 1.5 Skeleton File

```assembly
;
; file: skel.asm
; This file is a skeleton that can be used to start assembly programs.
%include "asm_io.inc"
segment .data
;
; initialized data is put in the data segment here
;
segment .bss
;
; uninitialized data is put in the bss segment
;
segment .text
        global  asm_main
asm_main:
        enter   0,0               ; setup routine
        pusha
;
; code is put in the text segment. Do not modify the code before
; or after this comment.
;
        popa
        mov     eax, 0            ; return back to C
        leave                     
        ret
```

这是作者建议我们使用的一种汇编语言开始的文件框架。

首先包含头文件“asm_io.inc”

接着是数据段 --  .data

用于存储初始化的数据

接着是bss段用于存放没有初始化的数据

.txt段用于写具体的代码

最终就是返回。