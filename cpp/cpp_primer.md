# Chapter 2 变量与基本类型
## 2.5 处理类型
### 2.5.1 类型别名
typedef定义类型别名：
```C++
typedef double wages; // wages是double同义词
typedef wages base, *p; // base是double同义词，p是double*同义词
```
也可使用using定义：
```C++
using SI = Sales_item
```
类型别名不是简单的替换，比如涉及的const指针的时候：
```C++
typedef char *pstring;
const pstring cstr = 0; // cstr 是char* const常量指针
const char* cstr2; // cstr2是指向const char的指针
```

### 2.5.2 auto类型说明符
auto定义的变量需要有初始值。auto语句可以声明多个变量，只要所变量初始数据类型一样：
```C++
auto i = 0, *p = &i; // 正确
auto sz = 0, pi = 3.14; // 错误
```
引用被作为初始值时，参与初始化的是引用对象的值，这时auto是引用对象的类型。\
auto会忽略掉顶层const，底层const保留。\
auto引用中初始值中的顶层const属性仍然保留。
```C++
int i = 0, &r = i;
auto a = r; // a是int

const int ci = i, &cr = ci;
auto b = ci; // b是int
auto c = cr; // c是int
auto d = &i; // d是int*
auto e = &ci; // e是int* const（初始值是一个指向常量的指针，即int* const）
const auto f = ci; // f是const int，顶层const要明确指出

auto &g = ci; // g是int常量引用
auto &h = 42; // 错误，不能为非常量引用绑定字面值
const auto &j = 42; //正确，常量引用可以绑定字面值

auto k = ci, &l = i; // k是int，l是int &
auto &m = ci, *p = &ci; // m是const int &, p是int* const。这里p和上面一样，不是直接把auto换成const int，而是一个指向const int的指针，即int* const。
auto &n = i, *p2 = &ci; // 错误，从i得到auto是int, 从&ci得到auto是const int
```

### 2.5.3 decltype类型指示符
decltype选择并返回操作数数据类型，不计算表达式的实际值，例如decltype(f())不调用f，只使用返回值类型。\
与auto不同，decltype直接返回包括顶层const和引用变量类型：
```C++
const int ci = 0, &cj = ci;
decltype(ci) x = 0; // x是const int
decltype(cj) y = x; // y是const int&，y绑定到x
decltype(cj) z; // 错误，z是引用，必须初始化

int i = 42, *p = &i, &r = i;
decltype(r + 0) b; // 正确，b是int
decltype(*p) c; // 错误，c是int&，需要初始化（解引用操作得到引用类型）。
decltype((i)) d; // 错误，d是int&，(i)被当做表达式，返回int&
```
由于一个变量是一个可以放在赋值语句左侧的表达式，因此变量加上括号后返回引用。


# Chapter 9 顺序容器

## 9.2 容器概览

### swap
array的swap是真正交换元素的。

## 9.3 顺序容器操作


### emplace
push和insert拷贝被传递的对象。c.emplace_back("aaa", 23, 34) 调用c存放的元素类型的构造函数A(string, int, int)。c.emplace_back()调用默认构造。

