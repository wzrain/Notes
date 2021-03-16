# C++
### static关键字
static局部变量位于全局数据区，可以定义于函数内部，作用域为定义的局部作用域（如定义它的函数内部）。static全局变量的作用域为文件内部，文件外部不可见，而普通全局变量可以通过extern被外部文件访问。\
static函数与static全局变量类似，用于声明仅在该文件内部可以被访问的函数。\
(C++) static静态成员存储于全局数据区，被所有对象共享，相比全局变量不会与其他命名冲突，同时可以通过private等进行访问控制。static静态成员函数没有this指针，不能访问非静态成员与非静态成员函数。

### 源文件到可执行文件的过程
预处理宏展开，编译得到汇编代码，汇编器生成.o二进制文件，链接器将.o文件链接为一个二进制文件，loader将可执行文件加载进内存。\
编译后引用其他文件中的函数变成符号，在链接的时候被解析。预编译阶段已经将头文件包含进来，保证编译器能正确生成唯一且正确的符号名字。链接的时候把符号替换为函数的地址。

静态链接将.o文件与引用的库一起打包到可执行文件中。Linux静态链接库扩展名为.a（win中为.lib），本质上是一组.o文件的集合。静态链接得到的可执行文件移植方便（因为可执行文件已经打包好，不需要运行环境中存在所需的库）。\
静态链接占用大量空间，库更新时需要全量更新。动态链接的库在运行时才被载入，不同的应用程序可以在内存中共享一个库的实例。Linux中通过dlopen函数将.so库加载到进程中，通过动态链接器ld.so链接到当前进程（查找未定义的符号的地址，分配内存，初始化等）。win中如果直接调用动态链接库的函数，在编译时的链接中会将该库的一小段.lib链接进去，包含该.dll的相关信息，运行时可以找到相应的库；或者程序真正运行的时候调用API显式将.dll载入内存。

### 虚函数
虚函数表保存函数指针，如果子类将父类的虚函数重写，则将原先父类虚函数在表中的位置换为子类虚函数。\
多继承情况下，派生类中有多个虚函数表，顺序为继承顺序。如果不是第一个基类，编译器会计算offset，使之访问正确的虚函数表。派生类重写了多个基类的同名虚函数后，非第一个基类的指向派生类的指针调用的时候使用thunk技术跳转到第一个虚函数表对应的位置。派生类新定义的虚函数在第一个基类的虚函数表后面，在后面的基类指针调用的时候借助thunk技术跳转的正确的位置。\
非虚函数并不占用对象内存，而是在以普通函数对待，参数中增加一个this指针。\
虚继承把虚基类的成员放在后面。因此将虚基类指针指向派生类对象时，需要把指针移到后面虚基类成员开始的地方。

### 内存管理
代码段、数据段（初始化的数据）、bss段（未初始化的数据）、堆、栈。\
五个区：堆、栈、全局/静态区、文字常量区、代码区。

### 智能指针
```C++
template <typename T>
class SmartPointer {
private:
    T* ptr;
    size_t *counter;
    void decrement() {
        if (ptr) {
            (*counter)--;
            if ((*counter) == 0) {
                delete ptr;
                delete counter;
            }
        }
    }
public:
    SmartPointer(T *p = 0) : ptr(p), counter(new size_t) {
        if (p) *counter = 1;
        else *counter = 0;
    }

    SmartPointer(const SmartPointer& sp) {
        ptr = sp.ptr
        counter = sp.counter;
        (*counter)++;
    }

    SmartPointer& operator=(const SmartPointer& sp) {
        if (sp.ptr == ptr) return *this;
        decrement();
        ptr = sp.ptr;
        counter = sp.counter;
        (*counter) ++;
        return *this;
    }

    T& operator*() {
        if (ptr) return *ptr;
        // throw exception
    }

    T* operator->() {
        if (ptr) return ptr;
        // throw exception
    }

    ~SmartPointer() {
        decrement();
        if (counter) delete counter;
    }
}
```
循环引用：
```C++
struct Node {
    int val;
    shared_ptr<Node> prev;
    shared_ptr<Node> next;
    Node(int v) : val(v) {}
};

shared_ptr<Node> head = make_shared<Node>(0);
head->next = make_shared<Node>(0);
head->next->prev = head;
// head的引用计数为2，离开作用域时减不到0，不会被析构，因此head->next依旧被引用着，也不会被析构。将prev改为weak_ptr解决循环引用问题。
```

