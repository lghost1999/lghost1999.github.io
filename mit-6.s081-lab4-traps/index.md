# MIT 6.S081 Lab4 Traps


<!--more-->

## 课程知识

### RISC和CISC

xv6运行在`RISC-V`处理器上，`RISC-V`是精简指令集(RISC)，和传统以`x86-64`为代表的复杂指令集(CISC)存在一下区别：

- `RISC`拥有更少的指令数量，`CISC`需要向后兼容，指令数目会不断变大
- `RISC`指令功能简单，可以减少CPU执行时间，`CISC`指令功能复杂，指令周期较长

### 栈

xv6每次调用函数，系统都会创建一个栈帧`Stack Frame`，系统通过移动栈指针`Stack Pointer`来完成`Stack Frame`的空间分配。栈从高地址向低地址增长，`Stack Poiner`需要向下移动来创建一个新的`Stack Frame`。xv6栈结构图如下所示：

<div align=center><img src="https://typora-lghost.oss-cn-shanghai.aliyuncs.com/img/202211082353571.png" style="zoom: 50%;" /></div>

每个`Stack Frame`均包含`Retrun Address`和指向前一个`Frame`的指针，同时会保存寄存器和一些本地变量，系统维护两个寄存器`SP(Stack Pointer)`和`FP(Frame Pointer)`，`SP`指向当前`Frame`的底部，`FP`指向当前`Frame`的顶部，可以通过`FP`找到每个栈中的固定位置的`Return Address`和指向前一个`Frame`的指针，保证函数正确调用和返回。

### xv6的trap机制

从用户空间到内核空间的切换被称为陷入`trap`，`trap`是异常控制流的一种，`trap`分为三类 系统调用，异常和设备中断。

xv6的`trap`处理过程，以系统调用为例：

<div align=center><img src="https://typora-lghost.oss-cn-shanghai.aliyuncs.com/img/202211090009320.png" alt="6.S081-lab4" style="zoom: 25%;" /></div>



1. 用户请求系统调用，将系统调用号写入`a7`寄存器，并调用`ecall`指令。

   ```asm
   # kernel/usys.S
   ...
   .global write
   write:
    li a7, SYS_write
    ecall
    ret
    ...
   ```

   `ecall`指令执行内容：

   - `ecall`将状态从用户态设置为内核态
   - 将`pc`值保存在`sepc`寄存器中，
   - 设置好`stvec`，即`trapline page`的起始位置`uservec`函数，跳转`stvec`寄存器指向的指令

   > `ecall`指令并不会切换页表，为了能在用户页表下可以执行`uservec`，用户页表必须包含`uservec`的映射。`xv6`利用`trampoline page`实现，`trampoline page`在用户空间和内核空间都映射到了相同的虚拟地址。`trampoline page`， 即蹦床页面很形象，即通过该页面从用户空间跳到了内核空间。


2. 执行`uservec`函数

   `uservec`执行的第一步是交换`a0`和`SSCARTCH`的值。`SSCARTCH`寄存器保存`trapframe page`的地址，这样我们就得到了`trapframe page`的地址，可以将`32`个寄存器保存在`trapframe page`中了。

   > xv6就每一个进程的`trapframe`分配一个页面，并安排它映射在用户虚拟地址的固定位置，位于`trampoline page`的下一个页面。

   ```asm
   # kernel/trampoline.S
   # swap a0 and sscratch
   # so that a0 is TRAPFRAME
   csrrw a0, sscratch, a0
   
   # save the user registers in TRAPFRAME
   sd ra, 40(a0)
   sd sp, 48(a0)
   sd gp, 56(a0)
   sd tp, 64(a0)
   sd t0, 72(a0)
   sd t1, 80(a0)
   
   ...
   
   sd t6, 280(a0)
   
   # save the user a0 in p->trapframe->a0
   csrr t0, sscratch
   sd t0, 112(a0)
   
   ```

   接下来加载内核栈到`sp`，确保内核程序正常运行；保存CPU的`id`到`tp`，用来获取当前进程；向`t0`写入`usertrap`的指针；向`t1`写入内核页表的地址并与`SATP`寄存器进行交换，完成页表的切换，跳转到`usertrap`执行

   ```asm
   # kernel/trampoline.S
   # restore kernel stack pointer from p->trapframe->kernel_sp
   ld sp, 8(a0)
   
   # make tp hold the current hartid, from p->trapframe->kernel_hartid
   ld tp, 32(a0)
   
   # load the address of usertrap(), p->trapframe->kernel_trap
   ld t0, 16(a0)
   
   # restore kernel page table from p->trapframe->kernel_satp
   ld t1, 0(a0)
   csrw satp, t1
   sfence.vma zero, zero
   
   # a0 is no longer valid, since the kernel page
   # table does not specially map p->tf.
   
   # jump to usertrap(), which does not return
   jr t0						
   ```

