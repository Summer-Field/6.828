## 6.3 Segment Protection

段保护有着所有的保护机制

1. Type checking
2. Limit checking
3. Restriction of addressable domain
4. Restriction of procedure entry points
5. Restriction of instruction set

段是保护机制的基本单位(unit)，段描述符(descriptor)存储了这些保护参数。当selector被装载进段寄存器并被访问时，CPU会自动进行保护检查。

### 6.3.1 Descriptors Store Protection Parameters

<img src="/Users/natsuno/Desktop/fig6-1.gif" alt="fig6-1" style="zoom:80%;" />

上图中被标白部分就是和保护机制相关的段描述符。

这些保护参数是在软件创建描述符的时候被加入的。应用系统的编程人员并不需要关心这些保护参数

当程序将selector装载进段寄存器是，不仅会装载段的基地址，也会把相应的保护信息给装进去。每一个段寄存器都有一些位是不可见的(base ,limit, type, privilege level)所以连续的保护检察不会因影响时钟。

#### 6.3.1.1 Type Checking

TYPE域由两个作用

- 区分不同的描述符
- 确定该段的作用

除开为应用程序提供的代码段和数据段，80386也为操作系统、gate提供了专门的描述符表。我们需要注意不是所有的描述符都定义了段，例如gate描述符就有别的目的。我们会在之后讨论。

代码段和数据段的描述符包含了了一些特殊的位来定义这个段的作用

- The writable bit in a data-segment descriptor specifies whether instructions can write into the segment.

- The readable bit in an executable-segment descriptor specifies whether instructions are allowed to read from the segment (for example, to access constants that are stored with instructions). 

  - A readable, executable segment may be read in two ways:一个可读可执行的段可能被两种方式读取

  1. Via the CS register, by using a CS override prefix.通过CS寄存器，覆盖CS寄存器的前缀
  2. By loading a selector of the descriptor into a data-segment register (DS, ES, FS,or GS).将描述符的选择子装载进段寄存器

#### 6.3.1.3 Privilege Levels

**权限(privilege)这个概念通过将0/3这两个数字赋值给CPU能够识别的关键对象**。这两个值就是我们说的`privelege level`。

- 0表示最高权限级别
- 3代表最低权限级别

下面这三个被CPU识别的对象包含权限级别

- 描述符包含一个域叫做描述符权限级别(Descriptor Privilege Level)DPL
- 选择子包含一个域叫做请求权限级别(Requestes Privilege Level)RPL。RPL用于代表selector指向的这个过程的权限级别
- 内部的处理器记录着当前权限级别(Current Privilege Level)。通常来说CPL等于处理器正在执行段的DPL。CPU随着不同的DPL改变而改变

[Figure 6-2](https://pdos.csail.mit.edu/6.828/2018/readings/i386/s06_03.htm#fig6-2) shows how these levels of privilege can be interpreted as rings of protection. The center is for the segments containing the most critical software, usually the kernel of the operating system. Outer rings are for the segments of less critical software.

<img src="/Users/natsuno/Desktop/fig6-2.gif" alt="fig6-2" style="zoom:80%;" />

通常来说说并不需要把所有的level都使用进来，通常来说就使用一级(0)或者两级(0/3)

### 6.3.2 Restricting Access to Data

为了定位某个操作的地址，80386程序必须将数据段的selector装载进对应的数据段寄存器(DS, EX, FS, GS, SS)。当访问某个数据时，处理器会自动比较权限等级。这个比较发生的时间是当某个目标段的描述附的选择子被装载进数据段寄存器时。正如我们在上面看到的有三种不同的检查

1. The CPL (current privilege level).
2. The RPL (requestor's privilege level) of the selector used to specify the target segment.
3. The DPL of the descriptor of the target segment.

指令会装载数据段寄存器(接下来会用到的段)当且仅当目标段的DPL的数字比max{CPL, RPL}的数值大。总的来说，一个进程只能访问和他特权等级相同或者更小的数据

随着CPL的改变，一个程序的可寻址域会改变。When CPL is zero, data segments at all privilege levels are accessible; when CPL is one, only data segments at privilege levels one through three are accessible; when CPL is three, only data segments at privilege level three are accessible. This property of the 80386 can be used, for example, to prevent applications procedures from reading or changing tables of the operating system.

<img src="/Users/natsuno/Desktop/fig6-3.gif" alt="fig6-3" style="zoom: 80%;" />

#### 6.3.2.1 Accessing Data in Code Segments

Less common than the use of data segments is the use of code segments to store data. Code segments may legitimately hold constants; it is not possible to write to a segment described as a code segment. The following methods of accessing data in code segments are possible:Load a data-segment register with a selector of a nonconforming, readable, executable segment.Load a data-segment register with a selector of a conforming, readable, executable segment.Use a CS override prefix to read a readable, executable segment whose selector is already loaded in the CS register.The same rules as for access to data segments apply to case 1. Case 2 is always valid because the privilege level of a segment whose conforming bit is set is effectively the same as CPL regardless of its DPL. Case 3 always valid because the DPL of the code segment in CS is, by definition, equal to CPL.