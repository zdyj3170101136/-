#### map

Map 通过使用hash函数对key值进行映射。使用链表法解决碰撞问题。



每一个bucket 通过八位数组保留key hash的高八位。

使用extra字段把所有的bucket串联在一起

bmap之间会通过一个overflower字段串联bucket



![截屏2020-07-12 下午12.18.06](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-12 下午12.18.06.png)

关键是如果key和value都不是指针，那么就会把bmap标记为不含指针，这样子能够避免扫描整个hmap。

那么多余的overflow会在hmap中有个字段专门存储。



#### bucket

存储hash值的高八位。



内部紧密存储，

先是八个key，然后是八个value；

ong1k紧密存储某些情况下就可以省略掉内存对齐字段，节省空间。

#### 装载因子

load factor表 示bucket的最大容量。

6.5.

如果太小，空间利用不当。太大那么就很容易出现碰撞的情况。



*6.5* * 2 ^ B 个元素。



为什么6.5，其实我们看图就知道。

go官方有一个测试数据吧，就是说有overflow bucket的比例是20%差不多。。

而随着装载因子增大，overflow的增长速率要越来越快。



这个时候如果要找到一个已经出现的key，那么需要查找4.25次。

找到一个已经没有出现的key要找6.5次。



随着load facotr增大，这些消耗也会变大。
bytes/entry = 总字节数(100 个bucket * 20个overflowbukket，总占用120 * 8 * 8) / 实际的项数（120 * 6。5） = 9。8
// 考虑高位bit的存在那么可能接近10
因此在负载因子为6.5的时候，
一个8字节的数据，会导致20%的桶溢出，随着负载因子增大，这个溢出概率也越来越大，增大的越来越快
实际上存者10.8字节的数据对每个8字节的key，而随着负载因子增大，这个也不断减小，但是减小的越来越慢
查找一个存在的项的时候，要找4。25次，随着负载因子增大而增大，间隔不变。
查找一个不存在的项的时候，要找6.5次，随着负载因子增大而增大，间隔不变。

```
// Picking loadFactor: too large and we have lots of overflow
// buckets, too small and we waste a lot of space. I wrote
// a simple program to check some stats for different loads:
// (64-bit, 8 byte keys and elems)
//  loadFactor    %overflow  bytes/entry     hitprobe    missprobe
//        4.00         2.13        20.77         3.00         4.00
//        4.50         4.05        17.30         3.25         4.50
//        5.00         6.85        14.77         3.50         5.00
//        5.50        10.55        12.94         3.75         5.50
//        6.00        15.27        11.67         4.00         6.00
//        6.50        20.90        10.79         4.25         6.50
//        7.00        27.14        10.15         4.50         7.00
//        7.50        34.03         9.73         4.75         7.50
//        8.00        41.10         9.40         5.00         8.00
//
// %overflow   = percentage of buckets which have an overflow bucket
// bytes/entry = overhead bytes used per key/elem pair
// hitprobe    = # of entries to check when looking up a present key
// missprobe   = # of entries to check when looking up an absent key
//
```

#### 桶的数量

如果桶的数量等于0，就会在赋值的时候再分配。

也就是说连一个bucket都不到的时候。

然后分配在堆上。



如果能在栈上分配的map，会在编译的时候传进来。



```go
// makemap implements Go map creation for make(map[k]v, hint).
// If the compiler has determined that the map or the first bucket
// can be created on the stack, h and/or bucket may be non-nil.
// If h != nil, the map can be created directly in h.
// If h.buckets != nil, bucket pointed to can be used as the first bucket.
func makemap(t *maptype, hint int, h *hmap) *hmap {
```

```
// Maximum number of key/elem pairs a bucket can hold.
bucketCntBits = 3
bucketCnt     = 1 << bucketCntBits
```

hash值为64bit。



用后b个bit计算应该落入哪个桶。



当桶的数量较小时，只会创建2  ^ B个桶。

比较大时，还会生成2 ^ （B - 4）个溢出桶。

溢出桶的内存和实际的bucket的内存时连续的，这样方便一点。

#### hash定位

正常hash时64位指针，最后b位bit定位桶，而高八位bit则是找出key载桶里的位置。

#### key和value

key 和 value通过大小移动位置判断。

因此实际实现，对于int64，int32这种长度确定的值，会有专门的优化。

提前按照长度走。

#### 扩容

- 装载因子超过值6.5（很bucket块要满了）
- 或者overflow的桶数量过多；当b小于15的时候，如的果overflow的桶超过2^b；当b大于16，如果overflow的数量超过2^15.
- 有些bucket被停的插入很多元素，然后又被删除，导致多空洞的ooverfl

那么就会扩容。



- 对于第一种，就是直接分配两倍的bucket数量
- 

对于第二种， 要开辟一个大小相同的新bucket空间，然后把老元素移动过去。

