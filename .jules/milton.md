## 2024-05-15 - [Reduce dictionary allocations in QuerySet clone]
**Learning:** `Query.clone` and `WhereNode.clone` allocate dictionaries/sets and make expensive `__init__` calls even when dealing with defaults/empty structures.
**Action:** When working on Query cloning performance, prefer checking emptiness before calling `.copy()` and prefer instantiating classes directly using `__new__` to bypass initialization overhead when the state logic is straightforward.
