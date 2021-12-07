---
layout: post
title: 'Numpy快速入门'
subtitle: '阅读官方QuickStart的一些分享笔记'
date: 2021-08-10
author: 周子博
categories: 笔记
cover: '/assets/img/2021-08-10/numpylogo.svg'
tags: Python
---

## What & Why

NumPy实质上就是一个科学计算的Python基础库，提供多维数组对象，及其各种派生对象如masked数组和矩阵，并且提供对这些对象的一系列快速操作，包括数学、逻辑、形状处理、排序、选择、I/O、离散傅里叶变换、基本线性代数、基本统计运算、随机模拟等等。

NumPy的核心是多维数组`n-dimensional arrays`，即`ndarray`对象，其封装了一个同数据类型的多维数组，同时提供了许多由预先编译好的代码来执行的操作，以提供更好的性能表现。

与Python `list`数据类型不同的是：

- `ndarray`创建时固定大小，改变数组大小时会新创建一个数组并删除原来的数组，而`list`允许动态增长。
- `ndarray`数组内的元素均为同一数据类型，在内存中也是同样的大小，`list`则没有要求。
- `ndarray`便于对大量数据进行高阶数学以及其他类型的运算操作，和`list`相比能提供更高的性能，使用更少的代码。
- 现在绝大多数的科学和数学Python库内部都是在使用NumPy数组。一般来说这些库都会支持Python原生数据类型的输入，然后在处理和计算前转成`ndarray`，计算结果通常也会返回`ndarray`。

> NumPy的目的就是为了具有C语言的速度，同时具有Python的代码便捷性。

## 理解基础

对于NumPy多维数组，需要先理解几个概念：

- `axis`：*轴*，NumPy把维度叫做轴，如果觉得不好理解可以先替换成*维度*，同时往下继续读，之后便会理解。*轴*是多维数组里最重要的概念，轴和维度在某些程度上概念相似，但是轴还具备了方向性，表示了同一个维度下数组前进的方向，或者说索引增长的方向。

  ```python
  # 只有一个轴/维度，该轴内有3个元素，所以该轴长度为3，轴的方向是从1->2->1
  [1, 2, 1]
  
  # 有两个轴/维度，第一个轴长度为2，第二个轴长度为3。如同C里的多维数组a[2][3]
  # 对[1, 0, 0]和[0, 1, 2]这两个元素来说，它们同在第一个轴，属于同一个维度
  # 但是轴前进的方向是从[1, 0, 0]指向[0, 1, 2]
  # 如同索引增长的方向a[0][] -> [1][]
  # 第二个轴的方向同理
  [[1, 0, 0],
  [0, 1, 2]]
  ```

- `ndim`：*数组轴/维度的数量*，如`a[2][3]`就是有两个轴/二维。
- `shape`：*表示数组各轴/维度长度的tuple*，比如`a[2][3]`的`shape`就是`(2,3)`。
- `size`：*数组全部元素的数量*，值等于`shape`里各元素的乘积。
- `dtype`：*描述数组数据类型的对象*，如`numpy.int32`，`numpy.int16`，`numpy.float64`等。
- `itemsize`：*数组中一个元素的大小，单位字节bytes*，大小等于`ndarray.dtype.itemsize`。比如`float64`的`itemsize`为8(=64/8)
- `data`：*存储数组实际元素的buffer*。

### 创建数组

通过`np.array()`传入一个Python序列(`list`或者`tuple`)创建一个NumPy数组。

```python
>>> import numpy as np
>>> a = np.array([2, 3, 4])
>>> a
array([2, 3, 4]) 
>>> a.dtype
dtype('int32')
>>> b = np.array([1.2, 3.5, 5.1]) 
>>> b.dtype
dtype('float64')
>>> type(b)
<class 'numpy.ndarray'>
```

`np.zeros()`/`np.ones()`/`np.empty()`这三种方法用来创建具有初始内容(placeholder)的多维数组，需要传入一个`tuple`来描述`shape`，如果不传入`dtype`则默认`float64`，所以用这类方式要考虑元素的类型`dtype`，避免造成内存空间上的浪费。类似的创建方式还有`np.fromfunction()`,`np.fromfile`等。

