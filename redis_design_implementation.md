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

# Chapter 7 压缩列表
压缩列表（ziplist）是列表和哈希的底层实现之一，当列表只包含少数项，并且每个项要么是小整数值，要么是长度短的字符串，则使用压缩列表作为列表实现。当哈希只包含少量kv，且kv要么是小整数值，要么是长度短的字符串，则使用压缩列表实现。

## 7.1 压缩列表的构成
压缩列表是Redis为了节约内存而开发的，是由一系列特殊编码的连续内存块组成的顺序性数据结构，一个压缩列表可以包含任意多个节点，每个节点保存一个字节数组或者整数值。\
zlbytes记录压缩列表占用字节数，在进行内存重分配或者计算列表末端位置时使用。zltail记录表尾节点与起始地址之间距离多少字节，这样可以快速确定表尾节点地址。zllen记录了压缩列表的节点数量，如果等于UINT16_MAX，则节点真实数量需要遍历列表才能得出。每个节点的长度由节点保存的内容决定。zlend等于0xFF用于标记列表末端。

## 7.2 压缩列表节点的构成
每个节点可以保存一个数组或者整数值，字节数组长度可以为2^6-1, 2^14-1或者2^32-1。整数长度可以为4位，1字节，3字节，int16_t，int32_t，int64_t。每个节点由previous_entry_length，encoding，content三个部分组成。\
previous_entry_length以字节为单位，记录前一个节点的长度。该属性本身的长度可以是1字节或者5字节，如果前一节点长度小于254字节，则长度为1字节，否则长度为5字节，其中第一个字节会被设置为0xFE(254)，之后四个字节用于保存前一节点的长度。这样程序可以通过指针运算计算出前一个节点的起始地址，使得压缩列表可以从表尾向表头遍历。\
encoding属性记录了节点的content保存的类型与长度。如果是一字节，两字节，或者五字节长，且高位为00、01、10的是字节数组编码。数组长度由除去最高两位后的其他位记录（encoding的长度分别对应2^6-1,2^14-1,2^32-1长的字节数组）。一字节长，值的最高位为11开头的是整数编码，整数值的类型和长度由编码除去最高两位之后的其他位记录。\
content属性保存节点的值。

## 7.3 连锁更新
如果一个压缩列表每个节点大小都介于250-253字节之间，这时表头插入一个插入一个长度大于254的节点，则后面每个节点的previous_entry_length都需要变为5字节长，这样每个节点的空间都需要重新分配。这种连锁更新（cascade update）最坏需要对压缩列表执行N次空间重分配操作，每次重分配最坏复杂度为O(N)，因此最坏复杂度为O(N^2)。平均而言，ziplistPush的平均复杂度为O(N)，性能不会太受影响。

## 7.4 压缩列表API

# Chapter 8 对象
Redis基于上述数据结构构建了一个对象系统，包含字符串对象、列表对象、哈希对象、集合对象和有序集合对象。Redis可以根据对象类型判断对象是否可以执行给定的命令，也可以针对不同场景为对象设置多种不同的实现，优化使用效率。\
Redis对象系统实现了基于引用计数的内存回收机制以及对象共享机制，可以通过多个数据库键共享同一个对象。\
Redis的对象带有访问时间记录信息，用于计算数据库键的空转市场，在服务器启动maxmemory功能时空转时长较大的键可能会被优先删除。

## 8.1 对象的类型和编码
Redis新建键值对时会创造一个键的对象以及一个值对象。
```C++
typedef struct redisObject {
    unsigned type:4; // 类型
    unsigned encoding:4; // 编码

    void *ptr; // 指向底层实现
} robj;
```
type属性记录了对象类型，key总是一个字符串对象，值可以是五种对象的任何一个。对一个key执行TYPE命令返回的是对应的value的类型。\
对象的底层实现数据结构由encoding属性决定，每种类型至少对应两种编码，即两种不同实现方式。字符串对象可以由整数值、SDS或者embstr编码的SDS实现。列表可以由压缩列表或者双端链表实现。哈希对象可以由压缩列表或者字典实现。集合对象可以由整数集合或者字典实现。有序集合可以由压缩列表或者跳跃表实现。这样可以优化不同场景的效率。例如列表包含很少的元素时，使用压缩列表节约内存，同时以连续块方式保存的压缩列表可以很快被载入缓存中。

## 8.2 字符串对象
如果一个字符串对象保存可以用long表示的整数值，则整数值被保存在ptr中，编码为int。如果保存大于32字节的字符串值，则使用SDS来保存字符串，编码为raw。如果小于32字节的字符串，则使用embstr编码的方式保存。这种编码方式会调用一次分配函数分配一块连续空间连续保存redisObject和对应的sdshdr结构，而不是像raw一样为两个结构分别分配空间。这样内存分配次数减少，内存释放次数相应减少，连续内存可以更好地利用缓存。\
可以用long double表示的浮点数是作为字符串值保存的，需要的时候程序会将其转换回浮点数类型。

### 8.2.1 编码的转换
int编码和embstr编码在条件满足时会被转换为raw编码。如果对对象执行某些命令（例如APPEND命令）将保存的整数值转换为字符串值，则int转换为raw。embstr本质是只读的，如果要修改，则首先转换为raw编码的对象再修改。这样embstr在被修改后总会变为raw。

### 8.2.2 字符串命令实现
SETRANGE在int和embstr转换为raw，之后在给定索引上设置相应字符。

