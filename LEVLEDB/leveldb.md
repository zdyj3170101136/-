#### lsm

log-structed-merged-tree

日志结构合并树。



LSM树的核心思想就是放弃部分读的性能，换取最大的写入能力。



LSM树的结构是横跨内存和磁盘的，包含memtable、immutable memtable、SSTable等多个部分。



memtable使用跳表实现的内存数据库，当写数据到memtable中时，会先通过WAL的方式备份到磁盘中，以防数据因为内存掉电而丢失。

当leveldb达到checkpoint点（memtable中的数据量超过了预设的阈值），会将当前memtable冻结成一个不可更改的内存数据库（immutable memory db），并且创建一个新的memtable供系统继续使用。

然后这个不可变的数据库，再以1:1的形式存储入硬盘生成sstable。

immutable memory db会在后台进行一次minor compaction，即将内存数据库中的数据持久化到磁盘文件中。

sstable分成不同的level，而由memtable生成的sstable会在第0层，然后sstable之间通过合并。生成更下层的sstable。（这也是merge的由来）。（major compaction）。



#### sync

put有两个重要的设置，merge和sync。

- 如果设置了sync，写日志之后会flush日志。
- 如果没有，不会

#### Storage

所有的文件比如日志啥的，最终还是要存在磁盘里头。

go使用storage实现了这种接口

- log（打印一跳日志或者到文件里头，用来记录）

- ```
  // SetMeta store 'file descriptor' that can later be acquired using GetMeta
  // method. The 'file descriptor' should point to a valid file.
  // SetMeta should be implemented in such way that changes should happen
  // atomically.
  SetMeta(fd FileDesc) error
  ```

还有Close。



当我们打开一个目录for storage的时候。

先判断路径存不存在

- 如果存在，判断是不是目录
- 如果不存在，那就新建一个目录

```
if fi, err := os.Stat(path); err == nil {
   if !fi.IsDir() {
      return nil, fmt.Errorf("leveldb/storage: open %s: not a directory", path)
   }
} else if os.IsNotExist(err) && !readOnly {
   if err := os.MkdirAll(path, 0755); err != nil {
      return nil, err
   }
} else {
   return nil, err
}
```



随后建立一个文件名为“LOCK”用来实现flock。

“LOG”的文件用来实现日志记录（和journal不一样）。



最后设置 

- runtime.SetFinalizer(fs, ().close)

(为了防止我们忘记关闭storage，这个函数会在被垃圾回收的时候调用)



Setmeta是用来生成manifest文件。









#### cache

cache使用散列函数映射到bucket。

一个bucket不超过32个数据

不够用了时候会resize。



resize的时候有一个后台进程会rehash。

如果r / w操作发现rehash还没ok，则会帮忙构建。



- cache：缓存sstable元数据以及文件句号手柄（500）
- bacche：存储databloack（8mb）

通过lrucache和引用计数实现。



#### log structed merger-tree

- 顺序写
- 通过index（文件偏移量）读取文件
- 通过compaction所见文件数量



- 通过稀疏索引，在sstable里图由于数据本身安序存放，key划分为若干个block
- 通过先写log解决memtable转换的问题

#### memdb

