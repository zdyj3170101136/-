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



leveldb的文件名。

日志内容以天数作为分割符。

- 大范围的年月日
- 小范围的时分秒以及ms

每个日志都直接写进磁盘，大于1m之后就减去，修改为LOG.OLD.



- 注意最多就两份日志，一个LOG，一个叫做LOG.OLD。
- 而且注意没有sync语意。



- 外部日志，提供给外面用的，会打印出来。

包括哪个模块，以@作为分割符。

一般对于时间精确到ns = 1 * 10 -9 除以1e3，也就是us



EINVAL 错误（sync 目录引发的错误）。



- 对于filestorage的内部日志。
- 只有当发生错误的时候才会记录，比如说往磁盘里头调用fs.Write失败。
- 所以这个LOG里头基本不会有filestorage的内部日志。



- 一个是前缀压缩，比如说用T代表TimeElapased，F代表NumFiles
- 对于操作通常就是一个比如说open，close，revocery， stat 表示操作类型
- 随后跟状态，opening，done，recovering等等
- 像sstable分为多层，那就用一个数组表示[0, 1]为从低到高的level

- ```go
  if err := syncDir(fw.fs.path); err != nil {
     fw.fs.log(fmt.Sprintf("syncDir: %v", err))
     return err
  }
  ```

![截屏2020-07-30 下午3.15.08](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-30 下午3.15.08.png)

```go
func (fs *fileStorage) printDay(t time.Time) {
   if fs.day == t.Day() {
      return
   }
   fs.day = t.Day()
   fs.logw.Write([]byte("=============== " + t.Format("Jan 2, 2006 (MST)") + " ===============\n"))
}

func (fs *fileStorage) doLog(t time.Time, str string) {
   if fs.logSize > logSizeThreshold {
      // Rotate log file.
      fs.logw.Close()
      fs.logw = nil
      fs.logSize = 0
      rename(filepath.Join(fs.path, "LOG"), filepath.Join(fs.path, "LOG.old"))
   }
   if fs.logw == nil {
      var err error
      fs.logw, err = os.OpenFile(filepath.Join(fs.path, "LOG"), os.O_WRONLY|os.O_CREATE, 0644)
      if err != nil {
         return
      }
      // Force printDay on new log file.
      fs.day = 0
   }
   fs.printDay(t)
   hour, min, sec := t.Clock()
   msec := t.Nanosecond() / 1e3
   // time
   fs.buf = itoa(fs.buf[:0], hour, 2)
   fs.buf = append(fs.buf, ':')
   fs.buf = itoa(fs.buf, min, 2)
   fs.buf = append(fs.buf, ':')
   fs.buf = itoa(fs.buf, sec, 2)
   fs.buf = append(fs.buf, '.')
   fs.buf = itoa(fs.buf, msec, 6)
   fs.buf = append(fs.buf, ' ')
   // write
   fs.buf = append(fs.buf, []byte(str)...)
   fs.buf = append(fs.buf, '\n')
   n, _ := fs.logw.Write(fs.buf)
   fs.logSize += int64(n)
}
```

日志的内容为当前的时间加上执行了啥啥操作

```go
func (fs *fileStorage) log(str string) {
   if !fs.readOnly {
      fs.doLog(time.Now(), str)
   }
}
```

![截屏2020-07-30 下午2.56.49](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-30 下午2.56.49.png)

- leveldb存储文件使用相同一个自增的sequecnce number 06d，作为序列化(为什么日志也要加上数字呢，因为本质上不同的日志就是不同的文件)（注意sequence number在session那边。）
- table加上.ldb前缀
- 临时文件加上.temp前缀
- 日志加上.log前缀.
- filedstorage自己还有一个LOG文件，记录日志
- 一个LOCK文件，用于实现文件锁语意
- 一个CURRENT文件，指向最新的manifest。

```go
func fsGenName(fd FileDesc) string {
   switch fd.Type {
   case TypeManifest:
      return fmt.Sprintf("MANIFEST-%06d", fd.Num)
   case TypeJournal:
      return fmt.Sprintf("%06d.log", fd.Num)
   case TypeTable:
      return fmt.Sprintf("%06d.ldb", fd.Num)
   case TypeTemp:
      return fmt.Sprintf("%06d.tmp", fd.Num)
   default:
      panic("invalid file type")
   }
}
```



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



- 释放文件锁
- close日志

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



```go
for i, index := range b.index {
   ik = makeInternalKey(ik, index.k(b.data), seq+uint64(i), index.keyType)
   if err := mdb.Put(ik, index.v(b.data)); err != nil {
      return err
   }
}
```

- 在memedb里头的key是internal key，value就是value本身。
- 而且keysize也不是varint，而就是len（），也就是int。
- 这里的varint keysize应该指的是batch编码后的结果。
- 但此时batch操作不是internal key。