## 8.3 列表对象
列表使用压缩列表实现时每个节点保存一个列表元素，使用链表实现时每个节点保存一个字符串对象，存储一个列表元素。字符串对象时唯一一种会被其他类型嵌套的的对象。

### 8.3.1 编码转换
如果列表对象所有字符串元素小于64字节，且元素数量小于512个，则采用ziplist编码，否则使用linkedlist编码。当任意条件不被满足时，对象的编码会自动转换，所有元素会从压缩列表转移到双端链表中。

### 8.3.2 列表命令实现
LPUSH在表头插入，RPUSH在表尾插入。LREM删除给定节点。LSET删除指定索引节点并重新在该索引插入给定节点。LTRIM删除不在指定索引范围内节点。

## 8.4 哈希对象
使用压缩列表实现时，key和value各对应一个压缩列表元素，插入时先插入key再插入value。使用字典实现时每个kv都用一个字典键值对保存。

### 8.4.1 编码转换
所有key和value长度都小于64字节且kv个数小于512时使用压缩列表，否则使用字典（hashtable编码）。

### 8.4.2 哈希命令的实现
HGETALL返回所有kv。

## 8.5 集合对象
集合对象可以使用整数集合作为底层实现，集合中的元素对应整数集合中的元素。集合对象也可用字典作为底层实现，字典每个key是一个字符串对象，包含集合的每个元素，字典的值为NULL。

### 8.5.1 编码的转换
如果集合对象的元素都是整数值，且元素数量不超过512个，则集合对象使用intset编码（整数集合实现）。如果两个条件不满足的话则执行编码转换操作，元素被转移到字典中。

### 8.5.2 集合命令的实现
SCARD返回元素数量，SMEMBERS返回所有元素。SRANDMEMBER返回任一元素，SPOP随机选取一个元素删除并返回。

## 8.6 有序集合对象
有序集合可以使用压缩列表（ziplist编码）作为底层实现，每个有序集合元素用两个紧挨的压缩列表元素保存，第一个保存有序集合成员，第二个保存元素分值。压缩列表元素按照分值从小到大进行排序。\
有序集合还可以使用zset结构（skiplist编码）作为底层实现，一个zset结构同时包含一个字典和一个跳跃表：
```C++
typedef struct zset {
    zskiplist *zsl;
    dict *dict;
} zset;
```
zsl跳跃表按照分值从小到大保存了所有集合元素，每个节点对应一个集合元素，跳跃表的object属性保存元素成员，score属性保存元素分值。这样有序集合可以进行范围操作（ZRANK，ZRANGE等）。dict为有序集合创建了成员到分值的映射，这样可以在常数时间找到给定成员的分值（ZSCORE）。有序集合每个元素成员都是一个字符串对象，分值是double类型浮点数。底层实现中跳跃表和字典会通过指针共享相同元素的成员和分值，所以不会浪费额外内存。这里跳跃表保证了“有序”，字典提供了“集合”的功能（查找等）。

### 8.6.1 编码的转换
当有序集合元素少于128个，且所有元素成员长度小于64字节时使用ziplist编码。

### 8.6.2
ZCARD返回元素数量。ZCOUNT统计给定范围内节点数量。ZRANGE返回范围内所有元素。ZRANK返回元素的排名。

## 8.7 类型检查与命令多态
Redis的命令可以分为两种类型，其中一种可以对任何类型的对象执行，比如DEL、EXPIRE、RENAME、TYPE、OBJECT等。另一种命令只能对指定对象执行。

### 8.7.1 类型检查的实现
因此在执行一个类型特定的命令之前，先检查对象是否为所需类型，如果类型不匹配则返回一个类型错误。例如执行LLEN前需要检查redisObject的type属性是否为REDIS_LIST。

### 8.7.2 多态命令的实现
Redis还会根据对象的编码方式选择正确的命令来执行命令。例如列表对象可以使用ziplist或者linkedlist编码，因此如果执行LLEN，则需要根据编码选择对应的命令实现（ziplistLen函数或者listLength函数）。这种是基于编码的多态，DEL、EXPIRE等是基于类型的多态。

## 8.8 内存回收
每个对象的引用计数信息在redisObject结构中的refcount属性记录：
```C++
typedef struct redisObject {
    // ...
    int refcount;
    //...
} robj;
```
创建新对象时被初始化为1，被一个新程序使用时引用计数自增，不再被使用时自减，为0则对象被释放。

## 8.9 对象共享
引用计数可以支持对象共享。例如键A和键B均包含一个整数值100的字符串对象作为值对象，则可以共享这一对象。目前Redis会在初始化服务器时创建0-9999的字符串对象，用于共享（此参数可以调整）。如果这时创建一个包含整数值100的字符串对象的键，则该对象引用计数为2，因为服务器程序也引用了该对象。\
Redis只对包含整数值的字符串对象进行共享，因为共享其他对象需要先检查两个程序所需的对象完全相同，这需要大量CPU时间，复杂度过高。

## 8.10 对象的空转时长
redisObject包含lru属性，记录对象最后一次被访问的时间：
```C++
typedef struct redisObject {
    // ...
    unsigned lru:22;
    // ...
} robj;
```
OBJECT IDLETIME命令可以通过当前时间减去lru时间。这一命令通过特殊实现，不会修改值对象的lru属性。\
如果服务器打开maxmemory属性，且回收内存算法为volatile-lru或者allkeys-lru，那么空转时长高的键会被优先释放，回收内存。