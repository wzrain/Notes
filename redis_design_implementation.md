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

### 8.6.2 有序集合命令的实现
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

# Chapter 9 数据库
## 9.1 服务器中的数据库
Redis服务器将所有的数据库都保存在redis.h/redisServer结构的db数组中，每个项都是一个redis.h/redisDb结构，代表一个数据库。程序根据dbnum属性决定创建多少个数据库，该属性值由配置的database选项决定：
```C++
struct redisServer {
    // ...
    redisDb *db;
    int dubnum;
    // ...
};
```

## 9.2 切换数据库
每个客户端都有自己的目标数据库，客户端执行读写命令的时候目标数据库就是操作对象。默认情况下目标数据库为0号数据库，客户端可以通过SELECT命令来切换目标数据库。\
服务器内部客户端状态redisClient结构的db属性记录了当前正在被客户端使用的目标数据库，属性是指向一个redisDb结构指针。SELECT命令通过修改指针指向的数据库对象实现其功能。
```C++
typedef struct redisClient {
    // ...
    redisDb *db;
    // ...
} redisClient;
```

## 9.3 数据库键空间
redisDb结构的dict字典保存了所有键值对，这个字典被称为键空间（key space）。键空间的每个键即是数据库的键，是一个字符串对象，值为数据库的值，每个值可以是五种对象的任意一种。
```C++
typedef struct redisDb {
    // ...
    dict* dict;
    // ...
} redisDb;
```
添加删除更新取值操作都可以使用值对象对应的命令完成。此外还有针对数据库本身的Redis命令，例如FLUSHDB命令，就是通过删除键空间中所有键值对来实现的，RANDOMKEY随机返回是一个键，DBSIZE返回键值对的数量。\
每次读写还会进行一些额外的维护操作。访问一个键后要更新服务器键空间的命中（hit）或者不命中（miss）次数，可以在INFO stats命令的keyspace_hits和keyspace_misses属性查看。访问一个键后会更新其lru时间，可通过OBJECT idletime命令查看闲置时间。如果访问时发现该键过期则先删除过期键。客户端使用WATCH监视某个键时，修改该键后会将此键标记为dirty，每次修改都会对脏键计数器的值增1，这个计数器会触发持久化以及复制操作。如果有数据库通知功能则服务器会发送相应的通知。

## 9.4 设置键的生存时间或过期时间
通过EXPIRE或者PEXPIRE命令以秒或者毫秒为精度设置生存时间（TTL），经过TTL后服务器自动删除生存时间为0的键。SETEX命令在设置一个字符串键的同时会为键设置过期时间，只可用于字符串键。\
EXPIREAT或者PEXPIREAT可以设置过期时间。TTL和PTTL命令返回该键剩余生存时间。

设置过期时间的命令本质上都使用PEXPIREAT实现。redisDb结构的expires字典保存了所有键的过期时间，过期字典的键是一个指向键空间中某个对象的指针，值为long long类型整数，为毫秒精度的时间戳。客户端执行PEXPIREAT命令时服务器会在过期字典中关联给定的键和对应过期时间。可以通过PERSIST命令移除过期时间，即删除过期字典中对应的键。TTL和PTTL命令通过查找过期字典计算生存时间，如果TTL返回值大于0则说明该键未过期。
```C++
typedef struct redisDb {
    // ...
    dict* expires;
    // ...
} redisDb;
```

## 9.5 过期键删除策略
定时删除通过创建定时器（timer）在键过期时立即删除，此策略对内存友好，不过对CPU时间不友好，尤其是过期键较多的情况下。此外创建定时器需要使用服务器的时间事件，当前时间事件实现方式为无序链表，查找复杂度为O(N)，因此创建大量定时器并不现实。\
惰性删除在访问键时先检查过期与否，如果过期则删除。该策略对CPU友好，不过对内存不友好，如果不被访问到则永远不会被删除（例如日志数据）。\
定期删除则是每隔一段时间对数据库进行一次检查，删除过期键的数量以及检查数据库的个数由算法决定。此策略为上述两种策略的折中，难点为删除频率以及执行时长。

## 9.6 Redis的过期键删除策略
Redis使用惰性删除与定期删除两种策略。惰性删除策略由db.c/expireIfNeeded函数实现，所有读写数据库的命令执行前都会调用该函数对输入键进行检查，如果已过期则该函数将键删除。这样每个命令需要将被删除的过期键按照不存在处理。定期删除策略由redis.c/activeExpireCycle实现，每次服务器周期操作redis.c/serverCron函数执行时，activeExpireCycle函数就会被调用，在规定时间内分多次遍历服务器的各个数据库，随机检查一部分键的过期时间，删除其中过期的键。其中的全局变量current_db会记录当前检查的进度，在下一次调用时接着上次的进度处理，所有数据库被检查完后current_db被设为0，开始新一轮检查。

## 9.7 AOF、RDB和复制功能对过期键的处理
在执行SAVE或者BGSAVE创建新的RDB文件时，数据库中的键会被检查，过期的键不会被保存到RDB文件中。启动Redis服务器时，如果开启RDB功能则RDB文件会被载入，如果服务器以主服务器模式运行，则载入RDB文件时文件中的键会被检查，未过期的键会被载入数据库中。从服务器模式下键不论是否过期都会被载入，不过主从数据同步时从服务器数据库会被清空，因此不会有影响。

服务器以AOF持久化模式运行时如果某个键已经过期，在删除前对AOF文件无影响，被删除后AOF文件会被append一条DEL命令，显式记录该键被删除。因此如果客户端访问某个过期键，则服务器从数据库中删除该键，并追加一条DEL命令到AOF文件，并向该访问返回一个空回复。执行AOF重写的过程中，已经过期的键不会被保存到重写后AOF的文件中。

服务器运行复制模式时，删除过期键由主服务器控制。主服务器删除过期键之后会显式向所有从服务器发送DEL命令。从服务器执行客户端的读命令时会像处理未过期的键一样处理过期键，只有收到DEL命令后才会被删除。这样可以保持主从一致性，在主服务器存在的过期键也会在从服务器中存在。

## 9.8 数据库通知
数据库通知是Redis 2.8版本新增加的功能，可以让客户端通过订阅给定的频道或者模式，来获知数据库中键的变化，以及命令执行情况。使用SUBSCRIBE命令进行订阅。键空间通知（__keyspace）关注某个键执行了什么命令，另一类通知是键事件（__keyevent）通知，关注某个命令被什么键执行了。\
服务器配置的notify-keyspace-events选项决定了服务器所发送的通知类型。如果发送所有类型的键空间通知和键事件通知，类型为AKE。所有类型的键空间通知，则类型为AK。所有类型的键事件通知，类型为AE。只发送和字符串键有关的键空间通知，类型设置为K$。只发送列表键有关的键事件通知，类型为El。

### 9.8.1 发送通知
发送数据库通知的功能是由notify.c/notifyKeyspaceEvent实现的：
```C++
void notifyKeyspaceEvent(int type, char* event, robj* key, int dbid);
```
type是想要发送通知的类型，程序根据类型判断是否对命令发送通知。event是事件的名称，key是产生事件的键，dbid是对应的数据库编号，函数根据这些参数构建通知的内容以及频道名。每当一个命令需要发现数据库通知时，其内部实现就会调用notifyKeyspaceEvent函数，例如SADD命令与DEL命令的实现中：
```C++
void saddCommand(redisClient* c) {
    // ...
    if (added) {
        // ...
        notifyKeyspaceEvent(REDIS_NOTIFY_SET, "sadd", c->argv[1], c->db->id);
        // ...
    }
    // ... 
}

void delCommand(redisClient* c) {
    int deleted = 0, j;

    for (j = 1; j < c->argc; j++) {
        if (dbDelete(c->db, c->argv[j])) {
            // ...
            notifyKeyspaceEvent(REDIS_NOTIFY_GENERIC, "del", c->argv[j], c->db->id);
            // ...
        }
    }
    // ...
}
```

### 9.8.2 发送通知的实现
notifyKeyspaceEvent函数首先判断通知是否为服务器允许发送的通知（查看notify_keyspace_events设置），如果是的话，再根据需要分别发送键空间通知和键事件通知。发送时会构建一个频道名字，并且将事件或者键的信息发布到对应频道。pubsubPublishMessage是PUBLISH命令的实现，订阅数据库通知的客户端收到的信息由这个函数发出。

# Chapter 10 RDB持久化
RDB持久化功能将Redis在内存中的数据库状态保存到RDB文件中，既可以手动执行，也可以根据配置定期执行。

## 10.1 RDB文件的创建与载入
SAVE和BGSAVE命令用于生成RDB文件。SAVE命令会阻塞服务器进程，直到RDB文件创建完毕为止。BGSAVE会派生出一个子进程负责创建RDB文件，服务器进程继续处理命令请求。创建RDB文件的工作由rdb.c/rdbSave函数完成，SAVE命令直接调用该函数，BDSAVE命令创建子进程后，在子进程中调用，函数执行完后向父进程发送信号，父进程负责轮询等待子进程的信号。\
RDB文件的载入工作是在服务器启动时自动进行的，Redis没有专门用于载入RDB文件的命令。由于AOF文件的更新频率比RDB文件高，因此如果AOF持久化功能开启，则服务器优先使用AOF文件还原数据库状态。

在BGSAVE执行期间，另外的SAVE和BGSAVE命令会被拒绝，防止多个进程同时调用rdbSave产生竞争。BGREWRITEAOF同样不能同时执行，如果BGSAVE正在执行，则BGREWRITEAOF命令会在BGSAVE结束后执行，如果BGREWRITEAOF正在执行，则BGSAVE会被拒绝，这两个命令不同是执行是出于性能考虑，减少同一时间的大量磁盘读写。

