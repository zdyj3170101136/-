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

ls -a 列出全部文件，包含隐藏文件

ls -h 以人类友好形式显示文件大小

ls -R 连同子目录的内容一起列出（递归列出），等于该目录下的所有文件都会显示出来 

![截屏2020-06-21 下午8.24.28](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-06-21 下午8.24.28.png)

filemode |  硬链接数量 （自身为1， 父目录为1，子目录n，本身是目录还有.为一）｜ 拥有者 ｜ 拥有组 ｜ 文件大小（B）｜mtime（月日分秒） ｜  文件名

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

修改文件datablock内容肯定要改变ctime。

#### ctime

修改文件属性

改变文件权限。

#### atime非连续

由于atime好像没啥用，频繁更新影响性能。因此除非两次修改atime小于一天，

atime是不会改变

#### mkdir

mkdir创建目录

-m 赋予权限

-p 绝对路径，把路径上的文件也创建好

####  touch

touch创建文件

修改时间戳。touch不会修改ctime选项，因为修改atime和mtime都会修改ctime。

#### rm

rm 删除文件

-r：递归，删除目录

-i：询问是否删除

-f：强制删除，不进行询问

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

#### cat

cat显示文件内容 -n 显示行数

\>>filename<<eof或>filename<<eof

重定向输出。

#### tac

反向查看

#### head

head -n显示前n行（默认10行）

n为正数（从前往后， n为负数，-1则只显示0行）

#### tail -n显示

输出最后n行

cat filename ｜ head -n 3000 ｜ tail -n 1000

从1000行到3000行

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

SIGTERM （默认，自行关闭）

-9（SIGKILL）无条件终止

SIGHLD：告诉父进程自己已经被终止

#### ps

ps 显示进程信息

a 依赖终端

x 不依赖终端

u 以用户为导向

ps aux 所有信息

![截屏2020-06-21 下午9.02.55](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-06-21 下午9.02.55.png)

User 拥有者 CPU cpu占用百分比 物理内存百分比 虚拟内存 实际物理内存

TT （依赖于哪个终端）

STAT（S：中断睡眠，可悲信号唤醒

R：在运行或运行队列

D：不可中断（kill无法杀死）

Z：僵尸进程，进程终止，但描述符美食坊）

#### top

top动态显示进程信息

-P cpu资源排序

-M 物理内存资源排序

#### grep

匹配某一行

‘'\>'匹配结尾\'<'匹配开头

'{m, n}匹配前面字符至少m次至多n次'

‘.’匹配一个

'*'匹配零个或多个

‘？’匹配零次或一次

‘+’一次或者多次



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