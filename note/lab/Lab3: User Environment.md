# Lab 3: User Environment

## Part A: User Environments and Exception Handling

`inc/env.h`文件中包含了用户环境的基本定义。快去读他！

```C
typedef int32_t envid_t;
// +1+---------------21-----------------+--------10--------+
// |0|          Uniqueifier             |   Environment    |
// | |                                  |      Index       |
// +------------------------------------+------------------+
//                                       \--- ENVX(eid) --/
```

第一个部分就是定义了一个32位的整数`envid` -- 环境变量。每个环境变量由三个变量组成。

- `0-9`：前十位是这个环境id的引索，我们可以通过`ENVX(eid)`宏来取得这个envid_t对应的引索。引索用于查找一个`envs[]`的数组
- `10-30`: `Uniqueifier`用于区分环境在不同时刻创建，但是他们共享同一个`Index`
- `31`: 所有真实存在的环境都是>0的，所以最高位的符号位通常被设置为0。如果某个`envid_t < 0`的话那么就会出错。比较特殊的是`envid_t == 0`他表示目前正在的环境中。

```c
#define LOG2NENV		10
#define NENV			(1 << LOG2NENV)//可以创建的环境个数
#define ENVX(envid)		((envid) & (NENV - 1))//用于取envid_t的index
```

```c
//结构体 Env 中env_status的值
enum {
	ENV_FREE = 0,
	ENV_DYING,
	ENV_RUNNABLE,
	ENV_RUNNING,
	ENV_NOT_RUNNABLE
};

// Special environment types
enum EnvType {
	ENV_TYPE_USER = 0,
};
```

最重要的数据结构`Env`：

```c
struct Env {
	struct Trapframe env_tf;	// Saved registers
	struct Env *env_link;		// Next free Env
	envid_t env_id;			// Unique environment identifier
	envid_t env_parent_id;		// env_id of this env's parent
	enum EnvType env_type;		// Indicates special system environments
	unsigned env_status;		// Status of the environment
	uint32_t env_runs;		// Number of times environment has run

	// Address space
	pde_t *env_pgdir;		// Kernel virtual address of page dir
};
```

kernel使用`sturct Env`这个数据结构来跟踪每个用户环境。在这个lab中我们仅仅会创建一个环境，但是我们需要设计JOS来支持多个用户环境来运行。lab4中就能够使用这个特性来`fork()`除一个新的用户环境。

我们可以在`kern/env.c`文件中看到三个可以和`struct Env`相关的全局变量

```c
struct Env *envs = NULL;		// All environments
struct Env *curenv = NULL;		// The current env
static struct Env *env_free_list;	// Free environment list
					// (linked by Env->env_link)
```

一旦JOS开始启动并且开始运行，`envs`就会指向一个`struct Env`的数组，他们就是整个系统中哦所有的`env`。在我们的设计中，JOS kerne最多能够同时支持`NENV`个正在运行的环境，尽管在某一时刻能够真正运行的环境会少很多。一旦一个环境被部署了，`envs`数组就会维护一个单独的`struct Env`给没有一个环境。

JOS会把所有的没有被激活的`struct Env`放在`env_free_list`中。这使得用户环境的的分配和释放变得简单。因为这样的话他们仅仅需要被从这个`env_free_list`中天健或移除

kernel会使用`curenv`指针指向现在这个时刻正在运行的环境。在启动的时候，在第一个环境被运行起来之前，curenv是被初始化设置为NULL的。

### Environment State

`struct Env`结构体被定义在`inc/env.h`文件中，如下（在以后的lab中会有更多的成员加入到这个结构体中）

```c
struct Env {
	struct Trapframe env_tf;	// Saved registers
	struct Env *env_link;		// Next free Env
	envid_t env_id;			// Unique environment identifier
	envid_t env_parent_id;		// env_id of this env's parent
	enum EnvType env_type;		// Indicates special system environments
	unsigned env_status;		// Status of the environment
	uint32_t env_runs;		// Number of times environment has run

	// Address space
	pde_t *env_pgdir;		// Kernel virtual address of page dir
};
```

