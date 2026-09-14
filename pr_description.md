**Repository:** `sf0rce/django`  •  **Base:** `main`

### What
Optimized the cloning mechanisms in the migrations framework's state classes (`ProjectState.clone()`, `StateApps.clone()`, and `ModelState.clone()`). I replaced `copy.deepcopy()` with shallow dictionary comprehensions and bypassed `__init__` overhead by instantiating new objects via `__new__` and explicitly mapping required attributes.

### Why
The bottleneck was structural and allocation-based. The autodetector and migration executor rely heavily on deeply copying `ProjectState` and its underlying objects (`StateApps`, `ModelState`) multiple times. Python's `copy.deepcopy()` adds tremendous overhead when run repeatedly in loops, especially on complex class structures with large model dictionaries. Bypassing `__init__` removes redundant logic executions and initialization of caches.

### Impact
- **Runtime:**
  - `ProjectState.clone()`: ~4x improvement (1.01s → 0.24s for 200 iterations).
  - `StateApps.clone()`: ~15x improvement (0.20s → 0.01s for 200 iterations).
  - `ModelState.clone()`: ~3x improvement (0.61s → 0.19s for 200 iterations).
- **Allocations:** Vastly reduced due to skipping `copy.deepcopy()` on internal state dictionaries in favor of deterministic mappings.
- **SQL / DDL count:** Unchanged.
- **Plan:** No DDL changes.

### Measurement
Benchmarked locally using `timeit` and `pytest-benchmark` on synthetic models simulating 1000 models (`app_0`..`app_49` with 20 models each) across 200 iterations for state objects in `django/db/migrations/state.py`.
- **Environment:** Django, SQLite in-memory backend, Python 3.12.

### Review notes
No public API changes. Internal operations remain functionally equivalent; care was taken to explicitly rehydrate the necessary attributes inside `clone()` that were typically created inside `__init__` (e.g. `apps_ready`, `_pending_operations = defaultdict(list)`, default `indexes` options).

### Verification
- PR target confirmed: `sf0rce/django:main`
- `python tests/runtests.py migrations migrations2 schema backends model_options invalid_models_tests` passed.
- `flake8 django tests` clean.
- Autodetector output unchanged.
