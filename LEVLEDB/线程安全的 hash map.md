#### 线程安全的 hash map

```go
type mBucket struct {
   mu     sync.Mutex
   node   []*Node
   frozen bool
}
```

cache使用bucket存储数据，开始的时候创建16个bucket，每个放32个数据。

- 总的数据量超过每个bucket32
- overflow的bucket超过128个

这个时候就要resize。



存在bucket里头的node用ns 和key表示

#### get

- get的时候使用hash函数选择对应的bucket。
- 获得mutex锁
- 遍历对应的bucket如果找到符合条件的node则返回
- 没找到则创建对应的node（如果设置了noset标签则没找到就返回）
- 将node添加过后，判断是否需要扩容。
- 需要扩容的话通过cas该表扩容标志位（防止已经有扩容真在进行中）
- 再通过cas改变旧cache指针的指向，让它指向新的cache
- （同时新的cache本身含有指向旧的cache的指针）
- 开启一个后台进程扩容
- 此时读写服务仍然正常提供。



由于CAS是原子操作

- 因此所有在cas（旧cache，新cache）之前的访问都将会访问到旧的cache

- 之后的访问都访问到新的cache



#### 后台进程如何扩容

- 逐个init新的bucket，将旧bucket的数据转移到新的bucket
- 转移完后，atomic操作将cache旧指针变为nil



#### 如何init bucket

- 旧bucket freeze
- 原子性的读取新bucket的指针以及新cache的指针
- 不断append旧bucket对印新bucket的数据的指针【】*node
- 通过【】*node生成一个bucket结构体
- CAS操作将新的bucket的指针从nil指向我们新创建的bucket结构体



-- initbucket没有加锁

-- 同时几个线程initbucket是可能的（但是我们通过atomicCAS操作进行了串行化）。



#### 新，旧 bucket并发限制

- get的时候，如果发现对应的bucket还没有初始化，则会帮忙初始化，因此新，旧bucket隔离。
- 对freeze的bucket桶的get操作将会直接返回。（确保取得最新的数据）
- get会不断for 循环，直到最新的bucket中的key

#### 难点

当我们resize的时候

- 对新bucket建立指针指向之前的物体
- 删除旧的指向原来物体的指针
- 删除旧的bucket

cas操作只能原子的改变一个变量的值





#### 删除

- 删除找到对应的bucket
- 获取锁
- 每个node上面引用计数减小一，然后等node为零才真正删除
- 如果删除导致当前存的数据太少了，那么就会shrink
- 同样cas改变resize标志位



- resize标志位用于shrink和grow串行