### 访问
c.front() = 42; 改变c中元素。\
auto v = c.front(); v不是引用. \
auto &vv = c.front(); (int a = 3; int &b = a, int c = b, int &d = b; // v相当于这里的c, vv相当于d, a b d是一个东西, c是一个copy。)

### 迭代器失效 (又见9.3.6)
c.erase(iter) 返回iter后面的迭代器，iter == c.end()是UB。vector和string中指向删除点后的迭代器指针引用都会失效 （vector, string存储空间重新分配后所有迭代器失效。）。deque删除除首尾元素外的任何元素时，所有迭代器指针引用失效；在首尾插入时迭代器失效，指针和引用不会失效。

list, forward_list指向容器的迭代器指针引用仍有效


### 9.3.4 forward_list
forward_list 没有递减迭代器，没有push_back，没有insert.\
before_begin()类似于自己实现时候的head sentinel。insert_after (emplace_after), erase_after依然返回删除节点后面那个节点的迭代器。

## 9.4 vector如何增长
扩张操作通常比list和deque还要快。

c.shrink_to_fit() 将capacity减少为size()，只适用于vector, string, deque。capacity()和reserve只适用于vector, string。
调用reserve后capacity将大于等于reserve的参数。resize只改变数目，不改变capacity。

添加元素且size==capacity，或调用resize或reserve时给定大小超过capacity时才可能重新分配内存空间。

## 9.5 string特殊操作
s.find_first_of(t) 查找s中第一个在t中出现的元素。

## 9.6 容器适配器
stack接受除array和forward_list，stack和queue默认用deque实现，priority_queue默认为vector。\
pq不可基于list（调整堆要求随机访问能力）。

# Chapter 10 泛型算法
find可以用在内置数组中，begin(arr)表示指针。泛型算法本身不会执行容器操作，只会运行于迭代器之上。

## 10.2
accumulate求和（可以用来将string数组连成string（定义了+运算符））。\
equal接受三个迭代器。接受单一迭代器来表示第二序列，假定第二个序列至少和第一个一样长。只要两个序列存储的元素可比较即可，不需要容器本身类型一致。、
fill_n(dest, n, val)将dest迭代器起n个元素设为val。\
back_inserter返回一个插入迭代器，给该迭代器赋值会调用push_back添加元素。// vector\<int> vec; fill_n(back_inserter(vec), 10, 0); \
unique使得每个元素只出现一次，返回指向所有元素都出现一次之后的位置的迭代器。// auto end_uniq = unique(vec.begin(), vec.end()); vec.erase(end_unq, vec.end()); 

## 10.3 定制操作
sort可接受一个二元谓词参数代替<运算符。

### lambda表达式

lambda表达式是一种可调用对象（还有函数和函数指针（6.7）以及重载了调用运算符的类（14.8））。可以理解为未命名的内联函数。\
\[capture list](para list) -> return type { body } \
capture list是所在函数中的局部变量。lambda表达式不能有默认参数。\
在只能接受一元谓词的find_if中使用只有一个参数的lambda表达式，可以通过捕获局部变量做到更多事情。\
for_each算法接受一个可调用对象，对每个元素调用这个对象。 \
编译器对每个lambda表达式生成一个新的重载了函数调用运算符的类，捕获的局部变量是类的成员（见14.8.1）。\
局部变量被lambda值捕获的时候，lambda在创建时就进行了捕获，因此之后的调用是根据创建时捕获的局部变量值进行计算，而不是调用时的局部变量值。\
函数返回的lambda不能包含引用捕获。\
lambda改变值捕获变量的值时要加mutable，改变引用变量捕获的时候要看该引用是否为const。\
lambda函数体中单一return语句无需指定返回值，branch内部返回时编译器会推断其返回void，需要显式指定返回值 -> int。

### bind
bind（#include \<functional>）可被看做一个通用的函数适配器，用来作为有捕获变量的lambda表达式的counterpart: \
bool check_size(const string& s, int sz) {
    return s.length() >= sz;
} \
=> \[sz](const string& s) {return s.length() >= sz;}; \
=> auto check = bind(check_size, _1, sz); check(s); // equals to check_size(s, sz); 这里check和上面的lambda一样只有一个参数s。\
auto g = bind(f, a, b, _2, c, _1); g(x, y)相当于f(a,b,y,c,x)。using std::placeholder:_1; \
bind(f, _2, _1)可以调换参数顺序。\
bind传递引用应该使用ref函数，bind(print, ref(os), _1, ' ');。ref返回一个包含引用的对象，该对象可拷贝。cref保存const引用。

## 10.6 特定容器算法
list和forward_list定义特有的sort, merge, remove, reverse, unique. 通用版本sort需要随机访问迭代器，不能用于list。因为只改变链接不真正交换元素，链表版本的这些算法比通用版本性能好。splice算法可以将另一链表的某一范围移动到本链表中。链表特有的操作会改变被操作的容器。

# Chapter 11 关联容器

## 11.2
有序容器关键字类型严格弱序。定义bool compare(...){...}，multiset\<A,decltype(compare)*>（使用一个函数指针）。

## 11.3 关联容器操作
map，set的insert返回是否存在要插入的key，是的话返回插入元素的迭代器。erase(iter)返回被删除元素的下一个元素的迭代器。\
multimap的find找到key对应的第一个元素，可以使用count计数key对应的元素个数（也可使用lower_bound, upper_bound定位，或者直接使用equal_range）。

## 11.4 无序容器
无序容器存储上组织为一组桶，哈希函数将元素映射到桶，重复关键字在一个桶中，无序容器性能依赖于哈希函数质量和桶的数量大小。不同关键字映射到一个桶时采用顺序搜索查找。

默认情况下无序容器使用==运算符，标准库为内置类型与指针以及string、智能指针等提供了hash\<key_type>。\
自定义类型可提供自己的hash模板版本（见16.5）。还可以提供函数替代==运算符和哈希函数：\
size_t hasher(const A& a) {
    return ...
} \
bool eq(const A& a1, const A& a2) {return ...} \
unordered_set\<A, decltype(hasher)\*, decltype(eq)*> \
（也可以用重载了函数调用运算符的类来代替），如果类定义了==运算符就不需要再重载函数。

# Chapter 12 动态内存
## 12.1 智能指针
### shared_ptr
模板参数为指针指向的类型。\
make_shared\<T>(args)返回一个shared_ptr，args为T的初始化参数。p = q时递减p的引用计数，递增q的。\
shared_ptr的析构函数会递减对应的引用计数，如果引用计数为0则同时释放对应的动态内存。\
shared_ptr使得多个对象之间可以共享数据。

### new delete
内存耗尽时new会返回bad_alloc，可使用new (nothrow) T来使其返回空指针（见19.1.2）。\
delete指向栈内存的指针或者指向已被释放的内存都是UB。\
悬空指针指向一块已经被释放的内存。
使用指针的智能指针构造函数是explicit的，不能隐式转换，只能直接初始化shared_ptr\<T> p(new T());。

其他方法：
```C++
shared_ptr<T> p(u); // 从unique_ptr接管对象所有权，将u置空 
shared_ptr<T> p(q, d); // 接管内置指针q所有权，使用自定义的可调用对象d代替delete 
shared_ptr<T> p(p2, d); // p是p2的拷贝，delete由d代替 
p.reset(); p.reset(q); p.reset(q, d); // 若引用计数为1，则reset释放对象，若传递内置指针q时令p指向q，否则将p置空。可选可调用对象参数d代替delete释放q。
```

void process(shared\<T> ptr)在接受参数时会拷贝参数，因此引用计数在函数内至少为2，ptr指向的内存在函数结束时不会被释放。如果传入一个临时的shared_ptr，如process(shared_ptr\<T>(x))，则函数结束时内存会被释放，这时x成为悬空指针，再调用*x会是UB。

get方法用于给无法使用智能指针的代码传递内置指针，使用get返回的指针的代码不能delete这个指针。因此不能将get返回的指针再绑定到某个shared_ptr，这样此shared_ptr在被销毁的时候指针被释放，原来的shared_ptr指向的内存就不可用了，两个独立的shared_ptr指向同一内存是UB。

可以将shared_ptr的思想推广，shared_ptr本质上可以对意外结束的程序进行后处理，例如delete。可以自定义delete去做更多的后处理，例如本来要关闭连接，因为程序崩溃，没有正常关闭，可使用：
```C++
shared_ptr<Conn> p(&conn, end_conn); 
void end_conn(Conn* p) {
    disconnect(p);
}
```
来保证链接关闭。此删除器接受一个conn*指针，处理shared_ptr保存的指针。

### unique_ptr
unique_ptr不支持拷贝赋值。
```C++
unique_ptr<T, D> u; // D为自定义删除器 
unique_ptr<T, D> u(d); // d为类型为D的对象 
u.release(); // 放弃对指针控制权，将u置空，返回指针
u.reset(); // 释放u指向的对象 
u.reset(q); // 令u指向指针q
unique_ptr<string> p2(p1.release()); 
p2.reset(p3.release());
```

可以拷贝一个将要被销毁的unique_ptr，例如函数返回：
```C++
unique_ptr<int> clone(int p) {
    unique_ptr\<int> ret(new int(p));
    ...
    return ret;
}
```
这里编译器执行一种特殊的拷贝（见13.6.2）。

unique_ptr管理删除器的方式和shared_ptr不同（在模板参数中指定），原因见16.1.6。
```C++
unique_ptr<Conn, decltype(end_conn)*> p(&conn, end_conn);  
```

### weak_ptr
weak_ptr不控制所指向对象生存期的智能指针，指向由shared_ptr管理的对象。weak_ptr绑定到shared_ptr不会改变shared_ptr的引用计数，即使有weak_ptr指向对象，指向对象的最后shared_ptr被销毁时对象也会被释放。\
因此，由于指向的对象可能不存在，需要用wp.lock()返回一个指向wp指向的对象的shared_ptr，如果对象不存在则返回空shared_ptr。
```C++
if (shared_ptr<int> sp = wp.lock()) {
    // safe
}
```

## 12.2 动态数组
可使用unique_ptr管理动态数组：
```C++
unique_ptr<int[]> up(new int[10]);
up.release() // delete []
```
shared_ptr不能直接管理数组，需要自己提供删除器。访问也不能直接用下标，需要get一个内置指针再做算术运算。

## 12.3 示例：文本查询程序
TODO


# Chapter 13 拷贝控制
拷贝初始化（拷贝构造或移动构造）在传递非引用类型参数，返回非引用类型对象，列表初始化数组元素或聚合类成员，容器insert或push时都会发生，因此往往不是explicit的构造函数，例如用size初始化vector:
```C++
vector<int> vec(10); // explicit的构造函数
void f(vector<int>); f(10) // wrong, 不能用explicit构造函数拷贝
```

需要析构函数的类通常也需要拷贝构造和赋值运算符：
```C++
class A {
private:
    int* p;
public:
    A(int v) : p(new int(v)) {}
    ~A() {delete p;}
};

A f(A a) {
    A b = a;
    return b;
} // 函数结束时a和b同时被销毁，a.p被delete两次
```

### =default =delete
=default表示显式要求编译器生成默认版本，在类内=default则被隐式地声明为内联函数（和直接在类内实现的函数一样），也可在类外=default，则不是内联。=delete必须出现在函数第一次声明的时候。\
不能删除析构函数，否则无法定义该类的变量或临时对象。可以动态分配该类的对象（指针指向），但是无法释放（delete不成功）。

可以通过将拷贝构造函数和赋值运算符设为private，且不实现它，这样即使友元和成员函数也无法进行拷贝。

## 13.2 拷贝控制与资源管理
### 行为像值的类
```C++
A& A::operator=(const A& ra) {
    auto np = new int(*(ra.p));
    delete p; // 给指针赋值前释放原来的对象防止内存泄漏
    p = np;
    return *this;
}
```

### 行为像指针的类
需要一个引用计数，保存在动态内存中用来在多个对象之间跟踪同一被引用对象的引用次数（类似于shared_ptr实现）。
```C++
A& A::operator=(const A& ra) {
    ++(*(ra.counter));
    if (--*counter == 0) {
        delete p;
        delete counter;
    }
    p = ra.p;
    counter = ra.counter;
    return *this;
}
```

## 13.3 交换操作
如果一个类B有一个类型为A的成员，即使A定义了swap（可通过定义为友元函数来访问私有成员），对B做swap时仍然会使用std::swap交换A。因此应该也对B定义swap函数：
```C++
void swap(B& lb, B& rb) {
    using std::swap;
    swap(lb.a, rb.a); // 类型特定的版本优先于std版本（见16.3, 18.2.3）
}
```

定义swap的类可用它来实现赋值（copy and swap)，参数为值：
```C++
A& A::operator=(A ra) {
    swap(*this, ra);
    return *this
} // 函数结束时ra被销毁，ra里的指针（这里指向的是本对象原来指向的对象）通过A的析构函数被delete
```
调用时传参会调用A的拷贝构造函数（如果传一个右值引用（例如std::move的返回值）的话，会调用移动构造函数移动所传的参数），其中new可能抛出异常，不过也在改变左侧运算对象之前发生。

## 13.4 示例：邮件处理（Message，Folder）
TODO

## 13.5 示例：动态内存管理类（StrVec）
TODO


## 13.6 对象移动
IO类或unique_ptr包含不能被共享的资源（IO缓冲或指针），这些对象不能被拷贝，可以被移动。新标准中容器可以保存不可拷贝的类型，只要能移动就行。\

### 13.6.1 右值引用
右值引用不能直接绑定在左值上，const左值引用可以绑定到右值上。
```C++
int i = 42;
int &r = i * 42; // wrong
int &&rr = i; // wrong
const int &r3 = i * 42; // correct
int &&r4 = i * 42; // correct
```
这里的r4在之后的使用中是左值。

std::move显式将左值转化为对应的右值引用，即承诺除了对被转化的左值赋值或销毁外不再使用它。\
move不提供using声明，直接std::move（见18.2.3）。

### 13.6.2 移动构造函数和移动赋值运算符
移动构造函数完成资源移动，保证源对象的销毁是无害的（不再指向被移动的资源）：
```C++
A::A(A &&a) noexcept // 不应抛出异常
    : p(s.p) 
{
    s.p = nullptr;
}
```
由于只接管资源，不分配资源，通常不会抛出异常，使用noexcept通知标准库，使其不用做额外工作。\
标准库容器为异常发生时自身行为提供保障，例如调用push_back异常时vector自身不改变。如果在push_back中重新分配内存时使用移动构造函数，移动到一半抛出异常，那么有一部分旧元素已经被改变，拷贝构造函数不会有这样的问题。因此除非知道移动构造函数不会抛出异常，那么vector重新分配内存时仍会使用拷贝构造函数。因此如果要将自定义对象放进vector，就必须声明自定义对象的移动构造函数不会抛出异常。

