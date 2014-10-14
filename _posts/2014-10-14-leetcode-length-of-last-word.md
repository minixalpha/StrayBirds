---
layout: default
title: Length of Last Word 
---

## 题目 

```
Given a string s consists of upper/lower-case alphabets and empty space characters ' ', return the length of last word in the string.

If the last word does not exist, return 0.

Note: A word is defined as a character sequence consists of non-space characters only.

For example, 
Given s = "Hello World",
return 5.
```

## 解答

需要注意从后向前遍历时，应该从非空格字符开始。
