# ADR-0007: Static Content Data Pipeline and Stable Type IDs

## Status
Proposed

## Date
2026-10-28

## Last Verified
2026-10-28 (against `design/gdd/item-and-module-database.md` IDB-3, IDB-4, IDB-26, IDB-27, IDB-30; `ship-section-and-layout-data-model.md` LY-2, LY-30; `life-support-graph.md` LS-2 to LS-5 and `GRAPH_BOOT_TYPE_ID`; `derelict-authoring-handmade-kits.md` DA-4, DA-41; `docs/architecture/tr-baseline.md` rows citing ADR-0007; ADR-0003 § 6; and `docs/engine-reference/godot/` `VERSION.md`, `breaking-changes.md`, `deprecated-apis.md`). No code exists and nothing here has been run on 4.7.2. The pinned engine reference has no core, scripting or export module, so every engine claim about `FileAccess`, `JSON`, `DirAccess` and export filters rests on training knowledge of APIs that predate 4.4 and is spike-bound (CONTENT-EXP-1).

## Decision Makers
Claude (author), running under `modes.automation: autonomous` (`project.yaml`), with every decision point logged in `production/session-logs/decision-log.md` for the user's review; the user has not yet reviewed this ADR. Reviewed by godot-specialist (engine: idiomatic; three blocking detail errors and fifteen minor notes, all applied) and technical-director (TD-ADR: CONCERNS; findings 1 to 12 applied in this revision). See Review Log.

## Summary
Sixty-five requirements from fifteen GDDs need static authored data (items, section templates, world records, tuning tables, lint configuration) to load identically in the editor, headless tests and a release export, to reach the simulation only as integers, and to keep ids stable across saves. This ADR fixes one pipeline: integer-only JSON under `assets/data/` listed in committed manifests (never a directory scan), read through an injected `ContentSource`, parsed with `JSON.new().parse`, cast through closed per-table schemas into write-once typed `RefCounted` catalogs, validated by one flat domain-2 boot-error list plus an offline lint that reuses the same loaders, and keyed by append-only id ledgers that CI diffs against a base revision.

## Engine Compatibility

| Field | Value |
|-------|-------|
| **Engine** | Godot 4.7.2 (stable), GDScript |
| **Domain** | Core (file I/O, JSON parsing, project export configuration, data-class design) |
| **Knowledge Risk** | LOW for the APIs (`FileAccess.open`, `FileAccess.get_open_error`, `FileAccess.get_as_text`, `JSON.new().parse`, `JSON.get_data` / `get_error_line` / `get_error_message`, `Array.make_read_only` / `is_read_only`, `roundi`, `Performance.get_monitor(OBJECT_COUNT)` all predate 4.4 and are not listed in `breaking-changes.md` for 4.4 to 4.7). MEDIUM confidence overall, because the pinned reference has no core/scripting/export module: JSON number typing, the include-filter glob syntax and the export of `.json` files are training-knowledge claims, bound to spike CONTENT-EXP-1. The 4.7 rows that matter are the typed-return override rule (every `ContentSource` override returns a `ContentRead` on all paths) and the packed-array element-write change (not engaged: catalogs have no setters). |
| **References Consulted** | `docs/engine-reference/godot/VERSION.md`, `breaking-changes.md` (4.4 `FileAccess.store_*` now return `bool`: irrelevant, runtime never writes content; 4.6 to 4.7 GDScript rows), `deprecated-apis.md` (untyped Array/Dictionary pattern row), `current-best-practices.md` (not read for this ADR; its GDScript 4.5 to 4.7 rows are cited through ADR-0003). No module file exists for this domain. |
| **Post-Cutoff APIs Used** | None required. Typed `Dictionary[K, V]` (4.4) may be used for key-indexed lookup maps. |
| **Verification Required** | **CONTENT-EXP-1 (gates acceptance).** On a `template_release` export for Windows **and for Linux / Steam Deck** (case-sensitive file system), with the preset filters in § 1.6, a headless boot reads every manifest-listed file with `FileAccess.open`, builds the same `ContentSet` as the editor, and yields identical content hashes (AC-178). Record: (a) whether `.json` files are exported raw via `include_filter` or as `JSON` resources through Godot's built-in JSON loader, and whether `FileAccess.open` on the original `res://` path works in both cases (both outcomes are recorded, since they decide whether `EXPORT_FILTER_MISSES_MANIFEST` is needed); (b) the `include_filter` / `exclude_filter` glob behaviour for nested paths (expected: comma-separated, `matchn`, `*` matches `/`); (c) that `JSON.new().parse` returns `float` for every number (`40000`, `40000.0`, `1e9`) and what `9007199254740993` parses to; (d) whether `get_as_text()` keeps a leading U+FEFF; (e) how the parser treats a duplicate key in one object, a trailing comma, a comment and trailing garbage; (f) `FileAccess.open` returns `null` for a missing file with `get_open_error()` set, in editor and export; (g) that `const` schema rows of plain literals compile and `ContentLoadResult.content` (not `set`) is accepted as a member name. **CONTENT-PERF-1 (does not gate acceptance):** AC-68 and AC-69 on the real `ItemCatalog` (25 Objects after all loader temporaries are out of scope, load P99, no new Objects over 10,000 queries, plus the identity checks in § Validation). |

## ADR Dependencies

