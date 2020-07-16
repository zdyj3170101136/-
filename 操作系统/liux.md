#### cd

cd /home    进入 '/ home' 目录

cd ..            返回上一级目录 

cd ../..         返回上两级目录 

cd               进入个人的主目录 

cd ~user1   进入个人的主目录 

cd -             返回上次所在的目录 

#### pwd

pwd显示当前绝对路径

#### ls

ls 查看目录中的文件 

ls -l 显示文件和目录的详细资料 

ls -a 列出全部文件，包含隐藏文件(!!!!)

ls -h 以人类友好形式显示文件大小

ls -R 连同子目录的内容一起列出（递归列出），等于该目录下的所有文件都会显示出来 

![截屏2020-06-21 下午8.24.28](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-06-21 下午8.24.28.png)

filemode |  硬链接数量 （自身为1， 父目录为1，子目录n，本身是目录还有.为一）｜ 拥有者 ｜ 拥有组 ｜ 文件大小（B）｜mtime（月日分秒） ｜  文件名

#### ln

软链接： ln -s slink source

硬链接： ln link source（硬链接常见所以无参数）

#### filemode

rwx（可读可写可运行）

d：目录

-：文件

c：字母行文件

l：链接

b：块文件

#### time

#### birthtime

创建时间

#### atime

上一次访问时间（ls不会有，cat会有）

atime改变则ctime一定会改变。

#### mtime

修改文件datablock的时间。

mtime的改变一定会引起ctime改变。

.当然，修改文件内容也一定会改变ctime(修改文件内容至少已经修改了inode记录上的mtime，这也是元数据)，也就是说mtime的改变一定会引起ctime的改变。

#### ctime

修改文件inode的时间。

atime改变一定引起ctime改变。



(1).atime只在文件被打开访问时才改变，若不是打开文件编辑内容(如重定向内容到文件中)，则ctime和mtime的改变不会引起atime的改变;



改变文件权限。

#### atime非连续

由于atime好像没啥用，频繁更新影响性能。因此除非两次修改atime小于一天，

atime是不会改变

#### mkdir

mkdir创建目录

-m 赋予权限

-p 绝对路径，把路径上的文件也创建好

####  touch

touch创建文件.



touch -a修改atime，-m修改mtime，



修改时间戳。touch不会修改ctime选项，因为修改atime和mtime都会修改ctime。

#### rm

rm 删除文件

-r：递归，删除目录

-i：询问是否删除

-f：强制删除，不进行询问



#### rmdir

删除空目录

#### cp

cp src dest。复制文件到一个目录

-p：文件的属性也复制

-r：递归复制

-d：如果指定-d，如果是链接文件，则复制的是链接文件，不然时文件本身

-a：prd三个选项合一

-f：目标存在时不会询问



cp -a /. /tmp（复制所有文件包括隐藏文件。直接复制文件的目录）

#### scp

scp file user@ip:port/path

在目标主机生成文件，把内容复制过去。

#### mv

Mv 移动文件

-i 询问是否覆盖

-f 强制覆盖

mv同文件系统小的覆盖其实是修改data block

设想下，如果/tmp/a/a移动到/tmp下并重命名为b，则其动作是直接向/tmp的data block中添加b的记录，如果此时正好/tmp下已有b目录，则先删除/tmp的data block中b目录对应的记录，再添加移动后的b记录。

但是现在不是重命名为b，而是覆盖/tmp/a，此时的动作按原理应该是先提示是否覆盖，如果是，则删除/tmp的data block中a对应的记录，但由于此时/tmp/a目录中还有文件，该记录无法删除(因为如果要删除了该记录，代表删除了/tmp/a整个目录，而删除整个/tmp/a目录需要删除里面所有的文件，在删除它们之前的一个动作是把/tmp/a中的所有目录和文件的inode号标记为未使用，但此刻要移动的源目录/tmp/a/a是在使用当中的)，所以提示目录非空而无法删除，这里所指的非空目录指的是/tmp/a，而非是

