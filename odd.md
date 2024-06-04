# Odd annotations

The way `__annotations__` work in Python has some odd corners. This document describes
these oddities.

## Converted annotations on `TypedDict` and `NamedTuple`

On normal classes, `__annotations__` holds exactly what the user wrote:

```pycon
>>> class X:
...     a: None
...     b: "str"
...
>>> X.__annotations__
{'a': None, 'b': 'str'}
```

But when NamedTuple and TypedDict are used, some types are converted during the creation
of the class:

```pycon
>>> class X(NamedTuple):
...     a: None
...     b: "str"
...
>>> X.__annotations__
{'a': <class 'NoneType'>, 'b': ForwardRef('str')}
>>> class X(TypedDict):
...     a: None
...     b: "str"
...
>>> X.__annotations__
{'a': <class 'NoneType'>, 'b': ForwardRef('str', module='__main__')}
```

(For those familiar with typing internals, this is the work of `typing._type_convert`.)

Creation of a `ForwardRef` object currently involves compiling the string into a code
object, which is relatively slow. Perhaps ironically, this means that a TypedDict with
stringified annotations (e.g., through `from __future__ import annotations`) can be
nearly 3x as slow to create as one without stringified annotations (results on Python
3.12.1):

```
% python -m timeit -s 'from typing import TypedDict' """
class X(TypedDict):
    a: int
    b: str
    c: int
    d: bool
"""
20000 loops, best of 5: 11.6 usec per loop
% python -m timeit -s 'from typing import TypedDict' """
class X(TypedDict):
    a: 'int'
    b: 'str'
    c: 'int'
    d: 'bool'
"""
10000 loops, best of 5: 28.8 usec per loop
```

They also throw an error on certain invalid annotations, courtesy of
`typing._type_check`. These are annotations that don't make sense in the type system,
but they work without error in normal classes:

```pycon
>>> class X(TypedDict):
...     a: Final
...
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File ".../python3.12/typing.py", line 2881, in __new__
    n: _type_check(tp, msg, module=tp_dict.__module__)
       ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File ".../python3.12/typing.py", line 202, in _type_check
    raise TypeError(f"Plain {arg} is not valid as type argument")
TypeError: Plain typing.Final is not valid as type argument
```

There's a few things I don't like about this behavior:

- It's inconsistent with normal classes
- It does unnecessary work at import time. Nothing in the logic for creating NamedTuples
  and TypedDicts relies on the type conversion logic for correctness.
- It's slow, and it's especially slow when `from __future__ import annotations` is
  active, even though part of the goal of that future import is to improve performance.

Unfortunately, I'm not sure this can be changed now, since existing tools that
introspect `TypedDict` may rely on this behavior. For `TypedDict` in particular, there
is the additional oddity that its `__annotations__` also contains annotations from base
classes. The converted `ForwardRef` instances preserve a reference to the module they
were defined in, but if we left the annotations as just strings, that would no longer
hold, making it more difficult to correctly resolve string annotations.

## Inherited annotations on `TypedDict`

`TypedDict` has another quirk: its `__annotations__` includes annotations from base
classes as well as those from the class itself:

```pycon
>>> class X(TypedDict):
...     a: int
...
>>> class Y(X):
...     b: str
...
>>> Y.__annotations__
{'a': <class 'int'>, 'b': <class 'str'>}
```

This is not true for normal classes:

```pycon
>>> class X:
...     a: int
...
>>> class Y(X):
...     b: str
...
>>> Y.__annotations__
{'b': <class 'str'>}
```

Or for `NamedTuple`:

```pycon
>>> class X(NamedTuple):
...     a: int
...
>>> class Y(X):
...     b: str
...
>>> Y.__annotations__
{'b': <class 'str'>}
```

Changing this would probably be too much of a compatibility break, however.

## "Simple" and non-simple annotations

Class (and module) annotations are collected from annotated variables in the body, as
introduced by [PEP 526](https://peps.python.org/pep-0526/). Normally these annotations
appear on bare variables, like `x: int = 0`, but the grammar also allows annotating
assignments where the left-hand side is an attribute or subscript node. These
assignments do not affect the `__annotations__` dictionary, which is reasonable because
it wouldn't be clear what key to use:

```pycon
>>> class X:
...     d = {}
...     d["x"]: int = 0
...     o = types.SimpleNamespace()
...     o.a: int = 0
...
>>> X.__annotations__
{}
```

More surprisingly, however, annotations are also omitted if the left-hand side is a
variable enclosed in parentheses:

```pycon
>>> class X:
...     (a): int = 0
...
>>> X.__annotations__
{}
>>> X.a
0
```

This behavior is
[explicitly specified](https://peps.python.org/pep-0526/#annotating-expressions) in PEP
526, and it is the reason why the
[`ast.AnnAssign`](https://docs.python.org/3/library/ast.html#ast.AnnAssign) node has a
`simple` attribute. I don't know why this was considered a good idea, though: it's
unintuitive that a useless set of parentheses would affect behavior.

Both mypy and pyright currently handle this incorrectly, in that they produce a
false-positive error on this working code:

```python
from dataclasses import dataclass

@dataclass
class X:
    (a): int

X()
```

Can we change this? It's core language behavior and was implemented intentionally, so
probably not.