移动赋值运算符需要正确处理自赋值：
```C++
A& A::operator=(A &&a) noexcept
{
    if (this != a) {
        free(); // 释放当前拥有的对象
        p = a.p;
        a.p = nullptr;
    }
    return this;
}
```
检查自赋值是因为参数可能是move调用的返回结果。

只有一个类没有定义任何自己版本的拷贝控制成员，且类中每个非static数据成员都可以移动时，编译器才会默认定义移动构造函数和移动赋值运算符。\
如果没有移动构造函数，传右值也是调用拷贝构造函数，将A&&转换为const A&。

移动迭代器的解引用运算符生成一个右值引用：
```C++
void StrVec:reallocate() {
    auto cap = size() ? 2 * size() : 1;
    auto first = alloc.allocate(cap);
    auto last = uninitialized_copy(make_move_iterator(begin()), make_move_iterator(end()), first);
    free();
    elements = first;
    first_free = last;
}
```
标准库不保证哪些算法适用迭代器，源对象可移动由程序员保证。

### 13.6.3 右值引用和成员函数
成员函数可提供接受非const的右值引用：
```C++
void StrVec::push_back(string&& s) {
    ...
    alloc.construct(first_free++, std::move(s));
}
```

也可指出this指针是左值/右值，通过在参数列表后放置引用限定符，且在const限定符后。例如赋值运算符的this只能是左值：
```C++
class Foo {
public:
    Foo& operator=(const Foo& rhs) &;
    Foo sorted() &&;
    Foo sorted() const &;
}
```
对右值执行sorted时可以直接在原址排序，因为没有其他用户。const右值或左值执行sorted需要排序前拷贝数据并返回拷贝的值。\
在定义重载函数时要么全部加引用限定符，要么全不加。

# Chapter 14 重载运算与类型转换
## 14.1
重载函数定义为成员函数时，左侧运算对象必须是所属类的对象，例如string s; string t = s + "hi"; // 等价于s.operator+("hi")。非成员函数等价于operator+(s, "hi")。

## 14.2 输入输出运算符


## 14.3 算术关系运算符


## 14.4 赋值运算符


## 14.5 下标运算符


## 14.6 递增递减运算符
使用一个不被使用的int类型形参来区分前置和后置运算符。

## 14.7 成员访问运算符
```C++
class Ptr {
public:
    std::string& operator*() const {
        return *p;
    }
    std::string* operator->() const {
        return &this->operator*();
    }
}
```
箭头运算符不能丢掉访问成员这一基本含义，返回值需要是一个指向类对象的指针（等价于(*p).mem）或者重载了operator->的对象（等价于p.operator()->mem）。

## 14.8 函数调用运算符
函数对象常作为泛型算法的实参，例如for_each算法（10.3.2）。

### 14.8.1 lambda是函数对象
编译器将lambda表达式翻译成未命名类的未命名对象，该类中含有一个重载的函数调用运算符:
```C++
stable_sort(words.begin(), words.end(), [](const string& a, const string& b) { return a.size() < b.size(); }); 
// =>
class SS {
public:
    bool operator() (const string& a, const string& b) const {
        return a.size() < b.size();
    }
};
stable_sort(words.begin(), words.end(), SS());
```
默认情况下lambda不改变捕获变量，因此重载函数为const（加mutable时可改变）。

值捕获的变量被存为类对应的数据成员并创建构造函数。该类不含有默认构造函数、赋值运算符以及默认析构函数。

### 14.8.2 标准库定义的函数对象

标准库定义的函数对象对指针同样适用，但是直接比较两个无关指针将产生UB。可以通过标准库函数来比较内存地址：
```C++
vector<string*> name;
sort(name.begin(), name.end(), [](string* a, string* b) { return a < b; }); // UB!
sort(name.begin(), name.end(), less<string*>()) // good
```

### 14.8.3 可调用对象与function
可调用对象：函数，函数指针，lambda表达式，bind创建的对象，重载了函数调用运算符的类。可调用对象也有类型，函数和函数指针的类型由返回值和实参类型决定。

不同类型可能具有相同调用形式：
```C++
// int (int, int)
int add(int i, int j);
auto mod = [](int i, int j) { ... };
struct divide {
    int operator()(int i, int j) {
        ...
    }
};
```
可定义一个函数表存储指向可调用对象的指针：
```C++
map<string, int(*)(int, int)> binops;
binops.insert({"+", add}); // good
binops.insert({"%", mod}); // no good, mod不是函数指针
```
可使用function类型解决此问题（#include\<functional>）。
```C++
function<int(int, int)> f1 = add;
function<int(int, int)> f2 = divide();
function<int(int, int)> f3 = [](int i, int j) {...};
cout << f1(4, 2) << endl; // output 6

map<string, function<int(int, int)>> binops;
map["+"] = add;
map["/"] = divide();
map["*"] = [](int i, int j) { ... };
binops["+"](10, 5); // add(10, 5)
```
重载函数的二义性可通过lambda表达式来消除，或者显式指定函数指针。

## 14.9 重载、类型转换与运算符
类型转换运算符可以面向void外的任意类型，只要该类型可作为函数返回类型。因此不可转换为数组类型，但是可转换为指针类型（包括数组指针，函数指针）。
```C++
class SmallInt {
public:
    SmallInt(int i = 0) : val(i) { ... }
    operator int() const { return val; }
private:
    std::size_t val;
};

SmallInt si; 
si = 4; // 4隐式转换为SmallInt
si + 3; // si隐式转换为int

SmallInt ssi = 3.14 // 3.14转换为int调用SmallInt(int)构造函数
ssi + 3.14 // ssi转换为int再转换为double
```
类型转换运算符隐式执行，无法传递实参，因此定义中无形参。隐式转换可能带来意外结果:
```C++
int i = 42;
cin << i; // istream没定义<<，因此cin被转换为bool，导致其被左移42位。
```
为防止此类情况，使用explicit类型转换运算符，这时仍可隐式调用构造函数，但是上面si + 3这种隐式类型转换不成立，应使用static_cast\<int>(si)显式转换。\
在if, while, do的条件部分、for的条件表达式、逻辑运算符（!, ||, &&）以及条件运算符（? :）中显式类型转换将被隐式执行。

### 14.9.2 避免二义性类型转换
两个类可能提供互相之间的类型转换：
```C++
struct B;
struct A {
    A() = default;
    A(const B&); // B转换成A
};
struct B {
    operator A() const; // B转换成A
};
A f(const A&);
B b;
A a = f(b); // 不知道调用上面哪个
A a1 = f(b.operator A());
A a2 = f(A(b));
```

一个类如果定义多个转换规则，这些转换的类型本身也可以互相转换，就会出现二义性，例如算术运算符：
```C++
struct A {
    A(int = 0);
    A(double);
    operator int() const;
    operator double() const;
};
void f2(long double);
A a;
f2(a); // 不知道是operator int() 还是operator double()
long lg;
A a2(lg); // 不知道哪个构造函数
```
上述二义性是由于所需的二次转换级别一致。如果不是long而是short的话，转换成int级别优于转换成double。

# Chapter 15 面向对象程序设计
obj->func()本质上是(*obj).func()，因此实际上还是看具体的对象的类型，如果一个父类引用形参传入一个子类形参，调用成员函数的时候还是调用子类的函数。

## 15.2 基类派生类
静态成员父类子类共享空间，静态成员函数是遵循访问控制规则的唯一函数。

## 15.3 虚函数
每个虚函数都必须提供定义。动态绑定只通过指针或引用调用才会发生。\
一旦某个类被声明为虚函数，所有派生类中都是虚函数。\
基类虚函数返回某个类A的指针或引用时，派生类对应虚函数可返回A的派生类B的指针或引用（这里要求B到A的转换必须可访问）。

使用override便于调试，发现某些没有覆盖基类函数的函数。覆盖只针对虚函数。\
final不允许派生类覆盖对应的虚函数。

如果虚函数有默认实参的话，默认实参的值由静态类型中定义的默认实参值决定。

## 15.4 抽象基类
含有纯虚函数的类是抽象基类，不能直接创建抽象基类对象。

## 15.5 访问控制与继承
派生类的成员和友元只能通过派生类对象访问受保护成员，派生类对基类受保护成员没有访问特权。\
继承方式对派生类能否访问基类成员没有影响，而是影响自己的成员的访问控制选项（即能不能被其他类型对象访问）。

只有公有继承时才可以把派生类转换为基类。\
所有继承方式中，派生类的成员和友元都能使用派生类向基类的转换。

