
## State Cloning Optimizations
When bypassing `__init__` with `__new__` in Django's migration state objects (`ModelState.clone`, `StateApps.clone`), you must explicitly set all dynamically initialized attributes to avoid downstream test failures. For example:
- `ModelState`: explicitly initialize `options.setdefault("indexes", [])` and `options.setdefault("constraints", [])` because these are added in `__init__` and downstream code relies on them.
- `StateApps`: explicitly set `apps_ready`, `models_ready`, and `_pending_operations` (which must be a `defaultdict(list)`) because they are used by the underlying `Apps` registry logic when resolving lazy operations.
Replacing `copy.deepcopy` with dictionary comprehension (e.g. `clone.all_models[app_label] = dict(app_models)`) yields a 10x-15x performance increase without altering semantics.
