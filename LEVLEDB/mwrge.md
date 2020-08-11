#### merge



- 最多七层



```go
writeMergeC:  make(chan writeMerge),
writeMergedC: make(chan bool),
writeLockC:   make(chan struct{}, 1),
writeAckC:    make(chan error),
```

这说明只能有一个线程对write进行操作。



- 包括delete和put

  

在writeJurnal之前从

- 从writeMergeC（struct{}, 0）中接受值（既可以是一个put，也可以是batch）（batch最多1 << 20字节）（1m）
- 往writeMergedC（bool）返回true，表示已经完成

```go
merge:
   for mergeLimit > 0 {
      select {
      case incoming := <-db.writeMergeC:
         if incoming.batch != nil {
            // Merge batch.
            if incoming.batch.internalLen > mergeLimit {
               overflow = true
               break merge
            }
            batches = append(batches, incoming.batch)
            mergeLimit -= incoming.batch.internalLen
         } else {
            // Merge put.
            internalLen := len(incoming.key) + len(incoming.value) + 8
            if internalLen > mergeLimit {
               overflow = true
               break merge
            }
            if ourBatch == nil {
               ourBatch = db.batchPool.Get().(*Batch)
               ourBatch.Reset()
               batches = append(batches, ourBatch)
            }
            // We can use same batch since concurrent write doesn't
            // guarantee write order.
            ourBatch.appendRec(incoming.keyType, incoming.key, incoming.value)
            mergeLimit -= internalLen
         }
         sync = sync || incoming.sync
         merged++
         db.writeMergedC <- true

      default:
         break merge
      }
   }
}
```

- 写端select操作往writemergeC中发送值

- 然后等待writeMergedC中接受值（unlock会返回false）
- 从ancC获取这次merge的结果
- 往writelockC（一个大小）的管道赛结构体（所以某种意义上来讲只有一个线程可以做实际的put操作）



- unlockwrite会发送false

```
// Acquire write lock.
if merge {
   select {
   case db.writeMergeC <- writeMerge{sync: sync, keyType: kt, key: key, value: value}:
      if <-db.writeMergedC {
         // Write is merged.
         return <-db.writeAckC
      }
      // Write is not merged, the write lock is handed to us. Continue.
   case db.writeLockC <- struct{}{}:
      // Write lock acquired.
   case err := <-db.compPerErrC:
      // Compaction error.
      return err
   case <-db.closeC:
      // Closed
      return ErrClosed
   }
} else {
```



#### flush

尝试刷memdb

如果compaction跟不上的话，会限制写入速率。



- 把batch操作的internalLen传进去



- 一个for循环不断循环此函数

- 查找level0层的sstable数量
- 如果level0层这个数量大于等于8（会睡眠1ms）（写入减慢）
- 如果大于12，则会暂停写，trigger tcpmaction
- 统计writedelay

```
tLen := db.s.tLen(0)
mdbFree = mdb.Free()
switch {
case tLen >= slowdownTrigger && !delayed:
   delayed = true
   time.Sleep(time.Millisecond)
case mdbFree >= n:
   return false
case tLen >= pauseTrigger:
   delayed = true
   // Set the write paused flag explicitly.
   atomic.StoreInt32(&db.inWritePaused, 1)
   err = db.compTriggerWait(db.tcompCmdC)
   // Unset the write paused flag.
   atomic.StoreInt32(&db.inWritePaused, 0)
   if err != nil {
      return false
   }
default:
   // Allow memdb to grow if it has no entry.
   if mdb.Len() == 0 {
      mdbFree = n
   } else {
      mdb.decref()
      mdb, err = db.rotateMem(n, false)
      if err == nil {
         mdbFree = mdb.Free()
      } else {
         mdbFree = 0
      }
   }
   return false
}
```

```
func (o *Options) GetWriteL0PauseTrigger() int {
   if o == nil || o.WriteL0PauseTrigger == 0 {
      return DefaultWriteL0PauseTrigger // 12
   }
   return o.WriteL0PauseTrigger
}

func (o *Options) GetWriteL0SlowdownTrigger() int {
   if o == nil || o.WriteL0SlowdownTrigger == 0 {
      return DefaultWriteL0SlowdownTrigger // 8
   }
   return o.WriteL0SlowdownTrigger
}
```

