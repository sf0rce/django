**Repository:** `sf0rce/django`  •  **Base:** `main`

### What
Replaced `copy.deepcopy(self.all_models)` in `StateApps.clone` with a shallow `dict.copy()` loop.

### Why
`StateApps.all_models` is a `defaultdict(dict)` mapping app labels to dictionaries mapping model names to model classes. When creating a migration state clone, Django used `copy.deepcopy`. Since the inner dicts map to actual model classes, `deepcopy` spends a massive amount of CPU attempting to introspect and "deep copy" Python classes, which just resolves to returning the same reference anyway. Replacing it with a shallow copy of the inner dictionaries maintains exact isolation but avoids the immense deepcopy traversal overhead.

### Impact
- **Runtime:** ~15-18% improvement on project state cloning for large graphs (from 11.94s down to 10.11s for 500 clones of a 1000-model state).
- **Allocations:** Negligible difference; primarily saves CPU overhead from `copy` module traversal.

### Measurement
```python
import os
import django
os.environ.setdefault("DJANGO_SETTINGS_MODULE", "tests.test_sqlite")
django.setup()

import time
import tracemalloc
from django.db.migrations.state import ProjectState, ModelState

state = ProjectState()
# 1000 models
for i in range(100):
    for j in range(10):
        state.add_model(ModelState(f"app_{i}", f"Model_{j}", []))

# Warm up
state.apps

tracemalloc.start()
start = time.perf_counter()

for _ in range(500):
    state.clone()

end = time.perf_counter()
current, peak = tracemalloc.get_traced_memory()
tracemalloc.stop()

print("RESULTS")
print(f"Time for 500 clones: {end - start:.4f}s")
print(f"Peak memory: {peak / 1024 / 1024:.2f} MB")
```

### Review notes
No public API changes. `ProjectState` behavior remains exactly identical.

### Verification
- PR target confirmed: `sf0rce/django:main`
- `python tests/runtests.py migrations migrations2 schema backends model_options invalid_models_tests` passed cleanly
- `flake8 django tests` clean
