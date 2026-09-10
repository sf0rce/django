## 2026-09-10 - [WhereNode clone optimization]
**Learning:** Bypassing `__init__` or `create()` using `__class__.__new__` during Node cloning is a dangerous anti-pattern that can break subclass initialization and is unsafe. However, replacing `for` loop `.append()` calls with list comprehensions offers a safe and measurable speedup (~10% for `QuerySet._clone()` on heavily filtered queries).
**Action:** Use list comprehensions for collection building in hot paths, but never bypass standard instance creation mechanisms (`.create()`) to ensure state consistency across subclasses.
