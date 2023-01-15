# MIT 6.S081 Lab6 COW


<!--more-->

## 课程知识

### 中断

​		中断是操作系统异常控制流的一种。中断由外部I/O设备触发，请求CPU进行处理。和系统调用，page fault一样，发生中断时，操作系统也需要保护现场，处理中断，中断处理程序完成之后，在返回当前指令继续执行。xv6通过`PLIC`(Platform Level Interrupt Control) 处理设备中断，来自外部设备的中断首先会到达PLIC，PLIC会将中断路由到某个CPU的核进行处理。

​		当一个设备在不可预知的时间需要注意时，中断是有意义的。但是中断有很高的CPU开销。网络和磁盘控制器的高速设备，需要使用一些技巧减少中断需求。比如可以对整批传入或传出的请求发出单个中断。另一个技巧驱动程序完全禁用中断，通过轮询方式，定期检查设备是否需要注意。如果设备执行操作非常快，轮询是有意义的，但是如果设备大部分空闲，轮询会浪费CPU时间。一些驱动程序根据当前设备负载在轮询和中断之间动态切换。

### 驱动

​		设备驱动程序用来管理设备，驱动可以执行设备操作，处理中断，和等待设备I/O的进程进行交互。大部分设备驱动可以分为上下两部分，上半部分要求硬件执行操作，在进程的内核线程运行，通过系统调用进行调用，如`read`和`write`，设备完成操作后，引发中断，下半部分即为中断处理程序，执行操作。

### 控制台I/O

​		xv6中，控制台驱动程序通过连接到RISC-V的UART串口硬件接受用户键入的字符，控制台驱动程序一次累积一行输入，使用`read`系统调用从控制台获取输入行。UART（Universal asynchronous receiver-transmitter）即通用异步收发传输器。UART硬件是一组内存映射的控制寄存器，通过`load`和`store`特定内存即可完成对硬件的控制。在xv6中，UART的内存映射地址起始于`0x10000000`，有几个宽度为一字节的UART控制寄存器。xv6的`main`函数调用`consoleinit`来初始化UART硬件。UART对接收到的每个字节的输入生成一个接收中断，对发送完的每个字节的输出生成一个发送完成中断。

​		当用户输入一个字符时，UART硬件要求RISC-V发出一个中断，从而激活xv6的陷阱处理程序。陷阱处理程序调用`devintr`，它查看RISC-V的`scause`寄存器，发现中断来自外部设备。通过PLIC查看发生中断的设备，如果是UART，`devintr`调用`uartintr`。`uartintr`从UART硬件读取所有等待输入的字符，并将它们交给`consoleintr`，它不会等待字符，因为未来的输入将引发一个新的中断。`consoleintr`在`cons.buf`中积累输入字符，直到一整行到达。当换行符到达时，`consoleintr`唤醒一个等待的`consoleread`。`consoleread`将监视`cons.buf`中的一整行，将其复制到用户空间，并返回到用户空间。

​		`write`系统调用对控制台的写入最终会调用`uartputc`函数，设备会维护输出缓冲`uart_tx_buf`，因此写进程不需要等待UART完成发送。`uartputc`将字符加入缓冲区后，调用`uartstart`函数开始传输之后返回，唯一的等待情况是缓冲区已满。每当UART发送一个字节后，就会产生一次中断。`uartintr`函数会调用`uartstart`函数判断传输是否完成，未完成就开始传输下一个缓冲的字符。因此，当进程写入多个字符时，第一个字节会通过`uartputc`调用`uartstart`进行传输，之后的字节将会被`uartintr`调用的`uartstart`进行传输。

​		控制台I/O需要通过UART驱动程序对读取UART控制寄存器，一次只能读取一个字节，并且数据会将传入的数据先拷贝的内核缓冲区，再拷贝到用户空间，无法在高数据速率下使用。需要高速移动大量数据的设备通常使用直接内存访问，DMA方式能够直接在用户空间缓冲区和设备硬件之间移动数据。DMA设备的驱动程序将在RAM中准备数据，然后使用对控制寄存器的单次写入来告诉设备处理准备好的数据。DMA广泛使用在磁盘和网络I/O。


&nbsp;
## 实验内容

### Implement copy-on write (hard)

​		本实验是实现`COW`机制，即`copy-on-write`写时拷贝，将子进程对父进程的内存拷贝，推迟到子进程实际需要物理内存时再进行物理页面的分配和复制。在`COW`中， `fork()`只为子进程创建一个页表，子进程的页表项指向父进程的物理页面。父进程和子进程中的所有用户`PTE`标记为不可写。当任一进程试图写入其中一个COW页时，CPU将强制产生页面错误。`page fault`处理程序为出错进程分配一页物理内存，将原始页复制到新页中，并修改出错进程中的相关PTE指向新的页面，将PTE标记为可写。

1. 在页表项中定义`COW`标志位

   ```C
   // kernel/riscv.h
   #define PTE_C (1L << 8)
   ```

