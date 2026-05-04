# bower-resolution-conflict-probe

This probe exercises three conflict-related Bower patterns in a single manifest.
All three patterns share the same Mend code path: resolution of the `resolutions`
field in `bower.json`.

## Patterns covered

### 1. `resolution-conflict`

The `resolutions` field pins `jquery` to `3.7.1`. This overrides the transitive
constraints declared by `bootstrap` and `jquery-ui`, which disagree on the
acceptable range. Without `resolutions`, Bower picks 3.7.1 anyway (bootstrap's
tighter constraint wins), but the `resolutions` entry forces the outcome
explicitly. Mend must read the `resolutions` field and report `jquery` at `3.7.1`
regardless of what the transitive dependents declare.

### 2. `scoped-conflict`

Three direct dependencies each pull `jquery` as a transitive at different version
ranges:

- `bootstrap` 3.4.1 declares `"jquery": "1.9.1 - 3"` (semver range, upper bound
  is the major version 3, i.e. `<=3.x`).
- `jquery-ui` 1.12.1 declares `"jquery": ">=1.6"` (no upper bound; resolves to
  4.0.0 when unconstrained).
- `backbone` 1.6.1 declares no jquery dependency (only `underscore`).

Without `resolutions`, these two ranges conflict: `jquery-ui` would resolve to
4.0.0 but `bootstrap` caps at `<=3.x`. Bower resolves the conflict by using the
most restrictive satisfying version (3.7.1). The `resolutions` field makes this
explicit. Mend's scoped-conflict handling is exercised because it must reconcile
two different constraints for the same transitive package across multiple parents.

### 3. `stale-resolution-entry`

The `resolutions` field includes `"removed-pkg": "1.0.0"`. This package is not
declared anywhere in `dependencies`, `devDependencies`, or as a transitive. It is
a deliberate stale reference — a package that was once a dependency and had its
resolution pinned, but was later removed from the dependency graph without
cleaning up the `resolutions` block.

Bower itself logs this as `"extra-resolution Unnecessary resolution: removed-pkg#1.0.0"`
but completes the install without error. Mend's UA fallback must likewise ignore
this entry cleanly: `removed-pkg` must not appear anywhere in the resolved
dependency tree. No error should be raised.

## Conflict analysis (verified with `bower install` on 2026-05-04)

Without `resolutions` in `bower.json`, Bower 1.8.14 resolves as follows:

| Package | Declared by | Constraint | Bower's candidate |
|---|---|---|---|
| jquery | bootstrap 3.4.1 | `1.9.1 - 3` | 3.7.1 (highest in range) |
| jquery | jquery-ui 1.12.1 | `>=1.6` | 4.0.0 (highest available) |
| jquery | (conflict resolution) | intersection | 3.7.1 (bootstrap wins) |

Conclusion: the conflict IS real. `jquery-ui`'s unbounded `>=1.6` would select
4.0.0, but `bootstrap`'s `1.9.1 - 3` cap prevents it. The `resolutions` entry
pins `3.7.1` explicitly, matching what Bower would pick via conflict resolution
but making the intent unambiguous.

Bower also logged:
```
bower jquery  extra-resolution  Unnecessary resolution: jquery#3.7.1
bower removed-pkg  extra-resolution  Unnecessary resolution: removed-pkg#1.0.0
```
The first warning is because Bower's natural resolution already settles on 3.7.1;
the `resolutions` entry is semantically redundant but structurally exercises the
code path. The second warning confirms the stale-resolution-entry pattern.

## Expected dependency tree

After `bower install`, 5 packages appear at the top level of `bower_components/`:

| Package | Version | Source | Group | Direct? | Parent |
|---|---|---|---|---|---|
| bootstrap | 3.4.1 | registry | main | yes | (root) |
| jquery-ui | 1.12.1 | registry | main | yes | (root) |
| backbone | 1.6.1 | registry | main | yes | (root) |
| jquery | 3.7.1 (forced by resolutions) | registry | main | no | bootstrap, jquery-ui |
| underscore | 1.13.8 | registry | main | no | backbone |

- Total packages: 5
- Direct: 3 (bootstrap, jquery-ui, backbone)
- Transitive: 2 (jquery shared by bootstrap and jquery-ui, underscore under backbone)
- `removed-pkg` does NOT appear in the tree (stale-resolution-entry assertion)
- All sources: Bower registry (registry.bower.io)

### Mend failure modes exercised

- Mend ignores `resolutions` field: jquery reported at wrong version or with
  conflict error.
- Mend adds `removed-pkg` to tree as a false dependency: stale resolution entry
  incorrectly materialized.
- Mend reports all 5 packages as direct (flat layout misread; parent-chain not
  inferred from each package's own `dependencies` field).
- jquery attributed only to bootstrap (jquery-ui's parent chain dropped).
- underscore missing or not attributed to backbone.

## Probe metadata

```
patterns:             resolution-conflict, scoped-conflict, stale-resolution-entry
pm:                   bower
manifest:             bower.json
lockfile:             none (Bower has no lockfile)
detection_mode:       manifest-only (no bower_components/ committed)
total_packages:       5
direct_packages:      3
transitive_packages:  2
stale_resolution:     removed-pkg (must NOT appear in tree)
generated:            2026-05-04
target:               remote
remote_repo:          https://github.com/mend-detection-qa/bower-resolution-conflict-probe
```