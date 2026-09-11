## 2026-09-08 - Fast Q instantiation and Node.create

**Learning:** `Q` object creation and combining (`&`, `|`) are hot paths during query filtering. In `Node.create()`, calling `Node(...)` followed by `obj.__class__ = cls` causes Python dynamic class reassignment overhead. In `Q.__init__`, calling `sorted(kwargs.items())` when `len(kwargs) <= 1` is redundant, and calling `super().__init__` duplicates list allocations for `children`.
**Action:** Use `cls.__new__(cls)` inside `Node.create()` for direct type instantiation. In `Q.__init__`, bypass `sorted()` for single/empty `kwargs` and assign `self.children` directly.
