---
layout: default
title: Maximal Rectangle 
---

## 题目 

```
Given a 2D binary matrix filled with 0's and 1's, find the largest rectangle containing all ones and return its area.
```

## 思路 

先解决子问题， 即 `matrix[0:i]` 中最大矩形，这可以看成以第 `i` 行为底边，由1组成柱体，求最大矩形面积，问题即转化成
[Largest Rectangle in Histogram](https://oj.leetcode.com/problems/largest-rectangle-in-histogram/) 问题。我们只需要维护一个
列表 H, H[i] 代表以当前行为底边，第 i 列的高度。 
