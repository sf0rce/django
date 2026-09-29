# Milton - Migration Optimization Run

### Successful Optimization: `StateApps.clone`
- **What**: Replaced `copy.deepcopy(self.all_models)` in `StateApps.clone` with a shallow `dict.copy()` loop.
- **Why**: `self.all_models` is a `defaultdict(dict)` where the inner values are generated Python model classes. `copy.deepcopy` traverses class attributes which is incredibly slow and unnecessary for isolation of dictionary keys/values.
- **Impact**: Reduced clone time by ~15-18% in a 1000-model test over 500 clones (from 11.9s to 10.1s).