`np.arange()`会用类似Python `range()`的方式创建一个有序的数组，`dtype`默认为`int`。

```python
>>> np.zeros((3, 4))
array([[0., 0., 0., 0.],
       [0., 0., 0., 0.],
       [0., 0., 0., 0.]])
>>> np.ones((2, 3, 4), dtype=np.int16)
array([[[1, 1, 1, 1],
        [1, 1, 1, 1],
        [1, 1, 1, 1]],

       [[1, 1, 1, 1],
        [1, 1, 1, 1],
        [1, 1, 1, 1]]], dtype=int16)
>>> np.empty((2, 3)) # 初始内容随机，为内存中的值
array([[3.73603959e-262, 6.02658058e-154, 6.55490914e-260],
       [5.30498948e-313, 3.14673309e-307, 1.00000000e+000]])
>>> np.arange(10, 30, 5) # 与Python range类似
array([10, 15, 20, 25])    

# 对于浮点数步进的arange，可能因为精度问题无法获得我们想要的数组内容，
# 可以使用linspace方法来等分一个数值区间
>>> np.linspace(0, 2, 9)
array([0.  , 0.25, 0.5 , 0.75, 1.  , 1.25, 1.5 , 1.75, 2.  ])
```

### 打印数组

当你调用`print(ndarray)`打印数组时，NumPy会用类似嵌套列表的方式显示该数组内容，按照以下规则打印：

- 从左到右打印最后一个轴/维度
- 从上到下打印倒数第二个轴/维度
- 从上到下打印剩下的轴/维度，但是每个维度的切片之间会间隔n-2个空行，n表示相邻两个切片属于倒数第n个维度
  
> 超过一定规模的多维数组，NumPy只会打印数组的corners，而跳过中间的部分

```python
>>> a = np.arange(36).reshape(2, 2, 3, 3) # 四维数组
>>> print(a)
[[[[ 0  1  2]
   [ 3  4  5]
   [ 6  7  8]]

  [[ 9 10 11]
   [12 13 14]
   [15 16 17]]]


 [[[18 19 20]
   [21 22 23]
   [24 25 26]]

  [[27 28 29]
   [30 31 32]
   [33 34 35]]]]

>>> print(np.arange(10000))
[   0    1    2 ... 9997 9998 9999]
```

### 基本操作

1. NumPy数组的算术运算都是逐元素的。运算结果会创建一个新的`ndarray`并填充结果返回。值得注意的是，在此规则下，`ndarray`表示的矩阵的乘法要用`@`或者`dot()`，否则用`*`运算符会表示逐元素相乘。

    ```python
    >>> a = np.array([20, 30, 40, 50])
    >>> b = np.arange(4)
    >>> b
    array([0, 1, 2, 3])
    >>> c = a - b
    >>> c
    array([20, 29, 38, 47])
    >>> b**2
    array([0, 1, 4, 9])
    >>> a < 35
    array([ True,  True, False, False])
    >>> A = np.array([[1, 1],
    ...               [0, 1]])
    >>> B = np.array([[2, 0],
    ...               [3, 4]])
    >>> A * B     # 逐元素相乘
    array([[2, 0],
          [0, 4]])
    >>> A @ B     # 矩阵乘法
    array([[5, 4],
          [3, 4]])
    >>> A.dot(B)  # 矩阵乘法
    array([[5, 4],
          [3, 4]])
    ```

2. 一些运算符如`+=`,`*=`不会创建一个新的数组返回，而是在现有数组上直接操作(In-place)。这可以节省开辟和释放内存的开销。
3. NumPy随机多维数组的写法：

    ```python
    rg = np.random.default_rng(1) # 随机数生成器 random number generator
    rg.random((2, 3)) # 创建对应维数和长度的随机数组，值范围(0,1)，自行缩放映射
    ```

