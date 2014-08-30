---
layout: default
title: Surrounded Regions
---

## 题目

```
Given a 2D board containing 'X' and 'O', capture all regions surrounded by 'X'.

A region is captured by flipping all 'O's into 'X's in that surrounded region.

For example,
X X X X
X O O X
X X O X
X O X X
After running your function, the board should be:

X X X X
X X X X
X X X X
X O X X
```

## 思路

从边界的 'O' 开始，广度优先搜索，可以搜索到的 'O'，就保留下来。

注意：

* 不要用深度优先搜索，会递归深度过深
* 记录是否访问过
