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

# Chapter 4 文件和目录
## 4.4 设置用户ID和设置组ID
与一个进程相关联的ID有6个或更多。实际用户ID和实际组ID标识我们是谁，取自登录时口令文件中的登录项。有效用户ID、有效组ID和附属组ID决定文件访问权限。保存的设置用户ID和设置组ID包含有效用户和有效组的拷贝。\
用户执行文件时有效用户和实际用户通常一致。可以设置标志位使得有效用户ID为文件所有者的用户ID，如果文件所有者是超级用户则文件执行时进程具有超级用户权限。

## 4.12 文件长度
对于符号链接文件长度是文件名中的实际字节数。\
有空洞的文件的长度为包含空洞的长度，但是du命令报告的磁盘块个数为存放字节的个数。如果复制一个空洞文件，所有的空洞被填满，磁盘块个数会大幅上涨（这时du命令返回的大小与ls可能仍然不同，因为文件系统使用了多余的块存放指向实际数据块的指针）。

## 4.14 文件系统
磁盘可以分成多个分区，每个分区包含一个文件系统，一个文件系统包括boot blocks, super block, 以及多个cylinder group。每个cylinder group包括多个i-node和实际的目录块数据块。i-nodes是包含文件信息的固定长度的项。\
一个目录块中的项包含文件名以及对应的i-node编号，两个目录项可以指向同一个i-node，该i-node指向对应的数据块。每个i-node有一个计数器记录指向它的目录项的个数，只有该值为0对应的文件才能被删除，因此unlink一个目录项不代表删除该文件。这种链接被称为硬链接。\
另一种链接是符号链接，符号链接的文件内容（数据块）保存符号链接指向的文件名。这种文件的i-node中的文件类型为S_IFLINK。\
文件重命名时只需要构造一个指向现有i-node的新目录项，并且删除老的目录项，链接计数不会改变。\
对于一个目录而言，如果是不包含任何子目录的目录，链接计数为2，包含命名该目录的目录项（该目录本身）以及该目录的"."目录项。其他目录至少链接计数至少为3，因为至少有该目录本身，该目录的"."目录项，以及子目录的".."目录项。因此每个子目录使得父目录的链接计数加1。

## 4.15 函数link、linkat、unlink、unlinkat和remove
```C++
int link(const char *existingpath, const char *newpath);
int linkat(int efd, const char *existingpath, int nfd, const char *newpath, int flag);
```
创建一个新目录项，引用现有文件。linkat的现有文件由efd和existingpath指定，相对路径通过文件描述符计算。现有文件时符号链接时通过flag控制新链接指向符号链接本身还是符号链接指向的实际文件。创建新目录项和增加链接计数是原子操作。\
只有超级用户可以创建指向目录的硬链接，否则可能会造成循环引用。

```C++
int unlink(const char *pathname);
int unlinkat(int fd, const char *pathname, int flag);
int remove(const char* pathname);
```
删除目录项，所引用文件的链接计数减1。关闭一个文件时首先检查打开该文件的进程个数，如果为0则检查链接计数，如果也为0则可以删除文件。open或者creat创建一个文件，即使立即unlink，由于该文件被进程打开，所以不会立即删除，这可以确保程序崩溃时临时文件不会遗失。如果pathname是符号链接则unlink删除符号链接而不是该链接引用的文件。对于文件remove与unlink功能相同。对于目录remove与rmdir功能相同。

## 4.17 符号链接
符号链接避开硬链接的限制，如要求链接和文件位于同一文件系统中，且只有超级用户可以创建指向目录的硬链接。\
符号链接可能在文件系统中引入循环，例如在目录中创建一个指向目录本身的符号链接，这时大多数查找路径名的函数都会出错。这种循环可以通过unlink符号链接消除，因为unlink的参数为符号链接时unlink不会“follow”该符号链接，即只处理符号链接本身的文件而不处理符号链接指向的文件。如果是硬链接的话unlink就无法消除该循环，因此link函数不允许构造指向目录的硬链接。

# Chapter 7 进程环境
## 7.3 进程终止
调用exit或者_exit或者_Exit是正常终止。_exit和_Exit立即进入内核，exit先执行一些清理处理（对所有打开的流调用fclose函数，使得所有缓冲数据被冲洗到文件上）。main函数返回整型值与用该值调用exit等价，即exit(0)等价于return (0)。