友元关系不能继承。友元的派生类和基类也不能作为友元。\
本类的友元可以通过本类的派生类的对象访问本类的成员。派生类的友元只能通过派生类对象访问基类成员。

可使用using改变访问级别，using语句中成员的权限由using所在的访问说明符决定。
```C++
class Base {
    protected:
        size_t n;
};
class Derived : private Base { // warning! private inheritance
    protected:
        using Base::n
}
```

struct默认public继承，class默认private继承。

## 15.6 继承中的类作用域
名字查找优先于类型检查，如果派生类的成员与基类的某个成员同名，派生类中会隐藏该基类成员，即使形参列表不同：
```C++
struct Base {
    int foo();
};
struct Derived : Base {
    int foo(int);
};

Derived d;
d.foo(); // wrong，名字查找会找到Derived::foo(int)，会导致参数不对应
d.Base::foo(); // correct
```

## 15.7 构造函数与拷贝控制
### 15.7.1 虚析构函数
指针静态类型可能与指向的对象动态类型不一致。虚析构函数保证执行正确的析构函数版本。\
虚析构函数同时使得这个类不会自动通过编译器合成移动操作（见13.6.2）。

### 15.7.2 合成拷贝控制与继承
基类的默认构造函数，拷贝构造，赋值运算符，或者析构函数被删除或不可访问时，派生类对应的成员也是被删除的。\
基类的析构函数不可访问或删除时，派生类不合成默认和拷贝构造函数和移动构造函数，因为基类的部分无法被删除。
```C++
class B {
public:
    B();
    b(const B&) = delete;
};
class D : public B {

};
D d;
D d2(d); // wrong
D d3(std::move(d)); // wrong，隐式使用拷贝构造函数
```
派生类可以自己显式实现拷贝构造函数。

派生类想要有移动操作时需要先在基类中显式定义基类的移动操作成员，这也同时要求显式定义拷贝操作（否则拷贝操作默认被删除，见13.6.2）。

### 15.7.3 派生类拷贝控制成员
```C++
class Base { ... };
class D : public Base {
public:
    D(const D& d) : Base(d) { ... }
    D(D&& d) : Base(std::move(d)) { ... }
};
```
```C++
D& D::operator=(const D& rhs) {
    Base::operator=(rhs);
    ...
    return *this;
}
```
派生类的析构函数只负责销毁自己的资源。

构造函数或析构函数调用虚函数时应该执行与该构造函数或析构函数相对应的虚函数版本。

### 15.7.4 继承的构造函数
新标准中派生类可以重用其直接基类的构造函数，使用using语句“继承”父类构造函数：
```C++
class D : public B {
public:
    using B::B;
}
// 生成D(a, b, c, d) : B(a, b, c, d)
```
using作用于构造函数时编译器产生对应的形参列表完全相同的属于派生类的构造函数。\
此using声明不会改变构造函数访问级别，也不能指定explicit或constexpr，这两个属性直接继承自基类。\
基类构造函数的默认实参不会继承，而是直接在派生类中获得多个继承的构造函数，每个构造函数省略掉一个含有默认实参的形参。\
默认、拷贝、移动构造函数不会被继承。

## 15.8 容器与继承
向存储父类的容器中添加子类元素只会把子类的父类部分拷贝进去。可以通过存放（智能）指针解决。可以将父类的智能指针指向子类对象。

### 15.8.1 Basket类
TODO

为了用户添加时不用自己调用make_shared，而是直接添加元素，可以通过添加一个虚函数来拷贝当前对象并返回指针。
```C++
class B {
public:
    ...
    virtual B* clone() const & { return new B(*this); }
    virtual B* clone() && { return new B(std::move(*this)); }
};
class D : public B {
public:
    ...
    virtual D* clone() const & {return new D(*this);}
    virtual D* clone() && {return new D(std::move(*this));}
};

class User {
public:
    void add(const B& b) {
        items.insert(std::shared_ptr<B>(b.clone()));
    }
    void add(B&& b) {
        items.insert(std::shared_ptr<B>(std::move(b).clone()));
    }
}
```

## 15.9 文本查询再探
TODO

# Chapter 16 模板与泛型编程
## 16.1 定义模板
### 16.1.1 函数模板
编译器可以自己推断模板参数进行实例化。

模板可定义非类型参数，表示一个值，通过特定类型名指定。实例化时被用户提供或者编译器推断出的值代替，这些值必须是常量表达式。
```C++
template<unsigned N, unsigned M> 
int compare(const char (&p1)[N], const char (&p2)[M]) {
    return strcmp(p1, p2);
}
compare("hi", "mom"); // 使用字符串长度代替N和M，由于有\0字符，模板被实例化为int compare(const char(&p1)[3], const char(&p2)[4]);
```

模板程序应尽量减少对实参类型的要求，例如用less\<T>代替\<。

调用函数时编译器只需掌握函数声明，使用类类型对象时，类定义必须可用，成员函数定义不必出现，因此头文件和源文件可分开。模板中为了实例化版本，编译器需要掌握模板函数或类模板成员函数的定义，因此模板的头文件通常既包括声明也包括定义。

### 16.1.2 类模板
使用类模板需要显式提供模板实参，编译器实例化时会重写模板类，替换模板实参。\
类模板中的含有模板参数的成员通常使用与类相同的模板参数:
```C++
template<typename T> class A {
public:
    std::vector<T> vec;
};
```

可以在类外定义成员函数template \<typename T> ret-type A\<T>::func(args)。\
成员函数被用到时才会被实例化。

处于类模板作用域中时可以直接使用模板名不提供实参：
```C++
template <typename T> class A {
public:
    A() {}
    A& operator++(); // 前置运算符
    A& operator--();
}
```
在类模板外定义时直到看到类名才进入作用域，因此类外定义函数时返回值要加模板参数，函数内部不需要。

类模板的非模板友元可以访问所有模板实例，友元是模板时，类可以授权给所有友元实例或部分实例。\
```C++
template <typename T> class Pal;
template <typename T> class A {
    // Pal的每个实例是A相同参数实例的友元
    friend class Pal<T>; // 需要另外声明Pal类
    // Pal2的每个实例是A的每个实例的友元
    template <typename X> friend class Pal2; // 不需要前置声明
    // Pal3是A所有实例的友元
    friend class Pal3; // 不需要提前声明
};
```

新标准中可以将自己的模板类型参数声明为友元。这不妨碍使用内置类型实例化类。
```C++
template <typename T> class A {
    friend T;
    ...
};
```

新标准中可以使用别名代替类模板：
```C++
template <typename T> using twin = pair<T, T>;
twin<string> authors; // pair<string, string>

template <typename T> using part = pair<T, unsigned>;
part<string> books; // pair<string, unsigned>
```

每个类模板实例都有自己的static成员实例。

### 16.1.3 模板参数
类的成员是一个类型（如string::size_type），我们通过作用域运算符访问这种类型成员和static成员。在模板类中，编译器看到T::mem无法判断mem是一个类型还是static数据成员，实例化时才会知道。但是面对类似于T::size_type * p的语句，编译器需要知道是在定义一个指针变量p还是在做乘法。默认情况下作用域运算符访问的是static成员，如果访问类型时需要显式通过typename关键字指定：
```C++
template <typename T>
typename T:: value_type foo(const T& c) {
    return c.empty() ? typename T::value_type() : c.back();
}
```

可以为函数和类模板提供默认实参：
```C++
template <typename T, typename F = less<T>>
int compare(const T& v1, const T& v2, F f = F()) {
    return f(v1, v2) ? 1 : -1;
}
bool i = compare(0, 42); // use less<int>
A a1, a2;
bool j = compare(a1, a2, compareA);
```
使用类模板默认实参时仍然需要在模板名后跟一个空尖括号。

### 16.1.4 成员模板
作为成员的模板函数不能是虚函数。例如unique_ptr的删除器，可能需要接受删除各种类型的对象：
```C++
// 重载函数调用运算符
class Del {
public:
    template <typename T> void operator() (T* p) const {
        delete p;
    }
}
double *p = new double;
Del d;
d(p);
int *ip = new int;
Del()(ip); // 临时Del对象上调用

unique_ptr<int, Del> p(new int, Del());
unique_ptr<string, Del> sp(new string, Del());
```

类模板可以定义成员模板，类和成员有独立的模板参数。
```C++
template <typename T> class A {
public:
    template <typename X> A(X b, X e); 
}

template <typename T>
template <typename X>
    A<T>::A(X b, X e) { ... }

int ia[] = { ... }
A<int> a1(begin(ia), end(ia)); // A<int>::A(int*, int*)
```