#### cat vi/vim区别

cat可以用来附加文件

cat -n textfile1 > textfile2 //文档内容加上行号后输出



vim是vi的升级版。不仅查看而且用于修改。



tail查看文件的最后几行，并且不停刷新，用于查看日志

https://blog.csdn.net/qq32933432/article/details/104244824

#### vim

- 命令模式（按下esc进入）：（w保存文件，q退出编辑器，wq退出且保存）
- 按i进入插入模式
- 正常模式（启动）

#### cat

cat显示整个文件内容 -n 显示行数

\>>filename<<eof或>filename<<eof

重定向输出。

https://blog.csdn.net/Design407/article/details/103735538

#### less

分页从前向后显示

#### echo

显示字符串

-e开启转意比如换行符。

```
echo -e "OK! \n" # -e 开启转义
echo "It is a test"
```

https://www.runoob.com/linux/linux-shell-echo.html

#### tac

反向查看

#### head

head -n显示前n行（默认10行）

n为正数（从前往后， n为负数，-1则只显示0行）

#### tail -n显示

输出最后n行

cat filename ｜ head -n 3000 ｜ tail -n 1000

从1000行到3000行

#### &

命令后加上&就可以后台执行。

会显示进程号

https://www.jianshu.com/p/cddb2e27a594



如果控制台挂掉照样break

#### jobs

查看多少后台执行的命令

#### fg

把后台任务掉到前台

https://blog.csdn.net/Design407/article/details/103735538

### 24、查看当前谁在使用该主机用什么命令? 查找自己所在的终端信息用什么命令?

答案：

查找自己所在的终端信息：who am i

查看当前谁在使用该主机：who

### 25、使用什么命令查看用过的命令列表?

答案：

history

#### df

![ch](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-10 下午8.08.45.png)

查看和文件系统有关的知识

#### find

find查找文件，打印出找到的所有文件

find filepath（要查找的路径名） 匹配符

-name 名字

-path 路径

![截屏2020-06-21 下午8.54.56](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-06-21 下午8.54.56.png)

![截屏2020-06-21 下午8.55.15](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-06-21 下午8.55.15.png)

通配符的书写

从上面结果可以看出，其实[]只能匹配单个字符，[0-9]表示0-9的数字，[1-20]表示[1-2]外加一个0，[1-23]表示[1-2]外加一个3，[1-22-3]表示[1-2]或[2-3]，迷惑点就是看上去是大于10的整数，其实是两个或者更多的单个数字组合体。也可以用这种方法表示多种匹配：[1-2,2-3]。

-type 文件类型

-perm 时间戳

-size 大小

find /tmp -type f -name "*.tmp" -exec rm -rf  '{}'  \;

{}会把文件名替换

![截屏2020-06-21 下午8.59.32](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-06-21 下午8.59.32.png)

使用。排除自身。

#### chmod

修改权限 -R递归修改

#### chown

改变所有这chown root：root test

#### zip

zip 源文件源文件 压缩文件

#### kill

kill向进程发送信号

SIGTERM （默认，自行关闭-15）

-9（SIGKILL）无条件终止

SIGHLD：告诉父进程自己已经被终止



实际会从管道中接受sigint，sigterm，sighup三个信号量。

- 接受三次（不区分具体哪个信号）

- 第一次优雅关闭

- 第二次提示用户关闭遇到困难

- 第三次强制关闭直接os.exit(1)。

  sigint（ctrl-c）

  sighup：当用户登陆linux会开启一个session，session中运行一个前台进程，和多个后台进程。当退出终端时，这个信号就会被发送。

  sigpipe当向一个已经关闭的进程发送信息，收到rst报文，收到sigpipe信号。

  告诉此进程，默认是结束关闭（tcp）

  https://blog.csdn.net/z_ryan/article/details/80952498
  
- SIGSEGV（访问野指针，导致的内存退出。也就是没和page table关联的信号量）

- ```
  stop（暂停状态）（T）：SIGSTOP信号。SIGCONT恢复运行。
  ```

