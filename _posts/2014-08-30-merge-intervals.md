---
layout: default
title: Merge Intervals
---


## 题目 

```
Given a collection of intervals, merge all overlapping intervals.

For example,
Given [1,3],[2,6],[8,10],[15,18],
return [1,6],[8,10],[15,18].
```

## 思路

先按 `start` 排序，逐步合并。

注意：

* 合并的依据是 `当前合并后的 end`，而不是 `当前结点的 end`， 与下一结点 `start` 比较
* 比较时是 `<=`, 不是 `<`，注意此边界
