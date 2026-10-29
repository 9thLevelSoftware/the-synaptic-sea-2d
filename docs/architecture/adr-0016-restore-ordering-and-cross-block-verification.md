# ADR-0016: Restore Ordering and Cross-Block Verification

## Status
Proposed

## Date
2026-10-29

## Last Verified
2026-10-29 (against `design/gdd/command-simulation-layer.md` CR-7, CR-8, CR-15, CR-18, States and Transitions and edge cases on `post_restore` and the binding table; `ship-section-and-layout-data-model.md` LY-26 c and the acyclicity argument; `life-support-graph.md` LS-21, LS-24; `modular-ship-assembly.md` MSA-32 and T7; `interaction-system.md` IA-36, IA-37; `suit-and-equipment.md` SE-13; `ship-body-and-interior-physics.md` BDY-31; `save-load-and-versioned-format.md` SL-12, SL-15; ADR-0001 to ADR-0007 as Proposed on 2026-10-28 and 2026-10-29; the wave-A drafts ADR-0011, ADR-0013 and ADR-0019; `docs/engine-reference/godot/` `VERSION.md`, `breaking-changes.md`, `deprecated-apis.md`, `current-best-practices.md`). No game code exists. Four small engine facts were probed on the 4.7.2 mono console binary on 2026-10-29 (§ Engine Compatibility); every other engine claim is marked UNVERIFIED with the spike or test that settles it.

## Decision Makers
Claude (author), running under modes.automation: autonomous (project.yaml); decisions logged for the user's review; the user has not yet reviewed this ADR.

## Summary
Thirteen persisted blocks, the kernel clock and one non-persisted derived block must be rebuilt, in an order every owner can rely on, by a `LOADING` pass that never plays on a half-restored world, never lets one block's rebuild read another block's unbuilt derived state, and rebuilds the kernel's ownership table last. This ADR fixes the pass as one numbered sequence (pre-check, clock install, per-block `restore` or `default_state` in `restore_after` order, per-block `post_restore` in the same order, the reproduction verdict, the binding rebuild, the exit), the compile-once `restore_after` DAG with its tie-break and boot codes, the write and read rules of `restore` and `post_restore`, where each cross-block verifier lives and in what order they run, the binding-source contract, the new-game variant, and the tests (order-permutation, save-equivalence, fault ladder, leak cycle) that make the rules checkable.

## Engine Compatibility

| Field | Value |
|-------|-------|
| **Engine** | Godot 4.7.2 (stable), GDScript |
| **Domain** | Core (kernel load pipeline, owner-callable contract, deterministic ordering) |
| **Knowledge Risk** | HIGH for the owner-callable contract this ADR drives (4.7 typed-return override rule; packed-array element writes call no setter; both in `breaking-changes.md`, both inherited from ADR-0003). MEDIUM for the rest: nothing here uses a post-cutoff API, but three engine behaviours the design leans on are probed or UNVERIFIED below. |
| **References Consulted** | `docs/engine-reference/godot/VERSION.md`, `breaking-changes.md` (4.6 to 4.7 GDScript rows), `deprecated-apis.md` (no entry for any API used), `current-best-practices.md` (typed GDScript, `RefCounted` plain-data blocks). The reference has no core or scripting module, so everything outside those four files is training knowledge unless probed. |
| **Post-Cutoff APIs Used** | None. The 4.7 typed-return override rule applies to every stub or subclass of an owner callable (ADR-0003 § 1) and to the fixture blocks of § 9. The 4.7 packed-array element-write change is avoided as in ADR-0003: rebuilt arrays are assigned whole, never mutated through a setter-observed property. |
| **Probes run 2026-10-29 (4.7.2.stable.mono console binary, `--headless --script`, scratch directory `C:/Users/dasbl/AppData/Local/Temp/claude/adr-probes/0016/`, outside the repo)** | **(P1) `StringName` ordering is not lexical (corroborates ADR-0005 P12, which states it first).** `&"body.state" < &"controller.state"` is `false`; `Array.sort()` over an `Array` of `StringName` returned `[layout, sanity.state, sanity.ledger, controller, suit, body, graph]` for one set and `[zz.one, mm.three, aa.two]` for another, and the engine review reproduced the wrong order on a 14-id set (`[layout, graph, sanity.perception, ... body]`); `Array[StringName].sort()` gives the identical wrong order, and the same four names created in the reverse order sorted alphabetically. The order therefore depends on interning order (a pointer or intern-order compare; mechanism UNVERIFIED) and may differ between builds, platforms and hosts, not only between runs. `String(a) < String(b)` and `Array[String].sort()` gave the alphabetical order every time. `StringName < StringName` passes the analyzer, so the compiler will not catch it; the fence is the existing ADR-0005 pattern `stringname_sort_for_canonical_order` and the ADR-0010 DET pack, and this ADR adds no comparison of its own: no kernel code orders `StringName` values (§ 2.2). **(P2) A Kahn topological sort of the MVP `restore_after` graph with a ready-list tie-break changes its result with the tie-break rule** (in GDScript, smallest and largest `String` id gave two different valid orders of the same 13 blocks, and a graph with one injected back edge yielded no order; the same algorithm with smallest registration index, run in a throwaway Python script because it is not an engine fact, gave a third order, § 2.4), which is why § 2 states the tie-break and § 8 tests against alternative linear extensions. **(P3) A scratch `RefCounted` that is built, copied from and dropped 100 times leaves `Performance.OBJECT_COUNT` unchanged (delta 0); the same loop with a two-object reference cycle left 200 extra objects (delta 200; `get_monitor` returns float, `200.0`, so compare with `int()`) and printed "202 ObjectDB instances were leaked at exit".** The engine review added two cycle types `OBJECT_COUNT` does not see: 1,000 self-referencing `Dictionary` and `Array` pairs left `OBJECT_COUNT` at 0 and raised static memory by 612,192 bytes, and a `RefCounted` that stores a lambda capturing `self` leaks one object per instance (delta 100 for 100 instances). `OBJECT_COUNT` therefore detects Object cycles only; containers and self-capturing lambdas need the ADR-0010 CNT-02 `MEMORY_STATIC` check (§ 10, KRN-RST-10; ADR-0010 P6). **(P4) `Performance.get_monitor` returns values headless for `OBJECT_COUNT`, `OBJECT_NODE_COUNT` and `OBJECT_ORPHAN_NODE_COUNT`, but render monitors are 0 under the dummy renderer** (`RENDER_TEXTURE_MEM_USED` reads `0.0` headless), and a baseline `OBJECT_ORPHAN_NODE_COUNT` of 1 appeared even in a bare `--script` run, so comparisons use deltas against cycle 1, never absolute numbers. 'Readable headless' is not 'measurable headless': render-texture legs run in the windowed lane W (ADR-0010 § 8). `RENDER_TEXTURE_MEM_USED` is a memory figure, not a texture count; whether a render-texture **count** is readable on 4.7.2 is UNVERIFIED and is ADR-0028's to settle. Whether `OBJECT_ORPHAN_NODE_COUNT` is populated in a release export is UNVERIFIED (treated as debug-binary only). **(P5) Reference semantics at restore.** `Packed*Array`, `Array` and `Dictionary` are reference types in GDScript: after `block.col = payload["col"]`, a write to `block.col[0]` changed `payload["col"]`; an alias taken from a member also writes through to it, and a nested `Array` stored via `b.d = payload` aliases the payload. Hence the copy-by-value rule of § 3.1. |
| **Verification Required** | (1) **Gates acceptance.** KRN-RST-01, KRN-RST-02 and KRN-RST-04 run green on a headless kernel skeleton (the KRN-LAT-02 build) with recorded stub blocks, together with the **stub-level** KRN-RST-05: recorded stub blocks carrying the MVP `restore_after` edges are restored under four distinct linear extensions of the DAG; a planted stub that reads a non-predecessor's Tier D state must diverge under at least one extension, and a stub that respects its edges must match under all four. The real-world read-view battery of KRN-RST-05 needs implemented Layout, Graph, Emitter and Body blocks (Migration Plan stories 5 and 6, blocked while those ADRs are Proposed) and is a Save/Load or Kernel epic closure gate, not an acceptance gate (§ 8). architecture.md § Spikes that gate ADRs lists this ADR under KRN-LAT-02 (GDD sync item). (2) The cost of a full MVP `LOADING` pass in GDScript on a release export (RM-DESKTOP and RM-DECK) is UNVERIFIED; it is measured by SAVE-WRK-1-SLICE and S1 (architecture § Spikes that gate ADRs) and decides nothing in this ADR, only the `loading_ms` figures of KRN-RST-14. (3) What an owner callable returns when it aborts on a script error in a release export is ADR-0003's Verification 1 (KRN-CTX-06); this ADR relies on its fail-closed result only through `ctx.fault` and the token check. (4) Whether a render-texture count is readable (TR-light-037) is ADR-0028's, and it is measured in the windowed lane W, not headless (P4). (5) GDScript `RefCounted` reference cycles are not collected (probe P3); `OBJECT_COUNT` sees Object cycles only, so KRN-RST-10 adds the ADR-0010 CNT-02 `MEMORY_STATIC` check (debug and editor binaries; release availability UNVERIFIED), and the scratch rules of § 3.1 are the guard, not the engine. (6) Whether `OBJECT_ORPHAN_NODE_COUNT` is populated in a release export is UNVERIFIED; KRN-RST-10 treats it as debug-binary only. |

## ADR Dependencies

| Field | Value |
|-------|-------|
| **Depends On** | ADR-0003 (block descriptor incl. `restore_after`, `binding_source`; contexts; `WriteScope` phases `INIT` and `RESTORE`; fault domains; state machine incl. `FAULTED` re-entry; Proposed), ADR-0006 (two-phase `restore` contract, staged payload shape, REBASED and `REBASE_BLOCKS`, failure classes, `restore_epoch` as an argument; Proposed), ADR-0002 (binding table half (b), `ControlHandle.request_restore`/`request_new_game`, `Dropped(RELOAD)`; Proposed), ADR-0001 (`load_seq`, `snap_baseline` on entry to `RUNNING`; Proposed), ADR-0004 (one `ALL` batch per block after `LOADING`; Proposed), ADR-0005 (uncached `state_hash` for the verdict, hash-cache reset; Proposed). |
| **Soft references (not acceptance prerequisites)** | ADR-0011 (Graph `restore`/`post_restore`, stock reconcile), ADR-0013 (Layout `restore`/`post_restore`), ADR-0015 (Body `post_restore`, `seat_bindings`) and ADR-0019 (Controller `restore_after`, `post_restore` order): all read as drafts only; none lists this ADR under Depends On and this ADR does not list them either. **Forward references (unwritten ADRs; each will list this ADR under Depends On when authored):** ADR-0020 (Interaction `post_restore` policy), ADR-0021 (Assembly verifier call sites and the weld property test), ADR-0022 and ADR-0023 (Attention and Horror restore), ADR-0025 (Sanity restore), ADR-0026 (Web Map `init_world` and restore reasons), ADR-0027 (Suit `post_restore` decode), ADR-0028 (pool rebuild under the LOADING leak cycle). |
| **Enables** | ADR-0017 (Emitter `post_restore`, `beam_bindings`, G4 save-equivalence). |
| **Blocks** | Epics: Command & Simulation Layer (the `LOADING` pass and registry freeze), Save/Load (restore stories, `REBASE_BLOCKS` gate), and every story that implements a block's `restore`, `default_state` or `post_restore`. |
| **Ordering Note** | Wave B. Written after ADR-0011, ADR-0013 and ADR-0019 (drafts) and in parallel with ADR-0012, ADR-0014 and ADR-0015; every assumption about a sibling is listed under Related. **Scope edges.** ADR-0003 owns the descriptor field list and the callable shapes; this ADR gives `restore_after` its meaning and adds four kernel fault codes it needs (18 to 21), one boot warning and three H1 allow-list names (foundation asks, § Key Interfaces). ADR-0006 owns what Save/Load does with the file and the failure classes; this ADR owns what the kernel does with the staged payload inside `LOADING`. ADR-0002 owns the shape of the binding table; this ADR owns the rebuild's order, its input format and its failure behaviour. Each owner ADR owns the content of its own `restore`, `post_restore` and verifier checks; this ADR owns the order they run in, what they may read and write, and the numbered list of checks (§ 5). |

## Context

### Problem Statement
Every owner GDD already promises something about restore, and the promises depend on an order nobody has fixed. Layout's `post_restore` derives door leaves from the Graph's persisted `area`; the Emitter's cold static build is right for open doors only if Layout's `post_restore` ran first (Emitter Open Question 26); Suit's `post_restore` decodes held instances with the restored Graph's step number; Interaction's `post_restore` wants to re-register subjects in a block it does not own; the Controller reads Suit's cap in its own; Assembly's verifier assumes four other blocks are whole; the kernel must rebuild the PlayerId ownership table from sources whose own state is rebuilt only in `post_restore`. The kernel GDD says "restore order follows declared `restore_after`" and "every `post_restore()` runs after every `restore()`", and says nothing about the order among `post_restore` calls, nothing about `default_state` and `init_world` order, and treats a failed `post_restore` as an edge case. The Layout GDD calls the `post_restore` order "informational only". Left like that, fifteen owners would each assume a different reading and the first save/load test that crosses two blocks would find the disagreement. The same gap hides a second question: what a failed load leaves behind, and how anyone proves a restored run equals an uninterrupted one (kernel G4).

