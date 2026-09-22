# milton's Journal — Critical Learnings

## 2026-09-22 - Avoid copy.deepcopy on boolean select_related in Query.clone
**Learning:** `Query.select_related` can be `False`, `True`, or a nested `dict`. Calling `copy.deepcopy` when `select_related` is `True` incurs unnecessary function call and lookup overhead for an immutable boolean.
**Action:** In `Query.clone()`, check `if self.select_related is not False and isinstance(self.select_related, dict):` before deepcopying to eliminate deepcopy overhead on boolean `select_related` while keeping `is not False` short-circuiting fast.
