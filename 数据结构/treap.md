Treap是一种平衡二叉树，不过Treap会记录一个优先级（一般来说是随机生成），即Treap在以关键码构成二叉搜索树的同时，还会按照优先级的高低满足堆的性质，因此得名Treap（Tree + Heap）。 Treap不是二叉堆，二叉堆必须是完全二叉树，但Treap不必是。

对于每个结点，该结点的优先级不大于其所有孩子的优先级。Treap引入优先级的原因就是防止BST（二叉搜索树）退化成一条链，从而影响查询效率。

所以对于结点上的关键字来说，它是一颗BST，而对于结点上的优先级来讲，它是一个小顶堆。其平均查找长度为 O(logn) O(log⁡n) 。 Treap有插入、删除、旋转和查询等基本操作，进而可以实现查询第 k k 大和查询关键字 x x 排名等功能。

一个随机的优先级，生成一个小顶堆。



![截屏2020-07-21 下午11.11.54](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-21 下午11.11.54.png)

#### 左旋转

![截屏2020-07-21 下午11.13.08](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-21 下午11.13.08.png)

**左旋**：将子树的根结点旋转到其根的左子树位置，同时根节点的右子节点成为该子树的根；

#### 右旋转

![截屏2020-07-21 下午11.14.16](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-21 下午11.14.16.png)

**右旋**：将子树的根结点旋转到其根的右子树位置，同时根节点的左子节点成为该子树的根。

```
// Copyright 2011 Numerotron Inc.
// Use of this source code is governed by an MIT-style license
// that can be found in the LICENSE file.
//
// Developed at www.stathat.com by Patrick Crosby
// Contact us on twitter with any questions:  twitter.com/stat_hat

// The treap package provides a balanced binary tree datastructure, expected
// to have logarithmic height.
package main

import (
   "fmt"
   "math/rand"
)

func main() {
   treeh := NewTree()
   treeh.Insert(3)
   treeh.Insert(5)
   treeh.Insert(9)
   treeh.Insert(10)
   treeh.Insert(6)
   treeh.Insert(1)
   treeh.Insert(2)
   fmt.Println(treeh.root)
   fmt.Println(treeh.root.left)
   fmt.Println(treeh.root.right)
   fmt.Println(treeh.root.left.left)
   fmt.Println(treeh.root.left.right)
   fmt.Println(treeh.root.right.left)
   fmt.Println(treeh.root.right.right)

}


// A Tree is the data structure that stores everything
type Tree struct {
   count   int
   root    *Node
}

// A Node in the Tree.
type Node struct {
   key      int
   priority int
   left     *Node
   right    *Node
}

func newNode(key int, priority int) *Node {
   result := new(Node)
   result.key = key
   result.priority = priority
   return result
}

// To create a Tree, you need to supply a LessFunc that can compare the
// keys in the Node.
func NewTree() *Tree {
   t := new(Tree)
   return t
}

// Insert an item into the tree.
func (t *Tree) Insert(key int) {
   priority := rand.Intn(25)
   t.root = t.insert(t.root, key,  priority)
}

func (t *Tree) insert(node *Node, key int,  priority int) *Node {
   if node == nil {
      t.count++
      return newNode(key,  priority)
   }
   if key < node.key {
      node.left = t.insert(node.left, key,  priority)
      if node.left.priority < node.priority {
         return t.leftRotate(node)
      }
      return node
   }
   if node.key < key {
   // 注意这里的判等
      node.right = t.insert(node.right, key, priority)
      if node.right.priority < node.priority {
         return t.rightRotate(node)
      }
      return node
   }

   return node
}

func (t *Tree) leftRotate(node *Node) *Node {
   result := node.left
   x := result.right
   result.right = node
   node.left = x
   return result
}

func (t *Tree) rightRotate(node *Node) *Node {
   result := node.right
   x := result.left
   result.left = node
   node.right = x
   return result
}
```