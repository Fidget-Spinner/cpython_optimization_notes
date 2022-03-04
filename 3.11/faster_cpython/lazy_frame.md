# Cheaper, Lazy Python Frames

## Background

Frame objects in CPython are created when calling a Python function. Note that
these frame objects are not the frames found in C, but rather custom
objects (structs) meant to represent a function's runtime state. This allows for
runtime frame inspection (used by debuggers), and the implementation of
generators, which pause and resume state.

For detailed information, please read the [excellent guide by zpoint](
https://github.com/zpoint/CPython-Internals/blob/master/Interpreter/frame/frame.md).

## Cheaper Frame Creation

How CPython created frames (`PyFrameObject`) pre-3.11 was to treat it as yet
another Python object (`PyObject`). This comes with some overhead -- frames
have to be allocated, and come with GC and refcounting data. Some optimizations
such as zombie frames were already present (see the link above), but 3.11 goes
further.

In 3.11, a new `InterpreterFrame` is used internally. This is a simple C struct
containing the bare information we need for execution, and no extra fields just 
for introspection/debugging or refcounting/GC. Each thread's state
(`PyThreadState`) holds a linked list of spare memory ("chunks"). On every new
frame creation, CPython finds an available chunk and uses it for our frame. This
requires simple pointer operations, and avoids any memory allocation. Memory is
only allocated if the chunk is too small to hold all the frame data, which is
rare. When the frame is done executing, the chunk is "popped" (unlinked), but
not deallocated. On next frame creation, these chunks are reused, making them
extremely cheap. This also means a frame which cannot fit into a chunk becomes
less likely over time.

## Lazy Frames

The name "lazy" means that the full pre-3.11 `PyFrameObject` is only created when
the user tries to directly access it via something like `sys._getframe()`.
This maintains compatibility with older code. Luckily, most normal user code
doesn't touch the frame object directly and benefit by using cheaper
frames. Read more [here](https://github.com/python/cpython/pull/27077#issuecomment-878221201).

Altogether, this sped up pyperformance by
[3-7%](https://github.com/python/cpython/pull/27077#issuecomment-881561647)!.

(Contributed by Mark Shannon in https://bugs.python.org/issue44590)