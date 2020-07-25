#### ext4文件系统
EXT4全称是Fourth extended filesystem，是在EXT3的基础上新增了高级功能，属于第四代的扩展文件系统。

#### inode

硬件以512kb处理数据，文件系统以4k大小处理数据。



，由于默认blocksize=4KB，所以每4个block就分配一个inode号。



在文件系统上的术语中，索引节点称为inode。在inode中存储了inode号（注，inode中并未存储inode num，但为了方便理解，这里暂时认为它存储了inode号）、文件类型、权限、文件所有者、大小、时间戳等元数据信息，最重要的是还存储了指向属于该文件block的指针，这样读取inode就可以找到属于该文件的block，进而读取这些block并获得该文件的数据。

(256 KB)

#### 文件系统结构

![截屏2020-07-10 下午6.42.20](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-10 下午6.42.20.png)

- bitmap：一个位表示块是否被使用

bmap的优化针对的是写优化，因为只有写才需要找到空闲block并分配空闲block。

- imap：标记inode号是否被使用
- inode table：为了避免一个inode占用一个block（将多个inode存在里头）

如果文件系统太大，bitmap也会过大，因此有了block group的概念。



- seperblock，存储每个块组的元信息（么文件系统怎么知道分了多少个块组呢？每个块组又有多少block多少inode号等等信息呢？还有，文件系统本身的属性信息如各种时间戳、block总数量和空闲数量、inode总数量和空闲数量、当前文件系统是否正常。只在0，1和3，5，7的幂次方）（为了防止超级块损坏）（文件系统元信息）

- GDT（虽然每个块组都需要块组描述符来记录块组的信息和属性元数据，但是不是每个块组中都存放了块组描述符。ext文件系统的存储方式是：将它们组成一个GDT，并将该GDT存放于某些块组中，存放GDT的块组和存放superblock和备份superblock的块相同，也就是说它们是同时出现在某一个块组中的。读取时也总是读取Group0中的块组描述符表信息。）



- datablock存放文件系统实际数据

  

- 对于常规文件，文件的数据正常存储在数据块中。

- 对于目录，该目录下的所有文件和一级子目录的目录名存储在数据块中。

  目录的子文件名的inode号存储在其datablock中。

  - 对于符号链接，如果目标路径名较短则直接保存在inode中以便更快地查找，如果目标路径名较长则分配一个数据块来保存。
  - 设备文件、FIFO和socket等特殊文件没有数据块，设备文件的主设备号和次设备号保存在inode中。
  - ![截屏2020-07-10 下午6.53.41](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-10 下午6.53.41.png)



#### 创建即删除的临时文件

目录data block的记录已经删除，但是该记录对应的inode结构仍然存在于inode table中。这种inode称为孤儿inode（orphan inode）：存在于inode table中，但却无法再索引到它。因为目录中已经没有该inode对应的文件记录了，所以其它进程将无法找到该inode，也就无法根据该inode找到该文件之前所占用的data block，这正是创建便删除所实现的真正临时文件，该临时文件只有当前进程和子进程才能访问。

#### 硬链接

每创建一个文件的硬链接，实质上是多一个指向该inode记录的inode指针，并且硬链接数加1。

删除文件的实质是删除该文件所在目录data block中的对应的inode行，所以也是减少硬链接次数，由于block指针是存储在inode中的，所以不是真的删除数据，如果仍有其他inode号链接到该inode，那么该文件的block指针仍然是可用的。当硬链接次数为1时再删除文件就是真的删除文件了，此时inode记录中block指针也将被删除。



