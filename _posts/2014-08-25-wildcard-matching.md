---
layout: default
title: Wildcard Matching 
---

## 题目

```
'?' Matches any single character.
'*' Matches any sequence of characters (including the empty sequence).

The matching should cover the entire input string (not partial).

The function prototype should be:
bool isMatch(const char *s, const char *p)

Some examples:
isMatch("aa","a") → false
isMatch("aa","aa") → true
isMatch("aaa","aa") → false
isMatch("aa", "*") → true
isMatch("aa", "a*") → true
isMatch("ab", "?*") → true
isMatch("aab", "c*a*b") → false
```


## 思路

关键点:

如果 `f1f2 AB` 与 `*f1f2 *C`　不匹配，那么即使 pattern 的第一个 `*` 将 `f1` 吃掉，　`f2A B` 与 `f1f2 *C`　仍然不可能匹配，因为如果 `B` 可以与 `*C` 匹配，那么 `AB` 一定可以与 `*C` 匹配。所以就不用考虑 `*` 将 `f1` 吃掉的情况了。