以下是这个结构体中的一些成员描述

**env_tf**:

这个`struct Treapframe`被定义在`icn/trap.h`中，当环境没有运行时（当另一个进程在运行时），他为环境保存了寄存器的值。当kernel或者其他环境在运行时，kernel会保存这些寄存器的值。当从user-mode切换到kernel-mode时，kernel就会保存这些寄存器的值，这样这个环境就能够被放回到他被移除的地方了。

```c
struct Trapframe {
	struct PushRegs tf_regs;
	uint16_t tf_es;
	uint16_t tf_padding1;
	uint16_t tf_ds;
	uint16_t tf_padding2;
	uint32_t tf_trapno;
	/* below here defined by x86 hardware */
	uint32_t tf_err;
	uintptr_t tf_eip;
	uint16_t tf_cs;
	uint16_t tf_padding3;
	uint32_t tf_eflags;
	/* below here only when crossing rings, such as from user to kernel */
	uintptr_t tf_esp;
	uint16_t tf_ss;
	uint16_t tf_padding4;
} __attribute__((packed));
```

**env_link**:

这是一个指向`env_free_list`中的下一个`Env`。`env_free_list`指向空闲环境中的第一个。

**env_id**:

kernel会为正在使用这个`struct Env`的环境存放一个唯一的值（尤其是在`env`这个数组中使用）。当一个用户的环境结束后，kernel可以将同样的这个`struct Env`分配给不同的用户环境，但是新的用户环境会和前一个环境使用不同的`env_id`，尽管新的环境正在重新使用`envs`数组中同一个元素。

**env_parent_id**:

kernel会在这里保存创造了这个环境的那个环境的`env_id`在这里作为`env_parent_id`。这样的话环境和环境之间就能够形成一个"家族树"，这样做的原因是当环境需要对另一个环境做出某些事情时，可以做出更明智的决定。

**env_type**:

这个变量是用于区分不同的特殊环境。对于大多数环境来说，他都会被设置为`ENV_TYPE_USER`。我们在之后的lab中，会介绍更多的一些特殊系统服务环境的种类。

**env_status**:

这个变量可以有以下几个种类的值：

- `ENV_FREE`:

  表明这个`struct Env`还没有没激活，他还任然在`env_free_list`上

- `ENV_RUNNABLE`:

  表明这个`struct Env`等待在处理器上运行

- `ENV_RUNNING`:

  表明这个`sturct Env`就是现在正在运行的环境

- `ENV_NOT_RUNNABLE`:

  表明这个`struct Env`是现在正在具有活力的环境，只是他还没有准备好可以运行。例如他正在等待从另一个环境中的 **进程间通信(InterProcess communication)**

- `ENV_DYING`:

  表明这个`sturct Env`是一个僵尸环境。在下一次traps to kernel发生之前，僵尸环境不会被释放。我们在Lab4之前是不会使用到这个falg的

**env_pgdir**:

这个变量保存着这个环境的页目录的内核虚拟地址

正如和Unix的进程一样，JOS的换件将`线程`和`地址空间`这两个概念连接在了一起。

- 线程：最初就是被定义为被保存的寄存器，也就是(`env_tf`这个成员)
- 地址空间：地址空间则是被定义在一个页目录以及页表(b被`env_pgdir`这个变量指向)。

为了运行一个环境，kernel必须同时设置**被保存的寄存器**以及**适当的地址空间**

JOS中的`struct Env`和xv6中的`struct proc`是同一个东西。这两个结构体都在`struct TrapFrame`中保存着环境（进程）的用户模式寄存器状态。在JOS中，独立的环境并没有像xv6那样有他们自己的内核栈。JOS的内核只允许一次性只有一个环境是活跃的，所以JOS仅仅需要一个kernel stack。

### Allocating the Environments Array