### sizeof
空类sizeof为1。成员函数和静态成员不影响sizeof的结果。多个成员不同排列会产生不同的sizeof结果（内存对齐）。多继承会继承得到多个父类的虚函数表指针。虚继承会多出一个虚基类表来记录虚继承关系。

### 指针 vs. 引用
引用在创建的时候需要被初始化，不可以与NULL关联，且之后不可修改关联的对象。\
指针传递本质上复制了原对象的地址副本，因此原对象本身的地址值不会改变（即又生成了个指向原对象的指针）。\ 
指针传递和引用传递都适用于需要改变参数对象，或者希望函数有多个“返回值”时。\
指针传递可以设置默认参数，易读（易于分辨传的是指针还是值，引用传递和值传递的调用代码是相同的）。如果要传只读的参数，使用const引用。运算符重载需要使用引用。


### gdb
c/continue: 继续执行到下一个断点。\
si/stepi: 执行一条指令。\
b/breakpoint function/file:line/*addr : 在某个函数或者某一行或者某一个指令地址（eip）加断点。\
info registers: 打印通用寄存器、eip、eflags、段选择寄存器等。\
x/Nx addr: 十六进制地显示从addr起始的N个word数据。x/Ni addr: 显示从addr其实的N个汇编指令（addr为$eip时显示当前指令）。\
symbol-file file: 把符号文件转换为file。\
thread n: 设置当前线程为线程n。info threads: 列出所有线程，包括其状态与所在的函数。

## STL
### vector
下标访问没有越界检查（效率高），at会进行越界检查。

push_back平均复杂度：对于扩容因子为m的vector，有n个元素时大约执行了log_m(n)次扩容，因此总共的扩容时间为Σ_{i=1}^{log_m(n)}m^i，即约为nm/(m-1)。

# Operating Systems
### semaphore vs. mutex
信号量管理资源的数量，一定程度上调度线程，mutex管理资源的使用权。

### PCB
process identification: pid, uid, ...
process state: registers, stack and frame pointers, PC, ...
process control: scheduling state (ready, suspended, ..., priorities), process structure (chidren ids, ...), IPC information (flags, signals, ...), memory management information, I/O status, ...

TCB (Thread Control Block): tid, stack pointer, PC, state (running, ready, ...), registers, pointer to PCB

### 条件变量
生产者消费者问题中如果使用互斥锁忙等，可能占用大量CPU。使用条件变量休眠或者唤醒线程：
```C++
std::deque<int> dq;
std::mutex mu;
std::condition_variable cond;

void func1() {
    int cnt = 10;
    while (cnt--) {
        std::unique_lock<std::mutex> locker(mu);
        dq.push_front(cnt);
        locker.unlock();
        cond.notify_one();
        std::this_thread::sleep_for(std::chrono::seconds(1));
    }
}

void func2() {
    int data = INT_MAX;
    while (data) {
        std::unique_lock<std::mutex> locker(mu);
        // can also be cond.wait(locker, [&](){return !dq.empty();})
        while(q.empty())
            cond.wait(locker); 
        data = q.back();
        q.pop_back();
        locker.unlock();
        std::cout << "t2 got a value from t1: " << data << std::endl;
    }
}
```
cond.wait(locker)会调用locker的unlock函数释放锁，线程被唤醒后再尝试重新加锁（因此这里不能使用lock_guard，因为没有lock/unlock成员）。\
这里需要使用while(q.empty())而不是if，因为线程被唤醒可能不是由于队列不空，或者是在重新获得锁之前队列又被清空，因此线程重新获得锁后继续前需要再检查一遍。\
C++还提供notify_all()函数唤醒所有等待线程。

### 孤儿进程、僵尸进程
孤儿进程：父进程退出子进程还没有退出，子进程被init进程（pid=1）收养，init进程负责该孤儿进程的状态收集，不会被系统产生危害。\
僵尸进程：子进程退出后父进程没有调用wait或者waitpid获取退出的子进程状态，则子进程描述符仍然存在于系统中。大量的僵尸进程会占用系统资源，用ps可以看到状态为Z。如果要消灭僵尸进程可以把对应的父进程终止，这样僵尸进程会被init收养，init会释放这些进程。

### 守护进程
守护进程运行在后台，独立于控制终端，周期性执行任务或者等待处理发生的事件。不需要用户输入，对系统或者某个用户程序提供服务。守护进程是非交互式程序，向stdout或者stderr输出都需要特殊处理。\
首先通过fork创建子进程，并使得父进程exit，使得子进程在后台运行。通过setsid，使得子进程成为会话组长（每个进程属于一个进程组，进程组号为进程组长的进程号，一个登录回话（session）包含多个进程组，共享同一个控制终端，通常从父进程继承下来，通过调用setsid使得进程成为新的会话组长和进程组长）。可以通过再创建一个子进程但是不setsid的方式来使得第二子进程无法打开新的控制终端。之后还需要关闭父进程打开的文件描述符，并通过chdir将工作目录设为根目录。将标准输入输出重定向到/dev/null。\
可以直接通过daemon(nochdir, noclose)创建，nochdir为0则表明工作目录改为根目录，noclose表明将输入输出定向到null。

### 多进程 vs. 多线程
多进程共享数据复杂，但是同步简单，占用内存多，CPU利用率低，创建销毁切换复杂（服务器响应等需要频繁创建环境的适合多线程），编程调试简单，一个进程挂掉不影响其他进程（适合对鲁棒性要求较高的系统（如自动驾驶）），易于扩展到多机。

### 多处理器调度
需要注意缓存一致性、数据同步、缓存亲和性。单队列调度实现简单，需要锁来保证原子性（性能损失），缺乏可扩展性，不利于缓存亲和性（例如五个任务轮流在四个CPU上执行，每个CPU基本都会执行到不同的任务，可以通过复杂的亲和度机制迁移任务到对应的CPU上）。多队列调度中每个CPU调度互相独立，不需要共享队列，天生具有良好的缓存亲和度，不过可能有负载不均衡的问题。Linux的O(1)调度和CFS调度采用多队列调度。

## Linux
### awk
awk '{print $1,$4}' text.txt // text.txt每一行以空格或者tab分割，输出每行的第1和第4项 \
awk -F, '{print $1,$4}' text.txt 或者 awk 'BEGIN{FS=,} {print $1,$4}' text.txt // 自定义分隔符（本例中为,） \
awk -F '[ ,]' ... // 先使用空格分隔，再使用逗号分隔

awk -va=1 '{print $1,$1+a}' text.txt // 自定义变量a \
awk '$1>2' text.txt // 过滤第一列大于2的行

### sort
对每行进行排序，-t表示行分隔符，-k表示按照第几列排序，-n表示按照数字排序（而不是字符串），-g表示按照通用数字排序，-r表示降序，-h按照文件大小排序，-u表示去重，-o表示写回文件（不能使用重定向符>）。\
sort -t ':' -k 3 -nr a.txt // 每行以冒号分隔后按照第3列降序数字排序\
du -h | sort -hr // 按照文件大小排序当前文件夹文件\
ps aux | sort -gr -k 4 | head -n 5 // 内存占用最多的5个进程

### I/O多路复用
多路复用的优势不是处理单个连接更快（需要select然后再read），而是可以处理更多的文件描述符（连接）。\
select/poll返回后还需要遍历fdset找到就绪的描述符。

epoll_create创建一个文件描述符，指定监听的文件描述符的最大值；epoll_ctl对epoll_create创建的描述符进行操作（添加、删除、修改监听的描述符，底层实现为红黑树），epoll_event指定对监听的描述符监听的具体事件，有事件发生时采用回调机制激活被监听的描述符；epoll_wait返回指定个数的事件，就绪的文件描述符被添加到一个ready list链表中，内核将链表复制到返回的events数组中。\
LT模式下应用程序可以不处理事件，下次调用epoll_wait还会触发。ET模式下应用程序需要立即处理该事件，下次调用epoll_wait不会触发，这样系统不会一直被不需要处理的描述符提醒。ET模式减少了epoll事件被触发的次数（需要使用非阻塞socket防止未被处理的文件描述符饿死），同时调用者如果发现recv的数据大小等于请求的大小，则说明可能还有数据没有被处理完。如果有大量的idle-connection，epoll的效率大大高于select/poll。

### 可执行文件的加载过程
shell进程fork一个子进程（写时复制），执行execve系统调用，传入可执行文件名与形参等参数，对进程栈进行初始化。之后寻找给定的可执行文件，ELF可执行文件包含一个指示加载信息的头，之后跟随了程序段（指令段（.text），数据段（.data），常量段（.rodata），未初始化变量（.bss）等）。装载程序建立一个虚拟地址的映射，通过文件头建立虚拟地址和可执行映射关系，将PC设置为文件入口。之后开始执行的时候通过缺页中断进行实际的物理页置换。

## Multithreading
### 线程交替打印
本质上也是生产者消费者模型。
```C++
std::mutex mu;
std::condition_variable cond;
bool flag = true;
void foo() {
    while (1) {
        std::unique_lock<std::mutex> locker(mu);
        cond.wait(locker, [&](){return flag;});
        cout << "foo" << endl;
        flag = !flag;
        locker.unlock(); // might not be necessary because wait in the next loop will implicitly unlock
        cond.notify_one();
    }
}
void bar() {
    while (1) {
        std::unique_lock<std::mutex> locker(mu);
        cond.wait(locker, [&](){return !flag;});
        cout << "bar" << endl;
        flag = !flag;
        locker.unlock();
        cond.notify_one();
    }
}
```

### 线程池
生产者消费者模式：主线程处理工作任务并存于工作队列中，工作线程从队列中取出任务进行处理。需要线程间的数据通信。\
领导者跟随者模式：任何时刻线程池只有一个领导者线程，事件到达时领导者线程负责消息分离，选出一个跟随者作为新领导者，自身变为工作者去处理时间，处理完毕后变为追随者。避免了线程间交换数据，提高了cache亲和性。


# Computer Networks
### RTT, MSL, TTL
RTT为发送报文到收到确认的时间。MSL为报文最大生存时间，手动设置为30s/1min/2min（TIME_WAIT等待2MSL确保自己最后的ACK到达，如果没到达则2MSL内会收到对方发送的新的FIN）。TTL为IP数据报可以经过的最大路由数。

### Cookie vs. Session
Cookie通过在客户端记录信息确定用户身份，Session通过在服务器端记录信息确定用户身份。\
HTTP协议是无状态的协议。一旦数据交换完毕，客户端与服务器端的连接就会关闭，再次交换数据需要建立新的连接。这就意味着服务器无法从连接上跟踪会话。在Session出现之前，基本上所有的网站都采用Cookie来跟踪会话。\
如果服务器需要记录该用户状态，就使用response向客户端浏览器颁发一个Cookie。客户端浏览器会把Cookie保存起来。当浏览器再请求该网站时，浏览器把请求的网址连同该Cookie一同提交给服务器。服务器检查该Cookie，以此来辨认用户状态。\
Cookie对象使用key-value属性对的形式保存用户状态，一个Cookie对象保存一个属性对（信息编码为二进制值），一个request或者response同时使用多个Cookie。\
Cookie具有不可跨域名性。\
Cookie的maxAge决定着Cookie的有效期，可以通过getter/setter读写该属性。为负数时表示仅当前窗口和子窗口有效。为0时表示即时失效，可用来删除Cookie。如果要删除某个Cookie，只需要新建一个同名的Cookie，并将maxAge设置为0，并添加到response中覆盖原来的Cookie。如果要修改某个Cookie，只需要新建一个同名的Cookie，添加到response中覆盖原来的Cookie。\
验证登录信息时不再查询数据库，可以把账号按照一定的规则加密后，连同账号一块保存到Cookie中。下次访问时只需要判断账号的加密规则是否正确即可。

Session是服务器端使用的一种记录客户端状态的机制，使用上比Cookie简单一些，相应的也增加了服务器的存储压力。Session相当于程序在服务器上建立的一份客户档案，客户来访的时候只需要查询客户档案表就可以了。\
每个来访者对应一个Session对象，所有该客户的状态信息都保存在这个Session对象里。Session机制决定了当前客户只会获取到自己的Session，而不会获取到别人的Session。各客户的Session也彼此独立，互不可见。\
服务器一般把Session放在内存里。只要用户继续访问，服务器就会更新Session的最后访问时间，并维护该Session。服务器会把长时间内没有活跃的Session从内存删除。这个时间就是Session的超时时间。如果超过了超时时间没访问过服务器，Session就自动失效了。\
Session需要使用Cookie作为识别标志，服务器向客户端浏览器发送一个名为JSESSIONID的Cookie，它的值为该Session的id（也就是HttpSession.getId()的返回值）。Session依据该Cookie来识别是否为同一用户。\
URL地址重写是对客户端不支持Cookie的解决方案。URL地址重写的原理是将该用户Session的id信息重写到URL地址中。服务器能够解析重写后的URL获取Session的id。

cookie不是很安全，别人可以分析存放在本地的COOKIE并进行COOKIE欺骗。\
当访问增多，减轻服务器性能方面，应当使用cookie。\
单个cookie保存的数据不能超过4K，很多浏览器都限制一个站点最多保存20个cookie。Session对象没有对存储的数据量的限制，其中可以保存更为复杂的数据类型。\
两者最大的区别在于生存周期，Session是IE启动到IE关闭 （浏览器页面一关，Session就消失了），Cookie是预先设置的生存周期，或永久的保存于本地的文件。

### 粘包
UDP是数据包协议，包与包之间没有关联，不会粘包。缺点是数据包超过限制则无法发送，缓冲容量小于包大小则可能出现数据丢失。\
TCP是流式协议，接收方按顺序接收字节流后重组为二进制数据。数据包较小时可能一次读到多个包，较大时一次读不完整。TCP优化机制可能将短时间内的多个较小的包合并发送。\
可以通过同时发送数据长度，或者指定特殊的结束字段，或者将额外信息打包到一个定长的以特定标志开头的头部数据中单独发送。

### http & https
http请求格式：第一行为方法+URI+版本，第二行开始为请求头（包含客户端环境，请求正文信息等），之后是请求正文。

http 1.1支持长连接，并增加了新的请求头（Keep-alive, close）等。这样还可以支持请求的流水线处理（例如一个包含有许多图像的网页文件的多个请求和应答可以在一个连接中传输；客户端不用等待上一次请求结果返回，就可以发出下一次请求）。http 1.1增加host头域，支持共享同一个IP的多个虚拟主机。http 1.1加入100状态码，服务器接收请求后返回100则客户端可以继续发请求。\
http 2.0使用二进制格式而不基于文本，引用多路复用机制（一个连接可以有多个request，与流水线不同的地方在于request可以并行执行），使用encoding压缩header大小，支持服务端推送（服务器在客户端请求之前发送数据，可以缓存，例如服务器返回.css请求时会将.js请求发送到客户端缓存）。

https:\
client hello: 支持的协议版本，加密方法等等，随机数\
server hello: 确认协议版本加密方法，发送证书，随机数\
client: 生成一个随机数（premaster secret）使用证书公钥加密，\
server: 与client分别通过三个随机数最终得到session key，互相发送hash值（消息验证码MAC）进行校验

判断http请求结束：请求头一直到\r\n\r\n表示结束，解析后如果Content-Length存在则知道长度，否则为Transfer-Encoding，读到\r\n0\r\n\r\n为止。

# Databases
### MySQL索引类型
普通索引：create index ... on table(column(...)) \
唯一索引：索引值可以为空。create unique index ... \
主键索引：一个表只有一个主键，不能为空，一般在建表时直接声明primary key(...)。\
组合索引：最左前缀都可以使用。\
全文索引：查找关键字，建表时声明fulltext(...)。数据量较大时，将数据放入一个没有全局索引的表中，然后再用CREATE index创建fulltext索引比较快。在match(...)时使用，比like%效率高。星号通配符只能在词后面。

哈希索引：无法用于排序分组以及部分或者范围查找。

### MyISAM vs. InnoDB
每个MyISAM在磁盘上存储成三个文件。分别为：表定义文件、数据文件、索引文件。InnoDB所有的表都保存在同一个数据文件中（也可能是多个文件，或者是独立的表空间文件），InnoDB表的大小只受限于操作系统文件的大小，一般为2GB。\
InnoDB支持事务，MyISAM不支持。\
InnoDB支持外键，MyISAM不支持。\
InnoDB是聚簇索引，MyISAM是非聚簇索引。\
InnoDB最小锁粒度是行锁，MyISAM是表锁，一个更新语句会锁住整张表，大量读查询时大粒度的锁可能速度更快。\
MyISAM保存了表的总行数，InnoDB没有（select count(*) from table会遍历整张表，有where的时候两种引擎处理方式相同）。、
MyISAM支持全文索引，InnoDB在MySQL 5.6之后也支持全文索引。

### MVCC
处理读写冲突，读操作只读取该事务开始前数据库的快照，避免脏读和不可重复读。\
避免不可重复读的原因是，同一个事务中多个读操作均读的是快照。读提交时每个快照都会生成获取最新的视图。\
snapshot isolation: every tuple gets an LSN, read highest LSN that is lower than the start timestamp, write check at the end whether the latest version of all written tuples is the one written (乐观锁); undo records of every modification they make, transaction is assigned a SCN (System Change Number), read blocks with an SCN
lower than the start SCN, if a block is too new, a copy is made, and undo records applied to recreate a version with a suitable.\
undo日志（before image，旧数据的copy）通过ROLL_PTR指针把每一行的修改连起来。ReadView结构维护一个未提交的事务列表，以及事务id的最小最大值，表示当前事务创建时仍然没提交的事务。该当前事务读时将日志中记录的事务id与未提交事务id范围比较，如果小于最小值则说明该事务创建时此记录已经提交，可以读。如果大于最大值说明此记录在当前事务生成后才出现，不可以读（因为不知道该事务是否已经提交）。如果在之间，则如果记录的事务id在未提交事务列表里，则不可以读；如果不在未提交事务列表的话，则可以读（因为这表明在当前事务生成前有个事务出现地比最小id晚但是已经提交）。如果是读提交的隔离级别，则每次读都会生成一个新的ReadView，如果是可重复读，则只使用事务开始时的ReadView。

使用next-key locks对间隙加锁，防止当前读（不是快照读）时出现幻读（一般出现在non-locking read （快照读，select）之后跟一个locking read （当前读，select for update，被认为是写，基于乐观锁机制进行更新））。

## Redis
### 缓存异常
缓存雪崩：大面积缓存同时失效，数据库短时间负载过大。可以设置随机过期时间，或者在数据库上加锁排队。\
缓存击穿：缓存中没有但是数据库中有，与缓存雪崩不同的是缓存击穿特指并发查同一条数据。可以设置热点数据永不过期，或者加互斥锁。\
缓存穿透：缓存与数据库上都没有，一般是攻击者。可以在接口层增加校验，或者使用布隆过滤器。

# Algorithms
### 排序
选择排序（不稳定）：每次遍历给第i个位置选择那个第i小的元素，并交换这两个元素。不稳定的原因是原来在第i个位置的元素可以被交换到任意位置，破坏原先大小相同元素的顺序。
```Python
for i in range(n):
    target = i
    for j in (i, n):
        if arr[j] < arr[target]:
            target = j
    swap(arr[i], arr[target])
```
插入排序（稳定）：数学归纳法，假定元素前面的数组已经排好序，将当前元素插入之前排好序的数组。
```Python
for i in range(1, n):
    j = i - 1
    key = arr[i]
    while j >= 0 and arr[j] > arr[i]:
        arr[j + 1] = arr[j]
        j -= 1
    arr[j + 1] = key
```
冒泡排序（稳定）：重复遍历数组，交换相邻的逆序数，直到没有逆序数对。可以设置flag来记录一次遍历是否发生交换，如果没有交换则说明数组已经排好序。
```Python
for i in range(n - 1): # 最多遍历n - 1次，因为每次遍历完最后一个数都是最大数
    for j in range(n - i): # 最大的i个数已经在之前的遍历中放在了对应的位置
        if arr[j] > arr[j + 1]:
            swap(arr[j], arr[j + 1])
```

### LRU
哈希表+双链表，双链表有单独head和rear节点指示头尾，便于添加删除。put时如果key在哈希表中需要增加size并new新的value，否则需要更改value。

### 递归BFS
```Python
def bfs(list):
    children = []
    for l in list:
        children.append(l.children)
    if len(children) > 0:
        return [list, bfs(children)]
    return list
```

# Other

## Design Patterns
### 单例模式
懒汉式：
```C++
class A {
public:
    A(const A&) =delete;
    A& operator=(const A&) =delete;
    static std::shared_ptr<A> getInstance() {
        if (instance == nullptr) { // 减少加锁的必要性
            std::lock_guard<std::mutex> lk(mu);
            if (instance == nullptr) { // 看是否其他线程已经完成了初始化
                instance = std::shared_ptr<A>(new A);
            }
        }
        return instance;
    }
private:
    A() {}
    static std::shared_ptr<A> instance;
    static std::mutex mu;
};

// 使用局部静态变量
class A {
public:
    A(const A&) =delete;
    A& operator=(const A&) =delete;
    static A& getInstance() {
        static A instance; // 静态变量初始化是线程安全的
        return instance;
    }
private:
    A() {}
};
```
饿汉式：
```C++
class A {
private:
    A() {}
    static A a = new A(); // 不能直接public，否则用户可以直接修改此成员
public:
    static A& getInstance() {
        return a;
    }
};
```

## Docker


## Distributed Systems
### Zookeeper
bigtable (chubby): distributed lock service (five replicas, one of which elected as master), namespace for directories and small files, hierarchy for tablet (tablet servers are chubby clients) locations \
files also store root metadata \
ensure single master via unique master lock

### RPC
远程调用像本地调用一样，传输数据没必要文本协议（http），可以直接二进制传输。\
RPC实现类（Stub）调用一个runtime library（socket, HttpClient, ...），将数据序列化为二进制，传给另一个service（负载均衡确定service的地址）。另一个service将二进制数据反序列化为请求对象，Stub解析请求对象得到要调用的接口，将请求对象传给实际的实现类执行。

### 2-Phase Commit 
投票阶段：协调者向参与者发送事务请求，参与者反馈事务结果但是不提交，记录日志；事务提交阶段：如果全部执行正常，协调者发送commit通知，参与者收到后提交，并返回ack，如果有参与者不正常或者超时未回复，协调者发送rollback。

2PC会有单点故障和同步阻塞问题（协调者下线时（尤其是commit消息发送之前）参与者阻塞）。网络异常时有某些commit未到达则可能部分commit（如果是协调者故障，可以通过协调者的log进行恢复；如果是参与者超时，则可以重新发送）。\
可以通过互询机制让参与者向其他参与者询问执行情况，避免因为与协调者的通信问题造成参与者阻塞以及暂时的数据不一致（部分commit）。

3PC：增加CanCommit阶段，协调者询问，参与者向协调者回复估计是否可以执行。如果CanCommit有no或者超时则直接abort事务。\
3PC中如果协调者在commit/rollback消息发送前下线或者消息超时，参与者不会阻塞，而是超时后继续commit（由于引入了CanCommit，概率上讲事务成功的概率大，不过仍然可能数据不一致）。

## Golang


## Message Queues
