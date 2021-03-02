[TOC]

## 课程大纲

关系型数据库，存储，Execution，并发控制，恢复，分布式数据库，Potpourri

一个数据库从上到下总体上的描述：

- Query Planning
- Operator Execution
- Access Methods
- Buffer Pool Manager
- **Disk Manager**

Lecture #03和Lecture #04将介绍如何在文件（或者说是磁盘）上储存数据。

## Disk-Oriented Database

DBMS假定primary storage location of the database是在**non-volatile disk**（**非易失性磁盘**）上的（在disk上的而不是在memory上）。

（基于这个assumption，我们可以避免出现数据丢失，或出现无效和错误的数据。在任何情况下都要意识到这个assumption）

DBMS管理non-volatile storage和volatile storage之间的数据移动，因为它无法直接在disk中操纵数据。

## 存储器层次体系

（从上到下，速度由快到慢，容量由小到大，价格由高到低）

### Volatile Devices

掉电数据丢失。

Random Access（随机存取，速度很快）、Byte-Addressable（按字节寻址，可直接存取比如某个64bits）

- CPU Registers

- CPU Caches (L1 L2 L3)

- DRAM

### Non-Volatile Devices

不需要恒定的电源来维持保存的数据。

Sequential Access（利于顺序存取，存取multiple contiguous chunks of data at the same time时比just read random locations效率更高）、Block/Page Addressable（按block或page寻址，比如想读某64bits可能要先读一个大小为4KB的page）

（在这些存储设备中，我们会尽量读更多的sequential的数据）

- SSD

- HDD

- Network Storage

我们现在关注的是如何从Memory向Disk移动数据。

（这节课的**Memory**表示DRAM，**Disk**表示SSD、HDD、Network Storage。而CPU Registers和CPU Caches的内容将会在CMU 15-721中介绍）

### persistant memory

有一种新的存储设备类型被称为**Non-Volatile Memory**（非易失性内存NVM），Byte-Addressable但不会掉电失去数据）。几乎和DRAM同样快，但和disk有着同样的persistence，这节课不涉及它。

我们关注于如何减小访问disk的延迟，而不去关注如何用registers或caches去优化。不同的存储器速度天差地别。

**sequential vs. random access**

在non-volatile storage上的随机存取比顺序存取慢得多。

DBMS希望将sequential access做到最多

- 算法尽量减小往random pages写数据的量，所以数据是在conitguous blocks中存储的。

- Allocating multiple pages at the same time is called an extent.（extent即区，它是由连续的pages组成的空间）

## 系统设计的目标

- 给予应用程序如此的假象：我们可以保存超过memory大小的数据。

- 因为读写disk的代价是昂贵的，所以要认真设法去避免大的延迟和performance degradation。

- maximize sequential access.

## Disk-Oriented DBMS

每个数据库文件中的数据被组织成了pages，第一个page是directory page。

为了操纵数据，需要把数据转移到memory（通过buffer pool来实现disk和memory之间的移动）。

execution engine用于执行查询。它会向buffer pool请求一个特定的page，然后buffer pool会处理将此page转移到memory的过程，之后会将一个指向此page的指针传递给execution engine。

当executino engine在memory的部分区域工作时，buffer pool manager会确保page是在那里的。