### Constraints
- **The kernel holds no domain state** (H1). Verifiers therefore cannot be a kernel-owned rule list; they run inside owner callables.
- **`restore` copies into the existing block object** and failure is `ctx.fault` only (ADR-0003 § 2, ADR-0006 § 2.2). Read-view facades are long-lived (ADR-0003 § 7).
- **`LOADING` is atomic within one boundary service** and never spans a frame (kernel GDD); no Save/Load I/O happens inside it (ADR-0006 § 3.3).
- **`WriteScope` phase `RESTORE` opens only the restoring block; `INIT` opens any block** (ADR-0003 § 4). `SnapshotContext` has no `submit_system`, so a `post_restore` cannot submit a command by construction.
- **Dirty marks in `INIT` and `RESTORE` scopes are discarded**; the kernel delivers one `ALL` batch per block on `LOADING` to `RUNNING` (ADR-0003 § 4, ADR-0004).
- **Determinism** (kernel G1 to G4, ADR-0005): order is a pure function of registration data, iteration is over sorted arrays, and a snapshot restored at any tick must yield the same `state_hash` trace as the uninterrupted run (G4).
- **No clamp, no repair at restore** (Save/Load SL-15 items 5 and 6; each owner GDD): a value outside its structural range is a reported failure.
- **`MAX_BLOCKS` is a registry capacity** (kernel GDD); exceeding it must fail boot (TR-cmd-035).
- **No pre-restore state survives an owner failure** after the first block has been replaced (kernel GDD edge case; TR-assembly-038). The GDD calls an automatic revert out of scope.
- **Integer-only, no wall clock in any decision** (ADR-0005, ADR-0010 SIM-02): pass timings are logged through the injected `Clock` port and decide nothing.

### Requirements
- One numbered `LOADING` sequence for restore and one for new game, with the state each failure leaves.
- A compile-once, printable, deterministic block order that rejects cycles, unknown ids and an over-full registry at boot.
- A rule for what `restore` and `post_restore` may read and write that a test can falsify.
- A single place that lists every cross-block restore check and the order they run in, with one failure naming block and check.
- A binding-source contract the Body and Emitter can implement without the kernel reading their blocks.
- A generic idempotence harness (fixture factory per block) and a checkable exit for `REBASE_BLOCKS`.
- A guarantee that a faulted `LOADING` never reaches play (ADR-0011's cross-ADR assumption).

## Decision

### Architecture Diagram

```
registry freeze (first request_new_game / request_restore boundary, ADR-0003 § 8)   [new step 5 below the hook schedule]
   RestorePlan: restore_after -> resolve ids -> dedupe -> Kahn (smallest registration index first) -> order[], rank[]
   boot codes: UNKNOWN_DEPENDENCY, RESTORE_CYCLE, BLOCK_LIMIT          printed by describe()

request_restore(payload, epoch, expected_hash, has_expected)            request_new_game(seed, epoch)
 R0 flush queue (Dropped(RELOAD))                                        N0 flush queue
 R1 PRE-CHECK (pure, whole payload, no restore() called)                 N1 epoch range
    fail -> FAULTED, every block, clock, half (b) intact
 R2 CLOCK install (T, world_seed, epoch); clear half (b)                 N2 CLOCK install (T=0, seed, epoch); clear half (b)
 R3 for b in order, WORLD persist blocks:                                N3 for b in order, every non-PROFILE block:
      payload has b -> restore(data, v, ctx)                                  default_state(ctx) -> ctx.put -> restore(put, v, ctx)
      absent        -> default_state -> restore(put)                    N4 for b in order with init_world: init_world(seed, INIT scope, ctx)
 R4 for b in order, WORLD persist + non-persist blocks:                  N5 for b in order: post_restore(ctx)
      post_restore(ctx)   [verifiers run here, § 5]
 R5 VERDICT  state_hash(PERSIST, T) vs expected (diagnostic only)        -
 R6 REBIND   clear half (b); sources in ascending String(block_id)       N6 REBIND (same)
 R7 EXIT     load_seq += 1; one ALL batch per block; snap_baseline; RUNNING (Dropped queue stays empty)

 any fault in R3, R4, N3..N5, R6/N6 -> stop at once; kernel FAULTED(LoadFailure{kind, stage, block, code, detail});
 no tick, no command admitted, no snapshot, no ALL batch, load_seq unchanged; exits only via request_restore / request_new_game
```

### 1. The `LOADING` pass: one numbered sequence

The pass is the kernel's, runs on the main thread inside one boundary service, and is driven by the compiled `RestorePlan` (§ 2). Stage names are closed constants (`LoadStage`): `FLUSH`, `PRECHECK`, `CLOCK`, `RESTORE`, `DEFAULT`, `INIT_WORLD`, `POST_RESTORE`, `VERDICT`, `REBIND`, `EXIT`.

**1.1 Restore pass (`request_restore`; legal from `BOOTING` after the freeze validation, `RUNNING`, `PAUSED`, and `FAULTED` when `registry_valid`).**

| # | Stage | What the kernel does | A fault here |
|---|---|---|---|
| R0 | `FLUSH` | Every queued command receives `Dropped(RELOAD)` (ADR-0002 § 5). | Cannot fail. |
| R1 | `PRECHECK` | A pure function over the whole payload and the registry, **before any `restore()`, `default_state()` or kernel write**. The visit order is fixed so the first failure is a pure function of the payload (§ 5.5): (a) every payload key, **converted to `String` and visited in ascending `String` order** (`CanonicalEncoder.sorted_string_keys`; never `Dictionary` iteration order, which is decoder order), names a registered, persisted block that is restorable in this pass, and `kernel.clock` is treated as known (`RESTORE_UNKNOWN_BLOCK`, domain 0 code 4; an in-session pass restores `WORLD` blocks only, so a `PROFILE` or non-persisted id is unknown here); (b) the `kernel.clock` entry is present with a `tick_rate` equal to `TICK_RATE` (`TICK_RATE_MISMATCH`, code 5); (c) `restore_epoch` in `[0, 2^32 - 1]` (`RESTORE_EPOCH_RANGE`, code 6); (d) walking `RestorePlan.world_order`, each block the payload carries has its stored `schema_version` in its `supported_versions` (`RESTORE_VERSION_UNSUPPORTED`, **new code 20**, foundation ask; without it only an owner could fault a version mismatch, after the swap). Checks run in that order; (a) follows ascending `String` payload-key order and (d) follows the registry DAG order, and the first failure named is the first by that order (KRN-RST-03 has a two-bad-blocks case). | `FAULTED` (trigger `load_failed`). **Pre-restore state is intact**: no block, the clock, half (b) of the binding table and the epoch are untouched; only the command queue was flushed. Save/Load's own pre-check (ADR-0006 § 1.10) makes this unreachable for a file; it is the backstop. |
| R2 | `CLOCK` | Install `T` and `world_seed` from the `kernel.clock` entry and `restore_epoch` from the argument (ADR-0003 § 10); **clear binding half (b)**. From here `kernel.clock` (`T`, `world_seed`) is readable as Tier S (§ 3.4). `ClockView.tick()` reads the recorded `T`, the last completed tick at the snapshot boundary, throughout R3 and R4; the first `RUNNING` tick is `T + 1` (kernel GDD tick definition; ADR-0001 driver boundary; the increment semantics of `T` are the kernel GDD's). `changed_region` after a restore is advisory; `version_epoch` is authoritative (ADR-0014, ADR-0017). | Cannot fail (R1 validated the inputs). |
| R3 | `RESTORE` | For each block of `RestorePlan.world_order` (persisted, `WORLD`, in DAG order): if the payload carries it, `restore(data, stored_version, ctx)` in a `RESTORE` scope; if it does not, `default_state(ctx)` then the block's own `restore` on the payload it `put` (§ 3.3, REBASED cause 3 is Save/Load's to report). | `FAULTED` at once; **blocks earlier in the order are already replaced, this block and later ones are not**. The failing block's own contents are untouched (its `restore` is two-phase, § 3.1). |
| R4 | `POST_RESTORE` | For each block of `RestorePlan.post_order` (every persisted `WORLD` block and every non-persisted block, **in the same DAG order**), `post_restore(ctx)` in a `RESTORE` scope for that block only. Cross-block verifiers run here (§ 5). | `FAULTED` at once; **every WORLD block is replaced**; derived state is rebuilt for the blocks before the failing one only. |
| R5 | `VERDICT` | Only if `has_expected_hash`: recompute the uncached `state_hash(PERSIST, T)` (ADR-0005 § 4.7) and record `MATCH` or `MISMATCH`; otherwise `NONE`. | Never faults; a mismatch is a diagnostic (kernel CR-15). |
| R6 | `REBIND` | Rebuild binding half (b) from the binding sources (§ 6). | `FAULTED` (`BINDING_REBUILD_CONFLICT`); half (b) stays empty. |
| R7 | `EXIT` | `load_seq += 1`, then one `ALL` batch per block (ADR-0004), `snap_baseline` for interpolated blocks, `RUNNING` (ADR-0001 § 4), entered through the `load_done` trigger (ADR-0003 § 9). The driver clears its accumulator. | Handler faults are counted, never fatal (ADR-0003 § 3). |

**Pass kind.** At the MVP only the in-session pass exists (zero `PROFILE` blocks). The cold-open pass kind (`IN_SESSION` or `COLD_OPEN`), which also visits `PROFILE` blocks in R3 and R4, is deferred to the story that adds the first `PROFILE` block; that story adds the kind to the `request_restore` request (an ADR-0002 foundation ask at that time) and compiles a cold-open view from the same `order`. `world_order` and `post_order` are the in-session views.

**1.2 New-game pass (`request_new_game`; TR-cmd-037).** The same spine with the restore replaced by construction:

| # | Stage | What the kernel does |
|---|---|---|
| N0 | `FLUSH` | As R0. |
| N1 | `PRECHECK` | `restore_epoch` range only (`RESTORE_EPOCH_RANGE`). |
| N2 | `CLOCK` | `T := 0`, install `world_seed` and `restore_epoch`; clear binding half (b). |
| N3 | `DEFAULT` | For each non-`PROFILE` block in DAG order: `default_state(ctx)`; a persisted block's `put` payload is applied through its own `restore` (§ 3.3); a non-persisted block resets its derived members. |
| N4 | `INIT_WORLD` | Under **one** `INIT` `WriteScope` (any block writable), for each block that registered `init_world` in DAG order, `init_world(world_seed, scope, ctx)`. At the MVP this is Web Map's, which calls Layout, Graph, Emitter, Interaction and Suit mutators in its own fixed order (architecture § Initialisation order). |
| N5 | `POST_RESTORE` | As R4: every block, DAG order, same verifiers, so a freshly built world is held to the invariants a loaded one is. |
| N6 | `REBIND` | As R6. |
| N7 | `EXIT` | As R7 (`load_done` fires; the first `RUNNING` tick is `T = 1`). There is no verdict (no expected hash). |

`init_world` is called **only** from stage N4 and **never** in a restore pass; a counter on a recorded stub block asserts it (KRN-RST-02). `PROFILE` blocks are not visited by an in-session restore or a new game (Pillar 5, ADR-0006 § 1.8); a cold-open restore includes them in R3 and R4, and a `PROFILE` block that has been neither restored nor defaulted in this process is defaulted once at the process's first `LOADING`. Zero `PROFILE` blocks exist at the MVP.

**1.3 What a failed pass leaves behind (the guarantee ADR-0011 asks for).** On a fault in R3, R4, N3 to N5 or R6/N6 the kernel records `LoadFailure{kind, stage, block_id, code, detail}` in diagnostics, closes the scope (fail-closed, ADR-0003 § 5), calls no further owner callable of the pass, and enters `FAULTED` through the `load_failed` trigger (ADR-0003 § 9; a boot failure uses `boot_validation_failed`). In `FAULTED`:

1. **No tick runs and no command is admitted** (`NOT_ACCEPTING`); the queue was flushed at R0/N0 and stays empty.
2. **No `ALL` batch is delivered and `load_seq` is not incremented**, so no presentation subscriber is told to rebuild from the half-restored blocks; `PresentationClock.state` reads `FAULTED` and the Chart Table shows the load-failed screen from that.
3. **Nothing is written to disk**: the kernel never autosaves or accepts `request_snapshot` in `FAULTED` (kernel GDD), so a half-restored world can never overwrite a good generation.
4. **Binding half (b) is empty** (cleared at R2/N2 and not rebuilt), so even a test ingress that bypassed the queue would fail the authority check.
5. **The only exits are `request_restore` and `request_new_game`** (legal when `registry_valid`, ADR-0003 § 9). Each starts a fresh pass that replaces every `WORLD` block's contents and rebuilds every derived member, so nothing from the failed pass survives into the next. KRN-RST-04 proves it: after any injected fault, a second `request_restore` of a good payload yields the same `state_hash` trace as a fresh kernel.

The failure is reported to the caller in the restore `ControlOutcome`, which today carries only the verdict (foundation ask to ADR-0002): `load_ok: bool`, `fail_stage`, `fail_block: StringName`, `fail_code: int`, `defaulted_blocks: Array[StringName]`. Save/Load maps `load_ok == false` with a block-owned code to its `RESTORE_FAILED` class and falls to the next candidate generation by a new pass (ADR-0006 § 1.10); the pre-restore live world is not recoverable and is not needed, because the two MVP callers (Continue from the title screen, and the death stub whose world is already forfeit) both discard it.

**1.4 No automatic revert.** The kernel does not snapshot the live world before a pass and does not revert on failure. A revert would be a second restore pass that could itself fail, would double the cost of every load for a path with no caller, and would contradict the kernel GDD ("automatic revert to it is out of scope"). The optional pre-restore in-memory snapshot "so diagnostics can diff" stays an option behind a dev flag and is not part of the contract.

### 2. `restore_after`: the DAG, its order, its limits

**2.1 Meaning.** `BlockDescriptor.restore_after` (ADR-0003 § 2) lists the blocks that must be **restored and post-restored before this one, and initialised (`default_state`, `init_world`) before this one**. One relation orders all four callables. A block may list a non-persisted block; a `WORLD` block listing a `PROFILE` block is legal and the edge is simply not visited by an in-session pass.

**2.2 Compilation at the freeze.** In the registry validation pass (ADR-0003 § 8, after the hook schedule) the kernel builds the `RestorePlan`:

1. Resolve every id to the block's **registration index**; an unregistered id is `UNKNOWN_DEPENDENCY` (code 10, existing). A duplicate entry in one list is dropped with a boot warning `RESTORE_AFTER_DUPLICATE` (warnings must be empty at the end of every CI run, as for hooks).
2. Run Kahn's algorithm over registration indices. **The ready list is ordered by registration index and the smallest ready index is always taken**, exactly the rule of the hook sort (ADR-0003 § 8), so there is one algorithm and a total order that is a pure function of registration data. Dictionary iteration never decides anything.
3. If unprocessed blocks remain with no ready block (this includes a block listing itself), boot fails with `RESTORE_CYCLE` (**new code**) naming the remaining blocks in registration order.
4. Record `order` (block indices in restore order) and `rank` (position per index), and derive three views: `world_order` (persisted `WORLD` blocks), `post_order` (`WORLD` persisted plus non-persisted blocks) and `bind_order` (blocks with a `binding_source`, ascending `String(block_id)`, § 6). `bind_order` is built **once, here**, with ADR-0005's sanctioned helper `CanonicalEncoder.sorted_string_keys` (or `Array[String].assign(ids)` followed by `sort()`) over the ids as `String`, and mapped back to registration indices. No kernel code compares `StringName` values (forbidden pattern `stringname_sort_for_canonical_order`, ADR-0005 P12, ADR-0010 DET pack; probe P1 corroborates). The plan's fields are private and read through `order_at(i)`, `rank_of(index)`, `size()` and the matching `world_*`, `post_*` and `bind_*` accessors: element writes through a held `PackedInt32Array` call no setter and the language has no readonly, so the plan is **immutable by convention, checked by the hash of `describe()` taken before and after a run** (KRN-RST-01). It is printable (`describe()`), like the hook schedule. Tests install an alternative order through `RestorePlan.from_order`, which validates that the order is a linear extension of the DAG.

Registering more than `MAX_BLOCKS` blocks fails boot with `BLOCK_LIMIT` (**new code**) at the first excess registration (TR-cmd-035). The cost of the sort at `MAX_BLOCKS` is a one-time O(n²) over at most that many entries.

**2.3 The tie-break is load-bearing.** Probe P2 shows several valid orders of the same MVP graph under different tie-breaks. Any block that reads a non-predecessor's derived state would work under one and fail under another. That is the point of making the order a total function of registration data and of testing against other linear extensions (§ 8): the rule "a block depends only on what it declares" is enforced by the permutation test, not by trust.

**2.4 The MVP order** (computed 2026-10-29 by a throwaway script applying § 2.2 to the architecture's registration order, Layout, Graph, Body, Controller, Emitter, Suit, Interaction, Assembly, Web, Attention, Horror, Sanity; the kernel prints the authoritative order at the freeze and this table is illustrative):

| Pos | Block | `restore_after` (from the owner GDD or ADR) | Source |
|---|---|---|---|
| 0 | `kernel.clock` | (kernel-internal, installed at R2, not in the DAG) | ADR-0003 § 10 |
| 1 | `layout.state` | `[]` (the root) | ADR-0013 § 1 |
| 2 | `graph.state` | `[layout.state]` | ADR-0011 § 1 |
| 3 | `body.state` | `[layout.state]` | BDY-5 |
| 4 | `suit.state` | `[]` | SE-34 |
| 5 | `controller.state` | `[body.state, suit.state]` | ADR-0019 § 2.1 |
| 6 | `emitter.state` | `[layout.state, graph.state, body.state, controller.state]` | EM-1 |
| 7 | `interaction.state` | `[layout.state, controller.state, graph.state, body.state, emitter.state]` | IA-36 |
| 8 | `assembly.state` | `[layout.state, graph.state, interaction.state, body.state]` | MSA-4 |
| 9 | `web.state` | `[layout.state, graph.state, body.state]` | WM-2 |
| 10 | `attention.state` | `[graph.state, emitter.state]` | AT registration |
| 11 | `horror.state` | `[layout.state, graph.state, emitter.state, attention.state]` | HE-3 |
| 12 | `sanity.state` | `[]` | SN-1 |
| 13 | `sanity.ledger` | `[sanity.state, graph.state, layout.state, horror.state]` | SN-1 |
| 14 | `sanity.perception` | non-persisted; **assumed** `[sanity.state, sanity.ledger]` (the GDD gives none; Open Question 1) | SN-1 |

That is 13 persisted blocks plus the clock (the 14 of SL-19) and one derived block. The DAG is acyclic: Layout is the only root of the layout chain, Suit and Sanity have no outgoing edge to the Controller or each other, and the Controller's edges go only to Body and Suit (verified by reading the owner GDDs; neither Body nor Suit declares an edge back, ADR-0019's cross-ADR assumption).

### 3. The block contract at restore: two phases, three read tiers

**3.1 Two phases per block, restated once (ADR-0006 § 2.2 is the rule; this ADR adds the pass-level consequences).** `restore(data, stored_version, ctx)` decodes and validates into scratch (phase 1, cold path, allocation allowed, **no write to the live block**), then, on success only, copies the contents **by value** into the existing block object and rebuilds the block's *static* derived members (phase 2). A failure is `ctx.fault(<OWNER>_RESTORE_*, detail)` with the live block untouched. **`restore()` owns nothing in the payload.** `Packed*Array`, `Array` and `Dictionary` are reference types in GDScript (probe P5), so phase 2 assigns every `Packed*Array` from `.duplicate()` (or rebuilds it element by element) and deep-copies every nested container it keeps (`duplicate(true)`), as ADR-0011 § 7.2 does; a block must never retain a reference to `data` or to any of its members. Reasons: the live block would alias the payload that Save/Load or the single re-armed `SnapshotContext` put-slot still holds; a default payload shared with a static or const default would become mutable global state; and `FixtureFactory` results would be mutated by the round trip, hiding or faking an idempotence failure. **Scratch hygiene.** The scratch object contains no reference cycle: no self-referencing `Array` or `Dictionary`, and no lambda or `Callable` capturing `self` stored in a `RefCounted` member. `OBJECT_COUNT` sees Object cycles only (probe P3), so the container and lambda cases are guarded by the CNT-02 leg of KRN-RST-10 and by code review (§ 8).

**3.2 What `restore` may read.** Its own `data`, and **Tier S** reads of blocks in its transitive `restore_after` closure (§ 3.4). It writes only its own block. It never calls another owner's mutator, never derives simulation state from `T` (the clock is installed at R2 and is read-only; a structural check may compare persisted stamps with `T`, as the Graph's hook-cursor check does, V3; `restore_epoch` is readable only inside `RngFacet.draw_epoch`, ADR-0003 § 10), and never reads a non-predecessor.

