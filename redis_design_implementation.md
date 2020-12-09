# Chapter 2 简单动态字符串
Redis需要可以被修改的字符串时使用简单动态字符串（simple dynamic string, SDS），例如SET msg "hello world"创建一个kv pair，key是保存"msg"的SDS，value是保存"hello world"的SDS。
## 2.1 SDS的定义
```C++
struct sdshdr {
    int len; // 已使用字节（SDS长度，不包含最后的'\0'）
    int free; // buf数组未使用字节
    char buf[];
};
```
## 2.2 SDS与C字符串的区别
获取SDS长度的复杂度为常数，strlen复杂度为O(N)。\
SDS API修改时会检查空间是否满足修改需求，不满足则自动扩展。\
每次增长或者缩短一个C字符串都需要进行内存重分配操作。Redis中的数据可能被频繁修改，因此实现了空间预分配与惰性空间释放两种优化策略，解除字符串长度与底层字符数组的长度关联。\
空间预分配：如果修改后SDS长度小于1MB则len与free相同，如果大于则free为1MB。\
惰性空间释放：缩短之后多出来的空间不释放而是作为free。

为了确保适应各种使用场景，SDS的API是二进制安全的，不会对其中数据做限制（例如字符串看到'\0'则认为是字符串终止，SDS使用len判断字符是否终止，保存'\0'是为了可以使用某些C字符串的API，例如printf, strcmp等）。

# Chapter 3 链表
链表是列表键，发布订阅功能，慢查询，监视器等等底层实现。
## 3.1 链表和链表节点实现
```C++
typedef struct listNode {
    struct listNode* prev;
    struct listNode* next;
    void* value;
}listNode;

typedef struct list {
    listNode* head;
    listNode* tail;
    unsigned long len; // 链表长度
    void *(*dup) (void* ptr); // 复制节点值
    void (*free) (void* ptr); // 释放节点值
    int (*match) (void* ptr, void* key); // 比较节点值和另一个值
} list;
```
使用void\*保存节点值，通过函数指针对不同类型设置特定函数，使得链表可以保存不同类型的值

## 3.2 链表和链表节点的API

# Chapter 4 字典
Redis的数据库以及哈希键使用字典作为底层实现。

## 4.1 字典实现
字典使用哈希表作为底层实现，一个哈希表有多个哈希表节点，每个节点保存字典的一个键值对。

```C++
// 哈希表
typedef struct dictht {
    dictEntry **table; // 哈希表数组
    unsigned long size; // 大小
    unsigned long sizemask; // 掩码用于计算索引值（等于size - 1，当size为2^n时，x & sizemask == x % size)
    unsigned long used; // 已有节点数量
} dictht;

// 哈希表节点
typedef struct dictEntry {
    void* key; // 键

    // 值可以是指针或者（unsigned）64位整数
    union {
        void* val;
        uint64_t u64;
        int64_t s64;
    } v;

    struct dictEntry *next; // deal with collision
} dictEntry;

// 字典
typedef struct dict {
    dictType *type;
    void *privdata;

    dictht ht[2];

    int rehashidx; // rehash索引，不rehash时值为-1
} dict;

typedef struct dictType {
    // 哈希函数
    unsigned int (*hashFunction) (const void* key);

    // 复制key的函数
    void* (*keyDup) (void *privdata, const void* key);

    // 复制value的函数
    void* (*valDup) (void *privdata, const void* obj);

    // 比较key
    int (*keyCompare) (void *privdata, const void* key1, const void* key2);

    void (*keyDestructor) (void *privdata, void* key);

    void (*valDestructor) (void *privdata, void* obj);
} dictType;
```
dict的type和privdata针对不同类型键值对，对多态字典提供支持。type属性指向dictType，保存一系列操作特定类型键值对的函数，用途不同的字典设置不同的函数实现。privdata则保存传给这些特定函数实现的可选参数。\
dict的ht属性包含两个dictht，一般情况下只使用ht[0]，ht[1]在rehash时使用。rehashidx记录rehash的进度。

## 4.2 哈希算法
将一个新的kv添加到字典的时候需要先根据kv的key计算出哈希值和索引值，再根据索引值添加到哈希表的指定索引：
```C++
hash = dict->type->hashFunction(key);
index = hash & dict->ht[x].sizemask;
```
字典被用作数据库或者哈希键的底层实现时Redis使用MurmurHash2算法计算哈希值。

## 4.3 解决键冲突
索引冲突时可以借助next建立链表。新节点插入链表表头的位置复杂度为常数。

## 4.4 rehash
哈希表保存的kv会随着操作执行而增加或者减少。为了让哈希表的load factor（元素个数/哈希表长度）保持在合理范围内，需要对哈希表进行扩展或者收缩，这可以通过rehash来完成。\
rehash时先对ht[1]分配空间，空间大小取决于要执行的操作，以及ht[0].used的值，如果是扩展操作，则ht[1]的大小为第一个大于等于ht[0].used\*2的2次方幂，如果是收缩操作，则ht[1]的大小为第一个大于等于ht[0].used的二次方幂。之后将ht[0]中的kv重新计算哈希值和索引值，放置在ht[1]中。之后将ht[1]设为ht[0]，ht[0]变为空表并被释放，在ht[1]处新建一个空白哈希表，为下一次rehash做准备。