一个进程可以最多登记32个函数（exit handler）由exit调用，通过atexit传入一个函数地址进行登记。POSIX.1中如果exec函数族中任意函数被调用，则所有exit handler被清除。

内核使程序执行的唯一方法是调用exec函数，exec调用用户进程的C启动例程，之后调用main函数。

## 7.5 环境表
每个程序能接受到一张环境表，是一个字符指针数组，与参数表类似，全局变量environ指向该指针数组，每个字符串为环境字符串，保存环境变量等。

## 7.6 C程序的存储空间布局
正文段：CPU执行的机器指令，通常只读，在内存中可以被共享。初始化数据段：即数据段。未初始化数据段：bss段在程序开始执行前将此段数据初始化为0或者空指针，不存放于程序文件中。栈。堆位于栈和未初始化数据段之间。

## 7.7 共享库
使用gcc -static阻止使用共享库，则以静态链接的方式编译，可执行文件大小很大。

## 7.8 存储空间分配
malloc的初始值不确定，calloc为指定数量指定长度的对象分配存储空间，初始化为0.realloc增减之前分配区的长度，可能需要将之前分配区的内容移到另一个足够大的区域，尾端新增区域的初始值不确定。这些分配函数返回的指针是对齐的。realloc对应的存储区可能会移动位置，所以不应将指针指向这些存储区。free释放的存储空间被送入可用存储区池，可用于之后的再次分配。\
分配的实际空间可能大于参数指定的空间大小，额外的空间用来记录一些管理信息，如分配块的长度，指向下一个分配块的指针等等。因此在超出分配区尾端或者起始位置之前进行写操作，可能会改写另一个块的管理信息。

## 7.9 环境变量
环境变量的解释完全取决于应用程序，例如shell设置了大量环境变量。使用getenv函数获取指定环境变量（而不是直接访问environ全局变量）。使用putenv覆盖之前的环境变量，使用setenv覆盖rewrite不为0的环境变量。setenv传递参数时必须为参数重新分配空间，putenv可以将栈中的字符串直接当参数传进去，因此可能会发生错误。\
环境表和环境字符串通常占用进程地址空间的顶部，不能向高地址扩展，也不能移动下方的栈，因此空间本身长度无法增加。如果修改某个环境变量时字符串长度变大，则需要为环境字符串分配新的空间。如果增加一个环境变量，则必须为环境表重新分配空间，并将原来的表放到的新的分配区，将environ指针指向新的表，如果表位于栈顶之上，则需要被放进堆中，不过大多数指针仍然指向栈顶之上的环境字符串。如果不是第一次增加一个环境变量，则只需调用realloc重新分配之前malloc出来的环境表。

## 7.10 函数setjmp和longjmp
goto语句无法跨越函数，使用setjmp和longjmp执行这种跳转功能。嵌套函数调用时，如果深层调用发现一个非致命性错误，可能需要逐层检查返回值。可以通过setjmp和longjmp函数直接跳转回指定调用栈帧。\
在希望返回到的位置调用setjmp，直接调用时返回值为0。参数为一个特殊类型jmp_buf，保存了在调用longjmp时恢复栈状态所需的所有信息，通常为一个全局变量，因为需要在函数间共享。当深层函数希望跳转回上层函数时，调用longjump，第一个参数为之前调用setjmp时传入的jmp_buf类型，第二个参数是一个非零整数，最终成为setjmp的返回值，此整数可以让我们能够调用多个longjmp，并在setjmp处通过不同的返回值知道是从哪里跳转回来的。

当longjmp跳转回上层函数时，大多数情况下局部变量和寄存器变量的值不会被回滚，可以定义变量为volatile使得局部变量不回滚。\
全局变量、静态变量和volatile变量不受优化的影响，longjmp后它们的值为最近被赋予的值。优化之后，局部变量和register变量都存放在寄存器中，volatile变量在内存中。因此内存中的变量具有longjmp调用时的值，而CPU和寄存器中的变量恢复为调用setjmp的值。


