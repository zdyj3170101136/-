#### transaction

```go
// If the batch size is larger than write buffer, it may justified to write
// using transaction instead. Using transaction the batch will be written
// into tables directly, skipping the journaling.
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

- 过大的时候会开启transaction



```go
b.replayInternal(func(i int, kt keyType, k, v []byte) error {
   return tr.put(kt, k, v)
})
```

- transaction的实质是遍历batch数据，不断的调用一个函数put k， v数据

```go
func (b *Batch) replayInternal(fn func(i int, kt keyType, k, v []byte) error) error {
   for i, index := range b.index {
      if err := fn(i, index.keyType, index.k(b.data), index.v(b.data)); err != nil {
         return err
      }
   }
   return nil
}
```



```go
func (tr *Transaction) put(kt keyType, key, value []byte) error {
   tr.ikScratch = makeInternalKey(tr.ikScratch, key, tr.seq+1, kt)
   if tr.mem.Free() < len(tr.ikScratch)+len(value) {
      if err := tr.flush(); err != nil {
         return err
      }
   }
   if err := tr.mem.Put(tr.ikScratch, value); err != nil {
      return err
   }
   tr.seq++
   return nil
}
```



- 相比较非transaction写memdb往往是写一群之后再增加sequence。
- put是往一个memdb中写了之后立即增加。因为这不涉及到原子变量的访问。另外这样子写10个key，因为sequence所以会不相同。



- 如果memdb大小超过一定数量，就会直接生成一个sstable，然后写入数据，和同步到磁盘
- 文件句柄保存在内存中。（没有frozen以及后台同步了）
- 新生成的sstable全部为level 0 层

#### discard

- 如果write发生错误，就会discard然后移除文件句柄
- reuse 文件编号

```go
func (tr *Transaction) discard() {
   // Discard transaction.
   for _, t := range tr.tables {
      tr.db.logf("transaction@discard @%d", t.fd.Num)
      if err1 := tr.db.s.stor.Remove(t.fd); err1 == nil {
         tr.db.s.reuseFileNum(t.fd.Num)
      }
   }
}
```

#### commit

```go
// Commit commits the transaction. If error is not nil, then the transaction is
// not committed, it can then either be retried or discarded.
//
// Other methods should not be called after transaction has been committed.
func (tr *Transaction) Commit() error {
   if err := tr.db.ok(); err != nil {
      return err
   }

   tr.lk.Lock()
   defer tr.lk.Unlock()
   if tr.closed {
      return errTransactionDone
   }
   if err := tr.flush(); err != nil {
      // Return error, lets user decide either to retry or discard
      // transaction.
      return err
   }
   if len(tr.tables) != 0 {
      // Committing transaction.
      tr.rec.setSeqNum(tr.seq)
      tr.db.compCommitLk.Lock()
      tr.stats.startTimer()
      var cerr error
      for retry := 0; retry < 3; retry++ {
         cerr = tr.db.s.commit(&tr.rec)
         if cerr != nil {
            tr.db.logf("transaction@commit error R·%d %q", retry, cerr)
            select {
            case <-time.After(time.Second):
            case <-tr.db.closeC:
               tr.db.logf("transaction@commit exiting")
               tr.db.compCommitLk.Unlock()
               return cerr
            }
         } else {
            // Success. Set db.seq.
            tr.db.setSeq(tr.seq)
            break
         }
      }
      tr.stats.stopTimer()
      if cerr != nil {
         // Return error, lets user decide either to retry or discard
         // transaction.
         return cerr
      }

      // Update compaction stats. This is safe as long as we hold compCommitLk.
      tr.db.compStats.addStat(0, &tr.stats)

      // Trigger table auto-compaction.
      tr.db.compTrigger(tr.db.tcompCmdC)
      tr.db.compCommitLk.Unlock()

      // Additionally, wait compaction when certain threshold reached.
      // Ignore error, returns error only if transaction can't be committed.
      tr.db.waitCompaction()
   }
   // Only mark as done if transaction committed successfully.
   tr.setDone()
   return nil
}
```

- commit其实就是生成一个修改db的旧version生成新的version
- 如果不成功等待1s重试
- 如果成功就修改db的sequence



- 触发tcompaction，等待compaction结束



```go
tr := &Transaction{
   db:  db,
   seq: db.seq,
   mem: db.mpoolGet(0),
}
```

- transaction也需要获取writelockC
- 使用当前的sequence



- 为什么transaction还需要先生成memtable
- 因为memtable会对同一个key做排序之类的操作。



#### transaction

- transaction也需要获取写锁
- transaction是原子性的操作
- expensive，因为生成大量的0层文件，compaction压力很大。而且当compaction的size很小的时候，sstable的大写也会变小。
- 这样子的compaction代价是不划算的。

```go
// OpenTransaction opens an atomic DB transaction. Only one transaction can be
// opened at a time. Subsequent call to Write and OpenTransaction will be blocked
// until in-flight transaction is committed or discarded.
// The returned transaction handle is safe for concurrent use.
//
// Transaction is expensive and can overwhelm compaction, especially if
// transaction size is small. Use with caution.
//
// The transaction must be closed once done, either by committing or discarding
// the transaction.
// Closing the DB will discard open transaction.
```



#### mpoll

- 如果mpoll里头为空，那我们会新建一个memdb并且引用计数为1
- 当我们释放memdb时，引用计数减少，如果为0，就塞到chan里头去（decref会在get，put等函数里头被调用）。



- 也许是因为就两个memdb一个now的一个frozen的
- 而get的线程释放完后就会消除，内存中存放着最多十几个？

- 也许是因为reset的时候会把一切恢复成原样，这样我们申请的时候处理的不是几m而可能就是几十字节而已。



-  mpoll通过一个大小为1的chan保持内存中最多有一个4m大小的memdb（调用skiplist函数来创建）。

- 如果对应的memdb的ref引用为1，我们可以直接reset来使用。

```go
func (db *DB) mpoolPut(mem *memdb.DB) {
   if !db.isClosed() {
      select {
      case db.memPool <- mem:
      default:
      }
   }
}

func (db *DB) mpoolGet(n int) *memDB {
   var mdb *memdb.DB
   select {
   case mdb = <-db.memPool:
   default:
   }
   if mdb == nil || mdb.Capacity() < n {
      mdb = memdb.New(db.s.icmp, maxInt(db.s.o.GetWriteBuffer(), n))
   }
   return &memDB{
      db: db,
      DB: mdb,
   }
}
```