硬链接只能对文件创建，无法对目录创建硬链接。之所以无法对目录创建硬链接，是因为文件系统已经把每个目录的硬链接创建好了，它们就是相对路径中的"."和".."，分别标识当前目录的硬链接和上级目录的硬链接。每一个目录中都会包含这两个硬链接，它包含了两个信息：(1)一个没有子目录的目录文件的硬链接数是2，其一是目录本身，即该目录datablock中的"."，其二是其父目录datablock中该目录的记录，这两者都指向同一个inode号；(2)一个包含子目录的目录文件，其硬链接数是2+子目录数，因为每个子目录都关联一个父目录的硬链接".."。很多人在计算目录的硬链接数时认为由于包含了"."和".."，所以空目录的硬链接数是2，这是错误的，因为".."不是本目录的硬链接。另外，还有一个特殊的目录应该纳入考虑，即"/"目录，它自身是一个文件系统的入口，是自引用(下文中会解释自引用)的，所以"/"目录下的"."和".."的inode号相同，它自身不占用硬链接，因为其datablock中只记录inode号相同的"."和".."，不再像其他目录一样还记录一个名为"/"的目录，所以"/"的硬链接数也是2+子目录数，但这个2是"."和".."的结果。



#### 软链接

软链接之所以也被称为特殊文件的原因是：它一般情况下不占用data block，仅仅通过它对应的inode记录就能将其信息描述完成；符号链接的大小是其指向目标路径占用的字符个数，例如某个符号链接的指向方式为"rmt --> ../sbin/rmt"，则其文件大小为11字节；只有当符号链接指向的目标的路径名较长(60个字节)时文件系统才会划分一个data block给它；它的权限如何也不重要，因它只是一个指向原文件的"工具"，最终决定是否能读写执行的权限由原文件决定，所以很可能ls -l查看到的符号链接权限为777。

#### 间接寻址

第13个指针i_block[12]是一级间接寻址指针，它指向一个仍然存储了指针的block即i_block[12] --> Pointerblock --> datablock。

第14个指针i_block[13]是二级间接寻址指针，它指向一个仍然存储了指针的block，但是这个block中的指针还继续指向其他存储指针的block，即i_block[13] --> Pointerblock1 --> PointerBlock2 --> datablock。

第15个指针i_block[14]是三级间接寻址指针，它指向一个任然存储了指针的block，这个指针block下还有两次指针指向。即i_block[13] --> Pointerblock1 --> PointerBlock2 --> PointerBlock3 --> datablock。

其中由于每个指针大小为4字节，所以每个指针block能存放的指针数量为BlockSize/4byte。例如blocksize为4KB，那么一个Block可以存放4096/4=1024个指针。

![截屏2020-07-10 下午7.01.22](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-10 下午7.01.22.png)

#### 查找文件

- **找到根文件系统的块组描述符表所在的blocks，读取GDT(已在内存中)找到inode table的block号。**

因为GDT总是和superblock在同一个块组，而superblock总是在分区的第1024-2047个字节，所以很容易就知道第一个GDT所在的块组以及GDT在这个块组中占用了哪些block。

- **在inode table的block中定位到根"/"的inode，找出"/"指向的data block。**

前文说过，ext文件系统预留了一些inode号，其中"/"的inode号为2，所以可以根据inode号直接定位根目录文件的data block。

- ***\*在"/"的datablock中记录了var目录名和var的inode号，找到该inode记录，inode记录中存储了指向var的block指针，所以也就找到了var目录文件的data block\******。**
- **在var的data block中记录了log目录名和其inode号，通过该inode号定位到该inode所在的块组及所在的inode table，并根据该inode记录找到log的data block。**
- **在log目录文件的data block中记录了messages文件名和对应的inode号，通过该inode号定位到该inode所在的块组及所在的inode table，并根据该inode记录找到messages的data block。**
- **最后读取messages对应的datablock。**

#### 删除

https://blog.51cto.com/14551658/2440543

inode中存储硬链接的数量。

删除文件，只是从目录的datablock中删除了文件记录。

文件的数据以及inode要等到引用计数为0才能释放。

#### 移动

同目录内重命名文件的动作仅仅只是修改所在目录data block中该文件记录的文件名部分，不是删除再重建的过程。



