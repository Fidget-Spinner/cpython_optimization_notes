# Tier 2 UOps Optimization Passes

Author: Ken Jin, with implementation help by Jules Poon.

This document explains how we (the authors) plan to optimize CPython superblocks.


## Overview

After translation of Tier 1 specialized bytecode to Tier 2 micro
operations, said micro operations need optimization to eventually
outperform Tier 1.

As with optimization in other languages, we require the following
phases:
- Analysis
- Optimization
- Code generation

We will not cover code generation as it is kept simple for the tier
2 interpreter.

The optimized micro operations will then be passed to an exeuction
engine, that is either the interpreter or the copy-and-patch JIT
compiler.

## Analysis

We plan to generate majority of the analysis and optimization passes
automatically, by defining bytecode semantics in the interpreter defintion.
This will reduce the maintenance burden and lines of code that has to be
checked into CPython. The general goal is that while certain instructions with
special meaning to CPython have to be handwritten, the
vast majority of other Python instructions's behavior in the
interpreter can be automatically inferred from the DSL.

To generate the intermediate representation, and perform
some optimizations, we shall use an abstract interpreter over the
Tier 2 uops. The abstract interpreter operates on and generates the IR
below by operating on symbolic terms (as defined in the paper
"SSA Translation Is an Abstract Interpretation" by Matthieu Lemerre
in POPL 2023).

CPython uops are then classified into 3 main types
in the interpreter DSL:
1. ``pure`` - like in functional languages,
these instructions have no side effects to the user, they
allow for greater optimizations.
2. ``guard`` - these instructions pass nodes through themselves.
3. ``impure`` - everything that is not the above.


### Intermediate Representation (IR)

The intended intermediate representation is the micro-ops themselves.

## Optimization

All the following optimizations are done in a single pass together with
the analysis stage:

### Type Propagation
We perform these automatically with the abstract interpreter's symbolic
terms. As part of a term's type, we also encode auxillary type information
in the type system, for example, dictionary versions or type verions.

These rules follow deductive steps. Type information is propagated forwards.
For example, in the IR:
```
# locals: [a: unknown, b: unknown, c: unknown]
# stack:  [a: unknown, b: unknown]
GUARD_BOTH_INT
# locals: [a: int, b: int, c: unknown]
# stack:  [a: int, b: int]
```

The key part is that type and constant propagation is done automatically,
without having to handwrite these deductive rules ourselves. Types are
expressed as part of the interpreter DSL:

Note that the following output above implies some sort of value numbering.

```
// Typed outputs means the types are these types after the operation.
pure op(_BINARY_OP_ADD_INT, (left, right -- res: ~(PYLONG_TYPE))) {
}

// Typed output with auxillary type information (type_version)
guard op(_GUARD_TYPE_VERSION, (type_version/2, owner -- owner: ~(GUARD_TYPE_VERSION_TYPE + type_version))) {
}
```

The abstract interpreter's then automatically generates nearly all of the
cases. With types, we can then eliminate type guards that are not
required, thus shrinking the code we eventually pass to the JIT
compiler.

### Constant propagation and evaluation

Like with type propagation, the abstract interpreter automatically
evaluates the bodies of pure instructions with their constant values.
We get this for free.

### Guard elimination
We eliminate guards when we see they are redundant using constant
or type information.

### True function inlining

