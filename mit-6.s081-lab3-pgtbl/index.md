# MIT 6.S081 Lab3 Pgtbl


<!--more-->

## 课程内容

### 虚拟内存

实现操作系统的强隔离性，需要硬件的支持。除了分离用户态内核态的设计，让用户程序与内核相互独立，还有虚拟内存机制的设计，让用户程序之间相互独立，每个进程拥有自己的地址空间。引入虚拟内存之后，每个进程拥有独立的地址空间，用户无需直接操作物理内存，只需使用该进程的虚拟地址，虚拟内存会将虚拟地址转化为物理地址，从而避免出现一个进程覆盖另一个进程的内存地址。

### 页表

页表是虚拟内存机制的核心，完成从虚拟地址到物理地址之间的映射。页表由页表项组成，页表项保存虚拟页号和物理页号映射。

> 页表是以页为粒度进行管理的，若以每个地址为粒度，则地址总线为64bit的机器需要管理2^64个地址，内存会被页表耗尽。

xv6中使用64位虚拟地址的低39bit，其中高27bit为页表索引，低12bit为页内偏移量`offset`，即页面大小为4KB。使用物理地址的低56bit，物理页号为其中高44bit，低12bit对应offset。通过页表索引得到物理块号，加上`offset`即为对应的物理地址。

<div align=center><img src="https://typora-lghost.oss-cn-shanghai.aliyuncs.com/img/202210061003923.png" alt="image-20221006100310806" style="zoom: 80%;" /></div>

页表保存在内存中，27bit的页表索引需要2^27个页表项来保存，每个页表项8B，需要占用大量连续内存，为此在xv6中引入三级页表。每个页表页拥有512个页表项，每个页表项都包含下一个页表页的物理地址，最低一级页表项对应的物理页号，加上低12bit的偏移量即为最终物理地址。

<div align=center><img src="https://typora-lghost.oss-cn-shanghai.aliyuncs.com/img/202210061430811.png" alt="image-20221006143036675" style="zoom:50%;" /></div>

相比于单级页表：

- 节省内存，内核不必为整个目录分配连续的内存页面。
- 按需分配，只有在需要用到该页表索引对应的物理页时才进行分配
- 缺点在于必须从内存中加载三个PTE将虚拟地址转换成物理地址，需要访问内存三次，需要借助`快表TLB`提升效率。

> TLB会保存虚拟地址到物理地址的映射关系，根据程序局部性原理，大大减少访存次数，提高寻址效率

xv6页表项为64bit，低10bit为各种标志位，包括读，写，执行等权限，中间44bit对应物理页号，高10bit保留。

<div align=center><img src="https://typora-lghost.oss-cn-shanghai.aliyuncs.com/img/202210061430472.png" alt="image-20221006143056431" style="zoom:50%;" /></div>

#### 内核页表

xv6为每个进程维护一个页表。描述每个进程的地址空间，还会维护一个单独的**描述内核地址空间的页表**，即内核页表。内核页表**映射整个物理内存**，内核设置了虚拟地址等于物理地址的映射关系，直接映射简化了读取或写入物理内存的内核代码。如通过虚拟地址查找物理地址时，先提取下一级页表的物理地址，然后将该物理地址作为虚拟地址获取下一级的页表项。

<div align=center><img src="https://typora-lghost.oss-cn-shanghai.aliyuncs.com/img/202210061509804.png" alt="image-20221006150941728" style="zoom: 50%;" /></div>



内核页表中除了直接映射之外，还存在特殊映射，即一个物理地址对应两个虚拟地址。

- 蹦床页面：
  - 蹦床页面被映射了两次，包括直接映射和虚拟地址空间的顶部
- 内核栈页面
  - 每个内核栈会被映射到高地址，便于在其虚拟地址下映射一个保护页(guard page)，内核栈溢出会引发一个异常，若栈溢出不会覆盖其他内存，	

#### 用户页表

xv6为每个进程维护一个用户页表，保存用户空间虚拟地址和物理地址之间的映射。

<div align=center><img src="https://typora-lghost.oss-cn-shanghai.aliyuncs.com/img/202210061552519.png" alt="image-20221006155215475" style="zoom:50%;" /></div>


&nbsp;
## 实验内容

### Print a page table (easy)

该实验需要我们打印所有已分配的页表，由于xv6采用三级页表，因此我们需要通过递归的方式按级打印页表项地址和下一级页表的物理地址，参考`/kernel/vm.c`中的`freewalk`函数。

```C
// kernel/vm.c
void 
vmprinthelper(pagetable_t pagetable, int k) 
{
  if(k == 3) return;
  for(int i = 0; i < 512; i++){
    pte_t pte = pagetable[i];
    if(pte & PTE_V){
      printf("..");
      for(int j = 0; j < k; j++) {
        printf(" ..");
      }
      printf("%d: pte %p pa %p\n", i, pte, PTE2PA(pte));
      vmprinthelper((pagetable_t)PTE2PA(pte), k + 1);
    }
  }
}

void 
vmprint(pagetable_t pagetable) 
{
  printf("page table %p\n", pagetable);
  vmprinthelper(pagetable, 0);
}
```

