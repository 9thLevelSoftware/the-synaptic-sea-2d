# ADR-0012: Graph Propagation Scheme

## Status
Proposed

## Date
2026-10-29

## Last Verified
2026-10-29 (against `design/gdd/life-support-graph.md` LS-8 to LS-13, LS-15 to LS-18, LS-24 to LS-26, D.1, D.6, D.8 to D.12, D.15, D.16 and Section E "Hooks, order, FAULTED, budget"; ADR-0001, ADR-0003 to ADR-0007 and ADR-0010 (all Proposed); ADR-0011 and ADR-0013 (drafts, unreviewed); the architecture hook schedule; Godot 4.7.2 probes of 2026-10-29 on the editor binary, recorded in Engine Compatibility)

## Decision Makers
Claude (author), running under modes.automation: autonomous (project.yaml); decisions logged for the user's review; the user has not yet reviewed this ADR.

## Summary
The Life-Support Graph must advance five coupled loops (power, fuel, oxygen, coolant, gas and heat exchange) and the failure lifecycle deterministically, inside a spike share of the kernel's P3 budget, and must survive a snapshot taken on any tick. This ADR fixes the propagation scheme: one logical step of twelve fixed phases split across three chained step hooks at divisor `GRAPH_STEP_TICKS` (offsets 0, 2, 4), exactly one pass per loop per step with three declared write disciplines (delta accumulators, ordered allocation, own-row), an input-queue drain whose timing is a pure function of kernel order, a fairness gate that floors damage at the moment it is applied, a keyed and namespaced RNG, dormant sections that cost nothing and are re-based on load, an eager-plus-guarded topology refresh, a restore-equivalence rule for the products that cross hooks (persisted hook products: two node columns, the CABLE load and the five-step power-use window; the intensive caches stay derived and exact), and the budgets and measurement protocol for the hooks, the snapshot and the restore.

## Engine Compatibility

| Field | Value |
|-------|-------|
| **Engine** | Godot 4.7.2 (stable, mono build probed), GDScript |
| **Domain** | Core (simulation scheduling, integer propagation, deterministic randomness) |
| **Knowledge Risk** | MEDIUM. Nothing here calls a post-4.3 engine API. The two 4.7 GDScript rows in `breaking-changes.md` bind it: packed-array element writes run no setter (the Graph has no setters and marks dirty keys explicitly, ADR-0011 § 10), and an override inherits a typed return (the hook callables are registered `Callable`s, not overrides, and every branch ends in `return ctx.complete()`). The engine reference has no core or scripting module (`modules/` holds animation, audio, input, navigation, networking, physics, rendering, ui), so packed-array, integer and loop-cost claims rest on ADR-0005's probe table (P1 to P22), on ADR-0011's probes and on this ADR's own probes, and stay UNVERIFIED on a release export until S1 (SP-ENG-1) and S3 run there. |
| **References Consulted** | `docs/engine-reference/godot/VERSION.md`; `breaking-changes.md` (4.6 to 4.7 GDScript rows); `deprecated-apis.md` (untyped Array/Dictionary row); `current-best-practices.md` (GDScript 4.7 rows); ADR-0005 Engine Compatibility P6 to P9, P14, P17 (cited by number); ADR-0011 Engine Compatibility (a) to (i). No `modules/*.md` applies. |
| **Post-Cutoff APIs Used** | None. `mini`, `maxi`, `clampi`, `absi`, `signi` (probed to exist and return `int` by ADR-0011), `posmod`, `PackedInt64Array`/`PackedInt32Array`/`PackedByteArray` element reads and writes, `fill`, and `Callable.call` with a fixed argument list. |
| **Verification Required** | Probed 2026-10-29 on `Godot_v4.7.2-stable_mono_win64_console.exe --headless --script` (editor binary, not a release export; script in `C:/Users/dasbl/AppData/Local/Temp/claude/adr-probes/0012/probe.gd`): **(a)** a namespaced RNG counter differs between node 5 and edge 5 at the same step, which the GDD's un-namespaced `(step_no << 20) \| subject_id` cannot do (ADR-0011 Risk 7). The probe ran the first-draft layout `(step_no << 20) \| (kind << 18) \| id` (129,452,998,661 versus 129,453,260,805 at `step_no` 123,456, reproduced by the engine review); § 8.2 now uses the wider layout `(step_no << 26) \| (kind << 24) \| id` (T7), whose values at the same step are 8,284,991,913,989 for node 5 and 8,285,008,691,205 for edge 5 (shell arithmetic, **not re-probed**; the property probed is only that the kind bits separate the keys and that the key stays a non-negative int64); **(b)** `raw & 0x7FFFFFFFFFFFFFFF` is non-negative for the signed constant `-7046029254386353131` and for `-1` (9,223,372,036,854,775,807), and `posmod` of the masked value by 1000 returns 677, so the burst test needs no float and no `%`; **(c)** an insertion sort of packed `(key << 16) \| arrival` entries over a `PackedInt64Array`, in the shipped form (a bounded `for j: int in range(i - 1, -1, -1)` with `break`, never `while`, PROP-01), re-timed by the engine review (E5) on random keys at 3.5 microseconds for 16 entries, 27.0 for 56 and 33.3 for 64, and **55.3 microseconds for 64 reverse-sorted entries, which is the worst-case bound** (editor binary; the earlier `while` form measured 4.0, 32.8 and 42.9 on random keys and is superseded). The 64-entry point is the MVP queue bound (§ 5); the VS-A bound is larger and is a re-measure item (Risk 10); **(d)** a read-modify-write loop over 4,096 elements costs 51.6 microseconds direct and 73.4 through an `int32` active-list indirection (about 5 nanoseconds per element extra), the price of the dormant-skipping lists of § 10; **(e)** per-element delta accumulate, apply and re-zero over 4,096 elements costs 134.5 microseconds against 69.5 for `duplicate()` plus a rewrite plus a swap by reference (the dense full-column case; the Graph's Jacobi columns are the compartment rows, 14 at MVP and about 120 in VS-A, so neither dominates, and § 3 chooses accumulators for reasons of conservation and sparsity, not speed); **(f)** `clampi` inside that loop adds about 17 microseconds per 4,096 calls (about 4 nanoseconds per call). **Caveat (E3): probes (d), (e) and (f) measure function-local `Packed*Array`s.** The engine review measured, on the same binary and per 4,096-element read-modify-write, 262 microseconds (about 64 nanoseconds per element) through an object member, 55 microseconds (about 13 nanoseconds) through a local alias, about 117 nanoseconds per call for a one-line method accessor and about 123 nanoseconds per call for a static `floor_div`-like call. The 5 and 4 nanosecond figures hold only for locals and cannot be extrapolated to member access (about 5 times); every phase therefore aliases its columns in locals first (§ 2, § 3, the excerpt in Key Interfaces), and S1 decides. **Still open and named:** S1 (SP-ENG-1: a stub Graph with 12 phases on RM-DESKTOP and RM-DECK release exports, accessor and `Callable.call` overhead per ms, `IntMath.floor_div` cost in the real phases) gates acceptance of every budget in § 13; S3 (snapshot, restore and `post_restore` time and size on the F-MVP fixture); KRN-LAT-02 for whether `ClockView.tick()` inside P3 names the executing tick or the next one (§ 1 makes `step_no` insensitive to it for offsets up to `GRAPH_STEP_TICKS - 2`); S0-a/S0-b on a `template_release` export for the aliasing and integer behaviour this ADR relies on through ADR-0011. |

## ADR Dependencies

| Field | Value |
|-------|-------|
| **Depends On** | ADR-0011 (tables, columns, `GraphScratch`, `GraphCaches`, descriptor store, queues, value types, fault domain 3; Proposed), ADR-0003 (`HookDescriptor`, `after` edges with `tolerates_stale`, `WriteScope` phases, mutator status contract, hook schedule compiled from `T mod C`, `STEP_BUDGET_MS` as an alarm; Proposed), ADR-0004 (dirty-key marks, dedupe, `MAX_DIRTY_KEYS` collapse, quiet writes; Proposed), ADR-0005 (`RngFacet.draw`, plain streams, `IntMath`, integer-only contract; Proposed), ADR-0001 (P0 to P6 tick phases, the executing tick `T`; Proposed), ADR-0006 (restore contract, snapshot hitch; Proposed), ADR-0007 (`GraphTuning`, boot tiers; Proposed), ADR-0010 (`## @hot`, HOT-01, CNT-01 to CNT-04, `WorkCounter`, ladder rungs, lint rules; Proposed). None is Accepted, so this ADR cannot be Accepted before them. |
| **Soft references (not acceptance prerequisites)** | ADR-0016 (restore ordering; the hook-cursor check is ADR-0011 § 7.2 step 13, CT-08; ADR-0016 does not list this ADR under Depends On and this ADR does not list it either). **Forward references (unwritten ADRs; each will list this ADR under Depends On when authored):** ADR-0021 (weld, cut and install transaction: calls `can_join`, `weld_join`, `cut_detach` and `ensure_derived` as defined in § 4, § 6 and § 9), ADR-0022 (Attention: `alarm_sections()`, `M17` STRESS, exogenous cause), ADR-0026 (Web Map: `M19` and dormant semantics), ADR-0027 (Suit: `M15`/`M16` timing and the grant), ADR-0028 (lamp brightness from `served_pm`), ADR-0031 (Debug Overlay: per-phase work counters). |
| **Enables** | ADR-0017 (Emitter State reads `served_pm` and `failures()` and deposits `M14` on the § 5 schedule, including from INIT). |
| **Blocks** | The Graph epic's propagation stories (every phase, the fairness gate, cascades, thaw rolls, dormant re-basing), the S1 benchmark harness story, the D.15 fixture stories (TR-derelict-012) and the restore-equivalence test story. Stories citing this ADR stay Blocked while it is Proposed. |
| **Ordering Note** | Write `graph_rules.gd` additions (`gas_flux`, `gate_floor`, `rng_subject`, `onset_valid`) and their unit tests first: ADR-0007's loader and the Item DB lints call the pure rules before any kernel exists. Land the S1 stub Graph (12 empty phases, the real active lists and accumulators) before the phase bodies, because it decides which budget lever is taken. The three ADR-0010 lint additions of Foundation Asks 2 and 3 should land with the first `propagate/` file. |

## Context

### Problem Statement

ADR-0011 fixed *what the Graph stores*. It left to this ADR *how the stored state advances*: which hooks, which phases in which order, how a phase may read and write so that one pass is order-independent and conserves, when queued inputs take effect, how the fairness gate is enforced, how randomness is keyed, what a dormant section costs, when topology is rebuilt, and which budgets the whole thing must meet and how they are measured. The GDD (LS-8) specifies the twelve phases and the staggered hooks in prose and defers the "explicit one-pass propagation scheme" to ADR-LSG-03. The Foundation ADRs have since made decisions the GDD could not know: hooks are `Callable`s registered with `after` edges that are validated and sorted once, a hook receives a `WriteScope` and a `HookContext` and ends in `return ctx.complete()`, `HookDescriptor` has no `before` edge, division goes only through `IntMath`, a faulted scope closes, zero-allocation claims are restated as `HOT-01` plus object-count checks, and the kernel snapshot may be taken on any tick.

The review of the GDD against those decisions also found four points the GDD gets wrong or leaves open, and this ADR must settle each rather than repeat them: **(1)** LS-1 (j) and ADR-0011 § 6.2 describe the Jacobi snapshot as a `next` column swapped by reference, which fits a dense per-element rewrite but not the pairwise fluxes of the exchange phases; **(2)** LS-9's `subject_id < 2^20` key is shared by node ids and edge ids, which overlap numerically (ADR-0011 Risk 7); **(3)** LS-21 says `served_pm` is not persisted "because phase 2 recomputes it before use", but three hooks and other blocks' hooks read it afterwards, and `post_restore` cannot reproduce it from the post-phase state (the battery has been debited, the room temperature has moved and a cable may have failed since), so a restored run would diverge from the uninterrupted one (ADR-0011 adopted the hook-product columns by cross-ADR ruling CT-04, 2026-10-29); **(4)** LS-17's M19 advances five kinds of absolute step stamp on load but omits two (`thaw_until`, a LOAD's `energised_at`), so a thaw window or an inrush window would be spent while the section slept.

### Constraints

- **Kernel (ADR-0001, ADR-0003):** a hook runs when `T mod divisor == offset`, in a total order built once from `after` edges and registration index; `after` edges between hooks that do not always co-run must declare `tolerates_stale`; P2 (apply) precedes P3 (hooks) inside a tick; a hook ends in `return ctx.complete()`; the first fault closes the scope; the snapshot is taken at a tick boundary after P5, on any tick; P3 target is 2.0 ms P99 with `STEP_BUDGET_MS` an alarm, never a cut-off. `HookDescriptor.after` is the only ordering primitive (no `before`).
- **Determinism (ADR-0005):** integer-only; ascending-id iteration; no Dictionary iteration; no `sort_custom`; keyed RNG through `RngFacet.draw` with a plain stream for everything the Graph draws, never reading `restore_epoch`; every division through `IntMath`; every intermediate below 2^62 (ADR-0011 § 2.3, GDD D.13).
- **Data model (ADR-0011):** the columns, the four classes, the tombstone and canonical-zero rules, `GraphScratch` for "the accumulators and flux arrays ADR-0012 declares", the descriptor store built once in phase 11, the queue tables and their bounds, the fault domain 3 list, and the writer-file rule (OWN-01).
- **Source rules (ADR-0010):** `:=` banned and every `var` explicitly typed; `## @hot` functions free of allocation tokens (`.new(`, `.duplicate(`, `.resize(`, `.append_array(`, string building, lambdas); no `await`; no recursion or iterate-to-convergence loop (GDD AC, lint requested in Foundation Ask 3); work counted through an injected `WorkCounter`, never by wall clock.
- **Capacity and bounds (registry, cited by name):** `MAX_GRAPH_NODES`, `MAX_GRAPH_EDGES`, `MAX_SEAMS_PER_SECTION`, `MAX_SPREAD_PER_STEP`, `MAX_FOOTPRINTS_PER_STEP`, `MAX_CREW`, `GRAPH_STEP_TICKS`, `W_MIN_WARNING_S` (derived `W_MIN_STEPS`), `THAW_ROLL_STEPS`, `BURST_P_PM`, `BURST_HP_TO`, `DERELICT_FAULT_COMPRESSION`, `GRAPH_HOOK_BUDGET_MS`. Never retyped here.
- **Performance (R12):** every wall-clock figure is a P99 on a Godot 4.7.2 release export on RM-DESKTOP and RM-DECK; the per-commit gates are deterministic counts.

### Requirements

- Three chained hooks at `GRAPH_STEP_TICKS`, twelve fixed phases, `step_no = floor(T / GRAPH_STEP_TICKS)` for all three, per-hook persisted cursors (TR-lsg-008, TR-lsg-009).
- One pass per loop, order-independent within a phase, no solver, no catch-up (TR-lsg-007), with the equalisation flux stable by construction (TR-lsg-032).
- One idempotent topology refresh that no hook ever reads half-applied (TR-lsg-010, TR-assembly-034).
- Keyed, restore-independent randomness and a persisted thaw window (TR-lsg-011, TR-lsg-012).
- The fairness gate as a sim rule with an exogenous bypass and a pure `alarm_sections()` (TR-lsg-013, TR-lsg-015).
- The mutator contract M1 to M19 and the three input queues with a deterministic drain schedule (TR-lsg-016, 017, 035, TR-suit-039).
- Weld, cut and discard data effects that only append or tombstone (TR-lsg-027, 028, TR-boarding-039, 041).
- Dormant sections skipped, frozen and re-based (TR-lsg-029, TR-webmap-041, TR-ftrail-038).
- Endogenous rates, derelict compression and the eight D.15 fault fixtures (TR-derelict-011, 012, 037).
- Hook, snapshot and restore budgets with a measurement protocol and work bounded by construction (TR-lsg-025, 026, 036, TR-interact-038).

## Decision

### 1. One step, three hooks, three cursors (TR-lsg-008)

