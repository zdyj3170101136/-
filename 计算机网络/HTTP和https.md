#### 五层模型

- 应用层：直接为应用程序提供服务（http，dns）报文
- 运输层：负责向两个主机进程之间的通信提供服务（tcp）报文段
  - 复用：把多个上层进程服务传递给下层
  - 把收到的信息交付给应用层
- 网络层：为不同主机之间提供服务。ip，icmp）数据包
- 数据链路层：为相领的两个节点之间传输数据 frame（战）
- 物理层，传输bit （）

## HTTP

http使用tcp连接。

端口号80，https端口号443.

分为请求行，首部行，空行，请求数据组成。

使用ASCII文本编写。

#### 请求行![截屏2020-07-09 下午1.11.31](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-09 下午1.11.31.png)

HTTP报文的第一行:请求行

方法字段｜URL字段｜HTTP版本

接下来是首部行

然后有一行空格。

随后跟随（entity body）内容。

#### 首部行

常用的一些header：

Connection：keepalive，表示持久连接。

Host：表示目标主机名（同一个IP地址可能会映射多个域名https://blog.csdn.net/codejas/article/details/82844032?utm_medium=distribute.pc_relevant_t0.none-task-blog-BlogCommendFromMachineLearnPai2-1.compare&depth_1-utm_source=distribute.pc_relevant_t0.none-task-blog-BlogCommendFromMachineLearnPai2-1.compare

http1.1所实现的）

User-Agent：表示用户机器类型

Accpet：表示可以接受的饭回值的类型

Accept-language：表示可以接受的语言

Accept-Encoding：表示可以接受的加密算法

Referer：表示来源。（可以限制访问来源，当从地址栏输入为空（能让浏览器直接访问））

（https://blog.csdn.net/shenqueying/article/details/79426884）

ETAG：服务器生成的用于缓存资源的值。它的原理是这样的，当浏览器请求服务器的某项资源(A)时, 服务器根据A算出一个哈希值(3f80f-1b6-3e1cb03b)并通过 ETag 返回给浏览器，浏览器把"3f80f-1b6-3e1cb03b" 和 A 同时缓存在本地，当下次再次向服务器请求A时，会通过类似 If-None-Match: "3f80f-1b6-3e1cb03b" 的请求头把ETag发送给服务器，服务器再次计算A的哈希值并和浏览器发送的值做比较，如果发现A发生了变化就把A返回给浏览器(200)，如果发现A没有变化就给浏览器返回一个304未修改。这样通过控制浏览器端的缓存，可以节省服务器的带宽，因为服务器不需要每次都把全量数据返回给客户端。

https://juejin.im/post/5b9e56acf265da0af93af0f2

Upgrade-Insecure-requests：表示自己可以将接下来的请求升级成https

（在https的页面加载http的资源浏览器就会报错，为了不用用户把整个网站http换成https；如果浏览器使用这个头；对面服务器返回确认，说明有这个资源。

就会自动从http升级为https）

https://www.cnblogs.com/panlq/articles/9615994.html

![截屏2020-06-17 下午10.23.48](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-06-17 下午10.23.48.png)


- 当用户点击网页链接会发送Referee字段

IF-NONE-MATCH：对于GET和HEAD请求，当服务器没有任何资源的ETAG与这个首部列出的相同，才会返回资源。不然返回304未修改。

#### 响应首部（来源自github）

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

Cache-Control：控制浏览器和中间服务器的缓存策略。private表示拥护浏览器可以缓存，cdn不能缓存。public表示都能缓存。max-age：表示最大缓存时间

