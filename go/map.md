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

load factor表 示bucket的最大容量和实际容量比值。

6.5.

如果太小，空间利用不当。太大那么就很容易出现碰撞的情况。

#### 桶的数量

如果桶的数量等于0，就会在赋值的时候再分配。

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