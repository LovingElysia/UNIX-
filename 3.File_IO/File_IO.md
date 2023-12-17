# 3.文件I/O

### 3.3 open函数和openat函数

```C++
#include<fcntl.h>

int open(const char *path,int flag,.../* mode_t mode*/)

int openat(int fd,const char *path,int flag,.../*mode_t mode*/)

``````

返回值：成功返回文件描述符 失败返回-1

1.参数path  

想要打开或创建的文件名

2.参数flag

使用1个或多个常量进行或运算组成flag参数

常见的有下列：

O_RDONLY 只读打开

O_WRONLY 只写打开

O_RDWR   读写打开

O_CREAT 若无该文件则创建该文件

3.fd参数

fd参数将open 和 openat函数进行区分  通常有以下三种情况：

* path参数指定绝对路径 此时open函数和openat函数是没有区别de
* path参数指定相对路径名称 fd参数指出了相对路径名在文件系统中开始的地址 fd参数是通过打开相对路径目录获得的
* path参数指定了相对路径名称 fd参数给定特殊值 AF_FDCWD 路径名从当前工作目录获取

### 3.4 creat函数

``````C++
#include<fcntl.h>

int creat(const char*path,mode_t mode)

``````

返回值 ：若成功返回文件描述符 若失败返回-1

此函数等效于：

``````C++
open(fd,path,O_RDONLY|O_CREAT|O_TRUNC);
``````

creat() 函数只能只读打开一个文件 如果你想写某个文件 必须先将其关闭后再用open打开，所以现在creat函数并不单独使用

### 3.5 close函数

``````C++
#include<fcntl.h>
int close(fd);
``````

返回值:成功返回0 失败返回 -1

当一个程序结束时,系统会隐式的为其关闭所有文件描述符

### 3.6 lseek函数

``````C++
#include<fcntl.h>
off_t lseek(int fd,off_t offset,int whence);
``````

对参数 offset 的解释与参数whence 有关

* SEEK_SET 将文件偏移量设置为 相对于文件起始处 offset 个字节
* SEEK_CUR 将文件偏移量设置为 相对于文件当前偏移量+offset个字节 offset可正可负(即将偏移量前移或后移)
* SEEK_END 将文件偏移量设置为 相对于文件末尾处 offset 个字节

返回值：若成功返回新的文件偏移量 若失败返回-1

示例：

``````C++
lseek(fd,0,SEEK_SET); //将文件偏移量设置为距离文件起始处0个字节 即文件开头
``````

以下方式可以确定打开文件的当前偏移量:

``````C++
off_t currpos
currpos = lseek(fd,0,SEEK_CUR);
``````

以上方法还能判断某个文件是否可设置偏移量 若文件描述符指向一个管道，fifo，网络套接字
则lseek 返回 -1 将errno 设置为ESPIPE

lseek本身不读取文件,它通过设置偏移量，可以让read write等I/O函数可以从指定偏移量开始读写 实现随机读写

在普通文件中文件偏移量应该必须为非负整数 但某些特殊文件可以为负整数 所以判断lseek的返回值不应使用<0,而应该使用==0来判断是否出错

文件偏移量允许大于当前文件的实际长度 对该文件的下一次读写将加长该文件的长度 并在文件中构成一个 **空洞** 文件未被写过的字节都被读为0  而这部分数据不会分配磁盘空间

示例生成一个空洞文件的程序:

``````C++
#include<fcntl.h>   //file_hole.cpp
#include<unistd.h>
char buf_1[]={'a','b','c','d','e','f'};
char buf_2[]={'A','B','C','D','E','F'};

int main()
{
    
    //创建一个空洞文件 
    int fd;
    fd = open("./file_hole_test",O_WRONLY|O_CREAT);
    write(fd,buf_1,sizeof(buf_1));
    lseek(fd,16384,SEEK_SET); //将偏移量设置为相对于文件起始处16384位
    write(fd,buf_2,sizeof(buf_2)); //write将会从偏移量的位置开始写入
    close(fd);
    return 0;
}                //file_hole.out
``````

运行这段程序:

``````shell
$ sudo od -c file_hole_test
[sudo] kevin 的密码： 
0000000   a   b   c   d   e   f  \0  \0  \0  \0  \0  \0  \0  \0  \0  \0
0000020  \0  \0  \0  \0  \0  \0  \0  \0  \0  \0  \0  \0  \0  \0  \0  \0
*
0040000   A   B   C   D   E   F
0040006
``````

od 命令查看文件的实际内容 -c参数以字符形式打印文件内容

### 3.6 reade()函数

``````C++
#include<fcntl.h>
sszie_t reade(int fd,void *buf,size_t nbytes);
``````

返回值:读到的字节数 若已经读到文件尾部则返回 0 若出错则返回-1

有多种情况可使实际读到的字节数少于要求读的字节数:

* 读普通文件时，在读到要求字节数之前已到达了文件尾端。例如，若在到达文件尾端之前有30个字节，而要求读100个字节，则read返回30。下一次再调用read时，它将返回0 (文件尾端)。

* 当从终端设备读时，通常一次最多读- -行(第18章将介绍如何改变这- -点)。

* 当从网络读时，网络中的缓冲机制可能造成返回值小于所要求读的字节数。

* 当从管道或FIFO读时，如若管道包含的字节少于所需的数量，那么read将只返回实际可用的字节数。

* 当从某些面向记录的设备(如磁带)读时，一次最多 返回一个记录。

* 当一-信号造成中断，而已经读了部分数据量时。我们将在10.5节进一步讨论此种情况。

### 3.8 wirte函数

``````C++
#include<fnctl.h>
ssize_t write(int fd,void *buf,size_t nbytes);
``````

**返回值**:若成功返回已经写的字节数 失败则返回-1

### 3.9 dup函数和dup2函数

``````C++
#include<unistd.h>
int dup(int fd);
int dup2(int fd,int fd2);
``````

**返回值** ：若成功返回新的文件描述符 若失败返回-1

dup2函数会现关闭fd2 然后再将 fd复制到fd2 并返回新的fd2 如果fd2=fd 则dup2会直接返回fd2而不关闭fd2 否则fd2的 FD_CLOSEXEC标志将会被清除

这些函数返回的文件描述符和fd共享同一个文件表项

复制文件描述符的另一个方法是用fcntl函数 实际上下列代码是等效的：

``````C++
newfd = dup(fd);
``````

等效于：

``````C++
newfd = fcntl(fd,F_DUPFD,0); //参数0表示选择大于等于0的未分配的最小文件描述作为新的文件描述符
                             //fd将会被复制给newfd
``````

而

``````C++
dup2(fd1,fd2);  //将fd2重定向到fd1
``````

等效于

``````C++
close(fd2);
fd2 = fcntl(fd,F_DUPFD,0);
``````

但其实第二种情况并不完全等效 因为dup2执行关闭和复制操作是一个原子操作 而fcntl是两个函数调用

### 3.10 sync函数 fsync函数 fdatasync函数

传统unix系统都提供缓冲区高速缓存 和 页高速缓存 而大多数磁盘IO都通过缓冲区进行 当我们向文件写数据时
通常会先复制到缓冲区 然后在排入写队列 晚些时候再写入文件 这被称之为**延迟写** 当系统需要重用缓冲区存放其他磁盘数据块时 它就会把所有延迟写数据写入磁盘

``````C++
#include<unistd>
int fsync(int fd);
int fdatasync(int fd);
void sync(void);
``````

**返回值**:若成功则返回0 出错则返回-1

* sync函数会将所有修改过的块缓冲区排入写队列 然后便返回 它并不等待实际写磁盘操作结束
通常称为update的守护进程会周期性的执行该函数 这就保证了定期flush内核的块缓冲区

* fsync函数只对有fd文件描述指定的一个文件起作用 并且等待磁盘写操作结束时才会返回

* fdatasync 函数与 fsync类似 但它只影响文件的数据部分 而出数据之外 fsync还会同步更新文件属性

### 3.10 fcntl函数

``````C++
#include<fcntl.h>
int fcntl(int fd,int cmd,.../*int arg*/)
``````

**返回值**：若成功则依赖于cmd 若出错则返回-1

#### fcntl有以下功能

1. 复制一个已有的文件描述符      cmd=F_DUPFD 或 F_DUPFD_CLOSEXEC
2. 获取或设置文件描述符标志      cmd=F_GETFD 或 F_SETFD
3. 获取或设置文件状态标志        cmd=F_GETFL 或 F_SETFL
4. 获取或设置文件异步I/O的所有权 cmd=F_GETOWN 或 F_SETOWN
5. 获取或设置记录锁             cmd=F_GETLK 或 F_SETLK 或 F_SETLKW

* F_DUPFD 复制文件描述符fd 新文件描述符作为返回值返回 此时参数3为一个整数N 表示希望得到一个尚未打开的 大于等于N的各值中取一个最小值作为新的文件描述符 它和fd拥有同一个文件表项 但独立拥有一套文件描述符标志 其中FD_CLOEXEC被抹除  这意味着该fd会被新进程所继承
* F_DUPFD_CLOEXEC 同上 但会为新文件描述符设置 FD_CLOSEXEC
* F_GETFD 将fd文件描述符标志作为返回值
* F_SETFD 对一个fd设置文件描述符标志 标志取决于参数3
* F_GETFL 对应于fd的文件状态标志作为函数值返回 open函数的flag参数都是文件状态标志其中前5个值是互斥的 一个文件的访问方式只能取这5个之一 因此必须用O_ACCMODE屏蔽字取得访问方式位
然后将其与5个值中的每个值相比较

| 文件状态标志 |说明|
|------------|---|
|O_RDONLY|只读打开
|O_WRONLY|只写打开
|O_RDER| 读写打开
|O_EXEC| 只执行打开
|O_SEARCH|只搜索目录打开
|O_APPEBD|追加写
|O_NONBLOCK|非阻塞
|O_SYNC|等待写完成(数据和属性)
|O_DSYNC|等待写完成(仅数据)
|O_RSYNC|同步读写
|O_FSYNC|等待写完成(仅freeBSD 和 MAC OS X)
|O_ASYNC|异步I/O

* F_SETFL 将文件标志设置为第3个参数 除前五个以外都可更改
* F_GETOWN 返回当前接收 SIGIO信号和SIGURG信号的进程ID或者进程组ID
* F_SETOWN 设置接收......进程组ID 正数arg表示进程ID 负数arg表示对应arg绝对值的进程组ID

示例程序：

``````C++
#include<fcntl.h>  //fcntl_test.cpp
#include<unistd.h>
#include<stdlib.h>
#include<stdio.h>
int main(int argc,char *argv[])
{

    int val;
    if(argc!=2)
    {
        _exit(0);
    }
    if((val=fcntl(atoi(argv[1]),F_GETFL,0))<0)
    {
        printf("fcntl error");
        exit(0); 
    }
    switch (val & O_ACCMODE)
    {
    case O_WRONLY:
        printf("write only");
        break;
    case O_RDONLY:
        printf("read only");
        break;
    case O_RDWR:
        printf("reade write");
        break;
    default:
        break;       
    }

    if(val & O_APPEND)
        printf(",append");
    if(val & O_NONBLOCK)
        printf(",NONBLOCK");
    if(val & O_SYNC)
        printf(", synchronous writes");
    #if !defined(_POSIX_C_SOURCE) && defined(O_FSYNC) && (O_FSYNC != O_SYNC)
        if(val&O_FSYNC)
            printf(",synchronous writes");
    #endif
    putchar('\n');
    exit(0);            
    return 0;   //fcntl_test.out
}
``````

测试：

``````shell
$./fcntl_test 0 0</dev/tty
read only
$ ./fcntl_test 1 1>temp1.foo
$ cat temp1.foo
write only
$ ./fcntl_test 2 2>>temp1.foo
write only,append
$ ./fcntl_test 5 5<>temp1.foo
reade write
``````

5<>temp1.foo 表示打开temp1.foo供文件描述符5读写

修改文件状态标志或描述符标志时需要谨慎 不可直接使用 F_SETFL或F_SETFD进行设置 这样会导致以前的标志位关闭

必须先使用F_GETFL 或 F_GETFD获得现在的标志值 然后按照期望修改 然后再设置

示例程序:

``````C++
#include<fcntl.h>
#include<stdio.h>
#include<unistd.h>
void set_fl(int fd,int flags)
{
    int val;
    if((val=fcntl(fd,F_GETFL,0))<0)  //先获取当前标志
    {
        perror("set_fl F_GETFL error");
        _exit(0);
    }
    val|=flags;                     //按期望修改
    if(fcntl(fd,F_SETFL,val)<0)     //再设置
    {
        perror("set_fl F_GETFL error");
        _exit(0);
    }
}
int main()
{
    int fd;
    fd = open("path",O_WRONLY|O_APPEND);
    set_fl(fd,O_WRONLY|O_APPEND|O_SYNC);
    return 0;
}
````````

当支持同步写O_SYNC时，系统CPU时间和时钟时间就会显著增加,但Linux并不允许通过fcntl函数设置O_SYNC标志

### 3.11 ioctl函数

``````C++
#include<sys/ioctl.h>
int ioctl(int fd,int request,...);
``````

**返回值**:出错返回-1 成功返回其他值

### 3.12 /dev/fd

较新系统都提供/dev/fd 其目录项是名为0，1，2等的文件 打开文件/dev/fd/n 相当于复制文件描述符N(前提是描述符N打开)

例如：

``````C++
newfd = open("/dev/fd/0",mode);
``````

等效于：

``````C++
newfd = duo(0);
``````

所以文件描述符newfd和0共享一个文件表项 如若0之前被只读打开 那么我们也只能对newfd进行读操作

即，系统忽略打开模式 但下列打开仍然成功

``````C++
fd = open("/dev/fd/0",RDWR);
``````

我们仍然不能对newfd进行写操作

但Linux有所不同 它的/dev/fd映射为指向底层物理文件的符号链接 当打开/dev/fd/0时实际上打开了一个与标准输入相关联的文件 因此,返回的文件描述符的模式与/dev/fd/0无关

### 3.13 原子操作

早期的open函数并不支持 O_APPEND 选项 考虑一个进程要将数据写到末尾,它必须进行以下程序

``````C++
lseek(fd,0,SEEK_END);
write(fd,buf,sizeof(buf));
``````

若有AB进同时对同一文件进行写操作  

1. A进程执行lseek操作，将文件偏移量设置为1500
2. 内核切换 B执行lseek函数 将文件偏移量设置为1500
3. B进行写入，此时文件长度被增加至1600
4. 内核切换 A进程写入 由于文件表项相互独立 A仍然从1500偏移量开始写，此时刚才写入的数据就会被覆盖

问题的逻辑出自于 先定位，再操作 解决的方法是将这两步操作作为原子操作  
任何大于一个函数调用的操作都不是原子操作

#### pread函数 和 pwrite函数

``````C++
#include<unistd.h>

ssize_t pread(int fd,void *buf,size_t nbytes,off_t offset);
ssize_t pwrite(int fd, void *buf,size_t nbytes,,off_t offset);
``````

**pread返回值**: 读到的字节数 读到末尾返回0 出错返回-1
**pwrite返回值**: 返回已写的字节数 出错返回-1

pread函数和pwrite函数 相当于调用lessk 再调用 read和wirte但又有重要区别  

* 调用pread时不能中断定位和读操作
* 不更新当前的偏移量

