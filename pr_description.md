milton: Optimize LocMemCache get_many, set_many, delete_many

**Repository:** `sf0rce/django`  •  **Base:** `main`

### What
Implemented native `get_many`, `set_many`, and `delete_many` methods on `django.core.cache.backends.locmem.LocMemCache`.

### Why
The bottleneck: `LocMemCache` historically inherited the batch methods from `BaseCache`. `BaseCache` implements batch ops by looping over the keys and calling the single-item methods (`get`, `set`, `delete`). This means for a batch of 10,000 items, `LocMemCache` would acquire and release its internal threading lock 10,000 times, and perform serialization (`pickle.dumps`/`pickle.loads`) while holding the lock. This structural bottleneck significantly impaired concurrent performance and increased single-threaded runtime overhead.

The optimization refactors these methods to:
1. Validate keys and perform serialization outside of the lock.
2. Acquire the threading lock exactly *once* per batch operation.
3. Apply all inner storage changes, and then release the lock.

### Impact
- **Runtime:** ~15% improvement on batch `get_many` / `set_many` for large sets. (Single-threaded)
- **Lock Contention:** Drastic reduction in lock contention across multi-threaded applications. The lock is now only held for the fast internal dict updates rather than slow key validation and pickle ops.
- **Allocations:** Negligible allocation delta, memory usage remains strictly bounded.

### Measurement
Raw benchmark numbers using a loop of 10000 key inserts over 10 iterations:
```
Base set_many: 0.0798s
Base get_many: 0.0730s
Base delete_many: 0.0660s

Optimized set_many: 0.0675s
Optimized get_many: 0.0592s
Optimized delete_many: 0.0518s
```
*Environment: Python 3.12, LocMemCache, 10,000 keys per operation*

### Review notes
This alters the public API behavior of `LocMemCache` batch methods natively. The original `BaseCache` behavior is completely preserved in effect (keys still expire correctly, caches are still culled according to `_cull_frequency`, etc.), but lock acquisitions are drastically optimized.

### Verification
- PR target confirmed: `sf0rce/django:main`
- `python tests/runtests.py cache` passed (0 failures out of 696 tests)
- `flake8 django tests` clean
- Backend coverage: LocMemCache batch operations fully exercised.
