# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Install (Python 3.10+ required; use a dedicated venv)
python3.12 -m venv .venv && source .venv/bin/activate
pip install -e ".[gui]"        # GUI (napari + qtpy)
pip install -e ".[all]"        # GUI + dev tools
pip install PyQt5              # required separately if napari[all] was used (llvmlite conflict)
pip install imagecodecs        # required for LZW-compressed TIFFs

# Run the GUI
acetree-py gui path/to/config.xml

# CLI
acetree-py load config.xml                             # dataset summary
acetree-py rename config.xml --output renamed.zip      # run naming + save
acetree-py export config.xml --format cell_csv         # export
acetree-py info config.xml --cell ABala                # query cell

# Tests
pytest tests/
pytest tests/test_naming.py                            # single file
pytest --deselect tests/test_gui_widgets.py::TestNapariIntegration  # skip napari integration

# Lint
ruff check acetree_py/
```

**Installation gotchas:**
- `napari[all]` pulls numba→llvmlite which fails on macOS with Homebrew LLVM (version mismatch). Use `napari` (no extras) and install `PyQt5` separately.
- `imagecodecs` is a core dependency needed for LZW-compressed TIFFs; it's already in `pyproject.toml` but must be installed.
- `pyproject.toml` uses `license = {text = "MIT"}` (table form) for compatibility with `packaging<24`.

## Architecture

### Dependency layers (no circular deps)

```
gui/  ──►  core/  ◄──  naming/
 │           │            │
 ├──►  editing/           │
 └──►    io/  ◄───────────┘
```

`core/`, `naming/`, `editing/`, `io/` have **zero GUI imports** — CLI and headless operation work without Qt/napari.

### Central orchestrator: `NucleiManager` (`core/nuclei_manager.py`)

Owns `nuclei_record: list[list[Nucleus]]` (outer index = **0-based offset from `movie.start_time`**, inner index = nucleus within timepoint).

- **`from_config(config)`** — load ZIP, resolve AuxInfo, call `process()`
- **`process()`** — set_all_successors → compute_red_weights → `IdentityAssigner.assign_identities()` → `build_lineage_tree()`
- **`nuclei_at(time)`** — converts 1-based display time to 0-based record index via `time - movie.start_time`
- **`find_closest_nucleus(x, y, z, time)`** — 3D Euclidean nearest, z scaled by `z_pix_res`

### Data model

**`Nucleus`** (core/nucleus.py): one detected cell at one timepoint.
- `identity`: auto-assigned Sulston name, cleared and re-derived each `process()` run
- `assigned_id`: user-forced name, preserved through re-naming — the only field that survives `_clear_all_names()`
- `status >= 1` = alive; `status = -1` = dead/invalid
- `predecessor`, `successor1`, `successor2`: 1-based indices linking timepoints (NILLI = -1)
- `hash_key = str(time * 100000 + index)` — used to match Nucleus ↔ Cell in the lineage tree

**`Cell`** (core/cell.py): a lineage tree node spanning birth→division/death. `end_fate` = ALIVE | DIVIDED | DIED.

**`LineageTree`** (core/lineage.py): `cells_by_name` and `cells_by_hash` lookups; built by `build_lineage_tree()`.

### Cell naming pipeline (`naming/`)

`IdentityAssigner.assign_identities()` in `naming/identity.py`:
1. `_clear_all_names()` — clears `identity` for all nuclei **except** those with `assigned_id` set **or** whose identity contains "polar" (polar bodies have no `assigned_id` but must not be erased or they'll be miscounted as alive cells).
2. Propagate `assigned_id` forward/backward through lineage chains.
3. `identify_founders()` (`naming/founder_id.py`) — topology-based 4-cell stage identification of ABa/ABp/EMS/P2. Looks for contiguous windows with exactly 4 alive non-polar nuclei. Uses division-timing heuristics to identify sister pairs and assign canonical names.
4. Forward pass via `DivisionCaller` (`naming/division_caller.py`) — applies pre-computed rules from `resources/new_rules.tsv` (~620 rules) to name daughters at each division.
5. Remaining unnamed cells get generic `Nuc_t_z_x_y` names.

**Naming coordinate modes** (auto-selected by available data):
- v2 AuxInfo (`.csv` with AP/LR vectors): full Wahba-solver canonical transform
- v1 AuxInfo: sign-flip matrix + 2D rotation
- No AuxInfo: lineage centroid axes derived from ABa/ABp/EMS/P2 positions (rotation-invariant)

### I/O (`io/`)

**Config** (`io/config.py`): `AceTreeConfig` dataclass parsed from XML. Key defaults that affect display: `split=0`, `flip=0` (single-channel data should not be split). `load_config()` resolves relative paths against the config file's directory.

**Nuclei ZIP** (`io/nuclei_reader.py`): returns `(nuclei_record, min_time)`. The min_time (e.g. 0 for datasets starting at t000) is stored in `movie.start_time` and used in `nuclei_at()` for correct time-indexing.

**AuxInfo lookup** (`core/nuclei_manager.py::from_config`): tried in priority order:
1. ZIP stem → `{stem}AuxInfo_v2.csv`, `{stem}AuxInfo.csv`
2. Config dir / ZIP name → same
3. Config file stem → `{config_stem}AuxInfo_v2.csv`, `{config_stem}AuxInfo.csv`  ← catches mismatch when ZIP is `name-edit.zip` but AuxInfo is `nameAuxInfo.csv`

**Image providers** (`io/image_provider.py`): 7 implementations selected by `create_image_provider_from_config()`. For interleaved multichannel TIFFs (`<image file="..." numChannels="N" channelOrder="CZ|ZC"/>`), `StackTiffProvider` de-interleaves natively — **do not wrap in `SplitChannelProvider`**.

### Editing system (`editing/`)

Command pattern with `execute()` / `undo()`. `EditCommand.structural = True` (default) triggers naming + tree rebuild; `structural = False` (e.g. `MoveNucleus`) only refreshes display. `EditHistory` maintains undo/redo stacks (max 1000).

### GUI (`gui/`)

`AceTreeApp` (`gui/app.py`) owns the napari Viewer, `NucleiManager`, `EditHistory`, and all dock widgets. `ViewerIntegration` draws nucleus circles as napari Shapes layers. `LineageWidget` renders the Sulston tree in a `QGraphicsView`.

## Key invariants to preserve

- **Polar body identity**: `_clear_all_names()` must skip nuclei whose `identity` contains "polar" (case-insensitive), because `assigned_id` is empty for polar bodies and they have no other marker.
- **Time indexing**: `nuclei_record` is 0-based from `movie.start_time`. Always use `time - movie.start_time` as the index, not `time - 1`.
- **AuxInfo filename matching**: ZIP files with suffixes like `-edit` won't match plain `AuxInfo.csv` files unless the config file stem is also tried as a candidate.
- **No GUI imports in core/naming/editing/io**: these layers must remain importable without Qt/napari installed.