# Chapter 14 高级I/O
## 14.2 非阻塞I/O
低速系统调用可能使得进程永远阻塞，例如文件类型数据不存在时读操作会永久阻塞调用者。非阻塞I/O使得open、read、write等操作不会永远阻塞，如果出错则立即返回。可以通过指定对应的文件描述符为O_NONBLOCK标志，如果描述符已打开还可以调用fcntl设置该标志。\
非阻塞I/O通过轮询的形式不断尝试对应的系统调用，不成功则返回错误。

## 14.3 记录锁
一个进程正在读或者修改文件的某个部分时，记录锁（record locking）可以阻止其他进程修改同一文件区。更合适的术语是字节范围锁（byte-range locking）。

fcntl函数可以实现记录锁功能，对应的cmd参数为F_GETLK、F_SETLK、或者F_SETLKW。第三个参数为指向一个flock结构的指针：
```C++
struct flock {
    short l_type; // F_RDLCK, F_WRLCK, F_UNLCK
    short l_whence; // SEEK_SET, SEEK_CUR, SEEK_END
    off_t l_start; // offset in bytes, related to l_whence
    off_t l_len; // length in bytes, 0 means lock to EOF
    pid_t l_pid; // returned with F_GETLK
};
```
l_start和l_whence组合表示锁区域的起始字节偏移量（与lseek类似，通过这两个参数以及当前偏移量计算需要的偏移量）。l_len为加锁的范围，为0则表示不论写多少数据锁的范围都直到文件末尾。\
多个进程对于一个给定的字节可以共享一个读锁（F_RDLCK），但是只能有一个进程享有写锁（F_WRLCK）。如果字节上已经有读锁则不能再加写锁。如果进程连续对某一个区间加不同的锁，则最后一次的锁最终有效。加读锁时描述符必须是读打开，写锁必须是写打开。\
fcntl函数参数为F_GETLK时判断flock指针参数描述的锁是否会被存在的锁排斥，那么flock指针的内容被重写为该存在的锁的信息。如果没有存在的锁排斥，那么flock指针的l_type被改为F_UNLCK。如果是本进程自己加的锁则不会被报告。如果参数为F_SETLK，则设置对应的锁，如果不成功则返回错误。F_SETLKW是F_SETLK的阻塞版本，如果加锁失败则对应进程休眠，如果锁可用或者休眠被信号中断则进程被唤醒。如果某一文件区间频繁被加读锁，那么如果某个进程想要加写锁，可能需要等待很长的时间。\
释放锁时如果释放字节区间中间的某一部分，则加锁的区域会自动分割成两部分，内核将维护两把锁。

锁与进程和文件本身关联。因此进程终止时建立的锁全部释放，并且描述符被关闭时，该进程通过此描述符引用的文件本身被加的锁都会被释放。因此如果通过dup该描述符（多个描述符指向一个文件表项）或者open同一文件路径（每个描述符指向单独的文件表项，这些文件表项指向同一个v-node）使得多个描述符关联同一文件，那么关闭一个的时候对应文件的锁就会被释放。\
fork出来的子进程不继承父进程的锁。子进程通过fork继承来的描述符需要调用fcntl才能获得自己的锁。执行exec后新程序可以继承原程序的锁，如果一个描述符是close-on-exec，则exec后对应的文件的锁被释放。\
FreeBSD的实现中v-node项包含指向lockf结构的指针，用来描述文件对应的锁的列表。\
给相对于文件尾端的字节范围加锁时，不能先调用fstat得到文件长度再去计算偏移量并且加锁，因为此操作不是原子的。事实上相对于SEEK_SET指定绝对偏移量，使用SEEK_CUR或者SEEK_END对相对某个点指定偏移量时都有可能出现相似问题，因为当前偏移量和文件尾端不断变化，不过变化不影响现有锁的状态，因此内核记录对应的锁时必须独立于这些相对位置。

## 14.4 I/O多路转接
当同时读写多个文件描述符时，不能阻塞任意一个，因为不知道哪个输入会得到数据。可以使用非阻塞I/O轮询，不过浪费大量CPU时间。或者使用异步I/O，进程通知内核当描述符准备好I/O时用信号通知。但是可移植性不佳，且可用信号个数可能远远小于文件描述符数量。\
I/O多路转接构造一张描述符列表，通过调用函数，直到描述符中的一个准备好时该函数才返回，返回时告知进程哪些描述符已经准备好。