**3.3 Defaults go through `restore`.** `default_state(ctx)` of a persisted block ends with `ctx.put(payload)` where `payload` has the shape and `schema_version` of a snapshot; the kernel applies it by calling **that block's own `restore`** with `stored_version = schema_version`. One copy path means the default is validated on every new game and every absent-block restore, a `default_state` that its own `restore` rejects fails the first test that builds a world, and the block's "registered-but-unrestored" hash (ADR-0011 § 4.6, ADR-0013 `counters` at 1) is what restore produces. If a persisted block's `default_state` returns without `put`, the pass faults with `DEFAULT_STATE_NO_PAYLOAD` (**new code**). For a non-persisted block `default_state` only resets derived members and `put` is not expected. **After `default_state` returns, the kernel moves the payload out of the `SnapshotContext` put-slot and clears the slot before calling `restore`**, so re-arming the same context never leaves a stale payload, and the owner's `restore` copies by value (§ 3.1) so the live block never aliases the default.

**3.4 Read tiers across blocks.** Every read-view method of every block is documented as one of two tiers (a `## @tier S` or `## @tier D` doc tag printed by the Debug Overlay; there is no lint for it and KRN-RST-05 is the only check):

- **Tier S (persisted-answerable).** Answerable from the block's persisted columns and the static derived members its `restore` rebuilt. Valid as soon as that block's `restore` returned. Layout's class 1 (`room_at`, `V`, `exposed_tiles`, `socket_plan`, `section_offset`, `body_of`, static rects; ADR-0013 § 10, TR-section-008), the Graph's `opening_into`, `seam_openings_into`, `section_counts_into`, `RoomSource` and `step_no()` (derived from the clock installed at R2, not from `hdr`), the Graph's `served_pm(id)` (read from the persisted `n_served`, CT-04) and its band reads (the persisted band memory `band_o2/p/t`), the Controller's chest and hand getters (read-time computation over persisted columns and `BodyView`; nothing derived is cached, CT-28), Body's `seat_bindings()`, `kernel.clock` (`T`, `world_seed`) from R2 onward, and any `ids below next_*` check are Tier S.
- **Tier D (derived).** Needs the block's own `post_restore`. Layout's class 2 (door-leaf rects, `clip_to_footprints`, `leaf_would_overlap`, `door_passage_into`), the Graph's intensive caches (`P`, `p_o2`, `o2_pm`, the pressure trend), topology caches and failure-descriptor store, the Emitter's static layer. Before the owner's `post_restore` ran, a Tier D read returns the template default (or neutral value) and raises the owner's `stale_*` diagnostic (Layout's `stale_leaf` is the first instance). This obligation applies only to the Tier D getters that a verifier or another block actually reads during the pass; other Tier D getters need no stale path, because nothing reads between the stages.

The rule that makes order matter and keeps it checkable: **in `post_restore(X)` the callable may read Tier S of any block and Tier D only of blocks that precede X in the DAG's closure.** `restore` is stricter: Tier S of closure predecessors only. No hook, command or presentation read happens between the stages, so a stale read can occur only inside the pass.

### 4. The `post_restore` contract

**4.1 Order.** `post_restore` runs in `post_order`, which is `order` filtered; it is the same DAG as `restore`. Reasons: architecture § Data Flow 7 already states it; Emitter Open Question 26 and ADR-0019 (Controller after Suit) need it; the cost is zero because the DAG exists; and "informational only" (Layout) means no rule depends on it, which is compatible. After this ADR Layout's `version`/`leaf_version` self-heal (LY-12 d) stays as belt and braces and is no longer what makes the Emitter's cold build correct for open doors.

**4.2 Write policy (closed).**

| Class | May `post_restore(X)` do it? | Rule |
|---|---|---|
| Rebuild X's **derived** members (caches, lookups, version counters, seeds of last-emitted values) | Yes | Writes no persisted column and draws no RNG. |
| Write X's **persisted** columns | **No**, except block X is on `REBASE_BLOCKS` (§ 9) | A normalisation belongs in `snapshot()`; see § 9. |
| Write any **other** block | **No** | `open_for` of another block in the `RESTORE` scope is `REFUSED` and a counted `WRITE_OUTSIDE_SCOPE` violation (ADR-0003 § 4). |
| Submit a command | **No** (impossible) | `SnapshotContext` has no `submit_system`. |
| Clamp, prune, repair or default a persisted value | **No** | A bad value is a fault; a well-formed but semantically dead reference restores as it is (TR-sanity-019). |
| Raise a fault | Yes | Only for a check of § 5; first failure wins. |

**4.3 Reconciliation is an ordinary hook rule, not a restore step.** Anything a block would like to "fix up after a load" (the Graph's stock reconcile, Interaction's subject window, Attention's beacon, Sanity's dead-subject prune) is performed by the owner's **next hook run under a normal `HOOK` scope**, and it must be **the same rule that runs every tick**, a no-op when state is consistent, with no `just_restored` flag. Reason: kernel G4 demands the same `state_hash` at every tick from a restored run and an uninterrupted run; a reconcile that exists only on the first post-restore run would diverge from the uninterrupted run whenever it does anything, and a reconcile that does nothing in the uninterrupted run is exactly an idempotent steady-state rule. KRN-RST-11 checks each owner's reconcile this way (600 ticks, hash per tick).

**4.4 Consequences for the cited rows.**
- **Layout/Graph stock reconcile (TR-itemdb-033).** The Graph keeps ADR-0011 § 7.3: a store above a lowered live `stock_cap` restores byte-identically (it is below the physical ceiling `max_stock_*`), and the clamp, the `discarded_*` credit, the `stock_clamped` count and the diagnostic naming node, old stock and cap run at the head of **every** phase 0 (a no-op in steady state; the first step after a load is simply the first that can find a store above the live cap). `post_restore` does not clamp (§ 4.2), so the K-4 verdict is `MATCH` and no REBASED cause is owed to ADR-0006. The Graph also persists its hook products (`n_served`, `n_draw`, CABLE `e0`, `power_use_w5`; ADR-0011 CT-04) so a restore landing between the power hook and the atmo or failure hook rebuilds none of them; `post_restore` re-derives P, p_o2, o2_pm and trend only (ADR-0011 § 7.4).
- **Catalogue tuning drift (TR-itemdb-032).** A `restore` or `post_restore` check is either **structural** (shape, range, id order, reference resolution, a physical ceiling) or it is not a restore check. A value that is a catalogue tunable (mass, `WEIGHT_HEAVY_G`, `GLOWSTICK_STACK_MAX`, `PATCH_CAP_BP`, `stock_cap`, a def's rated output) is never asserted, clamped, deleted or migrated at restore. A `def_id` that no longer resolves is a structural failure (`RESTORE_UNKNOWN_DEF`), and an alias renamed by ADR-0007's path resolves.
- **Interaction (TR-interact-016, -017, TR-boarding-005).** § 5 rows V10 and V11 and § 9: Interaction's `post_restore` may not write the Emitter or the Controller; `loose[].inst` decode is a verifier; channel and door-attempt dropping is an interim normalisation. **Subject model (closes Open Question 5).** Registrations are persisted and restore verbatim; a well-formed save needs no re-registration. `post_restore` only detects drift (V10, `IA_RESTORE_SUBJECT_DRIFT`) and writes nothing outside `interaction.state`; re-registering from a `post_restore` is refused (`open_for(emitter.state)` in a `RESTORE` scope is `REFUSED`, `WRITE_OUTSIDE_SCOPE`; Alternative 6). The repair is Interaction's ordinary hook rule through ADR-0017 E6/E7/E8, which ADR-0017 orders **after** `emitter.place`; its D2.5 pending columns absorb the difference, so the table reflects the repair at the following `emitter.place`, a lag of one tick at most. G4 is unaffected because a consistent save has empty drift.
- **Body (TR-shipbody-010).** `pos_prev := pos_curr` is a derived-member rebuild; `sub` is preserved (persisted, untouched); the trapped-actor probe counts and never faults; Body never rebuilds the kernel seat binding.
- **Attention (TR-attention-025).** `post_restore` rebuilds caches only; the beacon is reconciled by the first `attention.step` by comparing persisted `beacon_commanded` with Emitter state (§ 4.3); no cue fires on load because cues fire on transitions and the kernel's `ALL` batch carries no transition.
- **Sanity (TR-sanity-019).** `post_restore` must not prune; a well-formed dead `subject_ref` restores, is projected as it is and is invalidated by the next hook run.

### 5. Cross-block verification: where the checks live and the order they run in

**5.1 Decision.** Verifiers are **distributed, never kernel-held**. Each is a pure, read-only function in the owning block's `restore` or `verify` file, called from that block's `post_restore` (or `restore` where the check needs only Tier S of a predecessor), at the block's position in the DAG. The kernel only runs the callables in order and turns the first fault into `FAULTED`. This keeps H1 (no domain rule in the kernel), keeps the TR rows' wording ("in `post_restore`"), and needs no new registrar surface. The central artefact is **this ADR's numbered list**, which fixes what is checked and in which block's slot; the owner ADRs fix the check bodies.

**5.2 Verifier shape.** A verifier is `static func verify_restore(views: <Owner>VerifierViews, out: VerifyOut) -> int` returning `0` or a packed fault code of the owner's domain, with the invariant id and two ints written to the caller-owned `out` (no allocation; a Class A shape in the sense of ADR-0003 § 5). `<Owner>VerifierViews` is a plain `RefCounted` holding the read-view facades the check needs (SIM views only, never a `PERCEPTION_OUT` block), built by the composition root at `setup`. The block's `post_restore` does `var rc := X.verify_restore(views, out); if rc != 0: ctx.fault(rc, out.detail()); return ctx.complete()`. The same function is reused unchanged by runtime assertions (Assembly's release subset, § 5.4). Fault names follow `<DOMAIN>_RESTORE_<REASON>` for a `restore` check and `<DOMAIN>_VERIFY_<INV>` for a cross-block `post_restore` check, both in the owner's static domain (ADR-0003 § 6).

**5.3 The numbered list, in DAG order.** (Details of each check live in the cited owner document; "T" marks a check that reads Tier S only, so it is order-independent.)

| # | Slot (block, stage) | Check | Reads | Fault family | Owner |
|---|---|---|---|---|---|
| V1 | `layout.state` restore | Version, key sets, `template_id` resolves, `rot_q` 0, drift policy, dense ids and counters, same-body and weld/socket/forest consistency | own data, catalog | `LAYOUT_RESTORE_*` (4) | ADR-0013 § 10 |
| V2 | `layout.state` post_restore | Every `graph_ref_base` resolves (template-range counts match through `section_counts_into`; STUB rows of a RAGGED cut and seam edges are excluded; a discarded section's template range is all tombstones; ADR-0011 § 9.5); every `welds[].seam_id` resolves to a Graph seam; `graph.next_section_id == layout.next_section_id`; an `area` above `area_max` found here is `LAYOUT_RESTORE_DANGLING` | Graph T (`section_counts_into`, `seam_openings_into`, `opening_into`) | `LAYOUT_RESTORE_*` (4) | ADR-0013 § 10 (d); TR-section-009 |
| V3 | `graph.state` restore | The thirteen ordered steps of ADR-0011 § 7.2, including step 13, the hook-cursor check: with `T_last` from the clock installed at R2 and the stored `hdr.sched`, each `last_step_X` equals the step of the last tick at or before `T_last` on which hook X was due (the ADR-0012 § 1 step-0 value when none), and `last_step_no == last_step_failure`; failure is `RESTORE_HOOK_CURSOR` (Graph local code 26; local 25 is the runtime `HOOK_CURSOR` that ADR-0012 raises). The check is structural and byte-identical: it compares persisted stamps with `T`, asserts no catalogue tunable (§ 4.4), and a schedule change across a save is absorbed at the first hook run (ADR-0011 § 7.4) | own data, catalog, `kernel.clock` `T` (Tier S) | `GRAPH_RESTORE_*` (3) | ADR-0011 § 7.2, ADR-0012 § 1 |
| V4 | `graph.state` post_restore | Compartment `room_ref` resolves through `RoomSource` (`RESTORE_UNKNOWN_ROOM`); `n_o2 + n_inert` within `P_MAX x V x GAS_SCALE`; LS-24 structural invariants | Layout T and own | `GRAPH_RESTORE_*` (3) | ADR-0011 § 7.4 |
| V5 | `body.state` restore | Types, counts, row fields, order, canonicity, then every `body_id` live in Layout (`BDY_RESTORE_UNKNOWN_BODY`) | Layout T | `BDY_RESTORE_*` (5) | BDY-31, ADR-0015 |
| V6 | `body.state` post_restore | Repeat of V5's liveness as a diagnostic only; trapped-actor probe counted, never a failure | Layout T | counter | TR-shipbody-010 |
| V7 | `suit.state` post_restore | Held and carried instances decode with `from_array(a, graph_step_no)` (`graph_step_no` = `GraphView.step_no()`, derived from the kernel clock installed at R2 and **not** from `hdr`, whose cursors can trail by up to one step, ADR-0011 § 2.3; Tier S); wrong length or element type `SUIT_RESTORE_INSTANCE_FORM`; `validate_instance` failure; `controller.state.carry_ref == suit.state.carried_id` or both NONE (`SUIT_RESTORE_CARRY_MISMATCH`); seat row has a Body seat (`_SEAT_UNBOUND`) | Graph T, Controller T, Body T, Item DB | `SUIT_RESTORE_*` (9) | SE-13, TR-suit-012, TR-boarding-005, ADR-0027 |
| V8 | `controller.state` post_restore | Derived recompute of `cap_seen` only (the chest and hand points are computed at read time from the same tick's `pos_curr`, ADR-0019 § 2.4, and nothing is cached); no verifier | Suit D (Suit precedes) | none | ADR-0019 § 2.7 |
| V9 | `emitter.state` post_restore | `emitter.state` restore (`EmitterVerify`) plus `post_restore`: rebuild every derived structure in ascending id including the whole static layer; re-derive `body_id` from Layout and Body; copy each subject's `lit` bit verbatim; the pending set equals the live run's (ADR-0017 D2.5 makes pending subjects legal at the first validate) | Layout D (leaves), Graph D, Body, Controller | `EMITTER_RESTORE_*` (6) | EM-1, TR-emitter-008, ADR-0017 |
| V10 | `interaction.state` restore/post_restore | Each `loose[].inst` decodes with `from_array` and passes `validate_instance` (`INTERACTION_RESTORE_LOOSE_INSTANCE`); rebuild the candidate table and window `W` from restored `pos_curr`; **compare** the Emitter's registered subjects with `W` and record `IA_RESTORE_SUBJECT_DRIFT` as a diagnostic if they differ (`post_restore` re-registers nothing and writes nothing outside `interaction.state`; registrations are persisted and restore verbatim; the repair is Interaction's first ordinary hook run, which ADR-0017 orders after `emitter.place`, reflected at the following `emitter.place`; § 4.3, § 4.4) | Graph T, Body T, Emitter T | `INTERACTION_RESTORE_*` (8) | IA-36/37, TR-interact-016/-017, ADR-0020 |
| V11 | `interaction.state` post_restore (**proposed**) | The id in `controller.state.carry_ref` is below Interaction's instance-id counter and is not also a key of `loose[]` (one holder, Pillar 1). Not in Interaction's closed reason list today (Open Question 3). | Controller T, Suit T | `INTERACTION_RESTORE_*` (8) | ADR-0019 § 2.7 hand-off |
| V12 | `assembly.state` post_restore | INV-1, INV-2, INV-3, INV-4, INV-5, INV-6 (`MAX_BODIES` hard; `BODIES_LIVE_MAX` half warn-only, `MSA_BODIES_LIVE_WARN`), INV-7 and INV-10 (vacuous at restore: stored `pending` is NONE); first failing id names the fault. INV-4 is quoted as 'at most one Body pose row per body, and every row's body is live' (a sparse zero pose row is legal); this ADR assumes ADR-0021 adopts that wording in its invariant bodies and in the weld property test before V12's fixture row lands in KRN-RST-12 | Layout, Graph, Body, Interaction (all Tier S) | `ASM_VERIFY_INV_<n>` (10) | MSA-32 T7, TR-assembly-009, ADR-0021 |
| V13 | `web.state` post_restore | `p1 == p0 + v*n`, berth exact-sum, origin/target consistency, `leg_reversed`, epochs and loaded set; in LEG or ARRIVING `body_of_actor(traveller)` is the pod body; reject with `WEB_RESTORE_*`, never clamp or repair; submits nothing (impossible) | Layout T, Body T | `WEB_RESTORE_*` (11) | WM-16, TR-webmap-022, ADR-0026 |
| V14 | `attention.state` restore/post_restore | Field ranges, `rng_counter <= tell_epoch`, locus ids resolve against Graph and Emitter; rebuild caches only | Graph T, Emitter T | `ATTENTION_RESTORE_*` (12) | AT, TR-attention-025, ADR-0022 |
| V15 | `horror.state` restore/post_restore | Closed manifest of fields, place and nav ids resolve | Graph T, Emitter T, Attention T | `HORROR_RESTORE_*` (13) | HE-41, ADR-0023 |
| V16 | `sanity.state` / `sanity.ledger` restore/post_restore | Closed `SANITY_RESTORE_*` reasons; a **malformed** `subject_ref` (unknown kind, id outside range) fails, a well-formed dead one restores | Graph T, Layout T, Horror T | `SANITY_RESTORE_*` (14) | SN-restore, TR-sanity-019, ADR-0025 |

**5.4 Runtime reuse.** Assembly runs the same function (`ASM_VERIFY`) after each weld or cut apply: in debug and CI builds all of INV-1..INV-7 and INV-10 as an assertion, in release INV-1, INV-2, INV-4, INV-5 and INV-7 only, each with a `subset_mask` argument and the work bound `O(seams + bodies + actors)` that MSA-32 states. A failure there is `ctx.fault` of the apply (kernel `FAULTED` naming the invariant); the call sites and costs are ADR-0021's and the weld property test's (architecture § Spikes that gate ADRs). This ADR's contribution is that the *post-restore* use is the same code and runs at Assembly's DAG slot, after the four blocks it reads.

**5.5 First failure wins, deterministically.** Within a pass the DAG order, the block-internal check order and "stop at the first fault" make the reported `(stage, block, code)` a pure function of the payload (R1 visits payload keys in ascending `String` order and the registry in `RestorePlan` order, § 1.1; no `Dictionary` iteration order decides); the owners' mutation-fuzz tests (for example the Graph's 1,000-mutation fuzz) rely on it, and KRN-RST-12 asserts it on a numbered fixture table with one corrupt fixture per verifier above.

**5.6 When this is not enough.** If a verifier ever needs the Tier D state of a block that does not precede its own, there are two moves, in this order: declare the missing `restore_after` edge (usually free), or add a separate kernel verification stage after R4 with a registrar surface for verifier callables (Alternative 3). The second is deliberately not built now.

### 6. The binding rebuild (half (b))

**6.1 Registration.** A block that owns an entity a PlayerId acts as registers `BlockDescriptor.binding_source` at `BOOTING` (ADR-0003 § 2; the binding source is this field only). At the MVP: `body.state` (`seat_bindings() -> [(player_id, actor_id)]`, ascending `player_id`, `SEAT` rows only, derived per call; at the MVP seat == PlayerId, ruling R3; TR-shipbody-030) and `emitter.state` (`beam_bindings() -> [(player_id, beam_id)]`, ascending `player_id`, pure read; TR-emitter-009). `controller.state` is not a source: it stores the PlayerId and resolves the actor through Body (ADR-0019 § 2.1).

**6.2 Payload form.** `binding_source(ctx: SnapshotContext) -> CompletionToken` delivers through `ctx.put(pairs)` one **flat `PackedInt64Array`** `[player_id_0, entity_id_0, player_id_1, entity_id_1, ...]`, strictly ascending by `(player_id, entity_id)`, so no per-pair `Array` is allocated. The callable is a pure read valid at any boundary: it receives no `WriteScope`, builds a **new `PackedInt64Array` on every call** (never a member or cached array: `ctx.put` hands the kernel a reference, and a returned member would let the kernel read live state), and is called by the kernel at R6/N6 after every `post_restore`, where it reads rebuilt state, and by ADR-0002's `STRICT_DEBUG` `BINDING_DRIFT` check during `RUNNING`.

**6.3 The rebuild.** At R6/N6, after the verdict and as the last step of every pass, the kernel iterates `bind_order` (ascending **`String(block_id)`** of the source, fixed once at the freeze by `CanonicalEncoder.sorted_string_keys`, § 2.2; the kernel compares no `StringName`, pattern `stringname_sort_for_canonical_order`) and for each source: calls it, validates the array (even length; `player_id >= 1`; strictly ascending), and binds each pair as `player_id -> (source block, entity_id)` into the `PlayerBindings` object (ADR-0002 § 4). A key `(source block, entity_id)` delivered for two different PlayerIds, or the same `(player_id, source, entity_id)` delivered twice, or a malformed array, or a source that called `ctx.fault` or returned without the token, is `BINDING_REBUILD_CONFLICT` (domain 0 code 3) with detail `{reason, source, player_id, entity_id}` where `reason` is one of `CONFLICT_OWNER`, `DUPLICATE_TRIPLE`, `SHAPE`, `ORDER`, `PLAYER_RANGE`, `SOURCE_FAULT`. Equal entity numbers from two sources (Body actor 1, Emitter beam 1) are two keys and never conflict; comparisons are within one source. The rebuild is not a control-plane request and no command can interleave (the pass is one boundary service).

**6.4 Consequences.** Half (b) is never saved, never read from the payload and never written by a block (TR-saveload-037); the first `RUNNING` tick passes the authority check for PlayerId 1 against `(body.state, avatar actor)` and `(emitter.state, beam)` (KRN-RST-06 submits an `OWNED_BY_ISSUER` command on tick 1 of a restored run); Body's `post_restore` does not rebuild it (TR-shipbody-010); a fault leaves half (b) empty (§ 1.3). Half (a) (`peer_id -> PlayerId`) is never touched by a pass. A restored run never calls Emitter E9 `create_seat_beam` or Suit's E4 `bind_beam`; those run only under `init_world` (Web Map, INIT scope: E1 per section, E9 per bound seat, E4, then E10; ADR-0017, ADR-0027), and a restored run's binding comes from `seat_bindings` and `beam_bindings` through `binding_source`.

### 7. Pass timing and diagnostics

Per block and stage the kernel records `t_restore(b)` and `t_post(b)` through the injected `Clock` port (`now_us()`, ADR-0006 § Constraints, SIM-02 applies: logged, never branched on), plus `t_verdict` and `t_rebind`, into the `LoadTiming` diagnostics record. The sum is the `loading_ms` of the kernel GDD's load-budget formula; the per-owner figures (Graph `restore` plus `post_restore` at most 2 ms; the MVP world at most 3 ms with a 10 ms ceiling, TR-lsg-036) are checked by KRN-RST-14 against the registry names `LOADING_TARGET_MS` and the Graph's, reported and never a cut-off. A pass above its owner's figure is a diagnostic, like `STEP_BUDGET_MS`.

### 8. Tests that make the rules falsifiable

- **Order-permutation (G4) test, KRN-RST-05.** Two levels. **Stub level (gates acceptance; runs on the KRN-LAT-02 skeleton):** recorded stub blocks with the MVP `restore_after` edges are restored under at least four **other valid linear extensions** of the DAG (smallest id first, largest id first, reverse registration, and a shuffled-ready-list extension built by a **test-owned Fisher-Yates driven by `DetRng` (ADR-0005)**, never `Array.shuffle()` or `randi`, which use the global RNG that DET-01 bans), each installed through `RestorePlan.from_order`, which validates that the order is a linear extension. A planted stub that reads a non-predecessor's Tier D state must diverge under at least one extension; a stub that respects its edges must match under all four. **World level (a Save/Load or Kernel epic closure gate, not an acceptance gate):** for a fixed seeded world, snapshot at tick `T`, restore into fresh kernels under the alternative extensions, run 120 ticks each, and compare the per-tick `state_hash` trace, the per-tick `ControllerView` samples (`block_hash` cannot see the Controller's normalised fields) and a battery of read-view answers (`lit`, `top_emitter`, `room_at`, `served_pm` (a Tier S read of the persisted `n_served`), door-leaf rects, seat bindings) with the uninterrupted run. **Input model.** The Adapter's resync submission is applied in the first `RUNNING` tick's P2 (ADR-0019 § 8.5); a G4 fixture either snapshots from a rest Controller intent or replays the resync submission into the restored run. **Late-hook fixtures (ADR-0017 D2.5).** The world level includes fixtures that issue Emitter E3, E6, E7 and E8 on the snapshot tick from a hook ordered after `emitter.place`, an Interaction M6 on a lit lamp's node on the snapshot tick, and a late hook that sets `loco_state` to DEAD on the snapshot tick; each must give an equal trace after restore. A block that read a non-predecessor's unbuilt derived state, or depended on the wiring order, fails here.
- **Reconciliation symmetry, KRN-RST-11** (§ 4.3), per owner that defers a reconcile.
- **Fault ladder, KRN-RST-04.** For every stage and every block of the MVP fixture a stub injects `ctx.fault`; the test asserts the recorded `LoadFailure`, that no later callable ran, that no tick, `ALL` batch or `load_seq` increment happened, that `FAULTED` rejects commands and snapshots, and that a following good `request_restore` matches a fresh kernel.
- **No new lint pack.** The earlier RST pack is dropped. The `StringName` ordering fence is the existing ADR-0005 pattern `stringname_sort_for_canonical_order` and the ADR-0010 DET pack (no `.sort_custom(` in `src/kernel/`; ids ordered only through `CanonicalEncoder.sorted_string_keys`); since no kernel code compares `StringName` values, nothing further needs linting. The cross-block write fence is the run-time `RESTORE` scope (KRN-RST-08); `init_world` callers are counted by KRN-RST-02; a `FixtureFactory` per persisted block is checked by KRN-RST-09 (a block without one fails the registry enumeration); container cycles are guarded by KRN-RST-10 through CNT-02. **Review rule, not a lint:** no self-referencing container in restore scratch, and no lambda or `Callable` capturing `self` stored in a `RefCounted` member.
- **Copy-by-value test, KRN-RST-15** (§ 3.1): after each block's `restore`, mutate the live block's packed columns and nested containers and assert that the payload bytes are unchanged.

### 9. Idempotence, fixtures and `REBASE_BLOCKS`

**9.1 Rule.** Every persisted block is restore-idempotent: `snapshot -> encode -> decode -> restore -> snapshot` yields equal canonical bytes (ADR-0006 § 2.2). The way to get there is **normalise in `snapshot()`, not in `post_restore`**: a transient field the block would clear on restore (an in-progress channel, a door attempt, a pending completion, a controller ramp) is written in its cleared form by `snapshot()` while the live block keeps its value, so the restore reproduces the saved bytes exactly and `post_restore` has nothing to write. Assembly (`pending` written NONE, TR-assembly-012) and the Controller (PC-17) already do this.

**9.2 `FixtureFactory` and the generic harness (TR-saveload-034).** Each persisted block registers, in the test tree only, a `FixtureFactory` with `empty() -> Dictionary` (equal to its `default_state` payload), `typical(seed) -> Dictionary` and `extreme(seed) -> Dictionary` (caps reached, tombstones present, counters near limits). Every method returns a **fresh** payload on each call (no shared static or const), because a block must not alias a payload (§ 3.1) and a mutated fixture would hide or fake an idempotence failure. `tests/unit/restore/idempotence_test.gd` **enumerates the registry** (never a hand list), and for every persisted block and fixture asserts the round trip, that `default_state` equals `empty()`, and that the block restores from its own `default_state` payload. A persisted block without a factory fails KRN-RST-09.

**9.3 `REBASE_BLOCKS` is a checked list, not a convention.** The build constant `REBASE_BLOCKS` (Save/Load, SL-12) is a committed array in `src/save/` with one data row per member in `tests/fixtures/rebase_blocks.json`: `{block_id, owner_adr, reason, exit}`. CI fails if (a) a block is idempotent under § 9.2 but listed (a stale entry) or non-idempotent but not listed; (b) a build tagged for an outside release has a non-empty list. The gate flips with the same project-level `SHIPPED` decision as ADR-0007's id ledgers and ADR-0006 § 2.6. **Interim members at the MVP: `interaction.state`**, whose `post_restore` drops in-progress channels, door attempts and `last_result` and resolves a RESERVED FULL-job part (IA-36; OQ-39). Its exit is ADR-0020's: move those clears into `snapshot()` (the saved payload carries the dropped form and the resolved part), after which `post_restore` only rebuilds `W`. An older, un-normalised payload is still handled by a tolerant restore, as a migration. No other MVP block needs the list: `layout.state` restore is byte-identical and never rewrites a stored hash; content drift (`geometry_changed_fresh`, `visual_changed`) is diagnostic only and re-raised on each load until the instance changes (ADR-0013 § 10), so it is not a standing member.

### 10. Resource lifetime across passes (TR-light-037, this ADR's half)

A pass must leave the object, node and orphan-node counts where the previous pass left them: scratch objects are acyclic and dropped at the end of the owner's `restore`; contexts, scopes, tokens and read-view facades are the kernel's and the owners' long-lived objects, re-armed rather than rebuilt (ADR-0003 § 3, § 7); `restore` copies into the existing block, so no facade or pool handle goes stale. KRN-RST-10 alternates `request_restore` and `request_new_game` 100 times on the headless kernel and asserts **deltas against cycle 1**, never absolute numbers (a baseline `OBJECT_ORPHAN_NODE_COUNT` of 1 appears even in a bare `--script` run): `OBJECT_COUNT`, `OBJECT_NODE_COUNT` and `OBJECT_ORPHAN_NODE_COUNT` after cycle *n* equal those after cycle 1, compared with `int()` casts because `get_monitor` returns float; and the ADR-0010 CNT-02 `MEMORY_STATIC` tolerance check after warm-up (debug and editor binaries only; release availability is UNVERIFIED). `OBJECT_COUNT` detects Object cycles only (probe P3): a self-referencing container or a lambda capturing `self` is caught by the `MEMORY_STATIC` leg and by the § 3.1 review rule. The orphan-node monitor is debug-binary only. Probe P4: the monitors return values headless but render monitors are 0 under the dummy renderer, so render-texture legs run in the windowed lane W (ADR-0010 § 8). ADR-0028 extends the same loop with the build, body-swap and render-texture legs and the light and occluder pools; this ADR guarantees only that the pass itself leaks nothing.

### Key Interfaces

```gdscript
# Each listed class is its own file: GDScript allows one class_name per file (ADR-0003 does the same for fault classes).
# Callers build typed arrays (Array[BlockDescriptor], Array[String]); a bare array literal is rejected for a typed-array parameter.

# src/kernel/restore_plan.gd  -- built once at the registry freeze, immutable by convention afterwards (ADR-0003 § 8, step 5)
class_name RestorePlan extends RefCounted
var _order: PackedInt32Array             # private; registration indices in restore order (all blocks)
var _rank: PackedInt32Array              # private; _rank[index] -> position in _order
var _world_order: PackedInt32Array       # private; persisted WORLD blocks, in order
var _post_order: PackedInt32Array        # private; WORLD persisted + non-persisted blocks, in order
var _bind_order: PackedInt32Array        # private; blocks with binding_source, ascending String(block_id) (CanonicalEncoder.sorted_string_keys, once)
func size() -> int
func order_at(i: int) -> int
func rank_of(index: int) -> int
func world_size() -> int
func world_at(i: int) -> int
func post_size() -> int
func post_at(i: int) -> int
func bind_size() -> int
func bind_at(i: int) -> int
static func compile(blocks: Array[BlockDescriptor], max_blocks: int, out_errors: Array[String]) -> RestorePlan   # null + out_errors on failure
static func from_order(order: PackedInt32Array, blocks: Array[BlockDescriptor], out_errors: Array[String]) -> RestorePlan   # test seam: null unless `order` is a linear extension of the DAG
func describe() -> PackedStringArray     # printable; the Debug Overlay and the boot log; its hash is the immutability check

# src/kernel/load_stage.gd
class_name LoadStage extends RefCounted
const FLUSH: int = 0
const PRECHECK: int = 1
const CLOCK: int = 2
const RESTORE: int = 3
const DEFAULT: int = 4
const INIT_WORLD: int = 5
const POST_RESTORE: int = 6
const VERDICT: int = 7
const REBIND: int = 8
const EXIT: int = 9

# src/kernel/load_failure.gd  -- diagnostics record; surfaced in the restore ControlOutcome (foundation ask)
class_name LoadFailure extends RefCounted
var kind: int                                        # RESTORE | NEW_GAME
var stage: int                                       # LoadStage
var block_id: StringName
var code: int                                        # packed FAULT_CODE (ADR-0003 § 6)
var detail: Dictionary                               # cold path; ADR-0010 dictionary_variant.json entry (foundation ask)

# src/kernel/load_timing.gd  -- diagnostics record; logged through the injected Clock port, never branched on (§ 7)
class_name LoadTiming extends RefCounted
var t_restore_us: PackedInt64Array                   # per registration index
var t_post_us: PackedInt64Array                      # per registration index
var t_verdict_us: int
var t_rebind_us: int

# src/shared/restore/verify_out.gd  -- caller-owned out record for verifiers
class_name VerifyOut extends RefCounted
var inv_id: int
var a: int
var b: int
func detail() -> Dictionary                          # cold path, built only on a failure; dictionary_variant.json entry

# Owner side (per block; ADR-0003 callable shapes unchanged)
#   restore(data: Dictionary, stored_version: int, ctx: SnapshotContext) -> CompletionToken   # copies by value (§ 3.1); payload param is a dictionary_variant.json entry
#   post_restore(ctx: SnapshotContext) -> CompletionToken
#   default_state(ctx: SnapshotContext) -> CompletionToken      # persisted: ends with ctx.put(payload); the kernel moves it out and clears the slot before restore
#   init_world(world_seed: int, scope: WriteScope, ctx: SnapshotContext) -> CompletionToken
#   binding_source(ctx: SnapshotContext) -> CompletionToken     # ctx.put(a NEW PackedInt64Array [player_id, entity_id, ...] per call)
# Owner verifier convention:
#   static func verify_restore(views: <Owner>VerifierViews, out: VerifyOut) -> int   # 0 or a packed fault code

# Test side (tests/unit/restore/)
class_name FixtureFactory extends RefCounted         # one per persisted block; every method returns a fresh payload
func empty() -> Dictionary
func typical(seed: int) -> Dictionary
func extreme(seed: int) -> Dictionary
```

New kernel fault codes (domain 0, local numbers continuing ADR-0003 § 6's table after `MEMORY_BUDGET` 17; foundation ask, never renumbered), stated once: `18 RESTORE_CYCLE` (boot), `19 BLOCK_LIMIT` (boot), `20 RESTORE_VERSION_UNSUPPORTED` (pre-check), `21 DEFAULT_STATE_NO_PAYLOAD` (pass). That is four new codes (the earlier "five" counted the pass codes and the outcome record separately). `RESTORE_UNKNOWN_BLOCK` (4), `TICK_RATE_MISMATCH` (5), `RESTORE_EPOCH_RANGE` (6), `UNKNOWN_DEPENDENCY` (10) and `BINDING_REBUILD_CONFLICT` (3) already exist, so `BINDING_REBUILD_CONFLICT` is not new. New boot warning: `RESTORE_AFTER_DUPLICATE`.

**Foundation asks (the Foundation ADRs are not edited here).** (1) ADR-0003 § 6: codes 18 to 21 and the boot warning. (2) ADR-0003 § 10: the H1 allow-list is extended by exactly `restore_plan` (registries group) and `load_failure` and `load_timing` (diagnostics group). (3) ADR-0002: the restore `ControlOutcome` gains `load_ok`, `fail_stage`, `fail_block`, `fail_code`, `defaulted_blocks`; the cold-open pass kind joins the request with the first `PROFILE` block. (4) ADR-0010: `dictionary_variant.json` entries for `LoadFailure.detail`, `VerifyOut.detail()` and the `restore(data: Dictionary, ...)` payload parameter of owner callables in `src/kernel` and `src/shared` (SIM-09). (5) Registry: ADR-0016 joins `referenced_by` of the existing `stringname_sort_for_canonical_order` pattern.

### Implementation Guidelines
- **Must** drive restore, `post_restore`, `default_state` and `init_world` from the one compiled `RestorePlan`; **must never** iterate a `Dictionary` or the registry's insertion order to order them.
- **Must** build `bind_order` once at the freeze through `CanonicalEncoder.sorted_string_keys` (or `Array[String].assign(ids)` then `sort()`), mapped back to registration indices; **must never** compare, `sort()` or `sort_custom` `StringName` values in the kernel (pattern `stringname_sort_for_canonical_order`, ADR-0010 DET pack; probe P1).
- **Must** visit R1 check (a) over the payload keys converted to `String` in ascending order and checks (b) to (d) over the registry in `RestorePlan` order; **must never** let `Dictionary` iteration order name the first failure.
- **Must** run the pre-check over the whole payload before any owner callable or kernel write; **must never** call a `restore()` on the strength of a partially checked payload.
- **Must** stop the pass at the first fault and enter `FAULTED`; **must never** continue to later blocks, deliver an `ALL` batch, increment `load_seq`, or accept a snapshot after a faulted pass.
- **Must** apply a `default_state` payload through the block's own `restore`, moving the payload out of the `SnapshotContext` put-slot and clearing the slot before `restore` is called; **must never** let `default_state` write the live block directly.
- **Must** make `restore` copy by value (`.duplicate()` for every `Packed*Array`, `duplicate(true)` for nested containers, or element-wise rebuild); **must never** retain a reference to `data` or any member of it. `FixtureFactory` methods and `binding_source` each return a fresh array or payload per call.
- **Must** make `post_restore` rebuild derived members only (§ 4.2); **must never** clamp, prune, repair, default or write a persisted value or another block from it, except a `REBASE_BLOCKS` member's documented normalisation.
- **Must** express every after-load fix-up as the owner's ordinary hook rule that is a no-op on consistent state (§ 4.3); **must never** key behaviour on "first run after restore".
- **Must** read only Tier S of non-predecessors and only closure predecessors in `restore`; **must** tag every read-view method S or D in its doc comment (documentation, printed by the Debug Overlay; KRN-RST-05 is the check); **must** give a Tier D getter a neutral value and a `stale_*` diagnostic only if a verifier or another block reads it during the pass.
- **Must** derive nothing in `restore` from `T`; a structural check that compares persisted stamps with `T` (the Graph's hook cursors) is allowed.
- **Must** put verifier code in `*_verify.gd` as a pure `static` function returning 0 or a packed code and reuse it for runtime assertions; **must never** put a cross-block rule in the kernel.
- **Must** deliver binding pairs as a strictly ascending flat `PackedInt64Array`, newly built per call, from a pure read valid at any boundary; **must never** write the binding table, rebuild it from a block, or save it.
- **Must** keep restore scratch objects free of reference cycles (no self-referencing container; no lambda or `Callable` capturing `self` stored in a `RefCounted` member) and drop them before `restore` returns; **must** build typed arrays (`Array[BlockDescriptor]`, `Array[String]`) at the `RestorePlan` call sites.
- **Must** register a `FixtureFactory` for every persisted block and keep `REBASE_BLOCKS` equal to the measured non-idempotent set.
- **Must never** call `init_world` outside the pass driver, or in a restore pass.

## Alternatives Considered

### Alternative 1: `post_restore` order follows registration order, or is unordered
- **Description**: Run every `restore` in DAG order but every `post_restore` in registration order (or in no stated order, the Layout GDD's "informational only" reading), relying on each owner's self-heal.
- **Pros**: Nothing new to specify; works if no block reads another's derived state.
- **Cons**: Emitter Open Question 26 and ADR-0019 need the order; correctness of the Emitter's cold build for open doors would rest on a version-bump self-heal only; the dependency would be invisible and break on a wiring change.
- **Rejection Reason**: The DAG already exists and the order is free; stating it makes the dependency declared and testable (§ 8).

### Alternative 2: Tie-break by `String(block_id)` instead of registration index
- **Description**: Kahn with the smallest `block_id` taken first, so the order is a function of the descriptors alone and does not depend on the composition root's wiring.
- **Pros**: Independent of wiring order; reproducible from a saved registry list.
- **Cons**: A second ordering algorithm beside ADR-0003 § 8's hook sort; the architecture's registration order (Layout, Graph, Body, ...) is already the natural dependency order, so the result is less intuitive to print and review.
- **Rejection Reason**: One algorithm and one total order for hooks and restore is easier to hold in a reviewer's head; the cost (order changes when wiring changes) is exactly what KRN-RST-05 detects. The binding walk still uses `String(block_id)` because its order is externally visible in the conflict report.

### Alternative 3: A separate kernel verification stage with registered verifier callables
- **Description**: After R4 the kernel runs a list of `VerifierDescriptor` callables registered at `BOOTING`, so every verifier sees every block fully rebuilt.
- **Pros**: A verifier never reads a half-built block; one place names the first failing invariant; the Assembly verifier could live outside `post_restore`.
- **Cons**: A new registrar surface and descriptor (an ADR-0003 change); the TR rows say the checks run in `post_restore`; at the MVP every cross-block check reads Tier S or a predecessor, so the benefit is not yet needed.
- **Rejection Reason**: Not needed now; kept as the second move of § 5.6 if a verifier ever needs a non-predecessor's derived state.

### Alternative 4: Validate every block first, then commit every block (kernel-level two-phase)
- **Description**: Add a `validate_restore(data, ctx)` callable; run it for all blocks, and only if all pass call `restore`, so an owner failure leaves the pre-restore world intact.
- **Pros**: Pre-restore state survives any `restore` failure, not only the pre-check.
- **Cons**: A new mandatory descriptor field and double decode cost; cross-block validations (Body's liveness check against Layout's *restored* state, V5) cannot run before Layout is committed; no MVP caller can use an intact old world.
- **Rejection Reason**: The benefit has no caller (both callers discard the live world) and the cross-block checks make it impractical; the per-block two-phase of ADR-0006 plus the pre-check covers what can be covered.

### Alternative 5: Revert on failure from a pre-restore in-memory snapshot
- **Description**: Capture the live `WORLD` blocks before each pass and restore them if the pass faults.
- **Pros**: A failed load leaves the player in their world.
- **Cons**: A second restore pass that can itself fail; double cost on every load; contradicts the kernel GDD's "out of scope"; the Continue path has no live world to return to.
- **Rejection Reason**: § 1.4. Fallback to the next generation (a new pass) is Save/Load's mechanism.

### Alternative 6: Widen the `RESTORE` scope so `post_restore` may write predecessor blocks
- **Description**: Let `post_restore(X)` open the blocks in X's `restore_after` closure, so Interaction's `post_restore` can re-register subjects with the Emitter as IA-36 words it.
- **Pros**: Matches the wording of TR-interact-016 literally.
- **Cons**: Breaks ADR-0003 § 4 ("the restoring block only"); makes "post_restore changes only derived state" unprovable; the persisted subject registrations make the re-registration redundant; a hash-visible cross-block write at load invites G4 divergences.
- **Rejection Reason**: Redundant for a consistent save and unsafe for an inconsistent one; the drift is detected in `post_restore` and repaired by the ordinary hook (§ 4.3); ADR-0017 D10.2's 're-register idempotently in their own `post_restore`' is refused for the same reason. The cost is a GDD sync (below). If the owners later insist, the change is one row of ADR-0003 § 4 and a foundation ask, not a redesign.

### Alternative 7: Clamp in `post_restore` for the lowered `stock_cap` case
- **Description**: Run the Graph's stock reconcile in `post_restore` (ADR-0011 Open Question 2's alternative).
- **Pros**: The first run starts from an already reconciled store.
- **Cons**: The restore stops being byte-identical (`snapshot(restore(x)) != x`), forces a REBASED cause 4/5 entry in ADR-0006, and makes `post_restore` a writer of persisted state, which § 4.2 forbids for everyone.
- **Rejection Reason**: Phase 0 does the same arithmetic as any step, keeps K-4 at `MATCH`, and closes TR-itemdb-033 without touching the post_restore rule.

### Alternative 8: Order binding sources by the DAG or by registration index
- **Description**: Rebuild half (b) by walking `post_order` instead of ascending `block_id`.
- **Pros**: One order for everything.
- **Cons**: TR-cmd-013 states ascending `block_id`; the result is order-independent except for which conflict is reported first, and a block-id order is stable across wiring changes.
- **Rejection Reason**: Follow the GDD; the order is fixed once at the freeze through `CanonicalEncoder.sorted_string_keys` (probe P1 corroborates ADR-0005 P12).

## Consequences

### Positive
- One sequence for restore and one for new game, each with a named failure state; the "faulted `LOADING` never plays" guarantee is stated and tested.
- `post_restore` order is declared data; the Emitter's cold build no longer depends on a self-heal, and any hidden dependency is caught by the permutation test.
- The post_restore write policy is one table every owner can check; deferred fix-ups become ordinary hook rules, so G4 holds by construction.
- Verifiers are listed once, owned by their blocks, reused at runtime, and the first failure is deterministic (fuzzable).
- The `REBASE_BLOCKS` exit is a CI gate with a named owner, not a convention.
- The binding order is fixed once at the freeze through the sanctioned ADR-0005 helper, so it cannot depend on `StringName` interning order, and restore copies by value so no live block aliases a payload.

### Negative
- Foundation changes asked of ADR-0003, ADR-0002 and ADR-0010: four kernel codes (18 to 21) and one boot warning, three H1 allow-list names (`restore_plan`, `load_failure`, `load_timing`), the restore `ControlOutcome` fields, and `dictionary_variant.json` entries. No new lint pack.
- Interaction must change its design from "re-register at `post_restore`" to "detect at `post_restore`, repair in the hook" (the table reflects the repair at the following `emitter.place`, a lag of one tick at most), and move its normalisation into `snapshot()` before the first outside build.
- The pass stops at the first fault, so a corrupt save reports one problem at a time.
- A failed load cannot return the player to the pre-load world; the fallback is another generation.
- The permutation test multiplies the cost of every restore-affecting change in CI.

## Risks
- **R1: a block quietly depends on registration order.** Mitigation: KRN-RST-05 across at least four linear extensions on the MVP fixture, run on every push; a failing hash trace names the first diverging tick and block.
- **R2: Tier S/Tier D tags drift from the code.** Mitigation: the tags are doc comments printed by the Debug Overlay; no lint reads them, and KRN-RST-05 (the permutation test) is the only check. Residual risk is accepted.
- **R3: a deferred reconcile is not a no-op in the uninterrupted run.** Mitigation: KRN-RST-11 compares 600 ticks per owner; an owner that cannot make its fix-up idempotent files an ADR amendment before its story closes.
- **R4: the `kernel.clock` payload entry's shape (`tick_rate`) is not yet fixed by ADR-0006.** Mitigation: Open Question 2 and a foundation ask; until settled the `TICK_RATE_MISMATCH` pre-check cannot run on a payload without it.
- **R5: GDScript cost of a full pass is unmeasured** (Verification 2). Mitigation: it is a logged diagnostic, not a gate; S1 and SAVE-WRK-1-SLICE decide whether any rung is needed.
- **R6: `sanity.perception` has no declared `restore_after`.** Mitigation: assumed edges in § 2.4; if ADR-0025 declares none it can run before the blocks it rebuilds from. Open Question 1.
- **R7: a future PROFILE block's cold-open ordering is untested** at zero MVP blocks. Mitigation: the plan already visits `PROFILE` blocks in the cold-open pass; a story builds the test with the first PROFILE block.
- **R8: the proposed verifier V11 (one-holder) is outside Interaction's closed reason list.** Mitigation: it is marked proposed; accept or drop it with ADR-0020 (Open Question 3).

## GDD Requirements Addressed

Counts: 25 rows cite ADR-0016 in the traceability matrix's Planned ADR column (including the row re-tagged on 2026-10-29): **17 covered, 5 partial, 3 owned elsewhere**.

| GDD System | TR-ID | Requirement (abridged) | How This ADR Addresses It |
|---|---|---|---|
| command-simulation-layer.md | TR-cmd-013 | Kernel rebuilds binding half (b) as the last LOADING step from registered sources in ascending `block_id`; conflicts fault LOADING | § 6: R6/N6 is the last stage; `bind_order` is ascending `String(block_id)` built once at the freeze by `CanonicalEncoder.sorted_string_keys`; flat `PackedInt64Array` source payload; `BINDING_REBUILD_CONFLICT` with reasons; half (b) cleared at R2/N2 and left empty on a fault. Covered (data structure is ADR-0002's) |
| command-simulation-layer.md | TR-cmd-035 | Blocks declare `restore_after`; cycle or unknown id fails boot; `restore()` in DAG order; every `post_restore()` after every `restore()`; `post_restore` failure faults LOADING without reverting; more than `MAX_BLOCKS` fails boot | § 2.2 compile at the freeze (`UNKNOWN_DEPENDENCY`, `RESTORE_CYCLE`, `BLOCK_LIMIT`); § 1.1 R3 then R4 in the same order; § 1.3 and § 1.4 fault without revert. Covered |
| command-simulation-layer.md | TR-cmd-036 | Pre-checks before any `restore()`: unknown block, `TICK_RATE`, `restore_epoch >= 2^32`, schema version; fault with pre-restore state intact; missing persist block gets `default_state()` | § 1.1 R1 (four ordered checks, state intact; new `RESTORE_VERSION_UNSUPPORTED`), R3 (absent block through `default_state` and its own `restore`, § 3.3). Covered (Save/Load's file pre-check is ADR-0006's) |
| command-simulation-layer.md | TR-cmd-037 | New game: inside LOADING and a WriteScope, every `default_state()`, then `init_world` in `restore_after` order, then `post_restore()`, then the binding rebuild; `init_world` never on load | § 1.2 N3 to N6; one `INIT` scope; `init_world` only from N4 (counted by KRN-RST-02). Covered |
| item-and-module-database.md | TR-itemdb-033 | Restoring stock above a lowered live `stock_cap` succeeds, clamps to the cap, credits `discarded_*`, closes conservation, emits a diagnostic | § 4.4: accepts ADR-0011 § 7.3 (phase-0 reconcile, byte-identical restore, verdict MATCH); `post_restore` never clamps; KRN-RST-13. Covered (arithmetic is ADR-0011/0012's) |
| interaction-system.md | TR-interact-016 | `post_restore` drops channels, door attempts, `last_result`; resolves a RESERVED FULL-job part; keeps ledgers; re-registers all subjects idempotently before the first `emitter.place` | § 4.2 and § 4.3: own-block only; the drops are an interim `REBASE_BLOCKS` normalisation with exit into `snapshot()` (§ 9.3); subject registrations are persisted and restore verbatim, `post_restore` re-registers nothing and only detects drift (V10, `IA_RESTORE_SUBJECT_DRIFT`), and the repair is Interaction's ordinary hook rule, which ADR-0017 orders after `emitter.place`, so the table reflects it at the following `emitter.place` (one tick at most; G4 is unaffected because a consistent save has empty drift). **Partial**: the repair timing is ADR-0020's, and the GDD wording 're-registers before the first `emitter.place`' needs the sync below |
| interaction-system.md | TR-interact-017 | Each `loose[].inst` decoded in `post_restore` with `from_array(a, graph_step_no)`; wrong length, type or failed `validate_instance` fails with `INTERACTION_RESTORE_LOOSE_INSTANCE`, never clamping | § 5.3 V10; Graph `step_no` is Tier S so the decode needs no Graph `post_restore`. Covered |
| suit-and-equipment.md | TR-suit-012 | Held instances are six-element arrays decoded in `post_restore` with the restored Graph's absolute `step_no` after all blocks restore | § 5.3 V7; all `restore` calls precede all `post_restore` calls (§ 1.1) and `step_no` is Tier S. Covered |
| ship-section-and-layout-data-model.md | TR-section-008 | Static queries valid immediately after `restore()`; leaf-dependent queries only after Layout's `post_restore`, with `stale_leaf` before | § 3.4 generalises the two classes into Tier S and Tier D for every block; Layout is the first instance; the window is observable only inside the pass. Covered |
| ship-section-and-layout-data-model.md | TR-section-009 | `post_restore()` performs cross-block verification only (`graph_ref_base`, def ids, weld `seam_id`) and rebuilds `opening_leaves`, bumping versions for sections with open leaves | § 5.3 V2 at Layout's slot (the root, first); bumps are derived-member writes (§ 4.2). Covered |
| ship-body-and-interior-physics.md | TR-shipbody-010 | `post_restore` sets `pos_prev := pos_curr`, preserves `sub`, runs the trapped-actor probe, does not rebuild the seat binding; overlap is counted, never a load failure | § 4.4 and § 5.3 V6; § 6.4. Covered |
| ship-body-and-interior-physics.md | TR-shipbody-030 | Body publishes `seat_bindings()` ascending, SEAT rows only, derived per call, registered at BOOTING; the kernel LOADING step rebuilds from it | § 6.1 to § 6.3; KRN-RST-06 is the integration gate for AC-BDY-6 and AC-BDY-138 against the real kernel. Covered |
| modular-ship-assembly.md | TR-assembly-009 | Read-only cross-block verifier for INV-1..INV-7 and INV-10, all eight hard in `post_restore` (INV-6 `BODIES_LIVE_MAX` half warn-only) and INV-1/2/4/5/7 after each weld or cut in release | § 5.3 V12 and § 5.4 (same function, `subset_mask`, DAG slot after Layout, Graph, Interaction and Body). Covered (invariant bodies and apply-time costs are ADR-0021's) |
| boarding-and-salvage.md | TR-boarding-005 | `carry_ref` and the Suit carried instance restored together and cross-checked (`SUIT_RESTORE_CARRY_MISMATCH`); Interaction `post_restore` never touches `carry_ref`; channels dropped with a RESTORE abort | § 5.3 V7 (equality) and V11 (proposed one-holder), § 4.2 (no cross-block writes). Covered |
| web-map-and-transit.md | TR-webmap-022 | `post_restore` validates leg and berth invariants with `WEB_RESTORE_*`, never clamps or repairs, submits nothing; the first `transit.step` reconciles | § 5.3 V13; § 4.2 (submit impossible, no repair); § 4.3. Covered |
| attention-system.md | TR-attention-025 | `post_restore` rebuilds caches only; beacon reconciled in the first `attention.step`; no onset or breach cue on load | § 4.3 and § 4.4; `ALL` batch carries no transition. Covered |
| sanity-and-hallucination-rules.md | TR-sanity-019 | `post_restore` must not prune or mutate persisted state; a well-formed dead `subject_ref` restores and is pruned by the next hook run | § 4.2 (no prune, no repair) and § 4.3; V16 fails only a malformed ref. Covered |
| save-load-and-versioned-format.md | TR-saveload-034 | Every persisted block is restore-idempotent and supplies a fixture factory and `default_state()`; non-conforming blocks sit on an interim `REBASE_BLOCKS` list that must be empty at the first outside build | § 9: normalise in `snapshot()`, `FixtureFactory`, registry-enumerating harness, CI-checked `REBASE_BLOCKS` with named exits and the `SHIPPED` gate. Covered (idempotence rule itself is ADR-0006's) |
| item-and-module-database.md | TR-itemdb-032 | A save restores unchanged after catalogue tuning edits; no assert, clamp, delete or migrate; `catalogue_version` unchanged | § 4.4: restore checks are structural only; catalogue tunables are never restore checks; KRN-RST-13 tuning-drift matrix. **Partial**: the version-gating and hash side is ADR-0006's and the `catalogue_version` bump rule is ADR-0007's |
| emitter-state-and-light-exposure-query.md | TR-emitter-008 | `post_restore` rebuilds all derived state in ascending id incl. the whole static layer, keeps `lit`, re-derives `body_id`; a restore at any tick yields identical answers from T+1 (G4) | § 4.1 (Emitter after Layout, Graph, Body, Controller; resolves Emitter Open Question 26), § 5.3 V9, § 8 permutation test and KRN-RST-05 harness. **Partial**: the rebuild list itself is ADR-0017's |
| emitter-state-and-light-exposure-query.md | TR-emitter-009 | `beam_bindings() -> [(player_id, beam_id)]` ascending, pure read, registered at BOOTING so the rebuild binds seat beams after every `post_restore`; `toggle_beam` uses `OWNED_BY_ISSUER` with source `emitter.state` | § 6.1 to § 6.3: `binding_source` field, flat array, R6 after R4. **Partial**: the command descriptor and the method body are ADR-0017's |
| 2d-lighting-and-occlusion.md | TR-light-037 | 100 cycles of LOADING, build, body swap, restore and LOADING leave node, orphan-node and render-texture counts equal to cycle 1; pools rebuilt without leaking | § 10 and KRN-RST-10: the pass itself leaks nothing, checked by deltas against cycle 1 plus the CNT-02 `MEMORY_STATIC` leg (probes P3, P4). **Partial**: build, body-swap, render-texture and pool legs are ADR-0028's |

### Owned elsewhere (loaded and checked here; the decision is another ADR's)

| GDD System | TR-ID | Requirement (abridged) | Owner and this ADR's contribution |
|---|---|---|---|
| modular-ship-assembly.md | TR-assembly-038 | Two failure layers: a pre-check refuses an out-of-range `assembly.state` `schema_version` without touching the kernel; a block `restore()` failure sends LOADING to FAULTED and claims no surviving pre-restore state | ADR-0006 (Covered, Proposed). Contribution: the kernel-side ladder of § 1.1 and § 1.3 (pre-check state intact; R3/R4 faults leave a replaced prefix); `RESTORE_VERSION_UNSUPPORTED` as the kernel backstop |
| sanity-and-hallucination-rules.md | TR-sanity-037 | Restore reinstates Sanity state verbatim: no stage recompute, counters resume, the autosave state fully replaces the dead seat's frozen state | ADR-0025 (re-tagged 2026-10-29). Contribution: ordering only: `sanity.state` before `sanity.ledger` (§ 2.4) and whole-pass replacement of every `WORLD` block (§ 1.1 R3) |
| save-load-and-versioned-format.md | TR-saveload-037 | Saves identify players only by PlayerId in header `seats`; no peer id and no binding table stored; the kernel rebuilds bindings after restore so the first `RUNNING` tick passes authority | ADR-0006 (header `seats`) with ADR-0002 (table). Contribution: § 6.4 and KRN-RST-06 (first-tick authority after restore) |

## Performance Implications
- **CPU**: One-time DAG compile at the freeze, O(n²) over at most `MAX_BLOCKS` blocks. A pass is the sum of the owners' `restore` and `post_restore` plus a handful of kernel steps (about two `Callable.call` per block, three stages); owner figures are the Graph's `restore` plus `post_restore` at most 2 ms and the MVP world at most 3 ms with a 10 ms ceiling (TR-lsg-036), all UNVERIFIED on a release export (S1, SAVE-WRK-1-SLICE). The pass is one boundary service and may exceed a frame; the Pipeline holds the black quad (`LOADING_TARGET_MS`, `t_hold_pad`). The binding rebuild is O(sources x players) with players at most 4. Assembly's release verifier subset is its own allocation (`verifier_release`).
- **Memory**: `RestorePlan` is five `PackedInt32Array`s of at most `MAX_BLOCKS`; scratch objects are transient and acyclic; no new retained structure.
- **Load Time**: Inside `loading_ms` of the kernel GDD; pre-check and file decode are Save/Load's and off the main thread when hot.
- **Network**: None. The order is a pure function of registration data, so a co-op guest running the same build restores identically (VS-B).

## Migration Plan
No code exists; nothing migrates. Story order for the kernel epic: (1) `RestorePlan.compile` and `from_order` (bind_order through `CanonicalEncoder.sorted_string_keys`) and the boot codes with unit tests (cycle, self-edge, unknown, duplicate, limit, determinism and hash of `describe()`); (2) the pass driver over recorded stub blocks (KRN-RST-02 stage order, KRN-RST-03 pre-check matrix, KRN-RST-04 fault ladder, KRN-RST-07 `default_state` path); (3) binding rebuild and KRN-RST-06; (4) new-game pass; (5) the stub-level permutation harness (KRN-RST-05, built with stories 2 and 4 on the skeleton) and the idempotence harness (§ 9.2) with the first two real blocks (Layout, Graph); the world-level battery of KRN-RST-05 closes the Save/Load or Kernel epic once Layout, Graph, Emitter and Body are implemented; (6) per-block adoption inside each owner epic, each closing with its V-row fixture in KRN-RST-12; (7) `REBASE_BLOCKS` data file and CI gate; (8) KRN-RST-10 and KRN-RST-14. Foundation amendments (below) land before stories 1 and 2.

## Validation Criteria
- **KRN-RST-01** DAG compile: unknown id, self edge, two-block cycle, longer cycle, duplicate entry, `MAX_BLOCKS + 1` blocks; each fails boot with the right code and names the blocks; the order is identical across 100 compilations and across shuffled Dictionary insertion; the hash of `describe()` is unchanged after a run; `from_order` rejects a non-extension.
- **KRN-RST-02** Stage trace: recorded stubs show R0..R7 and N0..N7 in order, `post_restore` in the same order as `restore`, `init_world` only in N4 and never in a restore.
- **KRN-RST-03** Pre-check matrix (each check separately and in combination): kernel `FAULTED`, every block hash, the clock and half (b) unchanged; one case with two bad blocks asserts the block named is the first in ascending `String` payload-key order (and, for check (d), in `world_order`), whatever the payload's insertion order.
- **KRN-RST-04** Fault ladder (§ 8).
- **KRN-RST-05** Order-permutation / G4 equivalence (§ 8). The stub-level version gates acceptance; the world-level battery, the ADR-0017 D2.5 late-hook fixtures (E3, E6, E7, E8 on the snapshot tick; Interaction M6 on a lit lamp's node; a late hook setting `loco_state` to DEAD) and the per-tick `ControllerView` comparison are an epic closure gate.
- **KRN-RST-06** Binding rebuild: block ids chosen so StringName order differs from alphabetical; conflicts of every `reason`; source fault; empty half (b) after a failed pass; tick-1 authority.
- **KRN-RST-07** Absent persisted block restores via `default_state` and its own `restore`; a `default_state` with no `put` is `DEFAULT_STATE_NO_PAYLOAD`.
- **KRN-RST-08** Write fence: a cross-block `open_for` in `RESTORE` is `REFUSED`; the `state_hash` before and after `post_restore` is equal for every block not on `REBASE_BLOCKS`.
- **KRN-RST-09** Idempotence over the enumerated registry and `FixtureFactory` set; `REBASE_BLOCKS` equals the measured set.
- **KRN-RST-10** LOADING leak cycle (§ 10): deltas against cycle 1 with `int()` casts, plus the CNT-02 `MEMORY_STATIC` check on debug and editor binaries.
- **KRN-RST-11** Reconciliation symmetry per deferring owner.
- **KRN-RST-12** Verifier fixture table, one corrupt fixture per V-row, exact `(stage, block, code)` and identical on repeat; includes a corrupt-hook-cursor fixture (V3, `RESTORE_HOOK_CURSOR`, local code 26), a RAGGED-cut save (V2) and an INV-4 sparse-zero-pose-row fixture (V12 passes); a 1,000-mutation determinism run.
- **KRN-RST-13** Tuning-drift matrix (mass, `WEIGHT_HEAVY_G`, `GLOWSTICK_STACK_MAX`, `PATCH_CAP_BP`, `stock_cap`): restore succeeds, verdict `MATCH` in every case; the phase-0 clamp is asserted on the first Graph step's `stock_clamped` count and diagnostic.
- **KRN-RST-14** `loading_ms` per block against the registry names, logged on RM-DESKTOP and RM-DECK.
- **KRN-RST-15** Copy by value (§ 3.1, probe P5): after restore, mutating the live block's columns and nested containers leaves the payload bytes unchanged; `FixtureFactory` and `binding_source` results are fresh per call; the put-slot is empty after a `default_state` payload is applied.

## Related
- **Depends on**: ADR-0001, ADR-0002, ADR-0003, ADR-0004, ADR-0005, ADR-0006 (all Proposed). Acceptance requires ADR-0003 and ADR-0006 Accepted.
- **Wave-A drafts read**: ADR-0011 (§ 7.2 to § 7.4; Open Question 2 is closed by the agreed phase-0 reconcile), ADR-0013 (§ 10), ADR-0019 (§ 2.1, § 2.7).
- **Cross-ADR assumptions (after the Core adjudication)**: ADR-0011 owns the Graph's step-13 hook-cursor check (local code 26), `hdr.sched`, the persisted hook products and the `section_counts_into` definition (template range plus `stub_count` and `tombstoned`); ADR-0012 § 1 retargets its cursor note to ADR-0011 § 7.2; ADR-0013 restores byte-identically and never rewrites a stored hash; ADR-0017 orders the Interaction hook after `emitter.place`, accepts a one-tick lag, persists registrations (D10.2) and cites § 3.1 for the copy (D14.1); ADR-0019 publishes scalar `get_chest_x/_y` and `get_hand_x/_y` computed at read time, owns the resync input model (§ 8.5) and holds `carry_ref` as Suit's carried instance id; ADR-0020 owns the repair timing, V11 and the `snapshot()` normalisation exit; ADR-0021 adopts INV-4 as 'at most one Body pose row per body, and every row's body is live'; ADR-0015 cites § 6.2 for the `binding_source` payload; ADR-0014 closes OQ-1 on `changed_region` (advisory) against `version_epoch` (authoritative).
- **Siblings, assumptions listed in the workflow output**: ADR-0012 (phase-0 reconcile), ADR-0015 (`seat_bindings`, Body restore), ADR-0017 (Emitter restore order, `beam_bindings`), ADR-0020 (Interaction), ADR-0021 (Assembly verifier), ADR-0022/0023 (Attention, Horror), ADR-0025 (Sanity), ADR-0026 (Web Map), ADR-0027 (Suit), ADR-0028 (pools, leak legs).
- **GDDs**: `command-simulation-layer.md` CR-7, CR-8, CR-15, CR-18; `ship-section-and-layout-data-model.md` LY-26; `modular-ship-assembly.md` MSA-32; `interaction-system.md` IA-36/37; `suit-and-equipment.md` SE-13/34; `ship-body-and-interior-physics.md` BDY-31; `save-load-and-versioned-format.md` SL-12, SL-15; `emitter-state-and-light-exposure-query.md` Open Question 26.
- **Supersedes / contradicts**: none. Narrows IA-36's "re-register all subjects" and the Layout GDD's "informational only"; both are GDD syncs, not ADR conflicts.

## Open Questions
1. **`sanity.perception`'s `restore_after`.** The Sanity GDD gives none; § 2.4 assumes `[sanity.state, sanity.ledger]`. Owner: ADR-0025.
2. **Shape of the `kernel.clock` payload entry.** The pre-check needs a `tick_rate` in it; ADR-0006 § 3.6 includes the entry but does not list its fields. Owner: ADR-0006 (foundation ask).
3. **Verifier V11 (one holder of a carried item).** Accept into Interaction's closed reason list or drop. Owner: ADR-0020.
4. **When Interaction's normalisation moves into `snapshot()`.** Which fields and which story; the exit of the only interim `REBASE_BLOCKS` member. Owner: ADR-0020.
5. **Closed (2026-10-29): Interaction's repair order.** Registrations are persisted; `post_restore` only detects drift (V10); the repair is Interaction's ordinary hook rule, ordered after `emitter.place` by ADR-0017, reflected at the following `emitter.place` (at most one tick). No edge from Interaction is added (§ 4.4).
6. **Cost of a full pass in GDScript on a release export.** Unmeasured; S1 and SAVE-WRK-1-SLICE.
7. **A readable render-texture count on 4.7.2.** Owner: ADR-0028.
8. **Whether a verifier will ever need a non-predecessor's derived state.** If yes, § 5.6 move two (Alternative 3).
9. **Cold-open pass kind joins the request with the first `PROFILE` block (ruling R2).** How the `COLD_OPEN` pass kind is joined to the load request is deferred with Save/Load SL-9 (`PROFILE` files are deferred); no change to ADR-0002. Owner: Save/Load (SL-9).

## GDD sync
Applied by the orchestrator, not by this ADR (see the structured output for the exact strings): `command-simulation-layer.md` CR-8 (order covers `post_restore`, `default_state`, `init_world`; tie-break; new boot codes; pre-check version check; `binding_source` payload form); `ship-section-and-layout-data-model.md` L307 and the L535 row ("informational only" becomes a stated order); `emitter-state-and-light-exposure-query.md` Open Question 26 (granted); `interaction-system.md` IA-36 and IA-37 (`post_restore` re-registers nothing; drift is detected there and repaired by the first Interaction hook run, reflected at the following `emitter.place`; normalisation moves to `snapshot()`); `save-load-and-versioned-format.md` SL-12 and SL-15 (`REBASE_BLOCKS` data file, fixture factory, CI gate); `life-support-graph.md` LS-21 ("never by clamping" carve-out for the lowered-cap reconcile at the head of every phase 0, owed by ADR-0011); `architecture.md` Data Flow 7, the "restore_after order (MVP)" line, and the Spikes that gate ADRs row for KRN-LAT-02 (adds ADR-0016: stub-level KRN-RST-05).

## Registry Candidates
Proposed for `docs/registry/architecture.yaml` (not written by this ADR): state ownership `restore_plan` (kernel registry data, compile-once, private fields behind accessors); interface contracts `restore_pass_stages`, `binding_source_payload`, `post_restore_write_policy`, `restore_verifier_contract`, `rebase_blocks_list`; API decisions `restore_order_tiebreak` (registration index) and `default_state_through_restore`; forbidden patterns `post_restore_cross_block_write`, `post_restore_clamp_or_prune`, `restore_time_catalogue_assert`, `first_run_after_restore_branch`, `restore_retains_payload_reference`; referenced_by updates for `binding_table`, `owner_callable_contract`, `block_as_resource_or_node` and the existing `stringname_sort_for_canonical_order` pattern. The earlier candidates `block_id_ordering` and `stringname_ordering` are withdrawn (ADR-0005 P12 and the existing pattern cover them). The YAML is in the structured output.

## Review Log
Review Log, 2026-10-29 (engine review CONCERNS; TD-ADR review CONCERNS; Core-tier cross-ADR adjudication; autonomous "Revise flagged items"). Status stays Proposed.

Engine findings
- **E1** applied: Key Interfaces give each class its own file header (`load_stage.gd`, `load_failure.gd`, `load_timing.gd`, `restore_plan.gd`, `verify_out.gd`) with a one-class-per-file note.
- **E2** applied: `compile(blocks: Array[BlockDescriptor], max_blocks, out_errors: Array[String])`, typed `from_order`; callers build typed arrays; `LoadFailure.detail`, `VerifyOut.detail()` and the `restore` payload listed as ADR-0010 `dictionary_variant.json` foundation asks (CT-42).
- **E3** applied: P5 probe; copy-by-value rule in section 3.1 and Implementation Guidelines; put-slot cleared after `default_state` (section 3.3, CT-41); fresh arrays for `FixtureFactory` and `binding_source` (section 6.2, section 9.2); new test KRN-RST-15.
- **E4** applied: P3 extended (containers and self-capturing lambdas); KRN-RST-10 adds the CNT-02 `MEMORY_STATIC` leg with `int()` casts; review rule for scratch hygiene; `OBJECT_COUNT` stated as Object-cycle only (CT-42). The lint option became a review rule because the RST pack is dropped.
- **E5** applied: P4 reworded (render monitors 0 headless, lane W); KRN-RST-10 uses deltas against cycle 1; orphan monitor marked debug-binary only (Verification 6).
- **E6** applied with T7: R1 (a) visits payload keys as `String` in ascending order, (b) to (d) follow the registry order; section 5.5 states it.
- **E7** applied in part: P1 cites ADR-0005 P12 and states interning-order dependence across builds, platforms and hosts; KRN-RST-06 keeps ids created in reverse order. The sub-amendment "make RST-01 a member of the ADR-0005 lint with `<` and `>`" is declined: no kernel code compares `StringName` once `BlockIdOrder` is dropped (CT-40).
- **E8** applied: KRN-RST-05 shuffles with a test-owned Fisher-Yates on `DetRng`; `RestorePlan` fields are private behind accessors, immutability is by convention checked by the hash of `describe()`.

TD-ADR findings
- **T1** applied: Open Question 5 closed; Interaction subject model (persisted registrations, drift detected in `post_restore`, repair in the hook after `emitter.place`, one-tick lag) in section 4.4 and V10; TR-interact-016 downgraded to Partial; IA-36 sync reworded; the "emitter.place after the Interaction hook" assumption removed.
- **T2** applied through CT-08: V3 covers ADR-0011 section 7.2 step 13 (hook cursors); `kernel.clock` is Tier S from R2; section 3.2 reworded; KRN-RST-12 gains a corrupt-cursor row. The code is local 26 `RESTORE_HOOK_CURSOR`, not 25.
- **T3** applied: `BlockIdOrder`, RST-01 and the `stringname_ordering` pattern dropped; `bind_order` via `CanonicalEncoder.sorted_string_keys`; the existing pattern gains ADR-0016 in `referenced_by`; "real engine trap found" removed from Positive consequences.
- **T4** applied: acceptance gate is KRN-RST-01, -02, -04 plus stub-level KRN-RST-05; the world-level battery is an epic closure gate; the architecture.md Spikes row is a GDD sync item (that file is not edited here).
- **T5** applied: `LoadFailure` has its own file; `RestorePlan.from_order` test seam; pass kind deferred to the first `PROFILE` block; `binding_source` is a pure read valid at any boundary, including ADR-0002's `STRICT_DEBUG` `BINDING_DRIFT` during `RUNNING`.
- **T6** applied: foundation asks name `restore_plan`, `load_failure`, `load_timing` for the ADR-0003 section 10 allow-list; the count is corrected (four codes 18 to 21, one warning); R7 and N7 fire `load_done`, any pass fault fires `load_failed`.
- **T7** applied with E6: R1 (a) ascending `String` order with `kernel.clock` known, (d) over `world_order`; KRN-RST-03 gains a two-bad-blocks case.
- **T8** applied: the lint claim is dropped and KRN-RST-05 is named the only check; the `stale_*` obligation is limited to Tier D getters read during the pass; R2 reworded.
- **T9** applied: section 4.4 says "head of every phase 0"; KRN-RST-13 verdict is `MATCH` in every case with the clamp asserted on the first step's `stock_clamped` and diagnostic.

Cross-ADR rulings
- **CT-01**, **CT-25**, **CT-54**: no ADR-0016 text owed; section 3.4 already names the Graph port reads as Tier S.
- **CT-03** applied: V2 reads "template-range counts match; STUB rows and seam edges excluded".
- **CT-04** applied by alignment: section 4.4 records the persisted hook products and that `post_restore` re-derives P, p_o2, o2_pm and trend only; the phase-0 reconcile is agreed.
- **CT-05** applied by alignment (V7 and the step-number rule).
- **CT-08** applied (V3 step 13, section 3.2, section 3.4, KRN-RST-12, code 26).
- **CT-10** applied: V7 uses `GraphView.step_no()` derived from the clock, not `hdr`.
- **CT-11** applied: section 9.3 reads "layout.state restore is byte-identical; content drift is diagnostic only".
- **CT-28** applied: V8 is the derived recompute of `cap_seen` only.
- **CT-38** applied by alignment: section 4.4 and V10 use the detect-then-repair wording; Alternative 6 also refuses ADR-0017 D10.2's old wording.
- **CT-39** applied: V9 reworded; Open Question 5 closed; TR-interact-016 Partial; the ADR-0017 D2.5 late-hook fixtures, Interaction M6 and the DEAD `loco_state` case are in KRN-RST-05 and its Validation entry.
- **CT-40** applied (see T3); the ADR-0005 `<`/`>` extension is rejected.
- **CT-41** applied: section 3.1 `restore()` owns nothing in the payload; section 3.3 clears the put-slot before `restore`.
- **CT-42** applied: KRN-RST-10 cites CNT-02; `dictionary_variant.json` asks; the four new codes stated once (18 to 21) and `BINDING_REBUILD_CONFLICT` marked as existing; the RST pack is dropped.
- **CT-43** applied: R2 states `ClockView.tick()` during R3 and R4 and `T + 1` for the first `RUNNING` tick; `changed_region` advisory, `version_epoch` authoritative.
- **CT-44** applied: V12 quotes INV-4 as reworded and records the ADR-0021 assumption; fixture row added to KRN-RST-12.
- **CT-48** applied by alignment: section 6.4 records that a restored run never calls E9 or E4.
- **CT-49** applied: KRN-RST-05 states the input model (resync applied in the first `RUNNING` tick's P2; fixtures rest-intent or replay) and per-tick `ControllerView` comparison.
- **CT-51** applied by alignment: V7 and V11 as quoted (V11 stays proposed pending ADR-0020).

Rejected findings not applied: ADR-0005 `<`/`>` extension, ADR-0003 re-arm text, ADR-0006 staged-payload text, ADR-0019 G3/G4 amendment (all rejected or moved by the adjudication).
- **2026-10-29 confirm residuals:** Tier D lists the Graph's intensive caches (`P`, `p_o2`, `o2_pm`, trend), topology caches and descriptor store, no longer the served and band caches; Tier S gains `served_pm` (the persisted `n_served`), the persisted band memory and the Controller's chest and hand getters; the KRN-RST-05 read-view battery lists `served_pm` as a Tier S read; Enables row no longer lists ADR-0015 or ADR-0019 (Related).
- **2026-10-29 orchestrator rulings:** R2 (new Open Question 9: the cold-open pass kind joining the request with the first `PROFILE` block is deferred with Save/Load SL-9; no change to ADR-0002), R3 (`seat_bindings()` in § 6.1 reads `[(player_id, actor_id)]`, ascending `player_id`; at the MVP seat == PlayerId). Dependency-cycle break: the `Reads, as drafts only` sentence (ADR-0011, ADR-0013, ADR-0019) moved out of Depends On into a new `Soft references (not acceptance prerequisites)` row together with ADR-0015 and the unwritten ADR-0020/0021/0022/0023/0025/0026/0027/0028 consumers; Enables now lists ADR-0017 only.
