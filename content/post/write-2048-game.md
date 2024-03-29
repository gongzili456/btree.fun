---
title: "动手写游戏--2048"
date: "2014-11-26"
toc: false # Enable Table of Contents for specific page
categories:
  - "Blog"
  - "Project"
tags:
  - "java"
  - "game"
  - "programming"
---

2048 游戏大家都很熟悉了，今天来练练手。 ![](images/b03d53e397c9f1e46ca0eb9a89e00-265x300.png)
<!--more-->

## 规则:

- 在一个 4\*4 的方格中
- 同一行数字向一端移动，且相邻两个相同的数字相加合并，
- 再在随机位置产生一个随机数字 2 或 4，
- 当没有空余格（输）或最大值为 2048（胜）时游戏结束，

## 设计

该游戏最关键的逻辑是如何将每一行移到同一方向，且将相邻等值数字相加合并。

可以把所有方块用二维数组存储，假设我们拿类`Cell`表示每个方块，那我们的数据结构就是

```
Cell[][] cells = new Cell[4][4]
```

![](images/5da1ba76d21cf10a430f454344b7d-300x261.png)

由于游戏涉及到上下左右四个方向，难点就在于如何获取横向和纵向的每一排数，横向当然好取，可是纵向就麻烦了，这还牵扯到移动方向的问题，就更复杂了，

如果能有一个方式让我们用相同的方式就能取到横向或纵向的任意一行就好了，方法当然是有的。这就涉及到数学中的三角函数了。大家可以自行 Google“三角函数和差公式”。 也可直接进入[我的参考](http://www.cnblogs.com/ywxgod/archive/2010/08/06/1793609.html)

![](images/defa934db419c6aef2f1ba3277ef1-300x188.png)

如上图，经过旋转，我们很容易能根据 X 的值就能得到任意行数字了，当我们对该行操作完成后在经过旋转就恢复到原来样子了，注意整个操作是同一方向旋转 360 度。

## 总结

解决了获取任意行数字的问题，剩下的就简单了。也就不多说了，具体可以看我的代码：[Github](https://github.com/gongzili456/2048)

![](images/1c8bfddabe33b276faaba401d14f5-265x300.png)
