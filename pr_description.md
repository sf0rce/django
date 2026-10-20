⚡ milton: [Reduce allocations in WhereNode.clone]

**Repository:** `sf0rce/django`
**Base:** `main`

### 💡 What
Optimize `WhereNode.clone()` by replacing the iterative `for` loop and `.append()` calls with a single list comprehension.

### 🎯 Why
`WhereNode.clone()` is part of the extremely hot query cloning path triggered extensively during chained `.filter()` and `.exclude()` calls. The overhead of repeatedly looking up and calling `.append()` within a loop compounds across complex queries.

### 📊 Impact
- **Runtime:** `WhereNode.clone()` duration decreased by ~50% (0.33s -> 0.15s for 100K iterations). This contributes to an overall ~10% speedup in heavily filtered `QuerySet._clone()` operations (0.090s -> 0.082s for 10K iterations).
- **Allocations:** Reduced function call overhead and method lookups.

### 🔬 Measurement
```python
import timeit
# Using an in-memory SQLite setup and tests.queries.models.Tag
q = Tag.objects.filter(name='test', parent__name='test2').exclude(name='test3').order_by('name')
setup_code = "from __main__ import q\nw = q.query.where"
time_taken = timeit.timeit("w.clone()", setup=setup_code, number=100000)
print(f"Time taken: {time_taken} seconds")
```
- **Before:** 0.3309s
- **After:** 0.1574s

### ✅ Verification
- Confirmed PR target is `sf0rce/django:main`
- Passed `python tests/runtests.py queries model_fields expressions`
- Clean lint check (`flake8 django tests`)
