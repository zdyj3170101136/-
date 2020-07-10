## Tcp

Transmission control protocol(运输控制协议)

为了比特差错信道，引入了检验和，重传，重复分组。

![img](https://img-blog.csdn.net/20150428144449522?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbGlzaGVuZ2xvbmc2NjY=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

虚线表示服务器，实现表示客户端。



注意两边都能主动关闭，主动的变成fin_wait1状态，被动的变成close_wait状态

#### 格式

16位源端口号 ｜ 16位目的端口号

32位序号

32位确认号

4位首部长度 ｜ 6比特标志字段 ｜16位接收窗口

16位检验和 ｜ 紧急数据指针（无用）

数据

#### 校验和

ip/tcp/udp校验和原理都是一样的。

ip中的校验和只对ip首部。

icmp覆盖整个icmp报文（icmp头部加数据）

tcp/udp对整个tcp报文加上为首部校验。（udp检验和可选）



仅用于差错检测，不用于恢复



- 对报文每16位相加

- 结果取反作为校验和（每个数取反相加结果和相加后取反结果一样，这样可以减少运算次数）
- 校验时，把所有字节相加，包括检验和（最后都为1的话则通过）





1000 + 0100 = 1100

0111 + 1011 = 10010

10010+ 01101 = 11111



#### 实现

使用一个32位的数，不断把它和16位的数相加。

高16位保存进位。

最后低16位和高16位相加。

最后取反



#### 为何用反吗

计算校验和与字节序无关。

https://blog.csdn.net/tangchenchan/article/details/51212440

![截屏2020-07-09 下午5.12.51](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-09 下午5.12.51.png)

#### MSS

mss（最大报文段长度1460）：tcp报文段的数据大小（被封装在ip和tcp里头）（40字节首部）能通过最大链路层帧长度（1500）（以太网链路层）

#### 序列号和确认号

序列号：首个数据的字节流编号（初始随机，减少已经终止的连接的报文段被误认为是之后的连接的可能性（也许碰巧使用相同ip和端口号））

确认号：期望接收到的<u>下一字节</u>的编号



累积确认机制：只确认到该流第一个丢失的字节

#### Sync_retries = 6

在发送syn报文后没有接受到确认报文的最大重试时间：127s

2^0 + 2^1 + 2^2 + 2^3 + 2^4 + 2^5 + 2^6

https://www.dazhuanlan.com/2019/10/20/5dab43fbaadb1/

#### 超时重传

当报文段大于一定时间还没有收到，就会重传。

超时间隔不能太小（大于EstimatedRTT），避免不必要重传,因此要有一定余量。

不能太大，不然长时间不能重传，数据时延大。



tcp每过一段时间就会计算SamleRTT（报文段发出到确认被收到的时间），作为接下来的报文段的RTT。（不会为已经被重传的计算）

EstimatedRTT = sampleRTT * a + (1 - a) * EstimatedRTT

a = 7 / 8；指数加权，平缓波动。



DevRTT = （1 - b） * DevRTT + b *|sampleRTT - EstimatedRTT|。

反映SampleRTT波动程度，b = 0.25.



当SampleRTT波动大时，余量也大；波动小时，余量也小；

TimeoutiNterval = EstimatedRtt + 4 * DEVRTT

初始为1，超时后加倍；

只要收到ACK或者上层应用数据就会重新计算。



#### tcp_retries

Tcp_retries1: 3,超过这个次数，就会更新路由缓存(下一跳缓存)，然后mtu探测（底层链路层可能使用小于MTU）

tcp_retries2:15,超过这个次数就放弃重传。



然而，真正限制的是重传最大尝试时间。

以RTO_BASE = 200ms，计算尝试boundary次的的最大重传时间。

当大于这个时间，次数没到retries2也会放弃。



因此真正重传次数与RTT相关，当RTT比较小时，次数也会比较多。

```
  linear_backoff_thresh = ilog2(TCP_RTO_MAX/rto_base);

            if (boundary <= linear_backoff_thresh)
                    timeout = ((2 << boundary) - 1) * rto_base;
            else
                    timeout = ((2 << linear_backoff_thresh) - 1) * rto_base +
                            (boundary - linear_backoff_thresh) * TCP_RTO_MAX;
```

http://perthcharles.github.io/2015/09/07/wiki-tcp-retries/

https://segmentfault.com/a/1190000020183650

#### 比特差错的信道

由于实际信道会有比特差错。引入重传，检验和，ACK, 重复分组

因此

- 差错检测，TCP首部+数据+假首部（包括目的IP地址，源ip地址，tcp协议号，tcp数据段长度等内容）以16bit为单位求和，然后取反得到检验和。接收端把计算所有的和加上检验和后为1111111（16bit）

- 接守方反馈，接收方用肯定确认（ACK）来确认接受。

- 重传，发送方重传

  如果ACK受损

  一，如果询问ACK的意思，则询问的内容也会受损。

  二，如果增加检验和，让差错恢复。

  三，收到模糊不清的ack，直接重传当前数据。产生重复分组。



实际：使用方法三。

采用序号解决重复分组。



当接受到受损的分组，对上一次收到的正确分组重传一次ack。发送方接受到同一分组的两个ack就知道接收方没有正确接受之后的分组。



最开始采用停等协议（发送下个分组直到接收到确认后），一次只发送一个，因此序号只需要0， 1。

#### 丢包信道

通过定时器应对分组丢失。，定时器。

发送方未收到确认，不知道是因为ack丢失，还是发送的数据丢失。

因此：重传。

对每一个发送数据开启一个定时器，超时则重传。

#### 可靠传输

- 数据校验检查比特错误
- 肯定确认和重传机制应对模糊不清的ack
- 序号解决重传导致的重复分组问题
- 使用定时器的超时重传机制解决数据丢失问题
- 数据切片
- 对ip数据报重新排序保存数据正确性
- 流量控制
- 拥塞控制

#### 流水线传输

发送多个分组无需等待确认。

- 增大序号范围，采用32bit的序号
- 缓存分组，缓存已经发送但未确认的分组
- 累积确认，确认号表示所有已经接受的。只重传一个，当超时发生时，只会重传超时的报文段。



接收方对于重复的报文段重复确认，避免发送窗口无法移动。

#### 流量控制

为了防止接受缓存溢出。使用接收窗口。

接收窗口：发送端维护最后一个发送的字节流 - 最后一个接收到的ack小于rwnd。

rwnd动态更新。

rwnd = 接收方应用最后一个读取的字节 - 缓存里头的字节。

#### 为什么需要三次握手

发送方发送SYN了， 初始序列号， 进入SYNC_SEND；接受方（LISTEN）发送SYN， ACK， 进入SYNC_ECVD， 自己的初始序列号，发送方序列号 + ！；接收方发送ACK进入ESTABLISH,发送方发送ACK也进入ESTABLISH（并且携带数据）。

1) A --> B SYN my sequence number is X 2) A <-- B ACK your sequence number is X 3) A <-- B SYN my sequence number is Y 4) A --> B ACK your sequence number is Y