```
// Try to flush memdb. This method would also trying to throttle writes
// if it is too fast and compaction cannot catch-up.
```



#### Get

- 从memdb中读取
- 从frozen中读取
- 获取一个version（），使用version从sstable中读取
- 如果最大无效读取次数超过一定次数，触发tcompation

注意leveldb在每一层sstable中查找数据时，都是按序依次查找sstable的。

0层的文件比较特殊。由于0层的文件中可能存在key重合的情况，因此在0层中，文件编号大的sstable优先查找。理由是文件编号较大的sstable中存储的总是最新的数据。

非0层文件，一层中所有文件之间的key不重合，因此leveldb可以借助sstable的元数据（一个文件中最小与最大的key值）进行快速定位，每一层只需要查找一个sstable文件的内容。

![截屏2020-08-01 下午6.39.56](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-08-01 下午6.39.56.png)



一个version包括各个level的sstable的元数据。

#### version包括

- 最大level层的分数
- 分数最大的level

#### sstable元数据

- 最大和最小的key
- sstable的大小
- 文件句柄
- 还有一个seekLeft（文件大小除以16kb，最多无效读取次数）

```go
// tFile holds basic information about a table.
type tFile struct {
   fd         storage.FileDesc
   seekLeft   int32
   size       int64
   imin, imax internalKey
}
```

```go
type version struct {
   s *session

   levels []tFiles

   // Level that should be compacted next and its compaction score.
   // Score < 1 means compaction is not strictly needed. These fields
   // are initialized by computeCompaction()
   cLevel int 
   cScore float64

   cSeek unsafe.Pointer

   closing  bool
   ref      int
   released bool
}
```

#### 写入流程

- 获取transaction（如果大小大于4m，就不生成日志）

- merge请求
- 获取writeLock
- 尝试flush
- 尝试合并merge请求
- 写入日志
- rotateMem（如果internal len写入的batch数量大于memdb.Free（））
- 释放writeLock
- 往mergeack管道里头发送结果
- mdb unref减小引用。



```go
// If the batch size is larger than write buffer, it may justified to write
// using transaction instead. Using transaction the batch will be written
// into tables directly, skipping the journaling.
// 4M
if batch.internalLen > db.s.o.GetWriteBuffer() && !db.s.o.GetDisableLargeBatchTransaction() {
   tr, err := db.OpenTransaction()
   if err != nil {
      return err
   }
   if err := tr.Write(batch, wo); err != nil {
      tr.Discard()
      return err
   }
   return tr.Commit()
}
```

- 引用计数减小为0，就会reset 这个memdb，然后放回pool里头去。



#### compaction

- opendb的时候开启了两个协程
- 一个叫做mcompaction（不断for循环阻塞在一个管道上， 有值发送过来就执行compaction， 然后往这个cAuto（发送一个nil）， 告诉那边完成了）

- 一个叫做tcompaction（）



- mCompation是被动触发的

```go
func (db *DB) mCompaction() {
   var x cCmd

   defer func() {
      if x := recover(); x != nil {
         if x != errCompactionTransactExiting {
            panic(x)
         }
      }
      if x != nil {
         x.ack(ErrClosed)
      }
      db.closeW.Done()
   }()

   for {
      select {
      case x = <-db.mcompCmdC:
         switch x.(type) {
         case cAuto:
            db.memCompaction()
            x.ack(nil)
            x = nil
         default:
            panic("leveldb: unknown command")
         }
      case <-db.closeC:
         return
      }
   }
}
```





- 得到frozen 的memdb
- 暂停major compation
- 创建sstable
- pickMemdbLevel（）选择要写入的层

- remove frozen的memdb
- 触发major compaction



https://www.cnblogs.com/shijiaqi1066/p/12459678.html

新生成的sstable不一定在第0层

- 如果和第0层的文件有重叠
- 或者与leveii层的文件有重叠
- 或者与level2层的文件重叠的太多 都会加入level0层

![截屏2020-08-01 下午6.18.12](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-08-01 下午6.18.12.png)



在策略上，尽量要将新的compact的文件推到高level。

因为在level 0 需要控制文件过多，compaction IO和查找都比较耗费。

另一方面也不能推至过高level，一定程度上控制查找的次数，而且若某些范围的key更新比较频繁，后续往高层compaction IO消耗也很大。

