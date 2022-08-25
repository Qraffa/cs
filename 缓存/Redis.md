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

## 持久化

### RDB

rdb文件是由redis生成的一个二进制文件，用于保存某一时刻的redis数据库的状态的，包含了某一时刻的所有键值对的信息。

#### save和bgsave

redis由两个命令可以生成rdb文件，`save`和`bgsave`

save命令使用主线程来生成，因此其会阻塞操作命令的执行。bgsave命令则是fork一个子进程，由子进程来完成生成rdb文件的工作，因此其不会阻塞操作命令的执行。

#### 触发

除了使用命令手动触发rdb持久化外，redis还会根据配置文件来自动执行bgsave。redis中的`serverCron`函数每隔100ms执行一次（由参数hz控制，默认10，表示每秒执行10次）。在该函数中会检查是否满足执行bgsave的条件，如果当前由子进程在执行bgsave或aof重写，则不会再执行bgsave。

默认配置：

- 900秒内对数据库至少进行了1次修改
- 300秒内对数据库至少进行了10次修改
- 60秒内对数据库至少进行了10000次修改

```haskell
save 900 1
save 300 10
save 60 10000
```

### AOF

与rdb生成redis的快照不同，aof是通过保存被执行后的命令来记录数据库状态的。

在redis收到一条写操作指令后，先执行参数检查，执行指令逻辑等，再讲指令记录到AOF文件中。这样做可以避免额外的开销，只有执行成功的指令才会被记录到AOF文件中。

#### AOF写回策略

redis在执行过程中：

1. 执行完写操作命令后，将被执行的命令写入到aof_buf缓冲区中
2. 通过write系统调用，将aof_buf缓冲区的内容写入到aof文件中。

> 在现代操作系统中，用户调用write函数将数据写入到文件中时，os往往是先将需要写入的数据保存在内核缓冲区中，等到一定条件才将内核缓冲区的数据刷回到磁盘中，这样提高了写入的效率，但也带来了数据安全的问题，因此还提供了fsync等函数来强制os立刻将缓冲区的数据刷到磁盘中。

redis使用配置`appendfsync`参数来控制刷盘的时机

- always：每次执行完写操作命令后，将命令写入到AOF文件中，还会强制同步到磁盘中
- everysec：将命令写入到AOF文件中，不强制同步，而是每隔一秒将缓冲区的数据同步到磁盘中
- no：不做同步磁盘的控制，由os决定何时将缓冲区数据同步到磁盘中

三种策略的特点：

- always：效率最低，但能最大程度保证数据的安全性
- everysec：效率和安全折中，宕机时可能丢失少量的数据
- no：效率最高，但宕机时可能丢失大量数据

#### AOF重写

随着服务的运行，AOF中的命令会越来越多，AOF文件也会越来越大，并且其中有些命令已经是冗余的了。为了解决该问题，提供了AOF重写的能力，通过AOF重写，redis将产生一个新的AOF文件来替换掉旧的AOF文件。

**AOF重写实现：**AOF重写通过对当前数据库的状态来生成命令，从数据库中读取键现在的值，然后用一条命令记录该键值对，并写入到新的AOF文件中，当写入完成后，将新文件替换掉旧的AOF文件。

##### AOF后台重写

为了不影响主线程执行操作命令，redis通过fork子进程，由子进程来完成AOF的重写工作，这样由两个好处：

- 子进程在重写AOF的过程中，主进程仍然可以继续执行操作命令
- 子进程会有父进程的数据的副本，因此与线程不同的是，可以避免锁的使用，保证数据的安全性

**AOF重写时追加：**在重写过程中，主进程并不会阻塞，因此仍然可以完成写操作指令，那么此时子进程和父进程的内存数据状态就不一致了。

为了解决该问题，我们还需要对重写过程中产生的AOF继续记录下来，redis设置了一个AOF重写缓冲区，在子进程在进行AOF重写时，主进程对于写操作由如下工作：

1. 执行写操作命令
2. 将执行后的命令追加到AOF缓冲区中（给旧的AOF文件使用的）
3. 将执行后的命令追加到AOF重写缓冲区中

这样做的目的：

- 继续写入AOF缓冲区，那么旧AOF文件的缓冲区内容会被定期刷回到AOF文件中，旧AOF文件能正常工作
- 将命令写入到AOF重写缓冲区中，保证了新的AOF文件的数据库状态一致性

当子进程完成了对AOF的重写工作后，会向父进程发出一个信号，父进程收到信号后，阻塞的执行信号处理函数，主要完成以下工作：