[https://developers.google.com/web/fundamentals/performance/optimizing-content-efficiency/http-caching?hl=zh-cn#%E2%80%9Cmax-age%E2%80%9D](https://developers.google.com/web/fundamentals/performance/optimizing-content-efficiency/http-caching?hl=zh-cn#"max-age")

Content-Encoding：内容加密类型

CONTENT-SECURITY-POLICY：防止Xss攻击（往页面里插入script代码）。

浏览器将不会执行不让访问的脚本，从不信任的地方家在图片等等。（十分复杂）

https://www.jianshu.com/p/74ea9f0860d2

X-Frame-Options：不能让这个网页y以iframe的形式（框架）（隐藏框且透明度为0）的形显现。防止了劫持站点攻击。

X-XSS-Protection：（1；mode = block）当检测到跨站脚本攻击，将停止加载页面。（为什么还需要csp？因为csp可能太详细很难实现）https://www.anquanke.com/post/id/85422

https://www.cnblogs.com/fundebug/p/use-http-headers-to-protect-web.html

Expect-CT：

如果所使用的证书不支持ct机制，将报告给制定的url。

（防止证书机构break）

https://zhuanlan.zhihu.com/p/59584717

```
X-Content-Type-Options: nosniff：一定遵守content-type的规定。
```

Strict-Transport-Security：

为了预防中间人攻击，一旦你使用这个访问相应的网站，接下来的访问浏览器都会自动使用HTTPS。（防止第一次请求使用https，然后之后的请求使用不安全的http被重定向到黑客的网站）

https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Strict-Transport-Security

#### 跟缓存有关的

User-Agent和Vary。

Cache-control

etag

#### 跟控制请求有关的

Accept控制请求内容

Accerpt-encoding控制加密搁置。

Referer显示请求来源

host指明主机

tranfer-encoding：chunked

X-request-id：用于标记请求，出现文件快速定位，

https://blog.csdn.net/yongwan5637/article/details/90898556

#### 跟http有关的

Expect-Ct

Upgrade-Insecure-requests

Strict-Transport-Security

#### xss

CONTENT-SECURITY-POLICY（只能执行的脚本和网页）（xss攻击）

X-XSS-Protection（检测到攻击就不会显现网页）（xss攻击）

X-Frame-Options（不能让iframe框架显示）（劫持站点攻击）

X-Content-Type-Options: nosniff（阻止和contnt-type不一样的style和script）

#### Cookie

响应体的Set-Cookie为服务器发送的一段数据。

而浏览器会自动访问页面时填充cookie请求体。



![截屏2020-07-09 下午1.58.11](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-09 下午1.58.11.png)

expires：指定过期时间（到期后自动删除）。

SECURE:表明cookie只能通过https传递。

httponly：为了防止xss攻击，javascript的通过JavaScript的 [`Document.cookie`](https://developer.mozilla.org/zh-CN/docs/Web/API/Document/cookie) API无法访问带有 `HttpOnly` 标记的Cookie，

Path：表示了cookie的作用的url。（匹配子路径）

Same-site：

strict：

只有当前网页和发出请求的网页一样时才会带上cookie。（防止csrf攻击（伪造带有正确cookie的http请求））

lax：能让get请求发送。

https://www.ruanyifeng.com/blog/2019/09/cookie-samesite.html

https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Cookies





为什么__Host-user_session_same_site和session是一样的。

其他一样：但是samesite为strict（这样用来识别csrf攻击）(检查是否来自github的请求)。



第三个gh_session表明这是一个会话cookie，仅用于这次会话，当前页面关闭后就删除。（这个gh_session用于登陆，仅仅有账号密码但是没有这个也无法登陆）

https://ithelp.ithome.com.tw/articles/10205637?sc=iThelpR



这里的_gh_session和session_id都需要。但是没有过期时间，则关闭浏览器自动挂掉。

则实现了用户只关闭浏览器忘记退出时ok。

#### 安全有关的

secure(https)

Httpnly(防止xss攻击)

same-site（防止跨站请求伪造）



cookie数据存放在客户的浏览器（客户端）上，session数据放在服务器上，但是服务端的session的实现对客户端的cookie有依赖关系的；

cookie不是很安全，别人可以分析存放在本地的COOKIE并进行COOKIE欺骗，考虑到安全应当使用session；

session会在一定时间内保存在服务器上。当访问增多，会比较占用你服务器的性能。考虑到减轻服务器性能方面，应当使用COOKIE；

单个cookie在客户端的限制是3K，就是说一个站点在客户端存放的COOKIE不能超过3K；



简单来讲，cookie存所有有关状态的值；

session只存一个id。



作者：Fysddsw_lc
链接：https://juejin.im/post/5aa783b76fb9a028d663d70a
来源：掘金
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。


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



CLIENTHELLO:客户发送客户端支持的SSL版本，对称算法，mac算法，公钥算法，以及一个不重数。

CERTIFICATE服务器返回自己的证书

SERVERHELLO选择的加密算法和不重数random2和MAC算法。（会返回一个sessionid用于密钥复用）

CLIENTKEYEXCHANGE客户通过证书获取了服务器的公钥，生成第三个随机数，通过服务器端公钥非对称加密生成主密钥。

CHANGECIPHERSPEC：表示之后的信息都是加密的

四，服务器用私钥解密，得到主密匙。此时双方都有了三个随机数

五，然后服务器和各户端各自从主密钥，通过三个随机数计算出四个密钥。



FINISHED客户端发送所有握手报文的一个mac（第一条加密，证明自己拥有密钥，因为服务端可以对数据进行校验，判断对面是否有）

FINISHED服务器发送所有握手报文的一个mac（）能解出来说明server是之前证书的拥有者。（只有有私钥才能得到主秘钥）

这里因为之前都是明文通信。。。

![截屏2020-07-09 下午7.08.12](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-09 下午7.08.12.png)

https://blog.csdn.net/tterminator/article/details/50675540

由图可知，tls1.2两次RTT。



#### tls1.3

之前的tls1.2





最后两个是为了防止强制使用第一第二步安全性较弱的算法。





服务器到客户端的会话加密，服务器到客户端的会话mac加密。

客户端到服务器的会话加密，客户端到服务器的mac加密。

#### Session复用

HELLo携带SESSION ID，如果服务器能够识别，就不用重新握手。使用保存的主密钥产生所有的加密密钥。

#### mac

MD5：

把任意的数据都过运算生成128位的值。

不可逆，因为这个过程中损失了很多信息。



而mac通过报文m和秘钥s通过md5计算得到报文鉴别码。（注意此处密钥不是用来被加密的）

#### 为什么三个随机数

多个随机数种子生成秘钥不容易，防止暴力破解。

https://www.cnblogs.com/fengf233/p/11775415.html

连接重放攻击：A向b发送的所有报文如果被c截取到，那么他将会重新向b传输一次。而b不会察觉。



因此我们加入不重数防止连接重放攻击。

不同的ssl会话连接，使用不同的不重数。因此也使用不同的秘钥。

#### 数字签名

数字签名用来确认所有权。

发送者用私钥加密数据，并且只有公钥被加密的数据再次运算 才能够得到原来的数据。

公钥谁都可以知道，推导不出来私钥。



非对称：加密和解密不是同一个秘钥，比较慢。

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



类型字段用于是否关闭。



因此 mac以数据和ssl序号（实际没有）。

- 计算得到，防止了中间人攻击。
- 维护完整性，数据没有收到篡改

由于因特网检验和比较劣质，很容易找到相同的内容。

#### 关闭

关闭tcp连接不仅仅通过FIN字段。

避免被其他人主动发送导致没有收到完整的信息。



而是通过SSL类型字段。

尽管类型字段明文，但是通过MAC加密。

因此只有类型字段也标记关闭，才会真的关闭。





#### GET, POST

GET用于获取数据；post用于将数据发送到服务器以创建或者更新

- 从REST角度，GET是幂等，而POST不是
- 从请求参数，GET在URL之后通过urlencode编码，POST在请求体内多种编码方式。
- GET受限于url长度（不同浏览器不一样，如果是服务器当然没区别），post没有
- get常用于获取资源，post常用于更新（但是也可以获取数据）
- 缓存：get请求可以被缓存，post不会
- get请求会在浏览器里时记录，post不会
- post比get安全（数据会被历史记录缓存，其他人可以查看到）
- get请求可以加书签，post不会。（因为书签只记录url，并且只使用get发送请求）
- get会产生一个tcp数据包；而post两个：先发首部，再发数据。（通过客户端发送100continue确认服务器端能否支持大数据）

url编码，防止当value中出现相同的&。

https://www.w3schools.com/tags/ref_httpmethods.asp

https://www.sohu.com/a/353926375_468635



#### status

100 continue：可以处理请求



200 oK：请求成功处理



301:永久性转移

302：暂时性转移

304:已缓存（重定向）



400:Bad request。请求语法问题

401:unauthorized。身份没验证，请求未满足

403:拒绝请求（客户端错误）

404:内容不存在



500:服务器内部错误

503:暂时不可用。

#### 请求并发

对同一个host最多6个请求。



#### HTTP1.1和1.0区别

- 长连接，connection：keep-alive（同一时刻一个http请求）
- 带宽：能发送请求是先发header，如果对方能够处理返回100，不能返回401；随后再发送body部分。
- host首部，以前认为每台服务器绑定唯一ip，因此url不包括主机名。；但现在一台服务器多个虚拟主机，共享ip。因此加了HOST。如果没有HOST会报一个400错误。
- 分块传输的支持，transer-encoding：chunked
- 缓存，之前的话使用if-modified-since来做为缓存标砖；现在主要增加了etag来判断和if -math
- 新状态，410表示服务器资源被永久性删除

#### http1.1和http2.0的区别

- http2.0可以多路复用，（同一时刻多个http请求）。
- （二进制横表明自属于那个流）

（由于tcp慢启动，所以对于短时性的http连接变得十分低小）

- 头部数据压缩。使用算法对header部分进行压缩。

（静态字典，直接对每个字段用对应的数字表示）

- 服务器推送，服务器可以直接把数据推送给客户端



https://blog.csdn.net/ailunlee/article/details/97831912

- http2通过把http的数据分为frame 

https://blog.csdn.net/striveb/article/details/84230923?utm_medium=distribute.pc_relevant_t0.none-task-blog-BlogCommendFromMachineLearnPai2-1.nonecase&depth_1-utm_source=distribute.pc_relevant_t0.none-task-blog-BlogCommendFromMachineLearnPai2-1.nonecase