![截屏2020-07-08 下午3.55.37](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-08 下午3.55.37.png)

#### ps

ps 显示进程信息

a 依赖终端

x 不依赖终端

u 以用户为导向

ps -aux 所有信息

![截屏2020-06-21 下午9.02.55](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-06-21 下午9.02.55.png)

User 拥有者 CPU cpu占用百分比 物理内存百分比 虚拟内存 实际物理内存

TT （依赖于哪个终端）

STAT（S：中断睡眠，可悲信号唤醒

R：在运行或运行队列

D：不可中断（kill无法杀死）

T(停止状态)

Z：僵尸进程，进程终止，但描述符美食坊）

#### top

top动态显示负载

-P cpu资源排序

-M 物理内存资源排序

第一行：总共有多少任务，正在运行的任务数，睡眠状态，线程总数  系统运行时间



菲儿行：load average是显示的是cpu的负载情况，三个数分别是1分钟，5分钟，15分钟的平均负载情况，对于单核来说cpu负载大于1的时候说明负载已经严重了，多核的时候是大于n（n为核数）。这里有点争议，应为单核的时候大于1并不意味着cpu就是已经用尽了，所以这里有的人认为负载可以达到2n的时候才认为负载比较严重。
————————————————
版权声明：本文为CSDN博主「半碗面」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/qq_34861341/java/article/details/88712641



用户进程占用cpu比例，内核占用cpu比例，空闲比例

第四行：内存区域，

第五行：物理内存使用

- PID：进行的标识号
- CPU%：该进程占用的CPU使用率
- MEM%：该进程占用的物理内存和总内存的百分比
- TIME+：该进程启动后占用的总的CPU时间
- COMMAND：进程启动的启动命令名称
- TH：线程数量

http://ypk1226.com/2019/04/17/java/cpu-usage/

VM：虚拟内存以及交换分区

neetwork：网络负载，进，出

disk：磁盘负载，读，写

各个进程的状态

![截屏2020-07-08 下午3.28.44](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-08 下午3.28.44.png)

https://www.jellythink.com/archives/421

大于0.7*cpu内核数就需要关注

https://www.jellythink.com/archives/421



linux下会显示总的物理内存和空闲内存以及用作缓冲的内存。

以及各个进程状态。

#### iostat

显示io负载（右边的微cpu负载）

%iowait：CPU等待输入输出完成时间的百分比



如果%iowait的值过高，表示硬盘存在I/O瓶颈，%idle值高，表示CPU较空闲，如果%idle值高但系统响应慢时，有可能是CPU等待分配内存，此时应加大内存容量。%idle值如果持续低于10，那么系统的CPU处理能力相对较低，表明系统中最需要解决的资源是CPU。



tps，每秒传输次数



![截屏2020-07-08 下午3.44.28](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-08 下午3.44.28.png)

#### grep

匹配某一行

‘'\>'匹配结尾\'<'匹配开头

'{m, n}匹配前面字符至少m次至多n次'

‘.’匹配一个

'*'匹配零个或多个

‘？’匹配零次或一次

‘+’一次或者多次（比较容易忘）



-i忽略大小写 -l 列出文件名 -r 递归搜索目录

```
grep -i 'hello.*world' menu.h main.c
```

该命令用于列出menu.h和main.c中包含"hello"字符串且后面带有"world"字符串的所有行，hello和world中间可以有任意多个字符。注意正则表达式的"-i"选项使得grep忽略大小写，所以还能匹配"Hello, world!"。

-w 全匹配![截屏2020-06-21 下午9.13.31](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-06-21 下午9.13.31.png)



-n 显示行好

#### grep

-A -B -C



匹配行的后n行，前n行，前后n行



-v 反向匹配

ctrl + S停止滚动

ctrl + q恢复滚动

#### lsof

查看进程打开的文件

-c 程序名

-p pid

-i 查看port，协议 ， 4 ip4。

显示 device号 size inode号 fd文件描述符![截屏2020-06-21 下午9.20.00](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-06-21 下午9.20.00.png)

