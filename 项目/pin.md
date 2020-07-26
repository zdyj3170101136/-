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

然后对于internal的节点，其实就是迭代的寻找link，不断的找啊找。



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

- 然后通过blockservice返回一个馋存储的所有block的key
- 遍历这个chan，如果在gcset之中就调用删除的接口。

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

#### 持久化
由于pinner的本身是保存在内存里头的。

所以我们要把它保存在磁盘里头，怎么保存呢？



我们生成一个merkledag node。然后两个链接。

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

而所有的这种internal node，都会挂在/local/pins这个node之下

```
var pinDatastoreKey = ds.NewKey("/local/pins")
```