# ADR-0011: Life-Support Graph Data Model

## Status
Proposed

## Date
2026-10-29

## Last Verified
2026-10-29 (against `design/gdd/life-support-graph.md` LS-1 to LS-6, LS-14 to LS-21, LS-24, D.10, D.13 and Section E; ADR-0003 to ADR-0007 and ADR-0010, all Proposed; Godot 4.7.2 probes of 2026-10-29 on the editor binary, recorded in Engine Compatibility); revised 2026-10-29 after the per-ADR engine and TD reviews and the Core cross-ADR adjudication (Review Log)

## Decision Makers
Claude (author), running under modes.automation: autonomous (project.yaml); decisions logged for the user's review; the user has not yet reviewed this ADR.

## Summary
The Life-Support Graph is the largest simulation block (up to `MAX_GRAPH_NODES` nodes and `MAX_GRAPH_EDGES` edges) and its determinism, save format, hash, dirty keys and read API all depend on how its state is laid out in memory. This ADR fixes that layout: one `graph.state` block held as a struct of dense packed arrays in seven tables plus a def-name CSR pair and a scalar header, with fixed-point integer units and a documented column width per field; append-only ids, slots and type enums that are never renumbered or reused; a hard split of every member into PERSISTED, DERIVED, SCRATCH or DIAGNOSTIC (reflection-checked); the snapshot, restore and fault shapes; the read-view and descriptor-store layout; the dirty-key kinds and packing; the persisted hook products (`n_served`, `n_draw`, CABLE net load, the power-use window) and the schedule in force that make a restore at any tick exact; the shared value types (`JoinPlan`, `WeldResult`, `FailureFilter`, `FailureSlice`) that ADR-0012 and ADR-0021 build on; and the Graph's side of the Layout port (`GraphLayoutAdapter`, § 9.5).

## Engine Compatibility