```go
func (b *Batch) putMem(seq uint64, mdb *memdb.DB) error {
   var ik []byte
   for i, index := range b.index {
      ik = makeInternalKey(ik, index.k(b.data), seq+uint64(i), index.keyType)
      if err := mdb.Put(ik, index.v(b.data)); err != nil {
         return err
      }
   }
   return nil
}
```

http://huntinux.github.io/leveldb-skiplist.html

![img](https://leveldb-handbook.readthedocs.io/zh/latest/_images/internalkey.jpeg)

![img](http://img.blog.itpub.net/blog/2019/01/10/20236399a91871de.jpeg?x-oss-process=style/bb)

因此相同的key在跳表中的位置相同，而get会获取最大的sequence。



- 通过这样的比较规则，则所有的数据项首先按照用户key进行升序排列；
- 当用户key一致时，按照序列号进行降序排列，这样可以保证首先读到序列号大的数据项。
- 序列号一致的时候，put操作会在前面。



- 如果在一次batch操作之中，对同一个key多次添加操作，那么就会在memdb中只有一个位置存储

```go
ikey := makeInternalKey(nil, key, seq, keyTypeSeek)

// keyTypeSeek defines the keyType that should be passed when constructing an
// internal key for seeking to a particular sequence number (since we
// sort sequence numbers in decreasing order and the value type is
// embedded as the low 8 bits in the sequence number in internal keys,
// we need to use the highest-numbered ValueType, not the lowest).
const keyTypeSeek = keyTypeVal

```

```go
// Value types encoded as the last component of internal keys.
// Don't modify; this value are saved to disk.
const (
   keyTypeDel = keyType(0)
   keyTypeVal = keyType(1)
)
```

- 比较key的大小，key字节序更大的在更后面
- key相同，比较sequence，sequence更大的会在前面。
- sequence相同，put操作会在前面



- 注意sequence不同但是ukey相同的会紧密排列



- 比方说，刚开始sequence为3，插入时候使用4.插入了100个数据后变成104.
- get的时候使用104



- 而find操作会返回大于当前key的值。

- 具体来说就是sequence第一个小于。

- 由于put操作会在前面，则先返回put操作

- 先返回put操作（add wins）

- 而如果发现type为delete就返回errnotfound

- ```go
  // Find finds key/value pair whose key is greater than or equal to the
  // given key. It returns ErrNotFound if the table doesn't contain
  // such pair.
  //
  // The caller should not modify the contents of the returned slice, but
  // it is safe to modify the contents of the argument after Find returns.
  func (p *DB) Find(key []byte) (rkey, value []byte, err error) {
     p.mu.RLock()
     if node, _ := p.findGE(key, false); node != 0 {
        n := p.nodeData[node]
        m := n + p.nodeData[node+nKey]
        rkey = p.kvData[n:m]
        value = p.kvData[m : m+p.nodeData[node+nVal]]
     } else {
        err = ErrNotFound
     }
     p.mu.RUnlock()
     return
  }
  ```

```go
// 比较的时候先比较key，如果key一样
// 再比较num，num包括sequence序列号和操作类型
// 序列号更小的， 或者序列号相同，delete操作更大。

// 这样子来说在跳表中越小的越前。
func (icmp *iComparer) Compare(a, b []byte) int {
   x := icmp.uCompare(internalKey(a).ukey(), internalKey(b).ukey())
   if x == 0 {
      if m, n := internalKey(a).num(), internalKey(b).num(); m > n {
         return -1
      } else if m < n {
         return 1
      }
   }
   return x
}
```

```go
func memGet(mdb *memdb.DB, ikey internalKey, icmp *iComparer) (ok bool, mv []byte, err error) {
   mk, mv, err := mdb.Find(ikey)
   if err == nil {
      ukey, _, kt, kerr := parseInternalKey(mk)
      if kerr != nil {
         // Shouldn't have had happen.
         panic(kerr)
      }
      if icmp.uCompare(ukey, ikey.ukey()) == 0 {
         if kt == keyTypeDel {
            return true, nil, ErrNotFound
         }
         return true, mv, nil

      }
   } else if err != ErrNotFound {
      return true, nil, err
   }
   return
}
```

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

   #### 注解

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



##### 为什么

他们表明，在某种程度上直觉上，顺序磁盘访问比随机访问主内存更快。更相关的是，它们还显示对磁盘的顺序访问，无论是磁盘还是SSD，至少比随机IO快三个数量级。这意味着要避免随机操作。顺序访问非常值得设计。

因此，考虑到这一点，我们可以考虑一些思想实验：如果我们对写入吞吐量感兴趣，最好的方法是什么？一个很好的起点是简单地将数据附加到文件中。这种方法通常称为日志记录，日记记录或堆文件，是完全顺序的，因此可提供与理论磁盘速度相当的非常快的写入性能（通常为每个驱动器200-300MB / s）。





1. 

受益于简单性和性能，基于日志/日志的方法在许多大数据工具中都很受欢迎。然而，他们有一个明显的缺点。从日志中读取任意数据将比写入日志更耗时，包括反向时间顺序扫描，直到找到所需的密钥。

这意味着日志只适用于简单的工作负载，其中数据可以完整访问，如大多数数据库的预写日志，也可以是已知的偏移量，如Kafka等简单消息产品。



因此，我们需要的不仅仅是日志，以便有效地执行更复杂的读取工作负载，如基于密钥的访问或范围搜索。从广义上讲，有四种方法可以帮助我们：二进制搜索，哈希，B +或外部。

1. 搜索已排序文件：将数据保存到文件，按键排序。如果数据已定义宽度，请使用二进制搜索。如果不使用页面索引+扫描。
2. 散列：使用散列函数将数据拆分为存储桶，稍后可用于指导读取。
3. B +：使用可导航的文件组织，如B +树， ISAM 等。
4. 外部文件：将数据保留为日志/堆，并在其中创建单独的散列或树索引。
   所有这些方法都显着提高了读取性能（大多数情况下n-> O（log（n）））。唉，这些结构增加了顺序，并且该顺序阻碍了写入性能，因此我们的高速日志文件在此过程中丢失了。我猜你不能吃蛋糕。



#### snaoshot

#### snapshot

```go
// Snapshot.
snapsMu   sync.Mutex
snapsList *list.List
```



leveldb中的snapshot通过一个读写锁和链表提供。

- 链表中存的是snapshotelement
  - 包括当前序列号
  - 以及引用计数





- 初始时链表为空
- 使用当前的任期号和初始的引用计数为1初始化，随后设置se.e为对应链表的element

- acquireSnapshot用于back链表，返回最新的snapshotElemet
- 因此当get的时候，使用当前的sequence生成一个snapshot，结束时侯释放。
- 如果有多个get那么会使用相同的snapshotElement进行访问。（如果这中间有写操作更新了sequence，那么事实上会使用最新的seq用来生成）（不过好处在于在两个write操作的间隙，都是使用一个snapshotElement就行）（对于snapshotelement就不需要原子读取）。

```go
type snapshotElement struct {
   seq uint64  // 任期号
   ref int // 引用计数，初始为1
   e   *list.Element // 指向链表的element， 在removesnapshot的时候传入
}
```

```go
// Acquires a snapshot, based on latest sequence.
func (db *DB) acquireSnapshot() *snapshotElement {
   db.snapsMu.Lock()
   defer db.snapsMu.Unlock()

   seq := db.getSeq()

   if e := db.snapsList.Back(); e != nil {
      se := e.Value.(*snapshotElement)
      if se.seq == seq {
         se.ref++
         return se
      } else if seq < se.seq {
         panic("leveldb: sequence number is not increasing")
      }
   }
   se := &snapshotElement{seq: seq, ref: 1}
   se.e = db.snapsList.PushBack(se)
   return se
}
```



- 当引用计数减小为0，从链表中释放

```go
// Releases given snapshot element.
func (db *DB) releaseSnapshot(se *snapshotElement) {
   db.snapsMu.Lock()
   defer db.snapsMu.Unlock()

   se.ref--
   if se.ref == 0 {
      db.snapsList.Remove(se.e)
      se.e = nil
   } else if se.ref < 0 {
      panic("leveldb: Snapshot: negative element reference")
   }
}
```





- 对于用户来说，GetSnapShot就是得到当前最新的snapshotElement
- 然后通过snapshot中记载的sequence对db进行访问。

```go
// Snapshot is a DB snapshot.
type Snapshot struct {
   db       *DB
   elem     *snapshotElement
   mu       sync.RWMutex
   released bool
}

// Creates new snapshot object.
func (db *DB) newSnapshot() *Snapshot {
   snap := &Snapshot{
      db:   db,
      elem: db.acquireSnapshot(),
   }
   atomic.AddInt32(&db.aliveSnaps, 1)
   runtime.SetFinalizer(snap, (*Snapshot).Release)
   return snap
}

func (snap *Snapshot) String() string {
   return fmt.Sprintf("leveldb.Snapshot{%d}", snap.elem.seq)
}
```

```go
// Get gets the value for the given key. It returns ErrNotFound if
// the DB does not contains the key.
//
// The caller should not modify the contents of the returned slice, but
// it is safe to modify the contents of the argument after Get returns.
func (snap *Snapshot) Get(key []byte, ro *opt.ReadOptions) (value []byte, err error) {
   err = snap.db.ok()
   if err != nil {
      return
   }
   snap.mu.RLock()
   defer snap.mu.RUnlock()
   if snap.released {
      err = ErrSnapshotReleased
      return
   }
   return snap.db.get(nil, nil, key, snap.elem.seq, ro)
}
```