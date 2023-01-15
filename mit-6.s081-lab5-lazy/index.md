# MIT 6.S081 Lab5 Traps


<!--more-->
## 课程知识

### Page Fault	

​		page fault即页面错误异常，当CPU无法将虚拟地址转化为物理地址时，会生成page fault。RISC-V有三种页面错误：读页面错误、写页面错误和指令执行页面错误。page fault和其他异常一样，也使用和系统调用相同的`trap`机制，从用户空间切换到内核空间。对page fault的处理需要三个寄存器：STVAL，SCAUSE，SEPC，分别存储发生page fault的虚拟地址，发生page fault的原因和发生page fault的用户空间地址。RISC-V中所有SCAUSE的值：

<div align=center><img src="https://typora-lghost.oss-cn-shanghai.aliyuncs.com/img/202301081531247.png" alt="image-20230108153134089" style="zoom: 67%;" /></div>



### 使用Page Fault

#### Lazy page Allocation

​		当系统分配内存时，最终会调用`sbrk`函数，`sbrk`会扩大进程的`heap`的上边界，通过分配指定大小的物理内存，并将这些内存映射到用户程序的地址空间来扩大堆内存。`sbrk`函数默认是eager allocation，一旦调用了，内核会立即分配应用程序所需要的物理内存。带来的一个问题是，应用程序倾向申请多余自己所需要的内存，但是有部分内存永远也用不到，造成内存的浪费。

​		lazy allocation将实际的内存分配延迟到发生page fault时才进行。在page fault处理函数中分配物理页面，并映射到用户程序的地址空间。`sbrk`并不进行实际的内存分配，而只是改变进程的`p->sz`，当发生page fault时虚拟地址位于原`p->sz`和新`p->sz`之间时，为进程分配一个内存页。lazy allocations实行按需分配，可以减少内存的浪费，但是处理一次page fault需要用户态到内核态之间的切换，一个切换需要执行大量的store指令存取寄存器，会有时间性能的损耗。

#### Zero Fill On Demand

​		一个程序的地址空间，会包含堆区，栈区，BSS区域，data区域和text区域，当编译器生成二进制文件时，编译器会填充这三个区域，text区域是程序的指令，data区域存放初始化的全局变量，BSS区域存放未被初始化或者初始化为0的全局变量。C程序的内存布局：

<div align=center><img src="https://typora-lghost.oss-cn-shanghai.aliyuncs.com/img/202301081814281.png" alt="image-20230108181455216" style="zoom:67%;" /></div>



​		BSS段会有多个page来表示，所有page的内容均为0。我们只需要分配一个物理页面，将这个物理页面映射到所有程序的BSS段。需要设置PTE为只读，当我们需要尝试修改BSS的一个page，就会触发page fault，重新分配一个全0的物理页面，并映射到对应的虚拟地址。Zero可以节省内存，同时只需要分配一个全0物理页面，加快的程序的启动。

####  Copy On Write Fork

​		当执行`fork`函数时，会创建一个子进程，并且会为子进程分配物理内存，使得子进程和父进程拥有相同的地址空间。但是子进程通常在`fork`之后，会立即调用`exec`，替换进程的物理内存。将刚刚分配的page释放掉，造成性能损耗。也不能直接让父子进程共享物理内存，父进程和子进程对共享堆栈的写入会造成进程奔溃。

​		COW Fork让父子进程共享所有物理页面，但所有物理页面映射为只读。将父进程和子进程的PTE都设置为只读，当父进程或者子进程想要修改页面时，会触发page fault。page fault处理函数会分配一个新的物理页面，映射到该进程，并将PTE设置为可读写。我们需要在页表项中添加一项标志位，用来标识一个COW页面，用来区分正常的对只读地址写数据和对COW页面写数据。

​		引入COW Fork后，多个进程的虚拟地址都映射到了同一个物理页面，需要引入引用计数统计page，当我们释放虚拟page时，物理page引用计数减一，当page引用计数为0时，即可释放该物理页面。

#### Demand Paging

​		执行`exec`，操作系统会加载程序内存的text区域和data区域，以eager allocation的方式加载到内存存在一定弊端，程序的二进制文件从磁盘加载到内存代价很高，需要大量I/O操作，data区域大小远小于分配大小，并不需要通过eager方式进行分配。demand paging 在程序执行的时候才进行真正的分配。

​		demand paging为text区域和data区域分配好地址段，将PTE的有效位置为0，但并不分配真正的物理页面。应用程序从地址0向上增长，在执行时，地址0的指令触发page fault，page fault处理函数会从二进制文件中加载数据到内存，再将内存page映射到进程虚拟地址。若text区域和data区域大于真实物理内存，可以通过页面置换算法进行page淘汰。

#### Memory Mapped Files

​		memory mapped files将完整或者部分文件加载进内存，mmap将特定文件描述符的特定位置的数据映射到进程虚拟地址，就可以通过内存地址来读写文件。通过以lazy allocation的方式，并不会将文件从磁盘拷贝到内存，mmap借助VMA结构记录文件描述符，偏移量等元数据信息，用来保存虚拟地址对应的文件内容，当进程触发page fault的虚拟地址位于VMA内，操作系统才从磁盘中加载数据到内存，并映射到虚拟地址。

​&nbsp;
## 实验内容

### Eliminate allocation from sbrk() (easy)

该实验需要将实际调用堆内存代码删除，`sbrk()`只标记分配内存，并不真正进行分配

