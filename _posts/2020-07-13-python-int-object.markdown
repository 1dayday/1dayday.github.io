---
layout: post
title:  Python中的整型对象
date:   2020-07-13 14:56:43 +0800
tags:   Python CPython Python源码剖析 整数
---
* TOC
{:toc}

摘自：[《Python源码剖析》 — 陈儒](https://read.douban.com/ebook/1499455/)

Python 3.x已经移除了`PyIntObject`类型（定长对象）合并到`PyLongObject`类型（变长对象）；
这里我们对比两种不同版本（v2.7.18 / v3.8.3）的整型对象设计来学习。

```c
// v2.7.18
// https://github.com/python/cpython/blob/v2.7.18/Include/intobject.h#L23-L26
typedef struct {
    PyObject_HEAD
    long ob_ival;
} PyIntObject;
```

```c
// v3.8.3
// https://github.com/python/cpython/blob/v3.8.3/Include/longintrepr.h#L85-L88
struct _longobject {
    PyObject_VAR_HEAD
    digit ob_digit[1];
};
// https://github.com/python/cpython/blob/v3.8.3/Include/longobject.h#L10
typedef struct _longobject PyLongObject; /* Revealed in longintrepr.h */
```

# 小整数对象

> 在Python中，对小整数对象使用了对象池技术。由于整型对象是不可变对象，对象池里的每一个整型对象都能够被任意地共享。

```c
#ifndef NSMALLPOSINTS
#define NSMALLPOSINTS           257
#endif
#ifndef NSMALLNEGINTS
#define NSMALLNEGINTS           5
#endif

#if NSMALLNEGINTS + NSMALLPOSINTS > 0
/* Small integers are preallocated in this array so that they
   can be shared.
   The integers that are preallocated are those in the range
   -NSMALLNEGINTS (inclusive) to NSMALLPOSINTS (not inclusive).
*/
static PyLongObject small_ints[NSMALLNEGINTS + NSMALLPOSINTS];
```

Python中小整数的范围为**`[-5, 257)`**.

# 大整数对象

Python 2.x采用`PyIntBlock`维护通用整数对象池（包括小整数对象）。
当创建新的整数对象时，若整数对象池没有空闲的内存，则[`fill_free_list`](https://github.com/python/cpython/blob/v2.7.18/Objects/intobject.c#L47-L65)函数会申请创建一个新的整数对象块，并返回下一个空闲内存地址。
由于在创建对象池时已知对象类型为整型，Python实现时复用了对象的`ob_type`指针用于链接池中的整型对象形成链表，每次新建的对象会使用链表末端的内存地址。

**[Python 2.x中在Python退出前`PyIntBlock`(s)申请的内存不会被释放归还到系统！](https://github.com/python/cpython/blob/v2.7.18/Objects/intobject.c#L25-L27)**

*创建和删除PyIntObject对象时缓冲池的变化*
![175817.jpg](/assets/analysis-of-the-python-source-code/175817.jpg)

*不同PyIntBlock中的空闲内存的互连*
![175818.jpg](/assets/analysis-of-the-python-source-code/175818.jpg)
