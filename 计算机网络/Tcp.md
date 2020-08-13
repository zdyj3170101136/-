Tcp

Transmission control protocol(运输控制协议)

为了比特差错信道，引入了检验和，重传，重复分组。

![img](https://img-blog.csdn.net/20150428144449522?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbGlzaGVuZ2xvbmc2NjY=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

虚线表示服务器，实现表示客户端。

注意接收方接收到ack后才会进入establish状态

注意两边都能主动关闭，主动的变成fin_wait1状态，被动的变成close_wait状态



注意先close——wait后last——ack，毕竟last对吧。

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

tcp/udp对整个tcp报文加上为首部校验。（udp检验和可选）（五元组）



为什么重复校验。

现在来看确实比较多余。


去除IP头中的校验和字段将提bai高路由的选择效率，du路径上的所有路由器在zhi转发处理期间不必重新计算校验dao和。第2层和第4层中的校验和都已足够强壮，从而不必考虑对第三层校验和的需求。IPv6中，TCP和UDP传输协议都需要计算校验和，而IPv4 中的UDP校验和是可选的。
就是说在OSI七层协议中，IP属3/4层。其他层的功能足以代替IP中的校验和的功能，并且IP中的校验和比更合适。

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

a = 7 / 8；指数加权，平缓波动。（当前的最近samplertt重要）



DevRTT = （1 - b） * DevRTT + b *|sampleRTT - EstimatedRTT|。

反映SampleRTT波动程度，b = 0.25.（过去的平均指数重要）



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

- 数据校验和ack检查比特错误
- 重传机制应对模糊不清的ack
- 序号解决重传导致的重复分组问题
- 使用定时器的超时重传机制解决数据丢失问题
- 数据切片和pmtu探测解决mtu问题。
- 对收到的数据报重新排序保存数据正确性
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

发送方发送SYN了， 初始序列号client， 进入SYNC_SEND；接受方接受到ack（LISTEN）发送SYNACK（一条报文）， 进入SYNC_ECVD， 自己的初始序列号server，发送方序列号 + 1；接收方发送ACK进入ESTABLISH,ack为server + 1，sqe = client + 1.发送方发送ACK也进入ESTABLISH（并且携带数据）。

1) A --> B SYN my sequence number is X 2) A <-- B ACK your sequence number is X 3) A <-- B SYN my sequence number is Y 4) A --> B ACK your sequence number is Y

RFC 793



！！！！ 提到session和timestamp

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

synack的值等于ack + 1.



服务器资源在接受到发送端的ACK后才会分配资源；

为了防止发送者发送一堆syn报文来泛洪攻击。

实现的关键在于cookie的计算，cookie的计算应该包含本次连接的状态信息，使攻击者不能伪造。

cookie的计算：

服务器收到一个SYN包，计算一个消息摘要mac。

mac = MAC(A, k);

MAC是密码学中的一个消息认证码函数，也就是满足某种安全性质的带密钥的hash函数，它能够提供cookie计算中需要的安全性。

在Linux实现中，MAC函数为SHA1。

A = SOURCE_IP || SOURCE_PORT || DST_IP || DST_PORT || t || MSSIND

k为服务器独有的密钥，实际上是一组随机数。

t为系统启动时间，每60秒加1。

MSSIND为MSS对应的索引。
————————————————
版权声明：本文为CSDN博主「zhangskd」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/zhangskd/java/article/details/16986931



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



- 注意快速恢复状态的cwnd不是一，而是ssthresh的一半。

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

#### 同时打开，同时关闭（希望能够主动回答）

同时打开：两边同时发动SYN和SYNACK，建立了一条连接。

同时进入sync_sent状态->接受到syn后发送syn和ack-》进入sync_rcvd状态->同时接受到syn和ack后进入establish状态

![截屏2020-07-09 下午5.47.14](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-09 下午5.47.14.png)了四个报文

![截屏2020-07-09 下午5.46.56](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-09 下午5.46.56.png)



#### TIME_WAIT：

主动发送方等待两个2 * MSL（报文段最大生存时间一个msl，2min），保证对方收到ACK，如果没有收到，则对方重发FIN，本地将会重发ACK)（大于ttl）



