```go


/*
Copyright 2013 Google Inc.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

     http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
*/

// Package consistenthash provides an implementation of a ring hash.
package consistenthash

import (
   "hash/crc32"
   "sort"
   "strconv"
)

type Hash func(data []byte) uint32

type Map struct {
   hash     Hash
   replicas int
   keys     []int // Sorted
   hashMap  map[int]string
}

func New(replicas int, fn Hash) *Map {
   m := &Map{
      replicas: replicas,
      hash:     fn,
      hashMap:  make(map[int]string),
   }
   if m.hash == nil {
      m.hash = crc32.ChecksumIEEE
   }
   return m
}

// IsEmpty returns true if there are no items available.
func (m *Map) IsEmpty() bool {
   return len(m.keys) == 0
}

// Add adds some keys to the hash.
func (m *Map) Add(keys ...string) {
   for _, key := range keys {
      for i := 0; i < m.replicas; i++ {
         hash := int(m.hash([]byte(strconv.Itoa(i) + key)))
         m.keys = append(m.keys, hash)
         m.hashMap[hash] = key
      }
   }
   sort.Ints(m.keys)
}

// Get gets the closest item in the hash to the provided key.
func (m *Map) Get(key string) string {
   if m.IsEmpty() {
      return ""
   }

   hash := int(m.hash([]byte(key))) // 注意[]byte转换成int，因为hash函数是uint32。

   // Binary search for appropriate replica.
   idx := sort.Search(len(m.keys), func(i int) bool { return m.keys[i] >= hash })

   // Means we have cycled back to the first replica.
   if idx == len(m.keys) {
      idx = 0
   }

   return m.hashMap[m.keys[idx]]
}
```



- add函数把节点的id加进去。
  - 我们对每一个节点id，生成id + 1的50个虚拟节点的hash
  - keys存储排序过后的hash值
  - map存储对应的hash的id
- get得到对应的key的hash
  - 使用二分查找法，找到keys中的i使得对应的hash大于key的hash
  - 从hashmap中取出对应的节点id。
  - 如果id为len（m）则认为没有节点大于，故为0。
  - 事实上是key对应的是逆时针上最接近的（这里由于是go二分查找的特殊性）

