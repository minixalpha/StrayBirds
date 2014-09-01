---
layout: default
title: Word Ladder
---


## 题目 

```
Given two words (start and end), and a dictionary, find the length of shortest transformation sequence 
from start to end, such that:

Only one letter can be changed at a time
Each intermediate word must exist in the dictionary
For example,

Given:
start = "hit"
end = "cog"
dict = ["hot","dot","dog","lot","log"]
As one shortest transformation is "hit" -> "hot" -> "dot" -> "dog" -> "cog",
return its length 5.

Note:
Return 0 if there is no such transformation sequence.
All words have the same length.
All words contain only lowercase alphabetic characters.
```

## 思路 


```python
class Solution:
    # @param start, a string
    # @param end, a string
    # @param dict, a set of string
    # @return an integer
    def ladderLength(self, start, end, dict):
        if start == end: return 0
        dict.add(end)

        visit = set([start])
        queue = [(start, 1)]
        chars = map(chr, range(ord('a'), ord('z')+1))

        while queue:
            head, layer = queue.pop(0)
            if head == end: return layer
            i = 0
            while i < len(head):
                for e in chars:
                    nb = head[:i] + e + head[i+1:]
                    if nb == end: return layer + 1
                    if nb in dict and not nb in visit:
                        queue.append((nb, layer + 1))
                        visit.add(nb)
                i += 1
        return 0
```

题目的基本思路就是广度优先搜索，但有几个关键的地方需要注意:

* 设置元素是否已经访问过，应该在元素入队时，而不是出队时。如果在出队时设置，会导致元素已经入队，但还没出队，此时又访问到
这个元素，就会又把这个元素入队，队列中可能会有大量重复元素。

* 元素邻近节点，按照当前需要计算，不要一开始全部计算好

* 元素邻近节点的计算，不要通过遍历单词表的形式，这样会消耗 O(n) 的时间， n 为单词表长度。而要通过替换单个字符，然后通过
哈希表查看是否存在于单词表中，一共 26 * L 种候选邻近节点，每次判断是否在单词表中，时间 O(1), 总时间 O(L). 

* 判断是否在单词表中，用 set ，不要用 list, 保证时间复杂度 O(1), 而不是 O(n)

* Python 实现时，出队用 queue.pop, 不要用 queue = queue[1:]
