---
layout: post
title:  Python中的字符串对象
date:   2020-07-20 16:12:23 +0800
tags:   Python CPython Python源码剖析 字符串
---
* TOC
{:toc}

摘自：[《Python源码剖析》 — 陈儒](https://read.douban.com/ebook/1499455/)

**[`PyStringObject`](https://github.com/python/cpython/blob/v2.7.18/Include/stringobject.h#L12-L49)**

> Python 2.x中的字符串中`ob_sval`实际指向一段长度为`obsize`+ 1个字节的内存。
> 在Python字符串对象的中间可以出现`\0`，但字符串必须以`\0`结尾以满足C语言对字符串处理的要求，所以`ob_sval[obsize] == '\0'`。
>
> Python字符串的`ob_shash`用于缓存该字符串的[hash值](https://github.com/python/cpython/blob/v2.7.18/Objects/stringobject.c#L1274-L1306)，`ob_sstate`标记用于表示字符串是否经由`intern()`机制处理；
> 预存字符串的hash值和这里的`intern()`机制将Python虚拟机的执行效率提升了20%！！！😱

```c
typedef struct {
    PyObject_VAR_HEAD
    long ob_shash;
    int ob_sstate;
    char ob_sval[1];
} PyStringObject;
```

**[`PyString_Type`](https://github.com/python/cpython/blob/v2.7.18/Objects/stringobject.c#L3816-L3858)**

> 在`PyStringObject`的类型对象中，`tp_itemsize`被设置为`sizeof(char)`，即一个字节。
> 对于Python中的任何一种变长对象，`tp_itemsize`这个域是必须设置的，`tp_itemsize`指明了由变长对象保存的元素（item）的单位长度，所谓单位长度即是指一个元素在内存中的长度。
> 这个`tp_itemsize`和`ob_siz`e共同决定了应该额外申请的内存之总大小是多少。

## 创建PyStringObject对象

```c
// https://github.com/python/cpython/blob/v2.7.18/Objects/stringobject.c#L34-L165
PyObject * PyString_FromStringAndSize(const char *str, Py_ssize_t size);
PyObject * PyString_FromString(const char *str);
```

```c
/* PyStringObject_SIZE gives the basic size of a string; any memory allocation
   for a string of length n should request PyStringObject_SIZE + n bytes.

   Using PyStringObject_SIZE instead of sizeof(PyStringObject) saves
   3 bytes per string allocation on a typical system.
*/
#define PyStringObject_SIZE (offsetof(PyStringObject, ob_sval) + 1)
```

*新创建的PyStringObject对象的内存布局*
![178428.jpg](/assets/analysis-of-the-python-source-code/178428.jpg)

## 字符串对象的intern机制

1. Python 2.x中通过[`PyString_InternInPlace`](https://github.com/python/cpython/blob/v2.7.18/Objects/stringobject.c#L4767-L4802)接口对长度为0或1的字符串对象进行intern机制的处理，被经过intern机制处理的字符串对象指向同一片内存。
2. intern机制的关键，就是在系统中有一个名为`interned`的KV映射关系集合，记录着被intern机制处理过的`PyStringObject`对象。
3. 对于被intern机制处理了的`PyStringObject`对象，Python采用了特殊的引用计数机制；interned中的指针不能作为字符串对象的有效引用。
4. intern机制是在字符串对象被创建之后才起作用的，因为`PyDictObject`必须以`PyObject*`指针作为键。

## 字符缓冲池

```c
static PyStringObject *characters[UCHAR_MAX + 1];
```

> `PyStringObject`对象也设计了一个对象池`characters`，与整数对象不同的是，小整数的缓冲池是在Python初始化时被创建的，而字符串对象体系中的字符缓冲池则是以静态变量的形式存在着的。在Python初始化完成之后，缓冲池中的所有`PyStringObject`指针都为空。

## PyStringObject效率相关问题

> Python中通过`+`进行字符串连接的方法效率极其低下，其根源在于Python中的`PyStringObject`对象是一个不可变对象。
> 通过利用`PyStringObject`对象的`join`操作来对存储在`list`或`tuple`中的一组`PyStringObject`对象进行连接操作，这种做法只需要分配一次内存，执行效率将大大提高。
