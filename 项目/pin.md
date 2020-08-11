```go
// Pin the given node, optionally recursive
func (p *pinner) Pin(ctx context.Context, node ipld.Node, recurse bool) error {
   err := p.dserv.Add(ctx, node)
   if err != nil {
      return err
   }

   c := node.Cid()

   p.lock.Lock()
   defer p.lock.Unlock()

   if recurse {
      if p.recursePin.Has(c) {
         return nil
      }

      p.lock.Unlock()
      // temporary unlock to fetch the entire graph
      err := mdag.FetchGraph(ctx, c, p.dserv)
      p.lock.Lock()
      if err != nil {
         return err
      }

      if p.recursePin.Has(c) {
         return nil
      }

      if p.directPin.Has(c) {
         p.directPin.Remove(c)
      }

      p.recursePin.Add(c)
   } else {
      if p.recursePin.Has(c) {
         return fmt.Errorf("%s already pinned recursively", c.String())
      }

      p.directPin.Add(c)
   }
   return nil
}
```

如果是直接pin，那么就在directpin的集合中加上。

如果不是直接pin，就通过网络，先把这个节点的所有孩子都取到。

然后把节点添加到resursivepin这个set里头。

注意：：即使是recursivepin现在还没有涉及到node的child。

#### pin状态

首先对于node，我们会有pin这个操作。

- Direct 表示这个node本身被pin了
- inderect表示这个node是被递归的pin了，因为它有祖先是已经被recursive递归的pin的文件
- notpinned表示没有被pin的文件
- recursive表示node本身被pin加上所有的子node
- internal表示内部的node

我们从pin的状态中

```go
// Pin Modes
const (
   // Recursive pins pin the target cids along with any reachable children.
   Recursive Mode = iota

   // Direct pins pin just the target cid.
   Direct

   // Indirect pins are cids who have some ancestor pinned recursively.
   Indirect

   // Internal pins are cids used to keep the internal state of the pinner.
   Internal

   // NotPinned
   NotPinned

   // Any refers to any pinned cid
   Any
)
```

迭代的寻找child



```go
// hasChild recursively looks for a Cid among the children of a root Cid.
// The visit function can be used to shortcut already-visited branches.
func hasChild(ctx context.Context, ng ipld.NodeGetter, root cid.Cid, child cid.Cid, visit func(cid.Cid) bool) (bool, error) {
   links, err := ipld.GetLinks(ctx, ng, root)
   if err != nil {
      return false, err
   }
   for _, lnk := range links {
      c := lnk.Cid
      if lnk.Cid.Equals(child) {
         return true, nil
      }
      if visit(c) {
         has, err := hasChild(ctx, ng, c, child, visit)
         if err != nil {
            return false, err
         }

         if has {
            return has, nil
         }
      }
   }
   return false, nil
}
```



#### gc

我们gc的时候就通过这个pin状态。

运行一个标记，清楚算法。



- 其实就是通过pinner得到一个gcset集合。
  - 从pinner得到所有recursivepin，寻找它们的child加入gcset
  - 从bestEffortRoots，加上所有的child
  - 加上所有directpin
  - 加上所有internalpin，以及它们的child
- 然后通过blockservice返回一个馋存储的所有block的key的chan，
- 遍历这个chan，如果不在gcset之中就调用删除的接口。

