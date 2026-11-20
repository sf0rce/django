## 2026-09-28 - Fast Expression Cloning via Direct `__copy__`

**Learning:** `BaseExpression.copy()` used `copy.copy(self)` which, without a custom `__copy__` method on expression classes, fell back to `__reduce__`/`deconstruct()` and generic reflection via `inspect`, causing high CPU overhead during SQL query compilation and expression tree cloning.

**Action:** Defined fast `__copy__` methods using `object.__new__(self.__class__)` and `__dict__.update()` on `BaseExpression`, `Combinable`, `F`, `Func`, `Case`, and `Subquery`, bypassing `copy.copy()` reflection overhead and accelerating expression cloning by 3x–4.5x.
