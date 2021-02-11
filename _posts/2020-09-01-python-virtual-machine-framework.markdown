---
layout: post
title:  Python虚拟机框架
date:   2020-09-01 22:19:57 +0800
tags:   Python CPython Python源码剖析 虚拟机
---
* TOC
{:toc}

摘自：[《Python源码剖析》 — 陈儒](https://read.douban.com/ebook/1499455/)

## Python虚拟机中的执行环境

*可执行文件运行时的运行时栈*
![178484.jpg](/assets/analysis-of-the-python-source-code/178484.jpg)

> 对于一个函数而言，其所有对局部变量的操作都在自己的栈帧中完成，而函数之间的调用则通过创建新的栈帧完成。

> 在Python真正执行的时候，它的虚拟机实际上面对的并不是一个`PyCodeObject`对象，而是另一个对象——`PyFrameObject`。它就是我们所说的执行环境，也是Python对x86平台上栈帧的模拟。

**[`PyFrameObject`](https://github.com/python/cpython/blob/v3.8.3/Include/frameobject.h#L16-L46)**

```c
typedef struct _frame {
    PyObject_VAR_HEAD
    struct _frame *f_back;      /* previous frame, or NULL */
    PyCodeObject *f_code;       /* code segment */
    PyObject *f_builtins;       /* builtin symbol table (PyDictObject) */
    PyObject *f_globals;        /* global symbol table (PyDictObject) */
    PyObject *f_locals;         /* local symbol table (any mapping) */
    PyObject **f_valuestack;    /* points after the last local */
    /* Next free slot in f_valuestack.  Frame creation sets to f_valuestack.
       Frame evaluation usually NULLs it, but a frame that yields sets it
       to the current stack top. */
    PyObject **f_stacktop;
    PyObject *f_trace;          /* Trace function */
    char f_trace_lines;         /* Emit per-line trace events? */
    char f_trace_opcodes;       /* Emit per-opcode trace events? */

    /* Borrowed reference to a generator, or NULL */
    PyObject *f_gen;

    int f_lasti;                /* Last instruction if called */
    /* Call PyFrame_GetLineNumber() instead of reading this field
       directly.  As of 2.3 f_lineno is only valid when tracing is
       active (i.e. when f_trace is set).  At other times we use
       PyCode_Addr2Line to calculate the line from the current
       bytecode index. */
    int f_lineno;               /* Current line number */
    int f_iblock;               /* index in f_blockstack */
    char f_executing;           /* whether the frame is still executing */
    PyTryBlock f_blockstack[CO_MAXBLOCKS]; /* for try and loop blocks */
    PyObject *f_localsplus[1];  /* locals+stack, dynamically sized */
} PyFrameObject;
```

*Python虚拟机在运行时某个时刻的完整运行时环境*
![178485.jpg](/assets/analysis-of-the-python-source-code/178485.jpg)

> 在创建`PyFrameObject`对象时，额外申请的那部分内存中有一部分是给`PyCodeObject`对象中存储的那些局部变量的、`co_freevars`、`co_cellvars`使用的，而另一部分才是给运行时栈使用的。所以，`PyFrameObject`对象中的栈的起始位置（也就是栈底）是由`f_valuestack`维护的，而`f_stacktop`维护了当前的栈顶。

*新创建的PyFrameObject对象*
![178486.jpg](/assets/analysis-of-the-python-source-code/178486.jpg)

## 名字、作用域和名字空间

> Python中引入module的概念，其主要目的是将一些逻辑相关的代码放到一个module中，以备日后使用，即实现代码复用；而另一个目的则是为整个系统划分名字空间。

赋值语句（更确切地说，是具有赋值行为的语句，会影响名字空间）：
* `a = 1`
* `def f()`
* `class A(object)`
* `import abc`

> 一个对象的名字空间中的所有名字都称为对象的属性。在前面，我们看到了Python中有一类“拥有赋值行为”的语句，从另一个角度来看，实际上它们也是“拥有设置对象属性的行为”的语句。既然设置了属性，那么Python中还有一类“拥有访问对象属性的行为”的语句，我们将访问对象属性这个动作称之为“属性引用”。

> 在Python中，一个约束在程序正文的某个位置是否起作用，是由该约束在文本中的位置是否唯一决定的，而不是在运行时动态决定的。因此，Python是具有静态作用域（也称词法作用域）的。

> Python的名字引用的行为被它所支持的嵌套作用域影响，产生的就是最内嵌套作用域规则：由一个赋值语句引进的名字在这个赋值语句所在的作用域里是可见（起作用）的，而且在其内部嵌套的每个作用域里也可见，除非它被嵌套于内部的，引进同样名字的另一条赋值语句所遮蔽。

### LGB规则

*Python 2.2之前的LGB作用域规则*\
![178489.jpg](/assets/analysis-of-the-python-source-code/178489.jpg)

### LEGB规则

Local > Enclosing > Global > Builtin

> 从Python 2.2开始，Python引入了嵌套函数，这时的作用域规则才更接近最内嵌套作用域规则。Python实现闭包是为了实现最内嵌套作用域规则。

## Python虚拟机的运行框架

* Python 2.7.18 [`PyEval_EvalFrameEx`](https://github.com/python/cpython/blob/v2.7.18/Python/ceval.c#L688-L3364)
* Python 3.8.3 [`_PyEval_EvalFrameDefault`](https://github.com/python/cpython/blob/v3.8.3/Python/ceval.c#L744-L3818)

> 在`PyCodeObject`对象的`co_code`域中保存着字节码指令和字节码指令的参数，Python虚拟机执行字节码指令序列的过程就是从头到尾遍历整个`co_code`、依次执行字节码指令的过程。在Python的虚拟机中，利用3个变量来完成整个遍历过程。`co_code`实际上是一个`PyStringObject`对象，而其中的字符数组才是真正有意义的东西，这也就是说，整个字节码指令序列实际上就是一个在C中普普通通的字符数组。因此，遍历过程中所使用的这3个变量都是`char*`类型的变量：`first_instr`永远指向字节码指令序列的开始位置；`next_instr`永远指向下一条待执行的字节码指令的位置；`f_lasti`指向上一条已经执行过的字节码指令的位置。

## Python运行时环境初探

1. Python实现了对多线程的支持
2. Python中的一个线程就是操作系统上的一个原生线程
3. Python中的所有线程都使用虚拟机来完成计算工作
4. 一个线程将拥有一个`PyThreadState`对象保存关于当前线程的信息
5. Python以`PyInterpreterState`对象来实现进程这个抽象概念
6. Python通过一个全局解释器锁GIL（Global Interpreter Lock）来实现线程同步

*Python的运行时环境*
![178503.jpg](/assets/analysis-of-the-python-source-code/178503.jpg)