### 14.4.1 函数select和pselect
```C++
int select(int maxfdpl, fd_set *restrict readfds, fd_set *restrict writefds, fd_set *restrict exceptfds, struct timeval *restrict tvptr);
```
该函数返回就绪的描述符数目，超时返回0，出错返回-1。tvptr指定愿意等待的时间长度，为NULL则表示永远可以等待，0则表示不等待，测试所有描述符后直接返回。\
三个fd_set结构为指向描述符集的指针，说明了可读、可写、或者异常条件的描述符集合。可以将fd_set看做字节数组，每个可能的描述符为1位。如果三个fd_set都为NULL，则select函数相当于比sleep的时间更精确的一个等待函数。\
maxfdpl表示我们关心的最大描述符加1。可以直接设置为FD_SETSIZE（通常为1024），不过这个常量通常过大，因为一个应用可能只需个位数的描述符。这个参数使得内核只用在指定范围内寻找打开的描述符。\
关于描述符就绪，readfds中的描述符就绪表示read操作不会阻塞，writefds中的write操作不会阻塞，exceptfds中的描述符的异常条件正在pending。普通文件的描述符总是返回就绪。到达文件尾端时select认为该描述符可读，调用read返回0。\
pselect的时间精度更高，超时值被声明为const（pselect不会改变此值），且参数包含信号屏蔽字sigmask。

### 14.4.2 函数poll
```C++
int poll(struct pollfd fdarray[], nfds_t nfds, int timeout);

struct pollfd {
    int fd;
    short events; // events of interest on fd
    short revents; // events occurred on fd
};
```
poll不为每个条件（读、写、异常）构造单独的描述符集合，而是通过pollfd结构的数组，每个数组元素指定一个描述符编号以及感兴趣的条件。数组元素个数由nfds指定。timeout为-1则永远等待，0则不等待。一个描述符被挂断（POLLHUP，revents中返回）后不能再被写，但是仍有可能被读。

## 14.5 异步I/O

## 14.6 函数readv和writev
```C++
ssize_t readv(int fd, const struct iovec *iov, int iovcnt);
ssize_t writev(int fd, const struct iovec *iov, int iovcnt);

struct iovec {
    void *iov_base;
    size_t iov_len;
};
```
这两个函数用于在一次调用中读写多个非连续缓冲区。iov数组的元素个数由iovcnt指定。每个iov_base指向一个缓冲区。writev返回输出的字节总数，通常等于所有缓冲区长度之和。readv返回读到的字节总数。\
如果复制缓冲区后调用一次write，当数据增加时效率要低于writev，因为write需要将多个缓冲区复制到一个staging缓冲区，之后write调用时内核将该缓冲区的数据复制到内核缓冲区。writev调用中内核则直接将数据复制进自己的缓冲区，因此复制工作较少。

## 14.7 函数readn和writen
```C++
ssize_t readn(int fd, void* buf, size_t nbytes);
ssize_t writen(int fd, void* buf, size_t nbytes);
```
对于管道、FIFO以及某些设备（终端和网络等），一次read操作返回的数据少于要求的数据，write的返回值可能少于指定输出的字节数（例如内核输出缓冲区变满）。上述两个函数用于处理这种情况，按需多次调用read和write直至读写数据量达到要求。

