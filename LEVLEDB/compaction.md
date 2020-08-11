## compaction

#### compaction

#### minor compaction

数据持久化，生成一个sstable。

这是0层，但是0层的多个key的文件会有重叠，一个key存在多个范围

#### major compaction

- 0层文件数大于4个
- level i层文件大小超过10^i MB
- 某个文件无效读取次数过多

把多个上层sstable多路合并到下一层。（为了减少io开销）

- 一个下层文件 + 所有范围的上层文件，生成一个新的下层文件

主要会对不同版本的数据项进行合并。





- 如果有比较大的写buffer，没有必要做太多的0层压缩
- 如果写buffer比较小，由于我们每次read的时候都要读取level0层的全部文件，所以我们不要太多的文件。

```go
// We treat level-0 specially by bounding the number of files
		// instead of number of bytes for two reasons:
		//
		// (1) With larger write-buffer sizes, it is nice not to do too
		// many level-0 compaction.
		//
		// (2) The files in level-0 are merged on every read and
		// therefore we wish to avoid too many files when the individual
		// file size is small (perhaps because of a small write-buffer
		// setting, or very high compression ratios, or lots of
		// overwrites/deletions).
```
- minor compaction是一个时效性要求非常高的过程，要求其在尽可能短的时间内完成，否则就会堵塞正常的写入操作，因此**minor compaction的优先级高于major compaction**



- 为什么要多层level
- 0层文件key的只范围太大，因此如果1层文件数量太多，每次都会输入超级多的1层文件

#### 平衡读写

- 0层文件比较多时，写入减慢
- 0层文件太多时，写入暂停，直到major compaction完成


### 直接移动（很重要）

- source层的文件个数只有一个；
- source层文件与source+1层文件没有重叠；
- source层文件与source+2层的文件重叠部分不超过10个文件；
-  注：条件三主要是(避免mv到level + 1后，导致level + 1 与 level + 2层compact压力过大)

#### 错峰合并

如果同时好几层需要合并，那么开销很大。

多次查询不命中的需要被错峰合并。

- 如果某一个文件被查询多次且不命中，那么该文件错峰合并

- 对于，对于一个1MB的文件，对其合并的开销为25ms。因此当一个文件1MB的文件无效查询超过25次时，这个开销和合并差不多。

  因此，一个读取大约为40KB的数据。我们保守一点，设置为16kb。

  就是说文件大小 / 16kb就是最大的无效查询次数

  > 对于一个1MB的文件，其合并开销为（1）source层1MB的文件读取，（2）source+1层 10-12MB的文件读取（3）source+1层10-12MB的文件写入。

#### 多路合并

- 在i层找到所有和key有重叠的文件
- 在i  + 1层找到所有和i层有重叠的文件
- 在i层找到所有和i + 1层有重叠的文件（不会扩大i + 1层）



## 用户行为

由于leveldb内部进行compaction时有trivial move优化，且根据内部的文件格式组织，用户在使用leveldb时，可以尽量将大批量需要写入的数据进行预排序，利用空间局部性，尽量减少多路合并的IO开销。



除了level 0以外，任何一个level的文件内部是有序的，文件之间也是有序的。但是level（n）和level（n＋1）中的两个文件的key可能存在交叉。正是因为这种交叉，查找某个key值的时候，level（n） 的查找无功而返，而不得不去level(n＋1)查找。如果查找了多次，某个文件不得不查找，却总也找不到，总是去高一级的level，才能找到。这说明该层级的文件和上一级的文件，key的范围重叠的很严重，这是不合理的，会导致效率的下降。因此，需要对该level 发起一次major compaction，减少 level 和level ＋ 1的重叠情况。