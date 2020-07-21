```
{
   []byte{11},
   []byte{11},
},
{
   []byte{12},
   []byte{12},
},
{
   []byte{1},
   []byte{1},
},
{
   []byte{25},
   []byte{25},
},
{
   []byte{7},
   []byte{7},
},
```

测试数据如上。

#### 第一个数据

在findge函数中node = 0，h = 11。

由于cmp默认为1，所以height一直减小。

height减小到0，返回next为0，同时因为没有找到，返回false。

从height【0:12)全部都为0。

1
[11 11]
[0 0 0 12 16 0 0 0 0 0 0 0 0 0 0 0 0 1 1 1 0]

height为1，因此prevnode为0，更新height的第0层。

#### 第二个数据

findnode中当找到height为1时，为11的值。

对比后小于，因此height为1时候的prevnode为16。

生成随机head为1，然后在16号prevnode的第一个高度添加node为21.

1
[11 11 12 12]
[0 0 0 12 16 0 0 0 0 0 0 0 0 0 0 0 0 1 1 1 21 2 1 1 1 0]

#### 第三个数据

第三个数据是1，我们一直从height直到0，发现都小于。

因此prevnode全部为0.

因此nodedata附加，然后更新0层对应位置的height。

相当于插入0层节点的第一个指针的头部。

1
[11 11 12 12 1 1]
[0 0 0 12 26 0 0 0 0 0 0 0 0 0 0 0 0 1 1 1 21 2 1 1 1 0 4 1 1 1 16]

// 注意这里最后一个零的原因

#### 第四个数据

height为1的时候遍历到21。

因此prevnode为21，更新21的nodedata的0层的值。

1
[11 11 12 12 1 1 25 25]
[0 0 0 12 26 0 0 0 0 0 0 0 0 0 0 0 0 1 1 1 21 2 1 1 1 31 4 1 1 1 16 6 1 1 1 0]

#### 第五个数据

第五个数据是7，它会遍历到第一层的26，即key为1，随后往后遍历，到key为11，nodedata下标为16的。这时候大于。

因为返回下标26，false。

然后随机height为2，大于当前的最大高度。

因此我们的0层节点的相应指针初始化为新的nodedata36。

而26的第1个height我们更新为36。

插入一个节点。

2
[11 11 12 12 1 1 25 25 7 7]
[0 0 0 12 26 36 0 0 0 0 0 0 0 0 0 0 0 1 1 1 21 2 1 1 1 31 4 1 1 1 36 6 1 1 1 0 8 1 1 2 16 0]



#### random

这个random输出的数字，似乎相性很好。。。诡异啊

1
1
1
1
2
1
1
1
1
1
5
1
1
1
1
1
1
1
2
2
1
2
2
1
1
2
1
1
1
1
1
1
1
1
1
1
1
1
1
1
1
1
2
1
2
2
1
2
2
2

#### 源代码

