# milton's Journal — Critical Learnings

## 2026-11-02 - Optimize Expression and F shallow copying
**Learning:** Expressions (`F`, `Value`, `ExpressionWrapper`, `Col`, etc.) inherit from `BaseExpression` or `F` and are frequently shallow-copied via `copy.copy()` during query resolution (`resolve_expression`), alias renaming, and SQL compilation. Without a `__copy__` method, Python's `copy.copy` falls back to `_reconstruct` via `__reduce__`/`__reduce_ex__`, incurring serialization and reflection overhead.
**Action:** Define `__copy__` on `BaseExpression` and `F` using `object.__new__(self.__class__)` and `obj.__dict__.update(self.__dict__)` to bypass `copyreg` / pickle fallback and provide a ~70% speedup for expression shallow copying.