3. 执行`usertrap`函数

   首先设置寄存器，更改`STVEC`寄存器，指向内核空间`trap`处理代码的位置；将`SEPC`保存到`trampframe`中，防止程序执行过程中切换到另一个程序，另一个程序再调用系统调用导致`SEPC`数据被覆盖。

   ```C
   // kernel/trap.c
   w_stvec((uint64)kernelvec);
   
   struct proc *p = myproc();
   
   // save user program counter.
   p->trapframe->epc = r_sepc();
   ```

   针对`trap`的来源进行分析，如果陷阱来自系统调用`syscall`会处理它，根据系统调用号查找相应的系统调用函数，执行真正的系统调用，将返回值保存在`tramframe`的`a0`寄存器中；如果是设备中断，`devintr`会处理；否则它是一个异常，内核会杀死错误进程。最后调用`usertrapret`函数。

   ```C
   // kernel/trap.c
   if(r_scause() == 8){
       // system call
   
       if(p->killed)
         exit(-1);
   
       // sepc points to the ecall instruction,
       // but we want to return to the next instruction.
       p->trapframe->epc += 4;
   
       // an interrupt will change sstatus &c registers,
       // so don't enable until done with those registers.
       intr_on();
   
       syscall();
     } else if((which_dev = devintr()) != 0){
       // ok
     } else {
       printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
       printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
       p->killed = 1;
     }
   
     if(p->killed)
       exit(-1);
   ```

4. 执行`usertrapret`，内核需要为返回内核空间做准备

   关闭中断，更新`STVEC`寄存器，指向`uservec`；设置`kernel page table`的指针，内核栈指针，`usertrap`函数的指针到`trapframe`中；设置`SSTATUS`寄存器，便于下一次从用户空间到内核空间的跳转，并恢复程序计数器`pc`的值。

   ```C
   // kernel/trap.c
   p->trapframe->kernel_satp = r_satp();         // kernel page table
   p->trapframe->kernel_sp = p->kstack + PGSIZE; // process's kernel stack
   p->trapframe->kernel_trap = (uint64)usertrap;
   p->trapframe->kernel_hartid = r_tp();    	  // hartid for cpuid()
   ...
   w_sepc(p->trapframe->epc);
   ```

   最后通过调用`userret`函数，将`trapframe page`的地址和用户页表的地址作为参数存储在`a0`，`a1`寄存器中，返回到用户空间的时候才能完成`page table`的切换。

   ```C
   uint64 fn = TRAMPOLINE + (userret - trampoline);
   ((void (*)(uint64,uint64))fn)(TRAPFRAME, satp);
   ```

5. 最后执行`userret`

   首先切换页表

   ```asm
   csrw satp, a1
   sfence.vma zero, zero
   
   # put the saved user a0 in sscratch, so we
   # can swap it with our a0 (TRAPFRAME) in the last step.
   ld t0, 112(a0)
   csrw sscratch, t0
   ```

   接着恢复用户寄存器，最后调用`sret`，`sret`会打开中断，更改内核态为用户态，跳转pc指向的指令执行用户程序。


&nbsp;
## 实验内容

### RISC-V assembly (easy)

1. `RISC-V`使用`a0`~`a7`共`8`个寄存器存储函数参数，若函数参数超过`8`个则需要存储在内存中，通过代码可以看出，`13`保存在`a2`寄存器中

   ```asm
   # user/call.asm
   24:	4635                	li	a2,13
   ```

2. 通过代码看出，函数并没有对f调用

   ```asm
   # user/call.asm
   26:	45b1                	li	a1,12
   ```

3. `printf`的函数地址是`0x640`

   `auipc`指令将高20位立即数左移12位加上`pc`后，存入`ra`寄存器，`0x00000097`对应的高20位左移12位之后为`0x0`，`pc`寄存器为`0x30`，因此`ra`寄存器的值为`0x30`，加上偏移量即为`0x30 + 0x600(1536) = 0x630`

   ```asm
   # user/call.asm
   30: 00000097       auipc ra,0x0
   34: 600080e7       jalr  1536(ra) # 640 <printf>
   ```

4. `printf`的`jalr`之后的寄存器`ra`值是`0x38`

   `jalr`指令当前`PC+4`保存在`rd`中，因此`jalr`之后寄存器`ra`值为`0x34 + 0x4 = 0x38`

5. 程序输出`HE110 World`，大端需改成`0x726c6400` `57616`不需要改

   > 大端将高位存放在低地址，小段将高位存放在高地址
   >
   > 大端存储`0x00646c72`		`00-64-6c-72`
   >
   > 小段存储`0x00646c72`		`72-6c-64-00`

6. 没有参数传`a2`寄存器，`y`显示的值是原来`a2`寄存器的值。

### Backtrace(moderate)

backtrace实验需要打印函数执行过程中每个`stack frame`的地址。`stack frame`的格式已经介绍。

1. 首先根据提示添加函数添加到`kernel/riscv.h`中，获取保存在`s0`寄存器的帧指针`fp`。

   ```C
   static inline uint64
   r_fp()
   {
     uint64 x;
     asm volatile("mv %0, s0" : "=r" (x) );
     return x;
   }
   ```

