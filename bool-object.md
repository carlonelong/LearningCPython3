# Bool Object

“无是非之心，非人也。” ——《孟子·公孙丑上》

## Py\_True & Py\_False 对象

上一篇文章提到None对象是全局唯一的，同样True和False也是全局唯一的。在boolobject.h中

```
PyAPI_DATA(struct _longobject) _Py_FalseStruct, _Py_TrueStruct;
#define Py_False ((PyObject *) &_Py_FalseStruct)
#define Py_True ((PyObject *) &_Py_TrueStruct)
```

查看这两个对象的定义

```
struct _longobject _Py_FalseStruct = {
    PyVarObject_HEAD_INIT(&PyBool_Type, 0)
    { 0 }
};
struct _longobject _Py_TrueStruct = {
    PyVarObject_HEAD_INIT(&PyBool_Type, 1)
    { 1 }
};
```

二者都是PyLongObject，唯一的（好吧有两个）区别是在存值的地方一个是0一个是1。Python3已经没有了PyIntObject这个对象，而全部改成了PyLongObject，并冠以“int"的名称。其内存布局在下一篇文章中分析。

## PyTypeObject对象 & bool\_函数

PyBoolObject对应的PyTypeObject（去掉了没有定义的函数）

```
PyTypeObject PyBool_Type = {
    PyVarObject_HEAD_INIT(&PyType_Type, 0)
    "bool",
    sizeof(struct _longobject),
    bool_repr,                                  /* tp_repr */
    &bool_as_number,                            /* tp_as_number */
    bool_repr,                                  /* tp_str */
    Py_TPFLAGS_DEFAULT,                         /* tp_flags */
    bool_doc,                                   /* tp_doc */
    &PyLong_Type,                               /* tp_base */
    bool_new,                                   /* tp_new */
};
```

boolobject.c 是一个不到200行的小文件，里面定义了bool变量支持的各种运算。  
bool型变量只有一个创建的入口bool\_new

```
static PyObject *
bool_new(PyTypeObject *type, PyObject *args, PyObject *kwds)
{
    PyObject *x = Py_False;
    long ok;
    if (!_PyArg_NoKeywords("bool()", kwds))
        return NULL;
    if (!PyArg_UnpackTuple(args, "bool", 0, 1, &x))
        return NULL;
    ok = PyObject_IsTrue(x);
    if (ok < 0)
        return NULL;
    return PyBool_FromLong(ok);
}
```

其中 PyOBJECT\_IsTrue定义为

```
int
PyObject_IsTrue(PyObject *v)
{
    Py_ssize_t res;
    if (v == Py_True)
        return 1;
    if (v == Py_False)
        return 0;
    if (v == Py_None)
        return 0;
    else if (v->ob_type->tp_as_number != NULL &&
             v->ob_type->tp_as_number->nb_bool != NULL)
        res = (*v->ob_type->tp_as_number->nb_bool)(v);
    else if (v->ob_type->tp_as_mapping != NULL &&
             v->ob_type->tp_as_mapping->mp_length != NULL)
        res = (*v->ob_type->tp_as_mapping->mp_length)(v);
    else if (v->ob_type->tp_as_sequence != NULL &&
             v->ob_type->tp_as_sequence->sq_length != NULL)
        res = (*v->ob_type->tp_as_sequence->sq_length)(v);
    else
        return 1;
    /* if it is negative, it should be either -1 or -2 */
    return (res > 0) ? 1 : Py_SAFE_DOWNCAST(res, Py_ssize_t, int);
}
```

可见很多类型的对象都可以转成bool类型，Python解释器会依次查找每一种转换成bool类型的可能性。比如PyLongObject\(PyLong\_Type\)实现了tp\_as\_number-&gt;nb\_bool，PyListObject\(PyList\_Type\)实现了tp\_as\_sequence-&gt;sq\_length，PyDictObject\(PyDict\_Type\)实现了tp\_as\_mapping-&gt;mp\_length。这些都可以转换成bool。而自定义的类（不是内置类型的子类或者自己hack的类）对象都会走到 else 里面的 return 1逻辑。

当传入参数合法并且通过PyObject\_IsTrue转换后，会调用PyBool\_FromLong返回Py\_True或者Py\_False。

```
PyObject *PyBool_FromLong(long ok)
{
    PyObject *result;
    if (ok)
        result = Py_True;
    else
        result = Py_False;
    Py_INCREF(result);
    return result;
}
```

比如：

```
bool(3)
bool([[]])
bool({{1:1}})
```

都会返回True  
bool\_as\_number中定义的函数归定了bool变量在整数运算中的行为。

```
static PyObject *
bool_and(PyObject *a, PyObject *b)
{
    if (!PyBool_Check(a) || !PyBool_Check(b))
        return PyLong_Type.tp_as_number->nb_and(a, b);
    return PyBool_FromLong((a == Py_True) & (b == Py_True));
}
```

这个函数定义了bool变量在进行 & 运算时的操作。可以看到，其内部调用了PyLongObject的nb\_add函数，这是符合预期的。

```
True & 3
0 | False
```

等价于

```
1 & 3
0 | 0
```

而为什么

```
True == 1
False + 1 == 1
```

这些运算并没有在bool\_as\_number中体现。这些函数是在PyLongObject中定义的，在下一篇文章中会有所体现。

# 小结

1. Py\_True/Py\_False也是两个全局变量（单例）
2. bool变量之所以能从int/list/dict等对象转换，是因为这些对象都定义了一些特殊的操作。
3. bool变量定义了一些操作，以表现出整数的特性。这些操作并不全部在Py\_True/Py\_False包含的函数指针里面。
4. Markdown一点也不好用啊...



