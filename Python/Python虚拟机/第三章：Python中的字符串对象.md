## PyStringObject与PyString_Type

`PyStringObject`是对字符串对象的实现，它是一个拥有可变长度内存的对象。由于它是不可变的，当创建了一个`PyStringObject`后，该对象内部维护的字符串就不能再改变了。这一点特性使得`PyStringObject`对象可作为dict的键值（键值使用不可变对象）。但也使得字符串的操作（拼接）效率降低。

```c
typedef struct {
    PyObject_VAR_HEAD   // 内部有一个ob_size维护可变长度内存大小
    long ob_shash;
    int ob_sstate;
    char ob_sval[1];

    /* Invariants:
     *     ob_sval contains space for 'ob_size+1' elements.
     *     ob_sval[ob_size] == 0.
     *     ob_shash is the hash of the string or -1 if not computed yet.
     *     ob_sstate != 0 iff the string object is in stringobject.c's
     *       'interned' dictionary; in this case the two references
     *       from 'interned' to this object are *not counted* in ob_refcnt.
     */
} PyStringObject;
```

注意`PyStringObject`内部维护的字符串**末尾必须以'\0'结尾**。其中的`ob_shash`变量用来缓存该对象的hash值，可避免每次都重新计算该字符串对象的hash值。`ob_sstate`变量标记了该对象**是否已经intern机制的处理**。

下面是`PyStringObject`对应的类型对象——PyString_Type：

```c
PyTypeObject PyString_Type = {
    PyVarObject_HEAD_INIT(&PyType_Type, 0)
    "str",
    PyStringObject_SIZE,
    sizeof(char),                               /* tp_itemsize */
    string_dealloc,                             /* tp_dealloc */
    (printfunc)string_print,                    /* tp_print */
    0,                                          /* tp_getattr */
    0,                                          /* tp_setattr */
    0,                                          /* tp_compare */
    string_repr,                                /* tp_repr */
    &string_as_number,                          /* tp_as_number */
    &string_as_sequence,                        /* tp_as_sequence */
    &string_as_mapping,                         /* tp_as_mapping */
    (hashfunc)string_hash,                      /* tp_hash */
    0,                                          /* tp_call */
    string_str,                                 /* tp_str */
    PyObject_GenericGetAttr,                    /* tp_getattro */
    0,                                          /* tp_setattro */
    &string_as_buffer,                          /* tp_as_buffer */
    Py_TPFLAGS_DEFAULT | Py_TPFLAGS_CHECKTYPES |
        Py_TPFLAGS_BASETYPE | Py_TPFLAGS_STRING_SUBCLASS |
        Py_TPFLAGS_HAVE_NEWBUFFER,              /* tp_flags */
    string_doc,                                 /* tp_doc */
    0,                                          /* tp_traverse */
    0,                                          /* tp_clear */
    (richcmpfunc)string_richcompare,            /* tp_richcompare */
    0,                                          /* tp_weaklistoffset */
    0,                                          /* tp_iter */
    0,                                          /* tp_iternext */
    string_methods,                             /* tp_methods */
    0,                                          /* tp_members */
    0,                                          /* tp_getset */
    &PyBaseString_Type,                         /* tp_base */
    0,                                          /* tp_dict */
    0,                                          /* tp_descr_get */
    0,                                          /* tp_descr_set */
    0,                                          /* tp_dictoffset */
    0,                                          /* tp_init */
    0,                                          /* tp_alloc */
    string_new,                                 /* tp_new */
    PyObject_Del,                               /* tp_free */
};
```

能看到`PyStringObject`的类型对象中，`tp_itemsize`被设置为`sizeof(char)`，即一个字节。对于Python中的变长对象，这个域必须设置，因为它保存了每个元素的单位长度，它和`ob_size`共同决定了应该申请的内存总大小。

且其中的`tp_as_number`、`tp_as_sequence`、`tp_as_mapping`都被设置了，说明`PyStringObject`对数值操作、序列操作和映射操作都支持。

## 字符串对象的intern机制

当字符数组的长度为0或1时，需要进行一个特别的动作：`PyString_InternInPlace`，也就是Intern机制。此机制目的时为了对于被intern之后的字符串，在整个Python运行期间系统中都只有唯一的一个相应的`PyStringObject`。这个机制既节省了空间，又简化了对`PyStringObject`对象的比较（只需比较PyObject*即可）。`PyString_InternInPlace`负责完成对一个对象进行intern操作

```c
void PyString_InternInPlace(PyObject **p)
{
    register PyStringObject *s = (PyStringObject *)(*p);
    PyObject *t;
    if (s == NULL || !PyString_Check(s))
        Py_FatalError("PyString_InternInPlace: strings only please!");
    /* If it's a string subclass, we don't really know what putting
       it in the interned dict might do. */
    if (!PyString_CheckExact(s))
        return;
    if (PyString_CHECK_INTERNED(s))
        return;
    if (interned == NULL) {
        interned = PyDict_New();  // 没有字典则先进行创建
        if (interned == NULL) {
            PyErr_Clear(); /* Don't leave an exception */
            return;
        }
    }
    t = PyDict_GetItem(interned, (PyObject *)s);
    if (t) {
        Py_INCREF(t);  // 如果字典里已有，则增加引用计数
        Py_SETREF(*p, t);
        return;
    }
	
    // 放入到字典中
    if (PyDict_SetItem(interned, (PyObject *)s, (PyObject *)s) < 0) {
        PyErr_Clear();
        return;
    }
    /* The two references in interned are not counted by refcnt.
       The string deallocator will take care of this */
    Py_REFCNT(s) -= 2;
    // 调整s中的intern状态标志
    PyString_CHECK_INTERNED(s) = SSTATE_INTERNED_MORTAL;
}
```

此函数首先会包括两项检查内容：

- 检查传入的对象是否是个`PyStringObject`对象，intern机制只能用于`PyStringObject`对象，对于它的派生类对象也不可用。
- 检查传入的`PyStringObject`对下给你是否已经被intern机制处理过。

能注意到里面有一个`interned`是很关键的，它是在系统中的一个<Key, value>映射关系的集合，里面记录着被intern机制处理过的`PyStringObject`对象。当对一个`PyStringObject`对象a应用intern机制时，先检查`interned`中是否已存在，存在则直接将a指向它，并将原来的字符串对象引用计数减1。

![image-20200729110208838](E:%5CNote%5CPic%5Cimage-20200729110208838.png)

对于被intern机制处理了的`PyStringObject`对象，Python采用了特殊的引用计数机制。