## 10.2 自动间隔性保存
Redis允许用户通过设置服务器配置的save选项，服务器可以每隔一段时间自动执行BGSAVE命令。例如设置save 900 1, save 300 10, save 60 10000，则只要满足900秒内有一次修改，或者300秒内有10次修改，或者60秒内10000次修改，则执行BGSAVE命令。

Redis服务器启动时用户通过配置文件或者启动参数设置save选项，如果没有主动设置则服务器将上述设置设为默认条件。服务器会根据相应的选项设置redisServer结构的saveparams属性.该属性为一个数组，每个元素为一个saveparam结构，记录save选项的参数。\
redisServer还维持一个dirty计数器和一个lastsave属性。dirty计数器记录距离上一次执行SAVE或者BGSAVE后服务器对数据库状态进行了多少次修改，lastsave是unix时间戳，记录了上一次成功执行SAVE或者BGSAVE的时间。当服务器成功执行数据库修改命令后dirty计数器进行更新，增加新修改的次数。
```C++
struct redisServer {
    // ...
    struct saveparam *saveparams;
    long long dirty;
    time_t lastsave;
    // ...
};

struct saveparam {
    time_t seconds;
    int changes;
};
```
Redis服务器的serverCron函数默认每100ms执行一次，其中一项工作即为检查save选项的条件是否满足，满足的话则执行BGSAVE命令。可以使用redisSever.lastsave计算上次保存之后过了多长时间，再看此时间是否大于saveparam.seconds并且dirty属性的值是否大于对应的saveparams.changes。BGSAVE完成后dirty会被设置为0，lastsave保存为BGSAVE执行完成的时间.

## 10.3 RDB文件结构
RDB文件最开头长度5字节，保存“REDIS”五个字符，可以用来让程序快速检测所载入的文件是否为RDB文件。接下来为db_version变量，长度4字节，是一个字符串表示的整数，记录了RDB文件的版本号。接下来为databases部分包含零个或者任意多个数据库以及各个数据库中的键值对数据，如果数据库为空则为0字节。接下来为EOF常量，长度1字节，标志着RDB文件正文内容的结束，意味着所有键值对都载入完毕了。最后是check_sum，为8字节无符号整数，保存一个校验和，通过对前面部分内容计算得出，服务器载入RDB文件时会计算校验和，与check_sum比较，检测RDB文件是否损坏。

### 10.3.1 database部分
每个非空数据库在RDB文件中包含一个SELECTDB常量，长度为1字节，接下来保存一个db_number变量，为数据库号码，根据号码大小不同可以为1字节，2字节或者5字节，程序读入db_number后服务器调用SELECT命令进行数据库切换。之后key_value_pairs保存数据库中所有键值对数据，包含过期时间。

### 10.3.2 key_value_pairs部分
不带过期时间的键值对由TYPE，key，value三部分组成。TYPE长度为1字节，记录了value的类型以及底层编码，程序可以根据TYPE的值来决定如何读入以及解释value的数据。key总是字符串对象。\
如果带有过期时间，则过期时间位于每个pair最前端，EXPIRETIME_MS常量长度为1字节，告知程序读入的将是一个毫秒为单位的过期时间，ms为8字节长的带符号整数，记录毫秒为单位的时间戳，代表该键值对的过期时间。

### 10.3.3 value的编码
对于字符串对象，可以用int或者raw编码。使用int编码的字符串对象首先包含一个ENCODING变量，表示用8位或16位或32位来保存整数，之后的integer变量保存对应的整数。\
如果以raw形式编码，则字符串长度小于等于20字节时字符串被原样保存，否则被压缩保存。没有压缩的字符串首先包含一个len，表示字符串长度，之后string表示原字符串。如果压缩过，则保存结构首先包含一个REDIS_RDB_ENC_LZF常量，表示字符串被LZF算法压缩，之后compressed_len记录字符串被压缩后的长度，origin_len记录源字符串长度，compressed_string记录被压缩的字符串。

列表对象保存结构首先通过list_length记录列表长度，之后记录每个列表项。每个列表项为一个字符串对象，程序以处理字符串的方式保存和读入列表项，即包含长度和字符串的值。

集合对象首先通过set_size记录集合大小，之后每个集合元素以字符串方式被读入。

哈希表对象首先保存hash_size作为哈希表大小，之后每个kv以字符串对象的形式挨在一起保存。

有序集合对象首先通过sorted_set_size记录有序集合的大小，每个元素分为score和member两部分，分别以字符串形式保存。

整数集合编码的集合对象被保存为一个字符串对象，读入时先读入字符串对象，再转换为整数集合。

压缩列表实现的列表、哈希表或者有序集合首先被转换为一个字符串对象，读入时先读入字符串对象并转换为原来的压缩列表，之后通过TYPE的值设置对应对象类型。

## 10.4 分析RDB文件
使用od命令分析Redis服务器产生的RDB文件。给定-c参数可以通过ASCII编码的方式打印文件，-x参数以十六进制方式打印文件。

# Chapter 11 AOF持久化
AOF持久化保存服务器执行的写命令。被写入AOF文件的所有命令都是以Redis命令请求协议格式（纯文本格式）保存的。AOF文件会自动添加SELECT命令用于指定数据库。

## 11.1 AOF持久化的实现
### 11.1.1 命令追加
服务器执行完写命令后会以协议格式将对应的写命令追加到服务器的aof_buf缓冲区末尾：
```C++
struct redisServer {
    // ...
    sds aof_buf;
    // ...
};
```

### 11.1.2 AOF文件的写入与同步
Redis服务器进程就是一个事件循环，文件事件负责接收客户端命令请求并响应，时间事件负责执行serverCron等需要定时运行的函数。每次一个事件循环结束前，服务器会调用flushAppendOnlyFile函数，考虑是否将aof_buf中的内容写入和保存到AOF文件中。服务器配置的appendfsync选项的值决定该函数的行为，值为always则将缓冲区所有内容写入并同步到AOF文件，everysec表示将缓冲区写入AOF文件，如果上次同步的时间距现在超过一秒钟，则再对其进行同步，此同步工作由一个线程专门负责，no则表示不对AOF文件进行同步，同步时间由操作系统决定。\
现代操作系统中用户调用write函数写入文件时，写入数据通常保存在内存缓冲区中，缓冲区被填满或者超过指定时限后才将缓冲区中的数据写入磁盘。系统提供了fsync和fdatasync两个函数强制让操作系统立即将缓冲区中的数据写入磁盘，保证数据不丢失。因此理论上always选项最慢但是最多丢失一个事件循环中的命令数据，everysec可能会丢失一秒钟的命令，no则是最快的选项，不过由于缓存会积累数据，单次同步时间最长。

## 11.2 AOF文件的载入与数据还原
Redis读取AOF文件后，创建一个伪客户端用于执行AOF文件保存的写命令，从AOF文件中分析并读取一条写命令并由该伪客户端执行。

## 11.3 AOF重写
由于实际写命令会频繁执行，AOF文件体积可能会膨胀，Redis因此提供了AOF文件重写功能，创建一个新的AOF文件替代现有的AOF文件，两个文件保存的数据库状态相同，但是不会包含浪费空间的冗余命令。

### 11.3.1 AOF文件重写的实现
AOF文件重写不需要对现有AOF文件读取分析或写入，而是直接去查看数据库的当前状态。对于每个键，如果过期则不考虑，否则将该键对应的值用一条写命令（SET，RPUSH，HMSET，SADD，ZADD）表示写入新的AOF文件中，最后如果有过期时间同样需要被写入一条PEXPIREAT命令。\
实际应用中根据redis的设置，为了避免缓冲区的溢出，重写程序可能会使用多条命令记录同一个键的值，例如当前版本中如果一个集合键包含超过64个元素，则重写为多条SADD命令。

### 11.3.2 AOF后台重写
由于AOF重写函数进行大量写入操作，调用这个函数的线程将被长时间阻塞，如果服务器直接调用该函数则无法处理客户端发来的请求。因此AOF重写程序放到子进程中执行，不影响父进程处理请求，同时子进程带有父进程的数据副本，在不使用锁的情况下也可保证安全。\
不过子进程重写过程中父进程处理的新命令同样会继续修改数据库，造成AOF保存的数据库状态与当前状态不一致，为此Redis服务器设置一个AOF重写缓冲区，Redis服务器执行完写命令后同时将写命令写入AOF缓冲区以及AOF重写缓冲区。子进程完成重写后向父进程发送信号，父进程接到该信号后会调用一个信号处理函数，将AOF重写缓冲区中的内容写入刚才子进程重写好的AOF文件，这时该AOF文件与当前服务器状态一直，之后父进程将该文件改名，原子地覆盖现有AOF文件完成替换。整个后台重写过程只有此信号处理函数阻塞父进程。以上即为BGREWRITEAOF的实现原理。

# Chapter 12 事件
Redis服务器为事件驱动程序，服务器处理文件事件和时间事件。文件事件中服务器通过套接字与客户端连接，文件事件是服务器对套接字操作的抽象，服务器与客户端的通信会产生文件事件，服务器通过监听并处理这些事件完成一系列网络通信操作。时间事件是对定时操作的抽象。

## 12.1 文件事件
Redis基于Reactor模式开发了网络事件处理器，被称为文件事件处理器，使用I/O多路复用同时监听多个套接字，根据套接字执行的任务关联不同的事件处理器。被监听的套接字准备好执行accept、read、write、close等操作时，相应的文件事件就会产生，文件事件处理器就会调用套接字关联好的事件处理器来处理事件。

