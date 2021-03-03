[TOC]

## Table Indexes

（hash table用来做index是insufficient的，因为只能做单表查询）

**table index**是表中一部分列的复制品（which is organized and/or sorted），可以提高那些用到这些列的查询的效率。

DBMS确保了表的内容和indexes始终在逻辑上是一致的（logically in sync）。

trade-off：每个database创建的indexes的数目的多少。

- 更多indexes使查询更快，但同样占用了存储，还需要被维护。

由DBMS来找出并使用最适合的索引来执行查询。

## B+ Tree

### B-Tree Family

B-Tree，B+Tree，B*Tree，B-Link Tree

B-Tree和B+Tree的最主要区别：

- B-Tree在所有nodes中保存所有的keys和values（占用空间小但是多线程情况下update非常昂贵）
- B+Tree只在leaf nodes中保存values，inner nodes只用于指导搜索过程（update时，只更改leaf nodes）

现代的B+Tree实现结合了其它B-Tree变体的特点（比如B-Link Tree中使用的sibling pointers）

### B+Tree

**B+Tree**是一种自平衡树，它能够保持data是sorted的。

seach，sequential access，insertino，deletion都在O(log n)的时间复杂度完成。

- 相比于二叉搜索树，B+Tree的一个node可以拥有超过2个的children node。
- 可用于优化disk-oriented DBMS中读或写large blocks of data的操作。

### B+Tree Properties

形式上，B+Tree是一个**M-way搜索树**，拥有以下性质：

- 它是完全平衡的（perfectly balanced）：每个leaf node在树中都拥有相同的深度。
- 每个node（除了root node），是至少half-full的：$M/2-1\le \#keys\le M-1$
- 每个带有$k$个keys的inner node都有$k+1$个non-null children

### Nodes

每个B+Tree node都是由一个键值对的array构成的

- keys源于index基于的那些（个）attributes
- values取决于它是inner nodes还是leaf nodes
  - inner nodes：value array包含指向其它nodes的指针
  - leaf nodes：两种方法
    - record IDs：指一个指向tuple所在位置的指针
    - tuple data：直接在node中保存tuple的实际内容（二级索引必须将record id作为他们的values）

每个node的array通常是以基于keys的顺序排序的。

![1.PNG](https://i.loli.net/2021/03/03/YH4x5QznD2fgjlw.png)

#### Node Size

storage速度越慢，最佳的node大小就越大。

最佳的大小会根据workload而变化（leaf node scans，root-to-leaf traversals）

### Selection Conditions

如果一个query提供search key的任何attributes，那么DBMS就可以用B+Tree index。

这不同于hash index（hash index需要search key的所有attributes）

### Insertion

- 找到正确的leaf node **L**

- 将新的entry插入L（保证顺序）
  - 如果L有足够空间，做这件事
  - 否则，将L分裂为两个nodes：L和**L2**。均匀地重新分配entries，复制正中间的key。将指向L2的index entry作为L的parent
- 为了分裂一个inner node，均匀地重新分配entries，但是push up中间的那个key

### Deletion

如果删除操作导致$\#keys$小于half-full，我们必须进行merge操作。

- 找到正确的leaf node L
- 删除这个entry：
  - 如果L至少是half-full的，该操作完成
  - 否则，可以尝试redistribution，从sibling node中借
  - 如果redistributio失败了，将L和sibling合并
- 如果merge发生了，必须删除指向L的parent entry

#### Merge Threshold

一些DBMS不总是立即merge nodes，当它是half-full时。

延迟一个merge操作可能会减小organization的量。

更佳的做法是把他们放在那里，然后定期去重建整个tree。（merge操作时昂贵的，顶层的拆分也是昂贵的）

### @Variable Length Keys

- Pointers：将keys保存为指向tuple的attribute的指针（现在没人这么做了）
- variable length nodes：index中每个node的大小是不同的。需要小心地管理内存（是一个bad idea，也没人这么做了）
- padding：总是将key填充为key type的最大长度
- key map/indirection：将一个指针array嵌入node，这些指针指向key+value（more common）

### Non-Unique Indexes

- duplicate keys：相同的keys被保存了多次
- value lists：每个key只被保存一次，维护一个unique values的linked list

### @Duplicate Keys

两种办法：

- Append Record Id：将record ID作为key的一部分。
- Overflow Leaf Nodes：允许leaf node涌入overflow nodes（其中包含了duplicate keys）。（尽管没有保存多余的信息，这种方法维护和修改起来更加复杂）

### Clustered Indexes

（可以在创建table时定义一个clustered index）

此时table是根据primary key按顺序存储的（heap- or index-organized storage）

一些DBMS总是使用clustered index：如果一个table不包含primary key，那么DBMS将会自动将一个隐藏的row id作为primary key。（不是所有DBMS都支持这个）

### @Heap Clustering

tuple根据custering index规定的顺序在heap‘s pages中被排序。如果clustering index's attributes被用来访问tuples，DBMS可以直接跳转到pages

### @Index Scan Page Sorting

由于直接将tuples从一个unclustered index取出是低效的，DBMS可以首先找出需要的tuples，然后根据它们的page id将它们排序。