2. 为每个物理页面设置一个引用计数，进行初始化，在最后一个PTE对它的引用撤销时释放

   ```C
   // kernel/kalloc.c
   struct {
     struct spinlock lock;
     int cnt[PHYSTOP >> 12];
   } kref;
   ...
   void
   kinit()
   {
     initlock(&kmem.lock, "kmem");
     initlock(&kref.lock, "kref");
     freerange(end, (void*)PHYSTOP);
   }
   ...
   void
   freerange(void *pa_start, void *pa_end)
   {
     ...
     for(; p + PGSIZE <= (char*)pa_end; p += PGSIZE) {
       kref.cnt[(uint64)p / PGSIZE] = 1;
       kfree(p);
     }
   }
   ```

   ```C
   // kernel/kalloc.c
   void *
   kfree(void *pa)
   {
     ...
     acquire(&kref.lock);
     kref.cnt[(uint64)pa >> 12]--;
     if(kref.cnt[(uint64)pa >> 12] != 0) {
       release(&kref.lock);
       return;
     }
   
     release(&kref.lock);
   
     r = (struct run*)pa;
     ...
   }
   
   void *
   kalloc(void)
   {
     struct run *r;
   
     acquire(&kmem.lock);
     r = kmem.freelist;
     if(r) {
       kmem.freelist = r->next;
       acquire(&kref.lock);
       kref.cnt[(uint64)r >> 12] = 1;
       release(&kref.lock);
     }
     release(&kmem.lock);
     ...
   }
   ```

3. 添加`addref`两个函数，用来添加引用计数

   ```C
   // kernel/defs.h
   void            addkref(uint64);
   ```

   ```C
   // kernel/kalloc.c
   void addkref(uint64 pa) {
     if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
       return;
     acquire(&kref.lock);
     kref.cnt[pa >> 12]++;
     release(&kref.lock);
   }
   ```

4. 修改`uvmcopy`，子进程的PTE需要指向父进程来代替为子进程分配新的物理页面

   ```C
   // kernel/vm.c
   int
   uvmcopy(pagetable_t old, pagetable_t new, uint64 sz)
   {
     pte_t *pte;
     uint64 pa, i;
     uint flags;
   
     for(i = 0; i < sz; i += PGSIZE){
       if((pte = walk(old, i, 0)) == 0)
         panic("uvmcopy: pte should exist");
       if((*pte & PTE_V) == 0)
         panic("uvmcopy: page not present");
       pa = PTE2PA(*pte);
       flags = PTE_FLAGS(*pte);
       
       if(flags & PTE_W) {
         flags = (flags | PTE_C) & ~PTE_W;
         *pte = PA2PTE(pa) | flags;
       }
       addkref(pa);
       if(mappages(new, i, PGSIZE, pa, flags) != 0){
         goto err;
       }
       
     }
     ...
   }
   ```

5. 添加`cowalloc`函数，用来分配`cow`物理页面

   ```C
   // kernel/defs.h
   int             cowalloc(pagetable_t, uint64);
   ```

   ```C
   // kernel/vm.c
   
   int cowalloc(pagetable_t pagetable, uint64 va) {
     uint64 pa;
     pte_t *pte;
     uint flags;
      
     if (va >= MAXVA) 
       return -1; 
     va = PGROUNDDOWN(va);
   
     pte = walk(pagetable, va, 0);
     if (pte == 0 || (*pte & PTE_V) == 0) 
       return -1;
   
     pa = PTE2PA(*pte);
     flags = PTE_FLAGS(*pte);
      
     if (flags & PTE_C) {
       char *mem = kalloc();
       if (mem == 0) return -1;
       memmove(mem, (char*)pa, PGSIZE);
       flags = (flags & ~PTE_C) | PTE_W;
       *pte = PA2PTE((uint64)mem) | flags;
       kfree((void*)pa);
       return 0;
     }
     return 0;
   }
   ```

6. 修改`usertrap`，处理`page fault`，分配物理页面

   ```C
   // kernel/trap.c
   void
   usertrap(void)
   {
     ...
     if(r_scause() == 8){
       ...
     } else if(r_scause() == 13 || r_scause() == 15) {
       uint64 va = r_stval();
       if (va >= MAXVA 
           ||(va <= PGROUNDDOWN(p->trapframe->sp) && va >= PGROUNDDOWN(p->trapframe->sp) - PGSIZE)
           || cowalloc(p->pagetable, va) != 0) 
           p->killed = 1;
     } else if((which_dev = devintr()) != 0){
       ...
   }
   ```

7. 修改`copyout`函数，在遇到`page fault`时，分配物理页面

   ```C
   // kernel/vm.c
   int
   copyout(pagetable_t pagetable, uint64 dstva, char *src, uint64 len)
   {
     uint64 n, va0, pa0;
   
     while(len > 0){
       va0 = PGROUNDDOWN(dstva);
       if (cowalloc(pagetable, va0) != 0) {
           return -1;
       }
       
       pa0 = walkaddr(pagetable, va0);
       ...
   }
   ```

   