### 12.1.1 文件事件处理器的构成
文件事件处理器分为四个部分，分别是套接字、I/O多路复用程序、文件事件分派器（dispatcher）以及事件处理器。由于服务器连接多个套接字，多个事件可能会并发出现，I/O多路复用程序将产生事件的套接字放入一个队列中，通过此队列以有序、同步、每次一个套接字的方式向分派器传送套接字。上一个套接字产生的事件被处理完后，多路复用程序才会向分派器传送下一个套接字。分派器接受套接字，根据套接字产生的时间类型，调用相应的事件处理器。

### 12.1.2 I/O多路复用程序的实现
Redis的I/O多路复用程序通过包装select、epoll、evport、kqueue等函数库来实现，每个函数库在Redis源码对应一个单独文件，由于API相同，所以底层实现可以互换，Redis在底层实现中通过宏定义自动选择系统中性能最高的多路复用函数库作为底层实现。

### 12.1.3 事件的类型
多路复用程序可以监听多个套接字的ae.h/AE_READABLE和ae.h/AE_WRITABLE事件。套接字可读时（客户端对套接字执行write或者close操作）或者新的acceptable套接字出现时，套接字产生AE_READABLE事件。套接字可写时（客户端对套接字执行read操作），套接字产生AE_WRITABLE事件。如果同时产生两种事件，则优先处理AE_READABLE事件。

### 12.1.4 API
ae.c/aeCreateFileEvent接受一个套接字描述符、一个事件类型以及一个事件处理器，将套接字的对应事件加入到多路复用程序的监听范围并与事件处理器关联。ae.c/aeDeleteFileEvent取消对应的监听。ae.c/aeGetFileEvents接受一个套接字描述符，返回套接字正在被监听的事件类型。ae.c/aeWait函数接受一个套接字描述符、一个事件类型和一个毫秒数为参数，阻塞并等待套接字给定类型事件产生。ae.c/aeApiPoll在指定时间内，阻塞并等待所有监听状态套接字文件事件。ae.c/aeProcessEvents是事件分派器，先调用aeApiPoll等待事件产生，遍历所有事件并调用相应的事件处理器。ae.c/aeGetApiName函数返回I/O多路复用程序使用的函数库名称。

### 12.1.5 文件事件的处理器
networking.c/acceptTcpHandler函数是Redis的连接应答处理器，对连接“监听套接字”的客户端进行应答（accept）。服务器初始化的时候该处理器和AE_READABLE关联，当有客户端连接的时候（connect）套接字产生AE_READABLE事件，引发处理器执行。\
networking.c/readQueryFromClient是命令请求处理器，负责从套接字中读入命令请求（read）。客户端成功连接后套接字的AE_READABLE事件与命令请求处理器关联。客户端发送命令请求就会产生对应事件。\
networking.c/sendReplyToClient是命令回复处理器，负责将服务器执行命令后得到的响应返回给客户端（write），套接字的AE_WRITABLE事件与其关联。发送完毕后服务器解除命令回复处理器与AE_WRITABLE事件的关联。

Redis服务器运行时，监听套接字的AE_READABLE事件处于监听下，该事件的处理器为连接应答处理器。客户端发起连接时，该事件被触发，连接应答处理器执行，应答后创建客户端套接字，将客户端套接字的AE_READABLE时间与请求处理器关联，使得客户端可以发送命令请求。客户端发送命令请求时，客户端套接字产生AE_READABLE事件，引发命令请求处理器执行。客户端套接字的AE_WRITABLE事件与命令回复处理器关联，客户端读取命令回复的时候套接字产生AE_WRITABLE事件，触发命令回复处理器执行。命令回复全部写入后服务器解除客户端套接字的AE_WRITABLE事件与命令回复处理器关联。

## 12.2 时间事件
时间事件分为定时事件，即当前时间特定时间后执行，与周期性事件。一个时间事件有一个全局唯一id，以及一个when时间戳，记录时间事件的到达时间，以及一个时间事件处理器timeProc。如果时间事件处理器返回ae.h/AE_NOMORE则该事件为定时事件，否则为周期性事件，服务器根据时间事件处理器返回的值对when属性进行更新，使得事件周期性到达。

### 12.2.1 实现
所有时间事件放在无序链表中，时间事件执行器运行时遍历整个链表，查找已到达的时间事件，调用相应的处理器。无序指不按when属性排序。正常模式下只有serverCron一个时间事件，benchmark模式下也只有两个，这时链表几乎退化为指针，因此不影响性能。

### 12.2.2 API
ae.c/aeCreateTimeEvent接受一个时间与事件处理器，将新的时间事件添加到服务器。ae.c/DeleteFileEvent删除对应id的时间事件。ae.c/SearchNearestTimer返回距离当前时间最接近的时间事件。ae.c/processTimeEvents遍历所有已到达的时间事件并调用对应的事件处理器，如果是周期性事件还要更新when属性。

### 12.2.3 时间事件应用实例：serverCron函数
serverCron函数负责更新服务器各类统计信息，清理过期键值对，关闭清理链接失效的客户端，AOF或RDB持久化，主服务器定期同步。该函数以周期性事件运行。

## 12.3 事件的调度与执行
ae.c/aeProcessEvents负责事件调度与执行。该函数首先获取最接近的时间事件（SearchNearestTimer），设置剩余到达时间，如果为负数（即事件已经到达）则设为0。之后调用aeApiPoll并传入剩余到达时间作为最大阻塞时间（既可以保证aeApiPoll不会阻塞太长时间，又避免服务器对时间事件频繁轮询），之后处理文件事件与已到达的时间事件（processTimeEvents）。每个事件循环结束还需要判断是否将AOF缓冲区的数据写入AOF文件（不是同步）。aeProcessEvents置于一个循环中，加上初始化函数与清理函数，即构成Redis服务器主函数。\
如果文件事件处理完后仍未有时间事件到达，服务器将再次等待并处理文件事件。服务器不会中断事件处理，也不会抢占，因此事件处理器会尽可能减少阻塞时间，有需要时主动让出执行权，降低事件饥饿的可能性，例如命令回复处理器将回复写入客户端套接字时，如果写入字节过多，则break跳出循环，下次再写。时间事件的持久化操作也会放大子进程执行。时间事件的处理会比when中设置的到达时间较晚一些。

# Chapter 13 客户端
每个客户端对应一个redis.h/redisClient结构，保存了客户端当前的状态信息。服务器中保存了一个链表，保存了所有与服务器连接的客户端的状态结构：
```C++
struct redisServer {
    // ...
    list *clients;
    // ...
};
```

## 13.1 客户端属性
客户端包含的属性分为两类，一类是比较通用的属性，很少与特定功能相关，另一类与特定功能相关，例如db属性和dictid属性等等。
```C++
typedef struct redisClient {
    // ...
    int fd;
    robj* name;
    int flags;

    sds querybuf;
    robj** argv;
    int argc;
    struct redisCommand *cmd;

    char buf[REDIS_REPLY_CHUNK_BYTES];
    int bufpos;
    list* reply;

    int authenticated;

    time_t ctime, lastinteraction, obuf_soft_limit_reached_time;

    // ...
} redisClient;
```
fd为套接字描述符。伪客户端的fd属性为-1，命令请求来源于AOF文件或者Lua脚本，并不是网络，因此不需要套接字连接。普通客户端属性大于-1的整数。\
可以使用CLIENT setname为客户端设置一个名字。如果没有设置name属性指向NULL。\
flags标志属性记录了客户端角色以及所处状态。主从复制时主服务器与从服务器互为客户端，flag分别为REDIS_MASTER和REDIS_SLAVE。PUBSUB和SCRIPT LOAD命令尽管没有修改数据库，但是可能修改客户端或者服务器状态，因此需要通过REDIS_FORCE_AOF强制将命令写入AOF文件。\
querybuf属性作为输入缓冲区保存客户端发送的命令请求。缓冲区大小随内容动态缩小或者扩大。\
服务器对缓冲区中的命令解析后得出命令参数以及个数，保存于argv和argc属性中。argv是一个数组，每个项是一个字符串对象，argv[0]是要执行的命令，后面是参数，argc是数组长度。\
服务器会根据argv[0]的值在命令表中查找对应的实现函数，一个命令表是一个字典，键为SDS结构，保存命令名字，值是对应的redisCommand结构，保存了实现函数，命令标志，参数个数，总执行次数，总消耗时长等统计信息。客户端的cmd属性会指向对应的结构，服务器使用该结构以及参数执行命令。命令表查找不区分大小写。\
命令回复写到输出缓冲区中，每个客户端有两个输出缓冲区，一个大小固定另一个大小可变。固定大小缓冲区保存长度小的回复，另一个保存长度大的。buf属性对应固定大小缓冲区，bufpos表示当前buf使用的字节数量。可变大小缓冲区由reply链表表示，由多个字符串对象组成。\
authenticated属性记录客户端是否通过身份验证，0表示未通过，如果未通过则除了AUTH命令其他命令服务器都会拒绝执行。AUTH命令成功验证后该属性变为1。该属性仅在服务器开启身份验证功能时使用。\
ctime记录客户端创建时间，用来计算与服务器连接了多少秒。lastinteraction属性记录了客户端和服务器最后一次互动（发送命令或者发送回复）的时间，可以用来计算客户端空转时间。obuf_soft_limit_reached_time记录输出缓冲区第一次到达soft limit的时间。