2. 实现`backtrace`函数，首先获取帧指针`fp`，并获取页栈页面的顶部地址，接着就打印`fp`的地址，并通过`frame`中指向前一个`frame`的指针获取前一个`frame`。直到到达页顶部，即已经打印了所有`stack frame`的地址。（需要在`kernel/defs.h`中声明）

   ```C
   // kernel/print.c
   void
   backtrace() {
     uint64 fp = r_fp();
     uint64 stack_base = PGROUNDUP(fp);
     printf("backtrace:\n");
     while(fp < stack_base) {
       printf("%p\n", *((uint64*)(fp - 8)));
       fp = *((uint64*)(fp - 16));
     }
   }
   ```

### Alarm(Hard)

`alarm`实验要求我们在进程使用CPU的时间内，xv6定期向进程发出警报。

1. 首先添加系统调用，并在`user/usys.pl`中添加

   ```C
   // user/user.h
   int sigalarm(int ticks, void (*handler)());
   int sigreturn(void);
   ```

2. 首先在进程结构体中添加字段，并在`allocproc`进行初始化和`freeproc`回收

   ```C
   // kernel/proc.h
   struct proc {
       ...
       int interval;		//警报间隔
       int tickcnt;		//距离上一次调用经过了多少个时钟
       uint64 handler;		//警报处理函数
       ...
   }
   
   ```

   ```C
   // kernel/proc.c
   static struct proc*
   allocproc(void)
   {
     ...
     p->interval = 0;
     p->tickcnt = 0;
     p->handler = 0;
   
     // An empty user page table.
     p->pagetable = proc_pagetable(p);
     ...
   }
   ```

   ```C
   // kernel/proc.c
   static void
   freeproc(struct proc *p)
   {
     ...
     p->interval = 0;
     p->tickcnt = 0;
     p->handler = 0;
   }
   ```

3. 实现真正的`sys_sigalarm`函数和`sys_sigreturn`函数

   ```C
   // kernel/syscall.h
   #define SYS_sigalarm 22
   #define SYS_sigreturn 23
   ```

   ```C
   // kernel/syscall.c
   ...
   extern uint64 sys_sigalarm(void);
   extern uint64 sys_sigreturn(void);
   
   static uint64 (*syscalls[])(void) = {
   ...
   [SYS_sigalarm] sys_sigalarm,
   [SYS_sigreturn] sys_sigreturn,
   };
   
   ```

   ```C
   // kernel/sysproc.c
   uint64 
   sys_sigalarm(void) 
   {
     if(argint(0, &myproc()->interval) < 0)
       return -1;
     if(argaddr(1, &myproc()->handler) < 0)
       return -1;
     return 0;
   }
   
   uint64 
   sys_sigreturn(void)
   {
     return 0;
   }
   ```

4. 修改`usertrap`，进程警报间隔期满时，设置`pc`值为警报处理函数`handler`的地址，同时清除时钟计数器

   ```C
   // kernel/trap.c
   
   if(which_dev == 2) {
       p->tickcnt++;
       if(p->flag == 0 && p->tickcnt == p->interval) {
           p->tickcnt = 0;
           p->trapframe->epc = p->handler;
       }
       yield();
   }
   ```

5. 我们在`test0`测试中系统调用返回时返回的`handler`的地址，会导致无法回到用户程序之中，我们需要在`test1/test2`中处理正确返回到用户代码和防止重复调用的问题。我们需要在进程结构体中再加入两个字段，并在`allocproc`进行初始化和`freeproc`回收。

   ```C
   // kernel/proc.h
   struct proc {
       ...
       int interval;		//警报间隔
       int tickcnt;		//距离上一次调用经过了多少个时钟
       uint64 handler;		//警报处理函数
       int flag;			
     	struct trapframe *alarm_trapframe;
       ...
   }
     
   ```

6. 添加保存进程陷阱帧`p->trapframe`到`p->alarm_trapframe`，并置`flag`为`1`，防止`handler`重复调用

   ```C
   // kernel/trap.c
   
   if(which_dev == 2) {
       p->tickcnt++;
       if(p->flag == 0 && p->tickcnt == p->interval) {
           memmove(p->alarm_trapframe, p->trapframe, sizeof(struct trapframe));
           p->tickcnt = 0;
           p->trapframe->epc = p->handler;
           p->flag = 1;
       }
       yield();
   }
   ```

7. 实现`sys_sigreturn`函数，当`hanler`调用`sigreturn()`时恢复陷阱帧，同时将`flag`置零，保证下一个`handler`能够调用。

   ```C
   // kernel/sysproc.c
   uint64 
   sys_sigreturn(void)
   {
     memmove(myproc()->trapframe, myproc()->alarm_trapframe, sizeof(struct trapframe));
     myproc()->flag = 0;
     return 0;
   }
   ```

   
