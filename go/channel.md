#### channel

在多个协程之间同步和交流。

https://blog.csdn.net/u010853261/article/details/85231944

- 可以让协程睡眠

- 在多个协程之间传递值

- fifo语意

  

  #### 在堆上分配内存

  

- ![截屏2020-07-29 下午1.48.50](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-29 下午1.48.50.png)

#### 内部结构

- 两个双向等待队列
- 一个缓冲区

![截屏2020-07-29 下午1.52.12](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-29 下午1.52.12.png)



`chan struct{}` 类型的异步 Channel — `struct{}` 类型不占用内存空间，不需要实现缓冲区和直接发送（Handoff）的语义；



#### 直接传递

![截屏2020-07-29 下午1.56.05](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-29 下午1.56.05.png)