## 13.2 客户端的创建与关闭
对于普通客户端，使用connect函数链接到服务器时，服务器调用连接时间处理器创建相应的客户端状态，并添加到clients链表的末尾。\
如果客户端进程退出或者被kill，客户端和服务器连接关闭，或者客户端发送了不符合协议格式的命令请求，或者作为CLIENT KILL命令的对象，或者空转时间超过设置的timeout选项（除非客户端为主服务器且从服务器被阻塞（REDIS_BLOCKED）或者执行SUBSCRIBE或PSUBSECRIBE命令），或者命令请求大小超过输入缓冲区限制大小（默认1GB），或者命令回复大小超过输出缓冲区大小，则客户端会被服务器关闭。服务器使用硬性限制或者软性限制来检查输出缓冲区大小，如果是硬性限制，则超过缓冲区大小直接关闭，如果是软性限制，则通过obuf_soft_limit_reached_time记录下客户端到达软性限制的起始时间，如果缓冲区大小超过软性限制的时间超过设定时长则客户端被关闭，如果指定时间没超出则客户端不会被关闭，obuf_soft_limit_reached_time属性的值被清零。可以同时设置硬性限制以及软性限制。\
执行Lua脚本的伪客户端会被关联到服务器的lua_client属性中，直到服务器关闭才会被关闭:
```C++
struct redisServer {
    // ...
    redisClient* lua_client;
    // ...
};
```
执行AOF文件的伪客户端会在AOF载入完成后被关闭。

# Chapter 14 服务器
## 14.1 命令请求的执行过程
客户端将命令转换成协议格式，连接到服务器套接字，将协议格式的命令发送给服务器。服务器调用命令请求处理器，读取命令请求，保存至客户端状态的输入缓冲区。之后对输入缓冲区中的命令进行分析，提取命令参数以及参数个数，保存到argc和argv属性中。之后调用命令执行器，执行命令。\
命令执行器根据客户端argv[0]参数在命令表中查找指定命令，并保存到客户端状态cmd属性中。cmd属性为redisCommand结构，包括命令名称，命令实现函数的指针proc，命令参数个数（如果为负数则表明参数数量大于等于该负数的绝对值），命令的属性（读命令、写命令、命令会占用大量内存等等），服务器执行该命令的次数、执行命令耗费的总时长。\
之后命令执行器执行一些预备操作。检查cmd指针是否为NULL，如果为NULL则返回错误表明用户输入的指令找不到对应实现。根据cmd的参数个数属性检查输入参数是否正确。检查客户端是否通过验证。如果打开maxmemory功能则检查服务器内存占用情况，有需要时进行内存回收，如果内存回收失败则向客户端返回错误。如果上一次执行BGSAVE出错且stop-writes-on-bgsave-error功能打开，且即将执行的命令为写命令，则返回错误。如果客户端正在用SUBSCRIBE订阅频道或者PSUBSCRIBE命令订阅模式，则服务器只会执行SUBSCRIBE、PSUBSCRIBE、UNSUBSCRIBE、PUNSUBSCRIBE命令。如果服务器正在数据载入，那么命令标志必须带有l标志（例如INFO、SHUTDOWN、PUBLISH等），否则被拒绝。如果客户端正在执行事务，则服务器只会执行EXEC、DISCARD、MULTI、WATCH命令，其他命令都会放入事务队列。如果服务器打开监视器功能则即将执行的命令信息会被发送给监视器。复制或集群模式下还有更多预备操作。\
之后调用client->cmd->proc(client)执行命令实现函数。该函数产生命令回复，保存在输出缓冲区中，并为客户端套接字关联命令回复处理器，将命令回复返回给客户端。\
之后服务器执行一些后续工作。如果服务器开启慢查询日志，则慢查询日志模块会检查是否需要为刚执行的命令请求添加一条新的日志。更新被执行命令的redisCommand的总时长属性以及执行次数属性。AOF持久化模块会将刚才执行的命令写入AOF缓冲区。如果从服务器正在复制当前服务器，则刚才执行的命令被传播给所有服务器。\
当客户端套接字可写时，命令回复处理器将输出缓冲区中的命令回复发送给客户端，发送完毕后命令回复处理器清空输出缓冲区。\
客户端收到协议格式的命令回复后，将回复转换成人类可读的格式打印。

## 14.2 serverCron函数
serverCron函数默认每隔100ms执行一次。该函数负责管理服务器资源，保持服务器自身良好运转。\
```C++
struct redisServer {
    // ...
    time_t unixtime;
    long long mstime;

    unsigned lruclock:22;
 
    long long ops_sec_last_sample_time; // 上一次抽样时间
    long long ops_sec_last_sample_ops; // 上一次抽样时服务器已经执行的命令数量
    long long ops_sec_samples[REDIS_OPES_SEC_SAMPLES]; // 环形数组记录之前的抽样结果
    int ops_sec_idx; // 数组索引

    size_t stat_peak_memory;

    int aof_rewrite_scheduled;

    pid_t rdb_child_pid;
    pid_t aof_child_pid;

    int cronloops
};
```
为了减少通过系统调用获取当前时间的次数，redisServer结构的unixtime和mstime属性被用作时间缓存。serverCron更新该缓存，不过由于100ms一次，精确度不高，只用于打印日志、计算上线时间等功能。对于过期时间、慢查询日志等服务器还是会执行系统调用。\
lruclock属性保存了LRU时钟，也是时间缓存的一种，服务器计算键的空转时间的时候用lruclock属性减去对象的lru属性。serverCron函数每10s对lruclock属性更新。\
sreverCron函数中的trackOperationsPerSecond函数每100ms估算服务器在最近一秒钟处理的命令请求数量，可以用INFO stats查看。根据当前已经执行的命令数量以及上次抽样时执行的命令数量的差值以及时间差值可以计算两次之间执行了多少命令，从而估算平均每毫秒的命令个数。该估计值被放到环形数组中。\
每次serverCron函数会查看当前内存用量是否大于内存峰值，是的话记录到stat_peak_memory中，可以用INFO memory查看。\
serverCron函数会对shutdown_asap属性检查，该属性会在服务器接到SIGTERM信号时打开。如果打开则关闭服务器，关闭前会进行RDB持久化操作。\
serverCron会执行clientsCron函数，对一定数量的客户端检查，如果连接超时则释放客户端，如果输入缓冲区大小过长则释放当前输入缓冲区并重新创建一个默认大小的缓冲区。\
serverCron会执行databasesCron函数，对一部分数据库检查，删除过期键，对字典进行收缩操作等等（第9章）。\
BGSAVE执行期间BGREWRITEAOF命令被延迟，aof_rewrite_scheduled记录AOF重写是否被延迟。每次serverCron哈数会检查BGSAVE和BGREWRITEAOF是否正在执行，如果都没有且aof_rewrite_scheduled为1则执行被延迟的BGREWRITEAOF命令。、
如果执行BGSAVE的子进程id或者执行BGREWRITEAOF命令的子进程id不为-1，则serverCron执行wait3函数，检查子进程是否有信号到达。如果有则表明RDB文件生成完毕或者AOF重写完毕，则服务器需要执行对应的后续操作，例如替换RDB文件或者AOF文件。如果两个子进程id都为-1，则程序查看是否有AOF重写被延迟，如果有则开始新的重写操作，之后检查服务器自动保存条件是否满足，如果满足且服务器没有进行其他持久化操作，则服务器开始新的BGSAVE操作，之后检查AOF重写条件是否满足，如果满足且没有其他持久化操作则开始新的BGREWRITEAOF操作。\
如果AOF缓冲区有待写入的数据，serverCron函数会负责将缓冲区中的内容写到AOF文件中（第11章）。\
serverCron函数会关闭输出缓冲区超出限制的客户端（第13章）。\
每次serverCron函数执行，cronloops属性自增1。复制模块中“serverCron函数每执行N次就执行指定代码”的逻辑需要该属性。

## 14.3 初始化服务器
初始化服务器首先创建一个redisServer类型的实例，该初始化由redis.c/initServerConfig函数完成，该函数设置服务器运行ID、默认运行频率、默认配置文件路径、运行架构、端口号、持久化条件、LRU时钟、命令表等。\
用户可以通过给定配置参数或者指定配置文件来修改默认配置属性。服务器会对redisServer结构进行更新。\
initServer函数会初始化clients链表、db数组、pubsub_channels用于保存频道订阅信息、pubsub_patterns用于保存模式订阅信息、Lua环境属性lua、慢查询日志slowlog。这些数据结构需要之前的配置选项才能初始化。此外initServer还设置进程信号处理器、创建共享对象（比如经常用到的字符串、包含1-10000整数的字符串等）、打开服务器监听端口、为监听套接字关联连接事件处理器、为serverCron函数创建时间事件、打开或者创建AOF文件、初始化后台I/O模块。\
如果服务器启用AOF持久化功能则使用AOF还原数据库状态，否则使用RDB文件还原。\
之后服务器开始执行事件循环，开始处理客户端请求。

# Chapter 15 复制
用户可以通过执行SLAVEOF命令或者设置slaveof选项让一个服务器复制另一个服务器，被复制的服务器为主服务器，复制到的服务器为从服务器。主从服务器保存相同的数据。

## 15.1 旧版复制功能的实现
Redis复制功能分为同步和命令传播两个操作。同步操作将从服务器更新至主服务器当前状态。命令传播操作用于主服务器状态被修改时，使得主从服务器状态一致。

### 15.1.1 同步
从服务器向主服务器发送SYNC命令，收到SYNC的主服务器执行BGSAVE命令后台生成RDB文件，并使用缓冲区记录之后执行的所有写命令。BGSAVE执行完毕后RDB文件发送给从服务器，从服务器载入RDB文件更新自己的数据库状态。主服务器再将缓冲区的写命令发送给从服务器，从服务器执行对应的命令更新至主服务器当前状态。

