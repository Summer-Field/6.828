# 6.828 2017 Lecture 8: Isolation mechanisms

Today:
  user/kernel isolation
  xv6 system call as case study

### Multiple processes drive the key requirements:

- **multiplexing**
- **isolation**
- **interaction/sharing**

Isolation is often the most constraining requirement.

## What is isolation?

enforced separation to contain effects of failures

the **process is the usual unit of isolation**

- prevent process X from wrecking or spying on process Y
    r/w memory, use 100% of CPU, change FDs, &c

- prevent a process from interfering with the operating system in the face of malice as well as bugs

- a bad process may try to trick the h/w or kernel

**the kernel uses hardware mechanisms** as part of process isolation:

- user/kernel **mode flag**

- **address spaces**

- **timeslicing**
- **system call interface**

### User/Kernel Mode switch

the hardware user/kernel mode flag controls whether instructions can access privileged h/w called CPL on the x86, **bottom two bits of %cs register**

- **CPL=0 -- kernel mode -- privileged**
- **CPL=3 -- user mode -- no privilege**

x86 CPL protects many processor registers relevant to isolation

- I/O port accesses

- control register accesses (eflags, %cs4, ...)

- including **%cs itself**
    affects memory access permissions, but indirectly
    the kernel must set all this up correctly
    every serious microprocessor has some kind of user/kernel flag

* **how to do a system call -- switching CPL**
  Q: would this be an OK design for user programs to make a system call:
    set CPL=0
    jmp sys_open
    bad: **user-specified instructions with CPL=0**
  Q: how about a combined instruction that sets CPL=0,
    but *requires* an immediate jump to someplace in the kernel?
    **bad: user might jump somewhere awkward in the kernel**
  **the x86 answer:**
    there are only a few permissible kernel entry points ("vectors")
    **INT instruction sets CPL=0 and jumps to an entry point**
    but user code can't otherwise modify CPL or jump anywhere else in kernel
    **system call return sets CPL=3 before returning to user code**
    also a combined instruction (can't separately set CPL and jmp)

* the result: well-defined notion of user vs kernel
  **either CPL=3 and executing user code**
  **or CPL=0 and executing from entry point in kernel code**
  
  **?????????????????????CPL = 3 ????????????????????????????????????**
  
  **??????CPL = 0 ??????????????????????????????????????????????????????????????????**
  
    not:
    CPL=0 and executing user code
    CPL=0 and executing anywhere in kernel the user pleases
  
  ???	**????????????CPL = 0??????????????????????????????????????????CPL = 0???kerenel??????????????????????????????**

### Process Isolation

#### how to isolate process memory?

idea: **"address space"**

give each process some memory it can access
  for **its code, variables, heap, stack**

prevent it from accessing o**ther memory (kernel or other processes)**

#### how to create isolated address spaces?

xv6 uses x86 **"paging hardware"** in the memory management unit (MMU)
MMU translates (or "maps") every address issued by program

  CPU -> MMU -> RAM
         		 |
      	pagetable

  VA -> PA

  MMU translates all memory references: user and kernel, instructions and data instructions **use only VAs, never PAs**
**kernel sets up a different page table for each process**

?????????????????????????????????????????????

**each process's page table allows access only to that process's RAM**

??????????????????????????????????????????????????????RAM

## how xv6 system calls are implemented

xv6 process / stack diagram:
  user process ;  kernel thread
  user stack      ;  kernel stack
  two mechanisms:
   		**switch between user/kernel**
    		**switch between kernel threads**
  **trap frame**
  **kernel function calls...**
  **struct context**

* simplified xv6 user/kernel virtual address-space setup
  FFFFFFFF:
            ...
  80000000: kernel
            user stack
            user data
  00000000: user instructions
  **kernel configures MMU to give user code access only to lower half separate address space for each process but kernel (high) mappings are the same for every process**

system call starting point: 
  executing in user space, sh writing its prompt
  sh.asm, write() library function

```sh
break *0xd42
  x/3i
    0x10 in eax is the system call number for write
  info reg
    cs=0x1b, B=1011 -- CPL=3 => user mode
    esp and eip are low addresses -- user virtual addresses
  x/4x $esp
    ebf is return address -- in printf
    2 is fd
    0x3f7a is buffer on the stack
    1 is count
    i.e. write(2, 0x3f7a, 1)
  x/c 0x3f7a
```

INT instruction, kernel entry
  stepi
  info reg
    cs=0x8 -- CPL=3 => kernel mode
    **note INT changed eip and esp to high kernel addresses**
  **where is eip?**
    at a kernel-supplied vector -- only place user can go
    so user program can't jump to random places in kernel with CPL=0
  x/6wx $esp
    INT saved a few user registers
    err, eip, cs, eflags, esp, ss
  why did INT save just these registers?
    they are the ones that INT overwrites
  what INT did:
    switched to current process's kernel stack
    saved some user registers on kernel stack
    set CPL=0
    start executing at kernel-supplied "vector"
  where did esp come from?
    kernel told h/w what kernel stack to use when creating process

Q: why does INT bother saving the user state?
   how much state should be saved?
   transparency vs speed

saving the rest of the user registers on the kernel stack
  trapasm.S alltraps
  pushal pushes 8 registers: eax .. edi
  x/19x $esp
  19 words at top of kernel stack:
    ss
    esp
    eflags
    cs
    eip
    err    -- INT saved from here up
    trapno
    ds
    es
    fs
    gs
    eax..edi
  will eventually be restored, when system call returns
  meanwhile the kernel C code sometimes needs to read/write saved values
  struct trapframe in x86.h

Q: why are user registers saved on the kernel stack?
   why not save them on the user stack?

entering kernel C code
  the pushl %esp creates an argument for trap(struct trapframe *tf)
  now we're in trap() in trap.c
  print tf
  print *tf

kernel system call handling
  device interrupts and faults also enter trap()
  trapno == T_SYSCALL
  myproc()
  struct proc in proc.h
  myproc()->tf -- so syscall() can get at call # and arguments
  syscall() in syscall.c
    looks at tf->eax to find out which system call
  SYS_write in syscalls[] maps to sys_write
  sys_write() in sysfile.c
  arg*() read write(fd,buf,n) arguments from the user stack
  argint() in syscall.c
    proc->tf->esp + xxx

restoring user registers
  syscall() sets tf->eax to return value
  back to trap()
  finish -- returns to trapasm.S
  info reg -- still in kernel, registers overwritten by kernel code
  stepi to iret
  info reg
    most registers hold restored user values
    eax has write() return value of 1
    esp, eip, cs still have kernel values
  x/5x $esp
    saved user state: eip, cs, eflags, esp, ss
  IRET pops those user registers from the stack
    and thereby re-enters user space with CPL=3

Q: do we really need IRET?
   could we use ordinary instructions to restore the registers?
   could IRET be simpler?

back to user space
  stepi
  info reg

* Summary
 intricate design for User/Kernel transition
   how bad is a bug in this design?
   kernel must take adversarial view of user process
   doesn't trust user stack
   checks arguments
   page table confines what memory user program can read/write
   next lecture