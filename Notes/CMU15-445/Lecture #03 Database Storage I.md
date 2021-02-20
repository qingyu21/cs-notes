## Course Outline

一个数据库从上到下总体上的描述：

- Query Planning
- Operator Execution
- Access Methods
- Buffer Pool Manager
- **Disk Manager**

Lecture #03和Lecture #04将介绍如何在文件（或者说是磁盘）上储存数据。

## Disk Oriented Database

DBMS假定primary storage location of the database是在non-volatile disk（非易失性磁盘）上的（在disk上的而不是在memory上）。

（基于这个assumption，我们可以避免出现数据丢失，或出现无效和错误的数据。在任何情况下都要意识到这个assumption）

DBMS管理non-volatile storage和volatile storage之间的数据移动。

## Storage Hierarchy

（从上到下，速度由快到慢，容量由小到大，价格由高到低）

**Volatile**

掉电数据丢失。

Random Access（随机存取，速度很快）、Byte-Addressable（按字节寻址，可直接存取比如某个64bits）

- CPU Registers

- CPU Caches (L1 L2 L3)

- DRAM

**Non-Volatile**

不需要恒定的电源来维持保存的数据。

Sequential Access（顺序存取，存取a bunch of contiguous blocks时比just read random locations效率更高）、Block-Addressable（按block或page寻址，比如想读某64bits可能要先读一个大小为4KB的page）

（在这些存储设备中，我们会尽量读更多的sequential的数据）

- SSD

- HDD

- Network Storage

我们现在关注的是如何从Memory向Disk移动数据。

（这节课的Memory表示DRAM，Disk表示SSD、HDD、Network Storage。而CPU Registers和CPU Caches的内容将会在CMU 15-721中介绍）

（有一种新的存储设备类型被称为Non-Volatile Memory，Byte-Addressable但不会掉电失去数据）

Random access on non-volatile storage is usually much slower than sequential access.

**sequential vs. random access**

DBMS will want to maximize sequential access.

- Algorithms try to reduce number of writes to random pages so that data is stored in contiguous blocks.
- Allocating multiple pages at the same time is called an extent.

## System Design Goals

给予应用程序如此的假象：我们可以保存超过memory大小的数据。

减少对disk的读写开销。

maximize sequential access.

## Disk-Oriented DBMS