RFC 793



- 为了避免连接复用时无法分辨出seq是延迟的还是上一条链接的。

- 随机初始化序列号。（本质上是由于重传机制带来的包重复而不是过期链接，保证包不重复）

这个生成器会用一个32位长的时钟，差不多`4µs` 增长一次，因此 ISN 会在大约 4.55 小时循环一次

（`2^32`位的计数器，需要`2^32*4 µs`才能自增完，除以1小时共有多少µs便可算出`2^32*4 /(1*60*60*1000*1000)=4.772185884` ）

而一个段在网络中并不会比最大分段寿命（Maximum Segment Lifetime (MSL) ，默认使用2分钟）长，MSL 比4.55小时要短，所以我们可以认为 ISN 会是唯一的。



这时由发送方来判断当前连接是否是历史连接：

- 如果当前连接是历史连接，即 `SEQ` 过期或者超时，那么发送方就会直接发送 `RST` 控制消息中止这一次连接；
- 如果当前连接不是历史连接，那么发送方就会发送 `ACK` 控制消息，通信双方就会成功建立连接；

https://juejin.im/post/5c37f36b518825261f7350fa

第一，第二次没有数据为了防止泛洪攻击。



更多次数没有意义：
因为四次肯定能做到，三次最小就ok

CP 协议的设计可以让我们同时传递 `ACK` 和 `SYN` 两个控制信息，减少了通信次数，所以不需要使用更多的通信次数传输相同的信息；