![img](https://leveldb-handbook.readthedocs.io/zh/latest/_images/skiplist_intro.jpeg)

每间隔两个链表节点，增加一个指针，使得效率为O（N）。



- 以0.25的随机概率产生堆积的层高。



为什么

- 如果固定间隔相同，那么由于结构调整，插入和删除需要O（n）
- maxheight为12，，因为4^11能够处理百万级别的数据



由于跳表可以以数组的形式实现，因此紧密存储。

- 第一个字节用来存储本节点key-value数据在kvData中对应的偏移量；
- 第二个字节用来存储本节点key值长度；
- 第三个字节用来存储本节点value值长度；
- 第四个字节用来存储本节点的层高；
- 第五个字节开始，用来存储每一层对应的下一个节点的索引值；

关键这个p越大那么越密集，空间存储成本就会越大。

http://huntinux.github.io/leveldb-skiplist.html

![img](https://leveldb-handbook.readthedocs.io/zh/latest/_images/internalkey.jpeg)

![img](http://img.blog.itpub.net/blog/2019/01/10/20236399a91871de.jpeg?x-oss-process=style/bb)

因此相同的key在跳表中的位置相同，而get会获取最大的sequence。



#### 

之前已经阐述过为何这样的操作可以获得极高的写入性能，以及通过先写日志的方法能够保障用户的写入不丢失。

注解

其实leveldb仍然存在写入丢失的隐患。在写完日志文件以后，操作系统并不是直接将这些数据真正落到磁盘中，而是暂时留在操作系统缓存中，因此当用户写入操作完成，操作系统还未来得及落盘的情况下，发生系统宕机，就会造成写丢失；但是若只是进程异常退出，则不存在该问题。



#### batch 写

每个写操作都会尝试合并写

### 快照

快照代表着数据库某一个时刻的状态，在leveldb中，作者巧妙地用一个整型数来代表一个数据库状态。

在leveldb中，用户对同一个key的若干次修改（包括删除）是以维护多条数据项的方式进行存储的（直至进行compaction时才会合并成同一条记录），每条数据项都会被赋予一个序列号，代表这条数据项的新旧状态。一条数据项的序列号越大，表示其中代表的内容为最新值。

**因此，每一个序列号，其实就代表着leveldb的一个状态**。换句话说，每一个序列号都可以作为一个状态快照。

当用户主动或者被动地创建一个快照时，leveldb会以当前最新的序列号对其赋值。例如图中用户在序列号为98的时刻创建了一个快照，并且基于该快照读取key为“name”的数据时，即便此刻用户将"name"的值修改为"dog"，再删除，用户读取到的内容仍然是“cat”。



1. 在memory db中查找指定的key，若搜索到符合条件的数据项，结束查找；

2. 在冻结的memory db中查找指定的key，若搜索到符合条件的数据项，结束查找；

3. 按低层至高层的顺序在level i层的sstable文件中查找指定的key，若搜索到符合条件的数据项，结束查找，否则返回Not Found错误，表示数据库中不存在指定的数据；

   注解

   注意leveldb在每一层sstable中查找数据时，都是按序依次查找sstable的。

   0层的文件比较特殊。由于0层的文件中可能存在key重合的情况，因此在0层中，文件编号大的sstable优先查找。理由是文件编号较大的sstable中存储的总是最新的数据。

   非0层文件，一层中所有文件之间的key不重合，因此leveldb可以借助sstable的元数据（一个文件中最小与最大的key值）进行快速定位，每一层只需要查找一个sstable文件的内容。

#### rocksdb

RocksDB 

- memtable也有bloon过滤器

- 在0-1层使用snappy算法，之后使用zlib算法

- 对于memtable 使用huge page（huge page不是4kb，而是几m）

  更容易命中cache

- 多个memtable，多个线程合并文件


#### bloom

![截屏2020-07-15 下午1.52.18](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-15 下午1.52.18.png)

k个hash函数，计算结果与数组长度取余，对应位置1.（事实上只计算一次，然后根据原始hash值派生出更多的hash值。

比如说原始hash值不断的旋转。）

首先，与布隆过滤器准确率有关的参数有：

- 哈希函数的个数k；
- 布隆过滤器位数组的容量m;
- 布隆过滤器插入的数据数量n;

主要的数学结论有：

1. 为了获得最优的准确率，当k = ln2 * (m/n)时，布隆过滤器获得最优的准确性；
2. 在哈希函数的个数取到最优时，要让错误率不超过є，m至少需要取到最小值的1.44倍；



通常m / n 取10，然后我们通过k = 10 * 0.69生成k。



然后实际中我们只执行一次计算，生成一个hash值。



我们首先10 * 总共的key的数量，分配bloom过滤器的数组长度。

然后通过对这个hash不断的改变。。。

比方说不断加上它的异或值。

```
for _, kh := range g.keyHashes {
   delta := (kh >> 17) | (kh << 15) // Rotate right 17 bits
   for j := uint8(0); j < g.k; j++ {
      bitpos := kh % nBits
      dest[bitpos/8] |= (1 << (bitpos % 8))
      kh += delta
   }
}
```