![1.PNG](https://i.loli.net/2021/02/20/Cgq6l4AxVMjO9DQ.png)

看起来很像Virtual Memory

## Why Not Use The OS?

mmap：可以让OS将file pages映射到a process' address space。然后可以read或write这些memory locations，最后可以异步写回磁盘。

本质上是放弃了数据在Memory和Disk之间来回移动的控制权，而是让OS来帮我们管理。

![2.PNG](https://i.loli.net/2021/02/20/wsM3VBhWX1zgPkN.png)

内存满了之后，不得不让DBMS停止request this page的线程。因为磁盘调度程序要从磁盘中拿到这个page，并将其放到内存中。

数据库可能要去访问某些没在内存中的东西，所以可以将它交给另一条线程去做，它会停下来，但数据库不会。

如果仅仅是read，就可以通过一系列系统调用来减少我们遇到的一些问题，但当要write时，OS不知道某些pages必须要在其他pages之前先从内存写回disk。OS不知道写回操作是不是OK的。

但有一些办法：

- **madvise**: Tell the OS how you expect to read certain pages. （sequential or random）
- **mlock**: Tell the OS that memory ranges cannot be paged out. （阻止pages被回收）
- **msync**: Tell the OS to flush memory ranges out to disk.

DBMS (almost) always wants to control things itself and can do a better job at it.

- Flushing dirty pages to disk in the correct order.

- Specialized prefetching.

- Buffer replacement policy.

- Thread/process scheduling.

## File Storage

DBMS用一个或多个文件来在disk上保存数据，而且是专有的格式的（不能跨软件通用）。我们使用文件系统提供的API来读写。

早期系统会在raw storage上用custom filesystems，但现在大多数系统已经不会用了。因为得不偿失。

## Storage Manager

也被称为Storage Engine。

它用来维护disk上的数据。一些系统会有自己的读写调度，以改善pages的空间和时间局部性。

它会将这些文件组织成pages的一个集合。它会记录对pages的所有读写。会记录可用的空间来保存新的数据。

一个page本质上就是一个fix-sized（固定大小的）数据块。它可以存储任何东西（比如tuples，meta-data，indexes，log records等），但同一个page不能保存不同type的数据。一些系统会要求一个page是self-contained的，因为关于该page内容的meta-data保存在自身中。

DBMS会为每个独一无二的identifier（page ID）。DBMS使用一个indirection layer来实现page ids到实际位置的映射。

## Database Pages

有三种pages：

- Hardware Page (usually 4KB)
- OS Page (usually 4KB)
- Database Page (512B-16KB)

存储设备以hardware page的大小确保了原子写操作，这意味着如果database page比hardware page大，那么DBMS必须采取额外的措施来确保pages被安全地写入。

不同的DBMS有三种在位于disk的files中管理pages的方法：

- **Heap Files Organization**
- Sequential/Sorted File Organization
- Hashing File Organization

## Database Heap

一个heap file是一个pages的无序集合，tuples在其中是以随机的顺序保存的。（创建，获取，写，删page。）（需要支持遍历所有pages）。如果只有一个heap file，那么找到pages很容易。如果有多个heap file，需要meta-data来记录每个file中保存了哪些pages，还需要记录哪些有free space。

两种表示heap files的办法：Linked List和Page Directory（better）

## Heap File：Linked List

在文件的开始保存一个header page，它保存两个指针：

- HEAD of the free page list.
- HEAD of the data page list.

![3.PNG](https://i.loli.net/2021/02/20/5bh4fkdAiyEL6vI.png)

每个page都记录着它们有多少free slots。

如果DBMS要寻找一个特定的page，必须在data page list上做sequential scan，直到找到它。

## Heap File：Page Directory

DBMS维护着特定的pages，它们记录了data pages在database files中的位置，以及每一个page中free space的大小。

![4.PNG](https://i.loli.net/2021/02/21/brdGBy3Y8UFMRzV.png)

DBMS必须确保directory pages与data pages一致。

## Page Header

page=header+data

header：page data的metadata，比如Page Size，Checksum，DBMS Version，Transaction Visibility，Compression Information等。

一些系统要求page是self-contained的。

## Page Layout

在page中组织数据的方式：（假设只保存tuples）

- Tuple-oriented
- Log-structured

## Tuple Storage

一种不太可行的方案：记录这个page已经保存了多少tuple，在尾部添加新的tuple。但当tuples被删除时，或遇到了有可变长度attributes的tuples时会出现问题。

![5.PNG](https://i.loli.net/2021/02/21/7nfVbFNuZToOzIA.png)

## Slotted Pages

 slot array将slots映射到tuples的开始位置

![6.PNG](https://i.loli.net/2021/02/21/zx2Yh4TnEokB1Ip.png)

Header记录了the number of used slots，最后使用的slot的位置，and a slot array。

## Record IDs

每一个tuple被分配了一个独一无二的record identifier。Most common: **page_id** + **offset/slot**。Can also contain file location info。

## Tuple Layouts

一个tuple本质上是一串bytes，由DBMS来将这串bytes解释为attribute的types和values。

## Tuple Header

Tuple=Header+Attribute Data

Header（包括了meta-data）：Visibility info (concurrency control)，Bit Map for **NULL** values。

## Tuple Data

attributes（典型的保存顺序）是以建表时的顺序保存的。（尽管不是必须的，但是This is done for software engineering 

reasons）

## Denormalized Tuple Data

Can physically **denormalize** (e.g., "pre join") related tuples and store them together in the same page.

- Potentially reduces the amount of I/O for common workload patterns.

- Can make updates more expensive.

## Conclusion

- Database是以pages的形式组织的
- 记录pages的不同方法
- 保存pages的不同方法
- 保存tuples的不同方法

