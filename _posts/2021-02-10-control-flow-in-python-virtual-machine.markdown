---
layout: post
title:  Python虚拟机中的控制流
date:   2021-02-10 12:15:58 +0800
tags:   Python CPython Python源码剖析 虚拟机 控制流
---
* TOC
{:toc}

摘自：[《Python源码剖析》 — 陈儒](https://read.douban.com/ebook/1499455/)

## Python虚拟机中的`if`控制流

> 字节码指令序列的顺序是从左到右，从上到下。

```py
In [1]: import dis

In [2]: def func():
   ...:     a = 1
   ...:     if a > 10:
   ...:         print("a > 10")
   ...:     elif a <= -2:
   ...:         print("a <= -2")
   ...:     elif a != 1:
   ...:         print("a != 1")
   ...:     elif a == 1:
   ...:         print("a == 1")
   ...:     else:
   ...:         print("Unknown a")
   ...:

In [3]: dis.dis(func)
  2           0 LOAD_CONST               1 (1)
              2 STORE_FAST               0 (a)

  3           4 LOAD_FAST                0 (a)
              6 LOAD_CONST               2 (10)
              8 COMPARE_OP               4 (>)
             10 POP_JUMP_IF_FALSE       22

  4          12 LOAD_GLOBAL              0 (print)
             14 LOAD_CONST               3 ('a > 10')
             16 CALL_FUNCTION            1
             18 POP_TOP
             20 JUMP_FORWARD            62 (to 84)

  5     >>   22 LOAD_FAST                0 (a)
             24 LOAD_CONST               9 (-2)
             26 COMPARE_OP               1 (<=)
             28 POP_JUMP_IF_FALSE       40

  6          30 LOAD_GLOBAL              0 (print)
             32 LOAD_CONST               5 ('a <= -2')
             34 CALL_FUNCTION            1
             36 POP_TOP
             38 JUMP_FORWARD            44 (to 84)

  7     >>   40 LOAD_FAST                0 (a)
             42 LOAD_CONST               1 (1)
             44 COMPARE_OP               3 (!=)
             46 POP_JUMP_IF_FALSE       58

  8          48 LOAD_GLOBAL              0 (print)
             50 LOAD_CONST               6 ('a != 1')
             52 CALL_FUNCTION            1
             54 POP_TOP
             56 JUMP_FORWARD            26 (to 84)

  9     >>   58 LOAD_FAST                0 (a)
             60 LOAD_CONST               1 (1)
             62 COMPARE_OP               2 (==)
             64 POP_JUMP_IF_FALSE       76

 10          66 LOAD_GLOBAL              0 (print)
             68 LOAD_CONST               7 ('a == 1')
             70 CALL_FUNCTION            1
             72 POP_TOP
             74 JUMP_FORWARD             8 (to 84)

 12     >>   76 LOAD_GLOBAL              0 (print)
             78 LOAD_CONST               8 ('Unknown a')
             80 CALL_FUNCTION            1
             82 POP_TOP
        >>   84 LOAD_CONST               0 (None)
             86 RETURN_VALUE
```

### 比较操作

#### `COMPARE_OP`指令

> Python虚拟机在`COMPARE_OP`指令的实现中为`PyIntObject`对象建立了快速通道。

> Python的`COMPARE_OP`指令不仅管辖着两个对象之间比较操作，而且还覆盖了对象与集合之间关系的判断操作。

#### 比较操作的结果——Python中的`bool`对象

Python虚拟机中有这样两个对立而统一的代表成功和失败的对象：`Py_True`和`Py_False`。

#### 指令跳跃

> 在Python中，有一些字节码指令通常都是成对出现的，更准确地说是顺序出现的，这就为根据上一个字节码指令直接预测下一个字节码指令提供了可能。比如`COMPARE_OP`的后面就通常会紧跟`JUMP_IF_FALSE`或`JUMP_IF_TRUE`，而且通常在它们的后面，还会紧跟着一个`POP_TOP`指令。
>
> 因此Python虚拟机中提供了这样的字节码指令预测功能，如果预测成功，那么会省去很多无谓的操作，使执行效率获得提升。尤其是当这种字节码指令之间的搭配关系出现的概率非常高时，效率的提升尤其显著。

```c
#define JUMPTO(x)       (next_instr = first_instr + (x) / sizeof(_Py_CODEUNIT))
...
#define PREDICT_ID(op)          PRED_##op

#if defined(DYNAMIC_EXECUTION_PROFILE) || USE_COMPUTED_GOTOS
#define PREDICT(op)             if (0) goto PREDICT_ID(op)
#else
#define PREDICT(op) \
    do { \
        _Py_CODEUNIT word = *next_instr; \
        opcode = _Py_OPCODE(word); \
        if (opcode == op) { \
            oparg = _Py_OPARG(word); \
            next_instr++; \
            goto PREDICT_ID(op); \
        } \
    } while(0)
#endif
#define PREDICTED(op)           PREDICT_ID(op):
...
        case TARGET(COMPARE_OP): {
...
            PREDICT(POP_JUMP_IF_FALSE);
            PREDICT(POP_JUMP_IF_TRUE);
            DISPATCH();
        }
...
        case TARGET(POP_JUMP_IF_FALSE): {
            PREDICTED(POP_JUMP_IF_FALSE);
...
            if (cond == Py_False) {
                Py_DECREF(cond);
                JUMPTO(oparg);
                FAST_DISPATCH();
            }
...
            DISPATCH();
        }

        case TARGET(POP_JUMP_IF_TRUE): {
            PREDICTED(POP_JUMP_IF_TRUE);
...
            if (cond == Py_False) {
                Py_DECREF(cond);
                JUMPTO(oparg);
                FAST_DISPATCH();
            }
...
            DISPATCH();
        }
```

> 在整个指令跳跃的过程中，出现了两次跳跃，可能会令人比较迷惑，实际上这两次跳跃是在不同的层面上的跳跃。第一次是通过`PREDICT(POP_JUMP_IF_FALSE);`中的`goto`语句进行跳跃，这次跳跃影响的是Python虚拟机自身，即实现Python的C代码。而在`JUMP_IF_FALSE`的指令代码中通过`JUMPTO(oparg);`完成的跳跃是在Python应用程序层面的跳跃，影响的Python应用程序，是`.py`源文件中的Python代码。

## Python虚拟机中的`for`循环控制流

```py
In [1]: import dis

In [2]: def func():
   ...:     lst = [1, 2]
   ...:     for i in lst:
   ...:         print(i)
   ...:

In [3]: dis.dis(func)
  2           0 LOAD_CONST               1 (1)
              2 LOAD_CONST               2 (2)
              4 BUILD_LIST               2
              6 STORE_FAST               0 (lst)

  3           8 SETUP_LOOP              20 (to 30)
             10 LOAD_FAST                0 (lst)
             12 GET_ITER
        >>   14 FOR_ITER                12 (to 28)
             16 STORE_FAST               1 (i)

  4          18 LOAD_GLOBAL              0 (print)
             20 LOAD_FAST                1 (i)
             22 CALL_FUNCTION            1
             24 POP_TOP
             26 JUMP_ABSOLUTE           14
        >>   28 POP_BLOCK
        >>   30 LOAD_CONST               0 (None)
             32 RETURN_VALUE
```

### 循环控制结构的初始化

在`SETUP_LOOP`的[实现代码](https://github.com/python/cpython/blob/v2.7.18/Python/ceval.c#L2869-L2882)中，仅仅是简单地调用了一个`PyFrame_BlockSetup`函数。

```c
        TARGET_WITH_IMPL(SETUP_LOOP, _setup_finally)
        TARGET_WITH_IMPL(SETUP_EXCEPT, _setup_finally)
        TARGET(SETUP_FINALLY)
        _setup_finally:
        {
            /* NOTE: If you add any new block-setup opcodes that
               are not try/except/finally handlers, you may need
               to update the PyGen_NeedsFinalizing() function.
               */

            PyFrame_BlockSetup(f, opcode, INSTR_OFFSET() + oparg,
                               STACK_LEVEL());
            DISPATCH();
        }
```


#### [`PyTryBlock`](https://github.com/python/cpython/blob/v2.7.18/Include/frameobject.h#L10-L14)

```c
// https://github.com/python/cpython/blob/v2.7.18/Include/frameobject.h#L10-L50
typedef struct {
    int b_type;			/* what kind of block this is */
    int b_handler;		/* where to jump to find handler */
    int b_level;		/* value stack level to pop to */
} PyTryBlock;

typedef struct _frame {
    PyObject_VAR_HEAD
    struct _frame *f_back;	/* previous frame, or NULL */
    PyCodeObject *f_code;	/* code segment */
    PyObject *f_builtins;	/* builtin symbol table (PyDictObject) */
    PyObject *f_globals;	/* global symbol table (PyDictObject) */
    PyObject *f_locals;		/* local symbol table (any mapping) */
    PyObject **f_valuestack;	/* points after the last local */
    /* Next free slot in f_valuestack.  Frame creation sets to f_valuestack.
       Frame evaluation usually NULLs it, but a frame that yields sets it
       to the current stack top. */
    PyObject **f_stacktop;
    PyObject *f_trace;		/* Trace function */

    /* If an exception is raised in this frame, the next three are used to
     * record the exception info (if any) originally in the thread state.  See
     * comments before set_exc_info() -- it's not obvious.
     * Invariant:  if _type is NULL, then so are _value and _traceback.
     * Desired invariant:  all three are NULL, or all three are non-NULL.  That
     * one isn't currently true, but "should be".
     */
    PyObject *f_exc_type, *f_exc_value, *f_exc_traceback;

    PyThreadState *f_tstate;
    int f_lasti;		/* Last instruction if called */
    /* Call PyFrame_GetLineNumber() instead of reading this field
       directly.  As of 2.3 f_lineno is only valid when tracing is
       active (i.e. when f_trace is set).  At other times we use
       PyCode_Addr2Line to calculate the line from the current
       bytecode index. */
    int f_lineno;		/* Current line number */
    int f_iblock;		/* index in f_blockstack */
    PyTryBlock f_blockstack[CO_MAXBLOCKS]; /* for try and loop blocks */
    PyObject *f_localsplus[1];	/* locals+stack, dynamically sized */
} PyFrameObject;

// https://github.com/python/cpython/blob/v2.7.18/Objects/frameobject.c#L788-L798
void
PyFrame_BlockSetup(PyFrameObject *f, int type, int handler, int level)
{
    PyTryBlock *b;
    if (f->f_iblock >= CO_MAXBLOCKS)
        Py_FatalError("XXX block stack overflow");
    b = &f->f_blockstack[f->f_iblock++];
    b->b_type = type;
    b->b_level = level;
    b->b_handler = handler;
}
```

#### `list`的迭代器

```c
        TARGET_NOARG(GET_ITER)
        {
            /* before: [obj]; after [getiter(obj)] */
            v = TOP();
            x = PyObject_GetIter(v);
            Py_DECREF(v);
            if (x != NULL) {
                SET_TOP(x);
                PREDICT(FOR_ITER);
                DISPATCH();
            }
            STACKADJ(-1);
            break;
        }
```

> 在`GET_ITER`的指令代码中，Python虚拟机首先会获得位于运行时栈的栈顶的`PyListObject`对象，然后通过`PyObject_GetIter`获得`PyListObject`对象的迭代器。

> `PyObject_GetIter`是通过调用对象对应的类型对象中的`tp_iter`操作来获得与对象关联的迭代器的。在Python中，迭代器的概念实际上与Java、C#或C++中STL各容器的迭代器的概念是一致的，都是为容器中元素的遍历操作提供一层接口，将对容器的遍历操作与容器的具体实现分离开。这也是GoF中的Iterator Pattern思想的实现。在C++的STL中，有的迭代器是直接利用C的原生指针实现的，比如`vector`；有的则是通过一个类来实现的，比如`list`。虽然Python中的`PyListObject`对象在内存布局上与`vector`相同，但其对应的迭代器却和STL中的`list`是一致的，都通过额外的对象来实现。实际上，在Python中，不光是`PyListObject`对象，只要拥有迭代器的对象，这些迭代器都是一个实实在在的对象，很显然，它们也都是一个`PyObject`对象。

### 迭代控制

```c
        PREDICTED_WITH_ARG(FOR_ITER);
        TARGET(FOR_ITER)
        {
            /* before: [iter]; after: [iter, iter()] *or* [] */
            v = TOP();
            x = (*v->ob_type->tp_iternext)(v);
            if (x != NULL) {
                PUSH(x);
                PREDICT(STORE_FAST);
                PREDICT(UNPACK_SEQUENCE);
                DISPATCH();
            }
            if (PyErr_Occurred()) {
                if (!PyErr_ExceptionMatches(
                                PyExc_StopIteration))
                    break;
                PyErr_Clear();
            }
            /* iterator ended normally */
            x = v = POP();
            Py_DECREF(v);
            JUMPBY(oparg);
            DISPATCH();
        }
```

> `FOR_ITER`的指令代码会首先从运行时栈中获得`PyListObject`对象的迭代器，然后调用迭代器的`tp_iternext`开始进行迭代。迭代器的`tp_iternext`操作总是返回与迭代器关联的容器对象中的下一个元素，如果当前已经抵达了容器对象的结束位置，那么`tp_iternext`将返回`NULL`，这个结果预示着遍历结束。

```c
        PREDICTED_WITH_ARG(JUMP_ABSOLUTE);
        TARGET(JUMP_ABSOLUTE)
        {
            JUMPTO(oparg);
#if FAST_LOOPS
            /* Enabling this path speeds-up all while and for-loops by bypassing
               the per-loop checks for signals.  By default, this should be turned-off
               because it prevents detection of a control-break in tight loops like
               "while 1: pass".  Compile with this option turned-on when you need
               the speed-up and do not need break checking inside tight loops (ones
               that contain only instructions ending with goto fast_next_opcode).
            */
            goto fast_next_opcode;
#else
            DISPATCH();
#endif
        }
```

> `JUMP_ABSOLUTE`指令的行为是强制设定`next_instr`的值，将`next_instr`设定到距离`f->f_code->co_code`开始地址的某一特定偏移的位置。这个偏移的量由`JUMP_ABSOLUTE`的指令参数决定，所以，这条参数就成了`for`循环中指令回退动作最关键的一点。

### 终止迭代

> 在`FOR_ITER`指令的执行过程中，如果通过`PyListObject`对象的迭代器获得的下一个元素不是有效的元素（`NULL`），这就意味着迭代结束了。这个结果将直接导致Python虚拟机会将迭代器对象从运行时栈中弹出，同时执行一个`JUMPBY`的动作，向前跳跃。

> 在`SETUP_LOOP`处，Python虚拟机从`f->f_blockstack`中申请了一个`PyTryBlock`结构，并将执行`SETUP_LOOP`时的一些关于虚拟机的状态信息保存到了所获得的`PyTryBlock`结构中。在执行`POP_BLOCK`指令时，实际上是将这个`PyTryBlock`结构归还给了`f->f_blockstack`。同时，Python虚拟机抽取在`SETUP_LOOP`指令处保存在`PyTryBlock`中的信息，并根据其中存储的`SETUP_LOOP`指令处运行时栈的深度信息将运行时栈恢复到`SETUP_LOOP`之前的状态，从而完成了整个`for`循环结构。

## Python虚拟机中的`while`循环控制结构

```py
In [1]: import dis

In [2]: def func():
   ...:     i = 0
   ...:     while i < 10:
   ...:         i +=1
   ...:         if i > 5:
   ...:             continue
   ...:         if i == 20:
   ...:             break
   ...:         print(i)
   ...:

In [3]: dis.dis(func)
  2           0 LOAD_CONST               1 (0)
              2 STORE_FAST               0 (i)

  3           4 SETUP_LOOP              48 (to 54)
        >>    6 LOAD_FAST                0 (i)
              8 LOAD_CONST               2 (10)
             10 COMPARE_OP               0 (<)
             12 POP_JUMP_IF_FALSE       52

  4          14 LOAD_FAST                0 (i)
             16 LOAD_CONST               3 (1)
             18 INPLACE_ADD
             20 STORE_FAST               0 (i)

  5          22 LOAD_FAST                0 (i)
             24 LOAD_CONST               4 (5)
             26 COMPARE_OP               4 (>)
             28 POP_JUMP_IF_FALSE       32

  6          30 JUMP_ABSOLUTE            6

  7     >>   32 LOAD_FAST                0 (i)
             34 LOAD_CONST               5 (20)
             36 COMPARE_OP               2 (==)
             38 POP_JUMP_IF_FALSE       42

  8          40 BREAK_LOOP

  9     >>   42 LOAD_GLOBAL              0 (print)
             44 LOAD_FAST                0 (i)
             46 CALL_FUNCTION            1
             48 POP_TOP
             50 JUMP_ABSOLUTE            6
        >>   52 POP_BLOCK
        >>   54 LOAD_CONST               0 (None)
             56 RETURN_VALUE
```

## Python虚拟机中的异常控制流

### Python中的异常机制

```py
In [2]: dis.dis(lambda: 1 / 0)
  1           0 LOAD_CONST               1 (1)
              2 LOAD_CONST               2 (0)
              4 BINARY_TRUE_DIVIDE
              6 RETURN_VALUE
```

定位到异常抛出的位置[`intobject.c#i_divmod`](https://github.com/python/cpython/blob/v2.7.18/Objects/intobject.c#L571-L616)

```c
static enum divmod_result
i_divmod(register long x, register long y,
         long *p_xdivy, long *p_xmody)
{
    long xdivy, xmody;

    if (y == 0) {
        PyErr_SetString(PyExc_ZeroDivisionError,
                        "integer division or modulo by zero");
        return DIVMOD_ERROR;
    }
...
}
```

> 在`i_divmod`之后，Python的执行路径会沿着`PyErr_SetString`、`PyErr_SetObject`，一直到达`PyErr_Restore`。
> 最后，在`PyThreadState`的`curexc_type`中存放下了`PyExc_ZeroDivisionError`，而`curexc_value`中存放下了在`i_divmod`中设定的那个跟随`PyExc_ZeroDivisionError`的字符串`"integer division or modulo by zero"`。

> `PyThreadState`对象是Python为线程准备的在Python虚拟机一级保存线程状态信息的对象。
> 在Python启动，进行初始化的时候，会调用`PyThreadState_New`创建一个新的`PyThreadState`对象，并将其赋给`_PyThreadState_Current`，这个对象就是和当前的活动线程关联的线程状态对象。

> 在Python虚拟机处理异常的流程中，涉及了一个`PyTracebackObject`对象，在这个对象中记录栈帧链表的信息，Python虚拟机利用这个对象来将栈帧链表中每一个栈帧的当前状态可视化。`PyTracebackObject`对象也跟`PyFrameObject`对象一样，是一个链表结构。

> Python虚拟机意识到有异常抛出，并创建了`PyTracebackObject`对象之后，它会在当前栈帧中寻找`except`语句，以寻找开发人员指定的捕捉异常的动作，如果没有找到，那么Python虚拟机将退出当前的活动栈帧，并沿着栈帧链表向上回退到上一个栈帧。

*traceback对象链表与PyFrameObject对象链表*
![178637.jpg](/assets/analysis-of-the-python-source-code/178637.jpg)

### Python中的异常控制语义结构

```py
In [1]: import dis

In [2]: def func():
   ...:     try:
   ...:         raise Exception('i am an exception')
   ...:     except Exception as e:
   ...:         print(e)
   ...:     finally:
   ...:         print('the finally code')
   ...:

In [3]: dis.dis(func)
  2           0 SETUP_FINALLY           60 (to 62)
              2 SETUP_EXCEPT            12 (to 16)

  3           4 LOAD_GLOBAL              0 (Exception)
              6 LOAD_CONST               1 ('i am an exception')
              8 CALL_FUNCTION            1
             10 RAISE_VARARGS            1
             12 POP_BLOCK
             14 JUMP_FORWARD            42 (to 58)

  4     >>   16 DUP_TOP
             18 LOAD_GLOBAL              0 (Exception)
             20 COMPARE_OP              10 (exception match)
             22 POP_JUMP_IF_FALSE       56
             24 POP_TOP
             26 STORE_FAST               0 (e)
             28 POP_TOP
             30 SETUP_FINALLY           14 (to 46)

  5          32 LOAD_GLOBAL              1 (print)
             34 LOAD_FAST                0 (e)
             36 CALL_FUNCTION            1
             38 POP_TOP
             40 POP_BLOCK
             42 POP_EXCEPT
             44 LOAD_CONST               0 (None)
        >>   46 LOAD_CONST               0 (None)
             48 STORE_FAST               0 (e)
             50 DELETE_FAST              0 (e)
             52 END_FINALLY
             54 JUMP_FORWARD             2 (to 58)
        >>   56 END_FINALLY
        >>   58 POP_BLOCK
             60 LOAD_CONST               0 (None)

  7     >>   62 LOAD_GLOBAL              1 (print)
             64 LOAD_CONST               2 ('the finally code')
             66 CALL_FUNCTION            1
             68 POP_TOP
             70 END_FINALLY
             72 LOAD_CONST               0 (None)
             74 RETURN_VALUE
```

```c
        TARGET(RAISE_VARARGS)
            {
            u = v = w = NULL;
            switch (oparg) {
            case 3:
                u = POP(); /* traceback */
                /* Fallthrough */
            case 2:
                v = POP(); /* value */
                /* Fallthrough */
            case 1:
                w = POP(); /* exc */
            case 0: /* Fallthrough */
                why = do_raise(w, v, u);
                break;
            default:
                PyErr_SetString(PyExc_SystemError,
                           "bad RAISE_VARARGS oparg");
                why = WHY_EXCEPTION;
                break;
            }
            break;
            }
```

在`RAISE_VARARGS`指令之前，通过`LOAD_GLOBAL`、`LOAD_CONST`、`CALL_FUNCTION`构造出了一个异常对象，并将此异常对象压入运行时栈中。`RAISE_VARARGS`指令的工作就从把这个异常对象从运行时栈取出开始。这里`RAISE_VARARGS`后的指令参数是`1`，所以直接将异常对象取出赋给`w`，然后就调用`do_raise`函数。在`do_raise`中，最终将调用之前剖析过的`PyErr_Restore`函数，将异常对象存储到当前线程的状态对象中。

*Python中异常机制的流程图*
![178640.jpg](/assets/analysis-of-the-python-source-code/178640.jpg)
