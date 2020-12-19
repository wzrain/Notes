# Chapter 1 UNIX基础知识
## 1.5 输入与输出
文件描述符通常是一个小的非负整数，kernel uses it to identify files accessed by a process.\
open、read、write、lseek、close提供了不带缓冲的I/O，均使用文件描述符。标准输入输出对应的文件描述符为0和1。\
标准I/O函数为不带缓冲的I/O函数提供了带缓冲的接口，无需自己选取缓冲区大小作为函数参数。

# Chapter 3 文件I/O
与标准I/O相对照，文件I/O函数通常被称为“不带缓冲”的I/O，意思是指函数直接调用内核的系统调用。

## 3.2 文件描述符
打开一个现有文件或者创建一个新文件时，内核向进程返回一个文件描述符。读写文件时使用文件描述符标识该文件。

## 3.3 函数open和openat
```C++
#include <fcntl.h>
int open(const char *path, int oflag, ...);
int openat(int fd, const char* path, int oflag, ...);
```
oflag参数为打开文件的选项。这两个函数返回的文件描述符是最小的未用描述符的值。\
如果path是绝对路径则open相当于openat。path是相对路径的话，fd是一个文件描述符，指定这个相对路径在文件系统中的起始位置，这个参数通过打开相对路径所在的目录获得。\
这样openat使得线程可以使用相对路径打开除了当前工作目录之外其他目录下的文件。此外openat还可以帮助防止time-of-check-to-time-of-use (TOCTTOU)错误。如果一个file-based函数依赖另一个file-based函数的结果，那么文件可能在两个函数调用中间被修改，导致第一次调用的结果失效。TOCTTOU错误可能出现在有人试图降低privileged文件的权限或者更改privileged文件来打开一个security hole。

## 3.4 函数creat
```C++
int creat(const char* path, mode_t mode);
```
等价于
```C++
open(path, O_WRONLY | O_CREAT | O_TRUNC, mode);
```

## 3.5 函数close
关闭一个文件时释放进程加在该文件上的记录锁。

## 3.6 函数lseek
```C++
#include <unistd.h>
off_t lseek(int fd, off_t offset, int whence);
```
读写操作从当前偏移量开始，使得偏移量增加读写的字节数，打开文件时除非指定O_APPEND否则为0。\
lseek显式地为打开的文件设置偏移量。whence参数为SEEK_SET则设置为距离文件开始处offset个字节，为SEEK_CUR则给当前偏移量加offset，为SEEK_END则偏移量为文件长度加offset。offset可正可负。函数返回新的偏移量。如果文件描述符指向管道、FIFO或者套接字，则返回-1，errno设置为ESPIPE。由于某些设备偏移量可以能为负值，判断时必须判断是否为-1，而不是判断是否小于0。\
文件偏移量可以大于当前长度，下一次写的时候会在文件中构成一个空洞，没有被写过的字节都被读为0。这一部分不需要分配磁盘块。

## 3.7 函数read
```C++
ssize_t read(int fd, void* buf, size_t nbytes);
```

## 3.8 函数write
```C++
ssize_t write(int fd, const void* buf, size_t nbytes);
```

## 3.9 I/O的效率
如果磁盘块长度为4096字节，则缓冲区长度为4096的时候系统CPU时间最小，后面成倍增加的时候CPU时间没有有效提高。\
大多数文件系统采取某种预读（read ahead）技术改善性能。当检测到正在进行顺序读取时，系统试图读入比应用所要求的更多数据，这样即使缓冲区很小的时候时钟时间也不会很大。\
如果重复度量程序性能，后续得到的性能可能好于第一次，因为第一次运行使得文件进入高速缓存。

## 3.10 文件共享
每个进程在进程表中都有一个记录项，包含一张打开文件描述符表，每个描述符占用一项，每一项包含文件描述符flag以及指向一个文件表项的指针。内核对所有打开的文件维护一张文件表，每个文件表项包含文件状态的flag（read、write、append、sync、nonblocking），当前的文件偏移量，以及一个指向对应文件的v-node表项的指针。每个打开的文件或设备有一个v-node结构，包含文件类型信息以及指向操作该文件的函数的指针。对于大多数文件v-node还包含了该文件的i-node。这些信息在文件打开时从磁盘读取。i-node包含文件的所有者、大小、指向保存文件的数据块的指针等等。v-node结构的目的是对多文件系统类型提供支持。\
两个进程打开同一文件时，每个进程获得自己的文件表项，用于拥有进程自己的偏移量，两个文件表项的v-node指针指向同一个v-node。每个write完成后文件表项中的偏移量被更新，如果偏移量大于文件大小，则i-node中的文件大小被更新为当前这个偏移量。如果文件打开时设置了O_APPEND，则文件表项的文件状态flag中的对应位被设置，这样write的时候偏移量会被强行设为i-node中的当前文件大小，保证更改为append。\
可以有多个文件描述符项指向同一个文件表项。fork后父进程子进程的文件描述符共享同一个文件表项。\
文件描述符flag只用于一个进程的一个文件描述符，文件状态flag应用于指向对应文件表项的所有进程。\
以上操作在多个进程同时写同一个文件时可能会有问题，需要使用原子操作解决。

