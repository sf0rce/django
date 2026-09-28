# milton's Journal — Critical Learnings

## 2026-09-24 - Fast Recursive Copy for Query.select_related in Query.clone()
**Learning:** `copy.deepcopy()` incurs heavy Python frame overhead and internal memoization tracking when cloning `Query.select_related` field dictionaries. Furthermore, checking `if self.select_related is not False:` previously triggered `copy.deepcopy` even on boolean `True` values. Replacing `copy.deepcopy` with `isinstance(self.select_related, dict)` and a simple fast recursive dictionary copy function `_copy_select_related(d)` speeds up QuerySet cloning with `select_related` field dicts by ~37% (~1.30s down to ~0.81s for 100k clones) while avoiding unnecessary operations on boolean values.
**Action:** When cloning nested standard dict structures in internal Query/QuerySet state, avoid general-purpose `copy.deepcopy()` in favor of direct fast recursive dict copying functions.