```C
// kernel/sysproc.c
uint64
sys_sbrk(void)
{
  int addr;
  int n;

  if(argint(0, &n) < 0)
    return -1;
  addr = myproc()->sz;
  myproc()->sz += n ;
  return addr;
}
```



### Lazy allocation (moderate)

本实验在上一个实验的基础上进行，上一个实验中sbrk()并不真正分配内存，进程在访问虚拟地址，进行虚拟地址到物理地址的转化时，会出现`page fault`，我们需要在出现`page fault`时进行内存分配

1. 处理`page fault`，当出现对虚拟地址的读和写错误时进行页面分配

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
       va = PGROUNDDOWN(va);
       char *pa = kalloc();
       if(pa == 0) {
         p->killed = 1;
       } else {
         memset(pa, 0, PGSIZE);
         if(mappages(p->pagetable, va, PGSIZE, (uint64)pa, PTE_W|PTE_X|PTE_R|PTE_U) != 0){
           kfree(pa);
           p->killed = 1;
         }
       }
     } else if((which_dev = devintr()) != 0){
       ...
   }
   ```

   

2. 修改`uvmunmap`函数，在`sys_sbrk()`中标记分配的内存可能并没有分配实际的物理内存，因此在接触虚拟地址和物理地址映射时需要跳过

   ```c
   // kernel/vm.c
   void
   uvmunmap(pagetable_t pagetable, uint64 va, uint64 npages, int do_free)
   {
     ...
     for(a = va; a < va + npages*PGSIZE; a += PGSIZE){
       if((pte = walk(pagetable, a, 0)) == 0)
         continue;
       if((*pte & PTE_V) == 0)
         continue;
       if(PTE_FLAGS(*pte) == PTE_V)
         panic("uvmunmap: not a leaf");
       if(do_free){
         uint64 pa = PTE2PA(*pte);
         kfree((void*)pa);
       }
       *pte = 0;
     }
   }
   ```

   

###  Lazytests and Usertests (moderate)

本实验需要完成提示中的要求，进一步完善物理内存的延迟分配，通过测试程序

1. 处理`sbrk()`参数为负数的情况，需要在为负数时调用`uvmdealloc()`函数

   ```C
   // kernel/sysproc.c
   uint64
   sys_sbrk(void)
   {
     ...
     if(n < 0) {
       if(addr + n < 0) return -1;
       if(uvmdealloc(myproc()->pagetable, addr, addr + n) != addr + n) return -1;
     } 
     addr = myproc()->sz;
     ...
   }
   
   ```

2. 判断出现`page fault`的虚拟地址是否合法，排除高于`sbrk()`分配虚拟内存地址和低于用户栈的虚拟内存地址

   ```C
   // kernel/trap.c
   void
   usertrap(void)
   {
     ...
     else if(r_scause() == 13 || r_scause() == 15) {
       ...
       if(va >= p->sz || va <= PGROUNDDOWN(p->trapframe->sp)) {
         p->killed = 1;
       } else {
         char *pa = kalloc();
         ...
   }
   ```

3. `fork()`调用`uvmcopy`子进程会拷贝父进程物理内存，父进程物理内存并没有全部分配，需要跳过没有分配的部分

   ```C
   // kernel/vm.c
   int
   uvmcopy(pagetable_t old, pagetable_t new, uint64 sz)
   {
     ...
     for(i = 0; i < sz; i += PGSIZE){
       if((pte = walk(old, i, 0)) == 0)
         continue;
       if((*pte & PTE_V) == 0)
         continue;
       ...
     }
     ...
   }
   ```

4. 进程调用`read`和`write`系统调用时，不会通过页表硬件进行地址翻译，而是通过`walkaddr`完成虚拟地址到物理地址的转化，我们需要在`copyin`和`copyout`中进行`page fault`处理

   ```C
   // kernel/vm.c
   int
   copyin(pagetable_t pagetable, char *dst, uint64 srcva, uint64 len)
   {
     ...
     while(len > 0){
       va0 = PGROUNDDOWN(srcva);
       pa0 = walkaddr(pagetable, va0);
       if(pa0 == 0) {
         if(va0 >= myproc()->sz || va0 < myproc()->trapframe->sp) {
           return -1;
         } else {
           pa0 = (uint64) kalloc();
           if (pa0 == 0) {
             myproc()->killed = 1;
           } else {
             memset((void *)pa0, 0, PGSIZE);
             va0 = PGROUNDDOWN(va0);
             if(mappages(myproc()->pagetable, va0, PGSIZE, pa0, PTE_W|PTE_R|PTE_U) != 0) {
               kfree((void *)pa0);
               myproc()->killed = 1;
             }
           }
         }
       }
       ...
     }
     return 0;
   }
   ```

   ```C
   // kernel/vm.c
   int
   copyout(pagetable_t pagetable, uint64 dstva, char *src, uint64 len)
   {
     ...
     while(len > 0){
       va0 = PGROUNDDOWN(dstva);
       pa0 = walkaddr(pagetable, va0);
       if(pa0 == 0) {
         if(va0 >= myproc()->sz || va0 < myproc()->trapframe->sp) {
           return -1;
         } else {
           pa0 = (uint64) kalloc();
           if (pa0 == 0) {
             myproc()->killed = 1;
           } else {
             memset((void *)pa0, 0, PGSIZE);
             va0 = PGROUNDDOWN(va0);
             if(mappages(myproc()->pagetable, va0, PGSIZE, pa0, PTE_W|PTE_R|PTE_U) != 0) {
               kfree((void *)pa0);
               myproc()->killed = 1;
             }
           }
         }
       }
       ...
     }
     return 0;
   }
   ```

   