## 14.8 存储映射I/O
存储映射I/O将磁盘文件映射到存储空间的缓冲区上，这样从缓冲区取数据相当于读文件中的相应字节，将数据存入缓冲区相当于自动写入文件。这样I/O可以在不使用read和write时执行。
```C++
void *mmap(void* addr, size_t len, int prot, int flag, int fd, off_t off);
```
addr指定存储区的起始地址，通常为0，表示系统选择起始地址。函数返回值也是起始地址。fd参数为被映射文件的描述符。映射前需要先打开该文件（分配文件描述符、文件表项等等）。off和len为要映射的字节在文件中的偏移量以及映射区域长度。prot指定映射区的读写执行权限，该权限不得超过open的访问权限。\
映射存储区可以位于堆和栈之间。flag可以设置为MAP_FIXED，表明返回值必须等于addr，这不利于移植性（将addr设为0可获得最大可移植性）。MAP_SHARED表明对映射缓冲区的存储操作相当于直接write对应文件。MAP_PRIVATE表明存储操作会创建一个对应的文件的私有副本，后来对映射区的引用都是引用该副本（可以用于调试）。\
off和addr的值通常需要是虚拟页的长度的倍数，此参数可以用带_SC_PAGESIZE或者_SC_PAGE_SIZE的sysconf函数得到。如果映射区长度不是页长度的整数倍，那么没被填满的页用0填满。如果要通过修改这些用于填充页的字节来修改文件，需要先加长文件。\
与映射区相关的信号有SIGSEGV和SIGBUS，SIGSEGV表示进程试图访问不可用的存储区，例如试图向只读映射区存储数据。如果映射区某个部分访问时已经不存在，则产生SIGBUS信号，例如用文件长度映射了一个文件，但是另一个进程将该文件阶段，则试图访问截断部分会接收到SIGBUS信号。\
子进程可以通过fork继承存储映射区（因为子进程复制父进程地址空间），新程序不通过exec继承存储映射区。

```C++
int mprotect(void *addr, size_t len, int prot);
int msync(void *addr, size_t len, int flags);
```
调用mprotect更改现有映射的权限。\
如果修改的页通过MAP_SHARED映射，那么修改并不立即写回文件，写回脏页的时间由守护进程决定。可以调用msync将脏页冲洗到被映射的文件中，功能类似于fsync。如果映射为私有的，则对应的文件本身不会被修改。可以通过flag参数对如何冲洗存储区进行设置，如果希望写操作在返回前完成，则设置为MS_SYNC。\
```C++
int munmap(void *addr, size_t len);
```
进程终止时会自动解除存储映射区的映射，或者直接调用munp函数解除映射关系。关闭对应的文件描述符不会解除映射关系。调用munmap不会使得映射区的内容被写入文件。对于MAP_SHARED，写入文件是由内核的算法控制的。对于MAP_PRIVATE，更改在映射区解除映射后被丢弃。

可以用过fstat得到文件长度，作为mmap的参数。对于输出文件，可以通过ftruncate函数设置长度，如果不设置，mmap依然会成功，但是对对应映射区的第一次引用会产生SIGBUS信号。\
可以通过mmap和memcpy结合来复制文件，相比于read和write执行了较少的系统调用次数。read和write将数据从内核缓冲区复制到应用缓冲区，再复制回内核缓冲区。mmap和memcpy直接将数据从映射到地址空间的一个内核缓冲区复制到另一个内核缓冲区，这个复制过程是作为一个页错误的handler执行的，因为我们会引用不存在的页（每次读或者写一个新的页都会产生页错误。这样在系统调用和额外的复制操作以及页错误处理之间产生了一个tradeoff。

# Chapter 15 进程间通信
## 15.2 管道
管道只能在有公共祖先的两个进程使用，通常一个管道由一个进程创建，fork之后父进程和子进程之间可以使用。半双工管道是最常用的IPC形式，每当键入一个命令序列让shell执行时，shell为每一条命令单独创建一个进程，用管道将前一条命令的标准输出与后一条命令的标准输入连接。\
管道通过pipe函数创建，参数fd返回两个文件描述符，fd[0]为读打开，fd[1]为写打开，fd[1]的输出是fd[0]的输入。fstat函数对管道每一端返回一个FIFO类型的文件描述符。
```C++
int pipe(int fd[2]);
```
通常进程先调用pipe再调用fork创建父子进程之间的IPC通道，对于父进程到子进程的管道，父进程关闭读端（fd[0]），子进程关闭写端（fd[1]）。如果write一个读端被关闭的管道，产生信号SIGPIPE，如果忽略该信号或者捕捉该信号并从其处理程序返回，则write返回-1，errno为EPIPE。常量PIPE_BUF规定管道内核缓冲区大小，如果有多个进程同时写一个管道，要写的字节数超过PIPE_BUF，则所写数据会与其他进程所写的数据交叉。可以通过dup2(fd[0], STDIN_FILENO)将标准输入作为管道的读端。

