---
layout: post
title:  Python虚拟机中的一般表达式
date:   2020-10-20 22:42:16 +0800
tags:   Python CPython Python源码剖析 虚拟机 一般表达式
---
* TOC
{:toc}

摘自：[《Python源码剖析》 — 陈儒](https://read.douban.com/ebook/1499455/)

> 所谓的“一般表达式”包括最基本的对象创建语句，打印语句等。

## 简单内建对象的创建

\* 利用Python的[`dis`](https://docs.python.org/3/library/dis.html)工具对字节码指令进行了解析

```py
In [1]: import dis
In [2]: def func():
   ...:     i = 1
   ...:     s = 'Python'
   ...:     d = {}
   ...:     l = []
   ...:

In [3]: dis.dis(func)
  2           0 LOAD_CONST               1 (1)
              2 STORE_FAST               0 (i)

  3           4 LOAD_CONST               2 ('Python')
              6 STORE_FAST               1 (s)

  4           8 BUILD_MAP                0
             10 STORE_FAST               2 (d)

  5          12 BUILD_LIST               0
             14 STORE_FAST               3 (l)
             16 LOAD_CONST               0 (None)
             18 RETURN_VALUE
```

* `LOAD_CONST` 从`f->f_code->co_consts`中读取序号为0的元素，然后将其压入虚拟机的运行时栈中，其中`f`是当前活动的`PyFrameObject`对象。
* `STORE_NAME` 改变`local`名字空间(`f->f_locals`)，从而完成变量名到变量值之间映射关系的创建。

![178519.jpg](/assets/analysis-of-the-python-source-code/178519.jpg)\
*`i = 1`之后的虚拟机状态*

![178520.jpg](/assets/analysis-of-the-python-source-code/178520.jpg)\
*`s = 'Python'`执行过程中虚拟机状态的转变*

![178521.jpg](/assets/analysis-of-the-python-source-code/178521.jpg)\
*`d = {}`执行过程中虚拟机状态的转变*

![178522.jpg](/assets/analysis-of-the-python-source-code/178522.jpg)\
*Python虚拟机结束时的状态*

## 复杂内建对象的创建

```py
In [1]: import dis
In [2]: def func():
   ...:     i = 1
   ...:     s = 'Python'
   ...:     d = {'1': 1, '2': 2}
   ...:     l = [1, 2]
   ...:

In [3]: dis.dis(func)
  2           0 LOAD_CONST               1 (1)
              2 STORE_FAST               0 (i)

  3           4 LOAD_CONST               2 ('Python')
              6 STORE_FAST               1 (s)

  4           8 LOAD_CONST               1 (1)
             10 LOAD_CONST               3 (2)
             12 LOAD_CONST               4 (('1', '2'))
             14 BUILD_CONST_KEY_MAP      2
             16 STORE_FAST               2 (d)

  5          18 LOAD_CONST               1 (1)
             20 LOAD_CONST               3 (2)
             22 BUILD_LIST               2
             24 STORE_FAST               3 (l)
             26 LOAD_CONST               0 (None)
             28 RETURN_VALUE
```

## 其他一般表达式

### 符号搜索

> `LOAD_NAME`指令正是Python官方文档中所描述的变量的搜索会沿着局部作用域（local名字空间）、全局作用域（global名字空间）、内建作用域（builtin名字空间）依次上溯，直至搜索成功或全部搜完3个作用域，也就是我们之前提到的LGB规则。

### 数值运算

> 在`BINARY_ADD`指令的实现中，Python虚拟机为对象之间的加法运算建立了两条通道，一条快速通道，一条慢速通道。快速通道仅仅是为`PyIntObject`对象和`PyStringObject`对象准备的，而其他的对象都必须通过慢速通道完成加法运算。

> 虽然Python虚拟机为`PyIntObject`对象准备了快速通道，但是当加法运算的结果发生溢出时，Python虚拟机会放弃快速通道计算的结果，转向慢速通道。因为快速通道只能产生另一个PyIntObject对象作为加法运算的结果，而当溢出发生时，就意味着我们需要一个PyLongObject对象作为结果了，这个PyLongObject对象的创建只能在慢速通道中完成。

### 信息输出
