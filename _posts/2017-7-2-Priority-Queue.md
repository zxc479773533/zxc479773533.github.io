---
layout: post
title: 优先队列的C++实现
tags:
- C/C++
categories: Algorithms
---
Unique Studio Lab 第一期任务，在完成之后整理一下经验。PS:第一次接触C++，花了大半天速成C++，部分代码可能尚存在C语言的习惯。

## 定义和存储分析

优先队列：优先队列是计算机科学中的一类抽象数据类型。优先队列中的每个元素都有各自的优先级，优先级最高的元素最先得到服务；优先级相同的元素按照其在优先队列中的顺序得到服务。在接下来的讨论中，我们用该结点储存的数据来作为该结点的优先级，数据越大，表示优先级越高。

为了方便存储，优先队列往往用堆（即一种特殊的二叉搜索树）来实现，这样实现又两个好处：

* 每一个结点下方的数据都小于该结点
* 根结点拥有最高的优先级
* 具有O(log n)时间复杂度的插入元素性能，O(n)的初始化构造的时间复杂度。

这样的存储使得我们可以方便的得到当前优先级最高的结点，实现优先队列的首要目的。

## 基本方法与存储分析

我们讲该优先队列定义为一个类`class Priority_queue`，则该类应该有如下的方法

* empty():判断队列是否为空。
* size(): 返回队列中数据的个数
* top(): 返回队列的头部数据
* push(const T&): 在队列尾部增加一个数据
* pop(): 队列头部的数据出队

我们从上到下将每个元素从1开始标号，如下图所示