| Field | Value |
|-------|-------|
| **Engine** | Godot 4.7.2 (stable, mono build probed), GDScript |
| **Domain** | Core (simulation state layout, integer arithmetic, serialization shape) |
| **Knowledge Risk** | HIGH (raised from MEDIUM by review E7: `VERSION.md` rates 4.5, 4.6 and 4.7 HIGH, ADR-0010 rates its own domain HIGH, and this ADR depends on two 4.7 rows, `@abstract` (4.5) and typed `Dictionary` (4.4)). Everything this ADR uses predates 4.4 except typed `Dictionary[K, V]` (4.4), `@abstract` (4.5) and the two 4.7 GDScript rows in `breaking-changes.md`. The engine reference has no core or scripting module (`modules/` holds animation, audio, input, navigation, networking, physics, rendering and ui only), so packed-array semantics, integer behaviour and reflection are sourced from ADR-0005's probe table (P1 to P22) and from this ADR's own probes, and stay UNVERIFIED on a release export until spikes S0-a and S3 run there. |
| **References Consulted** | `docs/engine-reference/godot/VERSION.md`; `breaking-changes.md` (4.6 to 4.7 GDScript rows; 4.4 to 4.5 `duplicate_deep` row); `deprecated-apis.md` (untyped Array/Dictionary pattern; `duplicate()` for nested data replaced by `duplicate_deep()` since 4.5); `current-best-practices.md` (GDScript 4.7: typed-return override, packed-array element writes); ADR-0005 Engine Compatibility P1, P5, P7, P8, P9, P11, P12, P14, P17, P20, P22 (cited by number below). No `modules/*.md` applies. |
| **Post-Cutoff APIs Used** | `@abstract` (4.5) on the `RoomSource` port, which this ADR defines, and as the base class `GraphLayoutAdapter` extends (ADR-0013's `LayoutGraphPort`; ADR-0010 § 3); nothing from 4.5 to 4.7 in the layout itself. Typed `Dictionary[int, int]` (4.4) appears in derived lookup caches only, never in a payload. The two 4.7 rows that bind this ADR: **(1)** "Packed array element writes no longer call the setter" (`breaking-changes.md`, `current-best-practices.md`): the Graph has no packed-array setters and marks every dirty key explicitly through `scope.mark_dirty` (§ 10); probed again below (P-d). **(2)** "Overrides inherit a typed return": binds anything that overrides a typed owner callable or an `@abstract` port method, so `RoomSource` implementers and `GraphLayoutAdapter` write the explicit return type and return on all paths (§ 11, Implementation Guidelines). |
| **Verification Required** | Probed 2026-10-29 on `Godot_v4.7.2-stable_mono_win64_console.exe --headless --script` (editor binary, not a release export; scripts in `C:/Users/dasbl/AppData/Local/Temp/claude/adr-probes/0011/`): **(a)** the typed numeric builtins `mini`, `maxi`, `clampi`, `absi`, `signi` exist and return `int` for `int` arguments (3, 5, 5, 4, -1); **(b)** `get_property_list()` on a `RefCounted` subclass lists every script variable, underscore-prefixed ones included and in declaration order, selectable with `PROPERTY_USAGE_SCRIPT_VARIABLE` (the basis of the reflection test in § 6.3); **(c)** swapping two `PackedInt64Array` members by reference works and an alias taken before a member element write sees the write (ADR-0005 P7 reproduced); **(d)** `arr[i] = v` on a packed member runs no setter (P8 reproduced, and no setter is declared); **(e)** `PackedInt64Array.fill(0)` on 4096 elements costs 1.65 microseconds, `duplicate()` of 4096 elements 0.11 microseconds, and `duplicate()` followed by the first element write on 8192 `int32` elements 0.71 microseconds (`duplicate()` shares the buffer until the first write on either side; the copy is paid by the next mutator's first write, § Performance); **(f)** a typed `Dictionary[int, int]` answers `get(k, -1)`; **(g)** `PackedInt32Array` stores wrap to 32 bits (`3000000000` reads `-1294967296`, ADR-0005 P9) and `PackedByteArray` to 8 bits (`300` reads `44`); **(h)** `var_to_bytes` returns a `StringName` dictionary key as a `StringName` (type 21) and a `String` key as a `String` (type 4), which is why payload keys must be `String` (ADR-0006 § 1.2 walk rejects `StringName`); **(i)** a synthetic zero-filled payload with this ADR's columns measures 29,674 canonical TLV bytes (29,560 `var_to_bytes` bytes) for 105 nodes, 150 edges, 270 ports, 14 compartments, 1 seam, 3 sections and 2 queued entries per queue, and 33,144 bytes at 120 nodes and 172 edges (value-independent, so a real save differs by at most the key-text length). The fixture is synthetic: it assumes 60 `defs.off` entries and 1,500 `defs.utf8` bytes, which the ADR did not state before this revision; with that assumption the pre-revision reach-point (114 nodes) is reproducible, and the columns this revision adds (§ 3.11) are added by row arithmetic on the same fixture and re-measured by S3. Review probes of the same day on the same binary, all reproduced: **(j)** an out-of-bounds packed-array read logs `SCRIPT ERROR: Out of bounds get index` and returns 0, and an out-of-bounds write logs `Out of bounds set index` and aborts the rest of the calling function (§ 4.1 rule 6); **(k)** a `static var` is absent from `get_property_list()` under `PROPERTY_USAGE_SCRIPT_VARIABLE` (§ 6.3 Method C); **(l)** two `RefCounted` objects that reference each other leak at exit (`WARNING: ObjectDB instances were leaked at exit`, § 6.3 Method C); **(m)** a local alias of a packed member sees an element write and a `resize` (one shared buffer) and only whole-member re-assignment strands it, and `duplicate()` isolates (§ 6.2). **Still open and named:** S0-a and S0-b on a `template_release` export (Windows, Linux x86-64, Linux arm64 / Deck) for items (c), (d), (e), (g), (j) and (m); S1 (SP-ENG-1) for GDScript accessor, `Callable.call` and `IntMath.floor_div` cost in the real phases; S3 (snapshot, restore and `post_restore` time and size on the F-MVP fixture on RM-DESKTOP and RM-DECK); LINT-SPIKE-1 for loading every Key Interfaces snippet under the ADR-0010 § 5.1 warning configuration (the snippets here follow its typing rules but have not been load-checked). |

## ADR Dependencies

| Field | Value |
|-------|-------|
| **Depends On** | ADR-0003 (block descriptor, `WriteScope`, mutator status, fault domains, read-view facades; Proposed), ADR-0004 (dirty-key layout, `KeySpaceDescriptor`, quiet writes; Proposed), ADR-0005 (integer-only contract, `IntMath`, canonical encoding, sorted keys, cached hashes; Proposed), ADR-0006 (payload form, restore contract, `ctx.fault`; Proposed), ADR-0007 (type tables, `GraphTuning` as a `ScalarTable`, boot tiers, ledgers; Proposed), ADR-0010 (source layout, typing, lints; Proposed). Indirectly ADR-0001 (`ClockView.tick()` is the time source of `step_no()`) and ADR-0002 (the Graph registers no command). None is Accepted, so this ADR cannot be Accepted before them. |
| **Co-accepted with** | ADR-0013 (mutual ports: `LayoutGraphPort`, which `GraphLayoutAdapter` implements, and `RoomSource`, which Layout's `LayoutRoomSource` implements; neither may be Accepted alone; ADR-0010 § 2 allows sim-to-sim imports in either direction, so no source-layout change follows). The one hard edge of the pair is declared in ADR-0013's Depends On (the later-numbered ADR); this ADR deliberately does not list ADR-0013 there, so the dependency graph stays acyclic. |
| **Soft references (not acceptance prerequisites)** | ADR-0016 (reads this ADR's `restore`/`post_restore` sequence and the stock reconcile as a draft; restore ordering is not an acceptance prerequisite of either ADR). **Forward references (unwritten ADRs; each will list this ADR under Depends On when authored):** ADR-0021 (`weld_join`, `install_module` and `cut_detach` transaction), ADR-0022 and ADR-0023 (M17/M18 callers), ADR-0024 (`true_value` kinds), ADR-0026 (`set_section_dormant`), ADR-0027 (M15/M16), ADR-0028 (`served_pm` for the ambient mean). |
| **Enables** | ADR-0012 (propagation: consumes the tables, scratch class, descriptor store and value types defined here), ADR-0013 (id split and `graph_ref_base`; co-accepted with this ADR, and the hard edge is declared once, in ADR-0013's Depends On), ADR-0017 (typed getters for `served_pm`, `run_pm`, `node_state` and the `node_live`/`edge_live` guards). |
| **Blocks** | The Graph epic and every story that reads or writes `graph.state`; Item DB install, salvage and reinstall stories (TR-itemdb-016/031/040); the Instruments `load` reading (TR-instr-027); the Graph golden-hash and goldens-fixture stories (TR-lsg-031). Stories citing this ADR stay Blocked while it is Proposed. |
| **Ordering Note** | Write the shared pure files (`src/shared/graph/`: `GraphIds`, `GraphSchema`, `GraphLimits`, `graph_rules.gd`, `graph_products.gd`) and the `GraphTuning` freeze first, because ADR-0007's stage 1 loader and the Item DB (stage 2) call them before any kernel exists. The three ADR-0010 lint rules this ADR asks for (field-writer files, ARCH-06 exemption, owned-constant token list) should land with the first Graph file, not after; the OWN-01 configuration must already accept directory globs and per-member entries (§ 12.1) before the first `propagate/` file. |

## Context

### Problem Statement

The Graph GDD (LS-1) fixes the contract in prose: one persisted SIM block, struct-of-arrays, zero floats, store extensive and derive intensive, exact conservation, ascending-slot iteration, no Dictionary on the sim path, append-only ids. It defers eleven implementation questions to an ADR, of which this one takes three (ADR-LSG-01 struct-of-arrays layout, ADR-LSG-02 the fixed-point unit system, ADR-LSG-04 append-only topology identity). Meanwhile the Foundation ADRs have since made decisions the GDD could not know, and some of them change the GDD's wording: the payload is a plain Dictionary encoded by the kernel (ADR-0005 § 3.1), payload keys and values are restricted by the Save/Load type walk (ADR-0006 § 1.2), a restore reports through `ctx.fault` and copies into the existing block object (ADR-0006 § 2.2), division and modulus go only through `IntMath` and the repository has exactly one `integer_division` ignore (ADR-0005 § 1.4, ADR-0010 `SIM-05`), ARCH-06 forbids foreign `._member` access, and domain-3 fault codes are written "with its system ADR (ADR-0011 onward)" (ADR-0003 § 6).

Nothing about the layout can be left to the implementer. The columns are the hash input (ADR-0005), the save format (ADR-0006), the dirty-key vocabulary (ADR-0004), the read surface of eight consumer systems and the restore validator. A column added, renamed, re-typed or reordered after a build is shared is a `schema_version` bump and a migration. And the layout is the first thing the ADR-0012 phases, the ADR-0021 weld transaction and the ADR-0016 verifiers are written against.

### Constraints

- **Determinism (kernel, ADR-0005):** integer-only state; every intermediate below 2^62; dense arrays in ascending id order; no Dictionary iteration, no `sort_custom`, no `StringName` ordering; payload keys sorted; derived caches, diagnostics counters and anything non-persisted never in a payload or hash (TR-lsg-039).
- **Block contract (ADR-0003):** the block is a `RefCounted` held by its owner, never a `Resource` or `Node`; `restore` copies into the existing object because read-view facades are long-lived; every packed array in a `snapshot` payload is a duplicate; every mutator begins with `scope.open_for`; dirty keys only through `scope.mark_dirty`; faults are pre-packed integers `(DOMAIN_ID << 16) | local` with domain 3 fixed for the Graph.
- **Persistence (ADR-0006):** the stored payload is `var_to_bytes` of a Dictionary with `String` or `int` keys holding only `null`, `bool`, `int`, finite `float`, `String`, `Array`, `Dictionary` and `PackedByteArray`/`PackedInt32Array`/`PackedInt64Array`; `StringName`, vectors and objects are refused by the walk; failure is only `ctx.fault(<OWNER>_RESTORE_*)`; `snapshot(restore(snapshot()))` is byte-identical.
- **Source rules (ADR-0010):** `:=` banned, every `var` and `const` explicitly typed; no bare `/`, `%`, `/=`, `%=` and no untyped `min`/`max`/`clamp`/`abs`/`sign` in `src/sim/**` (`SIM-04`); exactly one `integer_division` ignore in `IntMath.trunc_div` (`SIM-05`, `PRJ-04`); no `assert` in `src/` outside `testing/` (`SIM-11`); no member, parameter or local named after a `@GlobalScope` utility or a base-class property (`SHADOWED_*` at error); accessors never return a `Packed*Array` (registry `packed_array_exposed_or_setter_tracked`); no `._member` access through a foreign receiver (`ARCH-06`); no `Node`, no signals in `src/sim/` (`SIM-01`, `SIM-13`).
- **Capacity (registry):** `MAX_GRAPH_NODES`, `MAX_GRAPH_EDGES`, `MAX_SEAMS_PER_SECTION`, `MAX_CREW`, `MAX_FOOTPRINTS_PER_STEP`; `MAX_DIRTY_KEYS` is the kernel's and must stay at or above 128 while `graph.state` is registered. Values are cited by name here and never retyped.
- **Performance:** the Graph's hook spike share (P99 and max over one hook tick), the 0.5 ms snapshot, 2 ms restore plus `post_restore`, and the 32 KB MVP snapshot are ADR-0012's budgets (TR-lsg-025/036) measured on a release export; this ADR owns the column count behind the 32 KB.
- **Pre-existing GDD wording that this ADR must reconcile, not repeat:** LS-1 (d) asks each Graph file to open with `@warning_ignore_start("integer_division")` (contradicts `SIM-05`); LS-1/TR-lsg-002 asks for underscore-prefixed fields written only by four files (no ADR-0010 rule expresses that and ARCH-06 forbids the access); TR-lsg-024 spells `GraphFault.raise(code, subject)` with no scope argument; LS-19 says Instruments derive `load` themselves while TR-instr-027 says the Graph publishes it; TR-itemdb-033 says a restore clamps an over-cap store while LS-21 says restore never clamps.

### Requirements

- One block `graph.state`, registered last of the Item DB, Layout, Graph chain, `restore_after [layout.state]`, `schema_version` 1 (TR-lsg-001).
- Struct-of-arrays on `PackedInt64Array`/`PackedInt32Array`/`PackedByteArray` only, writer-file discipline lint-enforced (TR-lsg-002).
- A unit table with a stored width and a documented maximum for every quantity; zero floats on the path; designer floats converted once into immutable `GraphTuning` (TR-lsg-003, TR-lsg-006).
- A written classification of every member as persisted, derived, scratch or diagnostic, verified by reflection and zero-and-diff (TR-lsg-004, TR-lsg-038, TR-lsg-039).
- Conservation counters in the persisted form with an exact-equality evaluator (TR-lsg-005).
- Ids, slots and type enums that never change meaning (TR-lsg-030 residual, TR-lsg-027/028 data side, TR-lsg-029 stamps).
- A column policy for install, salvage and reinstall (TR-itemdb-016/031/040).
- A pure, allocation-free read surface with a columnar descriptor store (TR-lsg-018/019, TR-instr-027, TR-itemdb-014, TR-light-034 data side).
- The dirty-key kind table, composite packing and bucket storage (TR-lsg-020 residual).
- The domain-3 fault list, the boot-code list and the restore reason set with deterministic precedence (TR-lsg-023/024/033 residuals).
- The read-by-name discipline for registry-owned constants (TR-lsg-043).

## Decision

### 1. One block, four classes, one layer

`graph.state` is one block (the GDD's two-block alternative is rejected: a weld writes both halves, and G4 wants one snapshot, R2). It is held as four sibling `RefCounted` objects, all created once by `GraphModule.setup()` and never replaced:

| Class | File (`src/sim/graph/`) | Holds | Persisted | Hashed | Written by |
|---|---|---|---|---|---|
| `GraphState` | `graph_state.gd` | the PERSISTED columns of § 3: seven tables, `hdr`, `defs` | yes | yes | the writer files of § 12.1: `propagate/**`, `graph_mutators.gd`, `graph_restore.gd` |
| `GraphCaches` | `graph_caches.gd` | DERIVED data rebuilt from `GraphState`, Layout and the catalog: id-to-slot maps, endpoint slots, adjacency CSR, components and trees, `path_cap`, the active lists (`active_nodes`, `active_edges`, `active_comps`, `active_seams`), `store_slots`, `machine_slots`, `exchange_links`, the POWER island list, the loop summaries (`loop_sum`: one component summary per loop, written by ADR-0012 phases 2 to 5, including the power summary), the intensive values (`P`, `p_o2`, `o2_pm`, trend), the descriptor store, `alarm_sections`, last-emitted buckets, `topology_version`, `topo_dirty`, lifecycle | no | no | per member, § 12.1 (`graph_derived.gd` for every member; also `propagate/**` and `graph_mutators.gd` for the members the matrix names) |
| `GraphScratch` | `graph_scratch.gd` | the ADR-0012 § 3 inventory: `PackedInt64Array` accumulators (`acc_n_o2`, `acc_n_inert`, `acc_q`, `acc_vent_mgu`) and the island (`island_*`), candidate (`cand_slot`, `cand_mode`), sort (`q_key`), traversal (`stack`, `queue`) and degree (`deg`, with the accessor `deg_max_of(e)`, filled at the head of phase 6) buffers; all zero at every hook exit | no | no | contents `propagate/**`; sizing `graph_derived.gd` (§ 12.1) |
| `GraphDiag` | `graph_diag.gd` | diagnostics counters: `saturations`, `stale_calls`, `gate_holds`, `stall_edges`, `footprints_dropped`, `stock_clamped`, `hash_invocations`, plus the ADR-0012 counters `late_rebuilds`, `queue_dropped`, `footprints_clamped` and `schedule_shift` (§ 7.4) | no | no | any Graph file (the one class every file may write) |

Splitting the non-persisted members into their own objects (instead of one wide `GraphState` with a DERIVED allowlist, the GDD's wording) makes the persisted-field check a one-line reflection test (§ 6.3): every script variable of `GraphState` is either a persisted column that `GraphSnapshot` writes into the payload or one of three holder references. TR-lsg-002 ("Graph state is a struct-of-arrays `GraphState extends RefCounted`") holds for the persisted object; the companions are the same module's. **Ownership is one-way, `GraphModule` to `GraphState` to the three holders:** `GraphCaches`, `GraphScratch`, `GraphDiag` and `GraphView` hold no field of type `GraphState` (a back-reference makes a `RefCounted` cycle, which leaks; § 6.3 Method C), and writers receive the state as a parameter.

**Other files.** `graph_module.gd` (`GraphModule`: `setup`, registration, owner callables), `graph_snapshot.gd` (builds the payload; reads `GraphState`, writes nothing), `graph_mutators.gd` (M1 to M19 and their `check_*`), the `propagate/` directory (ADR-0012's phases: the dispatcher `graph_propagate.gd` and one file per phase), `graph_restore.gd` (decode, validate, swap), `graph_derived.gd` (`ensure_derived`, `post_restore` derivation), `graph_faults.gd` and `graph_fault.gd`, `views/graph_view.gd`, `views/graph_view_cold.gd`, `views/failure_filter.gd`, `views/failure_slice.gd`, `views/join_plan.gd`, `views/weld_result.gd`, `views/salvage_record.gd`, `graph_layout_adapter.gd` (`GraphLayoutAdapter`, § 9.5; outside `ports/` because ADR-0010 ARCH-05 requires every file under `ports/` to be `@abstract`), `ports/room_source.gd`, `testing/` (test seams). M1's out record is ADR-0013's `SectionBases` (`src/sim/layout/views/`); there is no `instantiate_result.gd`. **Shared pure files** that `src/data/` loaders and the Item DB must reach without importing `src/sim/` (ADR-0007 § 1.9, ADR-0010 § 1.1): `src/shared/graph/graph_ids.gd` (`GraphIds`: every enum of § 4.7), `graph_keys.gd` (`GraphKeys`: the nine dirty-key kind constants `COMP` 1 .. `SECTION` 9 of § 10, and the `loop_anchor_key` and `subject_ref` packers), `graph_schema.gd` (`GraphSchema`: table and column specs, payload key names, `VERSION`), `graph_limits.gd` (`GraphLimits`: code-side width limits such as `ID_MAX`, `STEP_STAMP_MAX`), `graph_rules.gd` (`connectable`, `cap_seam` and the other pure D-formulas the Item DB lints call, TR-itemdb-036), `graph_products.gd` (the D.13 product list and `product_max_assertion`). `GraphTuning` lives at `src/data/graph/graph_tuning.gd` (a frozen class built by the loader from the ADR-0007 `ScalarTable`; `src/sim/graph/` names it as an injected published type).

**Registration** (`GraphModule`, in the kernel's `BOOTING`, last of the Item DB, Layout, Graph chain; ADR-0003 § 2 and § 6, ADR-0004 § 3):

| Descriptor field | Value |
|---|---|
| `block_id` / `owner_name` | `&"graph.state"` / `&"graph"` |
| `schema_version` / `supported_versions` | 1 / `[1]` |
| `block_class` / `persist` / `persist_scope` / `scope` | `SIM` / true / `WORLD` / `SHARED` |
| `gameplay` / `interpolated` | true / false (no `commit_baseline`, no `snap_baseline`) |
| `repl_hint` | `SHARED_SUMMARY` |
| `restore_after` | `[&"layout.state"]` |
| `snapshot`, `restore`, `default_state`, `post_restore`, `read_view`, `empty_view` | bound `Callable`s of `GraphModule` |
| `init_world`, `binding_source` | not registered (M1's only caller is Layout LM1 through `LayoutGraphPort`; LM1's callers are those ADR-0013 § 9 names, Web Map's `init_world` in the `INIT` scope at the MVP; the Graph issues no binding) |
| key space | `KeySpaceDescriptor` with the nine kinds of § 10 and `quiet_writes_allowed = true` |
| fault domain | `register_fault_domain(3, &"graph", GraphFaults.NAMES)` |
| hooks | `graph.propagate.power`, `.atmo`, `.failure`, divisor `tuning.graph_step_ticks`, offsets `tuning.hook_offset_power/atmo/failure` (data, ADR-0012 § 1: 6 and 0, 2, 4 at the baseline), chained by `after:` (ADR-0012 fixes the edges and phases; the descriptors are registered here), plus the plain RNG stream `graph.fail` registered through `facet.rng.register_stream(&"graph.fail", false)` |
| Layout port | `GraphModule.setup` constructs the `GraphLayoutAdapter` (§ 9.5) and exposes it as `graph.layout_port()`; the composition root passes it to `LayoutModule.bind_graph(port)` after `GraphModule.setup` and before the registry freeze |

The Graph registers no command (kernel H2; the domain commands live with Assembly, Interaction, Suit, Emitter State and Horror). Its published mutators are plain methods of `GraphMutators` that the composition root hands to those modules' `setup()` (ADR-0003 § 10).

### 2. The fixed-point unit system (ADR-LSG-02)

**2.1 Units.** Every stored, hashed or propagated quantity is an integer in one named unit. Scale constants come from the registry through `GraphTuning` (cited by name; never retyped here).

| Quantity | Unit and meaning | Stored as | Documented maximum (source) |
|---|---|---|---|
| Gas amount (`n_o2`, `n_inert`, tank `stock` for O2) | mGU: milli gas units, scale `GAS_SCALE`; 1 GU gives 0.1 kPa in one tile | `PackedInt64Array` | per compartment `n_max(c) = P_MAX x V_c x GAS_SCALE` (D.13); tank: the def's `stock_cap`, hard ceiling `GraphTuning.max_stock_gas` |
| Pressure, partial pressure | dkPa; **derived only** (`P = f(n, V)`, D.3) | never stored | `P_MAX`; standard `STD_PRESSURE_DKPA` |
| Volume | tiles, from Layout (`V`); **derived copy** | `GraphCaches` | `V_max` in `GraphTuning` (D.13) |
| Temperature | mC; compartment `T`, coolant `t_cool` | `PackedInt32Array` | `abs(dT)` maximum in D.13; ambient `T_AMB_MC` |
| Thermal remainder | `t_rem`, `tc_rem`: dJ carried so that `U = T x C + t_rem` is exact (two signed steps truncate toward zero, all others floor) | `PackedInt32Array` | below the compartment's `C` (bounded by `V_max x C_TILE`) |
| Power, energy | W is dJ per step; battery `stock` in dJ; `radiated_dj`, `produced_dj` | W in `PackedInt32Array`; stocks and counters in `PackedInt64Array` | per-field maxima in `GraphTuning` (D.13) |
| Liquids | mL (coolant, fuel); fuel burn counter in micro-mL (10^6 uml = 1 mL) | `PackedInt64Array` | the def's `stock_cap`; hard ceiling `GraphTuning.max_stock_liquid` |
| Integrity | bp, 0 to `HP_FULL` | `PackedInt32Array` | `HP_FULL` |
| Fractions | permille (served, run, throttle, circulation, intensity) | `PackedInt32Array` | 1000, plus overload up to the D.13 bound |
| Opening area | milli flow units on the gas path (`A_m = 1000 x flow units`, D.1); `area` and `area_max` are stored in flow units (64 = one standard door) | `PackedInt32Array` | `A_HOLE_MAX`, D.13 `A_m` bound |
| Time | **Graph steps**: `step_no = floor_div(T, GRAPH_STEP_TICKS)`; every persisted stamp (`fail_since_step`, `built_step`, `patched_at_step`, ...) is in steps, never kernel ticks | `PackedInt32Array` (stamps), `int` (header) | `STEP_STAMP_MAX` (`GraphLimits`, 2^30) |
| Enums, flags, bands | small closed sets | `PackedByteArray` | the `GraphIds` domain of each |
| Ids, slots, indices | non-negative ints | `PackedInt32Array` | `ID_MAX` (`GraphLimits`, 2^24); caps `MAX_GRAPH_NODES`, `MAX_GRAPH_EDGES` |
| Conservation counters, `next_*`, `last_step_*` | exact integer tallies | `PackedInt64Array` (per section), `int` (header) | no stored cap; D.13 bounds their growth rate |

**2.2 Width rule (why three array types).** `PackedInt64Array` holds every conserved or extensive quantity and every counter, because the GDD's gas maximum alone exceeds what a later content change could be allowed to wrap, and because all arithmetic is 64-bit anyway. `PackedInt32Array` holds ids, indices, step stamps, basis points, permille and the short physical quantities listed above, each with a declared maximum far below 2^31. `PackedByteArray` holds enums and flag bytes. A 32-bit store truncates silently (probe (g), ADR-0005 P9, P17), so the rule is: **arithmetic is done in plain `int`, and a value is range-checked against its column maximum before it is stored into an `int32` or `uint8` column**; the check is the mutator or phase's responsibility, is bounded by the column spec of § 3, and a violation is the numeric soft invariant of § 8.3. Values copied from the catalog into 32-bit columns (port caps, `req_max`, `hp_start`) are checked once at boot against the column maxima (`GRAPH_BOOT_COLUMN_RANGE`, § 8.4).

**2.3 Arithmetic discipline** (ADR-0005 § 1.3 to § 1.5, ADR-0010 `SIM-03`/`SIM-04`/`SIM-05`):
- Every flux is a non-negative magnitude with an explicit direction (LS-1 (d)), so every division on the propagation path has a non-negative numerator and `IntMath.floor_div` equals truncation. The two signed thermal remainder steps call `IntMath.trunc_div` by name.
- There is **no file-level `@warning_ignore_start("integer_division")` in `src/sim/graph/`**. LS-1 (d) asked for one and S0-b was to verify it; ADR-0005 P6 shows it compiles, but ADR-0010 `SIM-05`/`PRJ-04` allow exactly one `integer_division` ignore in `src/`, and `SIM-04` bans the `/` and `%` tokens outright, so the annotation would have nothing to silence and would itself be a finding. The cost is the one ADR-0005 measured (`floor_div` is about 12 times a bare operator, P14); whether that matters in the phases is S1's answer, and the remedy it names (inlining a proven non-negative form in named hot files) is an ADR-0010 amendment decided on that measurement (§ Open Questions 1). This is a `gdd_sync` item (LS-1 (d), TR-lsg-003 wording).
- `min`, `max`, `clamp`, `abs` and `sign` are written with the typed builtins `mini`, `maxi`, `clampi`, `absi`, `signi` (probe (a)). ADR-0010 `SIM-04` bans the untyped names and is silent about the typed ones; the Graph cannot be written without them, so this ADR relies on them and asks ADR-0010 to say so (foundation ask 2).
- Laws are linear or piecewise-linear integer tables; no `sqrt`, `exp`, `pow`, `sin`; per-step decay is `x -= floor_div(x * k_ppm, 1000000)` (LS-1 (g)). No float, no `Vector2`, no `Vector2i` (32-bit components, ADR-0005 P17) in any Graph signature or column.
- **Overflow is designed out, not detected.** `src/shared/graph/graph_products.gd` holds the D.13 product list as code (`PRODUCTS`: ordered operand-name pairs, running three-factor products asserted pairwise) and `product_max_assertion(tuning) -> int` (0 on success, else the index of the first pair with `max_a x max_b >= 2^62`). The maxima are `GraphTuning.max_*` constants; the loader calls the function in stage 1 and a violation is `GRAPH_BOOT_PRODUCT_OVERFLOW(a, b)` (domain 2). The worst product is `G_OPEN x A_m x abs(g)` with thin margin (D.13); raising `GAS_SCALE` or `V_max` trips boot rather than wrapping in play.
- `GraphView.step_no()` is `IntMath.floor_div(clock.tick(), tuning.graph_step_ticks)`, a pure function of `T` available at any call site (Failure & Trail compares it with its poll cache, FT-15; the Item DB's `validate_instance` compares against it, TR-itemdb-014). `hdr.last_step_no` is the last step whose failure hook (phase 11) completed (LS-8, ADR-0012 § 1) and is persisted for restore. It trails `step_no()` by up to one step, and equals it only from the end of the failure hook until the next power-hook tick.

### 3. Tables and columns (ADR-LSG-01)

**3.1 The table set.** Seven tables, a def-name pair and a scalar header. Rows of nodes, ports, edges, compartments, seams and sections are **dense, append-only and ordered by ascending id**; a table's arrays all have exactly the table's row count (`size()` equals the row count; there is no spare capacity in persisted columns, so `duplicate()` of a column is the exact payload and a length mismatch is detectable). Structural mutators grow tables with `resize` (the only allocation outside the steady state).

| Table | Row = | Row order | Row count source | Bytes per row (columns of § 3.2 to § 3.8) |
|---|---|---|---|---|
| `nodes` | one node slot, tombstones kept | ascending `id` | `next_node_id` allocations | 93 |
| `ports` | one port of one node, contiguous per node | ascending `(node slot, local order)` | allocated with the node | 10 |
| `edges` | one edge slot, tombstones kept | ascending `id` | `next_edge_id` allocations | 73 |
| `comps` | one COMPARTMENT node (EXTERIOR has no row) | ascending `node_id` | one per COMPARTMENT node | 56 |
| `seams` | one SeamRecord, `CUT` records kept | ascending `id` | `next_seam_id` allocations | 63 (+ list entries) |
| `sections` | one section record | ascending `id` | section creations | 150 |
| `queues` | four parallel groups: `pending_draw`, `pending_load`, `pending_trace`, `draw_grant` | arrival order | bounded (§ 3.8) | 8 to 16 per entry |
| `defs` | the interned def-name CSR pair `off` (`int32`), `utf8` (`uint8`) | append-only | one entry per distinct `def_id` ever installed | n/a |
| `hdr` | named `int` scalars: individual `_h_*` members of `GraphState`, built into a Dictionary of `int` values by the snapshot (not a packed array) | n/a | n/a | n/a |

**Why a fixed `ports` CSR and not `MAX_PORTS_PER_NODE`.** A node's ports are the contiguous run `[port_first, port_first + port_count)` of the `ports` table. Port counts differ widely (a junction has many, a lamp one), so a fixed per-node maximum either wastes most of a large table or caps content; the run costs two columns on `nodes` and keeps the table dense. Ports are allocated with their node (from the section template's hardpoint), never inserted later, so the run never moves. **Why `comps` is a separate table keyed to a node:** a COMPARTMENT is a node (kind 1) so an OPENING edge can name it as an endpoint like any other node; its atmosphere columns live in `comps` (one row per COMPARTMENT node) so the node table does not carry five dead columns for 90 percent of its rows. **The compartment id `cid` is the compartment's node id** (one id space; § 4.4). EXTERIOR (kind 2, node id 0, section 0) has no `comps` row: its pressure and temperature are the constants `P_EXT` and `T_AMB_MC`.

**3.2 `nodes` columns** (`int32` unless noted):

| Column | Type | Meaning |
|---|---|---|
| `id` | i32 | node id, never reused |
| `kind` | u8 | `GraphIds.NODE_KIND` (LS-3) |
| `flags` | u8 | bit 0 TOMBSTONE, 1 HAS_WARN, 2 EXOGENOUS (last damage cause), 3 BREAKER_CLOSED (CABLE_JUNCTION), 4 FATIGUED (hull), 5 EXPOSED (hull), 6 STUB (hull stub of a RAGGED cut), 7 reserved (zero) |
| `section_id` | i32 | owning section; 0 only for EXTERIOR |
| `def_ix` | i32 | index into `defs` (0 = none: COMPARTMENT, EXTERIOR, HULL_SEGMENT, ABSENT, SALVAGED) |
| `state` | u8 | `GraphIds.NODE_STATE` 0 OK .. 5 ABSENT (LS-10) |
| `hp`, `hp_cap` | i32 | bp, 0 to `HP_FULL`; `hp_cap` defaults to `HP_FULL` |
| `forced_mode`, `forced_next` | u8 | forced mode and the cascade queue (LS-12), mode ids |
| `fail_since`, `warn_since`, `mode_since`, `isolated_at`, `thaw_until`, `patched_at` | i32 | the six step stamps of LS-3 (back-dated values may be at or below 0, so these are signed); `warn_since` is meaningful only while HAS_WARN |
| `mode_key` | u8 | `(mode << 3) \| sev` of the dominant descriptor, 0 = none (`GRAPH_BOOT_MODE_ID` keeps the largest mode id at or below 31) |
| `compartment_id` | i32 | cid of the room the node sits in (a COMPARTMENT's own id) |
| `port_first`, `port_count` | i32, u8 | the node's port run in `ports` |
| `stock` | **i64** | store content in the store's own unit: BATTERY dJ, O2_TANK mGU, RESERVOIR and FUEL_TANK mL; 0 for every other kind |
| `req`, `req_max` | i32 | the owner-set request and its copy-at-install ceiling: LOAD and SCRUBBER `req_w`/`req_max`, PUMP `throttle_pm`, VENT `setpoint_pp`, REACTOR `out_w` |
| `aux_a`, `aux_b` | i32 | kind payload: REACTOR `fuel_rem`, `starve_steps`; LOAD `energised_at_step`, 0; RESERVOIR `t_cool`, `tc_rem`; HULL_SEGMENT `bound_compartment`, `tiles`; all others 0 |
| `tier`, `subtype` | u8 | LOAD priority tier and subtype (HEATER, LAMP, DOOR_MOTOR, ...); 0 elsewhere |
| `n_served` | i32 | **hook product** (CT-04, ADR-0012 § 11): the node's power satisfaction in permille (`sat_k` of its tier), written by phase 2 only and read by `served_pm(id)`; range 0 to 1000 (the overload ratio is `load_pm`, § 9.2); canonical zero on every node that is not a power consumer (`GraphSchema.is_power_consumer(kind)`) and on a tombstone, ABSENT or SALVAGED row |
| `n_draw` | i32 | **hook product**: power actually delivered to the consumer by the last phase 2, dJ per step (W, § 2.1); range 0 to the D.13 maximum for W in `GraphTuning`; written by phase 2 only; the same canonical zero |

**The overlay is normative**: each kind-specific reading of `stock`, `req`, `req_max`, `aux_a`, `aux_b`, `tier`, `subtype` is exposed only through a named accessor on `GraphView` or a writer (`reactor_fuel_rem(slot)`, `reservoir_t_cool(slot)`, `hull_bound_compartment(slot)`), never by raw column index outside the writer and snapshot files. (The stamp columns `thaw_until`, `isolated_at`, `warn_since` and `mode_since` are listed here and in § 3.4 as the pre-lever schema; § 3.11 pre-commits their move to a fluid-stamp side table.) The alternative (a side table per kind) costs an extra row-index map per table for the sake of a few bytes per row on about 105 nodes (Alternative 5).

**3.3 `ports` columns:** `type` (i32, `GraphIds.PORT_TYPE`, LS-4), `cap` (i32, loop units per step, copied at install; **retained when a module is salvaged or a seam is cut** so a SEVERED stub leaks at `OPEN_END_RATE_PM` of the port's cap), `local` (u8, `local_index`), `flags` (u8: bit 0 OPEN_END, bit 1 CAPPED).

**3.4 `edges` columns** (`int32` unless noted): `id`; `kind` (u8, LS-5); `loop` (u8); `flags` (u8: bit 0 TOMBSTONE, 1 HAS_WARN, 2 EXOGENOUS, 3 OPEN (VALVE), 4 CAPPED, 5 EXPOSED (DUCT), 6 STRAIN_GAS, 7 STRAIN_HEAT (the two SEAL_STRAIN hysteresis bits of a stored OPENING, D.14)); `a_node`, `b_node` (node ids), `a_port`, `b_port` (u8 local indices); `section_id`; `hp`, `hp_cap`; `state`, `forced_mode`, `forced_next`, `mode_key` (u8); `fail_since`, `warn_since`, `mode_since`, `isolated_at`, `thaw_until`, `patched_at`; `seam_id` (0 = none); `cap` (loop units per step); `e0` (DUCT `len`; OPENING `area`; CABLE net load, below), `e1` (OPENING `area_max`). OPENING `door_state` is derived from `area` (LS-19) and never stored. **`e0` of a CABLE is its net load**, a hook product written by ADR-0012 phase 2 only (CT-04): in W (dJ per step, § 2.1), a **non-negative magnitude** (the sign convention of § 2.3: every flux is a magnitude, and the direction is implied by the POWER tree rooted at the island's lowest node id and is never stored), range 0 to the D.13 maximum for W in `GraphTuning` (it may exceed the cable's `cap`; that excess is the overload `ρ`); canonical zero for a tombstone and for a cable with no consumer below it. **Edge endpoints are persisted as ids**, not slots; the endpoint slots are DERIVED (§ 4.2).

**3.5 `comps` columns:** `node_id` (i32, the cid), `room_ref` (i32, `layout_room_ref`, Layout's room id), `n_o2`, `n_inert` (**i64**), `t` (i32 mC), `t_rem` (i32), `fuel_puddle` (i32 mL), `band_o2`, `band_p`, `band_t` (u8, hysteresis memory), `p_prev` (i32, the previous run's `P`), `warn_o2`, `warn_p`, `warn_t` (i32 step stamps), `warn_bits` (u8). Derived and never here: `V`, `exposed_tiles` (Layout), `P`, `p_o2`, `o2_pm`, `pressure_trend`.

**3.6 `seams` columns:** `id`, `section_a`, `section_b`, `q`, `hp` (the seam's weld quality x 10), `leak_bp`, `l_seam`, `built_step`, `mode_since` (i32); `mode_key`, `state` (ACTIVE, CUT), `boundary` (SEALED, RAGGED; 0 while ACTIVE) (u8); and three CSR pairs into flat lists `edge_ids`, `opening_ids`, `stub_ids` (`edge_first`/`edge_count`, `open_first`/`open_count`, `stub_first`/`stub_count`, all i32). A SeamRecord is never deleted: a cut turns it into a tombstone with state CUT, its lists intact (the cut-hole ids `seam_view` reports). **The hull stubs of a RAGGED cut are ordinary HULL_SEGMENT node rows appended at the cut** (flags STUB, `hp` 2500 as LS-16, `aux_a` the bound compartment), listed in `stub_ids`; the GDD's "hull-stub records ... no new persisted field" is therefore satisfied by existing columns, and `cut_detach` can append rows, so `can_detach` must include `CAPACITY`, as the Layout GDD already lists it (`can_detach` reasons, amended 2026-10-27; ADR-0012/0021 implement it). Stub rows take fresh high ids outside the section's contiguous M1 range, so `section_counts_into` (§ 9.5) reports them as `stub_count` and excludes them from the template-range counts.

**3.7 `sections` columns:** `id`, `dormant_since`, `rate_pm`, `stress_pm`, `stress_until` (i32); `kind` (u8: HOME, DERELICT); `flags` (u8: bit 0 DORMANT); and sixteen `int64` counters: `vented_mgu`, `radiated_dj`, `spilled_cool_ml`, `produced_mgu`, `breathed_mgu`, `produced_dj`, `burned_uml`, `spilled_heat_mdj`, and `disc_` and `wd_` for each of the four media (gas mGU, coolant mL, fuel mL, energy dJ): `disc_mgu`, `disc_cool_ml`, `disc_fuel_ml`, `disc_dj`, `wd_mgu`, `wd_cool_ml`, `wd_fuel_ml`, `wd_dj`. (LS-21 names `discarded_*` and `withdrawn_*` "per medium" and M16 is legal on a BATTERY in dJ; the energy medium is the fourth. Open Question 4 asks the GDD to confirm.)

**3.8 `queues`, `defs`, `hdr`.**
- `pending_draw`: `draw_comp` (i32 cid), `draw_crew` (i32), `draw_mgu` (**i64**), arrival order, at most `MAX_CREW` x compartment-count entries.
- `pending_load`: `load_node` (i32), `load_req` (i32), arrival order, at most `GraphTuning.load_queue_max` (TR-lsg-017: 64; a registry gap, § Open Questions 5).
- `pending_trace`: `trace_kind` (u8), `trace_subject` (i32), `trace_mag` (i32), arrival order, at most `MAX_FOOTPRINTS_PER_STEP` per run across callers.
- `draw_grant`: `grant_crew` (i32), `grant_req`, `grant_got` (**i64**), the last applied grants, one per crew member.
- `defs`: `off` has `def_count + 1` entries with `off[0] = 0` and `off[1] = 0` (entry 0 is the empty string, so `def_ix` 0 means none); entry `k` is `utf8[off[k] : off[k+1]]`. Append-only; an entry stays after its module is salvaged.
- `hdr`: `next_node_id`, `next_edge_id`, `next_seam_id`, `next_section_id`, `last_step_no`, `last_step_power`, `last_step_atmo`, `last_step_failure`; the five-step power-use window `power_use_w0` to `power_use_w4`, collectively `power_use_w5` (ADR-0012's `delta_w_last5`, oldest first; a hook product written by phase 2 only, which Attention consumes; signed, bounded by the D.13 maximum for W); and the schedule in force `sched_div`, `sched_off_power`, `sched_off_atmo`, `sched_off_failure`, collectively `sched` (`GRAPH_STEP_TICKS` and `hook_offset_power/atmo/failure` as they were when the cursors were written; checked by § 7.2 step 13, reconciled by § 7.4). Seventeen values, **individual named `int` members `_h_*` of `GraphState`**, which the snapshot copies into one Dictionary of `int` values and the restore reads back into the members; they are not a packed array of slots: seventeen values do not justify a slot-order document, a restore diff names the key, and ADR-0005 § 3.6's "more than about 250 scalar fields" threshold is about arrays of rows, not a header. `last_step_X` is the `step_no` of the last completed run of hook X (ADR-0012 § 1); **`last_step_no` is the last step whose failure hook completed** (LS-8, § 2.3), so it trails `step_no()` by up to one step.

**3.9 Canonical zeros.** A row has exactly one encoding per logical state: **every column that does not apply to a row's kind or state holds the canonical zero** (0, or `HP_FULL` for `hp_cap` of a live subject only where LS-3 defines it). A mutator that changes a row's kind-dependent role (install, salvage, discard, cut) zeroes the columns the new role does not use; a tombstoned row keeps `id`, `kind` and `section_id` and zeroes everything else; `GraphSchema` records the "zero when" predicate per column and the restore validator and a debug check enforce it (a non-zero in an inapplicable column is `GRAPH_RESTORE_INVARIANT(canonical_zero)`).

**3.10 Payload shape.** `snapshot(ctx)` calls `ctx.put(payload)` where `payload` is **one Dictionary with `String` keys**: `"hdr"` (a Dictionary of the seventeen `int` values of § 3.8) and one nested Dictionary per table (`"nodes"`, `"ports"`, `"edges"`, `"comps"`, `"seams"`, `"sections"`, `"queues"`, `"defs"`) mapping column name to a **duplicate** of the packed column (ADR-0003 snapshot contract). Nested, not flat `"nodes.hp"` keys: the key text is shorter, the restore validator iterates tables by name, and ADR-0005's sorted key order applies at each level, so the hash order is `comps, defs, edges, hdr, nodes, ports, queues, sections, seams` and within a table the column names ascending. **Payload keys are `String`, never `StringName`**: the Save/Load walk rejects `StringName` as a key or value, and a `StringName` key round-trips as a `StringName` through `var_to_bytes` (probe (h)); the `GraphSchema` name constants are `String` constants. Renaming, adding, removing or re-typing a table, column or header key is a `schema_version` bump and a migration function (§ 7.5). `GraphSchema.fingerprint()` returns H64 over the sorted `table.column:type` lines; the committed value in `tests/fixtures/graph/schema.fingerprint` may change only together with a `schema_version` bump (CI), mirroring ADR-0006 § 2.6.

**3.11 Size.** The probed fixture of Engine Compatibility (i) gave **29.7 KB canonical bytes (29,674) for 105 nodes, 150 edges, 270 ports and 14 compartments** before this revision (the loaded set home plus both derelicts at the weld, about 250 subjects as LS-26 and S3 count them; the `defs` assumption of 60 offsets and 1,500 utf8 bytes is stated in (i)). The hook products add `n_served` and `n_draw` (8 bytes on each of 105 node rows, 840 bytes) and the power-use window (about 40 bytes); CABLE `e0` reuses an existing column. The loaded-set budget is therefore **about 30.5 KB** (29,674 + 840 + 40 = 30,554; the four `sched` ints add about 100 bytes of key text and values, so plan on 30.7 KB) against the 32,000-byte ceiling of TR-lsg-036 (ADR-0012's budget; 1 KB is 1,000 bytes there): about **4.6 percent headroom (about 4.2 percent with `sched`)**. The ceiling is reached about 6 nodes and 9 edges above the roster, near 111 nodes and 159 edges (the original figure of 114 nodes, nine above the roster, was computed before the hook-product columns). The MVP roster fits; a fourth derelict loaded at once does not.

**Lifetime-worst, not only at-weld (T8).** The fixture above is the roster at the weld. Rows are never removed: tombstones persist in the payload, a RAGGED cut appends stub HULL_SEGMENT rows, cut seams keep their lists, `defs` never shrinks, and the queues can sit at their bounds. ADR-0013 § 6 supports exactly one weld-then-cut cycle per socket (`CAPPED` and `CUT_OPEN` never return to `FREE`), and that rule is the bound that makes tombstone growth finite at the MVP. By row arithmetic one RAGGED-cycled socket leaves about 0.6 to 0.8 KB (a 63-byte seam record plus its list entries, its seam edges as 73-byte tombstones, two 93-byte stub rows; ESTIMATE), so a roster whose sockets have all cycled plausibly passes 32,000 bytes on top of 30.7 KB. **Decision: lever 1 is pre-committed, not held in reserve.** The fluid-stamp side table ships in the first `GraphSchema` (version 1 has no shared save yet, so this costs a design edit and no migration): `thaw_until`, `isolated_at`, `warn_since` and `mode_since` leave the `nodes` and `edges` tables for a `fstamps` table with one row per fluid subject (`GraphSchema.is_fluid_subject`), keyed by `(subject_kind, subject_id)`, with the canonical zeros of § 3.9; it saves about 1.8 KB at this roster. The tradeoff stated: one more table to snapshot, validate and fuzz, one more id-to-row map in `GraphCaches`, and a two-step read for four columns that are a direct index today. The schema commit that introduces `fstamps` applies the column moves to §§ 3.2 and 3.4 and commits the new fingerprint in the same commit; every byte figure in this section is the pre-lever figure. The remaining levers are (2) merging the two `int32` step stamps that are always read together and (3) asking ADR-0012 to raise the ceiling with the measurement. The **lifetime-worst fixture** (Validation: every socket welded and cut RAGGED once, queues at their bounds, maximal `defs`) decides whether lever 3 is needed; it is measured after lever 1, not guessed.

### 4. Append-only topology identity (ADR-LSG-04)

**4.1 The rules.**
1. **Ids come only from the four persisted counters** (`next_node_id`, `next_edge_id`, `next_seam_id`, `next_section_id`), which only increase. A counter is never decremented, not by an error path, not by a fault, not by a restore (a restore sets it from the payload, which the validator checks is above every id in the table, `GRAPH_RESTORE_COUNTER`). An id is never reused after its row is tombstoned or its section discarded.
2. **Slot order is id order.** A new row is appended at the end of its table with an id above every existing id; nothing is inserted or moved; nothing renumbers. There is **no compaction in the MVP**: removal is a tombstone flag (§ 3.9), and capacity (`MAX_GRAPH_NODES`, `MAX_GRAPH_EDGES`) counts slots including tombstones, which is what `CAPACITY` in `can_join` and M1 tests. VS-A's order-preserving compaction (LS-17) is out of scope; Risks item 6 records what it would break.
3. **A weld only appends** edge rows, one SeamRecord row, its list entries and its OPENING edges; it never touches an existing row's id, slot or endpoint (TR-lsg-027, data side). Its rows are appended in the order of the `JoinPlan`, which Layout canonicalises survivor-first (ADR-0013 § 6, § 7), so the seam rows of `graph.state` are identical for `attach(x, y)` and `attach(y, x)` (ADR-0013 records the whole-world `state_hash` equality as an assumption on ADR-0021's weld property test). `cut_detach` tombstones edge rows and flips the SeamRecord to CUT; a RAGGED cut also appends stub HULL_SEGMENT rows (§ 3.6) with fresh high ids outside the section's contiguous M1 range, which `section_counts_into` counts separately as `stub_count` (§ 9.5). `discard_section` tombstones every row of the section; mass leaves through the `disc_*` counters (TR-lsg-028, data side).
4. **Ids are bounded by construction:** every allocation tests `next_id < GraphLimits.ID_MAX` (2^24) and the table caps; an infallible mutator that would exceed either raises `GraphFaults.ID_SPACE_EXHAUSTED` (a defect, since the `check_*` that precedes it tests `CAPACITY`). 2^24 keeps `L/<loop>/<anchor>` and `F/<subject_ref>` keys inside the 32-bit key id (§ 10) and keeps `MAX_GRAPH_NODES` and `MAX_GRAPH_EDGES` inside the 24-bit id field of the RNG subject (ADR-0012 § 8.2: 2-bit kind, 24-bit id sized to `ID_MAX`, so a node and an edge draw key never overlap for any id the allocator can produce).
5. **A stale id is harmless.** An id that is negative, at or above its counter, was never allocated, or names a tombstone resolves to slot -1: a Class B mutator no-ops and returns 0/false, a Class A `check_*` returns its `UNKNOWN`/`NOT_*` reason, and a read returns **that getter's documented sentinel** (§ 9.1: `node_state` and `edge_state` return `GraphIds.STATE_NONE`, never the healthy 0; magnitudes return 0; `node_live`/`edge_live` are the guards a caller uses to decide liveness). Nothing indexes an array with an unchecked id.
6. **Validate every slot and index before the first column write.** An out-of-bounds packed-array read is a `SCRIPT ERROR` that logs and returns 0; an out-of-bounds write is a `SCRIPT ERROR` that **aborts the rest of the calling function** (probe (j)). Neither raises a `GraphFaults.*` code, there is no rollback, and `assert` is banned, so in a release export the result is a log line and a silently half-applied multi-column mutator. Therefore every mutator and every phase resolves and checks all slots and indices (rule 5, table caps, run bounds) before its first column write, and a mutator that cannot finish must not have started. CI treats any `SCRIPT ERROR` line in the log as a failure (the ADR-0010 `load_all` gate), and the restore fuzz and the stale-id mutator tests run with the log scanned (Validation).

**4.2 Id-to-slot is derived.** Persisted references between rows are ids (edge endpoints, `compartment_id`, `section_id`, `bound_compartment`, queue subjects). `GraphCaches` holds the dense maps `slot_of_node[id]`, `slot_of_edge[id]`, `slot_of_seam[id]`, `slot_of_section[id]` (`PackedInt32Array` indexed by id, -1 when absent, sized to `next_id`; private to the cache and exposed only through the methods `slot_of_node_id(id)`, `slot_of_edge_id(id)`, `slot_of_seam_id(id)`, `slot_of_section_id(id)`, each returning -1 for a stale id), the endpoint slot columns `edge_a_slot`, `edge_b_slot`, and the lookup-only `Dictionary[int, int]` `comp_of_room_ref` (never iterated). All are rebuilt eagerly by `ensure_derived()` after every structural mutator and in `post_restore` (§ 6), so a read never rebuilds in an arbitrary presentation frame. While no compaction exists, id equals slot for nodes and edges allocated contiguously, but **no code may assume it**: every access goes through the map, so VS-A compaction changes the cache and the allocator, not the callers.

**4.3 Cross-block references to Graph ids.** Layout stores `graph_ref_base {node_base, edge_base, comp_base}` per section; Interaction and the Perception Filter hold node and edge ids; Suit holds a compartment id through `actor_section`. All are plain ids resolved through `slot_of_*`; a Graph id never leaves the block as an array position (ADR-0005 § 3.6).

**4.4 Compartment identity.** `cid` is the **node id of the COMPARTMENT node**. M1 allocates a section's node ids contiguously in template order with the section's COMPARTMENT nodes first, in Layout room order, so `comp_base == node_base` and `cid(room k) == comp_base + k`; the returned `SectionBases {node_base, edge_base, comp_base}` (ADR-0013's out record, filled into a caller-owned object) keep the shape LS-18 and Layout's `graph_ref_base` expect. **M1's only caller is Layout LM1 through `LayoutGraphPort`; LM1's callers are those ADR-0013 § 9 names (Web Map's `init_world` in the `INIT` scope at the MVP)**, so Layout allocates `section_id` first and M1 receives it as an argument. `layout_room_ref` is Layout's block-local room id, stored in `comps.room_ref`; Layout owns `section_id` and `room_id` allocation and M1 takes `section_id` as an argument and only advances `next_section_id` to `max(section_id) + 1` (cross-ADR assumption on ADR-0013). `section_id` 0 is reserved for the world-level EXTERIOR; real sections start at 1.

**4.5 Ports.** A port is identified by `(node id, local_index)`; its row is in the node's run and the run never moves. SALVAGED and ABSENT nodes keep their ports (type, cap, local index) so an M5 reinstall can match the hardpoint.

**4.6 `default_state`.** The EMPTY block has the EXTERIOR node row (id 0, kind 2, section 0, state OK, no ports), the four counters at 1, `last_step_*` and the power-use window at 0, `sched` taken from the live `GraphTuning` (so a fresh block never counts a `schedule_shift`), and every other table empty (LS: "no sections, counters at 1"). A registered-but-unrestored block therefore hashes the same on every run. **Mechanism (ADR-0016 § 3.3):** `default_state(ctx)` builds this state as a payload with `GraphSchema` keys, `ctx.put`s it, and the kernel applies it through `GraphRestore.restore`; the payload passes every § 7.2 step (with no completed tick, every cursor is the step-0 value at step 13), and KRN-RST-07 tests that path.

**4.7 Stable type ids.** `src/shared/graph/graph_ids.gd` (`GraphIds`) holds every enum that reaches a payload, as `const` integers: `LOOP` (0..5), `NODE_KIND`, `PORT_TYPE`, `EDGE_KIND`, `MODE` (1..10 and 20..27), `NODE_STATE` (0..5), `BAND_O2`, `BAND_P`, `BAND_T`, `SEAM_STATE`, `SEAM_BOUNDARY`, `SECTION_KIND`, `LOAD_TIER`, `LOAD_SUBTYPE`, `TRACE_KIND`, `SUBJECT_KIND`, `CAUSE`, `MEDIUM`, `ACOUSTIC`, plus the two read-side enums `TRUE_VALUE` (§ 9.2) and `LOAD_CASE`, which are not payload enums and so are not in the lock file. ADR-0007 § 3.2 already makes these the baseline that `assets/data/graph/` must equal (`GRAPH_BOOT_TYPE_ID`, domain 2) and gives them no ledger, assigning their stability to this ADR. The stability mechanism: **a committed `assets/data/graph/ids.lock.json`** in the ADR-0007 ledger format (`space`, `format`, `state`, `pairs`, `retired_ids`), one `space` per enum above, generated from `GraphIds`, checked with the existing `LedgerCheck.verify` at boot and by the `LEDGER_NOT_APPEND_ONLY` base-revision gate in CI (WARN while `PRE_RELEASE`, ERROR once `SHIPPED`, flipped by the same one-way commit as the other ledgers). A new value is a new number; a retired value goes in `retired_ids`; an existing pair never changes. This adds one lock file and no new mechanism to ADR-0007 (foundation ask 4). The persisted enums (`band_*`, `state`, `boundary`, ...) are in the lock because they are written into saves, not because content authors extend them. **`SUBJECT_KIND` ordinals are fixed to the descriptor visit order of § 9.3: NODE 1, EDGE 2, COMPARTMENT 3, SEAM 4, OPENING 5** (the lock file freezes whatever is written first, so these are written as such). `GraphIds` also carries `ALARM_CAPABLE_MASK` (bit `m` set for each alarm-capable mode `m`: modes 1 to 5, 7 to 9 and 20 to 23; a code-side rule that boot checks against the mode table, ADR-0012 § 7) and `STATE_NONE` (-1, the not-live sentinel of § 9.1: outside every enum domain and never a payload value, so it is not in the lock file). **Always reference these as `GraphIds.<NAME>`:** the unqualified `OK` and `FAILED` are `@GlobalScope` `Error` constants (a `GraphIds` constant named `OK` or `FAILED` compiles without `SHADOWED_GLOBAL_IDENTIFIER`, but a bare use outside the class resolves to the global one).

### 5. Node-slot lifecycle: what carries, what resets, what is read live

Item DB IDB-18 and Interaction/Assembly use M5 `install_module`, M6 `salvage_node` and M7 `repair_step`; the Item DB calls no mutator. The columns of § 3.2 behave as follows (TR-itemdb-016, 031, 040; ADR-0021 orders the calls, this table is the data contract):

| Column group | M1 `instantiate_section` | M5 `install_module` (ABSENT or SALVAGED to OK) | M6 `salvage_node` (to SALVAGED) |
|---|---|---|---|
| identity (`id`, `kind`, `section_id`, `compartment_id`, ports run) | set from template | untouched | untouched |
| `def_ix` | interned from the template (0 for a bare authored VENT/LOAD, which M6 refuses with `NOT_SALVAGEABLE`) | interned from the installed `def_id` | cleared to 0 (the `SalvageRecord` carries the `def_id` string) |
| `hp`, `hp_cap`, `patched_at` | template (`hp_start`; `hp_cap` `HP_FULL`; `patched_at` 0) | **from the arguments**: `hp_start`, `hp_cap`, `patched_at_step` (a patched module returns capped and keeps its absolute decay clock; only a spent repair part, M7 FULL, lifts a cap) | recorded in the `SalvageRecord`, then zeroed |
| `stock` | template (a store installs empty unless authored) | **0** | the remaining stock moves into the section's `disc_*` counter (conservation closes), then **0** |
| `req`, `req_max` | template defaults | **copy-at-install**: `req` and `req_max` from the def (`req_w`, `req_max`); setpoints (`throttle_pm`, `setpoint_pp`) take the Graph's fresh defaults, never the salvaged module's | zeroed |
| port `cap` | template | **copy-at-install** from the def's ports | retained (SEVERED stub rate) |
| port `flags` | template | OPEN_END and CAPPED cleared | OPEN_END set on fluid ports (the connected edges' ends are SEVERED) |
| `state`, `forced_mode`, `forced_next`, `mode_key`, `fail_since`, `warn_since`, `mode_since`, `isolated_at`, `thaw_until`, HAS_WARN | template | OK on install, then **re-derived from `hp` by the next phase 9**: FAILED when `hp` is at or below `HP_FAIL`; a reinstall never raises `hp` and never clears an hp-caused failure; stamps and flags zeroed | `state` SALVAGED, the rest zeroed |
| kind payload (`aux_a`, `aux_b`, `tier`, `subtype`) | template | from the def where a def field applies (LOAD tier and subtype), else 0 | zeroed |
| hook products (`n_served`, `n_draw`) | 0 | 0 (written by phase 2 only, from the next solve) | zeroed |

**Read-live fields** (`waste_permille`, `out_w_rated`, `stock_cap`, `repair_part_id`, the emits record) are **never copied into a persisted column**; they are read through `def_ix` to the immutable `ItemDef` at each use. A hot loop may read them through a DERIVED per-slot column (`caches.def_waste_pm[slot]`, `caches.def_out_w_rated[slot]`) rebuilt by `ensure_derived`, which is a cache of the frozen catalog and not a copy of state. Because the catalog is frozen for the process lifetime, a catalogue tuning edit takes effect at the next start through exactly that rebuild, so a save restores unchanged (TR-itemdb-032).

### 6. Persisted, derived, scratch, diagnostic

**6.1 Store extensive, derive intensive** (LS-1 (c), TR-lsg-004). Persisted: the extensive quantities and the hysteresis memory. Derived: everything a pure function of persisted state, the catalog and Layout.

| Class | Members |
|---|---|
| **PERSISTED** (`GraphState`) | `n_o2`, `n_inert`, `t`, `t_rem`, `fuel_puddle`; `stock`, `tc_rem`, `t_cool`, `fuel_rem`, `out_w`; `hp`, `hp_cap`, `state`, `forced_mode`, `forced_next`; the six step stamps and `mode_key`; `thaw_until`; the band memory `band_o2/p/t`, `p_prev`, the warn trackers `warn_o2/p/t`, `warn_bits`; the two STRAIN bits per stored OPENING; `req`, breaker, valve `open`, `area`, `capped`, `open_end`; the section counters; the `pending_*` and `draw_grant` queues; the id counters, `last_step_*` and the schedule in force `sched_*`, the def table |
| **PERSISTED: hook products** (`GraphState`) | `n_served`, `n_draw`, CABLE `e0` (net load), `power_use_w0` to `power_use_w4` (collectively `power_use_w5`); **written by ADR-0012 phase 2 only**. They are the products of one hook that a later hook of the same step, or another block between hooks, reads and that the post-phase state can no longer reproduce (ADR-0012 § 11) |
| **DERIVED** (`GraphCaches`) | `P`, `p_o2`, `o2_pm`, `pressure_trend`; `V`, `exposed_tiles`; components, trees, `path_cap`, the POWER island list; the loop summaries `loop_sum`; the active lists `active_nodes`, `active_edges`, `active_comps`, `active_seams`, `store_slots`, `machine_slots`, `exchange_links`; the failure-descriptor store and `alarm_sections`; last-emitted dirty buckets; id-to-slot maps and endpoint slots; the def-row and `def_waste_pm` columns; `topology_version`, `topo_dirty`, lifecycle (`EMPTY`/`LIVE`). **`served_pm` is not derived**: it is read from the persisted `n_served` |
| **SCRATCH** (`GraphScratch`) | the ADR-0012 § 3 inventory, sized by `size_scratch()` in structural mutators and `post_restore` and **all zero at every hook exit**: the `PackedInt64Array` accumulators `acc_n_o2`, `acc_n_inert`, `acc_q`, `acc_vent_mgu`, and the island (`island_*`), candidate (`cand_slot`, `cand_mode`), sort (`q_key`), traversal (`stack`, `queue`) and degree (`deg`, read through `deg_max_of(e)`) buffers. The swap-by-reference rule (§ 6.2) remains available for a future field that is rewritten wholesale |
| **DIAGNOSTIC** (`GraphDiag`) | `saturations` (indexed by a `SatField` enum), `stale_calls`, `gate_holds`, `stall_edges`, `footprints_dropped`, `stock_clamped`, `hash_invocations`, `late_rebuilds`, `queue_dropped`, `footprints_clamped`, `schedule_shift` |

`P`, `p_o2`, `o2_pm` and the trend are pure functions of `n`, `V` and the persisted `p_prev`: they are recomputed at the end of their phase, at the end of phase 0 (the recompute ADR-0012 adds so that a restore at any tick rebuilds the exact value) and in `post_restore`, and a read returns the cached value, so `read_view` writes no persisted column (TR-lsg-018) and the first presented frame after a restore is correct. `run_pm` and `circ` are pure functions of persisted state computed where they are read (ADR-0012 § 11), not cache members. **`served_pm` is not recomputed**: a restore landing between the power hook (6n+0) and the atmo or failure hook (6n+2, 6n+4) cannot rebuild `served`, `draw` or a cable's net load from post-phase state (batteries are debited, the thermostat duty has moved, a cable may have failed, M10 may have changed `req`), and the restored run would diverge in `hp` by one step's damage (ADR-0012 § 11). So `served_pm(id)` reads the persisted `n_served`; the GDD's "`served_pm` is not persisted, because phase 2 recomputes it before use" holds only for phase 2 of the next step (GDD sync).

**6.2 Aliasing and the scratch rule.** S0-a, the packed-array aliasing question the GDD left open, is **closed for the data model by ADR-0005 P7 and probes (c), (d), (m)**: a `Packed*Array` Variant shares one reference, so assignment, argument passing and a getter all alias; an element write **and a `resize`** through any alias are visible through all (probe (m): after `al = a`, `a.resize(5)` gives `al.size() == 5`); only `duplicate()` isolates (by deferred copy-on-write, probe (e)), and only **re-assigning the member itself** (a swap, or restore phase 2) leaves an earlier alias pointing at the previous buffer. Consequences, as rules: **(1)** a function may take a local alias of a column for reads and writes within one phase; **(2)** a local alias is invalidated only by re-assigning the member (swap, restore phase 2); `resize` and element writes are visible through every alias, so the rule "never keep an alias across a kernel call, a `snapshot`, a `restore` or a structural mutator" is safe but conservative, and its cause is member re-assignment, not `resize`; **(3)** a snapshot is never a local alias: it is a `duplicate()` placed in the payload (ADR-0003), and a payload column that is not duplicated would see later live writes (probe (m)); **(4)** *superseded by ADR-0012 § 3 (accumulator apply).* ADR-0012 replaces the Jacobi `next` columns with `PackedInt64Array` delta accumulators in `GraphScratch`, applied to the live column in one ascending loop (`col[i] += acc[i]; acc[i] = 0`), so the live column is itself the start-of-phase snapshot and nothing is swapped or stranded. `GraphScratch` holds that inventory (§ 1, § 6.1) and is all zero at every hook exit. The swap-by-reference rule (`var t: PackedInt64Array = state_col; state_col = scratch_next; scratch_next = t`, allocation-free, probe (c)) remains available for a future field that is rewritten wholesale; anything holding a column reference reads it through the member each time (the `GraphView` facade reads `state` members, never a cached array), so a swap or a restore cannot strand a reader.

**6.3 The persisted-field check (TR-lsg-038).** Two CI checks, both in `tests/unit/graph/`:
- **Method A, reflection.** Enumerate `GraphState.new().get_property_list()` filtered by `PROPERTY_USAGE_SCRIPT_VARIABLE` (probed: includes underscore members, in declaration order). Every name must be either a column that `GraphSnapshot.build()` writes into the payload (the check builds a payload from a populated state and compares the key sets against `GraphSchema`; the `_h_*` header members are compared with the `hdr` keys) or one of the three holder references `_caches`, `_scratch`, `_diag`. The same enumeration over `GraphCaches`, `GraphScratch` and `GraphDiag` compares against the checked-in allowlist `tests/fixtures/graph/derived_members.json` (one entry per member: reason and rebuilding function), which fails on any member not listed and on any listed member that does not exist. The file carries the members ADR-0012 adds (§ 12.3) and has no `served_pm` entry (it is persisted, § 6.1).
- **Method B, zero and diff.** For each `GraphCaches` member, zero it (or `reset()` the cache) in a copy of a mid-fault state, run one step, and assert the copy's `state_hash` equals the unmodified control's after `ensure_derived()` has been allowed to run; a step that reads a cache it did not first recompute shows as a differing hash. `GraphScratch` members are cleared the same way. Both must pass.
- **Method C, no `static var` and no back-reference.** `GraphState`, `GraphCaches`, `GraphScratch` and `GraphDiag` declare **no `static var`**: reflection does not list one (probe (k), so Method A could never notice a persisted-looking static), and a static is shared by every instance, including the fresh scratch `GraphState` that `restore` builds, which would break the two-phase restore. Constants and static funcs only. The check is part of OWN-01's lexical rule (§ 12.1) or, until it lands, one line of the reflection test that scans the four files for `^static var`. And `GraphCaches`, `GraphScratch`, `GraphDiag` and `GraphView` hold **no field of type `GraphState`**: `RefCounted` reference cycles leak (probe (l)), and `GraphState` holds the three holders strongly. Ownership is one-way (`GraphModule` to `GraphState` to the three holders) and writers receive the state as a parameter, as ADR-0012's phases do. The test constructs and frees a full `GraphModule` (with its `GraphLayoutAdapter`, § 9.5) in a headless test and asserts no leak warning.

**6.4 Conservation (TR-lsg-005).** The sixteen section counters of § 3.7 are the persisted half of LS-1 (e). `src/shared/graph/graph_invariants.gd` holds pure evaluators over a `GraphState`, read only: `gas_total(state, section_id)`, `heat_total_mdj(state, section_id)`, `coolant_total(state, section_id)`, `fuel_total_uml(state, section_id)`, each the left side of the LS-1 (e) equality for that medium (gas: `n_o2 + n_inert` over compartments, plus O2_TANK stock, plus `vented_mgu`, `breathed_mgu`, `disc_mgu` and `wd_mgu`, minus `produced_mgu`; heat in milli-dJ over compartments' `T x C + t_rem` and every RESERVOIR's heat store, plus `1000 x (radiated_dj - produced_dj)` and `spilled_heat_mdj`; coolant and fuel as LS-1 (e) gives, fuel in micro-mL with `fuel_rem` subtracted). **A test asserts each total equals its value at the section's instantiation after every step of every fixture, on a running ship with sources, sinks and crew active.** The baseline is held by the test (it is a function of the template), not persisted, which keeps the LS-21 field list unchanged. The exact term lists stay in the GDD; the evaluators are the executable form and are what a reviewer diffs against LS-1 (e).

**6.5 Perception independence (TR-lsg-018).** `src/sim/graph/**` imports nothing from `src/presentation/` (`ARCH-02`), names none of `PERCEPTION_OUT`, `perceived(`, `PerceptionRef` (ADR-0010 `READ-03` token list for the directory), and requests no `PERCEPTION_OUT` view; a mutator argument is an id, a magnitude, an enum or a plan (LS-22). The fixture of LS-22 (a degraded ship with and without a Perception block set to "all OK") asserts the `state_hash` of `graph.state` is identical.

### 7. Snapshot, restore and `post_restore`

**7.1 Snapshot.** `GraphSnapshot.build(state, out)` walks `GraphSchema` and, for each table and column, places `column.duplicate()` into the nested Dictionary, then `ctx.put(payload)`. It reads nothing but `GraphState`, includes no derived or diagnostic member, and the same payload feeds `canonical_encode` (the kernel encodes; the Graph has no byte writer, ADR-0005 § 3.1). A snapshot may be taken on any tick, which is why the queues, `draw_grant`, hook cursors, hysteresis memory, hook products and the schedule in force are persisted. The `hdr` Dictionary is built here from the `_h_*` members (§ 3.8).

**7.2 Restore** (`GraphRestore.restore(data, stored_version, ctx)`, two-phase, ADR-0006 § 2.2). Phase 1 decodes `data` into a fresh scratch `GraphState` (every column `duplicate()`d out of `data`, no reference kept into it) and validates it. Phase 2, on success only, assigns the scratch columns into the live `GraphState` member by member (the object is never replaced, so facades stay valid) and calls `GraphDerived.reset_caches()` (the one cache write `graph_restore.gd` triggers, performed by the cache writer file). The scratch `GraphState` is dropped immediately after phase 2: the live members now share its buffers (assignment aliases, probe (m)), so nothing may keep it. Any failure is `ctx.fault(GraphFaults.RESTORE_<name>, detail)` with the live state untouched; there is no clamp and no partial apply. **Validation runs in a fixed order and the first failure wins**, so a given corrupt input always yields the same reason (the 1,000-mutation fuzz of the GDD depends on it):

| Step | Check | Reason |
|---|---|---|
| 1 | `stored_version` is 1 | `RESTORE_BAD_VERSION(stored, supported)` |
| 2 | exact table and column key sets (and `hdr` keys), `String` keys, expected element types | `RESTORE_BAD_TAG(key)` |
| 3 | all columns of a table have the same length; CSR runs (`ports`, seam lists, `defs`) lie inside their arrays | `RESTORE_BAD_LENGTH(field, expected, got)` |
| 4 | enum columns lie in their `GraphIds` domain | `RESTORE_BAD_ENUM(field, slot, value)` |
| 5 | numeric columns lie in `[min, max]` of their `GraphSchema` spec, including the hard ceilings `max_stock_*`; flag bytes use only defined bits; canonical zeros hold | `RESTORE_OUT_OF_RANGE(field, slot, value)`; a zero violation is `RESTORE_INVARIANT(canonical_zero)` |
| 6 | ids strictly ascending in nodes, edges, seams, sections | `RESTORE_ID_ORDER(slot)` |
| 7 | each `next_*` is above every id in its table | `RESTORE_COUNTER(counter, max_id)` |
| 8 | every live edge endpoint names a live node; no live edge points at a tombstone | `RESTORE_BAD_EDGE_ENDPOINT(edge_id)` |
| 9 | every seam sub-edge has its SeamRecord and every record's lists name rows of the right kind | `RESTORE_SEAM_MISMATCH(seam_id)` |
| 10 | queue lengths within bounds, queue subjects name live rows | `RESTORE_QUEUE(queue, length)` |
| 11 | every `defs` entry the nodes reference is known to the catalog (`ItemCatalog.has`, after `resolve_alias` once ADR-0007's alias story lands; the stored def string is never rewritten) | `RESTORE_UNKNOWN_DEF(def_id)` |
| 12 | any LS-24 structural check not named above | `RESTORE_INVARIANT(name)` |
| 13 | with `T_last` the last completed tick of the clock installed at ADR-0016 R2 (before R3) and the stored `sched`: for each hook X, `last_step_X` equals `floor_div(t_X, sched_div)`, where `t_X` is the largest tick at or below `T_last` with `posmod_i(t, sched_div) == sched_off_X`, or the ADR-0012 § 1 step-0 value (0) when there is none; and `last_step_no == last_step_failure` | `RESTORE_HOOK_CURSOR(hook, stored, expected)` |

Step 13 is a structural comparison of persisted stamps with the clock: it never derives simulation state from `T` and asserts no catalogue tunable (ADR-0016 § 3.2 and § 4.4), because the offsets compared are the ones stored in `sched`, not the live `GraphTuning`. Without it, a save whose cursor exceeds the step implied by `T` would restore successfully and the first hook would raise `HOOK_CURSOR` mid-play, turning a classifiable corrupt input into an in-play FAULTED. ADR-0016 V3 covers this step by reference to this table, and the 1,000-mutation fuzz reason table gains one row: a mutated cursor classifies as `RESTORE_HOOK_CURSOR`, after every earlier step. A schedule change across a save is not a restore fault; it is absorbed at the first hook run (§ 7.4).

**7.3 Out-of-range is hard, a cap tuning drift is not (resolves TR-itemdb-033 against LS-21).** Step 5's maxima are **physical hard ceilings** from `GraphTuning` (the `n_max(c)` of a room is checked in `post_restore`, § 7.4, because it needs `V`; a store's hard ceiling is `max_stock_*`), not the catalog's tunable `stock_cap`. A save whose store content exceeds a **lowered live `stock_cap`** is therefore valid: it restores byte-identically (so `snapshot(restore(snapshot()))` still holds and no REBASED cause is needed), and the **stock reconcile**, which runs at the head of every phase 0, clamps `stock` to the live cap, credits the excess to the section's `disc_*` counter (conservation closes), increments `GraphDiag.stock_clamped` (a legal event, **not** a saturation, so the all-zero saturations assertion stands) and emits a diagnostic naming node, old stock and cap. In steady state the reconcile is a no-op loop over the stores. The reconcile is a function of persisted state and the frozen catalog only, so replay stays deterministic. ADR-0016 § 4.4 agrees: the phase-0 reconcile is the decision, because it keeps the restore byte-identical (Open Question 2, resolved). **The window.** Between the restore and the first phase 0 (up to `GRAPH_STEP_TICKS - 1` ticks; for a section that restores dormant, until the section loads, since dormant sections are skipped, ADR-0012 § 10), readers and M16 `withdraw_store` see the over-cap stock as stored: `stock_pm(id)` clamps to 1000 (`mini(1000, ...)`), so no reader sees a store fuller than full, and M16 may withdraw over-cap stock in the window (it debits the stored amount and credits `wd_*`, so conservation closes and the later reconcile finds less to clamp). A `gdd_sync` item carves the case out of LS-21's "never by clamping".

**7.4 `post_restore`** (`GraphDerived.post_restore(ctx)`, runs after Layout's restore because of `restore_after`): **(1)** for each compartment row, look up `room_ref` through the `RoomSource` port; an unknown room is `RESTORE_UNKNOWN_ROOM(room_ref)` for the first such compartment (loud by design, no default `V`); **(2)** check `n_o2 + n_inert` against `P_MAX x V x GAS_SCALE` per compartment, `RESTORE_OUT_OF_RANGE("n_gas", slot, value)`; **(3)** rebuild every DERIVED cache (`ensure_derived()`, including the intensive values and the descriptor store, writing no persisted field and drawing no RNG): `post_restore` re-derives `P`, `p_o2`, `o2_pm` and the trend, and never `served`, `draw` or a cable's net load, which are persisted hook products (§ 6.1); **(4)** seed the last-emitted buckets from the restored values and the band and STRAIN memory from the payload (never re-derived from the values), so the first run reproduces an uninterrupted run and the kernel's one `ALL` batch is followed by silence (ADR-0012 owns the phase code that reads the seeds); **(5)** run the structural invariants of LS-24 in every build. Cross-block verifiers (INV-x) and the order among blocks' `post_restore` are ADR-0016's. A fault here fails the whole load (kernel `LOADING` to `FAULTED`), because `post_restore` runs after the swap; ADR-0016 must guarantee a faulted `LOADING` never enters play with the restored world (cross-ADR assumption).

**Schedule rule (CT-08).** The first run of each Graph hook after a load compares `hdr.sched` with the live `GraphTuning`. If they differ, it rewrites `hdr.sched`, skips the hook when `last_step_X` already equals the current step, and runs once otherwise; this is counted in `GraphDiag.schedule_shift`. It is legal only on that first run; from then on ADR-0012's `HOOK_CURSOR` check applies. A schedule change across a save is therefore handled at the first hook run, never faulted at restore (§ 7.2 step 13 compares the cursors with the schedule stored in the payload).

**7.5 Versioning.** `schema_version` is 1 and `supported_versions` is `[1]`. A future shape change registers a pure `(old Dictionary) -> new Dictionary` migration in `graph_restore.gd` and bumps both the version and the committed schema fingerprint in the same commit; retained golden saves in `tests/fixtures/saves/` are migration fixtures (TR-lsg-040, ADR-0006 § 2.6).

### 8. Faults

**8.1 Domain 3 table** (`src/sim/graph/graph_faults.gd`, `class_name GraphFaults`, `DOMAIN_ID = 3`, codes pre-packed `(DOMAIN_ID << 16) | local`, append-only, never reused; `NAMES` indexed by local code):

| Local | Name | Raised when |
|---|---|---|
| 1 | `NOT_IN_SCOPE` | (ADR-0003 § 6's example, kept) a mutator defect that is not the plain `REFUSED` |
| 2 | `RESTORE_BAD_VERSION` | § 7.2 step 1 |
| 3 | `RESTORE_BAD_TAG` | step 2 |
| 4 | `RESTORE_BAD_LENGTH` | step 3 |
| 5 | `RESTORE_BAD_ENUM` | step 4 |
| 6 | `RESTORE_OUT_OF_RANGE` | step 5 and § 7.4 (2) |
| 7 | `RESTORE_ID_ORDER` | step 6 |
| 8 | `RESTORE_QUEUE` | step 10 |
| 9 | `RESTORE_COUNTER` | step 7 |
| 10 | `RESTORE_BAD_EDGE_ENDPOINT` | step 8 |
| 11 | `RESTORE_SEAM_MISMATCH` | step 9 |
| 12 | `RESTORE_UNKNOWN_DEF` | step 11 |
| 13 | `RESTORE_UNKNOWN_ROOM` | § 7.4 (1) |
| 14 | `RESTORE_INVARIANT` | step 12, canonical zero |
| 15 | `STRUCT_ENDPOINT` | runtime: an edge endpoint slot out of range or a tombstone |
| 16 | `STRUCT_LENGTH` | runtime: column lengths of a table disagree |
| 17 | `STRUCT_ID_ORDER` | runtime: ids not strictly ascending |
| 18 | `STRUCT_COUNTER` | runtime: `next_id` at or below a live id |
| 19 | `STRUCT_CSR` | runtime: a port run or seam list inconsistent with its edges |
| 20 | `STRUCT_SEAM_RECORD` | runtime: a seam sub-edge whose record is missing |
| 21 | `ID_SPACE_EXHAUSTED` | an infallible mutator would pass `ID_MAX` or a table cap |
| 22 | `DESC_STORE_OVERFLOW` | phase 11 would exceed the descriptor store capacity (§ 9.3) |
| 23 | `STEP_RANGE` | `step_no` would pass `STEP_STAMP_MAX` |
| 24 | `SATURATION` | a numeric soft violation under `strict_numeric` (§ 8.3) |
| 25 | `HOOK_CURSOR` | runtime, raised by ADR-0012 § 1: at a hook head `step_no < last_step_X`, or `step_no == last_step_X` with `step_no > 0` (the clock moved backwards or the hook ran twice) |
| 26 | `RESTORE_HOOK_CURSOR` | § 7.2 step 13 |

The detail Dictionary carries `String` keys (`"field"`, `"slot"`, `"value"`, `"expected"`, `"got"`, `"subject"`); a fault is terminal and cold, so its allocation is acceptable. Codes 25 and 26 are appended in one batch: 25 is the runtime guard ADR-0012 raises and 26 is the restore check (CT-08; a reviewer's "restore check as local 25" is renumbered because 25 is the runtime code). Codes are append-only like the rest.

**8.2 Raising.** `GraphFault` (`graph_fault.gd`) is a `RefCounted` of statics with no state: `raise_in(scope, code, subject) -> int` calls `scope.fault(code, {"subject": subject})` and returns `-code`, so a Class A mutator writes `return GraphFault.raise_in(scope, GraphFaults.ID_SPACE_EXHAUSTED, id)` (ADR-0003 § 5); `raise_ctx(ctx, code, subject, detail)` is the same for `restore` and `post_restore`; `detail` is **caller-owned and is never written to** (the function copies it before adding `subject`, so a shared or const-derived Dictionary is safe). TR-lsg-024's `GraphFault.raise(code, subject)` gains the scope or context argument (`gdd_sync`). It never uses GDScript `assert`, which is stripped from release builds (`SIM-11`). A script error (an out-of-bounds packed-array access) is **not** a Graph fault: it raises no code and, for a write, aborts the function (§ 4.1 rule 6). "Debug also halts" (LS-24) is the kernel's `FAULTED` rule in a dev profile (ADR-0003 § 9), not Graph code. A structural violation is checked after every topology mutator in debug and tests, in `post_restore` in every build, and every run in debug.

**8.3 Numeric soft violations.** A value above its column maximum or below zero where it cannot be, a conservation drift or a clamped `dT` (LS-24): in release the writer clamps, increments `GraphDiag.saturations[field]` and continues; with `strict_numeric` (injected into `GraphModule.setup()` by the composition root: true in tests and `STRICT_DEBUG`) it raises `GraphFaults.SATURATION` instead. Every Graph test suite asserts `saturations` is all zeros. The two builds differ only after a defect, so the build-mode equivalence of ADR-0003 holds on a clean stream. A legal clamp (the stock reconcile, § 7.3) is not a saturation.

**8.4 Boot codes (domain 2, `ContentFaults`).** ADR-0007 § 2.3 makes every content boot error a domain-2 name and lets owner-named codes follow as data stories land; this ADR lists the Graph's: `GRAPH_BOOT_TYPE_ID` (exists), `GRAPH_BOOT_MODE_ID` (largest mode id above 31), `GRAPH_BOOT_POWER_CYCLE`, `GRAPH_BOOT_STABILITY` (D.12 gas and thermal sums for every template compartment), `GRAPH_BOOT_PRODUCT_OVERFLOW` (D.13), `GRAPH_BOOT_TUNING` (the relations of Section G: `HP_FAIL < LEAK_HP <= HP_OK`, `10 x Q_MIN >= max(LEAK_HP, BREACH_HP)`, band monotonicity, `ID_MAX <= 2^24` (the 24-bit id field of the RNG subject, ADR-0012 § 8.2) with `MAX_GRAPH_NODES` and `MAX_GRAPH_EDGES` below `ID_MAX`, `GRAPH_STEP_TICKS` equal to the registered divisor, and so on) and the new `GRAPH_BOOT_COLUMN_RANGE(table, column, def_id)` (a catalog value copied into a narrow column exceeds the column's documented maximum). All are BOOT tier (ADR-0007 § 2.4: running with them violated is unsafe or non-deterministic) and are evaluated over post-cast integers by pure functions in `src/shared/graph/`. The content lint of TR-lsg-033 is CI tier (it judges authored quality): `GRAPH_LINT_CAP_NEAR` (node or edge count within 10 percent of the caps), `GRAPH_LINT_FRAGILE` (a compartment D.12 sum above 500,000 ppm), `GRAPH_LINT_EQUILIBRIUM` (a heated compartment above 60 C), `GRAPH_LINT_FROZEN_PIPE` (a liquid pipe authored below `FREEZE_ON_MC` without a forced FREEZE), in `tools/ci/content_lint.gd`.

### 9. The read surface and the descriptor store

**9.1 `GraphView`.** `read_view(ctx)` is called once at the registry freeze and puts the one long-lived `GraphView` that `GraphModule.setup` constructed (`src/sim/graph/views/graph_view.gd`, getter-only) that holds references to the single `GraphState`, `GraphCaches`, the frozen `GraphTuning` and the read-only `ClockView`; `empty_view(ctx)` puts a `GraphView` built over an empty state whose getters return neutral values (`is_empty()` true). The view reads members each call (§ 6.2). It is **pure** in the sense that matters: **no getter writes any `GraphState` column, marks a dirty key or draws RNG, so `state_hash` is unchanged** (LS-19, and this keeps `validate`'s hash-unchanged guarantee). The one permitted side effect is the defect-only `ensure_derived()` guard (ADR-0012 § 4): if a writer forgot its rebuild and `topo_dirty` is set, the guard rebuilds `GraphCaches` and increments `GraphDiag.late_rebuilds`, which every test asserts is zero; in a correct build every structural mutator and `post_restore` have already finished by rebuilding, so the guard is a no-op. It returns **true values only**: there is no perceived variant, and the Perception Filter is the only non-diagnostic caller of `true_value()` (lint, TR-lsg-042 is ADR-0024's).

**Hot-path surface** (typed scalar getters; each allocates nothing; the typed signatures and `## @hot` tags for the HOT-01 list are in the `graph_view.gd` skeleton of Key Interfaces): `step_no()`, `last_step_no()` (the last step whose failure hook completed; it trails `step_no()` by up to one step, § 2.3), `node_live(id)`, `edge_live(id)`, `compartment_live(cid)`, `compartment_count()`, `cid_at(row)`, `compartments_in_section(section_id, out) -> int` (ascending cid), `compartment_p(cid)`, `compartment_p_o2(cid)`, `compartment_o2_pm(cid)`, `compartment_t(cid)`, `compartment_band_o2(cid)`, `compartment_band_p(cid)`, `compartment_band_t(cid)`, `compartment_trend(cid)`, `node_state(id)`, `node_kind(id)`, `load_subtype(id)`, `node_hp(id)`, `node_hp_cap(id)`, `node_forced_mode(id)`, `served_pm(id)` (reads the persisted `n_served`), `run_pm(id)`, `stock_pm(id)`, `node_req(id)`, `switch_pos(id)`, `edge_state(id)`, `edge_hp(id)`, `edge_flow(id)`, `opening_area(edge_id)`, `opening_area_max(edge_id)`, `section_of(id)`, `section_dormant(section_id)`, `section_rate_pm(section_id)`, `stress_active(section_id)`, `draw_grant_requested(crew)`, `draw_grant_granted(crew)`, `true_value(kind, id)`, and the descriptor-row getters of § 9.4. Queries returning a list take a caller-owned buffer and return the count, never a `Packed*Array` (registry `packed_array_exposed_or_setter_tracked`): `alarm_sections(out: PackedInt32Array) -> int` (ascending, duplicate-free `section_id`s; TR-lsg-015's return form becomes an out buffer, `gdd_sync`), `nodes_in(cid, out)`, `neighbors(cid, out)`, `open_ends(node_id, out)`, `hull_segments_of(cid, out)`, `failures(filter, out: FailureSlice) -> void`. The Dictionary-shaped returns of LS-19 (`compartment(cid)`, `node(id)`, `edge(id)`, `loop_component(...)`, `power_use(...)`, `summary()`) exist as **cold-path** methods for tests, the Debug Overlay and `SHARED_SUMMARY`, built per call, and are lint-excluded from `HOT-01` by living in `views/graph_view_cold.gd`. Accessors follow ADR-0010's naming rule (no `get`, `set`, `free`, `call`); the names above do not collide with native `Object` methods.

**Liveness and sentinels (CT-09).** A non-live id (negative, unknown, at or above its counter, or a tombstone) is not a fault. `node_live(id)` and `edge_live(id)` (O(1), allocation-free) are false for it, and **a caller that must decide liveness uses them**. Every other getter returns the documented sentinel below for a non-live id (and for a `cid` with no `comps` row or a descriptor row outside `[0, desc_count)`), never a value inside its domain that means something else; in particular the natural neutral 0 of `node_state` is `OK`, the opposite of not-live, so a tombstoned or never-allocated lamp must not read as healthy.

| Getter group | Sentinel for a non-live subject |
|---|---|
| `node_live`, `edge_live`, `compartment_live`, `section_dormant`, `stress_active` | false |
| `node_state`, `edge_state` | `GraphIds.STATE_NONE` (-1, outside the `NODE_STATE` domain 0..5) |
| `node_kind`, `section_of`, `load_subtype`, `switch_pos`, `cid_at` (row out of range) | -1 |
| `compartment_p`, `compartment_p_o2`, `compartment_o2_pm`, `compartment_band_*`, `compartment_trend` | -1 (the domains are non-negative) |
| `compartment_t` | `GraphLimits.TEMP_NONE` (a real temperature may be -1 mC) |
| `served_pm`, `run_pm`, `stock_pm`, `node_hp`, `node_hp_cap`, `node_forced_mode`, `node_req`, `edge_hp`, `edge_flow`, `opening_area`, `opening_area_max`, `section_rate_pm`, `draw_grant_*`, `true_value` | 0 |
| `desc_*(i)` | -1 |

A test drives every hot getter with id -1, an id at or above its counter and a tombstone, and asserts the documented sentinel (Validation).

**9.2 `true_value` kinds and `load_pm` (TR-instr-027).** `GraphIds.TRUE_VALUE` holds the kinds `P`, `P_O2`, `T` (compartment, `id` = cid), `STOCK_PM` (node), `SUPPLY`, `DEMAND` (POWER component, `id` = any node of it), `HP` (diagnostics) and the new **`LOAD_PM`** (POWER component). The Instruments GDD requires the load reading to be Graph-published and never derived by Instruments, and LS-19 said it was Instruments' own reading; this ADR sides with the Instruments requirement (a Graph-published value is falsifiable by one witness, and a second formula elsewhere is a second source of truth): `load_pm` is `0` when `supply` is `0`, else `mini(1250, IntMath.floor_div(1000 * demand, supply))`, with `supply` and `demand` as defined in LS-19 (`supply = S + D`, `demand = sum R_k`, inrush included). The three cases the GDD wants separate are readable without a flag byte: dead (`supply == 0`), idle (`demand == 0`), overload (`load_pm > 1000`); `power_component_case(id)` returns them as `GraphIds.LOAD_CASE`. The quantum `LOAD_QUANTUM_PERMILLE` (20) belongs to Instruments' needle and the dirty key; **it is not in `design/registry/entities.yaml`** and is cited here by name only (Open Question 6). `inst_engine_bus_load` stays unbuildable as a lie target until the GDD grants it, as TR-instr-027 says; that is Instruments' data, not a Graph switch. `gdd_sync`: LS-19's "Instruments' `load` is its own reading" is replaced by "published as `true_value(LOAD_PM)`".

**9.3 The descriptor store (TR-lsg-019).** `GraphCaches` holds a preallocated, parallel-column store with one row per live `FailureDescriptor` (the 17 fields of LS-14 plus `section_id`), rebuilt once per run in phase 11 and in `post_restore`:

| Column | Type | Column | Type |
|---|---|---|---|
| `key` | i64 | `intensity_pm` | i32 |
| `subject_kind` | u8 | `since_steps`, `mode_steps`, `grow_steps` | i32 |
| `subject_id`, `compartment_id`, `section_id` | i32 | `warn_elapsed` | i32 |
| `anchor_a`, `anchor_b` | i32 | `alarm_eligible`, `cause` | u8 |
| `loop`, `mode`, `medium`, `sev`, `acoustic`, `dir` | u8 | `count` | scalar (rows in use) |

- **Order, without a sort.** `key = (subject_kind << 40) | (subject_id << 8) | mode` (below 2^62 because `subject_id < ID_MAX`; ADR-0005 § 1.6 packs a sort key and the stable id into one int rather than `sort_custom`). Phase 11 visits subjects in the fixed order NODE slots, EDGE slots, COMPARTMENT rows, SEAM rows, OPENING edges, each ascending, and each subject's modes ascending, so **the store is built in strictly ascending key order** and "sorted `FailureDescriptor[]`" is a property of construction, **given two preconditions pinned here**: **(a)** the `GraphIds.SUBJECT_KIND` ordinals equal the visit order, NODE 1 < EDGE 2 < COMPARTMENT 3 < SEAM 4 < OPENING 5 (§ 4.7, frozen by `ids.lock.json`); **(b)** an OPENING-kind edge is visited **only in the OPENING pass** and is skipped by the EDGE pass, so no key can be produced twice. A debug check asserts strict ascent; `descriptors_of(subject_kind, subject_id, out: FailureSlice)` finds a subject's run with a `bsearch`-style loop on `key`.
- **Anchor.** For a NODE subject `anchor_a` is the node id and `anchor_b` the port local index; for an EDGE subject `anchor_a` is the edge id and `anchor_b` is -1; for a COMPARTMENT or SEAM subject both are the subject id and -1; for an OPENING subject `anchor_a` is the edge id and `anchor_b` is -1.
- **Capacity.** Rows are bounded by `GraphLimits.DESC_PER_SUBJECT_MAX` (3: the dominant mode, a FREEZE or HOT pre-stage, a PATCH_WEEP/WORN overlay) per node and edge, 5 atmospheric modes per compartment, one per seam and one AIRFLOW/SEAL_STRAIN per OPENING. `GraphCaches.size_descriptors()` sizes the columns from the table row counts in every structural mutator and in `post_restore`; steady state never resizes. Exceeding capacity in phase 11 is `GraphFaults.DESC_STORE_OVERFLOW` (structural: it cannot happen if the per-subject bound holds, and a unit test drives every subject kind through every mode). This is capacity, not Failure & Trail's `N_DESC_DESIGN_MAX`, which is its own design maximum of live descriptors (default 64) used to size its pools.
- The Graph never suppresses a descriptor: a live consequence-channel descriptor with `sev >= 1` is always a row.

**9.4 `FailureFilter` and `FailureSlice`** (shared with ADR-0012, § 11): the filter is a caller-owned, reusable parameter object; the slice is a caller-owned `PackedInt32Array` of **store row indices plus a count** (a filtered result is not a contiguous range, so "slice" is an index list). `failures(filter, out)` scans `count` rows once, appends matching row indices ascending to `out.idx`, sets `out.count` and allocates nothing. A reader gets column values by row through one typed getter per store column (`desc_count()`, `desc_key(i)`, `desc_subject_kind(i)`, `desc_subject_id(i)`, `desc_compartment_id(i)`, `desc_section_id(i)`, `desc_anchor_a(i)`, `desc_anchor_b(i)`, `desc_loop(i)`, `desc_mode(i)`, `desc_medium(i)`, `desc_sev(i)`, `desc_acoustic(i)`, `desc_dir(i)`, `desc_intensity_pm(i)`, `desc_since_steps(i)`, `desc_mode_steps(i)`, `desc_grow_steps(i)`, `desc_warn_elapsed(i)`, `desc_alarm_eligible(i)`, `desc_cause(i)`; the full list with tags is in the `graph_view.gd` skeleton), and a subject's run through `descriptors_of`. Because the store holds loaded sections only (ADR-0012 § 10), `FailureFilter.loaded_only` is retained for type stability and is always satisfied. Failure & Trail polls once per Graph step (comparing `step_no()` with its cache, FT-15), so the call is off the per-frame path; the whole presentation read side targets 0.1 ms P99 per frame (TR-lsg-025, ADR-0012).

### 9.5 Layout port (`GraphLayoutAdapter`, CT-01, CT-03)

ADR-0013 § 8 owns `LayoutGraphPort` (`@abstract`, `src/sim/layout/ports/layout_graph_port.gd`) and the four caller-owned records it fills (`SectionBases`, `OpeningRead`, `SeamOpenings`, `SectionCounts`, `src/sim/layout/views/`). This ADR is the implementer. The facade cannot be `GraphModule` (single inheritance) or `GraphView` (which must stay getter-only); it is a dedicated adapter, **`GraphLayoutAdapter extends LayoutGraphPort`** in `src/sim/graph/graph_layout_adapter.gd`, outside `ports/` because ADR-0010 ARCH-05 requires every file under `ports/` to be `@abstract`. `GraphModule.setup` constructs the single long-lived `GraphView` (the `read_view(ctx)` of § 9.1 puts that same instance at the registry freeze) and builds the adapter from `GraphMutators` and that view, and exposes it as `graph.layout_port()`; the composition root passes it to `LayoutModule.bind_graph(port)` after `GraphModule.setup` and before the registry freeze (a port call before `bind_graph` is an asserted defect on the Layout side). The setup order is therefore Layout (which exposes `layout.room_source()`), then Graph (which receives it), then `bind_graph`. The adapter holds `GraphMutators` and `GraphView`, and only `GraphModule` and Layout hold the adapter, so ownership stays one-way (§ 6.3 Method C); `LayoutRoomSource` must in turn hold Layout's state and derived objects and not `LayoutModule`, or the mutual ports would close a reference cycle (the leak test covers the pair).

| Method | Behaviour |
|---|---|
| `check_instantiate_section(views, section_id: int, template_id: String, kind: int) -> StringName` | the pure Class A check of M1 (`&""` = OK), the same function ADR-0012 calls `check_instantiate_section`: the LS-18 reasons plus `CAPACITY` (node, edge and `ID_MAX` slots, tombstones included) and `ONSET_UNKEYED` (`graph_rules.onset_valid`); Layout's `can_build` calls it |
| `instantiate_section(scope, section_id, template_id, kind, out: SectionBases) -> int` | M1 (§ 4.4): a status, the bases in `out` |
| `opening_into(edge_id: int, out: OpeningRead) -> bool` | false for a non-live or non-OPENING edge; else fills `area`, `area_max` and `door_state`, **derived from `area`** (LS-19) and never stored |
| `seam_openings_into(seam_id: int, out: SeamOpenings) -> bool` | false for an unknown seam; else `count` and `opening_ids` in `socket_plan.openings` order (the order `weld_join` appended them, § 4.1 (3)); **valid after restore**, because the lists are persisted (§ 3.6), and for a CUT record |
| `section_counts_into(section_id: int, out: SectionCounts) -> bool` | false for a section id the Graph has never held; else the counts of the rule below; reads **persisted columns only (Tier S)**, never a cache |

All five are allocation-free and write only into the caller-owned `out`. The three reads (`opening_into`, `seam_openings_into`, `section_counts_into`) are valid in APPLY, INIT, HOOK and `post_restore`; `check_instantiate_section` and `instantiate_section` are valid in APPLY and INIT (the INIT pass is stamped at the N2 clock, ADR-0012 § 5).

**`section_counts_into` and RAGGED stubs (CT-03).** It fills `nodes`, `edges` and `comps` with the **template-range** counts: the node rows in `[node_base, node_base + template.node_count)`, the matching edge range M1 created for this section, and the compartments `comp_base + k`. It fills `stub_count` with the STUB rows a RAGGED `cut_detach` created (§ 3.6): a stub's `section_of` is the cut section, and stubs, appended with fresh high ids, lie **outside** the template range, as do seam edges. It sets `tombstoned` true when every template-range row of the section is tombstoned (a discarded section), and fills `next_section_id` from `hdr`. Layout's `post_restore` check (d) therefore compares template-range rows only (ADR-0013 § 10); stubs resolve through `GraphView.section_of(id)` plus the seam's `stub_ids`, and `anchor_geometry(stub)` answers the cut socket's hatch span (ADR-0013 § 3, § 11). Without this definition every save made after a RAGGED cut would fail that check.

### 10. Dirty keys

ADR-0004 § 3 fixes the encoding `(KIND << 32) | id` and the per-block declaration; this ADR fixes the Graph's vocabulary (TR-lsg-020 residual). `KeySpaceDescriptor` for `graph.state`: `kind_ids [1..9]`, `quiet_writes_allowed = true` (a quantised family writes without a key whenever no bucket is crossed, so the missed-mark hash audit does not apply, ADR-0004 § 7; the guarantee is the per-mutator key test of Validation), no events.

| Kind | Id | Family | Key id | Fired by |
|---|---|---|---|---|
| COMP | 1 | `C/<cid>` | cid | one per compartment per run when the `T`, `P` **or** `p_o2` bucket changed |
| BAND | 2 | `B/<cid>` | cid | a band transition |
| NODE | 3 | `N/<id>` | node id | a node `state`, forced-mode or severity transition; an `hp` or store bucket change |
| EDGE | 4 | `N/<id>` (edge) | edge id | the same for an edge |
| DESC | 5 | `F/<subject_ref>` | `(subject_kind << 28) \| subject_id` | a descriptor intensity bucket change; **one key per subject**, not per descriptor (over-marking is legal and a subject has at most `DESC_PER_SUBJECT_MAX` rows) |
| LOOP | 6 | `L/<loop>/<anchor>` | `(loop << 24) \| anchor` | a loop-component summary quantum crossed |
| SEAM | 7 | `S/<seam_id>` | seam id | weld, cut, seam state change |
| TOPO | 8 | `T` | 0 | any structural mutator |
| SECTION | 9 | `SEC/<id>` | section id | structural change of a section |

The GDD lists eight family letters for nine kinds; `N/<id>` is the family of both NODE and EDGE (distinct kinds keep node and edge ids unambiguous) and `F/<id>` is DESC (Open Question 7 asks the Failure & Trail GDD to read it that way). **Packing:** loop in the top 8 bits and the component anchor (the lowest node id of the component) in the low 24, range-checked against `ID_MAX` at allocation; ADR-0004's "loop 16 and anchor 16" is its stated example, and 16 bits would be exhausted by id churn once VS-A re-creates sections with new ids. **Quanta** (`T` 1000 mC, `P` 10 dkPa, `p_o2` 5 dkPa, store 20 permille, intensity 100 permille, `hp` 250 bp) are `GraphTuning` constants. A bucket is `IntMath.floor_div(value, quantum)`, so negative temperatures bucket correctly (-1 and 0 differ). **Last-emitted buckets** are DERIVED columns in `GraphCaches` (one per compartment for each of `T`, `P`, `p_o2`; one per node for `hp` and `stock`; one per subject for intensity; one per loop component; updated at every mark, whoever marked it, so phases and `graph_mutators.gd` write them under the matrix of § 12.1), seeded in `post_restore` so a restore yields one `ALL` and then silence (ADR-0004 § 6). The design bound is the D.10 cascade of 38 keys at MVP, far under the floor of `MAX_DIRTY_KEYS` 128 the Graph asks the kernel to keep; a derelict instantiation legitimately collapses to `ALL` and the Projector rebuilds. Structural events use `T` and `SEC/<id>` only. Marks go through `scope.mark_dirty(&"graph.state", DirtyKey.make(kind, id))` inside the mutator or phase that changed the data; no setter marks anything (4.7 skips setters on element writes, probe (d)).

### 11. Published value types (shared with ADR-0012 and ADR-0021)

All are plain `RefCounted` classes with typed fields and preallocated packed buffers, owned by the caller and reused, so a weld or a failures query allocates nothing (ADR-0003 § 5: the status is the return, data goes into `out`). They live in `src/sim/graph/views/` (except the Layout-port records, which ADR-0013 § 11 owns in `src/sim/layout/views/`), are named by Assembly and Layout (a published surface), and carry no behaviour beyond `reset()` and appenders.

- **`JoinPlan`** (input fields are fixed by this ADR; `can_join` semantics, the closed reason set and the algorithm are ADR-0012's, and ADR-0012 may add *output* fields but not change input ones): `section_a`, `section_b`, `q`, `l_seam` (i32); `pair_count` and parallel `pair_node_a`, `pair_port_a`, `pair_node_b`, `pair_port_b` (i32/u8) and `pair_loop` (u8) of capacity `GraphLimits.JOIN_PAIRS_MAX`; `opening_count` and parallel `open_comp_a`, `open_comp_b`, `open_area_max` of capacity `JOIN_OPENINGS_MAX`. Output block, written by `can_join` into the caller's own plan and never into the Graph: `res_reason` (`StringName`, `&""` = OK), `res_edge_count`, `res_pair_slot_a`/`res_pair_slot_b`, `res_surge_dp`, `res_surge_warn`. `JOIN_PAIRS_MAX` and `JOIN_OPENINGS_MAX` are placeholder capacities (16 and 4) to be set from the largest socket in the section templates and asserted against them at boot (Open Question 5). `weld_join` re-resolves internally, so `res_*` is never trusted as apply input. The plan Layout hands over is canonicalised survivor-first (ADR-0013 § 6, § 7), and `weld_join` appends its rows in that plan's order (§ 4.1 (3)).
- **`WeldResult`**: `seam_id` (i32), `opening_count` (i32), `opening_ids` (`PackedInt32Array`, capacity `JOIN_OPENINGS_MAX`), the LS-15 shape `{seam_id, opening_ids[]}` as a caller-owned out object.
- **`SectionBases`** (ADR-0013's record, `src/sim/layout/views/`): `node_base`, `edge_base`, `comp_base`, M1's out (§ 4.4); `InstantiateResult` is deleted, not aliased. The other Layout-port records (`OpeningRead`, `SeamOpenings`, `SectionCounts`) are ADR-0013's too and are filled by § 9.5. **`SalvageRecord`**: `def_id` (`String`), `hp`, `state`, `hp_cap`, `patched_at_step` (M6's out, § 5; one string per salvage, never on a per-tick path).
- **`FailureFilter`**: `subject_kind_mask` (bitmask of `GraphIds.SUBJECT_KIND`), `section_id` (-1 any), `compartment_id` (-1 any), `min_sev`, `mode_mask` (bits 1..27), `medium_mask`, `consequence_only` (`medium != NONE`), `alarm_only`, `loaded_only` (exclude descriptors of dormant sections). Defaults select everything. **`FailureSlice`**: `idx` (`PackedInt32Array` sized to the store capacity), `count`; `reset()` sets `count` 0.
- **`RoomSource`** (`@abstract`, `src/sim/graph/ports/room_source.gd`): `room_volume_tiles(room_ref: int) -> int` (-1 when Layout has no such room) and `room_exposed_tiles(room_ref: int) -> int` (also -1 for an unknown room). Layout's `LayoutRoomSource extends RoomSource` adapter (ADR-0013 § 11, `src/sim/layout/`: `room_volume_tiles` wraps `V`, `room_exposed_tiles` wraps `exposed_tiles`) implements it; `layout.room_source()` returns it and the composition root passes it to `GraphModule.setup`. **Override hazards (4.7.2, review E6):** every override writes the explicit `-> int` and returns on all paths (an override that omits `-> int` and returns on one path fails with `Not all code paths return a value`, and under ADR-0010's warning configuration also with `has no static return type`); an override that misses the second abstract method fails with `Class ... must implement RoomSource.room_exposed_tiles() and other inherited abstract methods`. Test fakes are **hand-written `extends RoomSource` classes in `tests/support/`** (ADR-0010 § 3.5), not gdUnit4 `mock()`, which is unverified for an `@abstract` class. The same rule binds the overrides of `GraphLayoutAdapter`.

### 12. Writer files and owned constants

**12.1 The writer matrix (TR-lsg-002, CT-07).** The persisted columns are underscore-prefixed members of `GraphState` (`_n_hp`, `_e_hp`, `_c_n_o2`, ...; prefix per table: `_n_`, `_p_`, `_e_`, `_c_`, `_m_` seams, `_x_` sections, `_q_` queues, `_d_` defs, `_h_` header). Writers are named per class and, where a member needs more than the class default, per member. Explicit per-member writer files were chosen over routing every cache write through `GraphDerived`, because the last-emitted bucket update happens on every mark in hot code and an extra call per mark is the wrong place to spend budget.

| Class | Members | Writer files (`src/sim/graph/`) |
|---|---|---|
| `GraphState` | every member | `propagate/**`, `graph_mutators.gd`, `graph_restore.gd` |
| `GraphCaches` | every member | `graph_derived.gd` |
| `GraphCaches` | the intensive values (`P`, `p_o2`, `o2_pm`, trend), the descriptor store and `alarm_sections` | also `propagate/**` (ADR-0012 phases 0, 7 and 11) |
| `GraphCaches` | the loop summaries (`loop_sum`) | also `propagate/**` (ADR-0012 phases 2 to 5) |
| `GraphCaches` | the last-emitted buckets | also `propagate/**` and `graph_mutators.gd` (updated at every mark, whoever marked it) |
| `GraphScratch` | contents (accumulators, island, candidate, sort and degree buffers) | `propagate/**`; also `graph_derived.gd` for the two traversal buffers `stack` and `queue` that `ensure_derived` uses (ADR-0012 § 4) |
| `GraphScratch` | sizing (`resize`, ADR-0012's `size_scratch()`) | `graph_derived.gd` |
| `GraphDiag` | every member | any Graph file |

`propagate/**` is ADR-0012's directory (the dispatcher `graph_propagate.gd` and one file per phase). **Reader files** are `graph_snapshot.gd`, `views/graph_view.gd`, `views/graph_view_cold.gd`, `graph_layout_adapter.gd` and `graph_invariants.gd`; reads of a prefixed member through a foreign receiver elsewhere are findings. **ADR-0010 has no rule that expresses this, and ARCH-06 (registry `foreign_private_member_access`) forbids the very `state._n_hp[i] = v` access the writers need.** Foundation ask 1: a config-driven rule, `OWN-01`, reading `tools/lint/config/field_writers.json` (`{class, member_prefix, members, writer_files, reader_files}`, where `writer_files` entries are files or directory globs and `members` carries the per-member entries), which (a) exempts the listed writer and reader files from ARCH-06 for that class's prefix, (b) flags any assignment, compound assignment or mutating call (`resize`, `fill`, `set`, `append`, `push_back`, `insert`, `remove_at`, `sort`, `reverse`, `append_array`) on a member with that prefix in any other file, and (c) flags a `static var` in the four class files (§ 6.3 Method C). This absorbs ADR-0012's Foundation Ask 1. It is a lexical rule at file granularity (ADR-0010 accepts that field-writer rules are weaker than an AST would make them, OQ-J); the reflection test of § 6.3 and the per-mutator key test are the runtime backstop. ADR-0012's Key Interfaces comment cites this section instead of restating the set.

**12.2 Owned constants (TR-lsg-043).** This ADR is the rule owner; ADR-0010 supplies the mechanism through the existing `READ-03` (forbidden symbol in directory, token list committed per owner). Rules: **(1)** every Graph tunable lives in `assets/data/graph/` and is read as a lowercase typed accessor of the frozen `GraphTuning` (`tuning.decomp_p`), whose values equal the registry's `LOCKED` rows (`TUNING_SOURCE_DRIFT`, ADR-0007 § 2.2); **(2)** the kernel-owned `MAX_DIRTY_KEYS` and `TICK_RATE` are never read by the Graph at all except through injection (`tick_rate` reaches `GraphTuning`'s freeze through `ContentLoader`, which derives `W_MIN_STEPS` from it; `MAX_DIRTY_KEYS` is the kernel's boot check against its 128 floor); **(3)** `tools/lint/config/forbidden_symbols/graph.json` lists the upper-case names `MAX_DIRTY_KEYS`, `TICK_RATE`, `HYPOXIA_PP`, `DECOMP_P`, `COLD_DMG_ONSET_MC`, `HEAT_DMG_ONSET_MC` (and grows with every registry constant the Graph or Suit consumes) as symbols absent from `src/sim/graph/` and `src/sim/suit/`; the upper-case form appears only in data, the registry and the loader. Code-side limits that are properties of column widths (`ID_MAX`, `STEP_STAMP_MAX`, `DESC_PER_SUBJECT_MAX`, `JOIN_*_MAX`) are `GraphLimits` constants, not registry constants, and must stay consistent with them (`GRAPH_BOOT_TUNING` asserts `MAX_GRAPH_NODES` and `MAX_GRAPH_EDGES` below `ID_MAX`).

**12.3 Adopted from ADR-0012.** Members and constants this ADR carries because ADR-0012's design needs them: `GraphIds.ALARM_CAPABLE_MASK`; `GraphTuning` fields `hook_offset_power/atmo/failure`, `load_queue_max`, `w_min_steps`, the `burst_*` fields and the `rate_pm` inputs (`derelict_fault_compression`), with the RNG id-field relation checked by `GRAPH_BOOT_TUNING`; `GraphCaches` members `active_nodes`, `active_edges`, `active_comps`, `active_seams`, `store_slots`, `machine_slots`, `exchange_links`, the POWER island list, `loop_sum`, `alarm_sections`, the `GraphScratch` member `deg` (accessor `deg_max_of(e)`) and the sizing functions `size_derived()`, `size_scratch()`, `size_descriptors()` (bodies in `graph_derived.gd`); `GraphDiag` counters `late_rebuilds`, `queue_dropped`, `footprints_clamped` (and `schedule_shift`, § 7.4); the `ONSET_UNKEYED` reason of the M1 check (§ 9.5); and `FailureFilter.loaded_only`, retained for type stability and always satisfied (§ 9.4). `derived_members.json` lists every non-persisted member named here.

### Architecture Diagram

```
        composition root (src/game)                     assets/data/graph/  (type tables, graph_tuning.json, ids.lock.json)
        holds GraphModule strongly                                   |
              | setup(facet, tuning, catalog, rooms, strict_numeric, work)   v
              v                                         ContentLoader stage 1 -> GraphTuning (frozen), product_max_assertion,
        GraphModule ------------ registers ----------->           GRAPH_BOOT_* (domain 2)         src/shared/graph/: GraphIds, GraphSchema,
          |  block graph.state (SIM, WORLD, restore_after layout.state)                              GraphLimits, graph_rules, graph_products
          |  3 hooks (ADR-0012), key space, fault domain 3
          v
   +--------------------------------------------- src/sim/graph/ -----------------------------------------------+
   | GraphState (PERSISTED, SoA)        GraphCaches (DERIVED)          GraphScratch        GraphDiag             |
   |  hdr + nodes ports edges comps      slot maps, adjacency, P/p_o2,    accumulators,       saturations ...      |
   |  seams sections queues defs         descriptor store, alarm  island/sort bufs   (never hashed)       |
   |        ^   written only by          last-emitted buckets             ^ propagate/** only                          |
   |        |   graph_mutators / propagate / restore                      |                                         |
   |  graph_snapshot (read-only) --> payload {hdr, nodes, ports, ...}  graph_derived writes caches                  |
   |  GraphView (pure, typed getters, failures(filter, out)) <-- read_view facade;  GraphLayoutAdapter <-- layout.bind_graph                                   |
   +----------------------------------------------------------------------------------------------------------------+
          | snapshot payload (String keys, duplicated columns)      | scope.mark_dirty(kind, id)      | ctx.fault / scope.fault
          v                                                          v                                 v
   kernel: CanonicalEncoder -> H64 (touch-serial cache)      ChangeBatch -> presenters         domain 3 codes, FAULTED
   Save/Load: var_to_bytes (type walk) / restore(data) ---> GraphRestore (validate in order, then swap) -> post_restore
```

### Key Interfaces

Each `class_name` is its own file. All snippets follow ADR-0010 typing (no `:=`, explicit types); they are UNVERIFIED against the ADR-0010 § 5.1 warning gate until LINT-SPIKE-1 loads them (that load-check caveat stands). The `GraphSchema`, `GraphModule`, `GraphLayoutAdapter` and `GraphView` blocks are **signatures only (bodies elided)**; the remaining blocks are intended to load as written.

```gdscript
# src/shared/graph/graph_limits.gd  -- code-side width limits, not registry constants
class_name GraphLimits extends RefCounted
const ID_MAX: int = 1 << 24                 # node, edge, seam, section ids; keeps L/ and F/ key ids inside 32 bits
const STEP_STAMP_MAX: int = 1 << 30         # every int32 step stamp and last_step_* stays below this
const DESC_PER_SUBJECT_MAX: int = 3
const JOIN_PAIRS_MAX: int = 16              # PLACEHOLDER, Open Question 5
const JOIN_OPENINGS_MAX: int = 4            # PLACEHOLDER, Open Question 5
const TEMP_NONE: int = -(1 << 30)           # compartment_t sentinel for a non-live cid (a real temperature may be -1 mC)

# src/shared/graph/graph_schema.gd  -- table and column specs; the single source for snapshot, restore and the reflection test
# SIGNATURES ONLY (bodies elided; not loadable as written)
class_name GraphSchema extends RefCounted
const VERSION: int = 1
const T_NODES: String = "nodes"             # String constants: payload keys are never StringName
const T_PORTS: String = "ports"
const T_EDGES: String = "edges"
const T_COMPS: String = "comps"
const T_SEAMS: String = "seams"
const T_SECTIONS: String = "sections"
const T_QUEUES: String = "queues"
const T_DEFS: String = "defs"
const K_HDR: String = "hdr"
enum ElemType { I64, I32, U8 }
static func table_names() -> Array[String]                 # sorted
static func column_names(table: String) -> Array[String]   # sorted
static func elem_type(table: String, column: String) -> ElemType
static func min_value(table: String, column: String) -> int
static func max_value(table: String, column: String) -> int   # 2^63-1 where unbounded
static func enum_domain(table: String, column: String) -> int # GraphIds domain id, or 0
static func fingerprint() -> int                            # H64 over sorted "table.column:type" lines
static func is_power_consumer(kind: int) -> bool           # canonical-zero predicate of n_served and n_draw

# src/sim/graph/graph_state.gd  -- PERSISTED columns only (excerpt; one member per column of 3.2 to 3.8)
class_name GraphState extends RefCounted
const NF_TOMBSTONE: int = 1                 # node/edge flags byte, bit 0
const NF_HAS_WARN: int = 2
const NF_EXOGENOUS: int = 4
const NF_BREAKER_CLOSED: int = 8            # CABLE_JUNCTION
const NF_FATIGUED: int = 16
const NF_EXPOSED: int = 32
const NF_STUB: int = 64
const EF_OPEN: int = 8                      # edge flags byte, bit 3: VALVE open (edge bits otherwise as 3.4)
var _h_next_node_id: int = 1                # hdr
var _h_last_step_no: int = 0                # last step whose failure hook completed (2.3)
var _h_last_step_failure: int = 0
var _h_power_use_w0: int = 0                # w0..w4: the five-step window (hook product, phase 2 only)
var _h_sched_div: int = 0                   # sched_*: schedule in force when the cursors were written (7.2 step 13)
var _n_id: PackedInt32Array = PackedInt32Array()
var _n_kind: PackedByteArray = PackedByteArray()
var _n_flags: PackedByteArray = PackedByteArray()
var _n_hp: PackedInt32Array = PackedInt32Array()
var _n_stock: PackedInt64Array = PackedInt64Array()
var _n_served: PackedInt32Array = PackedInt32Array()   # hook products (3.2), written by phase 2 only
var _n_draw: PackedInt32Array = PackedInt32Array()
var _c_n_o2: PackedInt64Array = PackedInt64Array()
var _c_n_inert: PackedInt64Array = PackedInt64Array()
# no `static var` in GraphState, GraphCaches, GraphScratch or GraphDiag (6.3 Method C); header values are individual _h_* ints, the snapshot builds the hdr Dictionary
# holders: GraphState -> holder only; the holders never reference GraphState (6.3 Method C)
var _caches: GraphCaches
var _scratch: GraphScratch
var _diag: GraphDiag
# ... remaining columns follow 3.2 to 3.8; no setters, no getters returning a Packed*Array

# src/sim/graph/graph_faults.gd
class_name GraphFaults extends RefCounted
const DOMAIN_ID: int = 3
const NOT_IN_SCOPE: int = (DOMAIN_ID << 16) | 1
const RESTORE_BAD_VERSION: int = (DOMAIN_ID << 16) | 2
const RESTORE_UNKNOWN_ROOM: int = (DOMAIN_ID << 16) | 13
const ID_SPACE_EXHAUSTED: int = (DOMAIN_ID << 16) | 21
const HOOK_CURSOR: int = (DOMAIN_ID << 16) | 25            # runtime guard, raised by ADR-0012 1
const RESTORE_HOOK_CURSOR: int = (DOMAIN_ID << 16) | 26    # 7.2 step 13
const NAMES: Array[StringName] = [&"", &"NOT_IN_SCOPE", &"RESTORE_BAD_VERSION", &"RESTORE_BAD_TAG", &"RESTORE_BAD_LENGTH",
	&"RESTORE_BAD_ENUM", &"RESTORE_OUT_OF_RANGE", &"RESTORE_ID_ORDER", &"RESTORE_QUEUE", &"RESTORE_COUNTER",
	&"RESTORE_BAD_EDGE_ENDPOINT", &"RESTORE_SEAM_MISMATCH", &"RESTORE_UNKNOWN_DEF", &"RESTORE_UNKNOWN_ROOM",
	&"RESTORE_INVARIANT", &"STRUCT_ENDPOINT", &"STRUCT_LENGTH", &"STRUCT_ID_ORDER", &"STRUCT_COUNTER", &"STRUCT_CSR",
	&"STRUCT_SEAM_RECORD", &"ID_SPACE_EXHAUSTED", &"DESC_STORE_OVERFLOW", &"STEP_RANGE", &"SATURATION", &"HOOK_CURSOR", &"RESTORE_HOOK_CURSOR"]
	# (the full constant list follows § 8.1; a unit test checks constants and NAMES agree, ADR-0003 § 6)

# src/sim/graph/graph_fault.gd  -- statics only
class_name GraphFault extends RefCounted
static func raise_in(scope: WriteScope, code: int, subject: int) -> int:        # returns -code
	scope.fault(code, {"subject": subject})
	return -code
static func raise_ctx(ctx: CallContext, code: int, subject: int, detail: Dictionary) -> void:
	var d: Dictionary = detail.duplicate()                  # detail is caller-owned and is never written to (8.2)
	d["subject"] = subject
	ctx.fault(code, d)

# src/sim/graph/graph_module.gd  -- SIGNATURES ONLY (bodies elided)
class_name GraphModule extends RefCounted
const BLOCK_ID: StringName = &"graph.state"
# ADR-0012's setup: registers the block, the three hooks (offsets from tuning), the graph.fail stream (facet.rng.register_stream(&"graph.fail", false))
# and constructs the GraphView and the GraphLayoutAdapter (9.5); `rooms` is layout.room_source(); the ClockView source of step_no() is facet.clock (ADR-0003 10)
func setup(facet: ModuleFacet, tuning: GraphTuning, catalog: ItemCatalog, rooms: RoomSource, strict_numeric: bool, work: WorkCounter) -> void
func layout_port() -> LayoutGraphPort                       # the GraphLayoutAdapter, passed to LayoutModule.bind_graph by the composition root
func snapshot(ctx: SnapshotContext) -> CompletionToken      # builds the payload, ctx.put, return ctx.complete()
func restore(data: Dictionary, stored_version: int, ctx: SnapshotContext) -> CompletionToken
func post_restore(ctx: SnapshotContext) -> CompletionToken
func default_state(ctx: SnapshotContext) -> CompletionToken
func read_view(ctx: SnapshotContext) -> CompletionToken
func empty_view(ctx: SnapshotContext) -> CompletionToken

# src/sim/graph/graph_mutators.gd  -- Class A shape (excerpt); every mutator begins with open_for
# GraphMutators holds _state, _caches, _scratch, _rooms and _catalog, plus the read-only _clock (the ClockView) and _tuning (the frozen GraphTuning) that the step-stamping mutators such as M19 read for step_no (one-way ownership, 6.3 Method C); M8 changes connectivity, so it rebuilds and marks TOPO
func set_valve(scope: WriteScope, edge_id: int, open: bool) -> int:
	if not scope.open_for(GraphModule.BLOCK_ID):
		return MutatorStatus.REFUSED
	var slot: int = _caches.slot_of_edge_id(edge_id)
	if slot < 0:
		return MutatorStatus.OK                              # stale id: Class A check already rejected it; apply is infallible (4.1 rule 5)
	# 4.1 rule 6: every slot and index is validated above, before the first column write
	_state._e_flags[slot] = (_state._e_flags[slot] & ~GraphState.EF_OPEN) | (GraphState.EF_OPEN if open else 0)
	GraphDerived.mark_topo_dirty(_caches)                    # the cache writer sets topo_dirty (12.1)
	GraphDerived.ensure_derived(_state, _caches, _scratch, _rooms, _catalog)
	scope.mark_dirty(GraphModule.BLOCK_ID, DirtyKey.make(GraphKeys.EDGE, edge_id))
	scope.mark_dirty(GraphModule.BLOCK_ID, DirtyKey.make(GraphKeys.TOPO, 0))
	return MutatorStatus.OK

# src/sim/graph/ports/room_source.gd
# Overrides write the explicit `-> int` and return on every path; test fakes are hand-written `extends RoomSource` classes (11)
@abstract class_name RoomSource extends RefCounted
@abstract func room_volume_tiles(room_ref: int) -> int
@abstract func room_exposed_tiles(room_ref: int) -> int

# src/sim/graph/graph_layout_adapter.gd  -- concrete, NOT under ports/ (ARCH-05); SIGNATURES ONLY (bodies elided)
class_name GraphLayoutAdapter extends LayoutGraphPort
var _mutators: GraphMutators                 # holds GraphMutators and GraphView, never GraphState; nothing but GraphModule and Layout holds the adapter
var _view: GraphView
func check_instantiate_section(views: SimViews, section_id: int, template_id: String, kind: int) -> StringName   # pure Class A; &"" = OK; CAPACITY, ONSET_UNKEYED, LS-18
func instantiate_section(scope: WriteScope, section_id: int, template_id: String, kind: int, out: SectionBases) -> int   # M1
func opening_into(edge_id: int, out: OpeningRead) -> bool                  # door_state derived from area
func seam_openings_into(seam_id: int, out: SeamOpenings) -> bool           # socket_plan.openings order; valid after restore
func section_counts_into(section_id: int, out: SectionCounts) -> bool      # template-range counts + stub_count + tombstoned (9.5)

# src/sim/graph/views/graph_view.gd  -- SIGNATURES ONLY (bodies elided); getter-only; allocates nothing; sentinels are the table of 9.1
# "## @hot" marks a member of the HOT-01 list (ADR-0010 6.6)
class_name GraphView extends RefCounted
## @hot  floor_div(clock.tick(), tuning.graph_step_ticks); a pure function of T
func step_no() -> int
## @hot  last step whose failure hook (phase 11) completed (LS-8, ADR-0012 1); trails step_no() by up to one step
func last_step_no() -> int
## @hot  false for a negative, unknown or tombstoned id
func node_live(id: int) -> bool
## @hot  false for a negative, unknown or tombstoned id
func edge_live(id: int) -> bool
## @hot  false for a cid with no comps row
func compartment_live(cid: int) -> bool
## @hot
func compartment_count() -> int
## @hot  -1 outside [0, count)
func cid_at(row: int) -> int
## @hot  fills ascending cids of the section's compartments; returns the count
func compartments_in_section(section_id: int, out: PackedInt32Array) -> int
## @hot  -1 for a non-live cid
func compartment_p(cid: int) -> int
## @hot  -1 for a non-live cid
func compartment_p_o2(cid: int) -> int
## @hot  -1 for a non-live cid
func compartment_o2_pm(cid: int) -> int
## @hot  GraphLimits.TEMP_NONE for a non-live cid
func compartment_t(cid: int) -> int
## @hot  -1 for a non-live cid
func compartment_band_o2(cid: int) -> int
## @hot  -1 for a non-live cid
func compartment_band_p(cid: int) -> int
## @hot  -1 for a non-live cid
func compartment_band_t(cid: int) -> int
## @hot  -1 for a non-live cid
func compartment_trend(cid: int) -> int
## @hot  GraphIds.STATE_NONE (-1) for a non-live id
func node_state(id: int) -> int
## @hot  -1 for a non-live id
func node_kind(id: int) -> int
## @hot  LOAD subtype (GraphIds.LOAD_SUBTYPE) for a fixture filter; -1 for a non-live id or a non-LOAD node
func load_subtype(id: int) -> int
## @hot  0 for a non-live id
func node_hp(id: int) -> int
## @hot  0 for a non-live id
func node_hp_cap(id: int) -> int
## @hot  0 for a non-live id
func node_forced_mode(id: int) -> int
## @hot  reads the persisted n_served; 0 for a non-live id
func served_pm(id: int) -> int
## @hot  0 for a non-live id
func run_pm(id: int) -> int
## @hot  min(1000, ...); 0 for a non-live id
func stock_pm(id: int) -> int
## @hot  0 for a non-live id
func node_req(id: int) -> int
## @hot  -1 for a non-live id
func switch_pos(id: int) -> int
## @hot  GraphIds.STATE_NONE (-1) for a non-live id
func edge_state(id: int) -> int
## @hot  0 for a non-live id
func edge_hp(id: int) -> int
## @hot  0 for a non-live id
func edge_flow(id: int) -> int
## @hot  0 for a non-live or non-OPENING edge
func opening_area(edge_id: int) -> int
## @hot  0 for a non-live or non-OPENING edge
func opening_area_max(edge_id: int) -> int
## @hot  -1 for a non-live id
func section_of(id: int) -> int
## @hot  false for an unknown section
func section_dormant(section_id: int) -> bool
## @hot  0 for an unknown section
func section_rate_pm(section_id: int) -> int
## @hot  false for an unknown section
func stress_active(section_id: int) -> bool
## @hot  0 for an unknown crew slot
func draw_grant_requested(crew: int) -> int
## @hot  0 for an unknown crew slot
func draw_grant_granted(crew: int) -> int
## @hot  GraphIds.TRUE_VALUE kinds; 0 for a non-live id
func true_value(kind: int, id: int) -> int
## @hot  GraphIds.LOAD_CASE for a POWER component
func power_component_case(id: int) -> int
## @hot  ascending, duplicate-free section ids; returns the count
func alarm_sections(out: PackedInt32Array) -> int
## @hot
func nodes_in(cid: int, out: PackedInt32Array) -> int
## @hot
func neighbors(cid: int, out: PackedInt32Array) -> int
## @hot
func open_ends(node_id: int, out: PackedInt32Array) -> int
## @hot
func hull_segments_of(cid: int, out: PackedInt32Array) -> int
## @hot  fills out.idx with ascending store row indices and out.count; allocates nothing
func failures(filter: FailureFilter, out: FailureSlice) -> void
## @hot  the run of rows of one subject
func descriptors_of(subject_kind: int, subject_id: int, out: FailureSlice) -> void
## @hot  rows in use
func desc_count() -> int
## @hot  desc_*(i): -1 for i outside [0, desc_count)
func desc_key(i: int) -> int
## @hot
func desc_subject_kind(i: int) -> int
## @hot
func desc_subject_id(i: int) -> int
## @hot
func desc_compartment_id(i: int) -> int
## @hot
func desc_section_id(i: int) -> int
## @hot
func desc_anchor_a(i: int) -> int
## @hot
func desc_anchor_b(i: int) -> int
## @hot
func desc_loop(i: int) -> int
## @hot
func desc_mode(i: int) -> int
## @hot
func desc_medium(i: int) -> int
## @hot
func desc_sev(i: int) -> int
## @hot
func desc_acoustic(i: int) -> int
## @hot
func desc_dir(i: int) -> int
## @hot
func desc_intensity_pm(i: int) -> int
## @hot
func desc_since_steps(i: int) -> int
## @hot
func desc_mode_steps(i: int) -> int
## @hot
func desc_grow_steps(i: int) -> int
## @hot
func desc_warn_elapsed(i: int) -> int
## @hot
func desc_alarm_eligible(i: int) -> int
## @hot
func desc_cause(i: int) -> int
```

### Implementation Guidelines

- **Must** hold the persisted state in `GraphState` as `PackedInt64Array`/`PackedInt32Array`/`PackedByteArray` members and individual `_h_*` `int` members for the header (the snapshot builds the Dictionary), one array per column of § 3, every column of a table the same length; **must never** hold a `Node`, a `Resource`, a per-node object, a `Dictionary` of rows or a float in `GraphState`.
- **Must** put derived data only in `GraphCaches`, per-phase buffers only in `GraphScratch`, counters only in `GraphDiag`; **must never** add a persisted member without adding it to `GraphSchema` and bumping `schema_version` the same commit; **must never** add a cache without an entry in `derived_members.json`.
- **Must** write `GraphState`, `GraphCaches` and `GraphScratch` members only in the files of the writer matrix of § 12.1 (`GraphState`: `propagate/**`, `graph_mutators.gd`, `graph_restore.gd`; `GraphCaches`: `graph_derived.gd`, plus `propagate/**` for the intensive values, descriptor store and `alarm_sections`, plus `propagate/**` and `graph_mutators.gd` for the last-emitted buckets; `GraphScratch`: `propagate/**` for contents, `graph_derived.gd` for sizing) (OWN-01 once ADR-0010 accepts it); **must never** expose a getter or accessor that returns a `Packed*Array` of a column, or declare a setter on one.
- **Must** begin every mutator with `scope.open_for(GraphModule.BLOCK_ID)`, return `MutatorStatus.REFUSED` when it is false, mark dirty keys with `scope.mark_dirty` inside the mutator, and return `GraphFault.raise_in(...)` (a negative status) for a defect; **must never** use `assert`, a setter or a signal for any of this.
- **Must** allocate ids only through the four counters with the `ID_MAX` and table-cap tests, append rows in ascending id order, tombstone rather than remove, and zero inapplicable columns (canonical zeros); **must never** reuse, renumber, decrement or compact in the MVP.
- **Must** validate every slot and index before the first column write of a mutator or phase: an out-of-bounds packed-array access is a `SCRIPT ERROR` that aborts the function with no fault and no rollback in release; CI treats any `SCRIPT ERROR` log line as a failure (ADR-0010 `load_all` gate) and the fuzz corpus runs with the log scanned.
- **Must** resolve ids through the derived `slot_of_*` maps and treat a missing or tombstoned id as a no-op or a reason; **must never** assume id equals slot.
- **Must** do arithmetic in plain `int`, divide only through `IntMath.floor_div`/`ceil_div`/`trunc_div`, take `min`/`max`/`clamp`/`abs`/`sign` through `mini`/`maxi`/`clampi`/`absi`/`signi`, and range-check before a store into an `int32` or `uint8` column; **must never** use `/`, `%`, `/=`, `%=`, a float, `Vector2i`, `sqrt`, `pow`, `sin`, or `@warning_ignore_start("integer_division")` in `src/sim/graph/`.
- **Must** express every stamp in Graph steps and convert from ticks only at an API boundary through `GRAPH_STEP_TICKS`; **must never** store a kernel tick in a column.
- **Must** put only `String`-keyed Dictionaries of duplicated columns in the payload, with the table, column and header names of `GraphSchema`; **must never** put a `StringName` key, a live column reference, a derived value or a diagnostics counter in it.
- **Must** make `restore` validate in the fixed order of § 7.2 into a scratch `GraphState`, report only through `ctx.fault(GraphFaults.RESTORE_*)`, copy the contents into the existing object only on success, and never clamp; **must never** replace the block object or leave a half-applied restore.
- **Must** run the stock reconcile at the head of phase 0 and count it in `stock_clamped`; **must never** count it as a saturation.
- **Must** keep `GraphView` pure: no write to any `GraphState` column, no dirty mark, no RNG, `state_hash` unchanged; the one permitted side effect is the defect-only `ensure_derived()` guard (ADR-0012 § 4), which writes `GraphCaches` and increments `GraphDiag.late_rebuilds` (asserted zero in every test); return a `FailureSlice` filled into a caller-owned buffer, and a documented sentinel (§ 9.1) for a non-live id; **must never** return a live container or a `Packed*Array`.
- **Must** read every Graph tunable from the frozen `GraphTuning` by lowercase accessor and the kernel's constants through injection; **must never** declare `MAX_DIRTY_KEYS`, `TICK_RATE`, `HYPOXIA_PP`, `DECOMP_P`, `COLD_DMG_ONSET_MC` or `HEAT_DMG_ONSET_MC` in `src/sim/graph/` or `src/sim/suit/`.
- **Must** keep `src/shared/graph/` free of imports from any other `src/` directory and scene-tree free; **must never** `preload` a sibling system's script.
- **Must** keep `GraphIds` constants and `assets/data/graph/ids.lock.json` in agreement and change them only by adding; **must never** renumber, reuse or delete a value; always reference them as `GraphIds.<NAME>` (the unqualified `OK` and `FAILED` are `@GlobalScope` `Error` constants).
- **Must** declare no `static var` in `GraphState`, `GraphCaches`, `GraphScratch` or `GraphDiag` (reflection does not list it and it is shared by every instance, including the restore scratch object): constants and static funcs only.
- **Must** keep ownership one-way, `GraphModule` to `GraphState` to the three holders: `GraphCaches`, `GraphScratch`, `GraphDiag` and `GraphView` hold no field of type `GraphState`, and writers receive the state as a parameter; **must never** create a `RefCounted` cycle (test: construct and free a full `GraphModule` and assert no leak warning).
- **Must** write the explicit `-> int` (or the port's return type) on every override of `RoomSource` and `LayoutGraphPort` and return on all paths; test fakes are hand-written `extends RoomSource` classes in `tests/support/` (ADR-0010 § 3.5), not gdUnit4 `mock()`; **must never** place a concrete class under `ports/` (the `GraphLayoutAdapter` lives in `src/sim/graph/`).
- **Must** persist the hook products `n_served`, `n_draw`, CABLE `e0` and `power_use_w0..w4` (written by phase 2 only) and read `served_pm` from `n_served`; **must never** read `served_pm` from a cache or re-derive `served`, `draw` or a cable's net load in `post_restore`.

## Alternatives Considered

### Alternative 1: Array of per-node objects (`GraphNode`, `GraphEdge` as `RefCounted`)
- **Description**: One small object per node and edge held in typed arrays, the natural GDScript shape.
- **Pros**: Readable field access; kind-specific subclasses; no overlay table.
- **Cons**: Up to 12,288 live objects against the 25-Object catalog assertion and the kernel's joint memory check; per-object allocation on weld and instantiate; a snapshot must walk and copy every object (the 0.5 ms budget); no `to_byte_array()` fast path; reference cycles invite leaks; the hash would depend on walking order.
- **Rejection Reason**: LS-1 and the kernel H10 forbid per-node objects, and the cost model (one `duplicate()` per column) only holds for packed arrays.

### Alternative 2: Per-id Dictionaries (`nodes[id] = {...}`)
- **Description**: A Dictionary of Dictionaries keyed by id.
- **Pros**: Sparse ids are free; trivially extensible.
- **Cons**: Iteration order is insertion order and changes after erase-and-reinsert (ADR-0005 § 1.6); every access allocates or hashes; the Save/Load walk cost is per entry; ADR-0010 forbids iterating it in `src/sim/`.
- **Rejection Reason**: Violates the determinism ordering rules and the zero-allocation steady state.

### Alternative 3: Physical units with floats or SI integers (mmol, J, mK) and an ideal-gas constant
- **Description**: Model gas and heat in SI-like units.
- **Pros**: Familiar formulas.
- **Cons**: A 2D game with tile volumes needs no gas constant; the worst product in game units is far smaller (D.13); floats break R2 cross-platform hash identity; SI integers need the two-stage forms and the tank-pressure caveat of the technical-director brief.
- **Rejection Reason**: The GDD decided it (R1); this ADR only fixes the stored widths.

### Alternative 4: Two blocks (topology; flow, failure, atmosphere)
- **Description**: What the kernel GDD's plural "Blocks" implies.
- **Pros**: Smaller per-block payload; topology hashes rarely change.
- **Cons**: A weld writes both, the hash cache invalidates both, a snapshot needs two restores with an order between them, and the id-to-slot cache spans two objects.
- **Rejection Reason**: R2 of the GDD stands (one snapshot, one restore); the wording fix "Blocks" to "Block" goes to the kernel owner.

### Alternative 5: Side tables per node kind instead of overloaded kind-payload columns
- **Description**: `reactors`, `loads`, `stores`, `reservoirs`, `hulls` tables with exact fields.
- **Pros**: No overlay table; exact names; no zero columns.
- **Cons**: Five more row-index maps in the derived cache, a node-to-kind-row indirection on every phase loop, five more tables to validate and fuzz, and the saving is about 26 bytes on each of about 105 rows (2.7 KB) of which the overlay already uses 26 per row.
- **Rejection Reason**: Not worth the indirection at this row count; the overlay is normative and accessed only by named accessors, and § 3.11 names the one side table (fluid stamps) that would pay if size ever binds.

### Alternative 6: Flat payload keys (`"nodes.hp"`) instead of nested table Dictionaries
- **Description**: One flat Dictionary of about 120 keys.
- **Pros**: One level; the first draft of the GDD's wording.
- **Cons**: Longer key text in every snapshot and hash; the restore validator must group keys by prefix.
- **Rejection Reason**: Both pass the encoder and both hash in sorted order; nested wins on size and on a table-wise validator. Neither changes the hash definition of ADR-0005.

### Alternative 7: Persist the intensive atmosphere quantities (`P`, `p_o2`) for cheaper reads
- **Description**: Store the derived values beside the extensive ones.
- **Pros**: Reads without a derive step.
- **Cons**: A second copy that can disagree with `n` and `V`; larger snapshots; the zero-and-diff check would have to special-case it; Layout's `V` changes would need a re-base of the cache.
- **Rejection Reason**: LS-1 (c) and TR-lsg-004 for the intensive atmosphere values only (`P`, `p_o2`, `o2_pm`, trend): they are pure functions of `n`, `V` and `p_prev`, the derived cache is rebuilt eagerly (at the end of phase 7, at the end of phase 0 and in `post_restore`), so the reads are already O(1) and a restore rebuilds them exactly. This rejection does **not** extend to the hook products: `served`, `draw`, a cable's net load and the power-use window are persisted for restore equivalence, because a restore between the power hook and the atmo or failure hook cannot rebuild them from post-phase state (ADR-0012 § 11, § 6.1). That is a different argument (the value is not a function of the persisted state at the restore tick) and it is accepted; the cost is 8 bytes per node row (§ 3.11).

### Alternative 8: Clamp an over-cap store in `post_restore`
- **Description**: Reconcile the store against a lowered catalog `stock_cap` when the restore finishes.
- **Pros**: The state is conformant before the first tick.
- **Cons**: `snapshot(restore(snapshot()))` is no longer byte-identical for such a save, so ADR-0006 would need a REBASED cause or an idempotence exception, and the verify hash would differ from the stored one.
- **Rejection Reason**: Phase 0 of the first run does the same clamp as a normal deterministic step and keeps the restore exact; ADR-0016 § 4.4 agrees (Open Question 2, resolved).

### Alternative 9: Keep the GDD's `@warning_ignore_start("integer_division")` per file
- **Description**: Annotate each Graph file and use bare `/`.
- **Pros**: About 12 times cheaper division on the hot path.
- **Cons**: ADR-0010 `SIM-04` bans the token and `SIM-05`/`PRJ-04` allow exactly one ignore in the repository; the engine gate covers only the binary `/` (ADR-0005 P19).
- **Rejection Reason**: Contradicts two Foundation ADRs. If S1 shows the cost matters, the sanctioned route is an ADR-0010 amendment inlining a proven non-negative form in named hot files, decided on the measurement.

## Consequences

### Positive
- The layout is one document: tables, widths, maxima, the persisted/derived split and the payload shape are readable and testable (reflection, schema fingerprint, size fixture), so the hash and the save format cannot drift unnoticed.
- Packed columns give the cheapest possible snapshot (one `duplicate()` per column, deferred copy) and a restore that is a validate loop plus member assignment.
- Splitting `GraphCaches`, `GraphScratch` and `GraphDiag` from `GraphState` makes "every persisted field is in the snapshot and no derived field is" a one-line reflection test, and makes zero-and-diff exhaustive by construction.
- Ids never change meaning: other blocks' stored references (Layout bases, Interaction subjects, Perception refs) stay valid for the life of a save.
- The restore reason precedence is total, so the 1,000-mutation fuzz classifies every input deterministically.
- The salvage/reinstall column table settles the Item DB's three open rows without the Item DB calling a mutator.

### Negative
- Overloaded kind-payload columns (`stock`, `req`, `aux_a`, `aux_b`) trade readability for density; misuse is caught only by accessor discipline and the reflection/fuzz tests.
- Persisted references are ids with a derived id-to-slot map, one extra indirection and one more cache to keep consistent.
- Seven tables with about 70 columns mean a large `GraphSchema` and a large restore validator, and any column change is a schema bump.
- No compaction: tombstones accumulate in a session; VS-A needs its own mutator and a remap.
- The writer matrix needs two ADR-0010 changes (OWN-01 with directory globs and per-member entries, and the ARCH-06 exemption) before it is enforced; until then it is convention.
- The division rule makes every flux division about 12 times a bare operator until S1 says otherwise.
- The persisted hook products cost 8 bytes per node row and tighten the snapshot headroom to about 4 percent of the 32,000-byte ceiling; the lifetime-worst fixture decides whether lever 3 (a raised ceiling) is needed (§ 3.11).
- Layout and the Graph are co-accepted (mutual ports); neither ADR can be Accepted alone.

## Risks
1. **The 32 KB snapshot ceiling is tight (about 30.5 KB with the hook products, about 4.6 percent headroom at the loaded roster; the lifetime-worst state is larger).** Mitigation: the CI size fixture and the lifetime-worst fixture; lever 1 (the fluid-stamp side table) is pre-committed, levers 2 and 3 of § 3.11 follow; TR-lsg-036's ceiling is ADR-0012's and can be re-requested with the measurement.
2. **GDScript hot-path cost (accessors, `Callable.call`, `floor_div`) may miss the spike share.** Mitigation: S1 gates ADR-0012; columns are plain packed arrays so a GDExtension rung can take the same layout (ADR-0010 § 9); the remedy for division is named in Alternative 9.
3. **Release-export behaviour of packed-array aliasing and reflection is unverified.** The probes are editor-binary. Mitigation: S0-a/S0-b on `template_release`; the reflection test is a CI test on the editor/gdUnit binary, not a runtime path.
4. **Foundation ADRs are Proposed.** If ADR-0005 changes the payload walk, ADR-0006 the nested-Dictionary rule, or ADR-0010 the writer-rule outcome, § 3.10 and § 12.1 change. Mitigation: the dependencies are listed; the shape is isolated in `GraphSchema` and `GraphSnapshot`.
5. **Sibling ADRs not yet written** (ADR-0012, 0013, 0016, 0021) bind assumptions made here (cross-ADR list). Mitigation: each assumption is stated once and a conflicting sibling reopens it by number.
6. **VS-A compaction would renumber ids.** Layout's stored `graph_ref_base`, Interaction subjects and Perception refs hold Graph ids; a compaction that re-creates sections with new ids needs a remap those blocks do not have. Mitigation: out of MVP scope; the compaction mutator ADR must carry a remap table or move bases behind an indirection.
7. **Node and edge ids overlap numerically and ADR-0012's keyed RNG uses a subject field.** *Resolved* by ADR-0012 § 8.2 (2-bit kind, 24-bit id sized to `ID_MAX`; `GRAPH_BOOT_TUNING` asserts `ID_MAX <= 2^24` and both caps below `ID_MAX`).
8. **The hull-stub representation (appended HULL_SEGMENT rows) lets a cut consume capacity.** Mitigation: `can_detach` includes `CAPACITY`; a fixture cuts a seam at full capacity.
9. **Catalog drift at restore (unknown `def_id`, lowered cap).** Mitigation: `RESTORE_UNKNOWN_DEF` is loud by design; the cap case is reconciled in phase 0; a def rename needs the ADR-0007 alias path before `SHIPPED`.
10. **A script error is silent in a release export.** An out-of-bounds packed-array write aborts the function half-way with only a log line. Mitigation: § 4.1 rule 6 (validate before the first write) and the CI `SCRIPT ERROR` log scan.
11. **Co-acceptance with ADR-0013 and a possible ownership cycle.** The mutual ports (`LayoutGraphPort`, `RoomSource`) bind the two ADRs, and Layout holding the Graph adapter while the Graph holds `LayoutRoomSource` could close a `RefCounted` cycle. Mitigation: both Depends On rows name the pair; the adapter and `LayoutRoomSource` hold state objects, not their modules; the leak test covers the pair.

## GDD Requirements Addressed

TR ids are from `docs/architecture/tr-registry.yaml`; the Planned ADR column of `requirements-traceability.md` (including the rows re-tagged 2026-10-29) is the scope.

### Decided here (full)

| GDD System | Requirement | How This ADR Addresses It |
|------------|-------------|--------------------------|
| life-support-graph.md | TR-lsg-001: exactly one SIM block `graph.state`, `persist` true, `WORLD`, `SHARED`, `SHARED_SUMMARY`, version 1, `restore_after [layout.state]`, registered in BOOTING | § 1 registration table; `default_state` § 4.6; registered last of the Item DB, Layout, Graph chain |
| life-support-graph.md | TR-lsg-002: struct-of-arrays `GraphState extends RefCounted`, packed arrays only, no Node/Resource/per-node objects, underscore fields written only by four named files, lint-enforced | § 1 (four classes), § 3 (columns), § 12.1 (writer matrix; OWN-01 with directory globs and the ARCH-06 exemption are foundation ask 1) |
| life-support-graph.md | TR-lsg-003: all stored, hashed, propagated quantities integers in named units; zero floats; designer floats converted once into immutable `GraphTuning` | § 2.1 unit table, § 2.2 widths, § 2.3 discipline; `GraphTuning` frozen by the loader (ADR-0007 § 1.5) |
| life-support-graph.md | TR-lsg-004: extensive quantities stored, intensive derived; only hysteresis memory and warn trackers persisted | § 6.1 classification table (hook products persisted, intensive atmosphere values derived); §§ 3.2, 3.4, 3.5, 3.8 columns |
| life-support-graph.md | TR-lsg-005: per-section conservation counters keep gas, heat, coolant and fuel invariants exact | § 3.7 (sixteen `int64` counters), § 6.4 evaluators and the per-step test |
| life-support-graph.md | TR-lsg-006: every product below 2^62 verified by a boot assertion; no sqrt/exp/pow/sin | § 2.3 `graph_products.gd`, `product_max_assertion`, `GRAPH_BOOT_PRODUCT_OVERFLOW` (domain 2); SIM-03/04 |
| life-support-graph.md | TR-lsg-018: `read_view` pure, true values only, never reads PERCEPTION_OUT, `state_hash` identical with and without Perception content | § 9.1 purity, § 6.5 import/token rules and the fixture |
| life-support-graph.md | TR-lsg-019: typed scalar getters or a reusable view; `failures(filter)` slices a preallocated columnar store built once in phase 11, allocating nothing | § 9.1, § 9.3 store columns and capacity, § 9.4 `FailureFilter`/`FailureSlice` |
| life-support-graph.md | TR-lsg-024: structural violations raise `GraphFault` (never `assert`) causing FAULTED; numeric violations clamp in release, count in `graph_saturations`, tests assert zeros | § 8.1 codes, § 8.2 `GraphFault.raise_in`, § 8.3 (the scope argument is a GDD sync) |
| life-support-graph.md | TR-lsg-038: persisted-field set verified by reflection against a checked-in DERIVED allowlist and by zero-and-diff of each derived cache | § 6.3 Methods A and B; `tests/fixtures/graph/derived_members.json`; probed (b) |
| life-support-graph.md | TR-lsg-043: registry/kernel-owned constants read by name, never redeclared as literals in Graph or Suit, with a lint | § 12.2 (rule owner); mechanism is ADR-0010 `READ-03` with `forbidden_symbols/graph.json` |
| item-and-module-database.md | TR-itemdb-014: `validate_instance` compares against the Graph's 10 Hz `step_no`, never the 60 Hz tick | § 2.3 `GraphView.step_no()`; every persisted stamp is in steps |
| item-and-module-database.md | TR-itemdb-016: salvage then reinstall preserves hp, hp_cap, patched_at_step; stored quantities discarded and reinstall at 0; only a spent repair part lifts a cap | § 5 column table (arguments carry hp/hp_cap/patched_at; `stock` to `disc_*`, 0 on install) |
| item-and-module-database.md | TR-itemdb-031: copy-at-install (`req_w`, `req_max`, port caps, persisted) versus read-live (`waste_permille`, `out_w_rated`, `repair_part_id`) | § 5 (copy-at-install columns; read-live through `def_ix`, derived per-slot caches only) |
| in-world-instruments-and-inspect-view.md | TR-instr-027: `load` is a Graph-published `load_pm` true-value kind with dead/idle/overload cases | § 9.2 `LOAD_PM` and `power_component_case` (`LOAD_QUANTUM_PERMILLE` is a registry gap, cited by name) |

### Decided here (partial: this ADR decides its half; the other owner is named)

| GDD System | Requirement | How This ADR Addresses It / Other Owner |
|------------|-------------|--------------------------|
| life-support-graph.md | TR-lsg-020: int dirty keys, bucket quanta, one `C/<cid>` per compartment, 38-key worst cascade, `MAX_DIRTY_KEYS` floor 128 | § 10: kinds, packing, quanta as `GraphTuning`, bucket storage, `quiet_writes_allowed`. Key encoding and the 128 floor: ADR-0004. Which phase fires which key: ADR-0012 |
| life-support-graph.md | TR-lsg-022: payload via `var_to_bytes`, JSON only in header and dump, Graph packed-array layout | § 3.10 shape (nested, `String` keys, duplicated columns). Storage form and JSON rules: ADR-0006 |
| life-support-graph.md | TR-lsg-023: two-phase restore, version 1 only, `GRAPH_RESTORE_*` reasons via `ctx.fault`, `post_restore` rebuilds caches and seeds buckets | § 7.2 reason precedence, § 7.4 `post_restore`, § 7.5 version 1, § 8.1 codes. Restore contract and order: ADR-0006, ADR-0016. The phase code that reads the seeds: ADR-0012 |
| life-support-graph.md | TR-lsg-030: type ids explicit, stable, unique, append-only, never reused; tables load first, `graph.state` registers last | § 4.7 `GraphIds` plus `ids.lock.json` (the append-only gate ADR-0007 left to this ADR). Boot unique-check and load order: ADR-0007 |
| life-support-graph.md | TR-lsg-033: boot refuses invalid content/tuning with named `GRAPH_BOOT_*`; a content lint reports caps, D.12 sums, equilibrium, frozen pipes | § 8.4: the full boot-code list, BOOT versus CI tiers, lint names. Tier machinery and `GRAPH_BOOT_TYPE_ID`: ADR-0007 |
| item-and-module-database.md | TR-itemdb-033: restore with stock above a lowered live `stock_cap` clamps, credits `discarded_*`, closes conservation, emits a diagnostic | § 7.3: hard ceilings versus live cap, phase-0 stock reconcile, `stock_clamped`. Restore ordering and any `post_restore` placement: ADR-0016 |
| item-and-module-database.md | TR-itemdb-036: design-time lints call the Graph's pure functions (`cap_seam`, D.4, D.5) with Graph-sourced numbers | § 1 `src/shared/graph/graph_rules.gd` (importable by `src/data/` and tools); `GraphTuning` is the single source of numbers. Lint framework: ADR-0007 |
| item-and-module-database.md | TR-itemdb-040: on salvage/reinstall, `throttle_pm`/`setpoint_pp` take fresh defaults, `state`/`forced_mode` re-derived from hp | § 5 column table. Mutator order and transaction: ADR-0021 |
| boarding-and-salvage.md | TR-boarding-041: cutting a store routes stock to `discarded_*`; an uncapped stub leaks from the home pool only while the seam valve is open | § 5 (M6 stock to `disc_*`), § 3.3 (port cap retained, OPEN_END/CAPPED flags), § 3.6 (stub rows). Leak physics through the valve: ADR-0012 |
| 2d-lighting-and-occlusion.md | TR-light-034: ambient from `mean_fixture_pm` over Graph-served fixtures only; self-powered fixtures excluded | § 9.1 `served_pm(id)`, `load_subtype(id)` (lamp LOADs), `compartments_in_section(section_id, out)` (the per-section enumeration) and `nodes_in(cid, out)`; self-powered fixtures are not Graph LOAD nodes, so they are absent from the read. The mean and slew: ADR-0028 |

### Owned elsewhere (loaded or checked here, decided by another ADR)

| GDD System | Requirement | Where decided / what this ADR supplies |
|------------|-------------|--------------------------|
| life-support-graph.md | TR-lsg-007: ascending-slot iteration over dense append-only arrays, one Jacobi pass | ADR-0012. Supplies: dense append-only tables (§ 3.1, § 4.1), the `GraphScratch` accumulator inventory (§ 6.1, § 6.2) |
| life-support-graph.md | TR-lsg-010: one idempotent topology refresh; eager `ensure_derived` | ADR-0012. Supplies: the cache class, `topo_dirty`, the derived-cache list (§ 4.2, § 6.1) |
| life-support-graph.md | TR-lsg-012: persisted thaw window | ADR-0012. Supplies: `thaw_until` columns on nodes and edges (§ 3.2, § 3.4) |
| life-support-graph.md | TR-lsg-016, TR-lsg-017: 19 mutators, Class A/B, queued M14/M15/M17 | ADR-0012, ADR-0021. Supplies: the out-object types (§ 11), the `queues` storage and bounds (§ 3.8) |
| life-support-graph.md | TR-lsg-021: `snapshot()` is a Dictionary of named packed arrays encoded by the kernel | ADR-0005 § 3.1. Supplies: the concrete payload shape (§ 3.10) |
| life-support-graph.md | TR-lsg-025: hook spike share P99 0.5 ms, zero heap allocations per phase | ADR-0012, ADR-0010. Supplies: preallocated scratch and descriptor store, allocation-free getters (§ 6, § 9) |
| life-support-graph.md | TR-lsg-026: work per run bounded; node/edge capacity | ADR-0012. Supplies: slot-counted capacity and the `ID_SPACE_EXHAUSTED`/`CAPACITY` tie (§ 4.1) |
| life-support-graph.md | TR-lsg-027, TR-lsg-028: welds only append; `cut_detach` atomic along seams; `discard_section` tombstones | ADR-0012, ADR-0021. Supplies: the identity rules, tombstones and stub rows (§ 4.1, § 3.6) |
| life-support-graph.md | TR-lsg-029: dormant sections skipped, stamps advanced on load | ADR-0012, ADR-0026. Supplies: `dormant_since` and DORMANT flag columns, the six stamp columns (§ 3.2, § 3.7) |
| life-support-graph.md | TR-lsg-031: bit-identical goldens across platforms | ADR-0005. Supplies: canonical zeros (§ 3.9) and a fixed schema so the goldens are stable |
| life-support-graph.md | TR-lsg-034: exactly one P5 batch per structural mutator; no notification on a fault | ADR-0003, ADR-0021. Supplies: the `T`/`SEC` vocabulary (§ 10) |
| life-support-graph.md | TR-lsg-036: snapshot 0.5 ms and 32 KB, restore plus `post_restore` 2 ms | ADR-0012. Supplies: bytes per row, the 29.7 KB fixture (about 30.5 KB with the hook products), the lifetime-worst fixture and the pre-committed lever 1 (§ 3.11) |
| life-support-graph.md | TR-lsg-039: diagnostics counters never in the canonical layout or any hash | ADR-0005 § 3.6. Supplies: `GraphDiag` as its own class outside the payload (§ 1, § 6.1) |
| ship-section-and-layout-data-model.md | `graph_ref_base`, M1 instantiate, the opening/seam/section reads and the `RoomSource` seam | ADR-0013 § 3, § 8, § 11 owns `LayoutGraphPort`, its four records and `LayoutRoomSource`. Supplies: `GraphLayoutAdapter` (§ 9.5), contiguous M1 ranges (§ 4.4), `section_counts_into` with stubs (CT-03), the `RoomSource` definition (§ 11) |

## Performance Implications
- **CPU (all UNVERIFIED on a release export; S1/S3 pending)**: a typed getter is one method call plus an element read; the presentation read side per frame is a handful of getters and, once per Graph step, one `failures()` scan over `count` rows. Phase loops iterate ascending slots over local column aliases (probe: a read-modify-write loop over 4096 elements costs 58.7 microseconds in the debug editor, about 14 ns per element). Division through `IntMath.floor_div` is about 12 times a bare operator (ADR-0005 P14, editor); S1 decides whether that is inlined in named hot files. `snapshot` builds about 70 `duplicate()` entries and a few nested Dictionaries; `duplicate()` is deferred copy-on-write (0.11 microseconds per 4096-element column) and the **first write to each column after a snapshot pays the copy** (0.71 microseconds per 32 KB column, probed), so the cost lands on the first mutators after a save, not on the snapshot. Restore validation is a linear pass over about 70 columns and 600 rows (tens of thousands of element checks, estimated 0.2 to 0.5 ms in the editor, inside ADR-0012's 2 ms with `post_restore`).
- **Memory**: persisted columns at the MVP roster are about 30.5 KB (the payload size with the hook products, § 3.11); at the capacity ceilings (`MAX_GRAPH_NODES` and `MAX_GRAPH_EDGES` slots) about 0.35 MB of node columns and 0.6 MB of edge columns plus ports; caches and scratch are of the same order, so the whole block is estimated at a few MB at capacity and well under 1 MB at MVP. ESTIMATE; the kernel's joint memory assertion (CR-17) is the check.
- **Load Time**: restore validation and cache rebuild are linear in rows; type tables and `GraphTuning` load in stage 1 of the content loader (ADR-0007).
- **Network**: none at MVP; `SHARED_SUMMARY` (VS-B, ADR-0032) reads the same columns and is at most 1 KB.

## Migration Plan
There is no Graph code yet. Implementation order: (1) `src/shared/graph/` pure files and `GraphTuning` (loader stage 1) with their boot tests; (2) `GraphState`/`GraphCaches`/`GraphScratch`/`GraphDiag`, `GraphSchema`, `GraphSnapshot` and the reflection and schema-fingerprint tests; (3) `GraphFaults`, `GraphFault`, `GraphRestore` with the fuzz corpus; (4) M1 and `ensure_derived`; (5) the rest of the mutators and the view. The foundation asks and GDD syncs below land before the first story that depends on them. No existing code or save format is migrated.

## Validation Criteria
- **Schema**: `GraphSchema` columns equal the payload keys of a populated state; the committed fingerprint changes only with `schema_version`; every column has a min, max, type and canonical-zero predicate.
- **Reflection, zero-and-diff and ownership** (§ 6.3) pass on `GraphState`, `GraphCaches`, `GraphScratch` and `GraphDiag`: `derived_members.json` is exact (with the ADR-0012 members, without `served_pm`), no `static var` in the four files (`^static var` scan), no field of type `GraphState` in the holders or `GraphView`, and constructing and freeing a full `GraphModule` with its `GraphLayoutAdapter` and a `LayoutRoomSource` leaves no leak warning.
- **Round trip**: `snapshot(restore(snapshot()))` is canonical-byte identical, tombstones included, at ticks T..T+5 and mid-fault; `block_hash` twice with no `open_for` hits the cache.
- **Restore fuzz**: 1,000 seeded mutations classified V/I/B; every I yields exactly the reason of its row in § 7.2 (a mutated hook cursor yields `RESTORE_HOOK_CURSOR`, step 13); the live block is byte-identical before and after each failure; a store above a lowered live cap restores exactly and is reconciled in the first phase 0 with `stock_clamped` 1 and `saturations` all zero; a schedule change across a save counts one `schedule_shift` and faults nothing; the RE-1 restore-equivalence of ADR-0012 § 11 holds at every tick of the 12-tick window (hook products persisted).
- **Size**: the F-MVP fixture (105 nodes, 150 edges) snapshots to at most 32,000 bytes (about 30.5 KB expected with the hook products; the defs assumption of Engine Compatibility (i) stated with the figure); the **lifetime-worst fixture** (every socket welded and cut RAGGED once, the ADR-0013 § 6 bound; queues at their bounds; maximal `defs`) is measured against 32,000 bytes after lever 1 (§ 3.11) and decides whether a raised ceiling is requested; the per-table bytes match § 3.
- **Conservation**: all four totals equal their instantiation baseline after every step of every fixture; `saturations` all zeros.
- **Ids**: counters never decrease across weld, cut, discard, restore; no id is reused (property test over random mutator sequences); `ids.lock.json` agrees with `GraphIds`; a stale id no-ops/rejects for every mutator; every hot `GraphView` getter answers its documented sentinel for id -1, an id at or above its counter and a tombstone (§ 9.1).
- **No script errors**: no `SCRIPT ERROR` line in the log across the restore fuzz and the stale-id mutator tests (§ 4.1 rule 6).
- **Hook products**: `n_served`, `n_draw` and CABLE `e0` hold their range specs and canonical zeros on non-consumers and tombstones, only phase 2 writes them, and `post_restore` leaves them untouched.
- **Layout port**: `GraphLayoutAdapter` answers the five methods of § 9.5 against fixtures: `seam_openings_into` returns `opening_ids` in `socket_plan.openings` order before and after a restore; `opening_into` derives `door_state` from `area`; a RAGGED cut, save and restore passes Layout's verifier, with `section_counts_into` reporting template-range counts, the right `stub_count` and `tombstoned` for a discarded section.
- **Overflow**: a tuning with `GAS_SCALE` raised refuses at boot naming the operand pair; a fixture at every field maximum matches an arbitrary-precision oracle.
- **Purity and independence**: every `GraphView` getter leaves `state_hash` unchanged and `GraphDiag.late_rebuilds` at 0; the Perception-block fixture of LS-22 gives an identical `state_hash`.
- **Keys**: a per-mutator fixture in a fresh kernel asserts each mutator's discrete changes mark exactly the declared kinds and ids.
- **Lints** (once ADR-0010 accepts them): OWN-01, READ-03 token list for the six owned constants, SIM-03/04/05, HOT-01 on the phase functions, each with bad and good fixtures (FIX-01).
- **Engine**: probes (a) to (i) re-run on a `template_release` export; S0-a/S0-b, S1, S3 recorded.

## Related
- **Depends on (all Proposed)**: ADR-0003 (block contract, fault domains), ADR-0004 (dirty keys), ADR-0005 (determinism, encoding, `IntMath`), ADR-0006 (payload form, restore), ADR-0007 (type tables, loader, ledgers), ADR-0010 (layout, typing, lints). Indirectly ADR-0001, ADR-0002.
- **Co-owner / siblings**: ADR-0012 (propagation; shares `JoinPlan`, `WeldResult`, `FailureFilter`, `FailureSlice`), ADR-0013 (id split, Layout block; **co-accepted**: owns `LayoutGraphPort` and the Layout-port records, implements `RoomSource` through `LayoutRoomSource`), ADR-0016 (restore order; its V3 covers § 7.2 step 13 by reference, and V7 decodes with `GraphView.step_no()` derived from the kernel clock installed at R2, not from hdr), ADR-0017 (cites `node_live`/`edge_live`), ADR-0021 (weld/cut/install transaction).
- **GDDs**: `design/gdd/life-support-graph.md` (LS-1 to LS-6, LS-14 to LS-21, LS-24, D.10, D.13, Section E), `item-and-module-database.md` (IDB-18), `in-world-instruments-and-inspect-view.md` (IW-9), `failure-and-trail-system.md` (FT-15), `ship-section-and-layout-data-model.md` (`graph_ref_base`), `2d-lighting-and-occlusion.md`, `boarding-and-salvage.md`.
- **Registry**: `docs/registry/architecture.yaml` (fault_domain_table, block_persistence_payload, dirty_key_encoding, id_ledgers, foreign_private_member_access, packed_array_exposed_or_setter_tracked, integer_division_convention); `design/registry/entities.yaml` (constants cited by name).

## Open Questions
1. **Division cost.** If S1 shows `IntMath.floor_div` is material in the phases, an ADR-0010 amendment must allow an inlined non-negative form in named files; this ADR cannot grant it. Owner: ADR-0010 / S1.
2. **Where the stock reconcile runs.** *Resolved:* phase 0 of the first run, agreed with ADR-0016 § 4.4 (§ 7.3); no REBASED cause is needed.
3. **`cause` is not in LS-3's common-field list** though LS-14 returns it. *Resolved for the data model* by ADR-0012 § 7 (the EXOGENOUS clear rule); the GDD sync still names it as persisted (the EXOGENOUS flag bit).
4. **The fourth medium (energy, dJ) in `discarded_*`/`withdrawn_*`** follows from M16 on a BATTERY; the GDD should say so. Owner: GDD.
5. **Registry gaps for capacities this ADR needs**: the pending-load bound (64, TR-lsg-017), `JOIN_PAIRS_MAX`, `JOIN_OPENINGS_MAX`, `max_stock_gas`, `max_stock_liquid`. Values here are placeholders; Layout's socket templates and the Derelict kits set the join capacities. Owner: ADR-0013 and the registry.
6. **`LOAD_QUANTUM_PERMILLE` is absent from `entities.yaml`.** Owner: Instruments GDD / registry pass.
7. **The Failure & Trail and Emitter GDDs name `F/<id>`.** This ADR reads it as the DESC kind keyed per subject. Owner: those GDDs.
8. **Node/edge id overlap in the RNG subject field** (Risks 7). *Resolved* by ADR-0012 § 8.2 (2-bit kind, 24-bit id).

## Registry Candidates
Reviewable candidates for `docs/registry/architecture.yaml`; this ADR does not write the registry.

### state_ownership
```yaml
  - state: graph_state_block
    status: active           # ADR-0011 is Proposed
    owner_system: life-support-graph
    adr: docs/architecture/adr-0011-life-support-graph-data-model.md
    interface: "graph.state: GraphState (persisted SoA: hdr + nodes ports edges comps seams sections queues defs, PackedInt64/Int32/ByteArray columns, dense append-only ascending-id rows, tombstones kept, plus the persisted hook products n_served, n_draw, CABLE e0 and power_use_w0..w4 written by ADR-0012 phase 2 only) with companions GraphCaches (derived), GraphScratch, GraphDiag; reads through the pure GraphView (typed getters, failures(filter, out)); writes only through the Graph's own mutators inside a WriteScope"
    write_access: "writer matrix of ADR-0011 section 12.1: GraphState propagate/**, graph_mutators.gd, graph_restore.gd; GraphCaches graph_derived.gd, plus propagate/** for the intensive values, descriptor store and alarm_sections, plus propagate/** and graph_mutators.gd for the last-emitted buckets; GraphScratch contents propagate/** (and graph_derived.gd for the stack and queue buffers), sizing graph_derived.gd; GraphDiag any Graph file"
    referenced_by:
      - docs/architecture/adr-0011-life-support-graph-data-model.md
    added: 2026-10-29
    revised: ""
  - state: graph_entity_ids
    status: active
    owner_system: life-support-graph
    adr: docs/architecture/adr-0011-life-support-graph-data-model.md
    interface: "node, edge, seam and section ids from the persisted counters next_node_id, next_edge_id, next_seam_id, next_section_id (below GraphLimits.ID_MAX 2^24); cid is the COMPARTMENT node id; section_id 0 is the world EXTERIOR; M1 (called only by Layout LM1 through LayoutGraphPort) fills ADR-0013's SectionBases {node_base, edge_base, comp_base}; stale ids no-op and read as the section 9.1 sentinels"
    write_access: append-only, Graph allocator only
    referenced_by:
      - docs/architecture/adr-0011-life-support-graph-data-model.md
    added: 2026-10-29
    revised: ""
  - state: graph_type_ids
    status: active
    owner_system: life-support-graph
    adr: docs/architecture/adr-0011-life-support-graph-data-model.md
    interface: "src/shared/graph/graph_ids.gd constants (loops, node kinds, port types, edge kinds, modes, states, bands, seam states, boundary, section kind, load tier/subtype, trace kind, subject kind (NODE 1, EDGE 2, COMPARTMENT 3, SEAM 4, OPENING 5), cause, medium, acoustic; ALARM_CAPABLE_MASK, STATE_NONE) plus assets/data/graph/ids.lock.json in the ADR-0007 ledger format"
    write_access: append-only by reviewed code change
    referenced_by:
      - docs/architecture/adr-0011-life-support-graph-data-model.md
      - docs/architecture/adr-0007-static-content-data-pipeline-and-stable-type-ids.md
    added: 2026-10-29
    revised: ""
```

### interfaces
```yaml
  - contract: graph_read_view
    status: active           # ADR-0011 is Proposed
    pattern: direct_call
    producer: life-support-graph
    consumers:
      - failure-and-trail-system
      - emitter-state-and-light-exposure-query
      - suit-and-equipment
      - in-world-instruments-and-inspect-view
      - adaptive-audio-and-tells
      - perception-filter
      - interaction-system
      - save-load-and-versioned-format
    adr: docs/architecture/adr-0011-life-support-graph-data-model.md
    signal_signature: "GraphView (getter-only facade from read_view): step_no() = floor_div(clock.tick(), GRAPH_STEP_TICKS); typed scalar getters with documented not-live sentinels and the guards node_live/edge_live/compartment_live (compartment_p/p_o2/t/bands, node_state (STATE_NONE -1 when not live)/hp/load_subtype/served_pm (the persisted n_served)/run_pm/stock_pm, edge_state/hp, opening_area, section_of, section_dormant); last_step_no() = the last step whose failure hook completed; compartments_in_section(section_id, out: PackedInt32Array) -> int; desc_* row getters and descriptors_of; alarm_sections(out: PackedInt32Array) -> int; failures(filter: FailureFilter, out: FailureSlice) -> void (store row indices, no allocation); true_value(kind, id) with kinds P, P_O2, T, STOCK_PM, SUPPLY, DEMAND, LOAD_PM, HP; pure, true values only"
    referenced_by:
      - docs/architecture/adr-0011-life-support-graph-data-model.md
    added: 2026-10-29
    revised: ""
  - contract: graph_mutator_value_types
    status: active
    pattern: direct_call
    producer: life-support-graph
    consumers:
      - modular-ship-assembly
      - ship-section-and-layout-data-model
      - interaction-system
      - web-map-and-transit
    adr: docs/architecture/adr-0011-life-support-graph-data-model.md
    signal_signature: "caller-owned reusable RefCounted out/in objects with preallocated packed buffers: JoinPlan (inputs section_a, section_b, q, l_seam, pair_*, open_*; outputs res_*), WeldResult {seam_id, opening_count, opening_ids}, SectionBases {node_base, edge_base, comp_base} (ADR-0013's record in src/sim/layout/views/; InstantiateResult is deleted), SalvageRecord {def_id, hp, state, hp_cap, patched_at_step}, FailureFilter, FailureSlice {idx, count}; ADR-0012 owns can_join semantics"
    referenced_by:
      - docs/architecture/adr-0011-life-support-graph-data-model.md
    added: 2026-10-29
    revised: ""
  - contract: graph_room_source_port
    status: active
    pattern: direct_call
    producer: ship-section-and-layout-data-model
    consumers:
      - life-support-graph
    adr: docs/architecture/adr-0011-life-support-graph-data-model.md
    signal_signature: "@abstract RoomSource {room_volume_tiles(room_ref) -> int (-1 unknown), room_exposed_tiles(room_ref) -> int}; implemented by Layout's LayoutRoomSource (layout.room_source() is passed to GraphModule.setup); overrides write the explicit -> int and return on all paths; test fakes are hand-written extends RoomSource classes in tests/support (ADR-0010 3.5), not gdUnit4 mock(); read in post_restore and ensure_derived only"
    referenced_by:
      - docs/architecture/adr-0011-life-support-graph-data-model.md
    added: 2026-10-29
    revised: ""
  - contract: graph_layout_port_adapter
    status: active           # ADR-0011 and ADR-0013 are co-accepted (mutual ports)
    pattern: direct_call
    producer: life-support-graph
    consumers:
      - ship-section-and-layout-data-model
    adr: docs/architecture/adr-0011-life-support-graph-data-model.md
    signal_signature: "GraphLayoutAdapter extends LayoutGraphPort (src/sim/graph/graph_layout_adapter.gd, outside ports/ because ARCH-05 requires every file under ports/ to be @abstract), exposed as graph.layout_port() and passed to LayoutModule.bind_graph by the composition root: check_instantiate_section(views, section_id, template_id, kind) -> StringName; instantiate_section(scope, section_id, template_id, kind, out: SectionBases) -> int (M1; its only caller is Layout LM1); opening_into(edge_id, out: OpeningRead) -> bool (door_state derived from area); seam_openings_into(seam_id, out: SeamOpenings) -> bool (socket_plan.openings order, valid after restore); section_counts_into(section_id, out: SectionCounts) -> bool (template-range counts, stub_count, tombstoned; persisted columns only); allocation-free, caller-owned outs"
    referenced_by:
      - docs/architecture/adr-0011-life-support-graph-data-model.md
    added: 2026-10-29
    revised: ""
```

### api_decisions
```yaml
  - purpose: graph_payload_shape
    status: active           # ADR-0011 is Proposed
    api: "snapshot payload is one String-keyed Dictionary: hdr (Dictionary of 17 ints built from the _h_* members of GraphState) plus one nested Dictionary per table (nodes, ports, edges, comps, seams, sections, queues, defs) mapping column name to a duplicate() of the packed column; any table/column/header rename or re-type is a schema_version bump with a committed GraphSchema fingerprint"
    not: "StringName payload keys; live column references; derived caches or diagnostics counters in the payload; flat dotted keys"
    adr: docs/architecture/adr-0011-life-support-graph-data-model.md
    reason: "ADR-0006 type walk rejects StringName; the hash is over sorted keys; one persisted-field check per class"
    referenced_by:
      - docs/architecture/adr-0011-life-support-graph-data-model.md
    added: 2026-10-29
    revised: ""
  - purpose: graph_unit_system
    status: active
    api: "int64 for conserved/extensive quantities and counters, int32 for ids, step stamps, bp, permille, W and short physical quantities (range-checked before store), uint8 for enums and flags; all time stamps in Graph steps (step_no = floor_div(T, GRAPH_STEP_TICKS)); every product below 2^62 by the D.13 boot assertion; division only through IntMath; typed mini/maxi/clampi/absi/signi"
    not: "floats, Vector2i, bare / or %, a file-level integer_division ignore, kernel ticks in stored stamps"
    adr: docs/architecture/adr-0011-life-support-graph-data-model.md
    reason: "Determinism, cross-platform hashes and silent 32-bit truncation (ADR-0005 P9, P17)"
    referenced_by:
      - docs/architecture/adr-0011-life-support-graph-data-model.md
    added: 2026-10-29
    revised: ""
  - purpose: graph_dirty_key_kinds
    status: active
    api: "KIND COMP 1, BAND 2, NODE 3, EDGE 4, DESC 5, LOOP 6, SEAM 7, TOPO 8, SECTION 9; L/ key id = (loop << 24) | anchor; F/ key id = (subject_kind << 28) | subject_id (one per subject); quanta from GraphTuning; last-emitted buckets derived and seeded in post_restore; quiet_writes_allowed true"
    not: "per-field or per-step continuous keys; per-descriptor F keys; String keys"
    adr: docs/architecture/adr-0011-life-support-graph-data-model.md
    reason: "Keeps the design cascade at 38 keys and the packing inside the 32-bit key id"
    referenced_by:
      - docs/architecture/adr-0011-life-support-graph-data-model.md
      - docs/architecture/adr-0004-change-notification-and-dirty-keys.md
    added: 2026-10-29
    revised: ""
  - purpose: graph_fault_domain_codes
    status: active
    api: "domain 3 local codes 1..26 append-only (NOT_IN_SCOPE, RESTORE_* x14 with first-failure-wins precedence including 26 RESTORE_HOOK_CURSOR, STRUCT_* x6, ID_SPACE_EXHAUSTED, DESC_STORE_OVERFLOW, STEP_RANGE, SATURATION, and 25 HOOK_CURSOR, the runtime guard raised by ADR-0012); GRAPH_BOOT_* are domain-2 names in ContentFaults"
    not: "reuse or renumbering; restore failure by return value or clamp"
    adr: docs/architecture/adr-0011-life-support-graph-data-model.md
    reason: "ADR-0003 section 6 delegates the owner code list to the system ADR"
    referenced_by:
      - docs/architecture/adr-0011-life-support-graph-data-model.md
    added: 2026-10-29
    revised: ""
  - purpose: graph_owned_constants_rule
    status: active
    api: "Graph tunables are lowercase accessors of the frozen GraphTuning equal to registry LOCKED rows; MAX_DIRTY_KEYS, TICK_RATE, HYPOXIA_PP, DECOMP_P, COLD_DMG_ONSET_MC, HEAT_DMG_ONSET_MC never appear as symbols in src/sim/graph or src/sim/suit (READ-03 token list forbidden_symbols/graph.json)"
    not: "redeclaring a registry constant as a literal or const in Graph or Suit"
    adr: docs/architecture/adr-0011-life-support-graph-data-model.md
    reason: "One source of truth for tunables (TR-lsg-043)"
    referenced_by:
      - docs/architecture/adr-0011-life-support-graph-data-model.md
    added: 2026-10-29
    revised: ""
  - purpose: graph_hook_products_persisted
    status: active
    api: "n_served (permille), n_draw (dJ per step), CABLE e0 (net load, W, non-negative magnitude), power_use_w0..w4 (collectively power_use_w5) and the schedule in force sched_div/sched_off_* (collectively sched) are persisted and written by ADR-0012 phase 2 only; served_pm(id) reads n_served; post_restore re-derives P, p_o2, o2_pm and trend and never served, draw or a cable's net load; hdr.last_step_no is the last step whose failure hook completed; restore step 13 checks the hook cursors against the stored sched and the clock"
    not: "deriving served, draw or cable load in post_restore; persisting the intensive atmosphere values; reading served_pm from a cache"
    adr: docs/architecture/adr-0011-life-support-graph-data-model.md
    reason: "A restore between the power hook and the atmo or failure hook cannot rebuild the hook products from post-phase state (ADR-0012 section 11)"
    referenced_by:
      - docs/architecture/adr-0011-life-support-graph-data-model.md
    added: 2026-10-29
    revised: ""
  - purpose: graph_stale_id_sentinels
    status: active
    api: "a non-live id (negative, unknown, at or above its counter, tombstoned) is not a fault: Class B mutators no-op, Class A checks return UNKNOWN/NOT_*, GraphView getters return the documented sentinel (node_state/edge_state GraphIds.STATE_NONE -1, node_kind/section_of -1, compartment_t GraphLimits.TEMP_NONE, magnitudes 0) and node_live/edge_live/compartment_live are the guards; every slot and index is validated before the first column write because an out-of-bounds packed write is a SCRIPT ERROR that aborts the function"
    not: "a blanket neutral 0 (node_state 0 is OK); indexing an array with an unchecked id"
    adr: docs/architecture/adr-0011-life-support-graph-data-model.md
    reason: "A dead or never-allocated lamp must not read as healthy; a script error gives no fault and no rollback in release"
    referenced_by:
      - docs/architecture/adr-0011-life-support-graph-data-model.md
    added: 2026-10-29
    revised: ""
```

### performance_budgets
```yaml
  - system: graph_state_snapshot_size
    status: active           # ADR-0011 is Proposed; synthetic-fixture estimate, release measurement pending (S3)
    budget_ms: 0
    adr: docs/architecture/adr-0011-life-support-graph-data-model.md
    notes: "Column set of ADR-0011 section 3 measured 29.7 KB canonical bytes (29,674) at 105 nodes, 150 edges, 270 ports, 14 compartments (probed 2026-10-29, value-independent, synthetic defs of 60 offsets and 1,500 utf8 bytes); the persisted hook products (n_served, n_draw, 8 bytes per node row, plus the power-use window) bring the loaded roster to about 30.5 KB (about 30.7 KB with sched), about 4.6 percent headroom against the 32,000-byte ceiling of TR-lsg-036 (ADR-0012); reached near 111 nodes and 159 edges; a lifetime-worst fixture (every socket welded and cut RAGGED once) is a Validation item and lever 1 (fluid-stamp side table) is pre-committed; levers in section 3.11. budget_ms is not applicable (size budget)."
    referenced_by:
      - docs/architecture/adr-0011-life-support-graph-data-model.md
    added: 2026-10-29
    revised: ""
```

### forbidden_patterns
```yaml
  - pattern: graph_field_write_outside_writer_files
    status: active           # needs ADR-0010 OWN-01 (directory globs, per-member entries) and the ARCH-06 exemption (foundation ask 1)
    description: "Assignment or a mutating call on a GraphState/GraphCaches/GraphScratch underscore-prefixed member outside the writer matrix of ADR-0011 section 12.1 (GraphState: propagate/**, graph_mutators.gd, graph_restore.gd; GraphCaches: graph_derived.gd, plus propagate/** for the intensive values, descriptor store and alarm_sections, plus propagate/** and graph_mutators.gd for the last-emitted buckets; GraphScratch: propagate/** contents, graph_derived.gd sizing and the stack/queue buffers)."
    why: "GDScript has no private fields; one writer set keeps the dirty-key and touch-serial discipline checkable."
    adr: docs/architecture/adr-0011-life-support-graph-data-model.md
    added: 2026-10-29
  - pattern: graph_id_reuse_renumber_or_compaction
    status: active
    description: "Reusing, renumbering or decrementing a Graph node, edge, seam or section id or counter, compacting tables in the MVP, or changing a persisted GraphIds value (append-only; ids.lock.json)."
    why: "Layout bases, Interaction subjects, Perception refs and saves hold these ids."
    adr: docs/architecture/adr-0011-life-support-graph-data-model.md
    added: 2026-10-29
  - pattern: graph_holder_static_var_or_state_backref
    status: active           # lexical part needs ADR-0010 OWN-01 (static var scan)
    description: "A static var in GraphState, GraphCaches, GraphScratch or GraphDiag, or a field of type GraphState in GraphCaches, GraphScratch, GraphDiag or GraphView."
    why: "Reflection does not list a static var and it is shared by every instance, including the restore scratch object; a back-reference makes a RefCounted cycle that leaks."
    adr: docs/architecture/adr-0011-life-support-graph-data-model.md
    added: 2026-10-29
  - pattern: graph_derived_value_in_payload
    status: active
    description: "Putting a derived cache, scratch buffer, diagnostics counter or StringName key in the graph.state snapshot payload, or persisting the intensive atmosphere values P, p_o2, o2_pm or trend (the hook products n_served, n_draw, CABLE e0 and power_use_w0..w4 are persisted by design)."
    why: "Hash stability, TR-lsg-004/038/039, ADR-0006 type walk."
    adr: docs/architecture/adr-0011-life-support-graph-data-model.md
    added: 2026-10-29
```

### referenced_by_update (existing entries that gain this ADR in referenced_by)
- fault_domain_table (state_ownership, ADR-0003): add docs/architecture/adr-0011-life-support-graph-data-model.md (domain 3 local code list written here)
- block_persistence_payload (interfaces, ADR-0006): add this ADR (nested String-keyed Dictionary of duplicated columns; restore precedence)
- dirty_key_encoding (api_decisions, ADR-0004): add this ADR (Graph kinds and the L/ and F/ packing)
- id_ledgers (state_ownership, ADR-0007): add this ADR (graph ids.lock.json spaces)
- foreign_private_member_access (forbidden_patterns, ADR-0010): add this ADR (writer/reader file exemption request, with directory globs and per-member entries)
- packed_array_exposed_or_setter_tracked (forbidden_patterns, ADR-0010): add this ADR (alarm_sections and failures use caller-owned buffers)
- integer_division_convention (api_decisions, ADR-0005): add this ADR (no file-level ignore in src/sim/graph)

## GDD sync
Items to apply to `design/gdd/life-support-graph.md` (and siblings) when this ADR is accepted; none is applied by this ADR:
1. LS-1 (d): delete the per-file `@warning_ignore_start("integer_division")`; divisions go through `IntMath` (ADR-0005 § 1.4, ADR-0010 SIM-05); S0-b no longer verifies the annotation.
2. LS-1 / TR-lsg-002 / LS-21: GraphState holds persisted columns only; derived caches, scratch and diagnostics are `GraphCaches`, `GraphScratch`, `GraphDiag`; the DERIVED allowlist is the member list of those classes; the writer files are the matrix of § 12.1 (the GDD's four-file wording is replaced).
3. LS-21 / Section E "never by clamping": add the carve-out that a store above a lowered live `stock_cap` restores and is reconciled in the first phase 0 (`discarded_*`, diagnostic, no saturation).
4. LS-24 / TR-lsg-024: `GraphFault.raise_in(scope, code, subject)` and `raise_ctx(ctx, ...)`; "debug halts" is the kernel's FAULTED dev rule.
5. LS-19: `alarm_sections` and the list queries return through caller-owned buffers; `true_value` gains `LOAD_PM` and "Instruments' `load` is its own reading" is replaced by the Graph-published value; `step_no()` is `floor_div(T, GRAPH_STEP_TICKS)`.
6. LS-20: composite packing is `(loop << 24) | anchor`; `F/<id>` is the DESC kind keyed per subject; `N/<id>` is shared by NODE and EDGE kinds.
7. LS-3 / LS-14: name `cause` as persisted (EXOGENOUS flag bit); LS-21: name the energy medium in `discarded_*` and `withdrawn_*`; add the `stub_ids` list of a SeamRecord and state that RAGGED hull stubs are appended HULL_SEGMENT rows; `default_state` includes the EXTERIOR row.
8. LS-18 M1: ids are node-contiguous in template order, COMPARTMENT nodes first, so `comp_base == node_base`; `section_id` 0 is reserved for EXTERIOR; M1's only caller is Layout LM1 through `LayoutGraphPort` and its out record is ADR-0013's `SectionBases`; the M1 check carries `ONSET_UNKEYED`; `section_counts_into` counts the template range and reports RAGGED stubs separately (§ 9.5).
9. LS-21 / LS-4 / TR-lsg-004: `served_pm` is persisted as the hook product `n_served`, with `n_draw`, the CABLE `e0` net load and the five-step `power_use` window (CT-04, ADR-0012 § 11); "`served_pm` is not, because phase 2 recomputes it before use" is replaced; LS-4's list of values "derived on read or in the run and never persisted" drops `served_pm`; TR-lsg-004's "intensive ones (P, p_o2, served_pm, descriptors)" becomes "(P, p_o2, o2_pm, descriptors)" and its persisted list gains the hook products.
10. LS-19: the not-live sentinel table and the `node_live`/`edge_live`/`compartment_live` guards (§ 9.1); `load_subtype` and `compartments_in_section` join the read surface; `hdr.last_step_no` is the last step whose failure hook completed (already the GDD's LS-8 wording, which this ADR now matches).

## Foundation Asks
Changes this ADR needs in Foundation ADRs; none is worked around silently. The ADR-0010 items are blocking for enforcement, not for the design.
1. **ADR-0010: a field-writer rule and an ARCH-06 exemption.** New `OWN-01`, config-driven from `tools/lint/config/field_writers.json` (`{class, member_prefix, members, writer_files (files or directory globs), reader_files}`), which exempts the listed files from ARCH-06 for that prefix, flags assignment or mutating calls on such members elsewhere (§ 12.1) and scans the four class files for `static var` (§ 6.3 Method C). Needed for TR-lsg-002; ARCH-06 as written forbids the access the writers need. This absorbs ADR-0012's Foundation Ask 1 (the `propagate/**` directory glob and the per-member entries of the writer matrix).
2. **ADR-0010: name `mini`, `maxi`, `clampi`, `absi`, `signi` as permitted under `SIM-04`** (it bans only the untyped names; probed to exist, returning `int`). Also record Open Question 1 (hot-file division inlining) as the decided route if S1 requires it.
3. **ADR-0004 § 3: relax the packing example.** It gives "loop in the high 16 bits and anchor in the low 16"; the Graph uses 8 and 24 (§ 10) because 16 bits would be exhausted by VS-A id churn. State that composite packing is owner-defined and range-checked.
4. **ADR-0007: accept a Graph `ids.lock.json` under its ledger mechanism** (§ 3.2 says the code-side enums "need no ledger"); add the `GRAPH_BOOT_COLUMN_RANGE` and `GRAPH_LINT_*` names to the `ContentFaults` list. Also ADR-0010 `READ-03`: add `tools/lint/config/forbidden_symbols/graph.json` for TR-lsg-043 (§ 12.2).
5. **ADR-0006: confirm** that the type walk accepts a nested Dictionary of packed arrays to depth 3 and that a Graph restore needs no REBASED cause because the stock reconcile runs in the first phase 0 (§ 7.3).

## Review Log
Revised 2026-10-29 after the per-ADR engine review (verdict CONCERNS, non-blocking), the TD review (verdict CONCERNS, four blocking) and the Core cross-ADR adjudication (autonomous mode, "Revise flagged items"). Status stays Proposed. Right-sizing and spike-gating (T14): the layout is right-sized for a determinism-critical solo MVP, the alternatives are genuinely considered, and the release-export gaps (S0-a/S0-b, S1, S3, LINT-SPIKE-1) are named rather than claimed; the added Graph-side surfaces (hook products, layout adapter) are required by sibling ADRs, not extras.

| Id | Disposition |
|---|---|
| T1 / CT-04 (blocking) | Adopted. `n_served`, `n_draw` (§ 3.2), CABLE `e0` net load with sign convention and range (§ 3.4), `power_use_w0..w4` (§ 3.8); § 6.1 gains a persisted hook-products row and `served_pm` leaves DERIVED; § 7.4 (3) says `post_restore` never re-derives `served` or `draw`; Alternative 7 now rejects persisting only the intensive atmosphere values and cites ADR-0012 § 11; § 3.11, Risk 1, the size registry candidate and the Validation size line updated to about 30.5 KB (4.6 percent headroom; about 4.2 with `sched`); `served_pm` removed from the `derived_members.json` description; GDD sync items 9 and the structured sync list; Open Question 2 closed (phase-0 reconcile agreed with ADR-0016 § 4.4). |
| T2 / CT-05 (blocking) | Adopted. `hdr.last_step_no` is the last step whose failure hook completed, in § 2.3, § 3.8 and the `GraphView.last_step_no()` doc comment. |
| T3 / CT-07 (blocking) | Adopted, option (i). § 1 table and § 12.1 are one writer matrix per class (`propagate/**` directory globs, per-member writers for caches, scratch sizing in `graph_derived.gd`); Foundation Ask 1, the Implementation Guidelines and the forbidden-pattern candidate match. One addition beyond the ruling: `graph_derived.gd` also writes the scratch `stack`/`queue` buffers that ADR-0012 § 4 has `ensure_derived` use. |
| T4 / CT-01 (blocking) | Adopted. New § 9.5 and the `GraphLayoutAdapter` snippet (`src/sim/graph/graph_layout_adapter.gd`, outside `ports/`), `check_instantiate_section` naming, five methods, `SectionBases` replacing `InstantiateResult` (deleted, not aliased), § 1 `init_world` row and § 4.4 call chain (Web Map to LM1 to port), `graph.layout_port()` in setup, registry candidates updated. |
| T5, CT-08 | Adopted. Codes 25 `HOOK_CURSOR` (runtime, ADR-0012) and 26 `RESTORE_HOOK_CURSOR` (§ 8.1), `hdr.sched` (§ 3.8), § 7.2 step 13, the schedule rule and `GraphDiag.schedule_shift` (§ 7.4), fuzz reason table, ADR-0016 V3 covering it by reference (Related). |
| T6, CT-09 | Adopted. `node_live`/`edge_live` (and `compartment_live`, an addition) and the sentinel table in § 9.1 (`node_state` and `edge_state` return `GraphIds.STATE_NONE` -1; `compartment_t` uses `GraphLimits.TEMP_NONE` because -1 mC is a real temperature); § 4.1 (5); the sentinel test in Validation. |
| T7 | Adopted. § 9.1 purity reworded (no `GraphState` column write, no dirty mark, no RNG; the defect-only `ensure_derived()` guard writes caches and counts `late_rebuilds`); Implementation Guidelines aligned. |
| T8 | Adopted with a decision. § 3.11 gains the lifetime-worst analysis, cites ADR-0013 § 6's one-cycle-per-socket rule, corrects the reach-point to near 111 nodes, and **pre-commits lever 1** (the fluid-stamp side table) with its tradeoff; the exact column moves land in the schema commit that introduces `fstamps` (no shared save exists yet), so §§ 3.2 and 3.4 still list the pre-lever stamps and say so. The lifetime-worst fixture is a Validation item that decides whether lever 3 is requested. |
| T9 | Adopted. `graph_view.gd` Key Interfaces skeleton with `## @hot` tags and sentinels, `load_subtype`, `compartments_in_section`, the full `desc_*` list and `descriptors_of`. |
| T10 | Adopted. `SUBJECT_KIND` ordinals NODE 1, EDGE 2, COMPARTMENT 3, SEAM 4, OPENING 5 (§ 4.7, § 9.3), OPENING edges visited only in the OPENING pass, OPENING anchor defined. |
| T11 | Adopted. (a) `_h_*` int members, snapshot builds the hdr Dictionary; (b) `set_valve` rebuilds and marks EDGE and TOPO; (c) `setup` takes ADR-0012's `work`, the `graph.fail` registration, and names `facet.clock` (ADR-0003 § 10) as the `ClockView` source; (d) hooks row says `tuning.graph_step_ticks` and `tuning.hook_offset_*`; (e) `raise_ctx` copies `detail`. |
| T12 | Adopted. Open Questions 2, 3, 8 and Risk 7 marked resolved with pointers; 2^20 changed to 2^18 in § 4.1 (4) and § 8.4 (superseded by the 24-bit id field, see the 2026-10-29 confirm residuals entry below); § 12.3 lists the members adopted from ADR-0012; `loaded_only` documented as always satisfied. |
| T13 | Adopted. (a) § 4.6 `default_state` through `ctx.put` and `GraphRestore.restore`; (b) § 7.3 over-cap window (`stock_pm` clamps to 1000, M16 may withdraw, dormant case); (c) step 11 alias note. |
| T14 | No change required. |
| CT-02 (aligned) | Port signature, record types and the `LayoutRoomSource`/`bind_graph` setup order are ADR-0013's; § 9.5, § 11 and the Depends On row align to them. |
| CT-03 | Adopted. `section_counts_into` template-range counts, `stub_count`, `tombstoned` (§ 9.5), citations in § 3.6 and § 4.1 (3), RAGGED-cut fixture in Validation. |
| CT-06 | Adopted. `GraphScratch` is the ADR-0012 § 3 accumulator inventory (§ 1, § 6.1, § 6.2 (4) superseded); no `_c_n_total_at` accessor is added; the ADR-0012 `GraphCaches` and `GraphDiag` members and their `derived_members.json` entries are listed (§ 12.3). |
| CT-13 | Noted. `weld_join` row order follows the canonical JoinPlan (§ 4.1 (3), § 11). |
| CT-36 (aligned) | No change: this ADR's assumption list already agrees (M19 under INIT and P2 applies, never from a hook). |
| CT-53 | Adopted. Depends On row: co-accepted with ADR-0013 (mutual ports). |
| CT-54 | Aligned only: bare accessor names kept, one style throughout the view; Foundation Ask 1 absorbs the `field_writers.json` globs. |
| E1 | Adopted. § 4.1 rule 6, § 8.2, Implementation Guidelines, Risk 10, probe (j), and a Validation line (no `SCRIPT ERROR` across the restore fuzz and the stale-id tests). |
| E2 | Adopted. § 6.3 Method C and the guidelines: no `static var` in the four classes, scanned by OWN-01 or the reflection test. |
| E3 | Adopted. One-way ownership rule (§ 1, § 6.3 Method C, guidelines), leak test in Validation. |
| E4 | Adopted. § 6.2 (2) reworded (member re-assignment strands an alias, `resize` does not), (4) marked superseded by ADR-0012, scratch described as accumulators and flux arrays, § 7.2 drops the scratch `GraphState` immediately after phase 2. |
| E5 | Adopted. `GraphSchema` and `GraphView` blocks labelled signatures only, `raise_ctx` copies `detail`, LINT-SPIKE-1 caveat kept. |
| E6 | Adopted. § 11 `RoomSource` override hazards and hand-written fakes (ADR-0010 § 3.5); guidelines and registry candidate. |
| E7 | Adopted. Knowledge Risk raised to HIGH with the reason. |
| E8 | Adopted. The defs assumption (60 offsets, 1,500 utf8 bytes) is stated with probe (i) and the size figure; S0-a, S0-b and S3 stay on `template_release`. |
| E9 | Adopted. § 4.7 note: always reference `GraphIds.<NAME>`; unqualified `OK` and `FAILED` are `@GlobalScope` `Error` constants. |

Rejected findings of the adjudication (no change here): aliasing `InstantiateResult` to `SectionBases` (deleted instead), placing the facade under `ports/` (ARCH-05; it lives in `src/sim/graph/`), and numbering the restore cursor check as local 25 (it is 26).
- **2026-10-29 confirm residuals:** the RNG subject id field is 24 bits sized to `ID_MAX <= 2^24` (ADR-0012 § 8.2 owns it) in § 4.1 (4), § 8.4 `GRAPH_BOOT_TUNING`, Risk 7 and Open Question 8 (the 18-bit wording and the 2^18 relation are withdrawn); `GraphScratch` gains the `deg` buffer with `deg_max_of(e)` and `GraphCaches` gains `loop_sum` (§ 1, § 6.1, § 12.1 writer matrix, § 12.3); `GraphMutators` also holds the read-only `_clock` and `_tuning`; the diagram quotes `setup(..., strict_numeric, work)`; Enables row confirmed against the hard Depends On rows of ADR-0012, 0013, 0016 and 0017 (ADR-0034's ADR-0011 row is Soft, so no row here).
- **2026-10-29 orchestrator rulings:** no R-id touches this ADR. Dependency-cycle break only: ADR-0013 moved from Depends On to a new `Co-accepted with` row (the pair's hard edge is ADR-0013 -> ADR-0011); ADR-0016 and the unwritten ADR-0021 to ADR-0028 consumers moved out of Enables into a new `Soft references (not acceptance prerequisites)` row; Enables now lists exactly the written ADRs that list this ADR under Depends On (ADR-0012, ADR-0013, ADR-0017).
