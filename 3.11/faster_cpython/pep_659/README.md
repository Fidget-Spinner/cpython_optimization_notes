# PEP 659 Specializing Adaptive Interpreter

https://www.python.org/dev/peps/pep-0659/

This section benefits from knowledge of CPython bytecode
https://docs.python.org/3/library/dis.html.

To summarise, CPython first converts Python source code to lower-level
instructions (termed bytecode) for the CPython interpreter to execute. Most
bytecodes are extremely generalized to simplify the interpreter. This is where
the idea of specialization comes in -- if we can create specialized bytecode for
certain commonly-executed operations, then we can greatly boost their
performance! The other main idea is
[inline caching](https://en.wikipedia.org/wiki/Inline_caching), which caches
the result of an expensive operation at the instruction itself.

Specialization will have no discernible difference to end users other than
faster code. The current implementation falls back to the general instruction
when a user examines the bytecode.

(PEP 659 and initial CPython infrastructure implementation by Mark Shannon.)

## Specialization

Here are the main steps taken by the interpreter at runtime:

1. If a general instruction is executed frequently, swap out a general
   instruction for an adaptive instruction.
2. The adaptive instruction finds a specialized (faster) instruction to use for
   its current scenario. If found, swap out the adaptive instruction for a
   specialized one. Simultaneously, cache any useful (expensive) information.
3. When executing specialized bytecode, check that our assumptions hold true
   (this is called *guards*). If the guards fail, fall back on the general
   instruction. If the guards fail frequently, deoptimize the instruction back
   to the general or adaptive instruction.

### Example -- LOAD_GLOBAL_BUILTIN

An example makes this easier to understand. The following code:

```python
def f():
    len(my_list)
```

Results in the following bytecode:

```
  2           0 LOAD_GLOBAL              0 (len)
              2 LOAD_GLOBAL              1 (my_list)
              4 CALL_FUNCTION            1
```

`LOAD_GLOBAL` means Python will lookup the `len` object from the `globals()` and
`__builtins__` namespaces. While namespaces are implemented using dictionaries
with relatively fast lookups, this still adds up if you have to do it every time
the instruction is called. We can determine that most of the time,
a `LOAD_GLOBAL` is usually for loading builtins. This is an opportunity for
specialization.

Following the main steps taken by the interpreter (outlined above):

1. `LOAD_GLOBAL` becomes `LOAD_GLOBAL_ADAPTIVE` after being executed eight times.
2. `LOAD_GLOBAL_ADAPTIVE` finds that `len` belongs to the `__builtins__` module.
   So it overwrites itself with `LOAD_GLOBAL_BUILTIN`. Simultaneously, cache the
   index of the `len` object inside `__bulitins__.__dict__` (this is the
   module’s namespace).
3. When executing, check that both `globals()` and `__builtins__.__dict__`
   didn’t add new keys. This requires only two cheap integer comparisons because
   dictionary keys store an integer representing their state since CPython 3.11.
   Finally, index directly into `__builtins__.__dict__` with the cached index to
   get `len`. Again, this is very cheap compared to a dictionary lookup which
   requires hashing.

If you’ve read till here: congrats! You now understand exactly how
`LOAD_GLOBAL_BUILTIN` is implemented! `LOAD_GLOBAL` has other specialized forms,
but the most common case is accessing a builtin object.

Here’s the actual code for the steps in CPython:

1. https://github.com/python/cpython/blob/b34dd58fee707b8044beaf878962a6fa12b304dc/Python/ceval.c#L1565
2. https://github.com/python/cpython/blob/b34dd58fee707b8044beaf878962a6fa12b304dc/Python/ceval.c#L3159
   and https://github.com/python/cpython/blob/b34dd58fee707b8044beaf878962a6fa12b304dc/Python/specialize.c#L1008
3. https://github.com/python/cpython/blob/b34dd58fee707b8044beaf878962a6fa12b304dc/Python/ceval.c#L3197

`LOAD_GLOBAL_BUILTIN` was used for teaching purposes, but a similar idea has
already been implemented since Python 3.8 (using a very different
infrastructure). You can read more in "Prior work". At the time, it provided a ~
2-4% speed boost to pyperformance.

Documents providing a technical deep-dive:

- Instruction/bytecode layout and how swapping
  works https://github.com/python/cpython/blob/b34dd58fee707b8044beaf878962a6fa12b304dc/Python/specialize.c#L15
- Adding adaptive instructions how-to
  guide https://github.com/python/cpython/blob/b34dd58fee707b8044beaf878962a6fa12b304dc/Python/adaptive.md

## SuperInstructions/Combined Instructions

Currently, the adaptive interpreter infrastructure is also used to create
combined instructions. At runtime, the interpreter will look for common opcode
pairs and combine their behavior. The first instruction in the pair is then
rewritten with the new combined instruction. This eliminates eval-loop overhead
by avoiding DISPATCH() macro. E.g:

```
LOAD_FAST   0
LOAD_FAST   1
LOAD_CONST  0
```

Becomes:

```
LOAD_FAST__LOAD_FAST 0
LOAD_FAST            1
LOAD_CONST           0
```

You can think of this as two `LOAD_FAST` together. The combined instructions do
not use different bytecode format: they take only one
oparg. `LOAD_FAST__LOAD_FAST` will increment program counter inline [^1] to read
the oparg of the instruction after it without going through eval loop, then on
dispatch it will effectively "skip" the next instruction and go to LOAD_CONST.
The speedup comes from skipping eval-loop and dispatch overhead of one
instruction. The bytecode format is unchanged.

Currently, combined instructions do not yield great speedups (except in tuple
unpacking microbenchmarks). That’s because there’s high overhead in other areas
in CPython. Once other optimizations are applied, interpreter overhead will be
greater and combined instructions will shine.

[^1]
LOAD_FAST__LOAD_FAST: https://github.com/python/cpython/blob/07cf10bafc8f6e1fcc82c10d97d3452325fc7c04/Python/ceval.c#L1755

Specializer code
at https://github.com/python/cpython/blob/07cf10bafc8f6e1fcc82c10d97d3452325fc7c04/Python/specialize.c#L353

### FAQ: Why not do this with the compiler?

Disclaimer: This is just my personal opinion. I may be wrong.

From a performance perspective, it’s definitely better to emit combined
instructions at the last part of the compile step. However, the main benefit of
using the interpreter specializer is to hide combined instructions from normal
Python code. Although the bytecodes are unstable, they are a public interface
used by many projects. Exposing combined instructions will mean breaking a lot
of library code every time we change them. The other benefit is that we don’t
have to write public user docs since they’re a private implementation detail ;-)
. During tracing or inspection, the old (non-combined) instruction is shown
instead.

### Specialized Instructions
To learn more about the specialized opcodes, read them [here](./opcodes.md).