### 16.1.5 控制实例化
由于模板使用时才会被实例化，相同的实例可能出现在多个对象文件中，多个文件实例化同一模板会带来大量额外开销。新标准通过显式实例化避免这种开销：
```C++
extern template class A<string>; // 声明
template int compare(const int&, const int&); // 定义
```
编译器遇到extern模板声明时不会在本文件中生成实例化代码，extern承诺程序其他位置有该实例化的非extern的使用。一个实例化可以有多个extern声明，不过只有一个定义。
```C++
// Application.cc
extern template class A<string>;
extern template int compare(const int&, const int&);
A<string> sa1; // 实例化出现在其他地方
A<int> a1 = {0, 1}, a2(a1); // A<int>及其构造函数在本文件实例化
int i = compare(a1[0], a2[0]); // 实例化出现在其他位置

// templateBuild.cc
// 实例化以下定义
template int compare(const int&, cont int&);
template class A<string>;

// 最终Application.o文件将包含A<int>实例及对应的构造函数实例。templateBuild.o将包含<int> compare和A<string>实例。编译时需要将这两个.o文件链接在一起。
```
上述的实例化"定义"会实例化该模板所有成员，包括内联的成员函数，因为编译器遇到这种定义时不知道程序使用哪些成员函数。

### 16.1.6 效率与灵活性
shared_ptr在运行时绑定删除器，因此删除器不是shared_ptr的一个直接成员，因为运行时才知道删除器的类型。我们可以在shared_ptr生存期修改删除器的类型（通过reset）。因此假设shared_ptr使用del成员间接访问删除器，在shared_ptr析构时应该有类似del ? del(p) : delete p;之类的语句，且del(p)需要运行时跳转至del中存储的地址来执行del指向的函数。

删除器类型是unique_ptr的一部分，因此删除器可以直接存储在unique_ptr中，可以直接调用del(p)。del的类型或者是默认类型，或者是用户在编译前指定在模板参数中，甚至调用本身也可以在编译期被内联。编译时绑定删除器避免了运行时间接调用删除器的开销。运行时绑定使得用户重载删除器更方便。

## 16.2 模板实参推断
### 16.2.1 类型转换与模板类型参数
对于函数模板参数类型的参数，通常编译器都是生成一个新的实例，只有很少的时候采用类型转换。例如const转换，数组或函数指针转换（数组实参转换为指向首元素的指针，函数实参转换为一个该函数类型的指针）。算术转换、派生类基类转换以及用户定义的转换不能应用于函数模板。
```C++
template <typename T> T fobj(T, T);
template <typename T> T fref(const T&, const T&);
string s1 = "...";
const string s2 = "...";
fobj(s1, s2); // fobj(string, string), const被忽略(因为实参被拷贝，所以const与否无所谓)
fref(s1, s2); // fref(const string&, const string&), s1转换为const
int a[10], b[42];
fobj(a, b); // fobj(int*, int*)
fref(a, b); // wrong，如果形参是引用，数组不会转换为指针

long lng;
fobj(lng, 1024); // wrong
```

### 16.2.2 函数模板显式实参
某些情况下编译器可能无法推断模板实参类型，其他情况下用户可以控制实例化。这两种情况在返回值类型与模板参数列表中其他参数都不相同时常出现：
```C++
template <typename T1, typename T2, typename T3>
T1 sum(T2, T3); // 编译器无法推断T1，因为并不在函数参数列表中

auto val = sum<long long>(i, lng); // 显式指定T1为long long
```
显式实参时从左往右匹配的，因此不能推断出来的类型参数最好写前面，不然必须指定前面的所有模板参数。

正常的类型转换可以应用于显式指定的实参：
```C++
template <typename T> compare(const T&, const T&);
long lng;
compare(lng, 1024); // wrong
compare<long>(lng, 1024); // correct, compare(long, long), 1024被转换为long
compare<int>(lng, 1024); // correct, lng被转换为int
```

### 16.2.3 尾置返回类型与类型转换
用户不知道返回结果的类型时无法显式指定模板实参，可以使用尾置返回类型（trailing return type），在看到函数参数列表后声明返回值类型（出现在参数列表之后）：
```C++
// 输入两个迭代器，返回某个迭代器指向的元素，这时返回类型还不可知
template <typename It>
auto fcn(It b, It e) -> decltype(*b) {
    ...
    return *b;
}
```
这里解引用返回一个左值，decltype推断出来的类型是迭代器指向元素类型的引用，如int&。\
我们可能想要得到一个值而不是引用，但是对于传递参数类型我们一无所知。这时可以使用类型转换模板（#include \<type_traits>），这里可以用remove_reference，例如通过remove_reference\<decltype(*b)>::type得到元素类型（模板使用作用域运算符访问要加typename）:
```C++
template <typename It>
auto fcn2(It b, It e) ->
    typename remove_reference<decltype(*b)>::type
{
    ...
    return *b;
}
```

### 16.2.4 函数指针和实参推断
可以使用函数模板初始化一个函数指针或者为函数指针赋值，且通过指针类型来推断模板实参：
```C++
template <typename T> int compare(const T&, const T&);
int (*pf1)(const int&, const int&) = compare; // pf1指向int compare(const int&, const int&)
```

### 16.2.5 模板实参推断和引用
普通左值引用只能传左值，const引用可以传递任何类型实参，右值引用可以绑定右值。
```C++
template <typename T> void f1(T &p);
template <typename T> void f2(const T &p);
template <typename T> void f3(T&& p);
f1(i); // i是int，T推断为int
f1(ci); // ci是const int，T推断为const int
f1(5); // wrong，参数必须是左值
f2(i); // T推断为int
f2(ci); // T推断为int
f2(5); // const &可以绑定右值引用，T是int
f3(42); // T推断为int
```
将左值传递给右值引用参数时，编译器会推断类型参数T为实参的左值引用类型，因此调用f3(i)会将T推断为int&。通常不能直接定义引用的引用，不过可以通过类型别名或者模板类型参数间接定义。\
这种情况下如果间接创建一个引用的引用，引用形成折叠，除了X&& &&折叠成X&&（右值引用的右值引用折叠为右值引用），其他情况（X& &, X& &&, X&& &）都折叠成左值引用X&。\
这样我们可以对左值调用f3，编译器推断T为int&，相当于void f3\<int&>(int& &&)，这样实际形参是int&类型。\
因此如果一个函数参数是模板类型参数的右值引用（T&&），则它可以绑定到左值，且推断出的模板实参时左值引用，函数参数被实例化成普通的左值引用。因此我们实际上可以将任意类型的实参传递给T&&类型的函数参数。

编写右值引用参数的模板函数会有奇怪的影响：
```C++
template <typename T> void f3(T&& val) {
    T t = val; // 拷贝or引用
    t = fcn(t); // 如果上面是引用，val也会被改变
    if (val == t) { ... } // 引用的话条件就一直为true
}
```

实际中右值引用通常用于模板转发（16.2.7）或重载（16.3），使用右值引用的模板通常使用13.6.3见过的方法重载：
```C++
template <typename T> void f(T&&); // 非const右值
template <typename T> void f(const T&); // 左值和const右值
```

### 16.2.6 理解std::move
move本质上接受任何实参，因此它可以是一个函数模板。标准库定义move如下：
```C++
template <typename T>
typename remove_reference<T>::type&& move(T&& t) {
    return static_cast<typename remove_reference<T>::type&&>(t);
}
string s1("hi"), s2;
s2 = std::move(s1); // 赋值后s1的值不确定，本质上执行了static_cast<string&&>(string&)
```

### 16.2.7 转发
函数需要将某些实参连同类型转发给其他函数，这时需要保持被转发实参的所有性质，包括实参是否是const，以及左值还是右值。
```C++
template <typename F, typename T1, typename T2>
void flip(F f, T1 t1, T2 t2) {
    f(t2, t1);
}
```
此函数在f接受引用时会有问题，如果通过flip调用f，f不会影响实参，因为在调用flip时已经被拷贝。\
如果将参数定义为模板参数类型的右值引用，可以保持实参的类型信息，使用引用参数可以保持const属性，因为const在引用中是底层的：
```C++
template <typename F, typename T1, typename T2>
void flip(F f, T1&& t1, T2&& t2) {
    f(t2, t1);
}
void f(int, int&);
flip(f, j, 42); // t1得到左值j，T1被推断为int&，t1被折叠为int&，使得j被绑定，因此f改变t1的值也会改变j的值。
```
不过如果函数接受右值引用又会出问题：
```C++
void g(int&&, int&);
flip(g, i, 42); // 错误，g会得到一个左值t2
```

可以通过std::forward\<T>保持左值或右值属性。本质上forward\<T>返回T&&类型的值，如果T是左值引用，经过引用折叠之后还是左值引用，如果T是普通类型，实参是一个右值，转发后还是右值。
```C++
template <typename F, typename T1, typename T2>
void flip(F f, T1&& t1, T2&& t2) {
    f(std::forward<T2>(t2), std::forward<T1>(t1));
}
```

