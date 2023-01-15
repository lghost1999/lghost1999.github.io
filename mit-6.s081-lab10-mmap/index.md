# MIT 6.S081 Lab10 Mmap


<!--more-->

## 课程知识

### xv6中的mmap

memory mapped files将完整或者部分文件加载进内存，`mmap`将特定文件描述符的特定位置的数据映射到进程虚拟地址，就可以通过内存地址来读写文件。通过以`lazy allocation`的方式，并不会将文件从磁盘拷贝到内存，`mmap`借助`VMA`结构记录文件描述符，偏移量等元数据信息，用来保存虚拟地址对应的文件内容，当进程触发`page fault`的虚拟地址位于`VMA`内，操作系统才从磁盘中加载数据到内存，并映射到虚拟地址。

`mmap`和`unmap`系统调用：

```C
/**
* addr: 想要映射到的地址，null由内核选择一个地址完成映射
* length: 想要映射的地址段长度
* prot: 保护位，设置读、写、执行权限
* flags：MAP_PRIVATE, 更新文件不会写入磁盘 MAP_SHARED, 更新文件需要写入磁盘
* fd: 传入的文件描述符
* offset: 偏移量
**/
void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset);

/**
* addr: 想要映射到的地址，null由内核选择一个地址完成映射
* length: 想要映射的地址段长度
**/
int munmap(void *addr, size_t length);
```

### Linux中的mmap

linux内核使用`vm_area_struct`结构来表示一个独立的虚拟内存区域，一个进程使用多个`vm_area_struct`结构来分别表示不同类型的虚拟内存区域.各个`vm_area_struct`结构使用链表或者树形结构链接，方便进程快速访问。mmap函数就是要创建一个新的`vm_area_struct`结构，并将其与文件的物理磁盘地址相连。`vm_area_struct`结构中包含区域起始和终止地址以及其他相关信息，同时也包含一个`vm_ops`指针，其内部可引出所有针对这个区域可以使用的系统调用函数。这样，进程对某一虚拟内存区域的任何操作需要用要的信息，都可以从`vm_area_struct`中获得：

<div align="center"><img src="https://typora-lghost.oss-cn-shanghai.aliyuncs.com/img/202301151652481.png" alt="image-20230115165218398" style="zoom: 67%;" /></div>



`mmap`调用过程：

1. 进程在用户空间调用mmap函数，内核寻找一段空闲的满足要求的连续的虚拟地址，为该虚拟区分配一个`vm_area_struct`，该结构保存用户参数。将新建的`vm_area_struct`插入到链表中。
2. 通过待映射的fd，获取对应的文件指针，调用内核空间的系统调用函数`mmap(struct file *filp, struct vm_area_struct *vma)`，创建页表项，实现文件物理地址和进程虚拟地址的映射，但并没有分配物理页面和进行数据拷贝。
3. 进程访问映射空间，触发page fault，需要将文件数据拷贝到内存。调页过程先在交换缓存空间（swap cache）中寻找需要访问的内存页，如果没有则调用`nopage`函数把所缺的页从磁盘装入到主存中。若对文件进行了写操作，一定时间后系统会自动回写脏页面到对应磁盘地址。



### mmap和文件操作区别

`read/write`操作过程：

- 内核通过查找进程文件符表，定位到内核已打开文件集上的文件信息，从而找到此文件的`inode`。
- 通过`inode`查找要请求的文件页是否已经缓存在页缓存中。如果存在，则直接返回这片文件页的内容。
- 如果不存在，则通过`inode`定位到文件磁盘地址，将数据从磁盘复制到**页缓存**。之后再次发起读页面过程，进而将页缓存中的数据发给用户进程。

常规文件操作为了提高读写效率和保护磁盘，使用了页缓存机制。这样造成读文件时需要先将文件页从磁盘拷贝到页缓存中，由于页缓存处在内核空间，不能被用户进程直接寻址，所以还需要将页缓存中数据页再次拷贝到内存对应的用户空间中。**常规文件操作需要从磁盘到页缓存再到用户主存的两次数据拷贝。而`mmap`只在发生缺页中断时，将磁盘数据拷贝到页缓存中，只进行一次拷贝。**

`mmap`优点：

- 减少了数据拷贝次数，用内存读写代替I/O读写，提高了读写效率
- 实现了用户空间和内核空间的高效交互方式。两空间的各自修改操作可以直接反映在映射的区域内
- 提供进程间共享内存及相互通信的方式。不管是父子进程还是无亲缘关系的进程，都可以将自身用户空间映射到同一个文件或匿名映射到同一片区域。(动态链接库也是利用mmap)
- `mmap`通过懒加载的方式，节省内存，可用于实现高效的大规模数据传输。