#### 并发搬迁

将旧bucket搬迁到新butcke

不是一下完成的，而是我们调用m[a] = key;或者delete的时候搬迁的。

函数的动作是在 mapassign 和 mapdelete 函数中。也就是插入或修改、删除 key 的时候，都会尝试进行搬迁 buckets 的工作。先检查 oldbuckets 是否搬迁完毕，具体来说就是检查 oldbuckets 是否为 nil。（当发现当前正在进行扩容时）



每次搬原来的老bucket加上顺便多搬一个。



主要就是重新计算key的hash的低位，确保搬迁前后对于桶的映射不会出错。

然后如果没有协程正在使用老的bucket就把老的bucket清楚掉。



对于条件 1，从老的 buckets 搬迁到新的 buckets，由于 bucktes 数量不变，因此可以按序号来搬，比如原来在 0 号 bucktes，到新的地方后，仍然放在 0 号 buckets。

对于条件 2，就没这么简单了。要重新计算 key 的哈希，才能决定它到底落在哪个 bucket。例如，原来 B = 5，计算出 key 的哈希后，只用看它的低 5 位，就能决定它落在哪个 bucket。扩容后，B 变成了 6，因此需要多看一位，它的低 6 位决定 key 落在哪个 bucket。这称为 `rehash`。

![截屏2020-07-12 下午12.28.30](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-12 下午12.28.30.png)
作者：Stefno
链接：https://juejin.im/post/5ce4dd5ae51d4558936a9fde
来源：掘金
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

#### 随机遍历

一方面通过rehash，搬迁前后的key的位置不一样。

另一方面，当range的时候随机挑选一个bucketrange，这样能够帮助新手程序选。

#### math。NAN

这个函数的hash值每次计算都不一样。因此可以往一个map中插入任意多个math。nan。



你可能想到了，这样带来的一个后果是，这个 key 是永远不会被 Get 操作获取的！当我使用 `m[math.NaN()]` 语句的时候，是查不出来结果的。这个 key 只有在遍历整个 map 的时候，才有机会现身。所以，可以向一个 map 插入任意数量的 `math.NaN()` 作为 key。


作者：Stefno
链接：https://juejin.im/post/5ce4dd5ae51d4558936a9fde
来源：掘金
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

#### tophash

对于四位的hash值的老bucket；如果新的是5。

那么通常一个会分裂为两个，一个是新，一个是旧。

#### 遍历

遍历遍历的是新bucket。



遍历时由于有些bucket已经到了新的地方，而有些还没有。

比方说两倍扩容，有些分到了新的bucket有些分到了老的bucket。



因此虽然遍历时在新的bucket上遍历。但对于还没有搬迁的laobucket，则需要只取出对应的部分。

新 3 号 bucket 遍历完之后，回到了新 0 号 bucket。0 号 bucket 对应老的 0 号 bucket，经检查，老 0 号 bucket 并未搬迁，因此对新 0 号 bucket 的遍历就改为遍历老 0 号 bucket。那是不是把老 0 号 bucket 中的所有 key 都取出来呢？

并没有这么简单，回忆一下，老 0 号 bucket 在搬迁后将裂变成 2 个 bucket：新 0 号、新 2 号。而我们此时正在遍历的只是新 0 号 bucket（注意，遍历都是遍历的 `*bucket` 指针，也就是所谓的新 buckets）。所以，我们只会取出老 0 号 bucket 中那些在裂变之后，分配到新 0 号 bucket 中的那些 key。


作者：Stefno
链接：https://juejin.im/post/5ce4dd5ae51d4558936a9fde
来源：掘金
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

https://juejin.im/post/5ce4dd5ae51d4558936a9fde#heading-0

#### 字面量

对于字面量其实会在编译期间转换为对应的代码格式。

对于map其实就是使用make（）函数，然后一步步的赋值。





Go 语言中只要是可比较的类型都可以作为 key。除开 slice，map，functions 这几种类型，其他类型都是 OK 的。具体包括：布尔值、数字、字符串、指针、通道、接口类型、结构体、只包含上述类型的数组。这些类型的共同特征是支持 `==` 和 `!=` 操作符，`k1 == k2` 时，可认为 k1 和 k2 是同一个 key。如果是结构体，则需要它们的字段值都相等，才被认为是相同的 key。

顺便说一句，任何类型都可以作为 value，包括 map 类型。


作者：Stefno
链接：https://juejin.im/post/5ce4dd5ae51d4558936a9fde
来源：掘金
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

#### 顺序遍历
Golang中map的遍历输出的时候是无序的，不同的遍历会有不同的输出结果，如果想要顺序输出的话，需要额外保存顺序，例如使用slice，将slice中排序，再通过slice的顺序去读取。

先通过随机遍历获得全部的值。
然后通过排序加map取下标的方式顺序遍历。
【