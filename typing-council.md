# The Python Typing Council

I recently proposed [PEP 729](https://peps.python.org/pep-0729/), which
created a Typing Council to govern the Python type system, and now that
the PEP has been accepted, I am one of the inaugural members of the
Council. I am writing this document to gather some of my own thoughts
on how I'll make decisions while I'm on the Council.

Note that these are my personal thoughts, and other members of the Council
may not agree.

## Spec changes

Many of the decisions the Council will have to make will be around
spec changes. In fact, this is among the main reasons I proposed the
Council: we often find ourselves in broad agreement that something should
change, but there was no good way to codify that agreement.

We need to be careful about spec changes, because if the spec diverges from
what type checkers actually do, the spec loses its usefulness.

Here are some points I will look for when evaluating proposed changes
to the spec:

* What is the right behavior? Concretely: what is most useful to users,
  what is consistent with the rest of the type system, what makes the
  type system sound.
* What do type checkers currently do? If all type checkers currently behave
  one way, but the spec says something else, we should almost certainly
  change the spec.
* Are type checkers willing to implement the proposed behavior? More commonly,
  there will be some disagreement among type checkers. If type checker
  maintainers aren't able or willing to change their tool's behavior,
  we may not be ready for a change.
* Is the proposed behavior well-specified? Much of the current spec is vague.
  We should gradually work to improve that, and in particular new spec
  changes should be clear and unambiguous.

All proposed spec changes should be accompanied by changes to the
conformance test suite (once it exists).

## Stability

The Council's mandate contains three parts, and one of them is that the
type system should be stable. That's important because typing is now
an important tool for many real-world Python users, and we want their
code to continue working. For example, if library authors release a
library that exposes types using semantics specified in the PEPS or
now in the typing spec, they should reasonably expect that type checkers
in the future will interpret those semantics the same as they do now.

Backwards compatibility is very important (and often controversial)
to the Python community. We went through a huge compatibility break
a few years ago with the move from Python 2 to 3, and I think it's widely
agreed that transition was not handled well—though ultimately it
definitely led us to a better language.

However, CPython's backwards compatibility rules cannot be applied
directly to the type system. The type system does not have a release
cycle as such. Type checkers release much more frequently than
the interpreter does, and inevitably many of the changes they make
break compatibility in the sense that they introduce new errors or
change inferred types. Still, we should be very hesitant to mandate
changes to established semantics, especially for constructs that
have gone through the PEP process.

To be clear, the standard backwards compatibility rules do fully apply
to the runtime objects exported by the `typing` module, and I would
hold the `typing_extensions` module to the same standard. In other words,
I would consider approving a change that tweaks what `TypeGuard[]` means
to a type checker, but any change that affects runtime execution (such as
removing `typing.TypeGuard`) would need to go through a cycle of raising
`DeprecationWaring`.

## PEPs

One of the Council's jobs will be to decide on PEPs. Some of the factors
I'll personally look at when making such decisions will be:

* Complexity vs. benefit. New type system features increase complexity,
  which means more concepts for users to learn; more work for type checker
  implementers; and more opportunity for divergences in behavior across type
  checkers. We should weigh whether the benefits of the PEP outweigh these
  costs.
* Composability. Does the new feature interact well with other parts of the
  type system? Does it prevent us from making any other changes that we may
  want to make in the future?
* Clarity. Does the PEP contain a clear, unambiguous specification?
* Usability. Some of the most impactful typing PEPs have been those that
  simplified the way users interact with the type system, such as PEPs
  [585](https://peps.python.org/pep-0585/) and [604](https://peps.python.org/pep-0604/),
  and hopefully soon [649](https://peps.python.org/pep-0649/). Does the
  new PEP make the type system more intuitive and easier to use?
