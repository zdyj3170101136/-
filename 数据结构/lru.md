package main

import (
	"encoding/json"
	"expvar"
	"fmt"
	"github.com/syndtr/goleveldb/leveldb"
	"os"
)

func main() {
	leveldb.Batch{}
	var a []int
	r := json.NewDecoder(os.Stdin)
	for {
		if !r.More() {
			break
		}
		r.Decode(&a)
		fmt.Println(minimumMoves(a))
	}
}
type List struct {
	 root Element // 为element而不是*element就是为了init方便
	 len int
}

// Init initializes or clears list l.
func (l *List) init() *List {
	l.root.next = &l.root
	l.root.prev = &l.root
	l.len = 0
	return l
}

func (l *List) lazyinit() {
	if l.root.next == nil {
		l.init()
	}
}

func (l *List) PushFront(v interface{}) *Element {
	l.lazyinit()
	return l.insert(&Element{v:v}, &l.root)
}

func (l *List) Len() int {
	return l.len
}

func (l *List) Back() *Element {
	return l.root.prev
}

// 检查是否无需移动，并且list相同才移动
func (l *List) MoveToFront(e *Element) {
	// move是已经找到，这个时候不需要init
	if e.l != l || l.root.next == e {
		return
	}
	// see comment in List.Remove about initialization of l
	l.move(e, &l.root)
}

// move moves e to next to at and returns e.
func (l *List) move(e, at *Element) *Element {
	if at == e {
		return e
	}
	e.prev.next = e.next // 更改e的指向
	e.next.prev = e.prev
	// 插入
	return l.insert(e, at)
}

func (l *List) insert(a, b *Element) *Element {
	o := b.next
	b.next = a
	a.prev = b
	a.next = o
	o.prev = a
	a.l = l
	l.len++
	return a
}

func (l *List) Remove(e *Element) {
	// move是已经找到，这个时候不需要init
	if e.l != l {
		return
	}
	e.prev.next = e.next
	e.next.prev = e.prev
	e.next = nil // avoid memory leaks
	e.prev = nil // avoid memory leaks
	e.l = nil    // 重点是go中的避免内存泄漏。。。。。。。。。。。。。。。。。。。。。。。。
	l.len--
}

type Element struct {
	prev *Element
	next *Element
	l *List
	v interface{}
}

```
// Cache is an LRU cache. It is not safe for concurrent access.
type Cache struct {
   // MaxEntries is the maximum number of cache entries before
   // an item is evicted. Zero means no limit.
   MaxEntries int

   // OnEvicted optionally specifies a callback function to be
   // executed when an entry is purged from the cache.
   OnEvicted func(key Key, value interface{})

   ll    *list.List
   cache map[interface{}]*list.Element
}

// A Key may be any value that is comparable. See http://golang.org/ref/spec#Comparison_operators
type Key interface{}

type entry struct {
   key   Key
   value interface{}
}

// New creates a new Cache.
// If maxEntries is zero, the cache has no limit and it's assumed
// that eviction is done by the caller.
func New(maxEntries int) *Cache {
   return &Cache{
      MaxEntries: maxEntries,
      ll:         list.New(),
      cache:      make(map[interface{}]*list.Element),
   }
}

// Add adds a value to the cache.
func (c *Cache) Add(key Key, value interface{}) {
   if c.cache == nil {
      c.cache = make(map[interface{}]*list.Element)
      c.ll = list.New()
   }
   if ee, ok := c.cache[key]; ok {
      c.ll.MoveToFront(ee)
      ee.Value.(*entry).value = value
      return
   }
   ele := c.ll.PushFront(&entry{key, value})
   c.cache[key] = ele
   if c.MaxEntries != 0 && c.ll.Len() > c.MaxEntries {
      c.RemoveOldest()
   }
}

// Get looks up a key's value from the cache.
func (c *Cache) Get(key Key) (value interface{}, ok bool) {
   if c.cache == nil {
      return
   }
   if ele, hit := c.cache[key]; hit {
      c.ll.MoveToFront(ele)
      return ele.Value.(*entry).value, true
   }
   return
}

// Remove removes the provided key from the cache.
func (c *Cache) Remove(key Key) {
   if c.cache == nil {
      return
   }
   if ele, hit := c.cache[key]; hit {
      c.removeElement(ele)
   }
}

// RemoveOldest removes the oldest item from the cache.
func (c *Cache) RemoveOldest() {
   if c.cache == nil {
      return
   }
   ele := c.ll.Back()
   if ele != nil {
      c.removeElement(ele)
   }
}

func (c *Cache) removeElement(e *list.Element) {
   c.ll.Remove(e)
   kv := e.Value.(*entry)
   delete(c.cache, kv.key)
   if c.OnEvicted != nil {
      c.OnEvicted(kv.key, kv.value)
   }
}

// Len returns the number of items in the cache.
func (c *Cache) Len() int {
   if c.cache == nil {
      return 0
   }
   return c.ll.Len()
}

// Clear purges all stored items from the cache.
func (c *Cache) Clear() {
   if c.OnEvicted != nil {
      for _, e := range c.cache {
         kv := e.Value.(*entry)
         c.OnEvicted(kv.key, kv.value)
      }
   }
   c.ll = nil
   c.cache = nil
}
```

