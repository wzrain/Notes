# C++
### static关键字
static局部变量位于全局数据区，可以定义于函数内部，作用域为定义的局部作用域（如定义它的函数内部）。static全局变量的作用域为文件内部，文件外部不可见，而普通全局变量可以通过extern被外部文件访问。\
static函数与static全局变量类似，用于声明仅在该文件内部可以被访问的函数。\
(C++) static静态成员存储于全局数据区，被所有对象共享，相比全局变量不会与其他命名冲突，同时可以通过private等进行访问控制。static静态成员函数没有this指针，不能访问非静态成员与非静态成员函数。

# Operating Systems
### semaphore vs. mutex
信号量管理资源的数量，一定程度上调度线程，mutex管理资源的使用权。

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

## Linux

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


# Computer Networks
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


# Databases

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