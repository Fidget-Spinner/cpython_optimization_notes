# Prior work & historical ramblings

Although Python is highly dynamic, most code exhibits fairly static properties
in small regions. This is true for most dynamic languages, and has been known
for some time [^1].

Taking this example code:

```python
def f():
    ''.join(len(my_list))
```

Looking at the code, we can make a few guesses:
The `join` method will always be from the builtin type `str`. This will never
change.
`len` will likely always be the builtin `len` function, and not some custom
function written by the user. This is unlikely to change once validated.

PEP 659 improves upon prior work in many ways.

[^1]: See PyPy, or Stefan Brunthaler’s prior work in 2010: "Inline Caching meets
Quickening", and
[Yury Selianov’s email](https://mail.python.org/pipermail/python-dev/2016-January/142945.html)
on the `LOAD_GLOBAL` instruction cache