4. NumPy数组提供许多内置的一元运算操作，比如对一个`ndarray a`,有所有元素求和`a.sum()`,取最小`a.min()`,取最大`a.max()`。如果不传入任何参数，这些操作会忽视掉NumPy数组的轴/维度（或者说shape），单纯看作一系列数字操作。***如果传入`axis`参数，这些操作就会只沿着所声明的轴/维度的方向进行***。如果对轴的方向有问题请回顾[这里](##理解基础)。

    ```python
    >>> b = np.arange(12).reshape(3, 4)
    >>> b
    array([[ 0,  1,  2,  3],
          [ 4,  5,  6,  7],
          [ 8,  9, 10, 11]])
    >>>
    >>> b.sum(axis=0)     # 沿着第一个轴/维度的方向（列方向）求和
    array([12, 15, 18, 21])
    >>>
    >>> b.min(axis=1)     # 沿着第二个轴/维度的方向(行方向)求最小值
    array([0, 4, 8])
    >>>
    >>> b.cumsum(axis=1)  # 沿着第二个轴/维度的方向(行方向)求累加值
    array([[ 0,  1,  3,  6],
          [ 4,  9, 15, 22],
          [ 8, 17, 27, 38]])
    ```

### 通用函数

NumPy还提供了许多数学函数如`np.sin()`,`np.cos()`,`np.exp()`,`np.sqrt()`等等，这些被称为*universal functions*，`ufunc`，这些运算也是逐元素进行的，并且会创建一个新的数组作为结果返回。

```python
>>> B = np.arange(3)
>>> B
array([0, 1, 2])
>>> np.exp(B)
array([1.        , 2.71828183, 7.3890561 ])
>>> np.sqrt(B)
array([0.        , 1.        , 1.41421356])
>>> C = np.array([2., -1., 4.])
>>> np.add(B, C)
array([2., 0., 6.])
```

### 索引、切片、迭代

一维的数组索引、切片与Python序列一致。比较需要理解的是多维数组的索引切片。每个维度之间的切片索引用逗号间隔。如果提供的索引数比轴/维度的数量少，则未提供索引的轴会被人为切取全部内容。且NumPy支持用`...`来填充未给出切片索引的轴。

```python
>>> b # 一个(5, 4) shape的二维NumPy数组
array([[ 0,  1,  2,  3],
       [10, 11, 12, 13],
       [20, 21, 22, 23],
       [30, 31, 32, 33],
       [40, 41, 42, 43]])
>>> b[2, 3]
23
>>> b[0:5, 1]  # 每一行，第二列
array([ 1, 11, 21, 31, 41])
>>> b[:, 1]    # 同上
array([ 1, 11, 21, 31, 41])
>>> b[1:3, :]  # 第二和第三行，所有列
array([[10, 11, 12, 13],
       [20, 21, 22, 23]])
>>> b[-1]   # 等同于 b[-1, :]，取最后一行，所有列
array([40, 41, 42, 43])


>>> c = np.array([[[  0,  1,  2],  # 一个(2, 2, 3)shape的三维数组
...                [ 10, 12, 13]],
...               [[100, 101, 102],
...                [110, 112, 113]]])
>>> c[1, ...]  # 等同于 c[1, :, :] or c[1]
array([[100, 101, 102],
       [110, 112, 113]])
>>> c[..., 2]  # 等同于 c[:, :, 2]
array([[  2,  13],
       [102, 113]])
```

> 关于多维数组切片的结果是否是多维，是多少维度的数组，取决于各维度的切片索引。如果在某一个维度只取一个固定索引的值，则该维度便会消失。如上`b[2, 3]`，两个固定索引，所以两个维度都消失了，只得到一个元素。`b[1:3, :]`则得到一个二维数组。

关于NumPy数组的迭代，可以直接使用`for`循环按照第一个轴进行迭代

```python
>>> for row in b:
...     print(row)
...
[0 1 2 3]
[10 11 12 13]
...
```

也可以使用迭代器`ndarray.flat`进行逐元素的迭代

```python
>>> for element in b.flat:
...     print(element)
...
0
1
2
3
...

```

## Shape操作

### 改变ndarray shape

`ndarray.ravel()`展开成一维数组,`ndarray.reshape()`维度变换,`ndarray.T`返回转置数组。注意这些操作都不会改变原数组，而是返回一个新的数组。

```python
>>> a # shape (3, 4)
array([[3., 7., 3., 4.],
       [1., 4., 2., 2.],
       [7., 2., 4., 9.]])
>>> a.ravel()  # 展开成一维，展开方式是"C-style"，即沿着最后一个轴的方向展开，在二维就是逐行展开
array([3., 7., 3., 4., 1., 4., 2., 2., 7., 2., 4., 9.])
>>> a.reshape(6, 2)  # 返回修改shape后的数组
array([[3., 7.],
       [3., 4.],
       [1., 4.],
       [2., 2.],
       [7., 2.],
       [4., 9.]])
>>> a.T  # 返回转置后的数组
array([[3., 1., 7.],
       [7., 4., 2.],
       [3., 2., 4.],
       [4., 2., 9.]])
>>> a.T.shape
(4, 3)
>>> a.shape
(3, 4)
```

> NumPy normally creates arrays stored in this order, so ravel will usually not need to copy its argument, but if the array was made by taking slices of another array or created with unusual options, it may need to be copied.  
这一段没太读懂，是指正常创建的数组在ravel()展开时不需要拷贝？但是ravel返回的不是一个新的数组吗，为什么不需要拷贝呢？  
答：ravel reshape resize这些操作返回的都是一个新的ndarray对象，或者说视图而已，都不会涉及内存的拷贝。可以参考[这里](###视图/浅拷贝)，但是当从一个已经是切片的数组展开的话，或者是通过一些不寻常（复杂）的方法创建的话，NumPy会进行内存拷贝（数据深拷贝），这个应该是NumPy的优化，应该是为了避免对视图操作的叠加导致效率还不如深拷贝。

与`reshape()`返回新数组不改变原数组相比，`resize()`用法一致，但是改变了原数组。

```python
>>> a.resize((2, 6)) # 改变了数组本身
>>> a
array([[3., 7., 3., 4., 1., 4.],
       [2., 2., 7., 2., 4., 9.]])
```

如果在reshape的操作中某个轴/维度给了-1的长度，那么其长度是根据原数组的shape自动计算的。或者说，*全自动Reshape*。

```python
>>> a.reshape(3, -1)
array([[3., 7., 3., 4.],
       [1., 4., 2., 2.],
       [7., 2., 4., 9.]])
```

## 数组的拷贝与视图

### 无拷贝

简单的赋值不会引起任何拷贝

```python
>>> a = np.array([[ 0,  1,  2,  3],
...               [ 4,  5,  6,  7],
...               [ 8,  9, 10, 11]])
>>> b = a            # 不会创建新的对象
>>> b is a           # a和b是同一个ndarray对象的别名
True
```

对可变对象而言，函数调用传入的是引用，所以也不会有任何拷贝。

```python
>>> def f(x):
...     print(id(x))
...
>>> id(a)  # id is a unique identifier of an object
148293216  
>>> f(a)
148293216  # 与id(a)一致
```

### 视图/浅拷贝

不同的`ndarray`对象可以共享同一块数据，通过`view`方法可以创建一个新的`ndarray`对象，或者称为*视图*，其与原数组共享/引用同一块数据，所以是浅拷贝。

> 可以这样理解，所有的ndarray对象都只是对内存中连续数组的一个视图，以及操作方法的集合。虽然会标记数据属于哪个视图（第一次创建的时候），但是所有的视图都能对这一片内存的数据进行改动。视图，或者说`ndarray`对象自己的`reshape`，`resize`变换等操作，凡是不涉及到数据本身的改动的，都不会对其他视图造成影响。根据问题的需求，决定了内存中一块连续数据的理解方式，从而决定了需要怎样的视图。NumPy封装的就是根据不同的视图进行对应的操作。

```python
>>> c = a.view()
>>> c is a
False
>>> c.base is a            # c是观察a数据的视图
True
>>> c.flags.owndata
False
>>>
>>> c = c.reshape((2, 6))  # a数组的shape不会改变
>>> a.shape
(3, 4)
>>> c[0, 4] = 1234         # a数组的数据变了
>>> a
array([[   0,    1,    2,    3],
       [1234,    5,    6,    7],
       [   8,    9,   10,   11]])
```

切片会返回一个新的视图，所以会创建一个新的`ndarray`对象，但是不会发生内存拷贝。

```python
>>> s = a[:, 1:3]
>>> s[:] = 10  # s[:]是s的视图. 注意s=10 和s[:]=10的区别，应该是这样的写法才能进行广播？
>>> a
array([[   0,   10,   10,    3],
       [1234,   10,   10,    7],
       [   8,   10,   10,   11]])
```

### 深拷贝

`ndarray.copy()`方法会拷贝一个NumPy数组及其数据。

```python
>>> d = a.copy()  # 创建一个新的ndarray对象和新的数据
>>> d is a
False
>>> d.base is a  # d 与 a 并不共享任何数据
False
>>> d[0, 0] = 9999
>>> a
array([[   0,   10,   10,    3],
       [1234,   10,   10,    7],
       [   8,   10,   10,   11]])
```

如果要从一大片数据中筛选出一片有用的话，往往会在切片后使用`copy`深拷贝，然后删除原先的`ndarray`对象和其引用的数据，释放内存。

```python
>>> a = np.arange(int(1e8))
>>> b = a[:100].copy()
>>> del a  # the memory of ``a`` can be released.
```

如果使用切片的话，即时调用`del a`，因为`b`中仍有引用(`b.base`)，GC并不会回收`a`对象。

### 函数和方法概览/进阶索引使用技巧/官方小贴士

因为本文只是快速入门，进阶的技巧就不详细赘述了，请参考官方的文档学习一下，大概需要20分钟。

- [方法概览](https://numpy.org/doc/stable/user/quickstart.html#functions-and-methods-overview)
- [进阶索引使用技巧](https://numpy.org/doc/stable/user/quickstart.html#advanced-indexing-and-index-tricks)
- [小贴士](https://numpy.org/doc/stable/user/quickstart.html#tricks-and-tips)

## Additional

在重构我们的光照贴图后处理部分时，我遇到了一些问题，应该可以加深你对NumPy的理解。下面是几段简化后的代码，你可以在我们的工程中找到其原型：

```python
dlm = surfaceCache.directLightMap()
idlm = surfaceCache.indirectLightMap()
ret = np.zeros((width, height, 4))

# 对dlm进行后处理，结果写回ret
...
dlm = ret
# 对idlm进行后处理，结果写回ret
...
idlm = ret
lightmap = dlm + idlm
```

值得注意的是，这样的写法会导致第二次后处理结果写回`ret`时，`dlm`指向的内容也被覆盖。如果你读完了这篇入门，你应该能知道为什么。所以正确的写法应该是在第二次后处理前加上：

```python
# 对idlm进行后处理，结果写回ret
ret = np.zeros((width, height, 4))
...
idlm = ret
```

再看这一段代码

```python
normalMapAChannel = normalMap[:,:,3]
isMasked = idlm[:,:,0] < 0
normalMapAChannel = np.where(isMasked, -1, normalMapAChannel)
```

这里用到了`np.where`来根据`isMasked`布尔型数组来有条件地改变`normalMapAChannel`数组。请问`normalMapAChannel`作为`normalMap`的切片/视图，指向的是同一片数据，这样可以根据`isMasked`中的条件修改到`normalMap`中的值吗？

> 答案是：不能。原因是`np.where`返回了一个新的ndarray数组，并且赋值给了`normalMapAChannel`变量，并未通过该视图去改变`normalMap`中的数据。如果需要这样做，则需改写最后一句为：

```python
normalMap[:,:,3] = np.where(isMasked, -1, normalMapAChannel)
# 或者直接用布尔型数组作为索引，实现In-Place的赋值修改
normalMapAChannel[isMasked] = -1
```

## 参考

- [NumPy Manual - NumPy官网](https://numpy.org/doc/stable/index.html)
