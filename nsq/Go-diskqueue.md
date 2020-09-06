## Go-diskqueue

nsq通过diskqueue提供对消息的持久化。

![未命名](/Users/jieyang/Downloads/未命名.png)

- 通过writechan和writeResponsechan同步处理数据io。
- 通过定时器synctimeout和syncevery控制sync频率
- 每一轮for循环开头都会优先读取数据？



- 通过readpos和readfilenum判断是否还有数据可以读取



```go
// ioLoop provides the backend for exposing a go channel (via ReadChan())
// in support of multiple concurrent queue consumers
//
// it works by looping and branching based on whether or not the queue has data
// to read and blocking until data is either read or written over the appropriate
// go channels
//
// conveniently this also means that we're asynchronously reading from the filesystem
func (d *diskQueue) ioLoop() {
   var dataRead []byte
   var err error
   var count int64
   var r chan []byte

   syncTicker := time.NewTicker(d.syncTimeout)

   for {
      // dont sync all the time :)
      if count == d.syncEvery {
         d.needSync = true
      }

      if d.needSync {
         err = d.sync()
         if err != nil {
            d.logf(ERROR, "DISKQUEUE(%s) failed to sync - %s", d.name, err)
         }
         count = 0
      }

      if (d.readFileNum < d.writeFileNum) || (d.readPos < d.writePos) {
         if d.nextReadPos == d.readPos {
            dataRead, err = d.readOne()
            if err != nil {
               d.logf(ERROR, "DISKQUEUE(%s) reading at %d of %s - %s",
                  d.name, d.readPos, d.fileName(d.readFileNum), err)
               d.handleReadError()
               continue
            }
         }
         r = d.readChan
      } else {
         r = nil
      }

      select {
      // the Go channel spec dictates that nil channel operations (read or write)
      // in a select are skipped, we set r to d.readChan only when there is data to read
      case r <- dataRead:
         count++
         // moveForward sets needSync flag if a file is removed
         d.moveForward()
      case <-d.emptyChan:
         d.emptyResponseChan <- d.deleteAllFiles()
         count = 0
      case dataWrite := <-d.writeChan:
         count++
         d.writeResponseChan <- d.writeOne(dataWrite)
      case <-syncTicker.C:
         if count == 0 {
            // avoid sync when there's no activity
            continue
         }
         d.needSync = true
      case <-d.exitChan:
         goto exit
      }
   }

exit:
   d.logf(INFO, "DISKQUEUE(%s): closing ... ioLoop", d.name)
   syncTicker.Stop()
   d.exitSyncChan <- 1
}
```

#### option

```go
MaxBytesPerFile: 100 * 1024 * 1024,
SyncEvery:       2500,
SyncTimeout:     2 * time.Second,
```

#### sync

- syncevery，每写入活着读取2500个文件
- 每2s
- read读取成功，删除旧文件
- 发生了读取错误活着tailcorruption等元数据损坏行为



- 系统调用sync writefile然后close，writefile变为nil

- presistmetadata，持久化元数据  

- 获取元数据文件目录名 path.Join(d.dataPath, "%s.diskqueue.meta.dat"), d.name)

- 以随机数和tmp后缀和0600权限生成一个临时文件，写入临时文件，sync和rename。fmt.Sprintf("%s.%d.tmp", fileName, rand.Int())

- 元数据包括

  ```go
  fmt.Fprintf(f, "%d\n%d,%d\n%d,%d\n",
     atomic.LoadInt64(&d.depth),
     d.readFileNum, d.readPos,
     d.writeFileNum, d.writePos)
  ```

#### writeone

- 通过minMsgSize和maxMsgSize限定写入数据大小
- 以bigendian的形式先写入大小，再写入数据到writeBuf，每写一次重制writebuf，只写一次。
- 将数据写入writeFile，以writePos代表写入了多少数据。如果大于maxBytesPerFile（100m）则sync（包括元数据和文件）。以writefilenum表示写入文件名，递增。



- 理论上啊来讲，我们写入一条数据，数据大小还没有达到最大。
- 如果服务器突然断电，这个数据是没有sync到磁盘里头的。
- 而且元数据也不会记录。



- 注意只写入file once，如果writebuf写入err！= 0，表示没有完全写入，那么会writefile置为nil。本次失败。
- 下一次写入的时候生成会重新打开相同的文件。然后使用seek移动到同样的位置。