![1.PNG](https://i.loli.net/2021/02/20/Cgq6l4AxVMjO9DQ.png)

看起来很像Virtual Memory

## 为什么不使用OS来进行memory和disk之间的数据转移?

**mmap**（memory mapping）：它可以让OS将file pages映射到一个进程的address space。然后可以read或write这些memory locations，最后可以异步写回磁盘。

这本质上是放弃了数据在Memory和Disk之间来回移动的控制权，而是让OS来帮我们管理。（如果mmap遇到了page fault，进程会被阻塞）

![2.PNG](https://i.loli.net/2021/02/20/wsM3VBhWX1zgPkN.png)

内存满了之后，不得不让DBMS停止request this page的线程。因为磁盘调度程序要从磁盘中拿到这个page，并将其放到内存中。

数据库可能要去访问某些没在内存中的东西，所以可以将它交给另一条线程去做，它会停下来，但数据库不会。（我们希望当等待数据从disk转移到memory时，DBMS可以去处理其它的查询）

### 存在的问题

- 当多个线程去访问mmap files时，保证不出现page fault stalls是困难的。

- 如果仅仅是read，就可以通过一系列系统调用来减少我们遇到的一些问题，但当要write时，OS不知道某些pages必须要在其他pages之前先从内存写回disk。OS不知道写回操作是不是OK的。有多个writes时十分复杂。

当需要write时，永远不要在DBMS中使用mmap。（for correctness and performance reasions）

但有一些办法：

- **madvise**: Tell the OS how you expect to read certain pages. （告诉OS何时计划去读特定的pages）
- **mlock**: Tell the OS that memory ranges cannot be paged out. （阻止pages被回收）
- **msync**: Tell the OS to flush memory ranges out to disk.

DBMS (almost) always希望自己来执行这些过程，而且会做得更好（因为它更了解正在被访问的数据和正在被处理的查询）

- Flushing dirty pages to disk in the correct order.
- Specialized prefetching.
- Buffer replacement policy.
- Thread/process scheduling.

不要用OS来进行数据转移。

## Storage

### File Storage

DBMS用一个或多个文件来在disk上保存数据，而且是专有的格式的（不能跨软件通用）。我们使用文件系统提供的API来读写。

早期系统会在raw storage上用custom filesystems，但现在大多数系统已经不会用了。因为得不偿失。

#### Storage Manager

也被称为Storage Engine。

**Storage Manager**用来维护一个数据库的文件。（一些系统会有自己的读写调度，以改善pages的空间和时间局部性。）

它会将这些文件组织成pages的一个集合。它会记录对pages的所有读写。会记录可用的空间来保存新的数据。

### Database Pages

一个**page**本质上就是一个fix-sized（固定大小的）数据块。它可以存储任何东西（比如tuples，meta-data，indexes，log records等），但同一个page不能保存不同type的数据。一些系统会要求一个page是self-contained的，因为关于该page内容的meta-data保存在自身中。

DBMS会给每个page独一无二的identifier（page ID）。DBMS使用一个indirection layer来实现page ids到实际位置的映射。

有三种pages：

- Hardware Page (usually 4KB)
- OS Page (usually 4KB)
- Database Page (512B-16KB)

存储设备以hardware page的大小确保了原子写操作，这意味着如果database page比hardware page大，那么DBMS必须采取额外的措施来确保pages被安全地写入。

不同的DBMS有三种在位于disk的文件中管理pages的方法（如何在disk中找到一个特定的page）：

- **Heap Files Organization**
- Sequential/Sorted File Organization
- Hashing File Organization

### Database Heap

一个heap file是一个pages的无序集合，tuples在其中是以随机的顺序保存的。（创建，获取，写，删page。）（需要支持遍历所有pages）。

（DBMS可以通过page_id来定位一个特定的page）如果只有一个heap file，那么找到pages很容易。如果有多个heap file，需要meta-data来记录每个file中保存了哪些pages，还需要记录哪些有free space。

两种表示heap files的办法：Linked List和Page Directory（better）

##### Heap File：Linked List

在文件的开始保存一个header page，它保存两个指针：

- HEAD of the free page list.
- HEAD of the data page list.

![3.PNG](https://i.loli.net/2021/02/20/5bh4fkdAiyEL6vI.png)

每个page都记录着它们有多少free slots。

如果DBMS要寻找一个特定的page，必须在data page list上做sequential scan，直到找到它。

##### Heap File：Page Directory

DBMS维护着特殊的pages，它们记录了data pages在database files中的位置，以及每一个page中free space的大小。

![4.PNG](https://i.loli.net/2021/02/21/brdGBy3Y8UFMRzV.png)

DBMS必须确保directory pages与data pages一致。

### Page Header

**page=header+data**

header：page data的metadata，比如Page Size，Checksum，DBMS Version，Transaction Visibility，Compression Information，self-containment等。

### Page Layout

在page中组织数据的方式：（假设只保存tuples）

- Tuple-oriented
- Slotted-pages
- Log-structured

#### Tuple Storage

一种不太可行的方案：记录这个page已经保存了多少tuple，在尾部添加新的tuple。但当tuples被删除时，或遇到了有可变长度attributes的tuples时会出现问题。

![5.PNG](https://i.loli.net/2021/02/21/7nfVbFNuZToOzIA.png)

#### Slotted Pages

（最常见的方式）

 slot array将slots映射到tuples的开始位置（map slots to offsets）

![6.PNG](https://i.loli.net/2021/02/21/zx2Yh4TnEokB1Ip.png)

Header记录了被用过的slots的数量，最后被使用的slot的开始位置的offset，和一个slot array（它记录了每个tuple的开始位置）。

如果要添加一个tuple，slot array会从前向后增加，tuples会从后向前增加。当不足以容纳一个tuple数据时，认为这个page是满的。

#### Log-structured

不直接存typles，只存log记录（database以何种方式被更改过（insert，delete，update））。

读一个记录：从后向前扫描log文件，重建这个tuple。

写入速度快，潜在的慢速读写。在append-only storage上工作良好（无法返回并更新这些数据）。

为了避免长时间的读，DBMS可以用index来跳转到log中的特定位置。还可以周期性地compact这些log（后果是write amplification：不断重复写入相同的数据）。

#### Record IDs

（例如ROWID，CTID）

每一个tuple被分配了一个独一无二的record identifier。Most common: **page_id** + **offset/slot**。Can also contain file location info。

### Tuple Layouts

一个tuple本质上是一串bytes，由DBMS来将这串bytes解释为attribute的types和values。

#### Tuple Header

Tuple=Header+Attribute Data

Header（包括了tuple的meta-data）：Visibility information for concurrency control（哪些transaction创建/更改了这个tuple），Bit Map for NULL values。

它不需要去存schema的meta-data。

#### Tuple Data

actual data for attributes

attributes（典型的保存顺序）是以建表时的顺序保存的。（尽管不是必须的，但是This is done for software engineering reasons）

#### Unique Identifier

数据库中的每一个tuple都被分配了一个独一无二的identifier

最常见：page_id+(offset or slot)

应用程序无法用这些id来表示任何东西。

#### Denormalized Tuple Data

如果两个表是related的，DBMS可以pre-join他们，所以最后得到了一个相同的表。

- 读取更快（因为DBMS只用load一个page而不是两个分开的page）
- 然而这使更新的代价更大，因为DBMS需要更多的空间来保存每个tuple

Can physically **denormalize** (e.g., "pre join") related tuples and store them together in the same page.

- Potentially reduces the amount of I/O for common workload patterns.
- Can make updates more expensive.

Not a new idea.

- IBM System R did this in the 1970s.

- Several NoSQL DBMSs do this without calling it physical denormalization.

## Conclusion

- Database是以pages的形式组织的
- 记录pages的不同方法
- 保存pages的不同方法
- 保存tuples的不同方法