在lab2中，我们使用`mem_init()`为`pages`数组分配了内存，他就是一个内核的表来跟做哪些页面是空闲的哪些不是。我们现在需要修改`mem_inti()`函数为`env`数组分配内存。

#### **Exercise 1.** 

> Modify `mem_init()` in `kern/pmap.c` to allocate and map the `envs` array. This array consists of exactly `NENV` instances of the `Env` structure allocated much like how you allocated the `pages` array. Also like the `pages` array, the memory backing `envs` should also be mapped user read-only at `UENVS` (defined in `inc/memlayout.h`) so user processes can read from this array.
>
> You should run your code and make sure `check_kern_pgdir()` succeeds.

这个联系和lab2中做的事情是相通的。主要是对`mem_init()`函数进行修改。这里我们就来梳理一下`mem_init()`函数做具体做了什么

1. 调用`i386_detect_memoty()`函数确定`npages`, `napages_basemem`
2. 调用`boot_alloc()`为内核页表分配内存空间, 并初始化为0
3. 调用`boot_alloc()`为物理页管理结构`pages`分配内存空间并初始化为0
4. 调用`boot_alloc()`为用户环境管理结构`envs`分配内存空间并初始化为0
5. 调用`page_init()`初始化页面管理结构`pages`
6. 调用`boot_map_region()`将虚拟地址`UPAGES`映射到存放了页面管理结构`pages`的物理地址中
7. 调用`boot_map_region()`将虚拟地址`UENVS`映射到存放了环境管理结构`envs`的物理地址中
8. 调用`boot_map_region()`将虚拟地址`KSTACKTOP - KSTKSIZE`映射到物理地址为`bootstack`的物理地址中，这个是内核栈的映射
9. 调用`boot_map_region()`将虚拟地址在`KERNBASE`之上的所有虚拟地址映射到物理地址为`0`的地址空间中，这个意思就是kernel可以访问并控制所有的物理地址
10. 重新设置CR3寄存器
11. 重新设置CR0寄存器

具体的函数实现和lab2中基本一样就不提供了

### Creating and Running Environments

现在我们需要编写`kern/env.c`文件来运行一个用户的环境。因为我们目前还没有 **File System** ，所以我们现在需要设置kernel来装载一个嵌入在kernel内部的静态的二进制印象。JOS将这个二进制文件使用ELF文件格式嵌入到JOS之中

Lab3的GNUmakefile文件生成了一部分二进制影响存放在`obj/user`文件夹中。如果我们现在去看一下`/kern/Makefrag`我们会注意到一些**magic**：gcc 将所有的**.o文件**都连接到了kernel的可执行文件中。**-b binary option**连接选项让这些文件被作为没有处理的不能被解释的二进制文件，而不是通常编译器-o所生成的文件。（对于linker来说，这些文件不必要全部都是ELF格式的，他们可以是文本文件、甚至是相片）。在重新编译文件后如果我们去看一些`/kern/kernel.sym`文件我们就可以看到linker神奇的生成了一些名字十分费解的符号例如：`_binary_obj_user_hello_start`, `_binary_obj_user_hello_end`, and `_binary_obj_user_hello_size`。链接器通过压碎二进制文件的名字来生成这些符号名称，这些符号为内核提供了一种进入嵌入式二进制文件的方法（本质上就是提供了这些二进制文件的入口地址）而且我们可以看到他们被放在了BASE_MRMORY之中。

在kern/init.文件中的`i386_init()`函数，我们会看到一些二进制文件在这之中运行，然而我们并没有设置好用户环境的编写。所以接下来我们要做的就是设置用户环境。

#### **Exercise 2.** 

