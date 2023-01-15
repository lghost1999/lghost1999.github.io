# MIT 6.S081 Lab7 Thread


<!--more-->

## 课程知识

### 线程

任何操作系统都可能运行比CPU数量更多的进程，所以需要一个进程间分时共享CPU的方案。常见的是通过将进程**多路复用**到硬件CPU上，使每个进程产生一种错觉，即它有自己的虚拟CPU。当进程等待设备I/O和系统调用完成时，xv6使用`sleep`和`wakeup`机制唤醒切换；或者xv6周期性地强制切换长时间计算的进程。这种多路复用产生了每个进程都有自己的CPU的错觉，就像xv6使用内存分配器和硬件页表来产生每个进程都有自己内存的错觉一样。

**线程是单个串行执行代码的单元，只占用一个CPU并且顺序接收并执行指令**。每个CPU可以运行一个线程，一般来说，线程数远远多于CPU核数，每个CPU核会在多个线程之间切换。线程之间共享内存，多个线程都运行在同一个地址空间，需要通过锁来保证对共享数据的更新操作。线程具有：

- 程序计数器：表示当前执行指令的位置 （所在CPU的PC得到）
- 一组寄存器：保存变量  （所在CPU的寄存器得到）
- 栈：记录函数调用的记录，并反映了当前线程的执行点

xv6中线程的状态：

- RUNNING	正在运行的线程
- RUNABLE     就绪线程
- SLEEPING     等待I/O事件的阻塞线程

xv6中一个用户进程包含一个线程，线程控制了用户代码指令的执行，同时支持内核线程的概念，对于每个用户进程都有一个内核线程执行来自用户进程的调用，所有的内核线程共享内核内存。每个进程的结构体中包含`kstack`字段，指向内核栈；同时内核可以调用`myproc`函数获取当前CPU正在运行的进程。两种方法都可以区分不同的内核线程。



### 线程调度

线程调度有内核发起，内核会在定时器中断和系统调用时让出CPU，xv6为每个CPU核创建了一个线程调度器。在每个CPU核上，发生进程切换时将CPU的控制权从用户进程给到内核，用户进程对应的内核线程会让出CPU。

当一个程序运行时，用户进程的一个用户线程在运行，如果线程执行了一个系统调用或者中断陷入内核，相应的状态会保存在程序的`trapframe`中，包括PC和寄存器，CPU会被切换到内核栈上，内核线程会执行系统调用或者中断处理程序，处理完成后返回到用户空间，`trapframe`中保存的用户进程状态会被恢复。

从一个用户进程切换到另一个用户进程，以shell进程切换到cat进程为例：

<div align=center><img src="https://typora-lghost.oss-cn-shanghai.aliyuncs.com/img/202212281457296.png" alt="image-20221228145751189" style="zoom: 80%;" /></div>

从shell内核线程切换到cat内核线程的过程：

1. xv6首先将shell程序的内核寄存器保存在一个context对象
2. xv6恢复cat内核线程的context对象
3. xv6会继续在cat的内核线程栈上完成中断处理程序 （因为是cat程序发出的中断请求）
4. 恢复cat程序的trapframe中的用户进程状态，返回到用户空间的cat线程中继续执行



在xv6中用户线程的寄存器保存在`trapframe`中，内核线程的寄存器保存在`context`中。每一个CPU都有一个调度器线程，调度器线程也有自己的`context`对象。每个内核线程的`context`对象保存在用户进程对应的`proc`结构体中，调度器线程没有对应的进程，`context`对象保存在CPU结构体中，运行在CPU的线程，决定让出CPU时，会切换到CPU对应的调度器线程，并由调度器线程切换到下一个进程。

```C
// kernel/proc.h
struct context {
  uint64 ra;
  uint64 sp;

  // callee-saved
  uint64 s0;
  uint64 s1;
  uint64 s2;
  uint64 s3;
  uint64 s4;
  uint64 s5;
  uint64 s6;
  uint64 s7;
  uint64 s8;
  uint64 s9;
  uint64 s10;
  uint64 s11;
};
```

定时器中断具体过程：

首先进行中断处理程序`usertrap`，`usertrap`调用`yield`放弃CPU

```C
// kernel/trap.c
void
usertrap(void)
{
  ...
  //定时器中断
  if(which_dev == 2)
    yield();
  ...
}
```

`yield`获取当前进程的锁，并释放持有的锁，并修改进程状态为`RUNNABLE`，调用`sched`

```C
// kernel/proc.c
void
yield(void)
{
  struct proc *p = myproc();
  acquire(&p->lock);
  p->state = RUNNABLE;
  sched();
  release(&p->lock);
}
```

`sched`是**线程切换的核心**，检查条件，将当前内核线程的寄存器保存，并恢复当前CPU核的调度器线程的寄存器继续执行。调度器线程的`ra`寄存器保存`swtch`函数返回地址，`scheduler`函数，

