---
title: Python 序列之切片（slice）
date: 2020-09-28 20:54:38
categories:
    - 技术
tags:
    - Tech
    - Python
---
本文为Python列表切片功能的学习笔记，供您参考。
## 基本用法
Python中支持切片操作的序列类型有**列表（list）、元组（tuple）以及字符串（str**）。

以列表为例，`s[a:b]`表示对于列表s，返回从下标a的元素到下标**b的前一个元素**的子列表。注意这里的包含关系是`[a,b)`。跟有的语言不同（比如JavaScript），python中切片的两个参数都是下标，而非一个是下标，另一个是长度。

如果省略了某个参数，则表示从“**第一个元素开始**”或者“**到最后一个元素**”。例如：

```python
l = [10, 20, 30, 40, 50, 60]
print(l[3:])
print(l[:3])
>> [40, 50, 60]
>> [10, 20, 30]
```
这里有个小问题，为什么我们指定了第二个下标参数，却不包含这个位置的元素呢？这其实是很多语言特性的风格，与起始下标为0的风格是一致的。
<!--more-->
计算机科学家Dijkstra在一篇备忘录里对这个问题进行了简单探讨，他认为在[a,b)，(a, b]，[a, b]以及(a, b)四种表示序列范围的方法中，第一种是最优雅的。这种思路综合考虑了**子序列的连续性、上下界以及空序列**的问题，最后得出结论，该思路影响了编程语言的设计。详细内容请参考资料1，下面是他当年的手稿。

![备忘录手写稿](https://img-blog.csdnimg.cn/20200904143704668.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2p0el9NUFA=,size_16,color_FFFFFF,t_70#pic_center)

## 高阶用法
Python的切片还支持`s[a:b:c]`的形式，即对列表s在a和b之间**以c为间隔**取值。其中c的值可以为负数，意思是反向取值，这个**反转**列表提供了便利。如果省略a和b的参数，意味着针对整个序列操作。例如：
```python
s = 'bicycle'
print(s[::3])
print(s[::2])
print(s[::-1])
>> bye
>> bcce
>> elcycib

def foo(a, b, c):
    return a + c
print(foo(1, ..., 3))
```
第三行就是反转序列的例子。在这种用法的背后，实际上是通过a，b和c生成了一个**切片对**象：`slice(a, b, c)`。在对一个序列进行切片操作时，Python会调用序列上的`__getitem__`方法并传入切片对象：

```python
seq.__getitem__(slice(start, stop, step))
```
另外，还可以给列表切片赋值。例如：

```python
l = list(range(10))
l[2:5] = [0, 0]
print(l)
del l[5:7]
print(l)
>> [0, 1, 0, 0, 5, 6, 7, 8, 9]
>> [0, 1, 0, 0, 5, 8, 9]
```
首先将下标2，3，4的元素设置0和0，由于缺少一个目标值，第4位置的元素被删除了。然后利用del运算符删除下标5，6位置的元素，即将6和7两个数字删除，得到最后列表。注意，赋值等号的右边一定也是一个序列（iterable），而不能是单一的元素，否者会报错。

如果是一个多维列表，Python目前不支持切片操作，不过有些第三方的库提供了这个功能，比如**NumPy**，切片方式类似`a[m:n, j:k]`。感兴趣可以参考其官方网站，下载试用。
## 参考资料

 - http://www.cs.utexas.edu/users/EWD/transcriptions/EWD08xx/EWD831.html
 - *Fluent Python*. Chapter **2** Data Structrures
 - https://numpy.org/doc/stable/
