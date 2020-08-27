#### waitgroup

![截屏2020-07-16 上午11.20.23](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-16 上午11.20.23.png)

一个64位的变量表示state，一个32位的变量表示sema。

state中前32位表示counter，就是说有多少个任务正在执行。

而waiter表示有多少个程序睡眠等待。



Add（）表示有多少个任务在执行。

done表示有多少个已经执行完了。

wait表示主线程等待其他任务执行完。

#### wait原理

wait首先读取state，如果当前counter为0就返回。

如果然后用cas和for循环不断改变waiter的值。

改变成功了就挂到信号量sema上。（睡眠）

![截屏2020-07-16 上午11.22.46](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-16 上午11.22.46.png)

```go
// Done decrements the WaitGroup counter by one.
func (wg *WaitGroup) Done() {
   wg.Add(-1)
}
```

#### add

![截屏2020-07-16 上午11.24.46](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-16 上午11.24.46.png)

不断的增加counter上的值，表示任务有多少个正在执行；

执行完毕调用done（）减去counter上的值。

如果counter为0的话。

就调用waiter次信号量释放，把阻塞的全部唤醒。

#### mutex

//

唤醒标志说的是当前是否有协程正在抢这个锁。

这样的话ulnock的时候，如果已经有唤醒的协程了，那么就没有必要唤醒已经睡眠的了。





​	/** 互斥量可分为两种操作模式:正常和饥饿。
​	【正常模式】，等待的goroutines按照FIFO（先进先出）顺序排队，但是goroutine被唤醒之后并不能立即得到mutex锁，它需要与新到达的goroutine争夺mutex锁。
​	因为新到达的goroutine已经在CPU上运行了，所以被唤醒的goroutine很大概率是争夺mutex锁是失败的。出现这样的情况时候，被唤醒的goroutine需要排队在队列的前面。
​	如果被唤醒的goroutine有超过1ms没有获取到mutex锁，那么它就会变为饥饿模式。
​	在饥饿模式中，mutex锁直接从解锁的goroutine交给队列前面的goroutine。新达到的goroutine也不会去争夺mutex锁（即使没有锁，也不能去自旋），而是到等待队列尾部排队。
​	【饥饿模式】，锁的所有权将从unlock的gorutine直接交给交给等待队列中的第一个。新来的goroutine将不会尝试去获得锁，即使锁看起来是unlock状态, 也不会去尝试自旋操作，而是放在等待队列的尾部。如果有一个等待的goroutine获取到mutex锁了，如果它满足下条件中的任意一个，mutex将会切换回去正常模式：

	1. 是等待队列中的最后一个goroutine
	2. 它的等待时间不超过1ms。
	正常模式：有更好的性能，因为goroutine可以连续多次获得mutex锁；
	饥饿模式：能阻止尾部延迟的现象，对于预防队列尾部goroutine一致无法获取mutex锁的问题。
	*/
![截屏2020-07-16 上午11.55.20](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-16 上午11.55.20.png)

一个32位的变量和一个信号量组成。

29位用来表示阻塞的goroutine数量，一个是饥饿状态位，一个是唤醒状态位，最后一位是锁状态。

![截屏2020-07-16 上午11.56.57](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-16 上午11.56.57.png)

而所谓的lock操作其实就是在state上增一。

cas（如果为0，则增加一）

这是fast，后面还有一个slow：修改state没成功的情况。（变成函数，让qian）![截屏2020-07-16 上午11.59.33](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-16 上午11.59.33.png)

#### spin

第一次没获取到锁。

我们要尝试能否自旋：就是运行空的cpu指令。（毕竟有可能我等一会就可以了呢）

但是自旋需要占用cpu了，这是一个浪费（所以自旋最多四次，而且得是多核cpu，并且当前的p可运行队列为空，就是说没有一个协程等待这个线程在运行）。



#### 信号量

如果自旋四次还没获取，总不好一直占着cpu，因此我们要把自己挂到信号量上，睡眠。



#### runtime_mutex

Goroutine，如果是rumtime级别的mutex，就是那些阻塞在gorountne上面的，还没调用sleep休眠当前线程，触发操作系统调用。



而协程层面的不会，大不了换一个线程去运行对吧。（会在global run q）



#### rwmutex

![截屏2020-07-16 下午12.30.18](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-16 下午12.30.18.png)



LOCK：写锁间通过mutex竞争，然后把readerCount = readerCount - max。（修改为负数）。

然后readerWait加上readercount，表示获取写锁时需要等待的读锁释放数量。

然后写锁挂在writerswam信号量上睡眠。



其实readercount是用来表示当前有多少个读者的。

而写锁获取读锁的时候，如果当前有读者，那么要挂在信号量上然后等待释放对吧。

而readerwait就是获取写锁需要等待释放的读锁数量。



读锁通过增加readercount来表示现在有多少个读者。那么怎么才能让有写锁等待读锁释放的时候，后面的读者获取不到锁呢。这是通过写锁等待的时候，会把readercount减去一个很大的数，让它变为一个负数，然后读者增加readercount的时候发现是负数，就是休眠阻塞在readerseam上。



那么写锁什么时候才能唤醒呢，就是读锁释放的时候。

会减小readerwait，readerwait为零，表示写锁等待的读锁都释放了，写锁就会释放。



而写锁释放的时候，就增加readercount，让后面的读锁能够获取到，不会为负数了；

然后释放readerseam，把所有等待写锁的读锁都释放掉。



readerwait是用来奇数writerseam。（表示写锁信号量上有多少个进程在阻塞）



- readerwait和readercount的区别



然后RLOCK：readercount+ 1，表示新增一个当前读锁；

如果 < 0，说明当前有写锁在等待，则挂在readerseam之上。



而Runlock：readercount - 1，表示少了一个读者；

如果 < 0，说明有写锁在等待；

这个时候减小readerWait。

而如果readerwait为0，那么就释放writerseam。



Unlock：

就把readercount + max，变为正数，后面的reader就不用休眠。

然后释放readercount次readerseam。