所以PickLevelForMemTableOutput就是个权衡折中。如果新生成的sstable和Level 0的sstable有交叠，新产生的sstable就直接加入level 0，否则根据一定的策略，向上推到Level1 甚至是Level 2，但是最高推到Level2。

#### tcompation

```
func (v *version) needCompaction() bool {
   return v.cScore >= 1 || atomic.LoadPointer(&v.cSeek) != nil
}
```



- 一个cScore表示
- seek表示无效查询次数
- 以上对于所有的level 层数来说



#### major compaction

major compaction



#### 时机

- get的时候发现无效读取次数过多
- mcompaction之后
- flush的时候文件大于12个
- tablerangecompact



#### tablerangecompaction

输入一个level，以及最小的key和最大的key。

- 得到一个version，找到这个level中overlep对应的key作为compaction的输入。

- 会考虑compation不要太大

- ```go
  // Avoid compacting too much in one shot in case the range is large.
  // But we cannot do this for level-0 since level-0 files can overlap
  // and we must not pick one file and drop another older file if the
  // two files overlap.
  ```

- 这个是用于外部调用。

- ```go
  // CompactRange compacts the underlying DB for the given key range.
  // In particular, deleted and overwritten versions are discarded,
  // and the data is rearranged to reduce the cost of operations
  // needed to access the data. This operation should typically only
  // be invoked by users who understand the underlying implementation.
  //
  // A nil Range.Start is treated as a key before all keys in the DB.
  // And a nil Range.Limit is treated as a key after all keys in the DB.
  // Therefore if both is nil then it will compact entire DB.
  func (db *DB) CompactRange(r util.Range) error {
  ```

#### tcompaction

分为两种

- 当sstable的数量大于12的时候会阻塞等待直到compation完成
- 否则就直接返回
- 后台自动compaction

#### 如果需要compaction

- 先阻塞在tcomCmdC上
- 如果不需要，就遍历waitQueue返回结果。

#### 如果不需要

- 就遍历waitQueue返回结果。（注意不会进行检查）
- 它会把请求加入到一个waitQueue等待队列里头。



- 当它从其管道中收到要求进行major compaction的时候，如果这个l0层的文件数目小于12
- 就结果返回nil，让对面不用等待



- 然后进行autoCompaction



```
for {
   if db.tableNeedCompaction() {
      select {
      case x = <-db.tcompCmdC:
      case ch := <-db.tcompPauseC:
         db.pauseCompaction(ch)
         continue
      case <-db.closeC:
         return
      default:
      }
      // Resume write operation as soon as possible.
      if len(waitQ) > 0 && db.resumeWrite() {
         for i := range waitQ {
            waitQ[i].ack(nil)
            waitQ[i] = nil
         }
         waitQ = waitQ[:0]
      }
   } else {
      for i := range waitQ {
         waitQ[i].ack(nil)
         waitQ[i] = nil
      }
      waitQ = waitQ[:0]
      select {
      case x = <-db.tcompCmdC:
      case ch := <-db.tcompPauseC:
         db.pauseCompaction(ch)
         continue
      case <-db.closeC:
         return
      }
   }
   if x != nil {
      switch cmd := x.(type) {
      case cAuto:
         if cmd.ackC != nil {
            // Check the write pause state before caching it.
            if db.resumeWrite() {
               x.ack(nil)
            } else {
               waitQ = append(waitQ, x)
            }
         }
      case cRange:
         x.ack(db.tableRangeCompaction(cmd.level, cmd.min, cmd.max))
      default:
         panic("leveldb: unknown command")
      }
      x = nil
   }
   db.tableAutoCompaction()
}
```



```
func (db *DB) tableAutoCompaction() {
   if c := db.s.pickCompaction(); c != nil {
      db.tableCompaction(c, false)
   }
}
```

#### pickCompaction

- seek触发的就把对应的sstable加入就好
- size触发的并不是把level i层的所有文件都加入，而是从而size 触发的compaction稍微复杂一点，它需要考虑上一次compaction做到了哪个key，什么地方，然后大于该key的第一个文件即为level n的参与compaction的文件。
- 如果没有的话，就把第一个加入（也就是最旧的）
- 进行合并的层叫做sourceLevel