## 3.11 原子操作
如果有两个进程同时append一个文件，首先lseek找到文件尾端，再调用write。这个过程由于两个函数调用之间不是原子的，可能导致一个进程的更改被另一个进程覆盖。UNIX系统提供原子操作方法，在打开文件时设置O_APPEND标志，使得每次写操作自动从文件最后开始，不需要调用lseek。\
XSI扩展允许原子性地定位并执行I/O，使用pread和pwrite函数，调用pread相当于lseek+read，但是pread的定位和读操作无法中断，且当前文件偏移量不更新。\
open时同时制定O_CREAT和O_EXCL选项且文件已存在时open将会失败。如果没有这样的原子操作，在open时发现文件不存在之后尝试creat的过程之间可能会有另一个进程创建相同的文件，这时这一操作就会被覆盖。

## 3.12 函数dup和dup2
```C++
#include <unistd.h>
int dup(int fd);
int dup2(int fd, int fd2);
```
两个函数都可以用来复制现有的文件描述符，dup返回的是当前可用文件描述符中的最小数值。dup2使用fd2指定新的描述符的值，如果fd2已经打开则先将其关闭。如果fd等于fd2则返回fd2而不是关闭，否则的话fd2的FD_CLOEXEC标志被清除，这样如果进程调用exec的话fd2仍然是打开状态。\
新文件描述符与fd共享同一个文件表项，因此共享同一文件状态flag以及偏移量。每个文件描述符都有一套自己的文件描述符flag，它的执行时关闭（close-on-exec）标志总是由dup函数清除（fork后子进程复制父进程后执行exec之前本来需要手动close父进程打开的文件描述符，如果close-on-exec设置则子进程exec前继承自父进程的文件描述符被自动close）。\
调用dup(fd)相当于调用fcntl(fd, F_DUPFD, 0)，dup2(fd, fd2)相当于调用close(fd2)之后调用fcntl(fd, F_DUPFD, fd2)，只不过dup2是原子操作。

## 3.13 函数sync、fsync和fdatasync
```C++
int fsync(int fd);
int fdatasync(int fd);
void sync(void);
```
传统UNIX系统内核中有缓冲区高速缓存或者页高速缓存，用户向文件写数据时数据先被复制到缓冲区，然后排入队列，延迟写入磁盘。sync将修改过的块缓冲区排入写队列，不等待实际写磁盘操作结束。系统守护进程update周期性地调用sync定时flush块缓冲区。fsync只对fd参数对应文件起作用，等待磁盘写操作结束才返回，可以保证修改过的块立即写到磁盘。fdatasync类似fsync，只不过影响文件的数据部分。fsync还会更新文件属性。

## 3.14 函数fcntl
```C++
int fcntl(int fd, int cmd, ...);
```
cmd为F_DUPFD或者F_DUPFD_CLOEXEC时复制已有的文件描述符，为F_GETFD或者F_SETFD则设置文件描述符flag，为= F_GETFL或者F_SETFL则设置文件状态flag，为F_GETOWN或者F_SETOWN则设置异步I/O所有权，为F_GETLK, F_SETLK或者F_SETLKW则设置记录锁（14.3）。\
F_DUPFD复制文件描述符fd，返回值是大于等于第三个参数的可用文件描述符的最小值。新描述符有自己的文件描述符flag，且FD_CLOEXEC被清除。F_DUPFD_CLOEXEC不清除FD_CLOEXEC。\
F_GETFD返回当前文件描述符flag，当前只有FD_CLOEXEC被设置。F_SETFD设置文件描述符flag为第三个参数值。F_GETFL或者F_SETFL类似，返回或者设置文件状态flag，这里五个访问标志（O_RDONLY, O_WRONLY, O_RDWR, O_EXEC, 以及O_SEARCH）并不各占一位，而是互斥的，因此需要使用mask值O_ACCMODE取得访问位再看是五种里哪一种。例如如果设置O_SYNC标志位则表明写为同步的（但是linux中可能不会支持设置O_SYNC标志，如果fcntl参数有O_SYNC，linux不会返回错误，只是不设置该标志位），每次write都要等待写磁盘结束，通常write只是将数据排入队列，数据库系统可能需要同步写。\
F_GETOWN返回接受SIGIO和SIGURG信号的进程或进程组ID，F_SETOWN设置，正的第三个参数表示进程ID，负的第三个参数的绝对值表示进程组ID。

## 3.15 函数ioctl
```C++
int ioctl(int fd, int request, ...);
```
之前一些函数无法实现的I/O操作通常都能用ioctl表示。

## 3.16 /dev/fd
open("/dev/fd/0", mode)等效于dup(0)。可以用/dev/fd提高文件名参数的一致性。