## 16.3 重载与模板
函数模板可以被另一个模板或普通函数重载。对于一个调用，候选函数为所有实参推断成功的模板实例。可行函数按类型转换排序。同样好的匹配中优先选择非模板函数，接着选择更特例化的模板。
```C++
template <typename T> string foo(const T& t);
template <typename T> string foo(T* t);

string s("hi");
foo(s); // 调用第一个版本
foo(&s); // 两个都可行，第一个实例化为foo(const string*&)，第二个实例化为foo(string*)，第二个是精确匹配，选择第二个。
const string* sp = &s;
foo(sp); // 两个都可行，第一个实例化为foo(const string*&)，第二个实例化为foo(const string*)，这时选择更specialized的第二个版本（第二个更特例化指的是第一个比第二个更通用）。

string foo(const string& t);
foo("hi"); // 两个模板和上面的非模板都可行，第一个T绑定为char[3]，第二个T为const char，非模板函数要求const char*到string的转换，因此不是精确匹配。第二个模板需要进行数组到指针的转换，这个算作精确匹配。因此在两个模板函数中选择特例化的那个，即第二个。
```

## 16.4 可变参数模板
```C++
template <typename T, typename... Args>
void foo(const T& t, const Args& ... rest) {
    cout << sizeof...(Args) << endl;
    cout << sizeof...(rest) << endl;
}

int i = 0; double d = 3.1;
string s = "hhhh";
foo(i, s, 42, d); // void foo(const int&, const string&, const int&, const double&)
foo(s, 42, "hi"); // void foo(const string&, const int&, const char[3]&);
foo("hi"); // void foo(const char[3]&)


```
Args是模板参数包，表示零个或多个额外类型参数，rest是函数参数包。查看包中元素个数使用sizeof...运算符，返回常量表达式，并且不对实参求值。

### 16.4.1 编写可变函数参数模板
可变参数函数通常是递归的：
```C++
template <typename T>
ostream &print(ostream &os, const T &t) {
    return os << t;
}
template <typename T, typename... Args>
ostream& print(ostream &os, const T &t, const Args&... rest) {
    os << t << " ";
    print(os, rest...); // 递归
}
```
递归结尾时两个函数均匹配，此时选择非可变参数版本。

### 16.4.2 包扩展
上述print函数中，const Args&...对Args进行扩展，将“const Args&”这种模式（pattern）应用到Args包的每个元素中，即如果调用print(cout, i, s, 42)，则实例化为ostream& print(ostream&, const int&, const string&, const int&);。rest...对rest进行扩展。\
这种扩展可以推广：
```C++
template <typename... Args>
ostream& print(ostream &os, const Args&... rest) {
    print(os, foo(rest)...);  // 等价于对每个参数调用foo
    print(os, foo(rest...)); // 等价于调用接受rest函数列表的foo
}
```

### 16.4.3 转发参数包
使用forward机制可以将可变参数不变地传递给其他函数。例如标准库容器emplace_back成员函数应该是一个可变参数模板，接受所包含元素的构造函数的参数，如果想要使用移动构造函数，需要保持实参的类型信息。
```C++
class StrVec {
    template <class... Args> void emplace_back(Args&&... args) {
        alloc.construct(first_free++, std::forward<Args>(args)...); // 扩展到对每个参数ti进行转发std::forward<Ti>(ti)
    } 
};
svec.emplace_back(s1 + s2); // 调用移动构造函数
```

## 16.5 模板特例化
通用模板不一定总是对任何模板实参适合。
```C++
template <typename T> int compare(const T&, const T&);
template <size_t N, size_t M>
int compare(const char (&)[N], const char (&)[M]);

const char *p1 = "hi", *p2 = "mom";
compare(p1, p2); // 调用第一个模板，因为指针无法转换为数组的引用
compare("hi", "mom"); // 第二个模板
```
对上述函数，处理字符指针（而不是数组），可以为第一个版本的compare定义一个模板特例化版本（因为这里我们不想用大于或小于，而是想用strcmp来比较）。一个特例化版本是模板的一个独立定义：
```C++
template <>
int compare(const char* const &p1, const char* const &p2) {
    return strcmp(p1, p2);
}
// template <typename T> int compare(const T&, const T&)的特例化，T为const char*
```
特例化本质上是一个实例，而非重载版本。函数被定义为特例化版本还是独立的非模板函数，会影响函数匹配。compare("hi", "mom")会在两个模板函数中选择，并选择调用接受字符数组的参数，因为更特例化。如果处理字符指针的版本不是一个特例化，而是一个非模板的函数，就会有三个可行的函数版本，提供同样好的匹配，这时会选择非模板版本。

类模板也可以特例化。例如可以对自己的类定义特定的标准库hash模板。一个特例化hash模板必须定义一个重载调用运算符，返回size_t，result_type和argument_type为调用运算符的返回类型和参数类型，以及默认构造函数和拷贝赋值运算符。hash需要在std命名空间中定义：
```C++
class A {
    friend class std::hash<A>; // 由于hash访问私有成员，需要声明为A的友元
private:
    string a;
    unsigned b;
public:
    bool operator==(const A& rhs) {
        return a == rhs.a && b == rhs.b;
    }
}
namespace std {
template <> // 指出正在全特例化模板
struct hash<A> {
    typedef size_t result_type;
    typedef A argument_type;
    size_t operator(const A& a) const;
    // 使用合成的拷贝控制成员和默认构造函数
};
size_t hash<A>::operator()(const A& a) const {
    // 需要和A::operator==兼容
    return hash<string>()(a.a) ^ hash<unsigned>()(a.b);
}
}
```

与函数模板不同，类模板特例化不必提供所有模板参数的实参，可以指定一部分模板参数或者参数的一部分特性。部分特例化本身也是一个模板：
```C++
template <typename T> struct remove_reference {
    typedef T type;
}
// 特例化
// 左值引用
template <typename T> struct remove_reference<T&> {
    typedef T type;
}
// 右值引用
template <typename T> struct remove_reference<T&&> {
    typedef T type;
}

int i;
remove_reference<decltype(42)>::type a; // 原始模板
remove_reference<decltype(i)>::type b; // 第一个特例化版本
remove_reference<decltype(std::move(i))>::type c; // 第二个特例化版本
```

可以只特例化类的特定成员函数：
```C++
template <typename T> struct Foo {
    Foo(const T& t = T()) : mem(t) {}
    void Bar() { ... }
    T mem;
}
template <>
void Foo<int>::Bar() {
    ... // 特例化Foo<int>的成员Bar()
}
```

# Chapter 18 用于大型程序的工具
## 18.1 异常处理
### 18.1.1 抛出异常
栈展开沿着嵌套函数的调用链查找最近的抛出对象类型匹配的catch模块执行。找不到catch则执行标准库函数terminate。

构造函数抛出异常时，对象可能被构造了一部分，这一部分也需要被销毁。析构函数不应该抛出异常。

抛出一条表达式时，表达式的静态编译类型决定异常对象的类型，throw抛出一个指向派生类对象的基类指针时只有基类部分被抛出。

### 18.1.2 捕获异常
异常类型需要是完全类型（有明确定义），可以是左值引用，不能是右值引用。

应将catch继承链最低端的类放在前面，防止处理派生类的语句一直不被调用。catch只允许const转换，派生类向基类转换，以及数组转换为指针、函数转换为指针。

一个单独的catch可能不能完整处理某个异常，当前catch在执行操作后可能决定由调用链上一层的catch继续处理，通过空的throw语句将当前异常对象沿着调用链向上传递。

通过使用catch(...)捕获所有异常。

### 18.1.3 try语句块与构造函数
构造函数执行初始值列表时就可能抛出异常，这时构造函数内部的try还没生效。这时需要将构造函数写成函数try语句块，使得一组catch既可以处理构造函数体，也可处理初始化过程：
```C++
template <typename T>
Blob<T>::Blob(std::initializer_list<T> il) try :
    data(std::make_shared<std::vector<T>>(il)) {

} catch(const std::bad_alloc &e) { ... }
```

### 18.1.4 noexcept异常说明
编译器确认函数不会抛出异常，可以进行特殊的优化操作。noexcept可以为函数做不抛出说明，紧跟参数列表后面，在尾置返回类型之前，标识该函数不会抛出异常。

如果noexcept标识的函数仍然throw了，编译器不会报错，程序运行时调用terminate，因此noexcept是要么不会抛出异常，要么压根不知道如何处理异常。调用者无需为noexcept函数的异常处理负责。

noexcept说明符接受一个可选的bool类型实参，表明函数“是否”会抛出异常。通常与noexcept运算符混合使用。\
noexcept一元运算符返回bool类型右值常量表达式，与sizeof一样不会求表达式的值。noexcept(e)当e调用的所有函数都是noexcept且e本身不含throw语句时为true：
```C++
void f() noexcept(noexcept(g())); 
// 当g承诺不抛出异常时f也不会抛出异常
```

