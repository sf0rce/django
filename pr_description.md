milton: Optimize LocMemCache batch operations

**Repository:** `sf0rce/django`  •  **Base:** `main`

### What
Restructured `LocMemCache.get_many`, `set_many`, and `delete_many` to acquire the internal cache lock only once per operation instead of looping and making individual backend method calls. It also moves time-intensive operations like dictionary comprehensions, key validations, and the `pickle` and `unpickle` routines entirely outside of the lock context.

### Why
By default, these batch methods inherit the fallback implementations from `BaseCache`, which loop over the given keys and call `get()`, `set()`, and `delete()` individually. This pattern means the global internal thread lock (`self._lock`) is acquired and released `N` times for a batch of `N` keys. Moving these locks out of the `for` loops mitigates extreme lock contention in multithreaded workflows. Furthermore, by pulling `pickle.dumps` out of the loop and out of the lock context, serialization won't stall concurrent reads from parallel requests.

### Impact
- **Runtime:** Tested with an in-memory python benchmarking script (without external dependencies) with `N=100,000` keys. `get_many` time dropped from ~0.64s to ~0.54s (15.6% improvement). `set_many` dropped from ~0.82s to ~0.70s (14.6% improvement). `delete_many` dropped from ~0.59s to ~0.47s (20% improvement).
- **Allocations:** Modestly reduced object allocation overhead related to repeatedly instantiating lock context managers and redundant lookups on `self.make_and_validate_key`.

### Measurement
Benchmark numbers generated locally.
```
--- Scale: 100 ---
old set_many    0.00066s
new set_many    0.00056s
old get_many    0.00054s
new get_many    0.00050s
old delete_many 0.00048s
new delete_many 0.00039s

--- Scale: 1000 ---
old set_many    0.00646s
new set_many    0.00527s
old get_many    0.00570s
new get_many    0.00463s
old delete_many 0.00512s
new delete_many 0.00401s

--- Scale: 10000 ---
old set_many    0.06745s
new set_many    0.05734s
old get_many    0.06453s
new get_many    0.05987s
old delete_many 0.06617s
new delete_many 0.05343s

--- Scale: 100000 ---
old set_many    0.82165s
new set_many    0.70538s
old get_many    0.64315s
new get_many    0.53989s
old delete_many 0.58944s
new delete_many 0.46844s
```
*Environment details:* CPython 3.12, LocMemCache default options.

### Review notes
No behavioral changes to serialization, validation, or semantic meaning were introduced. Culling and timeouts remain respected via `self._set` calls inside the new loops.

### Verification
- PR target confirmed: `sf0rce/django:main`
- `python tests/runtests.py cache files mail serializers` passed perfectly.
- `flake8 django tests` passed without complaints.
- Backend coverage: purely `LocMemCache`.
