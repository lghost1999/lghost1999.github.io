# MIT 6.S081 Lab8 Lock


<!--more-->

## 课程知识

### 锁

​		现代计算机中多个CPU之间独立执行，同时多个CPU共享物理内存，并行访问内核中的数据结构时，需要使用锁来协调共享数据的更新以确保数据的一致性。举个例子，在xv6中，两个进程在两个不同的CPU上调用`wait`，`wait`通过`kfree`函数释放子进程的物理页面。在内核中，内核分配器通过链表的方式维护空闲页面，内核初始化时将所有的空闲页面以头插法的方式插入到空闲链表，分配物理页面时取出头节点，释放页面时将页面首地址插入到空闲链表头部。

```C
// kernel/kalloc.c

//空闲链表结构
struct run {
  struct run *next;
};

struct {
  struct spinlock lock;
  struct run *freelist;
} kmem;


// 释放页面
void
kfree(void *pa)
{
  ...
  r = (struct run*)pa;

  acquire(&kmem.lock);
  r->next = kmem.freelist;
  kmem.freelist = r;
  release(&kmem.lock);
}
```

​		在释放页面时包含两个操作，首先将页面插入到链表头部，再将链表头更新为新插入的页面，两个进程同时调用wait操作，就会出现竞态条件，导致出现错误，因此需要通过锁机制来保证指令执行顺序，`acquire`和`release`之间的指令序列被称为临界区域。锁用来串行化并发的临界区域，保证同一时刻只有一个进程在临界区运行。

<div align=center><img src="https://typora-lghost.oss-cn-shanghai.aliyuncs.com/img/202301111436725.png" alt="image-20230111143635584" style="zoom:50%;" /></div>

​		xv6中的所有锁：

<div align=center><img src="https://typora-lghost.oss-cn-shanghai.aliyuncs.com/img/202301111448089.png" alt="image-20221221182727326" style="zoom: 50%;" /></div>

### 死锁

​		死锁的四个条件：互斥，不剥夺，请求与保持，循环等待

​		如果在内核中执行的代码路径必须同时持有数个锁，那么所有代码路径以相同的顺序获取这些锁是很重要的。如果它们不这样做，就有死锁的风险。为了避免这种死锁，所有代码路径必须以相同的顺序获取锁，需要确定对于所有的锁对象的全局的顺序，先获取排序靠前的目录的锁，再获取排序靠后的目录的锁、

​		遵守全局死锁避免的顺序可能会出人意料地困难。有时锁顺序与逻辑程序结构相冲突，例如代码模块M1调用模块M2，但是锁顺序要求在M1中的锁之前获取M2中的锁。有时锁的身份是事先不知道的，也许是因为必须持有一个锁才能发现下一个要获取的锁的身份。



### 自旋锁

​		自旋锁通过忙等的方式，线程在获取锁的过程中保持执行，反复检查锁变量是否可用。一旦获取了自旋锁，线程会一直保持该锁，直到显示释放自旋锁。自旋锁避免了线程切换的开销，适合占用时间短的场景。

xv6中自旋锁表示为`spinlock`结构体，当锁可用时`locked`字段为0，当被持有时不为0

```C
// kernel/spinlock.h
struct spinlock {
  uint locked;       // Is the lock held?

  // For debugging:
  char *name;        // Name of lock.
  struct cpu *cpu;   // The cpu holding the lock.
};
```

自旋锁的实现逻辑：

```C
void
acquire(struct spinlock* lk) // does not work!
{
  for(;;) {
    //存在两个进程同时到达，同时占用该自旋锁，需要将下面这个两条作为原子指令执行
    //需要通过amoswap指令，在RISC-V上是amoswap r,a   amoswap 交换寄存器r和内存地址a的值
    if(lk->locked == 0) {
      lk->locked = 1;
      break;
    }
  }
}
```

自旋锁在xv6中的实现：

```c
// kernel/spinlock.c
void
acquire(struct spinlock *lk)
{
  //获取锁操作需要先关闭中断
  //spinlock需要处理两类并发，不同CPU之间和相同CPU上中断程序和普通程序之间的并发
  //如果自旋锁被中断处理程序所使用，那么CPU必须保证在启用中断的情况下永远不能持有该锁。
  push_off(); 
  if(holding(lk))
    panic("acquire");

  //在risc-v中, __sync_lock_test_and_set(&lk->locked, 1)对应amoswap指令
  //该函数返回交换之前lk-locked的值，若返回值为0，表示我们拿到了锁
  while(__sync_lock_test_and_set(&lk->locked, 1) != 0) ;

  //阻止CPU和编译器进行指令重排，编译器会优化指令顺序，在并发场景下对临界区和锁操作重排可能导致错误
  //任何在synchronize之前的load和store操作，都不能移动它之后，保证临界区操作都在获得锁之后进行
  __sync_synchronize();

  lk->cpu = mycpu();
}
```

