---
layout: post
title:  Python对象初探
date:   2020-07-07 19:23:19 +0800
tags:   Python CPython Python源码剖析
---
* TOC
{:toc}

摘自：[《Python源码剖析》 — 陈儒](https://read.douban.com/ebook/1499455/)

1. 一切都是对象（实例、类型、函数等等）
2. 一个对象一旦被创建，它在内存中的大小就是不变的了

# Python内的对象

**[`PyObject`](https://github.com/python/cpython/blob/v3.8.3/Include/object.h#L99-L108)**

```c
typedef struct _object {
    _PyObject_HEAD_EXTRA
    Py_ssize_t ob_refcnt;           // 引用计数
    struct _typeobject *ob_type;    // 类型信息
} PyObject;
```

> 在`PyObject`的定义中，整型变量`ob_refcnt`与Python的内存管理机制有关，它实现了基于引用计数的垃圾收集机制。

**[`PyVarObject`](https://github.com/python/cpython/blob/v3.8.3/Include/object.h#L113-L116)**
```c
typedef struct {
    PyObject ob_base;
    Py_ssize_t ob_size; /* Number of items in variable part */
} PyVarObject;
```

> 变长对象通常都是容器，`ob_size`这个成员实际上就是指明了变长对象中一共容纳了多少个元素。

图为不同Python对象与`PyObject`、`PyVarObject`的关系；
其中Python 3中`PyIntObject`类型已经统一到`PyLongObject`类型，是一个变长对象。
![175482.jpg](/assets/analysis-of-the-python-source-code/175482.jpg)

# 类型对象

**[`PyTypeObject`](https://github.com/python/cpython/blob/v3.8.3/Include/cpython/object.h#L177-L270)**

```c
typedef struct _typeobject {
    PyObject_VAR_HEAD
    const char *tp_name; /* For printing, in format "<module>.<name>" */
    Py_ssize_t tp_basicsize, tp_itemsize; /* For allocation */

    /* Methods to implement standard operations */

    destructor tp_dealloc;
    Py_ssize_t tp_vectorcall_offset;
    getattrfunc tp_getattr;
    setattrfunc tp_setattr;
    PyAsyncMethods *tp_as_async; /* formerly known as tp_compare (Python 2)
                                    or tp_reserved (Python 3) */
    reprfunc tp_repr;

    /* Method suites for standard classes */

    PyNumberMethods *tp_as_number;
    PySequenceMethods *tp_as_sequence;
    PyMappingMethods *tp_as_mapping;

    /* More standard operations (here for binary compatibility) */

    hashfunc tp_hash;
    ternaryfunc tp_call;
    reprfunc tp_str;
    getattrofunc tp_getattro;
    setattrofunc tp_setattro;
    ...
} PyTypeObject;
```

> 一个`PyTypeObject`对象就是Python中对面向对象理论中“类”这个概念的实现

Python的C API类型：
- 范型API AOL（Abstract Object Layer）：具有诸如`PyObject_*`的形式，可以应用在任何Python对象身上
- 类型相关的API COL（Concrete Object Layer）：通常只能作用在某一种类型的对象上，对于每一种内建对象，Python都提供了这样的一组API

下图展示了从`PyInt_Type`(Python 2.x)创建整数对象的流程
![175504.jpg](/assets/analysis-of-the-python-source-code/175504.jpg)
> 在调用`tp_new`完成“创建对象”（申请分配内存）之后，流程会转向`tp_init`，完成“初始化对象”的工作。
> 对应到C++中，`tp_new`可以视为`new`操作符，而`tp_init`则可视为类的构造函数。

**[`PyType_Type`](https://github.com/python/cpython/blob/v3.8.3/Objects/typeobject.c#L3613-L3655) —— 类型的类型**

初始化时引用数为1，成员数量为0

```c
// Objects/typeobject.c
PyTypeObject PyType_Type = {
    PyVarObject_HEAD_INIT(&PyType_Type, 0)
    "type",                                     /* tp_name */
    sizeof(PyHeapTypeObject),                   /* tp_basicsize */
    sizeof(PyMemberDef),                        /* tp_itemsize */
    (destructor)type_dealloc,                   /* tp_dealloc */
    ...
};

// Include/object.h
#define PyObject_HEAD_INIT(type)        \
    { _PyObject_EXTRA_INIT              \
    1, type },

#define PyVarObject_HEAD_INIT(type, size)       \
    { PyObject_HEAD_INIT(type) size },
```

![175552.jpg](/assets/analysis-of-the-python-source-code/175552.jpg)

# Python对象的多态性

*Polymorphism n. 多态性*

> 通过`PyObject`和`PyTypeObject`，Python利用C语言完成了C++所提供的对象的多态的特性。在Python创建一个对象，比如`PyIntObject`对象时，会分配内存，进行初始化。然后Python内部会用一个`PyObject*`变量，而不是通过一个`PyIntObject*`变量来保存和维护这个对象。其他对象也与此类似，所以在Python内部各个函数之间传递的都是一种范型指针 —— `PyObject*`。这个指针所指的对象究竟是什么类型的，我们不知道，只能从指针所指对象的`ob_type`域动态进行判断，而正是通过这个域，Python实现了多态机制。

# 引用计数

> `PyObject`中的`ob_refcnt`是一个[`Py_ssize_t`](https://www.python.org/dev/peps/pep-0353/)变量，这实际蕴含着Python所做的一个假设，即对一个对象的引用不会超过`Py_ssize_t`的最大值。
> 在每一个对象创建的时候，Python提供了一个[`_Py_NewReference(op)`](https://github.com/python/cpython/blob/v3.8.3/Include/object.h#L436-L444)宏来将对象的引用计数初始化为1。

# Python对象的分类

- Fundamental对象：类型对象
- Numeric对象：数值对象
- Sequence对象：容纳其他对象的序列集合对象
- Mapping对象：类似于C++中map的关联对象
- Internal对象：Python虚拟机在运行时内部使用的对象

![175809.jpg](/assets/analysis-of-the-python-source-code/175809.jpg)
