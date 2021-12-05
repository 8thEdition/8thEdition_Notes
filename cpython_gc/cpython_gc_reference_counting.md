> [主頁](https://hackmd.io/@8thEdition) > [深入Python記憶體回收機制(Garbage Collection)](https://hackmd.io/@8thEdition/garbage_collection) > 引用計數機制(Reference Counting)
# 引用計數機制 (Reference Counting)

在Python的世界中，萬物皆是物件(Object)。當我們在指定一個物件給一個變數時，變數實際上存的其實是物件的記憶體位置，而Python有提供id()這個函數讓我們獲得這個資訊。

---
```python
a = 1
print(id(a)) # Output: 4328795920
```
---

事實上我們可以想像成變數就像是標籤一樣，貼在被創建的物件上，而這個物件上有幾張標籤，就是我們所說的物件的引用數量(Reference Count)。實際上引用數量就在CPython定義好的PyObject當中：

---
```c
...
typedef struct _object {
    _PyObject_HEAD_EXTRA
    Py_ssize_t ob_refcnt;
    PyTypeObject *ob_type;
} PyObject;
...
```
> [CPython原始碼](https://github.com/python/cpython/blob/v3.10.0/Include/object.h#L105)
---
看到了嗎？上面的PyObject就是我們剛剛一直在說的物件喔，當中的ob_refcnt就是我們在說的物件被引用的數量啦！而當引用數量降到0時，原則上物件就會立即被回收了，這也是CPython垃圾回收最主要的機制。


那有沒有辦法可以得知物件的引用數量呢？當然有，我們可以使用sys.getrefcount()函數來得知，讓我們來試試創建整數物件，並看看它的引用數量：

---
```python
import sys

a = 1
print(sys.getrefcount(a)) # Output: 139
```
---
欸？怎麼會是139？如果照剛剛所說的，我只把a這張標籤貼在1這個物件上，引用數量不是應該是1嗎？原來是CPython為了提高效能所動的手腳。

在CPython中，-5~256這區段的整數被稱為小整數(small integer)，而由於這區段的整數最頻繁被使用，CPython為了避免不斷為這些整數分配及釋放記憶體空間，在程式執行開始已經預先為這些整數分配了記憶體位置及創建整數物件，而只要使用到這區間的整數，實際上都會拿到相同的物件喔！可以參考以下的CPython原始碼：

---
小整數的範圍就定義在[CPython原始碼](https://github.com/python/cpython/blob/v3.10.0/Include/internal/pycore_interp.h#L211)
```c
...
#define _PY_NSMALLPOSINTS           257
#define _PY_NSMALLNEGINTS           5
...
```
> 
---
而從創建int物件的其中一個函數，我們也可以看到若是屬於小整數，就會透過get_small_int -> __PyLong_GetSmallInt_internal直接回傳small_ints這個整數物件池的物件：

---
```c
...
/* Create a new int object from a C long int */
PyObject *
PyLong_FromLong(long ival)
{
    if (IS_SMALL_INT(ival)) {
        return get_small_int((sdigit)ival);
    }
    ...
}
...
```
> [CPython原始碼](https://github.com/python/cpython/blob/v3.10.0/Objects/longobject.c#L173)
---

```c
...
/* Create a new int object from a C long int */
static PyObject *
get_small_int(sdigit ival)
{
    assert(IS_SMALL_INT(ival));
    PyObject *v = __PyLong_GetSmallInt_internal(ival);
    Py_INCREF(v);
    return v;
}
...
```
> [CPython原始碼](https://github.com/python/cpython/blob/v3.10.0/Objects/longobject.c#L39)
---
```c
...
static inline PyObject* __PyLong_GetSmallInt_internal(int value)
{
    PyInterpreterState *interp = _PyInterpreterState_GET();
    assert(-_PY_NSMALLNEGINTS <= value && value < _PY_NSMALLPOSINTS);
    size_t index = _PY_NSMALLNEGINTS + value;
    PyObject *obj = (PyObject*)interp->small_ints[index];
    // _PyLong_GetZero(), _PyLong_GetOne() and get_small_int() must not be
    // called before _PyLong_Init() nor after _PyLong_Fini().
    assert(obj != NULL);
    return obj;
}
...
```
> [CPython原始碼](https://github.com/python/cpython/blob/v3.10.0/Include/internal/pycore_long.h)
---

至於為什麼數量是139呢？除了"a = 1"所貢獻的之外，CPython在執行的過程中，底層的運作邏輯也會用到數字1啦！所以才出現了139這個數字喔！不過說是這樣說，我們還是得做個實驗來證明看看事實真的是這樣嗎？

---
```python
a_256 = 256
b_256 = 256
print(f"{id(a_256)=}, {id(b_256)=}")
# Output: id(a_256)=4321302704, id(b_256)=4321302704

a_257 = 257
b_257 = 257
print(f"{id(a_257)=}, {id(b_257)=}")
# Output: id(a_257)=4324454928, id(b_257)=4324454928

```
---
a_256及b_256就如同我們剛剛所說的，會拿到CPython一開始執行就預先分配好的整數物件，因此id(a_256)與id(b_256)是相同的。但我們接著往下看a_257與b_257，不對啊！257又不在-5~256之間，拿到的物件不同，id(a_257)跟id(b_257)應該是不同的吧？理論上是沒錯的，但是CPython在將原始碼編譯為PyCodeObject時做了最佳化，會讓a_257及b_257拿到相同的257物件。我們可以用compile()這個函數來看看編譯出來的code object。

---
```python
import dis

source_code = '''\
a_257 = 257
b_257 = 257
print(f"{id(a_257)=}, {id(b_257)=}")
'''
code_object = compile(source_code, '', 'exec')
print(code_object.co_consts)
dis.dis(code_object)

'''
Output: 
code_object.co_consts=(257, 'id(a_257)=', ', id(b_257)=', None)
  1           0 LOAD_CONST               0 (257)
              2 STORE_NAME               0 (a_257)

  2           4 LOAD_CONST               0 (257)
              6 STORE_NAME               1 (b_257)

  3           8 LOAD_NAME                2 (print)
             10 LOAD_CONST               1 ('id(a_257)=')
             12 LOAD_NAME                3 (id)
             14 LOAD_NAME                0 (a_257)
             16 CALL_FUNCTION            1
             18 FORMAT_VALUE             2 (repr)
             20 LOAD_CONST               2 (', id(b_257)=')
             22 LOAD_NAME                3 (id)
             24 LOAD_NAME                1 (b_257)
             26 CALL_FUNCTION            1
             28 FORMAT_VALUE             2 (repr)
             30 BUILD_STRING             4
             32 CALL_FUNCTION            1
             34 POP_TOP
             36 LOAD_CONST               3 (None)
             38 RETURN_VALUE
'''
```
---
從上面結果可以看到code_object.co_consts中存了257這個物件，最終a_257及b_257都是拿到這個257物件喔，所以我們才會發現id(a_257)與id(b_257)是相同的！但這樣的話，我們可以怎麼證明若數字不在-5~256之前，拿到的整數物件真的會是不同的呢？很簡單，讓編譯器不知道我們要測試什麼數字就行了！

---
```python
while True:
    a = int(input())
    b = int(input())
    print(f"{a=}, {id(a)=}")
    print(f"{b=}, {id(b)=}")

'''
Input: -5
Input: -5
Output: a=-5, id(a)=4535388272
Output: b=-5, id(b)=4535388272
Input: 256
Input: 256
Output: a=256, id(a)=4535585168
Output: b=256, id(b)=4535585168
Input: 257
Input: 257
Output: a=257, id(a)=4536612016
Output: b=257, id(b)=4536612208

'''
```
---
終於跟我們想的結果一樣了！然而不小心話題也扯遠了...我們趕快再做個小實驗，再藉由sys.getrefcount()來看看引用數量的變化，以及gc.get_referrers()來看看物件到底是被誰引用的吧！


---
```python
import sys
import gc
a = 257
print(f"{sys.getrefcount(257)=}")
for ref in gc.get_referrers(257):
    print(f"{ref=}")
'''
Output:
sys.getrefcount(257)=4
ref=[b'import', b'dis', ...省略, 257, ...省略] # 編譯過程所產生的中間產物
ref=(0, None, 257, 'ref=') # co_consts
ref={'__name__': '__main__', ...省略, 'a': 257, ...省略}
'''


del a
print(f"{sys.getrefcount(257)=}")
for ref in gc.get_referrers(257):
    print(ref)
'''
Output:
sys.getrefcount(257)=3
ref=[b'import', b'dis', ...省略, 257, ...省略] # 編譯過程所產生的中間產物
ref=(0, None, 257, 'ref=') # co_consts
'''
```
---

從上述的結果來看，當我們del a這個變數後，引用數量的確減少了，不過第四行怎麼會數輸出4呢？我們接著來分析看看這些引用都是誰吧：
1. 在編譯的過程中，會掃過整份原始碼，並產生出中間物，這個中間物我們可以從gc.get_referrers()的結果看出來：ref=[b'import', b'dis', 257,... 257, 257, ...]，其中之一的"257"為第一個引用。
2. 還記得我們前面提到CPython在編譯時會做最佳化，同時把編譯時就知道的常數存進PyCodeObject的co_consts嗎？這個就是gc.get_referrers()所得到的第二個結果:ref=(0, None, 257, 'ref=')，當中的"257"就是第二個引用。
3. 至於第三個引用是誰呢？就是在呼叫sys.getrefcount()時，我們把257這個物件傳進去時，實際上sys.getrefcount()會用一個變數作為參數來接這個物件，這個就是第三個引用。不過這是暫時的，當跳出這個函數之後，引用就消失了。
4. 最後一個引用就不必多說了吧？就是我們自己宣告的a標籤！

由於前兩個都是編譯所造成的，我們在做實驗時能不能避免拿到前兩個引用呢？當然可以！就是使用我們之前的方法，只要讓編譯器在編譯時不知道我們要使用哪個數字就行啦！

---
```python
import sys
import gc
a = int(input())
print(f"{sys.getrefcount(a)=}")
for ref in gc.get_referrers(a):
    print(f"{ref=}")

'''
Intput: 257
sys.getrefcount(a)=2
ref={'__name__': '__main__', ...省略, 'a': 257, ...省略}
'''
```
---
瞧！這樣是不是就完全跟預期相同了呢！這就證明了每個物件的引用數量的確會在我們貼了一張新的標籤時增加，而實際上物件被多貼了一個標籤時，CPython就是呼叫下面這個函式，引用數量就是藉由"op->ob_refcnt++"所增加的：

---
```c
...
static inline void _Py_INCREF(PyObject *op)
{
#if defined(Py_REF_DEBUG) && defined(Py_LIMITED_API) && Py_LIMITED_API+0 >= 0x030A0000
    // Stable ABI for Python 3.10 built in debug mode.
    _Py_IncRef(op);
#else
    // Non-limited C API and limited C API for Python 3.9 and older access
    // directly PyObject.ob_refcnt.
#ifdef Py_REF_DEBUG
    _Py_RefTotal++;
#endif
    op->ob_refcnt++;
#endif
}
#define Py_INCREF(op) _Py_INCREF(_PyObject_CAST(op))
...
```
> [CPython原始碼](https://github.com/python/cpython/blob/v3.10.0/Include/object.h#L461)
---

而當我們撕了一張標籤時，CPython則是呼叫以下這個函式：

---
```c
...
static inline void _Py_DECREF(
#if defined(Py_REF_DEBUG) && !(defined(Py_LIMITED_API) && Py_LIMITED_API+0 >= 0x030A0000)
    const char *filename, int lineno,
#endif
    PyObject *op)
{
#if defined(Py_REF_DEBUG) && defined(Py_LIMITED_API) && Py_LIMITED_API+0 >= 0x030A0000
    // Stable ABI for Python 3.10 built in debug mode.
    _Py_DecRef(op);
#else
    // Non-limited C API and limited C API for Python 3.9 and older access
    // directly PyObject.ob_refcnt.
#ifdef Py_REF_DEBUG
    _Py_RefTotal--;
#endif
    if (--op->ob_refcnt != 0) {
#ifdef Py_REF_DEBUG
        if (op->ob_refcnt < 0) {
            _Py_NegativeRefcount(filename, lineno, op);
        }
#endif
    }
    else {
        _Py_Dealloc(op);
    }
#endif
}
#if defined(Py_REF_DEBUG) && !(defined(Py_LIMITED_API) && Py_LIMITED_API+0 >= 0x030A0000)
#  define Py_DECREF(op) _Py_DECREF(__FILE__, __LINE__, _PyObject_CAST(op))
#else
#  define Py_DECREF(op) _Py_DECREF(_PyObject_CAST(op))
#endif
...
```
> [CPython原始碼](https://github.com/python/cpython/blob/v3.10.0/Include/object.h#L461)
---
從上述原始碼中，我們呼叫了_Py_DECREF後，引用數量會經由"--op->ob_refcnt"減少。另外值得關注的點就是當op->ob_refcnt為0時，會呼叫_Py_Dealloc來銷毀這個物件，並且釋放其記憶體。不過實際上，物件不斷銷毀再創建對執行效率是有影響的，因此CPython根據不同的型別還設計了不同的緩存機制，例如整數就是使用到我們前面提過的小整數池(-5~256)，而我們也可以試著追追看其他型別是怎麼做緩存的，以下以浮點數作為例子：

首先我們已經知道引用數量降為0時，CPython會呼叫_Py_Dealloc，而我們從以下_Py_Dealloc的原始碼中實際上在銷毀物件時會去呼叫各型別的tp_dealloc：

---
```c
...
void
_Py_Dealloc(PyObject *op)
{
    destructor dealloc = Py_TYPE(op)->tp_dealloc;
#ifdef Py_TRACE_REFS
    _Py_ForgetReference(op);
#endif
    (*dealloc)(op);
}
...
```
> [CPython原始碼](https://github.com/python/cpython/blob/v3.10.0/Objects/object.c#L2282)
---

而浮點數的tp_dealloc是怎麼做的呢？來看下原始碼吧：

---
```c
...
#ifndef PyFloat_MAXFREELIST
#  define PyFloat_MAXFREELIST   100
#endif
...
PyTypeObject PyFloat_Type = {
    PyVarObject_HEAD_INIT(&PyType_Type, 0)
    "float",
    sizeof(PyFloatObject),
    0,
    (destructor)float_dealloc,                  /* tp_dealloc */
...
};
...
static void
float_dealloc(PyFloatObject *op)
{
#if PyFloat_MAXFREELIST > 0
    if (PyFloat_CheckExact(op)) {
        struct _Py_float_state *state = get_float_state();
#ifdef Py_DEBUG
        // float_dealloc() must not be called after _PyFloat_Fini()
        assert(state->numfree != -1);
#endif
        if (state->numfree >= PyFloat_MAXFREELIST)  {
            PyObject_Free(op);
            return;
        }
        state->numfree++;
        Py_SET_TYPE(op, (PyTypeObject *)state->free_list);
        state->free_list = op;
    }
    else
#endif
    {
        Py_TYPE(op)->tp_free((PyObject *)op);
    }
}

```
> [CPython原始碼](https://github.com/python/cpython/blob/main/Objects/floatobject.c#L1945)
---
從上述CPython的原始碼中我們可以看到，對於浮點數這個型別，CPython維護了一個free_list，當浮點數的引用數量降為0時，若free_list所緩存的數量小於100時，會先將浮點數存進free_list，否則才透過PyObject_Free直接銷毀物件。

但總覺得要看到"free"這個關鍵字才安心，畢竟這才真正代表真的有摧毀物件及釋放記憶體對吧？那我們再往下看看PyObject_Free做了什麼：

---
```c
...
void
PyObject_Free(void *ptr)
{
    _PyObject.free(_PyObject.ctx, ptr);
}
...
```
> [CPython原始碼](https://github.com/python/cpython/blob/v3.10.0/Objects/obmalloc.c#L707)
---

從上述原始碼中我們可以看到PyObject_Free又另外呼叫了_PyObject.free，若我們再往下追，會發現_PyObject.free實際上是呼叫了以下_PyObject_Free這個函數：

---
```c
...
static void
_PyObject_Free(void *ctx, void *p)
{
    /* PyObject_Free(NULL) has no effect */
    if (p == NULL) {
        return;
    }

    if (UNLIKELY(!pymalloc_free(ctx, p))) {
        /* pymalloc didn't allocate this address */
        PyMem_RawFree(p);
        raw_allocated_blocks--;
    }
}
...
```
> [CPython原始碼](https://github.com/python/cpython/blob/v3.10.0/Objects/obmalloc.c#L2229)
---
從名字上看來PyMem_RawFree就是我們要找的東西了，不過還是沒看到"free"啊！沒找到它決不罷休！來看看PyMem_RawFree做了什麼：

---
```c
...
void PyMem_RawFree(void *ptr)
{
    _PyMem_Raw.free(_PyMem_Raw.ctx, ptr);
}...
```
> [CPython原始碼](https://github.com/python/cpython/blob/v3.10.0/Objects/obmalloc.c#L593)
---

PyMem_RawFree呼叫了_PyMem_Raw.free，再往下追下去，我們發現_PyMem_Raw.free呼叫了_PyMem_RawFree：

---
```c
...
static void
_PyMem_RawFree(void *ctx, void *ptr)
{
    free(ptr);
}
...
```
> [CPython原始碼](https://github.com/python/cpython/blob/v3.10.0/Objects/obmalloc.c#L125)
---
看到"free"這個關鍵字了嗎？這就是我們要找的，終於，我們知道了CPython垃圾回收中的引用計數機制，也從CPython的原始碼中獲得了證實。

聽起來不錯，但我們一開始好像說到CPythoh的垃圾回收機制不止引用計數？這是為什麼呢？那是因為循環引用的問題，從而造成物件的引用數量永不為0，而永遠不會被釋放的問題，簡單舉個例子：

---
```python
import gc

a = []
a.append(a)
del a
print(gc.collect()) # Output 1
```
---
像上述的例子，由於指派給a的這個list物件循環引用了自己，造成引用數量為2，然而del a之後，只減少1個引用數量，那另一個怎麼辦呢？也就是我們其他兩種垃圾回收機制要處理的問題啦！也就是上面程式碼看到的gc.collect()所做的事情，而銷毀了多少物件，就是輸出的結果1囉！


## 參考
* [Unexpected value from sys.getrefcount](https://coderedirect.com/questions/228538/unexpected-value-from-sys-getrefcount)
* [The structure of .pyc files](https://nedbatchelder.com/blog/200804/the_structure_of_pyc_files.html)