```
func (d *diskQueue) fileName(fileNum int64) string {
   return fmt.Sprintf(path.Join(d.dataPath, "%s.diskqueue.%06d.dat"), d.name, fileNum)
}
```



#### 要点

- 写入最多一次语意

- 解藕，一个专门处理当writefile为nil时候的情况，这包括文件名，打开权限，日志记录等等。
- 一个专门处理文件过大需要sync的情况。
- 而之后的则按写入writebuf，写入wirtefile，修改元数据writepos递增进行，出现文件直接置换writefile为nil，方便快捷。



#### readone

- readone
- 使用readpos表示从文件读取的位置，nextreadpos表示下一次从文件读取的位置。
- readone会advance提前更改nextreadpos和nextreadfilenum；
- 只要当文件内容被成功传进管道了，才会moveforward，真正更改readpos为nextreadpos，更改readfilenum，并且删除旧文件，设置sync标示。
- 最多一次或者恰好一次读取语意。
- 通过bufio.NewReader()读取数据，每次4kb

```gogo
func (d *diskQueue) moveForward() {
   oldReadFileNum := d.readFileNum
   d.readFileNum = d.nextReadFileNum
   d.readPos = d.nextReadPos
   depth := atomic.AddInt64(&d.depth, -1)

   // see if we need to clean up the old file
   if oldReadFileNum != d.nextReadFileNum {
      // sync every time we start reading from a new file
      d.needSync = true

      fn := d.fileName(oldReadFileNum)
      err := os.Remove(fn)
      if err != nil {
         d.logf(ERROR, "DISKQUEUE(%s) failed to Remove(%s) - %s", d.name, fn, err)
      }
   }

   d.checkTailCorruption(depth)
}
```



#### 丢失

- 理论上来讲，如果写入文件后，系统断电，那么元数据会来不及持久化到磁盘；对于读取，如果数据发送给用户后断电，也会来不及持久化到磁盘。
- 这样下次重启的时候就会retrieveMetaData（），重新在相同的位置写入和读取。

#### corruption

- 通过minmsgsize和maxmsgsize判断文件是否损坏和限制写入大小。
- 如果写入大小不变，那么判断文件大小是否提前终止。



- 使用truncate模拟文件损坏



- 调用readchan读取时，错误会被隐藏在底层。

  

- 生成4个文件，前三个文件8个message。最后一个一个message。

- 第二个文件truncate，前三个message有效。读取19个文件直到剩一个文件在第四个文件。

- truncate损坏第四个。

- 随后一个新的for循环会调用readone，于是从第五个文件写入和读取。

- 随后的put会写入第五个文件。

- 然后往第五个文件里头写入损坏的内容。

  

```go
func TestDiskQueueCorruption(t *testing.T) {
   l := NewTestLogger(t)
   dqName := "test_disk_queue_corruption" + strconv.Itoa(int(time.Now().Unix()))
   tmpDir, err := ioutil.TempDir("", fmt.Sprintf("nsq-test-%d", time.Now().UnixNano()))
   if err != nil {
      panic(err)
   }
   defer os.RemoveAll(tmpDir)
   // require a non-zero message length for the corrupt (len 0) test below
   dq := New(dqName, tmpDir, 1000, 10, 1<<10, 5, 2*time.Second, l)
   defer dq.Close()

   msg := make([]byte, 123) // 127 bytes per message, 8 (1016 bytes) messages per file
   for i := 0; i < 25; i++ {
      dq.Put(msg)
   }

   Equal(t, int64(25), dq.Depth())

   // corrupt the 2nd file
   dqFn := dq.(*diskQueue).fileName(1)
   os.Truncate(dqFn, 500) // 3 valid messages, 5 corrupted

   for i := 0; i < 19; i++ { // 1 message leftover in 4th file
      Equal(t, msg, <-dq.ReadChan())
   }

   // corrupt the 4th (current) file
   dqFn = dq.(*diskQueue).fileName(3)
   os.Truncate(dqFn, 100)

   dq.Put(msg) // in 5th file

   Equal(t, msg, <-dq.ReadChan())

   // write a corrupt (len 0) message at the 5th (current) file
   dq.(*diskQueue).writeFile.Write([]byte{0, 0, 0, 0})

   // force a new 6th file - put into 5th, then readOne errors, then put into 6th
   dq.Put(msg)
   dq.Put(msg)

   Equal(t, msg, <-dq.ReadChan())
}
```

#### handlereaderror

当读取时出现错误了。

- 如果readfilenum等于writefilenum，则会自增writefilenum。

