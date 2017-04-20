---
layout: post
title: 如何翻转一个字符串
category: Python编程技巧
tags: [python, think]
keywords: python,think,effective
---

如何翻转一个字符串？

```python
s=[::-1] #pythone2.3以上
```

Python3支持反转中文，但不能反转encode后的中文。


```
>>> us='你好忙呀'
>>> us
'你好忙呀'
>>> us[::-1]
'呀忙好你'
>>> eus = us.encode('utf-8')
>>> eus
b'\xe4\xbd\xa0\xe5\xa5\xbd\xe5\xbf\x99\xe5\x91\x80'
>>> eus[::-1]
b'\x80\x91\xe5\x99\xbf\xe5\xbd\xa5\xe5\xa0\xbd\xe4'
>>> eus[::-1].decode('utf-8')
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
UnicodeDecodeError: 'utf-8' codec can't decode byte 0x80 in position 0: invalid start byte
```


为什么Python2不支持反正中文呢？  
原因在于2和3的str类型本质不一样。  
3中的str相当于2中的unicode，  
2中的str相当于3中的bytes。

