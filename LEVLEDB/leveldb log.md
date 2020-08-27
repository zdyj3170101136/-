#### 日志



log所写的数据

表示batchdata。

betchdata：

data：keytype（一字节）+ varintlen（lenkey）+key + varintlen（lenvalue）+ value

- 只包括batch的key + type和value部分还有len长度
- 不包括sequence（和skiplist的不同）

```go
func writeBatchesWithHeader(wr io.Writer, batches []*Batch, seq uint64) error {
   if _, err := wr.Write(encodeBatchHeader(nil, seq, batchesLen(batches))); err != nil {
      return err
   }
   for _, batch := range batches {
      if _, err := wr.Write(batch.data); err != nil {
         return err
      }
   }
   return nil
}
```

```go
// Batch is a write batch.
type Batch struct {
   data  []byte
   index []batchIndex

   // internalLen is sums of key/value pair length plus 8-bytes internal key.
   internalLen int
}
```



- 而这两个函数返回的都是纯的的key和value

```go
func (index batchIndex) k(data []byte) []byte {
   return data[index.keyPos : index.keyPos+index.keyLen]
}

func (index batchIndex) v(data []byte) []byte {
   if index.valueLen != 0 {
      return data[index.valuePos : index.valuePos+index.valueLen]
   }
   return nil
}
```

```go
type batchIndex struct {
   keyType            keyType
   keyPos, keyLen     int
   valuePos, valueLen int
}
```

#### 删除日志

而新生成的immutable memory db则会由后台的minor compaction进程将其转换成一个sstable文件进行持久化，持久化完成，与之对应的frozen log被删除。（持久化完成后，对应的frozen log会被删除）



注意。

当我们newMemdb的时候。

旧的log的文件句柄会被赋值给frozenlog，然后删除。

其实就是调用文件系统的remove接口。

```go
if db.journal == nil {
   db.journal = journal.NewWriter(w)
} else {
   db.journal.Reset(w)
   db.journalWriter.Close()
   db.frozenJournalFd = db.journalFd
}
```



https://leveldb-handbook.readthedocs.io/zh/latest/rwopt.html



![截屏2020-07-24 下午1.44.04](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-24 下午1.44.04.png)

为了增加读取效率，日志文件中按照block进行划分，每个block的大小为32KiB。每个block中包含了若干个完整的chunk。

一条日志记录包含一个或多个chunk。

- 每个chunk包含了一个7字节大小的header，前4字节是该chunk的校验码，紧接的2字节是该chunk数据的长度（little endian）
- 以及最后一个字节是该chunk的类型。其中checksum校验的范围包括chunk的类型以及随后的data数据。（不包括长度len）

chunk共有四种类型：full，first，middle，last。一条日志记录若只包含一个chunk，则该chunk的类型为full。若一条日志记录包含多个chunk，则这些chunk的第一个类型为first, 最后一个类型为last，中间包含大于等于0个middle类型的chunk。

```
a 2 byte little-endian uint16 length,
```

由于一个block的大小为32KiB，因此当一条日志文件过大时，会将第一部分数据写在第一个block中，且类型为first，若剩余的数据仍然超过一个block的大小，则第二部分数据写在第二个block中，类型为middle，最后剩余的数据写在最后一个block中，类型为last。



只需要1次磁盘IO，再加1次内存操作。

![截屏2020-07-24 下午2.07.35](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-24 下午2.07.35.png)

注意这个sequence number和internal key的sequence number相同。

但区别是journal中的sequence number是旧的，毕竟是先写日志。



```go
// Seq number.
seq := db.seq + 1

// Write journal.
if err := db.writeJournal(batches, seq, sync); err != nil {
   db.unlockWrite(overflow, merged, err)
   return err
}

// Put batches.
for _, batch := range batches {
   if err := batch.putMem(seq, mdb.DB); err != nil {
      panic(err)
   }
   seq += uint64(batch.Len())
}

// Incr seq number.
db.addSeq(uint64(batchesLen(batches)))
```

日志内容为 batch编码信息（注意是batch编码信息，不是internal key）。。。

#### 日志和memdb不同的一点

batch为key， 而memedb为sequence number。

- Sequence number(自增加一)
- entry number(包含的put 和 delete个数）

![img](https://leveldb-handbook.readthedocs.io/zh/latest/_images/batch.jpeg)

实际中所有的batch都会编码成以上的字节流。

如果不是delete操作还会加上value大小。



（和internalkey不同的是没有key和value的大小。以及对应的type）

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