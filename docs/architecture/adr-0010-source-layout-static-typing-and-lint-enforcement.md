# ADR-0010: Source Layout, Static Typing and Lint Enforcement

## Status
Proposed

## Date
2026-10-28

## Last Verified
2026-10-28 (against `docs/architecture/architecture.md` § System Layer Map, § Required ADRs, § Spikes, OQ-10, OQ-16, IC-10; `docs/registry/architecture.yaml`; ADR-0001, ADR-0002, ADR-0003 and ADR-0007 (all Proposed); `design/gdd/command-simulation-layer.md` CR-14 and its KRN acceptance criteria; `ship-body-and-interior-physics.md` BDY-3 and BDY-35; `ship-section-and-layout-data-model.md` LY-31; `docs/architecture/tr-baseline.md` and `requirements-traceability.md` rows citing ADR-0010; and `docs/engine-reference/godot/` `VERSION.md`, `breaking-changes.md`, `deprecated-apis.md`, `current-best-practices.md`). Eleven engine facts were probed on the installed 4.7.2 editor binary on 2026-10-28 (§ Engine Compatibility). Revised the same day against the engine review (E1 to E21), the TD-ADR review (T1 to T9) and the cross-ADR rulings TD-01, TD-02, TD-04, TD-06, TD-10 to TD-14, TD-15 to TD-18, TD-21, TD-23, TD-27, TD-30 and TD-33 (§ Review Log), and re-read against the now-written ADR-0004, ADR-0005, ADR-0006, ADR-0008 and ADR-0009 (all Proposed). No game code exists; nothing here has been run on a release export, and the pinned reference has no GDScript-tooling, export or Performance-monitor module, so every claim outside the probes is marked UNVERIFIED and bound to spike LINT-SPIKE-1.

## Decision Makers
Claude (author), running under modes.automation: autonomous (project.yaml); decisions logged for the user's review; the user has not yet reviewed this ADR.

## Summary
Twenty-six GDDs and three Proposed ADRs each name lints, "lint-enforced" rules, headless-test seams and zero-allocation claims, but no document says where code lives, which way imports may point, how an interface is written in GDScript, which tool enforces which rule, or how an allocation is measured. This ADR fixes one `src/` tree and an import-edge matrix, makes `@abstract` base classes ("ports") the interface mechanism for injected seams, promotes every GDScript warning to an error for `src/` and `tools/` through the engine's own `directory_rules`, and specifies one headless GDScript lint tool whose rules are each assigned to one of four enforcement tiers (engine parser, lexical scan, reflection, runtime counters and boot walks), together with a three-tier zero-allocation check, a pinned gdUnit4 6.2.2 lane layout, and the policy for the GDExtension escape hatch.

## Engine Compatibility

| Field | Value |
|-------|-------|
| **Engine** | Godot 4.7.2 (stable; mono editor installed locally, CI uses the standard build), GDScript only |
| **Domain** | Core / Scripting (GDScript typing and warnings, `@abstract`, script reflection, `Performance` monitors, project settings, test tooling) |
| **Knowledge Risk** | HIGH. `VERSION.md` marks 4.5, 4.6 and 4.7 HIGH; the rows that matter are 4.5 `@abstract` and variadic arguments, 4.7 typed-return inheritance on overrides, and 4.7 packed-array element writes skipping setters. The reference has no module for GDScript tooling, project settings, export or `Performance`, so those claims rest on probes (below) or training knowledge (marked UNVERIFIED). |
| **References Consulted** | `docs/engine-reference/godot/VERSION.md`, `breaking-changes.md` (4.6 to 4.7 GDScript rows; 4.5 additions), `deprecated-apis.md` (untyped `Array`/`Dictionary` row), `current-best-practices.md` (GDScript 4.5 and 4.7 sections; § Command Line, Tests and Parse Check, including the gdUnit4 6.2.2 / 4.7.2 observation of 2026-10-17). No `modules/*.md` file covers this domain. |
| **Post-Cutoff APIs Used** | `@abstract` classes and methods (4.5). `Dictionary[K, V]` typed dictionaries (4.4). Typed-return override inheritance (4.7, a constraint on how ports are implemented). `Script.is_abstract()` (believed 4.5, UNVERIFIED, not in the pinned reference; probed by LINT-SPIKE-1(a)). `debug/gdscript/warnings/directory_rules` (project setting; the release that added it is UNVERIFIED, its presence and behaviour on 4.7.2 are probed). |
| **Verification Required** | **LINT-SPIKE-1 (gates acceptance; runs beside the KRN-LAT-02 kernel skeleton, which is the first real code to lint):** (a) a `template_release` export on Windows and Linux/Deck with an `@abstract` port and an implementation boots, and a stub that omits an abstract method still fails at parse time in the editor; probe `Script.is_abstract()` and the behaviour of `can_instantiate()` on an `@abstract` port and on a static-only class such as `IntMath` (if `is_abstract()` exists, the `ARCH-05` "ports are `@abstract`" check and `load_all` use it, § 3.1, § 5.2); report whether gdUnit4 6.2.2 `mock()`/`spy()` can double an `@abstract` class (§ 8.1 assumes it cannot). (b) On the editor binary, `can_instantiate()` is false (or `load()` is null) for a script that fails because of a promoted warning, not only for an abstract-construction error; **P10 is re-observed with a project run, not only `--script`**; and the engine outcome is recorded for each of these fixtures (an unexpected pass moves the rule to the lexical tier): `var v: Vector2i = Vector2i(5, -3) / 2` under `INTEGER_DIVISION=2` and `NARROWING_CONVERSION=2` (expected silent, so `SIM-04` must catch it); an override with no `-> T` and no `return` (must fail on 4.7.2, § 3.3); an `@abstract` method with parameters under `UNUSED_PARAMETER=2` (e.g. `solid_in(rect: Rect2i, out: QueryBuffer) -> int`); a presentation script with a public `signal` emitted only from outside under `UNUSED_SIGNAL=2`; a `setup(clock: Clock)` that assigns `clock = clock` under `SHADOWED_VARIABLE=2`; a local named `position` in a `Node2D` script under `SHADOWED_VARIABLE_BASE_CLASS=2`; a `get(id)` method and a `class_name Logger` (§ 6.4 `ARCH-05`). Outcomes are written into § 1.2. (c) `Performance.OBJECT_COUNT` and `MEMORY_STATIC` availability in a release export (expected: `MEMORY_STATIC` is unavailable, which restricts tier CNT-02 to debug binaries); warnings are an analyzer feature compiled into debug-enabled binaries (editor, `template_debug`), so a `template_release` export is expected **not** to evaluate promoted warnings (UNVERIFIED): warning promotion is verified on the editor binary and optionally `template_debug` only, and the release export is not a warning gate. (d) the lexical scanner passes its fixtures, reports no false positive on the kernel skeleton, and runs over **every committed `.gd` file in the repository** (including `addons/gdUnit4/` and `.claude/scripts/`) without a crash, comparing its string-stripped output with a reload (§ 6.3); `.fixture` files are ignored by the editor; the shape of `Script.get_script_property_list()` for `Kernel` is recorded (category entry, `PROPERTY_USAGE_SCRIPT_VARIABLE` flag, inherited and `static` members; § 6.4 `KRN-01`). (e) wall time of load-all and of the whole lint run on the CI runner. (f) the fresh-clone `--import` crash (3 of 17 runs on 4.7.x per `current-best-practices.md`): the retry count (up to three) or a `.godot/` cache keyed on engine version, `project.godot` and `addons/` absorbs it (§ 8.4). (g) gdUnit4 6.2.2 on 4.7.2 from a committed `addons/gdUnit4/` (the add-on is **not yet in the repository**), including exit 101, the headless lane and the exact `commands.test` string. (h) whether `gdUnit4-action@v1` fails the job on exit 101. (i) the export pack scan (§ 4.3) from a scratch directory with `load_resource_pack(path, false)`. (j) the behaviour of the `IntMath` divisor guard (`b <= 0` returns 0 with a debug `assert`) in a headless debug run: whether the failed `assert` aborts the function and which log lines it prints (§ 5.4, § 8.6). **Spike S1 / SP-ENG-1 (not gating this ADR):** decides whether GDExtension rung 3 is ever entered (§ 9). |

**Probes run on 2026-10-28** with `Godot_v4.7.2-stable_mono_win64_console.exe --headless --path <scratch> --script`, scratch projects under `C:/Users/dasbl/AppData/Local/Temp/claude/adr-probes/0010/p1` to `p8` (outside the repository). All results are for the **editor binary**, not a release export.

| # | Probe | Result (probed 2026-10-28) |
|---|---|---|
| P1 | `ProjectSettings` property list for `debug/gdscript/warnings/` | Every warning from `UNTYPED_DECLARATION` to `GET_NODE_DEFAULT_WITHOUT_ONREADY` is a setting with levels 0 (ignore), 1 (warn), 2 (error). A `directory_rules` dictionary exists with default `{ "res://addons": 0 }`. Defaults: `untyped_declaration`, `inferred_declaration`, all four `unsafe_*`, `return_value_discarded`, `missing_await` = 0; `integer_division`, `narrowing_conversion` = 1; `inference_on_variant` = 2. |
| P2 | `directory_rules` value 2 | Engine error `decision < 0 \|\| decision >= WarningDirectoryRule::DECISION_MAX`: the only valid decisions are **0 (exclude) and 1 (apply)**. Severity is **global per warning**; a directory can only opt in or out. |
| P3 | `directory_rules = { "res://": 0, "res://src": 1 }` with `untyped_declaration` and `integer_division` at 2 | A violating script under `res://src/` and under `res://src/sub/` fails with `Parse Error ... (Warning treated as error.)` and `Failed to load script`; identical scripts under `res://lax`, `res://excl`, `res://other` load clean. A rule on `"res://"` itself works, and subdirectories inherit. A rule keyed on `"res://"` itself is accepted and is how everything outside `src/` is opted out (an invalid-value run, P2, left scratch scripts at the root subject to the global levels). `load()` still returns a non-null `GDScript` object after the error message. |
| P4 | `@abstract` | A subclass missing an abstract method fails: `Class "ImplMissing" must implement "PortBase.solid_in()" and other inherited abstract methods or be marked as "@abstract"`. `PortBase.new()` written in code fails: `Cannot construct abstract class "PortBase"`. A conforming subclass works and `is PortBase` is true. **Reflective `load("res://port.gd").new()` succeeds**, so the abstract check is a parse-time check on direct construction only. |
| P5 | Reflection on a loaded script | `get_script_method_list()` gives name, `args.size()`, `default_args.size()`, `return.type` and flags (static = 32, variadic = 16); `get_script_property_list()`, `get_script_constant_map()`, `get_script_signal_list()` work; `source_code` is readable in the editor binary; `ProjectSettings.get_global_class_list()` lists every `class_name`. `@warning_ignore_start` / `_restore` parse. |
| P6 | `Performance.OBJECT_COUNT` | +0 for 1,000 `Array[int]` appends, a 100,000-element `PackedInt32Array`, a 1,000-entry `Dictionary` and a lambda; +1 per `RefCounted.new()`. It counts Objects only. |
| P7 | `Performance.MEMORY_STATIC` | A 1,000,000-element `PackedInt32Array` raised it by 4,000,064 bytes; 10,000 retained small arrays raised it by 1,731,592 bytes; **100,000 transient arrays and strings freed within the loop raised it by 208 bytes** and `OBJECT_COUNT` by 0. It sees retained growth and is blind to transient allocation. |
| P8 | `is_same` and `is_read_only` | `is_same(getter(), getter())` is true for a stored array and false for two `duplicate()` results; `==` is true for both, so `==` cannot test identity. `make_read_only()` then `is_read_only()` works on a typed `Array[int]`. |
| P9 | Class cycles | Two scripts that name each other by `class_name` (`var b: CycB` / `var a: CycA`) load and instantiate. Cycles are a `preload` problem, not a `class_name` problem. |
| P10 | Warning visibility | A level-1 warning (`unused_variable`) in an applied directory printed **nothing** during `--import`, `--script` load or `--editor --quit` in headless mode. Only level-2 warnings are visible to a headless run (observed with `--script`/`--import`/`--editor --quit`; a project run was not probed, LINT-SPIKE-1(b); the conclusion to use level 2 holds either way). Whether `--import` itself surfaces a level-2 violation was not established (the probe project had none), so load-all, not import, is the gate. |
| P11 | Integer semantics | With `integer_division=2`, an unannotated `a / b` fails to load and `@warning_ignore("integer_division")` on the `return` statement lets it load. `-181/256 = 0`, `-362/256 = -1`, `-768/256 = -3`, `-7/2 = -3` (truncation toward zero); `-1 % 48 = -1`, `posmod(-1, 48) = 47`; `floor_div(-1, 48) = -1`, `floor_div(-48, 48) = -1`, `floor_div(95, 48) = 1`. The BDY-35 fixtures hold. |

Review confidence (engine review E20, recorded for the next reader): P1, P5, P6, P8 and P11 are high confidence; P2, P3, P4 (parse errors), P7 and P9 are good; the reflective `new()` bypass in P4 and the headless silence of level-1 warnings in P10 are the least certain. Nothing in the probes contradicts the pinned reference. The engine review itself was a desk review; a human or specialist rerun is still pending (§ Review Log).

## ADR Dependencies