| Field | Value |
|-------|-------|
| **Depends On** | ADR-0001 (composition root injects; no autoload or global; Proposed), ADR-0003 (fault domain 2 and the `*_faults.gd` convention; Proposed). |
| **Enables** | ADR-0011 (Graph data model: type tables loaded by this pipeline), ADR-0013 (Layout block: templates, ids and hashes), ADR-0017 (Emitter reads Failure & Trail's glow table as static content), ADR-0010 (lint tool enforces § 2 and § 3), and every Item DB, Layout template, Derelict Authoring, Horror, Sanity, Audio, Instruments, HUD and Chart Table data story. |
| **Blocks** | Epics: Item & Module Database, Ship Section & Layout (template loading), Derelict Authoring (lint and converter), and every story that adds a data file under `assets/data/`. |
| **Ordering Note** | **Forward references (not yet written, not blocking authorship):** ADR-0005 owns the canonical byte encoding and `H64`; this ADR fixes only the hash *input* (post-cast typed integers). ADR-0006 owns the save header; this ADR only defines and offers `catalogue_version`. ADR-0010 owns the source layout and the location of shared pure functions (§ 1.9). Acceptance requires ADR-0001 and ADR-0003 Accepted and CONTENT-EXP-1 passed. **Scope edge with ADR-0003:** content loads *before* the kernel exists (architecture § 8 steps 3 and 4), so content boot failures are handled by the composition root, not by the kernel's `boot_validation_failed` trigger (§ 2.3). This ADR uses ADR-0003's domain 2 exactly as that table defines it ("content data: item DB, templates, tuning"). |

## Context

### Problem Statement
Twenty-six GDDs make gameplay values, templates, catalogues and id spaces data-driven. Each GDD solved the same four problems locally and deferred the shared answer to this ADR: (1) the engine's JSON parser returns floats, but the simulation is integer-only; (2) a directory scan or an unlisted file can pass in the editor and fail or reorder in a release export; (3) a catalogue read by `validate`, P3 and presentation must be immutable, scene-tree free and allocation-free on lookup; (4) ids that reach a save, a snapshot or a hash must never change meaning. Without one decision, the Item DB, Layout, Graph, Derelict Authoring, Horror, Sanity and Audio would each ship a slightly different loader, and the first exported build would find the differences.

### Constraints
- **Integer-only simulation** (kernel, Graph LS-1, Layout LY-2). A float never reaches a sim read; designer floats convert once, at load.
- **No directory scan, no unordered iteration** (LY-2, IDB-4, LSG 64). The sim never iterates a `Dictionary`.
- **Same boot assertions in editor and export** (TR-sanity-026, AC-SN-178).
- **No autoload, static variable or group lookup** exposes content (registry `kernel_autoload_or_global`, ADR-0001); many catalogues coexist in one process (TR-itemdb-002).
- **Catalogs are never `Resource`** (registry `block_as_resource_or_node` extended by the same reasoning; SL-2; IDB-26 i).
- **No `assert`, no `push_error`, no `quit` in a loader** (`assert` is stripped in release; `push_error` trips gdUnit4; a loader that quits cannot be tested).
- **No `res://` path on any field that can reach a save** (IDB-26).
- **Unit tests do no file I/O** (coding standards): the loader needs a file-access seam.
- **Fault codes** follow ADR-0003 § 6: `(DOMAIN_ID << 16) | local`, one `*_faults.gd` per domain, `NAMES` indexed by local code, append-only.
- **Dependency direction** (architecture "Which way dependencies may point"): Foundation code under `src/data/` must not import `src/sim/`; no `preload` of a sibling system's script.
- **Load budget:** `ItemCatalog` load plus freeze creates exactly 25 Objects and P99 <= 14.9 ms desktop / 30.0 ms Deck, pending spike (TR-itemdb-024).
- **Pre-release saves are disposable** (LSG 832); ids become binding at the first build given to anyone outside the project.

### Requirements
- One `ContentLoader` path for every static table, used unchanged by the game, gdUnit4 fixtures and the offline lint.
- A closed schema per table; unknown key, missing key, wrong type, float where an integer is required, and out-of-range value are errors with a stable code.
- A committed manifest per content domain; load order is the manifest's order, itself required to be sorted.
- Immutable catalogs with get-only fields, read-only array exposure, non-allocating lookups and a `NO_DEF` sentinel for misses.
- Append-only id ledgers with a CI append-only gate and a boot check that live content agrees with the ledger.
- Boot errors are values (`Array[BootError]`), never side effects.
- An export configuration that provably includes every manifest-listed file.

## Decision

Three numbered sub-decisions, each reviewable alone: **§ 1 the loading pipeline**, **§ 2 boot validation and lints**, **§ 3 stable ids, ledgers and versioning.**

### Architecture Diagram

```
assets/data/<domain>/manifest.json   (committed, sorted, explicit; never a directory scan)
assets/data/<domain>/*.json          (integer-only JSON, LF line endings; one closed schema per file)
assets/data/<space>/ids.lock.json    (append-only id ledgers)
        │  read_text(path)
        ▼
ContentSource  ── FileContentSource (FileAccess; game, export, CI tool)
   (seam)      └─ MemoryContentSource (unit tests, no I/O)
        │
        ▼
ContentLoader.load_all() ── stage 1: type tables (Graph LS-2..5) + GraphTuning
   (RefCounted, no        ── stage 2: Item DB
    static / autoload)    ── stage 3: section templates (manifest order)
                          ── stage 4: world records, verb / tuning / lie / audio / instrument tables
                                  │ per file: parse → closed schema → int-cast → freeze
                                  │ errors: collect all within a stage; stop after the first failing stage
                                  ▼
                          ContentLoadResult { content: ContentSet | null, errors: Array[BootError] }
                                  │
              errors empty ◄──────┴──────► errors non-empty
                   │                              │
    composition root injects the frozen      root never constructs the kernel; draws a
    catalogs each module needs at setup()    text boot-failure surface in the step-2
                                             viewport (headless: prints, exits non-zero)

Offline: tools/ci/content_lint.gd = the same ContentLoader + the file-pure CI rules (§ 2.2)
CI steps: history gates (ledger, version bump) feed base-revision files as a second ContentSource
```

### 1. The loading pipeline

**1.1 Format.** Static content is JSON under `assets/data/<domain>/`, UTF-8 without BOM, LF line endings (`.gitattributes`: `*.json text eol=lf`), one closed schema per file. JSON is chosen over `Resource` (Alternative 1) and code-as-data (Alternative 2). MVP domains: `items/`, `graph/` (type tables, `GraphTuning`), `sections/`, `world/` (`mvp_world.json`, `legs.json`), `interaction/`, `suit/`, `sanity/`, `horror/`, `attention/`, `instruments/`, `audio/`, `ui/` (`hud_tuning`, `chart_layout`), `save/`, `lint/` (`content_lint_config`). A new domain adds a directory, a manifest and a schema; it needs no ADR. The Failure & Trail glow table (`GLOW_BASE_PM`, `SEV_GLOW_PM`) is Content data under this ADR, still authored by Failure & Trail (architecture IC-12).

**1.2 Manifests, not scans.** Every domain, the Item DB included, has a committed `manifest.json`: `{ "format": 1, "files": [ { "key": "<table or template_id>", "path": "<path relative to the domain directory>" } ] }`, with `files` sorted ascending by `key` (byte order). The loader opens exactly those paths. A manifest path that does not exist is `CONTENT_FILE_MISSING`; an unsorted or duplicate key is `CONTENT_MANIFEST`. A domain whose GDD fixes the file set (the Item DB's class files, `meta.json`, `ids.lock.json`) also has its schema assert that the manifest lists exactly that set, so there is one mechanism and one lint path. Game code never enumerates `res://assets/data/` with `DirAccess`. The offline lint may, and reports `UNLISTED_DATA_FILE` for any `.json` under `assets/data/` that no manifest names, and `MANIFEST_PATH_CASE` for any manifest path whose case differs from the file on disk (the Windows editor is case-insensitive; the package and the Deck are not).

**1.3 The file-access seam.** Each `class_name` is its own file.

```gdscript
# src/data/content_source.gd
class_name ContentSource extends RefCounted
## Reads one text file by path. Never throws; failure is a value.
func read_text(path: String) -> ContentRead:
	return ContentRead.failed(ContentFaults.FILE_MISSING, path)   # overridden; every override returns on all paths (4.7)
```

```gdscript
# src/data/content_read.gd
class_name ContentRead extends RefCounted
var ok: bool
var text: String
var error: int            # ContentFaults code when not ok
var path: String
```

`FileContentSource` calls `FileAccess.open(path, FileAccess.READ)`, checks for `null` and reads `FileAccess.get_open_error()` on failure, then `get_as_text()`. `open` is preferred over `FileAccess.get_file_as_string` only because it gives an explicit `null` check at the failure point; both report through `get_open_error()`. It does not use `ResourceLoader`. If `text.begins_with("﻿")` the read fails with `CONTENT_JSON_PARSE` (detail `BOM`), whatever the parser would do. `MemoryContentSource` serves a `Dictionary[String, String]` fixture and is the only source unit tests use.

**1.4 Parse, then cast, then freeze.** Every file goes through the same three steps, implemented once.

1. **Parse** with `JSON.new().parse(text)`, read with `get_data()`; never `JSON.parse_string` (which returns `null` and prints an error that trips gdUnit4). A failure yields `CONTENT_JSON_PARSE` carrying `get_error_line()` and `get_error_message()`; tests assert the code and the line, never the engine's message text.
2. **Cast** through the table's schema. JSON numbers arrive as `float` (CONTENT-EXP-1 c). `IntField.read(value, lo, hi)` accepts a `TYPE_INT`, or a `TYPE_FLOAT` that is integral (`value == floorf(value)`) and strictly below 2^53 in magnitude (`absf(value) < 9007199254740992.0`); a fractional value or a `TYPE_BOOL` is `CONTENT_FIELD_TYPE`, a magnitude at or above 2^53 is `CONTENT_INT_PRECISION` (the bound is exclusive because `9007199254740993` silently parses to 2^53), outside `lo..hi` is `CONTENT_FIELD_RANGE`. A designer-facing fractional field (a 1.5-tile radius) is declared `SCALED` with a `scale` and converted once by `roundi(x * scale)` (half away from zero, negatives included); the lint `FLOAT_ROUND_AMBIGUOUS` rejects a value whose scaled form lies within 1e-6 of a .5 boundary.
3. **Freeze** into a catalog (§ 1.5). After freeze nothing in the `ContentSet` changes for the life of the process.

**What parsing hides, the raw-text lint sees.** After parsing, `40000.0` and `40000` are the same float, and a duplicate key inside one object has silently kept its last value. So the offline lint scans the raw text of each file and reports `INT_LITERAL_SHAPE` (a `.` or exponent in an integer-typed field, or a literal at or above 2^53) and `JSON_DUP_KEY` (a key repeated inside one object). These are CI ERRORs. At runtime, `1e9` is refused by range and `40000.0` is accepted and converted once.

**Closed schemas live in code as literal rows.** A schema is part of the program, so it is GDScript, not data, under `src/data/<domain>/<table>_schema.gd`. GDScript `const` cannot hold constructed objects, so the schema is a `const` array of plain literal rows (`[name, kind, lo, hi, required, unit, scale]`), turned into typed `FieldSpec` objects by `static func fields() -> Array[FieldSpec]` when the loader starts. There is no `static var`; a static, immutable schema is not what `kernel_autoload_or_global` forbids (that rule covers access to the kernel, blocks and views). `kind` is one of `INT`, `SCALED`, `ENUM_ID`, `STRING_ID`, `INT_ARRAY`, `ROWS`. An unknown key is `CONTENT_FIELD_UNKNOWN`; a key on the domain's forbidden list (horror: `damage`, `accuracy`, `armour`, `dps`, `xp`, `unlock`, `health`, `hp`, `kill`, `loot`, `threat`, `aggro`, `hate`; items: price, rarity, recipe; interaction: stats, cooldown, reward) is `CONTENT_FIELD_FORBIDDEN`. The forbidden check runs before the unknown-key check, so the code names the intent. The only permitted non-schema key is `"_note"`, a string stripped by the loader and never hashed. Named structural invariants (§ 2.5) are declared in the same schema file.

**Tables are arrays of rows, never id-keyed objects.** A JSON object parses into a `Dictionary`, and the simulation never relies on dictionary order. Every table is an array of row objects `{ "id": ..., ... }` sorted ascending by its key; an out-of-order or duplicate row is `CONTENT_ORDER` / `CONTENT_DUP_KEY`. Load order, `defs_of_class` order and every later hash input are therefore a pure function of the file.

**Tick-expressed values.** A field whose `unit` is `ticks` or `px_per_tick` is valid only with the file header `authored_tick_rate`; the loader compares it with the kernel's `TICK_RATE` constant and refuses with `CONTENT_TICK_RATE` on mismatch (TR-pc-015, TR-interact-030). A file with no tick-unit field carries no header.

**1.5 Immutable catalogs ("the catalog pattern").** Every catalog follows one pattern, taken from the Item DB (IDB-26 i to v) and applied to all:

- A `RefCounted` class constructed with its inputs injected (`ItemCatalog.new(data, graph_tables)`), scene-tree free, with no static, singleton or autoload state, so many catalogs coexist in one process.
- **Nested-row tables** (items, templates, world records, instrument and lie-target rows) use a typed `RefCounted` row class (`ItemDef`, `RoomTemplate`...). **Flat scalar tables** (`GraphTuning`, `hud_tuning`, Attention and Save/Load knobs) use one generic `ScalarTable` built from the schema, exposing named `int` getters; they do not each need a row class.
- Fields are underscore-prefixed, written once by the loader inside `_freeze()`, and exposed through get-only accessors. **GDScript has no private fields**, so write-once is a convention enforced by the ADR-0010 lint (no write to a catalog `_field` outside its own `_freeze`) and by tests, not by the language.
- **Array exposure:** array fields are typed `Array[T]` (`Array[int]` for integer lists) stored once and marked `make_read_only()`; the getter returns that same array every call, so a lookup allocates nothing and a caller's `append` fails. **Packed arrays are never returned from a catalog**: GDScript passes packed arrays by reference and they cannot be made read-only, so one caller could corrupt what every other caller sees. A hot path that needs packed storage uses index accessors (`count()`, `at(i)`) and exposes no container. Per-call `duplicate()` is not allowed (it allocates, AC-69).
- Every query is pure and read-only, callable from `validate`, P3 and presentation (TR-itemdb-004). A miss returns the catalog's `NO_DEF` sentinel (a frozen row of the same class), never `null`.
- Iteration over a table is only over the sorted arrays (`defs_of_class`, `templates_in_order`), never over a `Dictionary`. A key-indexed map (a typed `Dictionary` is fine) exists for lookup only.
- Catalogs are not state blocks: they are written once before the kernel exists and have no setters, so `write_outside_writescope_or_setter_dirty` does not apply to them.
- No runtime reload. A changed data file takes effect at the next process start; a debug build may build a **new** `ContentSet` for a fresh kernel (tests, MIRROR comparison) but never mutates the frozen one.

**1.6 Export.** The release preset's `include_filter` is `assets/data/*.json` (intent: every `.json` below `assets/data/`, any depth; the editor matches each pattern with `matchn` against the full path, the path without `res://` and the file name, and `*` matches `/`; CONTENT-EXP-1 b confirms). `exclude_filter` removes `tests/*`, `tools/*`, `production/*`, `design/*`, `docs/*` and `addons/gdUnit4/*`. Two checks keep the filter honest: the CI lint `EXPORT_FILTER_MISSES_MANIFEST` parses the preset and tests every manifest path with the same three-way `String.matchn` rule, and the **exported-build smoke check** (AC-178, AC-SN-178) boots a `template_release` export headless to the first `RUNNING` tick and compares content hashes with the editor run. Godot also has a built-in loader for the `JSON` resource type; if CONTENT-EXP-1 shows that `.json` files are exported as resources and `FileAccess.open` fails on them, `FileContentSource` switches to `ResourceLoader` behind the same seam and nothing else changes.

**1.7 Load stages and the boot-order resolution.** Four fixed stages:

1. **Type tables and tuning other tables validate against:** Graph LS-2 to LS-5 and `GraphTuning` (`assets/data/graph/`).
2. **Item DB** (needs the LS-4 port-type ids and the pure `connectable`, IDB-30).
3. **Section templates** (manifest order; needs the Item DB for `def_id` and hardpoint checks and the LS tables for node kinds).
4. **World records, verb tables, tuning tables, lie targets, audio and instrument tables, chart layout, lint config.**

This resolves a wording difference between sources: architecture § 8 and TR-lsg-030 say "Item DB, then Layout, then `graph.state` registers last", while IDB-30 says the Item DB loads after the Graph's type tables. Both hold, of different things: **static type tables load first (stage 1), catalogs follow (stages 2 to 4), and the `graph.state` *block* registers last in the kernel's `BOOTING`** (block registration order is ADR-0003 / ADR-0016). Content never loads after the kernel starts, and the kernel never reads a file. "Loaded in BOOTING" in GDD and TR text (TR-section-003, LY-2) means "loaded during start-up, before kernel `BOOTING`" (see Migration Plan).

**1.8 Injection.** The composition root owns the `ContentLoader`, runs it once, and passes each module the frozen catalog it needs (`ItemCatalog`, `SectionTemplates`...) through `setup()`. Catalogs are already read-only, so no separate view facet is built in the MVP; the narrowing that matters is that no module receives the whole `ContentSet`. There is no `ContentSet` global.

**1.9 Where schema and validation code lives.** Every `*_schema.gd`, every named invariant, and every pure function the loader calls (the Graph's `connectable` and `cap_seam`, the Derelict BOOT checks) live under `src/data/` or the shared pure-function location ADR-0010 defines. The owning system owns that code **by review, not by import**: `src/data/` never imports `src/sim/`, and nothing `preload`s a sibling system's script (architecture "Which way dependencies may point", item 2).

### 2. Boot validation and lints

**2.1 Three tiers.**

| Tier | When | On failure | Members |
|---|---|---|---|
| **BOOT** | Every start, editor and export alike, inside `ContentLoader.load_all()` | `Array[BootError]` non-empty; the game does not start | schema errors; `ITEM_BOOT_*`; `GRAPH_BOOT_TYPE_ID`; Layout structural ERRORs (LY-30); the Derelict BOOT rules (`PLACEMENT_DEF_UNKNOWN`, `SELF_POWERED_LAMP_COUNT`, `BEACON_COUNT`, `STORY_SLOT_LIST`, `RESIDUAL_LIGHT_COUNT`, `DEAD_FAULT_NOT_DRIED`); `SANITY_BOOT_LIE_TARGET_DATA`; Attention abortability and survivable-dark invariants; Save/Load and Item DB knob invariants; `CONTENT_TICK_RATE`; ledger agreement |
| **CI** | Before merge and on push (§ 2.2) | CI fails (ERROR) or reports (WARN / INFO) | the rest of the 90-row Derelict lint table (tranches T1 to T3), `WEIGHT_CLASS_MARGIN_PM`, the junk tell, instrument lints (`BEYOND_WITNESS_MISMATCH`...), audio identity lint, `UNLISTED_DATA_FILE`, `MANIFEST_PATH_CASE`, `EXPORT_FILTER_MISSES_MANIFEST`, `INT_LITERAL_SHAPE`, `JSON_DUP_KEY`, `FLOAT_ROUND_AMBIGUOUS`, `GENERATED_FILE_DRIFT`, `LEDGER_NOT_APPEND_ONLY`, `VERSION_BUMP_MISSING`, `TUNING_SOURCE_DRIFT` |
| **ADVISORY** | CI | reported, never fails | published Tuning Knob ranges versus `tests/fixtures/<system>/safe_ranges.json` |

**2.2 One loader, two kinds of CI rule.**
- **File-pure rules** are a deterministic function of the content files and `content_lint_config` (rows `{name, value, owner, status: LOCKED | PLACEHOLDER}`). They run in `tools/ci/content_lint.gd` through the same `ContentLoader`, so parsing and schemas are never reimplemented. Output is ordered by `(template_id, lint_id, record_key)`; a waiver needs a recorded reason and no waiver may name `STORY_SLOT_FILLED_IN_MVP` (TR-derelict-024). The CI job runs `godot --headless --import` once before the tool so `class_name` scripts resolve.
- **History and cross-artifact gates** are not file-pure. `LEDGER_NOT_APPEND_ONLY` and `VERSION_BUMP_MISSING` compare with a **base revision**: the merge base for a pull request, the push's `before` SHA for a direct push to `main` (trunk-based). A CI step checks out the base revision's `assets/data/` and passes it to the same GDScript tool as a second `ContentSource`. `EXPORT_FILTER_MISSES_MANIFEST` parses `export_presets.cfg` as `ConfigFile`. `TUNING_SOURCE_DRIFT` compares `LOCKED` rows with `docs/registry/architecture.yaml`; GDScript has no YAML parser, so it is a non-GDScript CI step (the existing `.claude/scripts/registry.sh` reader or a small Python step) over the tool's JSON output.

**2.3 Boot errors are values; one domain; who acts on them.** A `BootError` is a plain class `{code: int, table: StringName, key: String, field: StringName, line: int, detail: String}` (the Item DB's `{code, class_or_table, id, def_id, field, detail}` generalised: `class_or_table` becomes `table`, `id` and `def_id` collapse into `key`, `line` is added).

**Every content boot error is a domain-2 code** in `src/data/content_faults.gd` (`class_name ContentFaults`, `const DOMAIN_ID := 2`), as ADR-0003 § 6 defines domain 2: "content data (item DB, templates, tuning)". GDD-named codes keep their names and are listed there: `ITEM_BOOT_*`, `GRAPH_BOOT_TYPE_ID`, the Layout structural and Derelict BOOT rule names, `SANITY_BOOT_LIE_TARGET_DATA`. The owner domains keep their **runtime and restore** faults (`GRAPH_RESTORE_UNKNOWN_DEF` stays in domain 3). This keeps the loader free of imports from Core and Feature fault files. The local codes are **one flat, append-only list** (`NAMES` indexed by local code, as ADR-0003 requires, no numeric sub-ranges), grouped in the file by comment headers per owning table. Initial pipeline-generic entries: `FILE_MISSING`, `JSON_PARSE`, `MANIFEST`, `FIELD_TYPE`, `FIELD_RANGE`, `FIELD_UNKNOWN`, `FIELD_FORBIDDEN`, `INT_PRECISION`, `ORDER`, `DUP_KEY`, `TICK_RATE`, `LEDGER`, `INVARIANT`, `SCHEMA_VERSION`; owner-named codes follow as each data story lands.

**Collection policy.** Within a stage the loader collects every error from every file, so a designer sees the whole list in one run; after a stage with errors it stops, because later stages validate against earlier ones.

**Content loads before the kernel exists**, so a non-empty error list is acted on by the **composition root**: it does not construct or boot the kernel, prints each error as `NAME table key field (line N): detail` using `ContentFaults.NAMES`, draws a text-only boot-failure surface in the step-2 viewport in a normal run (owned by the root; ADR-0030 may restyle it), and exits non-zero in headless or CI runs. The loader itself never quits. Domain 2 is **not** registered with the kernel at MVP: no runtime path raises a domain-2 code (restore-time unknown ids are raised by the holder in its own domain). It is registered by the composition root only if a later ADR adds such a path.

**2.4 Which rules are BOOT and which are CI.** A rule is BOOT if violating it makes the simulation unsafe or non-deterministic when run (schema, ids, ledger, tick rate, structural geometry, a placement that would call `fresh_instance` on an unknown def, Attention invariants). It is CI if it judges authored *quality* or cross-file content balance (the lint table, balance margins, fairness and traversal budgets). Applied once to the Derelict table so no story re-decides it: the six BOOT rules of TR-derelict-023 are BOOT (including `PLACEMENT_DEF_UNKNOWN`, which TR-derelict-016 requires to fail "lint/BOOT, not runtime"); `TILES_WORLD_MVP` and authored-node caps (TR-derelict-020), `G0_ZERO_PART_SURVIVABLE` (TR-derelict-014), the part ledger (TR-derelict-015), traversal and vacuum budgets (TR-boarding-025) and the palette and never-do rules (TR-derelict-029) are CI ERRORs. BOOT stays cheap and bounded: it runs on every start with a measured budget (Item DB: TR-itemdb-024, pending CONTENT-PERF-1); a CI rule too costly for boot never moves to BOOT.

**2.5 Coupled tuning ranges (OQ-9).** Published Tuning Knob ranges are *guidance*, generated from one fixture per system (`tests/fixtures/<system>/safe_ranges.json`) and CI-diffed against the GDD table (TR-sanity-025); they are never boot assertions. Boot asserts each owner's **named structural invariants** (`WEIGHT_MEDIUM_G < WEIGHT_HEAVY_G`, retry <= max_defer / 4...), declared in the table's schema file and evaluated over the post-cast integers, failing with `CONTENT_INVARIANT(table, name)`. A new invariant is a schema change reviewed with its owner, not a lint-config edit.

**2.6 Offline tools.** Generators (the Derelict plan-grammar converter, `tools/asset-pipeline/plans/<template_id>.txt` to `tile_runs`, wall runs and the rect table, DA-43) are headless GDScript tools that **write committed JSON**. They are deterministic (two runs byte-identical), the generated file is the runtime source of truth, and CI re-runs each generator and diffs (`GENERATED_FILE_DRIFT`). Nothing generates content at runtime. This settles TR-derelict-027's "converter ADR prerequisite": the converter needs no ADR of its own, only this convention.

### 3. Stable ids, ledgers and versioning

**3.1 The rule.** An id that reaches a save, a snapshot, a hash, a replicated record or a `SalvageRecord` is one of three things only:
1. a **stable string key** chosen by a human (`def_id`, `template_id`, `room_key`, `derelict_id`);
2. an **integer in an append-only ledger** (§ 3.2);
3. a **Graph type-table id** (loop, node kind, port type, edge kind, mode, state), fixed by code constants (§ 3.2).

It is never an array index, a file or manifest position, a `Dictionary` iteration position, a hash of a path, or a `ResourceUID`. **Item `type_id` specifically never reaches a save, snapshot or hash** (IDB-3: it stays in memory and in vertical-slice replication summaries only); persisted item records carry `def_id`.

**3.2 Ledgers and code-fixed tables.**

*Ledgers.* An id space whose keys are persisted or hashed and that content authors extend has a committed `ids.lock.json`: `{ "space": "item_defs", "format": 1, "state": "PRE_RELEASE", "pairs": [ { "key": "mod_o2_tank", "id": 1001, "dormant": false } ], "retired_ids": [ 1002 ] }`. Required for the MVP: `item_defs` (IDB-4, `assets/data/items/ids.lock.json`) and `section_templates` (`template_id` and each template's `room_key`s, `assets/data/sections/ids.lock.json`, ADR-LAY-04; nesting per OQ-B). Other spaces (horror `type_id`, sanity surface enums, instrument ids, audio tokens) are unique- and range-checked at boot and gain a ledger when they first reach a save or hash.

*Boot check.* The generic `LedgerCheck.verify(live_pairs, ledger)` refuses with `CONTENT_LEDGER` (named `ITEM_BOOT_TYPE_ID(class, id)` for the Item DB) when:
- two live entries share an id;
- an id lies outside its space's declared range;
- a key's prefix disagrees with its class;
- an id is in `retired_ids`;
- a shipped key appears with a different id;
- a ledger pair is missing from live content without `dormant` or a `retired_ids` entry.

A **dormant** pair is cut from the current build but may return unchanged; no other key may take its key or id. A **retired** id is never reused.

*Code-fixed Graph tables.* The Graph's type tables are fixed enumerations whose **baseline is the code-side enum constants** owned by the Graph (ADR-0011). Boot checks that `assets/data/graph/` equals those constants exactly, refusing a duplicate, a missing entry or a renumber with `GRAPH_BOOT_TYPE_ID(table, id)`. A renumber therefore needs a reviewed code change, and ADR-0011 owns their stability; they need no ledger.

*CI append-only gate (`LEDGER_NOT_APPEND_ONLY`).* CI compares each ledger with the base revision (§ 2.2):
- pairs are only added;
- an existing pair's key or id never changes;
- `retired_ids` only grows;
- `dormant` may flip either way.

While `state` is `PRE_RELEASE` the gate reports WARN, because pre-release saves are disposable. The commit that sets `state` to `SHIPPED`, made when the first build leaves the project, is one-way and turns the gate into an ERROR.

**3.3 Renames and removals** (IDB-27). The format reserves `renamed_from[]` and `replaced_by` on defs now. Their behaviour is needed only after `SHIPPED`: transitive resolution to `ALIAS_DEPTH_MAX` 8 at restore through `resolve_alias`, and `ITEM_BOOT_ALIAS` on a cycle, a self-alias or an alias equal to a live key. It is implemented in a story that is a **prerequisite of the `SHIPPED` flip**, not in the MVP epic. A removed def that names no `replaced_by` fails restore loudly with the holder's named failure (`GRAPH_RESTORE_UNKNOWN_DEF` for Graph nodes).

**3.4 Versioning.** `catalogue_version` is a single integer in `meta.json`:
- It is bumped on any def removal, rename or field-shape change.
- It is **not** bumped for a value tuned within its type (F-9).

`VERSION_BUMP_MISSING` compares the catalog's structural signature (keys, field set, removed or renamed defs) with the base revision. It is WARN while the ledger is `PRE_RELEASE` and ERROR after `SHIPPED`. This ADR offers the integer to Save/Load's header, and ADR-0006 places it.

`catalogue_hash` is **diagnostic only and never gates a load**. It is `H64` over the canonical bytes of the post-cast typed rows, sorted ascending by `def_id` (byte order), so a legal renumber never changes it. The encoding and `H64` are ADR-0005's. This ADR fixes the *input*: typed integers after cast, never file bytes, never key order, never float spelling.

**3.5 Template identity and hashes.** `template_id` and `room_key` are stable keys under the `section_templates` ledger. `geometry_hash`, `visual_hash` and `template_hash` are `H64` of the canonical encoding of the post-cast template (ADR-0005). They are computed once at load and never stored in the template file. A template edit between builds is handled by Layout's drift policy (LY-26 e). Changing one authored value by 1 changes `template_hash` and `state_hash` (TR-derelict-007).

### Key Interfaces

```gdscript
class_name ContentLoader extends RefCounted
func _init(source: ContentSource, kernel_tick_rate: int) -> void: ...
func load_all() -> ContentLoadResult: ...      # the only entry; runs stages 1-4 of § 1.7

class_name ContentLoadResult extends RefCounted
var content: ContentSet        # null when errors is non-empty ("set" would clash with Object.set)
var errors: Array[BootError]   # empty on success; all errors of the failing stage
var stage_failed: int          # 0 = none, else 1..4

class_name BootError extends RefCounted
var code: int                  # (2 << 16) | local, ContentFaults
var table: StringName
var key: String
var field: StringName
var line: int                  # JSON line when known, else 0
var detail: String

class_name IntField extends RefCounted
static func read(v: Variant, lo: int, hi: int) -> IntRead   # IntRead { ok, value, code }; boot-only, outside the 25-Object count

class_name LedgerCheck extends RefCounted
static func verify(live_pairs: Array[LedgerPair], ledger: IdLedger) -> Array[BootError]
```

(Each `class_name` above is its own file.) Catalog queries (`def`, `has`, `weight_class`, `repair_part_for`, `compatible_defs`, `defs_of_class`, `salvage_yield`) are specified by the Item DB GDD (IDB-28); this ADR fixes only their shape (§ 1.5).

### Implementation Guidelines

- **Must** load every static table through `ContentLoader` and a `ContentSource`; **must never** call `FileAccess`, `JSON` or `DirAccess` for content anywhere else in `src/`.
- **Must** parse with `JSON.new().parse` and cast every number through `IntField.read` or a declared `SCALED` field; **must never** use `JSON.parse_string` for content, and **must never** let a float from a parse reach a sim field, a hash input or a saved value.
- **Must** read only manifest-listed paths whose case matches the file on disk; **must never** enumerate `res://assets/data/` with `DirAccess` in game code.
- **Must** write tables as arrays of rows sorted by key; **must never** author an id-keyed JSON object or rely on parsed key order.
- **Must** declare schemas as `const` literal rows plus `static func fields()`; **must never** use `static var` in content code.
- **Must** make catalogs write-once, get-only, scene-tree free and constructed with injected inputs; **must never** make a catalog a `Resource`, a `Node`, an autoload or a static singleton.
- **Must** expose arrays as read-only typed `Array[T]` stored once; **must never** return a packed array or a per-call `duplicate()` from a catalog.
- **Must** return `Array[BootError]` from loaders with domain-2 codes from `ContentFaults`; **must never** call `assert`, `push_error` or `quit` in a loader, and **must never** raise a content boot error from an owner's own `*_faults.gd`.
- **Must** list every new data path in its manifest and keep the preset `include_filter` covering `assets/data/*.json`; **must never** add a data file that loads only in the editor.
- **Must** persist `def_id` / `template_id` / `room_key` strings; **must never** persist, hash or replicate an item `type_id` or any position-derived id; **must never** reuse a retired id or key.
- **Must** append to a ledger, never edit or remove a pair; **must** bump `catalogue_version` in the same commit as a structural change.
- **Must** append new codes to the end of `ContentFaults` and its `NAMES`; **must never** renumber a code.
- **Must** keep `src/data/` free of imports from `src/sim/`; **must never** `preload` a sibling system's script.
- **Must** keep generators deterministic and commit their output; **must never** generate content at runtime.
- **Must** feed unit-test fixtures through `MemoryContentSource`; **must never** read `res://assets/data/` from a unit test, except the dedicated shipped-data integration test.

## Alternatives Considered

### Alternative 1: `Resource` (`.tres` / `.res`) catalogs with typed `@export` fields
- **Description**: Each def and template is a `Resource` subclass authored in the inspector and loaded with `ResourceLoader`; the export pipeline includes it automatically.
- **Pros**: no export-filter problem; inspector authoring and validation; typed fields; UID references to sprites.
- **Cons**: a `Resource` is mutable, cache-shared state, so immutability becomes a convention and two tests can alias one def; a `.tres` can embed scripts; text diffs carry UID noise and reordering; ids drift toward resource paths or `ResourceUID`; IDB-26 i, SL-2 and `block_as_resource_or_node` already exclude it; the offline lint and the converter would need the editor's resource loader instead of plain files.
- **Rejection Reason**: Immutability, offline tooling and diffable append-only ledgers are the requirements. What `Resource` gives for free (export inclusion) is solved by one filter and one smoke check.

### Alternative 2: Code-as-data (GDScript `const` dictionaries and arrays)
- **Description**: Content authored directly as constants in `.gd` files.
- **Pros**: no parser, no float problem, typed at parse time.
- **Cons**: designers edit code; offline tools cannot read or generate it without executing GDScript; nested constants are mutable containers unless wrapped; ledger diffs are noisy; every data change becomes a code change.
- **Rejection Reason**: Content must be editable and generatable without touching code. Code constants are still used for what is part of the program: schemas, invariants, fault codes and the Graph's type enums.

### Alternative 3: CSV or a custom binary format
- **Description**: Flat tables as CSV, or a tool-written binary, read with `FileAccess`.
- **Pros**: CSV is spreadsheet friendly; binary is compact.
- **Cons**: nested fields (module `ports[]`, template rooms, `verbs[]`) are awkward in CSV; binary is opaque to review and merge; neither has a self-describing structure or a stable line number for errors.
- **Rejection Reason**: Nested, reviewable, line-addressable data is the main case. JSON gives that, and § 1.4 removes its number-typing weakness.

### Alternative 4: Directory scan with `DirAccess` instead of manifests
- **Description**: Load every `.json` found under a domain directory.
- **Pros**: adding a file needs no manifest edit.
- **Cons**: listing order and listed names differ between the editor and an export; a stray or half-saved file loads silently; load order is not a function of committed data.
- **Rejection Reason**: LY-2 and IDB-4 forbid it for these reasons. A manifest costs one line per file, and `UNLISTED_DATA_FILE` catches forgotten entries.

### Alternative 5: Runtime representation as nested `Dictionary` / `Array` (read-only flagged)
- **Description**: Keep each parsed row as a `Dictionary`, call `make_read_only()` after load, and query by key.
- **Pros**: least code; no per-row classes.
- **Cons**: `make_read_only` is shallow, so nested arrays stay mutable; untyped access defeats static typing and catches no misspelt field; it tempts order dependence; lookups return `Variant`.
- **Rejection Reason**: AC-69 (zero allocation) and AC-85 (immutability) need typed write-once rows with read-only arrays (IDB-26). Flat tables use the lighter `ScalarTable` instead (§ 1.5).

### Alternative 6: A `Content` autoload singleton
- **Description**: One global catalog reachable from anywhere.
- **Pros**: simplest access; content is immutable, so sharing is safe.
- **Cons**: breaks ADR-0001's no-global rule; one process cannot hold two catalogs, which MIRROR comparison, replay and catalog tests need (TR-itemdb-002); hides each system's dependency on content.
- **Rejection Reason**: Injecting each module's catalog costs one `setup()` argument and keeps the dependency visible and testable.

## Consequences

### Positive
- One loader, one casting rule and one error domain for every static table, instead of fifteen local versions.
- The editor, headless tests, the offline lint and the export run the same code; the exported-build smoke check catches the remaining differences automatically.
- Ids cannot silently change meaning: ledgers are append-only and CI-enforced from the day the first outside build ships; Graph type ids are pinned by code.
- Catalogs are immutable by construction and many can coexist, so tests and MIRROR comparison need no global reset.
- Unit tests never touch the file system.

### Negative
- Every data file needs a manifest entry and every table a schema in code: boilerplate per new table.
- Nested-row tables cost one class per table; adding a field touches schema, class and loader.
- No hot reload: a data tweak needs a restart (accepted for determinism).
- JSON has no comments (only the stripped `"_note"` key), no integer type and no duplicate-key error, so three properties (literal shape, duplicate keys, the 2^53 boundary) are enforced by a raw-text CI lint rather than the parser.
- Write-once catalog fields are a lint-and-test convention; GDScript cannot enforce them.
- **Two boot-failure surfaces:** a content failure is a root-drawn text screen and a non-zero exit, while a kernel registration failure is kernel `FAULTED`. Diagnosis must check both.
- History gates need a base-revision checkout in CI, and registry drift needs a non-GDScript step.
- BOOT work runs on every start; its budget is unmeasured until CONTENT-PERF-1.

## Risks
- **The export drops, remaps or resource-loads the data files.** Mitigation: CONTENT-EXP-1 gates acceptance; `EXPORT_FILTER_MISSES_MANIFEST` and the exported-build smoke check run in CI; the `ResourceLoader` fallback behind `ContentSource` is pre-decided.
- **Case-only path mismatch passes on Windows, fails on the Deck.** Mitigation: `MANIFEST_PATH_CASE` and a Linux leg in CONTENT-EXP-1.
- **`JSON` number typing differs from the training-knowledge claim.** Mitigation: `IntField.read` accepts both types; CONTENT-EXP-1 c records the real behaviour; `INT_LITERAL_SHAPE` is the backstop.
- **A duplicate key in an object silently keeps the last value.** Mitigation: `JSON_DUP_KEY` raw-text lint.
- **Float rounding at a .5 boundary differs across platforms.** Mitigation: one multiply and `roundi` per scaled field, `FLOAT_ROUND_AMBIGUOUS`, and golden content hashes once ADR-0005 lands.
- **A caller mutates a catalog array.** Mitigation: read-only typed arrays (`append` fails); no packed arrays exposed; tests assert `is_read_only()` rather than rely on the release-build error.
- **Boot cost grows with content.** Mitigation: BOOT limited to unsafe-if-violated rules (§ 2.4); per-table budgets from CONTENT-PERF-1; CI rules never migrate to BOOT.
- **Schema and file drift** (a field added to the file but not the schema). Mitigation: closed schemas make it `CONTENT_FIELD_UNKNOWN` at boot.
- **A ledger is edited by hand or regenerated after shipping.** Mitigation: the CI append-only gate and the one-way `SHIPPED` transition.
- **CI tool cannot resolve `class_name` scripts.** Mitigation: `godot --headless --import` before the lint step.

## GDD Requirements Addressed

| GDD System | Requirement | How This ADR Addresses It |
|------------|-------------|--------------------------|
| item-and-module-database.md | TR-itemdb-001/002/003: static immutable `ItemCatalog` loaded once, injected, typed `RefCounted` defs, no static or autoload state | § 1.5 catalog pattern, § 1.8 injection |
| item-and-module-database.md | TR-itemdb-004/005/023: pure read-only queries, `defs_of_class` ascending `type_id`, no allocation | § 1.5 (sorted read-only arrays, `NO_DEF`); CONTENT-PERF-1 and § Validation identity checks |
| item-and-module-database.md | TR-itemdb-006/007/021/026: `def_id` persists, `type_id` in memory, `ids.lock.json` boot refusal, renames and retirement | § 3.1 to § 3.3 |
| item-and-module-database.md | TR-itemdb-008/009/010: integer-only JSON under `assets/data/items/`, boot order and closed failure list, port validation against LS-4 | § 1.1, § 1.4, § 1.7, § 2.3 |
| item-and-module-database.md | TR-itemdb-020: single `catalogue_version`, diagnostic `catalogue_hash` | § 3.4 |
| item-and-module-database.md | TR-itemdb-024: catalogue load creates exactly 25 Objects; load budget | § 1.5; CONTENT-PERF-1 (loader temporaries excluded) |
| life-support-graph.md | TR-lsg-030: type ids explicit, stable, unique-checked, append-only; Item DB then Layout then `graph.state` last | § 3.2 code-fixed tables, § 1.7 resolution |
| ship-section-and-layout-data-model.md | TR-section-003/029: templates immutable, loaded at start-up from a committed manifest in sorted order, int-cast and bound-checked, structural lint at load | § 1.2, § 1.4, § 2.1 BOOT tier, § 3.5 |
| derelict-authoring-handmade-kits.md | TR-derelict-002/003/004/007/008: four template files, no new schema, integers only, hashes change on edit | § 1.1, § 1.4, § 3.5 |
| derelict-authoring-handmade-kits.md | TR-derelict-013/016/023/024/025: BOOT rules incl. `PLACEMENT_DEF_UNKNOWN`, pure deterministic lint, tranches | § 2.1, § 2.2, § 2.4 |
| derelict-authoring-handmade-kits.md | TR-derelict-027: plan-grammar converter is deterministic and a prerequisite | § 2.6 |
| web-map-and-transit.md | TR-webmap-006: roster and legs from `assets/data/world/`, linted on load | § 1.1, § 1.2, § 1.7 stage 4 |
| save-load-and-versioned-format.md | TR-saveload-024: boot asserts tuning-knob invariants against the data file | § 2.5 |
| player-controller-input.md | TR-pc-015: tick-expressed knobs invalidated by a `TICK_RATE` change, detected from the unit column | § 1.4 tick-expressed values |
| interaction-system.md | TR-interact-030: tuning knobs as tick counts under `assets/data/interaction/`; forbidden stats and rewards | § 1.4 closed schemas, forbidden keys |
| suit-and-equipment.md | TR-suit-028: integers-only JSON versus `Resource` (ADR candidate 2) | § 1.1, Alternative 1 |
| attention-system.md | TR-attention-010: boot lint enforces survivable-dark and abortability invariants | § 2.1, § 2.4, § 2.5 |
| horror-entities.md | TR-horror-006: per-type tuning keyed by `type_id`, forbidden keys rejected | § 1.4 forbidden keys, § 3.2 |
| sanity-and-hallucination-rules.md | TR-sanity-024/025/026: lie targets data-driven and boot-checked, safe-range fixture, data files in the export | § 1.6, § 2.3, § 2.5 |
| hud-and-suit-layer.md | TR-hud-026: `hud_tuning` validated at load, no balance keys | § 1.4, § 1.5 `ScalarTable` |
| in-world-instruments-and-inspect-view.md | TR-instr-007/008/009/028: instrument records, build-time lints, zones derived from band names | § 2.1 CI tier, § 2.2 |
| chart-table-web-map-and-menus.md | TR-charttable-013: chart layout is authored data with a deterministic fallback | § 1.1, § 2.2 |
| adaptive-audio-and-tells.md | TR-audio-011/012: identity-table lint; levels and tables in `assets/data/audio/` | § 1.1, § 2.1 CI tier |
| failure-and-trail-system.md / emitter-state-and-light-exposure-query.md | IC-12: the Emitter reads Failure & Trail's glow table, which is static data | § 1.1 (Content data, still authored by Failure & Trail) |

**Loaded and checked by this pipeline; the rule itself is owned elsewhere.** These baseline rows cite ADR-0007, but the behaviour is decided by the named owner. This ADR supplies only the loading and the BOOT or CI check. `/architecture-review` should re-tag them to the owner, so that no story treats them as decided here.

| TR | Rule | Owner of the rule | What this pipeline contributes |
|---|---|---|---|
| TR-itemdb-015, TR-boarding-019 | `salvage_yield` deterministic, from the applied `SalvageRecord` | Item DB GDD (IDB-17), Boarding | none (pure query on the frozen catalog) |
| TR-itemdb-016 | salvage then reinstall preserves hp, cap and patch clock | Item DB GDD (IDB-18), ADR-0011 | none |
| TR-itemdb-017 | `repair_part_for` total over five `subject_kind`s | Item DB GDD (IDB-19/20) | BOOT `ITEM_BOOT_REPAIR` |
| TR-itemdb-018/019 | `mass_g`, `weight_class`, junk tell margin; no price/rarity/recipe | Item DB GDD (IDB-5/21/22/29) | schema ranges, forbidden keys (BOOT); `WEIGHT_CLASS_MARGIN_PM` and junk tell (CI) |
| TR-itemdb-022 | glowstick parameters data-driven; curve evaluated by the Emitter | Item DB GDD (IDB-12), ADR-0017 | `SCALED` field conversion; invariant `GLOWSTICK_RADIUS_TILES < THROW_RANGE_TILES` (BOOT) |
| TR-itemdb-027 | salvage is the only item faucet | Item DB GDD (IDB-29) | forbidden keys (BOOT) |
| TR-itemdb-028 | dialect is a skin field, all MVP defs `foundry` | Item DB GDD (IDB-23) | schema enum (BOOT) |
| TR-derelict-001 | Authoring registers no block, hook or runtime state | Derelict GDD (DA-1) | none (content only) |
| TR-derelict-014 | zero-part run survivable (`G0_ZERO_PART_SURVIVABLE`) | Derelict GDD (DA-25/42), headless test | CI ERROR |
| TR-derelict-015 | part ledger: 4 loose parts, 800 permille | Derelict GDD (DA-21/25) | CI ERROR |
| TR-derelict-017 | exactly one BEACON and one EMERGENCY lamp | Derelict GDD (DA-23/24) | BOOT (`BEACON_COUNT`, `SELF_POWERED_LAMP_COUNT`) |
| TR-derelict-020 | tile volume <= `TILES_WORLD_MVP` 600, authored nodes <= 120 | Derelict GDD (D.1, DA-32/36) | CI ERROR |
| TR-derelict-028 | kit manifest rows; projector picks atlas by `template_id` | Derelict GDD (DA-5/38), ADR-0034 | schema and manifest (BOOT) |
| TR-derelict-029 | authored data is truth only; palette sanity band | Derelict GDD (DA-34/37) | CI ERROR |
| TR-boarding-014, TR-boarding-025 | dock glow placement; traversal and vacuum budgets over S0 to S4 | Boarding GDD (BS-19/20) | CI ERROR |

## Performance Implications
- **CPU**: Boot only. Parse, cast and freeze run once per start on the main thread, before the kernel exists; nothing here runs in the tick. Lookups are an array index or a frozen-map lookup that allocates nothing (AC-69). Item catalogue load P99 <= 14.9 ms desktop / 30.0 ms Deck is the target, pending CONTENT-PERF-1; other tables get budgets from the same spike.
- **Memory**: Frozen rows only. The Item DB holds exactly 25 Objects (23 `ItemDef`, the `NO_DEF` sentinel, one `ItemCatalog`) once loader temporaries (`JSON`, `ContentRead`, `IntRead`, `FieldSpec`, `BootError`) are released, and returns to baseline when released. Raw text and parsed `Dictionary` trees are dropped after freeze.
- **Load Time**: Added to cold start. If CONTENT-PERF-1 shows a problem, stages 3 and 4 load behind a splash, never in the tick; no threaded load is planned, because it would add a synchronisation seam to a path that must be deterministic.
- **Network**: None. Content is not replicated; co-op peers load the same files and compare `catalogue_hash` / template hashes as a diagnostic.

## Migration Plan
No code exists. GDD and architecture edits needed (see GDD sync):
- `item-and-module-database.md`:
  - IDB-26: close the export-filter and `catalogue_hash` ADR candidates; adopt the `BootError` shape and the Item DB manifest.
  - IDB-30: the composition root acts, not kernel BOOTING; codes are domain 2.
  - IDB-27: alias behaviour is a pre-`SHIPPED` story.
- `ship-section-and-layout-data-model.md` LY-2 and LY-30: filter syntax; a template ledger; structural-lint codes are domain 2; "loaded in BOOTING" means start-up before the kernel.
- `life-support-graph.md`: the type-table baseline is the code enums; the `GRAPH_BOOT_TYPE_ID` code moves to domain 2 (`GRAPH_RESTORE_*` stays in domain 3).
- `sanity-and-hallucination-rules.md`: `SANITY_BOOT_LIE_TARGET_DATA` is domain 2.
- `suit-and-equipment.md`: ADR candidate 2 is closed (JSON).
- `failure-and-trail-system.md` and `emitter-state-and-light-exposure-query.md`: IC-12 wording.
- `architecture.md`:
  - § 8: static type tables load first; `graph.state` registers last.
  - Add CONTENT-EXP-1 to § Spikes that gate ADRs.
- `docs/architecture/tr-baseline.md`: rewording of TR-lsg-030 and TR-section-003 ("loaded in BOOTING"). Re-tagging of the owner-elsewhere rows above happens at `/architecture-review`.

## Validation Criteria
- CONTENT-EXP-1 passes on Windows and Linux/Deck: an exported `template_release` boots headless to `RUNNING` and its content hashes equal the editor's.
- A unit test per loader rule: each pipeline-generic code is raised by a `MemoryContentSource` fixture with exactly one defect; a defect-free fixture loads clean; a fixture with defects in two files of one stage reports both.
- Two `ItemCatalog` instances built in one process share no state.
- **Zero allocation is tested two ways**, because `OBJECT_COUNT` counts only Objects and misses arrays:
  - (a) `OBJECT_COUNT` is unchanged over 10,000 queries;
  - (b) two calls to each array-returning accessor return the same array (`is_same`), and `is_read_only()` is true.
- The 25-Object load check runs after every loader temporary is out of scope.
- A CI job fails on:
  - an unlisted `.json`;
  - a case mismatch;
  - an include-filter miss;
  - a raw-text shape or duplicate-key violation;
  - a non-append-only ledger change or a missing `catalogue_version` bump after `SHIPPED`;
  - a generated file that drifts from its generator;
  - any `DirAccess`, `FileAccess`, `JSON.parse_string` or `JSON.new` use for content outside `src/data/` (the lint is ADR-0010's).
- **After ADR-0005 is Accepted** (it depends on ADR-0005): golden `catalogue_hash` and per-template hash vectors are committed and change only with the encoding.

## Related
- Depends on: ADR-0001, ADR-0003 (both Proposed).
- Forward references: ADR-0005 (canonical encoding, `H64`), ADR-0006 (save header carries `catalogue_version`), ADR-0010 (lint tool, source layout, shared pure functions), ADR-0011 (Graph type enums), ADR-0013 (layout block), ADR-0016 (restore ordering), ADR-0030 (boot-failure surface styling).
- GDDs: `item-and-module-database.md` (IDB-3, 4, 26, 27, 30), `ship-section-and-layout-data-model.md` (LY-2, LY-30), `life-support-graph.md` (LS-2 to LS-5, ADR-LSG-08), `derelict-authoring-handmade-kits.md` (DA-4, DA-41, DA-43), `save-load-and-versioned-format.md`.
- `docs/architecture/architecture.md` § 8, § Required ADRs, § Spikes that gate ADRs, Open Question OQ-9.
- `docs/registry/architecture.yaml`: entries proposed by this ADR (state, interface, API decisions, forbidden patterns).

## Open Questions
- **OQ-A.** The composition root alone acts on content boot errors (content precedes the kernel), and domain 2 is not registered with the kernel at MVP. Revisit with ADR-0003 if the kernel GDD's boot-failure screen must be kernel-owned, or if a runtime path starts raising domain-2 codes.
- **OQ-B.** Whether the `section_templates` ledger holds one entry per `room_key` or one per template with a nested list (ADR-LAY-04 detail). This ADR requires the ledger and leaves the nesting to Layout's schema.

## Review Log
- **2026-10-28 godot-specialist (engine).** Idiomatic for 4.7.2; no removed or deprecated API.
  - Blocking, all applied:
    - B1: `const Array[FieldSpec]` does not compile; now `const` literal rows plus `static func fields()`.
    - B2: packed arrays are passed by reference and cannot be read-only; now typed read-only `Array[T]` or index accessors.
    - B3: `OBJECT_COUNT` misses arrays; identity and `is_read_only` checks added.
  - Minor, applied:
    - `set` renamed to `content`.
    - The `get_file_as_string` rationale is corrected.
    - Explicit BOM check.
    - `JSON_DUP_KEY` lint.
    - The 2^53 bound is now exclusive.
    - `MANIFEST_PATH_CASE` and a Linux leg.
    - LF `.gitattributes`.
    - `--import` before headless CI.
    - `addons/gdUnit4` excluded from export.
    - `roundi` half-away-from-zero noted.
    - `TYPE_BOOL` refused.
    - One `class_name` per file.
- **2026-10-28 technical-director (TD-ADR): CONCERNS.** All twelve findings applied:
  - 1: all content boot codes are domain 2, matching ADR-0003; owner domains keep runtime and restore faults.
  - 2: one flat code list, no sub-ranges.
  - 3: schema form as B1.
  - 4: § 3.1 and § 3.2 reworded; Graph tables pinned by code enums; item `type_id` rule narrowed.
  - 5: § 1.9 places shared code under `src/data/`, with no `src/sim/` imports.
  - 6: § 2.2 splits file-pure from history gates; comparison base defined.
  - 7: `VERSION_BUMP_MISSING` WARN pre-release; aliases deferred to a pre-`SHIPPED` story; `ScalarTable` for flat tables; catalogs injected without view facets; golden vectors after ADR-0005.
  - 8: owner-elsewhere sub-table added; count corrected to 65.
  - 9: `PLACEMENT_DEF_UNKNOWN` moved to BOOT, and the Derelict table tiered once.
  - 10: two failure surfaces noted in Negative; surface owner named; domain-2 registration dropped at MVP; collection policy stated.
  - 11: TR-section-003 added to the Migration Plan; registry entries drafted.
  - 12: as engine minor notes; Item DB uses a manifest.
  - The TD's pre-Accept conditions stand: ADR-0001 and ADR-0003 Accepted, and CONTENT-EXP-1 passed.
