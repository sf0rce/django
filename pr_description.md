⚡ milton: [Optimization: Custom recursive clone for `select_related` avoiding deepcopy overhead]

**Repository:** `sf0rce/django`  •  **Base:** `main`

### 💡 What
Replaced `copy.deepcopy()` with a custom `_clone_dict()` function for copying the `select_related` dictionary in `Query.clone()`.

### 🎯 Why
The bottleneck was allocation-based and structural. `copy.deepcopy()` has a significant overhead due to its memoization and dispatch mechanism, especially on nested dictionaries commonly used for `select_related`. By implementing a tailored recursive `_clone_dict` function, we avoid this generic deepcopy overhead in hot paths during query construction and cloning. We intentionally kept it scoped strictly to `select_related` in `Query.clone()` after PR review noted that altering `QuerySet.__deepcopy__` changes its expected semantics and is not worth the slight performance increase in a less critical hot path.

### 📊 Impact
- **Runtime:** Over 2x performance improvement in cloning queries with complex `select_related` dicts. Benchmarked at ~0.74s for 100k iterations compared to the baseline ~1.54s for `copy.deepcopy()` on the same structure.
- **Allocations:** Reduced allocation and instruction count by avoiding the generic, heavy dispatch code of `copy.deepcopy`.
- **SQL count:** No change.

### 🔬 Measurement
```python
import os
import sys
import timeit
import copy

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'tests.test_sqlite')
import django
from django.conf import settings
settings.INSTALLED_APPS = ['tests.queries']
django.setup()

from django.db import models
from django.db.models.sql.query import Query
from tests.queries.models import Author, Item, ObjectC

def benchmark_clone():
    q = Query(Author)
    q.add_select_related(['item__creator', 'objectc__objectb__objecta'])

    # Run cloning multiple times
    for _ in range(1000):
        q.clone()

print("clone select_related:", timeit.timeit(benchmark_clone, number=100))
```
- Baseline: `1.538s`
- Optimized: `0.742s`

### ⚠️ Review notes
No public API changes. The clone semantics for `select_related` nested dictionaries remain functionally identical. Addressed review feedback by restricting changes to `Query.clone()` rather than globally altering `__deepcopy__`.

### ✅ Verification
- PR target confirmed: `sf0rce/django:main`
- `python tests/runtests.py model_fields queries expressions` passed
- `flake8 django tests` clean
