## HTTP

http使用tcp连接

分为请求行，首部行，空行，请求数据组成。

使用ASCII文本编写。

#### 请求行

HTTP报文的第一行:请求行

方法字段｜URL字段｜HTTP版本

接下来是首部行

然后有一行空格。

随后跟随内容。

#### 首部行

常用的一些header：

Connection表示持久连接。

Host表示目标主机名

User-Agent：表示用户机器类型

Accpet：表示可以接受的饭回值的类型

Accept-language：表示可以接受的语言

Accept-Encoding：表示可以接受的加密算法

Referer：表示来源

ETAG：服务器生成的用于缓存资源的值。它的原理是这样的，当浏览器请求服务器的某项资源(A)时, 服务器根据A算出一个哈希值(3f80f-1b6-3e1cb03b)并通过 ETag 返回给浏览器，浏览器把"3f80f-1b6-3e1cb03b" 和 A 同时缓存在本地，当下次再次向服务器请求A时，会通过类似 If-None-Match: "3f80f-1b6-3e1cb03b" 的请求头把ETag发送给服务器，服务器再次计算A的哈希值并和浏览器发送的值做比较，如果发现A发生了变化就把A返回给浏览器(200)，如果发现A没有变化就给浏览器返回一个304未修改。这样通过控制浏览器端的缓存，可以节省服务器的带宽，因为服务器不需要每次都把全量数据返回给客户端。

Upgrade-Insecure-requests：表示自己可以将接下来的请求升级成https

https://blog.csdn.net/hugejihu9/article/details/84939404

![截屏2020-06-17 下午10.23.48](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-06-17 下午10.23.48.png)
作者：多芒小丸子
链接：https://juejin.im/post/5b9e56acf265da0af93af0f2
来源：掘金
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

- 当用户点击网页链接会发送Referee字段

IF-NONE-MATCH：对于GET和HEAD请求，当服务器没有任何资源的ETAG与这个首部列出的相同，才会返回资源。不然返回304未修改。

#### 响应首部

![截屏2020-06-17 下午10.23.36](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-06-17 下午10.23.36.png)

HTTP协议好 ｜ STATUS ｜ 状态信息

DATE：这个对象被插入响应报文发送的时间。

CONTENTT-TYPE：表示是html文本

Server：服务器名字

Etag：资源唯一标识符

Set-Cookie：

Transfer-Encoding：Chunked。对于不好计算大小的内容，我们要想准确获取长度，就得分块。Chunked表示每个块用\r\n分割，并且每个块包含长度值。

最后一个分块长度为0，代表内容结束。

https://imququ.com/post/transfer-encoding-header-in-http.html

Vary：表示除了COntent的不同类型对资源做筛选之外的首部。为了防止缓存服务器对非常规首部比如User-agent返回错误的值

https://imququ.com/post/vary-header-in-http.html

Cache-Control：控制浏览器和中间服务器的缓存策略。private表示拥护浏览器可以缓存，cdn不能缓存。max-age：表示最大缓存时间

