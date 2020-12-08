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
