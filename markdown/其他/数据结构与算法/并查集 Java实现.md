# 并查集 Java实现

> [算法题目 力扣 <连通网络的操作次数>](https://leetcode-cn.com/problems/number-of-operations-to-make-network-connected/submissions/)
[知乎博客-算法学习笔记(1) : 并查集](https://zhuanlan.zhihu.com/p/93647900/)

## 前言
最近在学习中遇到这样一道题(如下所示), 在评论区一片"并查集"飘过, "并查集"是什么? 这不是典型的"亲戚"问题吗? 大学时就学过, 运用"深度优先遍历"方法去解就好了. 那 "并查集" 是什么?
> 连通网络的操作次数
> 用以太网线缆将 n 台计算机连接成一个网络，计算机的编号从 0 到 n-1。线缆用 connections 表示，其中 connections[i] = [a, b] 连接了计算机 a 和 b。
> 网络中的任何一台计算机都可以通过网络直接或者间接访问同一个网络中其他任意一台计算机。
> 给你这个计算机网络的初始布线 connections，你可以拔开任意两台直连计算机之间的线缆，并用它连接一对未直连的计算机。请你计算并返回使所有计算机都连通所需的最少操作次数。如果不可能，则返回 -1 。
> 示例 1：
> ![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240310215249.png#id=mtkS1&originHeight=224&originWidth=759&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
> 输入：n = 4, connections = [[0,1],[0,2],[1,2]]
输出：1
解释：拔下计算机 1 和 2 之间的线缆，并将它插到计算机 1 和 3 上。
> 示例 2：
> ![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240310215317.png#id=o799j&originHeight=226&originWidth=878&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
> 输入：n = 6, connections = [[0,1],[0,2],[0,3],[1,2],[1,3]]
输出：2

## 简介
并查集主要用于解决一些元素分组的问题。它管理一系列不相交的集合，并支持两种操作：

- 合并（Union）：把两个不相交的集合合并为一个集合。
- 查询（Find）：查询两个元素是否在同一个集合中。

当然，这样的定义未免太过学术化，看完后恐怕不太能理解它具体有什么用。所以我们先来看看并查集最直接的一个应用场景：亲戚问题。
> 题目背景
若某个家族人员过于庞大，要判断两个是否是亲戚，确实还很不容易，现在给出某个亲戚关系图，求任意给出的两个人是否具有亲戚关系。
题目描述
规定：x和y是亲戚，y和z是亲戚，那么x和z也是亲戚。如果x,y是亲戚，那么x的亲戚都是y的亲戚，y的亲戚也都是x的亲戚。
输入格式
第一行：三个整数n,m,p，（n<=5000,m<=5000,p<=5000），分别表示有n个人，m个亲戚关系，询问p对亲戚关系。
以下m行：每行两个数Mi，Mj，1<=Mi，Mj<=N，表示Mi和Mj具有亲戚关系。
接下来p行：每行两个数Pi，Pj，询问Pi和Pj是否具有亲戚关系。
输出格式
P行，每行一个’Yes’或’No’。表示第i个询问的答案为“具有”或“不具有”亲戚关系。

这其实是一个很有现实意义的问题。我们可以建立模型，把所有人划分到若干个不相交的集合中，每个集合里的人彼此是亲戚。为了判断两个人是否为亲戚，只需看它们是否属于同一个集合即可。因此，这里就可以考虑用并查集进行维护了。
## 并查集的引入
并查集的重要思想在于，用集合中的一个元素代表集合。我曾看过一个有趣的比喻，把集合比喻成帮派，而代表元素则是帮主。接下来我们利用这个比喻，看看并查集是如何运作的。
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240310215404.png#id=Vs4Ro&originHeight=396&originWidth=580&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
最开始，所有大侠各自为战。他们各自的帮主自然就是自己。（对于只有一个元素的集合，代表元素自然是唯一的那个元素）
现在1号和3号比武，假设1号赢了（这里具体谁赢暂时不重要），那么3号就认1号作帮主（合并1号和3号所在的集合，1号为代表元素）。
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240310215449.png#id=UCZym&originHeight=396&originWidth=580&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
现在2号想和3号比武（合并3号和2号所在的集合），但3号表示，别跟我打，让我帮主来收拾你（合并代表元素）。不妨设这次又是1号赢了，那么2号也认1号做帮主。
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240310215529.png#id=lJ83V&originHeight=492&originWidth=720&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
现在我们假设4、5、6号也进行了一番帮派合并，江湖局势变成下面这样：
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240310215619.png#id=EdeZe&originHeight=383&originWidth=580&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
现在假设2号想与6号比，跟刚刚说的一样，喊帮主1号和4号出来打一架（帮主真辛苦啊）。1号胜利后，4号认1号为帮主，当然他的手下也都是跟着投降了。
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240310215714.png#id=qABnE&originHeight=381&originWidth=580&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
好了，比喻结束了。如果你有一点图论基础，相信你已经觉察到，这是一个树状的结构，要寻找集合的代表元素，只需要一层一层往上访问父节点（图中箭头所指的圆），直达树的根节点（图中橙色的圆）即可。根节点的父节点是它自己。我们可以直接把它画成一棵树：
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240310215802.png#id=HTtPd&originHeight=469&originWidth=390&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
用这种方法，我们可以写出最简单版本的并查集代码。
## 最简单的并查集
### 存储结构
```java
class UnionFind {
    int[] parent;
}
```
### 初始化
假如有编号为1, 2, 3, ..., n的n个元素，我们用一个数组parent[]来存储每个元素的父节点（因为每个元素有且只有一个父节点，所以这是可行的）。一开始，我们先将它们的父节点设为自己。
```java
public UnionFind(int n) {
    parent = new int[n];
    for (int i = 0; i < n; i++) {
        parent[i] = i;
    }
}
```
### 查询
我们用递归的写法实现对代表元素的查询：一层一层访问父节点，直至根节点（根节点的标志就是父节点是本身）。要判断两个元素是否属于同一个集合，只需要看它们的根节点是否相同即可。
```java
public int find(int x) {
    if (parent[x] == x) {
        return x;
    } else {
        return find(parent[x]);
    }
}
```
### 合并
合并操作也是很简单的，先找到两个集合的代表元素，然后将前者的父节点设为后者即可。当然也可以将后者的父节点设为前者，这里暂时不重要。本文末尾会给出一个更合理的比较方法。
```java
public void merge(int x, int y) {
    int i = find(x);
    int j = find(y);
    if (i == j) {
        return;
    }
    parent[i] = j;
}
```
一个简单的并查集功能就完成了, 试一下
```java
/**
 * 判断节点x和节点y是不是有联系
 *
 * @param n           节点数量
 * @param connections 节点间关系 : 例子 connections[i] = [a, b] 表示 a 和 b 是亲戚。
 * @param x           节点x
 * @param y           节点y
 * @return x和节点y是不是有联系
 */
public boolean makeConnected(int n, int[][] connections, int x, int y) {
    UnionFind uf = new UnionFind(n);
    for (int[] conn : connections) {
        uf.merge(conn[0], conn[1]);
    }
    return uf.find(x) == uf.find(y);
}
```
## 改进
### 路径压缩
最简单的并查集效率是比较低的。例如，来看下面这个场景：
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240310215109.png#id=N2Bz0&originHeight=273&originWidth=295&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
现在我们要merge(2,3)，于是从2找到1，parent[1]=3，于是变成了这样：
![](https://gitee.com/eden2f/ImageHosting/raw/master/imgs/20210418230754.png#id=cCKqt&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240310215918.png#id=Chq4h&originHeight=404&originWidth=133&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
然后我们又找来一个元素4，并需要执行merge(2,4)：
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240310220210.png#id=bDz1U&originHeight=404&originWidth=298&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
从2找到1，再找到3，然后fa[3]=4，于是变成了这样：
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240310220254.png#id=VnYxa&originHeight=531&originWidth=133&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
大家应该有感觉了，这样可能会形成一条长长的链，随着链越来越长，我们想要从底部找到根节点会变得越来越难。
怎么解决呢？我们可以使用路径压缩的方法。既然我们只关心一个元素对应的根节点，那我们希望每个元素到根节点的路径尽可能短，最好只需要一步，像这样：
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240310220505.png#id=sXFPO&originHeight=531&originWidth=157&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
其实这说来也很好实现。只要我们在查询的过程中，把沿途的每个节点的父节点都设为根节点即可。下一次再查询时，我们就可以省很多事。这用递归的写法很容易实现：
#### 合并（路径压缩）
```java
public int find(int x) {
    if (parent[x] == x) {
        return x;
    } else {
        parent[x] = find(parent[x]);
        return parent[x];
    }
}
```
### 按秩合并
有些人可能有一个误解，以为路径压缩优化后，并查集始终都是一个菊花图（只有两层的树的俗称）。但其实，由于路径压缩只在查询时进行，也只压缩一条路径，所以并查集最终的结构仍然可能是比较复杂的。例如，现在我们有一棵较复杂的树需要与一个单元素的集合合并：
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240310220559.png#id=DtWFq&originHeight=557&originWidth=539&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
假如这时我们要merge(7,8)，如果我们可以选择的话，是把7的父节点设为8好，还是把8的父节点设为7好呢？
当然是后者。因为如果把7的父节点设为8，会使树的深度（树中最长链的长度）加深，原来的树中每个元素到根节点的距离都变长了，之后我们寻找根节点的路径也就会相应变长。虽然我们有路径压缩，但路径压缩也是会消耗时间的。而把8的父节点设为7，则不会有这个问题，因为它没有影响到不相关的节点。
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240310220639.png#id=lh0dX&originHeight=573&originWidth=720&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
这启发我们：我们应该把简单的树往复杂的树上合并，而不是相反。因为这样合并后，到根节点距离变长的节点个数比较少。
我们用一个数组rank[]记录每个根节点对应的树的深度（如果不是根节点，其rank相当于以它作为根节点的子树的深度）。一开始，把所有元素的rank（秩）设为1。合并时比较两个根节点，把rank较小者往较大者上合并。路径压缩和按秩合并如果一起使用，时间复杂度接近 [图片上传失败...(image-98c7ec-1611629139843)] ，但是很可能会破坏rank的准确性。
#### 存储结构
```java
class UnionFind {
    int[] parent;
    int[] rank;
}
```
#### 初始化（按秩合并）
```java
public UnionFind(int n) {
    parent = new int[n];
    for (int i = 0; i < n; i++) {
        parent[i] = i;
    }
    rank = new int[n];
    Arrays.fill(rank, 1);
}
```
#### 合并（按秩合并）
```java
public void merge(int x, int y) {
    int i = find(x);
    int j = find(y);
    if (i == j) {
        return;
    }
    if (rank[i] > rank[j]) {
        parent[j] = i;
    } else {
        parent[i] = j;
    }
    if (rank[i] == rank[j]) {
        rank[i]++;
    }
}
```
为什么深度相同，新的根节点深度要+1？如下图，我们有两个深度均为2的树，现在要merge(2,5)：
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240310220815.png#id=TG1qZ&originHeight=283&originWidth=400&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
这里把2的父节点设为5，或者把5的父节点设为2，其实没有太大区别。我们选择前者，于是变成这样：
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240310220851.png#id=eve0a&originHeight=414&originWidth=330&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
显然树的深度增加了1。另一种合并方式同样会让树的深度+1。
## 解题 连通网络的操作次数
```java
public class Solution {


    public int makeConnected(int n, int[][] connections) {
        if (connections.length < n - 1) {
            return -1;
        }

        UnionFind uf = new UnionFind(n);
        for (int[] conn : connections) {
            uf.merge(conn[0], conn[1]);
        }

        return uf.outlierNum - 1;
    }
}

class UnionFind {

    int[] parent;
    int[] rank;
    int outlierNum;

    public UnionFind(int n) {
        parent = new int[n];
        for (int i = 0; i < n; i++) {
            parent[i] = i;
        }
        rank = new int[n];
        Arrays.fill(rank, 1);
        outlierNum = n;
    }

    public int find(int x) {
        if (parent[x] == x) {
            return x;
        } else {
            parent[x] = find(parent[x]);
            return parent[x];
        }
    }

    public void merge(int x, int y) {
        int i = find(x);
        int j = find(y);
        if (i == j) {
            return;
        }
        if (rank[i] > rank[j]) {
            parent[j] = i;
        } else {
            parent[i] = j;
        }
        if (rank[i] == rank[j]) {
            rank[i]++;
        }
        outlierNum--;
    }

}
```
