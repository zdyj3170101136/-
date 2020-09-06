#### guid

生成一个分布式系统的唯一id

- 如果发生了时间倒流，出错
- 相同时间，通过10位的sequence序列号区分，如果sequence用完了，返回错误。

```go
func (f *guidFactory) NewGUID() (guid, error) {
   f.Lock()

   // divide by 1048576, giving pseudo-milliseconds
   ts := time.Now().UnixNano() >> 20

   if ts < f.lastTimestamp {
      f.Unlock()
      return 0, ErrTimeBackwards
   }

   if f.lastTimestamp == ts {
      f.sequence = (f.sequence + 1) & sequenceMask
      if f.sequence == 0 {
         f.Unlock()
         return 0, ErrSequenceExpired
      }
   } else {
      f.sequence = 0
   }

   f.lastTimestamp = ts

   id := guid(((ts - twepoch) << timestampShift) |
      (f.nodeID << nodeIDShift) |
      f.sequence)

   if id <= f.lastID {
      f.Unlock()
      return 0, ErrIDBackwards
   }

   f.lastID = id

   f.Unlock()

   return id, nil
}

// hex使用当前的guid生成一个messageid，大小为16字节。
func (g guid) Hex() MessageID {
	var h MessageID
	var b [8]byte

	b[0] = byte(g >> 56)
	b[1] = byte(g >> 48)
	b[2] = byte(g >> 40)
	b[3] = byte(g >> 32)
	b[4] = byte(g >> 24)
	b[5] = byte(g >> 16)
	b[6] = byte(g >> 8)
	b[7] = byte(g)

	hex.Encode(h[:], b[:])
	return h
}

```



- 测试hex copy16个字节的性能
- 测试类型转换的性能消耗
- 测试生成新的guid和hex转换的性能损耗

```go
func BenchmarkGUIDCopy(b *testing.B) {
   source := make([]byte, 16)
   var dest MessageID
   for i := 0; i < b.N; i++ {
      copy(dest[:], source)
   }
}

func BenchmarkGUIDUnsafe(b *testing.B) {
   source := make([]byte, 16)
   var dest MessageID
   for i := 0; i < b.N; i++ {
      dest = *(*MessageID)(unsafe.Pointer(&source[0]))
   }
   _ = dest
}

func BenchmarkGUID(b *testing.B) {
   var okays, errors, fails int64
   var previd guid
   factory := &guidFactory{}
   for i := 0; i < b.N; i++ {
      id, err := factory.NewGUID()
      if err != nil {
         errors++
      } else if id == previd {
         fails++
         b.Fail()
      } else {
         okays++
      }
      id.Hex()
   }
   b.Logf("okays=%d errors=%d bads=%d", okays, errors, fails)
}
```