### 15.1.2 命令传播
同步操作完成后主服务器可能再被修改，主服务器会将之后的写命令发送给从服务器执行。

## 15.2 旧版复制功能的缺陷
如果主从服务器因为网络原因中断了复制，重新连接后重新复制的效率很低，因为本来只需要更新断线时主服务器的更改，但是实际上RDB却是复制整个数据库状态。

SYNC命令非常消耗资源。RDB文件的生成耗费大量CPU内存和I/O资源。RDB文件的发送耗费大量网络资源，对主服务器响应请求的时间产生影响。从服务器载入RDB文件期间被阻塞无法处理请求。

## 15.3 新版复制功能的实现
从2.8版本开始Redis使用PSYNC命令代替SYNC执行同步操作。PSYNC具有完整重同步和部分重同步两种模式，完整重同步处理初次复制的情况，与SYNC命令基本一样。部分重同步处理断线后重复制的情况，重新连接后主服务器只把断开连接期间的写命令发送给从服务器。连接重新建立后主服务器向从服务器返回+CONTINUE回复，表示执行部分重同步。

## 15.4 部分重同步的实现
### 15.4.1 复制偏移量
主从服务器分别维护一个复制偏移量。每次主服务器向从服务器传送N个字节数据就给自己的复制偏移量加N。从服务器每收到N字节就给自己的复制偏移量加N。如果主从一致则复制偏移量相同。如果断线重连，从服务器通过PSYNC命令报告自己的复制偏移量与主服务器不同。

### 15.4.2 复制积压缓冲区
复制积压缓冲区是主服务器维护的固定长度的FIFO队列。主服务器进行命令传播时同时会将写命令放到复制积压缓冲区中，这样该缓冲区保存最近传播的一部分写命令，并且为队列中的每个字节记录相应的复制偏移量。如果从服务器发来的复制偏移量之后的数据仍然在复制积压缓冲区中，则主服务器执行部分重同步，否则执行完整重同步。\
缓冲区默认大小为1MB，如果主服务器执行大量写命令或者重连接所需的时间很长，可以加长缓冲区大小，安全起见可以设置大小为2倍的平均重连时间乘以每秒写命令数量。

### 15.4.3 服务器运行ID
每个服务器都有自己的运行ID，服务器启动时自动生成。从服务器对主服务器进行初次复制时，主服务器将自己的运行ID传送给从服务器。断线重连后，从服务器将当前连接的主服务器发送之前保存的运行ID，如果与当前主服务器ID相同，则说明断线前复制的是当前连接的主服务器，可进行部分重同步。否则说明不是之前的主服务器，执行完整重同步。

## 15.5 PSYNC命令的实现
如果从服务器没有复制过任何主服务器或者执行过SLAVEOF no one命令，则从服务器开始复制时向主服务器发送PSYNC ? -1命令，主动请求进行完整重同步。否则从服务器可以发送PSYNC \<runid> \<offset>命令，runid为上次主服务器的ID，offset为当前复制偏移量。\
主服务器可以返回+FULLRESYNC \<runid> \<offset>则表示执行完整重同步，runid为该主服务器的ID，offset为主服务器当前复制偏移量。如果主服务器返回+CONTINUE，则表明执行部分重同步。如果返回-ERR则表明主服务器版本低于2.8，无法识别PSYNC命令。

## 15.6 复制的实现
客户端向从服务器发送SLAVEOF \<ip> \<port>命令时，从服务器首先将ip地址和端口保存到redisServer的masterhost和masterport属性中。设置完成后向客户端返回OK，实际复制工作在OK之后执行。
```C++
struct redisServer {
    // ...
    char *masterhost;
    int masterport;
    // ...
};
```
从服务器根据ip地址和端口创建向主服务器的套接字连接，成功连接后从服务器为套接字关联一个专门处理复制工作的文件事件处理器，用于处理接收RDB文件、写命令等等。主服务器接受从服务器的链接后，将为该套接字创建客户端状态，将从服务器作为客户端看待。

从服务器成为客户端后首先发送PING命令，检查套接字读写状态正常以及主服务器可以正常处理命令请求。如果从服务器在规定时间内不能读取主服务器回复的内容，则断开重新连接主服务器。如果主服务器返回错误则表明主服务器暂时不能执行复制，则从服务器断开重连。如果从服务器读取到"PONG"回复则表明连接正常。

如果从服务器设置masterauth选项则进行身份验证，向主服务器发送AUTH命令，参数为masterauth的值。如果主服务器没有设置requirepass选项且从服务器没有设置masterauth选项，则不需要身份验证。如果AUTH发送的密码和主服务器requirepass选项设置的一致，则主服务器继续执行从服务器的命令，否则返回invalid password错误。如果主服务器设置requirepass但是从服务器没有设置masterauth则主服务器返回NOAUTH错误，反之返回一个no password is set错误。

身份验证成功后从服务器执行REPLCONF listening-port \<port-number>，向主服务器发送从服务器的监听端口号，主服务器会将该端口号记录到对应客户端redisClient属性的slave_listening_port属性中。该属性唯一作用是执行INFO replication时打印从服务器端口号。
```C++
typedef struct redisClient {
    // ...
    int slave_listening_port;
    // ...
} redisClient;
```

之后从服务器发送PSYNC命令，主服务器也会成为从服务器的客户端，用于发送缓冲区中的写命令（完整重同步）或者复制积压缓冲区中的写命令（部分重同步）。完成同步后进入命令传播阶段，主服务器将自己执行的写命令发送给服务器，从服务器接收写命令。

## 15.7 心跳检测
命令传播阶段，从服务器默认每秒一次向主服务器发送REPLCONF ACK \<replication_offset>，其中replication_offset为当前从服务器复制偏移量，用于检测网络连接状态，实现min-slaves选项以及检测命令丢失。如果主服务器超过一秒没有收到REPLCONF ACK命令则表明网络连接有问题。\
Redis的min-slaves-to-write和min-slaves-max-lag选项防止主服务器在不安全的情况下执行写命令。如果从服务器数量少于min-slaves-to-write或者每个从服务器延迟大于min-slaves-max-lag时，主服务器将拒绝执行写命令。\
如果主服务器发现REPLCONF ACK传来的复制偏移量小于自己的复制偏移量，则表明写命令可能半路丢失，主服务器根据对应复制偏移量在复制积压缓冲区中找到从服务器缺少的数据，重新发送给从服务器。与部分重同步不同，这一补发缺失的写命令操作在没有断线的情况下执行。

# Chapter 16 Sentinel
由一个或多个Sentinel实例组成的Sentinel系统监视任意多个主服务器以及对应的从服务器，当被监视的主服务器下线时，自动将某个从服务器升级为新的主服务器。\
主服务器下线时长超过用户设定的上限时，Sentinel系统挑选其中的一个从服务器升级为主服务器。之后Sentinel系统向所有从服务器发送新的复制指令，使它们成为新的主服务器的从服务器。所有从服务器开始复制新的主服务器时故障转移操作执行完毕。当下线的原主服务器上线时Sentinel将其设置为新的主服务器的从服务器。

## 16.1 启动并初始化Sentinel
使用redis-sentinel *sentinel.conf*或者redis-server *sentinel.conf* --sentinel启动sentinel。\
Sentinel本质上是一个特殊模式的服务器，因此启动Sentinel首先初始化一个普通服务器，具体步骤与第14章类似。不过初始化Sentinel不需要载入RDB或者AOF文件。Sentinel模式下服务器不使用持久化命令、数据库键值对方面的命令（SET、DEL、FLUSHDB）、事务命令等。复制命令（SLAVEOF）、PUBLISH命令只能在Sentinel内部使用。\
之后将一部分普通服务器使用的代码替换成Sentinel专用代码，例如服务器端口、命令表（sentinel.c/sentinelcmds），并且INFO命令的实现也不同。上述不使用的命令是因为命令表中没有载入对应的命令。PING、SENTINEL、INFO、SUBSCRIBE、UNSUBSCRIBE、PSUBSCRIBE、PUNSUBSCRIBE是客户端可以对Sentinel执行的全部命令。\
接下来服务器初始化一个sentinel.c/sentinalState结构，保存服务器中所有和Sentinel功能有关的状态（其余一般状态仍由redisServer结构保存）。
```C++
struct sentinelState {
    uint64_t current_epoch; // 用于故障转移

    // 所有被Sentinel监视的主服务器，键为主服务器的名字，值为指向sentinelRedisInstance指针
    dict* masters;

    int tilt; // 是否进入TILT模式

    int running_scripts; // 正在执行的脚本数量

    mstime_t tilt_start_time; // 进入TILT模式的时间

    mstime_t previous_time; // 最后一次执行时间处理器的时间

    list *scripts_queue; // FIFO队列，包含要执行的用户脚本
} sentinel;

typedef struct sentinelRedisInstance {
    int flags; // 记录实例类型及当前状态

    char *name; // 实例名字，主服务器名字由用户在配置文件设置，从服务器以及Sentinel名字由Sentinel自动设置，格式为ip:port

    char *runid; // 运行id

    uint64_t config_epoch; // 用于故障转移

    sentinelAddr *addr; // 实例的地址
    
    mstime_t down_after_period; // 无响应多久后被判断为主观下线

    int quorum; // 判断该实例客观下线所需支持投票数量

    int parallel_syncs; // 故障转移时同时对新的主服务器同步的从服务器数量

    mstime_t failover_timeout; // 刷新故障转移状态的最大时限

    // ...
} sentinelRedisInstance;

typedef struct sentinelAddr {
    char *ip;
    int port;
} sentinelAddr;
```
Sentinel的初始化引发对masters字典的初始化，根据被载入的Sentinel配置文件进行。\
最后还要创建向被监视的主服务器的链接，Sentinel为主服务器的客户端，可以向主服务器发送命令，从命令回复中获取信息。Sentinel向每个被监视的服务器创建两个异步网络连接，命令连接专门用于向主服务器发送命令，订阅连接订阅主服务器的__sentinel__:hello频道。目前的发布订阅功能中，被发送的信息不保存在Redis服务器中，如果信息发送时接收信息的客户端不在线或者断线，则信息丢失，因此Sentinel专门用一个连接接收该频道的消息。由于需要多个连接，因此创建异步连接。

