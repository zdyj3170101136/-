#### merge



在writeJurnal之前从

- 从writeMergeC（struct{}, 0）中接受值（既可以是一个put，也可以是batch）（batch最多1 << 20字节）
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

- 然后等待writeMergedC中接受值
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