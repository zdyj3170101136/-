#### datastore是一个接口。

flatfs是最底层的接口。

```
// CachedBlockstore returns a blockstore wrapped in an ARCCache and
// then in a bloom filter cache, if the options indicate it.
func CachedBlockstore(
   ctx context.Context,
   bs Blockstore,
   opts CacheOpts) (cbs Blockstore, err error) {
   cbs = bs

   if opts.HasBloomFilterSize < 0 || opts.HasBloomFilterHashes < 0 ||
      opts.HasARCCacheSize < 0 {
      return nil, errors.New("all options for cache need to be greater than zero")
   }

   if opts.HasBloomFilterSize != 0 && opts.HasBloomFilterHashes == 0 {
      return nil, errors.New("bloom filter hash count can't be 0 when there is size set")
   }

   ctx = metrics.CtxSubScope(ctx, "bs.cache")

   if opts.HasARCCacheSize > 0 {
      cbs, err = newARCCachedBS(ctx, cbs, opts.HasARCCacheSize)
   }
   if opts.HasBloomFilterSize != 0 {
      // *8 because of bytes to bits conversion
      cbs, err = bloomCached(ctx, cbs, opts.HasBloomFilterSize*8, opts.HasBloomFilterHashes)
   }

   return cbs, err
}
```

#### retrydatastore

这个接口构建在之前的flatfs以及leveldb之上。

主要提供了一个retry以及sleep语意。



就是如果操作失败，会进行重试。

- 最多重试6次
- 每次失败，睡眠200ms；第一次失败睡眠200ms，第二次就是400，一个线性的函数。



我们会把对flatfs层执行的操作包装成一个函数。

- 函数称为operation的err
- retry层的核心是一个runOp()的函数。
- 它执行operation这个函数。
- 然后如果是临时错误的话就会重试。
- **EMFILE Too many open files 24**
- 如果是因为打开的文件太多而导致的，那么将会进行重试



```go
func isTooManyFDError(err error) bool {
   perr, ok := err.(*os.PathError)
   if ok && perr.Err == syscall.EMFILE {
      return true
   }

   return false
}
```

```
rds := &retrystore.Datastore{
   Batching:    repo.Datastore(),
   Delay:       time.Millisecond * 200,
   Retries:     6,
   TempErrFunc: isTooManyFDError,
}
```

```
func (d *Datastore) runOp(op func() error) error {
   err := op()
   if err == nil || !d.TempErrFunc(err) {
      return err
   }

   for i := 0; i < d.Retries; i++ {
      time.Sleep(time.Duration(i+1) * d.Delay)

      err = op()
      if err == nil || !d.TempErrFunc(err) {
         return err
      }
   }

   return xerrors.Errorf(errFmtString, err)
}
```

#### arccached

然后在retrydatastore上我们又会包一层。

叫做cache。

我们使用一个arccached做缓存。



缓存的话是key - 是否存在，或者大小。（方便GETSIZE函数）



当我们get的时候。

- 先从这个cache里头查找这个key，如果发现说找到了这个key且显示key不存在，

  那我们就可以直接返回。

- 如果没有找到这个key或者显示有大小，那我们就从blockstore中查找

- 如果blockstore中没有找到，那我们就cache这个key，然后一个bool值false，表示没有找到。

- 如果找到了，我们就cache这个key的大小。

```##
func (b *arccache) Get(k cid.Cid) (blocks.Block, error) {
   if !k.Defined() {
      log.Error("undefined cid in arc cache")
      return nil, ErrNotFound
   }

   if has, _, ok := b.hasCached(k); ok && !has {
      return nil, ErrNotFound
   }

   bl, err := b.blockstore.Get(k)
   if bl == nil && err == ErrNotFound {
      b.cacheHave(k, false)
   } else if bl != nil {
      b.cacheSize(k, len(bl.RawData()))
   }
   return bl, err
}
```

#### put的时候也是同理

如果已经在cache里头且存在，那么就不用重复cache。

#### hash

如果找到且不在，那么就不用去has。

cache结果。

```
func (b *arccache) Has(k cid.Cid) (bool, error) {
   if has, _, ok := b.hasCached(k); ok {
      return has, nil
   }
   has, err := b.blockstore.Has(k)
   if err != nil {
      return false, err
   }
   b.cacheHave(k, has)
   return has, nil
}
```

#### bloom

这个arccache，一般是64 << 10。

最多存储65536个key。

bloom过滤器大小是512 << 10,

524288.

存储的项的数目明显大很多。

```
 errors.New("usage: New(float64(number_of_entries), float64(number_of_hashlocations)) 
```

