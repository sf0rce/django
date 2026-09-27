# milton ⚡ Journal

## 2026-06-19 - Fast Recursive Copy for Query.select_related in Query.clone()

**Learning:** `Query.clone()` previously used `copy.deepcopy(obj.select_related)` when `select_related` was set. In Django, `select_related` is either a boolean or a nested dictionary of relation names (e.g. `{'author': {'profile': {}}}`). Generic `copy.deepcopy` introduces massive Python runtime overhead due to memoization and reflection. Replacing it with a specialized recursive dict copy `_copy_select_related` yields a ~4x speedup on `select_related` dict copying and ~28% speedup on overall `QuerySet._clone()` calls when `select_related` is enabled.
**Action:** Avoid `copy.deepcopy` in hot paths like query cloning when the data structure types (dicts/lists/primitives) are well-defined.