https://draveness.me/whys-the-design-tcp-three-way-handshake/

但是两次绝对不行。

#### 半连接队列

没有完全三次握手

#### SYNC COOKIE

服务器资源在接受到发送端的ACK后才会分配资源；

为了防止发送者发送一堆syn报文来泛洪攻击。

步骤：当服务器接受到一个syn后，以源和目的ip，端口号， 当前时间计算一个cookie，作为初始序号发送SYNACK分组。并且不记忆任何东西。

当服务器接受到ACK后，重新计算cookie，如果发现等于ACK - 1，则分配资源。

（懒惰启用），仅仅当服务器满载时使用。

#### 拥塞控制

代价

- 巨大排队时延，当分组到达速率太快时，经历巨大的排队时延
- 需要重传以补偿因为缓存溢出而丢弃的分组
- 大时延可能会导致不必要的重传浪费带宽
- 每个上游路由器用于转发分组的流量将会被浪费

所以不能一次发送大量的包。



拥塞窗口cwnd：发送方可发送字节数最大值。

发送窗口取拥塞窗口和接收端窗口的最小值。



![截屏2020-07-09 下午8.27.36](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-09 下午8.27.36.png)

感知拥塞：

超时或者三个接收方的重复ack。



一，慢启动，初始时，cwnd的值为一个MSS值。每次收到传输报文段的首次确认就增加一个MSS。因此慢启动增长快，以指数的形式增长。ssthresh位64kb。

何时结束指数增长：

- 接受到丢包时，设置cwnd为1，并且sstresh为cwnd的一半（注意慢启动的丢包会重新导致进入慢启动状态）
- 当到达sstresh时，结束慢启动进入拥塞避免阶段
- 收到三个重复ack，执行快速重传（在定时器过期前重传报文），ssthresh等于一半的cwnd，将cwnd的值减半（然后加上3个MSS（一定的余量）），进入快速恢复。



二，拥塞避免，

每个RTT只将cwnd增加一个MSS。（通过对一个让rtt内发送的十个则一个ack只增长mss /10）

- 当出现超时时，cwnd重新设置为1，ssthresh = 一半的cwnd，进入慢启动
- 收到三个重复ack时，ssthresh等于一半的cwnd，将cwnd的值减半（然后加上3个MSS），进入快速恢复



三，快速恢复

tcp重传丢失的数据，如果只收到DUPACK，那么每个增加一个MSS。

- 收到新的ack，cwnd = ssthresh，进入拥塞避免状态。（从快速重传进入拥塞避免不用变为1，一个方便的）

- 当超时发生，cwnd设置为1， ssthress设置为一半的cwnd，进入慢启动



所以本质上：

超时进入慢启动

三个重复ack进入快速重传

新的ack从快速重传进入拥塞避免

#### SACK

选择确认，由于选项部分最大是40字节，所以最多四段。

![截屏2020-06-17 下午5.47.52](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-06-17 下午5.47.52.png)

#### 同时打开，同时关闭

同时打开：两边同时发动SYN和SYNACK，建立了一条连接。

同时进入sync_sent状态->接受到syn后发送syn和ack-》进入sync_rcvd状态->同时接受到syn和ack后进入establish状态

![截屏2020-07-09 下午5.47.14](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-09 下午5.47.14.png)了四个报文

![截屏2020-07-09 下午5.46.56](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-09 下午5.46.56.png)

#### 关闭

ESTABLISH主动端发送一个FIN报文，进入FIN_WAIT_1状态，当接受到ACK时进入FIN_WAIT_2状态。然后接受FIN（一直没有接受到FIN，超时后会释放资源）再发送ACK， 进入TIMEWAIT状态，等待30s后进入CLOSED状态。



被动端收到FIN后发送ACK进入CLOSE_WAIT状态，发送FIN，进入LAST_ACK状态，再接受到ACK后关闭。



FINZ_WAIT_2状态关闭发送，只能接受。半关闭。



#### TIME_WAIT：

主动发送方等待两个2 * MSL（报文段最大生存时间一个msl，2min），保证对方收到ACK，如果没有收到，则对方重发FIN，本地将会重发ACK)（大于ttl）

