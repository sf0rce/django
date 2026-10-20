# milton's Journal — Critical Learnings

## 2026-09-16 - Reduce allocations and latency in Query.clone for select_related
**Learning:** `copy.deepcopy(self.select_related)` introduces high CPU overhead in `Query.clone()` due to `copy.py`'s object tracking and memo dict management. Since `select_related` in Django queries is exclusively a nested dictionary of field names or a boolean, a dedicated recursive dict copy function avoids `copy.deepcopy` entirely and yields a ~38% speedup on `Query.clone()` for queries with `select_related`.
**Action:** Prefer direct recursive/specialized dict copies over generic `copy.deepcopy` in ORM cloning hot paths.