`mmap`缺点：

- 需要维护内存和磁盘文件的映射关系，占用一定内存资源
- 需要处理缺页中断


&nbsp;
## 实验内容

​		本实验实现一个内存映射文件的功能`mmap`，将文件映射到内存中，在进行文件操作直接通过对内存进行读写，使用`mmap`可以避免对文件大量`read`和`write`操作带来的内核缓冲区和用户缓冲区之间的频繁的数据拷贝。采用延迟分配的策略，在真正访问是才进行内存页的分配，为此需要在进程结构体中维护`mmap`相关信息的`VMA`。

1. 首先添加系统`mmap`和`munmap`系统调用，参考`syscall`实验

   ```C
   // user/user.h
   void *mmap(void*, int, int, int, int, int);
   int munmap(void*, int);
   ```

   ```perl
   // user/usys.pl
   entry("mmap");
   entry("munmap");
   ```

   ```C
   // user/syscall.h
   ...
   #define SYS_mmap   22
   #define SYS_munmap 23
   ```

   ```C
   // user/syscall.c
   ...
   extern uint64 sys_mmap(void);
   extern uint64 sys_munmap(void);
   
   static uint64 (*syscalls[])(void) = {
       ...
       [SYS_mmap]    sys_mmap,
   	[SYS_munmap]  sys_munmap,
   }
   ```

   ```C
   // user/sysfile.c
   uint64
   sys_mmap(void)
   {
       return 0;
   }
   
   uint64
   sys_munmap(void)
   {
       return 0;
   }
   ```

   ```makefile
   // Makefile
   UPROGS=\
   	...
   	$U/_mmaptest\
   ```

2. 定义`VMA`结构体，保存`mmap`系统调用的相关参数信息，加入到`proc`结构体中

   ```C
   // kernel/proc.h
   struct vma{
     uint64 addr;
     int length;
     int prot;
     int flags;
     struct file* mapped_file;
     int offset;
     int valid;
   };
   
   struct proc {	
     ...   
     struct vma vmas[16];
   };
   ```

3. 实现`sys_mmap`函数，获取系统调用参数，保存到相应的`VMA`结构体中。

   ```C
   // kernel/
   uint64
   sys_mmap(void)
   {
     int length, prot, flags, fd, offset, i;
     struct file* mapped_file;
     struct proc* p;
   
     if(argint(1, &length) < 0 || argint(2, &prot) < 0 || argint(3, &flags) < 0
        || argint(4, &fd) < 0 || argint(5, &offset) < 0)
       return -1;
   
     p = myproc();
     mapped_file = p->ofile[fd];
   
     // 文件不可写时，拥有PROT_WRITE权限映射不能是MAP_SHARED, 即不能对同一映射区域写入
     if((!mapped_file->writable) && (prot & PROT_WRITE) && (flags & MAP_SHARED))
       return -1;
   
     for(i = 0; i < 16; i++) {
       if(!p->vmas[i].valid) {
         p->vmas[i].addr = p->sz;
         p->vmas[i].length = length;
         p->vmas[i].prot = prot;
         p->vmas[i].flags = flags;
         p->vmas[i].mapped_file = mapped_file;
         p->vmas[i].offset = offset;
         p->vmas[i].valid = 1;
         break;
       }
     }
   
     if(i == 16) 
       return -1;
   
     filedup(mapped_file);
     p->sz += length;
   
     return p->vmas[i].addr;
   }
   ```

4. 在`usertrap`中完成`page fault`处理

   ```C
   // kernel/trap.c
   void
   usertrap(void)
   {
     ...
     if(r_scause() == 8){
       ...
     } else if(r_scause() == 13 || r_scause() == 15){
       uint64 va = r_stval();
       if(va >= p->sz || va < p->trapframe->sp) {
         p->killed = 1;
       } else {
         int i;
         // 找到发生page fault时虚拟地址所在的VMA
         for (i = 0; i < 16; i++) {
           if (p->vmas[i].valid) {
             if (p->vmas[i].addr <= va && (p->vmas[i].addr + p->vmas[i].length) > va)
               break;
           }
         }
   
         if(i == 16) {
           p->killed = 1;
         } else {
           uint64 mem = (uint64) kalloc();
           if (mem == 0){
             p->killed = 1;
           } else {
             memset((void *)mem, 0, PGSIZE);
             va = PGROUNDDOWN(va);
   		  
             ilock(p->vmas[i].mapped_file->ip);
             readi(p->vmas[i].mapped_file->ip, 0, mem, va - p->vmas[i].addr, PGSIZE);
             iunlock(p->vmas[i].mapped_file->ip);
   
             int flags = PTE_U;
             if(p->vmas[i].prot & PROT_READ) flags |= PTE_R;
             if(p->vmas[i].prot & PROT_WRITE) flags |= PTE_W;
             if(p->vmas[i].prot & PROT_EXEC) flags |= PTE_X;
   
             if(mappages(p->pagetable, va, PGSIZE, mem, flags) != 0) {
               kfree((void *)mem);
               p->killed = 1;
             }
           }
         }
       }
     } 
     ...
   }
   ```

