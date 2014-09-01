---
layout: default
title: Substring with Concatenation of All Words
---

## 题目

```
You are given a string, S, and a list of words, L, that are all of the same length. Find 
all starting indices of substring(s) in S that is a concatenation of each word in L exactly once
and without any intervening characters.

For example, given:
S: "barfoothefoobarman"
L: ["foo", "bar"]

You should return the indices: [0,9].
(order does not matter).
```


## 思路 


## 方法一 暴力法

最简单的方法是暴力法，但暴力也是有技巧的，没必要把所有单词数目统计完才比较，统计过程中，如果发现单词不在 L 中，或者数目
超过 L 中单词的数目，直接跳过即可。
