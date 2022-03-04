# Compact Object Layouts

## Background
Every object in Python has its own namespace. The namespace is usually
implemented as a dictionary, unless `__slots__` has been specified. You can
access this namespace via an object's `__dict__` attribute.

This namespace dictionary allows for dynamic modification of attributes. E.g.
```python
class Foo: ...

f = Foo()
f.y = 1
f.z = 2
```

While simplifying the implementation, the dictionary wastes space. It's yet
another CPython `PyObject` that contains additional fields for GC, refcounting,
type, etc. This is an opportunity for memory saving.

## Removing `__dict__`
The general idea is to maintain the keys and values as a dict would, but store
the pointers required to access them directly in the object itself. This means
that we don't need a dictionary object anymore and all the additional memory
overhead for maintaining that dictionary is gone.

For backwards compatibility, a new dictionary object is created when users try
to access `__dict__`. This currently requires only copying over the pointers
to the keys and values, which is cheap as it's not copying the entire keys
and values itself.

Please see [Inada-san's diagram](https://github.com/faster-cpython/ideas/issues/72#issuecomment-924568858) for a great illustration.

**Note**: [PEP 412 -- Key-Sharing Dictionary](https://www.python.org/dev/peps/pep-0412/)
means that the keys can also be shared, allowing for greater space savings!

(Ideas by Mark Shannon and Inada Naoki, implementation by Mark Shannon in
[bpo-45340](https://bugs.python.org/issue45340))
