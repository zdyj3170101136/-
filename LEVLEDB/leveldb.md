#### 日志

http://blog.itpub.net/31561269/viewspace-2375371/

snappy不追求压缩率，压缩率够用的情况下，最快。。。

https://leveldb-handbook.readthedocs.io/zh/latest/rwopt.html

https://leveldb-handbook.readthedocs.io/zh/latest/rwopt.html

为了增加读取效率，日志文件中按照block进行划分，每个block的大小为32KiB。每个block中包含了若干个完整的chunk。

一条日志记录包含一个或多个chunk。每个chunk包含了一个7字节大小的header，前4字节是该chunk的校验码，紧接的2字节是该chunk数据的长度（little endian），以及最后一个字节是该chunk的类型。其中checksum校验的范围包括chunk的类型以及随后的data数据。

chunk共有四种类型：full，first，middle，last。一条日志记录若只包含一个chunk，则该chunk的类型为full。若一条日志记录包含多个chunk，则这些chunk的第一个类型为first, 最后一个类型为last，中间包含大于等于0个middle类型的chunk。

由于一个block的大小为32KiB，因此当一条日志文件过大时，会将第一部分数据写在第一个block中，且类型为first，若剩余的数据仍然超过一个block的大小，则第二部分数据写在第二个block中，类型为middle，最后剩余的数据写在最后一个block中，类型为last。



日志内容为 batch编码信息

- Sequence number(自增加一)
- entry number(包含的put 和 delete个数）

![img](https://leveldb-handbook.readthedocs.io/zh/latest/_images/batch.jpeg)

实际中所有的batch都会编码成以上的字节流。

如果不是delete操作还会加上value大小。



- 其中internal key len不仅包括key的大小，还是internal key的大小（每条数据项额外8个字节）

```
// Batch is a write batch.
type Batch struct {
   data  []byte
   index []batchIndex

   // internalLen is sums of key/value pair length plus 8-bytes internal key.
   internalLen int
}


```

其实每个batch都是通过pool获得的

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

#### 版本控制

manifest文件内部包含多条session record。



manifest仅在sstable文件变化时变化。（每次compaction）

第一条session record记录全量信息。

后面的仅记录增量信息。

session record

- 最新的journal文件编号
- 已经持久话的最大sequnce number
- 新增的文件
- 删除的文件



1. 新建一个session record，记录状态变更信息；
2. 若本次版本更新的原因是由于minor compaction或者日志replay导致新生成了一个sstable文件，则在session record中记录新增的文件信息、最新的journal编号、数据库sequence number以及下一个可用的文件编号；
3. 若本次版本更新的原因是由于major compaction，则在session record中记录新增、删除的文件信息、下一个可用的文件编号即可；
4. 利用当前的版本信息，加上session record的信息，创建一个全新的版本信息。相较于旧的版本信息，新的版本信息更改的内容为：（1）每一层的文件信息；（2）每一层的计分信息；
5. 将session record持久化；
6. 若这是数据库启动后的第一条session record，则新建一个manifest文件，并将完整的版本信息全部记录进session record作为该manifest的基础状态写入，同时更改current文件，将其**指向**新建的manifest；
7. 若数据库中已经创建了manifest文件，则将该条session record进行序列化后直接作为一条记录写入即可；
8. 将当前的version设置为刚创建的version；

- 保证了增加删除sstable是原子状态

#### Recover

每次启动创建一个manifest，第一条session record记录当前快照。

过期的manifest会在下次启动时删除。

- 如果没有重启则还是会增长



启动的时候

- 读取最近的manifest文件
- 逐个apply操作，还原最新的version
- 删除旧的manifest

#### Current

current指向当前使用的manifest文件

（因为，如果生成新的manifest文件，还没来得及删除旧的，因此可能会同时存在多个）

当我们setmeta的时候。

- 我们首先对旧的current文件做一个.bak文件
- 然后创建一个current.num文件，先把当前的manifest文件名写进去
- 然后再rename，replace现在的current文件
- 最后syncdir



#### syncdir

注意每次写文件不仅要sync文件本身，还得sync目录

#### MVCC

- 只读sstable，每次compaction都只是多路合并创建新的文件

不会影响原来的正确性

- 每次sstable更新都会生成新版本的sstable，保证读写操作针对对应的版本进行
- 当compaction完成后试图删除某个sstable，会根据引用计数做一定的删除。

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





- 为什么要多层level
- 0层文件key的只范围太大，因此如果1层文件数量太多，每次都会输入超级多的1层文件

#### 平衡读写

- 0层文件比较多时，写入减慢
- 0层文件太多时，写入暂停，直到major compaction完成

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

#### bloom

bloom是用多个hash函数对一个数进行计算。

若所有位均为一，代表有可能存在。

#### sstable

sstable 是memtable经过minor compaction生成的磁盘数据。

- 避免日志文件过大
- 降低内存使用率

![img](https://leveldb-handbook.readthedocs.io/zh/latest/_images/sstable_logic.jpeg)

- filter block：bloom过滤器
- datab block：数据
- index block：存储data block的元数据
- footer： index block等的元数据



以block的形式存储。

每个block4kb。

- 压缩类型（snappy）
- crc校验

#### datablock

datablock分为restart point和具体的key value。

![img](https://leveldb-handbook.readthedocs.io/zh/latest/_images/datablock.jpeg)

对于restart point，每隔16个key记录一个key的内容。

不然则记录与上一个key不同的部分

- 通过restart point快速定位查找，再细粒度比较

- shared key ｜ unshred key ｜ value length ｜ unshared key ｜ value

#### filter block

![img](https://leveldb-handbook.readthedocs.io/zh/latest/_images/filterblock_format.jpeg)

filter block主要是每2kb放一个过滤器存放数据。

- 按照数据而不是block切分，主要某些block可能key比较小。



- sstable元数据和数据一起存放，不用动态计算和维护更新

#### index block

- 偏移量
- 最大的key

通过index block 比较在哪个data blcok。

通过data blcok的retart point找到精确的点



footer 报考meta index和index的偏移量。

以及magic word。



1. 首先判断“文件句柄”cache中是否有指定sstable文件的文件句柄，若存在，则直接使用cache中的句柄；否则打开该sstable文件，**按规则读取该文件的元数据**，将新打开的句柄存储至cache中；
2. 利用sstable中的index block进行快速的数据项位置定位，得到该数据项有可能存在的**两个**data block；
3. 利用index block中的索引信息，首先打开第一个可能的data block；
4. 利用filter block中的过滤信息，判断指定的数据项是否存在于该data block中，若存在，则创建一个迭代器对data block中的数据进行迭代遍历，寻找数据项；若不存在，则结束该data block的查找；
5. 若在第一个data block中找到了目标数据，则返回结果；若未查找成功，则打开第二个data block，重复步骤4；
6. 若在第二个data block中找到了目标数据，则返回结果；若未查找成功，则返回`Not Found`错误信息；



cache

- sstable文件句柄及其元数据；
- data block中的数据；



#### 为什么是两个

- index block每两个才会存储一个完整的key



- sstable是只读的。
- 元数据一起存储
- 引用计数实现无锁并发
- cache中的数据永远一致。



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

- 