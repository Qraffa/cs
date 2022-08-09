# Redis

## 基本数据类型

### 简单动态字符串SDS

SDS结构如下：

```c
struct __attribute__ ((__packed__)) sdshdr64 {
    uint64_t len; /* used */
    uint64_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
```

- len：记录已经使用的字节数
- alloc：记录当前字符串总分配的字节数
- flags：标示当前字符串的属性
- buf：底层字节数组

SDS特点：
传统C字符串在对字符串进行增加/删除字符的时候，都需要对字节数组进行重新内存分配，为了解决此问题，SDS实现了空间预分配和惰性回收的优化

1. 空间预分配
   当对SDS进行修改时，首先检查是否有足够的空闲空间，没有则需要进行空间扩展时，会为SDS分配额外的空间

- 当SDS长度小于1M时，则分配与SDS相同大小的空闲空间
- 当SDS长度大于1M时，则分配1M的空闲空间

2. 惰性回收
   当对SDS进行修改时，缩短了字符串后，并不会立即重新分配内存以回收空闲空间，而是会将空闲的记录下来，以便后续可以直接使用

字符串三种编码方式：int、raw、embstr

1. int
   保存long范围内的整型数字，编码格式为int
2. raw
   保存字符串，并且长度大于44个字节(OBJ_ENCODING_EMBSTR_SIZE_LIMIT)，使用SDS保存，编码格式为raw
3. embstr
   保存字符串，并且长度小于等于44个字节，编码格式为embstr

其中：embstr和raw都是由SDS动态字符串构成的。唯一区别是：raw是分配内存的时候，redisobject和 sds 各分配一块内存，而embstr是redisobject和raw在一块儿内存中。

### 链表

链表结构如下：

```c
typedef struct listNode {
    struct listNode *prev;
    struct listNode *next;
    void *value;
} listNode;

typedef struct list {
    listNode *head;
    listNode *tail;
    void *(*dup)(void *ptr);
    void (*free)(void *ptr);
    int (*match)(void *ptr, void *key);
    unsigned long len;
} list;
```

- head：链表头节点
- tail：链表尾节点
- dup：用于复制链表节点
- free：用于释放链表节点
- match：用于比较链表节点
- len：记录链表长度

链表的特点：

1. 双向链表，头尾指向NULL
2. 带有计数器len，获取链表长度，时间复杂度O(1)
3. 链表节点可以保存各种类型的值

### 字典

字典结构如下：

```c
typedef struct dict {
    dictType *type;
    void *privdata;
    dictht ht[2]; // 两个元素数组，一般只使用ht[0]，在rehash过程中会用到两个
    long rehashidx; /* rehashing not in progress if rehashidx == -1 */
    unsigned long iterators; /* number of iterators currently running */
} dict;

typedef struct dictht {
    dictEntry **table; // 哈希链表
    unsigned long size; // 大小
    unsigned long sizemask;
    unsigned long used; // 已有节点数
} dictht;

typedef struct dictEntry {
    void *key; // 键
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v; // 值
    struct dictEntry *next; // 指向下一个entry
} dictEntry;
```

