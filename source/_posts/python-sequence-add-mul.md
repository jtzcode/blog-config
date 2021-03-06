---
title: Python序列：+和*的微妙之处
date: 2020-09-28 20:57:04
categories:
    - 技术
tags:
    - Tech
    - Python
---
本文分享一下在Python序列上使用加号和乘号的特性，以及背后的一些有趣实现，供您参考。

### 序列的加和乘
在序列相加其实就是拼接，加号两侧的数据类型要相同。拼接过程不会修改原来对象，而是生成一个新对象来保存结果。这个操作比较常见，比如字符串的拼接和列表的拼接。

所谓序列的乘，其实是把序列乘以一个整数N，效果是把列表中元素**复制N份**，然后再拼接起来。比如：

```python
l = [0, 1, 2]
print(l * 3)
print(3 * 'abcd')

>> [0, 1, 2, 0, 1, 2, 0, 1, 2]
>> abcdabcdabcd
```
有了这个乘的操作，我们可以用来方便地初始化多维的序列。比如：
<!--more-->
```python
board = []
for i in range(3):
    row = ['_'] * 3
    board.append(row)
board[2][1] = 'A'
print(board)

>> [['_', '_', '_'], ['_', '_', '_'], ['_', 'A', '_']]
```
列表`board`的每个元素是一个长度为3的列表。这些列表每个元素初始化为占位符`_`，然后可以赋值。这里面有个小陷阱是，初始化时**使用同一个引用生成不同的元素**，这会导致很多元素都指向同样的内存。比如：

```python
wrong_board = [['_'] * 3] * 3
print(wrong_board)
wrong_board[0][1] = 'Z'
print(wrong_board)

>> [['_', '_', '_'], ['_', '_', '_'], ['_', '_', '_']]
>> [['_', 'Z', '_'], ['_', 'Z', '_'], ['_', 'Z', '_']]
```
上面的初始化看起来没有问题，但是由于乘法生成的列表都是同一个引用，修改元素时也会影响到所有相同引用的列表元素。这可能不是想要的结果。
### 序列的增量赋值
序列的增量赋值就是`+=`操作，它将列表拼接的结果赋值给原来列表。与相加之后再赋值相比，**增量赋值没有新的对象产生**，而是直接使用原对象作为结果。

没有新对象产生听起来是个好事，但有个前提。执行增量赋值的类必须**实现了`__iadd__`特殊方法**，在执行+=操作时就会调用这个方法。如果没有这个方法，就会先构造一个新的对象保存结果。像list这种可变序列，就实现了这个方法。

序列乘也有类似的增量赋值，背后原理也一样。

```python
daye = [1, 2, 3]
dama = (4, 5)
daye *= 2
dama *= 3
print(daye)
print(dama)

>> [1, 2, 3, 1, 2, 3]
>> (4, 5, 4, 5, 4, 5)
```
对于不可变序列（如元组），解释器会创建新对象，把原来对象的元素复制到新对象中，最后再append新的元素，过程比较繁琐。上面例子中赋值后的元组`dama`已经不是你3年前的`dama`了。

Python对`str`的处理比较特殊。虽然也是不可变序列，但是`str`的拼接太普遍，CPython对这个过程进行了优化。`str`在初始化时会**留出一定的额外空间供未来扩展**，就省去了拷贝原有字符的时间。

*Fluent Python* 这本书提到了一个有意思的题目，下面的代码会返回什么结果？

```python
t = (1, 2, [30, 40])
t[2] += [50, 60]
print(t)
```
你可能会说，元组是不可变序列，不能对其中的元素赋值，所以会抛出异常。确实如此，不过这个异常的背后有一个副作用，就是那个**赋值操作依然成功**了！也就是元素`t[2]`变成了`[30, 40, 50, 60]`。

这个现象反映出，Python执行增量赋值**不是一个原子操作**，而是分步进行。首先保存原来的元素值，然后把目标元素值改变。注意，这里是改变列表`[30, 40]`的值，因此是可变的。最后，把这个新值赋值给元组的元素。这一步才会出异常，不过前面一步的赋值已经完成了。

这提醒我们，最好不要在元组中放入可变的对象，因为你不知道什么时候会去改变它，而你的改动成功了却不自知。
### 参考资料

 - *Fluent Python*. Chapter 2.
 - https://stackoverflow.com/questions/12169839/which-is-the-preferred-way-to-concatenate-a-string-in-python