> In the file `env.c`, finish coding the following functions:
>
> - `env_init()`
>
>   Initialize all of the `Env` structures in the `envs` array and add them to the `env_free_list`. Also calls `env_init_percpu`, which configures the segmentation hardware with separate segments for privilege level 0 (kernel) and privilege level 3 (user).
>
> - `env_setup_vm()`
>
>   Allocate a page directory for a new environment and initialize the kernel portion of the new environment's address space.
>
> - `region_alloc()`
>
>   Allocates and maps physical memory for an environment
>
> - `load_icode()`
>
>   You will need to parse an ELF binary image, much like the boot loader already does, and load its contents into the user address space of a new environment.
>
> - `env_create()`
>
>   Allocate an environment with `env_alloc` and call `load_icode` to load an ELF binary into it.
>
> - `env_run()`
>
>   Start a given environment running in user mode.
>
> As you write these functions, you might find the new cprintf verb `%e` useful -- it prints a description corresponding to an error code. For example,
>
> ```
> 	r = -E_NO_MEM;
> 	panic("env_alloc: %e", r);
> ```
>
> will panic with the message "env_alloc: out of memory".

首先我们看`kern/init.c`文件中的第一段代码描述了**GDT（Global Descriptor Table）**，他说道我们需要使用不同的段分别为kernel-mode以及user-mode设置GDT，虽然我们用不到他来作为内存映射，但是我们需要使用的的权限管理(privilege management)。具体的参考80386 手册。`80386 md文件中有我的翻译`

> 接下来我们来看一写接下来可能会使用到的函数
>
> envid2env(envid_t envid, struct Env **env_store, bool checkperm)：作用就是将envid转换为env指向的结构体，判断这个envid是否符合要求，然后将这个envid指向的结构体赋值给`env_store`中。返回0，表示成功转换，返回-E_BAD_ENV则表示失败

##### env_init()

作用：初始化我们的`envs`数组

但是这里要求`env_free_list`指向的第一个环境必须是0，所以这里需要反向遍历数组让`env_free_list`指向env[0];

```c
// Mark all environments in 'envs' as free, set their env_ids to 0,
// and insert them into the env_free_list.
// Make sure the environments are in the free list in the same order
// they are in the envs array (i.e., so that the first call to
// env_alloc() returns envs[0]).
void
env_init(void)
{
	for(int i = ENVS - 1; i >=0; i--) {
    envs[i]->env_id = 0;
    envs[i]->env_link = env_free_list;
    env_free_list = &envs[i];
    env[i]->env_status = ENV_FREE;
  }
	env_init_percpu();

}
```

##### env_setup_vm()

作用：初始化我们的env内核部分的虚拟内存。本质上就是设置我们的页目录以及对应的页表，来实现内存部分的地址空间内容和kernel是一致的。

这个函数一开始我的想法为`e->env_pgdir`重新分配页表。也就是每个页目录中的页表项指向一个全新的页表，然后这个页表和kernel的页表内容是一致的。但是查看网络答案后，发现其实完全没有必要重新创建新的页表。我们完全可以用kernel已经创造好的页表来存储到我们的页目录中。这样可以节约大量的时间。而这里最简单的办法就是直接将`kern_pgdir`内存中的东西copy到我们的`e->env_pgdir`之中。这样，环境的页目录中内核空间的内容就和kernel所设置的一样了，并且不会增加新的空间。唯一需要修改的就是`UVPT`这一部分的内容了。具体函数如下。

