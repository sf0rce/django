⚡ milton: Optimize Query/WhereNode cloning via __new__ instantiation

**Repository:** `sf0rce/django`  •  **Base:** `main`

### 💡 What
Optimize `Query.clone()`, `WhereNode.clone()`, and `Node.__copy__` / `Node.__deepcopy__` to use `__new__` to bypass initialization overhead instead of calling constructors.

### 🎯 Why
Query cloning is heavily utilized when using chained queryset filtering or building queries.
`__init__` calls for `django.utils.tree.Node` and `Empty` logic for `Query.clone` were taking unnecessary overhead and doing extra assignments, which could be simply handled via shallow copies of children lists combined with `__new__` for allocation.
This completely removes `Empty` from `Query.clone()` as well.

### 📊 Impact
- **Runtime:** ~7-8% improvement on repeated QuerySet cloning. Benchmark went from 6.68s down to 6.17s.
- **Allocations:** Reduced `__init__` frames and list creations per `clone()` invocation.
- **SQL count:** Unchanged.

### 🔬 Measurement
```python
import os
import django
from django.conf import settings
import timeit
import cProfile
import pstats
from django.db import models, connection
from django.db.models import Q

settings.configure(
    DATABASES={'default': {'ENGINE': 'django.db.backends.sqlite3', 'NAME': ':memory:'}},
    INSTALLED_APPS=[],
)
django.setup()

class Author(models.Model):
    name = models.CharField(max_length=100)
    class Meta:
        app_label = 'benchmark'

class Book(models.Model):
    title = models.CharField(max_length=100)
    author = models.ForeignKey(Author, on_delete=models.CASCADE)
    class Meta:
        app_label = 'benchmark'

with connection.schema_editor() as schema_editor:
    schema_editor.create_model(Author)
    schema_editor.create_model(Book)

def clone_workload():
    q = Book.objects.all()
    for _ in range(100):
        q = q.filter(title__icontains="1")

print("Clone workload timing:")
print(timeit.timeit(clone_workload, number=1000))
```
Results:
Before: `6.6856793080000045`
After: `6.172713154000007`

### ⚠️ Review notes
N/A. Internal API changes, but query structure correctly copies state.

### ✅ Verification
- PR target confirmed: `sf0rce/django:main`
- `python tests/runtests.py queries expressions model_fields db_functions lookup queryset_pickle` passed
- `flake8 django tests` clean for changed files
