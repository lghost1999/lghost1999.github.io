# MIT 6.S081 Lab2 Syscall


<!--more-->

## 课程知识

### 什么是操作系统

操作系统是管理下层硬件资源，并为上层软件提供统一的抽象接口的**软件**。如果没有操作系统，进程可以直接运行在系统资源之上，甚至可以直接操作内存。操作系统可以保证系统资源的**强隔离性**，以实现多路复用和内存隔离。

具体来说，当我们用户空间有多个进程时，进程的调度需要靠进程自己释放和获得CPU资源，若出现崩溃则其他进程均无法运行。而有了操作系统，这些进程会被操作系统根据特定的调度算法进行执行，不会因为一个进程的崩溃而收到影响。

同样，用户程序可以直接对物理内存进行操作，进程可能会覆盖里一个进程的内存地址，导致程序崩溃。而有了操作系统，用户只需提供虚拟地址，OS会自动映射到物理地址进行操作，将和硬件的直接操作交给OS。



### 用户态和内核态

用户空间的程序运行在用户态，内核空间的程序运行在内核态。用户态下`CPU`可运行普通权限的指令，内核态下CPU可运行特权指令，特权指令包括直接操纵硬件的指令和设置保护的指令，例如设置页表寄存器，关闭时钟中断等

处理器中有一个标志位，1表示用户态，0为内核态。从用户态到内核态的切换是通过`ECALL`来实现的，`ECALL`接受一个数字作为参数。调用ECALL指令，ECALL会跳转到内核一个特定的由内核控制的位置。`syscall`函数接收到`ECALL`的参数，会调用实际的系统调用。

```perl
#/user/usys.pl
#usys.pl会被makefile调用，会被编译成usys.S汇编文件，当用户调用这些用户态程序时，便会进入usy.S执行
#可以看到用户态程序通过ecall指令跳转到内核，并且传入参数表示想要调用的系统调用
sub entry {
    my $name = shift;
    print ".global $name\n";
    print "${name}:\n";
    print " li a7, SYS_${name}\n";
    print " ecall\n";
    print " ret\n";
}
```

```asm
#/user/usys.S
#编译生成的fork()的汇编代码
#include "kernel/syscall.h"
.global fork
fork:
 li a7, SYS_fork
 ecall
 ret
```

ECALL指令提升硬件特权级别，并将PC更改为内核定义的入口点，入口点的代码切换到内核栈，执行实现系统调用的内核指令，当系统调用完成时，内核切换回用户栈，并通过调用`sret`指令返回用户空间，该指令降低了硬件特权级别，并在系统调用指令刚结束时恢复执行用户指令。

> 每个进程有两个栈区，用户栈区和内核栈区，当进程执行用户指令时，只有它的用户栈在使用，它的内核栈是空的。当进程进入内核时，内核代码在进程的内核栈上执行，用户栈仍然包含保存的数据，只是不处于活动状态。



### 宏内核和微内核

宏内核：整个操作系统的代码都运行在kernel mode中，集成度高，性能很好，缺点是内核很大，出现安全性问题的可能性也更大

微内核：微内核，内核只保留最基本的代码，如IPC，页表以及分时复用CPU等，大部分运行在用户空间。当我们需要调用这些系统调用时，通过内核作为中介进行调用。内核更安全，但是性能不行，需要两次内核空间到用户空间的切换。同时由于各部分隔离，共享`page cache`变得难以实现。



### xv6开机过程

1. 计算机上电，初始化并运行一个存储在ROM的引导加载程序，引导加载程序将xv6内核加载到内存中`(0x80000000)`
2. CPU从`_entry(kernel/entry.S)`开始运行xv6，`_entry`指令设置栈区，有了栈区，`_entry`调用C代码`start`
3. start设置内核态，时钟编程产生计时器中断，设置返回地址为main函数地址，禁用虚拟内存。通过调用mert进入main函数
4. `main()`初始化设备页表，调用`userinit()`创建第一个用户进程，通过系统调用`exec()`重新进入内核，`exec`返回到`init`进程用户空间
5. `init`进程创建控制台，用文件描述符0，1，2打开控制台文件，并启动一个shell



### xv6系统调用过程

1. 用户程序调用系统调用函数，将系统调用号存入`a7`寄存器，调用`ecall`指令。
2. `ecall`指令会进入内核定义的入口点，切换内核栈运行，依次执行`uservec`、`usertrap`和`syscall`。
3. `syscall`会获取`trapframe`中存储在`a7`寄存器的值，即系统调用号，执行对应的系统调用函数。
4. 处理结束之后需要将返回值放入`trapframe`的`a0`寄存器，调用`sret`，用户空间会获得系统调用的返回结果。


