⚡ milton: Reduce allocations in QuerySet cloning

**Repository:** `sf0rce/django`
**Base:** `main`

### 💡 What
Optimized the cloning hot path in the ORM. Updates `Query.clone`, `WhereNode.clone`, and `Node.create` to avoid unnecessary work:
- Initializing empty collections with literals instead of calling `.copy()` on empty structures in `Query.clone`
- Replacing `Node.create` instantiation `__init__` calls with `__new__` and directly mutating state
- Streamlining `WhereNode.clone` to instantiate by `__new__` and skipping the generic create wrapping

### 🎯 Why
During `QuerySet` manipulation (e.g. chaining filters), `Query.clone` and `WhereNode.clone` are executed frequently. Calling `.copy()` on empty dictionaries or sets generates unnecessary object allocations and incurs overhead. Creating objects using `__new__` instead of `__init__` saves redundant initializations in the standard library.

### 📊 Impact
- **Runtime:** `Query.clone` improved from ~0.43s to ~0.38s across 100,000 iterations (approx. 11% faster clone).
- **Allocations:** Reduced dict copies per clone.

### 🔬 Measurement
```python
import os
import django
from django.conf import settings
import timeit

settings.configure(
    INSTALLED_APPS=[
        'django.contrib.auth',
        'django.contrib.contenttypes',
    ],
    DATABASES={'default': {'ENGINE': 'django.db.backends.sqlite3', 'NAME': ':memory:'}}
)
django.setup()

from django.contrib.auth.models import User
q = User.objects.all().query

print("clone time:", timeit.timeit("q.clone()", setup="from __main__ import q", number=100000))
```

### ✅ Verification
- Confirmed PR target is `sf0rce/django:main`
- Passed `python tests/runtests.py queries expressions model_fields`
- Clean lint check (`flake8 django tests`)