### A kernel page table per process (hard)

xv6会为每一个用户进程提供一个用户页表，该页表包含用户内存的映射，而内核页表不包含这些映射，当内核需要使用在系统调用中传递的用户指针时，用户地址在内核中无效，需要先将其转化为物理地址，导致效率降低，我们的目标是允许内核直接解引用用户指针。

本实验需要先为每一个进程维护一个内核页表，进程在内核中执行时使用它自己的内核页表，同时需要为每个进程在内核页表中分配内核栈，最后需要将所有调用`kernel_pagetable`的地方替换为当前进程的内核页表。

1. 在进程结构体中添加内核页表`kernel_pagetable`属性

   ```C
   // kernel/proc.h
   struct proc {
       ...
       pagetable_t kernel_pagetable;
       ...
   };
   ```

2. 实现`kvmcreate()`函数，在新进程创建时调用，参考`kvminit()`

   ```C
   // kernel/vm.c
   pagetable_t
   kvmcreate() 
   {
     pagetable_t user_kernel_pagetable = (pagetable_t) kalloc();
     memset(user_kernel_pagetable, 0, PGSIZE);
     uvmmap(user_kernel_pagetable, UART0, UART0, PGSIZE, PTE_R | PTE_W);
     uvmmap(user_kernel_pagetable, VIRTIO0, VIRTIO0, PGSIZE, PTE_R | PTE_W);
     uvmmap(user_kernel_pagetable, CLINT, CLINT, 0x10000, PTE_R | PTE_W);
     uvmmap(user_kernel_pagetable, PLIC, PLIC, 0x400000, PTE_R | PTE_W);
     uvmmap(user_kernel_pagetable, KERNBASE, KERNBASE, (uint64)etext-KERNBASE, PTE_R | PTE_X);
     uvmmap(user_kernel_pagetable, (uint64)etext, (uint64)etext, PHYSTOP-(uint64)etext, PTE_R | PTE_W);
     uvmmap(user_kernel_pagetable, TRAMPOLINE, (uint64)trampoline, PGSIZE, PTE_R | PTE_X);
     return user_kernel_pagetable;
   }
   ```

3. 将`procinit`函数中在为每个进程在内核页表中分配内核栈的操作转移到`allocproc`函数中，需要注意的是我们需要在当前进程的`kernel_pagetable`分配内核栈，为此我们需要设置一个`uvmmap`函数，指明在哪个页表添加，参考`kvmmap`函数

   ```C
   // kernel/proc.c
   void
   procinit(void)
   {
     struct proc *p;
     
     initlock(&pid_lock, "nextpid");
     for(p = proc; p < &proc[NPROC]; p++) {
         initlock(&p->lock, "proc");
   
         // Allocate a page for the process's kernel stack.
         // Map it high in memory, followed by an invalid
         // guard page.
   
         // char *pa = kalloc();
         // if(pa == 0)
         //   panic("kalloc");
         // uint64 va = KSTACK((int) (p - proc));
         // kvmmap(va, (uint64)pa, PGSIZE, PTE_R | PTE_W);
         // p->kstack = va;
     }
     kvminithart();
   }
   ```

   ```C
   // kernel/proc.c
   static struct proc*
   allocproc(void)
   {
     ...
     p->pagetable = proc_pagetable(p);
       
     p->kernel_pagetable = kvmcreate();
     if(p->pagetable == 0){
       freeproc(p);
       release(&p->lock);
       return 0;
     }
   
     char *pa = kalloc();
     if(pa == 0)
       panic("kalloc");
     uint64 va = KSTACK((int) (p - proc));
     uvmmap(p->kernel_pagetable, va, (uint64)pa, PGSIZE, PTE_R | PTE_W);
     p->kstack = va;
     ...
   }
   
   ```

   ```C
   // kernel/vm.c
   void
   uvmmap(pagetable_t pagetable, uint64 va, uint64 pa, uint64 sz, int perm) 
   {
     if(mappages(pagetable, va, sz, pa, perm) != 0)
       panic("uvmmap");
   }
   ```

4. 修改`schduler`函数，将该进程的内核页表`kernel_pagetable`装载进`satp`寄存器执行

   ```C
   // kernel/proc.c
   ...
   w_satp(MAKE_SATP(p->kernel_pagetable));
   sfence_vma();
   swtch(&c->context, &p->context);
   ...
   ```

5. 实现释放进程内核页表函数`freewalk_kernel_pagetable`函数，参考`freewalk`函数，需要注意的是，每个进程的内核页表都映射了整个物理内存和进程本身的内核栈，内核栈单独`free`，我们需要避免`free`最低一级页表对应的物理块。（需要在`kernel/defs.h`中声明）

   ```C
   // kernel/vm.c
   void
   freewalk_kernel_pagetable(pagetable_t pagetable) {
     for(int i = 0; i < 512; i++){
       pte_t pte = pagetable[i];
       if((pte & PTE_V)){
         pagetable[i] = 0;
         //只要最低一级页表项中这些标志位可能非0
         if ((pte & (PTE_R|PTE_W|PTE_X)) == 0)
         {
           uint64 child = PTE2PA(pte);
           freewalk_kernel_pagetable((pagetable_t)child);
         }
       }
     }
     kfree((void*)pagetable);
   }
   ```