#### shutdoen

shutdown -h now 立刻关系

shutdown -r 重启

showdoen -h hour:minute 几小时几秒后关机

#### netstat

netstat查看网络情况

- t显示tcp端口

- u显示udp端口

- p显示正在使用的进程id

- n不进行dns解析(直接使用ip地址

- ```javascript
  netstat -pan | grep 5623（查看端口号）
  ```

![截屏2020-07-08 下午2.52.06](/Users/jieyang/Desktop/截屏2020-07-08 下午2.52.06.png)

### 42、大文件

使用head或者tail查看部分内容。

https://www.cnblogs.com/yuzhoushenqi/p/6635974.html

#### ping

测试报文是否可达

-c 重复次数

-i 几秒重复一次

基于icmp协议（ip层（包装在内））

ping返回的ttl通常系统默认值，经过一个路由器减小一。

linux默认64，因此可以用来判断操作系统类型

https://cloud.tencent.com/developer/article/1336806



主机发送8（0）号报文，对方送回0（0）号报文

**举一个例子来描述「ping」命令的工作过程：**

**1）**假设有两个主机，主机A（192.168.0.1）和主机B（192.168.0.2），现在我们要监测主机A和主机B之间网络是否可达，那么我们在主机A上输入命令：ping 192.168.0.2；

**2）**此时，ping命令会在主机A上构建一个 ICMP的请求数据包（数据包里的内容后面再详述），然后 ICMP协议会将这个数据包以及目标IP（192.168.0.2）等信息一同交给IP层协议；

**3）**IP层协议得到这些信息后，将源地址（即本机IP）、目标地址（即目标IP：192.168.0.2）、再加上一些其它的控制信息，构建成一个IP数据包；

**4）**IP数据包构建完成后，还不够，还需要加上MAC地址，因此，还需要通过ARP映射表找出目标IP所对应的MAC地址。当拿到了目标主机的MAC地址和本机MAC后，一并交给数据链路层，组装成一个数据帧，依据以太网的介质访问规则，将它们传送出出去；

**5）**当主机B收到这个数据帧之后，会首先检查它的目标MAC地址是不是本机，如果是就接收下来处理，接收之后会检查这个数据帧，将数据帧中的IP数据包取出来，交给本机的IP层协议，然后IP层协议检查完之后，再将ICMP数据包取出来交给ICMP协议处理，当这一步也处理完成之后，就会构建一个ICMP应答数据包，回发给主机A；

**6）**在一定的时间内，如果主机A收到了应答包，则说明它与主机B之间网络可达，如果没有收到，则说明网络不可达。除了监测是否可达以外，还可以利用应答时间和发起时间之间的差值，计算出数据包的延迟耗时。

https://zhuanlan.zhihu.com/p/45110873

#### traceroute

**1）**Traceroute会设置特殊的TTL值，来追踪源主机和目标主机之间的路由数。首先它给目标主机发送一个 TTL=1 的UDP数据包，那么这个数据包一旦在路上遇到一个路由器，TTL就变成了0（TTL规则是每经过一个路由器都会减1），因为TTL=0了，所以路由器就会把这个数据包丢掉，然后产生一个错误类型（超时）的ICMP数据包回发给源主机，也就是差错包。这个时候源主机就拿到了第一个路由节点的IP和相关信息了；

**2）**接着，源主机再给目标主机发一个 TTL=2 的UDP数据包，依旧上述流程走一遍，就知道第二个路由节点的IP和耗时情况等信息了；

**3）**如此反复进行，Traceroute就可以拿到从主机A到主机B之间所有路由器的信息了。

但是有个问题是，如果数据包到达了目标主机的话，即使目标主机接收到TTL值为1的IP数据包，它也是不会丢弃该数据包的，也不会产生一份超时的ICMP回发数据包的，因为数据包已经达到了目的地嘛。那我们应该怎么认定数据包是否达到了目标主机呢？

Traceroute的方法是在源主机发送UDP数据包给目标主机的时候，会设置一个不可能达到的目标端口号（例如大于30000的端口号），那么当这个数据包真的到达目标主机的时候，目标主机发现没有对应的端口号，因此会产生一份“端口不可达”的错误ICMP报文（3）返回给源主机。



icmp类型8（0）：回显请求报文

icmp类型0（0）：回显响应报文

https://zhuanlan.zhihu.com/p/36811342



icmp（3）（11）：发现ttl过期，返回自己的名字和ip地址（traceroute）

icmp（3）（3）：端口不可达（发送一个不存在端口）

   netstat命令：用来打印Linux中网络系统的状态信息，可让你得知整个Linux系统的网络情况。

   这里就拿IPV4的举例，/proc/net/目录下就有当前tcp udp 的连接状态和 基本信息，netstat就是打开这个目录下的tcp udp 

    然后解析出来，就是看主机是大小端，然后16进制转为10进制 就哦了



    ps命令：用于报告当前系统的进程状态。

     是根据proc/PID/stat status cmdline 三个文件来解析，我解析的是  进程名  ID 运行总时间 进程路径 高峰内存（peak） 线程数



stat：进程名  ID 启动时间 线程数

status 有关内存的

cmdline：进程路径，没有就是进程名。

这里主要说一下运行时间，我是声明一个指针数组char *msg[50]用” “切割读取到的字符串，运行（TIME）就等于

atoi(msg[13])+atoi(msg[14]),计时间隔导致的下一个 SIGALRM 发送进程的时延+启动时间。

————————————————
版权声明：本文为CSDN博主「jinwoyunni」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/lyw13522476337/java/article/details/79553360

#### fork

 在语句fpid=fork()之前，只有一个进程在执行这段代码，但在这条语句之后，就变成两个进程在执行了，这两个进程的几乎完全相同，将要执行的下一条语句都是if(fpid<0)……
    为什么两个进程的fpid不同呢，这与fork函数的特性有关。fork调用的一个奇妙之处就是它仅仅被调用一次，却能够返回两次，它可能有三种不同的返回值：
    1）在父进程中，fork返回新创建子进程的进程ID；
    2）在子进程中，fork返回0；
    3）如果出现错误，fork返回一个负值；
————————————————
版权声明：本文为CSDN博主「jason314」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/jason314/java/article/details/5640969



  还有人可能疑惑为什么不是从#include处开始复制代码的，这是因为fork是把进程当前的情况拷贝一份，执行fork时，进程已经执行完了int count=0;fork只拷贝下一个要执行的代码到新的进程。



exev找到可执行文件开始执行。



件，用它来取代当前进程的内容，并且这个取代是不可逆的，即被替换掉的内容不再保存，当可执行文件结束，整个进程也随之僵死。因为当前进程的代码段，数据段和堆栈等都已经被新的内容取代，所以exec函数族的函数执行成功后不会返回，失败是返回-1。可执行文件既可以是二进制文件，也可以是可执行的脚本文件，两者在加载时略有差别，这里主要分析二进制文件的运行。



系统中断，读入可执行文件检查权限



简短的说，整个在shell中键入./test执行应用程序的过程为：当前shell进程fork出一个子进程(子shell)，子进程使用execve来脱离和父进程的关系，加载test文件(ELF格式)到内存中。如果test使用了动态链接库，就需要加载动态链接器(或者叫程序解释器)，进一步加载test使用到的动态链接库到内存，并重定位以供test调用。***从test的入口地址开始执行test。

但是现代的动态链接器因为性能等原因都采用了延迟加载和延迟解析技术，延迟加载是动态连接库在需要的时候才被加载到内存空间中(通过页面异常机制)，延迟解析是指到动态链接库(以加载)中的函数被调用的时候，才会去把这个函数的起始地址解析出来，供调用者使用。动态链接器的实现相当的复杂，为了性能等原因，对堆栈的直接操作被大量使用。



https://os.51cto.com/art/201808/580647.htm