- 将AOF重写缓冲区中的内容写到新的AOF文件中
- 对新AOF文件重命名，原子的覆盖旧的AOF文件

### RDB与AOF的对比

**RDB特点：**

优点：

- RDB是redis数据在某个时间点的快照文件，格式非常紧凑，适合用于备份
- RDB适合容灾恢复，RDB文件是可以传输的单个紧凑文件
- RDB相比AOF恢复速度更快

缺点：

- RDB在redis宕机时更容易丢失数据
- RDB需要fork一个在磁盘上进行持久化工作的子进程，在数据量大、CPU性能受限的情况下，可能导致Redis在几毫秒甚至一秒内不对外服务

**AOF特点：**

优点：

- redis宕机时丢失的数据更少
- AOF日志更易于理解和解析，包含所有的操作

缺点：

- 在相同的数据情况下，AOF文件比RDB文件更大
- 取决于具体的fsync的策略，AOF可能比RDB慢

### 混合持久化

由于AOF文件中记录的是执行的命令，因此在恢复时，需要一条一条执行的执行来恢复数据库的状态，而RDB文件记录的是数据库的快照，因此恢复速度快于AOF的方式。

redis将RDB和AOF结合起来，提供了混合持久化的功能。当开启混合持久化后，在子进程进行AOF重写时，将当前数据库的快照记录以RDB的方式记录，然后再讲重写过程中产生的增量AOF日志记录，将两者结合生成一个新的AOF文件，该文件中前部分为RDB内容，后部分的AOF内容，然后使用新的AOF文件替换掉原AOF文件。

在恢复加载时，由于前部分是RDB，因此恢复速度快，加载完RDB后再加载后部分的AOF日志，往往增量的AOF日志数据量都比较小。

因此使用混合持久化保证了全量的数据，又保证了恢复加载时的速度。

## 主从复制

为了保证redis的高可用性，redis提供主从模式，通过主从架构，将数组冗余备份。在主从架构下，采用读写分离的方式，主节点可以进行读写操作，从节点只能进行读操作，写则是由主节点同步到从节点上。

### 同步

同步操作用于将**从服务器**状态更新至**主服务器**当前的数据库状态。

#### 完整重同步

**执行完整重同步的时机：**

- 在从服务器第一次要求同步时，会执行完整重同步。
- 在部分重同步不能将状态更新到一直时需要执行完整重同步。

**完整重同步的过程：**

1. 从服务器向主服务器发送命令，建立连接
2. 主服务器执行`bgsave`，生成RDB文件，同时开辟一个缓冲区（replication buffer），记录生成RDB时收到的写命令
3. 将生成的RDB文件发送给从服务器，从服务器会清空当前数据库，然后加载RDB文件
4. 将缓冲区中的数据发送给从服务器，从服务器执行这些命令，将数据库状态同步到主服务器的数据库状态

