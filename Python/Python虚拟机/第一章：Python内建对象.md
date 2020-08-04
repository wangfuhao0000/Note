Python中一切都是对象：一个整数、字符串、一种类型等。

## Python内的对象

Python中对象就是为C中的结构体在堆上申请的一块内存，一般对象不能被静态初始化，但唯一例外的是**类型对象：Python中所有的内建的类型对象都是被静态初始化的**。Python中一个对象一旦被创建，在内存中大小不变，这样的规则使得通过指针维护对象的工作变得简单。不然当一个对象进行扩展时，可能会因影响其它对象而改变地址，此时所有指向此对象的引用都需要改变。

### 对象的基石——PyObject

Python中所有的对象都拥有一些相同的内容，它们在`PyObject`中定义：

```c
typedef struct _object {
    _PyObject_HEAD_EXTRA
    Py_ssize_t ob_refcnt;   // 基于引用计数的垃圾收集机制
    PyTypeObject *ob_type;  // 类型信息
} PyObject;
```

`ob_type`时一个指向`_typeobject`结构体的指针，它对应着Python内部的一种特殊对象，用来指定一个对象类型的类型对象。

所以对象机制的核心很简单：一个是**引用计数**，一个是**类型信息**

```c
#define _PyObject_HEAD_EXTRA            \
    struct _object *_ob_next;           \
    struct _object *_ob_prev;
```

`PyObject`只是定义了每个Python对象都必须由的内容，除此以外不同对象还会站有一些额外的内存放置其它信息。例如Python中的整数对象：

```c
[intobject.h]
typedef struct {
    PyObject_HEAD
    long ob_ival;   // 存储真正的值
} PyIntObject;
```

### 定长对象和变长对象

整数对象的特殊信息是一个C中的整型变量，但如果是类似字符串的“可变”对象，就需要一个特殊的结构体来`PyVarObject`表示这一类“可变的”对象：

```C
#define PyObject_VAR_HEAD               \
    PyObject_HEAD                       \
    Py_ssize_t ob_size; /* Number of items in variable part */

typedef struct {
    PyObject_VAR_HEAD
} PyVarObject;
```

因为变长对象大小不确定，**所以相对于PyObject_HEAD来说Py_VAR_HEAD多了字段`ob_size`的产生**，变长对象通常是容器，而这个属性指明了边长对象一共容纳了多少个元素（而不是字节数）。

能看出`PyVarObject`实际上只是对`PyObject`的一个扩展，所以每个对象仍然是拥有相同的对象头部。这使得在Python内部对对象的引用变得非常同意，**我们只需用一个`PyObject`指针就可以引用任意的一个对象**。

## 类型对象

当申请对象空间时需要知道需要的内存大小，它是对象的元信息，此元信息与对象所属类型密切相关，可以先看一下类型对象`_typeobject`：

```c
typedef struct _typeobject {
    PyObject_VAR_HEAD
    const char *tp_name; /* For printing, in format "<module>.<name>" */
    Py_ssize_t tp_basicsize, tp_itemsize; /* For allocation */

    /* Methods to implement standard operations */

    destructor tp_dealloc;
    printfunc tp_print;
    getattrfunc tp_getattr;
    setattrfunc tp_setattr;
    cmpfunc tp_compare;
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
} PyTypeObject;
```

在`_typeobject`包含了4类信息：

- 类型名，`tp_name`，用于Python内部及调试时使用
- 创建该类型对象时分配内存空间大小的信息，`tp_basicsize`和`tp_itemsize`
- 与该类型对象相关联的操作信息
- 类型的类型信息

其实一个`PyTypeObject`对象就是Python中的“类”概念。

### 对象的创建



### 对象的行为

`PyTypeObject`中定义了大量的函数指针，它会指向某个函数或者NULL，它们可以视为**类型对象中所定义的操作**。例如`tp_hash`指明对于该类型对象，如何生成hash值等。**`PyTypeObject`中指定的不同操作信息也是一种对象区别于另一种对象的关键所在**。

`PyTypeObject`中有三组非常重要的操作族：`tp_as_number`、`tp_as_sequence`、`tp_as_mapping`，它们分别指向PyNumberMethods、PySequenceMethods和PyMappingMethods函数族：

