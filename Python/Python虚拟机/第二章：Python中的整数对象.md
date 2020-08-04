## 初识PyIntObject对象

`PyIntObject`是一个“不可变”对象，它是针对`PyIntObject`对象中所维护的那个真实的整数值而言的。也就是说**创建完一个`PyIntObject`对象后，不能改变该对象的值了**。

当堆中存在大量的数字对象时，会因为频繁的访问和销毁造成很大的压力。而Python使用了**整数对象池**：运行时的整数对象并非一个个独立的对象，而是通过一定的结构联结在一起的庞大的整数对象系统。后面能看到，几乎所有的内建对象都有自己所特有的对象池机制。

现在再看一下静态的整数对象的定义——`PyIntObject`

```c
typedef struct {
    PyObject_HEAD
    long ob_ival;
} PyIntObject;
```

Python中的整数对象`PyIntObject`就是对C中`long`类型的包装，而它的类型信息`PyInt_Type`如下：

```c
PyTypeObject PyInt_Type = {
  PyVarObject_HEAD_INIT(&PyType_Type, 0)
  "int",
  sizeof(PyIntObject),
  0,

  // int类型的相关方法和属性值
  ....

  (hashfunc)int_hash,                         /* tp_hash */

};
```

`PyInt_Type`中保存了关于`PyIntObject`对象的丰富元信息：`PyIntObject`对象应该占用的内存大小，`PyIntObject`对象的文档信息，更多的是**`PyIntObject`对象所支持的操作**。

![image-20200728140743395](E:%5CNote%5CPic%5Cimage-20200728140743395.png)

在`PyInt_Type`这个元信息集合中，需要特别注意`int_as_number`这个域：

```c
static PyNumberMethods int_as_number = {

};

typedef struct {
    binaryfunc nb_add;
    binaryfunc nb_subtract;
    binaryfunc nb_multiply;
    binaryfunc nb_divide;
    binaryfunc nb_remainder;
    binaryfunc nb_divmod;
    ternaryfunc nb_power;
    unaryfunc nb_negative;
    unaryfunc nb_positive;
    unaryfunc nb_absolute;
    //...
} PyNumberMethods;
```

这个`PyNumberMethods`中定义了一个对象作为数值对象时的所有可选的操作信息，其中定义了多种可选的操作（加减乘除）。它确定了对于一个整数对象，这些数值操作应该如何进行。`PyIntObject`确实是一个不可变的对象，操作完成后参与操作的对象都没有改变，而是一个全新的`PyIntObject`诞生。

**如果加法结果有溢出，那结果就会是一个`PyLongObject`对象**。

## PyIntObject对象的创建和维护

### 对象创建的3种途径

在`intobject.h`中能看到为了创建一个`PyIntObject`对象，Python提供了5种途径：

```c
PyAPI_FUNC(PyObject *) PyInt_FromString(char*, char**, int);
#ifdef Py_USING_UNICODE
PyAPI_FUNC(PyObject *) PyInt_FromUnicode(Py_UNICODE*, Py_ssize_t, int);
#endif
PyAPI_FUNC(PyObject *) PyInt_FromLong(long);
PyAPI_FUNC(PyObject *) PyInt_FromSize_t(size_t);
PyAPI_FUNC(PyObject *) PyInt_FromSsize_t(Py_ssize_t);
```

分别是从long值、字符串、Py_UNICODE、size_t、Py_ssize_t生成，我们只需要看从long生成即可，其它的最后都是要调用此。

```c
PyObject *
PyInt_FromLong(long ival)
{
    register PyIntObject *v;
#if NSMALLNEGINTS + NSMALLPOSINTS > 0
    if (-NSMALLNEGINTS <= ival && ival < NSMALLPOSINTS) {
        v = small_ints[ival + NSMALLNEGINTS];
        Py_INCREF(v);
#ifdef COUNT_ALLOCS
        if (ival >= 0)
            quick_int_allocs++;
        else
            quick_neg_int_allocs++;
#endif
        return (PyObject *) v;
    }
#endif
    if (free_list == NULL) {
        if ((free_list = fill_free_list()) == NULL)
            return NULL;
    }
    /* Inline PyObject_New */
    v = free_list;
    free_list = (PyIntObject *)Py_TYPE(v);
    (void)PyObject_INIT(v, &PyInt_Type);
    v->ob_ival = ival;
    return (PyObject *) v;
}
```

想要理解`PyIntObject`对象创建过程，需了解Python种整数对象在内存中的组织方式。前面知道了运行时整数对象在内存中并不是独立存在，而是**形成了一个整数对象系统**。所以先考察一下Python中整数对象系统的结构。

### 小整数对象

Python中对小整数对象使用了对象池技术，因为`PyIntObject`是不可变对象，**那么对象池里的每一个`PyIntObject`对象都能被任意共享**。

`intobject.c`中的`small_ints`就是小整数对象的对象池，或者叫一个`PyIntObject`池。

### 大整数对象

Python中对于小整数，在小整数对象池中完全地缓存其`PyIntObject`对象，而对其它整数，**Python提供一块内存空间，由这些大整数轮流使用**。

`PyIntBlock`的单向列表通过`block_list`维护，**每个block维护一个PyIntObject数组——objects**，它是真正用于存储被缓存的PyIntObject对象的内存。当某个block的objects中有些内存已使用，而另外的处于空闲，Python通过一个单向链表来将这些空闲内存组织起来。链表表头是`free_list`，这样在Python需要新内存来存储`PyIntObject`时。就可以快速获取内存。

![image-20200729100833802](E:%5CNote%5CPic%5Cimage-20200729100833802.png)

### 整数对象的产生