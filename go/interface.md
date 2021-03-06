#### interface

- 在 Go 中：实现接口的所有方法就隐式的实现了接口；

我们使用上述 `RPCError` 结构体时并不关心它实现了哪些接口，Go 语言只会在传递参数、返回参数以及变量赋值时才会对某个类型是否实现接口进行检查，这里举几个例子来演示发生接口类型检查的时机：

![截屏2020-07-22 下午1.16.29](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-22 下午1.16.29.png)



如果我们使用结构体指针初始化变量，那么可以使用（*）ptr自然的实现用结构体实现的接口。

结构体指针同理。



而我们使用结构体初始化变量，那么我们其实是值复制，创建了一个新的结构体，而虽然eface中保存的是指针，不过已经保存的是新结构体的指针了，因此结构体指针实现的接口不通过。

因为已经不是操作原来的结构体了。

![截屏2020-07-15 下午11.27.58](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-15 下午11.27.58.png)



作为指针的 `&Cat{}` 变量能够**隐式地获取**到指向的结构体，所以能在结构体上调用 `Walk` 和 `Quack` 方法。我们可以将这里的调用理解成 C 语言中的 `d->Walk()` 和 `d->Speak()`，它们都会先获取指向的结构体再执行对应的方法。![截屏2020-07-15 下午11.29.21](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-15 下午11.30.23.png)

![截屏2020-07-15 下午11.38.17](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-15 下午11.38.23.png)

#### interface

任何一个interface都占用16字节的空间，两个八字节的指针。



再structtype和functype或者interface中都会有pkgpathname。

如果为大写，表明可以再在包外部调用。

![截屏2020-07-15 下午11.39.29](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-15 下午11.39.29.png)

而hash存储着_type的hash，类型断言的时候用于对比。



首先无论是我们interface中的data是实际结构体的指针。

（如果传递指针，由于值拷贝，是结构体指针变量的副本

如果传递值，会复制值，然后data指针指向这个副本）。

这说明为什么结构体指针初始化变量和结构体实现接口也ok。

因为结构体指针初始化变量等价于结构体初始化变量（对于interface来说）。

#### 有方法的interface变量赋值

还存有interface变量的_type结构体。

#### 方法调用

interfacetype里头的存有method

![截屏2020-07-15 下午11.44.04](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-15 下午11.49.59.png)

func存储的是实现了interface的实际函数。

typeof先转换成空接口，然后把控接口用unsfe.Pointer修改类型转换成emptyinterface，

然后用eface的_type作为参数，返回一个实现了Type这个接口的变量。

#### reflect。typeof

![截屏2020-07-15 下午11.55.45](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-15 下午11.55.45.png)

#### reflect.valueof

也是首先转化为空接口。



而reflect。valueof根据字段名修改字段。

原理就是通过structtype对应的字段的_type获取名字，然后通过ptr指向的数据修改。

flag表示是否可以修改。



通过把对应的结构体变为interface{}也就时eface。

eface里头保存了这个结构体的type。

#### 反射修改值

![截屏2020-07-16 上午12.00.48](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-16 上午12.00.48.png)

#### 反射获取方法



![截屏2020-07-15 下午11.57.22](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-16 上午12.00.13.png)

https://www.cnblogs.com/shijingxiang/articles/12201984.html

![截屏2020-07-16 上午12.06.38](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-16 上午12.06.38.png)

#### 内嵌

![截屏2020-07-22 下午1.27.46](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-22 下午1.27.46.png)





![截屏2020-07-22 下午1.28.49](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-22 下午1.28.49.png)

内嵌入的初始化时自动赋值。



结构体中包含匿名（内嵌）字段叫嵌入或者内嵌；而如果结构体中字段包含了类型名，还有字段名，则是聚合。聚合的在JAVA和C++都是常见的方式，而内嵌则是Go 的特有方式。