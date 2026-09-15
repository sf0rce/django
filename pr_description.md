milton: [Optimization: LocMemCache batch operations]

**Repository:** `sf0rce/django`  •  **Base:** `main`

### What
Overrides `get_many`, `set_many`, and `delete_many` on `django.core.cache.backends.locmem.LocMemCache` to properly handle these commands in batch rather than sequentially inside single loops. Pickling serialization, unpickling serialization, and key validation steps have been extracted outside of the cache locking block to reduce locking time contention.

### Why
By default, the `LocMemCache` uses the fallback iterations in `BaseCache` to perform `get_many`, `set_many` and `delete_many`. This naive implementation iteratively acquires and releases the cache lock `self._lock` per loop element, which stalls operations and causes thread blockings for larger workloads. Also, executing operations like `pickle.loads` and `pickle.dumps` within the thread lock causes extra lock stall times. Batching the operations limits the thread lock acquire/release step down to just one call per bulk action and decreases overall iteration lock time.

### Impact
- **Runtime:** ~15-20% time improvement for larger batches for `LocMemCache.set_many` / `get_many` / `delete_many`. Reduced locking contention for multithreaded workflows.
- **Allocations:** Negligible change. Slightly higher footprint on `set_many` due to up-front memory pickling object allocation, offset by less time spent in a lock block.

### Measurement
Script:
```python
import time
from django.conf import settings
import django
from django.core.cache import cache

settings.configure(
    CACHES={
        'default': {
            'BACKEND': 'django.core.cache.backends.locmem.LocMemCache',
            'LOCATION': 'unique-snowflake',
        }
    }
)
django.setup()

def bench(scale, rounds):
    keys = [f"key_{i}" for i in range(scale)]
    values = {k: v for k, v in zip(keys, range(scale))}

    # Warm up
    cache.set_many(values)
    cache.get_many(keys)
    cache.delete_many(keys)

    set_times = []
    get_times = []
    del_times = []

    for _ in range(rounds):
        t0 = time.time()
        cache.set_many(values)
        set_times.append(time.time() - t0)

        t0 = time.time()
        cache.get_many(keys)
        get_times.append(time.time() - t0)

        t0 = time.time()
        cache.delete_many(keys)
        del_times.append(time.time() - t0)

    print(f"Scale: {scale}")
    print(f"set_many: {min(set_times):.4f}s")
    print(f"get_many: {min(get_times):.4f}s")
    print(f"delete_many: {min(del_times):.4f}s")

bench(100, 100)
bench(10000, 100)
```

Before:
```
Scale: 100
set_many: 0.0007s
get_many: 0.0006s
delete_many: 0.0005s
Scale: 10000
set_many: 0.0752s
get_many: 0.0636s
delete_many: 0.0567s
```

After:
```
Scale: 100
set_many: 0.0006s
get_many: 0.0005s
delete_many: 0.0004s
Scale: 10000
set_many: 0.0609s
get_many: 0.0494s
delete_many: 0.0452s
```

### Review notes
No backend public APIs were broken. Threading lock logic guarantees single blocks are operated on simultaneously, preventing stale values and dead locks.

### Verification
- PR target confirmed: `sf0rce/django:main`
- `python tests/runtests.py cache --parallel=1` passed
- Backend coverage: LocMemCache backend covered.