```
package main

import (
	"bytes"
	"fmt"
	"github.com/syndtr/goleveldb/leveldb/errors"
	"math/rand"
	"sync"
)

// Common errors.
var (
	ErrNotFound     = errors.ErrNotFound
	ErrIterReleased = errors.New("leveldb/memdb: iterator released")
)


const tMaxHeight = 12

func main() {
	db := New(1000)

	testdatas := []struct{
		key []byte
		value []byte
	} {
		{
			[]byte{11},
			[]byte{11},
		},
		{
			[]byte{12},
			[]byte{12},
		},
		{
			[]byte{1},
			[]byte{1},
		},
		{
			[]byte{25},
			[]byte{25},
		},
		{
			[]byte{7},
			[]byte{7},
		},
	}

	for _, test := range testdatas {
		db.Put(test.key, test.value)
		fmt.Println(db.kvData)
		fmt.Println(db.nodeData)
	}
	// 其实默认为12，打印第0个node的所有层高
	// 第i个height包括所有第i层的节点的nodedata的下标
	// 因此第一个height包括所有的node其实
	height := db.nodeData[nHeight]
	// 不断打印节点
	for i := 1; i <= height; i++ {
		start := 0

		if db.nodeData[start + nHeight + i] == 0 {
			break
		}
		fmt.Printf("第%d层节点如下\n", i)

		// 如果这一层的点不为空
		for ; db.nodeData[start + nHeight + i] != 0; {
			fmt.Printf("node date:kv offset, k length, v length, height, next...\n")
			newnode := db.nodeData[start + nHeight + i]        // 新节点在nodedata的下标
			newnodeHeight := db.nodeData[newnode + nHeight]    // 对应的height长度

			newnodeData := db.nodeData[newnode: newnode + 4 + newnodeHeight]
			fmt.Println(newnodeData)

			fmt.Printf("key value\n")
			// 把key和value也打印出来
			kvoffset := newnodeData[nKV]
			klen := newnodeData[nKey]
			vlen := newnodeData[nVal]
			fmt.Println(db.kvData[kvoffset: kvoffset + klen + vlen])
			start = db.nodeData[start + nHeight + i]
		}
	}
}


const (
	nKV = iota
	nKey
	nVal
	nHeight
	nNext
)



// DB is an in-memory key/value database.
type DB struct {
	rnd *rand.Rand       // 随机种子,记住命名这里是指针的原因，是因为很多是指针接收器和值接收器

	mu     sync.RWMutex  // 控制并发，不要指针
	kvData []byte // 存放kv数据
	// Node data:
	// [0]         : KV offset
	// [1]         : Key length
	// [2]         : Value length
	// [3]         : Height
	// [3..height] : Next nodes   //
	nodeData  []int
	prevNode  [tMaxHeight]int   // 注意是一个数组
	maxHeight int
	n         int
	kvSize    int
}

// New creates a new initialized in-memory key/value DB. The capacity
// is the initial key/value buffer capacity. The capacity is advisory,
// not enforced.
//
// This DB is append-only, deleting an entry would remove entry node but not
// reclaim KV buffer.
//
// The returned DB instance is safe for concurrent use.
func New(capacity int) *DB {
	p := &DB{
		rnd:       rand.New(rand.NewSource(0xdeadbeef)), // magicnumber，输出固定的数字
		maxHeight: 1, // 初始化为1
		kvData:    make([]byte, 0, capacity),
		nodeData:  make([]int, 4+tMaxHeight), // 四的倍数
	}
	p.nodeData[nHeight] = tMaxHeight   // 存储node的高度
	return p
}


func (p *DB) randHeight() (h int) {
	const branching = 4
	h = 1
	for h < tMaxHeight && p.rnd.Int()%branching == 0 {
		// 生成随机数，0，1，2，3。四分之一的概率增加
		h++
	}
	return
}

// bool表示找到的下标是否相等
// Must hold RW-lock if prev == true, as it use shared prevNode slice.
func (p *DB) findGE(key []byte, prev bool) (int, bool) {
	node := 0  // 从第零个节点开始遍历
	h := p.maxHeight - 1  // 从最高的指针开始搜索
	for {

		next := p.nodeData[node+nNext+h]
		cmp := 1                      // 说明默认高度减小（包括当前无指针的时候）
		if next != 0 {                // next不为零，说明此时指针不为空
			o := p.nodeData[next]     // 从h高度的指针的nodedata取出key的长度，我们知道【0，1）长度为1，然后比较大小
			cmp = bytes.Compare(p.kvData[o:o+p.nodeData[next+nKey]], key)
		}
		if cmp < 0 {                  // 持续从高度为height的里头找到大于等于当前的key
			// Keep searching in this list
			node = next  // 注意只有小于才会更新node
		} else {                       // 记录prevNode， 存储了每个高度的prevnode， prevnode表示所有大于等于当前节点的节点的值
			if prev {
				p.prevNode[h] = node
			} else if cmp == 0 {       // 如果相等就返回
				return next, true      // prevnode存储的是之前的节点的下标，因此是弄的；这里存的是要插入在之前的节点的下标，因此是next
			}
			if h == 0 {                // 如果高度为0，则直接返回(为什么这里返回next，因为我们会遍历0层节点到一层节点间的指针，next为可能等于的弄的)
				return next, cmp == 0
			}
			h--                        // 不然减小高度，精细化搜索
		}
	}
}


// Put sets the value for the given key. It overwrites any previous value
// for that key; a DB is not a multi-map.
//
// It is safe to modify the contents of the arguments after Put returns.
func (p *DB) Put(key []byte, value []byte) error {
	p.mu.Lock()
	defer p.mu.Unlock()

	if node, exact := p.findGE(key, true); exact {  // 如果相等，替换
		kvOffset := len(p.kvData)                 // 附加key，value
		p.kvData = append(p.kvData, key...)
		p.kvData = append(p.kvData, value...)
		p.nodeData[node] = kvOffset               // 更新偏移量
		m := p.nodeData[node+nVal]                // 更新value的元数据, 别忘了
		p.nodeData[node+nVal] = len(value)        // 更新kvsize的大小
		p.kvSize += len(value) - m                // 我们只记录最新的（这就是kvsize的来源）
		return nil
	}

	h := p.randHeight()  // maxheight初始为1，如果随机的height大于当前的最高height，那么首先更新maxheight
	if h > p.maxHeight { // 然后当前高度的height直接指向0层节点，因为0层节点具有所有其他height层节点的指针哈。
		for i := p.maxHeight; i < h; i++ {
			p.prevNode[i] = 0 // 指向0层节点
		}
		p.maxHeight = h
	}
	fmt.Println(h)

	kvOffset := len(p.kvData)
	p.kvData = append(p.kvData, key...)
	p.kvData = append(p.kvData, value...)  // 在kvdata的末尾添加key和value的数据
	// Node
	node := len(p.nodeData)                // 在nodedata末尾添加元数据和生成的随机高度
	p.nodeData = append(p.nodeData, kvOffset, len(key), len(value), h)   // 对于新节点，添加在后面
	for i, n := range p.prevNode[:h] {     // 如果height为1，就只有一个节点
		m := n + nNext + i                 // i表示高度，说明对于第n个node，它的第i位置插入一个元素
		p.nodeData = append(p.nodeData, p.nodeData[m])  // 当前node插入在prevnode和prevnode之前的node， 至少也会有一个为0,容易忘记！！！
		p.nodeData[m] = node
	}

	p.kvSize += len(key) + len(value)       // 增加大小
	p.n++
	return nil
}

// Get gets the value for the given key. It returns error.ErrNotFound if the
// DB does not contain the key.
//
// The caller should not modify the contents of the returned slice, but
// it is safe to modify the contents of the argument after Get returns.
func (p *DB) Get(key []byte) (value []byte, err error) {
	p.mu.RLock()   // 加读锁
	if node, exact := p.findGE(key, false); exact { // 重点是对exact的判断
		o := p.nodeData[node] + p.nodeData[node+nKey]
		value = p.kvData[o : o+p.nodeData[node+nVal]]
	} else {
		err = ErrNotFound
	}
	p.mu.RUnlock()
	return
}
```