https://blog.51cto.com/10706198/1775555

#### ttl

ttl是ip字段，每经过一个路由器减1，为零则发送icmp报文通知主机

#### 为什么需要四次挥手

主动方在进入了FIN_WAIT2状态后，保留接受功能；

这样被动方没有发送的数据发送完。发送完成之后，被动方也发送一个FIN相当于告诉主动方：自己数据已经发送完毕。对面可以关闭接受功能。



主动关闭方如果不进入TIME_WAIT，被动关闭方将会重传FIN，这个时候关闭方无法识别，会回复RST。



- 服务器关闭大量连接，出现大量TIME_WAIT状态，持续消耗资
- 客户端会消耗掉多有的端口，导致无连接可用

可以设置tcp_max_tw_buckets控制最大timewait数量

#### 同时关闭

ESTABLISH发送一个FIN，同时接受到FIN。

Fin_wait_1-》进入CLOSING状态-》接受到ACK-〉TIMEWAIT

#### KEEPALIVE

即使连接双方无数据发送，也会每隔2h发送一个数据包。

确保对方存活。

#### RST

在任何状态下，收到RST，则进入CLOSED状态。

发送端丢弃缓冲全部数据，接受端收到后也没有ACK

发生情况：

- 发送到不存在的端口
- 关闭一个非空的socket

#### SOCKET

- 监听套接字，处理网络上来的连接
- 传输套接字：向外传输tcp数据
- 待处理套接字：三次握手还没有完成

socket发送只是把数据拷贝到缓冲区，拷贝完饭回。

#### CLOSE_WAIT

服务器端如果积攒大量的COLSE_WAIT状态的socket，有可能将服务器资源减少，套筒无法提供服务：一个进程打开一个socket，然后此进程再派生子进程的时候，此socket的sockfd会被继承，socket是系统级的对象，因此socket的引用计数会变成2，调用close（sockfd）时，内核检查此fd对应的socket上的引用计数。如果引用计数大于1，那么将这个引用计数减1，然后返回，如果引用计数等于1，那么内核会真正通过发FIN来关闭TCP连接，调用shutdown（sockfd，SHUT_RDWR）时，内核不会检查此fd对应的socket上的引用计数，直接通过发FIN来关闭TCP连接

https://blog.csdn.net/realmeh/article/details/18239667

#### TCP沾包

Nagle算法，将多次间隔较小，数据量较小的数据，合并发送。

- 同一个连接，多个应用层的数据混在一起，一个包会混杂好几种应用层数据
- 发送当沾包，用于TCP为了优化，会把数据合在一起再传送
- 接受方沾包，是因为接收方没有及时取走数据。导致多个包的数据都在缓冲区，而接收方按固定的大小读取数据。

先写长度，把消息的长度发送出去。

https://blog.csdn.net/tiandijun/article/details/41961785



#### 糊涂窗口综合症

流量控制实现不良

接收方缓慢消耗，发送方缓慢产生数据都会导致这个问题。

由于服务器无法处理很多数据，会不断减小窗口大小。

最后可能不满一个报文，在几十到几百字节抖动。·

发送方：

- 只能最多一个未被确认的小段（当数据包大小达到mss立即发送，）

  https://blog.csdn.net/hzhsan/article/details/46429749

接收方（clark）

- 延迟向发送方修改窗口大小，除非可以增加一个报文段大小。或者接受缓存半空。

#### CUBIC

由于在拥塞避免状态，每隔一个RTT窗口大小才加1，这样太慢了。



采用一种三次函数的方法。

发生丢包时的窗口大小为Wmax，而被乘以0.2后减到了W2岁后进入拥塞避免阶段，则最大窗口应该在Wmax和Wmin的中间。

