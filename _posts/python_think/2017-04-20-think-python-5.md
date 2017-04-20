---
layout: post
title: 使用zip同时迭代多个可迭代对象
category: Python编程技巧
tags: [python, think]
keywords: python,think,effective
---

求字符串列表中长度最大项 ['Cecilia','Lisa','Maria']（写出3种以上方法）
答案：这里提供两种方法，不是最简，但有参考价值。

```python
names = ['Cecilia','Lisa','Maria']
max = 0
longest_name = None
for name,count in zip(names,[len(n) for n in names]):
    if count > max:
        longest_name = name
        max = count

print(max,longest_name)

letters = [len(n) for n in names]
for i,name in enumerate(names):
    count = letters[i]
    if count > max:
        longest_name = name
        max = count
print(max,longest_name)
```

