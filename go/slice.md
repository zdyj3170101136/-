#### ![截屏2020-07-16 下午8.41.44](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-16 下午8.41.44.png)slice

字面量的形式都会在编译的时候转换为对应的代码形式。

但是还是需要运行时候的支持，因为只有运行时候才会生成对应的代码。


如果容量比较大或者逃逸分析，那么就会通过makeslice生成。

#### 对于append（）

对于append和非append都会扩取容量，不同的是，append还会改写len。



#### 会预估一个期望容量

- 如果期望容量大于两倍直接期望
- 不然如果当前切片小于1024，就会翻倍
- 如果大于1024就会每次增加25%，直到新容量大于期望容量。



然后使用roundUpsize。



新分配一个内存空间，把旧的内容移动过去。



#### copy

copy使用memove移动



防止切片频繁扩容导致的复制。

func possibleBipartition() bool {

	a := [...]int{3, 2, 1}
	
	data := a
	sort.Sort(sort.IntSlice(data[0:]))
	fmt.Println(a)
	return true
}

// [...]表示数组
// 而[0:]表示生成切片，毕竟slice是生成切片的

https://draveness.me/golang/docs/part2-foundation/ch03-datastructure/golang-array-and-slice/