## 16.2 获取主服务器信息
Sentinel每十秒向被监视的主服务器发送INFO命令，通过回复获取主服务器当前信息，包括主服务器本身的信息，如运行id等，以及主服务器的从服务器信息，包括从服务器的ip地址和端口号，从而使得Sentinel可以自动发现从服务器。主服务器重启后运行id变化，Sentinel可以通过INFO的返回进行相应的更新。\
从服务器信息用于更新主服务器sentinelRedisInstance结构的slaves字典，该字典的键为自动设置的从服务器名字，值是从服务器对应的sentinelRedisInstance结构。主服务器对应结构的flags属性为SRI_MASTER，从服务器为SRI_SLAVE。

## 16.3 获取从服务器信息
Sentinel在获取到新的从服务器信息后除了会新建相应的sentinelRedisInstance结构外，还会创建从服务器的命令连接和订阅连接，并每十秒通过命令连接向从服务器发送INFO命令，得到从服务器的运行ID、角色、主服务器的ip地址和端口号、主从服务器连接状态、从服务器的优先级、复制偏移量等，这些同样保存于从服务器的sentinelRedisInstance结构中。

## 16.4 向主服务器和从服务器发送信息
默认情况下Sentinel每两秒一次向主服务器和从服务器发送PUBLISH命令，向服务器的__sentinel__:hello频道发送信息，包含Sentinel自己的ip地址、端口号。运行ID、当前的配置epoch，以及主服务器的信息（如果Sentinel监视从服务器，则是当前从服务器正在复制的主服务器的信息）。

## 16.5 接收来自主服务器和从服务器的频道信息
Sentinel与服务器建立订阅连接后Sentinel向服务器发送SUBSCRIBE \_\_sentinel__:hello命令，订阅服务器的__sentinel__:hello频道，直到连接断开。对于监视同一个服务器的多个Sentinel，一个Sentinel发送的信息会被其他Sentinel接收到，用于更新其他Sentinel对发送信息Sentinel的认知。当一个Sentinel从该频道中收到信息时，Sentinel会分析信息中包含的Sentinel信息，如果是自己发送的则丢弃，否则说明是其他Sentinel发来的，接收该信息的Sentinel可以根据信息更新主服务器的实例结构。

### 16.5.1 更新sentinels字典
主服务器的sentinelRedisInstance中包含sentinels字典，包含所有监视这个主服务器的Sentinel信息。字典的键为Sentinel的名字，格式为ip:port，值为Sentinel对应的sentinelRedisInstance结构。\
当一个Sentinel接收到其他Sentinel发来的信息时，接收信息的Sentin得到其他Sentinel的ip地址、端口号、运行ID、配置epoch以及正在监视的主服务器的参数（ip地址、端口号、运行ID、配置epoch）。之后Sentinel在masters字典中找到对应的主服务器，查看sentinels字典中有没有发送此信息的Sentinel信息，如果有则更新，否则创建新的对应该Sentinel的实例。这样监视同一个主服务器的多个Sentinel可以通过发送频道信息互相发现对方。

### 16.5.2 创建连向其他Sentinel的命令连接
当Sentinel通过频道信息发现新的Sentinel时，它还会创建一个连向新的Sentinel的命令连接，这样各个Sentinel之间可以通过向其他Sentinel发送命令请求进行信息交换。Sentinel之间不创建订阅连接。

## 16.6 检测主观下线状态
默认情况下Sentinel每秒一次向所有创建了命令连接的实例（主服务器、从服务器、其他Sentinel）发送PING命令，判断其他实例是否在线。实例返回+PONG、-LOADING、-MASTERDOWN为有效回复，其他是无效回复。如果一个实例在down_after_period毫秒内一直返回无效回复，则Sentinel会将对应实例的flags打开SRI_S_DOWN，表示实例进入主观下线状态。down_after_period由用户设置的down-after-milliseconds决定，该值决定所有相关实例的主观下线标准，多个Sentinel设置的主观下线时长可能不同。

## 16.7 检查客观下线状态
Sentinel将一个主服务器判断为主观下线之后，向其他Sentinel询问，看它们是否也认为主服务器进入下线状态，当Sentinel收到足够数量的下线判断后会将服务器判断为客观下线，进行故障转移操作。\
Sentinel向其他Sentinel发送SENTINEL is-master-down-by-addr \<ip> \<port> \<current_epoch> \<runid>命令，参数为被Sentinel判断为主观下线的主服务器的地址端口号以及Sentinel当前的配置epoch。runid可以为星号，表示命令仅用于检测主服务器客观下线的状态，或者该Sentinel的运行ID，用于选举领头Sentinel。\
一个Sentinel接收到上述命令时，会检查对应的主服务器是否下线，返回检查结果down_state、局部领头Sentinel运行ID leader_runid（为星号则表示仅用于检查主服务器下线状态、以及局部领头Sentinel的配置epoch，用于选举领头Sentinel，如果leader_runid为星号，则epoch为0。\
发送命令的Sentinel接收上述回复，统计其他Sentinel认为主服务器下线的数量，达到客观下线所需数量（quorum参数，不同Sentinel可能不同）时将主服务器flags属性的SRI_O_DOWN标志打开，表示主服务器进入客观下线状态。

## 16.8 选举领头Sentinel
主服务器被判断为客观下线时，监视该主服务器的各个Sentinel进行协商，选举领头Sentinel执行故障转移操作。\
所有在线的Sentinel都有成为领头的资格，每次选举之后所有Sentinel的epoch会自增。在一个epoch里，所有Sentinel都有一次将某个Sentinel设为局部领头的机会，局部领头一旦设置在这个epoch中不能更改。每个发现主服务器客观下线的Sentinel都会要求其他Sentinel将自己设为局部领头Sentinel，这是通过在SENTINEL is-master-down-by-addr命令中的指定自己的runid作为参数实现的。局部领头Sentinel先到先得，最先发送该命令要求的Sentinel成为局部领头Sentinel。发送命令的Sentinel收到回复后查看leader_epoch参数是否与自己的一致，一致的话再对比leader_runid参数的值，如果一致则表明自己的局部领头Sentinel。每个Sentinel统计自己被多少Sentinel选举为局部领头Sentinel，如果有Sentinel被半数以上的Sentinel设置为局部领头Sentinel，则它成为领头Sentinel。如果没有产生领头Sentinel则一段时间后重新选举。

## 16.9 故障转移
故障转移操作首先挑选一个状态良好数据完整的从服务器，发送SLAVEOF no one命令，将其转换为主服务器。\
领头Sentinel将下线主服务器的所有从服务器保存到列表中，删除所有下线或者断线的从服务器（保证剩余从服务器在线），删除五秒内没有回复过INFO命令的从服务器（保证通信正常），删除所有与已经下线的主服务器断开连接超过down-after-milliseconds * 10的从服务器（保证剩余从服务器没有过早与主服务器断开连接，表明其数据是较新的），之后将从服务器按照优先级排序，选出优先级最高的服务器，有多个优先级相同时选取复制偏移量最大的服务器（表明数据最新），如果仍相同则选运行id小的。\
发送SLAVE no one命令后领头Sentinel每秒发送一次INFO查看从服务器的角色信息，变为master说明从服务器升级为主服务器。\
之后领头Sentinel向其他从服务器发送SLAVEOF命令，指定新升级得到的主服务器的ip地址和端口号。\
最后将下线的主服务器设置为新主服务器的从服务器。原来的主服务器重新上线时Sentinel向它发送SLAVEOF命令。

# Chapter 17 集群
Redis集群是Redis提供的分布式数据库方案，通过分片（sharding）金慈宁宫数据共享，提供复制和故障转移功能。

## 17.1 节点
一个Redis集群通常由多个节点（node）组成，使用CLUSTER MEET \<ip> \<port>命令与指定的节点握手并将该节点加入自己所在的集群中。

### 17.1.1 启动节点
一个节点是一个运行在集群模式下的Redis服务器，服务器启动时根据cluster-enabled配置选项决定是否开启集群模式。一个节点会继续使用单机模式下的服务器组件，例如文件事件处理器、数据库键值对、持久化工作、复制工作、发布订阅、时间事件处理器（serverCron函数会调用集群模式特有的clusterCron函数）。集群特有的数据保存在cluster.h/clusterNode、cluster.h/clusterLink、cluster.h/clusterState结构中。

