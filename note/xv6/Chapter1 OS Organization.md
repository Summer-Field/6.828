# Chapter 1 Operating System Organization

###   Three requirements for OS

- #### Multiplexing

  - support several activities at once, **time-share** the resources of the computer among these processes

- #### Isolation

  - arrange for **isolation** between the process

- #### Interaction

  - possible for process to interact

> this book will only focus on the **monolithic kernel**（UNIX）

##  Abstracting physical resources

### Just implement system call with out API

#### advantage

- Application have its own library tailored to its need
- interact with HW directly to achieve high performance.

#### disadvantage

- With more than one procee, the application should well-behanved.  Inter process, using **cooperative time-sharing**, process trust each other.
- but for process which can't trust each other, it's better to have an **isolation.**

### Abstract resources

为了能够达到很强的隔离效果，阻止应用程序直接解除硬件，反而**抽象硬件到服务**中是十分有用的。

> 比较典型的例子就是**文件系统**，应用程序只能够通过open, close, read, write等操作系统提供的接口来访问文件，而不能直接向disk中读写。

文件系统向app提供路径，这样就可以让提供接口的操作系统来管理磁盘了。

#### 一些抽象的例子

- OS使用`exec()`决定将一个进程放在内存的哪里，如果内存不够甚至会放在disk中，而不是直接控制内存
- OS使用文件描述符fd来进行进程间通信。因为fd抹去了很多细节，并且提供了很好地交互手段。例如pipe的一端出问题失败了就会返回文件结束符，来告知另一端出问题了。

所以，操作系统的接口都是经过精心设计的，他给程序员提供了遍历和很强的隔离性。

## User mode, Kernel mode, System calls

#### 一个**强有力的隔离**需要应用**程序和操作系统有明确的界限**

- 一个进程失效不能够让OS崩溃
- Application 不能够修改OS的数据结构和指令

### 处理器为强有力的隔离给出了硬件支持

每个x86的处理器都有两个模式

- kernel mode：可以执行特权指令：读写磁盘或者其他I/O设备
- user mode：这个模式下如果执行特权指令，会切换到kernel mode，然后在kernel mode的软件会清理掉这个app

#### User space

一个app使用user-mode执行代码，就说她在**用户空间**中

#### Kernel space

软件运行在kernel-mode执行代码就说他在**内核空间**中

#### Kernel

运行在kernel-space的程序就是kernel

处理器提供了一个特殊的指令让处理器从user-mode到kernel-mode，并进入到kernel提供的入口。（x86的指令时INT）

让kernel来指定入口十分重要的，因为如果让app只到了这个入口地址，那么恶意软件就可以破坏内核。

## Kernel Organization

### Monolithic Kernel

**Monolithic Kernel**的意思就是**整个操作系统都在内核**中，所有系统调用的的实现都在内核模式中

在这种设计模式下，整个操作系统都有硬件特权。

#### 优点

- 不需要决定操作系统的哪一部分没有硬件特权
- 操作系统的不同部分可以相互合作

#### 缺点

- OS 不同部分的交互十分复杂。
  - 这就会导致系统开发人员出错，进而kernel出错，导致死机，需要重启计算机

### Microkernel

OS设计者将运行在kernel mode的代码量减少，将一部分**操作系统运行在user-mode**

下图就是一个Microkernel的设计：

<img src="/Users/natsuno/Downloads/IMG_0BB764AEC396-1.jpeg" alt="IMG_0BB764AEC396-1" style="zoom:50%;" />

这个如中，FS(File System)作为一个用户等级的进程在运行，操作系统运行为进程的服务叫做server。

为了让应用能够和File Server交互，kernel会提供**进程间交互的机制**(inter-process communication mechanism)，让一个进程向另一个进程发送信息。

在微内核中，内核的借口都是由十分低级的函数完成(starting app, sending message, accessing hardware)。这样的函数设计就能够让kernel足够的小，并且OS的大多数都在用户空间中运行。

### Real World

通常是两者的结合比如Linux

但是xv6是monolithic kernel

## Process Overview

xv6以及一些其他的Unix操作系统中，他们**隔离的单元就是进程**。

- 进程的抽象避免了一个进程检测，破坏另一个进程
- 避免了进程对内核的破坏

### 实现进程的机制

- user/kernel mode flag
- address space
- time-slicing of threads

### 进程为程序提供的抽象

- 独立的内存空间（或者说地址空间），别的进程不能够访问
  - xv6使用**页表**（由硬件实现）来为每个进程提供自己独立的地址空间
    - 页表将虚拟地址(x86指令操控的地址)映射为物理地址(处理器传递个内存的地址)
  - xv6为每个进程都提供了一个页表，来定义他们属于自己的地址空间
