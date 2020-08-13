缓存

- 嗯,在一些文件系统缓存中实现的标准的LRU淘汰算法是有一些缺点的。例如，它们对扫描读模式是没有抵抗性的。但你一次顺序读取大量的数据块时，这些数据块就会填满整个缓存空间，即使它们只是被读一次。当缓存空间满了之后，你如果想向缓存放入新的数据，那些最近最少被使用的页面将会被淘汰出去。在这种大量顺序读的情况下，我们的缓存将会只包含这些新读的数据，而不是那些真正被经常使用的数据。在这些顺序读出的数据仅仅只被使用一次的情况下，从缓存的角度来看，它将被这些无用的数据填满。



- 另外一个挑战是：一个缓存可以根据时间进行优化（缓存那些最近使用的页面），也可以根据频率进行优化（缓存那些最频繁使用的页面）。但是这两种方法都不能适应所有的workload。而一个好的缓存设计是能自动根据workload来调整它的优化策略。



2q queue。

frequent是最近看到的块的lru链表。

recent是刚刚看到的块的lru链表

#### get

- 从frequent链表取值
- 如果在recent链表中，则提升到frequent链表。

```go
// Get looks up a key's value from the cache.
func (c *TwoQueueCache) Get(key interface{}) (value interface{}, ok bool) {
   c.lock.Lock()
   defer c.lock.Unlock()

   // Check if this is a frequent value
   if val, ok := c.frequent.Get(key); ok {
      return val, ok
   }

   // If the value is contained in recent, then we
   // promote it to frequent
   if val, ok := c.recent.Peek(key); ok {
      c.recent.Remove(key)
      c.frequent.Add(key, val)
      return val, ok
   }

   // No hit
   return nil, false
}
```

#### add

- 如果frequent链表中有，则update值。
- 如果在recent链表中，则promote 到frequent链表
- 如果在recentevited链表中，则加入frequent链表
- 加入recent链表

```
// Add adds a value to the cache.
func (c *TwoQueueCache) Add(key, value interface{}) {
   c.lock.Lock()
   defer c.lock.Unlock()

   // Check if the value is frequently used already,
   // and just update the value
   if c.frequent.Contains(key) {
      c.frequent.Add(key, value)
      return
   }

   // Check if the value is recently used, and promote
   // the value into the frequent list
   if c.recent.Contains(key) {
      c.recent.Remove(key)
      c.frequent.Add(key, value)
      return
   }

   // If the value was recently evicted, add it to the
   // frequently used list
   if c.recentEvict.Contains(key) {
      c.ensureSpace(true)
      c.recentEvict.Remove(key)
      c.frequent.Add(key, value)
      return
   }

   // Add to the recently seen list
   c.ensureSpace(false)
   c.recent.Add(key, value)
   return
}
```

#### remove

remove仅仅只是从对应的链表中删除。

```
// Remove removes the provided key from the cache.
func (c *TwoQueueCache) Remove(key interface{}) {
   c.lock.Lock()
   defer c.lock.Unlock()
   if c.frequent.Remove(key) {
      return
   }
   if c.recent.Remove(key) {
      return
   }
   if c.recentEvict.Remove(key) {
      return
   }
}
```

#### ensurespace

- 如果recent的链表和frequent链表加起来大于size
- 如果recent链表超过指定大小，则从recent链表装到recentevicted链表
- 否则移除frequent链表

```
// ensureSpace is used to ensure we have space in the cache
func (c *TwoQueueCache) ensureSpace(recentEvict bool) {
   // If we have space, nothing to do
   recentLen := c.recent.Len()
   freqLen := c.frequent.Len()
   if recentLen+freqLen < c.size {
      return
   }

   // If the recent buffer is larger than
   // the target, evict from there
   if recentLen > 0 && (recentLen > c.recentSize || (recentLen == c.recentSize && !recentEvict)) {
      k, _, _ := c.recent.RemoveOldest()
      c.recentEvict.Add(k, nil)
      return
   }

   // Remove from the frequent list otherwise
   c.frequent.RemoveOldest()
}
```

#### 什么情况下

当recent链表太多，多余的会被送到recentevicted链表。



这个时候get不会从这个recentevicted链表之中取出值。

但是put的时候会尝试把这个recentevicted提升到frequent。



- 

#### 链表大小设置

- recent是0.25倍
- recentevicted是0.5倍。

```go
const (
   // Default2QRecentRatio is the ratio of the 2Q cache dedicated
   // to recently added entries that have only been accessed once.
   Default2QRecentRatio = 0.25

   // Default2QGhostEntries is the default ratio of ghost
   // entries kept to track entries recently evicted
   Default2QGhostEntries = 0.50
)
```

```go
// Determine the sub-sizes
recentSize := int(float64(size) * recentRatio)
evictSize := int(float64(size) * ghostRatio)
```

#### 做到

- 首次加入是加入recent链表
- 多次访问才会加入frequent链表。
- 避免了对一个大文件把所有lru都变成新的。



（对访问频率的计算近似。。）

这样可以避免突然接触到新
//清除经常使用的条目。



O（1）时间复杂度，没有复杂参数。

#### evict

注意recentevicted只会存储key，而没有value。



am：frequent。a1:recent

如果A *1*的最大大小的阈值太小，则很有可能会丢失重新引用的页面。如果阈值太高，则A *m*的大小将减小，并且只能存储较少的重新引用页面。这将影响整体性能。该方法的算法如下



https://medium.com/@koushikmohan/an-analysis-of-2q-cache-replacement-algorithms-21acceae672a