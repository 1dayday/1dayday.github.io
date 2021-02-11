---
layout: post
title:  Python中的Dict对象
date:   2020-08-06 22:43:34 +0800
tags:   Python CPython Python源码剖析 散列表
---
* TOC
{:toc}

摘自：[《Python源码剖析》 — 陈儒](https://read.douban.com/ebook/1499455/)

> C++的STL中的`map`就是一种关联容器，`map`的实现基于RB-tree（红黑树）。
> RB-tree是一种平衡二元树，能够提供良好的搜索效率，理论上，其搜索的时间复杂度为O(log2N)。\
> Python中同样提供关联式容器，即`PyDictObject`对象（也称`dict`）。
> `PyDictObject`没有如`map`一样采用平衡二元树，而是采用了散列表(hash table)，因为理论上，在最优情况下，散列表能提供O(1)复杂度的搜索效率。

> 装载率是散列表中已使用空间和总空间的比值。举例来说，如果散列表一共可以容纳10个元素，而当前已经装入了6个元素，那么装载率就是6/10。研究表明，当散列表的装载率大于2/3时，散列冲突发生的概率就会大大增加。

Python中采用开放定址法来解决散列冲突问题。在采用开放定址的冲突解决策略的散列表中，删除某条探测链上的元素时不能进行真正的删除，而是进行一种“伪删除”操作，必须要让该元素还存在于探测链上，担当承前启后的重任。

**[`PyDictObject`](https://github.com/python/cpython/blob/v2.7.18/Include/dictobject.h#L50-L89)**

```c
typedef struct {
    /* Cached hash code of me_key.  Note that hash codes are C longs.
     * We have to use Py_ssize_t instead because dict_popitem() abuses
     * me_hash to hold a search finger.
     */
    Py_ssize_t me_hash;
    PyObject *me_key;
    PyObject *me_value;
} PyDictEntry;

/*
To ensure the lookup algorithm terminates, there must be at least one Unused
slot (NULL key) in the table.
The value ma_fill is the number of non-NULL keys (sum of Active and Dummy);
ma_used is the number of non-NULL, non-dummy keys (== the number of non-NULL
values == the number of Active items).
To avoid slowing down lookups on a near-full table, we resize the table when
it's two-thirds full.
*/
typedef struct _dictobject PyDictObject;
struct _dictobject {
    PyObject_HEAD
    Py_ssize_t ma_fill;  /* # Active + # Dummy */
    Py_ssize_t ma_used;  /* # Active */

    /* The table contains ma_mask + 1 slots, and that's a power of 2.
     * We store the mask instead of the size because the mask is more
     * frequently needed.
     */
    Py_ssize_t ma_mask;

    /* ma_table points to ma_smalltable for small tables, else to
     * additional malloc'ed memory.  ma_table is never NULL!  This rule
     * saves repeated runtime null-tests in the workhorse getitem and
     * setitem calls.
     */
    PyDictEntry *ma_table;
    PyDictEntry *(*ma_lookup)(PyDictObject *mp, PyObject *key, long hash);
    PyDictEntry ma_smalltable[PyDict_MINSIZE];
};
```

Python 2.x中使用了一种关联容器`PyDictEntry`，在`PyDictEntry`中，`me_hash`域存储的是`me_key`的散列值，利用一个域来记录这个散列值可以避免每次查询的时候都要重新计算一遍散列值。
`PyDictEntry`对象可以在3中状态间转换：Unused态、Active态和Dummy态，如图所示。
![178446.jpg](/assets/analysis-of-the-python-source-code/178446.jpg)

`PyDictObject`对象实际上是一大堆`PyDictEntry`的集合，其中`ma_table`域是关联对象的关键所在。
当`PyDictEntry`数量少于8个(`PyDict_MINSIZE`)，`ma_table`域将指向`ma_smalltable`的起始地址；
当`PyDictEntry`数量超过8个时，Python将会申请额外的内存空间，并将`ma_table`指向这块空间。
![178447.jpg](/assets/analysis-of-the-python-source-code/178447.jpg)

## PyDictObject对象的创建和维护

### 对象创建 `PyObject * PyDict_New(void)`

1. 自动初始化全局Dummy态`PyDictEntry`对象
2. 创建`PyDictObject`对象，优先复用对象缓存池资源
3. 将`ma_smalltable`清零，同时设置`ma_fill`和`ma_used`均为`0`
4. 将`ma_table`指向`ma_smalltable`，并设置`ma_mask`为`7`

### 元素搜索策略

> Python为`PyDictObject`对象提供了两种搜索策略，[`lookdict`](https://github.com/python/cpython/blob/v2.7.18/Objects/dictobject.c#L295-L396)和[`lookdict_string`](https://github.com/python/cpython/blob/v2.7.18/Objects/dictobject.c#L398-L457)。实际上，这两种策略使用的是相同的算法，`lookdict_string`只是`lookdict`的一种针对`PyStringObject`对象的特殊形式。

```c
/*
The basic lookup function used by all operations.
This is based on Algorithm D from Knuth Vol. 3, Sec. 6.4.
Open addressing is preferred over chaining since the link overhead for
chaining would be substantial (100% with typical malloc overhead).

The initial probe index is computed as hash mod the table size. Subsequent
probe indices are computed as explained earlier.

All arithmetic on hash should ignore overflow.

(The details in this version are due to Tim Peters, building on many past
contributions by Reimer Behrends, Jyrki Alakuijala, Vladimir Marangozov and
Christian Tismer).

lookdict() is general-purpose, and may return NULL if (and only if) a
comparison raises an exception (this was new in Python 2.5).
lookdict_string() below is specialized to string keys, comparison of which can
never raise an exception; that function can never return NULL.  For both, when
the key isn't found a PyDictEntry* is returned for which the me_value field is
NULL; this is the slot in the dict at which the key would have been found, and
the caller can (if it wishes) add the <key, value> pair to the returned
PyDictEntry*.
*/
static PyDictEntry *
lookdict(PyDictObject *mp, PyObject *key, register long hash)
{
    register size_t i;
    register size_t perturb;
    register PyDictEntry *freeslot;
    register size_t mask = (size_t)mp->ma_mask;
    PyDictEntry *ep0 = mp->ma_table;
    register PyDictEntry *ep;
    register int cmp;
    PyObject *startkey;

    i = (size_t)hash & mask;
    ep = &ep0[i];
    if (ep->me_key == NULL || ep->me_key == key)
        return ep;

    if (ep->me_key == dummy)
        freeslot = ep;
    else {
        if (ep->me_hash == hash) {
            startkey = ep->me_key;
            Py_INCREF(startkey);
            cmp = PyObject_RichCompareBool(startkey, key, Py_EQ);
            Py_DECREF(startkey);
            if (cmp < 0)
                return NULL;
            if (ep0 == mp->ma_table && ep->me_key == startkey) {
                if (cmp > 0)
                    return ep;
            }
            else {
                /* The compare did major nasty stuff to the
                 * dict:  start over.
                 * XXX A clever adversary could prevent this
                 * XXX from terminating.
                 */
                return lookdict(mp, key, hash);
            }
        }
        freeslot = NULL;
    }

    /* In the loop, me_key == dummy is by far (factor of 100s) the
       least likely outcome, so test for that last. */
    for (perturb = hash; ; perturb >>= PERTURB_SHIFT) {
        i = (i << 2) + i + perturb + 1;
        ep = &ep0[i & mask];
        if (ep->me_key == NULL)
            return freeslot == NULL ? ep : freeslot;
        if (ep->me_key == key)
            return ep;
        if (ep->me_hash == hash && ep->me_key != dummy) {
            startkey = ep->me_key;
            Py_INCREF(startkey);
            cmp = PyObject_RichCompareBool(startkey, key, Py_EQ);
            Py_DECREF(startkey);
            if (cmp < 0)
                return NULL;
            if (ep0 == mp->ma_table && ep->me_key == startkey) {
                if (cmp > 0)
                    return ep;
            }
            else {
                /* The compare did major nasty stuff to the
                 * dict:  start over.
                 * XXX A clever adversary could prevent this
                 * XXX from terminating.
                 */
                return lookdict(mp, key, hash);
            }
        }
        else if (ep->me_key == dummy && freeslot == NULL)
            freeslot = ep;
    }
    assert(0);          /* NOT REACHED */
    return 0;
}
```

1. 根据hash值取模（采用掩码位与运算）获得冲突链上第一个`PyDictEntry`索引下标
2. 检查冲突链上第一个`PyDictEntry`对象
    * 在两种情况下搜索结束
        * `PyDictEntry`处于Unused态，表明搜索失败
        * `PyDictEntry`的`me_key`与参数`key`**值相同**，搜索成功
    * 当`PyDictEntry`处于Dummy态，则将其记录到`freeslot`变量
    * 当`PyDictEntry`处于Active态，若key**值相同**则搜索成功
3. 根据Python所采用的探测函数，遍历探测链上后续的`PyDictEntry`对象
    * 若检查到一个Unused态的对象
        * 如果`freeslot`不为空，则返回`freeslot`所指的对象（插入时优先修改Dummy态的对象）
        * 如果`freeslot`为空，则返回当前Unused态对象
    * 若key**值相同**，则搜索成功
    * 若hash值相同且对象非Dummy态（即Active态）
        * 若key**值相同**则搜索成功
    * 若检查到一个Dummy态对象且`freeslot`为空，则将其记录到`freeslot`变量

> `lookdict_string`背后有一个假设，即待搜索的key是一个`PyStringObject`对象。只有在这种假设成立的情况下，`lookdict_string`才会被使用。

> `lookdict_string`实际上就是一个`lookdict`对于`PyStringDict`对象的优化版本。在`lookdict`中有许多捕捉错误并处理错误的代码，因为`lookdict`面对的是`PyObject*`，所以会出现很多意外情况；而在`lookdict_string`中，完全没有了这些处理错误的代码。而另一方面，在`lookdict`中，使用的是非常通用的`PyObject_RichCompareBool`，而`lookdict_string`使用的是`_PyString_Eq`，要简单很多，这些因素使得`lookdict_string`的搜索效率要比`lookdict`高很多。

### 插入与删除

**插入**

1. 计算key的hash值
    * 如果key为`PyStringObject`则直接获取其`ob_shash`缓存
    * 若`ob_shash`未计算或key不为字符串对象，则调用`PyObject_Hash`计算hash值
2. 在`PyDictObject`中查询key
    * 搜索成功，返回Active态`PyDictEntry`对象，直接替换`me_value`
    * 搜索失败，返回Dummy态或Unused对象，则需完整设置`me_key`、`me_hash`和`me_value`
3. 如有必要则调整`PyDictObject`内存空间
    * 若**装载率 >= 2/3**则重新调整最小为`mp->ma_used * (mp->ma_used > 50000 ? 2 : 4)`
    * `dictresize`从`PyDict_MINSIZE`（默认为8）开始，以指数方式增加大小，直到超过上述最小值为止
    * `PyDict_SetItem()`保证在仅改变值的情况下（不改变装载率）不会调整`PyDictObject`对象的内存空间，这样可以保证在使用`PyDict_Next()`遍历字典的时候仅更改值是安全的，但不能增删key！

**删除**

1. 计算key的hash值
2. 在`PyDictObject`中搜索相应的`PyDictEntry`对象
    * 搜索失败则返回异常
    * 搜索成功则将对象置为Dummy态，并调整相关计数

[PyCon 2010: The Mighty Dictionary](https://www.youtube.com/watch?v=C4Kc8xzcA68)
视频结尾的QA中提到Python 2.x并**不会在减少`PyDictObject`元素数量时减少内存空间占用**，这需要将所有的KV复制到一个新的`PyDictObject`对象中！

## PyDictObject对象缓冲池

`PyDictObject`使用了与`PyListObject`一样的对象缓冲池机制，默认大小为80个。
