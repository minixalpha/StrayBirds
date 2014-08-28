---
layout: default
title: Longest Palindromic Substring 
---

## 题目 

```
Given a string S, find the longest palindromic substring in S.
You may assume that the maximum length of S is 1000, and there 
exists one unique longest palindromic substring.
```


## 思路 

### 方法一: 暴力法

将每个 (start, end) 对都检查一次，如果是回文，比较其长度与当前最大长度。有 O(n^2) 个这样的数对，每个检查是否为回文用
O(n) 时间，总时间为 O(n^3). 空间 O(1)


### 方法二：动态规则

暴力法中判断回文其实做了许多重复的事情。以下是动态规则表达式：

```
d[i:j] = True if d[i+1:j-1] == True and s[i] == s[j]
```

O(n^2) 时间， O(n^2) 空间。

### 方法三：Manacher  算法

算法基于回文子串中，中心对称的两点，以他们为回文中心的字符串有对称关系，将时间复杂度降低为 O(n)

```

-----------------------------
    L mi      C     i  R
    
```

C 是一个回文序列中心， L 和 R 是其左右边界。假如 P[i] 代表以 i 为中心的最大回文序列的长度。

如果以 mi 为中心的最大回文序列，其左边界小大于 L, 那么 `P[i] = P[mi]` ，因为 s[L:C-1] 与 s[C+1, R] 是左右对称的。

如果以 mi 为中心的最大回文序列，其左边界小于 L, 那么 不能确定 P[i] 与 P[mi] 是否相等，但 P[i] 右边界至少可到达 R. 此时
以 i 为中心，向左右扩张，一直到左右边界字符不相等。即可找到以 i 为中心的最大回文字符串。如果 i 右边界大于 R. 此时，中心
C 变为 i, R 变为 i 右边界。开始下一循环。