(**此处应该是客户端收到一个非法的报文段，而返回一个RST的数据报，表明拒绝此次通信，然后双方就产生异常，而不是收不到。**)，那么服务器就不能按正常步骤进入close状态。那么就会耗费服务器的资源。当网络中存在大量的timewait状态，那么服务器的压力可想而知。

（！！！！这种情况没啥问题，因为都是进入了closed状态）



如果客户端等待的时间不够长，当服务端还没有收到 `ACK` 消息时，客户端就重新与服务端建立 TCP 连接就会造成以下问题 — 服务端因为没有收到 `ACK` 消息，所以仍然认为当前连接是合法的，客户端重新发送 `SYN` 消息请求握手时会收到服务端的 `RST` 消息，连接建立的过程就会被终止。（主要防止主动关闭端重新建立连接）



经过2msl的时间足以让本次连接产生的所有报文段都从网络中消失，这样下一次新的连接中就肯定不会出现旧连接的报文段了。

https://draveness.me/whys-the-design-tcp-time-wait/

#### ttl

ttl是ip字段，每经过一个路由器减1，为零则发送icmp报文通知主机

#### 为什么需要四次挥手

主动方在进入了FIN_WAIT2状态后，保留接受功能；

这样被动方没有发送的数据发送完。发送完成之后，被动方也发送一个FIN相当于告诉主动方：自己数据已经发送完毕。对面可以关闭接受功能。



客户端主动关闭

- 客户端关闭大量连接，出现大量TIME_WAIT状态，持续消耗资(服务器主动关闭的情况)
- 如果客户端等待的时间不够长，当服务端还没有收到 `ACK` 消息时，客户端就重新与服务端建立 TCP 连接就会造成以下问题 — 服务端因为没有收到 `ACK` 消息，所以仍然认为当前连接是合法的，客户端重新发送 `SYN` 消息请求握手时会收到服务端的 `RST` 消息，连接建立的过程就会被终止。

可以设置tcp_max_tw_buckets控制最大timewait数量



