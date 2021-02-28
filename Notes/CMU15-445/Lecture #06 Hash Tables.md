## Course Status

Access Methods：read/write data from pages

data structures： hash tables，trees

## Data Structures

- Internal Meta-data

- Core Data Storage

- Temporary Data Structures

- Table Indexes

## Design Decisions

Data Organization：如何在memory/pages上组织data structure，需要保留哪些信息来提高访问效率

Concurrency

## Hash Tables

Space Complexity：O(n)

Operation Complexity：Average：O(1)，Worst：O(n)

Design Decision：

- Hash Function：将一个大的key space映射到一个小范围的int值上（trade-off：fast 、collision rate）
- Hashing Scheme：在hashing之后处理hash collisions（trade-off：large hash table、多余的指令来find/insert keys）

## Hash Functions

不使用密码学的hash function，我们希望：速度快，同时collision rate比较低

XXHash

## Static Hashing Schemes

### Linear Probe Hashing

单独的一个大的table of slots

解决collision：线性搜索下一个空闲的slot

必须保存key

#### Non-Unique Keys

- Separate Linked List
- Redundant Keys

### Robin Hood Hashing

每个key记录它与最优位置的距离。

On insert, a key takes the slot of another key if the first key is farther away from its optimal position than the second key.

### Cuckoo Hashing

Use multiple hash tables with different hash function seeds.

- On insert, check every table and pick anyone that has a free slot.

-  If no table has a free slot, evict the element from one of them and then re-hash it find a new location.

Look-ups and deletions are always **O(1)** because only one location per hash table is checked

 [open-source implementation](https://github.com/efficient/libcuckoo)

### Observation

需要DBMS知道需要保存的东西的数量

## Dynamic Hash Tables

它能够根据需要resize

### Chained Hashing

如果出现collision，就将具有相同hash key的element放入同一个bucket。维护一个buckets的链表。

### @@Extendible Hashing

Chained-hashing approach where we split buckets instead of letting the linked list grow forever.

Multiple slot locations can point to the same bucket chain.

Reshuffling bucket entries on split and increase the number of bits to examine.

- Data movement is localized to just the split chain.

### @@Linear Hashing

The hash table maintains a pointer that tracks the next bucket to split.

- When any bucket overflows, split the bucket at the pointer location.

Use multiple hashes to find the right bucket for a given key.

Can use different overflow criterion:

- Space Utilization

- Average Length of Overflow Chains

Splitting buckets based on the split pointer will eventually get to all overflowed buckets.

- When the pointer reaches the last slot, delete the first hash function and move back to beginning.

The pointer can also move backwards when buckets are empty



