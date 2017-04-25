---
layout: post
title: 迭代器作为参数时注意事项
category: Python编程技巧
tags: [python, think]
keywords: python,think,effective
---
### 问题
输入一组数字，输出每个数字在这组数字中的百分比。  

```python
def normalize(numbers):
    total = sum(numbers)
    result = []
    for value in numbers:
        percent = 100 * value / total
        result.append(percent)

    return result

nums = [15,35,80]
percents = normalize(nums)
print(percents) 
#结果：[11.538461538461538, 26.923076923076923, 61.53846153846154]
```
numbers在数据量大时就需要考虑用生成器代替了。  
比如：数据来源于文件

```py
def read_data(data_path):
    with open(data_path) as f:
        for line in f:
            yield int(line)


it = read_data('data.txt')
percents = normalize(it)
print(percents)     #结果为[]
```
此时，如果直接将迭代器作为参数传入，将得到非预期的结果。  
原因：  
迭代器只能产生一轮结果。  
正常情况下，迭代器通过next()获取下一个值，当迭代器里的值耗尽时，会抛出异常StopIteration（for循环会自动处理）。  
所以，当执行玩sum(numbers)是，it内的值已经被耗尽。for循环中取不到任何有效值，导致结果为空。  
解决：  
方案1:  

```py
percents = normalize(lambda: read_data(path))
```
方案2：
### 使用迭代器容器类

```py
class ReadData(object):
    def __init__(self, data_path):
        self.data_path = data_path

    def __iter__(self):
        with open(self.data_path) as f:
            for line in f:
                yield int(line)

it = ReadData('data.txt')
percents = normalize(it)
print(percents)
```
原理：
先举个例子：

```py
for x in foo
```
执行上述代码时，python实际上会调用iter(foo)。而内置的iter函数会调用foo.__iter__这个魔术方法。该方法必须返回迭代器对象，而这个迭代器对象本省实现里__next__的魔术方法。此后，for循环会在迭代器对象上反复调用内置的next函数，直到迭代器耗尽抛出StopIteration异常。  
所以，在类中只需要实现__iter__方法就能实现一个可迭代的容器类。在sum()和for调用这个类时，都会返回一个新的迭代器。从而实现我们题目中的需求。  
改进的normalize函数：

```py
def normalize(numbers):
    if iter(numbers) is iter(numbers):  #判断参数是否为迭代器类型
        raise TypeError('参数不能为迭代器.')
    total = sum(numbers)
    result = []
    for value in numbers:
        percent = 100 * value / total
        result.append(percent)

    return result
```
改进后的normalize同样可以接受列表或元组类型的参数。个中原因，同样遵循上述原理。
