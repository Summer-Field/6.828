# 

# 介绍

- 目标文件的三种类型
  - 可重定位文件：持有代码和数据来将目标文件链接成一个可执行文件或者共享目标文件
  - 可执行文件：持有适合执行的代码
  - 共享目标文件：持有适合为链接两个上下文的数据和代码。
    - 首先，连接编辑器可能会使用其他的**重定位或者共享目标文件**来处理它为另一个**目标文件**
    - 动态链接器会将他和可执行文件和别的共享文件融合在一起，形成一个进程映像

目标文件被汇编器和连接编辑器创造，目标文件就是为了直接在处理器上执行的二进制文件代表。那些需要别的虚拟机器的文件除外。

## 文件格式

目标文件可以分为两种类型：连接程序(building a program)、执行程序(running a program)。为了便捷性和高效性，这两种目标文件的格式是平行的，反映了不同的需求。图示如下

<img src="/Users/natsuno/Desktop/Screen Shot 2021-09-30 at 9.36.46 AM.png" alt="Screen Shot 2021-09-30 at 9.36.46 AM" style="zoom:80%;" />

首先我们来看文件最开头的`ELF Header`，他是整个文件组织的**地图**。对于连接的视角来说，Section有目标文件的一组信息：指令、数据、符号表、重新定文信息..

接着我么你来看`Program Header Table`，如果存在的话他就会告诉系统如何创建一个进程映像。用于创建一个进程映像的文件必须有`Program Header Table`重定位文件是不需要的。

`Section Header Table`：包含着描述文件的不同段。每个Section在Section Header Table中有一个入口，他会给这个Section一个名字、大小...用于连接的文件必须有Section Header，其他的目标文件可有可无。

> 除了ELF Header的位置是确定的以外其他的部分都不是确定的。

## 数据表示

。。。

## ELFHeader

一些目标文件的控制结构是可以增大的，因为ELF Header包含着他们具体的大小。如果目标文件的格式改变那么就可能会遇到控制结构比预想的大或者小。那么程序就可能因此忽略一些额外的信息。解决这些丢失信息的办法依赖于上下文，和extention被定义的具体描述

我们可以看到ELF HEADER的机构如下 

<img src="/Users/natsuno/Desktop/Screen Shot 2021-09-30 at 9.50.01 AM.png" alt="Screen Shot 2021-09-30 at 9.50.01 AM" style="zoom:80%;" />

- e_entry 
  - 这个成员给出了系统第一个跳转控制的虚拟地址，这样就可以开始进程。如果这个文件没有相关的入口，那么这个成员=0
- e_phoff 
  - 表示program header table的文件偏移量(in bytes)，如果没有program header table这个文件的大小为0
- e_shoff 
  - 表示文件section header table的偏移量(in bytes)，如果没有section header，0
- e_ehsize
  - ELF header 的大小(in bytes)
- e_phentsize 
  - 表示文件program header table的大小(in bytes)。所有的entry都应该是同样的大小
- e_phnum 
  - 表示program header table中entry的个数。因此`e_phnum`和`e_phsize`给煮了整个program header table的大小(in bytes)
- e_shentsize 同理
- e_shnum 同理
- e_shstrndx
  - 表示段名表的入口的index