### 17.1.2 集群数据结构
```C++
struct clusterNode {
    mstime_t time; // 节点创建时间
    char name[REDIS_CLUSTER_NAMELEN]; // 节点名字（40个十六进制字符组成的id）

    int flags; // 节点角色、状态等

    uint64_t configEpoch; // 用于故障转移

    char ip[REDIS_IP_STR_LEN];
    int port;

    clusterLink *link;

    // ...
};

typedef struct clusterLink {
    mstime_t ctime; // 连接创建时间

    int fd; // 套接字描述符

    sds sndbuf; // 输出缓冲区，保存等待发送给其他节点的消息

    sds rcvbuf; // 输入缓冲区，保存从其他节点接收到的消息

    struct clusterNode* node; // 与此连接相关的节点信息
} clusterLink;

// 每个节点保存一个clusterState结构，记录当前节点视角下的集群状态
typedef struct clusterState {
    clusterNode *myself; // 当前节点

    uint64_t currentEpoch; // 集群当前的epoch

    int state; // 集群当前状态（在线或下线）

    int size; // 集群中至少处理着一个槽的节点数量

    dict *nodes; // 集群节点名单，键为节点名字，值为节点对应的clusterNode结构

    // ...
} clusterState;
```

### 17.1.3 CLUSTER MEET命令的实现
客户端向节点A发送CLUSTER MEET命令，指定节点B的ip地址和端口号，将B添加进A的集群中。A首先和B握手，确认彼此存在。A为B创建一个clusterNode结构，添加到自己的clusterState.nodes字典中。之后A向B发送一条MEET消息，B收到后也会为A创建一个clusterNode结构，添加到自己的nodes字典中。之后B向A返回一条PONG消息。之后A向B返回一条PING消息，表明整个握手完成。最后A会将B的信息通过Gossip协议传播给集群中的其他节点，使得其他节点也和B握手。

## 17.2 槽指派
Redis集群通过分片方式保存数据库键值对，整个数据库被分为16384个槽（slot），每个键都属于其中一个槽，一个节点处理0-16384个槽。当每个槽都有节点处理时集群处于上线状态。否则处于下线状态。\
通过向节点发送CLUSTER ADDSLOTS命令可以将槽指派给节点负责。所有槽都被指派时集群进入上线状态。

### 17.2.1 记录节点的槽指派信息
```C++
struct clusterNode {
    // ...
    unsigned char slots[16384/8];
    int numslots;
    // ...
};
```
slots是二进制数组，包含16384位，位为1则表明节点处理对应的槽。因此指派以及检查是否指派的操作都可以O(1)时间完成。numslots即为数组中1的数量，即节点处理的槽的个数。

### 17.2.2 传播节点的槽指派信息
节点会将自己的slots数组通过消息发送给集群中其他节点，告知其他节点自己目前处理哪些槽。节点收到其他节点的slots数组时会在自己的nodes字典中找到该节点对应的clusterNode结构进行更新。

### 17.2.3 记录集群所有槽指派信息
```C++
typedef struct clusterState {
    // ...
    clusterNode *slots[16384];
    // ...
} clusterState;
```
slots[i]指向NULL则表明该槽没有被指派任何节点。此数组可以高效查找一个槽是否被指派给某个节点以及每个槽具体被指派给哪个节点，无需遍历nodes字典。clusterNode.slots数组则可以用于高效的发送特定节点的槽指派信息，而无需遍历clusterState.slots数组。

### 17.2.4 CLUSTER ADDSLOTS命令的实现
遍历命令参数中的所有槽，检查它们是否未指派，如果已经指派则给客户端返回错误，停止执行命令。如果都是未指派的槽，则设置clusterState.slots以及clusterNode.slots的对应项。\
命令执行完毕后节点发送消息告知集群其他节点自己负责处理哪些槽。

## 17.3 在集群中执行命令
客户端向某个节点发送数据库键有关的命令时，接收命令的节点会计算出命令要处理的数据库键属于哪个槽，检查这个槽是否指派给自己。如果是的话直接执行命令，否则向客户端返回MOVED错误，把客户端redirect到正确的节点，让客户端重新发送命令。

对于每个键，计算CRC-16校验和，对16383取模得到对应的键的槽。CLUSTER KEYSLOT命令通过此方法得到键的槽号。\
节点计算得到槽的编号后，查看自己的clusterState.slots数组，看对应的项是否保存对应自己的clusterState.myself结构，判断槽是否由自己负责。\
如果槽不由自己负责，则返回MOVED错误信息，信息包含键所在的槽，以及处理该槽的节点的ip地址和端口号。客户端因此可以转向对应的几点，重新发送命令。所谓的转向就是换一个套接字发送命令，如果还没创建套接字则先连接对应节点，再转向。集群模式下不会显式的提示MOVED错误，转向自动进行，如果使用单机模式MOVED错误会直接打印出来。

节点只能使用0号数据库。除了将键值对保存在数据库中之外，节点还会用clusterState结构中的slots_to_keys跳跃表保存槽和键之间的关系。
```C++
typedef struct clusterState {
    // ...
    zskiplist *slots_to_keys;
    // ...
} clusterState;
```
跳跃表每个节点的分值是一个槽号，成员为键，每当添加一个新的键值对时，节点会将键以及槽号关联到跳跃表，删除键值对时跳跃表中的节点也会被删除。跳跃表可以方便地对属于某个或者某些槽的所有数据库进行批量操作，例如CLUSTER GETKEYSINSLOT命令返回一些属于某个槽的键，通过遍历跳跃表可以实现。

## 17.4 重新分片
重新分片将已经指派给某一个节点（源节点）的槽重新指派给另一个节点（目标节点），相关槽所属的键值对也会移动到目标节点。重新分片可以在线进行，集群不需要下线。\
重新分片操作由redis-trib集群管理软件执行。redis-trib对目标节点发送CLUSTER SETSLOT \<slot> IMPORTING \<source_id>命令，让目标节点准备好导入属于slot的键值对。之后对源节点发送CLUSTER SETSLOT \<slot> MIGRATING \<target_id>迁移对应的键值对。redis-trib向源节点发送CLUSTER GETKEYSINSLOT \<slot> \<count>命令，获取最多count个属于slot的键。对获得的每个键，redis-trib向源节点发送MIGRATE \<target_ip> \<target_port> <key_name> 0 \<timeout>命令，将对应的键迁移至目标节点。重复执行上述步骤知道所有键迁移完成。最后redis-trib向集群任意一个节点发送CLUSTER SETSLOT \<slot> NODE \<target_id>命令，通过消息发送给整个集群，集群中所有节点都知道slot指派给了目标节点。如果涉及多个槽，则对每个槽执行上述步骤。

## 17.5 ASK错误
重新分片期间，属于被迁移的槽的一部分键值对保存在源节点中，另一部分保存于目标节点中。客户端向源节点发送命令想要处理属于该槽的键的时候，源节点如果没有找到对应的键，说明被迁移到目标节点，则向客户端返回ASK错误，指引客户端转向目标节点再次发送命令。与MOVED类似，在集群模式下ASK错误被隐藏，转向是自动进行的。

clusterState结构的importing_slots_from数组记录了当前节点正在从其他节点导入的槽：
```C++
typedef struct clusterState {
    // ...
    clusterNode* importing_slots_from[16384];
    // ...
} clusterState;
```
如果importing_slots_from[i]的值不为NULL，则说明当前节点正在从该值对应的结构代表的节点导入槽i。CLUSTER SETSLOT \<i> IMPORTING \<source_id>即是将该数组的第i项设置为source_id对应的clusterNode结构。

clusterState结构的migrating_slots_to数组记录了当前节点正在迁移至其他节点的槽：
```C++
typedef struct clusterState {
    // ...
    clusterNode* migrating_slots_to[16384];
    // ...
} clusterState;
```
如果migrating_slots_to[i]的值不为NULL，则说明当前节点正在向该值对应的结构代表的节点迁移槽i。CLUSTER SETSLOT \<i> MIGRATING \<target_id>即是将该数组的第i项设置为target_id对应的clusterNode结构。

如果节点收到关于键key的请求，但没有找到对应的键key，节点会检查自己的migrating_slots_to数组，看看对应的槽是否在迁移，如果确实在迁移则返回ASK错误引导客户端去正在导入槽的节点查找key。接到ASK错误的客户端会根据错误提供的ip地址和端口号转向正在导入槽的节点，发送一个ASKING命令，然后再发送原本想要执行的命令。\
ASKING命令的作用是打开客户端的REDIS_ASKING标志。一般情况下客户端向节点发送一个关于槽i的命令，但是i没有被指派给当前节点的话，节点向客户端返回MOVED错误。但是如果节点的importing_slots_from[i]不为空并且客户端的REDIS_ASKING标志打开，节点将破例执行关于该槽的命令。如果之前客户端不通过ASKING命令打开该标志，直接向节点执行原本的命令就会得到MOVED错误。REDIS_ASKING标志是一次性的，节点执行了一次命令后该标志会被移除。\
在客户端关于槽i的命令收到MOVED错误后，客户端可以永久转向MOVED指向的节点，之后每个关于槽i的命令都发送给该节点，因为该节点被指派了槽i。而如果收到ASK错误后，转向是一次性的，下一次客户端不会继续将命令发给ASK错误指向的节点，除非ASK错误再次出现。

## 17.6 复制与故障转移
Redis集群中的节点分为主节点和从节点。主节点用于处理槽，从节点复制某个主节点，并在主节点下线时代替主节点处理命令请求。如果有多个从节点，则集群将在其中选出一个成为主节点，其他从节点改为复制这个新的主节点。原来的主节点上线后成为新主节点的从节点。