- 拥有自己的CPU来执行程序指令

如下图所示，地址空间包含用零开始的用户进程。首先是指令，其次是全局变量，接着是堆栈，最后是堆(for malloc)进程能够自有扩展

<img src="/Users/natsuno/Downloads/IMG_568451AE356A-1.jpeg" alt="IMG_568451AE356A-1" style="zoom:80%;" />

#### 内核映射

内核的指令以及数据也会被映射进进程的地址空间。

为了给用户空间流出足够多的用户内存，内核被映射在0x80100000之上。

#### 进程在OS中的实现

xv6内核为每一个进程都维护了很多状态片，他们都被放到了`struct proc`结构体中。最重要的几个结构体是这个进程的`页表`, `内核栈`, `运行状态`

每个进程都有一个执行的线程来执行进程的指令。线程能够被挂起，或者在之后被恢复。

为了在进程间透明的转换，kernel会挂起现在正在运行的一个进程，然后让另一个进程的线程来运行。

#### 进程栈

许多线程的状态(本地变量，函数返回地址)，被**存储在线程的栈之中**。

每个进程中都有两个栈

- user-stack
  - 当进程在执行用户指令时，他就在使用user-stack，他的kernel-stack就失控的
- kernel-stack
  - 当进程进入到内核之中(system call or interrupt)，内核的代码就运行在内核栈之中。
  - 当进程在内核中时，他user-stack中的数据任然是被维护着的，只是没有被使用。

一个进程的线程会反复在user-stack和kernel-stack中跳转

kernel stack是**独立**的（并且被**保存在用户代码**中），这样，即使user-stack被破坏了，kernel-stack任然可以继续执行

#### 系统调用

**系统调用**会让处理器切换到kernel-stack，提升硬件特权等级，执行kernel的指令

系统调用结束完成，内核就会跳转到用户空间，硬件就会降低他的硬件特权登记，继续执行user指令。

- p->state描述了进程是否被部署、是否准备运行，正在运行中，等待I/O，或者已经退出

- p->pgdir描述了一个进程的页表，xv6会让硬件来处理p->pgdir

## Code: the first address space

**提供强有力的隔离的第一步就是让kernel运行在他自己的地址空间中。**

当PC启动后，他首先使用BIOS（存储在母板上一个不易失的内存中）初始化自己的硬件设备，接着在disk中寻找可启动的第一个扇区，将他(bootloader)装载到地址为`0x7c00`内存中并执行他，将控制权交个操作系统。随着bootloader的执行他会初始化堆栈、段寄存器，接着设置ce0寄存器启动页机制，从是模式进入保护模式。接着讲kernel从disk中加载到内存中，并且将控制权交个kernel。bootloader是由两个原文件组成的

- 由16位和32位的汇编代码编写的汇编文件`bootasm.S`
- 以及C代买编写的C文件`bootmain.c`

bootloader 会将kernel装载到物理地址为`0x100000`中。为什么不直接将他装载到0x80100000的原因是可能并没有那么大的物理内存。而为什么不装在0x0是因为他之下的`0xa0000`到`0x100000`是一些I/O硬件设备的映射

为了让剩余的kernel可以正常运行，entry会把一个从`0x80000000`开始页目录大小的虚拟地址映射到从`0x0`开始的物理地址。通常会使用二级页表来实现。

entry 0 将虚拟地址0~4M映射到物理地址的0~4M。只是为需要执行低地址时的一写代码，但是他最后会被溢移除

Entry 512对应的基地址其实就是`0x80000000`对应的一个大的页表。他会将KERNBASE + 0x400000映射到蓄力地址0:0x400000。这个映射会被kernel所使用，他将kernel比较高的虚拟地址空间映射到比较低的物理空间中。他将kernel的代码限制在了4MB大小。

接下来我们返回到entry中，他会把pgdir的物理地址存储在`%CR3`寄存器中,**%CR3中存储的一定是物理地址**。将虚拟地址存入到CR3是没有意义的，因为分页硬件现在还不知道如何分页，他还没有页表。

entrypgdir指向的时一个比较高的地址空间中。为了使能分页机制，会将CE0寄存器中的CE0_PG置为1.

在使能PG后，processor任然执行在比较低的地址空间中，如果我们没有将entry0映射到低的物理地址，操作系统在执行使能指令后的下一条指令时会崩溃