A **step** is the unit of Graph time. `step_no = IntMath.floor_div(clock.tick(), tuning.graph_step_ticks)` for all three hooks (`GRAPH_STEP_TICKS`, the same constant as the hooks' divisor). The three hooks of step *n* run on ticks `6n + o_power`, `6n + o_atmo` and `6n + o_failure` with the offsets 0, 2 and 4. The offsets are data in `GraphTuning` (`hook_offset_power`, `hook_offset_atmo`, `hook_offset_failure`), so the GDD's data-only revert (all three 0) is a data change. A boot check (`GRAPH_BOOT_TUNING`, ADR-0011 § 8.4) requires the offsets to be non-decreasing and each at most `graph_step_ticks - 2`; the second rule makes `step_no` identical whether `ClockView.tick()` inside P3 names the executing tick or the next one (KRN-LAT-02 has not yet said which), because `floor_div(6n + o + 1, 6)` equals `n` for every `o <= 4`.

| Hook id | Offset | `after` | Phases | Cursor written at the end |
|---|---|---|---|---|
| `graph.propagate.power` | `hook_offset_power` (0) | none | 0 to 3 | `last_step_power` |
| `graph.propagate.atmo` | `hook_offset_atmo` (2) | `{id: graph.propagate.power, tolerates_stale: true}` | 4 to 7 | `last_step_atmo` |
| `graph.propagate.failure` | `hook_offset_failure` (4) | `{id: graph.propagate.atmo, tolerates_stale: true}` | 8 to 11 | `last_step_failure` and `last_step_no` |

The two edges fix the resolved order (power, then atmo, then failure) even though the hooks never run on one tick at the baseline offsets; `tolerates_stale` is declared on both so the kernel's boot warning list stays empty (ADR-0003 § 8 check 3). The Graph registers the three descriptors consecutively, power first, so registration index agrees with the edges. Dependents name the hook they read: Suit's `suit.vitals` is `after: graph.propagate.atmo` at the same offset, Attention is `after: graph.propagate.power`, Emitter State's `emitter.place` is `after: graph.propagate.failure` (all registered by their owners, architecture hook schedule).

**Cursor semantics.** `last_step_X` is the `step_no` of the last completed run of hook X. `last_step_no` is the last step whose **failure** hook (phase 11) completed (LS-8, CT-05; ADR-0011 § 2.3 adopts this wording). It trails `step_no()` by up to one step, and equals it only from the end of the failure hook until the next power-hook tick. The readers that compare it (Failure & Trail's poll cache, FT-15, and Emitter State's FAULT_GLOW reconcile) want a step whose descriptors have been emitted. A run applies a guard before any write: `step_no < last_step_X`, or `step_no == last_step_X` with `step_no > 0`, is a defect (the clock moved backwards or the hook ran twice) and raises `GraphFaults.HOOK_CURSOR` (runtime local code 25, ADR-0011 § 8.1). Step 0 is the one exception, because a fresh block holds all cursors at 0 and cannot tell "step 0 not yet run" from "step 0 done". **The tick counter T is gapless:** forgiveness (ADR-0001) may drop accumulated lag wall time, never a T value, so a hook keyed on `T mod C` is never skipped and a step is never skipped or repeated. A cursor gap therefore arises only from a refused or faulted hook, or from a restore; it is **not** repaired: the Graph never catches up and never runs a step twice (TR-lsg-007).

**Cursors across a save (CT-08).** The restore-time cursor check is not this ADR's: ADR-0011 § 7.2 step 13 compares each stored cursor with the stored schedule `hdr.sched` (`GRAPH_STEP_TICKS` and the three hook offsets in force when the cursors were written; local code 26 `RESTORE_HOOK_CURSOR`). A schedule change across a save is handled at the **first run of each hook after a load**, not at restore: the hook compares `hdr.sched` with the live `GraphTuning`; if they differ it rewrites `hdr.sched`, skips the hook when `last_step_X` already equals the current step (the one case where the guard above would otherwise fire), and runs once otherwise, counting `GraphDiag.schedule_shift`. That is legal only on the first run; the `HOOK_CURSOR` guard applies from then on.

**The hook skeleton** (the same for all three; `hook_power` shown):

```gdscript
# src/sim/graph/graph_module.gd (additions; ADR-0011 § Key Interfaces holds the rest)
# hook_atmo and hook_failure have the same shape and the same unused-parameter prefix (_views); hook_failure passes the module's RngFacet and its `_stream_fail` handle to run_failure (E2).
func hook_power(_views: SimViews, scope: WriteScope, ctx: HookContext) -> CompletionToken:
	if not scope.open_for(GraphModule.BLOCK_ID):
		return ctx.complete()                                  # REFUSED: the kernel counts it; nothing written
	var step_no: int = IntMath.floor_div(_clock.tick(), _tuning.graph_step_ticks)
	if GraphPropagate.enter(scope, _state, _caches, _diag, GraphPropagate.HOOK_POWER, step_no) != MutatorStatus.OK:
		return ctx.complete()                                  # cursor defect: scope already faulted
	GraphPropagate.run_power(_state, _caches, _scratch, _diag, _tuning, step_no, scope, _work)
	return ctx.complete()
```

`GraphPropagate.enter` performs the cursor guard, the head-of-hook topology guard (§ 4) and nothing else; `run_power` runs phases 0 to 3 and, after each phase, returns early if `scope.faulted` (a structural fault closes the scope, ADR-0003 § 5, and the partial writes stay for the diagnostic dump); the last statement of `run_*` writes the cursor. A hook never reads a wall clock, never calls another owner's mutator, never submits a command and never allocates in steady state.

An EMPTY block (no sections instantiated, or `default_state`) runs the hooks over empty active lists: they cost nothing and the cursors still advance, so a later `INIT` or restore finds them consistent with `T`.

### 2. The twelve phases (TR-lsg-009)

The order is fixed and is the same on every run. Each phase is one `## @hot` static function in `src/sim/graph/propagate/` taking its collaborators as arguments (no member access through a hidden `self`), so a phase can alias a column in a local for the length of the phase (ADR-0011 § 6.2 rule 1).

| # | Hook | Phase | Reads | Writes | Write discipline (§ 3) | Keys fired (§ 12) |
|---|---|---|---|---|---|---|
| 0 | power | **Inputs.** (a) stock reconcile (ADR-0011 § 7.3, agreed with ADR-0016 § 4.4); (b) drain `pending_load`, `pending_draw`, `pending_trace` (§ 5); (c) apply `forced_next` to `forced_mode`; (d) **close out the intensive caches** of every compartment that (b) touched (`n_o2` debited by a draw, `t` moved by CHILL or BURN) | queues, stores, `forced_next` | `stock`, `req`, `n_o2`, `t`, `t_rem`, `hp`, `hp_cap`, flags, `forced_mode`, `draw_grant`, section counters; caches `P`, `p_o2`, `o2_pm`, trend of touched compartments | OWN; ALLOC for draws | `NODE`/`EDGE` for each subject whose `hp` bucket or `forced_mode` changed |
| 1 | power | **Topology guard.** No-op while `topo_dirty` is clear (§ 4) | edges, flags | caches only | none | none |
| 2 | power | **POWER solve per island** (D.6): reactor ramp, thermostat scaling, inrush, tiers, battery flow, merit-order dispatch, cable net flow | `req`, `out_w`, `stock`, breakers, topology | `out_w`, `stock`, `n_served`, `n_draw`, cable load, `produced_dj`, power summary | ALLOC per island | `LOOP` (power), `NODE` (battery store bucket) |
| 3 | power | **FUEL** (D.2 family): burn, `fuel_rem`, transfer tank to burner, `starve_steps` | stocks, `out_w`, `path_cap` | `stock`, `fuel_rem`, `burned_uml`, `starve_steps` | ALLOC per component | `LOOP` (fuel), `NODE` (store bucket) |
| 4 | atmo | **O2** (D.5, D.2): scrubber output, vent injection, duct and tank leaks | `n_served`, pool, compartments | tank `stock`, `acc_n_o2`, `acc_n_inert`, `vented_mgu`, `breathed`/`produced` counters | ALLOC (pool), JAC (rooms) | `LOOP` (O2), `NODE` (tank bucket) |
| 5 | atmo | **COOLANT** (D.8): `circ`, capture, radiator, `t_cool`, pipe and reservoir leaks | `n_served`, `n_draw`, reservoirs, `t` | `stock`, `t_cool`, `tc_rem`, `spilled_*`, `acc_q` | ALLOC (reservoir), OWN, JAC (heat) | `LOOP` (coolant), `NODE` (reservoir bucket) |
| 6 | atmo | **Atmosphere exchange** (D.1, D.2): stored OPENINGs, virtual holes, seam weeps | `n_o2`, `n_inert`, `V` | `n_o2`, `n_inert`, `vented_mgu` (apply `acc_n_*`) | **JAC** | none (marked in 7) |
| 7 | atmo | **Thermal step and close-out** (D.7, D.3): conduction, hull loss, machine and heater input, remainder carry; then intensive recompute, bands, pressure trend | `t`, `t_rem`, `acc_q`, `n_draw` | `t`, `t_rem`, `radiated_dj`, `band_*`, `p_prev`; caches `P`, `p_o2`, `o2_pm`, trend | **JAC**, then OWN | `COMP`, `BAND` |
| 8 | failure | **Damage** (D.9): overload, wear, duty, short burn, thaw rolls, patch decay, with the fairness floor | cable load, `n_served`, `n_draw`, modes, flags, `thaw_until` | `hp`, `thaw_until`, EXOGENOUS flag, `GraphDiag.gate_holds` | OWN | `NODE`/`EDGE` (`hp` bucket) |
| 9 | failure | **State evaluation:** `state`, mode triggers and pre-stages, `warn_since`, `fail_since`, `sev`, onsets, compartment warn trackers | `hp`, `t`, bands, `n_served` | `state`, `forced_mode` (own row), `mode_key`, `mode_since`, `isolated_at`, `warn_*`, `fail_since`, flags, `thaw_until`; then `ensure_derived` if a transition changed connectivity (§ 4) | OWN | `NODE`, `EDGE`, `SEAM` |
| 10 | failure | **Cascades:** cross-subject rules on this run's phase-9 states, admitted in ascending subject id up to `MAX_SPREAD_PER_STEP` | phase-9 states | `forced_next` | ALLOC (the spread budget) | none |
| 11 | failure | **Emit:** descriptor store, alarm list, descriptor intensity buckets, `last_step_no` | everything | `GraphCaches` only; `last_step_no`; the cursor | none | `DESC` |

Notes on the table, each a decision:

- **Phase 0 is first because it is the only place where the inputs of the whole world are folded in.** The three queues are the unconditional channels (kernel CR-9 path (a)); folding them in at one point makes every Graph result a function of the persisted state at the start of the step, which is what lets a snapshot taken on any tick be replayed (§ 11).
- **Phase 1 is a guard, not work** (§ 4). The GDD runs the topology refresh "at the head of whichever Graph hook first runs after a structural mutator"; here every writer rebuilds before it returns, the head of each hook (`GraphPropagate.enter`) is a defect-only guard, and phase 1 of the power hook is that guard retained for the numbering the consumers (Debug Overlay, GDD) know.
- **Phase 6 applies phase 4's buffered gas deltas and phase 5's buffered heat is applied in phase 7**, so both exchange phases read the start-of-run amounts (nothing writes `n_o2`, `n_inert` or `t` between the start of phase 4 and the apply in phase 6, and nothing writes `t` between the start of phase 5 and the apply in phase 7). The O2 and coolant phases therefore need no snapshot copy: the live column *is* the start-of-phase snapshot until the apply.
- **Phase 9 evaluates a subject's own triggers; phase 10 evaluates cross-subject chains.** A subject's own row may take a forced mode in phase 9 (FREEZE once the gate is open; SCRAM after sustained OVERHEAT or fuel starve), gated by § 7. A chain whose trigger is subject A's state and whose effect lands on subject B (LS-12) queues B's `forced_next` in phase 10, and phase 0 of the *next* step applies it: one hop per step, never an instant cascade. A candidate that does not fit the spread budget is not stored: its trigger condition still holds, so phase 10 proposes it again next run, and the finite subject count bounds the wait.
- **Phase 11's "block_hash dirty flag" row of the GDD is obsolete.** The kernel invalidates the cached hash by the touch serial of every granted `open_for` (ADR-0005 § 5.2), so the Graph does nothing. This is a `gdd_sync` item.
- **The intensive caches (`P`, `p_o2`, `o2_pm`, trend) are a pure function of the persisted state at every tick boundary (T2, option (a)).** Phase 0 is the only writer of `n_o2`, `n_inert` and `t` before phase 6 and phase 7 (`draw_air` debits, CHILL and BURN add or remove heat); phases 4 and 5 only fill accumulators, and phases 6 and 7 apply them. Phase 0 therefore ends by recomputing the four caches for exactly the compartments it touched, an O(touched) OWN write with no new persisted state (`p_prev`, the trend's other input, is written by phase 7 only), and phase 7 recomputes all of them at the end of the step. Between 6n+0 and 6n+2 the caches hold the post-draw values, so a restore landing on 6n+0 or 6n+1 recomputes in `post_restore` exactly what the uninterrupted run holds; the "exact" label of § 11 is true, and no sim reader (`emitter.place`, the Interaction and Horror hooks, Sanity's Perception filter over `P`, `P_O2`, `T`) sees a stale value. Structural mutators keep the invariant the same way: `ensure_derived` recomputes the intensive caches of every compartment it (re)builds. The dirty keys `COMP`/`BAND` stay phase 7's (§ 12), which compares each bucket with the last-emitted one, so phase 0's recompute fires none.

### 3. One pass: three write disciplines (TR-lsg-007, TR-lsg-032)

A **pass** is one traversal in ascending slot order over a loop's active list in which no element reads a value written by another element of the same phase. The GDD's phrase "Jacobi from a double-buffered start-of-phase snapshot" is made exact by three disciplines, and every column a phase writes is declared with one of them in the phase's doc comment (a reviewer and a test can check the declaration against the code):

- **JAC (Jacobi).** Every element reads only start-of-phase values and writes only into a `GraphScratch` accumulator; the accumulators are applied to the live column in one ascending loop at the end of the phase (`col[i] += acc[i]; acc[i] = 0`). Used by the gas exchange (phase 6 plus phase 4's room deltas), the heat input (phases 5 and 7). Because the live column is untouched until the apply, it *is* the start-of-phase snapshot, so there is no copy and no swap.
- **ALLOC (ordered allocation).** A shared quantity (an island's supply, a tank pool, a reservoir's level, the spread budget) is allocated to claimants in ascending id order against a running balance; each claimant reads only the balance and its own row. Used by the power solve, fuel, the O2 pool, reservoir leaks, the draw queue and cascades. This is a single sequential pass, not iteration.
- **OWN (own-row).** An element reads and writes only its own row and values no other element writes in the phase. Used by damage, state evaluation, `t_cool`, queue application to a named subject.

**Why accumulators, not `next` columns (a refinement of ADR-0011 § 6.2 (4) and LS-1 (j)).** A `next` column per field needs a full copy before the pass (every untouched row must be carried over) and a swap after; probe (e) shows the dense case costs about half of the accumulate-and-apply form, so speed does not decide it. Three other things do. The exchange flux is pairwise (one `Δn` is subtracted from one room and added to another), so an accumulator makes conservation visible in one place: the sum of the accumulator is zero by construction apart from the `vented` term, and a test asserts it per run. The accumulator form touches only the rows with a flux, which keeps VS-A quiescence (a section with every flux zero) a pure skip. And the persisted column is never aliased across a kernel call, so no snapshot can capture a half-swapped column. The swap-by-reference rule of ADR-0011 stays available for any future field that is rewritten wholesale; none exists at MVP, so `GraphScratch` holds this inventory (`PackedInt64Array` accumulators and the island, candidate, sort and traversal buffers) and is all zero at every hook exit (CT-06; ADR-0011 § 1, § 6.1 and § 6.2 (4) carry the same wording). The total gas amount of a room is read inline as `n_o2[r] + n_inert[r]` from locals; no accessor for it is added to `GraphState` (CT-06, E3).

**The scratch inventory** (all `GraphScratch`, sized by `size_scratch()` in `graph_derived.gd` from every structural mutator and from `post_restore`, never in a phase; each is all-zero at every hook exit, asserted in tests):

| Member | Type | Rows | Phase (fill / apply) |
|---|---|---|---|
| `acc_n_o2`, `acc_n_inert` | `PackedInt64Array` | compartments | 4, 6 / 6 |
| `acc_q` | `PackedInt64Array` | compartments (dJ) | 5, 7 / 7 |
| `acc_vent_mgu` | `PackedInt64Array` | sections | 6 / 6 (folded into `vented_mgu` at the apply) |
| `island_*` (supply, demand, battery draw, charge, root) | `PackedInt32Array` / `PackedInt64Array` | power components | 2 |
| `cand_slot`, `cand_mode` | `PackedInt32Array` / `PackedByteArray` | nodes + edges (candidate list) | 9 / 10 |
| `q_key` | `PackedInt64Array` | longest queue | 0 (drain sort) |
| `stack`, `queue` | `PackedInt32Array` | nodes (iterative traversals, § 4) | 1 |
| `deg` | `PackedInt32Array` | compartments (the count of openings incident to each room, § 4, filled at the head of phase 6; `GraphScratch.deg_max_of(e)` returns the larger `deg` of edge `e`'s two rooms, or the one room's `deg` for a hull segment) | 6 (head) / 6 (read, re-zeroed at the phase tail) |

**The equalisation flux (TR-lsg-032).** The gas flux across one opening is the D.1 formula and nothing else; it is a pure function in `graph_rules.gd` so the Item DB lints, `can_join` and the phase call the same code:

```gdscript
# src/shared/graph/graph_rules.gd
## @hot  Magnitude of the D.1 flux across one opening; the caller takes the direction from the sign of g = n_a*v_b - n_b*v_a.
static func gas_flux(g_abs: int, v_a: int, v_b: int, a_m: int, deg_max: int, g_open: int, gas_scale: int) -> int:
	var first: int = IntMath.floor_div(g_open * a_m * g_abs, v_a * v_b * gas_scale * 1000)
	var second: int = IntMath.floor_div(g_abs, (v_a + v_b) * deg_max)
	return mini(first, second)
```

The gradient `g` uses the exact integer amounts at the start of the phase, never the floored display pressures, so a floor can neither reverse a flux near balance nor break monotonicity. The product `g_open * a_m * g_abs` is on the D.13 product list (worst case about 3 x 10^17, under 2^62). The stability argument (D.12): the second term caps one edge at `|g| / ((V_a + V_b) x deg_max)`, which removes at most `1/deg_max` of the gradient across that edge; with `Σ k <= 10^6` per room (asserted at boot over every template room and by `can_join` for both seam rooms, reason `STABILITY`) the update is a convex combination, so the maximum `|ΔP|` over all open edges is non-increasing and no edge's flux direction alternates from one run to the next. The thermal analogue is the second sum of D.12 and is enforced the same way. This ADR does not re-derive D.12; it fixes that the property is a **test**, not an argument: the S1 sealed stability fixture (a room of degree 4 or more at 1010 dkPa among rooms at 200) asserts non-increasing `max |ΔP|` and no direction flip every run until quiescence, on every commit that touches `propagate/` (§ Validation).

**No iteration to convergence, no recursion, no catch-up.** No phase contains a loop whose bound is a convergence condition; every loop is bounded by an active-list count, a table row count or a literal constant (the insertion sort of § 5 is a bounded `for j: int in range(i - 1, -1, -1)` with `break`, never `while`). No function is recursive. The Graph never runs a step twice and never runs a missed step (§ 1). Foundation Ask 3 asks for the lint (`PROP-01`) that enforces the first two.


### 4. Topology refresh and derived caches (TR-lsg-010, TR-assembly-034)

**One function, three call sites.** `GraphDerived.ensure_derived(...)` is the single idempotent rebuild of every topology-dependent cache; it returns at once while `caches.topo_dirty` is clear. It is called (a) at the end of **every structural mutator** (M1 to M6, M8, M9, M19) and in `post_restore`, so a weld, cut, discard, install or valve flip is complete, with its caches, when the mutator returns (the weld-preview drag and `can_join` never pay a rebuild, and no hook can interleave with a mutator because the kernel is single-threaded); (b) at the **end of phase 9** when a phase-driven state change altered connectivity (a CABLE, CABLE_JUNCTION, PIPE, DUCT or VALVE crossing the FAILED boundary, or a DESTROYED subject), so phases 10 and 11 (the descriptor store, the alarm list) and every reader after the failure hook see the split islands, and the power hook of the next step solves them (C1, D.15 BROKEN-BREAKER: "on run 1 the island splits"); and (c) as the **head guard** of all three hooks (`GraphPropagate.enter`), which is the GDD's "at the head of whichever Graph hook first runs after a structural mutator". Because (a) and (b) rebuild before control leaves the writer, `topo_dirty` is never left set across a tick boundary in a correct build, so a `GraphView` call to `ensure_derived()` (ADR-0011 § 9.1) is always the no-op guard it is documented to be and **no read ever pays a rebuild in an arbitrary presentation frame** (LS-19). The head guard is therefore **defect-only**: a hook that finds `topo_dirty` set has met a writer that forgot its rebuild; the guard still rebuilds (so the run is correct) and increments `GraphDiag.late_rebuilds`, which every Graph test asserts is zero. A view-side rebuild must not clear the flag silently: `GraphView` calls the same function and the same counter, so a rebuild outside a writer is never masked.

**What `ensure_derived` builds**, in this order, all into preallocated `GraphCaches` members sized by `size_derived()` (structural mutators and `post_restore` size them; the hook path never resizes): the four id-to-slot maps and the edge endpoint slots (ADR-0011 § 4.2); the **active lists** `active_nodes`, `active_edges`, `active_comps`, `active_seams` (ascending slot, tombstones and dormant sections excluded, § 10), the typed sublists `store_slots` and `machine_slots`, and `exchange_links` (the stored OPENINGs, hull holes and seam weeps phase 6 walks); `V` and `exposed_tiles` per compartment through `RoomSource`; the adjacency CSR; the loop **components** for loops 1 to 4 and, for POWER, the **trees** and `path_cap`; the POWER island list; and the contents (not the sizes) of the scratch and the descriptor store. **`ensure_derived` never resizes (T9):** it recomputes counts and contents into members that `size_derived()`, `size_scratch()` and `size_descriptors()` (structural mutators and `post_restore` only) already sized, and calls none of them. A phase-driven connectivity change never changes a row count, so the phase-9 call cannot resize; the absence of `resize`, `duplicate` and `append_array` on the phase-9 path (including its `ensure_derived`) is part of its `HOT-01` coverage (§ 13.2). `degrees` and `deg_max` are **not** cached: an OPENING counts toward a room's degree only while its `area` is above 0 and a hull segment only while it has a hole, and both change without a structural mutator (M13, a phase-9 breach), so phase 6 recomputes them in one O(E) pass at its head into the scratch member `deg` (`deg_max_of(e)` is its accessor; the member is in the scratch inventory of § 3 and in ADR-0011's named `GraphScratch` list) (a refinement of LS-8's phase 1; `gdd_sync`).

**Connectivity predicate (the one definition).** An edge joins its two endpoints in a loop's component only if: a VALVE is open; a CABLE or seam cable has `state` not FAILED/DESTROYED (its `cap` is positive); a PIPE, DUCT or FUEL PIPE has `state` not FAILED/DESTROYED; a CABLE_JUNCTION has its breaker closed; and a port is not CAPPED or OPEN_END for flow purposes. OPENINGs are the atmosphere's exchange links, not loop topology, and do not enter components. This single predicate is why TR-boarding-041 holds without a special case: an uncapped stub of a cut derelict leaks from the pool of **its own component**, which includes the home network's pool only while the seam VALVE is open, because a closed valve splits the component.

**Algorithm.** Components are found by an iterative breadth-first traversal over `active_nodes` in ascending slot with a preallocated `stack`/`queue` in `GraphScratch`: the first unvisited node found is the lowest id of its component, so the root is the lowest id by construction; POWER trees take parent pointers and a BFS order, and tree flow is accumulated by walking that order in reverse. No function is recursive, and a POWER cycle found here (R12 forbids them; `POWER_CYCLE` rejects them at M1 and `can_join`) is `GraphFaults.STRUCT_CSR`. Cost is O(N + E), measured in S1; if the eager rebuild breaks the placeholder budgets of `weld_join` and `cut_detach` (the structural entry points of the GDD's S1 criterion), the lever is to rebuild only the loops named in a per-loop dirty mask, which changes no result.

### 5. The input queues and their drain (TR-lsg-017, TR-lsg-035, TR-suit-039)

The three unconditional channels are **queued**, persisted (ADR-0011 § 3.8) and **drained only in phase 0**, so a Graph step is a function of the persisted state at its start (§ 11).

**Enqueue** (inside the Class B mutator, in P2 or in a hook):

| Queue | Mutator | Enqueue rule | Bound | Overflow |
|---|---|---|---|---|
| `pending_load` | M14 `set_load_request(node_id, req_w)` | resolve the id (stale, tombstone, not LOAD, dormant section: no-op, `stale_calls`); **coalesce by node**: a second call for a node already queued overwrites its `load_req` in place | `GraphTuning.load_queue_max`, never reached while coalescing keeps the queue at one entry per LOAD node | drop and count `queue_dropped` (a content defect: more LOADs than the bound) |
| `pending_draw` | M15 `draw_air(compartment_id, crew_id, o2_mgu)` | resolve the cid; append `(cid, crew, mgu)` in arrival order; a non-positive `mgu` is dropped | `MAX_CREW` x compartment count (56 at MVP, about 480 in VS-A, more if a caller enqueues once per tick); `GRAPH_BOOT_TUNING` asserts every queue bound is at most 65,535, the width of the arrival field (E4) | drop and count `queue_dropped` |
| `pending_trace` | M17 `deposit_trace(kind, subject_id, magnitude)` | resolve; clamp the magnitude to its per-kind cap (counted `footprints_clamped`); STRESS with a magnitude other than the constant is dropped; append | `MAX_FOOTPRINTS_PER_STEP` | the call past the cap is dropped and counted `footprints_dropped` (a trace is a moment, not a queue) |

**The drain rule** (the kernel's total order, not a wall clock). A deposit drains at **the first execution of `graph.propagate.power` that follows it in kernel order**, where kernel order is: tick first, then P2 before P3, then position in the compiled schedule for that tick (ADR-0003 § 8: Kahn over `after` edges with the registration index as tie-break). `HookDescriptor` has no `before` edge, so a hook is "ordered before" the power hook exactly when the sort places it earlier on a tick; the GDD's "a hook that declares `before: graph.propagate.power`" cannot be registered and is withdrawn (`gdd_sync`). In practice:

| Caller | Where it runs | Drains |
|---|---|---|
| a deposit under **INIT** (M14 from Emitter State's INIT-scope E1, ADR-0017 D3.4; M19 from `init_world` makes no deposit) | N2 / INIT, before the first tick (stamped at the N2 clock: `T = 0`, `step_no` 0 for a new game) | phase 0 of the first `graph.propagate.power` execution after LOADING to RUNNING, which is the first RUNNING tick `T` with `T mod GRAPH_STEP_TICKS == hook_offset_power`; that tick is ADR-0017 D5.2's `arm_tick` |
| any P2 apply on a power-hook tick (`T mod GRAPH_STEP_TICKS == hook_offset_power`) | P2, before the P3 power hook | the same tick |
| any P2 apply on another tick | P2 | the next power-hook tick, at most `GRAPH_STEP_TICKS - 1` ticks later |
| Suit (`suit.vitals`, `after: graph.propagate.atmo`), Attention (`after: graph.propagate.power`) and Emitter State (`emitter.place`, `after: graph.propagate.failure`): the orderings their owners declare (architecture hook schedule) | P3, after the power hook of the same step | the next power-hook tick |
| any other hook placed after `graph.propagate.power` in the compiled order (Interaction, Horror, Controller, Assembly, Web Map and the rest: their positions belong to their ADRs, not to this one) | P3 | the next power-hook tick |
| a hook the compiled order places before the power hook on a power-hook tick | P3 | the same tick |

Risk 7's golden compiled-order test pins every ordering in the last three rows that this ADR does not declare; the general rule (the first power hook after the deposit in kernel order) is correct whatever the order turns out to be. TR-lsg-035 words the first rows as "phase 0 of the same tick"; that is true on a power-hook tick and otherwise reads "the next power-hook tick" (the TR text is clarified in `gdd_sync`, its meaning is unchanged). **An INIT pass is not a tick:** M14 under INIT enqueues into the persisted `pending_load` exactly as in P2 (so armed lamps have a defined drain tick), and M19 under INIT stamps `dormant_since` with the INIT `step_no` (0 for a new game) and its `TOPO`/`SECTION` marks are discarded under ADR-0003 § 4's INIT rule (the kernel delivers one `ALL` batch). Within a drain, entries are applied in **arrival order within the cap**, then in the order each queue's rule below requires.

**Drain** (phase 0 (b); every queue is emptied by the drain and `draw_grant` is rewritten):

1. **`pending_load`:** for each entry in arrival order, resolve the node (a node salvaged, or a section made dormant, since the call is skipped and counted), set `req = clampi(load_req, 0, req_max)`, and mark `NODE` if the value changed. Entries name distinct nodes, so order does not affect the result.
2. **`pending_draw`:** zero the whole `draw_grant` table, then sort the entries by the packed key `(cid << 32) | (crew << 16) | arrival_index` (E4, T6: `arrival_index` has 16 bits, `crew` 16 and `cid` the rest; `cid` is at most `GraphLimits.ID_MAX` = 2^24, so the key is below 2^56 and under the D.13 bound of 2^62) with a bounded insertion sort (a `for j: int in range(i - 1, -1, -1)` loop with `break`, over `x.q_key`; probe (c): 3.5 microseconds at 16 entries, 27.0 at 56 and 33.3 at 64 on random keys, 55.3 at 64 reverse-sorted, which is the bound; the typical queue is a few entries) and walk them: `got = mini(mgu, n_o2[c])`, `n_o2[c] -= got`, `breathed_mgu[section] += got`, `grant_req[crew] += mgu`, `grant_got[crew] += got`. The running balance is why the order is (compartment, crew): two crew sharing a thin room draw in a fixed order. `draw_grant` therefore holds this drain's totals until the next drain, and a restore on any tick returns the same grant (it is persisted). Inert gas is never drawn.
3. **`pending_trace`:** sort by `(subject_ref << 32) | (kind << 16) | arrival_index` (the same 16-bit arrival field; below 2^56) and apply in that order: CHILL and BURN add or remove heat in the compartment through the thermal remainder rule (`GraphThermal.add_heat`, which keeps `U = T x C + t_rem` exact); GOUGE lowers a HULL_SEGMENT's `hp` by the clamped magnitude and sets FATIGUED; RUPTURE sets `hp = min(hp, RUPTURE_HP)` on a hull, duct or pipe and starts the LEAK or BREACH; STRESS sets the section's `stress_until = step_no + STRESS_HOLD_STEPS` and `stress_pm = BREACH_PROPAGATION_MULT`. GOUGE and RUPTURE are **EXOGENOUS** (the flag of § 7), so they bypass the fairness gate; a GOUGE on a duct or cable is dropped and counted at MVP (VS-A).

**Suit's breath, synchronous or not (TR-suit-039).** The Graph returns the `draw_air` grant **only** through the persisted `draw_grant` table at the following drain; it has no synchronous path. Whether Suit's breath-source, rung and cap sequences are identical under a synchronous grant is ADR-0027's property to test. What this ADR guarantees is that the queued path is deterministic, persisted and restore-exact: `pending_draw` is in the snapshot, so a restore between the call and the drain credits the same `mgu` at the same drain.

**Bounded work.** A drain is O(queue lengths): at most the LOAD-node count, `MAX_CREW` x compartment count and `MAX_FOOTPRINTS_PER_STEP` entries. Nothing here allocates; the sort buffer is `x.q_key`, sized to the longest queue by `size_scratch()`.

### 6. The mutator contract (TR-lsg-016)

The Graph registers **no command** (kernel H2); its mutators are plain methods of `GraphMutators` handed to the owning modules' `setup()` (ADR-0003 § 10). Every mutator takes the `WriteScope` first and begins with `scope.open_for(GraphModule.BLOCK_ID)`; a `false` return is `MutatorStatus.REFUSED` (Class A) or a silent no-op (Class B), writes nothing and is never a fault (ADR-0003 § 5). A defect raises `GraphFault.raise_in(scope, code, subject)` and returns `-code`.

**Class A** (M1 to M13, M19) pairs each mutator with a pure `check_*(views, args) -> StringName` (`&""` is OK) that reads the caches and tables only, allocates nothing and writes nothing; the mutator is **infallible once its check passed**: it neither re-checks the domain nor returns a domain error, and it returns a status plus data in a caller-owned `out` object. **Class B** (M14 to M18) clamps, never rejects, and treats an unknown, tombstoned or dormant-section id as a no-op counted in `stale_calls`. The closed reason sets are the GDD's (LS-15 for `can_join`, LS-18 for the rest) with the one addition below; ADR-0021 closes the transaction around them.

| # | Mutator (check) | Scope phases | Timing | Writes and topology | Keys |
|---|---|---|---|---|---|
| M1 | `instantiate_section` (`check_instantiate_section`: the LS-18 reasons plus `ONSET_UNKEYED` and `CAPACITY`; the same pure function ADR-0011 § 9.5's `GraphLayoutAdapter` exposes through `LayoutGraphPort`) | APPLY, INIT | immediate | appends rows in template order and returns ADR-0013's `SectionBases {node_base, edge_base, comp_base}` through a caller-owned out; sets `rate_pm` and `stress_pm`; back-dated stamps set HAS_WARN; rebuilds | `T`, `SEC/<id>` (discarded in INIT; the kernel delivers one `ALL`) |
| M2 | `weld_join` (`can_join`, `can_join_static`) | APPLY | immediate | appends sub-edges, one SeamRecord and its lists, every sub-edge disconnected (area 0, valves closed, breakers open); `built_step = step_no`; rebuilds | `T`, `S/<seam>`, `SEC/<a>`, `SEC/<b>` |
| M3 | `cut_detach` (`check_cut_detach`) | APPLY | immediate, atomic | tombstones seam edges, flips the SeamRecord to CUT; SEALED caps ports, RAGGED appends `A_CUT` stub rows and sets OPEN_END; **every row it severs or converts gets the EXOGENOUS flag**; rebuilds | `T`, `S/<seam>`, `SEC/<a>`, `SEC/<b>`, `C/<cid>` of affected rooms |
| M4 | `discard_section` (`check_discard_section`) | APPLY | immediate | tombstones every row; mass leaves through `disc_*`; rebuilds | `T`, `SEC/<id>` |
| M5 | `install_module` (`check_install_module`) | APPLY, HOOK | immediate | ADR-0011 § 5 column table; rebuilds | `T`, `N/<id>` |
| M6 | `salvage_node` (`check_salvage_node`) | APPLY, HOOK | immediate | ADR-0011 § 5; EXOGENOUS flag on the node and its severed edges; rebuilds | `T`, `N/<id>` |
| M7 | `repair_step` (`check_repair_step`) | APPLY, HOOK | immediate | `hp`, `hp_cap`, `patched_at`; does not touch `state` (phase 9 re-evaluates) | `N`/`E` (`hp` bucket) |
| M8, M9 | `set_valve`, `set_breaker` | APPLY, HOOK | immediate | the OPEN or BREAKER_CLOSED flag; connectivity changes, so rebuilds; M9 close stamps `energised_at` on the branch's loads | `E`/`N`, `T` |
| M10 | `set_setpoint` | APPLY, HOOK | immediate | `req` of a PUMP (`throttle_pm`) or VENT (`setpoint_pp`), clamped; **not queued** | `N` |
| M11, M12 | `reset_node`, `cap_port` | APPLY, HOOK | immediate | clears SCRAM or FREEZE only if the clear condition holds (`BLOCKED` otherwise, a physical answer); CAPPED flag | `N` |
| M13 | `set_opening_area` | **APPLY only** | immediate | `area` (the only writer of `area`); no topology change | `E` |
| M14 | `set_load_request` | APPLY, HOOK, **INIT** (Emitter State's INIT-scope E1, ADR-0017 D3.4) | **queued** (§ 5; an INIT deposit drains at the first power hook after LOADING to RUNNING) | `pending_load` only | none until drained |
| M15, M17 | `draw_air`, `deposit_trace` | APPLY, HOOK | **queued** (§ 5) | the two queues only | none until drained |
| M16 | `withdraw_store` | APPLY, HOOK | immediate, returns the grant | clamps to `stock`, moves it to `wd_*`; never appears in `power_use` | `N` (store bucket) |
| M18 | `apply_damage` | APPLY, HOOK | immediate | `hp` through the § 7 gate for ENDOGENOUS, bypassing it for EXOGENOUS; sets or keeps the EXOGENOUS flag | `N`/`E` |
| M19 | `set_section_dormant` (`check_set_section_dormant`) | APPLY, **INIT** (Web Map `init_world` step (6), WM-7, makes A, B and the pod dormant) | immediate; under INIT stamps `dormant_since` with the INIT `step_no` (0 for a new game) | § 10; rebuilds | `T`, `SEC/<id>` (both discarded under INIT: one `ALL` batch, ADR-0003 § 4) |

Rules that apply to the whole table:

- **No mutator is called from the Graph's own hooks**, and **no phase calls a mutator**: phases write their columns directly (they are writer files, ADR-0011 § 12.1) and mark keys themselves (§ 12). A mutator reads a phase's results only through persisted columns. M19's callers are `init_world` under INIT and P2 applies (departure, pod_settle, berth, reconcile), never a Graph hook.
- **M1's only caller is Layout LM1 through `LayoutGraphPort`; LM1's callers are those ADR-0013 § 9 names (Web Map `init_world` in the INIT scope at the MVP).** The chain is Web Map to LM1 to the port, because Layout allocates `section_id` first; M1 takes `section_id` as an argument and returns `SectionBases` (CT-01). The out type is the port owner's name; there is no `InstantiateResult`.
- **`M13` and Layout (ADR-0013).** M13 writes `area`; no phase ever does, so Layout's template-door state is untouched in P3, and `set_opening_area` is never called from a hook (ADR-0013's guideline). The door-motor power precondition is Interaction's, not the Graph's.
- **`M10` is immediate and `req` is persisted**, so a throttle change mid-step is seen by the next phase that reads `req`; the heat a machine injects is taken from its delivered power `n_draw` of the last solve (§ 11), not from `req`, so a mid-step change never creates or destroys energy.
- **Stale and cross-section ids.** Class B calls on a dormant section's ids are no-ops (`stale_calls`); a Class A check on one returns `SECTION_NOT_LOADED` where the GDD's reason set has it, otherwise the closed reason the GDD names.
- **Budgets.** Every `check_*` is O(1) or O(plan size), allocation-free and `## @hot`. `check_set_opening_area` and `check_set_valve` (a handful of column reads each) may run in the Interaction hook's reads or validates, and count against its budget (TR-interact-038); an **applied** M13 (one column write plus one `mark_dirty`) is APPLY only and runs in P2 apply, where the 10 SYSTEM door resubmits per tick land (ADR-0013, ADR-0020), so its cost is not the Interaction hook's. S1 reports the Graph's P2 share under the TR-interact-038 load separately from the hook share (§ 13.6).

### 7. The fairness gate (TR-lsg-013, TR-lsg-015)

The gate is a **floor applied at the moment damage is applied**, never a post-pass that rewrites `hp` afterwards: it limits the reduction and can never raise `hp` (D.9). The rule is one pure function so phase 8, M18 and the tests call identical code:

```gdscript
# src/shared/graph/graph_rules.gd
## Lowest hp an endogenous loss may leave behind; 0 means no floor (gate open).
static func gate_floor(hp_before: int, hull_or_seam: bool, gate_open: bool, hp_fail: int, breach_hp: int) -> int:
	if gate_open:
		return 0
	if hull_or_seam and hp_before >= breach_hp:
		return breach_hp                       # the LEAK -> BREACH escalation is gated
	if hp_before > hp_fail:
		return hp_fail + 1                     # FAILED/DESTROYED is gated
	return 1                                   # already FAILED: DESTROYED is gated

## hp left after applying a loss; never above hp_before.
static func apply_loss(hp_before: int, loss: int, floor_hp: int) -> int:
	return hp_before - mini(loss, maxi(0, hp_before - floor_hp))
```

- **`gate_open`** is `HAS_WARN ∧ step_no - warn_since >= W_MIN_STEPS` for a node, edge or seam, and the compartment tracker's bit and stamp for a compartment descriptor (LS-13 (b), D.11). `W_MIN_STEPS` is derived once at boot as `IntMath.floor_div(W_MIN_WARNING_S x TICK_RATE, GRAPH_STEP_TICKS)` by the loader and handed to the Graph in `GraphTuning`; the Graph never names `TICK_RATE` (ADR-0011 § 12.2 token list).
- **Evaluation order inside a step.** Phase 8 evaluates the gate from the `warn_since`/HAS_WARN that phase 9 of an **earlier** run (or M1) stamped; phase 9 then stamps this run's `sev`; phase 11 evaluates `gate_open` again for the descriptors. A warning that begins in phase 9 of step *s* therefore has `warn_elapsed` 0 at phase 11 of *s* and first opens the gate when `step_no - warn_since` reaches `W_MIN_STEPS`. D.11's worked examples are reproduced exactly: the overload cable is held on steps 70 to 100 and fails on step 101; the compartment alarm flips at step 125.
- **Held damage is gone.** The excess is discarded, never carried over; `GraphDiag.gate_holds` records the subject and the steps held; a subject already below its floor takes no further endogenous damage until the gate opens and is never moved.
- **EXOGENOUS bypass and the cause flag.** A damage source declares its cause. EXOGENOUS (floor 0): player verbs (cut, vent, salvage, the `INRUSH_STEPS` overload after a breaker close, tagged in phase 8 from `energised_at`), M17 RUPTURE and GOUGE, and M18 with `cause = EXOGENOUS`. The node and edge EXOGENOUS flag bit (ADR-0011 § 3.2, the descriptor's `cause`) is **set by an exogenous apply and cleared when the subject's dominant mode returns to 0 in phase 9 or its `hp` is repaired to `HP_OK`**, so it names the cause of the *current episode*: an endogenous wear tick on a ruptured pipe does not flip the descriptor back to ENDOGENOUS. (ADR-0011 Open Question 3 asked the GDD to name `cause` as persisted; this rule says when it clears. `gdd_sync`.) A cut completing on a running machine is EXOGENOUS because M3 and M6 set the flag on what they sever: the Graph re-solves the island on its next power hook, dependants go STARVE (a status, not an `hp` change, so no gate and no minimum-warning wait), and the SEVERED descriptors carry `cause = EXOGENOUS`, which is what Attention reads to open no breach-window row (TR-boarding-039; Attention's own rule is ADR-0022).
- **Alarm eligibility.** `alarm_eligible = alarm_capable(mode) ∧ sev >= ALARM_SEV_MIN ∧ gate_open`, evaluated per descriptor in phase 11. `alarm_capable` is the closed LS-13 (a) set (modes 1 to 5, 7 to 9 and 20 to 23; never 6, 10, 24 to 27), held as `GraphIds.ALARM_CAPABLE_MASK` (a code-side rule, not tunable, bit `m` set for each capable mode `m`; boot checks it against the mode table's `alarm_capable` column, ADR-0007). **`alarm_sections()`** is a pure read: phase 11 sets a per-section seen byte in scratch during its single pass over the descriptor store and then writes `caches.alarm_sections` by walking the sections table in ascending id, so the list is ascending and duplicate-free by construction; `GraphView.alarm_sections(out)` copies it into the caller's buffer (ADR-0011 § 9.1). It is empty before the first capable descriptor's gate opens and holds the section from that run (the D.11 Example B behaviour).
- **Dormant sections** freeze `warn_since` (§ 10), so a dormant section's descriptors would never close their gate; `soonest_warning_remaining(loaded_only)` skips them when asked.

### 8. Endogenous damage, randomness, cascades and onsets (TR-lsg-011, TR-lsg-012, TR-derelict-011, TR-derelict-037, TR-ftrail-038)

**8.1 Damage (phase 8).** One pass over `active_edges` then `active_nodes` in ascending slot, OWN writes only. Each subject's loss is the D.9 sum `Δhp = IntMath.floor_div((Δovl + Δwear + Δshort) x rate_pm x stress_pm, 1000000)` with the product bounded by D.13, passed through `gate_floor` and `apply_loss` (§ 7) and then through the thaw and patch rules. Decisions this ADR adds to D.9:

- **`rate_pm` and `stress_pm` scale endogenous `hp` loss and nothing else.** `rate_pm` is read only in `phase_damage.gd` (a repository test greps for it, ADR-0010 `READ-03`-style, with a bad fixture): gas exchange, heat conduction and supply never see it, which is the GDD's "the breach rhythm makes the ship fail faster, not leak faster". `rate_pm` is set once, by M1: `1000` for a HOME section, `IntMath.floor_div(1000000, tuning.derelict_fault_compression)` for a DERELICT (`DERELICT_FAULT_COMPRESSION`, giving the doubled wear of TR-derelict-011) and persisted in the sections table. Clocks set by a stock draining (a tank emptying, a pool venting) do not pass through it, so the FUEL-DRIP and O2-SHIP clocks do not compress. The stress multiplier is `stress_pm` while `step_no < stress_until`, else 1000.
- **Leak wear is gated on the persisted isolation stamp (TR-derelict-037).** A subject in LEAK takes `LEAK_WEAR_BP_PER_100` only while its `isolated_at` stamp is 0, that is, while phase 9 of an earlier run has not found it isolated by a closed valve; behind a closed valve the stamp is non-zero, wear is skipped, `hp` is held and the leak resumes from the stored `hp` when the valve reopens. *(Amended 2026-10-29 — seeded-stamp clear rule, Derelict DA-17)* The stamp is a persisted fact, not a claim about a valve: an `isolated_at` seeded by M1 on a subject that has no isolating valve holds wear the same way, and § 8.4 states when it clears. The GDD words the test as "descriptor `intensity_pm` > 0"; the descriptor store is rebuilt in phase 11, after phase 8, so the persisted stamp is the same fact in a form phase 8 can read and a restore reproduces. A seam weep at `hp` above `LEAK_HP` carries no LEAK wear (D.9). The dependent authored-staging tests (Derelict Authoring's closed-valve fixtures) fail if this rule changes, which is the TR's requirement.
- **"Dead" faults emit `intensity_pm` 0 until reached (TR-derelict-011).** A derelict subject authored at or below `HP_FAIL` with a powered-off module has no flow, no served power and no coolant, so every flow-fraction descriptor it owns evaluates to intensity 0 and the descriptor is emitted with a large `since_steps`; nothing special-cases it, and it wakes when a solve or a leak reaches it.
- **Short burn and the island test.** A FAILED cable in SHORT keeps burning only while an island at either end holds a live source. That test is evaluated in phase 8 from the **persisted** `out_w` of the sources and `stock` of the batteries in the cable's component (one O(islands) pass into `x.island_alive` at the head of phase 8), not from a phase-2 cache, so a restore reproduces it (§ 11; a `gdd_sync` item rewords LS-11's "this run's phase-2 result").
- **The peak and the `INRUSH_STEPS` tag.** Overload damage within `INRUSH_STEPS` of the branch's `energised_at` is tagged EXOGENOUS (a breaker closed onto an overload is a player verb); after the window the same overload is ENDOGENOUS and gated.

**8.2 Randomness (phase 8 and phase 9).** The Graph registers one **plain** stream at `BOOTING`, `graph.fail`, through `ModuleFacet.rng` (ADR-0005 § 2.3, `epoch_keyed = false`), and draws through `RngFacet.draw(stream, counter)`. The counter is an owner-defined non-negative 63-bit key:

```gdscript
# src/shared/graph/graph_rules.gd
const RNG_NODE: int = 0
const RNG_EDGE: int = 1
const RNG_COMP: int = 2                       # reserved: spark ignition (VS-A)
## 26-bit subject key: 2-bit kind, 24-bit id (GraphLimits.ID_MAX = 2^24). Node and edge ids overlap numerically; the kind bits keep their draws apart.
static func rng_subject(kind: int, id: int) -> int:
	return (kind << 24) | id                  # id < GraphLimits.ID_MAX <= 2^24 by construction (ids are never reused and never exceed ID_MAX)
static func rng_counter(step_no: int, kind: int, id: int) -> int:
	return (step_no << 26) | rng_subject(kind, id)       # step_no < STEP_STAMP_MAX (2^30), so the key stays below 2^56
## Burst test: a pure integer comparison; no float, no bare modulo.
static func burst_hit(raw: int, burst_p_pm: int) -> bool:
	return IntMath.posmod_i(raw & 0x7FFFFFFFFFFFFFFF, 1000) < burst_p_pm
```

The id field is sized to `GraphLimits.ID_MAX` (2^24), not to the MVP slot counts (T7): ADR-0011 sets `ID_MAX` precisely because VS-A id churn (compaction, repeated weld and cut) can push ids past any slot count, and an 18-bit field fitted to `MAX_GRAPH_NODES` would let node and edge draw keys overlap again, the very defect this section fixes. The 2-bit kind field leaves room for the VS-A compartment-keyed spark ignition. The layout fixes ADR-0011 Risk 7 (a thaw roll for node 5 and for edge 5 drew the same value) and widens the GDD's `(step_no << 20) | subject` form to `(step_no << 26) | subject`; the GDD's key text and the TR-lsg-011 registry text change in `gdd_sync`, and ADR-0005's golden vectors for this counter are regenerated from `rng_counter` (a pure function, so any vector for the old layout is simply a different counter value, not a defect). `GRAPH_BOOT_TUNING` asserts the relation `GraphLimits.ID_MAX <= 2^24`; no per-call id check is needed because the allocator cannot exceed `ID_MAX`. ADR-0005 § 2.4 already allows "any owner-defined 63-bit key", so no Foundation change is needed. The draw is a pure function of `(world_seed, step_no, kind, id)`: it reads **no** restore epoch, so a reload replays the identical roll at the identical step (user ruling D16/X8), and it is unaffected by weld, cut, restore or content added elsewhere. Draws are not counted or stored; there is no counter to persist.

**The thaw window** (TR-lsg-012) is persisted state, never inferred. In phase 9, when a FREEZE clears (`T_room > FREEZE_OFF_MC`), the subject's `thaw_until` becomes `step_no + THAW_ROLL_STEPS`; a re-freeze (forced FAILED again) sets it to 0 at once. In phase 8 of every later run with `thaw_until != 0`: if `step_no <= thaw_until`, the subject rolls exactly once (`burst_hit(rng.draw(stream, rng_counter(step_no, kind, id)), burst_p_pm)`, with `rng` and `stream` the `RngFacet` and the `_stream_fail` handle that `hook_failure` passes into `run_failure`, E2); a hit sets `hp = min(hp, BURST_HP_TO)` (strictly between `HP_FAIL` and `LEAK_HP`, so it starts a LEAK and never FAILs the pipe by itself, and it passes through the gate with a floor that cannot bind), is ENDOGENOUS, and sets `thaw_until` to 0; if `step_no > thaw_until` the window ran out unspent and is set to 0 without a roll. A restore on any tick therefore neither re-rolls a spent window nor forgets an open one, and a dormant section's open window is re-based by M19 (§ 10). Only subjects that can freeze (coolant and liquid PIPEs, RESERVOIRs, liquid tanks) ever have a non-zero `thaw_until`.

**8.3 Cascades (phases 9 and 10).** Phase 9 evaluates a subject's **own** triggers and writes only its own row (OWN): the hazard modes of LS-11, the pre-stages, and the forced modes that are a function of the subject's own state (FREEZE's forced FAILED once the gate is open, SCRAM after sustained OVERHEAT or fuel starve), gated by § 7. Phase 10 evaluates the **cross-subject** chains of LS-12 on this run's phase-9 states: a rule whose trigger is subject A's state and whose effect lands on subject B appends a candidate `(slot, mode)` to `x.cand_*`. Candidates are produced in ascending (rule, subject) order and **admitted in that order up to `MAX_SPREAD_PER_STEP` new forced modes per run** (ALLOC against the spread budget); an admitted candidate writes B's `forced_next`, and phase 0 of the **next** step moves `forced_next` to `forced_mode` in ascending id order. The rest are not stored: their trigger conditions still hold, so the next run proposes them again, and because the subject count is finite and the lowest ids win, every candidate is admitted within `ceil(candidates / MAX_SPREAD_PER_STEP)` runs. The chain C1 to C8 therefore moves one hop per step at a bounded front speed (S1 records the front speed for a section driven to full failure). A target that has become SALVAGED, ABSENT, tombstoned or dormant before phase 0 is dropped.

**8.4 Onsets (TR-ftrail-038).** Phase 9 restamps a subject's `mode_since` only on the run its dominant descriptor's `(mode, sev)` key differs from the **persisted** `mode_key`, and stamps `isolated_at` on the run a closed valve first drives a flow-fraction descriptor to 0 and clears it when the valve reopens or the mode clears. Because the comparison is against the persisted key, a seeded triple `(mode_since, mode_key, isolated_at)` from M1 survives phase 9 of run 1 unchanged when the live key equals the seeded one. *(Amended 2026-10-29 — seeded-stamp clear rule, Derelict DA-17)* **Clear rule for a seeded stamp.** A non-zero `isolated_at` seeded by `instantiate_section` on a subject with **no isolating valve** (the Derelict B `pb_leak`, DA-17) persists while the seeded `(mode_since, mode_key, isolated_at)` triple matches the live key, so it holds leak wear (§ 8.1) exactly as a closed valve's stamp does, and it is cleared on the **first propagation run in which the subject's descriptor `intensity_pm` rises above 0** (flow reaches it), in addition to the existing clears when a closed valve reopens and when the mode clears. Phase 9 reads `intensity_pm` from the descriptor store of the previous run's phase 11, so the clear lands one run after the flow starts and the stamp reads 0 from then on; a later closed valve may stamp it afresh as usual. A stamp seeded on a subject whose `intensity_pm` is already above 0 at run 1 is cleared on run 1. **Seeded values M1 accepts:** `mode_since` is a signed i32 step stamp (ADR-0011 § 3.2: back-dated values may be at or below 0), so a negative `mode_since` with `mode_key != 0` is valid (`onset_valid` rejects only `mode_since != 0` with `mode_key == 0`, `ONSET_UNKEYED`) and the back-dating survives into `grow_steps`; `isolated_at` is positive (a stamp of 0 means not isolated, so an `isolated_at` seed is `>= 1`) and must be below `STEP_STAMP_MAX`. The Derelict `pb_leak` seed `(mode_since -14400, mode_key 11, isolated_at 1)` is accepted on this reading; a template that cannot back-date seeds `mode_since >= 1` and keeps the hold (it loses only the `grow_steps` back-dating). M1's check rejects `mode_since != 0` with `mode_key == 0` (`ONSET_UNKEYED`, the fixture loader's rejection), through the pure `graph_rules.onset_valid(mode_since, mode_key)` that the template validator of ADR-0007 also calls. M19 advances both onsets across a dormant interval (§ 10); a weld appends a **new** SeamRecord with `built_step` and `mode_since` set to the current step, so a re-weld after a cut restarts the seam's onsets while leaving every pre-existing subject's untouched.

### 9. Seams, welds, cuts and discard inside the propagation (TR-lsg-027, TR-lsg-028, TR-boarding-039, TR-boarding-041, TR-assembly-034)

The data effects of weld and cut are ADR-0011 § 4.1 (append only; tombstones; no compaction); this section fixes how the phases see them.

- **A weld moves nothing.** `weld_join` appends sub-edges, the SeamRecord and its lists with every sub-edge disconnected, so at the instant it returns the new rooms exchange nothing and the new cables carry nothing. It then rebuilds (§ 4). The seam weeps from the step it exists: phase 6 reads the seam list (`active_seams`) and treats `A_seam_m` as one more opening on each side (D.2, D.14); the wear-in taper of `leak_bp` is applied on period steps in phase 8 (OWN write on the seams table) and a seam `PATCH` (M7) sets it to 0 at once. Connecting is three lit verbs (M13, M8, M9), each a separate immediate mutator.
- **`can_join` is the stability gate.** Its closed reason set is LS-15's; its `STABILITY` reason re-computes the D.12 gas and thermal sums for both seam rooms with the plan's openings added and rejects above the bound, so the property that makes the one-pass exchange monotone (§ 3) is never broken by a weld (TR-lsg-032). `CAPACITY` counts slots including tombstones (ADR-0011 § 4.1 (2)) and, for a RAGGED cut, the appended stub rows. `JoinPlan` inputs are ADR-0011's and are not changed here; the `res_*` outputs are `can_join`'s and are never trusted as apply input (`weld_join` re-resolves).
- **A cut is atomic and only along seams.** `cut_detach` does everything in one P2 apply (§ 6), including the EXOGENOUS marking; the current step's exchange across the seam is simply not computed (the sub-edges are tombstoned before the next hook); a RAGGED cut's stub rows are ordinary HULL_SEGMENT rows (ADR-0011 § 3.6) that phase 6 treats as holes of area `A_CUT`, so the cut room vents to its steady state with no special code.
- **Discard.** `discard_section` tombstones the rows and moves the section's mass into `disc_*` so the conservation evaluators close (ADR-0011 § 6.4); ids are never reused.
- **Visibility.** A weld or cut applied in P2 on any tick is visible, whole, to the first Graph hook that runs after it, whichever of the three that is (power, atmo or failure by `T mod GRAPH_STEP_TICKS`): the mutator has already rebuilt every cache, and a hook runs strictly after P2 of its tick. There is no observable half-state. A step can therefore straddle a topology change (power phases on the old topology, atmo phases on the new one); no rule assumes same-step cross-loop consistency (LS-8).

### 10. Dormant sections (TR-lsg-029, TR-webmap-041)

A section is **dormant** when its derelict is not loaded. The flag is written only by M19.

- **Skipped by construction, at zero per-run cost.** `active_nodes`, `active_edges`, `active_comps` and `active_seams` list only live rows of non-dormant sections (§ 4); every phase iterates those lists, never the tables, so a dormant section contributes no instruction to any phase, no draw, no key and no descriptor. The price of the indirection is about 5 nanoseconds per active element (probe (d)). EXTERIOR (node 0) is always present and has no state; a vent to space from an active room credits `vented_mgu` to that room's section. The descriptor store is built from the active lists, so it holds **loaded sections only**; `FailureFilter.loaded_only` (ADR-0011 § 11) is kept for the type's stability and is always satisfied. A dormant section's descriptors are not visible to `failures()`; a consumer that needs one reads the cold-path views (Open Question 5).
- **Frozen and hashed unchanged.** No phase and no Class B call writes a dormant row (Class B calls are no-ops counted in `stale_calls`), so the section's persisted columns, and therefore the block hash contribution of its rows, are identical before and after a save and load while it sleeps. It cannot be welded (`SECTION_NOT_LOADED`); M19 refuses `SEAM_ACTIVE` when an ACTIVE seam joins it to a loaded section, so no flux ever couples a dormant room to an active one.
- **Unload (`dormant = true`):** set the DORMANT flag, set `dormant_since = step_no`, rebuild. Nothing else changes. Queue entries naming the section are dropped at the next drain (`stale_calls`).
- **Load (`dormant = false`)** re-bases every absolute step stamp so the section's clocks count only steps it ran, in one pass over the rows whose `section_id` is the section's (O(rows), not on any per-run path): with `d = step_no - dormant_since` (a negative `d` is a defect, `GraphFaults.STEP_RANGE`), add `d` to: `warn_since` where HAS_WARN is set; `fail_since` of every FAILED or DESTROYED subject; `mode_since` where `mode_key != 0`; `isolated_at` where non-zero; `patched_at` where `hp_cap < HP_FULL`; **`thaw_until` where non-zero** and **a LOAD's `energised_at` where non-zero** (the two stamps LS-17 omits: without them a thaw window or an inrush window would be spent while the section slept; `gdd_sync`); every SeamRecord's `built_step` and, where `mode_key != 0`, `mode_since`; and each compartment's `warn_o2`, `warn_p`, `warn_t` whose bit of `warn_bits` is set. A stamp that would pass `STEP_STAMP_MAX` is `GraphFaults.STEP_RANGE`. `stress_until` is deliberately **not** advanced (the breach rhythm's stress lapses while a section sleeps), `dormant_since` is then set to 0, the flag is cleared and the caches are rebuilt. There is **no fast-forward**: gas, heat, wear and the open leaks resume from the frozen values, so a derelict left mid-breach is still mid-breach (R13), and the fairness gate never counts as served a warning the player was away for.
- **What accrues only on loaded steps.** Because dormant rows are not visited, a derelict's lamp battery drain (a Graph LOAD on a BATTERY), its endogenous fault compression (`rate_pm`) and every leak clock advance on loaded steps only. A Suit top-up that shortens the lamp clock is Suit's M16 on the derelict's battery (ADR-0027); the Graph neither knows nor cares. The webmap requirement that a reload with a dormant section changes no partition hash, node `hp` or taken-salvage record is the property test of § Validation (a save at tick *t*, a restore, and a compare of the section's rows).

### 11. Hook products and restore equivalence (a finding of this ADR)

**The problem.** The kernel may snapshot on any tick (ADR-0003 § 8, ADR-0006), and the three hooks of a step are spread over ticks 6n+0, 6n+2 and 6n+4, so a restore can land between any two of them. A restored run must evolve exactly like the uninterrupted run (that is the point of persisting the queues, the cursors and the hysteresis memory). A *derived cache* is safe only if `post_restore` can rebuild exactly the value the uninterrupted run holds at that tick. LS-21 and ADR-0011 § 6.1 classify `served_pm` as derived, "because phase 2 recomputes it before use"; that is true for phase 2 of the next step, but the atmo hook, the failure hook and other blocks read it earlier, and the post-phase state no longer reproduces it: phase 2 has debited the batteries (the availability `D` of D.6 changes in the last step before a cell empties), the room temperature has moved (the heater's thermostat duty), a cable may have failed in phase 9, and a throttle may have changed (M10). A restore at 6n+3 would then feed phase 8 a different cable load or a different `served_pm` than the uninterrupted run, and the two runs would diverge in `hp` by one step's damage.

**The enumeration.** For every phase of the atmo and failure hooks, every input that an earlier phase of a *different* hook of the same step wrote, or that another block reads between hooks, classified:

| Value | Written by | Read by | Rule here |
|---|---|---|---|
| `served` (consumer satisfaction, permille) | phase 2 | phases 4, 5, 7, 8, 9; Emitter, Suit, Interaction, Instruments | **PERSISTED** (new node column `n_served`) |
| `draw` (power actually delivered to a consumer, dJ per step) | phase 2 | phases 5 and 7 (machine waste heat), 8 (HEATER `run_pm` for duty wear) | **PERSISTED** (new node column `n_draw`) |
| cable net load (W) | phase 2 | phase 8 (`ρ`, overload) | **PERSISTED** in the existing edge column `e0` for kind CABLE (the overlay is free there; ADR-0011 § 3.4) |
| reactor `out_w`, battery and tank `stock`, `fuel_rem`, `t_cool`, `tc_rem` | phases 2 to 5 | later hooks | already PERSISTED |
| island has a live source (`S + D > 0`) | derived from `out_w` and `stock` | phase 8 short burn, phase 9 SHORT | **RECOMPUTED AT CONSUMPTION** in phase 8 from persisted state (§ 8.1); not a cache |
| `circ`, `Q_cap`, `T_mach` (D.8) | phase 5 | phase 9 OVERHEAT and HOT, phase 11 | **RECOMPUTED AT CONSUMPTION** by `derive_t_mach(slot)` in phase 9 from the current persisted `t`, `t_cool`, level, `hp` and `n_served`; phase 5 uses its own local value for the heat it injects. Phase 9 therefore sees the room temperature of phase 7, one phase later than the GDD's phase-5 proxy (`gdd_sync`) |
| `P`, `p_o2`, `o2_pm`, trend | phase 0 (for the compartments it touched), phase 7 (all), structural mutators via `ensure_derived`, and `post_restore` | everyone (`emitter.place` every tick, the Interaction and Horror hooks, Sanity's Perception filter over `P`, `P_O2`, `T`) | pure function of `n`, `V`, `t` and the persisted `p_prev` **at every tick boundary**, because phase 0 closes out what it changed (§ 2, T2 option (a)): DERIVED, exact |
| bands, warn trackers, `p_prev` | phase 7 and 9 | phase 9, phase 11 | already PERSISTED (hysteresis memory) |
| `isolated_at`, `mode_key`, `mode_since`, `warn_since`, `fail_since` | phase 9 | phase 8 of the next step | already PERSISTED |
| `forced_next` | phase 10 | phase 0 of the next step | already PERSISTED |
| topology caches | `ensure_derived` | all | DERIVED; rebuilt from persisted connectivity state; equal at every hook head by § 4 |
| accumulators | phases 4 to 7 | within one hook | SCRATCH, zero at every hook exit |
| descriptor store, alarm list, loop summaries (`GraphCaches.loop_sum`, written by phases 2 to 5), last-emitted buckets | phase 11, phases 2 to 5 | Failure & Trail (presentation), **Emitter State fault glow (sim)**, Attention, Suit | DERIVED, re-derived in `post_restore`; may differ from the uninterrupted value **until the next phase 11 or next producing phase** when state moved since (window below); the last-emitted buckets are seeded from the restored values (ADR-0011 § 7.4 (4)) |
| `power_use` window (`delta_w_last5`) | phase 2 | Attention | **PERSISTED** as `hdr.power_use_w5` (five ints, CT-04); restored from it, never seeded flat. Attention consumes the window and KRN-RST-05 allows no exception list |

**Decision.** Two node columns, `n_served` and `n_draw` (`int32`, written by phase 2 only and zero for every node that is not a power consumer, canonical zero), an edge overlay (`e0` of a CABLE is its net load, with the sign convention and range ADR-0011 § 3.4 states) and the five-int header window `power_use_w5`, become **persisted hook products** (ADR-0011 adopts all of them by ruling CT-04; § 3.2, § 3.4, § 3.8 and § 6.1 there). Everything else in the table is persisted already, recomputed at its single point of consumption from persisted state, or accepted as a re-derived cache with a stated window. `run_pm` for a HEATER is `IntMath.floor_div(1000 x n_draw, maxi(1, req))` (the GDD's `floor(duty_t x served_pm / 1000)` without a separately stored `duty_t`); for every other consumer it stays `n_served` while `req > 0`, and the REACTOR's stays `floor(1000 x out_w / out_w_rated)` (GDD LS-19, `gdd_sync`). `GraphView.served_pm(id)` reads the persisted `n_served` column. `post_restore` re-derives `P`, `p_o2`, `o2_pm` and the trend and **never** `served`, `draw` or the cable load (ADR-0011 § 7.4 (3)).

**What this costs in the data model (adopted by ADR-0011, CT-04):** `n_served` and `n_draw` move `served_pm` out of the DERIVED class (ADR-0011 § 6.1, § 7.4 (3), Alternative 7, LS-21, TR-lsg-004 wording); `derived_members.json` loses `served_pm` and the schema fingerprint and `GraphSchema` gain two columns with the usual range and canonical-zero predicates; the F-MVP size grows by 8 bytes per node row, about **840 bytes at 105 nodes (plus 40 bytes for `power_use_w5`): from the probed 29,674 canonical bytes to about 30,514, and 30,554 with the window, against the 32 KB ceiling of TR-lsg-036, headroom from about 7 percent to about 4.6 percent**. ADR-0011 § 3.11 lever 1 (moving `thaw_until`, `isolated_at` and the non-fluid stamps into a side table, about 1.8 KB) restores about 10 percent and is the first lever if S3 measures above 31 KB. The alternatives (Alternatives 4 and 5 below) were to re-derive everything and accept the divergence, or to make every cache live-derived.

**What stays a window.** The descriptor store, the alarm list and the loop summaries are not persisted. In the uninterrupted run they hold the value **as of the last phase 11** (or the last producing phase); after a restore they hold the value **as of the restore tick**. They differ whenever Graph state moved in between (phases 0 to 7 of the current step and any mutator since the last phase 11), and the next phase 11 repairs the difference, so the window is at most one step (`GRAPH_STEP_TICKS - 1` ticks). Making them exact would mean rebuilding the store after every hook and every mutator (three O(subjects) rebuilds per step plus one per M13 resubmit), which costs more than the window is worth (Alternative 5). Their sim-visible reader is Emitter State's fault glow (`emitter.place`, every tick, ordered after the failure hook); their other readers are presentation. ADR-0017 decides whether fault-glow hysteresis makes the window insensitive; this ADR guarantees only that it is bounded by one step and measures it with RE-2.

**Tests (Validation).** *RE-1 (Graph persisted state):* for each restore tick *t* in a full 12-tick window across the fixture set (HEALTHY, PWR-OVL, the thaw window, a pending draw, a dormant derelict, **a restore at 6n+1 after a `draw_air` drain, and a restore at 6n+1 after a BURN trace**, which pin the phase-0 cache close-out of T2), snapshot at *t*, restore into a fresh kernel, and run to the end of the second failure hook after *t*: the `graph.state` block hash equals the uninterrupted run's **at every tick from *t*+1 (the first tick executed after the restore) onward**. Every hook product is persisted and the restore is byte-identical, so an equality that holds only from a later hook would hide a divergence that heals itself, the hardest kind to find later. *RE-2 (downstream):* the same run (with the same two added fixtures) compares `state_hash` over every SIM block and reports, per block, the first divergent tick; any divergence outside the descriptor-store window above fails, and the intensive caches (`P`, `p_o2`, `o2_pm`, trend) and `power_use_w5` are *not* in that window. *RE-3 (determinism):* two restores of the same bytes with the same epoch and commands give identical hashes at every tick. The three former boundary cases (battery final step, thermostat flip, cable failure between hooks) are ordinary RE-1 fixtures; there is no known-divergence list.

### 12. Dirty-key emission schedule (TR-lsg-020, residual)

ADR-0004 fixes the key encoding and the `MAX_DIRTY_KEYS` floor; ADR-0011 § 10 fixes the kinds, packing and quanta. This ADR fixes **which phase fires which key, on which tick**, with one rule: **a key is marked by the phase that wrote the data, on the tick it ran** (ADR-0011 § 10: "inside the mutator or phase that changed the data"), after comparing the new bucket with the `GraphCaches` last-emitted bucket, and the last-emitted bucket is updated at every mark, whoever marked it (a mutator or a phase), so a quiet value stays silent. Marking a key twice is harmless (the kernel dedupes) and under-marking is the defect.

| Kind | Fired in | Condition |
|---|---|---|
| `NODE` / `EDGE` | phases 2, 3, 4, 5 (store buckets); phases 0 and 8 (`hp` bucket, `forced_mode`); phase 9 (`state`, `forced_mode`, `sev` transitions) | the bucket of `hp` (250 bp) or of a store (20 permille) changed, or a discrete field changed. A phase-0 `RUPTURE` or `GOUGE` fires on the tick it is drained, not four ticks later |
| `LOOP` | the phase that recomputes the component summary: 2 (power), 3 (fuel), 4 (O2), 5 (coolant) | the summary crossed its 20 permille quantum |
| `COMP`, `BAND` | phase 7 only | one `C/<cid>` per compartment per step when the `T`, `P` or `p_o2` bucket changed since the last mark; a band transition fires `B/<cid>` |
| `SEAM` | phase 9 (state), phase 8 (`hp`, `leak_bp` bucket), M2, M3 | as `NODE` |
| `DESC` | phase 11 | the intensity bucket (100 permille) of a subject's descriptor changed; one key per subject |
| `TOPO`, `SECTION` | structural mutators (§ 6); `TOPO` also from phase 9 on the tick of the failure hook, when a phase-driven transition rebuilds the topology (§ 4), so the Projector rebuilds | `SECTION` never from a phase |

At the baseline offsets one hook runs per tick, so the marks of a tick are one hook's share: the D.10 worst designed cascade of 38 keys at MVP is spread over three ticks, and each tick is far under the floor of 128 the Graph asks the kernel to keep. With the data-only revert (all offsets 0) all three hooks mark on one tick; the 38-key MVP bound still holds, and the kernel's collapse to `ALL` above `MAX_DIRTY_KEYS` remains the safety. A derelict instantiation legitimately collapses to `ALL`. `post_restore` seeds every last-emitted bucket from the restored values (ADR-0011 § 7.4 (4)) so a restore yields one `ALL` and then silence.

### 13. Budgets and how they are measured (TR-lsg-025, TR-lsg-026, TR-lsg-036, TR-interact-038)

**13.1 The hook budget.** The Graph's budget of record is the **spike share** `GRAPH_HOOK_BUDGET_MS`: the sum of the Graph hooks that run on one hook tick at P99 and at max, on a Godot 4.7.2 release export (`forward_plus`, vsync on, 120-frame warm-up) on RM-DESKTOP and RM-DECK (R12). At the baseline offsets exactly one Graph hook runs per tick, so the share is the cost of the single hook that ran; with the revert it is their sum. `STEP_BUDGET_MS` is the kernel's alarm and the Graph never truncates, skips or reorders a phase because of it.

**The pass/fail figure is the worst single hook, not the step divided by three (T4).** The phase-to-hook split is fixed by the GDD (TR-lsg-009), and the failure hook (damage over every active node and edge, state evaluation, a possible `ensure_derived`, the descriptor build) is plausibly the heaviest, so an even split understates it. S1 and S0-f report P99 and max **per hook id** (power, atmo, failure) separately, plus the failure hook on a tick where phase 9 triggers `ensure_derived`; the figure compared with `GRAPH_HOOK_BUDGET_MS` is the maximum over the three hooks. **Ledger (CT-20):** the Graph's row in the ADR-0003 P3 share ledger is this worst-hook figure (0.5 ms P99 pending S1). On a Graph hook tick the Core entries sum to Graph 0.5 + Body 0.4 + Emitter 0.4 + `controller.step` 0.16 + `layout.doors` 0.2 = 1.66 ms against the 2.0 ms P3 target; the ledger is a foundation amendment to ADR-0003, and this ADR cites its row there rather than inventing a figure.

**Local arrays versus members (E3).** Probes (d) to (f) are local-array costs; member access measured about 64 nanoseconds per element and a method or static call about 120 nanoseconds on the editor binary; S1 decides. The phase budget rests on the D.16 coefficients and S1, not on those probes.

*Measurement protocol (S1 and S0-f).* The harness is an exported benchmark outside gdUnit4. Population: every `(tick, hook)` sample of the F-MVP 36,000-tick run after a 600-tick warm-up, with the scripted breach, reactor failure and three keyed failures of the GDD's S1 criterion, at offsets 0, 2, 4 and once unstaggered for comparison. Per-phase costs come from calling each `phase_NN` function alone (they are public statics in `propagate/`, `testing/GraphBench` wraps them) over F-MVP and a 4x synthetic fixture, 10^3 iterations each, which replaces the three coefficients of D.16. Also recorded: the cost of the **first hook after a snapshot** (copy-on-write: the first write to each shared column pays one copy, probed at 0.71 microseconds per 32 KB column by ADR-0011, about 70 columns), which must itself stay within the max; the Debug/Release factor; and the accessor, `Callable.call` and `IntMath.floor_div` overhead per millisecond that decides the levers.

*The ladder, in order, with what each rung may and may not do.* A rung is taken only when the one before it misses the share on the release export on RM-DECK:

| Rung | Lever | Status |
|---|---|---|
| 0 | typed GDScript, active lists, preallocated scratch, no per-run allocation, three disciplines of § 3 | this ADR |
| baseline | the 0 / 2 / 4 hook stagger (D.16's "rung 1"), in `GraphTuning`; reversible by data | this ADR, MVP baseline |
| ADR-0010 rung 1 | tuning inside the design: caps, loop merges, early-outs on empty queues and clean lists | no ADR change |
| ADR-0010 rung 2 | call-overhead removal: accessors cached in locals, a proven non-negative `floor_div` inlined in named `## @hot` files (needs the ADR-0010 amendment of ADR-0011 Open Question 1), merged passes | no result changes |
| VS-A only | island quiescence (D.16 "rung 2"), persisted flag | not MVP |
| ADR-0010 rung 3 | GDExtension for the propagate phases, GDScript kept as the golden reference | VS-A only, TD acceptance |

**MVP has no rung after ADR-0010 rung 2.** If S1 still misses, none of this ADR's levers solves it: the Graph re-requests its share from the technical-director (the D.16 and Open Question 31 route) and **the number moves, not the design**. The GDD's D.16 ladder numbers (stagger is its rung 1, quiescence its rung 2) and ADR-0010 § 9's (tuning, call overhead) name different things; this table reconciles them (`gdd_sync`).

**13.2 Zero allocation, restated (OQ-16, ADR-0010 § 7).** "Zero heap allocations per phase in steady state" is the conjunction of four checks. `HOT-01` over every function tagged `## @hot`: the twelve `phase_*` functions, `gas_flux`, `gate_floor`, `apply_loss`, `rng_counter`, `burst_hit`, the three queue drains, every `check_*`, and the hot getters of `GraphView`. `CNT-01`: `OBJECT_COUNT` equal before and after 600 ticks of F-MVP at steady state. `CNT-02` (debug binary): `MEMORY_STATIC` within the per-suite tolerance over 600 ticks, which sees retained growth. And the structural facts that make those checks meaningful: scratch, caches and the descriptor store are sized only by structural mutators and `post_restore` (`size_scratch()`, `size_derived()`, `size_descriptors()`); a phase never calls `resize`, `duplicate`, `append_array` or builds a string or Dictionary (this includes the `ensure_derived` call at the end of phase 9, which resizes nothing, § 4). `CNT-03` does not apply (no accessor returns a `Packed*Array`). The `HOT-01` tag list lives in `tools/lint/config/hot_functions/graph.json`.

**13.3 Work bounded by construction (TR-lsg-026).** Every phase loop is bounded by an active-list count or a literal: one pass per loop (§ 3); cascades by `MAX_SPREAD_PER_STEP`; trace writes by `MAX_FOOTPRINTS_PER_STEP`; seams by `MAX_SEAMS_PER_SECTION` (enforced by `can_join`); node and edge counts by `MAX_GRAPH_NODES` and `MAX_GRAPH_EDGES` (enforced by `CAPACITY` in M1 and `can_join`, and counted in slots including tombstones); dormant sections by their absence from the active lists. The per-commit gate is `CNT-04`: the injected `WorkCounter` counts, per phase, the nodes, edges, compartments, candidates and keys actually visited, and a test asserts the counts equal the closed forms (`|active_nodes|`, `|active_edges|`, ...) on each fixture, so a regression to an unbounded loop fails the build on any machine. The S1 wall-clock figures are the nightly benchmark (+25 percent over baseline fails), never a per-commit gate (ADR-0010 § 7).

**13.4 The presentation read side (0.1 ms P99 per frame).** Every `GraphView` hot getter is O(1) or O(k) in a small k (`nodes_in`, `neighbors`), allocation-free and tagged; `failures(filter, out)` scans the store once and is called once per step by Failure & Trail (FT-15), not per frame; `alarm_sections` copies a short list. The S1 read script reads, per frame, the `step_no`, the four atmosphere getters of each loaded compartment (14 at the weld set), `served_pm` of the lit emitters and the instrument needle values, and once per step one `failures()` and one `alarm_sections()`; its P99 per frame is the figure compared with the 0.1 ms. No getter ever calls a phase or a rebuild.

**13.5 Snapshot, restore and size (TR-lsg-036).** The Graph's budgets are: `snapshot(ctx)` (building the payload) at most 0.5 ms; the payload at most 32 KB canonical for the MVP ship; `restore` plus `post_restore` at most 2 ms; the canonical bytes byte-identical after restore, all measured on the release export on RM-DESKTOP and RM-DECK (S3, on the F-MVP fixture at the weld, mid-breach). The kernel's integration-world figures (3 ms snapshot and 10 ms restore for the 32-block world) are the kernel's (TR-cmd-032, ADR-0006 § 1.9); the Graph's share of them must not exceed its own numbers. Container work (canonical encode, hash, `var_to_bytes`) is not in the 0.5 ms: it is ADR-0005's and ADR-0006's and is measured by SAVE-WRK-1. Expected shape, all UNVERIFIED until S3: the snapshot is about 70 `duplicate()` calls and nine small Dictionaries (ADR-0011 probed 0.11 microseconds per 4,096-element column), far inside 0.5 ms; restore is a linear validation over about 70 columns and 600 rows plus a member-by-member copy, then `post_restore` (an `ensure_derived` rebuild, the bucket seeds and the structural invariants). Working split, to be replaced by S3: validation 0.8 ms, copy 0.1 ms, `post_restore` 1.1 ms. If the 32 KB is exceeded the levers are ADR-0011 § 3.11's, in its order, then a request to raise the ceiling with the measurement. The size after the hook-product columns is about 30.5 KB at the roster (plus 40 bytes for `power_use_w5`; § 11).

**13.6 The Interaction share (TR-interact-038).** Interaction's hook budget is Interaction's and ADR-0017's; what the Graph owes it is that every call it makes is bounded and allocation-free. `check_set_opening_area` and `check_set_valve` are a handful of column reads and may run in the Interaction hook's reads or validates (they count against its budget), and `GraphView` reads are O(1) getters. An **applied** M13 (one column write plus one `mark_dirty`) is APPLY only: the 10 SYSTEM door resubmits per tick are P2 commands (ADR-0013, ADR-0020), so their Graph cost lands in P2 apply, not in the Interaction hook (T8). The S1 harness runs the worst-case Interaction load of TR-interact-038 (10 SYSTEM door resubmits per tick, 12 attempts, 16 throws) against a real Graph and reports the Graph's **P2 share and its hook-side share separately**, to be reported against the respective budgets by ADR-0017's story; no Graph code runs on the Interaction hook's tick beyond the reads.

### Architecture Diagram

```
 tick T (kernel P0..P6)      P2 apply (commands -> Graph Class A/B mutators)        P3 hooks (compiled schedule, T mod C)
 -------------------------------------------------------------------------------------------------------------------------
  T = 6n+0                    M14/M15/M17 enqueue --------------------------+       graph.propagate.power   (phases 0..3)
  T = 6n+1                                                                  |
  T = 6n+2                                                                  |       graph.propagate.atmo    (phases 4..7)  <- Suit vitals (same tick, after)
  T = 6n+3                                                                  |
  T = 6n+4                                                                  |       graph.propagate.failure (phases 8..11) <- Emitter place (after)
  T = 6n+5                    snapshot/restore may fall here (any tick)     v
  T = 6n+6 = 6(n+1)+0         ----------------------------------------> drained at phase 0 of the first power hook that follows the deposit

 one STEP n  (step_no = floor(T / GRAPH_STEP_TICKS) for all three hooks; cursors last_step_power / _atmo / _failure / last_step_no)

   power hook                      atmo hook                          failure hook
   [0 inputs]  stock reconcile     [4 O2]      -> acc_n_o2/inert      [8  damage]   gate floor, thaw rolls (graph.fail)
               queues, forced_next [5 coolant] -> acc_q, t_cool       [9  state]    modes, onsets, warn trackers
   [1 guard]   ensure_derived?     [6 exchange] JAC apply n_*         [10 cascade]  <= MAX_SPREAD -> forced_next
   [2 power]   n_served,n_draw,    [7 thermal]  JAC apply t; bands,   [11 emit]     descriptors, alarm list, DESC keys,
               cable load (ALLOC)               COMP/BAND keys                      last_step_no
   [3 fuel]
        |  persisted products: n_served, n_draw, cable load, power_use_w5, out_w, stocks, band memory, forced_next, pending_*  |
        v                                                                                                          v
   GraphState (PERSISTED)  <--- written only by propagate/ phases, graph_mutators.gd, graph_restore.gd (OWN-01)
   GraphCaches (DERIVED)   active lists, slot maps, components/trees, intensive values, descriptor store, alarm list
   GraphScratch            acc_* accumulators, island_*, cand_*, q_key, stack/queue   (all-zero at every hook exit)
   GraphDiag               gate_holds, late_rebuilds, queue_dropped, work counters via WorkCounter port (never hashed)

 dormant section ----> absent from active_* lists ----> no phase touches it ----> M19 on load re-bases its absolute step stamps
```

### Key Interfaces

Each `class_name` is its own file. The snippets follow ADR-0010 typing (no `:=`, explicit types, typed `for` iterators `for i: int in n`, an underscore prefix on every parameter a body does not use, `IntMath` for division). The engine review probed the first draft under the ADR-0010 § 5.1 promoted-warning set (INFERRED_DECLARATION, UNUSED_PARAMETER, UNTYPED_DECLARATION, shadowing at error) and found three failures (untyped iterators, unused `views`, unused `d` and `scope`), all fixed below; the corrected snippets are **not yet re-probed**, so the sentence "load-checked in a scratch project under the ADR-0010 warning set" may be added only after LINT-SPIKE-1 or a re-probe loads them. ADR-0011's Key Interfaces (`GraphState`, `GraphFaults`, `GraphModule`, `GraphMutators`, `GraphSchema`) are not repeated; the additions are:

```gdscript
# src/sim/graph/graph_units.gd   (ruling R5: the WorkCounter unit ids of this module are per-module constants, the ADR-0017 `EmitterUnits` form; the port carries only add(unit: int, n: int) -> void)
class_name GraphUnits extends RefCounted
const EDGE: int = 1   # edge visits in a phase loop; further units are appended here as the CNT-04 closed forms of § 13.2 name them

# src/sim/graph/propagate/graph_propagate.gd   (writer files per class: ADR-0011 § 12.1 writer matrix; ADR-0010 Foundation Ask 1 accepts directory globs)
class_name GraphPropagate extends RefCounted
const HOOK_POWER: int = 0
const HOOK_ATMO: int = 1
const HOOK_FAILURE: int = 2

## Cursor guard + topology head guard; the only code that runs before a hook's phases.
static func enter(scope: WriteScope, s: GraphState, c: GraphCaches, d: GraphDiag, hook: int, step_no: int) -> int

static func run_power(s: GraphState, c: GraphCaches, x: GraphScratch, d: GraphDiag, t: GraphTuning, step_no: int, scope: WriteScope, w: WorkCounter) -> void
static func run_atmo(s: GraphState, c: GraphCaches, x: GraphScratch, d: GraphDiag, t: GraphTuning, step_no: int, scope: WriteScope, w: WorkCounter) -> void
# E2: ADR-0005's RngFacet.draw takes (stream: RngStream, counter: int) and the handle lives in GraphModule (_stream_fail), so the stream is passed beside the facet (not a GraphDraw object).
static func run_failure(s: GraphState, c: GraphCaches, x: GraphScratch, d: GraphDiag, t: GraphTuning, step_no: int, scope: WriteScope, rng: RngFacet, stream: RngStream, w: WorkCounter) -> void

# src/sim/graph/propagate/phase_exchange.gd   (one file per phase; every phase has this shape; the body is an illustrative excerpt, helper names are not final)
class_name PhaseExchange extends RefCounted
## @hot  Phase 6 (JAC): D.1 flux over stored OPENINGs, virtual holes and seam weeps; deltas into x.acc_n_*, applied in one loop at the end.
## Writes: n_o2, n_inert (JAC), vented_mgu (JAC). Reads start-of-phase n_o2, n_inert, V, area, deg.
static func run(s: GraphState, c: GraphCaches, x: GraphScratch, _d: GraphDiag, t: GraphTuning, _scope: WriteScope, w: WorkCounter) -> void:
	# E3: alias every column read in the loop in a local first (§ 2). A local costs about 13 ns per element, an object member about 64 ns (§ 13.1).
	var links: PackedInt32Array = c.exchange_links               # derived list: stored OPENINGs, hull holes, seam weeps
	var n_links: int = c.exchange_link_count
	var comp_a: PackedInt32Array = c.edge_a_comp_row
	var comp_b: PackedInt32Array = c.edge_b_comp_row
	var area_m: PackedInt32Array = c.edge_area_m
	var vol: PackedInt32Array = c.comp_v
	var n_o2: PackedInt64Array = s.n_o2
	var n_inert: PackedInt64Array = s.n_inert
	for k: int in n_links:
		var e: int = links[k]
		var ra: int = comp_a[e]
		var rb: int = comp_b[e]
		var g: int = (n_o2[ra] + n_inert[ra]) * vol[rb] - (n_o2[rb] + n_inert[rb]) * vol[ra]   # exact integer gradient of start-of-phase amounts; total amount read inline (CT-06)
		if g == 0:
			continue
		var dn: int = GraphRules.gas_flux(absi(g), vol[ra], vol[rb], area_m[e], x.deg_max_of(e), t.g_open, t.gas_scale)
		# direction from the sign of g; species split Δn_o2 = floor_div(dn * n_o2_src, n_tot_src); remainder inert; accumulate, never write n_* here
		x.add_flux(ra, rb, g > 0, dn, s)
	w.add(GraphUnits.EDGE, n_links)
	x.apply_gas(s, c)                                              # col[i] += acc[i]; acc[i] = 0, ascending

# src/sim/graph/graph_derived.gd
static func ensure_derived(s: GraphState, c: GraphCaches, x: GraphScratch, rooms: RoomSource, catalog: ItemCatalog) -> void   # idempotent; returns at once while not c.topo_dirty
static func derive_t_mach(s: GraphState, c: GraphCaches, t: GraphTuning, machine_slot: int) -> int                              # D.8 proxy from persisted state at the point of use

# src/sim/graph/graph_mutators.gd   (M19 shape; Class A with an out-less status)
func set_section_dormant(scope: WriteScope, section_id: int, dormant: bool) -> int:
	if not scope.open_for(GraphModule.BLOCK_ID):
		return MutatorStatus.REFUSED
	var step_no: int = IntMath.floor_div(_clock.tick(), _tuning.graph_step_ticks)
	if dormant:
		_set_dormant(section_id, step_no)
	else:
		var status: int = _rebase_stamps_and_wake(scope, section_id, step_no)    # GraphFault.raise_in(STEP_RANGE) on a negative delta or an overflowing stamp
		if status != MutatorStatus.OK:
			return status
	GraphDerived.ensure_derived(_state, _caches, _scratch, _rooms, _catalog)
	scope.mark_dirty(GraphModule.BLOCK_ID, DirtyKey.make(GraphKeys.TOPO, 0))
	scope.mark_dirty(GraphModule.BLOCK_ID, DirtyKey.make(GraphKeys.SECTION, section_id))
	return MutatorStatus.OK

# src/shared/graph/graph_rules.gd  (pure; importable by src/data and tools): gas_flux, gate_floor, apply_loss, rng_subject, rng_counter,
#   burst_hit, onset_valid(mode_since, mode_key) -> bool, alarm_capable(mode) -> bool  (bodies in § 3, § 7, § 8.2)

# registration (GraphModule.setup, BOOTING):  hooks and the stream
#   registrar.register_hook(HookDescriptor{hook_id &"graph.propagate.power",   owner_name &"graph", divisor tuning.graph_step_ticks, offset tuning.hook_offset_power,   after [],        callable hook_power})
#   registrar.register_hook(HookDescriptor{hook_id &"graph.propagate.atmo",    ..., offset tuning.hook_offset_atmo,    after [{&"id": &"graph.propagate.power", &"tolerates_stale": true}],  callable hook_atmo})
#   registrar.register_hook(HookDescriptor{hook_id &"graph.propagate.failure", ..., offset tuning.hook_offset_failure, after [{&"id": &"graph.propagate.atmo",  &"tolerates_stale": true}],  callable hook_failure})
#   _stream_fail = facet.rng.register_stream(&"graph.fail", false)
```

### Implementation Guidelines

- **Must** register exactly three hooks, consecutively and power first, with divisor `GraphTuning.graph_step_ticks`, offsets from `GraphTuning` (non-decreasing, each at most `graph_step_ticks - 2`) and both `after` edges declared `tolerates_stale: true`; **must never** register a fourth Graph hook, a command, or a hook with a `before` expectation.
- **Must** compute `step_no` as `IntMath.floor_div(clock.tick(), graph_step_ticks)` in every hook and write the hook's cursor (and `last_step_no` from the failure hook) as the last statement of the run; **must never** catch up a missed step, run a step twice, or read a wall clock.
- **Must** run the twelve phases in the fixed order of § 2 as `## @hot` static functions in `src/sim/graph/propagate/`, each declaring the write discipline (JAC, ALLOC or OWN) of every column it writes in its doc comment; **must never** write a column a phase has not declared, iterate a loop to a convergence condition, recurse, or read a column another element of the same phase has written unless through ALLOC's running balance.
- **Must** give every flux and heat input a JAC accumulator in `GraphScratch` and apply it in one ascending loop that re-zeroes it; **must never** leave a non-zero accumulator at hook exit.
- **Must** iterate only the derived active lists (ascending slot) in phases; **must never** iterate a table directly, iterate a Dictionary, or visit a tombstoned or dormant row.
- **Must** drain the three queues only in phase 0, in the order of § 5, and persist them; **must never** apply M14, M15 or M17 synchronously, or give a caller a synchronous grant.
- **Must** apply the fairness gate as `gate_floor` / `apply_loss` at the moment of damage, through the single function; **must never** rewrite `hp` after the fact, apply a gate to an EXOGENOUS cause, or let `rate_pm` or `stress_pm` reach gas exchange, heat conduction or supply (they appear only in `phase_damage.gd`).
- **Must** accept M14 and M19 under INIT (M14 drains at the first power hook after LOADING to RUNNING; M19 stamps `dormant_since` with the INIT `step_no`);
- **Must** draw only `graph.fail` through `RngFacet.draw(stream, counter)` (the facet and `_stream_fail` handed in by `hook_failure`) with `rng_counter(step_no, kind, id)` of the 26-bit subject key; **must never** use `draw_epoch`, read `restore_epoch`, use a float or a bare `%`, store a draw counter, or use a node id and an edge id as the same subject key.
- **Must** call `ensure_derived` at the end of every structural mutator, at the end of phase 9 after a connectivity-changing transition, and in `post_restore`, keep the hook head guard as defect-only, and count `late_rebuilds`; **must never** cache `degrees` or `deg_max`, or cache a hook product that is not persisted or recomputed at its point of use (§ 11).
- **Must** write `n_served`, `n_draw`, a CABLE's `e0` load and `power_use_w5` in phase 2 only (canonical zero elsewhere); **must never** read `served_pm` from a cache, or seed the power-use window flat in `post_restore`.
- **Must** end phase 0 by recomputing `P`, `p_o2`, `o2_pm` and the trend of every compartment it touched, so the intensive caches equal a pure function of persisted state at every tick boundary; **must never** leave them to lag a draw or a BURN until phase 7.
- **Must** alias every column a phase reads or writes in a local before the loop, read the total gas amount inline as `n_o2[r] + n_inert[r]`, and take no accessor or helper call per element unless S1 shows it is free; **must never** add a total-amount accessor to `GraphState`.
- **Must** pack the draw sort key as `(cid << 32) | (crew << 16) | arrival_index` and the trace key as `(subject_ref << 32) | (kind << 16) | arrival_index`, assert every queue bound at most 65,535 at boot, and sort with a bounded `for` and `break`; **must never** use `while` in `propagate/`.
- **Must** re-base every absolute step stamp of § 10 in `set_section_dormant(false)`; **must never** advance `stress_until`, fast-forward a section, or write a row of a dormant section from a phase or a Class B call.
- **Must** tag every function of § 13.2 `## @hot`, keep them free of the `HOT-01` tokens, size scratch and caches only in structural mutators and `post_restore`, and count work through the injected `WorkCounter`; **must never** claim "zero allocation" without `HOT-01`, `CNT-01` and `CNT-02`.
- **Must** report a structural defect through `GraphFault.raise_in` and stop the hook at the next `scope.faulted` check; **must never** use `assert`, truncate or skip a phase on a budget overrun, or clamp silently (a numeric clamp increments `saturations`).

## Alternatives Considered

### Alternative 1: One hook at divisor 6, offset 0, running all twelve phases
- **Description**: A single `graph.propagate` hook; the baseline of the GDD before the 2026-10-27 review.
- **Pros**: One registration; no cross-hook products; restore lands only between steps, so no hook-product question arises; no `tolerates_stale` edges.
- **Cons**: The whole step (about 0.8 ms for the home ship alone at the review's coefficients, about 1.7 ms at the weld) lands on one tick in six, against a 0.5 ms spike share of the kernel's 2 ms target that every other divisor-6 hook also draws on.
- **Rejection Reason**: It cannot meet `GRAPH_HOOK_BUDGET_MS`; the stagger is the baseline. It stays reachable as the data-only revert (all offsets 0), which this design supports without any rule change.

### Alternative 2: `next` columns copied and swapped by reference for the Jacobi phases (ADR-0011 § 6.2 (4), LS-1 (j))
- **Description**: A second column per double-buffered field; read the live one, write the `next`, swap at the end.
- **Pros**: The words of the GDD; no accumulator zeroing; dense rewrites are cheaper per element (probe (e): 69 against 134 microseconds per 4,096 elements).
- **Cons**: Every untouched row must be copied before the pass; a pairwise flux needs a read-modify-write on `next` anyway; conservation is invisible (no single place sums to zero); the persisted column is replaced by reference, so a column aliased across a hook boundary or captured by a snapshot is stranded; it does not skip quiescent rows.
- **Rejection Reason**: The Graph's Jacobi columns are the compartment rows (14 at MVP), where the dense-rewrite advantage is immaterial, and the accumulator form gives conservation tests a single place to look. The swap remains available for any future wholesale-rewritten field.

### Alternative 3: Iterate to convergence, or catch up missed steps
- **Description**: Relax the exchange until the gradient is below a tolerance, or replay skipped steps after a stall.
- **Pros**: Closer to the exact equilibrium; resilient to lag.
- **Cons**: Unbounded work per run; a different answer when the frame rate changes; the "dread clock" of D.15 is tuned to a single pass per step (the stall residual `ceil(1/k)` is intended behaviour).
- **Rejection Reason**: TR-lsg-007 and D.16: one pass, bounded by construction, no catch-up. A stalled frame delays ticks but never skips or repeats a step.

### Alternative 4: Re-derive every hook product at restore and accept the divergence (LS-21 as written)
- **Description**: Keep `served_pm` and the cable loads derived; `post_restore` recomputes them from the restored state.
- **Pros**: No schema change; smaller snapshot; the GDD's stated rule.
- **Cons**: Not exact: the battery availability, the thermostat duty, a cable failed in phase 9 and an M10 throttle all move between hooks, so a restore at 6n+1 to 6n+5 can feed the failure hook a different load than the uninterrupted run and diverge in `hp` by one step's damage, rarely and at the worst moments (a battery emptying, a cable failing).
- **Rejection Reason**: Replay equality is a project principle, and a rare divergence is the hardest kind to debug. The chosen cost is 840 bytes and two columns, plus 40 bytes for the five-step `power_use` window. ADR-0011 adopted the products by ruling CT-04, so there is no fallback and no accepted-divergence list.

### Alternative 5: Make every cache live-derived, or persist the whole step's products
- **Description**: Recompute the descriptor store and loop summaries after every hook and every mutator so they always equal a function of current state; or persist them.
- **Pros**: Exact at every tick for every reader.
- **Cons**: Three O(subjects) rebuilds per step plus one per M13 resubmit (10 per tick in the Interaction worst case); persisting the store adds kilobytes and a second copy that can disagree with the state (ADR-0011 Alternative 7's objection).
- **Rejection Reason**: The cost is out of proportion to a one-step window whose only sim reader is the fault glow. Bounded and measured instead (RE-2).

### Alternative 6: Lazy topology refresh only at the hook head
- **Description**: Mutators set `topo_dirty`; the next hook rebuilds (the GDD's literal text).
- **Pros**: Mutators stay cheap; one rebuild per burst of changes.
- **Cons**: A weld preview, `can_join` and every read between a mutator and the next hook would see stale caches or pay a rebuild at an arbitrary presentation frame; ADR-0021's transaction cannot verify the post-state.
- **Rejection Reason**: Eager in the mutator (the GDD's LS-19 amendment), with the hook head kept as a guard and a defect counter, gives both: complete caches on return and a safety net.

### Alternative 7: The fairness gate as a post-pass that corrects `hp`
- **Description**: Let damage land, then a pass raises any subject that crossed to FAILED without 100 steps of warning.
- **Pros**: Phase 8 stays simple.
- **Cons**: It heals (the GDD's first draft healed a plate ruptured to 3600 up to 4000), it needs the pre-damage value, and it cannot tell a gated loss from an exogenous one after the fact.
- **Rejection Reason**: D.9 states the floor limits the reduction and never raises; one pure `gate_floor` at the point of damage serves phase 8, M18 and the tests.

### Alternative 8: A sequential persisted RNG counter, or the un-namespaced `subject_id`
- **Description**: Draw from a counter stored in the block; or key by node id and edge id in one field.
- **Pros**: The counter is simple; the un-namespaced key is the GDD's text.
- **Cons**: A weld adds candidates and shifts the draws of unrelated subjects (TD R10); node 5 and edge 5 draw the same value (ADR-0011 Risk 7).
- **Rejection Reason**: A pure function of `(world_seed, step_no, kind, id)` is order-independent, replays after any restore and needs no persisted counter; the 2-bit kind field costs nothing.

### Alternative 9: Fast-forward a dormant section on load
- **Description**: Run the elapsed steps for the section when it wakes.
- **Pros**: A derelict ages while away.
- **Cons**: Unbounded work at the moment of arrival; contradicts the boarding rhythm ("a derelict that does not degrade on its own", R13) and the Web Map requirement that reload changes nothing.
- **Rejection Reason**: TR-lsg-029 and TR-webmap-041: frozen, re-based, no fast-forward.

### Alternative 10: Apply M14, M15 and M17 synchronously
- **Description**: Write the load request, the breath and the trace into the state at the call.
- **Pros**: No queue, no persisted queue columns, immediate grant for Suit.
- **Cons**: A caller's write order inside a tick becomes a determinism input; the cap of `MAX_FOOTPRINTS_PER_STEP` has no meaning; the grant is order-dependent between two crew; a mid-hook write races a phase that is reading.
- **Rejection Reason**: The queue makes a step a function of the persisted state at its start, which is also what makes a snapshot on any tick replayable.

### Alternative 11: One hook per phase (twelve hooks)
- **Description**: Register each phase as its own hook for finer scheduling.
- **Pros**: Finer budget attribution; arbitrary stagger.
- **Cons**: Twelve descriptors, eleven `tolerates_stale` edges, twelve cursors and eleven cross-hook product boundaries; far more snapshot windows; kernel schedule cost.
- **Rejection Reason**: Three hooks already give the stagger the budget needs; per-phase attribution is a benchmark-harness concern (§ 13.1), not a registration.

## Consequences

### Positive
- A step is a function of the persisted state at its start plus the commands of its ticks, so any-tick snapshots replay, and RE-1 gives the Graph's own state exact restore equivalence.
- The dread clocks of D.15 are properties of a bounded single pass; stability (the monotone gradient) is a test that runs on every commit, not an argument.
- Dormant sections cost nothing and are provably untouched; weld and cut effects are whole when the mutator returns.
- The fairness gate, the RNG key and the cause flag are one function or one rule each, used by the phase, the mutator and the test.
- Budgets are claims with a protocol: deterministic counts per commit (`CNT-04`), wall clock nightly on the target hardware, and a ladder that ends in "the number moves, not the design".

### Negative
- Two more persisted node columns, an `e0` overlay and the `power_use_w5` window (adopted by ADR-0011, CT-04) grow the F-MVP snapshot by about 840 bytes plus 40 bytes (headroom from about 7 to about 4.6 percent) and move `served_pm` out of the derived class.
- Phase 0 carries a small O(touched) cache close-out so that the intensive caches are exact at every tick boundary.
- The descriptor store, alarm list and loop summaries remain a one-step window after a restore; a sim reader (the fault glow) could see it.
- The eager rebuild puts an O(N + E) cost into every weld, cut, valve flip and dormancy toggle, and the active lists add about 5 nanoseconds per active element.
- Accumulators must be zeroed and checked at every hook exit; a forgotten re-zero is a determinism bug, caught only by the hook-exit scratch assertion.
- Phase 9 sees the room temperature of phase 7, one phase later than the GDD's phase-5 machine proxy, a small change in when OVERHEAT can start.
- A very large phase directory (twelve files) needs the OWN-01 writer rule to accept a directory.

## Risks
1. **The hook-product columns push the snapshot past the 32 KB ceiling** (ADR-0011 adopted them, CT-04; about 30.55 KB at the roster, 4.6 percent headroom). Mitigation: ADR-0011's lever 1 restores 1.8 KB; S3 measures before anything is committed.
2. **The spike share is missed on RM-DECK.** Mitigation: the ladder of § 13.1; the stagger is already the baseline; if the ADR-0010 rungs do not suffice the number moves (a TD decision), the design does not; the per-phase harness names the phase to attack first.
3. **`ClockView.tick()` inside P3 may name the next tick.** Mitigation: offsets at most `graph_step_ticks - 2` make `step_no` insensitive; KRN-LAT-02 confirms.
4. **The window after a restore could make fault glows flicker for up to five ticks.** Mitigation: RE-2 measures it; ADR-0017's hysteresis; the persisted-store alternative is rejected on size (Alternative 5).
5. **Inlining a non-negative `floor_div` in hot files needs an ADR-0010 amendment.** Mitigation: named as ADR-0010 rung 2, decided on S1's number; until then every division is `IntMath`.
6. **The one-hop cascade starves a high-id subject when more than `MAX_SPREAD_PER_STEP` candidates exist for many steps.** Mitigation: finite subject count; a fixture drives a section to full failure and records the front speed.
7. **`HookDescriptor` registration order is the tie-break among independent hooks**, so a change in module construction order could move a hook across the power hook and change a drain tick. Mitigation: the compiled order is printed by the Debug Overlay and a golden test asserts the relative order of the Graph's hooks and their six named neighbours.
8. **A dormant section's descriptors are invisible to `failures()`** (§ 10) though ADR-0011's `FailureFilter.loaded_only` suggests otherwise. Mitigation: stated in Open Question 5.
9. **Phase 9 runs after phase 7 changed the room temperature**, so the OVERHEAT and HOT triggers see a slightly different `T_mach` than the GDD's examples. Mitigation: the D.15 PUMP-STALL fixture (35 s) is re-run against the harness within its ±15 % band.
10. **The `pending_draw` insertion sort is quadratic and was measured only to 64 entries** (55.3 microseconds reverse-sorted). The structural bound is `MAX_CREW` x compartment count, about 480 in VS-A, and larger if a caller enqueues once per tick; the 16-bit arrival field keeps the order correct but not the cost. Mitigation: at MVP the bound is 56; S1 re-measures at the VS-A bound, and the lever is a counting pass by `cid` (queue entries are few per compartment) before the sort.
11. **The Graph is the worst-aligned Core entry on its hook tick** (CT-20: 1.66 ms of the 2.0 ms P3 target before the Suit, Interaction, Assembly, Web, Attention, Horror and Sanity hooks). Mitigation: the ADR-0003 P3 share ledger and S1's per-hook table; the number moves, not the design.

## GDD Requirements Addressed

TR ids and texts are from `docs/architecture/tr-registry.yaml`; the Planned ADR column of `requirements-traceability.md` (including the rows re-tagged 2026-10-29) is the scope. 28 planned rows, all addressed (18 full, 10 partial); TR-lsg-020 is a residual this ADR adds as partial; 13 further rows are owned elsewhere and supplied here.

### Decided here (full)

| GDD System | Requirement | How This ADR Addresses It |
|------------|-------------|--------------------------|
| life-support-graph.md | TR-lsg-007: iteration by ascending slot over dense append-only arrays, no Dictionary on the sim path, one Jacobi pass from a double-buffered start-of-phase snapshot per loop, no iterative solver, no catch-up | § 3 (JAC / ALLOC / OWN, accumulators as the start-of-phase snapshot, no convergence loop, no recursion, `PROP-01`), § 4 and § 10 (ascending active lists), § 1 (no catch-up) |
| life-support-graph.md | TR-lsg-008: three step hooks at divisor 6 with offsets 0 / 2 / 4 chained by `after:` with both edges `tolerates_stale: true`; one run = one step = 6 ticks, `step_no = floor(T/6)`, per-hook cursors persisted | § 1 (table, offsets as data with a boot relation, cursor semantics, guard, `HOOK_CURSOR`, `last_step_no` as the failure hook's) |
| life-support-graph.md | TR-lsg-009: twelve fixed phases in order (power 0-3, atmo 4-7, failure 8-11); cascades queue in phase 10 and apply in phase 0 of the next run, bounded by `MAX_SPREAD_PER_STEP` | § 2 (phase table), § 8.3 (own-row versus cross-subject, spread budget, one-hop latency, re-proposal instead of a backlog) |
| life-support-graph.md | TR-lsg-010: topology refresh is one idempotent function at the head of the first Graph hook after a structural mutator, zero cost while `topo_dirty` is clear; derived caches rebuild eagerly (`ensure_derived`) in structural mutators and `post_restore` | § 4 (eager in the mutator, head guard in all three hooks, `late_rebuilds`, phase-driven versus mutator-driven dirtiness, the connectivity predicate) |
| life-support-graph.md | TR-lsg-011: randomness is `rng_derive(world_seed, "graph.fail", (step_no<<20) \| subject_id)`, pure of persisted state, no restore-epoch input, `subject_id < 2^20` asserted | § 8.2 (plain stream `graph.fail`, `rng_counter` with a 2-bit kind and a 24-bit id sized to `GraphLimits.ID_MAX` so node and edge draws differ for any id the allocator can produce, probe (a); no epoch; no stored counter) |
| life-support-graph.md | TR-lsg-012: the thaw-burst roll window is persisted (`thaw_until_step`, `THAW_ROLL_STEPS`); a burst sets `hp = min(hp, BURST_HP_TO)` and closes the window; a restore neither re-rolls nor forgets it | § 8.2 (open in phase 9, roll in phase 8, exactly `THAW_ROLL_STEPS` rolls, close on hit or expiry, re-based by M19) |
| life-support-graph.md | TR-lsg-013: an endogenous transition to FAILED/DESTROYED or hull/seam LEAK to BREACH requires `W_MIN_STEPS` consecutive steps of a sev >= 1 descriptor, otherwise `hp` is floored at `HP_FAIL + 1` (`BREACH_HP` for breach); exogenous causes bypass | § 7 (`gate_floor` / `apply_loss` at the moment of damage, evaluation order, held damage discarded, the exogenous list and the cause flag rule) |
| life-support-graph.md | TR-lsg-015: `alarm_eligible` only for alarm-capable modes (1-5, 7-9, 20-23; never 6, 10, 24-27) at sev >= `ALARM_SEV_MIN` with an open gate; pure `alarm_sections()` ascending and duplicate-free | § 7 (`ALARM_CAPABLE_MASK`, phase 11 evaluation, the seen-byte construction of the ascending list, caller-buffer form from ADR-0011 § 9.1) |
| life-support-graph.md | TR-lsg-016: 19 mutators M1-M19 callable only inside a `WriteScope` in P2/P3 with the scope first; Class A has a pure `check_*` and an infallible apply, Class B (M14-M18) clamps, never rejects, no-ops on stale ids; the Graph registers no commands | § 6 (the table: scope phases, timing, writes, keys; the whole-table rules; ADR-0021 closes the transaction) |
| life-support-graph.md | TR-lsg-017: M14, M15 and M17 queue into persisted bounded `pending_load` (<= 64), `pending_draw` and `pending_trace` (<= 16 per run across callers), drained in phase 0 of the next run; `draw_grant(crew)` persists | § 5 (enqueue rules, coalescing, bounds, overflow counters, drain algorithms, the grant lifecycle) |
| life-support-graph.md | TR-lsg-026: work per run is bounded by construction: one pass per loop, `MAX_SPREAD_PER_STEP`, `MAX_FOOTPRINTS_PER_STEP`, capacity `MAX_GRAPH_NODES` / `MAX_GRAPH_EDGES`, `MAX_SEAMS_PER_SECTION`; dormant sections cost zero | § 13.3 (the bound of every loop, `CNT-04` closed forms per commit), § 10 (dormant sections absent from the active lists), § 9 (`can_join` capacity and seam limits) |
| life-support-graph.md | TR-lsg-032: equalisation flux uses the exact integer gradient of start-of-phase amounts, never reverses, keeps the maximum gradient non-increasing, bounded by the D.12 sum which `can_join` enforces with `STABILITY` | § 3 (`gas_flux`, the stability argument, the sealed-stability test as a gate), § 9 (`STABILITY` in `can_join`) |
| life-support-graph.md | TR-lsg-035: M17/M14/M15 deposits drain in phase 0 of the same tick when made by a P2 command or a hook ordered before the power hook, and at the next hook tick when made by a hook ordered after it, in arrival order within the cap | § 5 (the drain rule in terms of the kernel's compiled order, the caller table including the INIT row, the clarification that "same tick" means a power-hook tick, withdrawal of the `before:` wording) |
| life-support-graph.md | TR-lsg-036: snapshot of the MVP ship at most 0.5 ms and 32 KB, restore plus `post_restore` at most 2 ms (kernel integration world 3 ms and 10 ms), canonical bytes byte-identical after restore, on the release export (re-tagged 2026-10-29 from ADR-0006) | § 13.5 (budgets, what is inside and outside the 0.5 ms, the working split, S3, the size after the hook-product columns, the levers), § 11 (the size figure) |
| failure-and-trail-system.md | TR-ftrail-038: the Graph preserves a seeded onset triple (`mode_since_step` with matching `mode_key`) through phase 9, advances both onsets across a dormant interval, restarts them on re-weld; the fixture loader rejects a `mode_since_step` seeded without `mode_key` | § 8.4 (restamp only on a key change against the persisted key; `ONSET_UNKEYED` and `onset_valid`; a new SeamRecord on re-weld), § 10 (M19 re-basing of both onsets) |
| derelict-authoring-handmade-kits.md | TR-derelict-011: endogenous `hp` loss on a loaded DERELICT section runs at `rate_pm` 2000 (`DERELICT_FAULT_COMPRESSION`); dead faults emit `intensity_pm` 0 until power or coolant reaches them | § 8.1 (`rate_pm` set by M1 from the constant, read only in `phase_damage.gd`, never touching gas, heat or supply; dead faults need no special case) |
| derelict-authoring-handmade-kits.md | TR-derelict-012: exactly eight live faults (4 home, 2 A, 2 B), each within +-15 % of Graph D.15 at trigger, `MAX_FAULT_GLOW` respected | Validation Criteria: the eight D.15 fixtures (O2-SHIP, SEAM-WEEP, PWR-OVL, BROKEN-BREAKER, CLT-LEAK, PUMP-STALL, FUEL-DRIP, HULL-FATIGUE) run against the stepped Graph with the +-15 % assertion on every commit that touches `propagate/`; the glow cap is the Emitter's, asserted by the fixture over `failures()` |
| derelict-authoring-handmade-kits.md | TR-derelict-037: authored closed-valve staging relies on the Graph gating leak wear on `intensity_pm` > 0 (`hp` held at 0 intensity), so a change to that rule must fail the dependent tests | § 8.1 (wear gated on the persisted `isolated_at` stamp, the same fact in a form phase 8 can read); the Derelict closed-valve fixtures and the `pb_leak` seeded-stamp fixture (§ 8.4 clear rule) are the dependent tests |

### Decided here (partial: this ADR decides its half; the other owner is named)

| GDD System | Requirement | How This ADR Addresses It / Other Owner |
|------------|-------------|--------------------------|
| life-support-graph.md | TR-lsg-025: hook spike share P99 <= 0.5 ms and max <= 1.0 ms summed over the hooks of one hook tick on RM-DESKTOP and RM-DECK, zero heap allocations per phase in steady state, presentation read side <= 0.1 ms P99 per frame | § 13.1 (protocol, ladder), § 13.4 (read side), § 13.2 (the four-check restatement). The allocation mechanism (`HOT-01`, `CNT-01` to `CNT-04`, the `## @hot` tag) is ADR-0010 § 7; this ADR owns the tag list and the budgets |
| life-support-graph.md | TR-lsg-020: int dirty keys, bucket quanta, one `C/<cid>` per compartment per step, worst designed cascade 38 keys, `MAX_DIRTY_KEYS` not below 128 | § 12 (which phase fires which key on which tick, last-emitted update rule, the 38-key bound across hooks). Encoding and the 128 floor: ADR-0004; kinds, packing and quanta: ADR-0011 |
| life-support-graph.md | TR-lsg-027: welds only append edges; `can_join` (pure, closed reason set) validates; `weld_join` appends pre-validated sub-edges all disconnected and cannot fail; ids append-only | § 6 (M2), § 9 (disconnected start, weep, `STABILITY`, `CAPACITY`, re-resolve). The data side is ADR-0011 § 4; the transaction around it is ADR-0021 |
| life-support-graph.md | TR-lsg-028: `cut_detach` atomic in one P2 apply along seams (SEALED caps ports, RAGGED adds `A_CUT` hull-stub holes and `open_end` ports); `discard_section` tombstones and moves mass to `discarded_*` | § 6 (M3, M4), § 9 (stubs as ordinary rows treated as holes by phase 6, conservation). Rows and counters: ADR-0011; transaction order: ADR-0021 |
| modular-ship-assembly.md | TR-assembly-034: a weld or cut applied in P2 becomes visible to the Graph hooks only at the next hook after its topology refresh, with no observable half-state | § 4 (eager rebuild before the mutator returns), § 9 (visibility). Assembly's own transaction and the two-phase begin/complete: ADR-0021 |
| boarding-and-salvage.md | TR-boarding-039: a cut on a running machine is EXOGENOUS: the island re-solves at the next run, dependants go STARVE with no tell and no minimum-warning wait, the SEVERED descriptors open no Attention breach-window row | § 7 (M3 / M6 set the EXOGENOUS flag; STARVE is a status), § 4 (re-solve at the next power hook). Attention's breach-window rule: ADR-0022 |
| boarding-and-salvage.md | TR-boarding-041: cutting a store routes its stock to `discarded_*`; an uncapped stub leaks from the home pool only while the seam valve is open, otherwise from the section's own stock | § 4 (the connectivity predicate: a closed VALVE splits the component, so the stub's pool is its own), § 6 (M6). The `disc_*` routing and the retained port `cap`: ADR-0011 § 5 and § 3.3 |
| life-support-graph.md | TR-lsg-029: dormant sections skipped by every phase, persisted and hashed unchanged, cannot be welded; M19 on load advances all absolute step stamps by the dormant duration; no fast-forward | § 10 (active lists, the complete stamp list including `thaw_until` and `energised_at`, `stress_until` excluded, no fast-forward). The Web Map call site and the load/unload decision: ADR-0026 |
| web-map-and-transit.md | TR-webmap-041: a dormant section is frozen with no fast-forward on reload (partition hash, node `hp`, taken salvage and horror records unchanged); lamp drain and derelict fault compression accrue on loaded ticks only | § 10 (rows never written while dormant; loaded-steps-only accrual; the reload property test). Salvage and horror records, lamp top-up: ADR-0026, ADR-0027, ADR-0023 |
| suit-and-equipment.md | TR-suit-039: breath-source, rung and cap sequences are identical whether the Graph returns the `draw_air` grant synchronously or at the next run's phase 0; `pending_draw_mgu` is persisted | § 5 (queued only, persisted, restore-exact, (compartment, crew) order, grant lifecycle). Suit's sequence equivalence test: ADR-0027 |
| interaction-system.md | TR-interact-038: the Interaction hook meets a release-export budget under a worst-case load (4 seats, full ledgers, 16 throws, 12 attempts, 10 SYSTEM door resubmits per tick): mean <= 0.10 ms, p99 <= 0.50 ms, max below `STEP_BUDGET_MS`, zero probe use | § 6 and § 13.6 (every Graph call it makes is bounded, allocation-free and `## @hot`; the harness records the Graph's share). The Interaction hook's budget and `lit()`: ADR-0017 and ADR-0020 |

### Owned elsewhere (loaded or checked here, decided by another ADR)

| GDD System | Requirement | Where decided / what this ADR supplies |
|------------|-------------|----------------------------------------|
| life-support-graph.md | TR-lsg-014: non-player exogenous damage sources (M17 RUPTURE/GOUGE/STRESS, M18 EXOGENOUS) are preceded by their own 10-30 s tell stated in the caller's GDD | ADR-0022 (Attention) and ADR-0023 (Horror). Supplies: the EXOGENOUS bypass and the rule that the Graph neither emits nor checks the tell (§ 7) |
| life-support-graph.md | TR-lsg-018: `read_view` is pure and true-only; `state_hash` identical with and without Perception content | ADR-0011 § 9.1 and § 6.5. Supplies: phases write nothing a view reads impurely; the descriptor store is rebuilt only in phase 11 and `post_restore` |
| life-support-graph.md | TR-lsg-019: hot-path reads use typed getters or a reusable view object; `failures(filter)` slices a preallocated columnar store built once in phase 11 | ADR-0011 § 9. Supplies: the phase-11 build (§ 2), the loaded-sections-only content (§ 10), the 0.1 ms read-side protocol (§ 13.4) |
| life-support-graph.md | TR-lsg-021: `snapshot()` is a Dictionary of named packed arrays encoded by the kernel | ADR-0005 § 3.1, ADR-0011 § 3.10. Supplies: nothing beyond the hook-product columns and the `power_use_w5` window in the payload (§ 11) |
| life-support-graph.md | TR-lsg-022: payload stored with `var_to_bytes`, JSON only in the header and dump | ADR-0006 § 1.2, § 1.3. Supplies: nothing |
| life-support-graph.md | TR-lsg-023: two-phase restore, version 1 only, `ctx.fault(GRAPH_RESTORE_*)`, `post_restore` rebuilds caches and seeds buckets | ADR-0006 § 2.2, ADR-0011 § 7, ADR-0016. Supplies: the phase code that reads the seeds (§ 12), the cursor semantics that ADR-0011 § 7.2 step 13 checks and the first-run schedule rule (§ 1), the stock reconcile staying in phase 0 |
| life-support-graph.md | TR-lsg-024: structural violations raise `GraphFault.raise` (never `assert`); numeric violations clamp and count in `saturations` | ADR-0011 § 8. Supplies: the stop-at-`scope.faulted` rule in the hooks and the new `HOOK_CURSOR` code (§ 1) |
| life-support-graph.md | TR-lsg-030: type ids explicit, stable, unique-checked at boot, append-only | ADR-0007 and ADR-0011 § 4.7. Supplies: `ALARM_CAPABLE_MASK` and the RNG kind constants as code-side constants of `GraphIds` / `GraphRules` |
| life-support-graph.md | TR-lsg-031: the `graph.state` hash and `rng_derive` draws for fixed scripts bit-identical on three OSes | ADR-0005. Supplies: the scripts (HEALTHY, weld and cut, thaw rolls) the goldens run, and the namespaced counter vectors (§ 8.2) |
| life-support-graph.md | TR-lsg-033: boot refuses invalid content or tuning with named `GRAPH_BOOT_*` reasons | ADR-0007 and ADR-0011 § 8.4. Supplies: the hook-offset relation, the id-field relation (`ID_MAX <= 2^24`) and the queue-bound relation (at most 65,535) added to `GRAPH_BOOT_TUNING` (§ 1, § 5, § 8.2) |
| life-support-graph.md | TR-lsg-034: every structural mutator emits exactly one P5 batch of its declared keys; a fault emits none | ADR-0003 and ADR-0021. Supplies: the declared key set per mutator (§ 6) |
| life-support-graph.md | TR-lsg-038: persisted-field set verified by reflection and zero-and-diff | ADR-0011 § 6.3. Supplies: the member list for `derived_members.json` that this ADR adds (§ 4, § 3) and the exclusion of the persisted hook products |
| life-support-graph.md | TR-lsg-039: diagnostics counters never in the canonical layout or any hash | ADR-0005 § 3.6, ADR-0011 § 1. Supplies: the `GraphDiag` additions (§ 13) |

## Performance Implications
- **CPU (all UNVERIFIED on a release export; S1, S0-f, S3 pending)**: the GDD's review coefficients (5 / 10 / 15 microseconds per node / edge / compartment) give about 825 microseconds for the home ship alone and about 1.7 ms at the weld set per step; at the 0 / 2 / 4 stagger that is roughly 0.3 ms and 0.6 ms per hook, so the weld set is **at or just over the 0.5 ms share** on those coefficients, which is why S1 gates acceptance and the ladder of § 13.1 exists. The active-list indirection adds about 5 nanoseconds per element (probe (d)), the queue sort is about 33 microseconds typical and at most about 55 microseconds (reverse-sorted) at the 64-entry MVP bound (probe (c), `for` plus `break` form), and a `clampi` or `mini` adds about 4 nanoseconds per call (probe (f)); `IntMath.floor_div` is about 12 times a bare operator (ADR-0005 P14). **These per-element figures are local-array costs:** member access measured about 64 nanoseconds per element and a method or static call about 120 nanoseconds on the editor binary, which is why every phase aliases its columns in locals (§ 2, E3); S1 decides. The even split of the step over three hooks above is an assumption that S1's per-hook table replaces (T4): the failure hook is plausibly the heaviest, and the pass/fail figure is the worst single hook. The first hook after a snapshot pays one copy-on-write per written column (about 70 columns, tens of microseconds). The eager rebuild adds O(N + E) to each structural mutator.
- **Memory**: the accumulators are a handful of per-compartment and per-section `PackedInt64Array`s, the active lists and the island, candidate and queue buffers are `int32` arrays of the node and edge row counts, and the descriptor store is ADR-0011's: tens of kilobytes at capacity, well under 1 MB at MVP; nothing is allocated in a phase.
- **Load Time**: `post_restore` is one `ensure_derived` plus the bucket seeds and the structural checks (working budget 1.1 ms of the 2 ms); M19 on wake is one O(rows) pass.
- **Network**: none at MVP; `SHARED_SUMMARY` (VS-B) reads the same columns.

## Migration Plan
There is no Graph code yet. Implementation order: (1) `src/shared/graph/graph_rules.gd` additions (`gas_flux`, `gate_floor`, `apply_loss`, `rng_subject`, `rng_counter`, `burst_hit`, `onset_valid`, `alarm_capable`) with their unit tests and golden counter vectors; (2) the S1 stub Graph: the three hooks and twelve empty phases over the real active lists and accumulators on a synthetic F-MVP-sized fixture, plus the benchmark harness and `WorkCounter` hookup, so the ladder decision is made before the phase bodies are written; (3) `ensure_derived` with the connectivity predicate, components and trees, active lists and the head guard, with the late-rebuild assertion; (4) the hook skeleton, cursors and `HOOK_CURSOR`; (5) the phases in order 0 to 11, each landing with its D.15 fixture and its write-set test; (6) the queue enqueue and drain, then the mutators; (7) dormancy and the M19 re-basing; (8) the RE-1 to RE-3 restore-equivalence tests. The Foundation Asks and the GDD syncs below land before the stories that depend on them. No existing code or save format is migrated.

## Validation Criteria
- **Registration and cursors:** three descriptors with divisor `GRAPH_STEP_TICKS`, offsets 0 / 2 / 4, both edges `tolerates_stale`, an empty boot-warning list, and a golden test of the relative order of the Graph hooks and their neighbours (Risk 7); `step_no` equal for the three hooks with `ClockView.tick()` at T and at T+1; a replayed hook, a regressed clock and the step-0 exception behave as § 1 says; a schedule change across a save is absorbed at the first hook run (`schedule_shift` counted, skip when `last_step_X` equals the step, run once otherwise), and a corrupt cursor is ADR-0011's `RESTORE_HOOK_CURSOR` fixture.
- **Signatures and typing:** every `run_*` signature and every hook callable matches § Key Interfaces (`run_failure` receives the `RngFacet` and the `RngStream`); the Key Interfaces snippets load clean under the ADR-0010 § 5.1 promoted-warning set once re-probed (typed `for` iterators, underscore-prefixed unused parameters).
- **INIT:** an M19 call under INIT (Web Map `init_world`) is accepted, stamps `dormant_since` with the INIT `step_no` and leaves one `ALL` batch; an M14 deposit under INIT drains at phase 0 of the first power-hook tick after LOADING to RUNNING and that tick equals ADR-0017 D5.2's `arm_tick`.
- **Phase order and write sets:** a trace test shows phases 0 to 11 in order each step; for every phase, a column-diff harness runs it alone over each fixture and asserts the set of changed persisted columns is within the phase's declared write set and the discipline (JAC / ALLOC / OWN) declared in its doc comment.
- **Scratch hygiene:** every accumulator is all-zero at every hook exit on every fixture; `late_rebuilds` is 0; `saturations` is all zeros.
- **One pass and stability:** the S1 sealed-stability fixture (a room of degree 4 or more at 1010 dkPa among rooms at 200, every source and sink off) shows a non-increasing maximum `|ΔP|` over all open edges and no direction flip, run to quiescence, on every commit touching `propagate/`; a fixture at the D.12 bound still passes; `can_join` rejects the plan one opening over it with `STABILITY`.
- **Conservation:** the four ADR-0011 § 6.4 totals equal their instantiation baselines after every step of every fixture, including a cut, a discard, a stock reconcile and a dormant section; the sum of each gas accumulator is zero apart from the vented term.
- **Restore equivalence:** RE-1, RE-2 and RE-3 of § 11 over the 12-tick window and the fixture set (HEALTHY, PWR-OVL, a thaw window, a pending draw, a dormant derelict, a restore at 6n+1 after a draw drain, a restore at 6n+1 after a BURN); RE-1 requires `graph.state` hash equality at **every tick from *t*+1 onward**; a battery-final-step fixture, a thermostat-flip fixture and a cable-failure-between-hooks fixture are ordinary RE-1 fixtures, and the `power_use_w5` window is restored, not seeded flat.
- **Queues:** a property test deposits M14, M15 and M17 from P2, from hooks and (M14) from INIT on random ticks and asserts the drain tick is the first power hook after the deposit in kernel order, the (compartment, crew) and ascending-subject orders hold **with the queue length above 255 entries** (the `(cid << 32) | (crew << 16) | arrival_index` key; a fixture with 300 entries must not carry the arrival field into the crew field), the caps drop and count, `draw_grant` equals the drain totals, and a snapshot between deposit and drain restores to the same drain.
- **Gate:** the D.11 worked examples reproduce exactly (hold on steps 70 to 100, failure on 101; compartment alarm at step 125; ρ 1400 not held, ρ 1419 held one step); hull and seam floors; exogenous bypass for each source in § 7; a cut on a running machine yields STARVE with no `hp` change and `cause` EXOGENOUS; `alarm_sections()` is ascending and duplicate-free over a generated multi-section fixture and empty before any gate opens.
- **Randomness and thaw:** `rng_counter` golden vectors (node 5 and edge 5 at one step differ, and node and edge ids near `GraphLimits.ID_MAX` stay separate); a lint finds no `draw_epoch`, `restore_epoch`, float or bare `%` in `src/sim/graph/`; a fixed-seed thaw fixture rolls exactly `THAW_ROLL_STEPS` times, closes on a hit, expires unspent, restores mid-window to the same outcome, and re-bases across a dormant interval.
- **Cascades and onsets:** a fixture with more than `MAX_SPREAD_PER_STEP` candidates admits exactly the cap per run in ascending order and drains in `ceil(n / cap)` runs; one hop per step; a seeded `(mode_since, mode_key, isolated_at)` triple survives run 1; a seeded `isolated_at` on a valveless subject holds leak wear until the first run its `intensity_pm` exceeds 0 and then reads 0 (§ 8.4); `ONSET_UNKEYED` rejects an unkeyed onset; a re-weld restarts the seam's onsets only.
- **Dormancy:** `WorkCounter` sees zero visits to a dormant section; a save and restore while dormant leaves the section's rows byte-identical; M19 wake advances exactly the stamps of § 10 (including `thaw_until` and `energised_at`), not `stress_until`; `STEP_RANGE` on a negative delta; a derelict battery and a derelict fault clock advance on loaded steps only.
- **D.15 fixtures:** the eight MVP faults (TR-derelict-012) and the DERELICT variants land within +-15 % of the closed-form estimates; PUMP-STALL still lands at about 35 s with `T_mach` evaluated in phase 9 (Risk 9); the closed-valve fixture holds `hp` and resumes on reopening (TR-derelict-037).
- **Allocation and work:** `HOT-01` clean on every tagged function; `CNT-01` unchanged and `CNT-02` within tolerance over 600 ticks; `CNT-04` counts equal the closed forms; the four ADR-0010 blind-spot fixtures run against a Graph phase.
- **Budgets on hardware:** S1 / S0-f on RM-DESKTOP and RM-DECK release exports (hook share as P99 and max **per hook id** plus the failure hook on a phase-9 `ensure_derived` tick, the pass/fail figure being the maximum over the three; per-phase table, member-versus-local access cost, the Graph's P2 share and hook-side share under the TR-interact-038 load reported separately, the sort cost at the VS-A queue bound, first hook after a snapshot, Debug/Release factor, read side, Interaction worst case); S3 (snapshot at most 0.5 ms, at most 32 KB, restore plus `post_restore` at most 2 ms, byte-identical round trip); the structural entry points `can_join`, `instantiate_section`, `cut_detach`, `weld_join` as regression guards against the recorded baseline.
- **Lints (once ADR-0010 accepts them):** `PROP-01`, `OWN-01` over the `propagate/` directory, `HOT-01` against `hot_functions/graph.json`, and the `rate_pm` / `stress_pm` reader restriction, each with bad and good fixtures (`FIX-01`).

## Related
- **Depends on (all Proposed)**: ADR-0011 (data model), ADR-0003 (hooks, scope, mutator status, fault domains), ADR-0004 (dirty keys), ADR-0005 (determinism, RNG, `IntMath`), ADR-0001 (tick phases), ADR-0006 (restore), ADR-0007 (tuning, boot tiers), ADR-0010 (hot tags, lints, ladder).
- **Siblings and consumers**: ADR-0013 (Layout: rooms through `RoomSource`, no phase writes `area`), ADR-0016 (restore ordering; the cursor check lives in ADR-0011 § 7.2 step 13), ADR-0017 (Emitter State: `emitter.place` after the failure hook, fault-glow window), ADR-0021 (weld, cut, install transaction), ADR-0022 (Attention), ADR-0026 (Web Map: M19), ADR-0027 (Suit: M15, M16), ADR-0028 (lamp brightness), ADR-0031 (Debug Overlay counters).
- **GDDs**: `design/gdd/life-support-graph.md` (LS-8 to LS-13, LS-15 to LS-18, LS-24 to LS-26, D.1, D.6, D.8 to D.12, D.15, D.16, Section E), `failure-and-trail-system.md` (FT-15, onsets), `derelict-authoring-handmade-kits.md`, `boarding-and-salvage.md`, `web-map-and-transit.md`, `suit-and-equipment.md`, `modular-ship-assembly.md`, `interaction-system.md`.
- **Registry**: `docs/registry/architecture.yaml` (`kernel_p3_step_hooks`, `native_code_policy`, `packed_array_exposed_or_setter_tracked`, `integer_division_convention`, `dirty_key_encoding`); `design/registry/entities.yaml` (`GRAPH_STEP_TICKS`, `MAX_SPREAD_PER_STEP`, `MAX_FOOTPRINTS_PER_STEP`, `GRAPH_HOOK_BUDGET_MS`, `THAW_ROLL_STEPS`, `BURST_HP_TO`, `BURST_P_PM`, `DERELICT_FAULT_COMPRESSION`, `W_MIN_WARNING_S`, cited by name).

## Open Questions
1. **(Closed, CT-04.)** ADR-0011 adopted the hook-product columns, the CABLE `e0` overlay and `power_use_w5`; no fallback exists. Remaining: the GDD owner applies the LS-21, LS-4 and TR-lsg-004 wording (`gdd_sync`).
2. **Does `ClockView.tick()` inside P3 name the executing tick or the next?** The offset rule makes the Graph insensitive; KRN-LAT-02 should still record it. Owner: ADR-0001 / ADR-0003.
3. **Is the Emitter State fault glow insensitive to the one-step window after a restore?** RE-2 measures it. Owner: ADR-0017.
4. **Phase 9 evaluates `T_mach` from the room temperature of phase 7**, one phase later than the GDD's phase-5 proxy; the PUMP-STALL 35 s fixture must still land inside +-15 %. Owner: GDD D.8 / D.15 on the harness result.
5. **A dormant section's descriptors are absent from `failures()`.** If the Chart Table, Failure & Trail or the Debug Overlay needs them, they read the cold-path views or the filter's `loaded_only` becomes meaningful with a separate dormant store. Owner: ADR-0026 / ADR-0030 / ADR-0031.
6. **The HEATER's `run_pm` is derived from `n_draw / req`** instead of a stored `duty_t`; the GDD's LS-19 wording changes (`gdd_sync`). Owner: GDD.
7. **Registry gaps**: `load_queue_max` (64, TR-lsg-017), `hook_offset_*`, and `STRESS_HOLD_STEPS` as `GraphTuning` fields need registry rows or the data file as their source. Owner: registry pass, ADR-0007.
8. **The golden-vector label `graph.failure` in ADR-0005 § 2.4 versus the stream name `graph.fail`** (§ 2.4 table and GDD). The vector stays valid as a pure-function vector for that string; the label should be reconciled. Owner: ADR-0005.
9. **Apportioning the 2 ms P3 target among the divisor-6 hooks** (GDD Open Question 31). This ADR takes the Graph's `GRAPH_HOOK_BUDGET_MS` as given; the apportionment is decided by the technical-director on S1's per-hook table (T4), in the ADR-0003 P3 share ledger (CT-20). Owner: technical-director.
10. **The `cause` clear rule** (§ 7: clears when the dominant mode returns to 0 or `hp` reaches `HP_OK`) needs the GDD to name `cause` as persisted and to state the rule. Owner: GDD.
11. **Open dependency (ruling R4): per-`(tick, hook)` elapsed samples for S1.** S1 computes the P99 over hook ticks and needs the elapsed time of each `(tick, hook)` pair without instrumenting sim code. That is an ask to ADR-0031 (diagnostics) and the S1 harness, not to ADR-0003 or ADR-0001 (ADR-0003 is unchanged). Until it lands the S1 benchmark harness story stays Blocked. Owner: ADR-0031 / S1 harness.

## Registry Candidates
Reviewable candidates for `docs/registry/architecture.yaml`; this ADR does not write the registry.

### state_ownership
```yaml
  - state: graph_hook_products
    status: proposed           # ADR-0012 is Proposed; ADR-0011 adopts the columns by ruling CT-04
    owner: life-support-graph
    adr: docs/architecture/adr-0012-graph-propagation-scheme.md
    interface: "n_served (permille) and n_draw (dJ per step) per power consumer node and the net load of a CABLE edge (edge column e0), plus the five-int header window power_use_w5, written only by propagate phase 2, canonical zero elsewhere; persisted so a restore on any tick equals the uninterrupted run"
    read_by: [emitter-state, suit, interaction, in-world-instruments, attention, failure-and-trail]
    added: 2026-10-29
    revised: ""
```

### interfaces
```yaml
  - contract: graph_propagation_hooks
    status: proposed
    pattern: kernel_hook_chain
    producer: life-support-graph
    consumers: [suit, attention, emitter-state, sanity]
    signal_signature: "three HookDescriptors graph.propagate.power / .atmo / .failure, divisor GRAPH_STEP_TICKS, offsets from GraphTuning (0, 2, 4), after chain with tolerates_stale true on both edges; step_no = floor_div(clock.tick(), graph_step_ticks); cursors last_step_power / _atmo / _failure / last_step_no (the failure hook's step; trails step_no by up to one step); stored schedule hdr.sched compared at the first run after a load"
    adr: docs/architecture/adr-0012-graph-propagation-scheme.md
    not: "a fourth Graph hook; a before: edge; catching up a missed step; a hook offset above graph_step_ticks - 2"
    added: 2026-10-29
  - contract: graph_input_queue_drain
    status: proposed
    pattern: queued_unconditional_channel
    producer: life-support-graph
    consumers: [emitter-state, interaction, suit, horror, attention]
    signal_signature: "M14 set_load_request, M15 draw_air, M17 deposit_trace enqueue into persisted bounded queues and drain only in phase 0 of the first graph.propagate.power execution that follows the deposit in kernel order (INIT, then tick, P2 before P3, schedule position); an INIT deposit drains at the first power-hook tick after LOADING to RUNNING; grant returned only through the persisted draw_grant table"
    adr: docs/architecture/adr-0012-graph-propagation-scheme.md
    not: "a synchronous apply of M14, M15 or M17; a synchronous grant; a drain outside phase 0"
    added: 2026-10-29
```

### api_decisions
```yaml
  - purpose: graph_write_disciplines
    status: proposed
    api: "Every Graph phase declares per written column one of JAC (read start-of-phase values, write a GraphScratch accumulator, apply in one ascending loop that re-zeroes it), ALLOC (ascending-id allocation against a running balance) or OWN (own row only); loops iterate derived active lists; accumulators are all-zero at hook exit"
    not: "next-column copy and swap for pairwise fluxes; iterating to convergence; recursion; catch-up; iterating a table or Dictionary in a phase"
    adr: docs/architecture/adr-0012-graph-propagation-scheme.md
    added: 2026-10-29
  - purpose: graph_fairness_gate_function
    status: proposed
    api: "graph_rules.gate_floor(hp_before, hull_or_seam, gate_open, hp_fail, breach_hp) and apply_loss(hp_before, loss, floor_hp): a floor applied at the moment of damage that limits the reduction and never raises hp; EXOGENOUS causes pass floor 0; the node/edge EXOGENOUS flag is the descriptor cause of the current episode"
    not: "a post-pass that rewrites hp; rate_pm or stress_pm read outside the damage phase"
    adr: docs/architecture/adr-0012-graph-propagation-scheme.md
    added: 2026-10-29
  - purpose: graph_rng_subject_key
    status: proposed
    api: "stream graph.fail (plain); counter (step_no << 26) | (kind << 24) | id with kind 0 node, 1 edge, 2 reserved compartment and id < GraphLimits.ID_MAX <= 2^24; burst test posmod_i(raw & 0x7FFFFFFFFFFFFFFF, 1000) < burst_p_pm; no draw counter stored; no restore epoch"
    not: "draw_epoch for a Graph draw; a node id and an edge id sharing one subject field; a float or bare % in the burst test"
    adr: docs/architecture/adr-0012-graph-propagation-scheme.md
    added: 2026-10-29
  - purpose: graph_derived_rebuild
    status: proposed
    api: "GraphDerived.ensure_derived is idempotent and never resizes (size_derived/size_scratch/size_descriptors are structural-mutator and post_restore only); called at the end of every structural mutator and post_restore, at the end of phase 9 after a connectivity change, and as the defect-only head guard of each hook (late_rebuilds); degrees and deg_max are phase-6 scratch, not cached"
    not: "lazy-only topology refresh; a hook reading a half-applied topology; a resize on the hook path"
    adr: docs/architecture/adr-0012-graph-propagation-scheme.md
    added: 2026-10-29
```

### performance_budgets
```yaml
  - system: graph_hooks
    status: proposed
    budget_ms: 0.5
    adr: docs/architecture/adr-0012-graph-propagation-scheme.md
    notes: "GRAPH_HOOK_BUDGET_MS spike share: P99 0.5 and max 1.0 on a release export on RM-DESKTOP and RM-DECK, pass/fail on the maximum over the three hook ids (power, atmo, failure) reported separately, not the step divided by three; the ADR-0003 P3 share ledger row; pending S1 / S0-f; the number moves, not the design, if missed"
    added: 2026-10-29
  - system: graph_snapshot_restore
    status: proposed
    budget_ms: 0.5
    adr: docs/architecture/adr-0012-graph-propagation-scheme.md
    notes: "snapshot(ctx) payload build at most 0.5 ms and at most 32 KB for the MVP ship; restore plus post_restore at most 2 ms; kernel integration world 3 ms / 10 ms is the kernel's; pending S3"
    added: 2026-10-29
```

### forbidden_patterns
```yaml
  - pattern: graph_iterate_to_convergence_or_catch_up
    status: proposed
    description: "No Graph phase iterates to a convergence condition, recurses, repeats a step or runs a missed step; one pass per loop per step"
    adr: docs/architecture/adr-0012-graph-propagation-scheme.md
    added: 2026-10-29
  - pattern: graph_wall_clock_or_epoch_in_propagation
    status: proposed
    description: "No wall clock, no restore_epoch, no draw_epoch, no float and no bare % in src/sim/graph/; budgets are counted work units per commit and wall clock on hardware nightly"
    adr: docs/architecture/adr-0012-graph-propagation-scheme.md
    added: 2026-10-29
  - pattern: graph_hook_product_cache_not_persisted
    status: proposed
    description: "A value read by a later Graph hook or another block between hooks must be persisted, recomputed at its single point of use from persisted state, or listed as a re-derived cache with a stated window (ADR-0012 § 11); the intensive atmosphere caches are closed out by phase 0 so they hold at every tick boundary"
    adr: docs/architecture/adr-0012-graph-propagation-scheme.md
    added: 2026-10-29
```

### referenced_by_update (existing entries that gain this ADR in referenced_by)
- kernel_p3_step_hooks (performance_budgets, ADR-0003): add this ADR (the Graph's spike share)
- native_code_policy (api_decisions, ADR-0010): add this ADR (ladder reconciliation, rung order for the Graph)
- dirty_key_encoding (interfaces, ADR-0004): add this ADR (emission schedule)
- packed_array_exposed_or_setter_tracked (forbidden_patterns, ADR-0010): add this ADR (accumulators are private scratch)

## GDD sync
Items to apply to `design/gdd/life-support-graph.md` (and siblings) when this ADR is accepted; none is applied by this ADR:
1. **LS-8 (topology refresh) and phase 1:** "Topology refresh is one idempotent function run at the head of whichever Graph hook runs first after a structural mutator" gains "and eagerly at the end of every structural mutator (M1-M6, M8, M9, M19) and in `post_restore`; the head call is a guard counted by `late_rebuilds`"; `degrees` and `deg_max` leave the phase-1 list and are recomputed at the head of phase 6.
2. **LS-8 phase 11 row:** delete "`block_hash` dirty flag" (the kernel invalidates the hash by the touch serial, ADR-0005 § 5.2).
3. **LS-8 phases 9 and 10:** state that phase 9 writes a subject's own forced mode and phase 10 queues cross-subject chains; unadmitted candidates are re-proposed, not stored. Phase 9 evaluates `T_mach` from the current persisted temperatures (D.8).
4. **LS-1 (j) / ADR-LSG-01 and LS-8:** "double-buffered `_cur` / `_next` start-of-phase snapshot" becomes "the live column is the start-of-phase snapshot until the end-of-phase accumulator apply (JAC)".
5. **LS-9 and the TR-lsg-011 registry text:** "`rng_derive(world_seed, "graph.fail", (step_no << 20) | subject_id)`; `subject_id < 2^20` is asserted" becomes "counter `(step_no << 26) | rng_subject(kind, id)` with `rng_subject = (kind << 24) | id`, kind 0 node, 1 edge, 2 reserved, `id < GraphLimits.ID_MAX <= 2^24` (T7)".
6. **LS-17 (M19) and LS-18 M19:** the list of advanced stamps gains `thaw_until_step` (non-zero) and a LOAD's `energised_at_step` (non-zero); `stress_until_step` stays unadvanced; the descriptor store holds loaded sections only.
7. **LS-21 (and LS-19, LS-4 wording "extensive stored, intensive derived" for this one case):** "`served_pm` is not [persisted], because phase 2 recomputes it before use" becomes "`served_pm` (`n_served`), the power delivered to each consumer (`n_draw`), a cable's net load and the five-step `power_use` window are persisted hook products written by phase 2 only, so a restore on any tick reproduces the uninterrupted run; `post_restore` no longer derives them". LS-8 phase 0 gains "closes out the intensive caches of the compartments it touched" (T2).
8. **LS-11 (SHORT) and D.9:** "re-checks `S + D` > 0 on this run's phase-2 result" becomes "re-checks that an island at either end holds a live source from the persisted `out_w` and `stock` at phase 8".
9. **LS-19 `run_pm`:** a HEATER's `run_pm` "`⌊duty_t × served_pm / 1000⌋`" becomes "`⌊1000 × n_draw / max(1, req)⌋`".
10. **Edge Cases (footprint drain rule) and TR-lsg-035 text:** delete "a hook that declares `before: graph.propagate.power` drains at 6k" (no `before` edge exists); "phase 0 of the same tick when made by a P2 command or a hook ordered before" becomes "phase 0 of the first `graph.propagate.power` execution that follows the deposit in kernel order (tick, P2 before P3, compiled schedule position): the same tick when that tick is a power-hook tick and the deposit precedes the hook, otherwise the next power-hook tick; an INIT deposit drains at the first power-hook tick after LOADING to RUNNING".
11. **LS-13 / LS-14:** name `cause` as persisted (the EXOGENOUS flag) and state that it clears when the dominant mode returns to 0 or `hp` reaches `HP_OK`.
12. **LS-18 M1:** the closed reason set of `check_instantiate_section` gains `ONSET_UNKEYED` (a `mode_since_step` seeded without `mode_key`).
13. **D.16 ladder:** the GDD's rung numbers (stagger 1, quiescence 2, GDExtension 3) are reconciled with ADR-0010 § 9 (tuning, call overhead, GDExtension) as the table in ADR-0012 § 13.1 states.
14. **LS-24 (acceptance, lint):** "no loop iterates to a convergence condition, no function is recursive" is enforced by `PROP-01` (Foundation Ask 3).
15. **D.9 / LS-11 gating leak wear:** the test "descriptor `intensity_pm` > 0" becomes the persisted `isolated_at_step` stamp (non-zero behind a closed valve), the form phase 8 can read and a restore reproduces; `design/gdd/derelict-authoring-handmade-kits.md` carries the same wording. *(Amended 2026-10-29)* LS-11 and D.9 also carry the seeded-stamp clear rule of § 8.4 (cleared on the first run whose `intensity_pm` rises above 0).

## Foundation Asks
Changes this ADR needs in Foundation ADRs; none is worked around silently.
1. **ADR-0010: OWN-01 must accept a directory (absorbed into ADR-0010's Foundation Ask 1, CT-07 and CT-54).** `field_writers.json` accepts directory globs and per-member entries; the per-class writer matrix is ADR-0011 § 12.1 (GraphState: `propagate/**`, `graph_mutators.gd`, `graph_restore.gd`; GraphCaches and GraphScratch as that section states). This ADR no longer restates the set.
2. **ADR-0010: `HOT-01` tag list and permitted tokens.** Commit `tools/lint/config/hot_functions/graph.json` (the function list of § 13.2) as the owner's tag list, and confirm that `PackedXArray.fill`, element reads and writes, `mini`/`maxi`/`clampi`/`absi`/`signi`, `IntMath` calls and `Callable.call` with a fixed argument list are not HOT-01 tokens. The `WorkCounter` port carries only `add(unit: int, n: int) -> void`; unit ids are per-module constants (ADR-0010 OQ-O, closed by ruling R5), so this ADR's units live in the Graph module constants class `GraphUnits` (§ Key Interfaces) and no `WorkCounter.UNIT_*` port constant is assumed; `CNT-04` closed forms stay comparable because each module states its own unit table.
3. **ADR-0010: a convergence and recursion lint, `PROP-01`.** In `src/sim/graph/propagate/`: no `while` loop, no direct self-recursion, and no `for`/`range` bound that is assigned inside the loop. Needed for the GDD's acceptance criterion on the propagation file (LS-24, Section H).
4. **ADR-0003 / ADR-0001: say what `ClockView.tick()` returns inside P3** (the executing tick or the next) (ruling R4: the per-`(tick, hook)` elapsed samples S1 needs are no longer asked of ADR-0003 or ADR-0001; they are an open dependency on ADR-0031 and the S1 harness, Open Question 11). The Graph is insensitive to the answer for offsets up to `graph_step_ticks - 2`, but KRN-LAT-02 should record it. ADR-0001 also gains one sentence that the tick counter T is gapless (forgiveness drops lag wall time, never a T value; CT-37), and ADR-0003 holds the P3 share ledger in which this ADR cites its row (CT-20).
5. **ADR-0005 § 2.4: reconcile the stream label** (`graph.fail` in the table and the GDD, `graph.failure` in a golden-vector row) and note the Graph's 26-bit subject key of § 8.2 (`rng_subject = (kind << 24) | id`, counter `(step_no << 26) | rng_subject(kind, id)`) beside the GDD's `(step_no << 20) | subject` counter (ruling R11: the LS-9 GDD text and TR-lsg-011 join the Core GDD sync, item 5 above).

## Cross-ADR assumptions
Assumptions this ADR makes about sibling Core ADRs (also returned in the structured result):
- **ADR-0011** (the owner adopts these by rulings CT-01, CT-04 to CT-08): (1) persisted node columns `n_served` and `n_draw` (`int32`, canonical zero off power consumers), a CABLE edge's `e0` as its net load, and `hdr.power_use_w5` (five ints); `served_pm` leaves the DERIVED class (§ 6.1, § 7.4 (3), Alternative 7); the F-MVP size becomes about 30.5 KB plus 40 bytes; (2) `hdr.last_step_no` is the last step whose **failure** hook completed (LS-8, CT-05); (3) `GraphFaults` gains local code 25 `HOOK_CURSOR` (runtime, raised by § 1) and 26 `RESTORE_HOOK_CURSOR` (restore, § 7.2 step 13), and `hdr.sched` stores the schedule in force (CT-08); (4) `GraphScratch` holds the § 3 inventory (`PackedInt64Array` accumulators and the island, candidate, sort and traversal buffers) and is all zero at every hook exit; `GraphCaches` gains `active_nodes`, `active_edges`, `active_comps`, `active_seams`, `store_slots`, `machine_slots`, `exchange_links`, the island list, `alarm_sections`, `size_derived()` / `size_scratch()`; `GraphDiag` gains `late_rebuilds`, `queue_dropped`, `footprints_clamped`, `schedule_shift`; all but the columns are non-persisted members listed in `derived_members.json`; no total-amount accessor is added to `GraphState` (CT-06); (5) the writer matrix of § 12.1 (CT-07); (6) `GraphModule.setup` gains a trailing `work: WorkCounter` and uses `ModuleFacet.rng` to register `graph.fail`; (7) `GraphTuning` carries `hook_offset_power/atmo/failure`, `load_queue_max`, `w_min_steps`, `burst_*`, `rate_pm` inputs, and `GRAPH_BOOT_TUNING` checks `GraphLimits.ID_MAX <= 2^24` and every queue bound at most 65,535; (8) `check_instantiate_section` has the reasons `ONSET_UNKEYED` and `CAPACITY` and is the function `GraphLayoutAdapter` exposes through `LayoutGraphPort` (§ 9.5); (9) `GraphIds.ALARM_CAPABLE_MASK`; (10) the EXOGENOUS flag clears per § 7; (11) the descriptor store holds loaded sections only; (12) the first hook run after a load compares `hdr.sched` with live `GraphTuning` (§ 7.4 rule, § 1 here).
- **ADR-0013**: Layout supplies room volume and exposed tiles only through `RoomSource`; no Graph phase writes a door `area` or calls `set_opening_area`; `layout.doors` runs before the Graph hooks' dependents and never writes the Graph; M1 takes `section_id` as an argument, is called only by Layout LM1 through `LayoutGraphPort`, and returns ADR-0013's `SectionBases`.
- **ADR-0016**: the stock reconcile stays in phase 0 (agreed, ADR-0016 § 4.4); a faulted `LOADING` never enters play; `post_restore` of the Graph follows Layout's; the hook-cursor check is covered by reference to ADR-0011 § 7.2 step 13 (V3), which needs `kernel.clock` (T, world_seed) as a Tier S read from R2.
- **ADR-0021**: calls `can_join` / `check_*` in `validate` and the mutators in `apply` within P2 (and M5/M6/M7 within an Interaction hook), relies on `ensure_derived` being complete on return, closes the reason sets of § 6, and owns the no-rollback question (OQ-15); the Graph's mutators are infallible after their checks.
- **ADR-0017**: `emitter.place` is ordered after `graph.propagate.failure`, deposits M14 on the § 5 schedule (including from the INIT-scope E1, drained at the first power hook, which is D5.2's `arm_tick`), reads `served_pm` (persisted, exact) and `failures()` (re-derived, window of § 11), and decides its sensitivity to that window; its D14.4 drops the "if ADR-0011 declines" fallback text.
- **ADR-0022 / ADR-0023**: Attention and Horror state the exogenous tell for M17 and M18; Attention reads `cause`, `alarm_sections()`, `power_use` (restored exactly from `power_use_w5`) and sends STRESS with the registry constant as magnitude.
- **ADR-0026**: Web Map calls M19 in `init_world` under INIT and in P2 applies (departure, pod_settle, berth, reconcile), never from a Graph hook; it reaches M1 only through Layout LM1 and the port; it does not need a dormant section's descriptors from `failures()`.
- **ADR-0027**: Suit's `suit.vitals` calls M15 and M16 in its HOOK scope, accepts a grant that arrives at the next drain, and tests breath-sequence equality.
- **ADR-0003 / ADR-0001**: the P3 share ledger holds the Graph's worst-hook row (CT-20); T is gapless (CT-37).

## Review Log
Review Log dated 2026-10-29 (engine seat, Godot 4.7.2, verdict CONCERNS; TD-ADR seat, verdict CONCERNS; Core-tier cross-ADR adjudication, autonomous mode "Revise flagged items"). Status stays Proposed. Applied unless marked otherwise.

**Engine review**
- **E1 (high, blocking): applied.** Typed iterators in the PhaseExchange excerpt (`for k: int in n_links`) and in the § 5 sort text (`for j: int in range(i - 1, -1, -1)`, also § 3); `_views` on the hook skeleton and the note covering `hook_atmo` and `hook_failure`; `_d` and `_scope` in the excerpt. The "load-checked in a scratch project 2026-10-29" sentence is deliberately **not** added: the corrected snippets were not re-probed, and Key Interfaces says so.
- **E2 (medium): applied.** `run_failure(..., rng: RngFacet, stream: RngStream, w)`; `hook_failure` passes the facet and `_stream_fail`; § 8.2 names where `rng` and `stream` come from. The `GraphDraw` alternative is not taken.
- **E3 (medium): applied.** The excerpt aliases every column in locals and computes `n_o2[r] + n_inert[r]` inline (`_c_n_total_at` removed, no accessor added: also CT-06); the probe caveat (local 13 ns versus member 64 ns per element, method call about 117 ns, static call about 123 ns) is in Engine Compatibility, § 13.1 and Performance Implications; an Implementation Guideline was added.
- **E4 (medium) and T6 (low): applied, widened-key option.** Draw key `(cid << 32) | (crew << 16) | arrival_index`, trace key `(subject_ref << 32) | (kind << 16) | arrival_index`, bit widths and the D.13 bound (below 2^56) stated in § 5, `GRAPH_BOOT_TUNING` asserts every queue bound at most 65,535, a 300-entry fixture added to Validation. Risk 10 records that the quadratic sort was measured only to 64 entries.
- **E5 (low): applied.** Probe (c) is the `for` plus `break` form; typical figures 3.5, 27.0 and 33.3 microseconds, worst case 55.3 microseconds reverse-sorted at 64 entries, quoted as the bound; the `while` figures are marked superseded.
- **E6 (low) and CT-37: applied.** § 1 now says T is gapless, forgiveness drops lag wall time and never a T value, so no hook is skipped, and a cursor gap arises only from a refused or faulted hook or a restore.

**TD-ADR review**
- **T1 (high, blocking) and CT-36: applied.** M14 scope is APPLY, HOOK, INIT and M19 is APPLY, INIT; § 5 has the INIT drain row (first power-hook execution after LOADING to RUNNING, equal to ADR-0017 D5.2's `arm_tick`) with the N2-clock stamp, the `dormant_since` INIT stamp and the discarded TOPO/SECTION marks; the ADR-0026 assumption is replaced with the ruled wording.
- **T2 (high, blocking): applied, option (a).** Phase 0 ends by recomputing `P`, `p_o2`, `o2_pm` and trend for the compartments it touched (§ 2 table row 0 and note; structural mutators via `ensure_derived`); the "lags the breath by up to two ticks" note is deleted; the § 11 row stays DERIVED and exact with its sim readers named; RE-1 and RE-2 gain a restore at 6n+1 after a draw and after a BURN.
- **T3 (medium): applied.** RE-1 requires `graph.state` hash equality at every tick from *t*+1 (§ 11 Tests and Validation). The weaker fallback form is **not** kept: CT-04 removes the "if ADR-0011 declines" case.
- **T4 (medium): applied.** § 13.1 requires P99 and max per hook id plus the failure hook on a phase-9 `ensure_derived` tick; pass/fail is the maximum over the three; Performance Implications calls the even split an assumption S1 replaces; Open Question 9 stays with the technical-director.
- **T5 (low): applied.** The § 5 caller table keeps only the orderings ADR-0012 declares or owners confirm (suit.vitals after atmo, attention after power, emitter.place after failure), adds the generic "any hook placed after the power hook" row, and names Risk 7's golden compiled-order test.
- **T7 (low): applied, widened-field option.** `rng_counter = (step_no << 26) | (kind << 24) | id`, id sized to `GraphLimits.ID_MAX`; step_no below 2^30 keeps the key below 2^56; golden vectors (node 5 and edge 5 at step 123,456: 8,284,991,913,989 and 8,285,008,691,205) are arithmetic, not re-probed; the registry candidate, the GDD sync items and the Validation line follow.
- **T8 (low): applied.** The § 6 Budgets bullet and § 13.6 place applied M13 in P2 apply and report the Graph's P2 share separately from the hook share.
- **T9 (low): applied.** § 4 states that `ensure_derived` never resizes and the phase-9 call cannot resize; § 13.2 and the HOT-01 coverage include the phase-9 path.
- **T10 (low): applied.** Traceability sentence reworded (28 planned rows, 18 full and 10 partial, TR-lsg-020 residual, 13 owned elsewhere).
- **T11 (low): no change required.** The fixture-by-fixture Validation lists move into story acceptance criteria when the Graph epic is created.

**Cross-ADR rulings**
- **CT-01 (owner ADR-0011): aligned.** M1 row and Rules state the single caller (Layout LM1 through `LayoutGraphPort`, `SectionBases` out type, `check_instantiate_section` shared with `GraphLayoutAdapter`); no `InstantiateResult`.
- **CT-04 (owner ADR-0011): aligned.** `n_served`, `n_draw`, CABLE `e0` and `power_use_w5` are adopted persisted products; the "power_use window seeded flat" divergence and the "if ADR-0011 declines" fallback text (§ 11, Alternative 4, Risk 1, Open Question 1) are deleted; size is about 30.5 KB plus 40 bytes at about 4.6 percent headroom.
- **CT-05 (owner ADR-0011): aligned.** `last_step_no` is the failure hook's step and trails `step_no()` by up to one step.
- **CT-06 (owner ADR-0011): aligned.** `GraphScratch` inventory wording and the inline `n_o2 + n_inert`; no accessor.
- **CT-07 (owner ADR-0011): aligned.** Key Interfaces cites ADR-0011 § 12.1's writer matrix; Foundation Ask 1 is absorbed into ADR-0010's; the guideline restating the set is removed.
- **CT-08 (owner ADR-0011): aligned.** § 1 retargets the cursor check to ADR-0011 § 7.2 step 13, names local codes 25 (runtime) and 26 (restore), and states the first-run `hdr.sched` rule with `schedule_shift`; the ADR-0016 Enables row drops "hook cursor verifier".
- **CT-20 (owner ADR-0015): aligned.** § 13.1 cites the ADR-0003 P3 share ledger row and the 1.66 ms Core sum; Risk 11 and Foundation Ask 4 record it. No new numbers invented.
- **CT-54 (owner ADR-0010): no Core text beyond the alignments above.**

**Rejected findings:** none of the rejected items touches this ADR. The `can_instantiate` ask for static-only classes (CT-54) needs no text here.

- **2026-10-29 confirm residuals:** the scratch inventory of § 3 gains `deg` (with `GraphScratch.deg_max_of(e)`), and the loop summaries are named `GraphCaches.loop_sum` (§ 3, § 4, § 11) to match ADR-0011's member lists; Enables row no longer lists ADR-0016 or ADR-0011 (neither lists this ADR under a hard Depends On); the id field stays 24 bits (`ID_MAX <= 2^24`), and ADR-0011 now says so.
- **2026-10-29 orchestrator rulings:** R4 (Foundation Ask 4 no longer asks ADR-0003/ADR-0001 for per-`(tick, hook)` elapsed samples; recorded as open dependency, Open Question 11, on ADR-0031 and the S1 harness), R5 (`WorkCounter.UNIT_EDGE` becomes `GraphUnits.EDGE` in a Graph module constants class; Foundation Ask 2 states the port carries only `add(unit, n)`; ADR-0010 OQ-O closed), R11 (Foundation Ask 5 now names the 26-bit subject key of § 8.2; LS-9 and TR-lsg-011 join the Core GDD sync). Dependency-cycle break: the `Related, not Enables` sentence and the unwritten ADR-0021 to ADR-0031 consumers moved out of Enables into a new `Soft references (not acceptance prerequisites)` row; Enables now lists ADR-0017, which lists this ADR under Depends On.
- **2026-10-29 seeded-stamp clear rule (Derelict DA-17), autonomous-mode ruling: applied.** § 8.4 now states that an M1-seeded `isolated_at` on a subject with no isolating valve persists while the seeded `(mode_since, mode_key, isolated_at)` triple matches the live key, holds leak wear, and is cleared on the first propagation run in which the subject's descriptor `intensity_pm` rises above 0 (in addition to the clears on valve reopen and mode clear). § 8.1 cross-references it. M1 accepts a back-dated (negative) `mode_since` with `mode_key != 0` per ADR-0011 § 3.2 (signed stamps); `isolated_at` seeds are `>= 1`. TR-derelict-037 dependent tests gain the `pb_leak` seeded-stamp hold. Source: Derelict round-2 blocker 1.