如果(1)服务器没有执行BGSAVE或者BGREWRITEAOF命令，并且哈希表负载因子大于等于1，或者(2)服务器正在执行BGSAVE或者BGREWRITEAOF命令，且负载因子大于等于5，则程序会对哈希表进行扩展。因为执行BGSAVE或者BGREWRITEAOF命令时，Redis会创建当前服务器进程的子进程，因此需要提高负载因子避免哈希表扩展，避免不必要的内存写入操作。\
负载因子小于0.1时会自动对哈希表进行收缩操作。

## 4.5 渐进式rehash
为了避免rehash对性能造成影响，服务器分多次，渐进式将ht[0]中的kv移动到ht[1]。\
rehash启动时rehashidx被设为0，表示rehash开始。每次字典执行增删改查操作时，程序除了指定操作外，还会顺带将ht[0][rehashidx]位置上的所有kv移动到ht[1]，完成后rehashidx自增1。最终rehash完成时rehashidx被设为-1表示rehash完成。这样rehash操作被均摊到每个增删改查操作上。

渐进式rehash期间，字典的删除、查找、更新操作会在两个哈希表进行，如果ht[0]中没找到则在ht[1]中找。新插入的kv一律保存至ht[1]，保证ht[0]中的kv数量只减不增。

## 4.6 字典API

# Chapter 5 跳跃表

跳跃表（skiplist）是一种有序数据结构，通过在每个节点中维持多个指向其他节点的指针，达到快速访问节点的目的。支持平均O(logn)，最坏O(N)的节点查找，还可以通过顺序操作批量处理节点。Redis使用跳跃表作为有序集合的底层实现之一。另一个用到跳跃表的地方是集群节点内部数据结构。

## 5.1 跳跃表实现
zskiplistNode表示跳跃表节点，包含节点层数，每层有两个属性，前进指针和跨度，记录表尾方向的某一节点以及该节点与当前节点的距离。节点还包含后退指针，指向当前节点的前一个节点，以及分值，节点按照分值从小到大排列。每个节点还保存一个成员对象。表头节点的后退指针、分值和成员对象都不会被用到。zskiplist保存跳跃表相关信息，包含跳跃表头尾节点，层数最大的节点层数，跳跃表长度（不包含表头节点）。

### 5.1.1 跳跃表节点
```C++
typedef struct zskiplistNode {
    struct zskiplistLevel {
        struct zskiplistNode* forward;
        unsigned int span;
    } level[];

    struct zskiplistNode* backward;
    double score;
    robj* obj;
} zskiplistNode;
```
level数组可以包含多个元素，通过这些不同额层加快节点访问速度。每次创建新节点，程序随机生成一个介于1-32之间的值作为level数组大小，即层的高度。\
遍历跳跃表时首先访问表头，然后从span为1的最高层节点访问到下一个节点。如果某个forward为NULL则遍历结束。事实上每一“层”相当于一层索引，如果节点当前层的forward超过了查找范围，则说明要查找的元素在两个节点之间，则下移一层继续查找。因此查找过程是从高层开始的。\
跨度与遍历操作无关，而是用来计算排位的，在查找某个节点的过程中，将沿途访问过的所有层跨度累积起来就是目标节点在跳跃表中的排位。\
节点的成员对象是一个指针，指向字符串对象，保存一个SDS值。同一个跳跃表中各节点的成员对象是唯一的，但是分值可以相同，分值相同的节点将按字典序排序。

### 5.1.2 跳跃表
```C++
typedef struct zskiplist {
    struct zskiplistNode* header, *tail;
    unsigned long length;
    int level;
} zkiplist;
```

## 5.2 跳跃表API

# Chapter 6 整数集合
整数集合（intset）是集合的底层实现之一。

## 6.1 整数集合的实现
```C++
typedef struct intset {
    uint32_t encoding; // 编码方式
    uint32_t length; // 元素数量
    int8_t contents[]; // 元素数组
} intset;
```
各个元素在数组中按值得大小有序排列，数组中不包含重复项。length记录元素数量，即数组长度。contents数组的真正类型取决于encoding的值。如果encoding为INTSET_ENC_INT16，则contents的每个元素都是int16_t类型。根据整数集合的升级（upgrade）规则，当向一个底层为int16_t的数组添加int64_t类型的整数时，所有元素都会被转换成int64_t类型。

## 6.2 升级
添加一个类型比整数集合现有元素类型长的元素时，整数集合需要先升级。首先根据新元素的类型，扩展整数集合底层数组的大小，并为新元素分配空间。之后将底层数组现有元素都转换成新元素类型，并放置到正确的位上，维持有序性质不变。实现时是在原数组之后分配需要的新空间，之后将现有元素依次移到对应的位上（不升级时应该是使用memmove进行操作），并添加新元素。这样向整数集合添加新元素的复杂度为O(N)。

## 6.3 升级的好处
支持将各种类型的整数添加到集合中，提升了灵活性。同时相比于直接使用int64_t数组，节约了内存。

## 6.4 降级
整数集合不支持降级操作，即使删除了唯一一个int64_t元素，集合也会保持int64_t类型。

## 6.5 整数集合API