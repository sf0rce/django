## 2024-05-18 - Optimize WhereNode cloning
**Learning:** `WhereNode.clone()` uses `self.create()` which calls the base `Node.__init__` and iterates over children appending them one by one. This is unnecessarily slow and adds measurable overhead to QuerySet cloning.
**Action:** Replaced `self.create()` with `self.__class__.__new__(self.__class__)` and used list comprehensions in `WhereNode.clone()` to avoid `__init__` overhead and `.append()` loops. This applies as a general rule: when cloning tree nodes, prefer `__new__` and comprehensions over constructor patterns.
