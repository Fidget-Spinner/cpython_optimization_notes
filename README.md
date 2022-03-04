# CPython Optimization Notes

## Introduction
[CPython](https://docs.python.org/3/glossary.html) is the reference
implementation of the Python programming language.

In this document, I’ll summarise major speedups to the core parts of CPython
in the following versions:

- 3.11

To get the most out of this, you should have knowledge of Python (especially in
data model/dunder methods). Being able to read C is also a plus though not
strictly necessary. Some knowledge of common data structures (hash tables,
linkedlist, etc.) is also beneficial.

If you’re interested in the internals of CPython, and/or want to contribute, I
believe this document has some educational value for you. If you find any parts
hard to understand due to poor wording or plain inaccuracies, please open a PR!

Dive in by reading the [index for 3.11](./3.11/README.md).

## Related Reading

I only list resources I’ve read before. I’m in no way affiliated with any of
these works (except the devguide 😉):
- Complete beginner’s guide with visual diagrams:
  https://realpython.com/cpython-source-code-guide/
- High-level overview and deep-dive of many objects and parts of CPython:
  https://github.com/zpoint/CPython-Internals
- Like the previous guide, but arranged around topics:
  https://tenthousandmeters.com/ 
- CPython devguide: https://devguide.python.org/setup/
