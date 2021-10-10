# Specialized Opcodes

Does not include combined/super instructions.

## LOAD_GLOBAL

The compiler emits `LOAD_GLOBAL` when it sees a name that isn't assigned in the
local scope, and isn't in a closure.

Overall speedup: Unknown, but likely small in pyperformance. `LOAD_GLOBAL` already
had opcache (see “History”).

(Specialization and porting over from older infrastructure by Mark
in https://bugs.python.org/issue44338).

### LOAD_GLOBAL_BUILTIN

(Covered in-depth in [this example](./README.md#example----load_global_builtin))

Specialized for loading values from the `__builtins__` module.
The `__builtins__`
namespace is implemented as a dictionary of the names of builtin objects mapped
to the actual values. If you want to verify this for yourself,
run ``vars(__builtins__)`` or ``__builtins__.__dict__``.

The implementation stores the index of the object inside `__builtins__.__dict__`
. You might be wondering why a dictionary has an index? A Python dictionary
contains an array of integer indices. These indices lead to the actual object
inside the dictionary (assuming no hash collision). To learn more, I recommend
reading either https://github.com/zpoint/CPython-Internals or watching Raymond
Hettinger's PyCon talks on dictionaries.

This reduces LOAD_GLOBAL from two dictionary lookups (one for globals() and one
for `__builtins__.__dict__`) to zero. Effectively making `LOAD_GLOBAL` free (
other than some pointer arithmetic and interpreter overhead). `LOAD_GLOBAL` can
be made faster by only unboxing values or JIT.

### LOAD_GLOBAL_MODULE

Specialized for loading a global variable in `globals()`. Avoids one dict lookup
in `globals()` by caching index of object in the `globals()` dictionary. Need to
check that the dictionary keys' version of `globals()` didn't change.

### History

`LOAD_GLOBAL_BUILTIN`'s concept isn't entirely new. It was implemented under a
different infrastructure called the "opcache".

In Python 3.8, Yury and Inada-san worked on
a [`LOAD_GLOBAL` inline cache](https://github.com/python/cpython/pull/12884)
which cached the address (
pointer) of the lookup result. This saved us two dictionary lookups into
`globals()` and `__builtins__` on every `len` function call assuming both
dictionaries didn't change. The optimization sped up `LOAD_GLOBAL` by 40%. This
was a gigantic leap forward for CPython at the time.

The newer implementation has a few major improvements:

- The old opcache stored a pointer to the actual object required. This requires
  more stringent checks (otherwise we could be accessing a dangling pointer!).
  In 3.11, only the index of the dict is stored. Which is safe once the key is
  checked (this is a cheap pointer comparison). This is not only safer, but
  allows for the next optimization:
- The old opcache used the version information of the entire dict as a guard (
  see PEP 509). This means when any value in globals() or __builtins__ changed,
  the cache was invalidated. In the new 3.11 version, only the version of the
  dictionary's *keys* are used. Since we only cache the index (and not the
  object)
  , we don't need to care if the value changed, the location for a key's value
  is still at the same index (as long as the keys didn't change too)! This makes
  cache invalidation less frequent.
- A new LOAD_GLOBAL_MODULE for uncommon case of variable in `globals()`.

## LOAD_ATTR

(Specialization and porting over by Mark in https://bugs.python.org/issue44337.
Some forms were originally implemented in 3.10 using opcache by Pablo, Yury and
Guido.)
https://bugs.python.org/issue42093
https://bugs.python.org/issue42927)

### LOAD_ATTR_SPLIT_KEYS

### LOAD_ATTR_MODULE

### LOAD_ATTR_WITH_HINT

### LOAD_ATTR_SLOT

## LOAD_METHOD

`LOAD_METHOD` is a compiler-level specialization for method lookups and calls of
the form `obj.meth()`.

Overall speedup from specialization:
[1.02x faster pyperformance](https://github.com/python/cpython/pull/27722#issuecomment-898308598)
in initial implementation.

Read “Background & History” to learn more about `LOAD_METHOD`, and the type MRO
method cache (and `tp_version_tag`).

(Specialization by me, reviewed by Mark in https://bugs.python.org/issue44889.)

### LOAD_METHOD_CACHED

Any object that successfully loads an unbound method is specialized to this. The
object *must not* have an attribute shadowing the method name. At specialization
time, the unbound method object is cached (borrowed reference). For a method
call of the form `self.meth()`, at runtime, we need to verify that:

1. `type(self)`'s MRO was not modified. This is done via validating `type(self)
   .tp_version_tag` (the type's unique version tag). This is a unique uint32
   that resets to 0 whenever anything along a type's MRO (or the
   type's `__dict__` itself) is modified. It is also used in Python 2.6's method
   lookup cache (see [“Background & History”](#background-&-history)).

2. `self.__dict__` keys didn't change by checking dict key's version. This
   ensures that the method isn't shadowed by an attribute. If the object has
   no `__dict__`: great! skip this step.

The normal `LOAD_METHOD` convention is to call `_PyObject_GetMethod` which:

1. Searches through `type(self).__mro__` for the method via `_PyType_Lookup`.
   Applies descriptor protocol if necessary.
2. If `self` has a namespace (`__dict__`), check if the method is shadowed
   inside `self`'s namespace. If found, return that, else return step 1.'s
   result.

So we removed the `_PyType_Lookup` call, and the dictionary lookup in `self.__
dict__.` For normal Python classes, this is a 12% speedup in microbenchmarks. The
speedup is even greater for types with long MRO due to inheritance. For built-in
types (such as `dict`, `list`, `int`), there is almost no speedup for their method
loads since they have no `__dict__`, and the method is likely already cached
in `_PyType_Lookup`'s method cache.

With two cheap uint32 comparisons and some pointer arithmetic, we can
immediately return a cached unbound method. This is the fastest possible method
loading convention without reworking the object layout or a JIT.

Whacky note: notice we didn't even need the method's name! Normal LOAD_METHOD
requires the name of the method being loaded, `LOAD_METHOD_CACHED` doesn't.

You might also be wondering how we can verify that the borrowed reference to an
unbound method doesn't become a dangling pointer. I wrote a long explanation
here https://github.com/python/cpython/blob/b34dd58fee707b8044beaf878962a6fa12b304dc/Python/specialize.c#L973-L986
. This is probably my favourite part of it all. The code initially looked really
suspicious and made me somewhat uneasy :).

### LOAD_METHOD_MODULE

Specializing `module.meth()`. Copy of LOAD_ATTR_MODULE.

### LOAD_METHOD_CLASS

Specializing `cls.classmeth()`. We avoid the method and dict lookup, but it's
still somewhat slow because of bound classmethod creation.

### Background & History

In Python 2.6, a first-level lookup method lookup cache was implemented to speed
up all method lookups. This is a hashtable mapping `tp_version_tag` (type's MRO
version tag) and a method's name to a descriptor `PyObject *` (borrowed)
. (https://github.com/python/cpython/blob/8c1e1da565bca9cec792323eb728e288715ef7c4/Objects/typeobject.c#L3795)
. The idea is that if a type's MRO (and the MRO of any types it inherits from)
isn't modified, then we can just return the cached descriptor. This is used in
every _PyType_Lookup.

The CPython 3.11 LOAD_METHOD inline cache improves on the type method cache in a
few ways. It's faster because it avoids a few function calls and hashing.
Additionally, type method caches work well for programs that call a small set of
methods frequently. In CPython, the type method cache can only store 4096
methods. This means a program calling a wide range of Python methods along with
builtin methods may exhaust the cache. LOAD_METHOD specialization may suffer
from polymorphic method calling code, but that's quite uncommon in Python (and
also already covered by the type method cache).

#### LOAD_METHOD in the old opcache infrastructure

Similar to LOAD_GLOBAL and LOAD_ATTR, I'll be comparing the 3.11 version with
the old opcache version. There was an attempt to add LOAD_METHOD to the opcache
infrastructure in 3.10. But the benchmark numbers at the time were reportedly
not good. So it was never
added https://github.com/python/cpython/pull/23503/files

After writing the 3.11 implementation, I stumbled upon that PR when trying to
debug some obscure method cache bug. It's partially thanks to that PR I was able
to find out that the `tp_version_tag` was
bugged https://bugs.python.org/issue44914.

The idea is mostly the same, but the 3.11 version has some improvements which
allowed it to be faster:
- 3.11 emits more `LOAD_METHOD` due to `CALL_METHOD_KW`. Previously method calls with
keywords still used `LOAD_ATTR` and
`CALL_FUNCTION_KW` https://github.com/python/cpython/pull/26014.
- 3.11 requires less guards.
- 3.11 specializes for some other common forms.
- Most importantly: 3.11 has no dictionary lookup to determine if things are an 
  attribute! If the 10 version used ma_version_tag to avoid a dictionary lookup,
  it might've been as fast as the 3.11 version. (or maybe not? since python
  classes set their  instance attributes often, which means frequent opcache
  invalidation in 3.10).

It's only thanks to the 3.11 infrastructure that we can achieve greater
speedups. Nonetheless, 3.10's opcache was a beast which I have much respect for
paving the way to what we have in 3.11 today :).

#### What is LOAD_METHOD and CALL_METHOD?

Previously, method calls used `LOAD_ATTR` and `CALL_FUNCTION`. `LOAD_METHOD` and
`CALL_METHOD` were added in CPython 3.7 (first done by PyPy). In short, it speeds
up method calls by avoiding temporary objects (“bound methods”). When you write
self.meth(), usually the method `meth` takes in `self` as the first argument (
this is also called an instance method). How Python actually implements this is
by doing the equivalent logic in C:

```python
bound_method = type(self).meth.__get__(self)
bound_method()
```

(To learn more about `__get__`, read up on the descriptor protocol). The point is
that a temporary object called a “bound method” is eventually created just to
get our behavior above. A bound method is nothing more than a wrapper object to
pass `self` as the first argument to a callable being wrapped [1].

On every call, the temporary bound method is created then immediately discarded
after the call. If you call a method frequently, this is really expensive! So in
3.7, the `LOAD_METHOD` and `CALL_METHOD` opcode pairs were created. These opcodes
skip the bound method creation together by using the underlying descriptors and
calling them with `self`. Effectively:

```python
unbound_method = type(self).meth
unbound_method(self)
```

So we got a ~20% speedup for method calls in 3.7.

[1]:
To verify my claims:

```python
>>> type([].sort)
<class 'builtin_function_or_method'>
>>> type(list.sort)
<class 'method_descriptor'>
```

`builtin_function_or_method` is a bound method for a C method, while
`method_descriptor` is the actual method being wrapped! In CPython, there exists
multiple bound methods catering for different callables. Bound methods for C
methods (as seen above), builtin slot wrappers (e.g `''.__repr__`), python
methods (instancemethod) to name a few.

In CPython, to signal you accept `self` as the first argument, any unbound
method needs to set `PY_TPFLAGS_METHOD_DESCRIPTOR` in type flags.

## BINARY_SUBSCR

`BINARY_SUBSCR` is the instruction for subscription of the form `a[b]`.

Overall speedup from specialization:
[1.02x faster pyperformance](https://github.com/python/cpython/pull/27043#issuecomment-880070349)
in initial implementation.

(Specialization by Irit, reviewed by Mark in https://bugs.python.org/issue26280
.)

### BINARY_SUBSCR_LIST_INT, BINARY_SUBSCR_TUPLE_INT

Specialized form for subscripting a list or tuple object with a number that fits
into a C machine int (usually 32 bits). So `list[int]` or `tuple[int]`. The speedup
comes from avoiding a function call altogether! Normally Python has to do the
equivalent of this in C for a binary subscription:

```python
result = type(a).__getitem__(a, b)
```

In C, this calls the `__getitem__` function pointer. For lists and tuples, this
will handle edge cases like b being a gigantic long, or slice, etc. However,
lists and tuples are implemented as a C array of pointers to other objects. So
the fastest way would be to index directly:

```python
result = a[b]
```

This makes the operation consume zero function calls, and makes subscripting
list and tuples 30% faster!

Note: Profile Guided Optimization (PGO) can actually optimize frequent function
pointer calls and add a fast inline path (effectively doing what this
specialization does). However, that depends on representative PGO training data
and isn't too reliable. Also `__getitem__` may be overridden frequently, causing
this optimization to not be applied.

### BINARY_SUBSCR_DICT

Specialized form for subscripting a dict object with anything (`dict[Any]`). The
speedup comes from avoiding a function call through a function pointer, and
instead directly calling the dictionary subscript function in CPython. This
makes subscripting dictionaries 10% faster!

## BINARY_ADD

### BINARY_ADD_UNICODE

### BINARY_ADD_UNICODE_INPLACE_FAST

### BINARY_ADD_FLOAT

### BINARY_ADD_INT

## STORE_ATTR

### STORE_ATTR_SPLIT_KEYS

### STORE_ATTR_WITH_HINT

### STORE_ATTR_SLOT
