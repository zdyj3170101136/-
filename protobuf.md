#### tag-wireType-value

// 要么使用自描述的，比如varant

// 要么使用长度



- Tag是每个字段后的1，2，3
- wiretype指的是数据类型和value的长度

0:varint：可变长度（所有的int，## sint

，bool，enum）

1:64-bit：（double）

2:length-delimited（阅读前会含有长度）：string，bytes，嵌套对象

- value指的是实际的值





值为 tag << 3 | wireType的混合编码形式（可见一个字节总共八位，低三位表示wiretype）

#### varant

每个varant的第一个字节的第一位表示是否还有更多的字节来表示。

对于1，0000 0001.

而对于300.

1010 1100 0000 0010

：由于为little endian编码，所以。

000 0010    010 1100

拼接在一起

#### sint

Sint 相比较int在表示负数的时候有区别。

sint其实就是正负交错编码

0 0

-1 1

1 2

-2 3

在表示负数的时候int由于标志位的存在本质上

是用一个很大的正数来表示的。

#### enum，varint

enum，varint也是使用bool来编码

#### string

string是通过一个varnt 表示长度，随后是字符串的内容

#### 嵌套对象

一个结构体嵌套另一个结构体。

就会跟着嵌套对像的长度。

#### Repeated

对于repeated，如果同一个tag出现多次，就看成在一个数组里头的。

解码的时候merge。

- packed，packed不会对每个都增加一个tag，取而代之的是长度



https://www.jianshu.com/p/4b987ef826d3

#### optional

- 在 Protobuf 2 中，消息的字段可以加 required 和 optional 修饰符，也支持 default 修饰符指定默认值。默认配置下，一个 optional 字段如果没有设置，或者显式设置成了默认值，在序列化成二进制格式时，这个字段会被去掉，导致反序列化后，无法区分是当初没有设置还是设置成了默认值但序列化时被去掉了，即使 Protobuf 2 对于**原始数据类型**字段都有 hasXxx() 方法，在反序列化后，对于这个“缺失”字段，hasXxx() 总是 false——失去了其判定意义。
- 在 Protobuf 3 中，更进一步，直接去掉了 required 和 optional 修饰符，所有字段都是 optional 的， 而且对于**原始数据类型**字段，压根不提供 hasXxx() 方法。来自 Google 的 GRPC 核心成员Eric Anderson 在 StackOverflow 网站很好的解释了这个设计决策的原因：[Why required and optional is removed in Protocol Buffers 3](https://link.zhihu.com/?target=https%3A//stackoverflow.com/questions/31801257/why-required-and-optional-is-removed-in-protocol-buffers-3)

#### biggle -endian

数据的最高有效字节在地址最低处。



例如假设上述变量`x`类型为`int`，位于地址`0x100`处，它的值为`0x01234567`，地址范围为`0x100~0x103`字节，其内部排列顺序依赖于机器的类型。大端法从首位开始将是：`0x100: 01, 0x101: 23,..`。而小端法将是：`0x100: 67, 0x101: 45,..`。



Big-Endian优点：靠首先提取高位字节，你总是可以由看看在偏移位置为0的字节来确定这个数字是正数还是负数。你不必知道这个数值有多长，或者你也不必过一些字节来看这个数值是否含有符号位。这个数值是以它们被打印出来的顺序存放的，所以从二进制到十进制的函数特别有效。因而，对于不同要求的机器，在设计存取方式时就会不同。 [1] 



Little-Endian优点：提取一个，两个，四个或者更长字节数据的汇编指令以与其他所有格式相同的方式进行：首先在偏移地址为0的地方提取最低位的字节，因为地址偏移和字节数是一对一的关系，多重精度的数学函数就相对地容易写了。

计算机从低地址到高地址处理。