```c
// Initialize the kernel virtual memory layout for environment e.
// Allocate a page directory, set e->env_pgdir accordingly,
// and initialize the kernel portion of the new environment's address space.
// Do NOT (yet) map anything into the user portion
// of the environment's virtual address space.
//
// Returns 0 on success, < 0 on error.  Errors include:
//	-E_NO_MEM if page directory or table could not be allocated.
//
static int
env_setup_vm(struct Env *e)
{
	int i;
	struct PageInfo *p = NULL;

	// Allocate a page for the page directory
	if (!(p = page_alloc(ALLOC_ZERO)))
		return -E_NO_MEM;
e->env_pgdir = page2kva(p);
    memcpy(e->env_pgdir, kern_pgdir, PGSIZE); // use kern_pgdir as template 
    p->pp_ref++;
	// Now, set e->env_pgdir and initialize the page directory.
	//
	// Hint:
	//    - The VA space of all envs is identical above UTOP
	//	(except at UVPT, which we've set below).
	//	See inc/memlayout.h for permissions and layout.
	//	Can you use kern_pgdir as a template?  Hint: Yes.
	//	(Make sure you got the permissions right in Lab 2.)
	//    - The initial VA below UTOP is empty.
	//    - You do not need to make any more calls to page_alloc.
	//    - Note: In general, pp_ref is not maintained for
	//	physical pages mapped only above UTOP, but env_pgdir
	//	is an exception -- you need to increment env_pgdir's
	//	pp_ref for env_free to work correctly.
	//    - The functions in kern/pmap.h are handy.

	// LAB 3: Your code here.

	// UVPT maps the env's own page table read-only.
	// Permissions: kernel R, user R
	e->env_pgdir[PDX(UVPT)] = PADDR(e->env_pgdir) | PTE_P | PTE_U;

	return 0;
}
```

##### region_alloc()

这个函数为用户环境初始化二进制文件、堆栈以及处理器的一些标志位。这个函数仅仅会在内核初始化的时候调用，也就是在创建第一个用户进程之前。这个函数会从ELF二进制映像中将所有可加载的段装载到用户环境的内存中。在ELF文件的头部中说明了开始执行的虚拟地址。同时他会将在ELF文件中被标志在程序头部中被映射了但是没有present的段清零（bss.段）

这些操作和bootloader十分类似，处理bootloader需要从磁盘中读取文件。

最后这个函数会为函数的出实栈映射一个页来管理.

作用为环境`e`的虚拟地址位于`va`的内存分配长度为`len`到的内存，具体函数实现如下

```c
static void
region_alloc(struct Env *e, void *va, size_t len)
{
    // LAB 3: Your code here.
    // (But only if you need it for load_icode.)
    //
    // Hint: It is easier to use region_alloc if the caller can pass
    //   'va' and 'len' values that are not page-aligned.
    //   You should round va down, and round (va + len) up.
    //   (Watch out for corner-cases!)
    size_t pgnum = ROUNDUP(len, PGSIZE) / PGSIZE;//找到len需要的页面数
    uintptr_t va_start = ROUNDDOWN((uintptr_t)va, PGSIZE);//找到va对应的页的起始虚拟地址
    struct PageInfo *pginfo = NULL;
    cprintf("Allocate size: %d, Start from: %08x\n", len, va);
    for (size_t i=0; i<pgnum; i++) {
        pginfo = page_alloc(0);//分配页面
        if (! pginfo) {//分配失败
            int r = -E_NO_MEM;
            panic("region_alloc: %e" , r);
        }
      //映射
        int r = page_insert(e->env_pgdir, pginfo, (void *)va_start, PTE_W | PTE_U | PTE_P);
        if (r < 0) {
            panic("region_alloc: %e" , r);
        }
        cprintf("Va_start = %08x\n",va_start);
        va_start += PGSIZE;
    }
}
```

##### **load_icode()**

 这是本 exercise 最难的一个函数。作用是将 ELF 二进制文件读入内存，由于 JOS 暂时还没有自己的文件系统，实际就是从 *binary 这个内存地址读取。可以从` boot/main.c `中找到灵感。

> 了解ELF格式的文件
>
> 首先，ELF文件格式提供了两种视图，分别是链接视图和执行视图。
>  链接视图是以节（section）为单位，执行视图是以段（segment）为单位。链接视图就是在链接时用到的视图，而执行视图则是在执行时用到的视图。上图左侧的视角是从链接来看的，右侧的视角是执行来看的。可以看出，一个segment可以包含数个section。
>  本文关注执行，结构体Proghdr是用于描述段 (segment) 的 program header，可有多个。
>
> <img src="../pic/4482847-a5a265b2f69b39ad.webp" alt="4482847-a5a265b2f69b39ad" style="zoom:50%;" />

 大概需要做的事：