```C
// kernel/proc.c
void
sched(void)
{
  int intena;
  struct proc *p = myproc();

  if(!holding(&p->lock))
    panic("sched p->lock");
  if(mycpu()->noff != 1)
    panic("sched locks");
  if(p->state == RUNNING)
    panic("sched running");
  if(intr_get())
    panic("sched interruptible");

  intena = mycpu()->intena;
  swtch(&p->context, &mycpu()->context);
  mycpu()->intena = intena;
}
```

`scheduler`函数寻找要调度的进程，调用`switch`保存调度器线程的寄存器，恢复选择运行的线程的寄存器继续执行。

```C
void
scheduler(void)
{
  struct proc *p;
  struct cpu *c = mycpu();
  
  c->proc = 0;
  for(;;){
    // 打开中断避免死锁
    intr_on();
    
    int nproc = 0;
    for(p = proc; p < &proc[NPROC]; p++) {
      acquire(&p->lock);
      if(p->state != UNUSED) {
        nproc++;
      }
      if(p->state == RUNNABLE) {
        p->state = RUNNING;
        c->proc = p;
        swtch(&c->context, &p->context);

        c->proc = 0;
      }
      release(&p->lock);
    }
    ...
  }
}
```

最后从`swtch`->`sched`->`yield`->`usertrap`，最后返回到用户空间执行。


&nbsp;
## 实验内容

### Uthread: switching between threads (moderate)

本实验需要为用户级线程系统设计上下文切换机制，可以参考xv6进程切换

1. 定义用来存储寄存器信息的`context`

   ```asm
   // user/uthread.c
   struct thread_context{
     uint64 ra;			//返回地址
     uint64 sp;			//内核栈指针
     //Caller Saved Register会被编译器保存在当前的栈上
     //只需要保存Callee Saved Register
     uint64 s0;
     uint64 s1;
     uint64 s2;
     uint64 s3;
     uint64 s4;
     uint64 s5;
     uint64 s6;
     uint64 s7;
     uint64 s8;
     uint64 s9;
     uint64 s10;
     uint64 s11;
   };
   
   struct thread {
     char       stack[STACK_SIZE]; /* the thread's stack */
     int        state;             /* FREE, RUNNING, RUNNABLE */
     struct thread_context context;
   };
   ```

2. 添加汇编代码进行被切换线程context的保存和切换线程context的恢复

   ```asm
   // user/uthread_switch.S
   thread_switch:
   	sd ra, 0(a0)
   	sd sp, 8(a0)
   	sd s0, 16(a0)
   	sd s1, 24(a0)
   	sd s2, 32(a0)
   	sd s3, 40(a0)
   	sd s4, 48(a0)
   	sd s5, 56(a0)
   	sd s6, 64(a0)
   	sd s7, 72(a0)
   	sd s8, 80(a0)
   	sd s9, 88(a0)
   	sd s10, 96(a0)
   	sd s11, 104(a0)
   
   	ld ra, 0(a1)
   	ld sp, 8(a1)
   	ld s0, 16(a1)
   	ld s1, 24(a1)
   	ld s2, 32(a1)
   	ld s3, 40(a1)
   	ld s4, 48(a1)
   	ld s5, 56(a1)
   	ld s6, 64(a1)
   	ld s7, 72(a1)
   	ld s8, 80(a1)
   	ld s9, 88(a1)
   	ld s10, 96(a1)
   	ld s11, 104(a1)
   	ret    /* return to ra */
   ```

3. 修改`thread_schedule`函数

   ```C
   void 
   thread_schedule(void)
   {
       ...
       if (current_thread != next_thread) {
           ...
           thread_switch((uint64)&t->context, (uint64)&current_thread->context);
       }
       ...
   }
   ```

4. 修改`thread_create`函数

   ```C
   void 
   thread_create(void (*func)())
   {
     ...
     t->context.ra = (uint64)func; 
     t->context.sp = (uint64)(t->stack + STACK_SIZE);
   }
   ```

   

### Using threads (moderate)

该实验在多线程环境下存在不一致情况，代码通过头插法将key-value键值对插入到对应的BUCKET中，两个线程同时对同一个BUCKET操作会造成数据覆盖。需要对插入操作加锁进行限制。

```C
// notxv6/ph.c
...
pthread_mutex_t lock;

double
now()
    
...

static 
void put(int key, int value)
{
	...
    else {
        // the new is new.
        pthread_mutex_lock(&lock);
        insert(key, value, &table[i], table[i]);
        pthread_mutex_unlock(&lock);
    }
}

...

int
main(int argc, char *argv[])
{
    ...
    tha = malloc(sizeof(pthread_t) * nthread);
  	pthread_mutex_init(&lock, NULL);
    ...
}
```



### Barrier(moderate)

该实验需要实现屏障，保证所有线程到达才开启下一个round

```C
static void 
barrier()
{
    ...
    pthread_mutex_lock(&bstate.barrier_mutex);
    if (++bstate.nthread < nthread) {
        pthread_cond_wait(&bstate.barrier_cond, &bstate.barrier_mutex);
    } else {
        bstate.nthread = 0;
        bstate.round++;
        pthread_cond_broadcast(&bstate.barrier_cond);
    }
  	pthread_mutex_unlock(&bstate.barrier_mutex);
}
```






