

#### datastore

我们在一个叫做.ipfs的不可见目录下存储所有文件。

对于块，我们存储在blocks目录下。

![截屏2020-07-25 下午7.48.13](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-25 下午7.48.13.png)



当我们new一个blockstore的时候。

会在blocks这个目录下创建三个文件：

SHARDING: 表明使用了那个分片函数

_README： readme文件

diskUsage.cache：json序列化的当前磁盘的一个使用量以及精确级别accuracy。

![截屏2020-07-25 下午7.51.31](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-25 下午7.51.31.png)

```
err = WriteShardFunc(path, fun)
if err != nil {
   return err
}
err = WriteReadme(path, fun)
```

#### PUT, DELETE, RENAME 文件

- 首先我们尝试对这个key获得一个operation结构体。

- 我们把操作给它包装成operation这个结构体：

包含操作的type，key，value，tmp临时文件名，path以及真实文件名。



然后根据operation这个结构体的类型内部去调用不同的函数

- put
- delete
- rename

```
func (fs *Datastore) doOp(oper *op) error {
   switch oper.typ {
   case opPut:
      return fs.doPut(oper.key, oper.v)
   case opDelete:
      return fs.doDelete(oper.key)
   case opRename:
      return fs.renameAndUpdateDiskUsage(oper.tmp, oper.path)
   default:
      panic("bad operation, this is a bug")
   }
}
// op wraps useful arguments of write operations
type op struct {
	typ  opT           // operation type
	key  datastore.Key // datastore key. Mandatory.
	tmp  string        // temp file path
	path string        // file path
	v    []byte        // value
}

```



- 尝试获取一个opresult，用来存储write操作的结果的

```
// doWrite optimizes out write operations (put/delete) to the same
// key by queueing them and succeeding all queued
// operations if one of them does. In such case,
// we assume that the first succeeding operation
// on that key was the last one to happen after
// all successful others.
func (fs *Datastore) doWriteOp(oper *op) error {
   keyStr := oper.key.String()

   opRes := fs.opMap.Begin(keyStr)
   if opRes == nil { // nothing to do, a concurrent op succeeded
      return nil
   }

   // Do the operation
   err := fs.doOp(oper)

   // Finish it. If no error, it will signal other operations
   // waiting on this result to succeed. Otherwise, they will
   // retry.
   opRes.Finish(err == nil)
   return err
}
```

- 如何获取opresult呢？（其实就是可能多个协程同一个key进行删除和write）。



- 首先我们有一个sync.map里头存（key， opresult）对。
- opresult包括一个读写锁，我们先对opresult获取一个写锁。

- 从sync.map里头根据key查找是否有其他的opresult
- 如果没有，就返回我们的opresult
- 如果有，我们就尝试获得这个正在进行的opreration的读锁。
- 读取它的操作是否成功，如果它成功我们就返回nil，不用重复做。
- 如果不成功，那就做我们的operation
- 结束的时候释放写锁，记录我们操作的结果（因此之前的读锁，只有当另一个操作已经完成的时候才能执行）。

```
type opResult struct {
   mu      sync.RWMutex
   success bool

   opMap *opMap
   name  string
}

// Returns nil if there's nothing to do.
func (m *opMap) Begin(name string) *opResult {
   for {
      myOp := &opResult{opMap: m, name: name}
      myOp.mu.Lock()
      opIface, loaded := m.ops.LoadOrStore(name, myOp)
      if !loaded { // no one else doing ops with this key
         return myOp
      }

      op := opIface.(*opResult)
      // someone else doing ops with this key, wait for
      // the result
      op.mu.RLock()
      if op.success {
         return nil
      }

      // if we are here, we will retry the operation
   }
}
```

#### 从cid转换成文件名（content identifier内容标识符）

![截屏2020-07-25 下午7.37.44](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-25 下午7.37.44.png)

首先通过key，这个key指的是块的cid。

然后我们通过这个key获得存储的目录以及path。



- 一个目录下文件太大，所以我们会使用一个next-to-last shardfunc， 获得原始的cid的最后两位
- 给cid加上.data后缀