同文件系统下移动文件实际上是修改目标文件所在目录的data block，向其中添加一行指向inode table中待移动文件的inode指针，



对于不同文件系统内的移动，相当于先复制再删除的动作。见后文。



关于文件移动，在Linux环境下有一个非常经典网上却又没任何解释的问题：/tmp/a/a能覆盖为/tmp/a吗？答案是不能，但windows能。为什么不能？



因为覆盖首先要删除原来的目标，也就是/tmp下的a文件。

但是a下的a又在使用中，删除不了。

#### magic number

inode中表示文件系统类型。

#### 分段

![截屏2020-07-10 下午7.23.57](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-10 下午7.23.57.png)

![截屏2020-07-10 下午7.22.54](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-10 下午7.22.54.png)

https://www.jianshu.com/p/903af75665d9

#### flock

文件锁

防止多进程读写冲突。



- flock 提供的文件锁是建议性质的。所谓 “建议性锁”，通常也叫作非强制性锁，即一个进程可以忽略其他进程加的锁，直接对目标文件进行读写操作。因而，只有当前进程主动调用 flock去检测是否已有其他进程对目标文件加了锁，文件锁才会在多进程的同步中起到作用。表述的更明确一点，就是如果其他进程已经用 flock 对某个文件加了锁，当前进程在读写这一文件时，未使用 flock 加锁（即未检测是否已有其他进程锁定文件），那么当前进程可以直接操作这一文件，其他进程加的文件锁对当前进程的操作不会有任何影响。这就种可以被忽略、需要双方互相检测确认的加锁机制，就被称为 ”建议性“ 锁。
- 从man page 的描述中还可以看到，文件锁必须作用在一个打开的文件上，即从应用的角度看，文件锁应当作用于一个打开的文件句柄上。知所以需要对打开的文件加锁，是和 linux 文件系统的实现方式、及多进程下锁定文件的需求共同决定的，在下文对其原理有详细介绍（关闭时自动释放）



- sh共享锁
- ex独占锁
- NB（不加，阻塞直到获得锁；不阻塞直接返回）

https://zhuanlan.zhihu.com/p/25134841

#### 文件删除



一般来说，每个文件都有2个link计数器:i_count 和 i_link。

i_count的意义是当前文件使用者（或被调用）的数量,i_link 的意义是介质连接的数量（硬链接的数量）；可以理解为i_count是内存引用计数器，i_link是磁盘的引用计数器。

当一个文件被某一个进程引用时，对应i_count数就会增加；当创建文件的硬链接的时候，对应i_link数就会增加。
————————————————
版权声明：本文为CSDN博主「maintain001」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/u012999810/java/article/details/86288678



- dentry建立了文件名到inode的映射关系
- dcache是一个缓存，缓存了dentry



- 我们根据文件路径从目录项缓存dcache中查找对应的目录项dentry
- 然后根据dentry->d_inode找到对应的inode。
- 对这个dentry unlink，然后当引用数为零时删除



(1)找到文件的inode和data block(根据前一个小节中的方法寻找)；

(2)将inode table中该inode记录中的data block指针删除；

(3)在imap中将该文件的inode号标记为未使用；

(4)在其所在目录的data block中将该文件名所在的记录行删除，删除了记录就丢失了指向inode的指针（实际上不是真的删除，直接删除的话会在目录data block的数据结构中产生空洞，所以实际的操作是将待删除文件的inode号设置为特殊的值0，这样下次新建文件时就可以重用该行记录）；

(5)将bmap中data block对应的block号标记为未使用。



当(2)中删除data block指针后，将无法再找到这个文件的数据；当(3)标记inode号未使用，表示该inode号可以被后续的文件重用；当(4)删除目录data block中关于该文件的记录，真正的删除文件，外界再也定位也无法看到这个文件了；当(5)标记data block为未使用后，表示开始释放空间，这些data block可以被其他文件重用。