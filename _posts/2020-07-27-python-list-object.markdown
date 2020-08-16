---
layout: post
title:  Python中的List对象
date:   2020-07-27 12:07:03 +0800
tags:   Python CPython Python源码剖析 列表
---
* TOC
{:toc}

摘自：[《Python源码剖析》 — 陈儒](https://read.douban.com/ebook/1499455/)

**[`PyListObject`](https://github.com/python/cpython/blob/v3.8.3/Include/listobject.h#L23-L40)**

```c
typedef struct {
    PyObject_VAR_HEAD
    /* Vector of pointers to list elements.  list[0] is ob_item[0], etc. */
    PyObject **ob_item;

    /* ob_item contains space for 'allocated' elements.  The number
     * currently in use is ob_size.
     * Invariants:
     *     0 <= ob_size <= allocated
     *     len(list) == ob_size
     *     ob_item == NULL implies ob_size == allocated == 0
     * list.sort() temporarily sets allocated to -1 to detect mutations.
     *
     * Items must normally not be NULL, except during construction when
     * the list is not yet visible outside the function that builds it.
     */
    Py_ssize_t allocated;
} PyListObject;
```

`PyListObject`所采用的内存管理策略和C++中`vector`采取的内存管理策略是一样的，这里的`ob_size`和`allocated`的关系就像C++的`vector`中`size`和`capacity`一样。

当列表元素个数超过`allocated`或者不足`allocated // 2`（即一半）时，对列表长度进行调整为`size_t new_allocated = (size_t)newsize + (newsize >> 3) + (newsize < 9 ? 3 : 6);`

# PyListObject对象的创建与维护

#### 创建对象 `PyObject * PyList_New(Py_ssize_t size)`

1. 创建`PyListObject`对象，优先复用对象缓存池资源
2. 根据参数`size`分配列表元素内存空间

#### PyListObject对象缓冲池 `free_lists`

> `free_lists`中所缓冲的`PyListObject`对象是在一个`PyListObject`被销毁的过程中获得的。

参考代码 [`static void list_dealloc(PyListObject *op)`](https://github.com/python/cpython/blob/v3.8.3/Objects/listobject.c#L359-L381)

也就是说，Python在创建第一个`PyListObject`对象时会新分配内存，销毁后该列表元素会被释放，但列表对象本身会被保留在`free_lists`缓存池中；
Python 3.8.3`free_lists`缓存池的大小默认为80个。
我们可以实验验证：
```py
# * 这里使用IPython无法验证，说明IPython中本身已经占用了所有的缓存池空间
# 缓存池复用
>>> l1 = []
>>> id(l1)
4386008960
>>> del l1
>>> l2 = []
>>> id(l2)
4386008960
>>> l3 = []
>>> id(l3)
4386148576
>>> del l2
>>> del l3
>>>
>>> m, n = [], []
>>> id(m)
4386148576
>>> id(n)
4386008960
```
