---
layout: post
title: 列表推导与嵌套循环
category: Python编程技巧
tags: [python, think]
keywords: python,think,effective
---

生成一个新列表，对a=[1,2,3,4,5,6]中每个元素求平方：[1,4,9,16,25,36]？
如果a中的数据量非常庞大，又如何做？（要求代码精炼）
答案：  

```python
[x**2 for x in a]
(x**2 for x in a)
```

承接上一道题，如果a=[[1,2,3],[4,5,6]]，求结果[1,4,9,16,25,36]？如果要返回[[1, 4, 9], [16, 25, 36]]呢？  
答案：  
```
[x**2 for row in a for x in row]
[[x**2 for x in row] for row in a]
```
