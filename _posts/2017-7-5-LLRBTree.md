---
layout: post
title: 左倾红黑树的C++实现
tags:
- Graph
- C++
categories: Algorithms
---
Unique Studio Lab 第一期任务，在完成之后整理一下经验。

## 从二叉搜索树，AVL树谈起

### 二叉搜索树

二叉查找树(Binary Search Tree)，是指一棵空树或者具有下列性质的二叉树：

1. 若任意节点的左子树不空，则左子树上所有结点的值均小于它的根结点的值；
2. 若任意节点的右子树不空，则右子树上所有结点的值均大于它的根结点的值；
3. 任意节点的左、右子树也分别为二叉查找树；
4. 没有键值相等的节点。

二叉查找树相比于其他数据结构的优势在于查找、插入的时间复杂度较低，为O(log n)。但是，尽管期望是O(log n)，仍然有最坏的O(n)会出现【即：树非常高，结构退化为线性表】，为了解决这个问题，于是有了平衡树的出现。

二叉搜索树的算法比较简单，也不是重点，这里我就不再赘述。

### AVL树

平衡树这里就举一个经典的例子来说明，即AVL树(得名于它的发明者G.M. Adelson-Velsky和E.M. Landis)，它在二叉搜索树的基础上增加了以下要求：

* 任意一个结点的左右子树高度差为0或者1.

***这里的高度定义为树从底层至上的层数***

AVL树的基本算法完全继承二叉搜索树，需要新增的是为了使树变得平衡的操作，分析发现，假设结点A发生了失衡，则有以下四种情况：

1. 在A的左儿子的左子树上进行了一次插入
2. 在A的右儿子的右子树上进行了一次插入
3. 在A的左儿子的右子树上进行了一次插入
4. 在A的右儿子的左子树上进行了一次插入

显然1和2是对称的，3和4是对称的，我们称1和2的情形为单旋转，3和4的情形为双旋转。

两种单旋转如下图：

![](images/LLRBTree-01.jpg)
![](images/LLRBTree-02.jpg)

两种双旋转如下图：

![](images/LLRBTree-03.jpg)
![](images/LLRBTree-04.jpg)

可以看出实际上双旋转就是两个单旋转的结合而已。

事实上，这几种旋转的实现也是简单的，这里不多讲，我们的重点在后面。以上两种数据结构实现的C++代码可以在https://github.com/zxc479773533/uniquelab-2017-summer-assignment-1里面查看。

## 红黑树，左倾红黑树

红黑树(Red–black tree)，是每个节点都带有颜色属性的二叉查找树，颜色为红色或黑色。在二叉查找树强制一般要求以外，对于任何有效的红黑树我们增加了如下的额外要求：

1. 节点是红色或黑色。
2. 根是黑色。
3. 所有叶子都是黑色（把最后的NULL指针当做叶子）。
4. 每个红色节点必须有两个黑色的子节点。（从每个叶子到根的所有路径上不能有两个连续的红色节点。）
5. 从任一节点到其每个叶子的所有简单路径都包含相同数目的黑色节点。

这些条件就保证了：从根到叶子的最长的可能路径不多于最短的可能路径的两倍长。结果是这个树大致上是平衡的。红黑树和AVL树一样都对插入时间、删除时间和查找时间提供了最好可能的最坏情况担保。相对于AVL树来说，红黑树牺牲了部分平衡性以换取插入/删除操作时少量的旋转操作，整体来说性能要优于AVL树。

接下来我们讨论的是左倾红黑树(Left-leaning Red–Black Tree)，即增加了一条限制：所有的红色结点都位于其父结点的左儿子位置上。相对于红黑树来说，左倾红黑树的插入操作要讨论的多一些，删除操作则要方便一些。

>关于LLRBTree，我相信这篇论文是极好的 http://www.cs.princeton.edu/~rs/talks/LLRB/RedBlack.pdf，想要深刻，完全的学习LLRBTree可以阅读这篇论文。

## 左倾红黑树的基本操作和类实现

首先，树的结点类如下：

```c++
enum RBTColor {RED, BLACK};

class RBTNode {
    public:
        RBTColor color; //存储颜色
        T key; //存储数据
        RBTNode *left; //左子结点
        RBTNode *right; //右子结点
        RBTNode *parent; //父结点
    //构造函数
        RBTNode(RBTColor c, T value, RBTNode *l, RBTNode *r, RBTNode *p) :
        color(c), key(value), left(l), right(r), parent(p) {};
};
```

大概就是比二叉搜索树多了个存储颜色的单元。

基本操作有：

* insert(const T&)：插入指定的元素结点
* erase(count T&)：删除指定的元素结点
* clear()：清空该树
* count(const T&)：判断该数据是否存在
* empty()：判断该树是否为空
* size()：返回树中结点的个数

