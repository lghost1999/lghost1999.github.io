# MIT 6.S081 Lab1 Util

<!--more-->


6.S081是MIT开放的操作系统课程，围绕xv6展开教学和实验。xv6是简化的类UNIX系统，采用精简指令集[`RISC-V`](https://wikipedia.org/wiki/RISC-V)架构，运行在RISC-V微处理器上，实验可以在[`QEMU`](https://wikipedia.org/wiki/QEMU)上模拟运行，`QEMU`是一个硬件模拟器，用来模拟CPU和计算机，可以虚拟不同的硬件平台架构，在没有特定`RISC-V`硬件下也能运行xv6。本实验是基于[`6.S081 / Fall 2020`](https://pdos.csail.mit.edu/6.828/2020/schedule.html)。

## 课程知识

### 内核的概念

内核提供操作系统的基本功能，在开机时被加载进内存并常驻内存。内核不是进程，就是一段代码加数据的二进制文件。可以看成是一组系统调用的集合，不能主动执行，只能通过系统调用来为其他程序提供服务。例如用户要执行系统调用`open()`打开一个文件，就会由用户态切换为内核态，执行内核提供的`sys_open()`打开文件，返回`fd`。

内核组成：

- 管理用户进程的数据结构，如进程结构体，页表
- 管理各种硬件资源的数据结构，如抽象出的磁盘，I/O设备
- 各个模块提供的系统调用，如内存模块，文件系统，进程间通信(IPC)

### xv6的进程表示

Xv6进程由用户空间和内核空间组成，用户空间包含指令，数据和堆栈，内核空间包含每个进程状态。Xv6采用分时机制，保证多个进程并发执行。当一个进程没有执行时，Xv6保存CPU寄存器，并在下一次运行该进程时恢复它们。

```C
// kernel/proc.h
// Xv6进程结构体表示
struct proc {
  struct spinlock lock;

  // p->lock must be held when using these:
  enum procstate state;        // Process state
  struct proc *parent;         // Parent process
  void *chan;                  // If non-zero, sleeping on chan
  int killed;                  // If non-zero, have been killed
  int xstate;                  // Exit status to be returned to parent's wait
  int pid;                     // Process ID

  // these are private to the process, so p->lock need not be held.
  uint64 kstack;               // Virtual address of kernel stack
  uint64 sz;                   // Size of process memory (bytes)
  pagetable_t pagetable;       // User page table
  struct trapframe *trapframe; // data page for trampoline.S
  struct context context;      // 进程上下文，即寄存器内容
  struct file *ofile[NOFILE];  // Open files
  struct inode *cwd;           // Current directory
  char name[16];               // Process name (debugging)
};
```

### 常见的系统调用

#### fork()

一个进程可以使用fork系统调用创建一个新进程，子进程和父进程内存完全相同，在父进程中返回子进程的```pid```，在子进程中返回```0```,我们可以根据fork()返回值判断父子进程。子进程返回`0`，父进程返回子进程的`pid`。详见`kernel/proc.c`。

#### wait()

用来回收所有的子进程。wait()系统调用返回当前进程已退出**子进程的PID**，并将子进程的退出状态复制到传递给wait的地址, 0表示成功， 1表示失败，```wait()```会**一直等待到子进程退出**，若没有子进程，返回```-1```。如果我们由n个子进程，若想要等待所有的子进程都退出，父进程需要调用`n`次`fork()`。详见`kernel/wait.c`。

#### exec()

从文件系统中加载ELF格式的内存映像替换**调用进程**的内存，包含两个参数：文件名和字符串数组。exec()执行成功，它不向调用进程返回数据，而是使**指令从ELF header中声明的程序入口开始执行**，`exec()`一般配合`fork()`使用。详见`kernel/exec.c`。

### 文件描述符

进程需要读写由内核管理的对象，包括文件，管道，设备等，文件描述符将这些对象之间的差异**抽象**出来，隐藏不同类型文件之间的差异，使用文件描述符**统一进行I/O**。举个例子，cat程序并不需要知道是从文件，管道还是设备读取，也不需要知道写入到控制台、文件还是设备。

每个进程都有一个从文件描述符表，默认0表示标准输入，1表示标准输出，2表示标准错误，每次打开一个新文件，**优先分配最小的未使用的文件描述符，**可以实现I/O重定向。

引用文件的每个文件描述符都有一个与之关联的偏移量，`read`每次从文件的偏移量开始读取数据，`write`类似，exec()会替换调用进程的内存，但是不会改变修改子进程的描述符，并且**偏移量在父文件和子文件是共享的**。

### 管道

管道本质是一段**内核缓冲区**，一端用于读，一端用于写，可以作为一种**进程间通信**方式， **p[0]负责读，p[1]负责写**。

当读取端或写入端有多个文件描述符指向的时候。read会等待直到有新数据写入或者**所有指向写入端的文件描述符都被关闭**， write会等待直到有数据读出或者**所有指向读取端的文件描述符都被关闭**。

**当管道的读端和写端没有指向的文件描述符时，管道会自动回收**。同时相比于文件重定向，管道可以任意**传递长**的数据流，允许**并行执行**，且在进程间通讯时读写效率更高。


&nbsp;
## 实验内容

实验一并没有设计内核的修改和扩展，只是在用户空间利用xv6提供的系统调用完成一些公共程序。我们在接下来的实验中会用到以下系统调用：

| 系统调用                            | 描述                                                  |
| :---------------------------------- | :---------------------------------------------------- |
| int fork()                          | 创建一个进程，父进程返回子进程PID，子进程返回0        |
| int exit(int status)                | 终止当前进程，并将status报告给wait()函数，无返回      |
| int wait(int *status)               | 等待一个子进程退出，status保存退出状态，返回子进程PID |
| int getpid()                        | 返回当前进程的PID                                     |
| int exec(char *file, char *argv[])  | 加载一个文件并使用参数执行                            |
| int open(char* file, int flags)     | 打开一个文件，flags表示读/写，返回一个fd              |
| int close(int fd)                   | 释放打开的文件fd                                      |
| int write(int fd, char *buf, int n) | 从buf中写入n个字节到文件描述符fd， 返回n              |
| int read(int fd, char *buf, int n)  | 从fd中读取n个字节到buf中，返回读取的字节数            |
| int pipe(int p[])                   | 创建一个管道，把读写文件描述符放在p[0]和p[1]中        |
| int fstat(int fd, struct stat *st)  | 将打开文件fd的信息存入stat结构体中                    |
| int sleep(int n)                    | 使CPU休眠n个节拍                                      |



### sleep (easy）

sleep实验我们只需要xv6提供的```int sleep(int)```系统调用即可。

```C
int 
main(int argc, char *argv[]) {
    if(argc < 2) {
        fprintf(2, "Usage: sleep number\n");
        exit(1);
    }

    int n = atoi(argv[1]);
    sleep(n);
    
    exit(0);
}
```



### pingpong (easy)

pingpong实验使用管道进行父子进程的通信。实验需要使用两个管道，分别负责父进程的发送和子进程的发送，需要注意**阻塞**问题，当使用```fork()```之后，父子进程的```p[0]```都指向读取端，```p[1]```均指向写入端。为此我们**需要将多余的fd关闭**，保证管道能被回收。

```C
int
main(int agrc, char *argv[]) {
    int pid;
    int p2c[2], c2p[2];
    char buf[5];
    
    pipe(p2c);
    pipe(c2p);

    pid = fork();
    if(pid == 0) {
        close(p2c[1]);
        read(p2c[0], buf, 5);
        fprintf(2, "%d: received %s\n", getpid(), buf);
        close(p2c[0]);

        close(c2p[0]);
        write(c2p[1], "pong", 5);
        close(c2p[1]);

        exit(0);
    } else {
        close(p2c[0]);
        write(p2c[1], "ping", 5);
        close(p2c[1]);

        close(c2p[1]);
        read(c2p[0], buf, 5);
        fprintf(2, "%d: received %s\n", getpid(), buf);
        close(c2p[0]);

        wait((int *)0);
    }  
    exit(0);
}
```



### primes (moderate)/(hard))

primes实验是一个有趣的利用父子进程求解素数问题。利用[埃拉托色尼筛](https://en.wikipedia.org/wiki/Sieve_of_Eratosthenes)，父进程首先将```2 ~35```的素数放入管道中，fork()出子进程，读取管道中第一个数```k```，该数为素数，打印该数。并将管道中剩余的数全部取出，去掉```k```的倍数，剩余数存入管道，交给下一个子进程进行处理。直至管道为空。需要注意关闭没用的fd。

<div align=center><img src="https://typora-lghost.oss-cn-shanghai.aliyuncs.com/img/202210041931482.png" alt="未命名绘图" style="zoom: 25%;" /></div>

```C
void 
primes(int *p) {
    int i, k, pid;
    int np[2];

    if(read(p[0], &k, 4) == 0) {
        close(p[0]);
        exit(0);
    }

    fprintf(2, "prime %d\n", k);

    pipe(np);

    pid = fork();
    if(pid == 0) {
        close(np[1]);
        primes(np);
    } else {
        close(p[1]);
        close(np[0]);
        while(read(p[0], &i, 4) != 0) {
            if(i % k != 0) {
                write(np[1], &i, 4);
            }
        }
        close(np[1]);
        close(p[0]);
        wait((int*)0);
        exit(0);
    }
}

int 
main(int argc, char *argv[]) {
    int i, pid;
    int p[2];
    pipe(p);

    pid = fork();
    if(pid == 0) {
        close(p[1]);
        primes(p);
    } else {
        close(p[0]);
        for(i = 2; i <= 35; i++) {
            write(p[1], &i, sizeof(int));
        }
        close(p[1]);
        wait((int *)0);
    }
    exit(0);
}
```



### find (moderate)

find实验用来查找当前路径下所有指定文件名的文件，需要返回文件全路径，参考`user/ls.c`中关于目录文件的读取，即可实现。

```C++
//获取不带路径的文件名
char*
fmtname(char *path)
{
    static char buf[DIRSIZ+1];
    char *p;

    for(p=path+strlen(path); p >= path && *p != '/'; p--)
        ;
    p++;

    if(strlen(p) >= DIRSIZ)
        return p;
    memmove(buf, p, strlen(p));
    *(buf + strlen(p))= 0;
    return buf;
}

void 
find(char* path, const char* filename) {
    char buf[512], *p;
    int fd;
    struct dirent de;
    struct stat st;

    if((fd = open(path, 0)) < 0) {
        fprintf(2, "find: cannot open %s\n", path);
        return;
    }

    if(fstat(fd, &st) < 0){
        fprintf(2, "find: cannot stat %s\n", path);
        close(fd);
        return;
    }

    switch(st.type){
        case T_FILE:
            if(strcmp(fmtname(path), filename) == 0) {
                printf("%s\n", path);
            }
            break;

        case T_DIR:
            if(strlen(path) + 1 + DIRSIZ + 1 > sizeof buf){
                printf("ls: path too long\n");
                break;
            }
            strcpy(buf, path);
            p = buf+strlen(buf);
            *p++ = '/';
            while(read(fd, &de, sizeof(de)) == sizeof(de)){
                if(de.inum == 0 || strcmp(de.name, ".") == 0 || strcmp(de.name, "..") == 0)
                    continue;
                memmove(p, de.name, DIRSIZ);
                p[DIRSIZ] = 0;
                find(buf, filename);
            }
            break;
    }
    close(fd);
}

int
main(int argc, char *argv[]) {
    if(argc < 2) {
        fprintf(2, "Usage: find path filename\n");
        exit(1);
    } else if(argc == 2) {
        find(".", argv[1]);
    } else {
        find(argv[1], argv[2]);
    }

    exit(0);
}
```



### xargs (moderate)

xargs实验是编写一个简化版UNIX的xargs程序。xargs命令的作用，是将标准输入转为命令行参数，需要配合管道使用。实现xargs首先需要读取标准输入中的字符串，并将字符串添加到xargs所要执行的程序**参数列表最后作为参数**。若读取到多行， 需要为**每一行**的都fork()一个子进程进行执行。

```C++
int
main(int argc, char *argv[]) {
    int i;
    char *p, buf[512];

    if(argc < 2) {
        fprintf(2, "Usage: xargs command args...\n");
        exit(1);
    }

    if(read(0, buf, sizeof(buf)) == 0) {
        fprintf(2, "no arguments\n");
        exit(1);
    }
    
    p = buf;
    for(i = 0; buf[i] != 0; i++) {
        if(buf[i] == '\n') {
            buf[i] = 0;
            if(fork() == 0) {
                argv[argc++] = p;
                argv[argc] = 0;
                exec(argv[1], argv + 1);
                exit(0);
            } else {
                p = buf + i + 1;
                wait((int *)0);
            }
        }
    }
    exit(0);
}
```