### 17.6.1 设置从节点
向一个节点发送CLUTER REPLICATE <node_id>让节点成为node_id对应节点的从节点，开始对主节点进行复制。接收此命令的节点首先在自己的clusterState.nodes字典中找到node_id对应的节点，并将自己的clusterState.myself.slaveof指针指向对应的结构。
```C++
struct clusterNode {
    // ...
    struct clusterNode* slaveof;
    // ...
};
```
之后节点修改自己的myself.flags属性，关闭原本的REDIS_NODE_MASTER标识，打开REDIS_NODE_SLAVE标识，表明自己变成了从节点。最后节点调用复制代码，根据myself.slaveof结构中的ip地址和端口号，对主节点进行复制，复制的实现与单机Redis服务器的复制功能相同，相当于SLAVEOF \<master_ip> \<master_port>。\
一个节点成为从节点并开始复制主节点会通过消息发送给集群中其他节点，最终所有节点都会知道。集群中每个节点会在代表该主节点的clusterNode结构中更新slaves和numslaves属性，记录该主节点的从节点数量以及信息。
```C++
struct clusterNode {
    // ...
    int numslaves;
    struct clusterNode **slaves;
    // ...
};
```

### 17.6.2 故障检测
集群中每个节点会定期向集群中其他节点发送PING消息，检测对方是否在线，如果没有得到某个节点及时的PONG回复，则将该节点标记为疑似下线，将该节点对应的flags设置为REDIS_NODE_PFAIL。\
各个节点通过互相发送消息交换集群中各个节点的状态信息。当一个主节点A通过消息得知主节点B认为主节点C疑似下线时，主节点A在自己的nodes字典中找到C对应的clusterNode结构，将B的下线报告添加到该结构的fail_reports链表中。
```C++
struct clusterNode {
    // ...
    list *fail_reports;
    // ...
};

// 每个list项包含以下结构，表示该clusterNode对应节点被哪些节点在什么时刻认为下线
struct clusterNodeFailReport {
    struct clusterNode* node; // 报告该下线报告的节点

    mstime_t time; // 最后一次从上面的node成员代表的节点收到下线报告的时间，用于检查下线报告是否过期
} typedef clusterNodeFailReport;
```
如果集群中半数以上的负责处理槽的主节点都将某个主节点x报告为疑似下线，则x被标记为已经下线（FAIL），将x标记为FAIL的主节点会向集群广播一条关于x的FAIL消息，所有收到该消息的节点都会将x标记为已经下线。

### 17.6.3 故障转移
当一个从节点发现自己的主节点进入下线状态时，从节点对主节点进行故障转移。首先从该主节点的所有从节点中选取一个作为新的主节点，被选中的从节点执行SLAVE no one命令，成为新的主节点。新的主节点会撤销所有原主节点的槽指派，将这些槽全部指派给自己。之后新的主节点向集群广播一条PONG消息，让集群中其他节点知道该节点已经变成主节点并且接管原主节点的槽。之后新的主节点开始接收和自己负责处理的槽有关的命令。

### 17.6.4 选举新的主节点
当某个节点开始故障转移操作时，集群的配置epoch加1。对于每个epoch，集群每个负责处理槽的主节点都有投票机会，第一个向主节点要求投票的从节点会获得主节点的投票，当从节点发现自己正在复制的主节点下线时，从节点向集群广播一条CLUSTERMSG_TYPE_FAILOVER_AUTH_REQUEST消息，要求收到该消息并且有投票权的主节点向这个从节点投票，如果一个主节点有投票权（负责处理槽）并且尚未投票给其他从节点，那么主节点向要求投票的从节点返回CLUSTERMSG_TYPE_FAILOVER_AUTH_ACK消息，支持该从节点成为新的主节点。每个从节点统计自己受到多少有投票权的主节点支持，如果超过半数则该从节点成为新的主节点。每个epoch中每个主节点只能投票一次，因此超过半数的只能最多有一个。\
此方法与选举领头Sentinel的方法类似，二者均基于Raft算法的领头选举算法实现。

## 17.7 消息
集群中各个节点通过发送和接收消息进行通信，发送消息的节点为发送者（sender），接收消息的节点为接收者（receiver）。节点发送的消息主要有五种。\
MEET消息是在发送者接收到客户端的CLUSTER MEET命令时向接收者发送，请求接收者加入发送者所在的集群。\
PING消息是每个节点每隔一秒钟从已知节点列表中随机选出五个节点，对五个节点中最长时间没有发送过PING消息的节点发送，检测接收者是否在线，此外如果节点A最后一次收到节点B的PONG消息时间超过了timeout设置的一半，那么A也会向B发送PING消息，防止因为长时间没有选中B作为PING消息的接收者而导致对B的信息更新滞后。\
PONG消息是节点收到MEET或者PING消息时用于确认该消息已到达时返回的消息。一个节点也可以向集群广播PONG消息来让其他节点立即更新该节点的信息（例如故障转移成功执行后新的主节点向集群广播PONG消息）。\
FAIL消息是当一个主节点A判断另一个主节点B已经FAIL时向集群广播。所有收到FAIL消息的节点会立即将B标记下线。\
PUBLISH消息是当节点接收一个PUBLISH命令时节点会执行命令并向集群广播。所有接收到PUBLISH消息的节点会执行相同的PUBLISH命令。\
一条消息由消息头（header）和正文（data）组成。

### 17.7.1 消息头
消息由消息头包裹，消息头除了包含正文外还记录了发送者自身的信息。每个消息头用cluster.h/clusterMsg表示。
```C++
typedef struct {
    uint32_t totlen; // 消息长度
    uint16_t type; //消息类型
    uint16_t count; // 正文包含的节点信息数量，只在MEET、PING、PONG这三种Gossip协议消息使用
    uint64_t currentEpoch; // 发送者所处的配置epoch
    uint64_t configEpoch; // 如果是主节点，则记录发送者的配置epoch，如果是从节点则是该从节点对应的主节点的配置epoch

    char sender[REDIS_CLUSTER_NAMELEN]; // 发送者名字（id）
    unsigned char myslots[REDIS_CLUSTER_SLOTS/8]; // 发送者的槽指派信息

    char slaveof[REDIS_CLUSTER_NAMELEN]; // 如果是从节点则记录主节点的名字，如果是主节点则记录REDIS_NODE_NULL_NAME

    uint16_t port; // 发送者端口号
    uint16_t flags; // 发送者的标识

    unsigned char state; // 发送者所处集群状态

    union clusterMsgData data; // 消息正文
} clusterMsg;

union clusterMsgData {
    // MEET、PING、PONG消息的正文
    struct {
        /* Array of N clusterMsgDataGossip structures */
        clusterMsgDataGossip gossip[1];
    } ping;

    // FAIL消息
    struct {
        clusterMsgDataFail about;
    } fail;

    // PUBLISH消息
    struct {
        clusterMsgDataPublish msg;
    } publish;

    // 其他消息正文
    // ...
};
```
接收者可以通过消息中发送者自身的信息对自己nodes字典中发送者对应的clusterNode结构进行更新（例如槽指派信息、flags中节点状态如主从节点等信息）。

### 17.7.2 MEET、PING、PONG消息的实现
集群节点通过Gossip协议交换各自的状态信息。Gossip协议由MEET、PING、PONG三种消息实现，三种消息的正文由cluster.h/clusterMsgDataGossip结构组成，节点通过消息头的type属性判断一条消息是哪一种。\
每次发送MEET、PING、PONG消息时，发送者从已知节点列表中随机选择两个节点（可以是主节点或者从节点）将它们的信息保存到两个clusterMsgDataGossip结构中。该结构记录了被选中节点的名字、发送者与被选中节点最后一次发送和接收PING和PONG消息的时间戳、被选中的节点ip地址和端口号以及标识值。
```C++
typedef struct {
    char nodename[REDIS_CLUSTER_NAMELEN];

    uint32_t ping_sent; // 最后一次向该节点发送PING消息的时间
    uint32_t pong_received; // 最后一次从该节点接收到PONG消息

    char ip[16];
    uint16_t port;
    uint16_t flags;
} clusterMsgDataGossip;
```
当接收者收到MEET、PING、PONG消息时接收者访问两个clusterMsgDataGossip结构，根据自己是否认识对应的被选中节点来进行操作，如果接收者的已知节点列表没有该节点，则说明接收者第一次接触到被选中节点，接收者将根据结构中的ip地址和端口号与被选中节点握手。否则接收者对被选中节点的clusterNode结构进行更新。

### 17.7.3 FAIL消息的实现
在集群节点数量比较大的情况下，Gossip协议传播节点的下线信息会带来延迟。FAIL消息可以让集群里所有节点立即知道某个节点已经下线，从而判断是否需要将集群下线或者对下线主节点进行故障转移。\
FAIL消息正文由cluster.h/clusterMsgDataFail结构表示，该结构只包含一个nodename属性，为已经下线节点名字。
```C++
typedef struct {
    char nodename[REDI_CLUSTER_NAMELEN];
} clusterMsDataFail;
```
当超过一半的主节点标记该下线节点为FAIL时，其他节点可以判断是否需要将集群标记为下线或者开始故障转移。

### 17.7.4 PUBLISH消息的实现
当客户端向集群中的某个节点发送PUBLISH \<channel> \<message>命令的时候，该节点不仅会向channel频道发送消息message，还会向集群广播PUBLISH消息，所有接收到PUBLISH消息的节点都会channel频道发送message消息。\
PUBLISH消息的正文cluster.h/clusterMsgDataPublish结构表示：
```C++
typedef struct {
    uint32_t channel_len;
    uint32_t message_len;

    unsigned char bulk_data[8]; // 实际长度由保存内容决定，8字节只是为了对齐其他消息结构
} clusterMsgDataPublish;
```
其中bulk_data属性是一个字节数组，保存了channel参数和message参数，channel_len和message_len分别保存对应的长度，bulk_data的[0, channel_len)字节保存channel参数，[channel_len, channel_len + message_len)字节保存message参数。\
如果直接广播PUBLISH命令，不符合集群“各个节点通过发送和接收消息来通信”的规则。
