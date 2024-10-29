# Why can't we ...?

A common complaint about the Python type system is that it is too verbose
and requires too many imports. There are many concrete ideas that come up
from time to time for making the syntax more concise, but these tend to run
into practical problems. This document is intended to list those ideas with
the problems associated with them.

The problems listed here aren't necessarily fatal to each idea.
Some of them may be minor enough that the increase in concision is worth
it. Others could be worked around with a tweak to the idea. If you feel
that any of these ideas is worthwhile and you have the time and energy
to drive a language change, feel free to create a PEP or open a discussion
on https://discuss.python.org to move it forward.

This document may be updated in the future with additional ideas and
additional concerns.

## What we've already achieved

Before going over ideas that probably won't fly, it's worth reminding ourselves
of the improvements we've already made over the last decade of typed Python
development:

- [PEP 526](https://peps.python.org/pep-0526/) (Python 3.6): Replaced type
  comments for variable annotations with native syntax.
- [PEP 585](https://peps.python.org/pep-0585/) (Python 3.9): Added subscripting
  support to built-in and library types, removing the need for `typing.List` and
  similar objects.
- [PEP 604](https://peps.python.org/pep-0604/) (Python 3.10): Support for using
  `|` to create union types.
- [PEP 673](https://peps.python.org/pep-0673/) (Python 3.11): Added `typing.Self`,
  replacing a common complex pattern.
- [PEP 695](https://peps.python.org/pep-0695/) (Python 3.12): Added concise, native 
  syntax for generics, removing the need for manual variance declarations.
- [PEP 563](https://peps.python.org/pep-0563/), [PEP 649](https://peps.python.org/pep-0649/),
  and [PEP 749](https://peps.python.org/pep-0749/) went through a complex history,
  but the upshot for static typing users is that they can avoid worrying about
  runtime errors in type annotations.

## Dictionary literals as inline TypedDicts

### Idea

Allow writing a `TypedDict` type inline as a dictionary literal. Instead of:

```python
from typing import TypedDict

class Movie(TypedDict):
    name: str
    year: int

def print_movie(movie: Movie) -> None:
    print(f"{movie['name']} came out in {movie['year']}")
```

Write:

```python
def print_movie(movie: {"name": str, "year": int}) -> None:
    print(f"{movie['name']} came out in {movie['year']}")
```

Or in a type alias:

```python
type Movie = {"name": str, "year": int}
```

As a bonus, this also allows using keys that are not valid identifiers,
like the existing call-based syntax for creating a TypedDict.

The `NotRequired` and `ReadOnly` type qualifiers could also be used,
e.g. `{"name": str, "year": NotRequired[int]}`.

Note that some of the below problems can be mitigated by wrapping
the dictionary, e.g. `TypedDict[{"name": str}]`. The in-preparation
PEP 764 will propose this syntax.

### Problems

_`|` operator_: The `|` operator is already defined for dictionaries,
and its behavior doesn't match the expected behavior for unions in
the type system:

```pycon
>>> {"a": int} | {"b": str}
{'a': <class 'int'>, 'b': <class 'str'>}
>>> {"a": int} | {"a": str}
{'a': <class 'str'>}
```

This would make it difficult to introspect annotations using unions
of inline TypedDicts at runtime.

Creating a union of an inline TypedDict and a type object currently
throws an error:

```pycon
>>> {"a": int} | int
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: unsupported operand type(s) for |: 'dict' and 'type'
```

However, this is fixable by changing the implementation of `__or__`
on type objects.

_Inheritance_: How would the inline syntax support inheriting from another
TypedDict? The most obvious syntax that comes to mind is
`{"name": str, **BaseMovie}`, where `BaseMovie` is another TypedDict.
We'd want this to work both with TypedDicts created through `typing.TypedDict`
and with type aliases like `type Movie = {...}`. The `**` operator works by
calling `.keys()` and then calling `.__getitem__()` for each key. But the
latter operation already has a meaning; it specializes a generic TypedDict
or type alias.

_TypedDict keyword arguments_: The class syntax for `TypedDict` currently
supports one keyword argument, `total=False`, and the draft
[PEP 728](https://peps.python.org/pep-0728/) proposes adding more. It is not
obvious how these keyword arguments would be represented in the inline syntax.

_Generics_: If a generic inline TypedDict is defined using the old `TypeAlias`
syntax, like `MyDict: TypeAlias = {"a": T}`, it is impossible to specialize
the generic at runtime; `MyDict[int]` would throw KeyError. However, the native
syntax for type aliases in Python 3.12 fixes this; you could write
`type MyDict[T] = {"a": T}` and `MyDict[int]` would work as expected, because
subscripting gets handled by the type alias object, not the dict. Still, this could be
a problem for users who want to use the inline syntax and maintain compatibility
with Python 3.11 and earlier.

## Tuples as tuple types

### Idea

Allow writing `(int, str)` as a type instead of `tuple[int, str]`.
A single-element tuple type would be written as `(int,)`. Variadic
tuple types (`tuple[int, ...]`) could potentially be written as
`(int, ...)`.

### Problems

_Presence in subscripts_: In the Python AST, `X[a, b]` and `X[(a, b)]`
are represented identically. This is a problem, because with the
proposed syntax for tuple types, these two pieces of code could
represent different types. While type checkers that use their own
parsers could distinguish these two cases, that is not possible for
type checkers that rely on the Python AST (such as mypy and pyanalyze),
and anything that introspects types at runtime also would not be able
to tell the difference.

Fixing this, so that `X[a, b]` and `X[(a, b)]` behave differently at
runtime, is difficult because it would be a backwards-incompatible
change to the language.

_`|` operator_: Tuple objects currently do not support the `|` operator,
used for building unions. We could add it and make it return a `Union`,
but it might be unintuitive for users to give this operator a typng-specific
meaning, since `|` works on other builtin collections (set, dict) with
very different behavior.

_Generics_: Generic tuple types raise similar issues as discussed
for generic inline TypedDicts (above).

## Lists, sets, and dicts representing their own types

### Idea

We could allow `[int]` instead of `list[int]`, `{int}` instead of
`set[int]`, and `{str: int}` instead of `dict[str, int]`.

### Problems

_Conflict with TypedDict syntax_: The dict part of this idea conflicts
with the suggested inline syntax for TypedDicts above. Not necessarily
a problem since both are hypothetical, but it does mean that we can pick
only one. (Why can't we allow both? `{"int": str}` could represent a
`dict[int, str]`, because type expressions may be strings, or it could be
a TypedDict with a single item.)

_`|` operator_: This operator represents a union in the type system.
However, sets and dicts already define this operator with conflicting
semantics. Lists don't, and we could in theory make `[int] | [str]` return
`Union[[int], [str]]`, but adding that operation to such a basic type would
be confusing for users.

_Hashability_: Lists, sets, and dicts are not hashable. Various parts of the
`typing.py` runtime implementation rely on hashability for efficiently
deduplicating and caching typing-related objects. There are fallbacks for
non-hashable types, but they are much slower.

_Hashability, again_: The proposed syntax for set and dict literals requires
the inner type to be hashable (only the key type for dicts). While most type
forms are hashable, this is not universally true. For example, `Annotated`
metadata may not be hashable.

_Bad incentives_: This is a more subjective one. Style guidance often
suggests using abstract, covariant types, such as `Sequence` or `Mapping`,
instead of concrete, invariant ones like `list` or `dict`. Providing overly
native-feeling syntax for list, set, and dict types goes against this advice.
The most convenient option should be the best one.

## Builtin functions as type expressions

### Idea

Three common typing-related objects have similar names to existing
builtin functions. What if we allowed users to use the builtin functions
in type expressions?

```python
x: any  # instead of typing.Any
y: iter[str]  # instead of collections.abc.Iterable[str]
z: callable[[str], int]  # instead of collections.abc.Callable[[str], int]
```

### Problems

_Muddying the waters_: Functions are not valid in type expressions. Special-casing
a few builtin functions for this would be confusing for users.

_Implementation complexity_: For this idea to work, the `iter` and `callable`
builtins would have to be made subscriptable. There is an open proposal
to make all functions subscriptable ([PEP 718](https://peps.python.org/pep-0718/)).
If that is accepted, subscripting would work, though there might be room for
confusion because its meaning is different for these functions and for
functions in general. Without PEP 718, we might have to add a new kind of builtin function just
for `iter` and `callable`, so we can add subscription support. Not out of the question,
but it makes the language core more complicated.

Similarly, these functions would have to support the `|` operator.

_Iterable or iterator_: It's not clear whether `iter` would mean `Iterable`
or `Iterator`. `Iterable` is more commonly useful in types, but the `iter()`
function returns an `Iterator`, so maybe `Iterator` would be more consistent.
Since both interpretations are plausible, users will be confused.

_Potential conflicting interpretation_: There have also been suggestions
to make a function object valid in a type expression, representing the
callable type of the function. These two ideas would conflict.

## Bare literals

### Idea

Instead of writing `Literal[]`, why not allow writing literal objects
directly as themselves, such as using `1 | 2` instead of `Literal[1, 2]`.

Note that the typing spec allows the following types inside `Literal[]`:

- `str`
- `bytes`
- `int`
- Enums
- `bool`
- `None`

### Problems

_`|` operator_: Ints, bools, and some enums already support the `|` operator
in a conflicting meaning: `1 | 2` evaluates at runtime to `3`. This would
make it impossible to introspect such literal types at runtime.

`str` and `bytes` do not currently support `|` and it could be added with a
typing-specific meaning, but as discussed in other ideas above, this would
likely be confusing for users.

This problem is especially acute for this idea because literals very frequently
show up in unions, and because the runtime optimizes away the `|` operator when
used on literals, so even an approach that looks at bytecode cannot recover the
original literals.

_Confusion with stringified annotations_: Strings already have a meaning in annotations:
they represent stringified annotations, which type checkers are supposed to
(conceptually) call `eval()` on. Making bare strings mean `Literal` types
would conflict with this existing meaning.

## Contributing

If you know of another idea that belongs on this list, or another technical
problem that should be discussed, feel free to open an issue or PR on
https://github.com/JelleZijlstra/JelleZijlstra.github.io about it.

Remember though that this document is meant purely as a list of technical
concerns, not as a discussion forum.