```C
// kernel/spinlock.c
void
release(struct spinlock *lk)
{
  if(!holding(lk))
    panic("release");

  lk->cpu = 0;

  __sync_synchronize();

  //C标准编译器用多个存储指令实现赋值，先加载cache line，再更新cache line,可能会出现并发代码
  //使用__sync_lock_release对应amoswap指令，将lk->locked清0
  __sync_lock_release(&lk->locked);

  pop_off();
}

```

### 睡眠锁

​		自旋锁适合占用时间短的场景。在长时间保持锁的场景下，例如读写磁盘文件，对于磁盘的操作需要几十毫秒，如果使用自旋锁，则导致获取进程进程在自旋时浪费很长时间的CPU。同时自旋锁采用忙等，一个进程在持有自旋锁时不会让出CPU。

​		睡眠锁在等待时让出CPU，睡眠锁有一个被自旋锁保护的锁定字段，`acquiresleep`调用`sleep`让出CPU并释放自旋锁，其他线程可以在`acquiresleep`等待时执行。睡眠锁保持中断使能，不能用在中断处理程序中，也不能在自旋锁临界区使用，`acquiresleep`可能会让出CPU。

```C
// kernel/sleeplock.h
struct sleeplock {
  uint locked;       // Is the lock held?
  struct spinlock lk; // spinlock protecting this sleep lock
  
  // For debugging:
  char *name;        // Name of lock.
  int pid;           // Process holding lock
};

void
acquiresleep(struct sleeplock *lk)
{
  acquire(&lk->lk);
  while (lk->locked) {
    sleep(lk, &lk->lk);
  }
  lk->locked = 1;
  lk->pid = myproc()->pid;
  release(&lk->lk);
}
```

&nbsp;
## 实验内容

###  Memory allocator(moderate)

本实验需要将原来共有的空间列表改为为每个CPU维护一个空闲列表，当CPU需要内存时，从对应的空闲列表中获取，若对应的空闲列表为空，则从其他不为空的空闲列表中回去一个内存页。

1. 修改空闲列表，为每一个空闲列表初始化对应的锁

   ```C
   // kernel/kalloc.c
   struct {
     struct spinlock lock;
     struct run *freelist;
   } kmem[NCPU];
   
   void
   kinit()
   {
     char lockname[8];
     for(int i = 0; i < NCPU; i++) {
       snprintf(lockname, 6, "kmem%d", i);
       initlock(&kmem[i].lock, lockname);
     }
     freerange(end, (void*)PHYSTOP);
   }
   ```

2. 修改`kalloc`函数，在对应空闲列表为空时，去其他空闲列表获取，注意在调用`cpuid`函数时，需要关闭中断。

   ```C
   void *
   kalloc(void)
   {
     struct run *r;
       
     if(r) {
       kmem[id].freelist = r->next;
     } else {
       for(int i = 0; i < NCPU; i++) {
         if(i == id) continue;
         acquire(&kmem[i].lock);
         r = kmem[i].freelist;
         if(r) {
           kmem[i].freelist = r->next;
           release(&kmem[i].lock);
           break;
         }
         release(&kmem[i].lock);
       }
     }
     release(&kmem[id].lock);
     pop_off();
   
     if(r)
       memset((char*)r, 5, PGSIZE); // fill with junk
     return (void*)r;
   }
   ```

3. 修改`kfree`函数，将内存页添加到对应的空闲列表

   ```C
   void
   kfree(void *pa)
   {
     ...
     memset(pa, 1, PGSIZE);
   
     r = (struct run*)pa;
   
     push_off();
     int id = cpuid();
     acquire(&kmem[id].lock);
     r->next = kmem[id].freelist;
     kmem[id].freelist = r;
     release(&kmem[id].lock);
     pop_off();
   }
   ```

   

###  Buffer cache(hard)

本实验是将原来共有的磁盘块缓冲区转化为多个缓冲区，由原来的一个双向链表转化为多个双向链表，减少锁的争用，实现缓冲区分配和回收的并行执行。

1. 修改单个缓冲区为固定个哈希桶

   ```C
   // kernel/params.h
   #define NBUCKET      13
   ```

   ```C
   // kernel/bio.c
   struct bucket{
     struct spinlock lock;
     struct buf head;
   };
   
   struct {
     struct bucket bucket[NBUCKET];
     struct buf buf[NBUF];
   } bcache;
   ```

