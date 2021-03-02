[TOC]

## Database Storage

需要高效地将数据在disk和memory之间来回移动。

- Spatial Control（空间控制）：指pages在disk被写的物理位置（目标是保持经常被一起使用的pages在disk上尽可能在物理上彼此靠近）
- Temporal Control（时间控制）：指何时将pages读到memory，何时将pages写回disk（目的是最大程度地减少必须从disk读取数据的stalls。 ）

## Locks vs. Latches

### Locks

Locks是一种high level的logical primitive

- 保护数据库的logical contents（例如tuples，tables，databases） from other transactions
- Transactions will hold a lock for its entire duration. 
- Database systems can expose to the user which locks are being held as queries are run。
- Need to be able to rollback changes.

### Latches

Latches是一种low level的protection primitive（在OS中，这里的latches就像mutex）

- 保护数据库的critical sections of the DBMS's internal data structure from other threads（例如hash tables，regions of memory）
- Latches are held for only the duration of the operation being made.
- Do not need to be able to rollback changes

我们要在latch中用到的mutex实现被称为spin lock（自旋锁）

## Buffer Pool Manager

### Disk-Oriented DBMS

![1.PNG](https://i.loli.net/2021/02/21/iaMfBhoGcwlV4Wy.png)

Buffer Pool也叫Buffer Cache，它是由DBMS管理的Memory。

### Buffer Pool Organization

这段内存是完全由DBMS系统来看制，而不是请求OS来分配这些内存的。这里使用malloc来手动分配内存。

Memory region organized as an array of fixed-size pages，被称为**frame**（对应于之前使用的slot，frame是buffer pool内存区域中的block  or chunk，我们可以将page放在里面，而slot是我们在page中用来放置tuple的）。

When the DBMS requests a page, an exact copy is placed into one of these frames。

pages可以以任意顺序放在buffer pool中，在此之上，我们需要一个额外的indirection层，通过它就可以知道一个特定的page位于哪一个frame中了。

### Buffer Pool Meta-data

**page table** 记录着此时正在memory中的pages.（它是一个in-memory的hash table）它将page_id映射到buffer pool中的frame location。（不要与page directory混淆，page directory将page_id映射到database文件中的page的位置）

还维护着每个page附加的meta-data：

- **Dirty Flag**（从disk读取到这个page之后，是否有查询或事务对它进行了修改，（如果被修改过，之后要将其写回disk）（其实我们也需要去记录是谁进行了这项修改，通过log实现。我们要确保在先写完log后再去修改page，所以mmap是一个bad idea，因为我们无法保证OS在我们想将page写回磁盘之前不去做这件事））
- **Pin/Reference Counter**（引用计数，用来记录希望page保留在memory中的正在运行中的threads或queries的数量（一个线程访问它之前需要增加这个counter）。如果counter大于0，此时它不能够被写出磁盘，这也将阻止我们去移除那些还未被安全写回磁盘的page）（当请求一个不在buffer pool的page时，需要将page table的这个表项上锁Latch，因为同一时间可能由多个线程在运行）

#### Page Table vs. Page Directory

**page directory**是用来通过page ids来找到page在database files中的位置。我们对page directory做出的所有改变必须被持久化，它们必须被写回到磁盘上。（因为如果系统崩溃了，恢复后我们想要知道该在哪里找到我们拥有的page）

**page table**是内存中的内部映射，它将page id映射到它们在buffer pool中frame的位置。（它不需要在disk上进行备份，它不需要是持久化的，但我们必须确保它是线程安全的）

### Memory Allocation Policies

- **Global Policies:**Make decisions for all active transactions.
- **Local Policies:**（makes decisions that will make a single query or transaction run faster, even if it isn’t good for the entire workload. ）Allocate frames to a specific transaction without considering the behavior of concurrent transactions，Still need to support sharing pages。

大多数系统结合使用这二者。

## Buffer Pool Optimizations

### Multiple Buffer Pools

（for different purposes）

DBMS并不总是只有一个对应于整个系统的buffer pool。（per-database buffer pool，per-page type buffer pool）每个buffer pool可以有local policy。（caching policy，placement policy。根据workload来判断）

当不同的threads想要尝试访问buffer pool时，减少latch contention（因为有了multiple page tables）。

增加了locality

将desired pages映射到一个buffer pool有两种方法：

- **Approach #1: Object Ids**

  将一个object identifier与record ids集成到一起。维护一个从object到特定buffer pool的映射。

![2.PNG](https://i.loli.net/2021/02/22/3FnsV8mEqlIcD4X.png)

- **Approach #2: Hashing**

  通过hash来用page id选择buffer pool

![3.PNG](https://i.loli.net/2021/02/22/A5yh9lbuPVMB8Gf.png)

### Pre-Fetching

（由于第一批pages正在被处理，第二批pages可以被pre-fetching进buffer pool，通常当sequential访问许多pages时被使用）

减少DBMS由于要去disk读取数据而产生的stalls。

基于特定的query plan（Sequential Scans，Index Scans）可以做prefetch。

### Scan Sharing

也叫 **synchronized scans**

queries可以重复使用data（from storage，from operator computation）（与result caching 不同，）

需要当scan一个table时，多个query可以依附于一个单独的cursor。（query不必是同一个，而且可以共享中间结果）

当一个query需要scan而另一个query正在做scan，DBMS可能会将一个query的cursor依附于另一个query的cursor

DBMS记录着第二个query与第一个query的连接位置，以便在到达数据结构末尾时可以完成扫描。

### Buffer Pool Bypass

The sequential scan operator will not store fetched pages in the buffer pool to avoid overhead.

→ Memory is local to running query.

→ Works well if operator needs to read a large sequence of pages that are contiguous on disk.

→ Can also be used for temporary data (sorting, joins)

## OS Page Cache

大多数磁盘操作通过OS API来进行。

Unless you tell it not to, the OS maintains its own filesystem cache.

Most DBMSs use direct I/O (**O_DIRECT**)to bypass the OS's cache.

- avoid redundant copies of pages.

- avoid having to manage different eviction policies

## Buffer Replacement Policies

Goals：

- Correctness：如果某个数据没被真正的用完，那么就不能将它写出或移除
- Accuracy：确保移除的page是未来不太会被用到的那些page
- Speed：不希望寻找到的时间甚至比读取page的时间还长
- Meta-data overhead：大量meta-data会带来开销，不希望page的meta-data比page本身还要大

### Least-Recently Used（LRU）

维护一个page最后一次被访问时的timestamp（可以被保存到一个单独的数据结构（比如queue）来允许比如排序和提高效率）。

当DBMS需要去替换时，去替换那个拥有最老timestamp的page。（为了减少寻找时间，可以将page按顺序的排列被保持）

### Clock

LRU的近似算法（它无需追踪每一个page分别的timestamp），每个page有一个reference bit（当一个page被访问了，就将其置1）.

将pages组织为一个带有一个clock hand的环形的buffer，然后有一个可以旋转的指针，它能够检查这个reference bit，如果是1，将其置0，如果是0，就移除它。

（之所以说它是近似算法，是因为不会去精确地移除最近最少使用的那个page了。直观的来讲，如果这些page在一定时间内没被使用，那么它最近都不会再被使用）

（它在某些简单的情况下效果非常好，比如在进行point query时访问单个东西）

但CLock和LRU都容易受到sequential flooding的影响。

### Alternatives

#### Problems

sequential flooding：当一个query进行sequential scan时，它会访问每个单独的page，这可能污染我们的page cache（因为这些pages只读一次，之后再也不读）。而the most recently used page实际上是那些最不需要的page。

实际上希望移除的是那些最近被使用的，而不是那些最近最少被使用的。

可以使用multiple buffer pool，不同的buffer pool使用不同的替换策略。

#### Better Policies：LRU-K

Track the history of last *K* references to each page as timestamps and compute the interval between subsequent accesses.

The DBMS then uses this history to estimate the next time that page is going to be accessed.

#### Better Policies：Localization

The DBMS chooses which pages to evict on a per txn/query basis. This minimizes the pollution of the buffer pool from each query.

→ Keep track of the pages that a query has accessed.

Example: Postgres maintains a small ring buffer that is private to the query

#### Better Policies： Priority Hints

The DBMS knows what the context of each page during query execution.

It can provide hints to the buffer pool on whether a page is important or not.

### Dirty Page

dirty bit：表示自从一个page放入buffer pool后，它是否被修改过。

**FAST:** If a page in the buffer pool is not dirty, then the DBMS can simply "drop" it.

**SLOW:** If a page is dirty, then the DBMS must write back to disk to ensure that its changes are persisted.

Trade-off between fast evictions versus dirty writing pages that will not be read again in the future.

#### Backgound Writing

The DBMS can periodically walk through the page table and write dirty pages to disk.

When a dirty page is safely written, the DBMS can either evict the page or just unset the dirty flag.

Need to be careful that we don’t write dirty pages before their log records have been written…

## Other Memory Pools

The DBMS needs memory for things other than just tuples and indexes.

These other memory pools may not always backedby disk. Depends on implementation.

→ Sorting + Join Buffers

→ Query Caches

→ Maintenance Buffers

→ Log Buffers

→ Dictionary Caches

## Conclusion

The DBMS can manage that sweet, sweet memory better than the OS.

Leverage the semantics about the query plan to make better decisions:

→ Evictions

→ Allocations

→ Pre-fetching