```
 CacheOpts{
   HasBloomFilterSize:   512 << 10,
   HasBloomFilterHashes: 7,
   HasARCCacheSize:      64 << 10,
}
```

对于has请求，先从这个bloom过滤器里头判断有没有可能has。

如果完全没可能，都不需要去到这个cache里头去寻找。

```

// if ok == false has is inconclusive
// if ok == true then has respons to question: is it contained
func (b *bloomcache) hasCached(k cid.Cid) (has bool, ok bool) {
	b.total.Inc()
	if !k.Defined() {
		log.Error("undefined in bloom cache")
		// Return cache invalid so call to blockstore
		// in case of invalid key is forwarded deeper
		return false, false
	}
	if b.BloomActive() {
		blr := b.bloom.HasTS(k.Hash())
		if !blr { // not contained in bloom is only conclusive answer bloom gives
			b.hits.Inc()
			return false, true
		}
	}
	return false, false
}

func (b *bloomcache) Has(k cid.Cid) (bool, error) {
	if has, ok := b.hasCached(k); ok {
		return has, nil
	}

	return b.blockstore.Has(k)
}

```

get的时候同理，过滤掉不可能的请求。



而对于put的时候，就向bloom过滤器里头加上这个值。

因为有可能不存在，因此没必要做判断。

```
func (b *bloomcache) Get(k cid.Cid) (blocks.Block, error) {
   if has, ok := b.hasCached(k); ok && !has {
      return nil, ErrNotFound
   }

   return b.blockstore.Get(k)
}

func (b *bloomcache) Put(bl blocks.Block) error {
   // See comment in PutMany
   err := b.blockstore.Put(bl)
   if err == nil {
      b.bloom.AddTS(bl.Cid().Hash())
   }
   return err
}
```

#### BATCH

- 事实上我们对于一个文件的添加使用的dagService称做是bufferDag
- 里头有一个batch叫做批处理
- 操作都是在这个请求里头缓冲。
- 然后最后才commit。
- 也就是说缓冲写操作，读操作立即执行。
- remove操作也是立即执行。
- 不过写操作也不是无限缓冲，当大小大于8 << 20字节，或者是超过128个node之后就会自动的asyncCommit。



- 最大的commit的线程数量为逻辑cpu的个数 * 2，intel corei7上是8个。
- asyncCommit会开启一个协程，然后把当前的这个上下文，还有要写的节点复制进去。
- 以及一个chan，用来返回result的。
- 注意batch这个结构体是对单个文件的（也就是一个目录可能下面很多个文件）



- 虽然是axynccommit
- 但是Commit这个函数本身是同步的。

```go
// Constructs a node from reader's data, and adds it. Doesn't pin.
func (adder *Adder) add(reader io.Reader) (ipld.Node, error) {
   chnk, err := chunker.FromString(reader, adder.Chunker)
   if err != nil {
      return nil, err
   }

   params := ihelper.DagBuilderParams{
      Dagserv:    adder.bufferedDS,
      RawLeaves:  adder.RawLeaves,
      Maxlinks:   ihelper.DefaultLinksPerBlock,
      NoCopy:     adder.NoCopy,
      CidBuilder: adder.CidBuilder,
   }

   db, err := params.New(chnk)
   if err != nil {
      return nil, err
   }
   var nd ipld.Node
   if adder.Trickle {
      nd, err = trickle.Layout(db)
   } else {
      nd, err = balanced.Layout(db)
   }
   if err != nil {
      return nil, err
   }

   return nd, adder.bufferedDS.Commit()
}
```



```go
// Commit commits batched nodes.
func (t *Batch) Commit() error {
   if t.err != nil {
      return t.err
   }

   t.asyncCommit()

loop:
   for t.activeCommits > 0 {
      select {
      case err := <-t.commitResults:
         t.activeCommits--
         if err != nil {
            t.setError(err)
            break loop
         }
      case <-t.ctx.Done():
         t.setError(t.ctx.Err())
         break loop
      }
   }

   return t.err
}
```

```go
// ParallelBatchCommits is the number of batch commits that can be in-flight before blocking.
// TODO(ipfs/go-ipfs#4299): Experiment with multiple datastores, storage
// devices, and CPUs to find the right value/formula.
var ParallelBatchCommits = runtime.NumCPU() * 2
```

```go
var defaultBatchOptions = batchOptions{
   maxSize: 8 << 20, // 也就是8M

   // By default, only batch up to 128 nodes at a time.
   // The current implementation of flatfs opens this many file
   // descriptors at the same time for the optimized batch write.
   maxNodes: 128,
}
```

