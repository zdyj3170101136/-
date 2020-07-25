#### defer panic

我们先通过几个例子了解一下使用 `panic` 和 `recover` 关键字时遇到的一些现象，部分现象也与上一节分析的 `defer` 关键字有关：

- `panic` 只会触发当前 Goroutine 的延迟函数调用；
- `recover` 只有在 `defer` 函数中调用才会生效；
- `panic` 允许在 `defer` 中嵌套多次调用；

![截屏2020-07-25 下午2.30.17](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-25 下午2.30.17.png)

但是这个 defer 在多个协程之间是没有效果，在子协程里触发 panic，只能触发自己协程内的 defer，而不能调用 main 协程里的 defer 函数的。



panic之后，协程退出，整个程序也会退出