## 15.3 函数popen和pclose
```C++
FILE *popen(const char *cmdstring, const char *type);
int pclose(FILE *fp);
```
popen先执行fork，然后调用exec执行cmdstring，返回一个标准文件I/O指针，如果type为"r"则文件指针连接到cmdstring进程的标准输出，"w"则连接到其标准输入（可使用dup2作为内部实现）。pclose函数关闭标准I/O流，等到命令终止，返回shell终止状态。\
popen每次被调用时需要记住所创建的子进程id以及其文件描述符或者FILE指针。FILE指针可以使用fdopen函数从文件描述符获取。实现时可以在一个数组中保存子进程id并且使用文件描述符作为下标。当pclose被调用时，可以使用fileno标准I/O函数从FILE指针得到对应的文件描述符，从而取得对应的子进程id，用于作为waitpid的参数。该子进程id数组在第一次动态分配的时候长度应该为最大文件描述符个数。\
POSIX.1要求popen关闭在之前的popen调用中被打开且仍然在子进程中打开的流。因此需要在子进程中遍历子进程id数组，关闭打开的描述符。如果pclose的调用者为SIGCHLD信号建立了一个处理器，则pclose中的waitpid返回返回EINTR错误，如果调用者捕获了信号，waitpid被中断，则重新调用一次（while循环）。如果调用者自己调用了waitpid并获取了popen创建的子进程退出的状态，pclose再次调用waitpid时会发现子进程不存在，则errno为ECHILD，对于不是EINTR的错误，pclose直接返回-1，说明执行出错。

popen可以用来在应用程序和标准输入之间插入一个过滤器程序，对用户输入进行处理，过滤器接收用户输入，进行处理后通过popen的管道将处理后的数据通过自己的标准输出发送给真正处理用户输入的进程。

## 15.4 协同进程
过滤程序既接受另一个过滤程序的输入，又读取该过滤程序的输出，则成为协同进程。这时需要通过pipe创建两个管道，一个用作协同进程的标准输入，一个用作标准输出。父进程和子进程各自关闭不需要的管道端，子进程调用dup2函数将管道描述符移至标准输入（将管道的输出作为自己的标准输入）和标准输出，之后调用execl函数在子进程中执行特定的协同进程程序。\
协同进程如果使用标准I/O（fgets、printf等）而不是read/write系统调用读取数据，则程序不再工作。因为标准输入输出为管道，标准I/O库默认为全缓冲，因此协同进程被阻塞在读取标准输入时，父进程阻塞在读取管道，产生死锁。

## 15.5 FIFO
FIFO也称为命名管道，可以使得不相关的进程交换数据。FIFO是一种文件类型，创建FIFO类似于创建文件。
```C++
int mkfifo(const char* path, mode_t mode);
int mkfifoat(int fd, const char *path, mode_t mode);
```
使用mkfifo和mkfifoat创建FIFO后使用open打开，read/write等文件I/O函数也可以使用在FIFO上。如果不设置O_NONBLOCKING标志，则只读/写open阻塞到另一个进程写/读打开该FIFO为止，设置的情况下只读open立即返回，只写open返回-1。write一个没有被读打开的FIFO产生信号SIGPIPE。一个给定FIFO可以有多个写进程。\
FIFO可以用于shell命令将数据从一条管道传送到另一条，无需创建中间临时文件。还可以用于在客户端进程和服务器进程之间传递数据。每个服务器进程可以关联多个客户进程，客户进程将请求写到公共的FIFO中（请求长度小于PIPE_BUF，避免一个客户进程写多次，出现交叉）。服务进程返回响应时根据客户进程ID创建一个仅属于客户进程的FIFO。这种情况中服务器进程必须捕捉SIGPIPE信号，因为客户端可能没有读取响应就终止了，这时对应的FIFO是只有写进程没有读进程。

## 15.6 XSI IPC
IPC结构（消息队列、信号量、共享内存）使用非负整数标识符被引用。该标识符与文件描述符不同，不是小的整数，IPC结构被创建时相关标识符连续加1，直到达到整型最大值，然后转回到0。每个IPC对象与一个键相关联，作为外部命名。键由内核变换为标识符。