该类实现的代码如下：

```c++
class Set {
    public:
        void insert(const T& element); //插入指定的结点
        void erase(const T& element); //删除指定的结点
        void clear(); //清空树
        int count(const T& element) const; //返回该数据是否存在
        bool empty(); //返回该树是否为空
        size_t size(); //返回树中结点的个数
    private:
        RBTNode *root = NULL;
        /* 接口 */
        void ins(RBTNode* tree, RBTNode* node); //插入
        void del(RBTNode* node); //删除
        void _clear(RBTNode* tree); //删除
        int _count(const T&); //判断数据是否存在
        size_t _size(RBTNode*  tree); //计数
        /* 内部操作函数 */
        RBTNode* find(RBTNode* tree, const T&); //查找
        RBTNode* find_min(RBTNode* tree); //查找最小值
        RBTNode* find_max(RBTNode* tree); //查找最大值
        /* 结点相关操作 */
        RBTNode* son(RBTNode* tree); //返回单儿子时的儿子结点
        RBTNode* sibling(RBTNode* tree); //返回兄弟结点
        RBTNode* grandparent(RBTNode* tree); //返回祖父结点
        RBTNode* uncle(RBTNode* tree); //返回叔父结点
        /* 红黑树特有部分函数 */
        void leftrotation(RBTNode* tree); //执行左旋
        void rightrotation(RBTNode* tree); //执行右旋
        void changecolor(RBTNode* tree); //执行改色
        /* 插入删除的辅助函数 */
        void insadjust(RBTNode* node); //插入调整
        void del_adjust1(RBTNode* node); //删除调整一
        void del_adjust2(RBTNode* node); //删除调整二
        void del_adjust3(RBTNode* node); //删除调整三
        void del_adjust4(RBTNode* node); //删除调整四
};
```
## 辅助操作函数

**find**

就是二叉搜索树的查找操作，一个深搜就好。

```c++
/* 查找 */
RBTNode* Set::find(RBTNode* tree, const T& key) {
    if (tree == NULL)
        return NULL;
    else {
        if (tree->key == key)
            return tree;
        else if (tree->key > key)
            return find(tree->left, key);
        else
            return find(tree->right, key);
    }
}
```

**find_min 和 find_max**

结合二叉搜索树数据的特性，我们只需要一直向左搜或者向右搜即可。实现的代码如下：

```c++
/* 查找最小数 */
RBTNode* Set::find_min(RBTNode* tree) {
    if (tree->left == NULL)
        return tree;
    else
        return find_min(tree->left);
}

/* 查找最大数 */
RBTNode* Set::find_max(RBTNode* tree) {
    if (tree->right == NULL)
        return tree;
    else
        return find_max(tree->right);
}
```

此外，我们还需要一些快速返回附近结点的方法，从而使得我们在插入和删除的操作中可以少写一下代码。

四种返回附近结点的函数如下：

```c++
/* 返回单儿子时的儿子结点 */
RBTNode* Set::son(RBTNode* tree) {
    if (tree->left != NULL)
        return tree->left;
    else
        return tree->right;
}

/* 返回兄弟结点 */
RBTNode* Set::sibling(RBTNode* tree) {
    if (tree == tree->parent->left)
        return tree->parent->right;
    else
        return tree->parent->left;
}

/* 返回祖父结点 */
RBTNode* Set::grandparent(RBTNode* tree) {
    return tree->parent->parent;
}

/* 返回叔父结点 */
RBTNode* Set::uncle(RBTNode* tree) {
    RBTNode* g = grandparent(tree);
    if (tree->parent == g->left)
        return g->right;
    else
        return g->left;
}
```

## 一些基本操作的实现

借助辅助函数，可以轻松的实现几个简单的基本操作。

不多说了，直接放代码。

```c++
/* 清空树接口 */
void Set::clear() {
    _clear(root);
}

/* 清空树实现 */
void Set::_clear(RBTNode* tree) {
    while(tree != NULL) {
        delete(tree->left);
        delete(tree->right);
        delete(tree);
    }
}

/* 判断数据是否存在接口 */
int Set::_count(const T& key) {
    if (find(root, key) != NULL)
        return 1;
    else
        return 0;
}       

/* 返回该树是否为空 */
bool Set::empty() {
    if (root == NULL)
        return true;
    else
        return false;
}

/* 返回树中结点的个数 */
size_t Set::size() {
    return _size(root);
}

/* 计数 */
size_t Set::_size(RBTNode*  tree) {
    if (tree == NULL)
        return 0;
    else
        return 1 + _size(tree->left) + _size(tree->right);
}
```

## 调整树结构的操作

我们的重点，也是红黑树实现的难点在于插入和删除操作。在此之前，我们先解决几个调整树的基本操作。

### 左旋

### 右旋

### 改色