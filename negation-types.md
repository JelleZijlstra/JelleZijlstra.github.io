# Gradual negation types and the Python type system

Recently I've been spending a lot of time thinking about theoretical aspects of the
Python type system and how various existing and proposed features fit into it. We
recently added the
["Type system concepts"](https://typing.python.org/en/latest/spec/concepts.html) chapter
to the typing spec, which provides a solid foundation for the type system in a way that
was previously lacking. In addition, Astral's new type checker,
[ty](https://github.com/astral-sh/ty), is attempting to build directly on these core
concepts and to use more set theory-based concepts in its implementation of the type
system.

These two developments have prompted me to take a closer look at possible extensions of
the type system that align with the set-theoretic framework we now use. In particular,
this article covers _negation types_: types that contain all values that are not part of
some other type. I'll first review the theoretical framework, then discuss how negation
types could fit into Python's gradual type system, and close by discussing the practical
value of negation types.

## A brief tour of the set-theoretic gradual type system

For the following discussion of negation types to make sense, we first have to review
the core structure of the type system, as outlined in more detail in
["Type system concepts"](https://typing.python.org/en/latest/spec/concepts.html).

Python's type system is conceptually based on set-theoretic types: a type is understood
as a set of possible values. For example, the type `int` includes all values that are
instance of the class `int` or a subclass at runtime, and the type `Literal[43]`
includes only instances of exactly `int` (not a subclass) with the value `43`. Subtyping
is defined through the subset relation: because all members of `Literal[43]` are also
members of `int`, `Literal[43]` is a subtype of `int` and a value of type `Literal[43]`
can be used when an `int` is expected.

But what we just said only applies to what we call _fully static types_, which represent
known sets of possible values. In addition, we have _dynamic types_, which include a
_gradual form_ such as `Any`. (This term does not appear in the spec, but it is useful
to be able to talk about dynamic types, and the term seems natural.) We use the term
_gradual types_ for all types, dynamic or fully static. A dynamic type does not refer to
a single set of possible values, but can refer to any type within a range. We call these
types _materializations_, and in general, operations on gradual types are acceptable if
there is any materialization of the gradual type that could make the operation work. The
most prominent gradual type is `Any`, which can materialize to any type.

For example, type checkers emit an error if they see an attempt to add an `int` to a
`str` (`int + str`). But `int + Any` is allowed: the `Any` could materialize to a type
that would make the `+` operation work, such as `int` or `float`. Some errors may still
be detected even if `Any` is used, though. Consider the operation `int + (str | Any)`,
adding an `int` to something that may be either a `str` or `Any`. Now, no matter what we
materialize the `Any` to (e.g., `int + str | int`, `int + str | str`,
`int + str | None`), type checkers will reject the operation, because we might be adding
an `int` and a `str`.

If types are sets, we can use set operations on them. The union operation, written as
`|`, has been part of the Python type system since the beginning. The intersection
operation, which is not currently an explicit part of the type system, is conventionally
written as `&`; for example, `int & str` denotes the set of values that are members of
both the `int` and `str` types (which happens to be the empty set). In this article, we
focus on a third operatoin: negation.

## Negation types in a gradual type system

We'll use `~T` to denote the negation type of `T`: a type that contains all values that
are not part of `T` (assuming `T` is fully static). For example, the type `~bool`
contains all values that are not instances of `bool`. In practice we usually encounter
negation types in intersections. For example, the type `int & ~bool` contains all values
of type `int` that are not instances of `bool`.

Above I was careful to specify only fully static types. The behavior of negated dynamic
types is a bit less intuitive. For example, `~Any` is in fact the same type as `Any` (in
the spec's terminology, these are equivalent types). That's because `Any` can
materialize to any fully static type, and therefore also to the negation of any other
fully static type. Thus, for any materialization of `~Any`, there is an equivalent
materialization of `Any`, and vice versa. For example, if we have the `Any` in `~Any`
materialize to `int`, the type is `~int`, and the other `Any` can materialize directly
to `~int`. If instead we have the first `Any` materialize to `~int`, the type simplifies
to `~~int = int`, and the other `Any` can materialize to `int`.

### Top and bottom materializations

We can generalize this observation to transform away all negated dynamic types, but
first we need to introduce a new concept: the _top materialization_, or `Top[T]`. For
any type `T` (dynamic or fully static), `Top[T]` is a fully static type. For a fully static
type, `Top[T]` is the same as `T`; for a dynamic type, it is the union of all
materializations, and therefore every materialization is a subtype of `Top[T]`. For
example, the type `Top[list[Any]]` includes all materializations of `list[Any]`, and
therefore all instances of `list` regardless of their generic argument.

We can similarly create a _bottom materialization_ `Bottom[T]` that is a subtype of
every materialization of `T`, or the intersection of all materializations of `T`. (We'll
need this type later.) These concepts appear in the gradual typing literature, but they
have some issues when applied to the Python type system.

First, we call it the "top materialization", but by a literal reading of the spec, this
type is not a materialization of `list[Any]` at all: the spec
[says](https://typing.python.org/en/latest/spec/concepts.html#materialization) that we
create materializations by "replacing" occurrences of `Any` with another type, but we
cannot create a union type like `list[str] | list[int]` by replacing `Any` in
`list[Any]` with a different type, let alone the infinite union that is
`Top[list[Any]]`. This is a [known defect](https://github.com/python/typing/issues/2027)
in the spec; I haven't proposed a fix yet since I need to think more about the right
replacement wording, but a better definition of materialiation would allow for the union
or intersection of two materializations to also be a materialization.

Second, and more seriously, the bottom materialization of many dynamic types will be
equivalent to `Never`. An object that is both a `list[int]` and a `list[str]`, for
example, cannot exist. This in turn means that under the spec, all of these dynamic
types are assignable to each other, since they have a shared materialization. This I
believe is another defect in the spec, but we need to think more about how to fix it.
For our purposes in this article, we can simply stick our heads in the sand and pretend
that `Never` and `Bottom[T]` are different types. It's not a very satisfying solution,
but it allows some interesting reasoning. I hope to write more about this topic later.

### Simplifying negation types

Now, with `Top[T]` available as a tool, we can formulate a rule: for a dynamic type `T`,
`~T` is equivalent to `~Top[T] | T`. This is the union of all objects that are not part
of any materialization of `T`, plus some unknown part of the dynamic type `T`.

We can apply this rule to some example types:

- `~Any` is equivalent to `~Top[Any] | Any`. `Top[Any]` is `object`, so this reduces to
  `~object | Any = Never | Any = Any`.
- `~tuple[Any] = ~Top[tuple[Any]] | tuple[Any] = ~tuple[object] | tuple[Any]`.

Whether this counts as simplifying is debatable, since it usually makes for a more
complex type. Type checkers may have a harder time reasoning about this type; for
example, `Sequence[object]` should be assignable to `~tuple[object] | tuple[Any]`
(because the latter can materialize to `~tuple[object] | tuple[object] = object`), but
assignability to unions is usually implemented by checking each arm of the union
separately, which would yield an incorrect result in this case. However, this identity
does mean that if we reason about negation types, we can reason about fully static
negation types alone without loss of generality.

There are a few other obvious rules that can help simplify negation types:

- `~~T = T`
- `~(T | U) = ~T & ~U`
- `~(T & U) = ~T | ~U`
- `(T | U) & ~V = (T & ~V) | (U & ~V)`
- `T | ~T = object` (assuming fully static types)
- `T & ~T = Never` (assuming fully static types)
- `T & ~U = Never` if `T <: U` (assuming fully static types)
- `T & ~U = Never` if `Top[T] <: Bottom[U]`
- `T & ~U = T` if `T & U = Never`

### Assignability with negation types

Given two fully static types `T` and `U`, `T` is a subtype of (or, equivalently,
assignable to) `~U` if `T` and `U` are disjoint (their intersection is `Never`). If `T`
is a dynamic type, it is assignable to the fully static type `~U` if any materialization
of `T` is disjoint from `U`. Equivalently, we can check whether `Bottom[T]` is disjoint
from `U`; if `Bottom[T]` is not disjoint from `U`, then no other materialization can be
disjoint from `U`, since all materializations are supertypes of `Bottom[T]`.

If `U` is a dynamic type, then `T` is assignable to `~U` if the intersection of
`Bottom[T]` and `Bottom[U]` is uninhabited.

### Attributes on negation types

Given a value `x: T & ~U`, how should a type checker infer the type of an expression
like `x.attr`? In most cases, once the intersection has been fully simplified, the
negative part of the intersection can simply be ignored. A type like `int & ~Literal[1]`
has the same attributes as plain `int`; `object & ~str` does not differ from `object`.
It is important, of course, that the intersection first gets simplified.
`(int | str) & ~str` does not have the same attributes as `int`.

It gets more complicated with structural types in the negative case. Consider this
program:

```python
from typing import Protocol, ReadOnly

class HasIntStr(Protocol):
    x: ReadOnly[int | str]

class HasStr(Protocol):
    x: ReadOnly[str]

obj: HasIntStr & ~HasStr
reveal_type(obj.x)
```

(Assuming that `ReadOnly` is accepted on protocol attributes, as proposed by
[PEP 767](https://peps.python.org/pep-0767).)

Naively, we might think that this should reveal `int`: `obj.x` looks like it should be
`(int | str) & ~str = int`. But there could be instances of `HasIntStr` that are not
subtypes of `HasStr`, yet might return either an `int` or a `str` when the attribute is
accessed.

```python
from typing import final

@final
class HasIntStrSneaky:
    @property
    def x(self) -> int | str:
        return 1 if random() > 0.5 else "1"

    @x.setter
    def x(self, obj: int | str) -> None:
        pass
```

Indeed, even without such a pathological implementation, since the `x` attribute could
be mutable on implementations of `HasIntStr`, the type of the `x` attribute could change
at any time from an `int` to a `str` and vice versa.

Are there any variations of the `HasIntStr` + `HasStr` intersection where we _can_
narrow the attribute type? It seems to me that we can only narrow if every value that is
a member of the `HasIntStr` equivalent type must have either an `int` or a `str`
attribute, not both.

That is the case, for example, with the following classes:

```python
from typing import NamedTuple, Final, final
from dataclasses import dataclass

class HasIntStrNamedTuple(NamedTuple):
    x: int | str

class HasIntStrFinal:
    x: Final[int | str]

    def __init__(self, x: int | str) -> None:
        self.x = x

@final
@dataclass(frozen=True)
class HasIntStrFrozen:
    x: int | str
```

There might be room for debate over how many of the finality declarations we need to
rule out subclasses that might override the attribute; type checkers differ in their
behavior in this area. However, the `NamedTuple` case at least seems indisputable.

Attribute access on a fully simplified intersection with a negated type (`T & ~U`),
then, should differ in behavior from attribute access on the positive part of the
intersection (`T`) if something like the following conditions hold:

- `U` is a Protocol containing either only the attribute under consideration, or also
  some attributes that are present with the right type on all members of `T`. (In the
  latter case, the intersection could be simplified to an equivalent one where the
  Protocol contains only the relevant attribute.)
- Every member of `T` either has an attribute of the type declared by `U`, or an
  attribute of some other type distinct from `U`; they cannot have an attribute that may
  be of either type.

Additional slight complications arise if the `U` Protocol declares a mutable attribute,
or if the relation between the types of the attribute in `T` and `U` are more complex.

It's not quite attribute access, but similar reasoning applies to other operations on
objects, such as subscripting of tuple and `TypedDict` types. Here too there are some
conceivable cases where the negative parts of an intersection can have an influence:

```python
x1: tuple[int | str, ...] & ~tuple[str, *tuple[int | str, ...]]
# could be simplified to tuple[int, *tuple[int | str, ...]]
reveal_type(x1[0])  # int

class FirstItemIsStr(Protocol):
    def __getitem__(self, key: Literal[0], /) -> str: ...

x2: tuple[int | str, ...] & ~FirstItemIsStr
reveal_type(x2[0])  # int

class TD(TypedDict):
    key: ReadOnly[int | str]

class IntTD(TypedDict):
    key: int

x3: TD & ~IntTD
reveal_type(x3["key"])  # int

class KeyIsStr(Protocol):
    def __getitem__(self, key: Literal["key"], /) -> str: ...

x4: TD & ~KeyIsStr
reveal_type(x4["key"])  # int
```

## Negation types in practice

Now that we have some understanding of how negation types fit into the type system,
let's talk about the practical use of this concept.

### Use cases

There is a longstanding [issue](https://github.com/python/typing/issues/801) on the
python/typing tracker asking for a `Not` type. Let's review some of the reasons people
want this type.

(Note that the original post in the issue actually is _not_ asking for negation types,
but for something else. I have an idea for how to support that use case, but I'll talk
about that elsewhere.)

#### Type narrowing

The most obvious use case of negation types is in type narrowing, which is very
important in practice in the Python type system. In general, the negative branch of a
type narrowing construct (the branch where the condition does _not_ hold) introduces an
implicit negation type. This is explicitly specified for `TypeIs`:

```python
from typing import TypeIs

def is_int(x: object) -> TypeIs[int]:
    # ...

x: int | str
if is_int(x):
    # x is now of type (int | str) & int = int
else:
    # x is of now of type (int | str) & ~int = str
```

Conceptually it's also how most other forms of type narrowing supported by type checkers
work. For example, a check like `x is None` narrows the original type `T` to `T & None`
in the positive case and to `T & ~None` in the negative case; `isinstance(x, int)`
narrows to `T & int` in the positive and `T & ~int` in the negative case. In most cases,
type checkers handle this through specific special cases; however, the new `ty` type
checker from Astral internally uses explicit intersections to handle almost all type
narrowing.

You can see this in the behavior of mypy and pyright on code samples like this:

```python
from typing import Sequence

def f1(x: bool | str):
    if not isinstance(x, int):
        reveal_type(x)  # (bool | str) & ~int = str

def f2(x: Sequence[str]):
    if not isinstance(x, str):
        reveal_type(x)  # Sequence[str] & ~str = (approximately) Sequence[str]
```

The existence of type narrowing means type checkers must have some understanding of
negation types to be usable. However, these are implicit negation types; they are not
written explicitly by a user. Also, because they arise only as the types of expressions,
we mostly have to worry only about assignability _from_ these negation types, not about
assignability _to_ negation types. That's a "mostly" because assignability from negation
types does come up in cases involving contravariance.

For example, consider this program:

```python
from typing import Generic, TypeVar, Sequence

T_contra = TypeVar("T_contra", contravariant=True)

class X(Generic[T_contra]):
    def __init__(self, x: T_contra) -> None: pass
    def send(self, x: T_contra) -> None: pass

def f(x: X[Sequence[str]]): pass

def doit(o: Sequence[str]):
    if not isinstance(o, str):
        x = X(o)  # X[Sequence[str] & ~str]
        f(x)  # should be rejected: X[Sequence[str] & ~str] is not a subtype of X[Sequence[str]]
```

If we reason from basic principles of negation types, this program should be rejected,
and indeed `ty` rejects it. Other type checkers allow it, however, and it's not obvious
that allowing this program would cause any practical issues.

There are other cases where mypy and pyright currently show the wrong result as compared
with the answer expected from our theoretical treatment of negation types:

```python
from typing import TypeIs, Any, reveal_type

def is_list(x: object) -> TypeIs[list[Any]]:
    raise NotImplementedError

def f1(x: list[Any] | str):
    if is_list(x):
        reveal_type(x)  # list[Any]
    else:
        reveal_type(x)  # mypy and pyright: str (incorrect, should be str | list[Any])
```

#### Basic types

A few other use cases deal with specific types in the Python type system where negation
types could be useful.

One that is brought up frequently is the fact that `str` is an `Iterable[str]` and a
`Sequence[str]`. Thus, if a function accepts `Sequence[str]`, it is legal to pass it a
single `str`, but in practice a single `str` is more likely to be a typo (the user
forgot to put the string in a list) than intended behavior. Therefore, people would like
to be able to write `Sequence[str] & ~str` or `Iterable[str] & ~str` to guard against
this issue.

Another possible use case concerns the special case in the type system where the `float`
type includes `int`. Exactly how this special case should work is still under
discussion, and many people think it should be removed entirely, but my preferred
implementation is that we treat the `float` type annotation as if it meant
`float | int`. That would mean that with negation types users would be able to write
`float & ~int` to accept only actual floats and not ints.

A third suggestion (by Randolf Scholz) is to guard against typos in a case where a
function accepts some specific set of strings, say `"spam"` and `"ham"`. If you accept
only `Literal["spam", "ham"]`, any call that has just a `str` will be rejected, which
might be too strict (what if the string comes from parsing a configuration file?). But
if you write `str | Literal["spam", "ham"]`, the type is equivalent to `str` and any
string is accepted. The suggestion, then, is to write
`(str & ~LiteralString) | Literal["spam", "ham"]`. This would reject calls where the
user wrote an incorrect string literal (`f("sham")`), but still accept arbitrary
strings.

What these use cases have in common is that they would be viral, assuming we implement
negation types soundly. If I change a function to accept only `Sequence[str] & ~str`,
any function that accepts a `Sequence[str]` and passes it on to my function would have
to be changed; either I'd have to add `assert not isinstance(x, str)` (potentially
creating a runtime error) or I'd have to change my own annotation to
`Sequence[str] & ~str` too, pushing the problem on to my callers. This would be
especially invasive with the `str & ~LiteralString` idea; users might ask for virtually
all return annotations of `str` in typeshed to be changed to `str & ~LiteralString`.

However, a nice aspect of these use cases is that they involve fully static types, so we
sidestep the greater complexity of gradual negation types.

#### Accepting all types with some marker type

Another group of use cases involves code that conceptually accepts all objects, but
treats some particular type (often `None`) specially.

An example is this use case by Alexander Martin:

```python
@overload
def foo(a: None) -> str | None:
   ...

@overload
def foo[T: ~None](a: T) -> T:
   ...

def foo(a):
    ...

foo(1)  # Literal[1]
foo(None)  # str | None
foo(object())  # both overloads match after union expansion, so object | (str | None) = object
```

The last case would require a version of the spec's provision on
[argument type expansion](https://typing.python.org/en/latest/spec/overload.html#argument-type-expansion):
we would expand `object` into `None | ~None` and see that the two arms of the union each
match a different overload.

If you squint, Harald Husum's proposed use case with `Mapping.get` is similar:

```python
class Mapping[KT, VT_co](Collection[KT]):
    @overload
    def get(self, key: ~KT, /) -> None: ...
    @overload
    def get(self, key: object, /) -> VT_co | None: ...
    @overload
    def get[T](self, key: ~KT, /, default: T) -> T: ...
    @overload
    def get[T](self, key: object, /, default: T) -> VT_co | T: ...

m: Mapping[str, int]
m.get(1)  # None
m.get(1, 2)  # Literal[2]
m.get(object())  # int | None
m.get(object(), "default")  # int | Literal["default"]
```

These use cases don't have as many issues with virality as the previous category, but
they are more likely to require advanced aspects of negation types.

### Pragmatic negation types

Above, we came up with one category of use cases (type narrowing) where type checkers
must have some notion of negation types, but only implicit negation types and only in a
somewhat limited context. There are also some use cases that would require explicit
negation types, but those are extensions of the current type system. We also saw above
that full support for gradual negation types requires significant amounts of machinery
that the current type system does not require, such as the top and bottom
materialization; in addition, it becomes very important that type checkers can correctly
check whether or not an intersection is inhabited.

Is there a simpler variation of negation types that can work well in practice? We saw
earlier that attribute access on intersections with negation types is usually equivalent
to attribute access on just the positive part of the intersection; only some very
specific negation types influence the types of attributes. It seems to me that this is
more generally true: as long as we are not assigning _to_ a type involving a negation,
and as long as we simplify negations that can be trivially simplified, we can mostly
ignore the negation type.

The pragmatic implementation of a negation type would then be: simplify what you can,
then throw away any remaining negation types in the result, and proceed as if we weren't
using negation types. As we saw above, this is largely how existing type checkers work,
but they perform some incorrect simplifications, such as
`(str | list[Any]) & ~list[Any] = str`, in the presence of dynamic types. A correct
implementation needs to implement a version of the subtyping relation for dynamic types.

What do we lose when we throw away the negation type after simplification? We already
saw that we lose some opportunities to narrow the types of attributes or subscripts,
though only in a limited set of cases. For similar reasons, we may make incorrect
assignability judgments in certain cases:

```python
from typing import NamedTuple, Protocol, ReadOnly

class HasInt(Protocol):
    attr: ReadOnly[iny]

class HasStr(Protocol):
    attr: ReadOnly[str]

class HasIntStrNT(NamedTuple):
    attr: int | str

obj: HasIntStrNT & ~HasStr
_: HasInt = obj  # should be allowed, would be rejected if the negation type is dropped
```

Of course, pragmatic negation types would cause problems if users could actually write
out generalized negation types like `Sequence[str] & ~str`, but the type system does not
currently allow those. Overall, pragmatic negation types as described in this section
should generally work well in the current type system, which lacks explicit negation
types but supports implicit negation types in type narrowing contexts.

## Conclusion

Negation types are an interesting potential extension for the Python type system, but
for the moment I believe we should not add explicit negation types to the type system;
they add significant complexity and the use cases are not significant enough to justify
that complexity. However, implicit negation types, created through type narrowing, are
very much already part of the Python type system and type checkers must support them. I
outlined an implementation strategy that should work well in practice: first simplify
intersections with negation types, then forget about any remaining negation types.

With this contribution, I aim to start a series of articles discussing aspects of the
Python type system. Possible future topics include:

- Subtyping on gradual types
- Materialization and the top and bottom materializations
- Use cases for relations between types (such as assignability and subtyping)
- "Black-box" and "white-box" attributes

## Acknowledgments

Carl Meyer and Alex Waygood helped shape my thinking on this subject and Alex also
provided useful feedback on a draft.