[https://developers.google.com/web/fundamentals/performance/optimizing-content-efficiency/http-caching?hl=zh-cn#%E2%80%9Cmax-age%E2%80%9D](https://developers.google.com/web/fundamentals/performance/optimizing-content-efficiency/http-caching?hl=zh-cn#"max-age")

Content-Encoding：内容加密类型

CONTENT-SECURITY-POLICY：防止Xss攻击，浏览器将不会执行不让访问的脚本，从不信任的地方家在图片等等。

https://www.jianshu.com/p/74ea9f0860d2

X-Frame-Options：不能让这个网页y以iframe的形式（框架）的形显现。防止了劫持站点攻击。

https://www.cnblogs.com/fundebug/p/use-http-headers-to-protect-web.html

Expect-CT：如果所使用的证书不支持ct机制，将报告给制定的url。

https://zhuanlan.zhihu.com/p/59584717

Strict-Transport-Security：为了预防中间人攻击，一旦你使用这个访问相应的网站，接下来的访问浏览器都会自动使用HTTPS

https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Strict-Transport-Security

#### Cookie

响应体的Set-Cookie字段识别了用户，保存了用户状态；

而请求体的Cookie字段会携带此字段。



session：服务器存储所有人的id用于区别用户。存在Cookie里头。



token:服务器以id计算出一个token，然后传给浏览器；之后服务器再次收到后检察这个值是否由自己生成，如果是由自己生成，就可以直接获得用户的id。也是存在coookie里头一般。



#### 持久连接

同一个tcp连接传输多个请求。

多路复用机制。


#### DNS

dns通常运行在udp上，端口号53。

用于将域名转换为ip地址。

在http发送请求报文前，浏览器会抽取目标主机名，发送给dns客户端，dns客户端发送给服务器，找到对应域名的ip地址。

- 主机别名，一个主机名映射多个ip地址
- 负载分配：对同一个主机名分配到最近的ip，或者任务轻的ip。
- 缓存dns，减轻服务器压力





- 根dns服务器
- com，org，edu dns服务器
- 权威dns服务器，具有公共可访问主机的每个机构必须提供的服务器。
- 本地dns服务器，用于缓存其他服务器，的内容

用户-》本地-〉根；本地-》域服务器；本地-〉权威dns服务器。



#### 不重数



TCP握手



一，客户发送客户端支持的SSL版本，对称算法，mac算法，公钥算法，以及一个不重数。（SSL HELLO）

二，服务器返回自己的证书，选择的加密算法和不重数和选择的算法。

三，客户通过证书获取了服务器的公钥，生成第三个随机数，作为一个主密钥，通过公钥加密

四，服务器用私钥解密，得到主密匙。

五，然后服务器和各户端各自从主密钥，通过随机数计算出四个密钥。

六，客户端发送所有握手报文的一个mac（第一条加密）

七，服务器发送所有握手报文的一个mac（）



最后两个是为了防止强制使用第一第二步安全性较弱的算法。



服务器到客户端的会话加密，服务器到客户端的会话mac加密。

客户端到服务器的会话加密，客户端到服务器的mac加密。

#### Session复用

HELLo携带SESSION ID，如果服务器能够识别，就不用重新握手。使用保存的主密钥产生所有的加密密钥

#### 为什么三个随机数

多个随机数种子生成秘钥不容易，防止暴力破解。



连接重放攻击：A向b发送的所有报文如果被c截取到，那么他将会重新向b传输一次。而b不会察觉。



因此我们加入不重数防止连接重放攻击。

不同的ssl会话连接，使用不同的不重数。因此也使用不同的秘钥。



#### 数字签名

数字签名用来确认所有权。

发送者用私钥加密数据，并且只有公钥被加密的数据再次运算 才能够得到原来的数据。

公钥谁都可以知道，推导不出来私钥。



对称密钥：使用同一个密钥加密和解密，更快。

#### 证书

证书用来证明一个公钥属于某个实体。

CA用于颁发证书。CA会生成一个将对应身份和公钥以及ip绑定的证书，再用自己的私钥加密。而用户可以用存在浏览器中的公钥（默认保存），鉴别证书的真实性。

#### CT

为了防止证书颁发机构作乱。或者被劫持。

所有的证书都要登记在一个透明网站，任何人都能够看到。

因此EXPECT_CT将会对任何不在CT中的证书报警。

#### SSL记录

类型 ｜ 版本 ｜ 长度 ｜ 数据 ｜ mac

前三个版本不加密，更上层的tcp也不加密。

考虑中间人攻击：其会颠倒ssl报文段的顺序，调整tcp报文段序号。

导致接受端收到次序不正确的字节流。

因此 mac以数据和ssl序号（实际没有）。

- 计算得到，防止了中间人攻击。
- 维护完整性，数据没有收到篡改

由于因特网检验和比较劣质，很容易找到相同的内容。

#### 关闭

关闭tcp连接不仅仅通过FIN字段。

而是通过SSL类型字段。

尽管类型字段明文，但是通过MAC加密。

因此只有类型字段也标记关闭，才会真的关闭。

#### GET, POST

- 从REST角度，GET是幂等，而POST不是
- 从请求参数，GET在URL之后通过urlencode编码，POST在请求体内多种编码方式。
- GET受限于url长度（不同浏览器不一样，如果是服务器当然没区别），post没有
- get用于获取资源，post用于更新
- get会产生一个tcp数据包；而post两个：先发首部，再发数据。（通过客户端发送100continue确认服务器端能否支持大数据）

url编码，防止当value中出现相同的&。

https://www.sohu.com/a/353926375_468635



PUT 上传资源

Delete 更新

#### status

200 oK：请求成功处理



301:永久性转移

302：暂时性转移

304:已缓存（重定向）



400:Bad request。请求语法问题

403:拒绝请求（客户端错误）

404:内容不存在



500:服务器内部错误

503:暂时不可用。



#### HTTP1.1



- 长连接，connection：keep-alive

- host首部，以前认为每台服务器绑定唯一ip，因此url不包括主机名。；但现在一台服务器多个虚拟主机，共享ip。因此加了HOST。如果没有HOST会报一个400错误。