1. 根据 ELF header 得出 Programm header。
2. 遍历所有 Programm header，分配好内存，加载类型为 ELF_PROG_LOAD 的段。
3. 分配用户栈。

需要思考的问题：

1. 怎么切换页目录？
    lcr3([页目录物理地址]) 将地址加载到 cr3 寄存器。
2. 怎么更改函数入口？
    将 env->env_tf.tf_eip 设置为 elf->e_entry，等待之后的 env_pop_tf() 调用

```c
static void
load_icode(struct Env *e, uint8_t *binary)
{
    struct Proghdr *ph, *eph;
    struct Elf *elf = (struct Elf *)binary;
    if (elf->e_magic != ELF_MAGIC) {
        panic("load_icode: not an ELF file");
    }
    ph = (struct Proghdr *)(binary + elf->e_phoff);//指向某个program header
    eph = ph + elf->e_phnum;//指向最后一个elf header

    lcr3(PADDR(e->env_pgdir));
    for (; ph<eph; ph++) {
        if (ph->p_type == ELF_PROG_LOAD) {//判断这个头部是否可以加载
            if (ph->p_filesz > ph->p_memsz)
                panic("load_icode: file size is greater than memory size");
            region_alloc(e, (void *)ph->p_va, ph->p_memsz);//分配p_memsz的内存给e的p_va
            memcpy((void *)ph->p_va, binary + ph->p_offset, ph->p_filesz);//将ph对应的这个filesz大小的文件复制到p_va虚拟地址对应的内存
            memset((void *)ph->p_va + ph->p_filesz, 0, ph->p_memsz - ph->p_filesz);//将除了p_filesz大小的内存之外的内存清
        }
    }
    e->env_tf.tf_eip = elf->e_entry;
    // Now map one page for the program's initial stack
    // at virtual address USTACKTOP - PGSIZE.
    
    // LAB 3: Your code here.
    region_alloc(e, (void *) USTACKTOP-PGSIZE, PGSIZE);
    lcr3(PADDR(kern_pgdir));
}
```

##### **env_create()**

 作用是新建一个进程。调用已经写好的 env_alloc() 函数即可，之后更改类型并且利用 load_icode() 读取 ELF。

```cpp
void
env_create(uint8_t *binary, enum EnvType type)
{
    // LAB 3: Your code here.
    struct Env *e;
    int r = env_alloc(&e, 0);
    if (r<0) {
        panic("env_create: %e",r);
    }
    e->env_type = type;
    load_icode(e, binary);
}
```

##### **env_run()**

 启动某个进程。注释已经非常详细地说明了怎么做，主要说下 env_pop_tf() 这个函数。该函数的作用是将 struct Trapframe 中存储的寄存器状态 pop 到相应寄存器中。查看之前写的 load_icode() 函数中的 `e->env_tf.tf_eip = elf->e_entry` 这一段代码，经过` env_pop_tf() `之后，指令寄存器的值即设置到了可执行文件的入口。

```rust
void
env_run(struct Env *e)
{
    // Step 1: If this is a context switch (a new environment is running):
    //     1. Set the current environment (if any) back to
    //        ENV_RUNNABLE if it is ENV_RUNNING (think about
    //        what other states it can be in),
    //     2. Set 'curenv' to the new environment,
    //     3. Set its status to ENV_RUNNING,
    //     4. Update its 'env_runs' counter,
    //     5. Use lcr3() to switch to its address space.
    // Step 2: Use env_pop_tf() to restore the environment's
    //     registers and drop into user mode in the
    //     environment.

    // Hint: This function loads the new environment's state from
    //  e->env_tf.  Go back through the code you wrote above
    //  and make sure you have set the relevant parts of
    //  e->env_tf to sensible values.

    // LAB 3: Your code here.
    // panic("env_run not yet implemented");
    if (curenv && curenv->env_status == ENV_RUNNING) {
        curenv->env_status = ENV_RUNNABLE;
    }
    curenv = e;
    e->env_status = ENV_RUNNING;
    e->env_runs++;
    lcr3(PADDR(e->env_pgdir));
    
    env_pop_tf(&e->env_tf);
}
```