![Redis全量同步](http://magebyte.oss-cn-shenzhen.aliyuncs.com/redis/Redis%E5%85%A8%E9%87%8F%E5%A4%8D%E5%88%B6%E7%BB%88%E6%9E%81.png)

#### 部分重同步

为了解决在主从服务器网络断开再重连后，需要做一次完整的重同步，带来的巨大开销，提供了部分重同步。通过部分重同步，只需要将在网络断开时间内的命令，发送给从服务器同步状态即可。

**部分重同步的执行时机：**

- 当出现主从服务器网络中断后，从服务器重写连接，尝试执行部分重同步

**部分重同步的过程：**

1. 在主服务器运行过程中，会将写命令同时记录到`repl_backlog_buffer`缓冲区中
2. 主服务器会记录目前数据库的写入偏移量，以及每个从服务器会记录自己的写入偏移量。通过主从服务器偏移量的比较，可以判断主从服务器的数据库状态是否一致
3. 每个redis服务器都有自己的runID，在首次主从复制完成后，从服务器会将主服务器的runID记录下来
4. 断开重连后，从服务器通过`psync`命令，将保存的主服务器的runID，以及从服务器自己的offset发送给主服务器
5. 主服务器通过检查runID和offset等信息，将两者offset偏差的部分发送给从服务器即可

**缓冲区`repl_backlog_buffer`：**

该缓冲区存在于主服务器中，且只有一个，且是固定大小的，采用环形结构，因此新数据是会覆盖最旧的数据的。

因此如果在断开重连间隔的时间比较长，可能从服务器需要的偏移量的数据已经被覆盖掉了，此时将不能执行部分重同步，需要兜底到完重同步。

为避免上述的情况，应该设置合理大小的缓冲区：

```haskell
repl_backlog_buffer = second * write_size_per_seond
```

1. second：表示断开到重连的平均花费时间
2. write_size_per_seond：表示主服务器中平均每秒的写入量

### 命令传播

在完成同步操作后，主从服务器之间还会建立长连接，后续主服务器会通过该连接将写命令发送给从服务器，使用长连接避免连接频繁的建立销毁带来的开销。

#### 心跳

在命令传播阶段，除了同步增量的写命令外，还维护了心跳机制：PING命令和REPLCONF ACK命令

**REPLCONF ACK命令：**

从服务器每秒会向主服务器发送`REPLCONF ACK <replication_offset>`，包含以下三个作用：

- 检查主从服务器之间的网络状态

  主服务器会记录从服务器上一次发送命令的时间，计算出lag值

- 实现min-slaves机制

  通过以下两个配置：

  ```haskell
  min-replicas-to-write <number of replicas>
  min-replicas-max-lag <number of seconds>
  ```

  表示需要至少`<number of replicas>`个从服务器，并且他们的lag值都小于`<number of seconds>`，该写命令才可以被执行

- 检查命令丢失

  从服务器通过该命令将自己的偏移量发送给主服务器，当主服务器检查到从服务器的偏移量不一致时，将根据偏移量把缓冲区中的数据发送个从服务器

## 哨兵

### 哨兵主要功能

Redis哨兵是运行在特殊模式下的Redis服务器，主要用于监控各Redis服务器的状态，以及执行故障转移，主从切换等工作，来保证Redis的高可用性。

哨兵机制主要有以下功能：

- 监控：哨兵持续检查master和replica是否正常运行
- 通知：当监控中的Redis实例出现错误时，可以通知给其他进程
- 自动故障转移：当master出现故障时，哨兵启动故障转移，选举合适的replica成为新的master，并且调整其他的replica与新的master同步；并且通知客户端使用新的master地址进行连接

### 哨兵实现机制

哨兵与各Redis服务器通过建立命令连接以及订阅连接来通信。

哨兵首先会与master建立命令连接和订阅连接，然后与master通信，使用`info`命令获取master的replica，然后与这些replica同样建立命令连接和订阅连接。

在哨兵集群中，哨兵通过master的订阅频道发现了其他哨兵后，会与该哨兵建立命令连接，最终将整个哨兵集群连接起来。

### 故障转移

#### master下线

哨兵会每秒一次向其他Redis实例发送`PING`命令，以检查连接情况。如果实例在一定时间内，没有正常响应PING命令，则该哨兵会认为该实例已经是**主观下线（SDOWN）**，该时间有参数`down-after-milliseconds`控制。

当master被哨兵认为是SDOWN后，该哨兵需要进一步向其他哨兵确认，确认该master是否真的已经下线了。当一定数量的哨兵都确认该master已经下线后，该master由主观下线变更为**客观下线（ODOWN）**。只有master实例由ODOWN状态。

#### 哨兵Leader

当master被认为客观下线后，与该master建立了监控连接的哨兵会进行协商，选举出一个哨兵Leader，由该哨兵来进行故障转移的操作。

**选举Leader：**

- 每个发现master进入客观下线状态的哨兵都会成为候选者，并向其他哨兵发送命令，期望将自己设置为Leader

- 候选者会将票投给自己
- 在一次选举epoch中，一个哨兵只会投票一次，采用先到先得的原则
- 当候选者的票数满足以下情况后，该哨兵成为Leader
  - 获得哨兵集群中过半的票数
  - 获得的票数要大于等于该master配置的quorum的数量

#### 执行故障转移

当选举出Leader后，该Leader执行故障转移，包含以下几个过程：

1. 从该master的replica中选举新的master
2. 将原master的replica改为复制新的master
3. 将原master设置为新master的replica
4. 通知客户端，新master的变更信息

**选举新Leader：**

1. 首先将已经下线的replica去掉，保证选的是正常在线的
2. 将与原master断开超过`down-after-milliseconds * 10 ms`的replica去掉，保证选的replica网络良好，并且数据比较新
3. 根据replica的优先级、写入偏移量、runID选出新的Leader

**修改replica的复制目标**

向其他replica发送`slaveof`命令，让这些replica成为新master的从节点

**修改原master**

修改哨兵中记录的原master的相关信息，并且等原master上线后，向其发送`slaveof`命令，让其成为新master的从节点

## 集群

### 节点添加

Redis集群是一种分布式数据库方案，通过分片来进行数据管理，并提供复制和故障转移功能。

一个Redis集群通常由多个节点组成，在刚开始的时候，每个节点都是互相独立的，他们都处于只有自己的一个集群中，我们需要将多个节点连接起来，组成一个包含多个节点的集群。通过`CLUSTER MEET`命令。

### 槽

Redis集群通过分片的方式来保存数据库中的键值对，整个集群的数据库被划分为16384个槽，每个键值对会属于其中一个槽，每个节点可以负责管理0~16384个槽。当16384个槽都有节点在负责时，整个集群才算是上线状态。

Redis集群会通过消息来共享槽指派的信息，这样就可以直接槽由哪些节点负责。

> 槽数量为什么选择16384？
>
> 1. 因为集群中的节点需要通过心跳数据包来共享槽指派信息，如果槽数量太多会导致心跳数据包太大
> 2. redis集群中的主节点数量不会超过1000个，节点越多心跳数据包就越大，会带来网络拥塞
> 3. 槽位少，节点数少的时候，压缩率高

**客户端访问：**

对于一个键值对，Redis对key执行CRC16算法，再mod 16384，得到具体的槽号，将该键值对存放在负责该槽号的节点上。与单机数据库不同的是，集群中的Redis只能使用0号数据库。

在客户端连接集群后，客户端会缓存一份该集群的槽配置信息，然后在访问某个key时，同样执行上述过程得到key的具体槽号，再去具体的节点访问。

### 重新分片

Redis集群的重新分片可以将任意数量的已经指派给某个节点的槽，重新指派给另一个目标节点，并且这些槽相关的键值对也会从源节点迁移到目标节点。

**重新分片过程：**

1. 让目标节点准备好从源节点导入的槽，此时目标节点的槽状态为importing
2. 让源节点准备好从发送给目标节点的槽，此时源节点的槽状态为migrating
3. 从源节点的槽中获取键值对的key
4. 对于每个key，序列化然后发送给目标节点，目标节点恢复后，源节点便可以将该key删除
5. 当该槽的所有key都迁移完毕后，向集群中发布信息，通知该槽指派到新节点

#### MOVED错误和ASK错误

MOVED错误：当节点发现访问的键所在的槽不是自身负责的时候，会返回MOVED错误，指引客户端访问负责该槽的节点

ASK错误：当节点发现访问的键锁在的槽是自身负责时，先在自身查找该键值对；如果未找到，则检查该槽是否正在迁移，如果是在迁移状态，则返回ASK错误，指引客户端访问导入该槽的节点

ASKING命令：在访问一个正在导入槽的节点时，由于此时该节点还没有负责该槽，因此会返回MOVED错误，又回到原节点，形成**重定向循环**。因此在访问一个正在导入槽的节点时，需要先执行ASKING命令，让该节点执行一次该槽的相关命令。

### 故障转移

Redis集群中的节点分为主节点和从节点，主节点负责处理槽，从节点则复制主节点，并且当主节点下线后，升级为主节点。

**故障检测：**

集群中的每个节点都会定期向其他节点发送PING信息，以检查对方是否在线，如果在规定时间内没有收到正确的响应，则会认为该节点疑似下线（probable fail，PFAIL）

如果集群中过半的主节点都认为某个主节点x已经疑似下线，那么会从疑似下线变更为已下线（FAIL），并且通知其他节点，将该节点x标记为已下线。

**故障转移：**

当某个主节点的从节点发现自己的主节点下线后，会对下线的主节点进行故障转移：

1. 从下线的主节点的从节点中选择一个，令其成为新的主节点
2. 新主节点撤销所有对已下线节点的槽指派，并将这些槽指派给自身
3. 新主节点向集群广播PONG消息，通知其他节点该节点为主节点，并且接管了原来已下线的主节点的槽
4. 新主节点处理槽相关命令，故障转移完成

**选举新的主节点：**

1. 每次开始执行故障转移，集群的配置纪元+1，该值初始为0
2. 当某个从节点发现自己的主节点下线后，向集群广播`CLUSTERMSG_TYPE_FAILOVER_AUTH_REQUEST`消息，要求收到该信息的，有投票权的主节点为自己投票
3. 如果收到信息的主节点没有为其他节点投过票，并且它有投票权，则为发送消息的从节点投票
4. 当一个从节点收到过半的投票时，那么该从节点升级成为主节点
5. 如果在一个配置纪元中没有选举出主节点，则进入下一轮配置纪元的选举