现在会跳转到kernel的C代码中。并且在比较高的地址空间中运行

- 首先设置堆栈指针`%ESP`，这样即使低地址的映射消失了任然可以使用栈、
- 最后跳转到主函数中，这样做是有意义的，否则编译器会跳转到低地址的main中。main函数不能返回因为没有他的返回地址。现在kernel就在比较高的地址中运行了。

## Code: creatint the first process

现在我们来看看kernel如何创建一个用户等级的进程，并保证他们是完全隔离的。

在main函数初始化一些设备和子系统后，他会通过`userinit()`创建第一个进程。`userinit()`做的第一件事就是调用`allocproc()`在proctable中创建一个结构体`struct proc`来初始化进程状态的哪些部分，以便于kernel来执行。

每创建一个进程都会调用同`allocproc()`，然而`userinit()`仅仅会在第一个进程创建时执行。

#### 内核栈设置

`allocproc()`会扫描proc_table来找到哪些state是UNUSED的结构体。当他找到一个没有使用的slot时，`allocproc()`就会将他的状态设置为EMBRYO将他标志位已经使用，并给他分配一个**唯一的进程标识符pid**。接下来他就会为这个**进程的内核线程分配一个内核栈**。如果内存分配，state被设置为UNUSED，返回0给失败信号。

接下来就需要设置这个新的进程的kernel-stack。在`fork()`以及创建我们这第一个进程会调用函数`allocproc()`。

`allocproc()`函数会为一个新进程设置一个特殊的**内核栈**以及一组可以让他跳转到用户空间的内核寄存器。设置好的内核栈如下：

<img src="/Users/natsuno/Downloads/IMG_41C1E27D8708-1.jpeg" alt="IMG_41C1E27D8708-1" style="zoom:50%;" />

`alloc()`函数通过设置return program counter的值，这些值会让一个新进程的内核线程首先执行在`forkret`上执行然后是在`trapret`执行。内核线程会使用从p->context中复制过来的寄存器的值来完成这任务。因此将`p->context->eip`设置到`forkret`就会让内核线程执行`forkret`的开头部分代码。这个函数会返回到栈底。

上下文切换的代码正好将栈指针指向p->context的尾部。

`allocproc`会将`p->context`放在栈上`trapret`正好在他上面(`forkret`返回的地方)。`trapret`保存内核栈顶用户寄存器跳转到对应的process。

当我们调用fork时这个过程和创建第一个进程的过程是一样的，只是后者的进程会开始执行在地址为0的用户空间而不是fork的返回地址。

**从用户程序切换到kernel是通过系统调用、中断、异常的中断机制(interrupt mechanism)来实现的。**

**当一个进程在运行时，无论什么时候跳转到kernel中，xv6的trap代码都会保存用户寄存器到进程的内核栈之中**

当进程通过中断进入到kernel时`userinit`将寄存器的值写入到栈顶，以便返回到user-space时进程可以继续正常工作。这些值就是一个`strcut trapframe`他就会存储用户的寄存器值。现在内核栈就配置完成了。

第一个进程会执行一个小程序(initcode.S)。这个进程需要物理内存来保存这个程序，程序通用需要一个也表来管理他的用户空间。

`userinit()`调用`setupkvm`来创建页表。`setupkvm`和`userinit`这两个函数就会为进程创建一个地址空间。

## Code: Running the first process

在`userinit()`完成后，mpmain就会低啊用scheduler开始运行我们的进程。

scheduler会寻找runnable的进程，然后调用`swithcuvm()`函数来高速硬件切换页表。（所以这里我们就可以了解到，切换用户空间本质上就是通过修改寄存器中CR3寄存器中的值来切换用户空间。）switchuvm同时也会设置任务状态段`SEG_TSS`来告诉硬件在内核栈上执行系统调用、中断。

scheduler会将p->state设置为running，然后就会进行上下文切换。

**上下文并不是一个进程，而是特殊CPU上寄存器中的值**，所以调度器告诉switch保存现在硬件寄存器到per-cpu的内存中而不是某一个线程的上下文。

allocproc过去将p->context->eip指向forkret，所以ret开始执行`forkret()`。...

这是%eip = 0， %esp = 4096.这就是进程的虚拟地址空间。处理器的分页硬件将虚拟地址转化为为这个进程分配的物理地址。PTE_U告诉处理器这个页面是他可以使用的。

%CS寄存器低两位就是CPL，如果CPL = 3那么用户代码只能使用PTE_U被设置的页面，而不能修改%CR3这样的敏感寄存器。所以进程就只能使用他自己的内存。