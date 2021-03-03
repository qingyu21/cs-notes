[TOC]

## Course Status

Access Methods：read/write data from pages

data structures： hash tables，trees

## Data Structures

- Internal Meta-data：记录有关数据库的信息和系统状态（例如：page tables，page directories）

- Core Data Storage：用作tuples的基础存储

- Temporary Data Structures：DBMS可以在处理数据库时建立动态的数据结构，以加快查询速度。（例如：hash tables for joins）

- Table Indexes：用于简化查找特定tuples的过程的辅助数据结构。

### Design Decisions

Data Organization：如何在memory/pages上组织data structure，需要保留哪些信息来提高访问效率

Concurrency：考虑多线程同时访问数据结构时不会导致错误。

## Hash Tables

Space Complexity：O(n)

Operation Complexity：Average：O(1)，Worst：O(n)

Design Decision：

- **Hash Function**：将一个大的key space映射到一个小范围的int值上（trade-off：fast 、collision rate）（常用于将一个index映射到一个array of buckets or slots）
- **Hashing Scheme**：在hashing之后处理hash collisions（trade-off：large hash table、当collision发生时，执行多余的指令来find/insert keys）

## Hash Functions

不使用密码学的hash function（例如SHA-256）（因为没必要去考虑保护key的内容，它主要由内部使用，所以信息不会泄露到系统外部）

我们希望：速度快，同时collision rate比较低

当前最先进的hash函数：XXHash3

## Static Hashing Schemes

 static hashing scheme是指hash table的大小是固定的。

这意味着如果hash table的空间不足，必须从头开始构建一个更大的hash table（代价十分昂贵）（通常大小是原始hash table的两倍）

通常我们使用的slots数是预期元素的两倍。

理想情况的假设：

- 元素数量提前知道
- keys都是unique的
- 存在完善的hash函数：如果key1≠key2，那么hash(key1)≠hash(key2)

### Linear Probe Hashing

（最基础，通常也是最快的）

单独的一个大的table of slots

解决collision：线性搜索下一个空闲的slot（搜索时是环形的）

必须保存key（因为要比较key）

删除操作（直接删除entry可能会阻止之后搜索它下面的entry）：

- tombstones（most common），不直接删除这个entry，而是将它替换为一个tombstone entry，这告诉之后的搜索继续搜索。
- 删除之后，移动相邻的数据填充空的slot，但要注意只移动一开始移动过的entry

#### Non-Unique Keys

- Separate Linked List：不保存value，而是保存一个指向一个单独存储区域的指针，这个区域有一个保存所有values的链表。
- Redundant Keys（more common）：直接在table中保存这个key多次。

### Robin Hood Hashing

每个key记录它与最优位置的距离。

On insert, a key takes the slot of another key if the first key is farther away from its optimal position than the second key.

### Cuckoo Hashing

Use multiple hash tables with different hash function seeds（the same algorithm）.

- On insert, check every table and pick anyone that has a free slot.（如果都有，可以比较一些东西比如load factor。或者更通常就直接选一个随机的table）

-  If no table has a free slot, evict the element from one （random）of them and then re-hash the old one to find a new location.（这种情况下可能出现无限循环。此时可以使用新的seeds来重建所有hash tables（less common），或者用更大的tables重建hash tables（more common））

Look-ups and deletions are always **O(1)** because only one location per hash table is checked（but insertions may be more expensive）

 [open-source implementation](https://github.com/efficient/libcuckoo)

## Dynamic Hash Tables

当需要grow/shrink大小时，static hashing schemes需要重建tables

而dynamic hashing schemes能够根据需要resize它整个table（不用rebuild）。

有多种resizing方法（可以最大化读取或写入）：

### Chained Hashing

最常见的dynamic hashing scheme

在hash table的每个slot上维护一个buckets的链表。

如果出现collision，就将具有相同hash key的element插入这个buckets的链表。

### @Extendible Hashing

chained hashing的变体。

Chained-hashing approach where we split buckets instead of letting the linked list grow forever.

Multiple slot locations can point to the same bucket chain.

The core idea behind re-balancing the hash table is to to move bucket entries on split and increase the number of bits to examine to fifind entries in the hash table.

This means that the DBMS only has to move data within the buckets of the split chain; all other buckets are left untouched.

- The DBMS maintains a global and local depth bit counts that determine the number bits needed to find buckets in the slot array.
- When a bucket is full, the DBMS splits the bucket and reshufflfle its elements. If the local depth of the split bucket is less than the global depth, then the new bucket is just added to the existing slot array. Otherwise, the DBMS doubles the size of the slot array to accommodate the new bucket and increments the global depth counter.

### @Linear Hashing

Instead of immediately splitting a bucket when it overflflows，the hash table maintains a pointer that tracks the next bucket to split.

- When any bucket overflows, split the bucket at the pointer location.

Use multiple hashes to find the right bucket for a given key.

Can use different overflow criterion:

- Space Utilization

- Average Length of Overflow Chains

Splitting buckets based on the split pointer will eventually get to all overflowed buckets.

- When the pointer reaches the last slot, delete the first hash function and move back to beginning.

The pointer can also move backwards when buckets are empty



