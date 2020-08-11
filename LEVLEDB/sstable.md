

#### sstable



1. 第0层的sstable在4M左右;
2. 第i（i> 0）层的sstable每个sstable最大空间不超过2M;



sstable 是memtable经过minor compaction生成的磁盘数据。

- 避免日志文件过大
- 降低内存使用率

#### block

注意block的crc校验范围包括数据以及压缩类型。。。（重点）

![截屏2020-07-24 下午2.20.29](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-24 下午2.20.29.png)

![img](https://leveldb-handbook.readthedocs.io/zh/latest/_images/sstable_logic.jpeg)



- filter block：bloom过滤器
- datab block：数据
- index block：存储data block的元数据
- meta index block：存储filter的元信息
- footer： index block和meta index block的元数据



以block的形式存储。

每个block4kb。

注意大小不存在于block中。

而是由偏移量的形式存储在元信息中。



- 压缩类型（snappy）
- crc校验

#### datablock

datablock分为restart point和具体的key value。

![img](https://leveldb-handbook.readthedocs.io/zh/latest/_images/datablock.jpeg)

对于restart point，每隔16个key记录一个key的内容。

不然则记录与上一个key不同的部分

- 通过restart point快速定位查找，再细粒度比较

  

- shared key ｜ unshred key ｜ value length ｜ unshared key ｜ value

#### filter block![截屏2020-07-24 下午2.23.18](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-24 下午2.23.18.png)

![img](https://leveldb-handbook.readthedocs.io/zh/latest/_images/filterblock_format.jpeg)

filter block主要是每2kb放一个过滤器存放数据。

- 按照数据而不是block切分，主要某些block可能key比较小。



- sstable元数据和数据一起存放，不用动态计算和维护更新

#### index block

- 与meta index block类似，index block用来存储所有data block的相关索引信息。

  indexblock包含若干条记录，每一条记录代表一个data block的索引信息。

  一条索引包括以下内容：

  1. data block i 中最大的key值；
  2. 该data block起始地址在sstable中的偏移量；
  3. 该data block的大小；



![截屏2020-07-16 下午1.04.09](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-16 下午1.04.09.png)

通过index block 比较在哪个data blcok。

通过data blcok的retart point找到精确的点



footer 报考

- meta inde
- index的偏移量。

- magic word。





1. 首先判断“文件句柄”cache中是否有指定sstable文件的文件句柄，若存在，则直接使用cache中的句柄；否则打开该sstable文件，**按规则读取该文件的元数据**，将新打开的句柄存储至cache中；
2. 通过footer找到index block和meta index bloack。
3. 利用sstable中的index block进行快速的数据项位置定位，得到该数据项有可能存在的**两个**data block；
4. 利用index block中的索引信息，首先打开第一个可能的data block；
5. 利用filter block中的过滤信息，判断指定的数据项是否存在于该data block中，若存在，则创建一个迭代器对data block中的数据进行迭代遍历，寻找数据项；若不存在，则结束该data block的查找；
6. 若在第一个data block中找到了目标数据，则返回结果；若未查找成功，则打开第二个data block，重复步骤4；
7. 若在第二个data block中找到了目标数据，则返回结果；若未查找成功，则返回`Not Found`错误信息；



cache

- sstable文件句柄及其元数据；
- data block中的数据；



#### 为什么是两个

- index block每两个才会存储一个完整的key



#### 并发访问友好

- sstable是只读的。
- 元数据一起存储
- 引用计数实现无锁并发
- cache中的数据永远一致。
- sstable有版本号





#### snappy算法原理

![截屏2020-07-24 下午1.38.27](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-24 下午1.38.27.png)



从左到右遍历字符。

字典分为（prefixindex， string）。

每一个index都代表一串字符。

如果我们遇到了一串字符，我们就做最长前缀匹配，然后string是不匹配的字符。

我们有一个字典，number - 字符序列

- 若当前字符c未出现在词典中，则编码为`(0, c)`；
- 若当前字符c出现在词典中，则与词典做最长匹配，然后编码为`(prefixIndex,lastChar)`，其中，prefixIndex为最长匹配的前缀字符串，lastChar为最长匹配后的第一个字符；
- 为对最后一个字符的特殊处理，编码为`(prefixIndex,)`。

https://www.cnblogs.com/en-heng/p/6283282.html



snappy不追求压缩率，压缩率够用的情况下，最快。。。

