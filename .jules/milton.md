## 2026-10-29 - Avoid copy.deepcopy in Query.clone for select_related
**Learning:** `Query.select_related` stores either boolean flags (`True`/`False`) or nested dictionaries representing related field paths. `copy.deepcopy()` incurs huge reflection and memoization overhead during `Query.clone()`.
**Action:** Use `isinstance(self.select_related, dict)` and a fast recursive dictionary copying helper `_copy_select_related()` to clone `select_related` dictionaries without invoking `copy.deepcopy()`.
