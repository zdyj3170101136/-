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