| Field | Value |
|-------|-------|
| **Depends On** | ADR-0001 (composition root `src/game/game.gd`, `ProcessPriority`, injected facets, no autoload; Proposed), ADR-0002 (`Command.new` only in `src/kernel/`; named handles; Proposed), ADR-0003 (`CompletionToken` owner callables stay registered `Callable`s; H1 member allow-list; fault domains; Proposed), ADR-0007 (`src/data/`, `tools/ci/content_lint.gd`, § 1.9 shared pure functions, write-once catalogs; Proposed). |
| **Enables** | Every module ADR (0011 to 0031, 0034): each one names its directory, its ports and its lint pack against this ADR. The Foundation siblings **ADR-0004, ADR-0005, ADR-0006, ADR-0008 and ADR-0009** cite this ADR's paths and rule ids, so they carry ADR-0010 under *their* Depends On; this ADR lists them under Enables only (TD-27, no circular edge). Also ADR-0031 (dev-root gating, release exclusion) and ADR-0019 (`Input.*` reader allow-list, `input_surface.json`). |
| **Blocks** | The first code story of any epic (the tree and the project settings must exist first), the kernel skeleton (KRN-LAT-02), `/create-control-manifest` for the Foundation layer, and every story that cites a "lint-enforced" requirement. |
| **Ordering Note** | ADR-0004, ADR-0005, ADR-0006, ADR-0008 and ADR-0009 are written (Proposed) and cite this ADR; this ADR has been aligned to them (§ Review Log). **Forward references (not yet written, not blocking authorship):** ADR-0019, ADR-0031, ADR-0034 and the module ADRs. Acceptance requires ADR-0001, ADR-0002, ADR-0003 and ADR-0007 Accepted and LINT-SPIKE-1 passed. This ADR owns *where and how* a rule is enforced; the *content* of a rule belongs to the ADR or GDD that states it (the `IntMath` function list is ADR-0005's, the presentation exemption set is ADR-0009's, the pin key list is ADR-0008's). A rule is added to the lint tool by its owner in the same story that introduces the code it constrains (§ 6.6), subject to the MVP minimum of § 6.10. |

## Context

### Problem Statement
Three Proposed ADRs and 26 GDDs defer the same questions to this ADR: (1) the architecture names `src/sim/graph/`, `src/sim/emitter/`, `src/debug/`, `src/observer/` and `src/data/items/` while the Sanity GDD says `src/sanity/` (IC-10), and nothing names the composition root's neighbours, the home of pure helpers shared by Data and Core, or where a PROJECTOR script lives; (2) "cross-system references go through `class_name` and typed interface base classes" (architecture, dependency rule 2) without saying which mechanism, while ADR-0003 deliberately avoided `@abstract` for owner callables; (3) 37 requirements and at least 14 forbidden or "lint-enforced" rules in ADR-0001, ADR-0002, ADR-0003 and ADR-0007 assume a lint tool that does not exist and whose power nobody has stated, although GDScript has no public AST and several rules (field writers, writes outside a `WriteScope`, floats on a query path, process priorities set at run time) cannot be decided by a regex (OQ-16); (4) "zero heap allocations" (TR-lsg-025, TR-cmd-032) cannot be measured in GDScript, and `Performance.OBJECT_COUNT` counts only Objects (probe P6); (5) the BDY-3 rule to promote typing warnings to errors "for the Body and Layout directories" rested on an unverified directory scope; (6) the gdUnit4 pin, the headless and windowed lanes, and the test-seam export policy exist only as a scaffold (commit `d55b50b`) whose add-on is not committed; (7) OQ-10 asks for the rule under which GDScript hot paths may be replaced by native code.

Without one decision, each module author invents a layout, a lint script and a definition of "allocation-free", and the first integration of two systems finds the differences.

### Constraints
- **GDScript only, Godot 4.7.2, no autoload, no static singleton** (registry `kernel_autoload_or_global`). C# is not the project's language; the installed mono editor is a convenience and CI uses the standard build.
- **Simulation is integer-only, scene-tree free and deterministic** (kernel H9 and H10; architecture Principles 1 and 4). Any rule here must serve that, not just style.
- **No public GDScript AST or tokenizer exists** (UNVERIFIED beyond the reference; probe P5 found only script reflection). `gdtoolkit` 4.5.0 (Oct 2025) does not document `@abstract`, variadic or 4.6/4.7 grammar support (checked on PyPI 2026-10-28), so no blocking rule may depend on it.
- **Severity is global per warning; directories only opt in or out** (probe P2).
- **Only level-2 warnings are visible to a headless run** (probe P10); a level-1 warning is invisible to CI.
- **ADR-0003 §1 stands**: kernel-invoked owner callables are registered `Callable`s ending `return ctx.complete()`; nothing in this ADR reintroduces a `BlockOwner` base class.
- **Unit tests do no file I/O and run inside a SceneTree** (coding standards; gdUnit4), so "no SceneTree present" cannot be arranged and tree-freedom is a lint plus an orphan count (kernel GDD KRN acceptance criterion).
- **One programmer, LLM-assisted**: rules that need a parser the project must write and maintain are expensive; rules the engine already enforces are free.
- **Pre-release**: no saves, no shipped ids; the cheapest time to fix a directory name is now.

### Requirements
- One source tree, named for every module in the architecture's ownership tables, with a mechanical test for "is this file in the right place".
- An import-edge matrix that is checkable, so "simulation never reads presentation" and "no `preload` of a sibling system's script" fail a build instead of a review.
- One interface mechanism for seams, usable by test doubles, with parse-time errors for a missing method.
- Strict typing and integer discipline that the engine itself enforces on every script the game runs.
- One lint tool, one finding format, one exit-code table, a statement per rule of what it can and cannot see, and committed bad/good fixtures per rule.
- Allocation claims restated as checks that exist (OQ-16).
- A pinned test framework, two named lanes (headless, windowed), and a CI order.
- A written policy for native code that cannot be entered by accident.

## Decision

Nine numbered sub-decisions, each reviewable alone: **§ 1 source tree, § 2 import edges, § 3 interfaces (ports), § 4 scene-tree-free simulation and test seams, § 5 static typing and warnings, § 6 the lint tool, § 7 verifying allocation claims, § 8 test framework, lanes and CI order, § 9 GDExtension escape hatch.**

### Architecture Diagram

```
res://
  src/                              (project.godot: debug/gdscript/warnings/directory_rules include)
    shared/      L0  pure: scene-tree-free, I/O-free, no global state; IntMath, det/ (ADR-0005), shared pure functions (imports nothing)
    kernel/      L1  kernel, TickDriver, ProcessPriority, StateHasher, facets/ (presentation-visible API), ports/, testing/   (no Node)
    data/        L1  ContentLoader, schemas, catalogs, ContentFaults                  (shared only)
    save/        L1  Save/Load service, RefCounted and tree-free (FileAccess, Thread allowed; WallClock port)   (shared, kernel)
    sim/<12>/    L2  graph layout body emitter controller interaction suit assembly web
                     attention horror sanity; each: views/ ports/ testing/ + internals (no Node)
    presentation/<10>/ L3  pipeline perception lighting trail hud instruments chart audio
                     controller projector; reads sim/<x>/views and ports, kernel/facets
    game/        L4  composition root game.gd, KernelHost, SaveHost, MonotonicClock, SystemWallClock, boot walks (only place that sees all)
    debug/ observer/  dev-only, reached only by the guarded load() in game/
  tools/
    lint/        lint_main.gd, core/ (scanner, rules, report), <pack>_lint.gd, config/
    ci/          content_lint.gd and rule packs (ADR-0007), load_all.gd, lane_report.gd
  tests/
    unit/<system>/  (+ lint/ thin wrappers)  unit/lint/ (rule self-tests)  integration/<system>/
    render/ (windowed lane only)  fixtures/<system>/  fixtures/lint/<rule>/  support/
  addons/gdUnit4/   6.2.2, committed (directory_rules exclude)
  native/           rung 3 only: C++ source and .gdextension (empty until LINT/S1 evidence)

Enforcement tiers:   E1 engine parser      (warnings = errors, @abstract, load-all)
                     E2 lexical scan       (tools/lint: tokens, counts, import graph, tscn, project.godot)
                     E3 reflection         (tools/lint + gdUnit4: signatures, members, base types)
                     E4 runtime/boot walks (gdUnit4 integration: counters, priority walk, identity)
CI order: --import (retry up to 3, or cached .godot) -> load-all (E1) -> lint CLI (E2,E3) -> gdUnit4 headless (E3,E4) ; windowed lane and nightly benchmark separate
```

### 1. Source tree and module classes

**1.1 The tree.** Every GDScript file the game runs is under `src/` in exactly one of the directories below. The architecture's ownership tables are the module list; this section only adds names the tables left open.

| Directory | Layer | Contains | Node-derived classes? |
|---|---|---|---|
| `src/shared/` | L0 | Code that imports no other `src/` directory and is **scene-tree-free, I/O-free and has no global state** (it is not "engine-free": RefCounted engine value types such as `HashingContext`, `PackedByteArray` and `Marshalls` are allowed only in the files § 4.1 names). Contents: `IntMath` (`src/shared/int_math.gd`; the function list and guard are ADR-0005's, § 5.4 keeps only the path and the single-site rule); `src/shared/det/` (owned by ADR-0005: `DetRng`, `DetRngFloat` (`det_rng_float.gd`), `Ihash`, `Hex64`, `Hash64` (`hash64.gd`), `CanonicalEncoder` (`canonical_encoder.gd`), `sorted_keys.gd`); and the pure functions that both `src/data/` loaders and a sim module need (the Graph's `connectable` and `cap_seam`, the Derelict BOOT checks), under `src/shared/<owner>/` (ADR-0007 § 1.9). The owning system owns each file **by review (CODEOWNERS-style entry in `tools/lint/config/owners.json`), not by import.** | no |
| `src/kernel/` | L1 | Kernel, `TickDriver`, `ProcessPriority`, registries, `SimViews`, `WriteScope`, contexts, descriptors, `Command`, `StateHasher` (`src/kernel/state_hasher.gd`, ADR-0005) (ADR-0001 to 0005). Subdirectories: **`facets/` = the presentation-visible kernel API** (§ 1.2), `ports/` (abstract interfaces the kernel needs injected, e.g. `Clock`), `testing/` (§ 4.3). | no |
| `src/data/` | L1 | `ContentLoader`, `ContentSource` and its two implementations, schemas, catalogs, `ContentFaults`, per-domain subfolders such as `src/data/items/` (ADR-0007). | no |
| `src/save/` | L1 | Save/Load service, **all `RefCounted` with no tree access** (ADR-0006): `SaveService`, `WriteJob`, `WriteWorker`, `PreCheck`, `SaveCodec`, `SlotEpoch`, `RegistryView`, `BuildConstants`, and the `WallClock` port (`src/save/wall_clock.gd`, § 4.5). May use `FileAccess`, `DirAccess`, `Thread`, `JSON` and, in the named files of `IO-01`, `HashingContext`. The per-frame poll and the window-close interception are **not** here: they belong to `SaveHost` in `src/game/`. | no |
| `src/sim/<system>/` | L2 | One directory per simulation module: `graph`, `layout`, `body`, `emitter`, `controller`, `interaction`, `suit`, `assembly`, `web`, `attention`, `horror`, `sanity` (IC-10: `src/sim/sanity/`, never `src/sanity/`). Each may hold `views/` (read-view facades and value types that presentation may name), `ports/` (abstract interfaces other modules may name), `testing/` (test seams) and internals (blocks, mutators, hooks, `*_faults.gd`). | no |
| `src/presentation/<system>/` | L3 | `pipeline`, `perception`, `lighting`, `trail`, `hud`, `instruments`, `chart`, `audio`, `controller` (Input Adapter and Avatar Presenter), `projector` (Layout PROJECTOR and `body.projector`). | yes |
| `src/game/` | L4 | The composition root `game.gd` (ADR-0001); two thin host Nodes beside it, **`KernelHost`** (`src/game/kernel_host.gd`; the only `Node` that drives the tick) and **`SaveHost`** (`src/game/save_host.gd`; `process_mode = PROCESS_MODE_ALWAYS`, `process_priority = ProcessPriority.HOST_SERVICE`, calls `SaveService.poll_frame()` from `_process`, owns the window-close interception, holds no `FileAccess` or `Thread`; ADR-0006); `MonotonicClock` (`monotonic_clock.gd`, the production `Clock`) and `SystemWallClock` (`system_wall_clock.gd`, the production `WallClock`); the dev-build boot walks (§ 6.5); the guarded dev loaders. The one place allowed to name every layer. | yes |
| `src/debug/`, `src/observer/` | dev | Debug Overlay and observer log (ADR-0031). Reached only by the guarded runtime `load()` in `src/game/` after `ResourceLoader.exists()` (TR-debug-037); their root script declares no `class_name`. | yes |

**1.2 Placement decisions that were open.**
- **The PROJECTOR scripts live in `src/presentation/projector/`**, not in `src/sim/layout/` or `src/sim/body/`. This makes `src/sim/**` a tree-free directory with no per-file exception, so BDY-3's "the PROJECTOR script is the only Body script allowed to name `Node2D`, `Camera2D` or `Viewport`" becomes "no script under `src/sim/body/` names them". ADR-0034 still owns the scene shape.
- **`KernelHost` and `SaveHost` live in `src/game/`**, so `src/kernel/` and `src/save/` carry no `Node` and the kernel GDD's lint ("no block or kernel source references `Node`, `SceneTree`, `get_tree` or `Engine.get_main_loop`") and `SIM-01` have no exception (ADR-0001 chose `src/game/kernel_host.gd`; OQ-C is closed). `SaveHost` is a host adapter like `KernelHost`: it drives the RefCounted Save/Load service and uses the named `ProcessPriority.HOST_SERVICE` constant (-40, between `TICK_DRIVER` -50 and `PRESENTATION` 0, ADR-0001), never a numeric literal (`KRN-03`).
- **Presentation-visible kernel API (`src/kernel/facets/`).** These and only these kernel classes may be named from `src/presentation/`, `src/debug/` and `src/observer/`: `PresentationClock`, `SurfaceHandle`, `ControlHandle`, `ModuleFacet`, `BlockRegistrar`, `ChangeBatch`, `NotifyContext`, `CompletionToken`, `NotifyEvent`, `DirtyAccumulator`, `KeyBuf`, `NotifyFacet`, `SubscriptionHandle`, `SubscriberClass`, `DirtyKey`, `PresentationFacet`, `PresentationViews`. The list is data in `tools/lint/config/edges.json`. A `facets/` file may itself extend or use kernel-core classes (such as `CallContext`); presentation names only the `facets/` class, never the core class behind it. `NotifyFacet` may be named only by `src/game/` and by `PresentationFacet`: the composition root issues it and binds the subscriber class at issue, and presentation holds only the class-bound `PresentationFacet` (ADR-0004 § 2, ADR-0009 § 1). `KeyBuf` lives at `src/kernel/facets/key_buf.gd` so `DirtyAccumulator` has no kernel-core edge. ADR-0003, ADR-0004 and ADR-0009 cite these paths. There is no `KernelStateView`: the kernel state is `PresentationClock.state` and the restore marker is `PresentationClock.load_seq` (ADR-0001).
- **Shared pure helpers live in `src/shared/`**, importable by `src/data/` and every layer, because `src/data/` may not import `src/sim/` and loaders need the Graph's `connectable`. `IntMath` is the one place that divides integers (§ 5.4).
- **Scenes (`.tscn`) are allowed only** under `src/presentation/`, `src/game/`, `src/debug/`, `src/observer/`; scripts attached to a scene must be in the same set. No `.tscn` sets `process_priority` (ADR-0001).
- **Tests are not under `src/`** (`src/CLAUDE.md`): `tests/unit/<system>/`, `tests/integration/<system>/`, `tests/render/`, `tests/fixtures/`, `tests/support/`. Fakes and doubles (`FakeClock`, `FakeCollide`, `WorkCounter` doubles, stubs) live in `tests/support/`, never in `src/`.
- **Typing conventions the warnings force** (`UNUSED_PARAMETER`, `UNTYPED_DECLARATION`, `INFERRED_DECLARATION`, `SHADOWED_VARIABLE*`, `NATIVE_METHOD_OVERRIDE` and the rest at 2):
  - An unused parameter is `_`-prefixed (this includes parameters of port implementations and doubles); `for` iterators are typed (`for i: int in n`; an untyped iterator is a load error, P3). The snippets in § Key Interfaces follow this.
  - **`:=` is banned in `src/` and `tools/`** (`INFERRED_DECLARATION=2` makes every inferred declaration an error); every `var` and `const` has an explicit type. **Lambdas are fully typed**: `func(x: int) -> int:`, parameters and return.
  - **Injection pattern.** Member fields are `_`-prefixed and the `setup()` parameter is the bare name: `var _clock: Clock` and `func setup(clock: Clock) -> void: _clock = clock`. `func setup(clock: Clock): clock = clock` or `self.clock = clock` is a `SHADOWED_VARIABLE` load error when the parameter and the member share a name.
  - **No local, parameter or member reuses a Node (or other native base-class) property name** (`position`, `scale`, `size`, `name`, `owner`, `visible`, `modulate`, `rotation`, `script`, ...): that is `SHADOWED_VARIABLE_BASE_CLASS` in presentation code and, for a field named `script`, in any class. `ARCH-05` checks it by reflection (§ 6.4).
  - **Accessors are named `get_<noun>` or `lookup`, never `get`, `set`, `free`, `call`, `callv`, `connect`, `tr`, `to_string`, `has_method`, `notification` or `get_class`**: defining one in a catalog, view or kernel class is `NATIVE_METHOD_OVERRIDE`. A `class_name` may not equal a native class (the engine gained `Logger` in 4.5 and `AccessibilityServer` in 4.7; `DiagnosticsSink` and `ObserverSink` are near that space): `ARCH-05` checks it live against `ClassDB` (§ 6.4).
  - **A `const` is never initialised from a constructor of a non-builtin class** (`const X = Foo.new()` does not compile; the engine already enforces it). Typed constant arrays (`const NAMES: Array[StringName] = [...]`) and shifts (`(DOMAIN_ID << 16) | 1`) are valid constant expressions.
  - `EMPTY_FILE` and `ENUM_VARIABLE_WITHOUT_DEFAULT` fall under "every other warning" at 2: an enum-typed variable has an explicit default and no script is empty. `UNUSED_SIGNAL=2` applies to presentation signals declared for an outside emitter; UNVERIFIED whether an `@abstract` method's parameters trigger `UNUSED_PARAMETER`, and whether `UNUSED_SIGNAL` fires for a signal emitted only externally. Both are LINT-SPIKE-1(b) fixtures with the outcome to be recorded here; if either fires, the convention is `_`-prefixed abstract parameters and an `@warning_ignore("unused_signal")` entry on `warning_ignore.json` (§ 5.1 `PRJ-04`).
- **Naming.** One `class_name` per file; the file name is the snake_case of the class name; `*_faults.gd` for fault domains (ADR-0003); abstract ports are named for the role (`Clock`, `SolidQuery`, `OccluderSource`), not `IClock`. A script referenced by another script has a `class_name`. Accessors follow the `get_<noun>`/`lookup` convention above.

**1.3 Tools.** Headless GDScript tools live under `tools/`: `tools/lint/` (this ADR), `tools/ci/` (ADR-0007's `content_lint.gd`, the item-database and Derelict content rule packs, `load_all.gd`, `lane_report.gd`), `tools/asset-pipeline/` (generators, ADR-0007 § 2.6). A tool enumerating `src/`, `tests/` or `tools/` is **not** the forbidden `content_directory_scan`, which bans *game code* enumerating `res://assets/data`.

### 2. Dependency direction and import edges

**2.1 What an edge is.** The lint tool builds a module-level graph from a class map (class name to defining file) and, per file, every `class_name` token that survives comment and string stripping, plus every `preload(`/`load(`/`ResourceLoader` target. A file in module A that names a class defined in module B is an A to B edge. **The class map is built twice and the two must agree.** `ProjectSettings.get_global_class_list()` reads `.godot/global_script_class_cache.cfg`, which is refreshed only by an editor scan or `--import`, so a developer who adds a `class_name` and runs the CLI without importing would get a stale map and a false negative. The tool therefore also builds the map from a lexical `class_name`/`extends` scan of `src/`, `tests/` and `tools/`, and exits 2 (not a silent pass) when the two disagree; `--import` stays first in the CI order. (Tier E2 with an E3 class map; a class named only inside a comment or string is not an edge; a class reached only through a `Variant` is not seen, which the engine tier covers because `UNSAFE_*` is an error.)

**2.2 The matrix.** "Y" means allowed; blank means a finding (`ARCH-02`).

| from \ to | shared | kernel (core) | kernel/facets, kernel/ports | data | save | sim/X views, ports | sim/X internals | presentation | game | debug, observer |
|---|---|---|---|---|---|---|---|---|---|---|
| shared | Y | | | | | | | | | |
| kernel | Y | Y | Y | | | | | | | |
| data | Y | | | Y | | | | | | |
| save | Y | Y | Y | | Y | Y (views, ports only) | | | | |
| sim/X | Y | Y | Y | Y | | Y | own module and other sim modules | | | |
| presentation/X | Y | | Y | Y | | Y | | Y | | |
| game | Y | Y | Y | Y | Y | Y | Y | Y | Y | guarded load only |
| debug, observer | Y | | Y | Y | | Y | | Y | | Y |
| tests | Y | Y | Y | Y | Y | Y | Y | Y | Y (integration tests boot the composition root) | |

Reading the rules the architecture states in prose:
1. **Simulation to simulation, any direction, module granularity** (architecture rule 1). The matrix does not police which sim module calls which; the kernel's step-hook topo-sort polices acyclicity of hooks. A call into another module is by `class_name`, never `preload`.
2. **No compile-time cycles of `preload`** (architecture rule 2): `ARCH-03` bans `preload`, `load` and `ResourceLoader` of any `.gd` path anywhere in `src/` (use `class_name`). Probe P9 shows `class_name` cycles load, so cycles are not the risk this rule polices; hidden coupling is, and the matrix polices that.
3. **Foundation host services read Feature state only through the kernel** (rule 3): `save` has no edge to `sim/X` internals and never imports a module. Save/Load obtains owner read views through the kernel registry, and may *type* those variables with the published `sim/<x>/views/**` and `sim/<x>/ports/**` classes; naming a published view class is not importing the module. **`save` gets no `data` import** (the `save` to `data` cell stays blank): it may type variables with ADR-0007's published value types (the catalog and ledger value classes listed as `published_types` in `edges.json`), by the same exemption that applies to sim views, and the composition root injects the loaded tables into `SaveService.setup()`. `RegistryView` and `BuildConstants` are `src/save/` classes. Every other `save` to `sim` or `data` reference is an `ARCH-02` finding.
4. **Simulation never reads presentation** (rule 4): no sim edge to `presentation`, `game`, `debug`, `observer`. Presentation reaches the simulation only through `sim/<x>/views/**` and `sim/<x>/ports/**` classes, and reaches the kernel only through the `kernel/facets/**` list of § 1.2. It never names the **tick-phase deny list** `Kernel`, `SimViews`, `WriteScope`, `Command`, `BlockDescriptor`, `CallContext`, `ValidateContext`, `ApplyContext`, `HookContext`, `SnapshotContext` (ADR-0003 risk "a facet leak"; `ARCH-08`), and `KernelHost`, `TickDriver` (rule 7). The glob `*Context` is **not** used: presenter handlers legitimately take `NotifyContext` and return `CompletionToken` (ADR-0003, ADR-0004, ADR-0009), and both live in `kernel/facets/`. `NotifyFacet` is named only by `src/game/` and `PresentationFacet` (§ 1.2).
5. **`data` imports `shared` only** (ADR-0007 constraint): no `sim`, no `kernel`. The kernel tick rate reaches the loader as a constructor argument.
6. **Nothing imports `game`, `debug` or `observer`** except through the guarded load in `game/`.
7. **`Kernel`, `KernelHost`, `TickDriver` are named only in `kernel/` and `game/`** (registry `kernel_autoload_or_global`; `ARCH-08`). **`Command.new(` appears only in `src/kernel/`** (registry `command_constructed_outside_kernel`; `KRN-02`).

**2.3 Module and directory list is data.** The set of module directories, the matrix and the facet/port/view sub-directory names are committed in `tools/lint/config/modules.json` and `tools/lint/config/edges.json`; adding a module is a reviewed edit to those files in the same change as the first file (`ARCH-01`).

### 3. Interface mechanism: `@abstract` ports

**3.1 The choice.** A seam that is injected and that tests must double is an **`@abstract` base class, a "port"**, living in the `ports/` directory of the lowest layer that every consumer of the contract may import (the lowest layer that every consumer may import: `src/kernel/ports/` for `Clock`, which `src/sim`, `src/presentation` and `src/game` all reach; `src/sim/layout/ports/` for `OccluderSource`). Implementations `extends` the port; test doubles `extends` the port. A port is `RefCounted`, holds only `@abstract` methods, constants and (rarely) a documented non-abstract default; it holds no fields. The consumer types its `setup()` parameter as the port. A port with a single consumer may sit beside it (`src/save/wall_clock.gd`, the `WallClock` port of § 4.5); such files are listed as `port_files` in `modules.json` so `ARCH-05` still checks them.

Probed on 4.7.2 (P4): a subclass that omits an abstract method fails at parse time, which is the strongest guarantee available without a custom tool; direct construction of the port fails at parse time. The reflective `GDScript.new()` bypass is accepted: no game code constructs through reflection. If LINT-SPIKE-1(a) finds `Script.is_abstract()` (believed 4.5, UNVERIFIED), the E3 tier asserts it for every script under a `ports/` directory and for every `port_files` entry, closing the `ARCH-05` "ports are `@abstract`" check at the engine level instead of lexically.

**3.2 What it does not replace.**
- **ADR-0003 §1 stands.** Kernel-invoked owner callables (`validate`, `apply`, `hook`, `snapshot`, `restore`, ...) remain registered `Callable`s ending `return ctx.complete()`. That choice rested on many callables per owner, no inheritance coupling and stubs that register plain callables, not only on `@abstract` being unexercised. There is no `BlockOwner` port.
- **Ports are for injected collaborators**, not for owner callables and not for block contents: `Clock`, `ContentSource`, `SolidQuery`, `OccluderSource`, `WorkCounter`, `DiagnosticsSink`, `ObserverSink`, `WallClock`, and the stub points of TR-pc-045 and TR-charttable-038.
- **The duck-typed alternative is kept for one job**: `ContractCheck` (tier E3) compares a double that deliberately does *not* extend a port (a recorded stub for a sibling GDD system) against the real class by reflection (§ 6.4).

**3.3 The 4.7 interaction.** An override that **omits `-> T`** inherits the base method's return type in 4.7 and must then `return` on every path (breaking-changes, 4.6 to 4.7; the migration fix is `return null`). Implementations restate the full signature, including `-> T`, so the signature is visible and `REFLECT-SIG` can compare it; a missing `return` in a method that declares `-> int` was already a parse error before 4.7, so the new behaviour concerns the omitted annotation only. A LINT-SPIKE-1(b) fixture, an override with neither `-> T` nor `return`, must fail on 4.7.2. `REFLECT-SIG` (tier E3) compares, per entry of the implementation's and the port's `get_script_method_list()`: the name; each `args[i].type`, `.class_name` and `.hint_string`; the default-argument count; `return.type`, `return.class_name` and `return.hint_string`; and the flags word masked to static (32) and vararg (16). The type id alone is not enough: `Array[int]` and `Array[String]` share `TYPE_ARRAY` and differ only in `hint_string`, and an object type is told apart only by `class_name`. The engine's own override check is weaker than "identical".

**3.4 Fallback.** If LINT-SPIKE-1(a) shows `@abstract` does not survive a release export, ports become plain base classes whose methods call `push_error("ABSTRACT_CALLED")` and return a safe default, and `REFLECT-ABSTRACT` (tier E3, test-time) asserts every port implementation overrides every method named in the port. Call sites do not change. Cost of ports in a hot loop (one extra method dispatch) is measured by S1 (§ 9); ports sit at coarse seams (once per actor per tick, not per tile).

**3.5 Doubles.** Port doubles in `tests/support/` are **hand-written classes that `extends` the port**, not gdUnit4 `mock()`/`spy()` objects, unless LINT-SPIKE-1(a) proves `mock()` can double an `@abstract` class. `tests/` is outside the warning scope (OQ-B), so a double with a wrong signature is caught by the engine override check only; `REFLECT-SIG` therefore also runs over the doubles in `tests/support/` that extend a port.

### 4. Scene-tree-free simulation and test seams

**4.1 What "scene-tree free" means mechanically.** In `src/shared/`, `src/kernel/`, `src/data/`, `src/save/` and `src/sim/**` (all of L0 to L2): no class extends `Node`, `Resource` (blocks and catalogs are `RefCounted`), or any engine node; no token from the tree list (`SIM-01`: `Node`, `Node2D`, `Node3D`, `CanvasItem`, `CanvasLayer`, `Control`, `SceneTree`, `Viewport`, `SubViewport`, `Window`, `Camera2D`, `PackedScene`, `get_tree`, `get_node`, `get_node_or_null`, `find_child`, `add_child`, `queue_free`, `$`, `@onready`, `@export`, `_ready`, `_enter_tree`, `_process`, `_input`, `_unhandled_input`, `Engine.get_main_loop`, `Tween`, `Timer`, `RenderingServer`, `PhysicsServer2D`, `AudioServer`, `DisplayServer`, `Input`, `InputMap`). `SIM-01` carries no waiver. `Thread`, `FileAccess`, `DirAccess` and `HashingContext` are not in the list; their use is bounded by `IO-01` and `IO-02`: `HashingContext` is allowed only in `src/save/`, `src/data/`, `src/shared/det/hash64.gd` and `src/kernel/state_hasher.gd` (named files, not whole layers, TD-13); `Thread` only in `src/save/`. `KernelHost` and `SaveHost` are Nodes and live in `src/game/` (§ 1.1), so none of these directories needs an exception.

**4.2 Headless harness.** The simulation is constructed by tests exactly as the composition root does: a `Kernel` with an injected `KernelProfile` (never `OS.is_debug_build()`, TR-cmd-033), an injected `Clock`, injected catalogs from `MemoryContentSource`, and module `setup()` calls with `FakeX` doubles from `tests/support/`. `Kernel.step()` advances synchronously. Two kernels coexist in one process (TR-cmd-052): that is a test, `tests/unit/kernel/lifecycle/`, asserting independent `T`, queues and logs; a block bound to a second kernel fails boot with `BLOCK_ALREADY_BOUND`; `Kernel.dispose()` is idempotent and leaves `OBJECT_COUNT` and the orphan-node count where they started (tier E4, `CNT-01`). **Leak-freedom is designed, not assumed.** GDScript `RefCounted` objects are reference counted, so any cycle (Kernel to block to facet to Kernel, or a lambda stored on an object that captures `self`) is never freed and `OBJECT_COUNT` stays high. Facets and contexts that need the kernel hold it through `weakref()` or are cleared in `dispose()`; `dispose()` empties every registry, queue and subscriber list and nulls every back-reference; registered owner callables are `Callable(obj, &"method")` or bound methods and **never stored capturing lambdas** (a `Callable(obj, ...)` holds only an object id, a lambda captures strongly). A `CNT-01` fixture deliberately creates a cycle and expects the leak to be detected. `static var` is banned in `src/` (`ARCH-04`) because it is the only way two kernels could share state without the lint noticing it as an edge.

**4.3 Test seams in production classes.** `TestIngress.submit_raw`, `debug_set_counters`, the stubbed diagnostics timer, `set_knob`, the `WorkCounter` double hook and the like (TR-cmd-050, TR-emitter-032) live **only** in classes under `src/<module>/testing/` named `Test*`. They are importable only from `tests/` and other `testing/` directories (`ARCH-09`); the release export preset's `exclude_filter` gains the single glob `src/*/testing/*` (the filter list is ADR-0007's `content_export` entry; this ADR adds the one glob; Godot's `exclude_filter` glob lets `*` match `/` too, so it already covers `src/sim/layout/testing/x.gd` and a second `src/*/*/testing/*` would be redundant; the directory name stays exact, so a legitimate path such as `src/data/testing_notes/` is not excluded). The export check is two-sided: a lexical rule that no non-`testing/` file names a `testing/` class, and an exported-pack file-list scan that finds no `testing/` path. The scan runs **from a scratch directory holding only a minimal `project.godot`**: it calls `ProjectSettings.load_resource_pack(path, false)` and lists `res://src/**` with `DirAccess`; run inside the project, a pack mounted with `replace_files` merges into the live `res://` and every source file appears present (UNVERIFIED, LINT-SPIKE-1(i), added to the CONTENT-EXP-1 harness).

**4.4 No conditional compilation.** GDScript has no preprocessor. "Compiled in solely under the EM_WORK_COUNTER test flag" (TR-emitter-032) is realised by **injection**: a `WorkCounter` port, a counting implementation in `tests/support/`, and `NullWorkCounter` injected in every shipped and benchmarked build. Runtime `OS.has_feature()` tags (ADR-0031) gate dev features; they are not a substitute for exclusion.

**4.5 Time and randomness.** There is **one** forbidden-token list, owned here (`SIM-02`, no waiver); ADR-0001 and ADR-0005 § 1.7 point at it instead of keeping their own. It is the union of their entries and applies to all of `src/` outside the allow-list: any `Time.` member (`get_ticks_msec`, `get_ticks_usec`, `get_unix_time_from_system`, `get_datetime_dict_from_system`, `get_time_dict_from_system`, ...); `OS.get_ticks_*`, `OS.get_unix_time*`, `OS.delay_*` (Godot 3 spellings or rarely used names, listed so a copied call fails the lint as well as the parser); `Engine.get_frames_*`, `Engine.get_physics_frames`, `Engine.get_process_frames`; `get_process_delta_time`, `get_physics_process_delta_time`; `randi`, `randf`, `randi_range`, `randf_range`, `randfn`, `randomize`, `seed(`, `rand_from_seed`, `RandomNumberGenerator`, `Crypto`, `pick_random`, `shuffle`. Each token has a bad fixture.

The **allow-list is exactly four entries**: `src/game/kernel_host.gd`, the diagnostics timer (ADR-0001), `src/game/monotonic_clock.gd` (the production `Clock` that the Pipeline, Controller and others receive; TR-pxl-031, TR-pc-045) and `src/game/system_wall_clock.gd` (the production `WallClock`). ADR-0001's own clock allow-list is replaced by a pointer to this one. OQ-D is closed by this paragraph (`monotonic_clock.gd` and `system_wall_clock.gd`).

**Wall time has one sanctioned source: the `WallClock` port** (`src/save/wall_clock.gd`, `@abstract class_name WallClock`, `@abstract func now_unix() -> int`), implemented by `src/game/system_wall_clock.gd`. The port is kept out of the kernel. The composition root injects it **only** into `SaveService.setup()`. A wall-clock value never enters a block, a hash or an ordering decision: ADR-0006's `created_unix` and the `SaveInfo` time are display metadata, and every millisecond timeout names the injected `Clock` port. **Render-time timers** in the Adapter and Presenters (TR-pc-027) use `delta`, `present_time` or the injected `Clock` and contain no `Time.*` token; ADR-0005's separate render-timer allowance is withdrawn. Time-dependent logic receives `Clock.now_us()` or ticks by injection and tests drive a `FakeClock`.

### 5. Static typing and warnings as errors

**5.1 The settings.** The engine's analyzer is the authority for type flow (tier E1). `project.godot` pins, under `[debug]`:

- **Every `debug/gdscript/warnings/*` severity is 2 (error) except `RETURN_VALUE_DISCARDED`, which is 0.** No warning is left at 1, because a level-1 warning prints nothing headless (P10) and therefore cannot fail CI. The 2s include `UNTYPED_DECLARATION`, `INFERRED_DECLARATION`, `UNSAFE_PROPERTY_ACCESS`, `UNSAFE_METHOD_ACCESS`, `UNSAFE_CAST`, `UNSAFE_CALL_ARGUMENT`, `UNSAFE_VOID_RETURN`, `NARROWING_CONVERSION`, `INTEGER_DIVISION`, `INFERENCE_ON_VARIANT`, `MISSING_AWAIT`, `STATIC_CALLED_ON_INSTANCE`, `UNUSED_VARIABLE`, `UNUSED_PARAMETER`, `UNUSED_PRIVATE_CLASS_VARIABLE`, `UNUSED_SIGNAL`, `SHADOWED_VARIABLE`, `SHADOWED_VARIABLE_BASE_CLASS`, `SHADOWED_GLOBAL_IDENTIFIER`, `UNREACHABLE_CODE`, `STANDALONE_EXPRESSION`, `INT_AS_ENUM_WITHOUT_CAST`, `ENUM_VARIABLE_WITHOUT_DEFAULT`, `NATIVE_METHOD_OVERRIDE`, `REDUNDANT_AWAIT`, `CONFUSABLE_*`, and every other warning the 4.7.2 property list returns (P1). `PRJ-01` compares the live `ProjectSettings` values with `tools/lint/config/warning_levels.json`, so a new warning added by an engine patch release (default 1) fails the check until it is classified. Warnings are an analyzer feature of binaries built with debug support (the editor and `template_debug`); a `template_release` export is expected not to evaluate them (UNVERIFIED, LINT-SPIKE-1(c)), so **promotion is verified on the editor binary only** (`load_all`, `PRJ-01`) and the release export is not a warning gate. `EMPTY_FILE` and `ENUM_VARIABLE_WITHOUT_DEFAULT` are among "every other warning" at 2.
- **`directory_rules = { "res://": 0, "res://addons": 0, "res://tests": 0, "res://src": 1, "res://tools": 1 }`.** `src/` and `tools/` are promoted; `tests/` and `addons/` are excluded. Subdirectories inherit (P3). This settles BDY-3's "[UNV] whether a directory scope exists": it exists on 4.7.2, with include/exclude semantics and global severity.
- **One tier of strictness for all of `src/`**, simulation and presentation alike (§ Alternatives 5). Presentation code that needs to name a `Node` type annotates it (`$Label` assigned to a `Label` variable, `get_node("Label") as Label`, a `Dictionary` element assigned to a typed `int`, and `get_child(0)` assigned to `Node` loaded clean in a scratch project with the typing and four `unsafe_*` warnings at 2, probed 2026-10-28; the other warnings were not part of that probe), and uses `IntMath` or `float()` where it divides.
- **`PRJ-02` project invariants** are hosted by ADR-0008 § 5.5(a)'s single headless gdUnit4 test, which reads in **two tiers because a `ConfigFile` read cannot see the effective default of an absent key**. Rows marked *required* are checked for presence and value with `ConfigFile`. Rows where *absent is correct* (for example `physics/common/physics_interpolation`, absent or false) are checked with `ProjectSettings.get_setting` effective values. Structural `ConfigFile` checks (an absent or empty `[autoload]`) remain at E2. **ADR-0008 § 5.1 is the normative key list** (stretch and render pins, the `fallback_to_*` keys, `thread_model`); this ADR owns only the mechanism. The other invariants: `[autoload]` absent or empty; `application/run/low_processor_mode` false (ADR-0001); no `physics_jitter_fix` or `max_physics_steps_per_frame` coupling (ADR-0001, registry `physics_callback_tick_source`). The reader pattern is stated once: a typed assignment `var v: int = ProjectSettings.get_setting(&"...")` (as in ADR-0008's `RenderConfigReader`), **never `as`** on the Variant return. ADR-0009 § 5 cites "asserted by effective value (ADR-0008 § 5.1)" and keeps its runtime boot walk.
- **`PRJ-04` (E2, NW): no unlisted warning waiver.** The engine, not the lexical tool, raises the promoted warnings, so `# lint-allow` cannot waive them, and an `@warning_ignore("untyped_declaration")`, `("unsafe_method_access")` or `("unsafe_cast")` would silently reopen the hole this section closes. Every occurrence of `@warning_ignore`, `@warning_ignore_start` and `@warning_ignore_restore` in `src/` and `tools/` (the `_start` and `_restore` forms counted too, and the comma-separated argument list of `@warning_ignore("a", "b")` split into single warning ids) is matched against `tools/lint/config/allowlists/warning_ignore.json` (entries `{path, function, warning_id, reason, owner}`). The initial list has **one** entry: `integer_division` in `IntMath.trunc_div`. `SIM-05` is the special case "exactly one `integer_division` entry". `tests/` and `addons/` are outside the scope.

**5.2 The CI gate is "every script loads".** Promoted warnings fire when a script is parsed (P3). `tools/ci/load_all.gd` enumerates `res://src/**/*.gd` and `res://tools/**/*.gd`, `load()`s each, and fails when the result is null, `can_instantiate()` is false, or the process log contains a `SCRIPT ERROR:` line (a stray promoted warning prints as `Parse Error ... (Warning treated as error.)`). `load()` returns a non-null object even after a parse error (P3), so the `can_instantiate()` check and the log scan are both required; LINT-SPIKE-1(b) confirms `can_instantiate()` for the warning-as-error case. **Abstract and static-only scripts are UNVERIFIED:** P4 shows only that reflective `new()` succeeds, and `can_instantiate()` was not probed on an `@abstract` port or on a static-only class such as `IntMath`; a false result would fail every port. LINT-SPIKE-1(a) probes both; if either is false, `load_all` exempts scripts whose `is_abstract()` is true, or, where that API is missing too, relies on the `SCRIPT ERROR:` log scan alone for those scripts. The runner wiring (`extends SceneTree`, argument parsing, `quit(code)`) is in § 6.1. The framework's existing `.claude/scripts/godot-parse-check.gd` is the single-file variant used locally.

**5.3 Typing conventions the warnings do not catch** (lexical, `SIM-09`, scoped to `src/shared`, `src/kernel`, `src/sim`): no `Variant`-typed declaration except in allow-listed files (`src/data/*_read.gd`, command payload and snapshot code, and `src/shared/det/canonical_encoder.gd` with its reason, plus ADR-0005's named Variant pass-through signatures `Hash64.of_value(value: Variant)` in `src/shared/det/hash64.gd` and `canonical_encode(value: Variant)`, allow-listed by file and function name rather than whole file, all in `tools/lint/config/allowlists/dictionary_variant.json`; *amended 2026-10-28, post-confirm alignment with ADR-0005 § 3.8 a*); no bare `Array`; a bare `Dictionary` only where ADR-0002 (payloads) and ADR-0003 (snapshot, restore) require one; everything else uses `Array[T]` and `Dictionary[K, V]` (4.4; `deprecated-apis.md` "untyped Array/Dictionary"). The sim never iterates a `Dictionary` (ADR-0007), which the review checks and a heuristic rule flags (`for ... in <dictionary-typed name>`, advisory). Four engine rules shape every typed collection and are stated here once:
1. **`Array[T]` is invariant.** An `Array[Dog]` is not assignable to `Array[Animal]`, nor an `Array[Impl]` to `Array[Port]`; hold the elements as the wider type from the start or copy into a new typed array.
2. **Typed collections do not nest.** `Array[Array[int]]` and `Dictionary[int, Array[int]]` are not valid; a catalog of lists uses a wrapper class that holds an `Array[int]`, or an index array plus one flat `Array[int]`.
3. **Many engine calls return a bare `Array`** (`Dictionary.keys()`/`values()`, `Array.map()`, `ClassDB` lists). Assigning one to an `Array[T]` variable is a type error; use `typed.assign(untyped)`. `Dictionary[K, V].keys()` stays untyped (4.4).
4. **Sanctioned Variant unpacking:** a `typeof` check, then a typed assignment (`var i: int = v`) and `Array[T].assign(d.keys())`; never `as` on a Variant.

The sorted-key helpers (ADR-0005's only allowed iteration form) cannot be generic over the value type, so ADR-0005 declares two typed signatures on `CanonicalEncoder` in `src/shared/det/canonical_encoder.gd`: `static func sorted_int_keys(d: Dictionary) -> Array[int]` and `static func sorted_string_keys(d: Dictionary) -> Array[String]`, each calling `.keys()` once and `.assign`. ADR-0005 owns the names and the file (ruling TD-12). The bare-`Array` and `.keys()` tokens are allowed in that file only, through the same allow-list entry as the encoder, with reason and owner. Fixtures: the file passes, and the same code under `src/sim/` fails `SIM-09`. *(Amended 2026-10-28, post-confirm alignment: this was a separate `sorted_keys.gd` with `sorted_text_keys`, which disagreed with ADR-0005's Key Interfaces. OQ-N item (2) is answered by ADR-0005.)*

**5.4 Integer discipline.** *Engine half.* `INTEGER_DIVISION=2` makes `a / b` on two `int` operands a load error unless annotated, and nothing more. It does **not** fire for `/=`, for `Vector2i / int`, `Vector3i / int`, `Rect2i` helpers, `Vector2i % int` or `Vector2i * float` (which yields a `Vector2`); as understood it is not caught by `NARROWING_CONVERSION` when stored back into a `Vector2i` either (UNVERIFIED, LINT-SPIKE-1(b)); and no engine warning covers `%` or `%=` (reviewer probe on 4.7.2). `Vector2i` division truncates toward zero silently, which is the BDY-35 negative-division hazard. The lexical half below closes every gap.

*The single site.* The repository has **exactly one** annotated division: `@warning_ignore("integer_division")` on the `return a / b` statement in `IntMath.trunc_div` (`src/shared/int_math.gd`), enforced by `PRJ-04` (allow-list) and `SIM-05` (the count of `integer_division` ignores under `src/` equals 1, located in that function). **ADR-0005 owns the `IntMath` function list and the divisor guard** (§ 1.4 and § 1.7: `trunc_div`, `floor_div`, `posmod_i`, `ceil_div`, `lsr`, `low32`; for `b <= 0` the helper returns 0 with a debug `assert`; a divisor is always a positive constant or a catalog value validated at load). This ADR keeps the path, the single-site rule and the fixtures. The P11 fixtures (`-181/256 = 0`, `-362/256 = -1`, `-768/256 = -3`, `floor_div(-1, 48) = -1`) plus `b = 0`, `b < 0` and `INT64_MIN / -1` (all return 0 by the guard) are committed as the first unit test `tests/unit/shared/int_math_test.gd`; the `b <= 0` cases run in a suite whose expected `assert` log line is a suite-scoped `log_allowlist.json` entry (§ 8.6; whether the failed `assert` aborts the helper in a headless debug run is UNVERIFIED, LINT-SPIKE-1(j)). `SIM-11` bans `assert(` elsewhere in `src/` because it is stripped in release; the `IntMath` guard is its single allow-list entry, a debug diagnostic beside a defined release result, never the carrier of an invariant. Vector division: `IntMath.trunc_div_v2i` and `IntMath.floor_div_v2i` (per-component calls to the scalar helpers, so the single-site rule holds) are requested as additions to ADR-0005's list (OQ-N); until they exist, vector division is written per component through the scalar helpers. OQ-L is closed: `warning_ignore.json` holds exactly one site (ADR-0005 § 1.4 rule 2, § 1.7), and any future inline `%` in a hot path needs an ADR-0005 amendment naming one function, to which this ADR then adds only that function.

*Lexical rules* (tier E2, scope `src/shared`, `src/kernel`, `src/sim` outside `src/shared/int_math.gd`, because a lexical scan cannot know operand types):
- **`SIM-04`** bans the operator tokens **`/`, `%`, `/=` and `%=` outright**, with `/` or `%` inside strings and comments already stripped by the scanner and a `%` following a string literal treated as formatting (formatting is itself banned in hot code, § 7). It also bans `min`, `max`, `clamp`, `abs`, `sign`, `floor`, `ceil`, `round`, `sqrt`, `pow`, `lerp` and `int(`/`float(` conversions (use `mini`, `maxi`, `clampi`, `absi`, `signi`) and the trigonometric functions `sin`, `cos`, `tan`, `atan`, `atan2`, `asin`, `acos` outside the float allow-list. `src/shared/det/det_rng_float.gd` is exempt for its int-to-float promotion (`float(`, `floor`) only.
- **`SIM-03`** (§ 5.5) adds the member bans `.length(`, `.length_squared(`, `.normalized(`, `.distance_to(`, `.angle(`, `.aspect(`, `.lerp(`, `.rotated(` and `.snapped(`; multiplying a `Vector2i` by a float literal is already a `SIM-03` finding because every float literal is.
- **`SIM-15`** bans decimal integer literals above 9223372036854775807, which no engine warning catches.
- **Fixtures.** Each rule and each banned token has a bad fixture: `Vector2i(5, -3) / 2`, `a /= b`, `a %= b`, `x % 3`, each trigonometric function, each member ban and `9223372036854775808` (the `Vector2i` fixture is also LINT-SPIKE-1(b)). This is the grep lint of TR-emitter-034 and TR-shipbody-002, generalised from "the Emitter directory" and "the Body and Layout directories" to the whole simulation.

**5.5 Float-free simulation.** `SIM-03` bans `float` as a declaration or return type, float literals (a decimal point or exponent in a numeric token), `Vector2(`/`Vector3(`/`Transform2D` (use `Vector2i`), `INF`, `NAN`, `PI`, `TAU`, and the float-producing members of § 5.4 in `src/shared`, `src/sim` and (outside the allow-list `float_allowlist.json`) `src/kernel` and `src/data`. The allow-list is scoped per directory: for `src/kernel` and `src/data` it lists `presentation_clock.gd`, the diagnostics timer and the `*_read.gd` content readers (`kernel_host.gd` lives in `src/game/`, outside the scope); **for `src/shared` it lists exactly `src/shared/det/det_rng_float.gd` and `src/shared/det/canonical_encoder.gd`** (ADR-0005 § 2.1 and § 3.5). A lexical ban cannot see a float produced by an engine call that returns one; the engine tier (`NARROWING_CONVERSION`, `UNSAFE_*`, `INFERENCE_ON_VARIANT` at 2) catches the assignment, and the definitive check is runtime: **the strict-mode encoder (tests, `STRICT_DEBUG`, content hashes) rejects a float in a block that is not on the float allow-list; release encodes normalised binary64** (ADR-0005 § 3.5), so a float that reaches a block fails the first strict golden replay.

### 6. The lint tool

**6.1 Shape.** One headless GDScript runner, `tools/lint/lint_main.gd`, run as `godot --headless --path . -s res://tools/lint/lint_main.gd -- [--pack <name>] [--changed-since <rev>]`. **Runner wiring** (`lint_main.gd` and `tools/ci/load_all.gd` alike): launched with `-s`, each script `extends SceneTree`, does its work in `_initialize()`, reads its arguments with `OS.get_cmdline_user_args()` (the values after `--`) and ends with `quit(0)`, `quit(1)` or `quit(2)` per § 6.2; a script that never calls `quit` never exits, and one that forgets the code always exits 0. A fixture test asserts that a rule which throws maps to exit 2. Lint and CI exit codes are not fault domains (the kernel fault-domain space is unaffected). Why GDScript and not `gdtoolkit` (Python): reflection and the class map exist only in-engine (P5); ADR-0007's `content_lint.gd` already makes GDScript the CI tool language and shares the `ContentLoader`; one binary on the CI runner; `gdtoolkit` 4.5.0 does not document the 4.5+ grammar (UNVERIFIED, checked 2026-10-28). `gdformat`/`gdlint` may be run **advisorily** by a developer; no blocking rule depends on them.

- `tools/lint/core/`: `LintScanner` (the lexical front end), `LintRule`, `LintReport`, `LintConfig`.
- `tools/lint/<pack>_lint.gd`: one file per rule pack (`arch`, `sim`, `kernel`, `prj`, `alloc`, `msa`, ...). Owner ADRs add packs; TR-assembly-044's `tools/lint/msa_lint.gd` is such a pack.
- `tools/lint/config/*.json`: modules, edges, token lists, allow-lists (each entry `{path_glob, symbol, reason, owner, added}`), warning levels, kernel members, frozen classes, `owners.json`. These are tool configuration, parsed with `JSON.new()`; they are not game content.

**6.2 Finding format and exit codes.** One line per finding: `LINT-<PACK>-<NN> <error|warn> <path>:<line>: <rule_name>: <message>` (for example `LINT-MSA-3 error src/sim/assembly/x.gd:42: ...`). Exit codes: **0** no findings at error severity; **1** at least one error finding; **2** tool failure (import cache missing, class map stale or disagreeing with the lexical scan (§ 2.1), config unreadable, a rule threw). A `warn` finding never fails the run but is counted in the summary. Output is ordered by `(path, line, rule)` so two runs are byte-identical. The tool prints a summary of waivers in force.

**6.3 The lexical front end and its limits (tier E2).** `LintScanner` strips `#` comments and string literals in **every** GDScript form, replaces each string with `""` and preserves line numbers. Forms handled: double- and single-quoted strings (`"..."`, `'...'`), triple-quoted strings ("""...""" and '''...'''), the prefixed forms `r"..."`/`r'...'`, `&"..."`/`&'...'`, `^"..."`/`^'...'`, `$"..."`/`$'...'`, and the unique-node forms `%"..."` and `%Name`; backslash-escaped quotes inside a string; a `#` inside a string (not a comment). It tracks `func` boundaries by indentation so a rule can be scoped to a function name, including `@abstract func`, `static func` and `@warning_ignore(...)` prefixes on `func`, `func` inside inner classes, multi-line lambdas, and `if`/`for`/`while`/`match` suites; mixed tab and space indentation is reported, not guessed (the engine rejects it in a block). It reads `## @hot` and `# lint-allow:` markers from the original text. It does **not** build an AST. Consequently it **can** see: tokens, counts, class names, call-site names, function names, decorators, `.tscn`/`project.godot`/`export_presets.cfg` text through `ConfigFile`; it **cannot** see: the static type of an expression, which object a method is called on, data flow, whether a write happens "inside a WriteScope", runtime process priorities. Each of the latter is assigned to E1, E3 or E4 below, never silently dropped (§ 6.6 and the registry table that follows it). Line continuations with a trailing backslash and nested multi-line constructs are handled by fixtures, one fixture per form above; an unhandled construct is a tool bug (**exit 2**), not a silent pass. **Corpus check (LINT-SPIKE-1(d)):** the scanner runs over every committed `.gd` file in the repository, including `addons/gdUnit4/` and `.claude/scripts/`; it must not exit 2 on a file the engine loads cleanly, the string-stripped output must keep the original line count, and a second scan of that output must change nothing.

**6.4 Reflection tier (E3).** Run by the lint CLI after `--import` and by gdUnit4 suites, always in the **live editor binary** so that engine class lists track the patch release:
- `REFLECT-SIG` (the field list of § 3.3) and `REFLECT-ABSTRACT` (fallback only, § 3.4).
- `ARCH-05`, reflection half: every `class_name` satisfies `not ClassDB.class_exists(name)` (a `class_name` equal to a native class "hides a native class" and fails; the engine gained `Logger` in 4.5 and `AccessibilityServer` in 4.7); no method declared in a `src/` class appears in `ClassDB.class_get_method_list(base_native_class, false)` (defining `get`, `set`, `free`, `call`, `callv`, `connect`, `tr`, `to_string`, `has_method`, `notification` or `get_class` is `NATIVE_METHOD_OVERRIDE`); no member name appears in `ClassDB.class_get_property_list(base_native_class)` (a field named `script` is `SHADOWED_VARIABLE_BASE_CLASS`).
- `KRN-01`: `Kernel`'s **declared** fields equal the allow-list in `tools/lint/config/kernel_members.json` (`kernel.clock`, queue, registries, binding tables, command log, kernel state, diagnostics, ADR-0003 § 10). `Script.get_script_property_list()` returns more than the fields, so the comparison first filters to entries with `usage & PROPERTY_USAGE_SCRIPT_VARIABLE`, drops the category entry for the script itself, and treats base-script members explicitly (`Kernel` extends `RefCounted` directly, so none; a future base script's members would count and be listed). `static` members appear and are banned by `ARCH-04` anyway. Fixtures: a bad `Kernel` with one extra field, a good `Kernel` with exactly the allow-list. The observed list shape is recorded by LINT-SPIKE-1(d) (UNVERIFIED until then).
- `SIM-10` (every registered block's script has base `RefCounted`, `not (block is Node)`, `not (block is Resource)`).
- `ContractCheck.diff(real: Script, double: Script) -> Array[String]`, used by contract tests for the stubs of TR-pc-045 and TR-charttable-038. It compares, per entry of `get_script_method_list()`: the name; each `args[i].type`, `.class_name` and `.hint_string`; the default-argument count; `return.type`, `return.class_name` and `return.hint_string`; and the flags word masked to static and vararg. Recorded contract fixtures `tests/fixtures/contracts/*.json` hold exactly those fields and are generated by `tools/lint/gen_contracts.gd` from the real classes and committed; every stub must pass them.
- The same reflection pass writes the **published-identifier list** (hooks, commands, mutators, read-view methods, notification ids) from the kernel registry dump plus public-method reflection of `views/` and mutator classes to `build/lint/published.json` (generated, not committed), the machine-readable list TR-boarding-031 requires (deferred, § 6.10). The kernel exposes a dev-build registry dump (assumption on ADR-0003).

**6.5 Runtime and boot-walk tier (E4).** Executed by gdUnit4 integration tests that boot the composition root headless, and by `src/game/` in dev builds: `KRN-05` (ADR-0001's walk: every node with `process_priority` below `TICK_DRIVER` is a registered INPUT_SURFACE object; only `KernelHost` uses `TICK_DRIVER`'s value and only `SaveHost` uses `HOST_SERVICE`'s (the known-node set adds `SaveHost`); `KernelHost`, `SaveHost`, every INPUT_SURFACE and every presentation node resolves to the main-thread `process_thread_group` by following `INHERIT` up to the first explicit ancestor), `CNT-01` to `CNT-04` (§ 7), `PRJ-02` recheck, orphan-node and leaked-RefCounted counts after a suite, and ADR-0004's dirty-key check (forward reference).

**6.6 Rule inventory: what is inherited, where it is enforced.** `NW` marks rules with no waiver.

| Rule id | Statement | Source | Tier | Notes |
|---|---|---|---|---|
| ARCH-01 | Every `.gd` is in an allowed top directory and a known module directory | IC-10, this ADR | E2 | `modules.json` |
| ARCH-02 | Import edges follow § 2.2 | architecture rules 1 to 4; ADR-0007 `src/data` never imports `src/sim` | E2 + class map | class map = `get_global_class_list()` cross-checked by a lexical scan (§ 2.1); the facets list and `published_types` are data in `edges.json` |
| ARCH-03 (NW) | No `preload`, `load` or `ResourceLoader` of a `.gd` path in `src/`; non-script `preload` only in `presentation` and `game` | architecture rule 2 | E2 | one guarded `load()` of the dev roots in `game/` is allow-listed |
| ARCH-04 (NW) | `[autoload]` empty; no `static var` in `src/` | registry `kernel_autoload_or_global`; ADR-0007 "no `static var` in content code" | E2 (`ConfigFile` + tokens) | |
| ARCH-05 | One `class_name` per file; file name = snake_case of class; ports (under `ports/` and the `port_files` entries) are `@abstract`; **reflection half:** no `class_name` equals a native class, no method or member name clashes with its native base (§ 6.4) | § 1.2, § 3; engine review E6, E9 | E2 + E3 | the "ports are `@abstract`" check uses `Script.is_abstract()` if LINT-SPIKE-1(a) finds it, otherwise is lexical |
| ARCH-06 | No `._member` access through a foreign receiver (anything but `self`); catalog `_field` assignment only inside `_freeze` for classes in `frozen_classes.json` | ADR-0007 § 1.5 write-once fields | E2 (function-scoped) | GDScript has no private fields; a per-line `# lint-allow: ARCH-06` with reason covers same-class peer access |
| ARCH-07 | `.tscn` only under `presentation`, `game`, `debug`, `observer`; no `process_priority` key in any `.tscn` | ADR-0001 | E2 | |
| ARCH-08 (NW) | `Kernel`, `KernelHost`, `TickDriver` named only in `kernel/`, `game/`; the **tick-phase deny list** `SimViews`, `WriteScope`, `Command`, `BlockDescriptor`, `CallContext`, `ValidateContext`, `ApplyContext`, `HookContext`, `SnapshotContext` absent from `presentation/`, `debug/`, `observer/` (no `*Context` glob: `NotifyContext` is a `facets/` type); `NotifyFacet` named only by `src/game/` and `PresentationFacet` | ADR-0003 facet-leak risk; ADR-0004; ADR-0009; registry | E2 | lists in `edges.json` (§ 1.2) |
| ARCH-09 | `testing/` classes named only from `tests/` and other `testing/` | TR-cmd-050 | E2 + pack scan | § 4.3 |
| ARCH-10 | `debug/` and `observer/` are named by no script and no scene except the guarded loader; their root declares no `class_name` | TR-debug-037 | E2 | ADR-0031 owns the gating |
| ARCH-11 | A property whose type is a `Packed*Array` and that has a `set(` accessor is a finding in `src/` (4.7: `arr[i] = v` on a Packed property no longer calls the setter), unless the owner marks dirty explicitly through `scope.mark_dirty` (ADR-0004) | 4.7 breaking change; registry `write_outside_writescope_or_setter_dirty` | E2 | § 7 `CNT-03` states the matching accessor rule |
| SIM-01 (NW) | Tree/engine-server token list absent from L0 to L2 | kernel H10; TR-cmd-033, TR-suit-006, TR-section-002, TR-derelict-026, TR-webmap-031 | E2 | § 4.1 |
| SIM-02 (NW) | The unified wall-clock, frame-counter and global-RNG token list of § 4.5 absent from `src/` outside the four-entry allow-list (`kernel_host.gd`, the diagnostics timer, `monotonic_clock.gd`, `system_wall_clock.gd`) | kernel H9; ADR-0001; ADR-0005 § 1.7; BDY-3; TR-suit SE-37 | E2 | § 4.5; one list, owned here; ADR-0001 and ADR-0005 point at it |
| SIM-03 | Float-free declarations, literals, constructors (`float`, decimal literals, `Vector2(`, `Vector3(`, `Transform2D`, `INF`, `NAN`, `PI`, `TAU`) and float-producing members (`.length(`, `.length_squared(`, `.normalized(`, `.distance_to(`, `.angle(`, `.aspect(`, `.lerp(`, `.rotated(`, `.snapped(`) in `shared`, `sim`; confined elsewhere; the `src/shared` allow-list is exactly `det_rng_float.gd` and `canonical_encoder.gd` | kernel H9; LS-1; LY-2; TR-shipbody-002 | E2 + E1 + E4 | § 5.4, § 5.5 |
| SIM-04 | The operator tokens `/`, `%`, `/=`, `%=` banned outright, and untyped numeric builtins (`min`, `max`, `clamp`, `abs`, `sign`, `floor`, `ceil`, `round`, `sqrt`, `pow`, `lerp`, `int(`, `float(`, `sin`, `cos`, `tan`, `atan`, `atan2`, `asin`, `acos`), in `shared`, `kernel`, `sim` outside `IntMath` (`det_rng_float.gd` exempt for `float(` and `floor`) | TR-emitter-034; TD-14 | E2 | § 5.4; a lexical scan cannot know operand types, so no operand-type carve-out |
| SIM-05 | Exactly one `integer_division` ignore in `src/`, inside `IntMath.trunc_div` (the special case of `PRJ-04`) | TR-shipbody-002, BDY-35 | E2 | |
| SIM-06 (NW) | No `_physics_process`, `_integrate_forces`, `move_and_slide`, `move_and_collide`, `test_move`, `body_test_motion`, `RigidBody2D`, `CharacterBody2D`, `StaticBody2D`, `AnimatableBody2D`, `Area2D`, `PhysicsServer2D`, `physics_jitter_fix`, `max_physics_steps_per_frame` in `src/` | registry `physics_callback_tick_source`; BDY-3 | E2 + `PRJ-02` | the ADR-0033 debris projector will need a recorded allow-list entry |
| SIM-07 | No `await` in `shared`, `kernel`, `sim`, `data`; no `await ... process_frame` anywhere in `sim`/`presentation` | ADR-0003 (owner callables); registry `await_process_frame_in_sim_or_presentation` | E2 | `MISSING_AWAIT=2` also in E1 |
| SIM-08 | No `MultiplayerAPI`, `ENet*`, `WebSocket*`, `rpc`, `@rpc`, `multiplayer.` in `src/` | kernel H8; TR-cmd-033, TR-cmd-051 | E2 | ADR-0032 lifts it by editing the config |
| SIM-09 | No `Variant`, bare `Array`, undeclared bare `Dictionary` in `shared`, `kernel`, `sim` | `deprecated-apis.md` | E2 | § 5.3; allow-list entries `dictionary_variant.json` (includes `canonical_encoder.gd`) and `sorted_keys.gd` (bare `Array`, `.keys()`) |
| SIM-11 | No `assert(` in `src/` outside `testing/`; boot invariants and runtime guards return fault codes. The one allow-list entry is the debug `assert` of the `IntMath` divisor guard (ADR-0005), which sits beside a defined release result | TR-emitter-031; registry `loader_side_effect_or_foreign_boot_code` (`assert` is stripped in release) | E2 | |
| SIM-12 | Dynamic dispatch by string (`call(`, `callv(`, `call_deferred(`, `Object.get(`/`set(` by non-literal name, `has_method(`) absent from `shared`, `kernel`, `sim`; the deferred-call family `call_deferred`, `set_deferred`, `call_thread_safe` and `Engine.get_main_loop` additionally absent from `src/save/` and `src/data/` (presentation's UI use of `set_deferred` is unaffected) | closes the lexical blind spot of `SIM-01`/`SIM-02`; ADR-0006 (cites this rule and `IO-02`) | E2 (heuristic, advisory until fixtures settle) | `Callable.call` on a registered `Callable` is not dynamic dispatch by string and is allowed; deferred in § 6.10 |
| SIM-10 | Registered blocks derive `RefCounted`, are not `Node` or `Resource` | registry `block_as_resource_or_node`; CR-8 | E3 | |
| SIM-15 | No decimal integer literal above 9223372036854775807 in `shared`, `kernel`, `sim`, `data` (no engine warning catches an over-range literal) | TD-14 | E2 | bad fixture `9223372036854775808` |
| KRN-01 | Kernel member allow-list (H1) | CR-14 H1; ADR-0003 § 10; TR-cmd-051 | E3 | |
| KRN-02 (NW) | `Command.new(` only in `src/kernel/`; `issuer_peer`, `origin`, `seq`, `submission_id` never appear as payload keys | registry `command_constructed_outside_kernel`; ADR-0002 | E2 | the run-time handle seal is ADR-0002's |
| KRN-03 (NW) | No numeric literal assigned to `process_priority`; `TICK_DRIVER` named only in `kernel_host.gd` and `process_priority.gd`; `HOST_SERVICE` (-40) named only in `save_host.gd` and `process_priority.gd`; presentation priority is the `PRESENTATION` constant | ADR-0001 | E2 | |
| KRN-04 | `submit(` is called only inside `_process`, and only in the files of the objects named at `issue_surface_handle(`; the allow-list is the set of class names passed to `issue_surface_handle(` in `src/game/`, resolved through the class map (no hand-kept file list). At MVP that is the Input Adapter only; ADR-0009 § 7's general INPUT_SURFACE rule applies once a second `SurfaceHandle` is issued | ADR-0002 (owns the list; registry `presentation_submit_without_handle`); ADR-0009 § 7 cites this rule | E2 (function-scoped) | heuristic, fixture-backed; the run-time handle seal is ADR-0002's |
| KRN-05 | Process-priority and `process_thread_group` tree walk | ADR-0001 | E4 | an E2 regex cannot see run-time priorities |
| KRN-06 | `Engine.time_scale` assigned only in dev-feature files; `TIME_SCALE_NON_UNIT` and `low_processor_mode` boot checks | ADR-0001 | E2 + E4 | |
| KRN-07 | Owner callables registered through the registrar declare `-> CompletionToken` and their last statement is `return ctx.complete()` (an E2 check feasible at function scope on registration call sites) | registry `void_or_value_returning_owner_callable`; ADR-0003 | E2 (function-scoped) + runtime | the runtime half is ADR-0003's INCOMPLETE detection; a handler returning anything but the armed token is recorded INCOMPLETE (ADR-0004 § 2); deferred in § 6.10 |
| PRJ-01 | Warning levels match `warning_levels.json`; none at 1 | § 5.1 | E3 (`ProjectSettings`) | |
| PRJ-02 | Project invariants in two read tiers: `ConfigFile` presence for *required* rows, `ProjectSettings.get_setting` effective values for *absent is correct* rows; the normative key list is ADR-0008 § 5.1 | ADR-0001, ADR-0008, ADR-0009 | E2 + E4 | hosted by ADR-0008 § 5.5(a)'s one headless test (§ 5.1) |
| PRJ-03 | Every `src/` and `tools/` script loads | § 5.2 | E1 | `load_all.gd` |
| PRJ-04 (NW) | Every `@warning_ignore`, `@warning_ignore_start` and `@warning_ignore_restore` in `src/` and `tools/` (argument list split, `_start` and `_restore` forms counted) is matched against `allowlists/warning_ignore.json`; initial list: `integer_division` in `IntMath.trunc_div` | § 5.1; engine review E3 | E2 | the engine raises these warnings, so `# lint-allow` cannot cover them |
| IO-01 | `FileAccess`, `DirAccess`, `JSON`, `ResourceLoader` only in the allow-list: `src/data/file_content_source.gd`, `src/save/**`, `src/observer/**`, `src/game/` dev loader; `JSON.parse_string` nowhere. **`HashingContext` is allowed only in named files**: `src/save/**`, `src/data/**`, `src/shared/det/hash64.gd`, `src/kernel/state_hasher.gd` | ADR-0007 (registry `content_file_read`, `content_directory_scan`); ADR-0005 | E2 | named files, not whole layers; `src/sim/graph/` is not on the list (ADR-0005 § 3.1 withdrew owner `canonical_bytes()`) |
| IO-02 | `Thread`, `Mutex`, `Semaphore`, `WorkerThreadPool` only under `src/save/`, across all of `src/` (the one legal thread user). The only cross-thread payload is a `PackedByteArray` or `PackedInt64Array` handed over by value and never mutated by either side afterwards | ADR-0006 (cites this rule and `SIM-12` instead of restating them) | E2 | scoped to the directory, not a file list |
| SIM-13 | No `signal` declaration, `.connect(`, `emit_signal(` in `src/kernel/` and `src/sim/` | ADR-0004 § 1 (kernel observer registry; no engine signals for kernel notification); ADR-0009 lint list | E2 | settled rule; changing it requires an ADR-0004 amendment |
| DET-01 to DET-nn | The ADR-0005 § 1.7 table: RNG and clock tokens (covered by `SIM-02`), `hash(`, `get_instance_id`, `.sort_custom(`, `.shuffle(`, `.pick_random(`, `ResourceUID`, canonical byte writers (`encode_s64`, `encode_u32`, `encode_double`, `to_byte_array`) and hashing of a `var_to_bytes` result in `src/kernel/`, `src/sim/` and block owners (allowed in `src/shared/det/` and tests; `src/save/` may use them for container byte layout, `slot_epoch.bin` and integrity hashes such as `payload_sha256` and the trailer, whose output never feeds H64 or `state_hash`), `RESTORE_KEY_STRIDE` outside `DetRng`/`RngFacet` | ADR-0005 | E2 for the token rows; **Dictionary iteration and `StringName` sort are not decidable lexically**: use ADR-0005's own fallback (`sorted_int_keys`/`sorted_text_keys` of § 5.3 are the only allowed forms, raw forms banned by token; a test sorts the canonical key set only through `CanonicalEncoder`) | pack `tools/lint/det_lint.gd`; ADR-0005 owns the list, this ADR owns the mechanism |
| PRES-01 | `_physics_process` is banned in `src/presentation/`. `_process` is forbidden in any object registered through `PresenterDescriptor`; it is allowed only in `PipelineRoot` (the PresentationHost), INPUT_SURFACE objects at `ProcessPriority.INPUT_ADAPTER`, and layer-3/4 UI Controls. The exemption set is ADR-0009's | ADR-0009 (TD-02) | E2 (function-scoped) for tokens, E4 for registration facts | pack `tools/lint/presentation_lint.gd`; the shared presentation files (`placement.gd`, `presentation_stage.gd`, `presentation_driver.gd`, `PresentFrame`) live in `src/presentation/pipeline/`, so `ARCH-01` needs no new directory |
| PRES-02 | `tick_alpha` is read only in `presentation_driver.gd` (and `PresentationClock` itself) | ADR-0009 | E2 | |
| PRES-03 | Outside `placement.gd` and the named allow-list: no `set_position(`, `set_global_position(`, `global_position =`, `translate(`, `global_translate(`, `move_local_x(`, `move_local_y(`, `set_offset(`, `offset =`, `position =` or `transform =` on a CanvasItem; `Camera2D`/`Viewport` members only in the Pipeline root | ADR-0009 (TD-17) | E2 (function-scoped) | allow-list `allowlists/placement.json` (ADR-0009 § 4): `src/presentation/pipeline/viewport_wiring.gd` (`WorldContainer.position`, `HudRoot.size`, layer geometry) and the camera host's `frame_pre_draw` handler (`canvas_transform` and the parallax layer offsets, computed with Placement's integer `floori` helpers, not a bare `-floor(...)`) |
| PRES-04 | Resource scan: no Animation track on `:position`, `:global_position`, `:offset` or `:transform`, and no `tween_property` call on those names, for presentation nodes | ADR-0009 (TD-17) | E2 (`.tscn`/`.tres` text) | |
| PRES-05 | `sort_custom` and the packed-key sort rule apply to `src/presentation/pipeline/` as they do to the simulation | ADR-0009 (TD-17), ADR-0005 | E2 | |
| PRES-06 | No `await` in any script registered through `PresenterDescriptor` or that calls `.subscribe(`: a subscriber handler returns its `CompletionToken` synchronously | ADR-0004 § 2, ADR-0009 (TD-16) | E2 | the kernel's token-identity check is the runtime backstop; bad fixture |
| READ-01 | Reader allow-lists: symbol S may appear only in listed paths (`sanity.ledger`, raw attention level, `tell_remaining`, `true_value`, `Input.*`/`InputMap`/`InputEvent`) | TR-debug-038, TR-pc-032 | E2 | generic mechanism; the lists are owned by ADR-0022/0024/0019/0031 |
| READ-02 | Call-site allow-list: call S only inside function F of file P (`withdraw_store` only inside the V7 apply) | TR-suit-033 | E2 (function-scoped) | file and function granularity, not "inside a WriteScope" |
| READ-03 | Forbidden symbol in directory (band-constant literals, retired tokens, Sanity/Attention/`PERCEPTION_OUT` symbols in Suit, Sanity/Attention in HUD) | TR-suit-033, TR-boarding lints | E2 | token lists committed per owner |
| HOT-01 | In a function preceded by `## @hot`: no `.new(`, `.duplicate(`, `.duplicate_deep(`, `str(`, `String.`, string formatting, array/dictionary literals, `.slice(`, `.map(`, `.filter(`, `.reduce(`, `.sort_custom(`, `.keys()`, `.values()`, `.resize(`, `.append_array(`, lambdas, `Callable(`, `.bind(`, `PackedXArray(`, `Array(` | OQ-16; TR-lsg-025; TR-cmd-032 | E2 (function-scoped), deferred (§ 6.10) | § 7; an unwaived `.resize(` is a finding because it can reallocate |
| FIX-01 (NW) | Every rule has at least one bad and one good fixture and the bad fixtures fail | TR-emitter-034 | E3 (self-test) | § 6.7 |
| CNT-01 to CNT-04 | Object count, static memory, identity and read-only, work counters | OQ-16 | E4 | § 7 |
| CI-01 | Export presets, ledger and manifest history gates | ADR-0007 § 2.2 | E2 + shell | ADR-0007's `content_lint.gd`, not this tool |

**6.6b Registry forbidden patterns with no lexical rule.** Each registry `forbidden_patterns` entry that this ADR does not enforce lexically is assigned here, so none is silently dropped:

| Registry pattern | Tier and mechanism | Owner |
|---|---|---|
| `write_outside_writescope_or_setter_dirty` | Runtime: the `STRICT_DEBUG` write check (E4); lexical part only `ARCH-06` and `ARCH-11` (file and function granularity, never "inside a WriteScope") | ADR-0003, ADR-0004 |
| `void_or_value_returning_owner_callable` | `KRN-07` (E2) plus ADR-0003's runtime INCOMPLETE detection | ADR-0003 |
| `presentation_submit_without_handle` | `KRN-04` (E2) plus the run-time handle seal (E4) | ADR-0002 |
| `cosmetic_phase_from_tick_counter` | Runtime and review: `PRES-02` bans `tick_alpha` outside the driver; use of `present_time` for cosmetic phase is asserted by presenter tests. No lint | ADR-0001, ADR-0009 |
| `position_derived_persisted_id` | Runtime, owned by ADR-0007: ledger and golden-vector gates (`CI-01`, `content_lint.gd`). No lint here | ADR-0007 |
| `peer_id_keyed_persistent_state` | Runtime and review, owned by ADR-0002 (peer id never a state key). No lint here | ADR-0002 |

The Related list names the stances this ADR enforces lexically or by reflection; the table above covers the rest of the registry's current `forbidden_patterns`.
**6.7 Fixtures and the self-test.** Each rule has `tests/fixtures/lint/<rule_id>/bad_*.gd.fixture` and `good_*.gd.fixture` (the `.fixture` extension keeps the editor from parsing intentionally broken scripts and from registering their `class_name`s; UNVERIFIED that Godot ignores the extension, LINT-SPIKE-1(d)). `tests/unit/lint/` has one gdUnit4 suite that runs every rule over its fixtures: a bad fixture yields at least one finding with that rule id and a good fixture yields none (`FIX-01`). A rule without fixtures fails the suite.

**6.8 Where lints run (one library, two entry points).** TR-cmd-051 wants lints as blocking gdUnit4 tests under `tests/unit/kernel/lint/`; TR-assembly-044 wants a harness that runs before the suite and exits non-zero with `LINT-MSA-n` lines. Both are satisfied by one library: the CLI (`tools/lint/lint_main.gd`) runs every pack before gdUnit4 in CI; each system also has a thin wrapper `tests/unit/<system>/lint/<system>_lint_test.gd` that calls the same pack and asserts zero error findings under that system's path. They cannot disagree because they run the same code over the same config.

**6.9 Waivers.** A finding on a line is waived by a trailing `# lint-allow: <RULE-ID> <reason of at least 10 characters>`. Rules marked NW ignore waivers. The run prints every waiver in force and fails when the waiver count rises above the committed `tools/lint/config/waiver_baseline.json` without the baseline being edited in the same change. Allow-list entries need a named owner. `PRJ-04` is NW: a warning waiver exists only as an entry of `warning_ignore.json`.

**6.10 MVP minimum and deferred rules.** The ADR specifies about fifty rules, four tiers and several generators; a solo, LLM-assisted MVP does not need them all at once. **Required at KRN-LAT-02:** the § 5.1 settings and `PRJ-01`; `load_all` (`PRJ-03`); `ARCH-01`, `ARCH-02`, `ARCH-03`, `ARCH-04`, `ARCH-08`; `SIM-01`, `SIM-02`, `SIM-03`, `SIM-04`, `SIM-05`; `KRN-02`, `KRN-03`; `FIX-01`. **Every other rule lands with its owner's first story** (§ 6.6). **Deferred until the first Feature-layer module:** `HOT-01`, `SIM-12`, `CNT-02`, the waiver baseline (§ 6.9), `KRN-07` and `build/lint/published.json`. Typing cost on `tools/`: `tools/` is under warnings-as-errors, so the lint tool is fully typed while it parses JSON; the cost is confined to one `LintConfig` class that reads each config value by the `typeof` check and typed assignment of § 5.3 rule 4.

### 7. Verifying allocation and other unmeasurable claims

OQ-16: "zero heap allocations" cannot be measured directly. The claim is restated, wherever a GDD or TR says it, as the conjunction of four checks. Each says what it can see (probes P6 to P8).

| Check | Mechanism | Sees | Blind to |
|---|---|---|---|
| **HOT-01** (E2) | Functions the owner tags `## @hot` are scanned for allocation tokens (§ 6.6). The tag list is owned by each performance-bearing ADR (P3 hooks, queries such as `solid_in` and `lit`, `Layout.rects_in`, the Graph phases). | **Transient allocation**: arrays, strings, lambdas, `duplicate` | an allocation inside an untagged callee; an engine call that allocates internally |
| **CNT-01** (E4) | `Performance.get_monitor(Performance.OBJECT_COUNT)` equal before and after N ticks (N = 600 by default) at steady state after warm-up; a fixture that deliberately creates a reference cycle (Kernel to facet to Kernel) must be reported as a leak, which proves the check can fail (§ 4.2). | Leaked or churned **Objects** (every `RefCounted`, Node, Resource) | arrays, dictionaries, strings, packed arrays, lambdas (P6) |
| **CNT-02** (E4, debug binary only) | `MEMORY_STATIC` within a per-suite tolerance (committed constant, initial 4,096 bytes, tuned by LINT-SPIKE-1) after warm-up over N ticks. | **Retained growth** of any container (P7: +1.7 MB for 10,000 retained small arrays) | transient allocation freed inside the tick (P7: 208 bytes for 100,000 transient arrays); and availability in a release export is UNVERIFIED (expected unavailable, so this is a developer and CI-debug check, never a release P99). `MEMORY_STATIC` deltas also include engine-internal caches (the `StringName` table, the resource cache), which the tolerance must absorb. |
| **CNT-03** (E4) | For every array-returning accessor of a catalog or read view: `is_same(f(), f())` and `f().is_read_only()` (ADR-0007's check, P8). `==` is never used for identity. **Accessors return `Array[T]` or `Dictionary[K, V]` marked read-only, never a `Packed*Array`**: packed arrays have no `make_read_only`/`is_read_only` and GDScript passes them by reference, so a stored `PackedInt32Array` returned from a getter is mutable by the caller. The only `Packed*` fields allowed in `src/` are private to their owner, plus the § 9 native boundary and the `src/save/` boundary (ADR-0007 forbids exposing them; the rule extends to views). | A getter that copies, or exposes a mutable container | an allocation inside the getter that is returned and kept identical |
| **CNT-04** (E4) | Deterministic **work counters** injected through a `WorkCounter` port (null in shipped builds, § 4.4): counted work units (tiles tested, edges relaxed) equal the closed-form count from the GDD (TR-emitter-032 D.14). | Algorithmic budget regressions, independent of machine speed | per-operation cost |

Consequence for the wording of acceptance criteria: "makes no heap allocations per call" is replaced by "`HOT-01` clean on the tagged functions; `CNT-01` unchanged over 600 ticks; `CNT-02` within tolerance over 600 ticks on the debug binary; `CNT-03` for each accessor". Wall-clock claims are separate: they are P99 on a release export on RM-DESKTOP and RM-DECK (delegated ruling R12), a nightly benchmark failing at +25% over the S1 baseline per machine (TR-shipbody-046), never a per-commit gate; the per-commit performance gates are deterministic counts (`CNT-04`, and instrumented `collide`/`section_at` call counts).

### 8. Test framework, lanes and CI order

**8.1 Pin.** **gdUnit4 6.2.2**, committed under `addons/gdUnit4/` (the directory does not exist in the repository today; `tests/README.md` step 5 and the CI `version: installed` setting depend on it). Basis: `current-best-practices.md` records 6.2.2 running on 4.7.2 with exit codes 0, 100, 101, 105 observed on 2026-10-17; the gdUnit4 README lists Godot to 4.7.1 only, and the release page, as fetched on 2026-10-28, does not mention 4.7 at all. The pin is therefore "observed, not declared", and LINT-SPIKE-1(g) re-observes it from the committed add-on. Upgrading the add-on is an ADR amendment with the spike repeated. Lane H runs exactly the `commands.test` string of `project.yaml`, which carries both `--ignoreHeadlessMode` and `--remote-debug tcp://127.0.0.1:0` (the two expected `ERROR:` lines it provokes are allow-listed in § 8.6). Port doubles are hand-written classes, not gdUnit4 mocks (§ 3.5). The add-on is excluded from the warning scope (`directory_rules`) and from release exports (`exclude_filter addons/gdUnit4/*`, ADR-0007).

**8.2 Lanes.**

| Lane | Where | What | Gate |
|---|---|---|---|
| **H** headless | GitHub-hosted runner, standard 4.7.2 build | `tests/unit`, `tests/integration`; lint CLI; load-all; content lint | blocking on push to `main` and every PR |
| **W** windowed | a runner with a real window and GPU matching RM-DESKTOP; a nightly job on a Steam Deck / RM-DECK-class device for the render-tagged subset | `tests/render/`: pixel, frame-diff, checkerboard, draw-call and flash-scan checks (TR-pxl-037, TR-percfilter-033, TR-debug-045, TR-ftrail-033 rendered criteria, TR-charttable-038 capture checks) | blocking on merge to `main` once the runner exists (open question); until then the lane report states "not run" |
| **P** benchmark | exported release harness on RM-DESKTOP and RM-DECK | P99 timings for S1, ADR-0012/0014/0015/0017 | nightly, +25% over baseline fails |

Render suites live only under `tests/render/` and are **never given to lane H**, so a headless job cannot pass them. `tools/ci/lane_report.gd` lists every suite under `tests/render/` in the lane-H summary as `SKIPPED (windowed lane only)`, so a skipped check is visible and never reads as a pass (TR-percfilter-033, TR-debug-045). Each rendered test additionally calls `RenderGuard.require_nonzero(draw_count_or_pixels)` from `tests/support/`, which fails when the capture is blank or the draw count reads 0, so it cannot pass vacuously. The headless runner covers pure functions, scene-tree walks, `project.godot` parsing and lints only (TR-pxl-037).

**8.3 Exit codes and orphans.** gdUnit4 exits 0 pass, 100 failed, 101 passed but leaked nodes, 103 refused to run headless, 104 Godot too old, 105 a test script does not parse. **101 counts as failure** (TR-pc-045): the CI step runs the command in `project.yaml` (`commands.test`) and fails on any non-zero exit. `gdUnit4-action@v1` already used by `.github/workflows/tests.yml` may or may not fail on 101 (UNVERIFIED, LINT-SPIKE-1(h)); if it does not, the workflow invokes `GdUnitCmdTool.gd` directly with an explicit exit-code check. "No test cases found" with exit 0 is a failure for any suite directory that is expected to hold tests (`tools/ci/lane_report.gd` counts).

**8.4 CI order (lane H).** (1) `godot --headless --path . --import`, **retried up to three times** on a non-zero exit (the fresh-clone crash, 3 of 17 runs, exit 139, passes on repeat; if crashes are independent, two consecutive ones occur in about 3 percent of runs, roughly one PR in thirty, and three in about 0.5 percent), **or** run with `.godot/` cached in CI keyed on the engine version plus a hash of `project.godot` and `addons/` so that the crash path is rare; (2) `tools/ci/load_all.gd` (`PRJ-03`); (3) `tools/lint/lint_main.gd` (all packs, E2 and E3); (4) `tools/ci/content_lint.gd` (ADR-0007) with the base-revision steps; (5) gdUnit4 over `tests/unit` and `tests/integration` (E3, E4); (6) `lane_report.gd`. Steps 2 and 3 precede the suite so a parse or layering error is reported before a flood of test failures, and so TR-assembly-044's "runs in CI before the gdUnit4 suite" holds. The derelict acceptance suite (about 50 integration tests) must finish within 5 minutes with zero orphan nodes and zero leaked RefCounted handles (TR-derelict-032); its budget is part of lane H.

**8.5 Fixtures.** Seeded vector files, golden traces and corpora live under `tests/fixtures/<system>/` (TR-pc-045, TR-suit-037). A pinned fixture is committed with its SHA-256 in `tests/fixtures/<system>/fixtures.sha256`; `tools/ci/` verifies the hashes before any assertion runs, so a changed fixture fails as "fixture changed" and is never regenerated in CI (TR-suit-037).

**8.6 Engine log output in lane H.** The job fails on any `SCRIPT ERROR:` line (load errors) and on any `ERROR:` or `WARNING:` line **not** matched by `tests/fixtures/log_allowlist.json`, whose entries are `{suite, message_regex, reason, owner}` and apply only while the named suite runs. Initial entries: the two `ERROR:` lines about the remote port `127.0.0.1:0` (global, documented in `tests/README.md`); the `assert` line of the `IntMath` divisor-guard fixtures (suite `int_math_guard`, § 5.4, LINT-SPIKE-1(j)); the `bytes_to_var` errors provoked by ADR-0006's corrupt-input fuzz suite; the `~Thread` WARNING and ObjectDB leak report from the quit-during-write test (ADR-0006 OQ-G). An entry with no suite, or one that matches more than 20 distinct lines, fails review. This keeps expected engine noise from hiding a real engine error in an unrelated suite; whether gdUnit4 6.2.2 exposes per-test log capture to implement it more tightly is UNVERIFIED (LINT-SPIKE-1(g)); the fallback is the whole-log scan above.

### 9. GDExtension escape hatch (OQ-10)

**9.1 Default.** The game ships **GDScript only** through MVP. Every performance budget is measured on a release export (R12) and "pending spike" until S1 reports. The ladder before native code is: **rung 0** as written; **rung 1** tuning inside the GDScript design (caps, divisors, data layout: structure-of-arrays, preallocated buffers, early-outs); **rung 2** call-overhead removal (caching accessors in locals, folding a `Callable.call` or a port dispatch out of an inner loop, merging passes; S1 measures "accessor and `Callable.call` overhead per ms"); **rung 3** a GDExtension. Rung 3 is allowed only for the Life-Support Graph's propagate phases (architecture: "Graph (rung 3, VS-A only)"), unless an ADR amendment names another pure function.

**9.2 Trigger.** Rung 3 is entered only when S1 / SP-ENG-1 on RM-DECK (the slower reference) shows the stub-Graph P99 over its budget share **after** rung 1 and rung 2 are applied and recorded, and the technical-director accepts an amendment to this ADR (the `godot-gdextension-specialist` listed in `project.yaml` reviews it). It is never decided inside a story.

**9.3 Constraints on any native module.**
- **Same seam, same behaviour.** The port stays a GDScript `@abstract` class (for example `GraphPropagator`). A GDExtension class can extend only a native (ClassDB) class, never a GDScript class, so a native object is not `is GraphPropagator` and a `setup(p: GraphPropagator)` parameter would reject it. The extension therefore registers a plain `RefCounted`-derived native class (for example `GraphPropagatorNative`) with the same method set, and a thin GDScript adapter `NativeGraphPropagator extends GraphPropagator` owns the native object and forwards each call; the composition root injects the adapter, so call sites do not change. A `ContractCheck` fixture compares the native class's `ClassDB.class_get_method_list` with the port's `get_script_method_list()`. The GDScript implementation is kept, is always built, remains the **golden reference** and the fallback, and the full test suite plus the replay `state_hash` comparison (ADR-0005) run against **both** implementations in CI; a divergence fails the build.
- **Integer-only, tree-free API.** Inputs and outputs are scalars and `PackedInt32Array`/`PackedInt64Array`; no `Node`, no engine singletons beyond array allocation, no threads, no floats, no RNG, no clock, no file I/O. C++ is compiled with `-fwrapv` (signed overflow wraps like GDScript's int64) and fixed-width integer types; no `-ffast-math`.
- **No threads in the module.** Threads exist only in `src/save/` (`IO-02`); the only cross-thread payload anywhere is a `PackedByteArray` or `PackedInt64Array` handed over by value and never mutated by either side afterwards.
- **Platforms.** Windows x86_64 and Linux x86_64 (Steam Deck), built in CI as artifacts, not committed. Sources under `native/src/`, binaries under `native/bin/`, `native/<module>.gdextension` with `compatibility_minimum` set to 4.7. The matching `godot-cpp` branch for 4.7 is UNVERIFIED. Build system: SCons (the project's stated build system).
- **Selection** is made by the composition root (it constructs one implementation and injects it), not by a global flag read inside the simulation. A missing library in a development build falls back to GDScript and logs; a release export check asserts the library is packaged.
- **Not allowed:** C# (the mono editor is not the shipping runtime; CI and export templates are standard builds), GDExtension for presentation or `Node`-bound code, and any native code before the S1 evidence.

### Key Interfaces

```gdscript
# src/kernel/ports/clock.gd   (a port; implementation src/game/monotonic_clock.gd; double tests/support/fake_clock.gd)
@abstract
class_name Clock
extends RefCounted
## Monotonic microseconds for time-dependent rules outside the tick (Pipeline, Controller). Never wall time.
@abstract func now_us() -> int

# tests/support/fake_clock.gd
class_name FakeClock
extends Clock
var _t: int = 0
func now_us() -> int:             # restate the signature including -> T: visible, and comparable by REFLECT-SIG (4.7 override rule)
	return _t
func advance(us: int) -> void:
	_t += us

# src/save/wall_clock.gd   (a port; implementation src/game/system_wall_clock.gd; injected only into SaveService.setup())
@abstract
class_name WallClock
extends RefCounted
## Display-metadata wall time, seconds since the Unix epoch. Never enters a block, a hash or an ordering decision.
@abstract func now_unix() -> int

# injection pattern: member fields are _-prefixed and the setup parameter is the bare name (no SHADOWED_VARIABLE)
var _clock: Clock
func setup(clock: Clock) -> void:
	_clock = clock

# src/shared/int_math.gd          (the only integer division in src/; ADR-0005 owns the function list and the divisor guard)
# IntMath.trunc_div(a: int, b: int) -> int carries the single @warning_ignore("integer_division") (PRJ-04, SIM-05).
# floor_div, posmod_i, ceil_div, lsr and low32 are ADR-0005's signatures and are not restated here.

# tools/lint/core/lint_rule.gd
class_name LintRule
extends RefCounted
var id: StringName                # e.g. &"SIM-04"
var pack: StringName
var severity: int                 # 2 error, 1 warn
var waivable: bool
func check_file(_file: LintFile, _report: LintReport) -> void:
	return
# LintReport.add(rule_id: StringName, path: String, line: int, message: String) -> void

# tools/lint/core/contract_check.gd
class_name ContractCheck
extends RefCounted
static func diff(_real: Script, _double: Script) -> Array[String]:   # [] when name, each arg (type, class_name, hint_string), defaults, return (type, class_name, hint_string) and static/vararg flags all match
	return []

# Hot-function tag (a doc comment the scanner reads; no runtime effect)
## @hot
func solid_in(_rect: Rect2i, _out: QueryBuffer) -> int:
	return 0
```

The bodies of the last two functions (`LintRule`, `ContractCheck`) are placeholders in this document; the signatures are binding. The `IntMath` signatures are ADR-0005's.

### Implementation Guidelines

- **Must** place every script in the directory § 1.1 assigns and name it per § 1.2; **must never** create a top-level directory under `src/` or a module directory not listed in `modules.json`, and **must never** put sim code at `src/sanity/` or any path outside `src/sim/<system>/`.
- **Must** reach another module through its `class_name`; **must never** `preload`, `load` or `ResourceLoader` a `.gd` path in `src/`, and **must never** let `src/data/`, `src/save/` or `src/kernel/` name a `src/sim/` class.
- **Must** let presentation name only `sim/<x>/views/**`, `sim/<x>/ports/**` and the `kernel/facets/**` list of § 1.2; **must never** give a presentation, debug or observer file `Kernel`, `SimViews`, `WriteScope`, `Command`, `BlockDescriptor`, `CallContext`, `ValidateContext`, `ApplyContext`, `HookContext` or `SnapshotContext`, and **must never** let presentation name `NotifyFacet` (it holds the class-bound `PresentationFacet`).
- **Must** express an injected seam as an `@abstract` port in the provider's `ports/` directory, restate the full signature in implementations and doubles, and take the port type in `setup()`; **must never** write `IFoo` names, an abstract `BlockOwner`, or a double that merely has matching method names without extending the port (use `ContractCheck` for the one case where it cannot).
- **Must** keep `src/shared`, `src/kernel`, `src/data`, `src/save`, `src/sim/**` free of the SIM-01 token list; **must never** attach a script from those directories to a scene, add an autoload, or declare a `static var`.
- **Must** give every `var` and `const` an explicit type, write lambdas fully typed, prefix member fields with `_` and take the bare name in `setup()`, and name accessors `get_<noun>` or `lookup`; **must never** use `:=`, define a method named `get`/`set`/`free`/`call`/`connect`, or reuse a Node property name for a local.
- **Must** inject `WallClock` only into `SaveService.setup()` and take presentation time from `delta`, `present_time` or the injected `Clock`; **must never** place a wall-clock value in a block, a hash or an ordering decision, or write a `Time.*` token outside the four-entry allow-list.
- **Must** return `Array[T]` or `Dictionary[K, V]` marked read-only from accessors; **must never** expose a `Packed*Array` from an accessor or give a `Packed*Array` property a `set(` accessor in `src/`.
- **Must** return a subscriber handler's `CompletionToken` synchronously; **must never** `await` in a presenter or subscriber script (`PRES-06`).
- **Must** keep the MVP-minimum rule set (§ 6.10) green from KRN-LAT-02; **must never** describe a deferred rule as enforced before its owner's story has landed it.
- **Must** put the PROJECTOR scripts in `src/presentation/projector/`, and `KernelHost` and `SaveHost` in `src/game/`.
- **Must** use the `IntMath` helpers (ADR-0005's list) for integer division and modulo in `shared`, `kernel`, `sim`; **must never** write the tokens `/`, `%`, `/=`, `%=`, `min`/`max`/`clamp`/`abs`/`sign`/`floor`/`ceil`/`round`, a float, a float-producing member (`.length(`, `.normalized(`, ...), or add a second `@warning_ignore("integer_division")` there.
- **Must** keep all `debug/gdscript/warnings/*` at 2 (RETURN_VALUE_DISCARDED at 0) and the `directory_rules` of § 5.1; **must never** add an `@warning_ignore` form that is not an entry of `warning_ignore.json` (`PRJ-04`); **must never** leave a warning at 1 or add a path to `directory_rules` with value 0 under `src/`.
- **Must** add the lint rule (and its bad/good fixtures) for a new constraint in the same story that introduces the code it constrains; **must never** state a rule as "lint-enforced" in a story without naming its rule id and tier.
- **Must** tag allocation-sensitive functions `## @hot`, keep them free of the HOT-01 tokens, and add the `CNT-01` to `CNT-03` assertions to the suite that exercises them; **must never** claim "zero allocation" without those checks.
- **Must** keep test seams in `src/<module>/testing/` classes named `Test*` and inject `WorkCounter`, `Clock` and similar through ports; **must never** read `OS.has_feature` or a compile flag to switch sim behaviour.
- **Must** put doubles and fakes in `tests/support/` and render-only suites in `tests/render/`; **must never** let a render-tagged check run in lane H or pass with a zero draw count.
- **Must** commit `addons/gdUnit4/` at the pinned version and treat exit 101 as failure; **must never** change the pin without repeating LINT-SPIKE-1(g).
- **Must** record any new waiver or allow-list entry with a reason and an owner; **must never** waive an NW rule.
- **Must** keep the GDScript implementation of any natively accelerated module as the golden reference; **must never** enter GDExtension rung 3 without the S1 evidence and an accepted amendment, and **must never** use C#.

## Alternatives Considered

### Alternative 1: `gdtoolkit` (`gdlint`/`gdformat`, Python) as the lint tool
- **Description**: Run `gdlint` with custom Python checks over its parse tree in CI.
- **Pros**: A real parse tree (lark grammar) for structural rules; mature formatter; no code to write for style rules.
- **Cons**: Latest release 4.5.0 (Oct 2025) documents no `@abstract`, variadic or 4.6/4.7 support (UNVERIFIED either way); a second toolchain (Python, pinned versions) beside the Godot binary; no reflection, no class map, no `ProjectSettings` access, so E3 rules would need a second tool anyway; cannot share `ContentLoader` with ADR-0007's lint.
- **Rejection Reason**: Correctness of blocking rules cannot depend on a parser whose 4.7 grammar coverage is unknown. It stays available for advisory formatting; a later ADR may adopt it for E2 rules after a grammar probe.

### Alternative 2: Engine warnings and tests only, no lexical tool
- **Description**: Rely on warnings-as-errors, `@abstract`, and gdUnit4 tests for every rule.
- **Pros**: Zero tool code.
- **Cons**: The engine does not see tokens (`Time.get_ticks_usec`, `_physics_process`, `preload` of a sibling), import edges, `.tscn` priorities, or `%` on integers; tests only find a violation on a code path they execute.
- **Rejection Reason**: Roughly half of the inherited rules are lexical by nature (§ 6.6). The lexical tier is small and fixture-tested.

### Alternative 3: A hand-written GDScript parser (real AST)
- **Description**: Write a GDScript parser in GDScript to decide type-aware rules (who writes a field, what is a float).
- **Pros**: Could express field-writer and scope rules exactly.
- **Cons**: A large, maintained component tracking a language that changed in 4.5, 4.6 and 4.7; type-flow rules duplicate the engine's analyzer, which is already authoritative and free.
- **Rejection Reason**: The engine analyzer (E1) covers type flow; reflection (E3) covers shape; runtime (E4) covers dynamic facts. What remains is honestly weaker (file/function granularity) and is documented as such.

### Alternative 4: Duck-typed contracts checked by a boot-time registry lint
- **Description**: No base classes; each seam is a method-name list; a boot-time registry compares every injected object with the list via `has_method`.
- **Pros**: No dependence on `@abstract`; doubles are just objects.
- **Cons**: A missing method is found at boot or first call, not at parse time; checks only method names unless extended with reflection; `has_method` strings drift silently; no editor completion.
- **Rejection Reason**: `@abstract` gives a parse-time error on 4.7.2 (P4) for the same cost. The duck-typed form survives only as `ContractCheck` for recorded stubs.

### Alternative 5: Two strictness tiers (sim strict, presentation lax) by warning severity
- **Description**: BDY-3's original idea: error severity for the Body and Layout directories only; filter CI output by path.
- **Pros**: Presentation authors face fewer errors.
- **Cons**: Severity is global per warning and directories only opt in or out (P2); a per-path tier needs severity 1, which is invisible headless (P10), so CI would need a second parse pipeline; presentation code that mixes ints and floats is exactly where an accidental `/` creeps into a sim-visible value (draw positions feed `world_to_hud`, aim, mouse).
- **Rejection Reason**: One tier for `src/` and `tools/` is simpler and fully engine-enforced; the extra annotation burden in presentation is small (typed locals, `IntMath`).

### Alternative 6: Plain base class whose methods `push_error`
- **Description**: The architecture's stated fallback.
- **Pros**: Works even if `@abstract` misbehaves in export.
- **Cons**: A missing override is a runtime error on first call; no parse-time guarantee.
- **Rejection Reason**: Kept as the documented fallback (§ 3.4) behind the same call sites, used only if LINT-SPIKE-1(a) fails.

### Alternative 7: Flat vertical-slice layout (`src/<feature>/{sim,presentation}`)
- **Description**: One directory per feature holding both its simulation and presentation code.
- **Pros**: Everything for a feature in one place.
- **Cons**: The most important import rule (simulation never reads presentation) stops being a directory test; per-directory tree-freedom, float-freedom and warning scope all become per-file; the architecture's layer table and the GDDs' paths already use `src/sim/<system>/`.
- **Rejection Reason**: A layer-first tree makes the architecture's central rules mechanical.

### Alternative 8: C# or GDExtension-first for performance
- **Description**: Write the Graph and other hot code natively from the start.
- **Pros**: Predictable speed.
- **Cons**: No measurement justifies it (S1 pending); C# is outside the project's language and the shipping builds; native code before the design is stable multiplies change cost; determinism parity must be proven against a reference.
- **Rejection Reason**: Evidence-gated ladder (§ 9): GDScript first, native only on S1 evidence, GDScript kept as the reference.

## Consequences

### Positive
- "Simulation never reads presentation", "no `preload` of a sibling", "no wall-clock in the sim" and "no autoload" become build failures instead of review comments.
- The engine enforces typing and integer discipline on every script the game runs, with no tool code; BDY-3's unverified directory scope is resolved.
- A missing method in a test double or an implementation is a parse error, not a runtime surprise.
- Every "lint-enforced" requirement now names a rule id, a tier and its blind spots; OQ-16 is closed for its wording.
- Allocation claims have checks that exist, and the document says what each cannot see.
- One CI order puts parse and layering errors before test noise, and render checks cannot silently pass in a headless job.
- Native code has a written gate and a parity requirement, so it cannot arrive as a convenience.

### Negative
- Everything under `src/` and `tools/` must be fully typed: no `:=`, no untyped locals, no `Variant` shortcuts. Authoring is slower, and engine-patch warning additions can break the build until classified (`PRJ-01`).
- The lexical tier is heuristic: it works at file and function granularity, cannot see types, and a stripped-string or continuation edge case is a tool bug to fix. Field-writer rules are weaker than a real AST would make them.
- Level-1 warnings are unusable headless, so the project loses the "warn but pass" middle state.
- `tests/` is excluded from the warning scope; test code is untyped-tolerant by design.
- Layering costs a `views/`/`ports/`/`facets/` convention that every module author must follow, and a module that needs a second-sim-module internal must go through its published surface.
- The windowed lane depends on hardware (a GPU runner, a Deck) the project may not have wired up.
- The rule inventory is large for one programmer; § 6.10 limits what is required at KRN-LAT-02, and `tools/` itself must be fully typed while it parses JSON (confined to one `LintConfig` class).
- A banned operator token (`/`, `%`) is rejected even on non-integer operands in `shared`, `kernel` and `sim`; that is deliberate (a lexical scan cannot know the type) and the cost is routing division through `IntMath`.

## Risks
- **`@abstract` does not survive a release export, or reflective construction creeps in.** Mitigation: LINT-SPIKE-1(a); fallback § 3.4 behind unchanged call sites; no game code constructs via `GDScript.new()`.
- **Warnings-as-errors makes a patch release unbuildable** (a new default-1 warning). Mitigation: `PRJ-01` names the unclassified warning; CI pins the exact engine version.
- **`directory_rules` behaves differently in an export or a later patch.** Mitigation: it is a project setting probed on 4.7.2; warning promotion is verified on the editor binary (and optionally `template_debug`) only, because a `template_release` export is not a warning gate (UNVERIFIED); `@abstract` and the Performance monitors are checked on the release export by LINT-SPIKE-1(a) and (c); `load_all` runs on the exact CI binary.
- **Lexical false negatives** (a banned token built dynamically, `call("get_tree")`). Mitigation: dynamic dispatch by string is itself flagged in `shared`, `kernel`, `sim` (`SIM-12`); the runtime tier catches what reaches state.
- **Lexical false positives stall work.** Mitigation: per-line waivers with reasons, NW only for the few non-negotiable rules, fixtures for edge cases, a waiver baseline.
- **A rule is added by an owner ADR without fixtures or a tier.** Mitigation: `FIX-01` fails the suite; the inventory table is the checklist.
- **`MEMORY_STATIC` noise or release unavailability makes `CNT-02` flaky.** Mitigation: tolerance tuned by LINT-SPIKE-1; debug-binary-only; never a release gate; the other three tiers do not depend on it.
- **No windowed runner exists.** Mitigation: the lane report states "not run"; render ACs stay open rather than passing; the open question names the decision owner.
- **gdUnit4 6.2.2 is observed on 4.7.2 but not declared.** Mitigation: committed add-on, spike re-observation, upgrade only by amendment.
- **Presentation `views/` convention is violated because a read view's class lives inside a sim module's internals.** Mitigation: `ARCH-02` fails it; ADR-0009 and each module ADR place view classes in `views/`.
- **The `--import` crash fails PRs for unrelated reasons.** Mitigation: retry up to three times or cache `.godot/` (§ 8.4); LINT-SPIKE-1(f) measures it.
- **A lexical rule misreads a GDScript string, indentation or lambda form.** Mitigation: one fixture per form (§ 6.3), a corpus run over every committed `.gd` file, and "unhandled construct = exit 2".
- **A stale `global_script_class_cache.cfg` hides a violating edge.** Mitigation: the class map is built twice and a disagreement is exit 2 (§ 2.1).
- **The 4.7 packed-array element write skips a setter and a dirty key is missed.** Mitigation: `ARCH-11`, and `CNT-03` forbids exposing packed arrays.

## GDD Requirements Addressed

Requirement text follows `docs/architecture/tr-baseline.md` and `requirements-traceability.md`. The six rows marked **baseline** are those whose last column cites ADR-0010 in `tr-baseline.md`; the remainder are the rows `requirements-traceability.md` (the `/architecture-review` completeness pass) assigns to ADR-0010.

### Primary: this ADR decides the rule or the mechanism

| GDD System | Requirement | How This ADR Addresses It |
|------------|-------------|--------------------------|
| command-simulation-layer.md | **TR-cmd-033 (baseline):** kernel scene-tree independent (headless, synchronous `Kernel.step()`), `KernelProfile` injected and never read from `OS.is_debug_build()`, no networking code at MVP (CR-14 H8/H10, CR-19(e)) | § 4.1 `SIM-01` (L0 to L2 tree-free, `KernelHost` in `src/game/`), § 4.2 headless harness and injected profile, `SIM-08` bans networking symbols |
| command-simulation-layer.md | TR-cmd-050: test seams (`TestIngress.submit_raw`, `debug_set_counters`, stub timer, `set_knob`) excluded from release exports and verified by an export file-list scan | § 4.3 `testing/` directories, `ARCH-09`, the single `exclude_filter` glob `src/*/testing/*`, scratch-directory pack file-list scan (UNVERIFIED, LINT-SPIKE-1(i)) |
| command-simulation-layer.md | TR-cmd-051: kernel structure lint-enforced in blocking gdUnit4 tests under `tests/unit/kernel/lint/` (member allow-list, no per-verb command classes, no kernel-registered domain types, no networking symbols, no presentation references from sim) | § 6.8 one library with CLI and gdUnit4 wrappers, `KRN-01`, `SIM-08`, `ARCH-02`, `ARCH-08` |
| command-simulation-layer.md | TR-cmd-052: multiple kernel instances in one process, `BLOCK_ALREADY_BOUND`, idempotent `dispose()` with zero leaked RefCounted/orphans | § 4.2, `ARCH-04` (no `static var`), `CNT-01` |
| item-and-module-database.md | TR-itemdb-035: CI lint suite in `tools/ci/item_db_lints` with its own unit tests, BLOCKING/ADVISORY severity per rule | § 1.3, § 6.1: content rule packs stay under `tools/ci/` with ADR-0007's `content_lint.gd`; code rules use the same finding format and severity column; shared fixtures layout § 8.5 |
| pixel-perfect-render-pipeline-camera.md | TR-pxl-031: time-dependent rules read an injected monotonic `Clock.now_us()`, never wall time or render delta | § 3, `Clock` port in Key Interfaces, `MonotonicClock` in `src/game/`, `SIM-02` allow-list (§ 4.5; `WallClock` is a separate Save/Load-only port), `FakeClock` in `tests/support/` |
| pixel-perfect-render-pipeline-camera.md | TR-pxl-037: pixel-level verification needs a windowed runner; headless covers pure functions, tree walks, `project.godot` parsing and lints only | § 8.2 lane W, render suites only under `tests/render/`, lane report prints `SKIPPED` |
| emitter-state-and-light-exposure-query.md | TR-emitter-031: boot invariants are functions returning a fault code, never `assert()`, refused identically in Release, proven by a Release `--quit-after` smoke | Lint half: `SIM-11` bans `assert(` in `src/` outside `testing/` so release-stripped checks cannot carry boot invariants; the code-return convention is ADR-0007 § 2.3; the smoke is ADR-0007's exported-build check run in lane H |
| emitter-state-and-light-exposure-query.md | TR-emitter-032: the emitter runs headless against injected seams only (FakeClock, Layout, Graph, Body, Controller, Suit, Diagnostics, CommandBus, WorkCounter); the work counter is compiled in only under EM_WORK_COUNTER | § 3 ports, § 4.4: counter realised by injection (`NullWorkCounter` in shipped builds), not conditional compilation; `CNT-04` |
| emitter-state-and-light-exposure-query.md | TR-emitter-034: no bare integer `/` or `%` in `src/sim/emitter/`; checked helpers `floor_div`, `posmod_i`; grep lint with committed bad/good fixture per banned token | § 5.4 `IntMath`, `SIM-04` (operator tokens `/`, `%`, `/=`, `%=` banned outright), `SIM-05`, `PRJ-04`, `FIX-01` fixtures; scope widened to the whole simulation |
| player-controller-input.md | TR-pc-045: gdUnit4 headless harness with exit 101 counting as failure; seeded fixtures under `tests/fixtures/controller/`; recorded contract fixtures under `tests/fixtures/contracts/` that every stub must pass | § 8.3 exit-101 policy, § 8.5 fixtures, § 6.4 `ContractCheck` and `gen_contracts.gd` |
| failure-and-trail-system.md | **TR-ftrail-033 (baseline):** resolver unit tests run headless under `tests/unit/failure_and_trail/` against data-file fixtures; rendered criteria on a windowed `ft-render` job on RM-DESKTOP and RM-DECK (FT-40) | § 8.2 lanes H and W (`ft-render` is a named job of lane W), § 8.5 fixtures; `tests/unit/failure_and_trail/` follows § 1.2 naming |
| suit-and-equipment.md | **TR-suit-006 (baseline):** hook, mutators and block contain no `Node`; scene-tree-free (SE-37, H10) | `SIM-01` over `src/sim/suit/`, plus the `SIM-10` block-shape check; wrapper test `tests/unit/suit/lint/` |
| suit-and-equipment.md | TR-suit-033: boundary rules enforced by CI lints on source and data (no band-constant literals, no retired tokens, Controller reads only `locomotion_cap` and `stowed_heavy`, no Sanity/Attention/`PERCEPTION_OUT` symbol in Suit, HUD reads only via `true_value`, `withdraw_store` only inside V7 apply, non-negative dividends) | § 6.6 `READ-01` to `READ-03` generic mechanisms with Suit's token lists as the first instance; function-granularity limit stated in § 6.3; non-negative dividends via `SIM-04` and `IntMath` |
| suit-and-equipment.md | TR-suit-037: Graph fixture traces, demand corpora and golden integrator files pinned by SHA-256, never regenerated in CI | § 8.5 `fixtures.sha256` verified before assertions |
| ship-section-and-layout-data-model.md | **TR-section-002 (baseline):** Layout data module scene-tree free and integer-only; a headless test asserts it references no `Node` and no float exists (LY-3 a, LY-31) | `SIM-01`, `SIM-03`, `SIM-10` over `src/sim/layout/`; PROJECTOR moved to `src/presentation/projector/` (§ 1.2) so the directory has no exception; wrapper `tests/unit/layout/lint/` |
| ship-section-and-layout-data-model.md | TR-section-043: engine behaviours probed by SP-ENG-1 on a 4.7.2 release export before dependent ADRs are accepted | Probes P4 to P11 on the editor binary are recorded; the release-export repeat is LINT-SPIKE-1 and S1; JSON number typing and packed-array writes are ADR-0005/0007 |
| ship-body-and-interior-physics.md | **TR-shipbody-002 (baseline):** Body-frame state integer-only; Body and Layout promote typing warnings to errors; exactly one annotated integer `/` site inside `trunc_div` (BDY-3, BDY-35) | § 5.1 `directory_rules` (scope exists, P2/P3), § 5.4 `IntMath.trunc_div` as the only annotated site (`SIM-05`), P11 fixtures; scope widened from Body and Layout to all of `src/` |
| ship-body-and-interior-physics.md | TR-shipbody-038: integer-semantics assumptions (`posmod`, truncation, float32 canvas offsets, `OBJECT_COUNT`, directory-scoped warning levels) unverified on 4.7.2, gated by spike S2 | Probed on the editor binary: truncation, `posmod`, `%`, `OBJECT_COUNT` (P6, P11) and directory scope (P2, P3). The canvas coordinate rule is ADR-0008 § 6 (presentation coordinates are integer-valued float32, exact for abs(v) <= 2^24 base px; `Vector2i` converts to `Vector2` only at `Placement.put` and the `canvas_transform` write; ADR-0015 owns the bound if body-frame extents could exceed 2^24 base px); release-export repeat is LINT-SPIKE-1 |
| modular-ship-assembly.md | TR-assembly-044: single headless lint harness (`tools/lint/msa_lint.gd` with allow-list files) in CI before the gdUnit4 suite, one `LINT-MSA-n` line per finding, non-zero exit, covers source and data under `assets/data/` | § 6.1 pack file `tools/lint/msa_lint.gd`, § 6.2 finding format and exit codes, § 8.4 CI order; the data half reads `assets/data/` through ADR-0007's `FileContentSource` |
| derelict-authoring-handmade-kits.md | **TR-derelict-026 (baseline):** content acceptance needing the Graph runs as headless gdUnit4 integration tests under `tests/integration/derelict_authoring/` via the kernel headless path; systems stay scene-tree free (DA-1, DA-42) | § 4.2 headless harness, § 8.2 lane H, `SIM-01` |
| derelict-authoring-handmade-kits.md | TR-derelict-032: headless acceptance suite (about 50 tests) at most 5 minutes in CI, zero orphan nodes and zero leaked RefCounted handles | § 8.3 exit 101 failure, § 8.4 budget, `CNT-01` |
| boarding-and-salvage.md | TR-boarding-030: headless scripted harness with a Web Map stub; golden schedules generated from the owners' real code, never typed in tests | § 4.2 harness, § 8.5 pinned fixtures; the schedules are read from owner constants (a `READ-03` rule forbids restating the constants in `tests/`) |
| boarding-and-salvage.md | TR-boarding-031: every hook, command, mutator, read-view and notification identifier published in a machine-readable list so a script generates the lint allow-lists | § 6.4 generated `build/lint/published.json` from the kernel registry dump and public-method reflection |
| web-map-and-transit.md | TR-webmap-031: scripts integer-only, scene-tree-free: no float, random or wall-clock call, no Node/SceneTree, non-negative divisors, 8-byte seed never stored as a GDScript int, no `rng_derive` stream registered | `SIM-01` to `SIM-04` cover tree, clock/RNG, float and division; the seed and stream rules are ADR-0005's (see Owned elsewhere) |
| attention-system.md | TR-attention-037: Attention testable headless; `emitters()`, `power_use` and the beat-jitter provider injected | § 3 ports, § 4.2 harness, `tests/support/` stubs |
| horror-entities.md | TR-horror-042: test harness injects every neighbour read through stubs with a stub-or-real switch per read; replay runner returns `state_hash` every 600 ticks | § 3 ports, § 6.4 `ContractCheck` (stub must match real), § 4.2 harness; `state_hash` is ADR-0005's |
| perception-filter.md | TR-percfilter-033: render-tagged checks report SKIPPED, never PASS, when the baseline capture is blank or the draw count reads 0 | § 8.2 `RenderGuard.require_nonzero`, `lane_report.gd` SKIPPED listing |
| chart-table-web-map-and-menus.md | TR-charttable-038: neighbours taken by injection; per-frame draw list, forced-`H`, frame-capture and `force_stage` hooks as data; contract tests keeping doubles schema-equal; render-capture on a pinned non-headless runner | § 3 ports, § 6.4 `ContractCheck` against recorded fixtures, § 8.2 lane W |
| debug-overlay-true-values.md | TR-debug-038: CI lints enforce reader allow-lists (`sanity.ledger`, raw attention level and `tell_remaining`, `true_value`), none routed to a PRESENTATION-class subscriber | § 6.6 `READ-01` mechanism; the owner ADRs (0022, 0024, 0031) supply the lists |
| debug-overlay-true-values.md | TR-debug-045: rendered-pixel tests run only in a named windowed job; headless reports them SKIPPED; each asserts a nonzero draw or pixel count | § 8.2 lane W and `RenderGuard` |

### Owned elsewhere: this ADR supplies only the lint, check, lane or fixture rule

| GDD System | Requirement | What this ADR supplies |
|------------|-------------|------------------------|
| 2d-lighting-and-occlusion.md | TR-light-031 (ADR-0028 + ADR-0010): pure engine-free plan step and apply step; incremental pass equals full reconcile | `SIM-01`-style purity applied to the plan step's directory by an ADR-0028 pack; injectable `Clock` and recorded-trace stub per § 3; the plan/apply split itself is ADR-0028 |
| player-controller-input.md | TR-pc-032 (ADR-0019 + ADR-0010): `Input.*`, `InputMap`, `InputEvent`, `_input`, `_unhandled_input` only in the Adapter's registered files; a lint enforces this | `READ-01` mechanism and the `_input` token list; the Adapter file list is ADR-0019's allow-list entry |
| ship-body-and-interior-physics.md | TR-shipbody-046 (ADR-0015 + ADR-0010): CI performance gates are deterministic counts from instrumented `collide`/`section_at` wrappers; wall-clock P99 nightly at +25% over S1 | § 7 `CNT-04` and § 8.2 lane P; wrappers and bounds are ADR-0015's |
| web-map-and-transit.md | (second row of TR-webmap-031, ADR-0005 + ADR-0010) | seed width and `rng_derive` stream registration are ADR-0005 |
| hud-and-suit-layer.md | TR-hud-031 (ADR-0030 + ADR-0010): HUD reaches neighbours through injectable interfaces, scripted `(T, tick_alpha)` pinner, scene-state hash for a headless lane | § 3 ports and `PresentationClock` double; the hash and pinner are ADR-0030's |
| adaptive-audio-and-tells.md | TR-audio-031 (ADR-0029 + ADR-0010): headless request-capture harness on the dummy audio driver; dev-build-only tap bus absent from release | § 4.3 `testing/` exclusion and `ARCH-09`; the harness is ADR-0029's |
| debug-overlay-true-values.md | TR-debug-037 (ADR-0031 + ADR-0010): observer and dev roots loaded only by a guarded runtime `load()`; no `preload`, scene, resource or `class_name` reference; release has no entry from either root | `ARCH-03` allow-listed guarded load, `ARCH-10`, `ARCH-07`; export gating is ADR-0031 |
| debug-overlay-true-values.md | TR-debug-039 (ADR-0010 + ADR-0003, partial): testability seams on the composition root and kernel (injectable dev-tag provider, subscriber id at mutator entry, injectable frame delta and clock, stubbable Filter/Sanity/Graph, forced `state_hash` per tick, call-spy seams, injected `ObserverSink`) | § 3 ports and § 4.4 injection convention, `Clock` port; the kernel-side seams are ADR-0003's and ADR-0031's |

## Performance Implications
- **CPU (runtime):** None added to release builds: lint rules, `## @hot` tags and `directory_rules` have no run-time cost; boot walks and counters run in dev and test builds only. Full static typing lets the GDScript compiler use typed opcodes (`deprecated-apis.md`: typed variables have "GDScript compiler optimizations"; magnitude UNVERIFIED, S1 measures). A port adds one method dispatch per call; ports sit at coarse seams (§ 3.4), and S1 measures the per-call overhead.
- **CPU (CI):** Load-all, the lint CLI and the lint self-tests are budgeted at no more than 120 s combined on the lane H runner at MVP code size (pending LINT-SPIKE-1(e)); a `--changed-since` mode reduces developer runs.
- **Memory:** None at run time. `CNT-02` is a debug-binary check only.
- **Load time:** None in release. Dev builds add the boot walks (milliseconds, unmeasured).
- **Network:** None; no networking code at MVP (`SIM-08`).

## Migration Plan
The repository today has `src/CLAUDE.md` only, `tests/` scaffolding and `.github/workflows/tests.yml` (commit `d55b50b`), no `addons/`, and no `directory_rules`/warning settings in `project.godot`.
1. **LINT-SPIKE-1 alongside the kernel skeleton (KRN-LAT-02):** create the tree, `IntMath` and its P11 unit test, the `Clock` port and `FakeClock`, and a minimal `tools/lint/` (scanner and the MVP minimum of § 6.10), then run checks (a) to (j) of § Engine Compatibility.
2. **`project.godot`:** write the warning levels and `directory_rules` of § 5.1 and `PRJ-02` invariants; add `warning_levels.json`; run `load_all.gd`.
3. **Commit `addons/gdUnit4/` 6.2.2**; add the CI steps of § 8.4; move to a direct `GdUnitCmdTool` call if the action does not fail on 101.
4. **Rule packs land with their modules:** each module's first story adds its pack entry, fixtures and `tests/unit/<system>/lint/` wrapper.
5. **GDD sync** (§ GDD and architecture sync, below): BDY-3, BDY-35, LY-31 and the Sanity path are corrected by `/propagate-design-change` before the stories that cite them.
6. **Windowed lane:** stand up the runner when RM-DESKTOP access is decided (open question); until then the lane report says "not run".
No source code has to be refactored: nothing exists to migrate.

### GDD and architecture sync
Applied by `/propagate-design-change` before the stories that cite them (this ADR edits none of them):
- `ship-body-and-interior-physics.md`: BDY-3 (the directory scope exists on 4.7.2; the PROJECTOR script lives in `src/presentation/projector/`; widen from Body and Layout to all of `src/`; the clock-token list is `SIM-02`), BDY-35 (the one annotated site is `IntMath.trunc_div` in `src/shared/`).
- `ship-section-and-layout-data-model.md`: LY-31 widens to all of `src/`; the PROJECTOR scripts are in `src/presentation/projector/`.
- `sanity-and-hallucination-rules.md`: the path in AC-SN-30, AC-SN-82 and AC-SN-130 becomes `src/sim/sanity/` (IC-10).
- `emitter-state-and-light-exposure-query.md`: `EM_WORK_COUNTER` becomes an injected `WorkCounter` port; the helper names cite `IntMath`.
- `command-simulation-layer.md`: the KRN lint cites `SIM-01`; `KernelHost` is `src/game/kernel_host.gd`.
- `docs/architecture/architecture.md`: § Spikes that gate ADRs gains LINT-SPIKE-1 (gating ADR-0010); the `@abstract` choice is recorded where the architecture lists the two interface options (about line 559); IC-10, OQ-10 and OQ-16 are marked resolved by this ADR.
- `tests/README.md`: the gdUnit4 download step names v6.2.2.
Sibling ADR alignment (Depends On and Enables edges, paths) is recorded in the Review Log, not here.


## Validation Criteria
- LINT-SPIKE-1 (a) to (j) pass, with the CI binary, on Windows and Linux/Deck for (a) and (c).
- `load_all.gd` fails on a fixture script that violates `UNTYPED_DECLARATION`, `INTEGER_DIVISION`, `UNSAFE_METHOD_ACCESS` and `MISSING_AWAIT` placed under `src/`, and passes on the same scripts under `tests/` and `addons/` (on the editor binary; the release export is not a warning gate).
- `PRJ-01` fails when any warning is set to 1 and when a warning is absent from `warning_levels.json`.
- For every rule in § 6.6 with a lexical or reflection tier: a bad fixture yields a finding with that id, a good fixture yields none, and the suite fails if a rule has neither (`FIX-01`).
- The import-edge matrix test: a fixture tree with one violating edge per forbidden cell reports `ARCH-02` once per cell; a fixture with a `class_name` cycle between two sim modules reports nothing.
- `PRJ-04` fails on any `@warning_ignore`, `@warning_ignore_start` or `@warning_ignore_restore` that is not an entry of `warning_ignore.json`, including a multi-argument form and a second `integer_division` entry; `SIM-05` finds exactly one `integer_division` ignore in `src/`.
- `SIM-04` fails the bad fixtures `Vector2i(5, -3) / 2`, `a /= b`, `a %= b` and `x % 3`; `SIM-03` fails `.normalized(`; `SIM-15` fails `9223372036854775808`.
- `IntMath` unit tests hold the P11 fixtures on the CI binary, plus the `b = 0`, `b < 0` and `INT64_MIN / -1` cases (each returns 0).
- A test double that omits a port method fails to parse; a recorded stub whose parameter drifts from `Rect2i` to `Vector2i`, or whose return drifts from `Array[int]` to `Array[String]`, fails `ContractCheck`; `KRN-01` fails a `Kernel` with one extra field and passes the exact allow-list; `PRES-06` fails a presenter containing `await`.
- Allocation: a deliberately leaking fixture (retains one array per tick) fails `CNT-02` on the debug binary and passes `CNT-01`; a fixture that creates one `RefCounted` per tick fails `CNT-01`; a fixture returning `duplicate()` from a getter fails `CNT-03`; a `## @hot` fixture containing `str(` fails `HOT-01`. These four fixtures prove each tier's blind spot is documented, not assumed.
- Lane H summary lists every `tests/render/` suite as SKIPPED; a render fixture with a zero draw count fails `RenderGuard`.
- Two kernels in one process finish a suite with the same `OBJECT_COUNT` and orphan count as before (TR-cmd-052); a fixture with a deliberate reference cycle is reported as a leak.
- The derelict acceptance suite finishes within 5 minutes with exit code 0, not 101.
- After ADR-0005 is Accepted: a float placed in a block snapshot fails the encoder in the first replay (the runtime half of `SIM-03`).
- If GDExtension rung 3 is ever entered: the GDScript and native implementations produce equal `state_hash` over the replay corpus on both platforms.

## Related
- Depends on: ADR-0001, ADR-0002, ADR-0003, ADR-0007 (all Proposed).
- Enables (these cite this ADR's paths and rule ids and carry ADR-0010 under their own Depends On): ADR-0004 (change notification, dirty-key check), ADR-0005 (determinism, `src/shared/det/`), ADR-0006 (Save/Load, `SaveHost`, `WallClock`), ADR-0008 (stretch pins, windowed lane renderer), ADR-0009 (presentation edges, `PRES-*` rules). Forward references (not yet written): ADR-0019 (`Input.*` allow-list), ADR-0031 (dev-root gating, `observer`/`debug` export exclusion), ADR-0034 (PROJECTOR scene shape inside `src/presentation/projector/`).
- Supersedes nothing. Does not reopen ADR-0003 § 1.
- GDDs: `command-simulation-layer.md` (CR-14, CR-19, KRN criteria), `ship-body-and-interior-physics.md` (BDY-3, BDY-35), `ship-section-and-layout-data-model.md` (LY-3, LY-31), `suit-and-equipment.md` (SE-37), `failure-and-trail-system.md` (FT-40), `derelict-authoring-handmade-kits.md` (DA-1, DA-42), `modular-ship-assembly.md`, `item-and-module-database.md`, `emitter-state-and-light-exposure-query.md`.
- `docs/architecture/architecture.md` § System Layer Map, § Required ADRs, § Spikes that gate ADRs, OQ-10, OQ-16, IC-10.
- `docs/architecture/architecture-review-2026-10-28.md`, `requirements-traceability.md` (ADR-0010 rows).
- `docs/registry/architecture.yaml`: stances this ADR enforces lexically or by reflection (`kernel_autoload_or_global`, `physics_callback_tick_source`, `await_process_frame_in_sim_or_presentation`, `command_constructed_outside_kernel`, `block_as_resource_or_node`, `content_directory_scan`, `content_file_read`, `mutable_or_resource_catalog`, `loader_side_effect_or_foreign_boot_code`, `content_export`, `frame_order`, `simulation_clock`); the remaining `forbidden_patterns` are assigned to other tiers or owners in § 6.6b.
- `docs/engine-reference/godot/current-best-practices.md` § Command Line, Tests and Parse Check.

## Open Questions
- **OQ-A (decision owner: user/producer).** Is there hardware and a runner for the windowed lane (a Windows machine with a GPU equivalent to RM-DESKTOP wired to CI, and a Steam Deck for the nightly)? GitHub-hosted runners have no GPU; a software rasteriser is not representative. Until decided, every render acceptance criterion stays open and the lane report says "not run".
- **OQ-B.** Should `tests/` come into the warning scope later (gdUnit4 test code is likely untyped-heavy, UNVERIFIED)? Excluded now so tests are not blocked by the add-on's API; revisit after the first 100 tests.
- **OQ-M.** ADR-0006 asks how the lint and gdUnit4 tolerate the `~Thread` WARNING, the ObjectDB leak report at exit, and the `bytes_to_var` `ERROR` lines that corrupt-file fuzz tests provoke: see § 8.6.
- **OQ-N (requests to ADR-0005; decision owner ADR-0005).** (1) Add `IntMath.trunc_div_v2i` and `IntMath.floor_div_v2i` (per-component calls to the scalar helpers). (2) Name `sorted_int_keys` and `sorted_text_keys` (§ 5.3) as the two typed forms of `sorted_keys`. (3) State whether `det_rng_float.gd` needs a `/`; the default is multiplication by a constant, because `SIM-04` exempts that file for `float(` and `floor` only.
- **OQ-F.** Whether `gdUnit4-action@v1` fails on exit 101 (LINT-SPIKE-1(h)).
- **OQ-G.** Whether `@abstract` and the `directory_rules` behaviour survive a release export, whether `MEMORY_STATIC` exists there, whether a `.fixture` extension is ignored by the editor, and whether `can_instantiate()` is false for the warning-as-error case. Also whether `Script.is_abstract()` exists, whether `can_instantiate()` is true for a port and for a static-only class, whether `template_release` evaluates promoted warnings, and the `KRN-01` property-list shape. All are LINT-SPIKE-1 items; their negative outcomes have documented fallbacks (§ 3.4, `CNT-02` debug-only, directory rename, log-scan only).
- **OQ-H.** The dictionary and `Variant` allow-lists (`dictionary_variant.json`) depend on the exact payload and snapshot shapes ADR-0002/0003 fix; their initial contents are written when the kernel skeleton exists.
- **OQ-I.** Which functions carry `## @hot` is each performance-bearing ADR's list (0012, 0014, 0015, 0017, 0028); this ADR provides only the mechanism.
- **OQ-J.** A real AST (via a maintained, probed `gdtoolkit` version or a project-written tokenizer) may later tighten field-writer and scope rules; revisit if the lexical tier's false-negative rate proves material at the first Feature-layer module.
- **OQ-K.** Presentation-to-sim access assumes read-view classes live in `sim/<x>/views/` and that a facade the Pipeline root reads through is a `kernel/facets/` entry (`PresentationViews`, § 1.2). ADR-0009 and ADR-0003's `SimViews` facade classes must agree. (The `save` typing question is settled by § 2.2 rule 3.)

## Registry Candidates
Copied from the Foundation run file and corrected by the review rulings (TD-01, TD-04, TD-11, TD-13, TD-14, TD-15, TD-16). The orchestrator merges them into `docs/registry/architecture.yaml` serially after the revisions land; this ADR does not edit the registry. None of these entries exists in the registry today, so there is no `revise_existing` item. `referenced_by_update` lines append this ADR to an existing entry.

```yaml
# --- api_decisions ---
  - purpose: source_layout
    status: active           # ADR-0010 is Proposed
    api: "Layer-first tree: src/shared (L0, imports nothing; scene-tree-free, I/O-free, no global state; IntMath and src/shared/det), src/kernel and src/data and src/save (L1; save is RefCounted only), src/sim/<system> for graph, layout, body, emitter, controller, interaction, suit, assembly, web, attention, horror, sanity (L2), src/presentation/<system> for pipeline, perception, lighting, trail, hud, instruments, chart, audio, controller, projector (L3; shared presentation files in pipeline/), src/game (composition root, KernelHost src/game/kernel_host.gd, SaveHost src/game/save_host.gd, MonotonicClock, SystemWallClock), src/debug and src/observer (dev only, guarded load); published surfaces are kernel/facets (the presentation-visible kernel API list), kernel/ports, sim/<x>/views, sim/<x>/ports; test seams only in src/<module>/testing/"
    not: "src/sanity/ or any module path outside src/sim/<system>/; PROJECTOR scripts under src/sim/; a scene or Node-derived script in shared, kernel, data, save or sim (SaveHost and KernelHost are Nodes and live in src/game); any script under src/ outside the listed directories; presentation naming a kernel class outside kernel/facets"
    adr: docs/architecture/adr-0010-source-layout-static-typing-and-lint-enforcement.md
    reason: "Makes simulation-never-reads-presentation, tree-freedom and warning scope directory tests, and resolves IC-10"
    referenced_by:
      - docs/architecture/adr-0010-source-layout-static-typing-and-lint-enforcement.md
    added: 2026-10-28
    revised: ""
  - purpose: presentation_visible_kernel_api
    status: active           # ADR-0010 is Proposed
    api: "Presentation, debug and observer code may name only these kernel classes, all under src/kernel/facets/: PresentationClock, SurfaceHandle, ControlHandle, ModuleFacet, BlockRegistrar, ChangeBatch, NotifyContext, CompletionToken, NotifyEvent, DirtyAccumulator, KeyBuf, NotifyFacet, SubscriptionHandle, SubscriberClass, DirtyKey, PresentationFacet, PresentationViews; NotifyFacet is named only by src/game and PresentationFacet; ARCH-08 denies Kernel, KernelHost, TickDriver, SimViews, WriteScope, Command, BlockDescriptor, CallContext, ValidateContext, ApplyContext, HookContext, SnapshotContext; the list is data in tools/lint/config/edges.json"
    not: "a *Context glob ban (NotifyContext is a legal facets type); KernelStateView (state is PresentationClock.state, the restore marker PresentationClock.load_seq); presentation holding NotifyFacet"
    adr: docs/architecture/adr-0010-source-layout-static-typing-and-lint-enforcement.md
    reason: "Presenter handlers must name NotifyContext and CompletionToken (ADR-0003, ADR-0004, ADR-0009); the deny list names only tick-phase types"
    referenced_by:
      - docs/architecture/adr-0010-source-layout-static-typing-and-lint-enforcement.md
    added: 2026-10-28
    revised: ""
  - purpose: interface_mechanism
    status: active           # ADR-0010 is Proposed; LINT-SPIKE-1(a) confirms @abstract survives a release export
    api: "@abstract base class (port) in the ports/ directory of the lowest layer every consumer may import (single-consumer ports listed as port_files); implementations and hand-written test doubles extend it and restate the full signature including the return type; consumers type setup() parameters as the port; REFLECT-SIG compares each argument's type, class_name and hint_string and the return type, class_name and hint_string"
    not: "an abstract BlockOwner for owner callables (ADR-0003 section 1 keeps registered Callables); IFoo names; duck-typed objects checked only by has_method; reflective GDScript.new() on a port; gdUnit4 mock() of a port unless LINT-SPIKE-1 proves it works; a plain push_error base class except as the documented fallback"
    adr: docs/architecture/adr-0010-source-layout-static-typing-and-lint-enforcement.md
    reason: "Probed 2026-10-28 on 4.7.2: a subclass missing an abstract method and a direct PortBase.new() are parse errors"
    referenced_by:
      - docs/architecture/adr-0010-source-layout-static-typing-and-lint-enforcement.md
    added: 2026-10-28
    revised: ""
  - purpose: gdscript_warning_policy
    status: active           # ADR-0010 is Proposed
    api: "project.godot [debug] gdscript/warnings/*: every severity 2 except RETURN_VALUE_DISCARDED 0, none at 1; directory_rules {res://: 0, res://addons: 0, res://tests: 0, res://src: 1, res://tools: 1}; CI gate tools/ci/load_all.gd loads every src and tools script and fails on null, can_instantiate() false or a SCRIPT ERROR log line; every @warning_ignore, _start and _restore form in src and tools must be an entry of tools/lint/config/allowlists/warning_ignore.json (PRJ-04), whose only initial entry is integer_division in IntMath.trunc_div; promotion is verified on the editor binary, not on a release export"
    not: "warnings left at level 1 (invisible in a headless run); a second strictness tier for presentation; directory_rules decision values other than 0 and 1 (engine error); an @warning_ignore of any warning that is not on the allow-list; := in src or tools"
    adr: docs/architecture/adr-0010-source-layout-static-typing-and-lint-enforcement.md
    reason: "Probed 2026-10-28: severity is global per warning, directory_rules is include/exclude only, level-1 warnings print nothing headless"
    referenced_by:
      - docs/architecture/adr-0010-source-layout-static-typing-and-lint-enforcement.md
    added: 2026-10-28
    revised: ""
  - purpose: integer_division_site
    status: active           # ADR-0010 is Proposed
    api: "IntMath in src/shared/int_math.gd (function list and divisor guard owned by ADR-0005); the single @warning_ignore(\"integer_division\") in src/ sits on the return statement of IntMath.trunc_div; the operator tokens /, %, /= and %= are banned outright in shared, kernel and sim outside IntMath (SIM-04); over-range decimal literals are banned (SIM-15)"
    not: "a bare integer or vector / or % in shared, kernel or sim; min, max, clamp, abs, sign, floor, ceil, round, int(), float() or trig functions in those directories; a second integer_division ignore or a '# nn:' marker site"
    adr: docs/architecture/adr-0010-source-layout-static-typing-and-lint-enforcement.md
    reason: "Truncation toward zero and signed % probed on 4.7.2; INTEGER_DIVISION fires only for int / int, so Vector2i division, /= and % need the lexical rule; ADR-0005 section 1.4 and 1.7 adopt the same single site (OQ-L closed)"
    referenced_by:
      - docs/architecture/adr-0010-source-layout-static-typing-and-lint-enforcement.md
    added: 2026-10-28
    revised: ""
  - purpose: lint_forbidden_token_list
    status: active           # ADR-0010 is Proposed
    api: "One forbidden-token list for time and randomness, owned by ADR-0010 SIM-02: any Time. member; OS.get_ticks_*, OS.get_unix_time*, OS.delay_*; Engine.get_frames_*, get_physics_frames, get_process_frames; get_process_delta_time, get_physics_process_delta_time; randi, randf, randi_range, randf_range, randfn, randomize, seed(, rand_from_seed, RandomNumberGenerator, Crypto, pick_random, shuffle; allow-list exactly kernel_host.gd, the diagnostics timer, monotonic_clock.gd, system_wall_clock.gd; ADR-0001 and ADR-0005 point at it"
    not: "a second token list in another ADR; render-time timers using Time.* (they use delta, present_time or the injected Clock); wall time entering a block, a hash or an ordering decision"
    adr: docs/architecture/adr-0010-source-layout-static-typing-and-lint-enforcement.md
    reason: "Unions ADR-0005 section 1.7 and ADR-0001 entries so the lint has one owner"
    referenced_by:
      - docs/architecture/adr-0010-source-layout-static-typing-and-lint-enforcement.md
    added: 2026-10-28
    revised: ""
  - purpose: lint_tool_and_tiers
    status: active           # ADR-0010 is Proposed
    api: "tools/lint/lint_main.gd (headless GDScript, extends SceneTree, OS.get_cmdline_user_args(), quit(0|1|2)) with core/ library and tools/lint/<pack>_lint.gd packs; one finding format LINT-<PACK>-<NN> <error|warn> <path>:<line>: <rule>: <message>; exit 0 clean, 1 findings, 2 tool failure (including a stale class map); tiers E1 engine parser, E2 lexical scan, E3 reflection, E4 runtime counters and boot walks; every rule has bad and good fixtures under tests/fixtures/lint/<rule>/ and a '# lint-allow: RULE reason' waiver unless marked no-waiver; run by CLI before gdUnit4 and by per-system wrappers tests/unit/<system>/lint/; MVP minimum rule set required at KRN-LAT-02"
    not: "gdtoolkit or any Python parser as a blocking gate; a rule stated as 'lint-enforced' without a rule id and tier; a lint that claims to see types, data flow or WriteScope membership; treating lint exit codes as kernel fault domains"
    adr: docs/architecture/adr-0010-source-layout-static-typing-and-lint-enforcement.md
    reason: "GDScript has no public AST; reflection and the class map exist only in-engine; ADR-0007 already makes GDScript the CI tool language"
    referenced_by:
      - docs/architecture/adr-0010-source-layout-static-typing-and-lint-enforcement.md
    added: 2026-10-28
    revised: ""
  - purpose: allocation_verification
    status: active           # ADR-0010 is Proposed
    api: "zero-allocation acceptance wording = HOT-01 token scan of '## @hot' functions (transient allocation) + OBJECT_COUNT unchanged over 600 ticks + MEMORY_STATIC within tolerance over 600 ticks on the debug binary (retained growth) + is_same and is_read_only() on every array accessor + injected WorkCounter counts for algorithmic budgets; accessors return Array[T] or Dictionary[K, V] marked read-only, never a Packed*Array; Kernel.dispose() leaves no reference cycle (weakref back-references, no stored capturing lambdas)"
    not: "claiming 'zero heap allocations' from OBJECT_COUNT alone (arrays, dictionaries, strings and lambdas add 0); using == for identity; conditional compilation flags for counters; MEMORY_STATIC or MEMORY_STATIC_MAX as a per-window or release-build gate; exposing a Packed*Array from an accessor"
    adr: docs/architecture/adr-0010-source-layout-static-typing-and-lint-enforcement.md
    reason: "Probed 2026-10-28: OBJECT_COUNT blind to arrays; MEMORY_STATIC sees retained (1.7 MB) but not transient (208 B for 100k) allocation; packed arrays have no read-only marker"
    referenced_by:
      - docs/architecture/adr-0010-source-layout-static-typing-and-lint-enforcement.md
    added: 2026-10-28
    revised: ""
  - purpose: test_framework_pin
    status: active           # ADR-0010 is Proposed; LINT-SPIKE-1(g) re-observes it
    api: "gdUnit4 6.2.2 committed under addons/gdUnit4 against Godot 4.7.2; lane H headless runs exactly the project.yaml commands.test string (tests/unit, tests/integration), lane W windowed (tests/render only), lane P nightly release-export benchmark; exit 101 (leaked nodes) counts as failure; render suites never run in lane H and are listed SKIPPED; unlisted engine ERROR or WARNING log lines fail lane H; --import retried up to three times or .godot cached; port doubles are hand-written, not mock()"
    not: "changing the gdUnit4 version without repeating the spike; a render-tagged check that can pass headless or with a zero draw count; treating 'No test cases found' as a pass"
    adr: docs/architecture/adr-0010-source-layout-static-typing-and-lint-enforcement.md
    reason: "6.2.2 observed running on 4.7.2 (current-best-practices.md, 2026-10-17); the gdUnit4 README lists 4.7.1 at most"
    referenced_by:
      - docs/architecture/adr-0010-source-layout-static-typing-and-lint-enforcement.md
    added: 2026-10-28
    revised: ""
  - purpose: native_code_policy
    status: active           # ADR-0010 is Proposed
    api: "GDScript only through MVP; ladder rung 0 as written, rung 1 tuning, rung 2 call-overhead removal, rung 3 GDExtension only for the Graph propagate phases after Spike S1 / SP-ENG-1 evidence on RM-DECK and an accepted ADR amendment; the port stays a GDScript @abstract class, the extension registers a plain RefCounted-derived native class and a GDScript adapter extends the port and forwards calls; the GDScript implementation stays as golden reference, both run the full suite and replay state_hash in CI; integer-only API, no Node, no threads, built with -fwrapv, Windows and Linux x86_64"
    not: "C# (the mono editor is not the shipping runtime); GDExtension for presentation or Node-bound code; a native class that is expected to satisfy `is Port`; native code before S1 evidence; selecting the implementation by a global flag read inside the simulation"
    adr: docs/architecture/adr-0010-source-layout-static-typing-and-lint-enforcement.md
    reason: "A GDExtension class can extend only a native class, so the adapter is the only way to keep one seam and unchanged call sites"
    referenced_by:
      - docs/architecture/adr-0010-source-layout-static-typing-and-lint-enforcement.md
    added: 2026-10-28
    revised: ""
  - purpose: wall_clock_port
    status: active           # ADR-0010 is Proposed
    api: "WallClock (@abstract, src/save/wall_clock.gd, now_unix() -> int) is the only wall-time source; implemented by src/game/system_wall_clock.gd (the fourth SIM-02 allow-list entry); injected only into SaveService.setup(); created_unix and SaveInfo time are display metadata and never an ordering, gate or hash input"
    not: "a Time.* token outside the SIM-02 allow-list; a wall-clock value in a block, hash or ordering decision; the port living in src/kernel; a render-timer exemption"
    adr: docs/architecture/adr-0010-source-layout-static-typing-and-lint-enforcement.md
    reason: "ADR-0006 needs a header timestamp; the monotonic Clock port cannot supply it (TD-11)"
    referenced_by:
      - docs/architecture/adr-0010-source-layout-static-typing-and-lint-enforcement.md
    added: 2026-10-28
    revised: ""
# --- interfaces ---
  - contract: wall_clock
    status: active           # ADR-0010 is Proposed
    pattern: direct_call     # injected port (setup parameter)
    producer: system-wall-clock   # src/game/system_wall_clock.gd
    consumers:
      - save-load-service        # SaveService.setup() only
    adr: docs/architecture/adr-0010-source-layout-static-typing-and-lint-enforcement.md
    signal_signature: "WallClock.now_unix() -> int"
    referenced_by:
      - docs/architecture/adr-0010-source-layout-static-typing-and-lint-enforcement.md
    added: 2026-10-28
    revised: ""
# --- forbidden_patterns ---
  - pattern: preload_or_load_of_script_in_src
    status: active           # ADR-0010 is Proposed
    description: "No preload, load or ResourceLoader of a .gd path anywhere in src/ (reference other systems by class_name); non-script preload only in src/presentation and src/game; the one guarded load() of the dev roots in src/game is the sole exception. No waiver (ARCH-03)."
    why: "Architecture dependency rule 2: a preload cycle fails to parse and hides coupling; the import-edge matrix is checked on class_name tokens."
    adr: docs/architecture/adr-0010-source-layout-static-typing-and-lint-enforcement.md
    added: 2026-10-28
  - pattern: scene_tree_or_engine_server_in_sim_layers
    status: active           # ADR-0010 is Proposed
    description: "No Node-derived class, scene-tree token (Node, SceneTree, get_tree, get_node, $, @onready, @export, _process, _input), engine server (RenderingServer, PhysicsServer2D, AudioServer, DisplayServer) or Input singleton in src/shared, src/kernel, src/data, src/save or src/sim. KernelHost and SaveHost live in src/game. No waiver (SIM-01)."
    why: "Kernel H10 and architecture Principle 1: the simulation runs headless; gdUnit4 runs inside a tree so tree-freedom is a lint, not an arrangement."
    adr: docs/architecture/adr-0010-source-layout-static-typing-and-lint-enforcement.md
    added: 2026-10-28
  - pattern: float_or_wall_clock_in_sim
    status: active           # ADR-0010 is Proposed
    description: "No float declaration, float literal, float-producing member (.length, .normalized, .distance_to, ...), Vector2/Vector3/Transform2D, or any SIM-02 token (Time.*, Engine frame counters, get_process_delta_time, global RNG) in src/shared and src/sim; kernel and data confine floats and clock reads to the float allow-list (presentation_clock.gd, the diagnostics timer, *_read.gd) and the four-entry SIM-02 allow-list; the src/shared float allow-list is exactly det_rng_float.gd and canonical_encoder.gd."
    why: "Kernel H9, determinism by construction; a float or a wall-clock read that reaches state breaks replay and the state hash."
    adr: docs/architecture/adr-0010-source-layout-static-typing-and-lint-enforcement.md
    added: 2026-10-28
  - pattern: integer_division_outside_int_math
    status: active           # ADR-0010 is Proposed
    description: "The operator tokens /, %, /= and %= appear only inside IntMath in shared, kernel and sim; there is exactly one @warning_ignore(\"integer_division\") in src/, on IntMath.trunc_div; min, max, clamp, abs, sign, floor, ceil, round and trig functions are not used on simulation values (use mini, maxi, clampi, absi, signi)."
    why: "Negative operands truncate toward zero and % keeps the sign (probed on 4.7.2); Vector2i division truncates silently and escapes INTEGER_DIVISION; one tested site removes the class of off-by-one bugs."
    adr: docs/architecture/adr-0010-source-layout-static-typing-and-lint-enforcement.md
    added: 2026-10-28
  - pattern: unlisted_warning_ignore
    status: active           # ADR-0010 is Proposed
    description: "Every @warning_ignore, @warning_ignore_start and @warning_ignore_restore in src/ and tools/ is an entry of tools/lint/config/allowlists/warning_ignore.json (path, function, warning id, reason, owner); no waiver (PRJ-04)."
    why: "The engine raises promoted warnings, so a lint-allow comment cannot cover them; an unlisted ignore reopens the typing hole warnings-as-errors closes."
    adr: docs/architecture/adr-0010-source-layout-static-typing-and-lint-enforcement.md
    added: 2026-10-28
  - pattern: static_var_in_src
    status: active           # ADR-0010 is Proposed
    description: "No static var in src/ and no [autoload] section in project.godot. No waiver (ARCH-04)."
    why: "The only way two kernels or catalogs in one process could silently share state; TR-cmd-052 and TR-itemdb-002 require independent instances."
    adr: docs/architecture/adr-0010-source-layout-static-typing-and-lint-enforcement.md
    added: 2026-10-28
  - pattern: presentation_names_sim_internals_or_kernel_core
    status: active           # ADR-0010 is Proposed
    description: "src/presentation, src/debug and src/observer name only sim/<x>/views, sim/<x>/ports and the kernel/facets list; never Kernel, KernelHost, TickDriver, SimViews, WriteScope, Command, BlockDescriptor, CallContext, ValidateContext, ApplyContext, HookContext or SnapshotContext; NotifyFacet only in src/game and PresentationFacet. src/sim never names presentation, game, debug or observer; src/data names shared only. Checked on class_name tokens (ARCH-02, ARCH-08)."
    why: "Simulation never reads presentation; a facet leak would give presentation write or submit rights (ADR-0003 risk)."
    adr: docs/architecture/adr-0010-source-layout-static-typing-and-lint-enforcement.md
    added: 2026-10-28
  - pattern: foreign_private_member_access
    status: active           # ADR-0010 is Proposed
    description: "No access to a ._member through a receiver other than self, and no assignment to a frozen catalog _field outside its own _freeze (ARCH-06); a same-class peer access carries a '# lint-allow: ARCH-06 reason' waiver."
    why: "GDScript has no private fields; write-once catalog fields and block internals are conventions the lint must hold (ADR-0007 section 1.5)."
    adr: docs/architecture/adr-0010-source-layout-static-typing-and-lint-enforcement.md
    added: 2026-10-28
  - pattern: render_test_in_headless_lane
    status: active           # ADR-0010 is Proposed
    description: "Pixel, frame-diff, draw-call and flash checks live only under tests/render/ and run only in the windowed lane; the headless job lists them SKIPPED, and every rendered test asserts a nonzero draw or pixel count through RenderGuard."
    why: "A headless runner has no rasteriser; a vacuous pass reads as evidence (TR-pxl-037, TR-percfilter-033, TR-debug-045)."
    adr: docs/architecture/adr-0010-source-layout-static-typing-and-lint-enforcement.md
    added: 2026-10-28
  - pattern: await_in_presenter_or_subscriber
    status: active           # ADR-0010 is Proposed
    description: "No await in any script registered through PresenterDescriptor or that calls .subscribe( (PRES-06); a subscriber handler returns its CompletionToken synchronously and the kernel's token-identity check records anything else INCOMPLETE."
    why: "A suspended coroutine returns a state object instead of the armed token; ADR-0004 section 2."
    adr: docs/architecture/adr-0010-source-layout-static-typing-and-lint-enforcement.md
    added: 2026-10-28
  - pattern: thread_or_deferred_call_outside_save
    status: active           # ADR-0010 is Proposed
    description: "Thread, WorkerThreadPool, Mutex and Semaphore only under src/save/ (IO-02); call_deferred, set_deferred, call_thread_safe and Engine.get_main_loop absent from src/shared, kernel, sim, data and save (SIM-12); the only cross-thread payload is a PackedByteArray or PackedInt64Array handed over by value and never mutated afterwards."
    why: "Threads are confined to the one legal user; deferred calls reorder work outside the tick."
    adr: docs/architecture/adr-0010-source-layout-static-typing-and-lint-enforcement.md
    added: 2026-10-28
  - pattern: packed_array_exposed_or_setter_tracked
    status: active           # ADR-0010 is Proposed
    description: "Accessors return Array[T] or Dictionary[K, V] marked read-only, never a Packed*Array; a Packed*Array property with a set( accessor is a finding in src/ unless the owner marks dirty explicitly (ARCH-11, CNT-03)."
    why: "Packed arrays have no read-only marker and pass by reference; 4.7 element writes skip setters."
    adr: docs/architecture/adr-0010-source-layout-static-typing-and-lint-enforcement.md
    added: 2026-10-28
# --- referenced_by_update ---
# kernel_autoload_or_global (append to the existing entry's referenced_by)
      - docs/architecture/adr-0010-source-layout-static-typing-and-lint-enforcement.md   # ARCH-04, ARCH-08 (target: forbidden_patterns/kernel_autoload_or_global, under its adr references)
# physics_callback_tick_source (append to the existing entry's referenced_by)
      - docs/architecture/adr-0010-source-layout-static-typing-and-lint-enforcement.md   # SIM-06, PRJ-02
# await_process_frame_in_sim_or_presentation (append to the existing entry's referenced_by)
      - docs/architecture/adr-0010-source-layout-static-typing-and-lint-enforcement.md   # SIM-07
# command_constructed_outside_kernel (append to the existing entry's referenced_by)
      - docs/architecture/adr-0010-source-layout-static-typing-and-lint-enforcement.md   # KRN-02
# write_outside_writescope_or_setter_dirty (append to the existing entry's referenced_by)
      - docs/architecture/adr-0010-source-layout-static-typing-and-lint-enforcement.md   # ARCH-06, ARCH-11 (file and function granularity only; WriteScope membership is a runtime check, ADR-0003)
# void_or_value_returning_owner_callable (append to the existing entry's referenced_by)
      - docs/architecture/adr-0010-source-layout-static-typing-and-lint-enforcement.md   # KRN-07 (E2 function-scoped; runtime INCOMPLETE detection is ADR-0003)
# presentation_submit_without_handle (append to the existing entry's referenced_by)
      - docs/architecture/adr-0010-source-layout-static-typing-and-lint-enforcement.md   # KRN-04 (keyed to issue_surface_handle names; handle seal is ADR-0002)
# block_as_resource_or_node (append to the existing entry's referenced_by)
      - docs/architecture/adr-0010-source-layout-static-typing-and-lint-enforcement.md   # SIM-01, SIM-10
# content_directory_scan (append to the existing entry's referenced_by)
      - docs/architecture/adr-0010-source-layout-static-typing-and-lint-enforcement.md   # IO-01 (a tools/ enumerator of src/ is not game code scanning assets/data)
# mutable_or_resource_catalog (append to the existing entry's referenced_by)
      - docs/architecture/adr-0010-source-layout-static-typing-and-lint-enforcement.md   # ARCH-06, CNT-03
# loader_side_effect_or_foreign_boot_code (append to the existing entry's referenced_by)
      - docs/architecture/adr-0010-source-layout-static-typing-and-lint-enforcement.md   # SIM-11 (IntMath guard assert is the single allow-list entry)
# content_file_read (append to the existing entry's referenced_by)
      - docs/architecture/adr-0010-source-layout-static-typing-and-lint-enforcement.md   # IO-01
# frame_order (append to the existing entry's referenced_by)
      - docs/architecture/adr-0010-source-layout-static-typing-and-lint-enforcement.md   # KRN-03, KRN-05 (HOST_SERVICE -40 for SaveHost)
# simulation_clock (append to the existing entry's referenced_by)
      - docs/architecture/adr-0010-source-layout-static-typing-and-lint-enforcement.md   # SIM-02 (four-entry allow-list: kernel_host.gd, diagnostics timer, monotonic_clock.gd, system_wall_clock.gd)
# content_export (append to the existing entry's referenced_by)
      - docs/architecture/adr-0010-source-layout-static-typing-and-lint-enforcement.md   # ARCH-09: proposes one exclude_filter glob src/*/testing/*; the entry itself is ADR-0007's and must not be edited in place (new entry or ADR-0007 amendment)
```

## Review Log
- **2026-10-28 godot-specialist (engine): verdict BLOCKING; revision applied.** E1 applied (§ 9.3 rewritten: the port stays a GDScript `@abstract` class, the extension registers a plain native class, a GDScript adapter `extends` the port and forwards, plus a `ContractCheck` fixture of the native method list). E2 applied (§ 5.4: `INTEGER_DIVISION` catches only `int / int`; `SIM-04` bans the tokens `/`, `%`, `/=`, `%=` outright; `SIM-03` member bans; vector helpers requested of ADR-0005 as OQ-N; bad fixture `Vector2i(5, -3) / 2` in LINT-SPIKE-1(b)). E3 applied under the id **`PRJ-04`** (not SIM-14) with `SIM-05` as its special case. E4 applied: the divisor guard is ADR-0005's (TD-12) and returns 0 with a debug `assert` for `b <= 0`, covering `b = 0`, `b < 0` and `INT64_MIN / -1`; fixtures added; the `assert` is the one `SIM-11` allow-list entry and its log line a suite-scoped § 8.6 entry; the headless behaviour of the failed `assert` is UNVERIFIED (LINT-SPIKE-1(j)). E5 applied (§ 1.2 conventions: `_`-prefixed fields and bare `setup()` parameter, no `:=`, typed lambdas, no Node property names; `EMPTY_FILE`, `ENUM_VARIABLE_WITHOUT_DEFAULT`; the `@abstract` parameter and `UNUSED_SIGNAL` outcomes are UNVERIFIED fixtures). E6 applied (`ARCH-05` reflection half against `ClassDB`; accessor naming). E7 applied (§ 3.3 reworded; override fixture). E8 applied (§ 3.3, § 6.4: argument and return `type`, `class_name`, `hint_string`, masked flags). E9 applied (`Script.is_abstract()` and `can_instantiate()` probes in LINT-SPIKE-1(a); hand-written doubles § 3.5). E10 applied (`KRN-01` filter and fixtures). E11 applied (§ 6.3 full string and construct list, corpus run). E12 applied (class map built twice, exit 2 on disagreement). E13 applied (retry up to three times or cache `.godot/`; runner wiring in § 6.1; throw-to-exit-2 fixture). E14 applied (`MEMORY_STATIC_MAX` deleted; `CNT-03` Packed rule; `ARCH-11`). E15 applied (§ 4.2 leak-freedom rules, cycle fixture). E16 applied (single glob, scratch-directory pack scan, LINT-SPIKE-1(i)). E17 applied (§ 5.3 four rules; `sorted_int_keys`/`sorted_text_keys`). E18 applied (promotion verified on the editor binary only). E19 applied (optional const convention added). E20 applied (P10 re-observation in LINT-SPIKE-1(b); confidence note; this review stays a desk review, a human or specialist rerun is still pending). E21 applied (exact `commands.test` string in § 8.1; `REFLECT-SIG` also runs over doubles).
- **2026-10-28 technical-director (TD-ADR): verdict CONCERNS; revision applied.** T1 applied as ruled by TD-01 (explicit deny list, facets list in § 1.2, `NotifyFacet` restriction, no `*Context` glob; the review amendment's `KernelStateView` is superseded, see Declined). T2 applied as ruled by TD-02 (`PRES-01` reworded; shared presentation files stay in `src/presentation/pipeline/`; no `common/` directory). T3 applied: `src/shared/` redefined, `src/shared/det/` row, `HashingContext` limited to named files (TD-13), per-directory `float_allowlist.json`, `DET-01` path, `IntMath` snippet replaced by a pointer to ADR-0005 (TD-12), `SaveHost` at `src/game/save_host.gd` (TD-10). T4 applied as ruled by TD-11 (`WallClock` port in `src/save/`, `system_wall_clock.gd` the fourth `SIM-02` entry, render-timer allowance withdrawn). T5 applied as ruled by TD-06 (`KRN-04` keyed to `issue_surface_handle` names). T6 applied (§ 6.6b registry table; Related restated). T7 applied (§ 6.10 MVP minimum and deferred list, typing-cost note). T8 applied (`can_instantiate()` and `is_abstract()` probes, `load_all` exemption). T9 applied (GDD and architecture sync section, Ordering Note, SIM-13 settled, OQ-L closed, LINT-SPIKE-1 for architecture.md § Spikes).
- **Rulings applied.** TD-01: ARCH-08, § 2.2 rule 4, § 1.2 facets list, `edges.json`. TD-02: PRES-01 wording. TD-04: `HOST_SERVICE` -40 in `KRN-03`/`KRN-05`, `KernelHost` at `src/game/kernel_host.gd` (OQ-C closed), no `KernelStateView`, `SIM-02` allow-list pointer. TD-06: `KRN-04`. TD-10: `SaveHost` row, § 4.1, diagram. TD-11: § 4.5, Key Interfaces, OQ-D closed. TD-12: § 1.1, § 5.4, `DET-01`, OQ-E closed. TD-13: § 4.1, `IO-01`, § 5.3 idiom, § 5.5, `dictionary_variant.json`, `float_allowlist.json`, `src/shared` definition. TD-14: unified `SIM-02`, trig bans, `/=` and `%=`, `SIM-15`, § 5.4 correction. TD-15: `IO-02`, `SIM-12`, § 9.3 payload note. TD-16: `PRES-06`. TD-17: `PRES-02` to `PRES-05`. TD-18: placement allow-list in `PRES-03`. TD-21: `PRJ-02` two read tiers hosted by ADR-0008 § 5.5(a). TD-23: TR-shipbody-038 row cites ADR-0008 § 6. TD-27: Enables only for siblings, `save` has no `data` import, `RegistryView` and `BuildConstants` in `src/save/`, Ordering Note refreshed, SIM-13 active. TD-30: OQ-L closed. TD-33: Registry Candidates section; lint exit codes are not fault domains (§ 6.1).
- **Declined or superseded.** T1 amendment's `KernelStateView` entry and its "KernelHost, TickDriver in the deny list" are replaced by TD-01 (the state is `PresentationClock.state`; `KernelHost` and `TickDriver` stay under rule 7). T2 amendment's `src/presentation/common/` directory is superseded by TD-02. T4 amendment's `src/kernel/ports/` placement of `WallClock` is superseded by TD-11 (`src/save/wall_clock.gd`). T5 amendment's `input_surface.json` is superseded by TD-06. E3's rule id `SIM-14` is renamed `PRJ-04` because the scope includes `tools/`. E4 option (a) is taken in ADR-0005's form, with a debug `assert` instead of a fault code, which required the single `SIM-11` allow-list entry.
- **Still open after this revision.** OQ-A, OQ-B, OQ-F, OQ-G, OQ-H, OQ-I, OQ-J, OQ-K, OQ-M and the new OQ-N. Every claim marked UNVERIFIED stays bound to LINT-SPIKE-1 (a) to (j). The engine and TD reviews were desk reviews against this text; they are not a sign-off, and the Accepted gate still needs ADR-0001, ADR-0002, ADR-0003 and ADR-0007 Accepted and LINT-SPIKE-1 passed.