CPython 3.11 obtained frame inlining. This eliminated most of the
overhead of function calls. However, "fake frames"
were still needed. These are the instructions `_PUSH_FRAME` and `_POP_FRAME`.
A similar idea was introduced in Cinder (Instagram's fork of CPython).

In CPython 3.13, we plan for true function inlining -- that is, no
frame creation at all in the happy case, not even fake frames.

The uops tracer already projects traces through Python function boundaries.
The goal is to eliminate `_PUSH_FRAME` and `_POP_FRAME`. Doing this is easy,
but doing this while maintaining CPython semantics is very hard.

The goal is to change this trace of instructions:
```
ADD_TO_TRACE(_CHECK_PEP_523, 1, 0)
ADD_TO_TRACE(_CHECK_FUNCTION_EXACT_ARGS, 1, 4828)
ADD_TO_TRACE(_CHECK_STACK_SPACE, 1, 0)
# Frame creation
ADD_TO_TRACE(_INIT_CALL_PY_EXACT_ARGS, 1, 0)
ADD_TO_TRACE(_SAVE_RETURN_OFFSET, 4, 0)
ADD_TO_TRACE(_PUSH_FRAME, 1, 0)
ADD_TO_TRACE(_SET_IP, 0, 0)
ADD_TO_TRACE(_CHECK_VALIDITY, 0, 0)
ADD_TO_TRACE(_RESUME_CHECK, 0, 0)
...
# Frame removal
ADD_TO_TRACE(_POP_FRAME, 0, 0)
```
to this:
```
_CHECK_PEP_523(1, 12, 0)
_CHECK_FUNCTION_EXACT_ARGS(1, 12, 4828)
_CHECK_STACK_SPACE(1, 12, 0)
_SETUP_TIER2_FRAME(128, 0, 0)
 # Minimal bookkeeping, no frame creation
_PRE_INLINE(7, 0, 39)
_SET_FRAME_NAMES(0, 0, 93831592813424)
...
_POST_INLINE(4, 0, 18446744073709551615)
_SET_FRAME_NAMES(0, 0, 93831592813424)
```


#### Locals interleaving

By interleaving the locals of the new call with the stack of the old
function frame, we can achieve zero argument copying.

Before a function call, the stack looks like this:

| callable | self_or_null | arg1 | arg2              |
|----------|--------------|------|-------------------|
|          |              |      | ^ (stack pointer) |

Notice that the arguments are already laid out as how the locals
of the new "frame" will be. Thus, we just need to expand the stack
to accomodate for more locals. Finally, since we are now pointing
into the current stack as our locals, we offset all `LOAD_FAST`
and `STORE_FAST` into the current stack.

| callable | self_or_null                              | arg1 | arg2 |                   |
|----------|-------------------------------------------|------|------|-------------------|
|          | ^ `LOAD_FAST 0 + offset` of inlined frame |      |      | ^ (stack pointer) |

Finally, we must update the attribute names' tuple that the current 
frame points to, so that the inlined call's attribute loads load
the correct attribute names. This is just a simple assignment.

#### Compatiblity
100% compatibility with all existing CPython code, and
all uses of the C API. However, we do not claim compatiblity with native
stack/frame profilers that do not go through the C API.

The core idea is that in the happy case we have no frames, and
enjoy inlined functions. In the not-so-happy case, such as tracebacks,
deoptimizations, ``sys._getframe``, or ``locals()``, we
can reconstruct all information we need on-demand. We do this by storing
reconstruction information at the end of a trace, and keep a pointer
to it in the current frame. We update the pointer in the `_PRE_INLINE`
and `_POST_INLINE` instructions. This makes the happy case much faster.

As a proof of concept, the author has already implemented full function
inlining with reconstruction in their branch of the tracer. It has
full compatibility with tracebacks and deoptimizations. Thus
proving that frame reconstruction is indeed feasible.

### Object creation inlining

Python object creation currently
requires the pushing and popping of at least two "fake frames". One for
a wrapper around `__init__`, and one for `__init__` itself.

By applying the same ideas in true function inlining, we can eliminate
all function frame require for object creation.

This is not to be confused with scalar replacement.

### Limited value numbering

A limited form of (value numbering)[https://en.wikipedia.org/wiki/Value_numbering] is 
```
x = y
# This has _GUARD_BOTH_INT
y + y
# should no longer have _GUARD_BOTH_INT
x + x
```

We want the second guard to be eliminated. The analyzer should see
that they are the same values.


### [Tentative] Common subexpression elimination

We will factor our common expressions by hash-consing the symbolic term
nodes.

This is tentative, as the cost of this on the optimizer's memory
and speed footprint is yet to be determined.

## Memory usage
The intermediate representation's expression nodes shall be allocated
from a bump allocator. So there will be a fixed memory cost associated
with that part. Representing frames and the interpreter state can also
be pre-calculated.


Code generation just uses a simple fixed length array.