```go
// Pick a compaction based on current state; need external synchronization.
func (s *session) pickCompaction() *compaction {
   v := s.version()

   var sourceLevel int
   var t0 tFiles
   if v.cScore >= 1 {
      sourceLevel = v.cLevel
      cptr := s.getCompPtr(sourceLevel)
      tables := v.levels[sourceLevel]
      for _, t := range tables {
         if cptr == nil || s.icmp.Compare(t.imax, cptr) > 0 {
            t0 = append(t0, t)
            break
         }
      }
      if len(t0) == 0 {
         t0 = append(t0, tables[0])
      }
   } else {
      if p := atomic.LoadPointer(&v.cSeek); p != nil {
         ts := (*tSet)(p)
         sourceLevel = ts.level
         t0 = append(t0, ts.table)
      } else {
         v.release()
         return nil
      }
   }

   return newCompaction(s, v, sourceLevel, t0)
}
```



#### newCompaction

```go
func newCompaction(s *session, v *version, sourceLevel int, t0 tFiles) *compaction {
   c := &compaction{
      s:             s,
      v:             v,
      sourceLevel:   sourceLevel,
      levels:        [2]tFiles{t0, nil},
      maxGPOverlaps: int64(s.o.GetCompactionGPOverlaps(sourceLevel)),
      tPtrs:         make([]int, len(v.levels)),
   }
   c.expand()
   c.save()
   return c
}
```



- expand
- save



- 取出souveLevel i层的要进行compaction的最小的imin和最大的imin
- 通过这个找到soucelevel i + 1层的所有key范围重叠的文件
- 然后在不扩大i + 1层文件的数量的情况下，尽量扩大i 层/

```go
// Expand compacted tables; need external synchronization.
func (c *compaction) expand() {
   limit := int64(c.s.o.GetCompactionExpandLimit(c.sourceLevel))
   vt0 := c.v.levels[c.sourceLevel]
   vt1 := tFiles{}
   if level := c.sourceLevel + 1; level < len(c.v.levels) {
      vt1 = c.v.levels[level]
   }

   t0, t1 := c.levels[0], c.levels[1]
   imin, imax := t0.getRange(c.s.icmp)
   // We expand t0 here just incase ukey hop across tables.
   t0 = vt0.getOverlaps(t0, c.s.icmp, imin.ukey(), imax.ukey(), c.sourceLevel == 0)
   if len(t0) != len(c.levels[0]) {
      imin, imax = t0.getRange(c.s.icmp)
   }
   t1 = vt1.getOverlaps(t1, c.s.icmp, imin.ukey(), imax.ukey(), false)
   // Get entire range covered by compaction.
   amin, amax := append(t0, t1...).getRange(c.s.icmp)

   // See if we can grow the number of inputs in "sourceLevel" without
   // changing the number of "sourceLevel+1" files we pick up.
   if len(t1) > 0 {
      exp0 := vt0.getOverlaps(nil, c.s.icmp, amin.ukey(), amax.ukey(), c.sourceLevel == 0)
      if len(exp0) > len(t0) && t1.size()+exp0.size() < limit {
         xmin, xmax := exp0.getRange(c.s.icmp)
         exp1 := vt1.getOverlaps(nil, c.s.icmp, xmin.ukey(), xmax.ukey(), false)
         if len(exp1) == len(t1) {
            c.s.logf("table@compaction expanding L%d+L%d (F·%d S·%s)+(F·%d S·%s) -> (F·%d S·%s)+(F·%d S·%s)",
               c.sourceLevel, c.sourceLevel+1, len(t0), shortenb(int(t0.size())), len(t1), shortenb(int(t1.size())),
               len(exp0), shortenb(int(exp0.size())), len(exp1), shortenb(int(exp1.size())))
            imin, imax = xmin, xmax
            t0, t1 = exp0, exp1
            amin, amax = append(t0, t1...).getRange(c.s.icmp)
         }
      }
   }

   // Compute the set of grandparent files that overlap this compaction
   // (parent == sourceLevel+1; grandparent == sourceLevel+2)
   if level := c.sourceLevel + 2; level < len(c.v.levels) {
      c.gp = c.v.levels[level].getOverlaps(c.gp, c.s.icmp, amin.ukey(), amax.ukey(), false)
   }

   c.levels[0], c.levels[1] = t0, t1
   c.imin, c.imax = imin, imax
}
```