```go
// if you can't properly read from the current write file it's safe to
// assume that something is fucked and we should skip the current file too
```

- rename bad file，自增readfilenum，跳到读取下一个文件。

```go
// jump to the next read file and rename the current (bad) file
```



#### moveforward

- 真正改变readFileNum和readPos，depth--
- 如果还有下个文件，那么needsync = true，需要修改元数据
- 移除旧文件。

```go
func (d *diskQueue) moveForward() {
   oldReadFileNum := d.readFileNum
   d.readFileNum = d.nextReadFileNum
   d.readPos = d.nextReadPos
   depth := atomic.AddInt64(&d.depth, -1)

   // see if we need to clean up the old file
   if oldReadFileNum != d.nextReadFileNum {
      // sync every time we start reading from a new file
      d.needSync = true

      fn := d.fileName(oldReadFileNum)
      err := os.Remove(fn)
      if err != nil {
         d.logf(ERROR, "DISKQUEUE(%s) failed to Remove(%s) - %s", d.name, fn, err)
      }
   }

   d.checkTailCorruption(depth)
}
```

#### checktailcorruption

- 每读取一次文件会递减depth，写入文件则会增加depth，每次moveforward成功读取就会检查depth
- 如果由于depth不为0引起的，记录错误。
- 并且会记录由于readFilenum和readpos不相同引起的错误。



- 任意一个readfilenum或者readpos大于（表示两者都不小于），代表读取到了文件末尾，但是却没有读取那么多的文件。（因为handlereadwrror仅仅是跳过当前损坏的文件）。我们就会skiptonextrwfile，重置depth为0，设置sync/

```go
func (d *diskQueue) checkTailCorruption(depth int64) {
   if d.readFileNum < d.writeFileNum || d.readPos < d.writePos {
      return
   }

   // we've reached the end of the diskqueue
   // if depth isn't 0 something went wrong
   if depth != 0 {
      if depth < 0 {
         d.logf(ERROR,
            "DISKQUEUE(%s) negative depth at tail (%d), metadata corruption, resetting 0...",
            d.name, depth)
      } else if depth > 0 {
         d.logf(ERROR,
            "DISKQUEUE(%s) positive depth at tail (%d), data loss, resetting 0...",
            d.name, depth)
      }
      // force set depth 0
      atomic.StoreInt64(&d.depth, 0)
      d.needSync = true
   }

   if d.readFileNum != d.writeFileNum || d.readPos != d.writePos {
      if d.readFileNum > d.writeFileNum {
         d.logf(ERROR,
            "DISKQUEUE(%s) readFileNum > writeFileNum (%d > %d), corruption, skipping to next writeFileNum and resetting 0...",
            d.name, d.readFileNum, d.writeFileNum)
      }

      if d.readPos > d.writePos {
         d.logf(ERROR,
            "DISKQUEUE(%s) readPos > writePos (%d > %d), corruption, skipping to next writeFileNum and resetting 0...",
            d.name, d.readPos, d.writePos)
      }

      d.skipToNextRWFile()
      d.needSync = true
   }
}
```

```go
skipToNextRWFile
```



- 当checktailcorruption或者deleteallfiles时使用，删除所有已写入文件。
- writefilenum++表示从下一个文件开始写入。



#### benchmark

BenchmarkDiskQueuePut1048576-8        	     140	  25953213 ns/op	  40.40 MB/s

BenchmarkDiskWrite1048576-8           	      46	  28506507 ns/op	  36.78 MB/s

BenchmarkDiskWriteBuffered1048576-8   	      37	  29790324 ns/op	  35.20 MB/s

BenchmarkDiskQueueGet1048576-8        	      27	  43278763 ns/op	  24.23 MB/s



- queuePut是一个for 循环写入diskqueue
- diskwrite是一个for循环写入文件
- diskwritebuffered是通过buffer写入文件
- diskqueueuget是for循环从diskqueue里头读取。



BenchmarkDiskQueueGet16-8             	   18039	     68005 ns/op	   0.24 MB/s

BenchmarkDiskWrite16-8                	  216861	      5017 ns/op	   3.19 MB/s

BenchmarkDiskWriteBuffered16-8        	 2854642	       448 ns/op	  35.70 MB/s

BenchmarkDiskQueuePut16-8             	   41624	     24888 ns/op	   0.64 MB/s



测试的目的