![](https://segmentfault.com/img/remote/1460000040206824)

渐进式rehash：
随着哈希表中元素的增删操作，为了让哈希表的负载因子保持在一定范围内，我们需要对哈希表进行扩容或者缩容操作。但如果一个哈希表的元素非常多，我们如果想一次执行完全部元素的rehash操作，开销是非常大的。因此使用渐进式rehash，将整个rehash操作均摊到多个对哈希表的操作中，减少了系统的抖动。

实现方法：

1. 为ht[1]分配空间。如果是扩容操作，则ht[1]的大小为第一个大于等于 ht[0].used * 2 的 2^n （2 的 n 次方幂）；如果是缩容操作，则ht[1]的大小为第一个大于等于 ht[0].used 的 2^n 。
2. 将字典中的rehashidx字段置为0，表示正在进行rehash操作
3. 在rehash期间，每次对字典的增删改查操作，除了指定的操作外，还会将ht[0]中的部分键值对rehash到ht[1]中
4. 随着操作的不断进行，在某个时间点，ht[0]中所有的键值对都rehash到ht[1]中，释放ht[0]的空间，ht[1]重新变成ht[0]

其中，在rehash过程中，如果是查找操作，则先查找ht[0]，在查找ht[1]；如果是增加操作，则直接添加到ht[1]中

### 跳表

zset内部使用dict存储value和score的映射关系，使用skiplist来维护score排序功能。

```c
typedef struct zskiplistNode {
    sds ele;
    double score;
    struct zskiplistNode *backward;
    struct zskiplistLevel {
        struct zskiplistNode *forward;
        unsigned long span;
    } level[];
} zskiplistNode;

typedef struct zskiplist {
    struct zskiplistNode *header, *tail;
    unsigned long length;
    int level;
} zskiplist;
```

一个普通的单链表查询一个元素的时间复杂度为O(N)，即便该单链表是有序的。使用跳跃表（SkipList）是来解决查找问题的，它是一种有序的数据结构，不属于平衡树结构，也不属于Hash结构，它通过在每个节点维持多个指向其他节点的指针，而达到快速访问节点的目的。

跳跃表其实可以把它理解为多层的链表，它有如下的性质：

- 多层的结构组成，每层是一个有序的链表
- 最底层（level 1）的链表包含所有的元素
- 跳跃表的查找次数近似于层数，时间复杂度为O(logn)，插入、删除也为 O(logn)
- 跳跃表是一种随机化的数据结构(通过抛硬币来决定层数)

redis中跳表特点：

- 在节点中同时维护了跨度span，用于表示该节点到起指向的下一个节点的相隔的节点个数，因此可以使用该字段来计算rank
- zset中首先用score排序，score一样的情况下，使用value排序
- redis中选择随机层数时，层数晋升的概率是1/4

### 压缩链表

ziplist结构如下：

```c
// ziplist
// <zlbytes> <zltail> <zllen> <entry> <entry> ... <entry> <zlend>

// ziplist entry
// <prevlen> <encoding> <entry-data>
```

- zlbytes：记录整个ziplist占用的字节数
- zltail：ziplist尾节点的偏移量
- zllen：ziplist中entry数量
- zlend：特殊entry，标记整个ziplist的结尾
- prevlen：记录前一个entry的长度
- encoding：保存当前节点的类型

当zset和hash元素个数比较少时，会使用ziplist进行存储。ziplist是一块连续的内存空间，entry直接紧密排列。
其中encoding用于存储当前节点的类型，包括不同长度的字符串、不同类型的整数。content用于存储entry的实际内容

### quicklist

quicklist结构如下：

```c
typedef struct quicklist
{
    quicklistNode *head;
    quicklistNode *tail;
    unsigned long count;                  /* total count of all entries in all ziplists */
    unsigned long len;                    /* number of quicklistNodes */
    int fill : QL_FILL_BITS;              /* fill factor for individual nodes */
    unsigned int compress : QL_COMP_BITS; /* depth of end nodes not to compress;0=off */
    unsigned int bookmark_count : QL_BM_BITS;
    quicklistBookmark bookmarks[];
} quicklist;

typedef struct quicklistNode
{
    struct quicklistNode *prev;
    struct quicklistNode *next;
    unsigned char *zl;
    unsigned int sz;                     /* ziplist size in bytes */
    unsigned int count : 16;             /* count of items in ziplist */
    unsigned int encoding : 2;           /* RAW==1 or LZF==2 */
    unsigned int container : 2;          /* NONE==1 or ZIPLIST==2 */
    unsigned int recompress : 1;         /* was this node previous compressed? */
    unsigned int attempted_compress : 1; /* node can't compress; too small */
    unsigned int extra : 10;             /* more bits to steal for future usage */
} quicklistNode;
```

每个quicklistNode中都存储了一个ziplist，quicklistNode之间双向链接起来

在插入时，如果当前节点中的ziplist大小没有超出限制，则直接将数据插入到ziplist中。否则，需要新创建一个quicklistNode节点，然后插入数据。

### 整数集合

intset结构如下：

```c
typedef struct intset {
    uint32_t encoding;
    uint32_t length;
    int8_t contents[];
} intset;
```

- encoding：编码方式，决定contents数组元素的真正类型
- length：contens数组的长度
- contents：每个元素在数组中升序排序，且不重复

intset升级：
如果新添加的元素类型比现在有的元素类型要长，则需要对intset进行升级。

升级过程：

- 根据新元素类型大小，扩展底层数组空间大小，并为新元素分配空间
- 把现有的元素替换成新元素类型，并保持升序排序
- 添加新元素

### 总结

- 字符串(string)：int、raw、embstr
  数字类型且在long范围则为int；字符串类型，大于一定字节用raw，小于等于一定字节用embstr

- 列表(list)：quicklist

- 集合(set)：intset、hashtable
  当集合中所有元素都为整数，且小于512个时，为intset。其他则为hashtable

- 有序集合(zset)：ziplist、skiplist + dict
  当集合中元素数量小于128，且所有元素长度小于64字节，为ziplist。其他为skiplist + dict

    ```c
    typedef struct zset {
        dict *dict;
        zskiplist *zsl;
    } zset;
    ```

    当使用ziplist时，每个集合元素需要使用两个紧挨在一起的ziplist节点来保存，member在前，score在后
    当使用skiplsit时，使用dict来创建从member到score的映射；skiplistNode中的ele保存memeber，score保存score。两个集合直接使用指针来共享member和score。

- 散列(hash)：ziplist、hashtable
  当键值对的长度小于64字节，且键值对个数小于512时，为ziplist。其他为hashtable
  使用ziplist时，添加键值对时，将键和值分别添加到ziplist尾部。