```c
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
    inquiry nb_nonzero;
    unaryfunc nb_invert;
    binaryfunc nb_lshift;
    binaryfunc nb_rshift;
    binaryfunc nb_and;
    binaryfunc nb_xor;
    binaryfunc nb_or;
    coercion nb_coerce;
    unaryfunc nb_int;
    unaryfunc nb_long;
    unaryfunc nb_float;
    unaryfunc nb_oct;
    unaryfunc nb_hex;
    /* Added in release 2.0 */
    binaryfunc nb_inplace_add;
    binaryfunc nb_inplace_subtract;
    binaryfunc nb_inplace_multiply;
    binaryfunc nb_inplace_divide;
    binaryfunc nb_inplace_remainder;
    ternaryfunc nb_inplace_power;
    binaryfunc nb_inplace_lshift;
    binaryfunc nb_inplace_rshift;
    binaryfunc nb_inplace_and;
    binaryfunc nb_inplace_xor;
    binaryfunc nb_inplace_or;

    /* Added in release 2.2 */
    /* The following require the Py_TPFLAGS_HAVE_CLASS flag */
    binaryfunc nb_floor_divide;
    binaryfunc nb_true_divide;
    binaryfunc nb_inplace_floor_divide;
    binaryfunc nb_inplace_true_divide;

    /* Added in release 2.5 */
    unaryfunc nb_index;
} PyNumberMethods;
```

在PyNumberMethods中，定义了作为一个数值对象应该支持的操作。例如一个对象可被视为数值对象，则对应的类型对象PyInt_Type中，`tp_as_number.nb_add`就指定了对该对象进行加法操作时的具体行为。同样PySequenceMethods和PyMappingMethods分别定义了作为一个**序列对象（list）**和**关联对象（dict）**应该支持的行为。

PyTypeObject中允许一种类型同时指定三种不同对象的行为特性。

### 类型的类型

注意到`PyTypeObject`定义的开始有一个`PyObject_VAR_HEAD`，这表示Python中的类型实际也是个对象，而判断一个对象是否是一个类型是通过`PyType_Type`判断的：

```c
PyTypeObject PyType_Type = {
    PyVarObject_HEAD_INIT(&PyType_Type, 0)
    "type",                                     /* tp_name */
    sizeof(PyHeapTypeObject),                   /* tp_basicsize */
    sizeof(PyMemberDef),                        /* tp_itemsize */
    (destructor)type_dealloc,                   /* tp_dealloc */
    0,                                          /* tp_print */
    0,                                          /* tp_getattr */
    0,                                          /* tp_setattr */
    0,                                          /* tp_reserved */
    (reprfunc)type_repr,                        /* tp_repr */
    0,                                          /* tp_as_number */
    0,                                          /* tp_as_sequence */
    0,                                          /* tp_as_mapping */
    0,                                          /* tp_hash */
    (ternaryfunc)type_call,                     /* tp_call */
    0,                                          /* tp_str */
    (getattrofunc)type_getattro,                /* tp_getattro */
    (setattrofunc)type_setattro,                /* tp_setattro */
    0,                                          /* tp_as_buffer */
};
```

PyType_Type很重要，**所有用户自定义class对应的PyTypeObject对象都是通过此对象创建**。当我们打印某个类型的`__class__`时总是打印`<type 'type'>`，而它时所有class的class（Metaclass）。然后看一下PyIntType是怎么和PyType_type建立关系的：

![image-20200728113510817](E:%5CNote%5CPic%5Cimage-20200728113510817.png)

## Python对象的多态性

通过PyObject和PyTypeObject，Python利用C语言完成了对象的多态性。Python创建一个对象时会分配内存并初始化，然后Python内部会用一个**`PyObject*`变量，而不是一个`PyIntObject*`变量来保存和维护这个对象**。所以在Python内部各个函数之间传递的都是一种泛型指针——**PyObject***。只能从指针所指对象的`ob_type`域动态进行判断对象的类型，而这个域也让Python实现了多态。

例如下面的例子：

```c
void Print (PyObject* object)
{
    object->ob_type->tp_print(object);
}
```

传递给`Print`函数不同的类型（例如`PyIntObject*`）就会调用对应的`tp_print`函数。

## 引用计数

Python通过对一个对象的引用计数的管理来维护对象在内存中的存在与否，即对象中的`ob_reefcnt`变量。Python中主要通过`Py_INCREF(op)`和`Py_DECREF(op)`两个宏来增加和减少一个对象的引用计数，**当一个对象的引用计数减少到0后，`Py_DECREF`将调用该对象的析构函数来释放该对象所占有的内存和资源**。实际这个析构动作是通过在对象对应的**类型对象中定义**的一个函数指针`tp_dealloc`实现的。

这里触发对象销毁的时间隐约透着**Observer模式**，即当`ob_refcnt`减为0时触发。要注意Python中的类型对象是超越引用计数规则的，即永远不会被析构。

但要注意调用析构函数并不意味着一定会调用`free`来释放内存，否则频繁地申请、释放内存会使Python效率降低。Python中大量使用了内存对象池的技术，可以避免频繁地申请和释放内存空间。