```
// GC performs a mark and sweep garbage collection of the blocks in the blockstore
// first, it creates a 'marked' set and adds to it the following:
// - all recursively pinned blocks, plus all of their descendants (recursively)
// - bestEffortRoots, plus all of its descendants (recursively)
// - all directly pinned blocks
// - all blocks utilized internally by the pinner
//
// The routine then iterates over every block in the blockstore and
// deletes any block that is not found in the marked set.
func GC(ctx context.Context, bs bstore.GCBlockstore, dstor dstore.Datastore, pn pin.Pinner, bestEffortRoots []cid.Cid) <-chan Result {
   ctx, cancel := context.WithCancel(ctx)

   unlocker := bs.GCLock()

   bsrv := bserv.New(bs, offline.Exchange(bs))
   ds := dag.NewDAGService(bsrv)

   output := make(chan Result, 128)

   go func() {
      defer cancel()
      defer close(output)
      defer unlocker.Unlock()

      gcs, err := ColoredSet(ctx, pn, ds, bestEffortRoots, output)
      if err != nil {
         select {
         case output <- Result{Error: err}:
         case <-ctx.Done():
         }
         return
      }
      keychan, err := bs.AllKeysChan(ctx)
      if err != nil {
         select {
         case output <- Result{Error: err}:
         case <-ctx.Done():
         }
         return
      }

      errors := false
      var removed uint64

   loop:
      for ctx.Err() == nil { // select may not notice that we're "done".
         select {
         case k, ok := <-keychan:
            if !ok {
               break loop
            }
            if !gcs.Has(k) {
               err := bs.DeleteBlock(k)
               removed++
               if err != nil {
                  errors = true
                  select {
                  case output <- Result{Error: &CannotDeleteBlockError{k, err}}:
                  case <-ctx.Done():
                     break loop
                  }
                  // continue as error is non-fatal
                  continue loop
               }
               select {
               case output <- Result{KeyRemoved: k}:
               case <-ctx.Done():
                  break loop
               }
            }
         case <-ctx.Done():
            break loop
         }
      }
      if errors {
         select {
         case output <- Result{Error: ErrCannotDeleteSomeBlocks}:
         case <-ctx.Done():
            return
         }
      }

      gds, ok := dstor.(dstore.GCDatastore)
      if !ok {
         return
      }

      err = gds.CollectGarbage()
      if err != nil {
         select {
         case output <- Result{Error: err}:
         case <-ctx.Done():
         }
         return
      }
   }()

   return output
}
```

#### gc和pin的冲突。

gc的时候和pin的状态其实是冲突的。

gc的时候不能pin对吧。

pin可以同时进行。

这两个都对读无影响。



- gc的时候获取这个写锁

- pin的时候获取读锁。

  

- 我们还需要一个int32位的变量。



- gc的时候增加这个变量
- 获取写锁
- 减小这个变量（atomic）
- 为什么要减小呢？（这是因为如果我们已经获取了写锁，那么其实之后就靠读写锁限制并发就好）



#### 为什么需要这个atomic变量？

比方说可能我们添加一个大文件。获取了这个读锁。

然后之后我们开启了gc。

这个文件可能是一个目录由很多个文件所组成。

然后gc程序就不得不等待我这个大文件全部搞完后才能开始gc。



所以我们获取了读锁之后，每次添加单个文件的时候，还要判断一下这个变量是否大于0。

如果大于0，那么说明有gc尝试获取写锁。

那我们就要释放掉我们获取的读锁。

然后再去获取这个读锁。



- 我们addFile的时候，先获取这个pinlock
- 然后判断gcrequested》0,即有gc程序正在等待获取写锁
- pin当前的root，防止被gc回收。
- 释放之前获取的读锁
- 再次去获取读锁



#### 释放再获取为什么要这么干？

这是因为写锁需要等待所有的读锁释放掉才能获取写锁。

而我们重新获取读锁其实是睡眠等待这个写锁释放才能获取。

意义不一样。





```go
// AddAllAndPin adds the given request's files and pin them.
func (adder *Adder) AddAllAndPin(file files.Node) (ipld.Node, error) {
   if adder.Pin {
      adder.unlocker = adder.gcLocker.PinLock()
   }
   defer func() {
      if adder.unlocker != nil {
         adder.unlocker.Unlock()
      }
   }()

   if err := adder.addFileNode("", file, true); err != nil {
      return nil, err
   }
```

