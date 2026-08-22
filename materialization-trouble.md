# The trouble with materialization and assignability

Python has a gradual type system, using types like `Any` to integrate dynamic or
untyped code into its type system. The intuition I use to explain `Any` is that an
operation involving `Any` is allowed if there can be a precise type replacing
the `Any` that makes the operation work.

The [current typing specification](https://typing.python.org/en/latest/spec/concepts.html#materialization)
attempts to formalize this intuition with a notion called *materialization*:

> Given a gradual type A, if we replace zero or more occurrences of Any in A with some type (which can be different for each occurrence of Any), the resulting gradual type B is a materialization of A.

In turn, a gradual type A is assignable to another type B if there is some materialization
of A that is a subtype of some materialization of B. It's an elegant concept once you understand it, but it turns
out that it is not a full description of behaviors we want from Python's type system.

## Three claims

Before I outline the specific problems, I want to make three claims about these
issues.

- *These problems are mostly theoretical*. The problems identified here manifest
  in the assignability relation between types. But we now have several completely
  independent type checkers in Python, and for the most part they all agree on the
  correct outcome for the issues identified here. That makes me think we need to
  find some theoretical model that can explain these behaviors; we mostly don't need
  to identify the correct behaviors.
- *But they do need to be fixed*. Type checkers agree on the basics, but
  we need tools to understand how they ought to behave in more exotic cases, or
  when new features are added to the type system. Indeed many of the hardest cases
  involve intersection types, a feature that is not currently in the standardized type system
  but that may be added in the future. Without a solid theoretical foundation,
  it is difficult to ensure new features behave consistently.
- *They need to be fixed together*. Below I identify a number of distinct
  problems, but they all relate to the same general theme in which the spec's
  current definition of assignability doesn't fit the behavior we want. Addressing only
  some of the problems without dealing with the rest still leaves us with a
  specification that doesn't describe the actual type system; we need to
  fix the problem comprehensively.

## Replacing Any isn't enough

I reported [an issue](https://github.com/python/typing/issues/2027) last year about one
problem: if materialization only happens through replacing `Any`, then a type
`list[Any]` cannot be assignable to `list[int] | list[str]`, or vice versa. Replacing the
single `Any` cannot produce a union of two incompatible kinds of lists. Thus, the current
definition implies that `list[Any]` is not equivalent to `list[Any] | list[Any]`, though the
spec [explicitly says](https://typing.python.org/en/latest/spec/concepts.html#union-types)
that these are equivalent. This same
problem appears in nested positions: type checkers today allow `list[list[Any]]` to be
consistent with `list[list[int] | list[str]]`, and that seems desirable.

A second problem concerns recursive types. Consider:

```python
type R = list[tuple[Any, R]]
type InnerUnion = list[tuple[int, InnerUnion] | tuple[str, InnerUnion]]
type Alternating = list[tuple[int, list[tuple[str, Alternating]]]]
```

Current type checkers accept both `InnerUnion` and `Alternating` as assignable
to `R`, but neither is derivable by replacing the single use of `Any` in the definition of `R`.
`InnerUnion` can be understood as a materialization of `R` if the element type of
`R` is expanded to `tuple[Any, R] | tuple[Any, R]` and the two occurrences of
`Any` are replaced with `int` and `str`. `Alternating` needs a different kind of
expansion: we must first look through the recursive occurrence of `R` one more time,
so that the outer `Any` can become `int` and the next `Any` can become `str`.

Substitution-based materialization also fails for gradual types that appear in
generic parameters. Consider a generic type `Co[T]` whose type parameter is covariant
(for example, `frozenset[T]`), and recall that `int` and `str` are disjoint. Existing
type checkers generally treat types such as `Co[int | Any]` and `Co[int] | Co[str]`
as consistent with each other. That behavior suggests that `Co[int] | Co[str]`
should be understood as one possible materialization of `Co[int | Any]`.

A rule that only copies gradual expressions before substitution does not produce this
materialization. It can copy the whole `Co[int | Any]`, or it can copy the inner
`int | Any` inside the type argument, but it provides no way to lift the resulting
split through the covariant type constructor to obtain a union of two `Co[...]` types.

If intersection types are added to the type system, additional problems appear.
Similar to the first problem with unions, self-intersections (`T & T` for the same `T`,
where `T` is a type) involving gradual types are not equivalent to their basic type. Consider the types
`int | tuple[Any]` and `(int | tuple[Any]) & (int | tuple[Any])`. The first must
materialize to some type that is a union of `int` with a one-element tuple type.
But the second can materialize to, for example, `(int | tuple[int]) & (int | tuple[str])`,
which simplifies to `int` if we assume that the two tuple types are disjoint.
But note that this is in a sense the opposite of the problem with unions: there introducing
the union allowed a materialization that we want to exist; here introducing the intersection
allows a materialization we probably don't want to exist.

Distributivity of unions is also lost. Consider the types `list[Any] & (list[int] | list[str])`
and `(list[Any] & list[int]) | (list[Any] & list[str])`. By ordinary distributivity
rules, these are the same type. Yet by substituting Any, the second can materialize to
`(list[int] & list[int]) | (list[int] & list[str])` = `list[int] | list[str]`, while the
first can only materialize to `list[int]`, `list[str]`, or `Never`.

## Materialization-based assignability also runs into trouble

The most important relation between types in our type system is assignability:
if we have something of type A, can we assign it to a target of type B?
In Python, assignability for fully static types is defined by subtyping, which in turn is based on set containment:
is the set of values contained in type A a subset of the set for type B? For gradual types, we need an intermediate step of
materialization: is there a pair of materializations A' and B' of A and B such
that A' is a subtype of B'?

Most of the problems listed above concern too few materializations being allowed,
but sometimes there are also too many. Consider the type `tuple[Any]`, any single-element
tuple. If we replace its `Any` with `Never`, we get `tuple[Never]`: a single-element tuple
with a non-existent element. Such a tuple cannot exist (a one-element tuple must
have an element that exists), so this type is uninhabited and equivalent to `Never`.
And `Never` is a subtype of every type. Therefore, `tuple[Any]` (which can materialize to `Never`)
is also assignable to every type. We probably don't want that!

This argument holds within containers too, so it is not enough to ban materializing the
whole type to `Never`. For example, the types `list[tuple[Any]]` and `list[tuple[Any, Any]]`
can both materialize to a type equivalent to `list[Never]` by substituting `Any` with `Never`
and observing that a fixed-size tuple with a `Never` element cannot exist. `list[Never]`
is not itself an uninhabited type (an empty list can inhabit it), but it is a common materialization,
and thus `list[tuple[Any]]` and `list[tuple[Any, Any]]` are mutually assignable.

Adding intersections to the type system increases the scope of this problem.
A type like `int & Any` (which intuitively denotes some unknown subset of the ints)
can materialize to `Never` and is therefore assignable to every other type. This is not as
clearly undesirable as the above examples, and indeed ty, a type checker that already has
good support for intersections, currently allows this assignment. Nevertheless, I believe
this is undesirable; it makes intersections involving `Any` significantly less safe and useful.

## What, if anything, is the bottom materialization?

The [top and bottom materializations](https://jellezijlstra.github.io/negation-types#top-and-bottom-materializations)
are helpful concepts for thinking about gradual types and materializations in some languages.
The top materialization is a supertype of all other materializations; the bottom materialization
is a subtype of all others. This allows for an understanding of gradual types as an interval
between two types, the top and bottom materializations. That's considerably more appealing than
using raw materialization, which produces a potentially infinite family of types.

However, the bottom materialization is a problematic concept in Python if we see types purely
as sets of values. Many bottom materializations (e.g., `Bottom[list[Any]]` or `Bottom[tuple[Any]]`)
have no members and are therefore logically equivalent to `Never`. Therefore, there are types
that are subtypes of `Top[list[Any]]` and supertypes of `Bottom[list[Any]]`, but that are not
materializations of `list[Any]`: instances of subclasses of `list`.

We don't exactly need the bottom materialization to exist as an independent concept; there
are useful models of the type system that can do without it. But it is a useful tool, and
we should figure out whether it can be made a coherent part of the type system.

## Plugging the holes

I've spent some time thinking about ways to solve these problems, but haven't come up with
a fully satisfactory solution. Here are a few approaches:

- *Plug the holes one by one*. Expand the definition of materialization to cover missing
  materializations; tweak the definition of assignability so undesirable assignability
  relations go away. Perhaps this can be made to work, but new adjustments often lead
  to new problems, and we lose the elegance of the original definitions.
- *Distinct Nevers*. Extend the type system with some distinct types that are empty
  but not the same as `Never`. This allows the bottom materialization to be a meaningful
  part of the type system, and it plugs some holes related to assignability.
  However, it needs a new set of definitions for these extra types, and we risk
  infecting the type system with lots of mostly meaningless empty types that
  appear as a result of type narrowing.
- *Dropping materialization*. The materialization relation is not directly important
  to most of the type system; assignability is. One approach, then, is to stop
  using materialization as the primary tool for defining gradual types, and instead
  directly define assignability on gradual types using a set of explicit rules
  of the kind "`A | B` is assignable to `C` if `A` and `B` are both assignable to `C`".
  I currently feel this is most promising, but it is not as elegant as the original
  model based on materialization.

More work is needed to solve these problems.