服务器进程可以在指定键IPC_PRIVATE上创建新的IPC结构，返回的标识符存放在文件中以便客户进程取用，IPC_PRIVATE保证创建的是新的IPC结构。\
可以在公用头文件中定义一个客户进程与服务器进程都认可的键，如果该键已经与某个IPC结构关联，则服务器进程负责处理错误重新创建。\
客户进程和服务器进程也可以认同一个路径名和0-255之间的项目ID，调用ftok函数将这两个值变换为一个键。

每个IPC关联一个ipc_perm结构，定义权限与所有者，修改这些字段类似于对文件调用chown和chmod。

IPC结构在系统范围起作用，没有引用计数。例如进程创建一个消息队列，并在队列中放入了几则消息，然后终止，则该消息队列以及内容会留在系统中，不会被删除。直到某个进程调用msgrcv读消息或者msgctl/ipcrm(1)删除队列。对于管道而言，最后一个引用管道进程终止后管道被完全删除，对于FIFO而言，最后一个引用FIFO的进程终止时，FIFO的名字保留在系统中，数据已经被删除。\
IPC结构在文件系统中没有名字，因此不能用访问文件的函数访问或者修改，因此内核中增加了十几个全新系统调用。不能用ls命令查看IPC对象，不能用rm命令删除，不能用chmod命令修改权限。\
IPC不使用文件描述符，所以不能使用I/O多路复用函数。因此很难一次使用多个IPC结构或者在文件或设备I/O使用IPC结构。

由于需要某种技术获得队列标识符，消息队列不是无连接的（无连接指无需调用某种形式的打开函数就能发送消息的能力）。

## 15.7 消息队列
消息队列是消息的链接表，由消息队列标识符标识（队列ID）。msgget创建新队列或者打开现有队列，msgsnd添加新消息到队列尾端。每个消息包含一个unsigned long表示消息类型，一个非负长度以及实际数据字节。msgrcv用于从队列中取消息。消息先进先出，也可以按照消息类型字段取消息。每个队列关联一个msqid_ds结构，定义队列当前状态：
```C++
struct msqid_ds {
    struct ipc_perm msg_perm; // permissions
    msgqnum_t msg_qnum; // number of messages on queue
    msglen_t msg_qbytes; // max number of bytes on queue
    pid_t msg_lspid; // pid of last msgsnd()
    pid_t msg_lrpid; // pid of last msgrcv()
    time_t msg_stime; // last msgsnd() time
    time_t msg_rtime; // last msgrcv() time
    time_t msg_ctime; // last change time
    // ...
}
```
```C++
int msgget(key_t key, int flag);
```
msgget得到一个key，通过内核变换为标识符。ipc_perm中的mode成员按照flag中的权限位，msqid_ds其余位设为0或者当前时间或者系统限制值。
```C++
int msgctl(int msqid, int cmd, struct msqid_ds *buf);
```
cmd参数指定对msqid队列执行的命令，IPC_STAT获取对应的msqid_ds结构放入buf，IPC_SET将buf结构的字段设置到队列相关的结构中。IPC_RMID删除消息队列以及仍在队列中的所有数据。
```C++
int msgsnd(int msqid, const void* ptr, size_t nbytes, int flag)
```
ptr指向一个unsigned long表示消息类型以及紧跟的消息字节，nbytes为消息字节的长度。消息队列删除时仍在使用队列的进程会返回错误，而不是等到最后一个进程结束后才删除队列内容。
```C++
ssize_t msgrcv(int msqid, void *ptr, size_t nbytes, long type, int flag);
```
ptr存储返回的消息类型，nbytes为数据缓冲区长度。type为0则返回队列头的消息，type>0则返回对应type的第一条消息，type<0则返回消息类型小于等于type绝对值的消息，如果有若干个则取类型值最小的消息。

客户进程和服务器进程的双向数据流可以使用消息队列或者全双工管道。

