#### 我们使用leveldb存储当前到底有哪些cid这些信息，以及提供对一个cid的查询返回结果。

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

