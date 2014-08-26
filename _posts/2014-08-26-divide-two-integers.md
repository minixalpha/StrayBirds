---
layout: default
title: LeetCode: Divide Two Integers
---


## 题目

```
Divide two integers without using multiplication, division and mod operator.
```


## 思路

采用二进制除法的方法，每次从头部取一个二进制位，如果当前除数大于被除数，则结果的这一位设置为１，
前面取的二进制位组成的数要减去被除数。

最重要的一点是：如果当前除数小于被除数，继续取下一个二进制位，此时如果除数大于被除数，则一定商１，而不会是二，因为每
次取一个二进制位，都相当于将原来的除数翻倍再加上新二进制位，所以绝对不会商２。这就保证了余数一定是小于被除数的。和
整数除法相一致。
