---
layout: post
title: 左倾红黑树的C++实现
tags:
- C/C++
categories: Algorithm
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

![](https://raw.githubusercontent.com/zxc479773533/zxc479773533.github.io/master/_posts/images/LLRBTree-01.jpg)

![](https://raw.githubusercontent.com/zxc479773533/zxc479773533.github.io/master/_posts/images/LLRBTree-02.jpg)

两种双旋转如下图：

![](https://raw.githubusercontent.com/zxc479773533/zxc479773533.github.io/master/_posts/images/LLRBTree-03.jpg)

![](https://raw.githubusercontent.com/zxc479773533/zxc479773533.github.io/master/_posts/images/LLRBTree-04.jpg)

可以看出实际上双旋转就是两个单旋转的结合而已。

事实上，这几种旋转的实现也是简单的，这里不多讲，我们的重点在后面。以上两种数据结构实现的C++代码可以在[这里](https://github.com/zxc479773533/uniquelab-2017-summer-assignment-1)查看。

## 红黑树，左倾红黑树

红黑树(Red–black tree)，是每个节点都带有颜色属性的二叉查找树，颜色为红色或黑色。在二叉查找树强制一般要求以外，对于任何有效的红黑树我们增加了如下的额外要求：

1. 节点是红色或黑色。
2. 根是黑色。
3. 所有叶子都是黑色（把最后的NULL指针当做叶子）。
4. 每个红色节点必须有两个黑色的子节点。（从每个叶子到根的所有路径上不能有两个连续的红色节点。）
5. 从任一节点到其每个叶子的所有简单路径都包含相同数目的黑色节点。

这些条件就保证了：从根到叶子的最长的可能路径不多于最短的可能路径的两倍长。结果是这个树大致上是平衡的。红黑树和AVL树一样都对插入时间、删除时间和查找时间提供了最好可能的最坏情况担保。相对于AVL树来说，红黑树牺牲了部分平衡性以换取插入/删除操作时少量的旋转操作，整体来说性能要优于AVL树。

然而事实上，完成红黑树的代码是极其复杂的（大约需要四百来行），而且要讨论的情形十分之多，详见我的github上。

接下来我们讨论的是左倾红黑树(Left-leaning Red–Black Tree)，即增加了一条限制：所有的红色结点都位于其父结点的左儿子位置上。相对于红黑树来说，左倾红黑树实现的核心代码就要少了很多，要讨论的情形也少了一些

>关于LLRBTree，我相信[这篇论文](http://www.cs.princeton.edu/~rs/talks/LLRB/RedBlack.pdf)是极好的，想要深刻，完全的学习LLRBTree可以阅读这篇论文。

## 左倾红黑树的基本操作和类实现

首先，树的结点类如下：

```c++
enum RBTColor {BLACK, RED};

class RBTNode {
  public:
    bool color; //存储颜色
    T key; //存储数据
    RBTNode *left; //左子结点
    RBTNode *right; //右子结点
  //构造函数
    RBTNode(RBTColor c, T value, RBTNode *l, RBTNode *r) :
    color(c), key(value), left(l), right(r) {};
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
  //析构函数
    ~Set() {delete root;}
  private:
    RBTNode *root = NULL;
    int currentsize = 0;
    /* 接口 */
    RBTNode* ins(RBTNode* tree, T key); //插入
    RBTNode* del(RBTNode* tree, T key); //删除
    void _clear(RBTNode* tree); //清空
    /* 内部操作函数 */
    RBTNode* find(RBTNode* tree, T key) const; //查找
    bool isred(RBTNode* node); //判断是否为红色
    RBTNode* find_min(RBTNode* tree); //查找最小值
    RBTNode* rotate_left(RBTNode* node); //执行左旋
    RBTNode* rotate_right(RBTNode* node); //执行右旋
    RBTNode* color_flip(RBTNode* node); //执行改色
    RBTNode* fix_up(RBTNode* node); //调整
    RBTNode* move_red_left(RBTNode* tree); //移动红色到左侧
    RBTNode* move_red_right(RBTNode* tree); //移动红色到右侧
    RBTNode* delete_min(RBTNode* tree); //删除tree的最大结点
};
```
## 辅助操作函数

**find**

就是二叉搜索树的查找操作，一个深搜就好。

```c++
/* 查找 */
RBTNode* Set::find(RBTNode* node, T key) const {
  RBTNode* p = node;
  while (p != NULL) {
    if (p->key > key)
      p = p->left;
    else if (p->key < key)
      p = p->right;
    else
      return p;
  }
  return NULL;
}
```

**find_min**

***find_min 和 find_max 这两个函数我们只需要一个就可以，这里写find_min***

结合二叉搜索树数据的特性，我们只需要一直向左搜即可。实现的代码如下：

```c++
/* 查找最小数 */
RBTNode* Set::find_min(RBTNode* tree) {
  if (tree->left == NULL)
    return tree;
  else
    return find_min(tree->left);
}
```

**isred**

此外，由于我们把NULL结点也当做黑色，所以我们需要一个判断结点是否为红色的函数来辅助操作。实现的代码如下

```c++
/* 判断是否为红色 */
bool Set::isred(RBTNode* node) {
  if (node == NULL || node->color == BLACK)
    return false;
  else
    return true;
}
```

## 一些基本操作的实现

借助辅助函数，可以轻松的实现几个简单的基本操作。

不多说了，直接放代码。

```c++
/* 清空树接口 */
void Set::clear() {
  _clear(root);
  root = NULL;
  currentsize = 0;
}

/* 清空树实现 */
void Set::_clear(RBTNode* tree) {
  if (tree == NULL)
    return;
  _clear(tree->left);
  _clear(tree->right);
  delete(tree);
}

/* 返回该数据是否存在 */
int Set::count(const T& key) const {
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
  return currentsize;
}
```

## 调整树结构的操作

我们的重点，也是红黑树实现的难点在于插入和删除操作。在此之前，我们先解决几个调整树的基本操作。


### 左旋

类似于AVL树，左旋和右旋操作目的是平衡树的高度，以满足红黑树的 “所有路径上黑色结点的数目相同” 的条件。

红黑树左旋的基本操作和AVL树一样，唯一不同的是要交换k1和k2结点的颜色，左旋演示的动画效果如下：

![](https://raw.githubusercontent.com/zxc479773533/zxc479773533.github.io/master/_posts/images/LLRBTree-05.gif)

实现的代码如下：

```c++
/* 执行左旋 */
RBTNode* Set::rotate_left(RBTNode* k1) {
  RBTNode *k2 = k1->right;
  k1->right = k2->left;
  k2->left = k1;
  k2->color = k2->left->color;
  k2->left->color = RED;
  return k2;  //返回结构顶点
}
```

### 右旋

类似于左旋的算法，我们可以轻易写出右旋的算法，右旋演示的动画效果如下：

![](https://raw.githubusercontent.com/zxc479773533/zxc479773533.github.io/master/_posts/images/LLRBTree-06.gif)

实现的代码如下：

```c++
/* 执行右旋 */
RBTNode* Set::rotate_right(RBTNode* k1) {
  RBTNode *k2 = k1->left;
  k1->left = k2->right;
  k2->right = k1;
  k2->color = k2->right->color;
  k2->right->color = RED;
  return k2; //返回结构顶点
}
```

在写左旋和右旋的时候，要注意以下几点：

* 由于我们把NULL指针当做黑色的叶子结点，因此必须讨论k2的儿子是否存在的问题。
* 更改父结点的时候，要讨论父结点不存在的情况，这个时候就不用管父结点的儿子指针。
* 旋转的时候如果动了根结点，要重新指定根结点的值。

### 改色

改色的操作在红黑树中并不存在，是左倾红黑树的一个操作，目的是为了消除红色结点在右侧的情况，改色的过程如下图：

![](https://raw.githubusercontent.com/zxc479773533/zxc479773533.github.io/master/_posts/images/LLRBTree-07.png)

如果一个黑色结点X的两个儿子都是红色，那么把这两个儿子改为黑色，把X改为红色。该过程实现的代码如下：

```c++
/* 执行改色 */
RBTNode* Set::color_flip(RBTNode* k1) {
  k1->color = !k1->color;
  if (k1->left != NULL)
    k1->left->color = !k1->left->color;
  if (k1->right != NULL)
  	k1->right->color = !k1->right->color;
  return k1;
}
```

## 插入和删除

插入和删除是红黑树里的重头戏，也是最为麻烦的部分。当然，左倾红黑树的出现使得插入操作可以和调整同时进行，代码就优化了很多，读者可以在我的github里对比两者代码的区别。

首先我们依照基本结构写出调整函数：

```c++
/* 调整 */
RBTNode* Set::fix_up(RBTNode* node) {
  if (isred(node->right))
    node = rotate_left(node);
  if (node->left != NULL && isred(node->left) && isred(node->left->left))
    node = rotate_right(node);
  if (isred(node->left) && isred(node->right))
    color_flip(node);
  return node;
}
```

图示如下：

![](https://raw.githubusercontent.com/zxc479773533/zxc479773533.github.io/master/_posts/images/LLRBTree-08.png)

经过一番这样的操作，父结点以下的结构都调整好了，接下来只需要递归的考虑父结点的问题就好了。

### 插入操作insert

插入操作任然是建立在二叉搜索树的基础上的，只不过我们现在是一边调整，一边插入。

我们递归的在插入之后执行调整函数即可。实现的代码如下：

```c++
/* 插入指定的结点接口 */
void Set::insert(const T& key) {
  if (find(root, key) == NULL) {
    currentsize++;
    root = ins(root, key);
  }
  root->color = BLACK;
}

/* 插入指定的结点实现 */
RBTNode* Set::ins(RBTNode* tree, T key) {
  /* 如果是空，刚好插入 */
  if (tree == NULL)
    return new RBTNode(RED, key, NULL, NULL);
  /* 标准插入 */
  if (tree->key > key)
    tree->left = ins(tree->left, key);
  else
    tree->right = ins(tree->right, key);
  fix_up(tree);
  return tree;
}
```

### 删除操作erase

相比较插入，删除操作要稍微复杂一点。

我们首先看两个函数，move_red_left 和 move_red_right。

顾名思义，这两个函数是要移动红色结点到一边，如果某一个结点node，它的左儿子结点lchild和lchild的左儿子都是红色，那么肯定违反了红黑树的性质，所以我们要移动一个结点到右侧，但是为了保证各支路上黑色结点数量相等，所以我们要移动一共红色结点过去，这就是这两个函数的意义所在。

```c++
/* 移动红色到左侧 */
RBTNode* Set::move_red_left(RBTNode* tree) {
  /* 第一步，反色 */
  color_flip(tree); //两边一起改为红保证黑结点数量
  /* 第二步：如果如果tree的右儿子上出现了两个连续红色结点 */
  if (tree->right != NULL && isred(tree->right->left)) {
    tree->right = rotate_right(tree->right);
    tree = rotate_left(tree);
    color_flip(tree);
  }
  return tree;
}

/* 移动红色到右侧 */
RBTNode* Set::move_red_right(RBTNode* tree) {
  /* 第一步，反色 */
  color_flip(tree); //两边一起改为红保证黑结点数量
  /* 第二步：如果如果tree的左儿子上出现了两个连续红色结点 */
  if (tree->left != NULL && isred(tree->left->left)) {
    tree = rotate_right(tree);
    color_flip(tree);
  }
  return tree;
}
```

这两个函数的图解如下：
![](https://raw.githubusercontent.com/zxc479773533/zxc479773533.github.io/master/_posts/images/LLRBTree-09.png)

接下来，我们先看一个简单的问题，怎样删掉最小结点。

最小的结点一定是一直向左得到的那个结点，但是，如果删除的是黑色结点，就打破了红黑树各路上黑色结点个数相同这一性质，于是我们不停的把红色结点左移，保证我们删掉的是红色结点，最后再递归的修正上去。实现的代码如下：

```c++
/* 删除tree的最小结点 */
RBTNode* Set::delete_min(RBTNode* tree) {
  /* 删除操作 */
  if (tree->left == NULL) {
    delete(tree);
    return NULL;
  }
  /* 若子结点和孙结点有不为黑的，就不能操作，否则有两个连续红色 */
  if (!isred(tree->left) && !isred(tree->left->left))
    tree = move_red_left(tree);
  /* 向下递归 */
  tree->left = delete_min(tree->left);
  /* 修正 */
  return fix_up(tree);
}
```

最后，我们看删除操作，和二叉搜索树一样的，我们仍然需要分三种情况：

1. 如果要删除的结点是叶子，那么直接删掉；
2. 如果要删除的结点只有一个儿子，那么用它的儿子代替它；
3. 如果要删除的结点有两个儿子，用它右子树上最小的结点替换它，并在右子树上删除最小的结点，此时回到情形2。

但是和二叉搜索树不同的，这里涉及到颜色的问题，所以我们也要注意三点：

1. 向下探索的时候，向左就把红色结点左移，向右就把红色结点右移以保证黑色结点数量；
2. 如果左儿子是红色，那么在向右和相等时右旋一次，大于的时候不会这样，因为左倾红黑树右儿子不可能是红色；
3. 删除之后递归执行fix_up函数使得红黑树性质得以保持。

```c++
/* 删除指定的结点接口 */
void Set::erase(const T& key) {
  if (find(root, key) != NULL) {
    currentsize--;
    root = del(root, key);
    if (root != NULL)
      root->color = BLACK;
  }
}

/* 删除指定的结点实现 */
RBTNode* Set::del(RBTNode* tree, T key) {
  /* 若没有要删除的结点 */
  if (tree == NULL)
    return NULL;
  /* 向左 */
  if (tree->key > key) {
    /* 若左儿子与左儿子的左儿子同时为黑色 */
  	if (tree->left != NULL && !isred(tree->left) && !isred(tree->left->left))
      tree = move_red_left(tree);
    tree->left = del(tree->left, key);
  }
  /* 相等或者向右 */
  else {
  	/* 若左侧是红结点，那么两种情况都要右旋保持平衡 */
    if (isred(tree->left))
      tree = rotate_right(tree);
    /* 如果相等且右侧没有结点 */
    if (tree->key == key && tree->right == NULL) {
      RBTNode *t = tree->left;
      delete(tree);
      return t;
    }
    /* 若右儿子与右儿子的左儿子同时为黑色 */
    if (tree->right != NULL && !isred(tree->right) && !isred(tree->right->left))
      tree = move_red_right(tree);
    /* 不相等，可以向右递归了 */
    if (tree->key < key)
      tree->right = del(tree->right, key);
    /* 若相等 */
    else {
      tree->key = find_min(tree->right)->key;
      tree->right = delete_min(tree->right);
    }
  }
  tree = fix_up(tree);
  return tree;
}
```

## 完整的实现

这里我就不再贴代码了，还费时间，具体见如下github地址

[头文件](https://github.com/zxc479773533/uniquelab-2017-summer-assignment-1/blob/master/set.h)&emsp;&emsp;
[具体实现文件](https://github.com/zxc479773533/uniquelab-2017-summer-assignment-1/blob/master/set.cpp)