![](https://raw.githubusercontent.com/zxc479773533/zxc479773533.github.io/master/_posts/images/Priority-Queue-01.png)

观察可以发现，每个结点的子结点都是该结点的两倍和两倍加一，每个结点的父结点都是由该结点除以二取整得到。所以，如果我们用数组实现的话，就可以利用下标的二倍关系方便的得到父结点或者子结点。结合C++的特性，我们用
vector来实现该优先队列。这样就不必担心该优先队列数据上限的问题。

为了讨论的方便，优先队列的数据类型使用double，该类实现的代码如下：
```c++
class Priority_queue {
  public:
          
    bool empty() const; //判断队列是否为空
    size_t size() const; //返回队列中数据的个数
    const T& top() const; //返回队列的头部数据
    void push(const T& val); //在队列尾部增加一个数据
    void pop(); //队列头部的数据出队
  private:
    vector<T> PriQueue; //存储类型vector
    int currentsize = 0; //属性：当前队列大小
};

```
需要说明的是，这里我们使用了改类的一个属性currentsize来储存优先队列的长度，这样就省去了之后反复使用vector的size，push_back等方法。

## 算法实现

### empty()

队列是否为空，我们直接根据currentsize来判定。

```c++
/* 判断队列是否为空,若为空，返回true */
bool Priority_queue::empty() const {
	if (currentsize == 0)
    return true;
  else
    return false;
}
```

### size()

实现size直接查看currentsize属性即可，这样省去了再搜索遍历的麻烦。

```c++
/* 返回队列中数据的个数 */
size_t Priority_queue::size() const {
  return currentsize;
}
```

### top()

直接返回队列的第一个结点即可，代码实现如下：

```c++
/* 返回队列的头部数据 */
const T& Priority_queue::top() const {
  return PriQueue[1];
}
```

### push(const T&)

增加数据我们采用上滤法，即我们把新的结点插入到vector的最后，之后重复如下操作：

比较该结点和其父结点数据的大小，若该结点比其父结点大，则交换这两个结点。

直至不满足上述的情况时终止。不难理解这样的操作不会破坏原二叉树的数据大小关系。只不过这里采用最后再放入新插入的值，避免了再使用一个交换函数，减少复杂度。代码实现如下：

```c++
/* 在队列尾部增加一个数据 */
void Priority_queue::push(const T& val) {
  if (PriQueue.size() == 0)
    PriQueue.resize(1);
  if (currentsize == PriQueue.size() - 1) //重新分配空间
    PriQueue.resize(PriQueue.size() * 2);
  int k = ++currentsize; //k是新的空位置
  for ( ; k > 1 && val > PriQueue[k / 2]; k /= 2)
    PriQueue[k] = PriQueue[k / 2]; //上滤，把空位置上移
  PriQueue[k] = val; //插入新值
}
```

### pop()

获取当前队首的结点当然简单，直接返回根结点即可，但是问题在于如果删除根结点之后，剩下的结点如何移动。

类似于插入的方法，这里我们使用下滤法，即从根结点开始，重复如下操作：

比较该结点子结点的两个数据的大小，将两者中大的和该结点交换。

直至根节点变成树叶为止。接下来交换根节点和vector的最后一个数据，并删除vector尾部的数据。这里实现的方式和插入类似，不再使用交换函数。代码实现如下：

```c++
/* 队列头部的数据出队 */
void Priority_queue::pop() {
	if (empty())
    cout << "This priority queue is empty!" << endl;
  else {
    int i, child; //child是i的子结点序号
    T temp = PriQueue[currentsize--]; //将最后一个位置的值储存
    for (i = 1; i * 2 <= currentsize; i = child) {
      child = i * 2;
      if (child != currentsize && PriQueue[child] < PriQueue[child + 1]) //选择大的子结点
        child++;
      if (temp < PriQueue[child]) //下虑，把空位置下移
        PriQueue[i] = PriQueue[child];
      else
        break;
    }
    PriQueue[i] = temp;
  }
}
```

## 最终的头文件和实现

最终完善的头文件如下：

```c++
/* 说明，该优先队列以数据表示优先级，数据越大，表示优先级越高 */
#include <iostream>
#include <algorithm>
#include <vector>

using T = double; //数据类型，本次任务要求使用double
using std::vector; //存储类型使用vector
using std::cout;
using std::endl; //测试用输出

class Priority_queue {
  public:
          
    bool empty() const; //判断队列是否为空
    size_t size() const; //返回队列中数据的个数
    const T& top() const; //返回队列的头部数据
    void push(const T& val); //在队列尾部增加一个数据
    void pop(); //队列头部的数据出队
  private:
    vector<T> PriQueue; //存储类型vector
    int currentsize = 0; //属性：当前队列大小
};
```

具体实现的cpp文件如下:

```c++
#include "priority_queue.h"

/* 判断队列是否为空,若为空，返回true */
bool Priority_queue::empty() const {
  if (currentsize == 0)
    return true;
  else
    return false;
}

/* 返回队列中数据的个数 */
size_t Priority_queue::size() const {
  return currentsize;
}

/* 返回队列的头部数据 */
const T& Priority_queue::top() const {
  return PriQueue[1];
}

/* 在队列尾部增加一个数据 */
void Priority_queue::push(const T& val) {
  if (PriQueue.size() == 0)
    PriQueue.resize(1);
  if (currentsize == PriQueue.size() - 1) //重新分配空间
    PriQueue.resize(PriQueue.size() * 2);
  int k = ++currentsize; //k是新的空位置
  for ( ; k > 1 && val > PriQueue[k / 2]; k /= 2)
    PriQueue[k] = PriQueue[k / 2]; //上滤，把空位置上移
  PriQueue[k] = val; //插入新值
}

/* 队列头部的数据出队 */
void Priority_queue::pop() {
  if (empty())
    cout << "This priority queue is empty!" << endl;
  else {
    int i, child; //child是i的子结点序号
    T temp = PriQueue[currentsize--]; //将最后一个位置的值储存
    for (i = 1; i * 2 <= currentsize; i = child) {
      child = i * 2;
      if (child != currentsize && PriQueue[child] < PriQueue[child + 1]) //选择大的子结点
        child++;
      if (temp < PriQueue[child]) //下虑，把空位置下移
        PriQueue[i] = PriQueue[child];
      else
        break;
    }
    PriQueue[i] = temp;
  }
}
```