```go
// Get commits and gets a node from the DAGService.
func (bd *BufferedDAG) Get(ctx context.Context, c cid.Cid) (Node, error) {
   err := bd.b.Commit()
   if err != nil {
      return nil, err
   }
   return bd.ds.Get(ctx, c)
}
```

```go
// BufferedDAG implements DAGService using a Batch NodeAdder to wrap add
// operations in the given DAGService. It will trigger Commit() before any
// non-Add operations, but otherwise calling Commit() is left to the user.
type BufferedDAG struct {
   ds DAGService
   b  *Batch
}

// NewBufferedDAG creates a BufferedDAG using the given DAGService and the
// given options for the Batch NodeAdder.
func NewBufferedDAG(ctx context.Context, ds DAGService, opts ...BatchOption) *BufferedDAG {
   return &BufferedDAG{
      ds: ds,
      b:  NewBatch(ctx, ds, opts...),
   }
}
```



```go
// AddMany many calls Add for every given Node, thus batching and
// commiting them as needed.
func (t *Batch) AddMany(ctx context.Context, nodes []Node) error {
   if t.err != nil {
      return t.err
   }
   // Not strictly necessary but allows us to catch errors early.
   t.processResults()

   if t.err != nil {
      return t.err
   }

   t.nodes = append(t.nodes, nodes...)
   for _, nd := range nodes {
      t.size += len(nd.RawData())
   }

   if t.size > t.opts.maxSize || len(t.nodes) > t.opts.maxNodes {
      t.asyncCommit()
   }
   return t.err
}
```



- aysncCommitt后分配的切片的大小初始化不为0.。。
- 注意如果直接添加16m的数据，是不会开启两个线程去commit
- 而是会开启一个线程对16m的数据做commit（不过我们是一个个添加块的所以不会出现这种操作）

```
func (t *Batch) asyncCommit() {
   numBlocks := len(t.nodes)
   if numBlocks == 0 {
      return
   }
   if t.activeCommits >= ParallelBatchCommits {
      select {
      case err := <-t.commitResults:
         t.activeCommits--

         if err != nil {
            t.setError(err)
            return
         }
      case <-t.ctx.Done():
         t.setError(t.ctx.Err())
         return
      }
   }
   go func(ctx context.Context, b []Node, result chan error, na NodeAdder) {
      select {
      case result <- na.AddMany(ctx, b):
      case <-ctx.Done():
      }
   }(t.ctx, t.nodes, t.commitResults, t.na)

   t.activeCommits++
   t.nodes = make([]Node, 0, numBlocks)
   t.size = 0

   return
}
```



#### 如何显示进度条

```go
func (adder *Adder) addFile(path string, file files.File) error {
   // if the progress flag was specified, wrap the file so that we can send
   // progress updates to the client (over the output channel)
   var reader io.Reader = file
   if adder.Progress {
      rdr := &progressReader{file: reader, path: path, out: adder.Out}
      if fi, ok := file.(files.FileInfo); ok {
         reader = &progressReader2{rdr, fi}
      } else {
         reader = rdr
      }
   }
```



当我们从文件中读取文件的时候。

我们先把它包装成一个progressReader这么一个结构体。

后面chunker分块的时候读取也是从这里读取。

- 作用就是每读取256kb（1024 * 256 字节）的时候就会往一个管道里面塞一个结构体
- 包括当前已经从这个文件的路径名以及已经读取了多少byte。

```go
type progressReader struct {
   file         io.Reader
   path         string
   out          chan<- interface{}
   bytes        int64
   lastProgress int64
}

func (i *progressReader) Read(p []byte) (int, error) {
   n, err := i.file.Read(p)

   i.bytes += int64(n)
   if i.bytes-i.lastProgress >= progressReaderIncrement || err == io.EOF {
      i.lastProgress = i.bytes
      i.out <- &coreiface.AddEvent{
         Name:  i.path,
         Bytes: i.bytes,
      }
   }

   return n, err
}
```



另外对命令行添加文件我们是开启一个协程来处理

- 从events这个窗口里头循环得到结果
- 包括这个文件的名字
- hash
- byte流
- 然后把它emit发射出去



```go
for event := range events {
      output, ok := event.(*coreiface.AddEvent)
      if !ok {
         return errors.New("unknown event type")
      }

      h := ""
      if output.Path != nil {
         h = enc.Encode(output.Path.Cid())
      }

      if !dir && name != "" {
         output.Name = name
      } else {
         output.Name = path.Join(name, output.Name)
      }

      if err := res.Emit(&AddEvent{
         Name:  output.Name,
         Hash:  h,
         Bytes: output.Bytes,
         Size:  output.Size,
      }); err != nil {
         return err
      }
   }

   return <-errCh
},
```