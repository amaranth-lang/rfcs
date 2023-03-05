- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [amaranth-lang/rfcs#0000](https://github.com/amaranth-lang/rfcs/pull/0000)
- Amaranth Issue: [amaranth-lang/amaranth#0000](https://github.com/amaranth-lang/amaranth/issues/0000)

# Minimal type annotation support

## Summary
[summary]: #summary

Add the minimal amount of functionality required for Amaranth developers to be able to use type hints in their code that are recognized by IDEs such as Visual Studio Code.

## Motivation
[motivation]: #motivation

[PEP 484] type hints have been steadily gaining popularity and there's been numerous requests to add some level of support for PEP 484 in Amaranth. There are some difficulties in adding these to Amaranth code, which is why Amaranth isn't currently typed; discussing this is out of scope for this RFC.

[RFC 1] adds functionality to Amaranth itself relying on the type hint syntax. The example given in the RFC is:

```python
from amaranth import *
from amaranth.lib import data


class IEEE754Single(data.Struct):
    mantissa: 23
    exponent: 8
    sign:     1
```

Unfortunately, `23` is not a valid Python type hint, nor is the more explicit form of `unsigned(23)`. While this definition will result in IDEs picking up the *existence* of the fields, they will not show any completion for `IEEE754Single().mantissa.<cursor>`.

The following definition, while somewhat uglier, does provide full IDE autocompletion functionality:

```python
class IEEE754Single(data.Struct):
    mantissa: Value[23]
    exponent: Value[8]
    sign:     Value[1]
```

[PEP 484]: https://peps.python.org/pep-0484/
[RFC 1]: 0001-aggregate-data-structures.md

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The `amaranth.hdl.Value` class becomes [subscriptable]; calling `Value[s]`, where `s` must be shape-castable, returns a [typing.GenericAlias] object with `s` as its sole argument. This object can be used as usual in type hints, and in addition it is accepted by `amaranth.lib.data.Struct` and `amaranth.lib.data.Union`.

[subscriptable]: https://docs.python.org/3/library/stdtypes.html#types-genericalias
[typing.GenericAlias]: https://docs.python.org/3/library/types.html#types.GenericAlias

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

A [PEP 560] `__class_getitem__(cls, shape)` method is added on `amaranth.hdl.Value`. It checks that `shape` is shape-castable, and (since it is inherited as normal) that `cls in (amaranth.hdl.Value, amaranth.hdl.Const, amaranth.hdl.Signal)` (to prevent confusing invocations like `Operator[1]`, which are meaningless). There is no semantic distinction at runtime between `Value[1]`, `Signal[1]`, or `Const[1]` (as Python type hints are generally not interpreted at runtime), but it may be convenient to the developer.

[PEP 560]: https://peps.python.org/pep-0560/

The behavior of this method differs depending on the Python version.
* On Python 3.7-3.8 (where `types.GenericAlias` is not available, and `list[int]` is an error), it returns `shape` (if it returned `cls`, the result would not be usable with `data.Struct`).
* On Python 3.9+ (where `types.GenericAlias` is available), it returns `types.GenericAlias(cls, (shape,))`.

`amaranth.lib.data.Struct` and `amaranth.lib.data.Union` are updated to recognize `types.GenericAlias(cls, ...)` where `cls is Value` as a supported annotation, and use the sole argument as the actual field shape. Annotations with any other `cls` (e.g. `Signal[1]`) are not supported since `Struct` places no restrictions on the underlying value and these would be unsound.

## Drawbacks
[drawbacks]: #drawbacks

As described this does not work with mypy, even if `Value` is made a `Generic[T]`:

```python
from amaranth import *
from amaranth.lib import data


class S(data.Struct):
    f: Value[1] # error: Invalid type: try using Literal[1] instead?  [valid-type]
```

If we ship this, and developers want mypy support, we'll eventually have to change the way we use type hints for mypy support.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

Rationale:

* This is a conservative design with no impact on the core Amaranth language.
* There is a significant unmet need for type hint support for IDEs.
  * Adding `Struct` and `Union` in a way that is strictly incompatible with any usage of type hints (as it currently is), whether IDEs or type checkers, will deepen that need.

Alternatives:

* (Proposed by [@emilazy]) Since mypy expects a type as a generic type parameter, we could define a set of types like `amaranth.typing.U0` to `amaranth.typing.U128` so that something like this could work:

  ```python
  from amaranth import *
  from amaranth.typing import *
  from amaranth.lib import data


  class S(data.Struct):
      f: Value[U1]
  ```

## Prior art
[prior-art]: #prior-art

None the author is aware of.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

* Is the plan for Python 3.7-3.8 acceptable?
  * Returning a non-type from `Value[...]` is a misuse of the API.
  * Is there anyone who cares about Python 3.7-3.8 **and** that?
    * Python 3.7 is EOL in a few months.
* Should we allow `Const[s]` at all?
  * The utility seems limited.
* Should we *require* `Value[s]` syntax in `data.Struct` and `data.Union`?
  * Having two equivalent syntaxes accepted, with the only difference being that one of them works with IDEs and the other doesn't, seems like a bad idea.
  * But `Value[s]` is really poorly supported on Python 3.7-3.8.
    * Can we backport it somehow?
  * We can always deprecate non-`Value[]` syntax after Python 3.8 support is dropped in a year.

## Future possibilities
[future-possibilities]: #future-possibilities

* Support for type hints in downstream code can be extended.
* Compatibility with mypy/pytype could be added.
* Amaranth itself could be annotated with type hints.

## Acknowledgements
[acknowledgements]: #acknowledgements

[@VioletEternity] tested VSCode support for annotations.

[@emilazy] provided valuable feedback on integration with typecheckers.

[@VioletEternity]: https://github.com/VioletEternity
[@emilazy]: https://github.com/emilazy
