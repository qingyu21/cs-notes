[TOC]

## B+Tree Design Choices

### Node Size

（一般来说可以把node想象成一个page）

存储媒介不同，最优的node size也不同（通常存储设备的速度越慢，最优的node size越大）：

- hard drives通常是MB级的：减少寻找数据的次数，分摊昂贵的读磁盘时间（通过a large chunk of data）
- in-memrory database可能会使用小至512B的page size，为了使整个page适应CPU cache的大小，也为了减小数据碎片。
- SSD：约10KB

（可以有两个buffer pool，一个用于index pages，一个用于data pages，pages的size可以者只为不同）

也会依赖于workload的类型：

- point queries会更偏向尽量小的page，来减少被载入的不必要的额外信息。（root-to-leaf traversals）
- large sequential scan可能会偏向大的page，来减小需要做的读取次数。（如leaf node scans）

### Merge Threshold

B+Tree规定过删除后合并下边不合规的node，但有时暂时违反这个规则更好，可以减少删除操作的数量。

（并不总是合并node，当它小于half full的。延迟merge操作可能减少reorganization的量）

例如，急切的merge操作会导致崩溃（大量连续的delete和insert操作会导致不断的merge和split）

（可以让多余的node暂时存在，然后定期去rebuild整个树）

可以批量合并（一次完成多个合并操作），这样可以减少必须在树上执行昂贵的write latches的时间。

### Variable Length Keys

- Pointers：不直接保存keys，而是保存一个指向这个key的pointer。（这种方法是低效的，因为必须要为每个key去chase a pointer（比如当我们进行traveral时）。唯一一种在生产环境中使用到它的场景是嵌入式设备（需要节约空间））
- Variable  Length Nodes：在node中保存keys，但允许可变长度的nodes。这是不可行的，因为管理可变长度node的内存管理开销很大。
- Padding：将所有key的大小扩大到最大key的大小。（大多数情况下，这是一种巨大的内存浪费。所以也没人用）
- Key Map/ Indirection：（几乎所有人都在用）将keys替换为an index to the key-value pair in a separate dictionary。（这样可以节约大量空间，而且可能会缩短点查询（因为index指向的key-value对和leaf nodes指向的是同一个））由于dictionary index的大小较小，所以有足够的空间在index旁边放置每个key的前缀，从而潜在地允许某些index searching和leaf scanning甚至不必chase the pointer（如果prefix和search key完全不同）

### Intra-Node Search

一旦我们到达了一个node，我们仍然需要在这个node中进行搜索（通过一个inner node寻找下一个node，或在一个leaf node中找到我们的key value）这过程相对来说很简单，但是有一些tradeoffs：

