## 2026-09-28 - Optimize `BaseExpression.__copy__` and `F.__copy__`

**Learning:** `BaseExpression` and `F` (and their subclasses like `Value`, `Func`, `Case`, `When`) are decorated with `@deconstructible`. Standard `copy.copy()` on a `@deconstructible` object falls back to Python's `__reduce__` mechanism and calls `.deconstruct()`, creating new tuples/dicts and instantiating classes through `__new__` on every `copy.copy()` call.
**Action:** Define explicit `__copy__(self)` methods on `BaseExpression` and `F` using `object.__new__(self.__class__)` and `obj.__dict__.update(self.__dict__)` to perform lightweight fast shallow copies without triggering `__reduce__` or `.deconstruct()`.