RFC 793 中虽然指出了 TCP 连接需要在 `TIME_WAIT` 中等待 2 倍的 MSL，但是并没有解释清楚这里的两倍是从何而来，比较合理的解释是 — 网络中可能存在来自发起方的数据段，当这些发起方的数据段被服务端处理后又会向客户端发送响应，所以一来一回需要等待 2 倍的时间[5](https://draveness.me/whys-the-design-tcp-time-wait/#fn:5)。

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



中间点相当于丢包时候的cwnd，我们知道通常丢包后设置cwnd = 1，ssthresh = cwnd / 2。

然后慢启动阶段倍数增长，当cwnd = ssthresh 时拥塞避免，1增长一个rtt。



而图中左阶段就是进入拥塞避免阶段，区别在于此时ssthresh = 0.2cwnd。

开始三次函数增长。

右边是探测多余带宽。



- 开始时很快

- 接近Wmax时增长慢

- b = 0.2 《 0.5是为了让收敛更慢，协议更为稳定

- 相比较BIC算法的二分查找，以及每RTT查找一次，导致RTT小的更容易到达顶点

  https://blog.csdn.net/dog250/article/details/53013410

  

  图中r就代表收敛速度。
  
  ![截屏2020-07-15 上午11.13.15](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-15 上午11.13.15.png)
  
  非常显然，beta决定了整个曲线对称范围围成区域的高度，而C则控制了从起始窗口到达丢包窗口的时间。
  
  由于r与C成反比，那么C越大，则探测到最大窗口的时间越短，反之则越久。由此，我们只要控制beta和C两个参数，就能控制CUBIC算法的行为了。
  
  
  
  
  
  ![截屏2020-07-15 上午11.10.19](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-15 上午11.10.19.png)
  
  #### ![截屏2020-07-15 上午11.11.13](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-15 上午11.11.13.png)最大tcp连接
  
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

#### 流量控制缺点

接收方每次收到数据包，可以在发送确定报文的时候，同时告诉发送方自己的缓存区还剩余多少是空闲的，我们也把缓存区的剩余大小称之为接收窗口大小，用变量win来表示接收窗口的大小。

 

发送方收到之后，便会调整自己的发送速率，也就是调整自己发送窗口的大小，当发送方收到接收窗口的大小为0时，发送方就会停止发送数据，防止出现大量丢包情况的发生。



不是通过offset设置而是通过能发的窗口。



#### 心跳包

tcp中如果关闭会有fin，对方能够得到连接断开的消息。

但是

- 操作系统崩溃，不会发送fin
- 而keepalive是操作系统处理，即使进程死锁或者阻塞，依然能够正常工作。而keepalive无法。



如果TCP连接中的另一方因为停电突然断网，我们并不知道连接断开，此时发送数据失败会进行重传，由于重传包的优先级要高于keepalive的数据包，因此keepalive的数据包无法发送出去。只有在长时间的重传失败之后我们才能判断此连接断开了。

![截屏2020-07-22 下午9.35.33](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-22 下午9.35.33.png)

#### tcp和udp同一端口

3、TCP和UDP传输协议监听同一个端口后，接收数据互不影响，不冲突。因为数据接收时时根据五元组`{传输协议，源IP，目的IP，源端口，目的端口}`判断接受者的。

#### sack

https://zhuanlan.zhihu.com/p/101702312

### 带选择确认的重传

改进的方法就是 SACK（Selective Acknowledgment），简单来讲就是在快速重传的基础上，**返回最近收到的报文段的序列号范围**，这样客户端就知道，哪些数据包已经到达服务器了。

来几个简单的示例：

- case 1：第一个包丢失，剩下的 7 个包都被收到了。

当收到 7 个包的**任何一个**的时候，接收方会返回一个带 SACK 选项的 ACK，告知发送方自己收到了哪些乱序包。注：**Left Edge，Right Edge 就是这些乱序包的左右边界**。

```text
Triggering    ACK      Left Edge   Right Edge
             Segment

             5000         (lost)
             5500         5000     5500       6000
             6000         5000     5500       6500
             6500         5000     5500       7000
             7000         5000     5500       7500
             7500         5000     5500       8000
             8000         5000     5500       8500
             8500         5000     5500       9000
```

- case 2：第 2, 4, 6, 8 个数据包丢失。
- 收到第一个包时，没有乱序的情况，正常回复 ACK。
- 收到第 3, 5, 7 个包时，由于出现了乱序包，回复带 SACK 的 ACK。
- 因为这种情况下有很多碎片段，所以相应的 Block 段也有很多组，当然，因为选项字段大小限制， Block 也有上限。

```text
Triggering  ACK    First Block   2nd Block     3rd Block
          Segment            Left   Right  Left   Right  Left   Right
                             Edge   Edge   Edge   Edge   Edge   Edge

          5000       5500
          5500       (lost)
          6000       5500    6000   6500
          6500       (lost)
          7000       5500    7000   7500   6000   6500
          7500       (lost)
          8000       5500    8000   8500   7000   7500   6000   6500
          8500       (lost)
```

不过 SACK 的规范「[RFC2018](https://link.zhihu.com/?target=https%3A//tools.ietf.org/html/rfc2018)」有点坑爹，接收方可能会在提供一个 SACK 告诉发送方这些信息后，又「食言」，也就是说，接收方可能把这些（乱序的）数据包删除掉，然后再通知发送方。以下摘自「RFC2018」：

> Note that the data receiver is permitted to discard data in its queue that has not been acknowledged to the data sender, even if the data has already been reported in a SACK option. **Such discarding of SACKed packets is discouraged, but may be used if the receiver runs out of buffer space.**

最后一句是说，**当接收方缓冲区快被耗尽时**，可以采取这种措施，当然并不建议这种行为。。。

由于这个操作，发送方在收到 SACK 以后，也不能直接清空重传缓冲区里的数据，一直到接收方发送普通的，ACK 号大于其最大序列号的值的时候才能清除。另外，重传计时器也收到影响，重传计时器应该忽略 SACK 的影响，毕竟接收方把数据删了跟丢包没啥区别。



- 需要注意的是只有收到失序的分组时才会可能会发送SACK，TCP的ACK还是建立在累积确认的基础上的。也就是说如果收到的报文段与期望收到的报文段的序号相同就会发送累积的ACK，SACK只是针对失序到达的报文段的。