- Linear：最简单的方法是在node中遍历每个key直到找到我们的key。一方面这样不用保持keys有序，insertions和deletions也就更快。另一方面，这种方法相对来说较为低效，每次搜索的时间复杂度为$O(n)$
- Binary：效率更高的搜索方法是先保持所有node有序，然后进行binary seach。每次搜索的时间复杂度为$O(ln(n)$。然而，insertions变得更为昂贵，因为每个node必须保持有序。
- Interpolation：这种方法利用了关于这个node的所有metadata（如max element，min element，average等），然后利用它去产生一个该key近似的位置。（比如我们要找$8$，我们知道$10$是最大的key，$10-(n+1)$是最小的key，然后我们就知道了从最大key开始向下搜索$2$个slots，因为在这种情况下，距离最大key仅一个slot的key必须为9）（尽管这种速度最快，但是只出现在academic database中，因为应用性受限（keys要有特定的属性）而且很复杂）

## Optimizations

### Prefix Compression

（比suffix truncation更加common）

同一个node内的keys大多有prefix的部分重叠。相比于把这个prefix在每个key中保存多次，我们可以仅仅在node开始保存一次，而每个slot中保存的是独一无二的部分。

### Deduplication

（去重）

在index允许non-unique keys的情况下，我们可能会得到包含the same key over and over with different values attached的leaf nodes。

一个优化方法是只写这个key一次，然后在它后面加上它的所有相关的value。

### Suffix Truncation

大多数情况下，inner node中的key entry只是用作路标，而不是用于他们的实际key values（即使一个key存在于index中，我们仍然需要搜索到底部，来确保它没有被删除）。

我们可以利用这一点，对于在一个给定的inner node中的每个key，我们可以只保存the minimum differentiating prefix。

虽然可以小到只保存单个不同的character/digit，但在末尾留下一些冗余的数字是有益的（这样可以减小由于an identical prefix will occur而发生的an indeterminable insertion的可能性）。这种情况下，必须搜索到树的底部来找到完整的key（或者在删除key的情况下，取决于依赖的leaf node的最小或最大的key）

### Bulk Insert

（大多数情况下，我们是提前拥有所有keys的，所以不必去一个一个的insert。与其从上到下建立，不如从下到上建立）

一个B+Tree刚建立时，以通常的方式插入keys会导致不断的split操作。

由于我们已经给出leave nodes的sibling pointers，如果我们构造一个leaf nodes的sorted linked list，然后使用每个leave nodes的第一个key从低到上轻松地构建index，那么初始插入数据的效率会高得多。

注意，根据我们的上下文，我们可能希望尽可能地紧密打包leaves来节约空间，或者在每个leave中留出空间，以便在split之前有更多的insert。

### Pointer Swizzling

（very common）

因为B+Tree的每个node都保存在buffer pool的一个page中，所以每次加载一个新的page时，我们都需要从buffer pool中取出它，这需要进行latching和lookups。

要完全跳过这一步，我们可以保存实际的raw pointers来代替page IDs（也就是swizzling），从而完全防止了buffer pool的获取。

在正常遍历index时，与其手动获取整个树并手动放置pointers，我们可以简单地保存page查找的结果pointer。

注意，我们必须跟踪哪些pointers被swizzled，并在它们所指向的page被解除锁定和破坏时将它们deswizzle为原来的page ids。

## Additional Index Usage

### Implicit Indexes

如果创建了一个primary key或者一个unique constraint，大多数DBMS将会自动创建一个implicit index来强制完整性约束，而referential constraints不会做这些。

### Partial Indexes

在许多情况下，用户可能不需要一个index服务于所有table中的所有tuple，而可能只希望它服务于其中的一个子集。

partial index是一种具有some condition或者“where clause”的index，这样它就包含了所有tuple的一个subset。

这可能会减少维护表的开销，并避免了不必要的数据污染buffer pool。

### Covering Indexes

（也叫 index-only scans）

如果处理query需要的所有attributes都在index中可得。DBMS不必去取出这个tuple，DBMS可以直接基于从index中得到的数据来完成整个query。

换句话说，covering indexes只用于在表中定位data records，而不是返回数据。（这减少了对buffer pool资源的竞争）

### Index Include Columns

它允许用户在索引中插入额外的列，以支持仅限索引的查询。

这些额外的列存储在叶节点中，实际上并不是search key的一部分。

### Function/Expression Indexes

indexes也可以被用来在functions或expressions上创建。

一个index不需要以它们在base table的方式保存keys。

相反，它可以将functions或expressions的输出保存为key，而不是原始值。

由DBMS来识别哪些query可以使用该索引。

## Trie Index

（也叫Digital Search Tree，Prefix Tree）

B+Tree中的inner node keys无法用来判断一个key是否存在于index中，每次寻找一个key，你必须遍历leaf node。

Trie Index允许我们在树的顶部知道是否存在一个key。Trie Index使用了key的digital representation来逐个检查前缀，而不是比较整个key。

它有许多有用的属性：

- 形状只取决于key space和长度，不需要再平衡操作。
- 所有的操作复杂度都是$O(k)$，$k$是key的长度（到leaf node的路径表示leaf的key）
- 一个trie level的跨度是每个partial key/digit所代表的bits数。

最简单的Tire：1-bit Span Trie，这个1可以改为2，8，16等

## Radix Tree

（也被称为Patricia Tree）

它是Trie的一种变体。与Trie index不同的是，只有一个child的node被省略。在key不同之前，nodes被合并以表示最大的前缀。

Radix tree会产生误报，所以DBMS必须总是检查original tuple来检查key是否matches。

Radix tree的高度取决于keys的长度，而不是像B+Tree那样是key的数量。到leaf node的路径表示leaf node的key。并非所有attribute types可以被分解为一个radix tree的binary comparable digits。

## @Inverted Indexes

它保存了一个从words到records的映射，这个映射包含了那些target attribute中的words。这有时被称为full-text search indexes。它对关键字搜索特别有用。

大多数的DBMS本身都支持它，但也有专门的DBMS，其中inverted indexes是唯一可用的table index data structure。

### Query Types

inverted indexes允许用户执行三种不能在B+树上执行的查询。

- 首先，倒排索引允许短语搜索（phrase searches），它定位records that contain a list of words in the given order
- 它们还允许近距离搜索（proximity searches），记录两个单词在n个单词中出现的位置。
- 允许通配符搜索（wildcard searches），它可以找到records that contain words that match some pattern (e.g., regular expression)

### Design Decisions

两个主要的design decisions是what to store和when to update。

- Inverted indexes需要至少能够存储the words contained in each record (由标点字符分隔).They may also include additional information such as the word frequency, position, and other meta-data.
- Updating an inverted index every time the table is modified is expensive and slow.Because of this, nverted indexes usually maintain auxiliary data structures to stage updates and then update the index in batches.