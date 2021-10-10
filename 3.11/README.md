# Index and TODO:

## Faster CPython:

### Python startup optimizations:
- [ ] Frozen startup imports

### Faster Python function calls
- [ ] Cheaper Python frames
- [ ] Inlined Python calls

### PEP 659 Specializing Adaptive Interpreter
- [X] [Specialization](./faster_cpython/pep_659/README.md)
- [X] [Superinstructions/Combined instructions](./faster_cpython/pep_659/README.md#superinstructionscombined-instructions)
- [Opcodes](./faster_cpython/pep_659/opcodes.md):
  - [LOAD_GLOBAL](./faster_cpython/pep_659/opcodes.md#load_global)
    - [X] LOAD_GLOBAL_BUILTIN
    - [X] LOAD_GLOBAL_MODULE
    - [X] History
  - LOAD_ATTR
    - [ ] LOAD_ATTR_SPLIT_KEYS
    - [ ] LOAD_ATTR_MODULE
    - [ ] LOAD_ATTR_WITH_HINT
    - [ ] LOAD_ATTR_SLOT
  - [LOAD_METHOD](./faster_cpython/pep_659/opcodes.md#load_method)
    - [X] LOAD_METHOD_CACHED
    - [X] LOAD_METHOD_MODULE
    - [X] LOAD_METHOD_CLASS
    - [X] Background & History 
  - [BINARY_SUBSCR](./faster_cpython/pep_659/opcodes.md#binary_subscr)
    - [X] BINARY_SUBSCR_LIST_INT
    - [X] BINARY_SUBSCR_TUPLE_INT
    - [X] BINARY_SUBSCR_DICT
  - BINARY_ADD
    - [ ] BINARY_ADD_UNICODE
    - [ ] BINARY_ADD_UNICODE_INPLACE_FAST
    - [ ] BINARY_ADD_FLOAT
    - [ ] BINARY_ADD_INT
  - STORE_ATTR
    - [ ] STORE_ATTR_SPLIT_KEYS
    - [ ] STORE_ATTR_WITH_HINT
    - [ ] STORE_ATTR_SLOT
- [X] [Prior work & historical ramblings](./faster_cpython/pep_659/ramblings.md)