&nbsp;
## 实验内容

实验二和实验一相反，已经帮我们实现好了用户程序，需要涉及到内核的修改和扩展，要求我们实现系统调用，保证用户程序正常运行。

### System call tracing（moderate）

system call tracing实验要求我们实现一个可以追踪调用情况的系统调用，我们需要创建一个`sys_trace`的系统调用，该系统接受一个`mask`参数，若`mask`第n位为1，即表示我们需要显示该系统调用被调用情况，打印出进程id、系统调用的名称和返回值。为此，我们需要在每次系统调用结束之后，检查该进程`mask`对应位上是否为`1`，即调用是否需要打印。

1. 首先添加系统调用号

   ```C
   // kernel/syscall.h
   ...
   #define SYS_trace  22
   ```

2. 在函数指针数组`syscalls[]`中加入`sys_trace`

   ```c
   // kernel/syscall.c
   static uint64 (*syscalls[])(void) = {
       ...
   	[SYS_trace]   sys_trace,
   }
   ```

3. 在进程结构体中添加`mask`属性，保存需要追踪的系统调用

   ```C
   // kernel/proc.h
   struct proc {
       ...
       int mask;
   }
   ```

4. 修改`fork()`,确保派生的任何子进程也能进行追踪

   ```C
   // kernel/proc.c
   int
   fork(void)
   {
       ...
       np->mask = p->mask;
       release(&np->lock);
       return pid;
   }
   ```

5. 实现`sys_trace()`

   ```C
   // kernel/sysproc.c
   uint64
   sys_trace(void)
   {
     int n;
     struct proc *p = myproc();
     if(argint(0, &n) < 0)
       return -1;
     p->mask = n;
     return 0;
   }
   ```

6. 修改`syscall(void)`

   ```C
   // kernel/syscall.c
   //保存系统调用的名称，便于打印
   static char* names[23] = {
     "fork", "exit", "wait", "pipe", "read", "kill","exec", "fstat", 
     "chdir", "dup", "getpid", "sbrk", "sleep", "uptime", "open",
     "write", "mknod", "unlink", "link", "mkdir", "close", "trace", "sysinfo"
   };
   
   void
   syscall(void)
   {
     int num;
     struct proc *p = myproc();
   
     num = p->trapframe->a7;
     if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
       p->trapframe->a0 = syscalls[num]();
       if(p->mask >> num & 1) printf("%d: syscall %s -> %d\n", p->pid, names[num - 1], p->trapframe->a0);
     } else {
       printf("%d %s: unknown sys call %d\n",
               p->pid, p->name, num);
       p->trapframe->a0 = -1;
     }
   }
   ```

### Sysinfo（moderate）

sysinfo实验要求我们实现显示运行信息的系统调用，包括显示空闲内存的字节数和进程数。空闲内存的字节数的统计参考`kalloc.c`，进程数的统计参考`proc.c`

1. 首先添加系统调用号

   ```C
   // kernel/syscall.h
   ...
   #define SYS_sysinfo 23
   ```

2. 在函数指针数组`syscalls[]`中加入`sys_trace`

   ```C
   // kernel/syscall.c
   static uint64 (*syscalls[])(void) = {
       ...
   	[SYS_sysinfo] sys_sysinfo,
   }
   ```

3. 统计空闲内存的字节数（需要在`kernel/defs.h`中声明）

   ```C
   // kernel/kalloc.c
   int 
   freecount(void) 
   {
     int freecnt = 0;
     struct run *r;
     r = kmem.freelist;
     while(r) {
       freecnt++;
       r = r->next;
     }
     return freecnt * PGSIZE;
   }
   ```

4. 统计进程数（需要在`kernel/defs.h`中声明）

   ```C
   // kernel/proc.c
   int 
   proccount(void)
   {
     int proccnt = 0;
     struct proc *p;
   
     for(p = proc; p < &proc[NPROC]; p++) {
       if(p->state != UNUSED) {
         proccnt++;
       }
     }
     return proccnt;
   }
   ```

5. 实现`sys_info()`

   ```C
   uint64
   sys_sysinfo(void)
   {
     uint64 addr;
     struct sysinfo si;
     if(argaddr(0, &addr) < 0)
       return -1;
     si.freemem = freecount();
     si.nproc = proccount();
     //将sysinfo结构体复制回用户空间
     if(copyout(myproc()->pagetable, addr, (char *)&si, sizeof(si)) < 0)
       return -1;
     return 0;
   }
   ```

   


