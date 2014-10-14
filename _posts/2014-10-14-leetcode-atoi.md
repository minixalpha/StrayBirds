---
layout: default
title: String To Integer
---

## 题目

```
Implement atoi to convert a string to an integer
```

## 解答

关键的地方有

* 从第一个非空格字符开始
* 可能有正负号，但只能有一个
* 从第一个数字开始，到第一个非数字为止
* 如果大于整数最大值，返回整数最大值，最小值也是
