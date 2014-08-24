---
layout: default
title: LeetCode:Scramble String
---

* 题目

```
Given a string s1, we may represent it as a binary tree by partitioning it to two non-empty substrings recursively.

Below is one possible representation of s1 = "great":

    great
   /    \
  gr    eat
 / \    /  \
g   r  e   at
           / \
          a   t
To scramble the string, we may choose any non-leaf node and swap its two children.

For example, if we choose the node "gr" and swap its two children, it produces a scrambled string "rgeat".

    rgeat
   /    \
  rg    eat
 / \    /  \
r   g  e   at
           / \
          a   t
We say that "rgeat" is a scrambled string of "great".

Similarly, if we continue to swap the children of nodes "eat" and "at", it produces a scrambled string "rgtae".

    rgtae
   /    \
  rg    tae
 / \    /  \
r   g  ta  e
       / \
      t   a
We say that "rgtae" is a scrambled string of "great".

Given two strings s1 and s2 of the same length, determine if s2 is a scrambled string of s1.
```

* 思路

使用递归解决，如何划分子问题?

S, T

S可能在任何一个地方分成两个子树，假如是　`i`, 则子树为 `S[:i], S[i:]`，
那么这一步，可能 `scrable`，也可能不　`scrable`。

* 如果不　`scrable`，子问题为　(S[:i], T[:i]), (S[i:], T[:i])
* 如果　`scrable`，则　S 前　`i` 个对应　T 后　`i` 个，子问题为　(S[:i], T[len-i:]), (S[i:], T[:len-i])


