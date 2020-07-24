```
// Copyright (c) 2012, Suryandaru Triandana <syndtr@gmail.com>
// All rights reserved.
//
// Use of this source code is governed by a BSD-style license that can be
// found in the LICENSE file.

// Package cache provides interface and implementation of a cache algorithms.
package cache

import (
   "sync"
   "sync/atomic"
   "unsafe"

   "github.com/syndtr/goleveldb/leveldb/util"
)

// Value is a 'cacheable object'. It may implements util.Releaser, if
// so the the Release method will be called once object is released.
type Value interface{}

// The hash tables implementation is based on:
// "Dynamic-Sized Nonblocking Hash Tables", by Yujie Liu,
// Kunlong Zhang, and Michael Spear.
// ACM Symposium on Principles of Distributed Computing, Jul 2014.

const (
   mInitialSize           = 1 << 4
   mOverflowThreshold     = 1 << 5 // 32个
)

type mBucket struct {
   // 对每个bucket我们都通过mutex限制对单一bucket的并发访问
   mu     sync.Mutex
   node   []*Node // 注意变量名叫node
   frozen bool // 使用mutex保护的变量，不用atomic
}

func (b *mBucket) freeze() []*Node {
   b.mu.Lock() // 注意这里也加锁，frozen就不用
   defer b.mu.Unlock()
   if !b.frozen {
      b.frozen = true
   }
   return b.node
}

// 注意如果是frozen的bucket，那么不会对其访问，所以return false， done表示返回nil到底是frozen不能找，还是没有找到
// added表示是否有往里头添加node，当且仅当返回noset
func (b *mBucket) get(r *Cache, h *mNode, hash uint32, key uint64, noset bool) (done, added bool, n *Node) {
   b.mu.Lock()

   if b.frozen {
      // 防止在getbucket和get中间已经被freeze
      b.mu.Unlock()
      return
   }

   // Scan the node.
   for _, n := range b.node {
      if n.hash == hash  && n.key == key {
         b.mu.Unlock()
         return true, false, n
      }
   }

   // Get only.
   if noset {
      b.mu.Unlock()
      return true, false, nil
   }

   // Create node.
   n = &Node{
      r:    r,
      hash: hash,
      key:  key,
   }
   // Add node to bucket.
   b.node = append(b.node, n)
   b.mu.Unlock()  // 提早释放锁

   // Update counter.
   grow := atomic.AddInt32(&r.nodes, 1) >= h.growThreshold

   // Grow.
   // 这样，旧的mnode是1，就无法对它进行resize。
   if grow && atomic.CompareAndSwapInt32(&h.resizeInProgess, 0, 1) {
      // 避免已经有resize正在进行中

      // 生成一个新的mnode，并且用cas操作改变cache的指向
      // 旧的cas
      nhLen := len(h.buckets) << 1
      nh := &mNode{
         buckets:         make([]unsafe.Pointer, nhLen),
         mask:            uint32(nhLen) - 1,
         pred:            unsafe.Pointer(h),
         growThreshold:   int32(nhLen * mOverflowThreshold),
      }

      ok := atomic.CompareAndSwapPointer(&r.mHead, unsafe.Pointer(h), unsafe.Pointer(nh))
      if !ok {
         panic("BUG: failed swapping head")
      }
      // 开启协程去init
      go nh.initBuckets()
   }

   return true, true, n
}

type mNode struct {
   buckets         []unsafe.Pointer // []*mBucket,注意是指针
   mask            uint32
   pred            unsafe.Pointer // *mNode
   resizeInProgess int32 // cas变量全设置为int32，除了跟hash有关的部分

   growThreshold   int32
}

// 注意要返回mbucket
func (n *mNode) initBucket(i uint32) *mBucket {
   // 注意参数是i, 和对b == nil的判断
   if b := (*mBucket)(atomic.LoadPointer(&n.buckets[i])); b != nil {
      return b
   }

   p := (*mNode)(atomic.LoadPointer(&n.pred))
   // 读取之前的node的mask
   if p != nil {
      var node []*Node
      if n.mask > p.mask {
         // Grow.
         pb := (*mBucket)(atomic.LoadPointer(&p.buckets[i&p.mask])) // pb表示旧的bucket
         if pb == nil {
            // 这里啥意思？？？
            // 这里其实指的是如果一个新的cache还没有全部grow又有一个grow
            pb = p.initBucket(i & p.mask)
         }
         m := pb.freeze() // freeze冻结，这样旧的cache旧无法对旧的bucket进行访问
         // Split nodes.
         //把旧的bucket中应该分到新的bucket的node存储
         for _, x := range m {
            if x.hash&n.mask == i {
               node = append(node, x)
            }
         }
      }
      b := &mBucket{node: node}
      // 同样是cas操作，防止中间已经有了变化
      if atomic.CompareAndSwapPointer(&n.buckets[i], nil, unsafe.Pointer(b)) {
         return b
      }
   }
    // 注意这里需要重复返回
   return (*mBucket)(atomic.LoadPointer(&n.buckets[i]))
}

func (n *mNode) initBuckets() {
   for i := range n.buckets {
      n.initBucket(uint32(i))
   }
   atomic.StorePointer(&n.pred, nil) // 释放旧的，让其可以被垃圾回收， 为什么这里不用cas呢？因为是只有一个会改变。
}

// Cache is a 'cache map'.
type Cache struct {
   mu     sync.RWMutex  // 这个mutex对于cache读和删除都是加读锁，主要是为了避免close
   mHead  unsafe.Pointer // *mNode
   nodes  int32 // 注意node的总数存储在这里
}

// NewCache creates a new 'cache map'. The cacher is optional and
// may be nil.
func NewCache() *Cache {
   h := &mNode{
      buckets:         make([]unsafe.Pointer, mInitialSize),
      mask:            mInitialSize - 1, // mask是32位的hash函数用来对hash函数取到对应的bucket的
      growThreshold:   int32(mInitialSize * mOverflowThreshold),
   }
   for i := range h.buckets {
      h.buckets[i] = unsafe.Pointer(&mBucket{})
   }
   r := &Cache{
      mHead:  unsafe.Pointer(h),
   }
   return r
}

// 这里使用原子操作是因为get的时候生成新的mhead会动态改变，包括mbucket
func (r *Cache) getBucket(hash uint32) (*mNode, *mBucket) {
   h := (*mNode)(atomic.LoadPointer(&r.mHead))
   i := hash & h.mask
   b := (*mBucket)(atomic.LoadPointer(&h.buckets[i]))
   if b == nil {
      b = h.initBucket(i)  // 如果这个bucket还没有初始化
   }
   return h, b
}

// Get gets 'cache node' with the given namespace and key.
// If cache node is not found and setFunc is not nil, Get will atomically creates
// the 'cache node' by calling setFunc. Otherwise Get will returns nil.
//
// The returned 'cache handle' should be released after use by calling Release
// method.
func (r *Cache) Get(ns, key uint64, setFunc func() (value Value)) *Handle {
   r.mu.RLock() // 加读锁
   defer r.mu.RUnlock()

   hash := murmur32(ns, key, 0xf00)
   for {
      h, b := r.getBucket(hash)
      done, _, n := b.get(r, h, hash, key, setFunc == nil)
      if done {
         if n != nil {
            n.mu.Lock() // 多个get会遇到同一个node，使用mutex对node的值的修改并发
            if n.value == nil {
               if setFunc == nil {
                  n.mu.Unlock()
                  return nil
               }

               n.value = setFunc()
               if n.value == nil {
                  n.mu.Unlock()
                  return nil
               }
            }
            n.mu.Unlock()
            return &Handle{unsafe.Pointer(n)}
         }
         break // 注意如果n为nil就break返回nil
      }
   }
   return nil
}

// Node is a 'cache node'.
type Node struct {
   r *Cache

   hash    uint32
    key uint64

   mu    sync.Mutex // 由于这个mu是小写，我们get的时候内部序列化
   value Value
}

// 因此我们还需要有一个handle，保证对node本身的并发操作正常进行。
// 由于还有一个release操作会把这个n置为nil，所以对node的操作也是并发。
// Handle is a 'cache handle' of a 'cache node'.
type Handle struct {
   n unsafe.Pointer // *Node
}

// Value returns the value of the 'cache node'.
func (h *Handle) Value() Value {
   n := (*Node)(atomic.LoadPointer(&h.n))
   // 为什么atomic加载？因为这个node可能会被删除
   if n != nil {
      // 为了能够多次调用，如果为nil就返回nil
      return n.value
   }
   return nil
}

// Release releases this 'cache handle'.
// It is safe to call release multiple times.
func (h *Handle) Release() {
   nPtr := atomic.LoadPointer(&h.n)
   if nPtr != nil && atomic.CompareAndSwapPointer(&h.n, nPtr, nil) {
      n := (*Node)(nPtr)
      n.unrefLocked()
   }
}
func murmur32(ns, key uint64, seed uint32) uint32 {
   const (
      m = uint32(0x5bd1e995)
      r = 24
   )

   k1 := uint32(ns >> 32)
   k2 := uint32(ns)
   k3 := uint32(key >> 32)
   k4 := uint32(key)

   k1 *= m
   k1 ^= k1 >> r
   k1 *= m

   k2 *= m
   k2 ^= k2 >> r
   k2 *= m

   k3 *= m
   k3 ^= k3 >> r
   k3 *= m

   k4 *= m
   k4 ^= k4 >> r
   k4 *= m

   h := seed

   h *= m
   h ^= k1
   h *= m
   h ^= k2
   h *= m
   h ^= k3
   h *= m
   h ^= k4

   h ^= h >> 13
   h *= m
   h ^= h >> 15

   return h
}
```