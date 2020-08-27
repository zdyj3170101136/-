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



####

```go
// For this user key:
// (1) there is no data in higher levels
// (2) data in lower levels will have larger seq numbers
// (3) data in layers that are being compacted here and have
//     smaller seq numbers will be dropped in the next
//     few iterations of this loop (by rule (A) above).
// Therefore this deletion marker is obsolete and can be dropped.
```



```go
// NewMergedIterator returns an iterator that merges its input. Walking the
// resultant iterator will return all key/value pairs of all input iterators
// in strictly increasing key order, as defined by cmp.
// The input's key ranges may overlap, but there are assumed to be no duplicate
// keys: if iters[i] contains a key k then iters[j] will not contain that key k.
// None of the iters may be nil.
//
// If strict is true the any 'corruption errors' (i.e errors.IsCorrupted(err) == true)
// won't be ignored and will halt 'merged iterator', otherwise the iterator will
// continue to the next 'input iterator'.
```

我们使用一个迭代器，不断返回按照比较器最小的key。（也就是跳表中最前面的）

- put操作比delete小
- sequence大的比较小

从本例可以看出：switch人第一个expr为true的case开始执行，如果case带有fallthrough，程序会继续执行下一条case,不会再判断下一条case的expr。



- lastukey，表示上一次flush之后的ukey



- 我们会通过snaplist的链表得到当前最小的任期号。

- 小于最小的任期号，表示当前没有活跃的读取正在进行读取。

- ```go
  // Gets minimum sequence that not being snapshotted.
  func (db *DB) minSeq() uint64 {
     db.snapsMu.Lock()
     defer db.snapsMu.Unlock()
  
     if e := db.snapsList.Front(); e != nil {
        return e.Value.(*snapshotElement).seq
     }
  
     return db.getSeq()
  }
  ```



- 迭代器不断取出最小的internal key
- parse解码获得sequecnce， ukey， type



#### 对于相同的ukey

- 我们按照任期号从大到小， 任期号相同则put操作在前面遍历数据



#### 遍历到一个新的ukey的时

我们会尝试flush，生成一个i + 1层文件，要么当前这个文件满了，要么与i + 2层文件重叠过多。



无论是否flush。

- 使用lastukey
- lastseq设置为最大

#### 如果ukey相同

- 对于第一个ukey。使用lastseq这个变量记录seq

- 注意第一个肯定会添加进里头，无论是put还是delete。

- 因为lastseq初始值最大。

  



- 如果lastseq小于等于minsequence，那么就丢掉。（因为已经有了一个更新的了）（不管是put还是delete）

这里一方面是说，在sequence相同的时候，已经有了一个key添加到里头去了。

另一方面是说，太老了可以删除。毕竟memetable的find绝对不会返回这条数据。

- 更新lastseq为当前的sequence
- fallthourg会不去判断条件而执行下一条指令



- 感觉是｜｜操作



（由于case是会顺序执行的）

（因此这里如果当前sequence的第一条是delete。假设更上层没有数据，那么delete标记本身就可以被删除）。

（这也表示一种暗示，在这条delete操作的之后的put或者delete操作都会被删除）

（主要对于delete操作会做一个额外判断）。

- 如果出现了一个delete。

- 并且delete的sequence小于当前活跃的最小的sequence（而不是lastseq）
- 并且更高层没有这个key
- 丢弃这个delete操作
- 这说明这个delete操作没有用了。



#### case

活跃的sequence为11， 12，1 3。



- 比如说相同sequence，put多次。分别11，12，13.使用不同的任期号比如11，12进行遍历，结果不一样。
- 当我们加入了11之后，10，9，8都得删掉对吧。为什么用小于等于呢。因为同一个sequence只有一个put。
- 为什么要考虑等于呢，因为sequence相同的时候，delete操作和put操作相同。



- 如果说一个sequence，delete，11.
- 如果用11查找，找不到任何数据的。
- 但是我们不能立即删除它。因为如果删除了，那么sequence为3的sstable的put操作就无法删除。
- （毕竟在它之后很可能没有对相同key的put操作）
- 所以我们要判断，如果更高层没有数据，那么就可以直接删除了。
- 如果还有，不能删。而是要传递delete

