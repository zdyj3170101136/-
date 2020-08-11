#### blockservice

flatfs处理的是这个字节流。

然后blockservice主要提供block存储和获取，

- 同步获取block, get block

- 异步通过chan返回block, get blocks

- ```
  AllKeysChan// 通过管道获取所有存储的key
  ```

```
// GetBlock gets the requested block.
GetBlock(ctx context.Context, c cid.Cid) (blocks.Block, error)
```

```
// GetBlocks does a batch request for the given cids, returning blocks as
// they are found, in no particular order.
//
// It may not be able to find all requested blocks (or the context may
// be canceled). In that case, it will close the channel early. It is up
// to the consumer to detect this situation and keep track which blocks
// it has received and which it hasn't.
GetBlocks(ctx context.Context, ks []cid.Cid) <-chan blocks.Block
```



- ```go
  // If checkFirst is true then first check that a block doesn't
  // already exist to avoid republishing the block on the exchange.
  checkFirst bool
  ```

默认为true

#### addblock

- 检查这个block的cid是否有效
- 检查我们是否已经存储了
- 如果没有则存储
- 存储成功则告诉exchange自己有这个block（这个可以让其他的ipfs能够通过kad从我们这个ipfs中获取）

```
// AddBlock adds a particular block to the service, Putting it into the datastore.
// TODO pass a context into this if the remote.HasBlock is going to remain here.
func (s *blockService) AddBlock(o blocks.Block) error {
   c := o.Cid()
   // hash security
   err := verifcid.ValidateCid(c)
   if err != nil {
      return err
   }
   if s.checkFirst {
      if has, err := s.blockstore.Has(c); has || err != nil {
         return err
      }
   }

   if err := s.blockstore.Put(o); err != nil {
      return err
   }

   log.Event(context.TODO(), "BlockService.BlockAdded", c)

   if s.exchange != nil {
      if err := s.exchange.HasBlock(o); err != nil {
         log.Errorf("HasBlock: %s", err.Error())
      }
   }

   return nil
}
```

#### exchange就是底层一个content-routing以及账单体系

```
// A BasicBlock is a singular block of data in ipfs. It implements the Block
// interface.
type BasicBlock struct {
   cid  cid.Cid
   data []byte
}

// NewBlock creates a Block object from opaque data. It will hash the data.
func NewBlock(data []byte) *BasicBlock {
   // TODO: fix assumptions
   return &BasicBlock{data: data, cid: cid.NewCidV0(u.Hash(data))}
}
```

block其实就是两个字段，一个是cid一个是[]byte的byte流。

主要就是传入字节流，然后调用hash函数计算出对应的cid。



#### splitting

我们在分完片，整个dag树构建好之后。

把根加入dag。



分片是以256kb的大小使用poll读取数据





#### 关于中间节点的大小

- 为什么要中间节点，如果说你就一个中间节点对吧。
- 那中间节点要等叶子结点全部构建好之后才能生成。（？似乎没啥关系）



- 我们的数据在切片好之后就会传递给上层服务

- 元数据大概是47字节。

- 中间节点块大小为8KB，叶子结点数据大小为256kb。（注意这只会在trikle树中用到）

- 最多174个link，三十多m的数据。（只会trikle树中使用）

- ```go
  // rough estimates on expected sizes
  var roughLinkBlockSize = 1 << 13 // 8KB
  ```

- ```go
  var roughLinkSize = 34 + 8 + 5   // sha256 multihash + size + no name + protobuf framing
  ```



- 优化了Trickle-dag以按顺序读取数据，而merkle-dag则改善了随机访问时间。在长视频中使用点滴模式可能是有道理的，但根据我的经验，这并没有太大的区别。
- 从定义的意义上来说，tricker-dag是一个merkle-dag，但是当我们考虑merkle-dag时，我们默认会考虑平衡树DAG。在具有其他结构（列表列表）的细流模式中则不是这种情况。



- 分片生成根块
- 为中间节点添加child的link的时候把child加到dagservice存储里头去。（因为这之前child还是需要的，比如读取其hash啥的）

```go
func Layout(db *h.DagBuilderHelper) (ipld.Node, error) {
   if db.Done() {
      // No data, return just an empty node.
      root, err := db.NewLeafNode(nil, ft.TFile)
      if err != nil {
         return nil, err
      }
      // This works without Filestore support (`ProcessFileStore`).
      // TODO: Why? Is there a test case missing?

      return root, db.Add(root)
   }

   // The first `root` will be a single leaf node with data
   // (corner case), after that subsequent `root` nodes will
   // always be internal nodes (with a depth > 0) that can
   // be handled by the loop.
   root, fileSize, err := db.NewLeafDataNode(ft.TFile)
   if err != nil {
      return nil, err
   }

   // Each time a DAG of a certain `depth` is filled (because it
   // has reached its maximum capacity of `db.Maxlinks()` per node)
   // extend it by making it a sub-DAG of a bigger DAG with `depth+1`.
   for depth := 1; !db.Done(); depth++ {

      // Add the old `root` as a child of the `newRoot`.
      newRoot := db.NewFSNodeOverDag(ft.TFile)
      newRoot.AddChild(root, fileSize, db)

      // Fill the `newRoot` (that has the old `root` already as child)
      // and make it the current `root` for the next iteration (when
      // it will become "old").
      root, fileSize, err = fillNodeRec(db, newRoot, depth)
      if err != nil {
         return nil, err
      }
   }

   return root, db.Add(root)
}
```