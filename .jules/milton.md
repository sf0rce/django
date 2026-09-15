## 2026-09-08 - Optimize Query.clone dict lookups and pop overhead

**Learning:** `Query.clone()` is executed on almost every `QuerySet` transformation (`.filter()`, `.exclude()`, `.chain()`). Calling `obj.__dict__.pop("base_table", None)` incurs dictionary method lookup and hash table mutation overhead on uncloned/clean queries. Also, repeated accesses to `self.__dict__` during attribute cloning can be optimized by binding `d = self.__dict__`.
**Action:** Use conditional key presence checks (`if "base_table" in d: del obj.__dict__["base_table"]`) and local `d` binding in `Query.clone()` to shave allocations and dict lookup overhead.