5. 修改`uvmunmap`和`uvmcopy`，实现`lazy allocaton`

   ```C
   // kernel/vm.c
   int
   uvmcopy(pagetable_t old, pagetable_t new, uint64 sz)
   {
     ...
     for(i = 0; i < sz; i += PGSIZE){
       if((pte = walk(old, i, 0)) == 0)
         panic("uvmcopy: pte should exist");
       if((*pte & PTE_V) == 0)
         continue;
       ...
       }
     }
     ...
   }
   ```

   ```C
   // kernel/vm.c
   void
   uvmunmap(pagetable_t pagetable, uint64 va, uint64 npages, int do_free)
   {
     ...
     for(a = va; a < va + npages*PGSIZE; a += PGSIZE){
       if((pte = walk(pagetable, a, 0)) == 0)
         panic("uvmunmap: walk");
       if((*pte & PTE_V) == 0)
         continue;
       ...
     }
   }
   ```

6. 实现`sys_munmap`函数，获取系统调用参数，解除映射关系

   ```C
   uint64
   sys_munmap(void)
   {
     uint64 addr;
     int length, i;
     struct proc *p;
   
     if(argaddr(0, &addr) < 0 || argint(1, &length) < 0)
       return -1;
   
     p = myproc();
     for(i = 0; i < 16; ++i) {
       if (p->vmas[i].valid == 1) {
         if (p->vmas[i].addr <= addr && (p->vmas[i].addr + p->vmas[i].length) > addr)
               break;
       }
     }
   
     if (i == 16) {
       return -1;
     }
     
     // 若flags参数为MAP_SHARED，需要将页面写回磁盘
     if(p->vmas[i].flags == MAP_SHARED && (p->vmas[i].prot & PROT_WRITE) != 0) {
       filewrite(p->vmas[i].mapped_file, addr, length);
     }
     
     //根据addr和length解除映射，并修改对应VMA的addr和length
     if (p->vmas[i].addr == addr && p->vmas[i].length == length) {
       uvmunmap(p->pagetable, addr, length/PGSIZE, 1);
       fileclose(p->vmas[i].mapped_file);
       p->vmas[i].valid = 0;
     } else if (p->vmas[i].addr == addr) {
       uvmunmap(p->pagetable, addr, length/PGSIZE, 1);
       p->vmas[i].addr += length;
       p->vmas[i].length -= length;
     } else if (p->vmas[i].addr + p->vmas[i].length == addr + length){
       uvmunmap(p->pagetable, addr, length/PGSIZE, 1);
       p->vmas[i].length -= length;
     }
   
     return 0;
   }
   ```

7. 修改`fork`函数，复制父进程的`VMA`并增加文件引用计数

   ```C
   // kernel/proc.c
   int
   fork(void)
   {
     ...
     for(i = 0; i < 16; ++i) {
       if(p->vmas[i].valid) {
         memmove(&np->vmas[i], &p->vmas[i], sizeof(p->vmas[i]));
         filedup(p->vmas[i].mapped_file);
       }
     }
     
     safestrcpy(np->name, p->name, sizeof(p->name));
     ...
   }
   ```

8. 修改`exit`函数，将已映射的文件解除映射

   ```C
   // kernel/proc.c
   void
   exit(int status)
   {
     ...
     for(int i = 0; i < 16; ++i) {
       if(p->vmas[i].valid) {
         if(p->vmas[i].flags == MAP_SHARED && (p->vmas[i].prot & PROT_WRITE) != 0) {
           filewrite(p->vmas[i].mapped_file, p->vmas[i].addr, p->vmas[i].length);
         }
         fileclose(p->vmas[i].mapped_file);
         uvmunmap(p->pagetable, p->vmas[i].addr, p->vmas[i].length / PGSIZE, 1);
         p->vmas[i].valid = 0;
       }
     }
     ...
   }
   ```

   