noexcept的函数指针只能指向noexcept函数。可能抛出异常的函数指针也可以指向noexcept函数。基类虚函数不抛出异常，则派生类中的重写的虚函数也必须抛出，反之不一定。\
编译器合成拷贝控制成员时，如果所有成员和基类的操作都承诺noexcept，则合成的成员是noexcept的。\
如果我们没有为自定义的析构函数提供noexcept声明，编译器将根据自己合成析构函数时得到的异常说明来定义我们自定义析构函数的异常说明。

### 18.1.5 异常类层次

## 18.2 命名空间
### 18.2.1 定义
全局作用域中定义的名字定义在全局命名空间中，以隐式方式声明，例如::member_name表示全局命名空间中的一个成员。

新标准支持定义内联命名空间，其内部的名字可以被外层命名空间直接使用：
```C++
// "fifth.h"
inline namespace Fifth {
    class Query { ... };
}

// "fourth.h"
namespace Fourth {
    class Query { ... };
}

// "book.h"
namespace Book {
    #include "fifth.h"
    #include "fourth.h"
}

Book::Query ...; // 不需要显式指定fifth edition
Book::Fourth::Query ...; // 需要显式指定fourth
```

未命名的命名空间定义的变量拥有静态生命周期，在第一次使用前创建，直到程序结束才销毁。\
一个未命名的命名空间可以在一个文件中不连续，但是不能跨越多个文件。两个不同文件中的未命名命名空间无关，它们内部可以定义相同的名字。

### 18.2.2 使用命名空间成员
命名空间可以有别名：
```C++
namespace cplusplus_primer { ... };
namespace primer = cplusplus_primer;
namespace Qlib = cplusplus_primer::QueryLib;
```

using声明的范围从声明开始到using所在的作用域结束为止，外层作用域的同名实体被隐藏。using指示直接使用命名空间，相当于把命名空间成员放到包含命名空间本身和using指示的最近作用域，一般是最近的外层作用域：
```C++
namespace A {
    int i, j;
}
namespace B {
    int i, j, k;
}
int j = 0;
void foo() {
    using A::j; // using声明，相当于给局部作用域添加了变量j
    cout << j << endl; // A::j，全局j被隐藏
    using namespace B; // using指示，相当于给全局作用域添加了变量j
    cout << j << endl; // 仍然是A::j
}
void bar() {
    using namespace B;
    ++i; // B::i
    ++j; // 二义性
    ++::j; // 全局j
    ++B::j; // B::j
    int k = 6;
    ++k; // 局部k，B::k（可看做全局k）被隐藏
}
```

### 18.2.3 类、命名空间与作用域
只有在使用点之前被声明的名字才可以使用：
```C++
namespace A {
    int i;
    int f() { return j; } // j不可用
    int j = i;
}
```

给函数传递一个类类型的对象时，除了查找函数的常规作用域，还会在实参类的所属命名空间中查找，所以当我们调用std::cin/cout输入输出std::string时，编译器还会查找operator>>对应的命名空间，并最终找到string对应的输入输出运算符。

由于move和forward函数接受一个右值引用形参，可以匹配任何类型，因此只要我们定义一个接受一个形参的move函数，就会导致冲突。因此最好使用带限定语的std::move。

没有其他声明的友元函数会被隐式地变为最近的外层命名空间的成员：
```C++
namespace A {
    class C {
        // 这两个函数都成为A的成员
        friend void f2(); 
        friend void f(const C&);
    }
}

int main() {
    A::C obj;
    f(obj); // 可以通过查找obj对应的命名空间（A）找到f
    f2(); // 无法找到A::f2
}
```

### 18.2.4 重载与命名空间
using声明的是一个名字，而非一个特定的函数，不能指定形参列表。使用using声明可以将该函数的所有重载版本引入当前作用域，这些函数将重载作用域中已有的同名函数，如果有形参列表也相同的情况则出现错误。

using指示则是将命名空间内的函数提升到命名空间自己的作用域中，这些函数被添加到这个作用域的重载集合中。如果有形参列表相同的情况也不会错误，只要指明是哪个函数版本即可。

## 18.3 多重继承与虚继承
### 18.3.1 多重继承
C++11中允许派生类从一个或多个基类中继承构造函数。继承多个拥有同样参数列表的构造函数会导致错误。
```C++
struct B1 {
    B1() = default;
    B1(const string&);
};
struct B2 {
    B2() = default;
    B2(const string&);
};

// error
struct D1 : public B1, public B2 {
    using B1::B1;
    using B2::B2;
};

// D2 has to define its own constructor
struct D2 : public B1, public B2 {
    using B1::B1;
    using B2::B2;
    D2(const string& s) : B1(s), B2(s) {}
    D2() = default;
};
```
### 18.3.2 类型转换与多个基类
### 18.3.3 多重继承下的类作用域
### 18.3.4 虚继承
IO标准库的istream和ostream分别继承了base_ios的抽象基类，base_ios负责保存流的缓冲内容并管理流的条件状态。iostream同时继承istream和ostream，读写流的内容。\
如果iostream对象希望在同一个缓冲区操作，要去条件状态同时反映输入输出情况，则其最好只有一份base_ios的拷贝。

虚派生只影响从虚基类的派生类中进一步派生的类，不会影响虚基类的派生类本身。

如果虚基类中有成员x，派生类D1和D2均覆盖了x，则同时继承D1和D2的D需要重新定义新的x，不然会产生二义性错误。

### 18.3.5 构造函数与虚继承
最底层派生类负责初始化虚基类，如果最底层派生类没有显式初始化虚基类，则虚基类的默认构造函数将被调用，如果虚基类没有默认构造函数则发生错误。

```C++
class ZooAnimal;
class Bear : public virtual ZooAnimal {};
class Character;
class BookCharacter : public Character {};
class ToyAnimal;
class TeddyBear : public BookCharacter, public Bear, public virtual ToyAnimal {};

// when init TeddyBear, the sequence of calling constructors:
// ZooAnimal();
// ToyAnimal();
// Character();
// BookCharacter();
// Bear();
// TeddyBear();
```

# Chapter 19 特殊工具与技术
## 19.1 控制内存分配
某些程序需要自定义内存分配细节，比如使用关键字new将对象放置在特定内存空间中。因此程序需要重载new和delete运算符。

### 19.1.1 重载new和delete
new表达式执行时首先调用operator new或者operator new[]标准库函数，分配一块足够大的原始的未命名的内存空间用来存储对象或者数组。之后编译器运行相应的构造函数来构造对象，并传入初值。构造完成后返回指向该对象的指针。delete则先执行析构函数，再调用operator delete或者operator delete[]释放空间。\
应用程序可以再全局作用域定义自己的operator new和operator delete，也可以定义为成员函数。如果被分配的对象是类类型，则编译器首先在类中查找，如果该类含有operator new或者operator delete成员，则表达式将调用这些成员。否则编译器在全局作用域查找匹配的函数，如果找到用户自定义版本则执行该版本的new或者delete表达式。如果都没有则使用标准库定义的版本。可以使用作用域运算符忽略定义在类中的函数（::new或者::delete）。

类型nothrow_t定义在new头文件中，同时定义了一个名为nothrow的const对象，用户可通过此对象请求new的非抛出异常版本。与析构函数类似，operator delete也不允许抛出异常，重载时需要noexcept。\
此运算符函数定义为类成员时是隐式静态的，因为operator new用在对象构造之前，operator delete用在对象销毁之后，而且它们不能操纵任何数据成员。\
operator new必须返回void*，第一个形参必须是size_t且该形参不能含有默认实参。operator new的该形参指定对象所需的字节数，operator new[]的该形参指定数组所有元素所需的空间。\
自定义operator new可以提供额外的形参，这时使用new的定位形式，将实参传给新增的形参。第二个实参为void\*的operator new不能被用户重载。\
operator delete返回void，第一个形参类型必须是void\*。将operator delete定义为类的成员时该函数可以包含另外一个类型为size_t的形参，初始值为第一个形参所指对象的字节数，此形参可用于删除继承体系中的对象，如果基类有一个虚析构函数，则传递给operator delete的字节数将因动态类型有所区别。

可以使用malloc和free函数，malloc函数接受size_t表示待分配字节数，返回指向分配空间的指针。返回0表示分配失败。free函数接受void\*，是malloc返回的指针的副本，free将相关内存返回给系统。free(0)无意义。
```C++
void* operator new(size_t size) {
    if (void* mem = malloc(size)) {
        return mem;
    }
    else {
        throw bad_alloc();
    }
}
void operator delete(void* mem) noexcept { free(mem); }
```

### 19.1.2 定位new表达式
与allocator的allocate和deallocate类似，operator new和operator delete负责分配或者释放内存空间，但是不会构造或者销毁对象。不同的是operator new分配的空间无法使用construct构造对象，而是应该使用定位new构造对象。可以使用定位new传递一个地址，如new (place_address) type，其中place_address为指针，同时可以提供初始化列表。如果只有一个地址值，那么定位new就使用的是operator new(size_t, void*)，这是无法被用户自定义的版本。此函数不分配任何内存，只是返回指针实参。new表达式负责在指定的地址初始化对象。因此定位new允许我们在特定的，预先分配的内存地址上构造对象。\
传给定位new表达式的指针无需是operator new分配的内存，但是
传给construct的指针必须是同一个allocator分配的空间。

