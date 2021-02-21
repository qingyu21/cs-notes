## Database Storage

- Spatial Control：Where to write pages on disk.（The goal is to keep pages that are used together often as physically close together as possible on disk.）
- Temporal Control：When to read pages into memory, and when to write them to disk.（The goal is minimize the number of stalls from having to read data from disk。）

## Disk-Oriented DBMS

![1.PNG](https://i.loli.net/2021/02/21/iaMfBhoGcwlV4Wy.png)

Buffer Pool也叫Buffer Cache，它是由DBMS管理的Memory。

## Buffer Pool Organization

这段内存是完全由DBMS系统来看控制，而不是请求OS来分配这些内存的。这里使用malloc来手动分配内存。

Memory region organized as an array of fixed-size pages，被称为frame（对应于之前使用的slot，frame是buffer pool内存区域中的block  or chunk，我们可以将page放在里面，而slot是我们在page中用来放置tuple的）。

When the DBMS requests a page, an exact copy is placed into one of these frames。

pages可以以任意顺序放在buffer pool中，在此之上，我们需要一个额外的indirection层，通过它就可以知道一个特定的page位于哪一个frame中了。

## Buffer Pool Meta-data

The **page table** keeps track of pages that are currently in memory.

还维护着每个page附加的meta-data：**Dirty Flag**（从disk读取到这个page之后，是否有查询或事务对它进行了修改（其实我们也需要去记录是谁进行了这项修改，通过log实现。我们要确保在先写完log后再去修改page，所以mmap是一个bad idea，因为我们无法保证OS在我们想将page写回磁盘之前不去做这件事）），**Pin/Reference Counter**（引用计数，用来记录希望page保留在memory中的正在运行中的threads或queries的数量。此时它不能够被写出磁盘，这也将阻止我们去移除那些还未被安全写回磁盘的page）（当请求一个不在buffer pool的page时，需要将page table的这个表项上锁Latch，因为同一时间可能由多个线程在运行）

## Locks vs. Latches

Locks是一种high level的logical primitive

- Protects the database's logical contents （tuple，table，database） from other transactions
- Held for transaction duration.
- Need to be able to rollback changes.

Latches是一种low level的protection primitive（在OS中，这里的latches就像mutex）

- Protects the critical sections of the DBMS's internal data structure from other threads
- Held for operation duration
- Do not need to be able to rollback changes

我们要在latch中用到的mutex实现被称为spin lock（自旋锁）

## Page Table vs. Page Directory

page directory是用来通过page ids来找到page在database files中的位置。我们对page directory做出的所有改变必须被持久化，它们必须被写回到磁盘上。（因为如果系统崩溃了，恢复后我们想要知道该在哪里找到我们拥有的page）

page table是内存中的内部映射，它将page id映射到它们在buffer pool中frame的位置。（它不需要在disk上进行备份，它不需要是持久化的，但我们必须确保它是线程安全的）

## Allocation