- 了解写入diskqueue和直接写入的性能区别，明显小文件时更明显，因为不用每个信息都sync
- 而通过buffer的write明显小文件更快，因为不用每个信息都系统调用写入。

从2的四次方到2的20次方。

间隔为4。

也就是标准的16字节到2m做测试。



#### empty

当从emptychan中触发时。

- skiptoNextRWFILE
- 移除metadata元数据，只保留内存中的。
- 注意skiptonextfilenum不会设置sync



- skiptonextRwfile。会把当前writefilenum之前的所有文件删除
- 然后重新设置depth为0，相当于重启。

func (d *diskQueue) deleteAllFiles() error {
   err := d.skipToNextRWFile()

   innerErr := os.Remove(d.metaDataFileName())
   if innerErr != nil && !os.IsNotExist(innerErr) {
      d.logf(ERROR, "DISKQUEUE(%s) failed to remove metadata file - %s", d.name, innerErr)
      return innerErr
   }

   return err
}

func (d *diskQueue) skipToNextRWFile() error {
   var err error

   if d.readFile != nil {
      d.readFile.Close()
      d.readFile = nil
   }

   if d.writeFile != nil {
      d.writeFile.Close()
      d.writeFile = nil
   }

   for i := d.readFileNum; i <= d.writeFileNum; i++ {
      fn := d.fileName(i)
      innerErr := os.Remove(fn)
      if innerErr != nil && !os.IsNotExist(innerErr) {
         d.logf(ERROR, "DISKQUEUE(%s) failed to remove data file - %s", d.name, innerErr)
         err = innerErr
      }
   }

   d.writeFileNum++
   d.writePos = 0
   d.readFileNum = d.writeFileNum
   d.readPos = 0
   d.nextReadFileNum = d.writeFileNum
   d.nextReadPos = 0
   atomic.StoreInt64(&d.depth, 0)

   return err
}

#### test

测试元数据是否会在写入后sync。

```
TestDiskQueueSyncAfterRead
```





```go

```



- 开启四个线程隔一段时间写入数据
- 写入完毕后关闭再重新打开
- 开启四个线程隔一段时间取出数据。

```go
func TestDiskQueueTorture(t *testing.T) {
   var wg sync.WaitGroup

   l := NewTestLogger(t)
   dqName := "test_disk_queue_torture" + strconv.Itoa(int(time.Now().Unix()))
   tmpDir, err := ioutil.TempDir("", fmt.Sprintf("nsq-test-%d", time.Now().UnixNano()))
   if err != nil {
      panic(err)
   }
   defer os.RemoveAll(tmpDir)
   dq := New(dqName, tmpDir, 262144, 0, 1<<10, 2500, 2*time.Second, l)
   NotNil(t, dq)
   Equal(t, int64(0), dq.Depth())

   msg := []byte("aaaaaaaaaabbbbbbbbbbccccccccccddddddddddeeeeeeeeeeffffffffff")

   numWriters := 4
   numReaders := 4
   readExitChan := make(chan int)
   writeExitChan := make(chan int)

   var depth int64
   for i := 0; i < numWriters; i++ {
      wg.Add(1)
      go func() {
         defer wg.Done()
         for {
            time.Sleep(100000 * time.Nanosecond)
            select {
            case <-writeExitChan:
               return
            default:
               err := dq.Put(msg)
               if err == nil {
                  atomic.AddInt64(&depth, 1)
               }
            }
         }
      }()
   }

   time.Sleep(1 * time.Second)

   dq.Close()

   t.Logf("closing writeExitChan")
   close(writeExitChan)
   wg.Wait()

   t.Logf("restarting diskqueue")

   dq = New(dqName, tmpDir, 262144, 0, 1<<10, 2500, 2*time.Second, l)
   defer dq.Close()
   NotNil(t, dq)
   Equal(t, depth, dq.Depth())

   var read int64
   for i := 0; i < numReaders; i++ {
      wg.Add(1)
      go func() {
         defer wg.Done()
         for {
            time.Sleep(100000 * time.Nanosecond)
            select {
            case m := <-dq.ReadChan():
               Equal(t, m, msg)
               atomic.AddInt64(&read, 1)
            case <-readExitChan:
               return
            }
         }
      }()
   }

   t.Logf("waiting for depth 0")
   for {
      if dq.Depth() == 0 {
         break
      }
      time.Sleep(50 * time.Millisecond)
   }

   t.Logf("closing readExitChan")
   close(readExitChan)
   wg.Wait()

   Equal(t, depth, read)
}
```