2. 修改`binit`函数，初始化`NBUCKET`个哈希桶

   ```C
   // kernel/bio.c
   void
   binit(void)
   {
     struct buf *b;
   
     char lockname[10];
     for(int i = 0; i < NBUCKET; i++) {
       snprintf(lockname, 10, "bcache%d", i);
       initlock(&bcache.bucket[i].lock, lockname);
   
       bcache.bucket[i].head.prev = &bcache.bucket[i].head;
       bcache.bucket[i].head.next = &bcache.bucket[i].head;
     }
   
     for(b = bcache.buf; b < bcache.buf+NBUF; b++){
       b->next = bcache.bucket[0].head.next;
       b->prev = &bcache.bucket[0].head;
       initsleeplock(&b->lock, "buffer");
       bcache.bucket[0].head.next->prev = b;
       bcache.bucket[0].head.next = b;
     }
   }
   ```

3. 添加时间戳属性，使用时间戳获取最近最久未使用的块

   ```C
   // kernel/buf.h
   struct buf {
     ...
     uint timestamp;
   };
   ```

4. 修改`bget`函数，对缓存不命中的数据块分配，从当前散列桶开始查找，找到每个桶中最近最久未使用的数据块进行分配，若当前桶中不存在未使用的缓冲区，则到下一个桶中查找。需要注意对数据块的缓存查找和缓存不命中的缓冲区分配是一个原子操作，需要注意锁的申请和释放。

   ```C
   // kernel/bio.c
   static struct buf*
   bget(uint dev, uint blockno)
   {
     struct buf *b;
   
     int id = blockno % NBUCKET;
   
     acquire(&bcache.bucket[id].lock);
   
     // Is the block already cached?
     for(b = bcache.bucket[id].head.next; b != &bcache.bucket[id].head; b = b->next){
       if(b->dev == dev && b->blockno == blockno){
         b->refcnt++;
   
         acquire(&tickslock);
         b->timestamp = ticks;
         release(&tickslock);
         
         release(&bcache.bucket[id].lock);
         acquiresleep(&b->lock);
         return b;
       }
     }
   
     // Not cached.
     // Recycle the least recently used (LRU) unused buffer.
     b = 0;
     for(int i = id, k = 0; k < NBUCKET; i = (i + 1) % NBUCKET, k++) {
       if(i != id) {
         if(!holding(&bcache.bucket[i].lock))
           acquire(&bcache.bucket[i].lock);
         else
           continue;
       }
   
       struct buf *t;
       for(t = bcache.bucket[i].head.next; t != &bcache.bucket[i].head; t = t->next){
         if(t->refcnt == 0 && (b == 0 || t->timestamp < b->timestamp))
           b = t;
       }
   
       if(b) {
         if(i != id) {
           b->next->prev = b->prev;
           b->prev->next = b->next;
           release(&bcache.bucket[i].lock);
   
           b->next = bcache.bucket[id].head.next;
           b->prev = &bcache.bucket[id].head;
           bcache.bucket[id].head.next->prev = b;
           bcache.bucket[id].head.next = b;
         }
   
         b->dev = dev;
         b->blockno = blockno;
         b->valid = 0;
         b->refcnt = 1;
   
         acquire(&tickslock);
         b->timestamp = ticks;
         release(&tickslock);
   
         release(&bcache.bucket[id].lock);
         acquiresleep(&b->lock);
         return b;
       } else {
         if(i != id)
           release(&bcache.bucket[i].lock);
       }
     }
     panic("bget: no buffers");
   }
   ```

5. 修改`brelse`函数，`bpin`函数和`bunpin`函数，适配固定个散列桶

   ```C
   // kernel/bio.c
   void
   brelse(struct buf *b)
   {
     if(!holdingsleep(&b->lock))
       panic("brelse");
   
     releasesleep(&b->lock);
   
     int id = b->blockno % NBUCKET;
     acquire(&bcache.bucket[id].lock);
     b->refcnt--;
     
     acquire(&tickslock);
     b->timestamp = ticks;
     release(&tickslock);
     
     release(&bcache.bucket[id].lock);
   }
   ```

   ```C
   // kernel/bio.c
   void
   bpin(struct buf *b) {
     int id = b->blockno % NBUCKET;
   
     acquire(&bcache.bucket[id].lock);
     b->refcnt++;
     release(&bcache.bucket[id].lock);
   }
   ```

   ```C
   // kernel/bio.c
   void
   bunpin(struct buf *b) {
     int id = b->blockno % NBUCKET;
   
     acquire(&bcache.bucket[id].lock);
     b->refcnt--;
     release(&bcache.bucket[id].lock);
   }
   ```

   