## 19.3 枚举类型
可使用enum class（或struct）定义限定作用域的枚举类型。不限定作用域的枚举类型省略掉关键字class（或struct），枚举类型名字可选。
```C++
enum class peppers {red, yellow, green};
enum color {red, yellow, green};
enum stoplight {red, yellow, green}; // 错误，重复定义枚举成员

color eyes = green;
peppers p = green; // 错误，peppers枚举成员不在有效作用域中
peppers p2 = peppers::red;
```
默认情况下枚举值从0开始，依次加1，不过可以为特定枚举成员指定专门的值，不同枚举成员可以有相同枚举值。没有指定则默认为之前枚举成员的值加1。\
枚举成员是const，因此初始化枚举值时需要常量表达式。可以定义枚举类型的constexpr变量，也可以将一个枚举类型作为switch表达式，非类型模板形参等等。

初始化enum对象或者赋值必须使用该类型的一个枚举成员或者另一个对象。不限定作用域的枚举类型对象可以自动转换成整型。
```C++
enum class open_modes {a, b, c};
open_modes om = 2; // 错误
open_modes om = open_modes::c;

int j = color::red;
int i = peppers::red; // 错误
```

新标准中可以在enum名字后加上冒号以及表示此enum的类型。默认情况下限定作用域的enum成员类型是int，不限定作用域不存在默认类型。
```C++
enum intValues : unsigned long long {
    charType = 255, shortType = 65535, intType = 65535,
    longType = 4294967295UL,
    longlongType = 18446744073709551615ULL
};
```

新标准中可以提前声明enum，不限定作用域必须指定成员类型：
```C++
enum intValues : unsigned long long;
enum class open_modes;
```

枚举值可以用unsigned char来作为底层类型，但是不管底层类型如何，对象和枚举成员最起码会提升为int。
```C++
enum Tokens { A = 128, B = 129 };
void f(unsigned char);
void f(int);
unsigned char uc = B;
f(B); // call f(int)
f(uc); // call f(unsigned char)
```

## 19.4 类成员指针
成员指针指向类的成员。类的静态成员不属于任何对象，指向静态成员的指针与普通指针没有区别。成员指针类型包含类的类型和成员类型，初始化时可以指向类的某个成员但是不指定该成员所属的对象，对象可以在使用时提供。

### 19.4.1 数据成员指针
```C++
class A {
public:
    char get() const;
    char get(int a, int b) const;
    char get_cursor() const;
private:
    string contents;
};

const string A::*pdata; // 可以指向A类的const string成员
pdata = &A::contents;

A a, *b = &a;
auto s = a.*pdata; // 解引用pdata得到a的contents成员
s = b->*pdata; // 获得b所指对象的contents成员
```
成员指针指向成员而非实际数据，使用的时候需要绑定到该类类型的对象上。

### 19.4.2 成员函数指针
指向成员函数的指针也需要指定目标函数的返回类型和形参列表，如果成员函数是const成员则还需要声明const限定符。如果成员重载，则需要显式指定函数类型：
```C++
auto pf = &A::get_cursor; // 指向A的某个常量成员函数，该函数不接受任何实参，返回一个char
char (A::*pf2)(int, int) const;
pf2 = &A::get;

A a, *b = &a;
char c1 = (b->*pf)();
char c2 = (a.*pf2)(0, 0);
```
可以通过类型别名将成员函数指针像其他类一样使用：
```C++
using Action = char(A::*)(int, int) const;
A& action(A&, Action = &A::get); // 第二个参数为Action类，默认实参为&A::get

A aa;
action(aa); // 使用默认实参

class A {
public:
    // ...
    A& up();
    A& down();

    using Action = A& (A::*)()

    A& move(int i);
private:
    static Action Menu[]; // 函数表，保存每个对应函数的指针
};

A& A::move(int i) {
    return (this->*Menu[i])();
}
```

### 19.4.3 将成员函数用作可调用对象
想通过成员函数指针进行函数调用，首先需要利用运算符.\*或者->\*绑定至特定对象，因此成员函数指针不是可调用对象。\
可以使用function生成可调用对象。
```C++
auto fp = &string::empty;
find_if(vec.begin(), vec.end(), fp); // 错误，find_if内部调用if (fp(*it))，会产生调用错误
function<bool (const string&)> fcn = &string::empty; // function将fcn(*it)转换为((*it).*p)()
```
这里function需要显式指定函数类型，其中函数第一个形参通常为相应对象的类型（即成员函数所属的类）。

可以使用mem_fn推断成员类型：
```C++
find_if(vec.begin(), vec.end(), mem_fn(&string::empty()))

auto f = mem_fn(&string::empty); // f接受一个string或者string*，不需要像function里一样显式指定
f(*vec.begin()); // 使用.*调用
f(&vec[0]); // 使用->*调用
```

还可以使用bind生成可调用对象：
```C++
find_if(vec.begin(), vec.end(), bind(&string::empty, _1));

auto f = bind(&string::empty, _1);
f(*vec.begin()); // 使用.*调用
f(&vec[0]); // 使用->*调用
```

## 19.8 固有的不可移植的特性
不可移植特性指因机器而异的特性，含有不可移植特性的程序转移到另一台机器上时通常需要被重新编写。例如算术类型大小在不同机器上不同。
### 19.8.1 位域
类可以将非静态成员定义为位域（bit-field），一个位域中含有一定数量的二进制位。位域类型为整型或枚举类型。通常使用无符号类型。位域声明形式为成员名字后紧跟冒号以及常量表达式，用于指定成员占的二进制位数：
```C++
typedef unsigned int Bit;
class File {
    Bit mode: 2;
    Bit modified: 1;
    // ...
public:
    enum modes { READ = 01, WRITE = 02, EXECUTE = 03 };
    File &open(modes);
    // ...
};
```
类的内部连续定义的位域可能会被压缩在同一个整数的相邻位。如何压缩是与机器相关的。\
取地址运算符（&）无法作用于位域。

通常使用位运算符操作位域。通常类也会定义一组内联的成员函数以检验或设置位域的值。

### 19.8.2 volatile限定符
直接处理硬件的程序可能包含值由程序之外的过程控制的数据元素。这种对象应该被声明为volatile，告诉编译器该对象不能被优化。
```C++
volatile Task* curr_task; // 指向一个volatile对象
volatile int arr[10]; // 每个元素都是volatile
volatile Screen bitmapBuf; // 每个成员都是volatile 
```
volatile与const互不影响。\
只有volatile的成员函数才能被volatile的对象调用。

合成的拷贝或移动构造函数以及赋值运算符不能用于volatile对象初始化或赋值。合成的成员接受的形参为常量应用，不能绑定到volatile对象上。一个类可以自定义拷贝或移动操作：
```C++
class A {
public:
    A(const volatile A&);
    // 将volatile对象赋值给非volatile对象
    A& operator=(volatile const A&);
    // 将volatile对象赋值给volatile对象
    A& operator=(volatile const A&) volatile;
};
```

### 19.8.3 链接指示：extern "C"
C++程序需要调用其他语言编写的函数，例如C语言函数。其他语言函数名字也需要在C++中进行声明，指定返回类型以及形参列表。编译器检查调用的方式与普通C++函数相同，生成的代码有所区别。
```C++
// probably appeared in <cstring>
extern "C" size_t strlen(const char*);
extern "C" {
    int strcmp(const char*, const char*);
    char* strcat(char*, const char*);
}

// 可以直接应用于整个头文件
// 头文件中所有普通函数声明都被认为是链接指示的语言编写的
// 链接指示可以嵌套，如果头文件中包含链接指示则不受影响
extern "C" {
    #include <string.h>
}
```
指向其他语言编写的函数的指针必须与函数本身使用相同的链接指示。一个指向C函数的指针不能在初始化或赋值操作后指向C++函数，反之亦然：
```C++
void (*pf1)(int);
extern "C" void (*pf2)(int);
pf1 = pf2; // 错误，pf1和pf2类型不相同
```
链接指示对作为返回类型或者形参类型的函数指针也有效：
```C++
// f1为一个C函数，形参为指向C函数的指针
extern "C" void f1(void(*) (int));

// 给C++函数传入C函数指针需要使用类型别名
extern "C" typedef void FC(int);
void f2(FC*);
```
如果在C和C++编译同一个源文件，可以利用预处理器__cplusplus变量：
```C++
#ifdef __cplusplus
extern "C"
#endif
int strcmp(const char*, const char*);
```
C语言不支持重载函数，因此一个C链接指示只能说明重载函数中的一个。使用类类型形参的C++函数只能在C++程序中调用。