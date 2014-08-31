---
layout: default
title: Median of Two Sorted Arrays 
---


## 题目

```
There are two sorted arrays A and B of size m and n respectively. Find 
the median of the two sorted arrays. The overall run time complexity should be O(log (m+n)).
```

## 思路 

把问题转化为在两个已经排序的数组中，查找第 k 个数的问题。这样，如果总数是奇数，则中位数为第 `(la + lb) /2 + 1` 个数。
如果总数是偶数，中位数为第 `(la + lb) / 2` 个数与第 `(la + lb) / 2 + 1`  个数的平均值。


```python
class Solution:
    def _findk(self, A,  B, k):
        if len(A) > len(B): return self._findk(B, A, k)
        if len(A) == 0: return B[k-1]
        if k == 1: return min(A[0], B[0])

        ia = min(k / 2, len(A))
        ib = k - ia
        
        if A[ia - 1] == B[ib - 1]:
            return A[ia - 1]
        elif A[ia - 1] < B[ib - 1]:
            return self._findk(A[ia:], B, k-ia)
        else:
            return self._findk(A, B[ib:], k-ib)

    # @return a float
    def findMedianSortedArrays(self, A, B):
        total = len(A) + len(B)
        if total % 2 != 0:
            return self._findk(A, B, total / 2 + 1)
        else:
            return (self._findk(A, B, total / 2) 
                  + self._findk(A, B, total / 2 + 1)) / 2.0

```


注意几个地方，一是保证调用  `_findk` 时， A 的数目小于 B 的数目，这样有两个好处：

* len(A) == 0 只用判断一次
* ia - min(k / 2, len(A)) 与 b = k - ia 保证 A[ia-1], B[ib-1] 是有效的，否则 A 的数目很大， B 的数目很少，会导致 k 很大， ib
就会超出 B 的范围。