6. 在`freeproc`函数中调用释放进程内核页表`freewalk_kernel_pagetable`函数

   ```C
   // kernel/proc.c
   if(p->kernel_pagetable) {
   	freewalk_kernel_pagetable(p->kernel_pagetable);
   }
   ```

7. 我们还需要修改`kvmpa`函数，将`kernel_pagetable`换成当前进程的内核页表。该函数将栈上虚拟地址转化为物理地址

   ```C
   // kernel/vm.c
   uint64
   kvmpa(uint64 va)
   {
     uint64 off = va % PGSIZE;
     pte_t *pte;
     uint64 pa;
     
     pte = walk(myproc()->kernel_pagetable, va, 0);
     if(pte == 0)
       panic("kvmpa");
     if((*pte & PTE_V) == 0)
       panic("kvmpa");
     pa = PTE2PA(*pte);
     return pa+off;
   }
   ```

### Simplify `copyin`/`copyinstr`（hard）

为了避免通过遍历进程页表来获取物理地址，我们需要将用户空间的映射添加到该进程的内核页表中。从而使得`copyin`和`copyinstr`可以直接操作用户指针。由于内核的虚拟内存从0开始的底部有一块空闲的地址，我们可以将映射添加到该部分，需要注意的是用户进程的最大大小限制为小于内核的最低虚拟地址。即`0xC000000`，`PLIC`寄存器的地址。

1. 实现`u2kvmcopy`函数，将进程的页表复制到进程的内核页表中，需要注意的是我们需要将用户页表中的`PTE_U`标志位去掉，以便可以在内核中执行。参考`uvmcopy`函数

   ```C
   // kernel/vm.c
   void
   u2kvmcopy(pagetable_t pagetable, pagetable_t kernelpt, uint64 oldsz, uint64 newsz){
     pte_t *pte_from, *pte_to;
     oldsz = PGROUNDUP(oldsz);
     for (uint64 i = oldsz; i < newsz; i += PGSIZE){
       if((pte_from = walk(pagetable, i, 0)) == 0)
         panic("u2kvmcopy: pte does not exist");
       if((pte_to = walk(kernelpt, i, 1)) == 0)
         panic("u2kvmcopy: pte walk failed");
       uint64 pa = PTE2PA(*pte_from);
       uint flags = (PTE_FLAGS(*pte_from)) & (~PTE_U);
       *pte_to = PA2PTE(pa) | flags;
     }
   }
   
   ```

2. 在`fork`函数，`exec`函数和`sbrk`函数中添加对`u2kvmcopy`函数的调用，确保用户映射能够复制到进程的内核页表中。

   ```C
   // kernel/exec.c
   int
   exec(char *path, char **argv)
   {
     ...
     stackbase = sp - PGSIZE;
   
     u2kvmcopy(pagetable, p->kernel_pagetable, 0, sz);
     ...
   }
   ```

   ```C
   // kernel/proc.c
   int
   fork(void)
   {
     ...
     u2kvmcopy(np->pagetable, np->kernel_pagetable, 0, np->sz);
     // copy saved user registers.
     *(np->trapframe) = *(p->trapframe);
     ...
   }
   ```

   ```C
   // kernel/proc.c
   int
   growproc(int n)
   {
     ...
     if((sz = uvmalloc(p->pagetable, sz, sz + n)) == 0) {
         return -1;
     }
     u2kvmcopy(p->pagetable, p->kernel_pagetable, sz - n, sz);
     ...
   }
   ```

3. 在`userinit`函数中将第一个进程的用户页表添加进该进程的内核页表。

   ```C
   // kernel/proc.c
   void
   userinit(void)
   {
     ...
     p->sz = PGSIZE;
     u2kvmcopy(p->pagetable, p->kernel_pagetable, 0, p->sz);
     ...
   }
   ```

4. 替换`copyin`和`copyinstr`（需要将`copyin_new`和`copyinstr_new`添加进`kernel/defs.h`）。

   ```C
   // kernel/vm.c
   int
   copyin(pagetable_t pagetable, char *dst, uint64 srcva, uint64 len)
   {
     return copyin_new(pagetable, dst, srcva, len);
   }
   
   int
   copyinstr(pagetable_t pagetable, char *dst, uint64 srcva, uint64 max)
   {
     return copyinstr_new(pagetable, dst, srcva, max);
   }
   ```

5. 为了确保用户进程的最大大小限制为小于内核的最低虚拟地址，需要在`sbrk`加入限制

   ```C
   // kernel/proc.c
   int
   growproc(int n)
   {
     ...
     if(n > 0){
       if (PGROUNDUP(sz + n) >= PLIC){
         return -1;
       }
       ...
     } 
     ...
   }
   ```

   

   
