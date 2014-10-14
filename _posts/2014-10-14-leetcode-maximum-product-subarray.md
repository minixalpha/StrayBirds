---
layout: default
title: Maximum Product Subarray
---

## 题目 

```
Find the contiguous subarray within an array (containing at least one number) which has the largest product.

For example, given the array [2,3,-2,4],
the contiguous subarray [2,3] has the largest product = 6.
```

## 解答

需要注意两点：

1. 如果 dp[i] 代表以 i 为结尾的数组最大乘积，dp[i+1] 并不等于 max(A[i+1], A[i+1] * dp[i])，原因在于可能出现 A[i+1] 为负数的情况，
例如 [-2, 3, -4]，dp[0] = -2, dp[1] = 3, dp[-4] 等于 24, 而不是 -4。所以实际上，最优表达式应该是：

```
dpMax[i+1] = max(A[i+1], A[i+1] * dpMax[i], A[i+1] * dpMin[i])
dpMin[i+1] = min(A[i+1], A[i+1] * dpMax[i], A[i+1] * dpMin[i])
```

2. 根据以上表达式，发现每次迭代只用到上一次的结果，所以 dpMax[], dpMin[], 可以变成 curMax, curMin，进一步降低空间复杂度。