```go
func (adder *Adder) maybePauseForGC() error {
   if adder.unlocker != nil && adder.gcLocker.GCRequested() {
      rn, err := adder.curRootNode()
      if err != nil {
         return err
      }

      err = adder.PinRoot(rn)
      if err != nil {
         return err
      }

      adder.unlocker.Unlock()
      adder.unlocker = adder.gcLocker.PinLock()
   }
   return nil
}
```

```go
// GCLocker abstract functionality to lock a blockstore when performing
// garbage-collection operations.
type GCLocker interface {
   // GCLock locks the blockstore for garbage collection. No operations
   // that expect to finish with a pin should ocurr simultaneously.
   // Reading during GC is safe, and requires no lock.
   GCLock() Unlocker

   // PinLock locks the blockstore for sequences of puts expected to finish
   // with a pin (before GC). Multiple put->pin sequences can write through
   // at the same time, but no GC should happen simulatenously.
   // Reading during Pinning is safe, and requires no lock.
   PinLock() Unlocker

   // GcRequested returns true if GCLock has been called and is waiting to
   // take the lock
   GCRequested() bool
}
```

```go
type gclocker struct {
   lk    sync.RWMutex
   gcreq int32
}

// Unlocker represents an object which can Unlock
// something.
type Unlocker interface {
   Unlock()
}

type unlocker struct {
   unlock func()
}

func (u *unlocker) Unlock() {
   u.unlock()
   u.unlock = nil // ensure its not called twice
}

func (bs *gclocker) GCLock() Unlocker {
   atomic.AddInt32(&bs.gcreq, 1)
   bs.lk.Lock()
   atomic.AddInt32(&bs.gcreq, -1)
   return &unlocker{bs.lk.Unlock}
}

func (bs *gclocker) PinLock() Unlocker {
   bs.lk.RLock()
   return &unlocker{bs.lk.RUnlock}
}

func (bs *gclocker) GCRequested() bool {
   return atomic.LoadInt32(&bs.gcreq) > 0
}
```

#### 内存set

recursivePin，directpin，internalPIn这三个set。

通过读写锁限制访问。



由于pinner的本身是保存在内存里头的。

所以我们要把它保存在磁盘里头，怎么保存呢？



我们生成一个merkledag node作为internal node，然后两个链接。

- link1， 指向recursive pin的
- Link2， 指向direct pin的
- 然后把这个node本身使用dagservice存到里头去。

```go
// Flush encodes and writes pinner keysets to the datastore
func (p *pinner) Flush(ctx context.Context) error {
   p.lock.Lock()
   defer p.lock.Unlock()

   internalset := cid.NewSet()
   recordInternal := internalset.Add

   root := &mdag.ProtoNode{}
   {
      n, err := storeSet(ctx, p.internal, p.directPin.Keys(), recordInternal)
      if err != nil {
         return err
      }
      if err := root.AddNodeLink(linkDirect, n); err != nil {
         return err
      }
   }

   {
      n, err := storeSet(ctx, p.internal, p.recursePin.Keys(), recordInternal)
      if err != nil {
         return err
      }
      if err := root.AddNodeLink(linkRecursive, n); err != nil {
         return err
      }
   }

   // add the empty node, its referenced by the pin sets but never created
   err := p.internal.Add(ctx, new(mdag.ProtoNode))
   if err != nil {
      return err
   }

   err = p.internal.Add(ctx, root)
   if err != nil {
      return err
   }

   k := root.Cid()

   internalset.Add(k)

   if syncDServ, ok := p.dserv.(syncDAGService); ok {
      if err := syncDServ.Sync(); err != nil {
         return fmt.Errorf("cannot sync pinned data: %v", err)
      }
   }

   if syncInternal, ok := p.internal.(syncDAGService); ok {
      if err := syncInternal.Sync(); err != nil {
         return fmt.Errorf("cannot sync pinning data: %v", err)
      }
   }

   if err := p.dstore.Put(pinDatastoreKey, k.Bytes()); err != nil {
      return fmt.Errorf("cannot store pin state: %v", err)
   }
   if err := p.dstore.Sync(pinDatastoreKey); err != nil {
      return fmt.Errorf("cannot sync pin state: %v", err)
   }
   p.internalPin = internalset
   return nil
}
```

