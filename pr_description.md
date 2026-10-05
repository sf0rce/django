⚡ milton: Optimize Node.create to avoid object class reassignment

**Repository:** `sf0rce/django`
**Base:** `main`

### 💡 What
Updated `Node.create()` in `django/utils/tree.py` to instantiate `cls` directly using `object.__new__(cls)` instead of instantiating `Node(...)` and subsequently mutating `obj.__class__ = cls`.

### 🎯 Why
`Node.create()` is called heavily during `Q` object operations (combinations, filter tree generation, node copying) and SQL clause resolution. The previous implementation instantiated a base `Node` and mutated its type via `obj.__class__ = cls`, which incurs unnecessary object creation and CPython type reassignment runtime overhead.

### 📊 Impact
- **Node.create speedup:** ~22% runtime improvement across 1M calls.
- **Q object operations speedup:** ~20% faster `Q & Q` and `Q.create()` operations.

### 🔬 Measurement
```python
import timeit
from django.db.models import Q
from django.utils.tree import Node

# Before:
# Node.create: 0.978s (1M iterations)
# Q & Q: 0.273s (100k iterations)

# After:
# Node.create: 0.751s (1M iterations)
# Q & Q: 0.219s (100k iterations)
```

### ✅ Verification
- Confirmed PR target is `sf0rce/django:main`
- Passed `python tests/runtests.py utils_tests queries expressions model_fields`
- Clean lint check (`flake8 django tests`)
