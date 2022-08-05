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



## 日志



