---
layout: post
title: sort方法中的key
category: Python编程技巧
tags: [python, think]
keywords: python,think,effective
---
### 问题
对数组numbers = [8,3,1,2,5,4,7,6]，进行排序。要求它的子集group = [3,2,5,7]中的元素排在其他元素前面（整体仍然有序），即期望结果是:[2, 3, 5, 7, 1, 4, 6, 8]。  
答案：  

```python
def sort_prior(values,group):
    def helper(x):
        if x in group:
            return (0,x)
        return (1,x)
    values.sort(key=helper)

numbers = [8,3,1,2,5,4,7,6]
group = [3,2,5,7]

sort_prior(numbers,group)
print(numbers)
```

解析：  
解题关键在于对sort()方法中key参数意义的理解。
看看官网的解释：  
> *key* specifies a function of one argument that is used to extract a comparison key from each list element (for example, key=str.lower). The key corresponding to each item in the list is calculated once and then used for the entire sorting process. The default value of None means that list items are sorted directly without calculating a separate key value.

key参数接受一个函数，这个函数传入的参数是待排序的列表中的元素，然后返回一个可排序的值，用这个值去参与整个排序过程。  
举例:

```python
>>> arr = ['B','a']
>>> arr.sort()
>>> arr
['B', 'a']
>>> arr.sort(key=str.lower)  #数组中的元素按str.lower处理后的结果来进行排序
>>> arr
['a', 'B']
```
回过头来看看原来的问题。  
numbers中的元素经过helper处理后变成：  
```python
[(1,8),(0,3),(1,1),(0,2),(0,5),(1,4),(0,7),(1,6)]  
```
而元组的排序规则是，先对第一个元素比较，小的排前，大的拍后，相等则比较第二元素，以此类推。所以结果为：
```python 
[(0, 2), (0, 3), (0, 5), (0, 7), (1, 1), (1, 4), (1, 6), (1, 8)]  
```
最后按原来参数对应的值形成最终结果。  

### list.sort() 与 sorted(list)的区别  
list.sort()会直接改变list的值  
sorted是将listcopy一份后再返回排序结果