```
func (fs *Datastore) encode(key datastore.Key) (dir, file string) {
   noslash := key.String()[1:]
   dir = filepath.Join(fs.path, fs.getDir(noslash))
   file = filepath.Join(dir, noslash+extension)
   return dir, file
}

func NextToLast(suffixLen int) *ShardIdV1 {
	padding := strings.Repeat("_", suffixLen+1)
	return &ShardIdV1{
		funName: "next-to-last",
		param:   suffixLen,
		fun: func(noslash string) string {
			str := padding + noslash
			offset := len(str) - suffixLen - 1
			return str[offset : offset+suffixLen]
		},
	}
} `
```

#### PUT 函数 block流程

- 在当前目录下建立一个“put-”的临时文件。

- 把内容写到临时文件里头去。
- 关闭临时文件
- 然后rename临时文件
- 最后syncdir
- 使用一个defer函数，用于当临时文件没有关闭或者没有removed的时候关闭或者removed。

```
func (fs *Datastore) doPut(key datastore.Key, val []byte) error {

   dir, path := fs.encode(key)
   if err := fs.makeDir(dir); err != nil {
      return err
   }

   tmp, err := ioutil.TempFile(dir, "put-")
   if err != nil {
      return err
   }
   closed := false
   removed := false
   defer func() {
      if !closed {
         // silence errcheck
         _ = tmp.Close()
      }
      if !removed {
         // silence errcheck
         _ = os.Remove(tmp.Name())
      }
   }()

   if _, err := tmp.Write(val); err != nil {
      return err
   }
   if fs.sync {
      if err := syncFile(tmp); err != nil {
         return err
      }
   }
   if err := tmp.Close(); err != nil {
      return err
   }
   closed = true

   err = fs.renameAndUpdateDiskUsage(tmp.Name(), path)
   if err != nil {
      return err
   }
   removed = true

   if fs.sync {
      if err := syncDir(dir); err != nil {
         return err
      }
   }
   return nil
}
```

#### diskusage 

当我们开启一个打开这个blocks文件的时候。



首先会统计磁盘使用量。

- 我们找出blocks这个目录下的所有文件。

- 用洗牌算法打乱顺序。
- 遇到文件加上文件的大小，遇到目录就加上目录的大小



为了防止这个目录过大。

- 最多扫描2000个文件
- 花的时间超过5s



因此，我们写进diskusage.cache的时候还要带上这个精确级别：

- exact 表示精确
- timeout 表示超时
- approx 表示估计，扫描超过2000个文件



随后开启一个goroutine， 循环阻塞在两个chan上面。

- 一个是定时器管道， 2s。
- 一个是struct{}管道

当我们创建新的目录

或者建立一个新的文件的时候

或者删除一个旧文件就会往这个管道中赛值。

然后有一个变量叫做diskusage（内存中），我们atomic原子操作改变它的值。

（然后如果这段时间， 这个改变量超过百分之一， 我们就更新磁盘里头的文件存储的值）。



- 创建一个叫做du-的临时文件
- 往里头写进json序列化的精确级别以及估计的字节大小。
- 然后改变文件名

```
func (fs *Datastore) checkpointDiskUsage() {
   select {
   case fs.checkpointCh <- struct{}{}:
      // msg sent
   default:
      // checkpoint request already pending
   }
}

func (fs *Datastore) checkpointLoop() {
   defer close(fs.done)

   timerActive := true
   timer := time.NewTimer(0)
   defer timer.Stop()
   for {
      select {
      case _, more := <-fs.checkpointCh:
         du := atomic.LoadInt64(&fs.diskUsage)
         fs.dirty = true
         if !more { // shutting down
            fs.writeDiskUsageFile(du, true)
            if fs.dirty {
               log.Error("could not store final value of disk usage to file, future estimates may be inaccurate")
            }
            return
         }
         // If the difference between the checkpointed disk usage and
         // current one is larger than than `diskUsageCheckpointPercent`
         // of the checkpointed: store it.
         newDu := float64(du)
         lastCheckpointDu := float64(fs.storedValue.DiskUsage)
         diff := math.Abs(newDu - lastCheckpointDu)
         if lastCheckpointDu*diskUsageCheckpointPercent < diff*100.0 {
            fs.writeDiskUsageFile(du, false)
         }
         // Otherwise insure the value will be written to disk after
         // `diskUsageCheckpointTimeout`
         if fs.dirty && !timerActive {
            timer.Reset(diskUsageCheckpointTimeout)
            timerActive = true
         }
      case <-timer.C:
         timerActive = false
         if fs.dirty {
            du := atomic.LoadInt64(&fs.diskUsage)
            fs.writeDiskUsageFile(du, false)
         }
      }
   }
}
```

#### sync

为了防止同时sync的函数太多，效率低。



我们会有一个空结构体的管道，大小为16。



- 会往这个管道里头赛空结构体
- sync
- 从管道中取出空结构体



保证最多16个线程在对这个磁盘做sync操作，其他的都会睡眠等待。

```
// don't block more than 16 threads on sync opearation
// 16 should be able to sataurate most RAIDs
// in case of two used disks per write (RAID 1, 5) and queue depth of 2,
// 16 concurrent Sync calls should be able to saturate 16 HDDs RAID
//TODO: benchmark it out, maybe provide tweak parmeter
const SyncThreadsMax = 16

var syncSemaphore chan struct{} = make(chan struct{}, SyncThreadsMax)

func syncDir(dir string) error {
   if runtime.GOOS == "windows" {
      // dir sync on windows doesn't work: https://git.io/vPnCI
      return nil
   }

   dirF, err := os.Open(dir)
   if err != nil {
      return err
   }
   defer dirF.Close()

   syncSemaphore <- struct{}{}
   defer func() { <-syncSemaphore }()

   if err := dirF.Sync(); err != nil {
      return err
   }
   return nil
}

func syncFile(file *os.File) error {
   syncSemaphore <- struct{}{}
   defer func() { <-syncSemaphore }()
   return file.Sync()
}
```