##### env_pop_tf()

```c
struct PushRegs {
	/* registers as pushed by pusha */
	uint32_t reg_edi;
	uint32_t reg_esi;
	uint32_t reg_ebp;
	uint32_t reg_oesp;		/* Useless */
	uint32_t reg_ebx;
	uint32_t reg_edx;
	uint32_t reg_ecx;
	uint32_t reg_eax;
} __attribute__((packed));

struct Trapframe {
	struct PushRegs tf_regs;
	uint16_t tf_es;
	uint16_t tf_padding1;
	uint16_t tf_ds;
	uint16_t tf_padding2;
	uint32_t tf_trapno;
	/* below here defined by x86 hardware */
	uint32_t tf_err;
	uintptr_t tf_eip;
	uint16_t tf_cs;
	uint16_t tf_padding3;
	uint32_t tf_eflags;
	/* below here only when crossing rings, such as from user to kernel */
	uintptr_t tf_esp;
	uint16_t tf_ss;
	uint16_t tf_padding4;
} __attribute__((packed));

void
env_pop_tf(struct Trapframe *tf)
{
	asm volatile(
		"\tmovl %0,%%esp\n"				//将%esp指向tf地址处
		"\tpopal\n"						//弹出Trapframe结构中的tf_regs值到通用寄存器
		"\tpopl %%es\n"					//弹出Trapframe结构中的tf_es值到%es寄存器
		"\tpopl %%ds\n"					//弹出Trapframe结构中的tf_ds值到%ds寄存器
		"\taddl $0x8,%%esp\n" /* skip tf_trapno and tf_errcode */
		"\tiret\n"						//中断返回指令，具体动作如下：从Trapframe结构中依次弹出tf_eip,tf_cs,tf_eflags,tf_esp,tf_ss到相应寄存器
		: : "g" (tf) : "memory");
	panic("iret failed");  /* mostly to placate the compiler */
}
```

首先我们看`PushRegs`结构



下面是用户代码执行之前的一个过程，让我们来看看他有什么作用

- start(kern/entry.S)
  1. 将内核页目录地址`entry_pgdir`地址装载进CR3寄存器中
  2. 重新赋值CR0寄存器开启分页机制，从实模式进入到保护模式之中
  3. 跳转到保护模式中的虚拟地址中去运行，初始化两个堆栈指针`ebp`, `esp`
  4. 调用`i3860_init()`函数
- i386_init(kern/init.c)
  - cons_init()
    1. `cga_init()`：文本模式中一些CGA/VGA相关初始化
    2. `kbd_init()`：键盘初始化
    3. `serial_init()`：串口初始化
  - mem_init()：
    1. 调用`i385_detect_memory()`函数确定剩余内存，并初始化`npages`, `npages_basemem`两个变量
    2. 分配内核页目录地址`kern_pgdir`并进行相关初始化
    3. 分配空间存放`pages`,`envs`两个管理数据结构
    4. 调用`page_init()`函数来进行页面的初始化，本质上就是对pages变量中管理的页面来进行初始化
    5. 对**内核的虚拟空间**进行初始化，完成一些对应的映射
    6. 最后重新启用分页机制，切换到新的页目录`kern_pgdir`中，并重新设置CR0寄存器
  - env_init()
    1. 初始化`envs`所管理的数据结果，将所有的用户环境(进程置为无)
    2. 并初始化`j per_cpu`
  - trap_init(still incomplete at this point)
  - env_creater()
    1. 调用`load_icode()`函数将某个**可执行文件装载到内**存中，然后将可执行文件的的**入口地址和我们新创建的进程绑定**
  - env_run()
    - 切换页目录
    - env_pop_tf()：弹出页帧进入程序

当我们完成了上面的函数后，