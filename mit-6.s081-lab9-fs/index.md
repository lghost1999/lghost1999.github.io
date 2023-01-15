# MIT 6.S081 Lab9 FS


<!--more-->

## 课程知识

### 文件系统

文件系统的目的是组织和存储数据。文件系统通常支持用户和应用程序之间的数据共享，以及**持久性**，以便在重新启动后数据仍然可用。xv6中的文件系统较小，整体可以分为7层

| 层级                           | 描述                                             |
| ------------------------------ | ------------------------------------------------ |
| 文件描述符（File descriptor）  | 使用统一接口对资源进行操作，抽象了Unix资源       |
| 路径名（Pathname）             | 提供分层路径名，通过递归来解析                   |
| 目录（Directory）              | 层次化的命名空间，由多个页表项组成               |
| 索引结点（Inode）              | 代表文件对象，实现了read/write等基本文件调用     |
| 日志（Logging）                | 日志可以用于崩溃后恢复                           |
| 缓冲区高速缓存（Buffer cache） | 缓存磁盘块到内存，避免频繁读写并同步磁盘块的访问 |
| 磁盘（Disk）                   | 提供持久化存储                                   |

文件系统常见的系统调用：

```C
//为file1创建另一个名为file2的文件，硬连接，引用计数+1
int link(char* file1, char* file2);
//删除file2文件，真实的inode的节点可能不会删除，引用计数-1
int unlink(char* file);
//向文件读出n个字符
int read(int fd, char* buf, int n);
//向文件写入n个字符
//write和read都没有针对文件的offset参数，文件系统维护了文件的offset。
int write(int fd, char* buf, int n);
```



### 磁盘

最常见的存储设备包括SSD和HDD，SSD完成一个磁盘块的读取是0.1~1 ms，HDD通常在10ms

扇区（sector）是磁盘驱动可以读写的最小单元，通常为512B。磁盘块（block）是文件存取的最小单位，由文件系统定义通常一个block对应多个sector。

xv6上磁盘块的布局结构：

<div align=center><img src="https://typora-lghost.oss-cn-shanghai.aliyuncs.com/img/202212281639759.png" alt="image-2022122816397594" /></div>

block 0：引导扇区，包含了启动操作系统的代码

block 1：超级块，包含文件系统的元数据(文件系统大小，数据块数，索引节点数和日志中的块数)

block 2 ~ block 31: 日志块

block 32 ~ block 44: 索引节点块，一个inode节点64B

block 45 : bitmap block 追踪正在使用的数据块

block 46 ~ : 数据块



### 日志层

文件系统需要考虑崩溃恢复。许多文件系统操作都涉及对磁盘的多次写入，可能会出现崩溃导致文件系统处于不一致状态。例如执行上述`echo "hi" > x`的操作，可能会出现标记为文件的inode节点的其他属性未更新，或者留下了已分配的block 595却未被引用等多种情况。

xv6不会直接写入磁盘上的文件系统，会在磁盘log记录所有磁盘写入的描述，然后向磁盘写入一条特殊commit，系统调用将写入磁盘上的文件系统，写完后清除日志。

- 如果崩溃发生在操作提交之前，磁盘上的登录将不会被标记为已完成，将不会进行恢复

- 如果崩溃发生在操作提交之后，恢复将重放所有写入操作，可能会重复这些操作。

日志驻留在超级块中指定的已知固定位置。它由一个头块和一系列更新块的副本组成。头块包含一个扇区号数组以及日志块的计数。磁盘上的头块中的计数或者为零，表示日志中没有事务；或者为非零，表示日志包含一个完整的已提交事务，并具有指定数量的日志块。在事务提交时Xv6才向头块写入数据，在此之前不会写入，并在将日志块复制到文件系统后将计数设置为零。事务中途崩溃将导致日志头块中的计数为零；提交后的崩溃将导致非零计数。

日志系统将多个系统调用写入累积到一个事务中，单个提交可能涉及多个完整系统调用。同时提交多个事务称为组提交，组提交减少了磁盘操作的数量。



### inode

