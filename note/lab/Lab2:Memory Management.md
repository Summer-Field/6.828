# Lab2:Memory Management

## Introduction

内存管理有两个部分组成

- physical memory allocator：kernel能够分配物理内存并且可以在之后释放他
  - 我们的allocator会作用在4096Bytes的单元上  --  页
  - 我们需要做的就是控制记录了物理页的数据结构来分配或者释放、以及有多少进程正在共享分配的页
  - 我们也需要写一些规则来分配释放内存的空闲页
- virtual memory：内核和应用程序使用的**虚拟地址被映射到物理地址**
  - 当指令使用内存或者请求页表的集合时，x86硬件 **内存管理单元（Memory Management Unit）**执行了映射这个活动
  - 我们会修改JOS来设置MMU的page table

### Getting started

我们需要获取