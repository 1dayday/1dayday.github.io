---
layout: post
title:  Python的编译结果 —— Code对象与pyc文件
date:   2020-08-16 13:02:34 +0800
tags:   Python CPython Python源码剖析 编译原理
---
* TOC
{:toc}

摘自：[《Python源码剖析》 — 陈儒](https://read.douban.com/ebook/1499455/)

# Python程序的执行过程

1. Python解释器对`.py`文件中的Python源代码进行编译，产生一组Python的字节码（Bytecode），存在对应的`.pyc`文件中
2. Python虚拟机将编译结果按顺序一条一条执行字节码，从而完成Python程序的执行动作

# PyCodeObject对象

**[`PyCodeObject`](https://github.com/python/cpython/blob/v2.7.18/Include/code.h#L9-L30)**

```c
/* Bytecode object */
typedef struct {
    PyObject_HEAD
    int co_argcount;		/* #arguments, except *args */
    int co_nlocals;		/* #local variables */
    int co_stacksize;		/* #entries needed for evaluation stack */
    int co_flags;		/* CO_..., see below */
    PyObject *co_code;		/* instruction opcodes */
    PyObject *co_consts;	/* list (constants used) */
    PyObject *co_names;		/* list of strings (names used) */
    PyObject *co_varnames;	/* tuple of strings (local variable names) */
    PyObject *co_freevars;	/* tuple of strings (free variable names) */
    PyObject *co_cellvars;      /* tuple of strings (cell variable names) */
    /* The rest doesn't count for hash/cmp */
    PyObject *co_filename;	/* string (where it was loaded from) */
    PyObject *co_name;		/* string (name, for reference) */
    int co_firstlineno;		/* first source line number */
    PyObject *co_lnotab;	/* string (encoding addr<->lineno mapping) See
				   Objects/lnotab_notes.txt for details. */
    void *co_zombieframe;     /* for optimization only (see frameobject.c) */
    PyObject *co_weakreflist;   /* to support weakrefs to code objects */
} PyCodeObject;
```

> Python编译器在对Python源代码进行编译的时候，对于代码中的一个Code Block，会创建一个`PyCodeObject`对象与这段代码对应。
> 当进入一个新的名字空间，或者说作用域时，我们就算是进入了一个新的Code Block了。

> 对于Python编译器来说，`PyCodeObject`对象才是其真正的编译结果，而`.pyc`文件只是这个对象在硬盘上的表现形式，它们实际上是Python对源文件编译的结果的两种不同存在方式。
> 在Python中，类、函数、模块都对应着一个独立的名字空间。

> "当作为模块被`import`的时候，或者已有`.pyc`文件但需要更新的时候会编译出`.pyc`文件。"

# `.pyc`文件的生成
 
Python 2.x中[`write_compiled_module`](https://github.com/python/cpython/blob/v2.7.18/Python/import.c#L945-L993)这段代码讲述了Python在将编译得到的`PyCodeObject`写入到`.pyc`文件中时到底进行了怎样的动作；这段代码在Python 3.3中被重新实现，见[issue13959](https://bugs.python.org/issue13959)及[相关变更](https://github.com/python/cpython/commit/16475adcbb9b8131da2a1615bfbeb34a358e7400#diff-12fb18f1056d99e8480487a481f553620)。

一个`.pyc`文件中实际上包含了三部分独立的信息：
1. Python的magic number

    Python所定义的一个整数值，不同版本的Python实现都会定义不同的`pyc_magic`，用来保证Python的兼容性

2. `.pyc`文件创建的时间信息
3. `PyCodeObject`对象

**[`WFILE`](https://github.com/python/cpython/blob/v2.7.18/Python/marshal.c#L57-L67)**

```c
typedef struct {
    FILE *fp;
    int error;  /* see WFERR_* values */
    int depth;
    /* If fp == NULL, the following are valid: */
    PyObject *str;
    char *ptr;
    char *end;
    PyObject *strings; /* dict on marshal, list on unmarshal */
    int version;
} WFILE;
```

# Python的字节码

* Python 2.7.18 [opcode.h](https://github.com/python/cpython/blob/v2.7.18/Include/opcode.h)
* Python 3.8.3  [opcode.h](https://github.com/python/cpython/blob/v3.8.3/Include/opcode.h)

> [`dis` — Disassembler for Python bytecode](https://docs.python.org/3/library/dis.html)
