# Probe: workspace-default-groups-python-req

## Metadata

| Field               | Value                                      |
|---------------------|--------------------------------------------|
| Pattern             | `workspace-default-groups-python-req`      |
| PM                  | uv                                         |
| PM version tested   | 0.12.22                                    |
| Categories          | `lockfile_format`, `tree_structure`        |
| Schema version      | 1.1                                        |
| Generated at        | 2026-10-02T00:31:22Z                       |

## Purpose

This probe exercises two lockfile schema changes introduced in
**uv 0.12.22**:

1. **Workspace-member default groups** — the lockfile now records
   which dependency groups are enabled by default for each workspace
   member using a `[package.metadata.default-groups]` entry. For
   example, `core` opts into `["dev"]` by default, and `app` opts into
   `["test"]`. Mend's Unified Agent lockfile parser must not reject or
   silently drop packages when it encounters this new field.

2. **Dependency-group Python requirements** — dependency groups can
   now declare their own `requires-python` constraint. In this probe
   the `docs` group on `core` carries a Python requirement
   (`>= 3.10`), expressed in the lockfile via
   `[package.metadata.requires-python-per-group]` and via a
   `python_full_version >= '3.10'` marker on the `mkdocs` package
   entry inside the member's `[package.dev-dependencies]`. Mend must
   parse this without crashing and attribute the `mkdocs` dep to the
   `docs` group with an appropriate marker.

## Workspace layout

```
workspace-default-groups-python-req-20261002-003122/
├── pyproject.toml          (virtual workspace root)
├── uv.lock                 (shared lockfile, uv 0.12.22 schema)
├── .python-version         (3.11 — Mend PIP-chain precedence)
├── packages/
│   ├── core/
│   │   ├── pyproject.toml  (requests main dep; dev + docs groups)
│   │   └── src/core/__init__.py
│   └── app/
│       ├── pyproject.toml  (click + core workspace dep; test group)
│       └── src/app/__init__.py
├── README.md
└── expected-tree.json
```

## Dependency groups summary

### `packages/core`

| Group  | Packages               | Default? | Python req |
|--------|------------------------|----------|------------|
| main   | requests 2.32.3        | always   | —          |
| dev    | pytest 8.3.5           | yes      | —          |
|        | coverage 7.6.10        | yes      | —          |
| docs   | mkdocs 1.6.1           | no       | >=3.10     |

### `packages/app`

| Group  | Packages               | Default? | Python req |
|--------|------------------------|----------|------------|
| main   | click 8.1.8, core 0.1.0| always  | —          |
| test   | pytest 8.3.5           | yes      | —          |

## Transitive closure (all packages in uv.lock)

Registry packages (PyPI):
- `certifi 2024.12.14` — transitive of `requests`
- `charset-normalizer 3.4.1` — transitive of `requests`
- `click 8.1.8` — direct dep of `app`
- `colorama 0.4.6` — transitive of `click` (win32 marker only)
- `coverage 7.6.10` — dev group of `core`
- `idna 3.10` — transitive of `requests`
- `iniconfig 2.0.0` — transitive of `pytest`
- `markdown 3.7` — transitive of `mkdocs`
- `mkdocs 1.6.1` — docs group of `core`, marker `python_full_version >= '3.10'`
- `packaging 24.2` — transitive of `pytest`
- `pluggy 1.5.0` — transitive of `pytest`
- `pytest 8.3.5` — dev group of `core` + test group of `app`
- `pyyaml 6.0.2` — transitive of `mkdocs`
- `requests 2.32.3` — direct dep of `core`
- `urllib3 2.3.0` — transitive of `requests`

Local (workspace) packages:
- `core 0.1.0` — editable workspace member, dep of `app`

## Mend config

**Bucket B — no `.whitesource` required.**

uv supports partial dynamic Python version detection:
Mend reads `.python-version` (higher precedence in the PIP chain)
and `[project] requires-python` from `pyproject.toml`. No `.whitesource`
is needed for this probe because the pattern does not target a specific
Python version mismatch or versioning regression.

Note: the `uv` tool itself is **not** in the `install-tool` list —
only the `python` key is pinnable via `scanSettings.versioning`. For
this probe, Python version detection is left to Mend's dynamic path.

## Python version detection

Mend's PIP-chain file precedence:
1. `.python-version` (this probe: `3.11`) — **Mend uses this**
2. `[project] requires-python` in `pyproject.toml` (fallback)

Both declare Python 3.11-compatible constraints. Mend will detect
Python 3.11 for this workspace.

## Mend failure modes targeted

1. **Parse failure on `default-groups`** — if the UA parser does not
   recognize `[package.metadata.default-groups]`, it may abort
   lockfile parsing entirely → 0 deps detected. Expected tree encodes
   the full transitive closure so this failure surfaces as a total miss.

2. **Parse failure on `requires-python-per-group`** — the new
   `[package.metadata.requires-python-per-group]` field triggers the
   same risk. If the parser errors out, 0 deps are detected.

3. **Group marker not respected** — if the parser does not apply
   `python_full_version >= '3.10'` to `mkdocs`, it either always
   includes or always excludes it. The expected tree encodes `mkdocs`
   with `marker: "python_full_version >= '3.10'"` so the comparator
   can detect this.

4. **Workspace members dropped** — only the virtual root is scanned,
   both `core` and `app` member deps are missed entirely.

5. **Group assignments flattened** — all packages collapsed to
   `group: "main"` losing `dev`, `docs`, and `test` group labels.

## Notes on expected-tree.json

- Schema version 1.1 is used because this is a workspace (multi-member)
  probe. The umbrella tree has `pm: "uv"`, `root.name: "workspace"`,
  and a flat `packages` map containing all resolved packages plus the
  workspace members as `source: "local"` entries.
- `mkdocs` carries `marker: "python_full_version >= '3.10'"` and
  `group: "docs"`.
- `colorama` carries `marker: "sys_platform == 'win32'"` and
  `group: "main"` (it is a conditional transitive of `click`).
- `pytest` appears once in `packages` (version 8.3.5) and is
  referenced by both `core` (group `dev`) and `app` (group `test`).
  The tree collapses it to a single entry; `group` is set to `"dev"`
  (first attribution from the lockfile walk). A warning entry documents
  the multi-group attribution.
