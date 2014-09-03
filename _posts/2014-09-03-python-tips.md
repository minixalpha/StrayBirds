--
layout: default
title: Python 中需要注意的问题
---


* split

split() 和 split(' ') 含义是不一样的，没有参数时，分割符号连续时会被合并成一个分割符号，有参数时就不会。

```
>>> '  '.split(' ')
['', '', '']
>>> '  '.split()
[]
>>> 
```