![img](https://allen-kevin.github.io/pictures/paper%20read/20171223095005.png)

在拥塞避免阶段，使用时间t（距离上一次丢包的时间间隔）来计算窗口大小。

当大于Wmax，则依照凸的曲线探测。



- 开始时很快

- 接近Wmax时增长慢

- b = 0.2 《 0.5是为了让收敛更慢，协议更为稳定

- 相比较BIC算法的二分查找，以及每RTT查找一次，导致RTT小的更容易到达顶点

  https://blog.csdn.net/dog250/article/details/53013410

  #### 最大tcp连接

  四元组来唯一标识一个TCP连接：{local ip，本地端口，远程ip，远程端口}，最大tcp连接为客户端ip数×客户端端口数，对IPV4，不考虑ip地址分类等因素，最大tcp连接数大约2 ^ 32（ip数）* 2 ^ 16（端口数）

#### Nigle

避免网络中充斥着大量的小型数据包



对于大于MSS的数据直接发送。

如果仍然有未确认的小数据包，将当前数据塞进缓冲区。

（最多一个未被确认的小数据包）

https://blog.csdn.net/sinat_35261315/article/details/79392116?utm_medium=distribute.pc_relevant_t0.none-task-blog-BlogCommendFromMachineLearnPai2-1.nonecase&depth_1-utm_source=distribute.pc_relevant_t0.none-task-blog-BlogCommendFromMachineLearnPai2-1.nonecase

#### 延迟确认

一个报文到达，不会立马回复ack；

而是会延迟回复，希望顺便能将数据和ack一起回复。

（只能延迟单个，如果有两个延续的报文则还是得回答）

https://www.cnblogs.com/zhangkele/p/10080845.html

如果nagle和延迟确认混在一起会很麻烦。



- nagle发送一个，等待确认
- 延迟回复方等待200ms超时后才会回复ack
- 于是nagle又重新发送ack

#### bbr

#### 丢包算法的缺陷

- 高速尤其是无线网络，链路错误率很高的，不能认为丢包代表拥塞。
- 丢包算法往往表示表示的是路由器缓存溢出，而路由器缓存溢出时候的速度不代表最快速度。



![截屏2020-07-09 下午9.09.18](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-09 下午9.09.18.png)

- RTprop：光信号从A端到B端的最小时延（其实是2倍时延，因为是一个来回，但不影响讨论），这取决于物理距离
- BtlBw：在A到B的链路中，它的带宽取决于最慢的那段链路的带宽，称为瓶颈带宽，可以想象为光纤的粗细
- BtlBufSize：在A到B的链路中，每个路由器都有自己的缓存，这些缓存的容量之和，称为瓶颈缓存大小
- BDP：整条物理链路（不含路由器缓存）所能储藏的比特数据之和，BDP = BtlBw * RTprop

#### 因此上半图

- 当链路传输的数据没有超过（最慢的链路的带宽 * 光信号最短时延），传输速度在时延。
- 之后，路由器开始启动缓存来存储比特数据，因此通过一个斜率增长
- 当路由器缓存填满后，开始丢失数据。

#### 下半图

- 开始的时候传送速率单调增长
- 到达链路极限带宽后不变。

#### 四个状态

![截屏2020-07-09 下午9.18.21](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-09 下午9.18.21.png)

- 当连接建立，指数倍增发送速度，如果三次之后发现这个时间延迟不再增长，就说明管道被填满。
- 排空：排空阶段，指数降低发送速率，将多占的两倍buffer慢慢排空。
- 探测状态：不断尝试增大发送速率，看带宽是否还能增长，没有则降低排空多处来的包。
- 延迟探测：每过10s，如果延迟不变，就延迟探测，这段时间只发送很少量的包，不占用缓存，看看最快在哪里。
- https://blog.csdn.net/ebay/article/details/76252481
- https://www.jianshu.com/p/08eab499415a

#### 由于tcp的重传机制

发送端发送的时候记录时间戳。

接受端回复ack时候使用发送端的时间戳。

发送端收到tck后，用当前时刻 - ack中的time就能得到ett。

https://perthcharles.github.io/2015/08/27/timestamp-intro/



所以计算rtt不精确，通过timestamp精确计算rtt

https://blog.csdn.net/u011130578/article/details/44918667?utm_medium=distribute.pc_relevant_t0.none-task-blog-BlogCommendFromMachineLearnPai2-1.compare&depth_1-utm_source=distribute.pc_relevant_t0.none-task-blog-BlogCommendFromMachineLearnPai2-1.compare



序列号快速回绕。（使用同一个地址的nat两台机器时间不同步）

由于tcp序列号32位且以字节计数。传送大数据时很快会绕过。

难以发现这个是新的还是旧的数据。



而时间戳就可以解决这个问题。