而所有的这种internal node，都会挂在/local/pins这个node之下。

```
var pinDatastoreKey = ds.NewKey("/local/pins")
```

这样在初始化的时候，就可以通过这个node。

获得所有internal node，重建内存中的pin set。



#### add

#### addFileNode

会maybepauseforgc



- 调用addFile， 文件切块，layout然后bufferdag commit
- 调用addDir

#### 最后

- pin and flush pin

#### checkifPINED

- 检查是否在这个recursivePIN这个集合中
- 检查是否在directPin这个集合中
- 检查是否在internalPIN这个集合中
- 对于indirectPIN则只能遍历recursivePin的的每一个key，取出所有的graph图来。



而removepin：

- 从recursivepin中删除
- 从directpin中删除



所以如果一个块同时存在于两个文件中。

unpin第二个文件。

第一个文件的根hash还在这个recursivepin集合中。

还是能够遍历得到的。



还能够pin住一个文件的子快使其不被回收。

但是不能够仅仅删除一个文件的子块。

```
// Unpin a given key
func (p *pinner) Unpin(ctx context.Context, c cid.Cid, recursive bool) error {
   p.lock.Lock()
   defer p.lock.Unlock()
   if p.recursePin.Has(c) {
      if !recursive {
         return fmt.Errorf("%s is pinned recursively", c)
      }
      p.recursePin.Remove(c)
      return nil
   }
   if p.directPin.Has(c) {
      p.directPin.Remove(c)
      return nil
   }
   return ErrNotPinned
}
```

```
// isPinnedWithType is the implementation of IsPinnedWithType that does not lock.
// intended for use by other pinned methods that already take locks
func (p *pinner) isPinnedWithType(ctx context.Context, c cid.Cid, mode Mode) (string, bool, error) {
   switch mode {
   case Any, Direct, Indirect, Recursive, Internal:
   default:
      err := fmt.Errorf("invalid Pin Mode '%d', must be one of {%d, %d, %d, %d, %d}",
         mode, Direct, Indirect, Recursive, Internal, Any)
      return "", false, err
   }
   if (mode == Recursive || mode == Any) && p.recursePin.Has(c) {
      return linkRecursive, true, nil
   }
   if mode == Recursive {
      return "", false, nil
   }

   if (mode == Direct || mode == Any) && p.directPin.Has(c) {
      return linkDirect, true, nil
   }
   if mode == Direct {
      return "", false, nil
   }

   if (mode == Internal || mode == Any) && p.isInternalPin(c) {
      return linkInternal, true, nil
   }
   if mode == Internal {
      return "", false, nil
   }

   // Default is Indirect
   visitedSet := cid.NewSet()
   for _, rc := range p.recursePin.Keys() {
      has, err := hasChild(ctx, p.dserv, rc, c, visitedSet.Visit)
      if err != nil {
         return "", false, err
      }
      if has {
         return rc.String(), true, nil
      }
   }
   return "", false, nil
}
```

#### gc本身其实是异步的。

它会用一个chan返回result。



命令行外部调用的直接返回这个chan就行。

如果是内部的话，就需要等待这个result全部完毕。

```go
// DefaultDatastoreConfig is an internal function exported to aid in testing.
func DefaultDatastoreConfig() Datastore {
   return Datastore{
      StorageMax:         "10GB",
      StorageGCWatermark: 90, // 90%
      GCPeriod:           "1h",
      BloomFilterSize:    0,
```



- 定时gc：每1h一次，调用maybegc
- conditionalgc：调用maybegc



- maybegc，是一个storage通常最大容量是10gb，然后有一个高水位：如果超过90%那么就会开启gc。



- 当我们添加单个文件的时候，我们会maybePauseForGC
- 在文件的粒度上面检查
- 注意addDir也包括文件