文件系统中核心的数据结构就是inode和file descriptor。inode表示文件对象，并不依赖于文件名，通过自身编号进行区分，文件系统内部通过数字引用inode。inode必须有一个link count来跟踪指向这个inode的文件名的数量，只能在link count为0的时候被删除。实际中还有一个openfd count，也就是当前打开了文件的文件描述符计数，一个文件只能在这两个计数器都为0的时候才能被删除。后者主要与用户进程进行交互。

```C
// kernel/fs.h
struct dinode {
  short type;           // 表明文件还是目录
  short major;          // Major device number (T_DEVICE only)
  short minor;          // Minor device number (T_DEVICE only)
  short nlink;          // 指向该inode的文件名数
  uint size;            // 文件数据大小
  uint addrs[NDIRECT+1];// 文件数据块地址，0~11为数据块地址，最后一个为间接地址块的地址
  // 一个block可以装载256个block编号，每一个block编号为4B
};
```

间接索引块记录：

<img src="https://typora-lghost.oss-cn-shanghai.aliyuncs.com/img/202212281740994.png" alt="image-20221228174030930" style="zoom:50%;" />

inode包含12个直接块，最后一个元素为间接块的地址，可以从inode中列出的块加载文件的前12个字节，间接块中加载256个字节，xv6中最大文件为268 KB。



### 目录

在xv6中，每个目录包含多个目录项（directory entry），每个entry由固定的格式：

```C
// kernel/fs.h
struct dirent {
  // 文件或者子目录的inode编号
  ushort inum;
  // 文件或者子目录名
  char name[DIRSIZ];
};
```

在xv6中查找路径名`/y/x`：

- 从root inode开始查找，读取root inode内容，在block 32的64~128字节的位置
- 读取root inode包含的所有block，block内容由目录项组成。找到目录y，对应的inode编号为a
- 从inode a对应的所有block查找，找到文件x，返回对应的inode编号b，即为最终结果



输入`echo "hi" > x`底层实现：

- 写入block 33节点，首先将inode节点标记从空闲改为文件状态，并写回磁盘，再写入inode的其他属性。
- 写入根目录对应的block 46，因为此时正在向根目录创建一个新文件
- 写入block 32节点，更新根目录对应的inode，修改对应的size字段
- 写入block 45节点，选择未被使用的节点595进行写入，需要将该bit设置为1
- 写入block 595节点，写入具体内容，每个字符写入一次
- 写入block 33节点，更新inode的size字段，并修改`addrs`地址，将block 595编号写入


&nbsp;
## 实验内容

###  Large files(moderate)

xv6上一个`inode`节点包含12个直接块和一个间接块，xv6文件最多为`12 + 256 = 268`个块，即`268KB`, 需要将一个直接块改为二级间接块，这样最多可以容纳`12 + 256 + 256 * 256`个块，满足大文件需求。

1. 修改一个直接块为二级间接块

   ```C
   // kernel/fs.h
   #define NDIRECT 11
   #define NINDIRECT (BSIZE / sizeof(uint))
   #define NIINDIRECT ((BSIZE / sizeof(uint)) * (BSIZE / sizeof(uint)))
   #define MAXFILE (NDIRECT + NINDIRECT + NIINDIRECT)
   
   struct dinode {
     ...
     uint addrs[NDIRECT+2];   // Data block addresses
   };
   ```

   ```C
   // kernel/file.h
   struct inode {
     ...
     uint addrs[NDIRECT+2];
   };
   ```

2. 修改`bmap`分配二级间接块

   ```C
   // kernel/fs.c
   static uint
   bmap(struct inode *ip, uint bn)
   {
       ...
       if(bn < NIINDIRECT){
       if((addr = ip->addrs[NDIRECT + 1]) == 0)
         ip->addrs[NDIRECT + 1] = addr = balloc(ip->dev);
       bp = bread(ip->dev, addr);
       a = (uint*)bp->data;
   
       if((addr = a[bn / NINDIRECT]) == 0){
         a[bn / NINDIRECT] = addr = balloc(ip->dev);
         log_write(bp);
       }
       brelse(bp);
   
       bp = bread(ip->dev, addr);
       a = (uint*)bp->data;
       if((addr = a[bn % NINDIRECT]) == 0) {
         a[bn % NINDIRECT] = addr = balloc(ip->dev);
         log_write(bp);
       }
       brelse(bp);
       return addr;
     }
     
     panic("bmap: out of range");
   }
   ```

