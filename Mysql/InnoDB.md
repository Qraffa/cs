# InnoDB

InnoDB存储引擎整体架构：

![img](https://qraffa-1304595678.cos.ap-guangzhou.myqcloud.com/img/innodb-architecture.png)

## 索引





## Buffer Pool

![mysql](D:\chromedownload\mysql.png)

### 简介

InnoDB的数据都以页的格式存储在磁盘的表空间中，为了弥补CPU和磁盘之间的速度差异，在CPU和磁盘之间构建一个缓存，用于管理数据页，提高访问效率，该缓存就是buffer pool。

整个buffer pool会划分成多个buffer pool实例，每个实例管理自己的chunk和链表等结构，这些结构与表空间一样，都是以相同大小的数据页(默认16KB)来管理的。

一个buffer pool实例是由一个或多个buffer chunk组成的，在初始化时，buffer pool以一个chunk为单位向操作系统申请内存空间，一个chunk就是一片连续的内存空间，默认为128M。

![1706](https://qraffa-1304595678.cos.ap-guangzhou.myqcloud.com/img/1706.png)

### Page Hash

每个buffer pool实例都维护自己的page hash，当需要访问一个page时，首先通过【表空间号+页号】作为key，快速在buffer pool中查找，避免LRU链表的全链表扫描。

### FREE链表

FREE链表中管理的是未被使用的空闲的page，在buffer pool初始化的时候，会向操作系统申请内存，然后将chunk中的所有page都加入到FREE链表中。当从磁盘加载一个page到buffer pool时，从FREE链表获取一个空闲page，如果获取不到，则会从LRU链表和FLUSH链表中淘汰旧page和刷dirty page来回收page。

### LRU链表

所有从磁盘加载到buffer pool的page都会添加到LRU链表中，由LRU链表管理和维护。其中LRU链表划分为young区域和old区域，通过参数`innodb_old_blocks_pct`参数控制，默认为37，表示LRU链表的5/8为young区域，3/8为old区域。

![image-20220805193218228](https://qraffa-1304595678.cos.ap-guangzhou.myqcloud.com/img/image-20220805193218228.png)

当向buffer pool访问一个page时：

1. 通过page hash查找是否已经存在，如果不存在则从磁盘加载，并添加到LRU链表的old区域的头部
2. 如果已经存在，则看该page在old区还是young区，如果在old区
   1. 判断距离第一次访问该page的时间，由参数`innodb_old_blocks_time`控制，单位为ms，默认为1000。如果在该时间间隔内，则不会移动到young区域
   2. 如果超过该时间间隔，则移动到young区域头部
3. 如果在young区，如果该page位于young区域的后1/4部分，则会移动到young区域的头部，否则不移动

### FLUSH链表

被修改过的page都认为是dirty page，该page的控制块信息会被加入到FLUSH链表中。在FLUSH链表上的page，一定也在LRU链表上，但反之则不成立。在FLUSH链表中的page，都保存了其第一次修改时的LSN。

### 刷脏

- 后台线程会定时从LRU链表的尾巴扫描页面，如果发现脏页，则刷到磁盘
- 后台线程会定时从FLUSH链表中刷新一部分脏页到磁盘中
- 当用户线程访问页面，但没有空闲页时，会首先尝试从LRU链表释放掉未修改的页，如果没有，则需要将LRU链表的一个脏页刷到磁盘中

## 锁

![mysql (1)](https://qraffa-1304595678.cos.ap-guangzhou.myqcloud.com/img/mysql%20(1).png)

### 锁内存结构

InnoDB中锁本质是一个内存中的结构，对一条记录加锁的本质就是生成一个锁结构与之关联。

锁结构：

```c
// 5.7
/** Lock struct; protected by lock_sys->mutex */
struct lock_t {
	trx_t*		trx;		/*!< transaction owning the
					lock */
	UT_LIST_NODE_T(lock_t)
			trx_locks;	/*!< list of the locks of the
					transaction */

	dict_index_t*	index;		/*!< index for a record lock */

	lock_t*		hash;		/*!< hash chain node for a record
					lock. The link node in a singly linked
					list, used during hashing. */

	union {
		lock_table_t	tab_lock;/*!< table lock */
		lock_rec_t	rec_lock;/*!< record lock */
	} un_member;			/*!< lock details */

	ib_uint32_t	type_mode;	/*!< lock type, mode, LOCK_GAP or
					LOCK_REC_NOT_GAP,
					LOCK_INSERT_INTENTION,
					wait flag, ORed */
}

/** Record lock for a page */
struct lock_rec_t {
	ib_uint32_t	space;		/*!< space id */
	ib_uint32_t	page_no;	/*!< page number */
	ib_uint32_t	n_bits;		/*!< number of bits in the lock
					bitmap; NOTE: the lock bitmap is
					placed immediately after the
					lock struct */
};
```

- trx：记录锁所在的事务信息
- index：对于行锁，记录该锁属于的索引
- hash：用于构造hash，所有的锁都保存在Lock_sys->hash中
- lock_rec_t：对于行锁，保存了表空间，页号，以及使用了多少比特位
- type_mode：表示锁的类型，由32位整数构成
  - lock_mode：占低4位
    - LOCK_IS=0：表示共享意向锁
    - LOCK_IX=1：表示排他意向锁
    - LOCK_S=2：表示共享锁
    - LOCK_X=3：表示排他锁
    - LOCK_AUTO_INC=4：表示AUTO_INC锁，自增锁
  - lock_type：占5~8位
    - LOCK_TABLE=16：表示表级锁
    - LOCK_REC=32：表示行级锁
  - rec_lock_type：占剩余高位
    - LOCK_WAIT=256：表示锁等待
    - LOCK_ORDINARY=0：表示next-key锁
    - LOCK_GAP=512：表示gap锁
    - LOCK_REC_NOT_GAP=1024：表示锁记录本身
    - LOCK_INSERT_INTENTION=2048：表示插入意向锁

### 锁类型

#### 表级锁

- 表级S锁、X锁

```mysql
lock tables t read; // S
lock tables t write; // X
```

- 表级IS锁、IX锁

  在对表中某些记录加S锁之前，需要先加表级IS锁；在对表中某些记录加X锁之前，需要先加表级IX锁。IS锁和IX锁的作用在于后续加表级S锁和X锁时，判断该表是否已经有被锁的记录。

  |      | `X`      | `IX`       | `S`        | `IS`       |
  | :--- | :------- | :--------- | :--------- | ---------- |
  | `X`  | Conflict | Conflict   | Conflict   | Conflict   |
  | `IX` | Conflict | Compatible | Conflict   | Compatible |
  | `S`  | Conflict | Conflict   | Compatible | Compatible |
  | `IS` | Conflict | Compatible | Compatible | Compatible |

- 表级AUTO-INC锁

  AUTO-INC锁的两种实现方式：

  - 采用AUTO-INC锁。在执行插入语句时，加表级AUTO-INC锁，为每条待插入的记录分配值，在该语句执行结束后，释放锁。
  - 采用轻量级锁。在为插入语句生成自增值时，加AUTO-INC锁，生成值后就释放锁，而不用等待整个插入语句执行完。

  通过参数`innodb_autoinc_lock_mode`控制：

  - innodb_autoinc_lock_mode=0，使用AUTO-INC锁。
  - innodb_autoinc_lock_mode=2，使用轻量级锁。
  - innodb_autoinc_lock_mode=1，在插入记录数量确定的情况下使用轻量级锁，插入记录数量不确定时使用AUTO-INC锁。

#### 行级锁

- record lock：记录锁，仅把一条记录锁上
- gap lock：间隙锁，锁一段范围，不锁记录本身
- next-key lock：record lock + gap lock，锁一段范围，并且锁记录本身
- insert intention lock：插入意向锁，在插入一条记录时，需要判断插入位置是否已经有了gap锁，如果有的话，当前插入操作需要等待。但插入意向锁之间不互斥，即一个区间内，多个插入意向锁都可以申请成功。
- 隐式锁：对于新插入的记录，如果没有冲突则不创建锁结构。但当一个事务插入一个记录后，在事务没有结束前，另一个事务想要对该记录进行加锁，则首先会检查该记录的事务id，如果该事务id是活跃的话，此次加锁需要等待，因此第二个事务会为第一个事务创建一个锁结构，然后再为自己创建一个锁结构并等待。

## 事务

### 事务特性

- 原子性：整个数据库事务是不可分割的工作单位。即整个事务中所有操作都成功，整个事务才算成功，一旦有一个操作出现错误，数据库需要回退到执行事务前的状态。

- 一致性：指事务将数据库从一种状态转变为下一种一致性状态，在事务执行前和事务结束后，数据库的一致性约束没有改变。
- 隔离性：要求每个读写事务的对象对其他事务的操作对象能相互分离。

- 持久性：事务一旦提交成功，其结果就是永久性的，即使系统故障，数据库也能将数据恢复。

InnoDB中事务四个特性的实现：

- 原子性：通过undo日志
- 一致性：通过其他三个特性以及数据库本身层面的支持
- 隔离性：锁和MVCC
- 持久性：通过redo日志

### 并发事务的问题

- 脏写：一个事务修改了另一个未提交事务修改过的数据，则发生了脏写

  事务1修改x=1；事务2修改x=2，y=2；事务2提交；事务1修改y=1；事务1提交。最终x=2，y=1。

- 脏读：一个事务读取到了另一个未提交事务修改过的数据，则发生了脏读。或者事务A读取到未提交事务B修改过的数据，然后事务B回滚，那么事务A读取到的是不存在的数据。

- 不可重复读：一个事务修改了另一个未提交事务读取到的数据，则发生了不可重复读。或者事务A读取数据x，事务B修改数据x，并且事务B提交，此时事务A再次读取数据x，会得到修改后的数据。

- 幻读：一个事务先根据某搜索条件查询出一些记录，该事务还未提交时，另一个事务写入了一些符合该搜索条件的记录，则发生了幻读。或者事务A先读取符合条件的记录，然后事务B写入了符合条件的记录，此时事务A再次读取，则会发现两次读取到的数据集是不一样的。

### 事务隔离级别

- READ UNCOMMITTED(读未提交)：一个事务还未提交，其他事务就可以读到该事务修改的数据。
- READ COMMITTED(读已提交)：一个事务提交后，其他事务才能读到该事务修改的数据。
- REPEATABLE READ(可重复读)：一个事务在执行过程中，多次读相同的数据，得到的结果是相同的。
- SERIALIZABLE(可串性化)：对访问的记录加锁，后续的事务对相同记录进行读写操作时，如果发生读写冲突，则后访问的事务需要等待前面的事务结束，才能继续执行。

|          | 脏读   | 不可重复读 | 幻读   |
| :------- | :----- | :--------- | :----- |
| 读未提交 | 可能   | 可能       | 可能   |
| 读已提交 | 不可能 | 可能       | 可能   |
| 可重复读 | 不可能 | 不可能     | 可能   |
| 可串性化 | 不可能 | 不可能     | 不可能 |

### MVCC

**前置知识**

对于聚簇索引记录，都会包含必要的如下两个隐藏列：

- trx_id：一个事务对聚簇索引记录修改后，都会把该事务id赋值给trx_id
- roll_pointer：每次对聚簇索引记录修改后，都会将**旧的版本**写入undo日志，该列相当于一个指针，通过该指针可以找到该记录的旧版本。每对记录进行一次改动，都会记录一条undo日志，每条undo日志也都有roll_pointer，这样该记录的undo日志就通过该字段串成了一个链表，称为版本链。其中版本链的头节点就是该记录的最新修改的值。

ReadView包含如下四个重要字段：

- m_ids：生成readview时，当前系统中活跃的事务id列表
- min_trx_id：生成readview时，当前系统中活跃的事务id的最小值，也即m_ids中的最小值
- max_trx_id：生成readview时，系统分配给下一个事务的事务id值，不是m_ids中的最大值
- creator_trx_id：生成该readview的事务的事务id

**访问数据**

当生成readview后，访问某记录时，流程如下：

- 如果被访问的该记录的trx_id与readview的creator_trx_id相等，说明该记录是当前事务改动的，可以访问
- 如果被访问的该记录的trx_id小于min_trx_id，说明该记录在创建readview前就已经提交，可以访问
- 如果被访问的该记录的trx_id大于等于max_trx_id，说明该记录是在创建readview后才修改的，不可访问
- 如果被访问的该记录的trx_id介于min_trx_id和max_trx_id之间，则检查trx_id是否在m_ids中，如果存在，则说明该记录是当前活跃事务修改的，不可访问；反之则可以访问
- 遇到不可访问的记录，会根据roll_pointer查找链表中下一个记录，直到找到该链表的最后一个记录，如果最后一个记录也不可访问，说明整个该记录不可被读取，因此查询结果中不应该包含该数据

#### 事务隔离级别实现

- READ UNCOMMITTED(读未提交)：直接读取记录的最新版本即可
- READ COMMITTED(读已提交)：在每次读取数据前都生成一个readview
- REPEATABLE READ(可重复读)：在第一次读取数据时生成一个readview
- SERIALIZABLE(可串性化)：使用加锁来访问

## 日志

### redo日志

#### redo日志背景

为了读写效率考虑，InnoDB会将数据页从磁盘缓存到buffer pool中，对于数据页的更新也是修改buffer pool中的页面。但事务要求保证持久性，因此需要将改动持久化，如果我们单纯地将改动的数据页刷回磁盘的话，存在如下缺陷：一个是对于极小的改动也需要刷完整的数据页(默认16KB大小)；二是一个事务包含多个语句，可能涉及多个数据页的改动，这将产生大量的随机IO，效率低。因此使用redo日志记录我们对于数据页的修改，只要将redo日志持久化就可以认为事务持久化了，如果后续系统崩溃可以使用redo日志进行恢复。

#### mini-transaction

将InnoDB对于底层数据页的一次原子操作定义为一个mini-transaction，简称为mtr。一个事务操作可能包含多条语句，一条语句可能需要多个mtr，一个mtr可能包含一组redo日志。一个mtr的操作需要是原子的，即一组redo日志的操作也需要是原子的，比如在恢复时，需要以一组redo日志为单位进行处理，一组redo日志是不可分割的整体。

![1913](https://qraffa-1304595678.cos.ap-guangzhou.myqcloud.com/img/1913.png)

#### redo log buffer

redo日志是以redo日志块(block)的形式组织和存储的。一个块大小为512B，其中头12B和尾4B用于存储管理信息，中间的496B用于存储redo日志记录。

![](https://qraffa-1304595678.cos.ap-guangzhou.myqcloud.com/img/1914.png)

在生成redo日志的时候，也不是直接将redo日志写入磁盘的，而是首先写入到缓存中，称为redo log buffer。在初始化时，数据库会向系统申请一片连续的内存空间作为redo log buffer，默认大小16MB，redo log buffer也就是若干个连续的redo log block组成。

![1916](https://qraffa-1304595678.cos.ap-guangzhou.myqcloud.com/img/1916.png)

向redo log buffer中写入redo日志记录时，是按block顺序写入的，并且一个mtr的一组redo日志是整个写入到redo log buffer中的，而不是一条一条写入的。

#### redo日志文件

**redo日志刷盘**

redo日志除了写入到redo log buffer外，需要持久化到磁盘中，也就是写入到redo日志文件中。触发redo日志从buffer中刷到磁盘有以下几个时机

- log buffer空间不足：当redo log buffer空闲空间小于50%时，会将redo日志刷到磁盘
- 事务提交：为了事务的持久性，需要将redo日志持久化到磁盘，该行为由参数`innodb_flush_log_at_trx_commit`控制
  - innodb_flush_log_at_trx_commit=0：事务提交时，只是将redo日志写入到log buffer中，由后台线程异步的刷到磁盘中
  - innodb_flush_log_at_trx_commit=1：事务提交时，都将redo日志刷到磁盘redo日志文件中，默认为该值
  - innodb_flush_log_at_trx_commit=2：事务提交时，将redo日志写入到redo日志文件中，但不会保证刷到磁盘上，由操作系统来管理刷盘
- 后台线程每秒一次将log buffer刷到磁盘中
- MySQL正常关闭时
- 做checkpoint时

**redo日志文件**

redo日志文件和log buffer一样由大小为512B的块组成，redo日志文件默认大小为48MB，默认情况下会存在两个redo日志文件，两个redo日志文件循环写，也就先写入一个，将一个写满就往下一个写，然后回到第一个。

![1920](https://qraffa-1304595678.cos.ap-guangzhou.myqcloud.com/img/1920.png)

每个redo日志文件的前4个块，用于存储一些管理信息，其中第一个redo日志的前4个块中还要存储checkpoint的相关信息。

#### LSN

定义LSN(log sequence number)为当前总共写入的redo日志量，LSN的起始值为8704。每次向log buffer中写入redo日志记录后，都会响应增加LSN的大小，因此每一个mtr的产生的一组redo日志都有唯一的一个LSN与之对应。

当我们将log buffer刷到磁盘后，我们会响应更新` flushed_to_disk_lsn`的值，表示已经刷到redo日志文件的lsn。

#### checkpoint

由于redo日志文件的大小是有限的，因此当我们将buffer pool中的脏页刷到磁盘后，那么该页的redo日志就可以认为不再需要了，因此可以覆盖写这部分。当我们对数据页做出修改后，会将该页加入到buffer pool的flush链表的头结点中，并且在第一次修改时，会记录修改该页的mtr的**起始LSN**；在之后每次对该页进行修改后，还会记录修改该页的mtr的**结束的LSN**。

比如这里修改了页a，该mtr的起始LSN的8716，该mtr生成的redo日志大小为200，因此结束时的LSN为8916，会记录这两个值。

![1933](https://qraffa-1304595678.cos.ap-guangzhou.myqcloud.com/img/1934.png)

当我们将flush链表的一个脏页刷回磁盘后，我们计算此时系统中最早修改的脏页的oldest_modification，那么小于该LSN值的redo日志都可以被覆盖写入，使用`checkpoint_lsn`来记录该值。比如我们将上面的脏页a刷回到磁盘中，那么此时系统中最早修改的脏页为页c，则checkpoint_lsn=8916，则小于8916值得redo日志都可以被覆盖。我们称上面这样一次过程为一次checkpoint，除了更新checkpoint_lsn外，我们还要更新redo日志文件中的checkpoint信息。

![1938](E:\Blog\innodb\mysql怎样运行\1938.png)

#### redo日志恢复

**恢复起点**

首先我们根据redo日志文件的checkpoint信息得到checkpoint_lsn，因为小于checkpoint_lsn的redo日志相关的脏页都已经被刷到磁盘中了，因此在恢复时我们就不需要考虑这部分了，则我们恢复的起点位置就是checkpoint_lsn对应在redo日志文件中的偏移量。

**恢复终点**

在redo日志文件中，我们是按顺序写入redo日志块的，一个写满写入下一个，因此如果我们遇到一个不满的redo日志块，那么该位置就是redo日志的结束位置，也就是恢复终点。

**恢复**

我们根据redo日志的相关信息，将表空间id和页号相同的redo日志记录，按redo日志生成顺序hash到同一个槽中，以链表形式组织起来，这样恢复时按页的单位将相关的redo日志应用即可，一定程度上避免的随机IO。

### undo日志