```go
func (b *tableCompactionBuilder) run(cnt *compactionTransactCounter) error {
   snapResumed := b.snapIter > 0
   hasLastUkey := b.snapHasLastUkey // The key might has zero length, so this is necessary.
   lastUkey := append([]byte{}, b.snapLastUkey...)
   lastSeq := b.snapLastSeq
   b.kerrCnt = b.snapKerrCnt
   b.dropCnt = b.snapDropCnt
   // Restore compaction state.
   b.c.restore()

   defer b.cleanup()

   b.stat1.startTimer()
   defer b.stat1.stopTimer()

   iter := b.c.newIterator()
   defer iter.Release()
   for i := 0; iter.Next(); i++ {
      // Incr transact counter.
      cnt.incr()

      // Skip until last state.
      if i < b.snapIter {
         continue
      }

      resumed := false
      if snapResumed {
         resumed = true
         snapResumed = false
      }

      ikey := iter.Key()
      ukey, seq, kt, kerr := parseInternalKey(ikey)

      if kerr == nil {
         shouldStop := !resumed && b.c.shouldStopBefore(ikey)

         if !hasLastUkey || b.s.icmp.uCompare(lastUkey, ukey) != 0 {
            // First occurrence of this user key.

            // Only rotate tables if ukey doesn't hop across.
            if b.tw != nil && (shouldStop || b.needFlush()) {
               if err := b.flush(); err != nil {
                  return err
               }

               // Creates snapshot of the state.
               b.c.save()
               b.snapHasLastUkey = hasLastUkey
               b.snapLastUkey = append(b.snapLastUkey[:0], lastUkey...)
               b.snapLastSeq = lastSeq
               b.snapIter = i
               b.snapKerrCnt = b.kerrCnt
               b.snapDropCnt = b.dropCnt
            }

            hasLastUkey = true
            lastUkey = append(lastUkey[:0], ukey...)
            lastSeq = keyMaxSeq
         }

         switch {
         case lastSeq <= b.minSeq:
            // Dropped because newer entry for same user key exist
            fallthrough // (A)
         case kt == keyTypeDel && seq <= b.minSeq && b.c.baseLevelForKey(lastUkey):
            // For this user key:
            // (1) there is no data in higher levels
            // (2) data in lower levels will have larger seq numbers
            // (3) data in layers that are being compacted here and have
            //     smaller seq numbers will be dropped in the next
            //     few iterations of this loop (by rule (A) above).
            // Therefore this deletion marker is obsolete and can be dropped.
            lastSeq = seq
            b.dropCnt++
            continue
         default:
            lastSeq = seq
         }
      } else {
         if b.strict {
            return kerr
         }

         // Don't drop corrupted keys.
         hasLastUkey = false
         lastUkey = lastUkey[:0]
         lastSeq = keyMaxSeq
         b.kerrCnt++
      }

      if err := b.appendKV(ikey, iter.Value()); err != nil {
         return err
      }
   }

   if err := iter.Error(); err != nil {
      return err
   }

   // Finish last table.
   if b.tw != nil && !b.tw.empty() {
      return b.flush()
   }
   return nil
}
```





#### next

- 对每一个sst，我们抽象成一个iterator
- 我们保存，每一个文件最小的key到内存里头来。
- 然后再使用for循环遍历，找到最小的key是哪一个文件里头的。
- 我们保存下标index。这方便从对应的iterator中取出数据

```go
type mergedIterator struct {
   cmp    comparer.Comparer
   iters  []Iterator
   strict bool

   keys     [][]byte
   index    int
   dir      dir
   err      error
   errf     func(err error)
   releaser util.Releaser
}
```

```go
func (i *mergedIterator) next() bool {
   var key []byte
   if i.dir == dirForward {
      key = i.keys[i.index]
   }
   for x, tkey := range i.keys {
      if tkey != nil && (key == nil || i.cmp.Compare(tkey, key) < 0) {
         key = tkey
         i.index = x
      }
   }
   if key == nil {
      i.dir = dirEOI
      return false
   }
   i.dir = dirForward
   return true
}
```



https://www.cnblogs.com/ym65536/p/10995048.html

#### baselevel

baselevel判断这个key是否不存在i + 2层等更上层。

```go
func (c *compaction) baseLevelForKey(ukey []byte) bool {
   for level := c.sourceLevel + 2; level < len(c.v.levels); level++ {
      tables := c.v.levels[level]
      for c.tPtrs[level] < len(tables) {
         t := tables[c.tPtrs[level]]
         if c.s.icmp.uCompare(ukey, t.imax.ukey()) <= 0 {
            // We've advanced far enough.
            if c.s.icmp.uCompare(ukey, t.imin.ukey()) >= 0 {
               // Key falls in this file's range, so definitely not base level.
               return false
            }
            break
         }
         c.tPtrs[level]++
      }
   }
   return true
}
```