3. 修改`itrunc`回收二级间接块

   ```C
   // kernel/fs.c
   void
   itrunc(struct inode *ip)
   {
       ...
       if(ip->addrs[NDIRECT + 1]){
       bp = bread(ip->dev, ip->addrs[NDIRECT + 1]);
       a = (uint*)bp->data;
       for(i = 0; i < NINDIRECT; i++){
         struct buf *t;
         uint *b;
         t = bread(ip->dev, a[i]);
         b = (uint*)t->data;
         for(j = 0; j < NINDIRECT; j++){
           if(b[j])
             bfree(ip->dev, b[j]);
         }
         brelse(t);
         bfree(ip->dev, a[i]);
       }
       brelse(bp);
       bfree(ip->dev, ip->addrs[NDIRECT + 1]);
       ip->addrs[NDIRECT + 1] = 0;
     }
   
     ip->size = 0;
     iupdate(ip);
   }
   ```

   

###  Symbolic links(moderate)

本实验将实现一个系统调用`symlink(char *target, char *path)`，该调用在引用由`target`命名的文件的路径处创建一个新的符号链接。

1. 添加系统调用，参考`syscall`实验

   ```C
   // user/user.h
   int symlink(char*, char*);
   ```

   ```perl
   // user/usys.pl
   entry("symlink");
   ```

   ```C
   // user/syscall.h
   ...
   #define SYS_symlink 22
   ```

   ```C
   // user/syscall.c
   ...
   extern uint64 sys_symlink(void);
   
   static uint64 (*syscalls[])(void) = {
       ...
       [SYS_symlink] sys_symlink,
   }
   ```

   ```C
   // user/sysfile.c
   uint64
   sys_symlink(void)
   {
       return 0;
   }
   ```

   ```makefile
   // Makefile
   UPROGS=\
   ...
   $U/_symlinktest\
   ```

2. 添加宏定义

   ```C
   // kernel/fcntl.h
   #define O_NOFOLLOW 0x800
   ```

   ```C
   // kernel/stat.h
   #define T_SYMLINK 4
   ```

3. 实现`sys_symlink`函数

   ```C
   // kernel/sysfile.c
   uint64
   sys_symlink(void)
   {
     char target[MAXPATH], path[MAXPATH];
   
     if(argstr(0, target, MAXPATH) < 0 || argstr(1, path, MAXPATH) < 0)
       return -1;
   
     begin_op();
     struct inode *node = create(path, T_SYMLINK, 0, 0);
     if(node == 0) {
       end_op();
       return -1;
     }
   
     if(writei(node, 0, (uint64)target, 0, MAXPATH) < MAXPATH) {
       iunlockput(node);
       end_op();
       return -1;
     }
   
     iunlockput(node);
     end_op();
     return 0;
   }
   ```

4. 修改`sys_open`函数，支持符号链接，如果链接文件也是符号链接，则必须递归地跟随它，直到到达非链接文件为止。

   ```C
   // kernel/sysfile.c
   uint64
   sys_open(void){
       ...
       if(ip->type == T_SYMLINK && !(omode & O_NOFOLLOW)) {
           for(int i = 0; i < 10; ++i) {
             if(readi(ip, 0, (uint64)path, 0, MAXPATH) != MAXPATH) {
               iunlockput(ip);
               end_op();
               return -1;
             }
             iunlockput(ip);
             ip = namei(path);
             if(ip == 0) {
               end_op();
               return -1;
             }
             ilock(ip);
             if(ip->type != T_SYMLINK)
               break;
           }
   
           if(ip->type == T_SYMLINK) {
             iunlockput(ip);
             end_op();
             return -1;
           }
     	}
       
       if((f = filealloc()) == 0 || (fd = fdalloc(f)) < 0){
       ...
   }
   ```




