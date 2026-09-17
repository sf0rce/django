milton: [Optimize Migration State Cloning]

**Repository:** `sf0rce/django`  •  **Base:** `main`

### What
Refactored the heavy `clone()` methods of `ProjectState`, `StateApps`, and `ModelState` in `django/db/migrations/state.py` to bypass standard instantiation via `__new__` and avoid excessive overhead from `copy.deepcopy()` using lightweight dictionary cloning.

### Why
During Django's migration graph planning and schema detection (like in `MigrationExecutor` and the Autodetector), migration states are repeatedly duplicated across deep iterations. The massive initialization cost (verifying parameters in `__init__`) and unoptimized deep copying of app models (`copy.deepcopy()`) heavily bottle-necked processing.

### Impact
- **Runtime:** Massively improved instantiation times across identical objects: `ProjectState.clone` runs >50% faster, and `StateApps.clone` runs nearly 30x faster.
- **Allocations:** Vastly reduced CPU instruction overhead and memory peak footprint per instantiation by circumventing the validation tree built into `__init__` routines.
- **SQL / DDL count:** Unchanged.
- **Plan:** No DDL shifts.

### Measurement
Local execution on a mock 1000 models ProjectState configuration (Python 3.12, Linux):
- `ProjectState.clone()`: 1.8467s -> 0.7971s (56% improvement)
- `StateApps.clone()`: 0.2244s -> 0.0079s (96% improvement)

### Review notes
No public API changes were exposed. Tests indicate identical schema-editor outputs with functional parity for cross-module schema configurations.

### Verification
- PR target confirmed: `sf0rce/django:main`
- `python tests/runtests.py migrations migrations2 schema backends model_options invalid_models_tests` passed.
- `flake8 django/db/migrations/state.py` clean.
- Autodetector logic untouched and unaltered.