## 15.8 信号量
信号量的值为正则进程可以使用资源，进程将信号量的值减1，如果信号量为0则进程休眠直到信号量的值大于0。进程不再使用资源时信号量的值加1。信号量的读取和减一操作应该为原子的。\
信号量的创建（semget）是独立于初始化（semctl）的，因此不能原子地创建信号量集合。
```C++
int semget(int key, int nsems, int flag);
```
key被变换为标识符，可以创建新集合或者引用现有集合，nsems是集合中信号量的个数。
```C++
int semctl(int semid, int semnum, int cmd, union semun arg);
union semun {
    int val;
    struct semid_ds *buf;
    unsigned short *array;
};
```
cmd命令运行在semid指定的信号量集合上，semnum指定信号量集合的一个成员。IPC_STAT获取semid_ds结构，IPC_SET设置对应的semid_ds，IPC_RMID删除对应的信号量集合。GETVAL/SETVAL返回或设置对应的信号量成员的值。GETPID返回最后一次操作对应信号量成员的pid，GETNCNT/GETZCNT返回信号量成员对应的正在等待的进程数量，GETALL/SETALL操作所有的信号量值。
```C++
int semop(int semid, struct sembuf semoparray[], size_t nops);
struct sembuf {
    unsigned short sem_num; // 信号量成员
    short sem_op; // 操作
    short sem_flg; // IPC_NOWAIT, SEM_UNDO
};
```
semop执行信号量集合上的操作数组，操作为正值时如果没有指定undo标志则信号量加上该值，指定undo标志则减去该值。操作为负值则表明进程想获得资源，信号量的值大于sem_op的绝对值值则可以获取，undo标志指定时信号量的值加上sem_op的绝对值。信号量的值小于sem_op绝对值时，如果进程选择阻塞则挂起（semncnt加1），否则返回错误。sem_op为0则表明进程希望等待到信号量的值为0。\
SEM_UNDO标志是的内核记住信号量对应的进程分配了多少资源，进程终止时按照相应的值对信号量进行处理。

## 15.9 共享存储
共享存储允许多个进程共享给定的存储区，数据不需在客户进程和服务器进程之间复制，所以是最快的IPC。信号量可用于同步共享内存的访问。共享内存与内存映射的区别在于共享内存没有相关的文件。

shmget函数可指定共享内存段的长度。shmctl函数还支持SHM_LOCK以及SHM_UNLOCK操作，用于对共享内存加锁或解锁。
```C++
void* shmat(int shmid, const void* addr, int flag);
```
addr为0则连接到系统选择的第一个可用地址上，addr不为0时，没有指定SHM_RND则直接连接到addr，指定SHM_RND时取整。SHM_RDONLY表示只读共享。shmat返回实际地址。
```C++
int shmdt(const void* addr);
```
shmdr函数解除与共享内存段的连接，addr参数为之前调用shmat的返回值。

对于相关的进程，可以使用/dev/zero设备，调用mmap创建映射区且无需存在实际文件。\
匿名存储映射中，mmap调用时指定MAP_ANON标志，文件描述符为-1，得到一个匿名区域，可与后代进程共享。

## 15.10 POSIX信号量
POSIX信号量无需操作信号量集合，删除时等到信号量最后一次引用被释放后才停止工作。\
未命名信号量只存在于内存中，要求使用信号量的进程必须可以访问内存（同一个进程的线程或者映射到相同内存的线程）。命名信号量可以直接通过名字访问。
```C++
sem_t *sem_open(const char *name, int oflag, ...)
```
sem_open创建或者使用现有信号量，oflag参数设置O_CREAT时创建新的，并额外指定mode参数指定权限，并指定一个初始值。该函数返回一个信号量指针，可以作为其他信号量函数的参数。设置O_EXCL保证只创建一个。\
sem_close释放信号量相关资源。内核可以自动关闭打开的信号量。sem_unlink函数删除信号量的名字，等到最后一个打开信号量的引用关闭后销毁信号量。
```C++
int sem_trywait(sem_t *sem);
int sem_wait(sem_t *sem);
```
wait用于信号量减一，trywait为非阻塞的。阻塞一段时间可以用sem_timewait。sem_post对信号量增1，释放资源，并唤醒阻塞在wait函数上的进程。
```C++
int sem_init(sem_t* sem, int pshared, int value);
```
sem_init创建未命名信号量，pshared表明是否在多个进程使用，value为初始值，sem为初始化后的信号量。如果多个进程共享则sem指向进程共享的内存。sem_destroy丢弃使用完的信号量。sem_getvalue获取信号量的值。

## 15.11 客户进程-服务器进程属性