![截屏2020-08-01 下午6.01.15](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-08-01 下午6.01.15.png)

1. 红星标注的为起始输入文件；
2. 在level i层中，查找与起始输入文件有key重叠的文件，如图中红线所标注，最终构成level i层的输入文件；

（事实上这只对level0层有效，因为只有level0层会有重叠）

1. 利用level i层的输入文件，在level i+1层找寻有key重叠的文件，结果为绿线标注的文件，构成level i，i+1层的输入文件；
2. 最后利用两层的输入文件，在不扩大level i+1输入文件的前提下，查找level i层的有key重叠的文件，结果为蓝线标准的文件，构成最终的输入文件；

（很重要）



本次选择了第3个文件，如果只是把`[b, e]`更新到level 1，那么就会导致读取时数据错误。因为多个文件之间数据是有重叠的，数据之间的先后无法判断，而更新到1级就意味着认为数据更早。

对应的做法就是当选出文件后，判断还有其他文件有重叠，把这些文件都加入进来，这个例子对应的就是把文件1 2都加进来。

代码上，先通过`GetRange`获取输入文件的键范围，然后根据键范围得到一个最全的文件列表。

https://izualzhy.cn/leveldb-PickCompaction

```go
// Returns tables whose its key range overlaps with given key range.
// Range will be expanded if ukey found hop across tables.
// If overlapped is true then the search will be restarted if umax
// expanded.
// The dst content will be overwritten.
func (tf tFiles) getOverlaps(dst tFiles, icmp *iComparer, umin, umax []byte, overlapped bool) tFiles {
   dst = dst[:0]
   for i := 0; i < len(tf); {
      t := tf[i]
      if t.overlaps(icmp, umin, umax) {
         if umin != nil && icmp.uCompare(t.imin.ukey(), umin) < 0 {
            umin = t.imin.ukey()
            dst = dst[:0]
            i = 0
            continue
         } else if umax != nil && icmp.uCompare(t.imax.ukey(), umax) > 0 {
            umax = t.imax.ukey()
            // Restart search if it is overlapped.
            if overlapped {
               dst = dst[:0]
               i = 0
               continue
            }
         }

         dst = append(dst, t)
      }
      i++
   }

   return dst
}
```



#### 提前结束

我们会把与之前选出的文件有重叠的level i + 2层取出来。

```go
// Compute the set of grandparent files that overlap this compaction
// (parent == sourceLevel+1; grandparent == sourceLevel+2)
if level := c.sourceLevel + 2; level < len(c.v.levels) {
   c.gp = c.v.levels[level].getOverlaps(c.gp, c.s.icmp, amin.ukey(), amax.ukey(), false)
}
```

- 当生成level i + 1层的文件的时候，会判断这个文件是否和grandfather重叠的字节数过多。（重叠的文件超过10个）
- 也会控制leveli + 1层的文件数量，最大为25。

- ```
  db.s.o.GetCompactionTableSize(c.sourceLevel + 1),或者超过了i + 1层的tablesize大小
  
  不过默认都为2m
  ```

```go
func (c *compaction) shouldStopBefore(ikey internalKey) bool {
   for ; c.gpi < len(c.gp); c.gpi++ {
      gp := c.gp[c.gpi]
      if c.s.icmp.Compare(ikey, gp.imax) <= 0 {
         break
      }
      if c.seenKey {
         c.gpOverlappedBytes += gp.size
      }
   }
   c.seenKey = true

   if c.gpOverlappedBytes > c.maxGPOverlaps {
      // Too much overlap for current output; start new output.
      c.gpOverlappedBytes = 0
      return true
   }
   return false
}
```



```
maxGPOverlaps: int64(s.o.GetCompactionGPOverlaps(sourceLevel)),
```

https://izualzhy.cn/leveldb-PickCompaction



```
func (o *Options) GetCompactionGPOverlaps(level int) int {
   factor := DefaultCompactionGPOverlapsFactor
   if o != nil && o.CompactionGPOverlapsFactor > 0 {
      factor = o.CompactionGPOverlapsFactor
   }
   return o.GetCompactionTableSize(level+2) * factor // 10
}
```