# ADR-0015: Hand-Rolled Integer Kinematics and the Ship Body

## Status
Proposed

## Date
2026-10-29

## Last Verified
2026-10-29 (against `design/gdd/ship-body-and-interior-physics.md` BDY-1 to BDY-35, D.1, D.12 and the Acceptance Criteria it cites; ADR-0001 to ADR-0010, ADR-0011, ADR-0013 and ADR-0019, all Proposed; the engine facts marked "probed 2026-10-29" were run on the 4.7.2 mono console binary, editor build, `--headless --script`, from `C:/Users/dasbl/AppData/Local/Temp/claude/adr-probes/0015/`; revised 2026-10-29 after the engine and TD-ADR reviews, whose probes ran in `adr-probes/review-ADR-0015`; see the Review Log)

## Decision Makers
Claude (author), running under modes.automation: autonomous (project.yaml); decisions logged for the user's review; the user has not yet reviewed this ADR.

## Summary
Ship Body owns every actor's integer position inside a ship, every ship's pose in the exterior frame, and the only motion code on any simulation path, but no document fixes whether that motion uses an engine physics body, how the 1-px sweep is made fast enough, what shape `body.state` has, or how a weld and a cut move actors from one frame to another. This ADR decides: **no engine physics anywhere on a simulation path** (zero physics servers, bodies or areas, enforced by lint over `src/sim/`); motion is the pure **`LocomotionStep`** (`step(LocoState, LocoIntent, CollideSource) -> LocoResult` as the specification and test surface, `step_into` as the allocation-free hot form, shared by Body and, later, Horror Entities); collision is a **swept analytic clamp** against the one `solid_in` result per moving actor per tick, with the literal 1-px stepper kept as the test oracle (proven result-identical on 300,000 random fixtures in a 2026-10-29 probe); `body.state` is one SHARED `SIM` block of id-sorted struct-of-arrays columns with a **five-key plain-Array snapshot**, a closed fault domain 5 and an `interpolated` previous/current pair rolled inside the hook; and a weld (BM3) or cut (BM4) **rebases actors from Layout's returned records**, conserving every actor's absolute position and `(section, section-local)` pair, with a cut that would entomb an actor refused rather than displacing anyone.

## Engine Compatibility

| Field | Value |
|-------|-------|
| **Engine** | Godot 4.7.2 (pinned 4.7; installed 4.7.2.stable.mono, `docs/engine-reference/godot/VERSION.md`) |
| **Domain** | Core (simulation: kinematics and collision on integers; Physics is the domain whose engine facility this ADR declines to use) |
| **Knowledge Risk** | HIGH for the domain label (Physics: 4.6 and 4.7 changed Jolt 3D and the 2D server notes), MEDIUM for what the module actually touches. The module uses no physics, rendering or scene API; the HIGH items it does touch are the 4.7 packed-array element-write change (every dirty mark here is explicit inside a mutator or the hook), the 4.7 typed-return override rule (applies to the `@abstract` ports of § 2) and the float32 canvas bound for world poses |
| **References Consulted** | `docs/engine-reference/godot/VERSION.md`; `breaking-changes.md` (4.6 to 4.7 GDScript rows: "Overrides inherit a typed return", "Packed array element writes"); `deprecated-apis.md`; `current-best-practices.md` § GDScript (4.7); `modules/physics.md` ("2D physics UNCHANGED (still Godot Physics 2D)"; Jolt is the 3D default and does not apply). **`docs/engine-reference/godot/modules/` has no core, scripting or tilemap file**, so every claim about packed-array methods, `Node2D.position` precision and GDScript cost below is either probed or marked UNVERIFIED. Project ADRs: 0001 to 0010 (§ Related), 0011, 0013, 0019 |
| **Post-Cutoff APIs Used** | Packed-array element writes that skip the property setter (4.7; no setter is relied on anywhere); `@abstract` ports (4.5, chosen by ADR-0010) for `CollideSource`, `BodyLayoutPort` and `PlanMismatchSink`. No physics, `Node`, `Resource` or `TileMapLayer` API. **`@abstract` release fallback (ADR-0014, LINT-SPIKE-1(a)):** if `@abstract` does not survive a release export, the three ports become plain base classes whose methods `push_error("ABSTRACT_CALLED")` and return a safe default (`0`, `false`); every call site, `LayoutCollide` and the test doubles stay unchanged. The one-line `@abstract class_name X extends RefCounted` form was probed to parse and instantiate on 4.7.2 (engine review) |
| **Verification Required** | **V1 (probed 2026-10-29, done):** `PackedInt64Array.bsearch(v)` returns the insertion index (2 for 5 in `[1, 3, 2^53+7]`) and is exact for 2^53+7; `insert` and `remove_at` keep order; `duplicate()` is independent of later element writes; a plain assignment `b = a` of a packed array aliases (a write through `b` is visible through `a`). **V2 (probed 2026-10-29, done):** `-181 / 256 = 0`, `-362 / 256 = -1`, `-768 / 256 = -3` (truncation toward zero), `posmod(-1, 48) = 47`, `-7 % 3 = -1`; `mini`, `maxi`, `absi`, `signi`, `clampi` exist and return ints. **V3 (probed 2026-10-29, done):** `Node2D.position` stores float32: `2^23 + 1` and `-(2^23 + 1)` round-trip exactly, `2^24 + 1` collapses to `2^24` (this closes the `[UNV: build precision]` of BDY-26 for the **standard editor build**; the release-export repeat rides the ADR-0008 pipeline spike; reproduced by the engine review: `8388609` and `-8388609` round-trip, `16777217` becomes `16777216`; exact integer representability is not sub-pixel precision, because the float32 ulp is 0.5 px at 2^22 and 1 px at 2^23, so the `floor()` in `canvas_offset_render` is load-bearing, § 10). **V4 (probed 2026-10-29, done):** analytic clamp versus literal 1-px stepper on 300,000 seeded random fixtures (0 to 16 rectangles of 1 to 40 px, start offsets covering overlap and non-overlap, steps -3..3 on both axes, a stub one-section predicate): 70,658 fixtures began with the foot box inside a rectangle (the exempt path) and 293,765 requested a non-zero step; **0 mismatches** in either `moved_x` or `moved_y`. Indicative cost on the editor binary: 2.4 microseconds per actor-tick for the analytic clamp at K = 16 (both axes) against 23.8 for the stepper; these are **not** gates. The probe scripts `sweep_probe.gd` and `p2.gd` currently live in a user Temp directory; they are to be committed to `tests/support/probes/adr-0015/` as the Body epic's task 0 (Migration Plan), so the evidence is reproducible by a reviewer; until then V4 is an author-run result the review did not re-run. **V5 (UNVERIFIED, spike S1 / SP-ENG-1):** the GDScript cost of the whole `body.integrate` hook on a release export on RM-DESKTOP and RM-DECK (TR-shipbody-034), including accessor and abstract-port call overhead and four-tick catch-up. **V6 (UNVERIFIED, S1):** struct-of-arrays versus array-of-records for the actors table. **V7 (UNVERIFIED, release export):** the `trunc`, `posmod` and `floor_div` fixtures of V2 on a `template_release` export (LINT-SPIKE-1 of ADR-0010 carries the repeat; TR-shipbody-038's "spike S2" is that run). **V8 (answered by probe 2026-10-29, ADR-0034's CT-32 ruling):** `Node.reparent` preserves the global transform by default, so a `BODY_TRANSFER` uses `reparent(p, false)` (`keep_global_transform` false) and the owning presenter rewrites the node's position from `BodyView.pos_x/pos_y` in the same frame before draw; this ADR decides none of it and publishes only the getters. **V9 (probed 2026-10-29 by the engine review, done):** a `const A := PackedInt32Array([..])` does not compile ("Assigned value for constant isn't a constant expression"), `const A: PackedInt32Array = [..]` does (§ 2); writing an `int` outside int32 into a `PackedInt32Array` element wraps silently with no error (`1 << 31` becomes `-2147483648`, `(1 << 32) + 5` becomes `5`), which is why § 11.2 range-checks every int32 column value; `PackedInt64Array.bsearch(v)` always returns an insertion index, possibly `size()` (§ 4.5); assigning a `bool` into a `PackedByteArray` element is an invalid assignment (§ 7); assigning a packed array to a member replaces the reference and makes the member alias the source (§ 11.2); `PackedInt32Array.assign()` does not exist on 4.7.2; `>>` of a negative literal is a parse error and a negative variable shifts arithmetically (§ 10) |

## ADR Dependencies

| Field | Value |
|-------|-------|
| **Depends On** | ADR-0003 (block descriptor incl. `interpolated`, Class A mutators with `scope` first and `OK 0 / REFUSED 1 / -fault_code`, `WriteScope`, fault domain 5, `ctx.complete()`, hook `after` dictionaries, long-lived read facades); ADR-0004 (`KeySpaceDescriptor`, `DirtyKey.make`, `emit_event`, `MAX_DIRTY_KEYS`); ADR-0005 (`IntMath`, canonical TLV encoding, block hash, `Hash64`); ADR-0006 (snapshot payload types, two-phase `restore`, `ctx.fault`, closed `REBASE_BLOCKS`); ADR-0007 (`BodyTuning` ScalarTable, boot tiers); ADR-0010 (`src/sim/body/`, ports, `HOT-01`, `SIM-0x` rules, lint tiers); ADR-0001 and ADR-0002 (P2/P3 phases, hook registration, binding sources); ADR-0013 (`AttachResult`, `DetachResult`, `section_at`, rect semantics, `body_state` / `body_is_live`, `section_contains`); ADR-0014 (`SolidQuery.solid_in`, `QueryBuffer` with inclusive `spans`, the cell-coupling invariant `BROADPHASE_CELL_COUPLING`). None is Accepted, so this ADR cannot be Accepted before them |
| **Co-accepted with** | ADR-0019 (mutual ports: `BodyView` is read by the Controller, `ControllerView` is read by Body: ADR-0015 reads `octant`, `gait_effective` and `has_row`, ADR-0019's `ControllerView` chest and hand getters read `BodyView.pos_x/pos_y`; neither may be Accepted alone; CT-53). The one hard edge of the pair is declared in ADR-0019's Depends On (the later-numbered ADR); this ADR deliberately does not list ADR-0019 there, so the dependency graph stays acyclic. |
| **Soft references (not acceptance prerequisites)** | ADR-0016 (restore order and the seat-binding rebuild from `binding_source`, which ADR-0016 reads from this ADR as a draft; ADR-0016 does not list this ADR under Depends On). **Forward references (unwritten ADRs; each will list this ADR under Depends On when authored):** ADR-0021 (the weld and cut transaction calls BM3 and BM4), ADR-0023 (Horror reuses `LocomotionStep`), ADR-0026 (Web Map calls BM1, BM5, BM6, BM7), ADR-0027 (Suit reads `actor_section`), ADR-0020 (Interaction reads `foot_boxes_into`), ADR-0028 (Lighting anchors on `pos_curr`). |
| **Enables** | ADR-0017 (Emitter anchors on `pos_curr`), ADR-0018 (shadow origin reads `pos_x`, `pos_y`, `prev_x`, `prev_y`), ADR-0019 (co-accepted: the Controller's chest and hand getters read `BodyView.pos_x/pos_y`), ADR-0034 (`body.projector` subscribes to this block). |
| **Blocks** | The Ship Body epic (block, hook, BM1 to BM8, read view, `LocomotionStep`, sweep equivalence tests); the Controller story that needs a body to move; every Feature story that calls a Body mutator |
| **Ordering Note** | Wave B. ADR-0013 and ADR-0019 (wave A drafts) are built on directly; **ADR-0014 (same wave) is now a dependency, not a forward reference**: § 5 is written against its published `SolidQuery.solid_in(body_id, ax0, ay0, ax1, ay1, out)` and `QueryBuffer` verbatim. ADR-0016 (wave B), ADR-0021, ADR-0023, ADR-0026 and ADR-0034 are written afterwards and consume the interfaces fixed here. Where this ADR and the Body GDD differ (§ Related, "GDD sync"), the ADR is the proposal and the GDD is updated to it, not the reverse |

## Context

### Problem Statement
Four designed GDDs (Ship Body, Player Controller, Layout, Pipeline) already assume that nothing on a simulation path uses Godot physics, because the ship is a composite of welded sections, it grows and shrinks at runtime, its actors must replay bit-identically from a command stream (G3, G4), and the moving ship must move nothing inside it. The GDD states the behaviour precisely and leaves the code decisions to two named candidates, ADR-BODY-01 (the motion model and the sweep formulation) and ADR-BODY-02 (the `body.state` shape and restore idempotence). Without this ADR each of the following gets decided by whoever writes the story: whether the 1-px sweep ships or a faster form does, how the hot path stays allocation-free in GDScript, what exactly `LocomotionStep` takes and returns (Horror Entities reuses it), what `body.state` looks like as a block under ADR-0003 (the GDD predates the `interpolated` flag, the `scope`-first mutator shape and the reason/fault split), how a weld or cut moves actors using Layout's records, and which reads a SIM hook may take.

### Constraints
- Integer-only, scene-tree-free, deterministic, no wall clock and no RNG in the block or the hook (ADR-0010 `SIM-01` to `SIM-04`; kernel H9, H10; ADR-0005). No float declaration anywhere under `src/sim/body/`.
- Only `IntMath` divides (`SIM-04`, `SIM-05`); `/`, `%`, `min`, `max`, `abs`, `sign`, `clamp` are banned in `src/sim/`; `mini`, `maxi`, `absi`, `signi`, `clampi` are used (probed).
- Every kernel-invoked owner callable ends `return ctx.complete()` (ADR-0003 § 1). Mutators are Class A: `scope` first, `OK 0 / REFUSED 1 / -fault_code`, data through a caller-owned `out` (ADR-0003 § 5).
- Packed arrays are shared by reference in 4.7.2 (V1): a snapshot payload must be a duplicate, and no member may alias another.
- The hook runs inside `STEP_BUDGET_MS` as one hook among many; the cited figure is eight actors at 0.4 ms of the 2 ms P3 target, pending S1 (TR-shipbody-034). All wall-clock numbers are UNVERIFIED until S1 records them.
- The foot box is 16 by 8 base px, half-open, origin at the top-centre; collision is against Layout's `solid_in` rectangles exactly (24-px wall footprint), with no margin and no second rectangle set (TR-shipbody-017).
- The body is never displaced: no push, shove, snap or eject, by Body or by anything else (Pillar 1 and 2; PCI:650).

### Requirements
- Position of an actor is a pure function of the command stream and Layout's state (replay G3, G4).
- A body in motion and a body at rest produce a byte-identical interior (TR-shipbody-004).
- `snapshot(restore(snapshot()))` is canonical-byte identical for `body.state` alone, so it is not in `REBASE_BLOCKS` (TR-shipbody-009, -041).
- A weld conserves every moved actor's `(actor_section, actor_local_px)`; a cut conserves absolute position; neither mutator leaves a half-applied state (TR-shipbody-023, -024, -040).
- The hook never faults on a data defect; defects are absorbed deterministically and counted (TR-shipbody-013).
- The hook is allocation-free per tick and its performance gates are deterministic counts in CI, with wall-clock P99 as a nightly benchmark (TR-shipbody-014, -046).

### Binding stances respected (registry check, `docs/registry/architecture.yaml`)
- `scene_tree_or_engine_server_in_sim_layers` and `physics_callback_tick_source`: no engine server, no `_physics_process`; the tick source is ADR-0001's. This ADR adds the Body-specific forbidden pattern in § 1.
- `float_or_wall_clock_in_sim` and `integer_division_outside_int_math`: every division in this ADR is `IntMath.trunc_div` (residue) or `IntMath.floor_div` (px to tile).
- `mutator_status` (ADR-0003): Class A shape followed exactly; the `can_` prefix is the published-name divergence ADR-0013 already recorded and Body repeats it (§ 9).
- `block_persistence_payload` (ADR-0006): plain data, no live reference, restore reports through `ctx.fault` only and copies into the existing block object.
- `owner_callable_contract`: every callable below ends in `return ctx.complete()`.
- `non_integer_or_undeclared_dirty_key`: Body's keys are `DirtyKey.make(kind, id)` with kinds `BODY 2` and `ACT 4` declared in its `KeySpaceDescriptor`.
- `packed_array_exposed_or_setter_tracked` and `live_reference_in_snapshot_payload`: the SoA columns are never returned to a caller; `snapshot` copies; no setter is relied on.
- `canonical_encode_and_hash`: Body ships no byte writer and never feeds `var_to_bytes` output to a hash (TR-shipbody-008).
- `presentation_write_outside_allow_list`: presentation reads Body only through `BodyView`.

## Decision

### Architecture Diagram

```
 Layout (ADR-0013)                    Controller (ADR-0019)                  Assembly/Transit (ADR-0021/0026)
  solid_in (ADR-0014)                  controller.state: octant, gait_eff     apply: Graph -> Layout -> Body (one scope)
  section_at / section_contains        view only; Body never writes it          |  BM3 rebase_on_attach  BM4 rebase_on_detach
  body_state / AttachResult            |                                         |  BM1 place  BM2 remove  BM5 transfer
  DetachResult                         |                                         |  BM6 velocity  BM7 pose
        |  read via ports              v                                         v
        |        +----------------------------- src/sim/body/ -----------------------------+
        +------> |  BodyModule (owner)                                                      |
                 |   body.state  (RefCounted: BodyState persisted SoA + BodyDerived)        |
                 |   hook  body.integrate  divisor 1  after [controller.step, layout.doors] |
                 |      (i) pose: world_prev := world; world += vel   (ascending body_id)    |
                 |      (ii) actors, ascending actor_id:                                     |
                 |           prev := curr; load row -> LocoState scratch                      |
                 |           LocomotionStep.step_into(state, intent, collide, result)        |
                 |             residue (trunc_div) -> ONE solid_in via LayoutCollide -> clamp     |
                 |           write row back; mark ACT key; counters                           |
                 |   LocomotionStep / LocoState / LocoIntent / LocoResult  (pure, reused by  |
                 |      Horror Entities, ADR-0023)                                            |
                 |   ports: CollideSource  BodyLayoutPort  PlanMismatchSink                   |
                 +-----------------------------------+--------------------------------------+
                                                     | read view (long-lived facade), events, dirty keys
              +--------------------------------------+------------------------------------------+
              v                                      v                                          v
   PRESENTATION (Avatar, camera,           PROJECTOR body.projector (ADR-0034)         SIM readers: Suit (actor_section),
   canvas offsets: pos/prev, world/prev)   reparents on BODY_TRANSFER                  Interaction (foot_boxes), Emitter (pos),
                                                                                       controller.noise (moved, after integrate)
```

### 1. No engine physics on any simulation path (ADR-BODY-01, TR-shipbody-001, TR-pc-013)

**Decision.** Nothing under `src/sim/` (all twelve module directories, not only Body and Layout) uses a Godot physics body, area, server, shape, query, tile collision or navigation mesh. Motion and collision are `LocomotionStep` and the rectangle set of Layout, and nothing else. The Godot 2D server that exists (Godot Physics 2D, `modules/physics.md`, "2D physics UNCHANGED") is used for nothing by this module; the one candidate use is the presentation-only debris of ADR-0033, which lives under `src/presentation/` and is excluded from `ALL_SIM` hashing by construction.

**Who enforces what.** The token lint is **ADR-0010 `SIM-06`** (owner: it lists `RigidBody2D`, `CharacterBody2D`, `StaticBody2D`, `AnimatableBody2D`, `Area2D`, `PhysicsServer2D`, `move_and_collide`, `move_and_slide`, `test_move`, `body_test_motion`, `_physics_process`, `_integrate_forces`, `physics_jitter_fix`, `max_physics_steps_per_frame`); `SIM-06` runs over **all of `src/`** with no waiver (it is marked NW), which subsumes the GDD's "whole Body and Layout directories" criterion, and the one future exemption (the presentation-only debris of ADR-0033) is a recorded allow-list entry on a `src/presentation/` path, never on a `src/sim/` or `src/shared/` path. `SIM-06`'s list does not name `PhysicsBody2D`, `PhysicsDirectSpaceState2D`, engine tile collision or navigation meshes, which TR-shipbody-001 does; this ADR adds those tokens (`PhysicsBody2D`, `PhysicsDirectSpaceState2D`, `PhysicsShapeQueryParameters2D`, `NavigationServer2D`, `NavigationAgent2D`, `NavigationRegion2D`, `physics_layer`, `get_rid(`) and the further 2D-physics tokens that can answer a collision or overlap question on a sim path (`RayCast2D`, `ShapeCast2D`, `KinematicCollision2D`, `PhysicsRayQueryParameters2D`, `PhysicsPointQueryParameters2D`, `PhysicsDirectBodyState2D`, `CollisionObject2D`, `CollisionShape2D`, `CollisionPolygon2D`, `direct_space_state`, `get_world_2d`, and the TileMapLayer property `collision_enabled`) to lint `BDY-01` for `src/sim/**` (§ 12) and asks ADR-0010 to fold them all into `SIM-06` (most are `Node` subclasses that `SIM-01` would catch only by exact name). The engine's 2D physics server still exists and ticks with no bodies, which is harmless: the guarantee is **no use**, not "not instantiated". Float-free, `Node`-free, clock-free and RNG-free are `SIM-01` to `SIM-04`, `SIM-15`; the typing warnings promoted to errors for all of `src/` are ADR-0010 § 5, so the GDD's directory-scoped `directory_rules` promotion for Body and Layout is subsumed (the CI gate is that every script loads).

**Why not a lint-only claim.** GDScript inference can produce a float that no token list names (`var y := a * 0.5`). The promoted warnings (`INFERRED_DECLARATION`, `UNTYPED_DECLARATION`, `UNSAFE_*`, `NARROWING_CONVERSION`, `INTEGER_DIVISION`) plus `SIM-03` close that hole; the Body test suite additionally includes a reflection test that every member of `BodyState`, `BodyDerived`, `LocoState`, `LocoIntent` and `LocoResult` has type `int`, `bool`, `PackedInt32Array`, `PackedInt64Array` or `PackedByteArray` (§ 12, V-lint).

**Consequence stated once.** There is no physics-replay dependency: G3 and G4 hold for actor position because position is `f(command stream, Layout state)` on integers. A moving ship moves nothing inside it (§ 10); rotation is never a simulation fact (`heading_q` is reserved and must be 0).

### 2. Module shape: owner, state, kinematics, ports (ADR-0010 `src/sim/body/`)

| File (under `src/sim/body/`) | Class | Role |
|---|---|---|
| `body_module.gd` | `BodyModule` (`RefCounted`) | The owner. `setup()` is called by the composition root; registers the block, the hook, the key space, the fault domain, the diagnostics channel. Holds the mutators BM1 to BM8 and the `can_*` checks |
| `body_state.gd` | `BodyState` | The persisted columns only (§ 4) |
| `body_derived.gd` | `BodyDerived` | Non-persisted, non-hashed columns and scratch: `pos_prev`, `blocked`, `world_prev`, the `LocoState` / `LocoResult` scratch, the `QueryBuffer`, counters |
| `locomotion/locomotion_step.gd` | `LocomotionStep` | The pure kinematics function (§ 6) |
| `locomotion/loco_state.gd`, `loco_intent.gd`, `loco_result.gd` | `LocoState`, `LocoIntent`, `LocoResult` | The three value types (§ 6) |
| `locomotion/locomotion_tables.gd` | `LocomotionTables` | Constants only: `FP_ONE` (the 1/256 fixed point), `UNIT_X[8]`, `UNIT_Y[8]`, `ODO_W[8]`, `OCTANT_NONE` (8). **Declaration form (probed on 4.7.2):** `const UNIT_X: PackedInt32Array = [256, 181, ...]` (or `Array[int]`), never `const UNIT_X := PackedInt32Array([...])` or `PackedInt32Array()` in a `const`, which do not compile. A const array is read-only, so the hot path only reads it. Generated from, and golden-tested against, the registry entry `locomotion_axis_step` and the Controller's octant numbering (PCI D.3); the golden test also loads `LocomotionTables` and asserts the three arrays have size 8 and the expected `typeof` (one-line load test); Body never restates a speed |
| `sweep/axis_clamp.gd` | `AxisClamp` | The analytic clamp and the exempt-actor sub-step loop (§ 5) |
| `sweep/rect_spans.gd` | `RectSpans` | The one place a `QueryBuffer` span record is converted to a half-open rectangle (§ 5.1); `AxisClamp`, `SweepOracle` and every test double decode through it |
| `testing/sweep_oracle.gd` | `SweepOracle` | The literal 1-px stepper; test-only, never referenced from a non-test script (lint `BDY-05`) |
| `views/body_view.gd`, `views/body_view_empty.gd` | `BodyView` | The long-lived getter-only read facade and its `empty_view` twin (§ 11) |
| `views/place_result.gd` | `PlaceResult` | Out record for BM1 |
| `ports/collide_source.gd` | `CollideSource` (`@abstract`) | Body's own query and containment seam (§ 5.1): a Body-side port that wraps ADR-0014's `SolidQuery` plus `section_at`; `LayoutCollide` (in `src/sim/body/`) is its production adapter |
| `ports/body_layout_port.gd` | `BodyLayoutPort` (`@abstract`) | The Layout reads Body needs outside a `SimViews` call: body state (LIVE, DISSOLVED, UNKNOWN), section containment, offsets (§ 9, § 4.3); its implementation is a Body-side adapter in `src/sim/body/` over ADR-0013's `LayoutViews` |
| `ports/plan_mismatch_sink.gd` | `PlanMismatchSink` (`@abstract`) | The injectable defect sink (§ 9) |
| `body_faults.gd`, `body_reasons.gd`, `body_ids.gd`, `body_tuning.gd` | `BodyFaults`, `BodyReasons`, `BodyIds`, `BodyTuning` | Domain 5, check reasons, structural constants, the data-driven knobs |

The module holds its injected ports and tuning in **private** members (`_collide`, `_layout`, `_sink`, `_tuning`), so the `setup` parameters of the same plain names do not trip `SHADOWED_VARIABLE` (§ Implementation Guidelines).

**Layer rules.** Body is L2 (`src/sim/body/`). A sim module may call another by `class_name` (ADR-0010 § 2.2 rule 1), so Horror Entities calls `LocomotionStep` directly and uses the same `CollideSource` (`LayoutCollide`) with its own body id; no edge into `src/shared/` is needed, and `LocomotionStep` stays beside the Body code it is tested with. The three ports live in `src/sim/body/ports/` because every consumer (Body, Horror, tests) is L2 and may import that directory; they sit in a `ports/` directory, the sanctioned location of ADR-0010 § 3.1 (as `src/sim/layout/ports/` does for ADR-0013), so no `port_files` entry is needed. Nothing under `src/sim/body/` names `Node`, `Node2D`, `Camera2D`, `Viewport` or `Resource`; the PROJECTOR is `src/presentation/projector/` (ADR-0010 § 1.2, ADR-0034).

**Constants are data (coding standard).** The Body knobs `FOOT_BOX_W`, `FOOT_BOX_H`, `MAX_ACTORS`, `WORLD_COORD_LIMIT`, `VEL_MAX_PX_PER_TICK`, `DETACH_DRIFT_PX_PER_TICK`, `SWEEP_COST_LOG_MS` and `BODY_COORD_LIMIT` are one `BodyTuning` `ScalarTable` in the default-config data (ADR-0007 § 1.5), read once at `setup`. Their values are the registry's (`design/registry/entities.yaml`); this ADR cites names only. `BODY_COORD_LIMIT` is the one knob this ADR introduces (T8, CT-24): the bound on a body-frame coordinate, in base px; its default is **derived**, not asserted: `MAX_SECTIONS * MAX_SECTION_EXTENT_PX` (registry names; tens of thousands of px at the registry values), with a schema ceiling of 2^24 that exists only so an int32 column and a float32 canvas coordinate stay exact; a new registry value is requested (OQ-8). Structural constants that define the fixed-point format or an enum (`FP_ONE`, `UNIT_*`, `ODO_W`, `KIND_*`, `OCTANT_NONE`, key kinds) are code constants because changing one changes saved meaning and breaks replay; they are not tuning. Three named boot invariants (ADR-0007 `CONTENT_INVARIANT`): `BODY_FOOT_W_EVEN` (`FOOT_BOX_W` is even and positive, so the half-width is `FOOT_BOX_W >> 1` with no division and no negative `>>` operand), `BODY_COORD_LIMIT_COVERS_LAYOUT` (`BODY_COORD_LIMIT >= MAX_SECTIONS * MAX_SECTION_EXTENT_PX`) and `BODY_CANVAS_BOUND` (`2 * WEB_COORD_BUDGET_PX + BODY_COORD_LIMIT < 2^23`, the canvas bound of § 10). The broadphase coupling (`BROADPHASE_CELL_PX >= max(FOOT_BOX_W, FOOT_BOX_H) + V_RUN`, TR-shipbody-037) is **not** restated here: it is ADR-0014's boot invariant `BROADPHASE_CELL_COUPLING` (`BroadphaseCoupling.holds`), the single check, so two formulas cannot diverge on a retune. `DETACH_DRIFT_PX_PER_TICK` carries a schema range of `[0, 0]`, so a non-zero value is `CONTENT_FIELD_RANGE` at boot (TR-shipbody-026). **The bounds ledger:** lowering `WORLD_COORD_LIMIT`, `VEL_MAX_PX_PER_TICK`, `MAX_ACTORS` or `BODY_COORD_LIMIT` makes saves unloadable, so `tools/lint/config/body_bounds.lock.json` records `{schema_version, world_coord_limit, vel_max, max_actors, body_coord_limit}` and the boot lint `BODY_BOUND_LOWERED` fails when the loaded tuning is below the ledger for the same `schema_version`; raising a bound is free, lowering ships with a `schema_version` bump and a migration (the ledger row is then appended, never edited).

### 3. The block, the hook and what the kernel is told (ADR-BODY-02, ADR-0003 § 2)

**Registration** (all fields mandatory, ADR-0003 § 2; the GDD's BDY-5 list is completed with the three fields it predates):

| Field | Value |
|---|---|
| `block_id` / `owner_name` | `&"body.state"` / `&"body"` |
| `schema_version` / `supported_versions` | 1 / `[1]` |
| `block_class` | `SIM` (never `PERCEPTION_OUT`; no per-player variant, TR-shipbody-029) |
| `persist` / `persist_scope` / `scope` | true / `WORLD` / `SHARED` |
| `gameplay` | **true** (hooks and Assembly/Transit applies change it; the paused-write test applies) |
| `interpolated` | **true** (§ 8): the previous/current pair is Body's derived `pos_prev` and `world_prev`; `commit_baseline` and `snap_baseline` are therefore mandatory and supplied |
| `repl_hint` | `SHARED_SUMMARY` (unused until VS-B) |
| `restore_after` | `[&"layout.state"]` |
| callables | `snapshot`, `restore`, `default_state`, `post_restore`, `read_view`, `empty_view`, `commit_baseline`, `snap_baseline`, `binding_source` (the seat binding source of BDY-7: the GDD's `seat_bindings()` is reworded to `[(player_id, actor_id)]`, ascending `player_id`, `SEAT` rows only, derived per call; at the MVP `seat` equals the PlayerId, ruling R3); no `init_world` (Web Map's `init_world` calls BM1 and BM7 inside the `INIT` scope) |
| commands | **none**. Body registers no `CommandTypeDescriptor` and subscribes to no Layout dirty key (BDY-21) |
| hook | `HookDescriptor{hook_id &"body.integrate", owner_name &"body", divisor 1, offset 0, after [{id &"controller.step", tolerates_stale false}, {id &"layout.doors", tolerates_stale false}], callable}` |
| key space | `KeySpaceDescriptor` kinds `[2, 4]` named `[BODY, ACT]`; event ids `[1, 2, 3]` named `[BODY_TRANSFER, BODY_DOCK_CONTACT, BODY_HULL_SEPARATION]`; `quiet_writes_allowed = false` |
| fault domain | `register_fault_domain(BodyFaults.DOMAIN_ID /* 5 */, &"body", BodyFaults.NAMES)` |
| diagnostics | one channel `body.diag` (field list is ADR-0031's) |
| PROJECTOR subscriber | **not registered here**: `body.projector` is ADR-0034's class and subscribes to `body.state` through the facets of ADR-0004 and ADR-0009 |

**Hook order.** `body.integrate` is a **partial order**: after `controller.step` and `layout.doors`, which are mutually unordered. `controller.step` follows `suit.vitals` (ADR-0019 § 2.4); `emitter.place` is `after: [body.integrate, graph.propagate.failure]` (ADR-0017); `controller.noise` is `after: [{id: body.integrate, tolerates_stale: false}]` (ADR-0019 § 2.4) and `horror.step` is after `body.integrate` (ADR-0023, § 12 fixture). All four are divisor 1, so the `STALE_EDGE_UNDECLARED` boot warning cannot fire for them. An unknown hook id fails boot (`UNKNOWN_DEPENDENCY`), so the ids Body names are exactly the ones their owners register: `controller.step`, `layout.doors`, `suit.vitals`, `emitter.place`, `controller.noise`, `horror.step`.

**Why `gameplay = true` and not a derived block.** Body's persisted state changes during play, so it is not `persist = false`, and a paused kernel must not let a `PAUSED_APPLY` write it (ADR-0003 § 4: `PAUSED_APPLY` opens only blocks named in a command's `writes_blocks`; no pausable command names `body.state`).

**Why one block and not two.** A `body.frames` block next to `body.state` would add a second restore node, a second owner registration and a split between a pose row and the actors standing on it that a weld's single apply must keep consistent. The GDD rejected it (BDY-5); this ADR agrees: the two tables share `restore`, `snapshot`, one hash and one dirty-key space.

### 4. State shape: id-sorted struct-of-arrays (ADR-BODY-02, TR-shipbody-003, -006, -014)

#### 4.1 Persisted columns (`BodyState`)

```gdscript
class_name BodyState extends RefCounted
# scalar
var next_actor_id: int = 1                   # allocated here, never reused (CR-8); first id 1
# actors: parallel columns, ascending actor_id; a row is found only by _find_actor_row (4.5), never cached across a mutator
var a_id: PackedInt64Array        # int64 (a 2^53+7 id must not truncate)
var a_kind: PackedInt32Array      # BodyIds.KIND_SEAT = 1, KIND_AGENT = 2 (0 is invalid); append-only
var a_seat: PackedInt32Array      # PlayerId >= 1 for SEAT, at most BodyIds.MAX_SEAT; 0 for AGENT
var a_body: PackedInt32Array      # Layout body_id, 1 .. 2^31 - 1 (range-checked by restore and every mutator, 11.2)
var a_x: PackedInt32Array         # pos_curr, body-frame base px, |.| <= BODY_COORD_LIMIT (may be negative after a detach re-base)
var a_y: PackedInt32Array
var a_sub_x: PackedInt32Array     # residue, 1/256 px, |sub| <= 255
var a_sub_y: PackedInt32Array
var a_oct: PackedInt32Array       # last_octant 0..7, 8 = NONE: Body's own record, not the Controller's
var a_odo: PackedInt64Array       # int64 odometer, travels as a native int64 in the binary payload
# bodies: SPARSE (a missing row means world 0, vel 0, heading 0), ascending body_id
var b_id: PackedInt32Array
var b_wx: PackedInt32Array        # world pose of the body-frame origin, whole base px, |.| <= WORLD_COORD_LIMIT
var b_wy: PackedInt32Array
var b_vx: PackedInt32Array        # whole base px per tick, |.| <= VEL_MAX_PX_PER_TICK
var b_vy: PackedInt32Array
var b_head: PackedInt32Array      # heading_q: reserved, MUST be 0, persisted so a future rotation is a schema bump
```

`kind` values are assigned here (the GDD names the two kinds and fixes no number): `SEAT = 1`, `AGENT = 2`, append-only. Int32 is sufficient for every position column because a body-frame coordinate is bounded by `BODY_COORD_LIMIT` (at most 2^24, derived from the section extents, § 2) and a world coordinate by `WORLD_COORD_LIMIT` (2^22 in TR-shipbody-026). **Sufficient is not self-enforcing:** writing an int outside int32 into a `PackedInt32Array` element wraps silently with no error or warning (V9), so every value destined for an int32 column is range-checked **before** the write, in `restore` (§ 11.2, `BDY_RESTORE_COLUMN_RANGE`) and in every `can_*` check and mutator argument (§ 9, `BDY_ARG_RANGE`; an apply-time argument no check could have passed is `BDY_APPLY_PRECONDITION`). The bounds ledger (§ 2) is the single statement of the limits. Only `a_id` and `a_odo` are int64, exactly the GDD's two s64 columns (AC-BDY-137: an odometer of 2^53 + 7 must not truncate).

**Sparse form is canonical.** A body row whose six values are `(world 0, 0), (vel 0, 0), heading 0` is **deleted**, never stored (BM6, BM7 delete it, and BM4 does not insert one, § 9; no hook write can produce one, § 7); `restore` rejects such a row as `BDY_RESTORE_NONCANONICAL`, otherwise `snapshot(restore(x)) != x`. `MAX_BODIES` (Layout's) bounds the table.

#### 4.2 Derived columns (`BodyDerived`), never persisted, never hashed, never replicated

```gdscript
class_name BodyDerived extends RefCounted
var a_px: PackedInt32Array        # pos_prev x   (index-aligned with BodyState.a_*)
var a_py: PackedInt32Array
var a_blocked: PackedByteArray    # 1 = a non-zero step was requested and nothing moved
var b_wpx: PackedInt32Array       # world_prev (index-aligned with BodyState.b_*)
var b_wpy: PackedInt32Array
var loco_state: LocoState         # caller-owned scratch, allocated once at setup
var loco_intent: LocoIntent
var loco_result: LocoResult
var buffer: QueryBuffer           # ADR-0014's buffer type, sized at setup to MAX_FOOTPRINT_RECTS_PER_BODY
var counters: PackedInt64Array    # indexed by BodyIds.CNT_*; reset by tests only
```

Two private functions, `_actor_row_insert(idx)` and `_actor_row_remove(idx)` (and the body pair), are the **only** code that touches both column sets, so the persisted and derived columns cannot drift; a unit test asserts equal lengths after every mutator, every restore and every hook run. A reflection test classifies every member of `BodyState` and `BodyDerived` as persisted or derived (the ADR-0011 pattern) and a zero-and-diff test zeroes every derived member and checks `snapshot()` bytes unchanged.

**Row index discipline.** A row index is found by `_find_actor_row` / `_find_body_row` (§ 4.5) at the start of a call and is valid only until the next write to that table. No row index, `Dictionary` key, section id or `actor_section` answer is cached across a mutator or a restore (TR-shipbody-028; a stale section id is the failure mode). **No packed column is cached in a member** of `BodyView`, the hook or `LayoutCollide`: only loop-local aliases that are read and then dropped (§ 7), because a restore must be able to rebuild the columns without leaving a stale alias behind (§ 11.2).

#### 4.3 Why struct-of-arrays now, and what would reverse it

The GDD treated SoA as a working assumption and opened Open Question 46 (SoA versus array-of-records). **Decision: SoA.** Eight actors and eight bodies are tiny; what matters is that the hook visits every actor every tick and a `RefCounted` per actor would add an object dereference per field read, and that a typed `Array[ActorRow]` makes the snapshot a per-row copy while columns copy as ten and six `duplicate()` calls. The snapshot shape (§ 5) is independent of the in-memory layout, so **spike S1 can reverse this with no schema change** (V6). Insertion is by `bsearch` plus `insert` and removal is `remove_at`, kept sorted by the mutators and never sorted at tick time (V1 probed both on `PackedInt64Array`). Iterating a `Dictionary` is never used for any table (insertion order after erase and re-insert is not a hashing input).

#### 4.4 Ids and the dirty-key width

`actor_id` is int64 but a dirty key's `id` must fit 32 bits (ADR-0004 § 3: `DirtyKey.make(kind, id)`, `id <= 0xFFFFFFFF`). Ids are never reused, so a very long campaign could in principle grow the counter, but at the MVP (one actor per seat, respawn rows, `MAX_ACTORS` live) 2^32 is unreachable. It is nevertheless stated: `can_place` refuses with `BDY_PLACE_ID_EXHAUSTED` when `next_actor_id > ACTOR_ID_KEY_MAX` (0xFFFFFFFF), and `restore` rejects an `actor_id` or `next_actor_id` above it as `BDY_RESTORE_ACTOR_ID`. Body ids are Layout's (31-bit), so they always fit.

#### 4.5 Finding a row: absence is detected, never assumed

`PackedInt64Array.bsearch(v)` **always returns an insertion index** (probed, V9: `bsearch(2)` on `[1, 3, 2^53+7]` returns 1; with the default `before = true` a present value returns its own index and `bsearch(3, false)` returns the index after it). For an absent id the index can be `size()`, and an unchecked `a_id[row]` read is out of range (in a release export it logs an error and yields 0): exactly the stale-row failure R10 worries about, and one that "no cached index" does not prevent. Therefore the **only** lookup is one private function per table:

```gdscript
func _find_actor_row(id: int) -> int   # -1 unless row < a_id.size() and a_id[row] == id   (bsearch with before = true)
func _find_body_row(id: int) -> int    # same shape over b_id
```

Every getter of `BodyView` (§ 11.3) returns its neutral value (0, `false`, body 0) on `-1`; every mutator that takes an `actor_id` or `body_id` for an existing row returns `BDY_UNKNOWN_ACTOR` (or the body reason of that mutator) on `-1`; insertion uses the raw `bsearch` index, which is the correct insertion point. A unit test looks up absent ids at the first, middle, last and past-the-end positions of both tables and asserts `-1` and a neutral answer from every getter.

### 5. Collision: one query, one analytic clamp, the stepper as oracle (ADR-BODY-01, TR-shipbody-017, -018, -019, -036)

#### 5.1 The seam to Layout: `CollideSource` over ADR-0014's `SolidQuery`

```gdscript
@abstract
class_name CollideSource extends RefCounted
# Fills `out` (ADR-0014 QueryBuffer) with the SOLID rectangles of body `body_id` that intersect the half-open box
# [ax0, ax1) x [ay0, ay1), body-frame base px, unique, ascending rect_key, as INCLUSIVE span records
# (4 ints per record: x0, y0, x1 - 1, y1 - 1); returns the count. Allocation-free. No bind: body_id is explicit per call.
@abstract func query_into(body_id: int, ax0: int, ay0: int, ax1: int, ay1: int, out: QueryBuffer) -> int
# true iff the foot ORIGIN (x, y) lies in a section of body `body_id` (Layout section_at != NONE).
@abstract func origin_in_body(body_id: int, x: int, y: int) -> bool
```

`CollideSource` is **Body's port** (`src/sim/body/ports/`), kept because Horror reuses `LocomotionStep` (ADR-0023). The production implementation `LayoutCollide extends CollideSource` (in `src/sim/body/`) holds an injected ADR-0014 `SolidQuery` (`_solid`) and the `BodyLayoutPort` (`_layout`): `query_into` calls `_solid.solid_in(body_id, ax0, ay0, ax1, ay1, out)` and returns its count, `origin_in_body` is `_layout.section_at(body_id, x, y) != 0`. It is the only permitted wrapper over `SolidQuery` (ADR-0014 states the same). The stateful `bind(body_id)` of the first draft is gone, as ADR-0014 § 5 argues for its own ports ("explicit `body_id`, no per-body handle"): `LocoState` carries the `body_id` and `step_into` passes it on each call (§ 6). Tests supply `FakeSolidQuery extends SolidQuery` driving `LayoutCollide`, and a contract test runs `LayoutCollide` against the real `LayoutSolidQuery`.

**The span-to-half-open conversion happens in exactly one place: `RectSpans`.** ADR-0014 returns the **occupied span** (inclusive last cell), not half-open ends. `RectSpans` (`src/sim/body/sweep/rect_spans.gd`) is four static one-liners over `out.spans`: `x0 = s[4k]`, `y0 = s[4k + 1]`, `x1 = s[4k + 2] + 1`, `y1 = s[4k + 3] + 1`. `AxisClamp`, `SweepOracle` and every test double decode through them (doubles build records with `QueryBuffer.append_record(key, x0, y0, x1 - 1, y1 - 1, flags)`), and a unit test asserts that `spans[` is indexed nowhere else under `src/sim/body/` and `tests/support/`. Reading a span's last cell as the half-open end would sink an actor one pixel into every wall, and `SWEEP-EQ` would not notice if oracle and clamp shared the mistake, hence the single helper and a fixture of a **1-px-wide leaf** (`x1 - x0 == 1`, so `spans[4k+2] == spans[4k]`; § 5.5).

**The buffer.** `BodyDerived.buffer` is a `QueryBuffer` sized once at boot to `MAX_FOOTPRINT_RECTS_PER_BODY` (the most any query can return), so `QueryBuffer.grow()` (cold) never runs inside `step_into`. After each query a non-OK `out.status` or a non-zero `out.grew` is a **counted defect** (`BDY_QUERY_DEFECT`, never a fault; both are unreachable at the MVP). The result is the SOLID set in its **current leaf state** for the tick (post-cut from P2, post-`layout.doors` from P3); there is no inflated margin, no "perceived" variant and no second rectangle set (ADR-0014's guarantee). **Assumption on ADR-0023:** Horror uses this same `CollideSource` with its own body id.

**Why an `@abstract` port and not a `Callable`.** The GDD binds `collide` to the `solid_in` query "once at boot" (its draft name was `solid_in_into`). A stored `Callable` invoked with `.call(` is a `SIM-12` dynamic-dispatch finding and `Callable(` / `.bind(` are `HOT-01` tokens; an abstract-class method call is a direct typed call, mockable and lint-clean. The cost is one method call per query, not per rectangle (ADR-0014 probed a six-argument call through an `@abstract` port at 0.195 us against 0.188 us direct; V5 re-measures on a release export). No port method is named `bind`: the lexical `HOT-01` scan cannot tell a port method from `Callable.bind`, and no `HOT-01` exemption is carved for it.

**Unordered input is tolerated.** The sweep takes a minimum over candidates, so duplicates, an empty result and any order give the identical `moved`; in debug builds `query_into` results are checked for strictly ascending `keys` and a violation increments `diag.solid_in_unordered` (TR-shipbody-036; the order guarantee is ADR-0014's, the tolerance is Body's). A non-OK `status` or a non-zero `grew` after the query is counted as `BDY_QUERY_DEFECT` and otherwise ignored.

#### 5.2 The query box (D.12)

For an actor at `pos` with requested whole-pixel steps `(sx, sy)`, `|sx|, |sy| <= 3`, the single query is the Minkowski cover of the foot box over the x-then-y path:
`[pos.x - hw + mini(0, sx), pos.x + hw + maxi(0, sx)) x [pos.y + mini(0, sy), pos.y + FH + maxi(0, sy))` with `hw = FOOT_BOX_W >> 1` and `FH = FOOT_BOX_H` (at most 19 by 11 cells). The y-phase foot box lies inside this cover because the x-phase moved at most `|sx|` pixels toward the cover's edge. A rectangle the query does not return cannot be hit by construction. **A stationary actor** (move octant NONE, or both steps zero after the residue update) **issues no query**.

#### 5.3 The analytic clamp (non-exempt actor)

Rectangles here are half-open `[x0, x1) x [y0, y1)` as decoded by `RectSpans` (`x1 = spans[4k + 2] + 1`, `y1 = spans[4k + 3] + 1`, § 5.1). Let `E` be the set of returned rectangles whose half-open intersection with the **start** foot box is non-empty (cell count `ov(r) > 0`); `E` is computed from the query result at the start of the actor's tick and fixed for that tick. For an actor with `E` empty, one axis step of `n` pixels (`d = signi(n)`, `a = absi(n)`) is clamped as follows. Let the foot box be `[bx0, bx1) x [by0, by1)`.

- **x axis.** Consider each returned rectangle `r` whose y-range overlaps the box's (`r.y0 < by1 and r.y1 > by0`). Since the start box does not intersect `r`, `r` lies wholly east (`r.x0 >= bx1`) or wholly west (`r.x1 <= bx0`). Moving east, `limit = mini(limit, r.x0 - bx1)` for each east rectangle; moving west, `limit = mini(limit, bx0 - r.x1)` for each west one. `moved = d * limit` with `limit` starting at `a`.
- **y axis**, run after the x axis with the box at its post-x position: the same with the roles swapped (rectangles whose x-range overlaps the box's; south limit `r.y0 - by1`, north limit `by0 - r.y1`).

This is exactly what the 1-px loop returns: a rectangle blocks at the first pixel at which the destination box intersects it, every later pixel is also blocked (moving further toward it cannot clear it), and the loop's first refusal stops the axis. A 1-px-thick rectangle (a door leaf at a small `area`) cannot be skipped because the limit is the exact gap, not a sample.

**First-legal-row corollary (the designed examples).** Walking north into a wall whose footprint rows are `[y0, y0 + 24)`, the box rests with `pos.y = y0 + 24`; walking south it rests with `pos.y = y0 - FOOT_BOX_H` (box bottom flush). Walking into a wall adds 0 to the odometer.

#### 5.4 The exempt actor (a start box inside a SOLID rectangle)

When `E` is non-empty (a door leaf closed under the actor, a restore against a changed template, a template defect; a cut can no longer cause it, § 9 BM4) the body is **never displaced**. A one-pixel sub-step on either axis is admitted iff both: (a) **per-rectangle overlap does not increase** for every `r` in `E` (`ov_after(r) <= ov_before(r)`, measured in cells; a total-area test would let a box astride two abutting rectangles cross a hull line, the GDD's `fixture_airlock` case), and (b) **exit containment**: the destination foot origin lies in a section of the actor's own body (`CollideSource.origin_in_body`). Every rectangle **not** in `E` stays fully solid. The implementation computes the clamp of § 5.3 on the non-`E` rectangles once (the limit is a prefix-closed bound, so it composes with the sub-step loop by minimum), then loops over at most `limit <= 3` candidate sub-steps per axis (at most 6 per actor-tick), testing (a) against each member of `E` and (b) once per sub-step. This is the GDD's `T_exempt` model, not an analytic formula, and it is deliberately the literal predicate: the exempt path is a defect-only path and its cost is bounded by `n_sub <= 6` times `|E|`. `E` is never stored (no flag is persisted or hashed); every exempt actor-tick increments `PC_FOOTPRINT_OVERLAP`, and in debug builds fires `diag.actor_in_footprint`. The three accepted residual traps (a box wholly inside one rectangle may exit on either side; a box astride two abutting rectangles can only slide or back out; the hatch-cap trap) and the exposure rule for a footprint-resident actor (**EXPOSED**, never "the last room's air") are the GDD's BDY-16 and BDY-29 verbatim and are not re-decided; Body offers no rescue.

**The cap trap's single guard.** After the 2026-10-28 ruling the cut path cannot create it: BM4 refuses (§ 9). Its remaining sources are a restore of an older save and a template defect, so the bounded read-only **trapped-actor probe** (§ 11) exists for diagnostics only.

#### 5.5 The oracle and the equivalence proof

The literal 1-px stepper (`SweepOracle`) is the specification: before each pixel the destination box is tested against every candidate (non-`E` fully solid; `E` by condition (a); containment (b) when `E` is non-empty); the first refusal stops that axis. It lives under `testing/`, is never shipped in the hot path (lint `BDY-05`) and is the oracle of the **equivalence property test** (AC-BDY-23): every start offset in a 48 by 48 window, all 49 step pairs in -3..3 by -3..3 and the five fixture layouts (open floor, corridor, door sliver, abutting pair, hatch cap), each fed through the **span encoding** (`QueryBuffer.append_record` with `x1 - 1`, decoded by `RectSpans`) with one extra fixture of a **1-px-wide leaf** (`x1 - x0 == 1`), compared on `moved_x`, `moved_y` and the final position, with zero tolerance. **Probed 2026-10-29 (V4):** the same comparison on 300,000 seeded random fixtures (70,658 exempt starts) found 0 mismatches; the scripts are to be committed under `tests/support/probes/adr-0015/` (V4, Migration Plan task 0). The probe is evidence for the decision, not the test: the test is the committed fixture set.

**Decision:** the analytic clamp is the **shipped baseline**; the stepper is the oracle and a debug cross-check mode (`BODY_SWEEP_CHECK`, off in release, compares both for every actor-tick in a CI soak and fails on any difference).

### 6. `LocomotionStep`: the pure kinematics function and its three types (TR-shipbody-015, -016, TR-pc-013, -014)

```gdscript
class_name LocoState extends RefCounted          # the actor's kinematic state; no actor id
var body_id: int                                 # the body frame x, y are expressed in; only addresses collide calls, never written by step_into
var x: int; var y: int                           # pos_curr, body-frame base px
var sub_x: int; var sub_y: int                   # residue, 1/256 px
var last_octant: int = LocomotionTables.OCTANT_NONE   # Body's own record of the octant last integrated
var odometer: int                                # int64, never placed in a Vector2i

class_name LocoIntent extends RefCounted         # the only two Controller fields Body reads
var octant: int                                  # 0..7, 8 = NONE
var v_eff: int                                   # whole base px per tick requested; Controller's gait_effective

class_name LocoResult extends RefCounted         # the new state plus what happened
var x: int; var y: int; var sub_x: int; var sub_y: int; var last_octant: int; var odometer: int
var moved_x: int; var moved_y: int               # whole px actually moved this tick
var blocked: bool                                # a non-zero step was requested and moved == (0, 0)
var exempt: bool                                 # E was non-empty at the start of the tick
var octant_invalid: bool; var veff_invalid: bool # defects absorbed this call
var queries: int; var rect_tests: int; var section_tests: int   # counters for the CI count gates (§ 13)

class_name LocomotionStep extends RefCounted     # immutable after _init; no scene tree, no state between calls
func _init(v_eff_max: int, decel_ladder_enabled: bool) -> void
# Specification and unit-test surface. Allocates one LocoResult.
func step(state: LocoState, intent: LocoIntent, collide: CollideSource) -> LocoResult
# Hot form. Writes into the caller-owned `out`; returns nothing; identical results to step() (AC-BDY-91).
func step_into(state: LocoState, intent: LocoIntent, collide: CollideSource, out: LocoResult) -> void
```

**Why this shape.** TR-shipbody-015 fixes the name and the three arguments `step(state, intent, collide)`; TR-shipbody-014 fixes that a value-returning `step` and an allocation-free `step_into` return identical results. `LocoState.body_id` is the one field added to the TR's three-argument form: it names the frame the position is in, is copied from the row (Horror: from its own state), is never modified by `step_into`, and is what keeps `CollideSource` stateless now that `bind` is gone. One implementation (`step` constructs a result and calls `step_into`) means there is no second code path to keep equivalent. The hook (§ 7) loads the row into its scratch `LocoState`, calls `step_into`, and writes the row back: eight integer copies in and out, against an implementation that addresses columns by index and could not be shared with Horror. The `v_eff_max` constructor argument is the Controller's `v_max = v_run` from the same tuning table (the loader asserts the equality, TR-pc-014's cross-system check); `decel_ladder_enabled` is `STOP_DECEL_TICKS > 0`, dormant at the MVP default.

**The algorithm (BDY-9 to BDY-13, D.1, D.14).** Let `o` be the **move octant** (decided after step 1 has absorbed the defects): `NONE` when `v_eff == 0`; else `intent.octant` when it is not `NONE`; else, only when the ladder is enabled and `state.last_octant != NONE`, `state.last_octant`; otherwise `NONE` with `octant_invalid` set (the Controller promised a non-zero speed with no direction: IDLE, counted `BDY_OCTANT_INVALID`).
1. **Defects absorbed first.** `octant` outside 0..8 is treated as `NONE` (`octant_invalid`); `v_eff` outside `[0, v_eff_max]` is clamped for this call (`veff_invalid`). Neither can fault.
2. **The single reset predicate (BDY-12).** If `o != state.last_octant`, `sub` is set to `(0, 0)` **before** the step. After the step `last_octant := o`. So `sub` resets on a stop, on any `v_eff == 0` tick (STAGGERED, DOWNED, DEAD, `IN_POD`, CHANNELLING, the end of a ladder) and on any octant change; it does **not** reset on a gait change in one octant, on a refused step or in `post_restore`. Comparing with Body's **own** persisted `last_octant`, not the Controller's restore-normalised `octant`, is what makes a key held across a restore continue the residue (BDY-31, TR-shipbody-016).
2a. **No direction, no integration (the guard that keeps the tables in range).** If `o == NONE`: `sub := (0, 0)`, `last_octant := NONE`, `moved := (0, 0)`, `blocked := false`, the odometer is unchanged, no `collide` call is made, and `step_into` **returns here, before any table is indexed**. `UNIT_X`, `UNIT_Y` and `ODO_W` have size 8 and `OCTANT_NONE` is 8, so steps 3 and 6 index them only with `o` in 0..7; without this guard an idle actor would read index 8 of an 8-element const array, a runtime error inside the hot hook. A fixture with `v_eff == 0` and octant NONE on the hot form (`LOCO-IDLE-1`) pins it, and the early return is why an idle tick costs no residue arithmetic.
3. **Residue of both axes first.** `sub_x += LocomotionTables.UNIT_X[o] * v`, `sub_y += UNIT_Y[o] * v`; `step_a = IntMath.trunc_div(sub_a, FP_ONE)`; `sub_a -= step_a * FP_ONE`. `trunc`, never `floor`, so west mirrors east (V2: `-181 / 256 = 0`). Both residues advance **before** either sweep, so a refused step stores no energy and loses none (PCI:289, PCI:908).
4. **No step, no query.** If `step_x == 0 and step_y == 0`, `moved = (0, 0)`, `blocked = false`, no `collide` call (every `v_eff == 0` tick has already returned at step 2a; this covers the roughly 29 % zero-motion ticks of a carry diagonal and the first ticks of a slow start).
5. **One query, x then y.** One `collide.query_into(state.body_id, ...)` for the cover of § 5.2; compute `E`; clamp x (§ 5.3 or § 5.4), advance `x`; clamp y at the new x; advance `y`.
6. **Odometer and blocked.** `odometer += (absi(moved_x) + absi(moved_y)) * LocomotionTables.ODO_W[o]`; `blocked = (moved_x == 0 and moved_y == 0)` given a non-zero request.

**Rounding is two named helpers, never mixed (BDY-35).** `trunc` toward zero is the sub-pixel rule, used only in step 3, through `IntMath.trunc_div` (the one annotated `/` in `src/`). **Floor** division is the px-to-tile rule, used only where a pixel becomes a tile or a section-local coordinate (`section_at`, the broadphase, `room_at` inputs), through `IntMath.floor_div` (`b > 0`, any sign of `a`; body-frame coordinates may be negative after a detach re-base). `src/sim/body/` contains no other `/` or `%`.

**What Body reads from the Controller, and nothing else (TR-shipbody-016).** Per seat, from `controller.state` through its view: `get_octant(player_id)` and `get_gait_effective(player_id)` (v_eff), plus `has_row(player_id)` to detect an unbound seat (ADR-0019's getter names; its `row_count` stays for the give-up solo test). Body derives no gait, speed, facing, carry, channel or `loco_state` fact. ADR-0019 § 2.6 listed `intent_octant` among the fields Body reads; ADR-0019 strikes it (CT-27), Body does not read it (TR-shipbody-016 binds Body to two fields), and Interaction and Assembly remain its readers.

**Reuse by Horror Entities (ADR-0023).** Horror keeps its own position, residue and `last_octant` in `horror.state` (HE-23) and calls `step_into` with its own `LocoState` (whose `body_id` is the horror's body) and the same `CollideSource`, through Body's foot box and the same rectangles (assumption on ADR-0023, § Related). `LocomotionStep` has no reference to `body.state`, so a horror can never be an actor row at the MVP (`AGENT` rows stay unused).

### 7. The hook `body.integrate`: order, totality, allocation-freedom (TR-shipbody-012, -013, -014)

```
body.integrate(views, scope, ctx):
  if not scope.open_for(&"body.state"): return ctx.complete()          # counted violation, not a fault
  # (i) pose, ascending body_id
  for each body row b:
      world_prev[b] := world[b]                                          # derived, before the add
      if vel[b] != (0,0):
          if |world + vel| would exceed WORLD_COORD_LIMIT on either axis:
              vel[b] := (0,0) ; world unchanged ; world_prev[b] := world ; count BDY_WORLD_LIMIT
              # world is non-zero here: |vel| <= VEL_MAX_PX_PER_TICK <= WORLD_COORD_LIMIT cannot overflow from world 0,
              # so the freeze never yields a canonical-empty row. No hook write can; row deletion is BM4/BM6/BM7's job.
          else: world += vel ; mark BODY/<b>
  # (ii) actors, ascending actor_id
  for each actor row i:
      pos_prev[i] := pos_curr[i]                                         # the in-hook roll (section 8), also for stationary actors
      if layout.body_state(body_id) != LIVE: skip (not stepped, not moved), count BDY_ORPHAN_ACTOR; continue
      if kind == AGENT: treat as v_eff 0, sub := 0, last_octant := NONE ; count BDY_AGENT_UNSTEPPED   # no intent source at the MVP
      elif not controller.has_row(seat): treat as v_eff 0, sub := 0, last_octant := NONE ; count BDY_SEAT_UNBOUND
      else: intent := (get_octant(seat), get_gait_effective(seat)) from the Controller view
      load LocoState scratch (including body_id) ; LocomotionStep.step_into(scratch, intent, collide, result) ; write the row back
      a_blocked[i] = 1 if result.blocked else 0 ; if result.exempt: count PC_FOOTPRINT_OVERLAP     # a byte, never a bool
      if row changed (pos, sub, last_octant or odometer): mark ACT/<actor_id>
  # (iii) nothing else; actor_section is derived on demand (BDY-29), never per tick
  return ctx.complete()
```

**The hook is total (TR-shipbody-013).** A data defect is absorbed deterministically and counted, never a `scope.fault`: an orphan actor (body not LIVE in Layout, DISSOLVED or unknown) is skipped with its row kept (`BDY_ORPHAN_ACTOR`; an orphan row exists only between a Layout mutator and its Body follow-up inside one apply, or through a caller defect, because BM8 releases a dissolved body, § 9); an invalid octant is `NONE` (`BDY_OCTANT_INVALID`); `v_eff` outside range is clamped for the call (`BDY_VEFF_INVALID`); an unbound `SEAT` is not stepped, its residue reset (`BDY_SEAT_UNBOUND`); an `AGENT` row is not stepped at the MVP (the hook has no intent source for it), treated exactly as an unbound seat and counted (`BDY_AGENT_UNSTEPPED`); a pose that would leave `WORLD_COORD_LIMIT` has its `vel` zeroed that tick with `world` unchanged (`BDY_WORLD_LIMIT`; the technical-director's freeze form was adopted over saturation, so the hook never clamps a coordinate). Only a violation of the block's own table invariants (a duplicate `actor_id`, a negative counter) is `FAULTED`-class, and `restore` rejects those before they can exist.

**Order is fixed so diagnostics and dirty-key insertion are reproducible:** poses first in ascending `body_id`, then actors in ascending `actor_id`. Actors do not affect one another (they do not collide, BDY-17), so actor order cannot change a position; the order exists for reproducibility only.

**Allocation-free by construction, checked by counters (TR-shipbody-014).** The hook and the functions it calls are tagged `## @hot` (list in § 13): no `.new(`, array or dictionary literal, `.duplicate(`, `.resize(`, `str(`, lambda, `Callable(`, `.bind(` or packed-array constructor appears in them (`HOT-01`). The scratch records and the `QueryBuffer` are allocated once at `setup`; the sink and the ports are injected at `setup`; the `ControllerView` facade is fetched once with `views.get_view(&"controller.state")` and held in a member (facades are long-lived and never replaced, because `restore` copies in place, ADR-0003 § 2 and § 7), so no `StringName` lookup is repeated per tick. Loop-local aliases of packed columns (`var xs: PackedInt32Array = state.a_x`) copy a reference, not the data (V1: assignment aliases), and are used only for reading and dropped before the function returns (a packed column is never stored in a member, § 4.2). `blocked` is stored as a byte (`a_blocked[i] = 1 if result.blocked else 0`; assigning a `bool` into a `PackedByteArray` element is an invalid assignment, V9) and read back as `a_blocked[i] != 0`. The claim "allocation-free" is stated only as `HOT-01` clean plus `CNT-01` unchanged over 600 ticks plus `CNT-02` within tolerance, never as a bare assertion (ADR-0010 § 7).

**Stationary actors cost almost nothing.** A stationary actor does the residue update (or nothing when already at rest), the `pos_prev := pos_curr` copy and, if its `sub` changed, a dirty mark. It issues no `query_into`, no `section_at` and no `origin_in_body` call. The only `origin_in_body` calls the hook itself makes are the exit-containment tests of an exempt actor's candidate sub-steps (at most six per actor-tick). An ordinary tick makes none (TR-shipbody-012, -046).

### 8. `interpolated = true`: where the previous/current pair is rolled (ADR-0003, ADR-0009, ADR-0008)

`body.state` is the block presentation interpolates, so it is registered `interpolated = true`, which makes `commit_baseline` and `snap_baseline` mandatory. **Decision:** the pair is rolled **inside the hook and inside every mutator**, not at P4:
- The hook sets `pos_prev := pos_curr` (and `world_prev := world`) before it advances an actor or pose, for **every** row, moved or not. After the hook the pair is `(position at the start of tick T, position at the end of tick T)`.
- Every mutator that moves an actor discontinuously (BM1, BM3, BM4, BM5) sets `pos_prev := pos_curr` in the same write; BM7 and BM4's new row set `world_prev := world`; so a transfer never interpolates across the break (BDY-6, BDY-22, BDY-26).
- `commit_baseline(ctx)` is registered as a **documented no-op** that returns `ctx.complete()`. The reason is that a P4 roll would leave `prev == curr` at P5, when the presenters read the pair, and would destroy `moved_x(actor)` and `moved_y(actor)` (`pos_curr - pos_prev` per axis), which `controller.noise` must read after `body.integrate` in the same tick (TR-shipbody-042). KRN-ALPHA-02 (a baseline roll must not change canonical state) holds trivially.
- `snap_baseline(_key: int, ctx: HookContext)` sets `pos_prev := pos_curr` and `world_prev := world` for every row (the `_key` argument is deliberately ignored, the block is `SHARED`, and is underscore-prefixed so `UNUSED_PARAMETER` stays clean). The kernel calls it on every transition into `RUNNING` (ADR-0001 § 4), which is the **only** restore- and resume-continuity mechanism; `PAUSED` to `RUNNING` needs it because `tick_alpha` restarts and the last tick's one-to-three-pixel slide must not replay (KRN-ALPHA-01).
- `post_restore` additionally derives `pos_prev := pos_curr` and `world_prev := world`. That write is **Body-internal derivation of a non-persisted field** (BDY-6, BDY-31), so any reader between `LOADING` and `RUNNING` sees a consistent block; it is not the camera-continuity mechanism, which is `snap_baseline` plus the camera CUT on a `load_seq` change (ADR-0008 § 6.1).

**Foundation ask:** ADR-0003 should state that an owner may keep its interpolated pair inside its own hook and register `commit_baseline` as a no-op, and that the `key` of `snap_baseline` is ignored for `SHARED` blocks (§ Related).

### 9. Mutators BM1 to BM8: Class A, `scope`-first, check then write (ADR-BODY-02, TR-shipbody-020, -021, -023, -024, -025, -040)

All eight are **Class A** (ADR-0003 § 5), called only from Assembly's or Transit's single `apply`, **after** the Layout mutator in the same `WriteScope` (the order inside one scope is Graph, Layout, Body: ADR-0021's contract). Because Layout runs first, a Body apply-time predicate sees Layout's **post-mutator** state (a welded-in body is already DISSOLVED, a cut's new body already exists), which is why BM3 and BM8 test it that way below. Each has a pure check. **Naming (repeats ADR-0013's recorded divergence):** the published names are `can_*`, not `check_*`; the shape is exactly ADR-0003's (pure, takes `views` and scalars, returns `&""` for OK and a reason otherwise); the prefix divergence is settled by the Foundation amendment to ADR-0003 § 5 (CT-58), which admits `can_*` with the identical shape, so no rename is pending.

```gdscript
# src/sim/body/body_module.gd
class_name BodyModule extends RefCounted
func setup(facet: ModuleFacet, tuning: BodyTuning, collide: CollideSource, layout: BodyLayoutPort, sink: PlanMismatchSink) -> void   # stores _tuning, _collide, _layout, _sink

func can_place(views: SimViews, body_id: int, x: int, y: int, kind: int, seat: int) -> StringName
func place_actor(scope: WriteScope, body_id: int, x: int, y: int, kind: int, seat: int, out: PlaceResult) -> int      # out.actor_id
func can_remove(views: SimViews, actor_id: int) -> StringName
func remove_actor(scope: WriteScope, actor_id: int) -> int
func can_rebase_attach(views: SimViews, surviving_body_id: int, absorbed_body_id: int, shift_x: int, shift_y: int) -> StringName      # validate phase, before LM2: both bodies LIVE
func rebase_on_attach(scope: WriteScope, surviving_body_id: int, absorbed_body_id: int, shift_x: int, shift_y: int, moved_sections: PackedInt32Array) -> int   # apply phase, after LM2: absorbed DISSOLVED
func can_rebase_detach(views: SimViews, old_body_id: int, shift_x: int, shift_y: int, cap_rects: PackedInt32Array) -> StringName      # cap_rects = detach_preview(seam_id).rects
func rebase_on_detach(scope: WriteScope, old_body_id: int, new_body_id: int, moved_sections: PackedInt32Array, shift_x: int, shift_y: int, cap_rects: PackedInt32Array) -> int
func can_transfer(views: SimViews, actor_id: int, to_body_id: int, x: int, y: int) -> StringName
func transfer_actor(scope: WriteScope, actor_id: int, to_body_id: int, x: int, y: int, cause: int) -> int
func can_set_velocity(views: SimViews, body_id: int, vx: int, vy: int) -> StringName
func set_body_velocity(scope: WriteScope, body_id: int, vx: int, vy: int) -> int
func can_set_pose(views: SimViews, body_id: int, wx: int, wy: int) -> StringName
func set_body_pose(scope: WriteScope, body_id: int, wx: int, wy: int) -> int
func can_release_body(views: SimViews, body_id: int) -> StringName
func release_body(scope: WriteScope, body_id: int) -> int      # BM8: after Layout's LM4, same apply
```

**Three corrections to the GDD's mutator text, forced by ADR-0003.** (1) Every mutator takes `scope: WriteScope` first and begins `scope.open_for(&"body.state")`; on `false` it writes nothing and returns `REFUSED` (1), a counted violation that the caller does not escalate. (2) A mutator that produces data writes it to a caller-owned `out`: BM1's `actor_id` is `PlaceResult.actor_id`, not the return value (the GDD's "BM1 returns `actor_id >= 1` or a negative code" and TR-shipbody-020's wording become "BM1 returns the status; the id is in `out`"). (3) The frame shift is two integers `shift_x, shift_y` (Layout's `AttachResult.frame_shift_x/_y` and `DetachResult.frame_shift_x/_y`), and `cap_rects` is Layout's `DetachResult.rects`, a flat `PackedInt32Array` of four ints per rectangle (half-open, pre-cut body frame), consumed **verbatim** from ADR-0013 § 7; `rebase_on_attach` additionally takes `AttachResult.moved_sections` (the apply-time ownership check of BM3 needs it, and ADR-0021 passes the record unchanged). Layout returns records, so Body rebases from them without a second Layout call.

**Reasons are not faults (the GDD conflated them).** `can_*` return a **reason** (`BodyReasons`, `StringName`s spelled as the GDD spells them, e.g. `&"BDY_PLACE_CAPACITY"`), surfaced as `VALIDATION_FAILED(body.<REASON>)`. A mutator's apply-time defect is a **fault code** of domain 5 (`BodyFaults`, pre-packed `(5 << 16) | local`, append-only, `NAMES` indexed by local code). The mapping is one-to-one by name, with two apply-time-only faults that have no check (`BDY_REBASE_SECTION_NOT_OWNED`, `BDY_APPLY_PRECONDITION`) and two check-only reasons that have no fault (`BDY_REBASE_POSE_OUT_OF_RANGE`, because BM4 saturates instead of failing, and `BDY_ARG_RANGE`, whose apply-time twin is `BDY_APPLY_PRECONDITION`). `BDY_REBASE_ACTOR_IN_CAP` is both: a reason of `can_rebase_detach` (so a caller's `validate` asks Body the question, CT-26) and the apply-time fault BM4 raises if a caller skipped the check.

| Local | Name | Raised by |
|---|---|---|
| 1 to 15 | `BDY_RESTORE_VERSION`, `_TYPE`, `_COUNT`, `_ACTOR_ID`, `_KIND`, `_OCTANT`, `_SUB_RANGE`, `_SUB_INVARIANT`, `_ODOMETER`, `_HEADING_UNSUPPORTED`, `_POSE_RANGE`, `_CAPACITY`, `_ORDER`, `_NONCANONICAL`, `_UNKNOWN_BODY` | `restore` via `ctx.fault` (§ 11) |
| 16 to 21 | `BDY_PLACE_UNKNOWN_BODY`, `_PLACE_INSIDE_FOOTPRINT`, `_PLACE_CAPACITY`, `_PLACE_BAD_KIND`, `_PLACE_DUPLICATE_SEAT`, `_PLACE_ID_EXHAUSTED` | BM1 apply-time re-check; BM5 uses the first two |
| 22 | `BDY_UNKNOWN_ACTOR` | BM2, BM5 |
| 23 to 27 | `BDY_REBASE_SAME_BODY`, `_REBASE_UNKNOWN_BODY`, `_REBASE_SECTION_NOT_OWNED`, `_REBASE_SHIFT_OFF_GRID`, `_REBASE_ACTOR_IN_CAP` | BM3, BM4 |
| 28, 29 | `BDY_VEL_UNKNOWN_BODY`, `BDY_VEL_OUT_OF_RANGE` | BM6, also a `can_rebase_detach` reason |
| 30, 31 | `BDY_POSE_UNKNOWN_BODY`, `BDY_POSE_OUT_OF_RANGE` | BM7 |
| 32 | `BDY_APPLY_PRECONDITION` | any mutator reached with an argument no check can have passed (for example a negative count, or an int32-column value outside its bound); BM3 when the absorbed body is still LIVE (Layout's mutator did not run first) |
| 33 | `BDY_RESTORE_COLUMN_RANGE` | `restore` (§ 11.2): a value destined for an int32 column outside its bound (`body_id` 1 to 2^31 - 1, `seat` 0 to `MAX_SEAT`, `pos_x/_y` within `BODY_COORD_LIMIT`, body-row ids). Appended to the restore codes of local 1 to 15; the numbering is append-only |
| 34, 35 | `BDY_RELEASE_ACTORS_PRESENT`, `BDY_RELEASE_BODY_NOT_DISSOLVED` | BM8 and `can_release_body` |

(Local 21 `BDY_PLACE_ID_EXHAUSTED` and local 32 are the two additions to the GDD's reason set; local 33 to 35 and the check-only reason `BDY_ARG_RANGE` are added by this revision. `BDY_REBASE_UNKNOWN_BODY` means the surviving or old body is not LIVE, or the absorbed body is UNKNOWN to Layout.)

**Check-then-write, fail-closed (TR-shipbody-020, -040).** Each mutator evaluates its **whole** apply-time predicate before its first write. On a defect it (a) writes nothing (tables, `next_actor_id`, dirty set, event queue exactly as before), (b) calls `sink.report(mutator_id, code)` (default sink: increments `BDY_PLAN_MISMATCH`, active in every build; in debug it also `push_error`s), (c) calls `scope.fault(code, detail)` so the scope closes and the caller's later mutators write nothing (ADR-0003 § 5), and (d) returns `-code`. The caller, per ADR-0003, returns `ctx.complete()` at once after the first negative status. No path uses `assert()` (compiled out of release; `SIM-11`). The GDD's injectable `plan_mismatch_sink` Callable becomes the `@abstract` port `PlanMismatchSink` (`report(mutator_id: int, code: int) -> void`) with `CountingSink` (default) and `RecordingSink` (tests), because `Callable.call(` is a `SIM-12` token; tests exercise every defect in the ordinary debug gdUnit4 runner.

**The one saturate-and-continue branch must not fault.** BM4's new pose row, when `world_old + shift` leaves `±WORLD_COORD_LIMIT` or `vel_old` plus drift leaves `±VEL_MAX_PX_PER_TICK`, **saturates and returns `OK`**: the sink counts `BDY_WORLD_LIMIT` (where applicable) and `BDY_PLAN_MISMATCH`, and **`scope.fault` is not called**, because the Graph and Layout have already applied the cut and a fault would kill the rest of the apply. A row `restore` would reject (`BDY_RESTORE_POSE_RANGE`) would make the next autosave unloadable, so this branch is load-bearing (the death-restart policy restores the last autosave). Class (i) defects are checked first: a call with both an unknown new body and a range overflow returns the code.

**BM1 `place_actor`.** Allocates `actor_id` from `next_actor_id` (first id 1, never reused after removal), inserts the row at its `bsearch` position with `pos_prev := pos_curr`, `sub := (0, 0)`, `last_octant := NONE`, `odometer := 0`, runs the BDY-16 detection event for the new actor (a placement inside a SOLID rectangle is refused: `BDY_PLACE_INSIDE_FOOTPRINT`). Preconditions (`can_place`): body LIVE in Layout (`BodyLayoutPort.body_is_live`, defined as `body_state == LIVE`), `body_id` in 1 to 2^31 - 1, `x` and `y` within `BODY_COORD_LIMIT` and `seat` in 0 to `BodyIds.MAX_SEAT` (`BDY_ARG_RANGE`), foot box clear of every SOLID rectangle (one `query_into`), `actor_count < MAX_ACTORS`, `kind` is `SEAT` or `AGENT`, a `SEAT` has `seat >= 1` and an `AGENT` has `seat == 0`, no other `SEAT` row has this seat, `next_actor_id <= ACTOR_ID_KEY_MAX`. Marks `ACT/<id>`.

**BM2 `remove_actor`.** Deletes the row (and the derived row); marks `ACT/<id>`.

**BM3 `rebase_on_attach(surviving, absorbed, shift_x, shift_y, moved_sections)` (weld).** Only actors on the **absorbed** body are touched: `pos_curr += shift`, `body_id := surviving`, `pos_prev := pos_curr`, `sub` and `last_octant` **kept** (TR-shipbody-023: a whole-pixel shift on the 24-px grid cannot disturb a residue; a property test asserts both are byte-identical before and after). Host-body actors are untouched ("a weld moves nothing already built"). The absorbed pose row is deleted (its `vel` is discarded; the survivor's row is unchanged). Emits one `BODY_TRANSFER` per moved actor (`frame_delta = +shift`) and one `BODY_DOCK_CONTACT`.

**The predicate is split by phase (T1, CT-22).** Layout's LM2 marks the absorbed body DISSOLVED **before** BM3 runs (the order is Graph, Layout, Body), so one "both live" predicate re-evaluated at apply time would fault on every real weld after the Graph and Layout had already applied (the R5 no-rollback state, on the happy path). Therefore:
- `can_rebase_attach` (called from `validate`, **before** LM2) requires: the two ids differ (`BDY_REBASE_SAME_BODY`); **both bodies LIVE** per `BodyLayoutPort.body_state` (`BDY_REBASE_UNKNOWN_BODY`); `shift` on Layout's 24-px grid (`BDY_REBASE_SHIFT_OFF_GRID`, a Layout defect); and `shift` and every moved actor's shifted position within `BODY_COORD_LIMIT` (`BDY_ARG_RANGE`).
- `rebase_on_attach` (apply, **after** LM2) requires: the ids differ; **surviving is LIVE**; **absorbed is DISSOLVED** in Layout (`body_state`; UNKNOWN is `BDY_REBASE_UNKNOWN_BODY`, and an absorbed body still LIVE means the Layout mutator did not run first, `BDY_APPLY_PRECONDITION`); `body_of(s) == surviving` for every `s` in `moved_sections` (`BDY_REBASE_SECTION_NOT_OWNED`); the shift on the grid and the argument bounds as above.
`BodyLayoutPort.body_is_live` is defined as `body_state == LIVE`; `body_state(body_id)` (LIVE, DISSOLVED, UNKNOWN) is a read ADR-0013 owes (its "reads owed to Core siblings" table, CT-17). A `MUT-CONS` case runs LM2 and then BM3 against the **real Layout double**, not a stub that keeps both bodies live. The mutator runs in P2 and `body.integrate` in P3 of the same tick, so the shifted actor is stepped this tick from the shifted position with a zero interpolation delta. **Invariant (a test, TR-shipbody-023):** for every moved actor `(actor_section, actor_local_px)` is identical before and after, and `abs = world(body) + pos_curr` is conserved iff the docking invariant holds (§ 10).

**BM4 `rebase_on_detach(old, new, moved_sections, shift_x, shift_y, cap_rects)` (cut).** Order: (1) the **whole predicate first**: ids differ, `old` and `new` are both LIVE, every `moved_sections` member is owned by `new` (`BDY_REBASE_SECTION_NOT_OWNED`: the call-order trap, because Layout's mutator has already run), `shift` on the grid, and `actors_overlapping(old, cap_rects)` is empty (`-BDY_REBASE_ACTOR_IN_CAP` otherwise, writing nothing; Body rules REFUSE, never EJECT: the 2026-10-28 delegated ruling G6; the same question is asked **before** the Graph and Layout mutators by `can_rebase_detach(views, old, shift_x, shift_y, cap_rects)`, which returns `BDY_REBASE_ACTOR_IN_CAP` as a reason, so every cut origin's `validate` runs one Body check: a PLAYER cut is refused, a SYSTEM cut deferred (TR-shipbody-039); no cut path reaches BM4 without it, an assumption on ADR-0021). (2) For each actor on `old`, in ascending `actor_id`, the **side is the section containing its foot ORIGIN evaluated against the reconstructed pre-cut geometry**: for `s` in `moved_sections`, the pre-cut offset is `section_offset(s) + shift` (Layout has already re-based them); the origin is in `s` iff `BodyLayoutPort.section_contains(s, x - (offset_x(s) + shift_x), y - (offset_y(s) + shift_y))` (containment is by the section's floor area, never a bounding box, because hull-abutting sections have overlapping bounding boxes); an origin in no moved section is tested against the remaining body with the unchanged `section_at(old, x, y)`; an origin in no section at all stays on `old` and counts `BDY_TRANSFER_NO_SECTION` (defect-only; CI asserts zero in a clean build). (3) Actors on the new side: `pos_curr -= shift`, `body_id := new`, `pos_prev := pos_curr`, `sub := (0, 0)`, `last_octant` kept (harmless, `sub` is zero); one `BODY_TRANSFER` each (`frame_delta = -shift`). (4) Computes the new pose `world := world_old + shift`, `vel := vel_old` plus the default-zero drift (so both halves co-move; saturating as below) and **inserts the row only if it is not canonical-empty**; when `world == (0, 0)` and `vel == (0, 0)` the new body is the sparse zero and **no row is created** (a directed `MUT-CONS` case: an old body at world `(-576, -96)` cut at a section whose pre-cut offset is `(576, 96)`, at rest; storing it would fail the next restore as `BDY_RESTORE_NONCANONICAL`). An inserted row has `world_prev := world`. (5) Emits `BODY_HULL_SEPARATION`. (6) Runs the BDY-16 detection event for every actor on the touched bodies (moved or not), because the SOLID set changed under them. **A cut cannot create the cap trap**, because step (1) refuses it. **Conservation (TR-shipbody-024, -048):** detach conserves `abs` exactly for every actor on either side. **A weld followed by a cut of the same seam is not in general the identity on `pos_curr`** (detach re-bases to the minimum section id of the new component, ADR-0013 § 7; a weld shifts by the absorbed offset), so any round-trip test asserts `abs` and `(section, section-local)`, never raw `pos_curr`.

**BM5 `transfer_actor(actor_id, to_body_id, x, y, cause)` (pod hop).** `pos_curr := (x, y)`, `body_id := to_body_id`, `pos_prev := pos_curr`, `sub := (0, 0)`, `last_octant` kept; emits `BODY_TRANSFER{pod}` with `frame_delta = (x, y) - pos_curr(old)`. Body never searches for a free arrival cell (that would be a displacement): `can_transfer` answers `BDY_PLACE_INSIDE_FOOTPRINT` and Transit owns the alternative; two crewmates arriving at one cell is legal. `cause` is `POD` at the MVP. `can_transfer` also requires `to_body_id` LIVE and in 1 to 2^31 - 1 and `x`, `y` within `BODY_COORD_LIMIT` (`BDY_ARG_RANGE`).

**BM3, BM4, BM5 change only `body_id`, `pos_curr`, `pos_prev` and `sub` of an actor row** and nothing in `controller.state` (carry, DEAD state, channel fields). The Controller holds no write on Body and clears only its own channel fields on a body edge (ADR-0019 § 1.3), so a same-tick Controller write after a pod hop cannot displace the arrival cell (TR-shipbody-048; the property test runs a hop and a Controller tick in one scope and compares `controller.state` bytes).

**What each transfer resets (the TR-pc-035 reading, T4, CT-27).** All three set `pos_prev := pos_curr`. BM4 (the new side) and BM5 zero `sub`. BM3 keeps `sub` and `last_octant` by TR-shipbody-023, although it changes `body_id` for every absorbed-body actor. TR-pc-035's "resets `sub`" is therefore read as covering a pod hop and a detach only; an amendment of that TR text is requested (GDD sync), and ADR-0019's assumption row repeats this reading.

**BM6 `set_body_velocity`** creates or updates the pose row's `vel` (a zero velocity on a zero-world row deletes it); **BM7 `set_body_pose`** creates or updates `world_x, world_y` and leaves `vel`; it raises no event and touches no actor row, so the interior is unchanged (BDY-2). `can_set_pose` rejects `abs(coord) > WORLD_COORD_LIMIT` (`BDY_POSE_OUT_OF_RANGE`). BM7's only caller is Web Map & Transit inside the atomic applies the Web Map GDD lists (`init_world`, departure, `pod_arrive`, `pod_settle`, `berth`, `unberth`, each `berth_step`); a re-origin is a pure translation of every live body in one apply (ADR-0026's lint `REORIGIN_IS_TRANSLATION`), so no `rebase_world` mutator exists, and a dormant body's pose is never read by the simulation (lint `BDY-04`, § 12) (TR-webmap-008).

**BM8 `release_body(scope, body_id)` and `can_release_body(views, body_id)` (T2, CT-23).** Layout can dissolve a body **outside a weld**: LM4 `discard_section_geometry` (Web Map) marks the section's body DISSOLVED. Without a Body path, the pose row and any actor rows on that body would survive, the hook would tolerate them as `BDY_ORPHAN_ACTOR`, and restore step 6 would reject them (`BDY_RESTORE_UNKNOWN_BODY`), so the next autosave would be unloadable: the same failure BM4's saturation exists to avoid. BM8 deletes the pose row of a **DISSOLVED** body (a body with no row is the legal sparse zero: OK, no write), marks `BODY/<id>`, raises no event and touches no actor row. It refuses with `BDY_RELEASE_ACTORS_PRESENT` while any actor row is on the body, and with `BDY_RELEASE_BODY_NOT_DISSOLVED` unless `BodyLayoutPort.body_state` is DISSOLVED. It is the one Body mutator that runs after its Layout mutator for a body that mutator dissolved, so its predicate accepts DISSOLVED; BM6 and BM7 called after LM4 fault with `*_UNKNOWN_BODY`, so zeroing the pose first is not an alternative. **The discard sequence (an assumption on ADR-0026, not yet on disk):** transfer or remove the actors on the body (BM5 / BM2), then the Graph's mutator, then LM4, then BM8, in the one apply. Restore stays strict: skipping and counting orphan rows at restore was rejected because it hides the defect and breaks `snapshot(restore(x)) == x`. ADR-0013's LM4 note records that BM8 must follow it in the same apply.

**Consistency, including after every call that returned a code or saturated (TR-shipbody-040).** Five invariants hold: I1 every `actor_id` is unique, ascending and below `next_actor_id`; I2 body rows ascend and none is canonical-empty; I3 every `body_id` in either table is LIVE in Layout; I4 `restore(snapshot())` accepts the block; I5 the derived columns are index-aligned with the persisted ones. **I1, I2 and I5 hold after every Body call. I3 and I4 are invariants of the whole apply**, holding when the apply completes (after the last Body follow-up of a Layout mutator), not between a Layout mutator and its Body call: after LM2 or LM4 the Layout side has changed and Body has not yet caught up. A property test runs a generated sequence of **Layout and Body mutators** (LM2, LM3 and LM4 against the real Layout double, then the Body follow-ups BM3, BM4 and BM8; valid and defective calls, including each fault path, the saturating BM4 and the BM4 canonical-empty case) and checks I1, I2 and I5 after **every** call and I3 and I4 after every complete apply.

**Weld legality takes no pose input (TR-shipbody-025).** It is Layout's `can_attach` plus the Graph's `can_join` and nothing else. `docked(a, b, shift_x, shift_y) -> bool` is a pure read (equal `vel` on both bodies and `world_b - world_a == shift`, which lies on the 24-px grid by Layout D.6) offered to Assembly and Transit and **gating nothing inside Body**. A PLAYER-origin weld is OFFERED only when `docked` holds; this is a named must-repeat for ADR-0021 and ADR-0026 (the offer gate is the only guard of the exterior hull jump; the camera cut hides camera motion, not the absorbed hull's jump).

### 10. Pose, velocity and the inertial interior (BDY-2, BDY-26 to BDY-28; TR-shipbody-004, -026)

**World pose is exterior-only.** `world_*` locates the body-frame origin in the exterior frame; `vel_*` is whole base px per tick (no sub-pixel exterior residue in schema v1; a slower drift would be a schema bump). `body.integrate` adds `vel` to `world` once per tick, before any actor steps (§ 7). **Nothing keyed on a body-frame coordinate reads `world_*` or `vel_*`.** The proof is a fixture: set a non-zero `vel` on the avatar's body, run 600 ticks, and compare `canonical_encode(snapshot()["actors"])` with a body at rest: byte-identical (TR-shipbody-004). A lint (`BDY-04`) backs it: the getters `world_x`, `world_y`, `vel_x`, `vel_y`, `world_prev_*` and `world_pose` may be named only in `src/sim/web/`, `src/sim/assembly/` (for `docked`), `src/presentation/` and Body itself.

**The interior is an inertial frame.** Ship motion applies no force, push, tilt, drift or knockback to any actor; nothing moves an actor except `LocomotionStep` and the transfer arithmetic of BM3, BM4, BM5. `DETACH_DRIFT_PX_PER_TICK` is locked at 0 until the Vertical Slice defines an axis and a sign (schema range `[0, 0]`, boot error otherwise; the interior never sees it).

**Range and the float32 bound.** `|world| <= WORLD_COORD_LIMIT` (2^22) and `|vel| <= VEL_MAX_PX_PER_TICK` (16 per tick); `heading_q` must be 0 and is rejected on restore (`BDY_RESTORE_HEADING_UNSUPPORTED`, the mirror of Layout's `rot_q`). The 2^22 bound exists so that a canvas offset, the difference of two world coordinates and therefore within ±2^23, stays inside the range `Node2D.position` represents exactly (V3, probed 2026-10-29: `2^23 + 1` round-trips, `2^24 + 1` collapses to `2^24`). This closes BDY-26's `[UNV: build precision]` for the **standard editor build**; the release-export repeat and the camera math are ADR-0008's. If a release probe fails, the fallback is TR-shipbody-038's: a sign-split expression or a lower `WORLD_COORD_LIMIT`, which is a schema bump by the bounds ledger of § 2. Web Map & Transit's own budget (`WEB_COORD_BUDGET_PX` at most `WORLD_COORD_LIMIT / 32`) is stated against the default and is a Web Map lint.

**The body-frame bound and the canvas bound (T8, CT-24).** Two numbers other ADRs cite. (1) `|pos| <= BODY_COORD_LIMIT` for every actor position and every frame shift, restore-checked (`BDY_RESTORE_COLUMN_RANGE`) and mirrored in every check; Layout's `can_attach` must keep section offsets inside it (a must-repeat for ADR-0013); ADR-0017's `I-BOUNDS` takes its body-frame limit from here, and with `BODY_COORD_LIMIT <= 2^24` its doubled relative values stay below 2^26. (2) The canvas bound: `|world_a - world_b| + |pos| < 2^23`, which keeps a canvas coordinate exact in float32 and keeps `x + 0.5` exact for ADR-0018's light origin (the float32 ulp is 0.5 px at 2^22 and 1 px at 2^23). It holds under `WEB_COORD_BUDGET_PX` and is boot invariant `BODY_CANVAS_BOUND` (`2 * WEB_COORD_BUDGET_PX + BODY_COORD_LIMIT < 2^23`); if a retune breaks it, the world limit is lowered (a schema bump by the ledger), never the check. ADR-0017, ADR-0018 and ADR-0034 cite these two numbers.

**Derived exterior interpolation state.** `world_prev` (§ 8) is reset to the new `world` on every discontinuous write (BM7, BM4's new row, BM6 deleting a row, the `BDY_WORLD_LIMIT` freeze, `post_restore`, `snap_baseline`). The renderer computes `canvas_offset_render(b, tick_alpha) = floor(off_prev + (off_curr - off_prev) * tick_alpha)` per axis with `off_curr = world(b) - world(avatar_body)` and `off_prev = world_prev(b) - world_prev(avatar_body)` (the **current** avatar body for both, so a body switch never interpolates across two reference frames). That formula is a presentation computation (ADR-0008, ADR-0034); Body only publishes `world`, `world_prev` and `vel`. Two facts the presentation side must not lose: exact integer representability of `Node2D.position` is not sub-pixel precision, so the `floor()` is **load-bearing** (an unfloored interpolation at a large offset would quantise to the 0.5 or 1 px ulp), and `canvas_offset_render` must floor before assigning to `Node2D.position`. And on the Body side, `>>` must never receive a negative operand in `src/sim/body/`: a negative literal is a parse error and a negative variable shifts arithmetically; the only use, `FOOT_BOX_W >> 1`, is positive and boot-checked even (`BODY_FOOT_W_EVEN`).

### 11. Persistence, restore idempotence, the read view and events (ADR-BODY-02, TR-shipbody-006, -008, -009, -028, -029, -040, -041, -043)

#### 11.1 The snapshot payload: five keys, plain Arrays

`snapshot(ctx)` puts a plain `Dictionary` with **exactly five keys** (TR-shipbody-008; the shape is independent of the in-memory columns):

| Key | Type | Content |
|---|---|---|
| `next_actor_id` | int | the counter |
| `n_bodies` | int | pose rows |
| `n_actors` | int | actor rows |
| `bodies` | `Array` of ints, `n_bodies * 6` | row-major, ascending `body_id`: `body_id, world_x, world_y, vel_x, vel_y, heading_q` |
| `actors` | `Array` of ints, `n_actors * 10` | row-major, ascending `actor_id`: `actor_id, kind, seat, body_id, pos_x, pos_y, sub_x, sub_y, last_octant, odometer` |

No float, no `pos_prev`, no `blocked`, no diagnostic, no `Vector2i`, and `odometer` is never placed in a `Vector2i`. The int64 columns are copied into plain `Array`s at snapshot time (native int64 elements, exact for 2^53 + 7), ids are already ascending (the mutators keep the tables sorted; `snapshot()` verifies sortedness in debug and sorts only a table a defective caller left unsorted, never taking `Dictionary` order), and the arrays are fresh builds (never a live reference to a column, ADR-0003 § 2). **The canonical bytes are the kernel's, not Body's**: the TLV encoding of exactly this payload (`CanonicalEncoder`, ADR-0005: DICT keys sorted by `String(key)` code point, i.e. `actors, bodies, n_actors, n_bodies, next_actor_id`; every int an 8-byte little-endian INT), so s32 and s64 widths no longer exist in the bytes. Body ships no byte writer and never uses `var_to_bytes` for any hashed byte (`DET-01`; TR-shipbody-008).

**Why plain Arrays when ADR-0006 permits Packed arrays.** ADR-0006 allows `PackedInt32Array` and `PackedInt64Array` in a payload and ADR-0011 and ADR-0013 use them for their large tables. Body's payload is at most `MAX_ACTORS * 10 + MAX_BODIES * 6 + 3` integers (a few hundred bytes), so the Array-of-int cost is negligible, and TR-shipbody-008 fixes "plain Arrays". The practical advantage is in restore: a decoded payload is a `Variant` tree in which a float, a string or a bool can sit in a slot (a hand-edited or corrupt save container that still parses), and the type walk of `BDY_RESTORE_TYPE` is exactly what keeps a float from being silently truncated into a range-valid integer and breaking `snapshot(restore(x)) == x`. A packed array would hide that class by construction on the binary path. This reasoning is about the **binary path only**: the JSON dump is diagnostics and never a load path (ADR-0006 § 1.3), and `JSON.parse` yields floats for every number, so integers could not round-trip through it anyway. The alternative is not engine-blocked (ADR-0005 encodes typed and untyped arrays identically); it is declined for the reasons above. Snapshot and restore code therefore uses bare `Array` and `Variant`, which ADR-0010 `SIM-09` bans in `src/sim`; the ADR-0010 amendment of CT-54 carries an allow-list entry (a `dictionary_variant.json`-style entry) for the `body_state` snapshot and restore files. The `Array` form costs one pass over ~200 elements at restore (cold path) and is the form the golden file (AC-BDY-110) hashes.

#### 11.2 `restore(data, stored_version, ctx)`: two-phase, first failure wins, never clamps

Phase 1 decodes and validates into a scratch (cold path, allocation allowed) and on the **first** failure calls `ctx.fault(BodyFaults.<CODE>, detail)` and writes nothing, leaving the live block untouched; phase 2, only on success, copies the contents into the existing `BodyState` columns **element by element** (`resize`, then an index-copy loop; this is a cold path) and rebuilds `BodyDerived`. The scratch is built from fresh locals that are discarded after phase 2, never stored in a member, and none of it aliases `data`. **Neither the object nor any column reference is replaced**: the read facade is long-lived (ADR-0003 § 2), and writing `state.a_x = scratch_a_x` would replace the packed-array reference, leave the live column aliasing the scratch and leave any pre-held alias on the old data (V9; `PackedInt32Array.assign()` does not exist on 4.7.2, so it cannot be the in-place copy). A test restores and then reads through a pre-held `BodyView`. Reporting is **only** through `ctx.fault`, never a return tuple, and the callable ends `return ctx.complete()`. Validation order:

1. `stored_version` in `[1]` and the key set is exactly the five keys: `BDY_RESTORE_VERSION`.
2. Every element of `bodies` and `actors` is a plain integer (not a float, bool, string, null; no missing field): `BDY_RESTORE_TYPE` (this runs before every range check).
3. `n_bodies`, `n_actors` non-negative and column lengths exactly `n_bodies * 6` and `n_actors * 10`: `BDY_RESTORE_COUNT`.
4. Per row, in row order, **first the column ranges (E3)**: every value destined for an int32 column lies inside its bound, `body_id` (actor rows and body rows) in 1 to 2^31 - 1, `seat` in 0 to `BodyIds.MAX_SEAT`, `pos_x` and `pos_y` within ±`BODY_COORD_LIMIT`: `BDY_RESTORE_COLUMN_RANGE` (a wrapping write into a `PackedInt32Array` raises nothing, V9, so without this rule a value at or above 2^31 would pass phase 1, wrap in phase 2 and break "first failure wins" and `snapshot(restore(x)) == x`). Then: `actor_id` at least 1, below `next_actor_id` and at most `ACTOR_ID_KEY_MAX` (`BDY_RESTORE_ACTOR_ID`; a duplicate id is also this code); `kind` is `SEAT` or `AGENT`, `seat` is non-negative, an `AGENT` has seat 0, a `SEAT` has seat at least 1 and no two `SEAT` rows share a seat (`BDY_RESTORE_KIND`); `last_octant` in 0..8 (`BDY_RESTORE_OCTANT`); `abs(sub)` at most 255 (`BDY_RESTORE_SUB_RANGE`); the residue invariant: a non-zero `sub` with `last_octant` NONE, a sign opposing `UNIT[last_octant]`, a non-zero off-axis residue for a cardinal octant, or `abs(sub_x) != abs(sub_y)` on a diagonal octant (`BDY_RESTORE_SUB_INVARIANT`); `odometer` non-negative (`BDY_RESTORE_ODOMETER`); `heading_q == 0` (`BDY_RESTORE_HEADING_UNSUPPORTED`); `abs(world)` and `abs(vel)` within the **current** limits (`BDY_RESTORE_POSE_RANGE`); actor count at most `MAX_ACTORS` (`BDY_RESTORE_CAPACITY`).
5. Body rows strictly ascending by `body_id` (a duplicate is `BDY_RESTORE_ORDER`), actor rows ascending by `actor_id` (`BDY_RESTORE_ORDER`), and no body row with world and vel both `(0, 0)` (`BDY_RESTORE_NONCANONICAL`).
6. Every `body_id` in either table is LIVE in Layout (`body_state == LIVE`; the rows of a DISSOLVED body are rejected, which is why BM8 exists), checked against the already-restored `layout.state` through `BodyLayoutPort` (`restore_after: [layout.state]` guarantees it restored first): `BDY_RESTORE_UNKNOWN_BODY`.

A foot box found overlapping a SOLID rectangle after restore is **not** a failure (a template that changed between builds must not make a save unloadable): BDY-16 applies and the counter runs.

**Why a restore is idempotent and `body.state` is not in `REBASE_BLOCKS` (TR-shipbody-041).** `REBASE_BLOCKS` is the closed list of ADR-0006 § 2.2 / SL-12 of blocks for which `snapshot(restore(snapshot()))` may differ from the stored bytes. For `body.state`: (a) `restore` accepts only canonical rows and never clamps, so every accepted payload re-encodes to itself; (b) the sparse rule (`NONCANONICAL`) removes the one place two payloads could mean the same state; (c) no derived value enters the payload; (d) `post_restore` reads and writes only derived columns. Therefore the round trip is byte-identical for `body.state` alone, the membership question has a fixed answer (**not a member**), and the test is the GDD's: restore and re-snapshot at every tick of a 600-tick run in three phases (cardinal, diagonal with `sub != 0`, stopped), plus a synthetic odometer of 2^53 + 7. This is a per-block idempotence fact; ADR-0006 § 2.2 fixes only the mechanism (§ Related lists the one-line ask).

**`post_restore(ctx)`** (a) derives `pos_prev := pos_curr`, `world_prev := world`; (b) re-checks Layout liveness as a **diagnostic** only (once every block has restored the kernel holds no intact prior state to fall back to); (c) runs the trapped-actor probe (§ 11.4) for every exempt actor, diagnostic only. **A restore never rebuilds the kernel's seat binding itself**: the kernel's `LOADING` pass does, from Body's `binding_source` callable, which puts `PackedInt64Array` pairs `[player_id, actor_id, player_id, actor_id, ...]` in ascending `player_id` (`SEAT` rows only, derived from the table on each call, no persisted field; at the MVP seat == PlayerId, so the `seat` column is the `player_id`, ruling R3) via `ctx.put`; the payload shape is ADR-0016 § 6.2's (a flat `PackedInt64Array`, ascending by `(player_id, entity_id)`), and the earlier open ask to ADR-0002 § 4 is withdrawn (CT-25). `seat` is persisted precisely because the binding table is not saved. `snap_baseline` then runs on entry to `RUNNING`.

**`default_state(ctx)`** puts the empty block (`next_actor_id` 1, no rows). A key held across a restore reads `N` versus `N` in Body's own `last_octant` comparison and the residue continues; a key released during the save/load window resets `sub` exactly as a stop does in uninterrupted play. The claim is block-level G4-exactness; pair-level G4 for Controller-owned state (octant normalisation, the run ramp) is the Controller's.

#### 11.3 The read view (TR-shipbody-029, -043)

`read_view(ctx)` is called once at the registry freeze and puts a long-lived getter-only `BodyView` over the block (with `body_view_empty.gd` as the `empty_view` twin: neutral values, `is_empty()` true). All hot reads are typed scalars or fill a caller-owned buffer, so nothing allocates (`CNT-03` per accessor). A getter given an unknown actor or body returns its neutral value via `_find_actor_row` / `_find_body_row` returning `-1` (§ 4.5). The Controller reads `pos_x(actor)` and `pos_y(actor)` as scalars and builds a `Vector2i` only at its adapter boundary (ADR-0019 § 6.1, CT-27). `body.state` is `SIM`, never `PERCEPTION_OUT`; there is no per-player variant of any Body read; the Perception Filter draws its class-(c) rig at the true position and sanity never touches the block.

| Read (names; scalars unless noted) | Readers |
|---|---|
| `pos_x(actor)`, `pos_y(actor)`; `prev_x(actor)`, `prev_y(actor)` | PRESENTATION; Emitter ACTOR anchor (`pos_curr` is already an integer: the anchor is `pos_curr` itself, no flooring step exists); Lighting shadow origin. **SIM readers of `prev_*` only from a hook ordered after `body.integrate`** |
| `body_of_actor(actor)`, `seat_of_actor(actor)`, `actor_of_seat(seat)`, `kind_of(actor)`, `actor_count()` | Controller, Emitter partition, Lighting masks, Interaction |
| `foot_box_into(actor, out)`, `foot_boxes_into(body, out) -> int` (ascending `actor_id`, **unfiltered, including unbound seats**) | Interaction `validate` (gathers boxes for Layout's `leaf_would_overlap`), Assembly |
| `actor_section(actor) -> section_id or 0`, `actor_local_x(actor)`, `actor_local_y(actor)` | Suit. **Derived on demand** as `section_at(body_of(actor), pos_curr)`; `actor_local_px = pos_curr - section_offset`; never stored, never cached across a mutator; the foot ORIGIN decides; NONE on a Layout body is a defect-only state counted `BDY_TRANSFER_NO_SECTION` (CI asserts zero in a clean build) |
| `actors_overlapping_into(body, rects: PackedInt32Array, out) -> int`; `actors_on_into(body, out) -> int` (ascending) | Assembly `validate` of a PLAYER cut, Transit |
| `world_x/_y(body)`, `vel_x/_y(body)`, `world_prev_x/_y(body)` | `body.projector`, Pipeline canvas offsets, Transit, `docked` (lint `BDY-04`) |
| `docked(a, b, shift_x, shift_y)`, `body_count()`, `odometer(actor)`, `trapped_into(out) -> int` | Assembly, Transit, Presenter (odometer frame index), Debug, Save/Load's autosave guard |
| `moved_x(actor)`, `moved_y(actor)` := `pos_curr - pos_prev`, derived on each call | **SIM-readable only by a hook registered `after: [body.integrate]`** (§ 12 lint `BDY-02`): then it is a difference of two tick-boundary values and G4-exact; before `body.integrate` it would read last tick's value in play and 0 on the first post-restore tick. The reader is `controller.noise` (TR-shipbody-042) |
| `blocked(actor)` := `a_blocked[row] != 0` | **PRESENTATION and DIAGNOSTIC only; never a SIM hook** (lint `BDY-03`): a SIM hook that read it would make it sim state that must be persisted and hashed (a schema bump, deliberately) |

**Neighbour contracts (TR-shipbody-043), stated as Body's half.** Suit calls Layout `room_at(section_id, actor_local_px)` with **section-local** pixels (never body-frame `pos_curr`) and treats a NONE answer, or a plate-cell answer, as exterior exposure; a footprint-resident exempt actor (wall or hull-plate cap) is therefore EXPOSED, while a door-leaf exemption answers the door's `sill_room` and is not. Interaction reads `foot_boxes_into` unfiltered and ascending. Emitter and Lighting anchor on `pos_curr`. Those are the neighbours' own ADRs (0027, 0020, 0017, 0028); Body supplies the reads and fixes their order and domain. The GDD's `PERCEPTION` statement (PCI:216) is unchanged.

#### 11.4 Trapped-actor probe (diagnostic only)

At each detection event (placement, a transfer, `post_restore`, and a weld or cut for every actor on the touched bodies) and on each `trapped_into` read, Body runs for every actor whose start box gives a non-empty `E` a bounded read-only probe: for each of the four axis directions it repeats the admitted 1-px sub-step of § 5.4 up to `TILE_PX` times from the actor's pose, writing nothing, and asks whether the box reaches a pose that intersects no SOLID rectangle. If none of the four directions does the actor is TRAPPED: `BDY_TRAPPED_ACTOR` counts and `trapped_into` lists it. Cost: at most `4 * TILE_PX` overlap tests plus `section_at` calls per flagged actor, once per event and never inside `body.integrate`. The list is recomputed on every read, never cached, not persisted, not hashed, so a stale entry cannot suppress an autosave. The autosave guard is Save/Load's must-repeat (Body will never rescue).

#### 11.5 Events and dirty keys

Dirty keys use `DirtyKey.make(kind, id)` = `(kind << 32) | id` with `BODY 2` (key space of this block; reuse of Layout's number is legal because keys are namespaced by block) and `ACT 4`. **Every tick in which a persisted field changed marks a key** (`quiet_writes_allowed = false`): the hook marks `ACT/<actor_id>` for each actor whose position, `sub`, `last_octant` or `odometer` changed and `BODY/<body_id>` for each body whose `world` changed; mutators mark exactly the keys of the table in § 9 (`ACT/<id>` per moved actor, `BODY/<id>` per touched pose row). At most `MAX_ACTORS` plus `MAX_BODIES` keys per tick, far under `MAX_DIRTY_KEYS`; above it the kernel reports `ALL`. This is stricter than the ADR-0004 § 7 illustrative list that names Body among quiet blocks, and it is chosen so the debug missed-mark audit covers Body (§ Related asks that list to be amended). Keys are marked inside the mutator or hook, last, never from a setter (4.7 packed-array element writes call no setter).

Events go through `scope.emit_event(&"body.state", kind, a0..a5)` and are P5 notifications that carry ids only:

| Kind (id) | a0 | a1 | a2 | a3 | a4 | a5 |
|---|---|---|---|---|---|---|
| `BODY_TRANSFER` (1) | actor_id | old_body | new_body | frame_delta_x | frame_delta_y | cause (WELD 1, DETACH 2, POD 3) |
| `BODY_DOCK_CONTACT` (2) | surviving_body | absorbed_body | 0 | 0 | 0 | 0 |
| `BODY_HULL_SEPARATION` (3) | old_body | new_body | 0 | 0 | 0 | 0 |

**Decision:** three event kinds, with `cause` an argument of `BODY_TRANSFER` (one kind per transfer would triple a subscriber's switch and the `PC_BODY_TRANSFER{detach|pod}` counter keys on cause). The GDD's seven-field `BODY_TRANSFER` record loses `tick` (the batch carries it) and `seat` (a subscriber reads `seat_of_actor(actor_id)` from the view; at the MVP an actor removed in the same batch is not a transfer), and `frame_delta_px` splits into two ints. `frame_delta` is the displacement the transfer applies to `pos_curr`, NEW minus OLD: `+shift` on a weld, `-shift` on a cut, `to_pos - pos_curr(old)` on a pod hop. The Pipeline rebases by it on a weld and a detach (the camera keeps its lag and the avatar's screen position does not move) and cuts on a pod hop or where the new body's bounds clamp; `body.projector` reparents the actor's Presenter node in the same tick (ADR-0034). Events raised in `INIT` and `RESTORE` scopes are discarded (ADR-0004 § 5).

### 12. Determinism, tick indexing, lints and the fixtures that pin them

**Determinism contract (G3, G4).** Order: bodies by ascending `body_id`, pose before actors, actors by ascending `actor_id`. Sweep input order is irrelevant (minimum over candidates) and asserted in debug. No float state, no wall clock, no RNG in the block or the hook (`SIM-02`, `SIM-03`). Tick-source independence: Body is correct for any number of `body.integrate` runs per rendered frame up to `MAX_TICKS_PER_FRAME`; presenters see only the pair of the frame's last tick; four catch-up ticks at the maximum speed move at most 12 px, under the Pipeline's `JUMP_CUT`. Body does not choose the tick source (ADR-0001).

**Tick-indexing convention (TR-horror-033, the Body half).** Seat commands issued in a rendered frame are admitted in P1 and applied in P2 of the next tick `t` that follows; `controller.step` (P3) writes `octant` and `gait_effective` from them; `body.integrate` in the **same tick** reads them and moves the actor; `horror.step` (ADR-0023) is ordered `after: body.integrate` and reads positions of tick `t` after they are written. Pinned by the fixture `TICKINDEX-1`: submit a `set_move_intent` east at frame `f`, run until the first tick that applies it, and assert in that tick (a) `controller.state.octant` is east after `controller.step`, (b) `pos_curr.x` advanced by `v_eff` after `body.integrate`, (c) a probe hook ordered `after: body.integrate` (standing in for `horror.step`) reads the advanced position, (d) a probe hook registered **without** the edge is rejected by the boot lint `STALE_EDGE_UNDECLARED` if it reads `moved`. The rule "a measured input-to-display latency above `INPUT_LATENCY_TICKS` on the RM-DESKTOP and RM-DECK release exports forces `UNFOLD_LO` to 100" is **a data edit to the Horror tuning table, never code**; the S1 export run records the measured latency in its report so the edit has a number to act on (the threshold and `UNFOLD_LO` are Horror's; this ADR owns the convention and the measurement hook).

**Lint pack `BDY-01` to `BDY-08`** (rule ids registered with ADR-0010's `modules.json` / `rules` config; tier E2 lexical unless noted):
- `BDY-01`: no script under `src/sim/body/` names `Node`, `Node2D`, `Camera2D`, `Viewport` (a specialisation of `SIM-01`), and `LocomotionStep` has no reference to `BodyState`. It also bans, in all of `src/sim/**`, the physics and navigation tokens `SIM-06` omits (the full list of § 1, including `RayCast2D`, `ShapeCast2D`, `KinematicCollision2D`, the 2D query-parameter types, `CollisionObject2D`, `CollisionShape2D`, `CollisionPolygon2D`, `direct_space_state`, `get_world_2d` and `collision_enabled`).
- `BDY-02`: the identifiers `moved_x`, `moved_y`, `prev_x`, `prev_y` appear outside `src/sim/body/` and `src/presentation/` only in files listed in `tools/lint/config/body_readers.json` as `{file: hook_id}`; a CI test builds the real registry and asserts every listed hook has `after` containing `{id: body.integrate, tolerates_stale: false}` (TR-shipbody-042; the kernel's topo-sort polices acyclicity, `STALE_EDGE_UNDECLARED` polices the stale flag). The `body_readers.json` lint text lists `controller.noise` as the MVP reader, registered after `body.integrate` (`after` containing `{id: body.integrate, tolerates_stale: false}`) (ruling R10).
- `BDY-03`: the identifier `blocked` (the view getter) appears only under `src/presentation/` and `src/debug/`.
- `BDY-04`: `world_*`, `vel_*`, `world_pose` readers only in `src/sim/web/`, `src/sim/assembly/`, `src/presentation/`, `src/sim/body/` (TR-shipbody-004, TR-webmap-008's "dormant poses are never read by the simulation").
- `BDY-05`: `SweepOracle` is referenced only from `tests/` and `src/sim/body/testing/`.
- `BDY-06`: every function in the `## @hot` list below is `HOT-01` clean (this is `HOT-01` applied; the list is the content).
- `BDY-07`: no `Callable`-typed member or parameter in `src/sim/body/` except the kernel-facing registered callables of `BodyModule.setup` (the sink and collide seams are ports).
- `BDY-08`: `BodyTuning` literals: no numeric literal equal to a registered Body knob appears in `src/sim/body/` outside `locomotion_tables.gd` and tests (the data-driven rule).

**Fixtures (named so stories can cite them):** `fixture_airlock` (abutting exempt rectangles at the hull line, D.11), `fixture_chain` (chain 1 `(0, 0)`, 3 `(576, 96)`, 4 `(1056, 96)` cut at the (1, 3) seam, new offsets 3 `(0, 0)` and 4 `(480, 0)`: the detach re-base), `TICKINDEX-1`, `INERTIAL-1` (600 ticks with `vel` set), `SWEEP-EQ` (the 48 by 48 by 49 by 5 grid, fed through the span encoding, plus the 1-px-wide leaf), `MUT-CONS` (the generated mutator sequence of § 9), `BODY-RT` (the three-phase restore round trip, plus the out-of-range fixture with `2^31` and `2^32 + 5` in every int32 column of a decoded payload, which must fail as `BDY_RESTORE_COLUMN_RANGE` with the live block untouched, and the restore-then-read-through-a-pre-held-`BodyView` case), `GOLDEN-1` (the canonical bytes of a fixed payload, AC-BDY-110), `LOCO-IDLE-1` (`v_eff == 0` with octant NONE on the hot form `step_into`, which must return at step 2a without indexing a table), `ROW-ABSENT-1` (absent ids at the first, middle, last and past-the-end positions of both tables, § 4.5).

**MVP-required versus landing with the first triggering story (T12; ADR-0010 § 6.10 lands rules with their owner's first story).** Required at the Body epic's first story: `SIM-06` plus `BDY-01`, `BDY-02`, `BDY-05`, `SWEEP-EQ`, `BODY-RT` and `MUT-CONS`. Landing with their first triggering story: `BDY-07` and `BDY-08`, the `BODY_SWEEP_CHECK` CI soak, the bounds ledger and its `BODY_BOUND_LOWERED` lint (first needed when a bound is first lowered), `BDY-03`, `BDY-04` and `BDY-06` (with the first reader or `## @hot` function that needs them). The probe scripts of V4 are committed under `tests/support/probes/adr-0015/` as task 0.

### 13. Performance, counting gates and the `## @hot` list (TR-shipbody-034, -046)

**Budget, stated once (CT-20).** **The ledger entry is the whole `body.integrate` hook: 0.05 ms per actor P99, 0.4 ms for eight, pending S1. It includes ADR-0014 `solid_in` calls, the clamp, the row copy and the dirty marks. S1 reports the whole-hook figure, with `solid_in` time as a sub-counter.** ADR-0014's per-actor `solid_in` figure is therefore a share inside this entry, not an addition to it, and this entry is Body's row in the ADR-0003 P3 share ledger (the Core hooks on the worst-aligned tick already sum to about 1.7 ms of the 2 ms target, so no cost may be charged twice). `body.integrate` is budgeted as **one hook** (pose step, every actor step including the exempt worst case, every `solid_in` it issues, commit, dirty marking, diagnostics), not per actor: the cited figure is eight actors at 0.05 ms each, 0.4 ms, 20 % of the kernel's 2 ms P3 target; four actors at 0.2 ms is 10 %; a catch-up frame of `MAX_TICKS_PER_FRAME` ticks is four hooks back to back (1.6 ms on the cited figure, 27 % of `FRAME_SIM_BUDGET_MS`). `STEP_BUDGET_MS` is only a logging threshold. **Every wall-clock figure is pending spike S1** on RM-DESKTOP and RM-DECK in a release export (Godot 4.7.2 `template_release`, `forward_plus`, vsync on, the Pipeline's 120-frame warm-up, with the renderer and driver read-back recorded): **P99 at most 90 % of the frame ceiling** (16.6 ms desktop, 33.3 ms deck), per-actor sweep logged above `SWEEP_COST_LOG_MS`. The developer machine is indicative only. The probe numbers of V4 (2.4 versus 23.8 microseconds per actor-tick at K = 16) are **not** a gate and are not evidence the hook fits; they show only that the analytic clamp is about an order of magnitude cheaper than the stepper on the editor binary, which is why it is the baseline.

**CI gates are deterministic counts, not time (TR-shipbody-046).** The test build wraps `CollideSource` in `CountingCollide` (a double extending `CollideSource` that counts `query_into(body_id, ...)` and `origin_in_body(body_id, ...)` calls and delegates) and reads `LocoResult.rect_tests` / `section_tests`; the fixtures W1 to W5 of D.12 assert: **at most one `query_into` per moving actor per tick and none for a stationary one**; at most `3 * K` rectangle tests for a moving, non-exempt actor; at most `K + n_sub * |E|` overlap evaluations and `n_sub <= 6` `origin_in_body` calls for an exempt actor; `section_at` calls bounded by `4 * TILE_PX` per flagged actor per detection event and **zero** per ordinary tick (W2/W2b/W2c, W5). Wall-clock P99 is a **nightly benchmark** that fails at +25 % over the S1 baseline per machine. The `CNT-01` (object count unchanged over 600 ticks), `CNT-02` (allocation within tolerance on the debug binary) and `CNT-03` (per accessor) assertions run in the suite that exercises the hook.

**`## @hot` list (this ADR owns it, ADR-0010 § 7):** `BodyModule._hook_integrate`, `LocomotionStep.step_into`, `AxisClamp.clamp_x`, `AxisClamp.clamp_y`, `AxisClamp.exempt_axis`, `BodyView` scalar getters (`pos_x`, `pos_y`, `prev_x`, `prev_y`, `body_of_actor`, `seat_of_actor`, `actor_of_seat`, `world_*`, `vel_*`, `moved_*`, `blocked`, `odometer`, `actor_section`, `actor_local_*`), `BodyView.foot_boxes_into`, `actors_overlapping_into`, `actors_on_into`, `LayoutCollide.query_into`, `LayoutCollide.origin_in_body`, `RectSpans.x0/y0/x1/y1`, `BodyModule._find_actor_row`, `BodyModule._find_body_row`. Mutators and `snapshot`/`restore` are cold (they may allocate).

### Key Interfaces

```gdscript
# src/sim/body/body_ids.gd  (structural constants; saved meaning, never tuned)
class_name BodyIds extends RefCounted
const KIND_SEAT: int = 1
const KIND_AGENT: int = 2
const K_BODY: int = 2                      # dirty-key kinds of this block's KeySpaceDescriptor
const K_ACT: int = 4
const EV_BODY_TRANSFER: int = 1
const EV_BODY_DOCK_CONTACT: int = 2
const EV_BODY_HULL_SEPARATION: int = 3
const CAUSE_WELD: int = 1
const CAUSE_DETACH: int = 2
const CAUSE_POD: int = 3
const ACTOR_ID_KEY_MAX: int = 0xFFFFFFFF
const INT32_MAX: int = 0x7FFFFFFF         # bound of body_id and of MAX_SEAT; every int32 column value is checked against its bound before the write
const MAX_SEAT: int = INT32_MAX           # a narrower PlayerId bound (ADR-0002) is enforced by the Controller and may lower this only with a ledger row
const CNT_ORPHAN_ACTOR: int = 0   # ... one index per counter: OCTANT_INVALID, VEFF_INVALID, SEAT_UNBOUND, WORLD_LIMIT,
                                  # TRANSFER_NO_SECTION, PLAN_MISMATCH, TRAPPED_ACTOR, FOOTPRINT_OVERLAP, SOLID_IN_UNORDERED, AGENT_UNSTEPPED, QUERY_DEFECT

# src/sim/body/body_faults.gd  (domain 5; pre-packed; append-only; NAMES indexed by local code)
class_name BodyFaults extends RefCounted
const DOMAIN_ID: int = 5
const BDY_RESTORE_VERSION: int = (DOMAIN_ID << 16) | 1
# ... through 15 (_UNKNOWN_BODY), then
const BDY_PLACE_UNKNOWN_BODY: int = (DOMAIN_ID << 16) | 16
# ... through 31 (BDY_POSE_OUT_OF_RANGE), then
const BDY_APPLY_PRECONDITION: int = (DOMAIN_ID << 16) | 32
const BDY_RESTORE_COLUMN_RANGE: int = (DOMAIN_ID << 16) | 33
const BDY_RELEASE_ACTORS_PRESENT: int = (DOMAIN_ID << 16) | 34
const BDY_RELEASE_BODY_NOT_DISSOLVED: int = (DOMAIN_ID << 16) | 35
const NAMES: Array[StringName] = [&"", &"BDY_RESTORE_VERSION" ]  # indexed by local code; complete list in the file

# src/sim/body/ports/*.gd
@abstract class_name CollideSource extends RefCounted      # Body's port; LayoutCollide holds a SolidQuery (ADR-0014); no bind
@abstract func query_into(body_id: int, ax0: int, ay0: int, ax1: int, ay1: int, out: QueryBuffer) -> int   # inclusive span records
@abstract func origin_in_body(body_id: int, x: int, y: int) -> bool

@abstract class_name BodyLayoutPort extends RefCounted     # implemented by an adapter over Layout's LayoutViews
@abstract func body_is_live(body_id: int) -> bool                         # == (body_state(body_id) == LIVE)
@abstract func body_state(body_id: int) -> int                            # LIVE / DISSOLVED / UNKNOWN (ADR-0013 reads owed, CT-17)
@abstract func section_at(body_id: int, x: int, y: int) -> int            # 0 = NONE
@abstract func section_contains(section_id: int, local_x: int, local_y: int) -> bool
@abstract func section_offset_x(section_id: int) -> int
@abstract func section_offset_y(section_id: int) -> int
@abstract func body_of(section_id: int) -> int

@abstract class_name PlanMismatchSink extends RefCounted
@abstract func report(mutator_id: int, code: int) -> void

# src/sim/body/views/place_result.gd
class_name PlaceResult extends RefCounted
var actor_id: int

# src/sim/body/locomotion/*.gd : LocoState (with body_id), LocoIntent, LocoResult, LocomotionStep (section 6)
# src/sim/body/locomotion/locomotion_tables.gd: structural constants, declared in the form that compiles on 4.7.2
class_name LocomotionTables extends RefCounted
const FP_ONE: int = 256
const OCTANT_NONE: int = 8
const UNIT_X: PackedInt32Array = [256, 181, ...]   # values elided here: generated from the registry and golden-tested, size 8
const UNIT_Y: PackedInt32Array = [...]
const ODO_W: PackedInt32Array = [...]   # indexed only with o in 0..7 (step 2a returns before OCTANT_NONE)

# src/sim/body/sweep/rect_spans.gd: the one span-to-half-open conversion (section 5.1)
class_name RectSpans extends RefCounted
static func x0(s: PackedInt32Array, k: int) -> int: return s[4 * k]
static func y0(s: PackedInt32Array, k: int) -> int: return s[4 * k + 1]
static func x1(s: PackedInt32Array, k: int) -> int: return s[4 * k + 2] + 1
static func y1(s: PackedInt32Array, k: int) -> int: return s[4 * k + 3] + 1

# BodyModule private lookups (section 4.5): _find_actor_row(id) -> int and _find_body_row(id) -> int, -1 when absent
# src/sim/body/body_module.gd : BodyModule with setup and the can_*/mutator list (section 9)
# src/sim/body/views/body_view.gd : BodyView (section 11.3)
```

(Each `class_name` is its own file. Signatures are the contract; bodies are the Body epic's.)

### Implementation Guidelines

- **Must** keep every `src/sim/body/` script `RefCounted`, scene-tree free and float-free; **must never** name a physics type, `Node`, a clock, an RNG or `randi`/`randf` (lint `SIM-01`, `SIM-02`, `SIM-03`, `SIM-06`).
- **Must** divide only through `IntMath` (`trunc_div` for the residue, `floor_div` for px-to-tile); **must never** write `/`, `%`, `min`, `max`, `abs`, `sign` or `clamp` in `src/sim/body/`.
- **Must** register `body.state` with `interpolated = true`, `gameplay = true`, `restore_after [layout.state]`, the hook with both `after` edges, the key space with `quiet_writes_allowed = false`, and fault domain 5; every kernel-invoked callable ends `return ctx.complete()`.
- **Must** write every mutator as `name(scope, ..., [out]) -> int` with `scope.open_for` first, the **whole** predicate before the first write, a defect that calls the sink, `scope.fault` and returns `-code`, and keys marked last; **must never** use `assert()`; **must never** call `scope.fault` on BM4's saturating branch.
- **Must** keep the persisted and derived columns in lock-step through the two private row functions; **must never** cache a row index, an `actor_section` answer or a `Dictionary` key across a mutator or a restore.
- **Must** copy columns into the snapshot payload (`Array` of ints, five keys); **must never** return a column, alias one, use `var_to_bytes` for a hashed byte or write a canonical byte writer.
- **Must** make exactly one `query_into` per moving actor per tick and none for a stationary one; **must never** ship the 1-px stepper in a non-test path.
- **Must** read only `octant` and `gait_effective` (and `has_row`) from the Controller; **must never** read `loco_state`, derive a gait or write any `controller.state` field.
- **Must** treat the rectangle set as read-only and as the only collision truth; **must never** inflate it, add a second set, push, shove, snap or eject an actor.
- **Must** keep `world_*` and `vel_*` out of every body-frame computation; **must never** let a SIM hook read `blocked`, or read `moved_*` / `prev_*` unless it is `after: [body.integrate]`.
- **Must** obtain every Body knob from `BodyTuning` and refuse a lowered bound without a `schema_version` bump (`BODY_BOUND_LOWERED`); **must never** hold a literal for a registered knob.
- **Must** cite `docs/registry` names for constants and **must never** retype a registry value.
- **Must** keep injected ports and tuning in private members (`_collide`, `_layout`, `_sink`, `_tuning`) and prefix every intentionally unused kernel-signature or test-double parameter with an underscore (`snap_baseline(_key: int, ctx)`, `_hook_integrate(views, scope, ctx)` when `views` is unused), because `SHADOWED_VARIABLE`, `SHADOWED_VARIABLE_BASE_CLASS`, `SHADOWED_GLOBAL_IDENTIFIER` and `UNUSED_PARAMETER` are errors (ADR-0010 § 5; probed); **must never** name a port method `bind` or carve a `HOT-01` exemption.
- **Must** declare table constants as `const NAME: PackedInt32Array = [..]` (or `Array[int]`); **must never** write `const NAME := PackedInt32Array([..])` or `PackedInt32Array()` in a `const`, and **must** treat a const array as read-only.
- **Must** find every row through `_find_actor_row` / `_find_body_row` and treat `-1` as absent (neutral getter value, `BDY_UNKNOWN_ACTOR` from a mutator); **must never** read `a_id[row]` from a raw `bsearch` index.
- **Must** range-check every value destined for an int32 column before the write (`BDY_RESTORE_COLUMN_RANGE`, `BDY_ARG_RANGE`, `BDY_APPLY_PRECONDITION`); **must never** rely on a narrowing warning, because a packed-array element write wraps silently.
- **Must** store `blocked` as a byte (`a_blocked[i] = 1 if r.blocked else 0`); **must never** cache a packed column in a member of `BodyView`, the hook or `LayoutCollide`, or replace a column reference in `restore` (resize and copy instead).
- **Must** decode every `QueryBuffer` record through `RectSpans`; **must never** index `spans` elsewhere or read an inclusive last cell as a half-open end.
- **Must** floor before assigning to `Node2D.position` (presentation side) and **must never** pass a negative operand to `>>` in `src/sim/body/`.

## Alternatives Considered

### Alternative 1: One `CharacterBody2D` per actor (or one `RigidBody2D` per ship) with engine collision shapes built from Layout
- **Description**: Build `CollisionShape2D`s from Layout's rectangles and move actors with `move_and_slide`.
- **Pros**: Free sliding, no custom sweep code, familiar to Godot users.
- **Cons**: Contradicts four designed GDDs; G3 and G4 would depend on a recorded physics replay (the server is not an integer system); a shape rebuild at every weld and every cut is unmeasured; engine collision needs margins and a safe-margin parameter that is a second rectangle set; rotation can leak inside the ship (Pipeline C2); the position would no longer be a pure function of the command stream.
- **Rejection Reason**: Determinism and the Pillar 1 rule that collision, light and drawing read one rectangle set. Not a close call.

### Alternative 2: Engine-physics exterior hybrid (the ship's pose from a physics body, interiors on integers)
- **Description**: A `RigidBody2D` carries the hull while actors integrate in its frame.
- **Pros**: Real inertia and collision for hulks.
- **Cons**: A second authority for the pose; the pose would not be reproducible; Web Map & Transit already scripts constant-velocity legs; "a tumbling hulk" is an exterior sprite, never a simulation fact.
- **Rejection Reason**: Two authorities for one fact. Exterior motion is `world += vel` on integers.

### Alternative 3: Ship the literal 1-px stepper
- **Description**: Test the destination box against every candidate for each of up to six sub-steps.
- **Pros**: Trivially correct, the specification itself, no equivalence proof needed.
- **Cons**: About `7 * K` rectangle tests per moving actor-tick against about `3 * K` for the clamp; the probe measured 23.8 versus 2.4 microseconds per actor-tick at K = 16 on the editor binary (indicative, not a gate), and in interpreted GDScript the tests dominate the hook.
- **Rejection Reason**: Cost. It is kept as the test oracle and as a debug cross-check mode, so correctness does not depend on trusting the clamp.

### Alternative 4: Continuous swept-AABB with time of impact (float or fixed-point)
- **Description**: Compute a fractional hit time per rectangle and move to it.
- **Pros**: One computation regardless of step size.
- **Cons**: Needs division and fractional state, which the integer rules forbid; a rounding rule at the contact introduces a second residue; the result is not provably equal to the designed 1-px behaviour (a 1-px sliver, the first-legal-row corollary).
- **Rejection Reason**: The step is at most 3 px, so a per-axis integer clamp is exact and needs no fractions.

### Alternative 5: Array of `RefCounted` actor records instead of struct-of-arrays
- **Description**: One object per actor with typed fields, kept in an `Array[ActorRow]`.
- **Pros**: Readable; each actor's fields travel together.
- **Cons**: One object dereference per field read in the hook; a per-row copy in the snapshot; the derived columns and the persisted ones would live on the same object, weakening the persisted/derived split that the reflection test enforces.
- **Rejection Reason**: SoA is the working assumption of the GDD and the cheaper hot loop; the snapshot shape is independent, so spike S1 can reverse it without a schema change (V6, OQ-1). Not a one-way door.

### Alternative 6: Body derives sections by scanning, or caches `actor_section`
- **Description**: Keep a per-actor section id updated each tick.
- **Pros**: O(1) read for Suit.
- **Cons**: A stale value across a mutator (Suit reads after P2 of the next tick may have run a transfer); a second containment implementation to keep equivalent with Layout's.
- **Rejection Reason**: `section_at` is Layout's (ANSWERED in the GDD); Body derives on demand and never caches.

### Alternative 7: Push out, eject or snap an actor that ends inside a footprint
- **Description**: Resolve a door-leaf closure or a cut by displacing the actor to the nearest free cell.
- **Pros**: Never leaves a trapped actor.
- **Cons**: An unannounced displacement the player did not cause (Pillar 2); contradicts PCI:650 "the body is never displaced" and LY-11; hides the authoring or Assembly defect that produced the overlap.
- **Rejection Reason**: Struck by the design. The guards are Assembly's refusal and deferral, BM4's apply-time refusal and the exempt-actor predicate.

### Alternative 8: Position in the Controller block (PC-14 the other way)
- **Description**: `controller.state` owns the actor position and calls `LocomotionStep`.
- **Pros**: Fewer blocks.
- **Cons**: A pod hop or a detach changes the actor's body, which only Body tracks; the Layout projection inside an input system inverts the layering; replay would need the Controller's block to carry Layout knowledge.
- **Rejection Reason**: ADR-0019 § 1.2 ratifies PC-14; Body owns `pos`, `sub`, `odometer`.

### Alternative 9: Roll `pos_prev` in `commit_baseline` at P4
- **Description**: Let the kernel's baseline roll maintain the pair, as a block with a plain current value would.
- **Pros**: Uses the kernel mechanism as written.
- **Cons**: At P5, when presenters read, `prev == curr`, which erases interpolation; it would also zero `moved` before any P5 reader and break the same-tick `controller.noise` contract.
- **Rejection Reason**: The roll is in the hook and the mutators; `commit_baseline` is a documented no-op. The cost is an ADR-0003 clarification (§ Related).

### Alternative 10: Packed arrays in the snapshot payload
- **Description**: Emit `PackedInt32Array` / `PackedInt64Array` columns.
- **Pros**: Matches ADR-0011 and ADR-0013; fewer variants on the binary path.
- **Cons**: Contradicts TR-shipbody-008's "plain Arrays" and hides the type-walk failure class on the binary path. (The JSON dump is diagnostics only and never a load path, ADR-0006 § 1.3; the alternative is not engine-blocked, ADR-0005 encodes typed and untyped arrays identically.)
- **Rejection Reason**: The payload is a few hundred bytes; plain Arrays keep the TR text and the `BDY_RESTORE_TYPE` check meaningful. Reversible with a schema bump and a golden-file update.

### Alternative 11: `Callable` for `collide` and for the mismatch sink
- **Description**: Bind `solid_in` and the sink as stored `Callable`s.
- **Pros**: No port class.
- **Cons**: `Callable.call(` is a `SIM-12` finding; `Callable(` / `.bind(` are `HOT-01` tokens; no typed signature; harder to double.
- **Rejection Reason**: `@abstract` ports are ADR-0010's pattern and cost one direct method call.

### Alternative 12: Snap each body's canvas offset to the 24-px grid, or hide a docking jump behind the camera cut
- **Description**: Make a non-docked weld invisible to the exterior layer.
- **Pros**: Welds could be offered anywhere.
- **Cons**: Snapping moves a hull the player sees move; the camera cut hides camera motion, not the absorbed hull's jump (struck by the GDD's re-review).
- **Rejection Reason**: The offer gate (`docked`) is the only guard; it gates the offer, never Body's validate.

## Consequences

### Positive
- No physics server, body or area exists on any simulation path, so G3 and G4 do not depend on a recorded physics replay and a weld or cut rebuilds no engine shape.
- One implementation of the kinematics (`LocomotionStep.step_into`) serves Body now and Horror later; the value form and the hot form return identical results by construction.
- The analytic clamp is an order of magnitude cheaper than the literal stepper on the editor binary and is proven equal to it by a committed property test and, in the probe, on 300,000 random fixtures; the oracle remains in the repository as a debug cross-check.
- `body.state` is a plain, tiny, canonical block: five keys, a closed fault domain, an idempotent restore with a fixed answer for `REBASE_BLOCKS`, and a post-mutator invariant checked after every call, including faulted ones.
- A cut cannot entomb an actor (BM4 refuses with an apply-time net behind Assembly's V5), and nothing in the simulation ever displaces an actor.
- Reasons and faults are separated, so Assembly's `validate` reasons and the kernel's fault channel no longer share a namespace by accident.
- The performance gates are deterministic counts that run in every CI build; wall-clock stays a nightly benchmark.

### Negative
- `interpolated = true` with a no-op `commit_baseline` departs from the plain reading of ADR-0003 and needs a one-line clarification there.
- SoA means two parallel column sets that must stay index-aligned; the two row functions and the alignment test are the cost.
- Plain-Array snapshots differ in style from ADR-0011 and ADR-0013 (deliberate, § 11.1).
- The exempt path is a literal per-sub-step predicate, not a closed form; it is bounded (six sub-steps) and defect-only, but it is code that the analytic clamp does not cover.
- BM1's return convention and the frame-shift argument shape change from the GDD, and the GDD, TR-shipbody-020 and four consumer GDDs must be synced.
- Body now depends on things from siblings that are drafts and must be co-accepted: ADR-0013's `body_state`, `body_is_live` and `section_contains` reads, ADR-0014's `SolidQuery.solid_in` and `QueryBuffer`, and ADR-0019's `ControllerView.has_row` (ADR-0015 and ADR-0019 are a co-acceptance pair).

## Risks
- **R1: GDScript cost of the whole hook is unmeasured (V5).** All wall-clock claims are UNVERIFIED. *Mitigation*: S1 on both reference machines before the hook story closes; the count gates catch algorithmic regressions meanwhile; the rung-3 GDExtension escape hatch (ADR-0010 § 9) is available only for the Graph at VS-A and is not assumed here.
- **R2: The clamp and the stepper diverge on an input the fixtures do not cover.** *Mitigation*: the committed grid (48 by 48 by 49 by 5), the random-fixture soak in CI with `BODY_SWEEP_CHECK`, and the oracle kept in-tree; any difference is a test failure.
- **R3: `solid_in`'s order or content differs from ADR-0014's published shape once it is implemented (ADR-0014 is a draft).** *Mitigation*: the sweep is order-independent; `LayoutCollide` is the single adapter; the unordered debug signal; a contract test against the real ADR-0014 implementation.
- **R4: Index misalignment between persisted and derived columns after a mutator.** *Mitigation*: only two row functions touch both; the alignment test after every mutator, restore and hook run.
- **R5: A mutator faults mid-apply and leaves Graph and Layout applied but Body not.** *Mitigation*: this is the documented no-rollback property of ADR-0003 § 5 (a fault is terminal until reload); Body's check-then-write guarantees only that Body itself is never half-applied; ADR-0021 may snapshot and restore the four touched blocks.
- **R6: Float32 canvas bound fails on a release export.** *Mitigation*: V3 is probed on the editor build; the repeat rides ADR-0008's spike; the fallback is a sign-split or a lowered limit with a schema bump (§ 2 ledger).
- **R7: A late consumer adds a SIM read of `blocked` or `world_*`.** *Mitigation*: lints `BDY-02` to `BDY-04`; the invariance fixture.
- **R8: A save made with a larger `MAX_ACTORS` is rejected by a build with a smaller one (`BDY_RESTORE_CAPACITY`)** against the principle that a build change must not make a save unloadable. *Mitigation*: the bounds ledger refuses a lowered bound without a schema bump; Open Question 13 of the GDD is answered by treating the cap as a save-compatibility fact.
- **R9: The `controller.noise` contract silently regresses if a hook drops its `after` edge.** *Mitigation*: `BDY-02` plus the registry-level CI test, and `TICKINDEX-1`.
- **R10: Stale reasoning about row indices.** *Mitigation*: no cached index, section id or Dictionary key across a mutator (a code-review checklist item and a test that mutates between reads); an absent id is detected by `_find_actor_row` (§ 4.5), and a cached packed-column alias is forbidden (§ 4.2).
- **R11: A Layout mutator dissolves a body without a Body follow-up, leaving orphan rows that make the next autosave unloadable.** *Mitigation*: BM8 `release_body` after LM4, the discard sequence recorded as an assumption on ADR-0026, I3 stated as an invariant of the whole apply and LM2 / LM4 in the `MUT-CONS` generator.

## GDD Requirements Addressed

All rows are the TR ids whose Planned ADR column names ADR-0015 in `docs/architecture/requirements-traceability.md` (35 rows, including the three re-tagged 2026-10-29), plus the three rows that cite ADR-0015 only in a note (loaded or checked here, decided elsewhere). The 2026-10-29 review adds TR-shipbody-005, -011 and -027 (Shared) and TR-shipbody-010 and -037 (Owned elsewhere), whose Foundation or sibling notes defer the residue to the Body ADR or which this ADR in fact decides or cites. Registry text is `docs/architecture/tr-registry.yaml`.

### Decided here (26)

| TR | GDD system | Requirement (short) | How this ADR addresses it |
|---|---|---|---|
| TR-pc-013 | player-controller-input.md | Movement is hand-rolled integer kinematics (`LocomotionStep`, 1/256 fixed point, axis-separated sweeps, diagonal 181/256), no physics body | § 1 (no engine physics), § 6 (`LocomotionStep`, `trunc_div`, `FP_ONE`), § 5 (axis-separated clamp); position is a pure function of command stream and Layout |
| TR-shipbody-001 | ship-body-and-interior-physics.md | No physics server or node on any simulation path; lint finds zero matches | § 1: `SIM-06` over all of `src/`, no waiver, plus the `BDY-01` supplementary tokens; reflection type test; debris exemption deferred to ADR-0033 on a presentation path |
| TR-shipbody-003 | ship-body-and-interior-physics.md | One body per welded set keyed by Layout's `body_id`; Body stores only pose, velocity, `heading_q`, actor rows and never allocates or edits `body_id` or geometry | § 4.1 (tables), § 9 (mutators write only Body rows), `BDY-01` and a grep criterion for Layout-owned names as Body-written fields |
| TR-shipbody-004 | ship-body-and-interior-physics.md | Nothing keyed on a body-frame coordinate reads `world_*` or `vel_*`; 600 ticks with `vel` gives a byte-identical `actors` encoding | § 10: fixture `INERTIAL-1`, lint `BDY-04` |
| TR-shipbody-006 | ship-body-and-interior-physics.md | Persisted schema v1: per-actor row, sparse per-body row, `next_actor_id`; `pos_prev`, `blocked`, diagnostics derived and never persisted or hashed | § 4.1 (persisted columns), § 4.2 (derived columns), § 11.1 (payload excludes them), reflection and zero-and-diff tests |
| TR-shipbody-012 | ship-body-and-interior-physics.md | Bodies integrate pose first in ascending `body_id`, then actors in ascending `actor_id`; a stationary actor issues no query | § 7 (hook order and the no-query rule) |
| TR-shipbody-013 | ship-body-and-interior-physics.md | The hook is total: data defects absorbed deterministically and counted, never FAULT | § 7 (defect table and counters; freeze form for `BDY_WORLD_LIMIT`) |
| TR-shipbody-014 | ship-body-and-interior-physics.md | Allocation-free per tick: id-sorted SoA, caller-owned scratch, `solid_in`; `step_into` and `step` identical | § 4.2, § 5.1, § 6, § 7, § 13 (`## @hot` list, `HOT-01`, `CNT-01` to `CNT-03`) |
| TR-shipbody-015 | ship-body-and-interior-physics.md | `LocomotionStep` is a pure scene-tree-free RefCounted function `step(state, intent, collide)`; `trunc` for the residue, FLOOR only for px-to-tile | § 6 (types, signatures, rounding helpers), § 2 (Horror reuse by `class_name`) |
| TR-shipbody-016 | ship-body-and-interior-physics.md | Body reads only `octant` and `v_eff`; residue resets only when the move octant differs from Body's own `last_octant` | § 6 (reset predicate, the two fields; the `intent_octant` mismatch with ADR-0019 is recorded) |
| TR-shipbody-017 | ship-body-and-interior-physics.md | Foot box `[pos.x-8, pos.x+8) x [pos.y, pos.y+8)`, half-open, top-centre origin; collision against `solid_in` exactly, no margin, no second set | § 5.1 (`CollideSource`, one rectangle set), § 5.2 (box and cover from `BodyTuning` `FOOT_BOX_W/H`), § 5.3 (first-legal-row corollary) |
| TR-shipbody-018 | ship-body-and-interior-physics.md | One `solid_in` query per moving actor per tick swept analytically; analytic clamp is the baseline, 1-px stepper the oracle, proven identical by property test | § 5.2 to § 5.5 (clamp, oracle, `SWEEP-EQ`, probe V4, `BODY_SWEEP_CHECK`) |
| TR-shipbody-019 | ship-body-and-interior-physics.md | An actor starting inside SOLID rectangles may only take non-increasing per-rectangle sub-steps with destination origin in its own body; never pushed; actors do not collide | § 5.4 (exempt predicate, per-rectangle test, exit containment, no displacement); actors do not block one another (§ 7) |
| TR-shipbody-020 | ship-body-and-interior-physics.md | BM1 to BM7 are Class A, called after the Layout mutator, check-then-write, return OK or negative FAULT_CODE, no `assert()`, injectable sink counting `BDY_PLAN_MISMATCH` | § 9 (shapes under ADR-0003 § 5: BM1's id moves to `out`; `PlanMismatchSink` port; `scope.fault`; the non-faulting BM4 saturation; BM8 `release_body` is added to the TR's seven) |
| TR-shipbody-021 | ship-body-and-interior-physics.md | Each mutator has a pure `can_*` read the caller's `validate` invokes first | § 9 (seven `can_*` signatures, reasons as StringNames) |
| TR-shipbody-023 | ship-body-and-interior-physics.md | BM3 shifts only absorbed-body actors by `+frame_shift_px`, keeps `sub`/`last_octant`, deletes the absorbed pose row; `(actor_section, actor_local_px)` unchanged | § 9 BM3 and its invariant test |
| TR-shipbody-024 | ship-body-and-interior-physics.md | BM4 assigns by the section containing the foot origin against reconstructed pre-cut geometry, shifts `C_new` by `-frame_shift_px` with `sub` zeroed, creates the pose row, refuses with `-BDY_REBASE_ACTOR_IN_CAP`; out-of-range pose row saturates and returns OK | § 9 BM4 (order, `section_contains`, cap guard, saturation without `scope.fault`) |
| TR-shipbody-025 | ship-body-and-interior-physics.md | Weld legality takes no pose input; `docked` is a pure read gating nothing in Body; a PLAYER weld is offered only when `docked` holds | § 9 (closing paragraph), § 11.3; must-repeat for ADR-0021 and ADR-0026 |
| TR-shipbody-026 | ship-body-and-interior-physics.md | Integer whole-px pose and velocity within bounds; `heading_q` 0; lowering a bound needs a schema bump; `DETACH_DRIFT_PX_PER_TICK` locked at 0 | § 2 (schema range `[0, 0]`, bounds ledger `BODY_BOUND_LOWERED`), § 10 |
| TR-shipbody-028 | ship-body-and-interior-physics.md | `actor_section` equals `section_at(body_of(actor), pos_curr)`, derived on demand and never cached; `actor_local_px = pos_curr - section_offset`; no section is a counted defect | § 11.3, § 4.2 (no cached index or section) |
| TR-shipbody-029 | ship-body-and-interior-physics.md | `body.state` is SIM, never PERCEPTION_OUT; its read view is readable by PRESENTATION; `blocked` only by PRESENTATION and DIAGNOSTIC | § 3 (`block_class`), § 11.3, lints `BDY-02`, `BDY-03` |
| TR-shipbody-034 | ship-body-and-interior-physics.md | Hook cost budgeted as a whole, pending S1 on RM-DESKTOP and RM-DECK; P99 at most 90 % of the ceiling; 8 actors cited at 0.4 ms of the 2 ms P3 target | § 13 (the whole-hook ledger entry, including the `solid_in` calls; S1 prerequisite, no figure asserted) |
| TR-shipbody-040 | ship-body-and-interior-physics.md | After every mutator call, including faulted or saturated ones, the block is consistent | § 9 (invariants I1 to I5 and the `MUT-CONS` property test; check-then-write) |
| TR-shipbody-041 | ship-body-and-interior-physics.md | `body.state` is excluded from `REBASE_BLOCKS` because its round trip is idempotent | § 11.2 (the four-point idempotence argument, `BODY-RT`) |
| TR-shipbody-042 | ship-body-and-interior-physics.md | A SIM hook may read `moved` or `pos_prev` only if registered `after: [body.integrate]`; a lint rejects any other SIM reader | § 11.3, § 12 lint `BDY-02` and the registry CI test; `controller.noise` is the reader |
| TR-webmap-008 | web-map-and-transit.md | Body pose is set only via BM7 inside atomic applies; re-origins are pure translations; dormant bodies' poses are never read by the simulation | § 9 (BM7, callers), § 10 and lint `BDY-04` (the `REORIGIN_IS_TRANSLATION` lint itself is ADR-0026's) |

### Shared: this ADR decides its half (12)

| TR | GDD system | Requirement (short) | Body's half decided here; the other half |
|---|---|---|---|
| TR-pc-014 | player-controller-input.md | Speeds are whole base px per tick; `v_max = v_run` must satisfy invariants I1 and I2; the loader refuses data violating any of the 19 invariants | § 6 (`v_eff` range check, `v_eff_max` constructor argument asserted equal to the Controller's `v_run` at load); the 19-invariant loader and the camera constants are ADR-0019, ADR-0007 and ADR-0008 |
| TR-pc-035 | player-controller-input.md | `controller.step` detects a body change and clears channel fields; `pos_prev == pos_curr` and `sub` reset on a transfer | § 9: BM3, BM4 and BM5 all set `pos_prev := pos_curr`; BM4 (new side) and BM5 zero `sub`; **BM3 keeps `sub` and `last_octant`** by TR-shipbody-023 (a whole-pixel 24-grid shift cannot disturb a residue). TR-pc-035's "resets `sub`" is read as covering a pod hop and a detach only, and an amendment of the TR text is requested (GDD sync); the Controller's channel clearing is ADR-0019 |
| TR-section-038 | ship-section-and-layout-data-model.md | LM2 and LM3 return records so Body rebases from them; `detach_section` reads no actor state; the actor-in-cap rejection belongs to Body and Assembly | § 9: BM3/BM4 consume `AttachResult`/`DetachResult` verbatim (two-int shifts, `moved_sections`, `rects` as four ints per rectangle); cap guard in BM4; the records are ADR-0013 |
| TR-shipbody-036 | ship-body-and-interior-physics.md | `solid_in` returns unique rectangles in ascending key order; the sweep tolerates empty, duplicate or unordered results and signals `diag.solid_in_unordered`; the set is post-cut, post-leaf of that tick | § 5.1 (tolerance, debug signal, the assumed same-tick set); the order and uniqueness guarantee is ADR-0014 |
| TR-shipbody-043 | ship-body-and-interior-physics.md | Suit calls `room_at` with section-local pixels, NONE or plate is exposure; Interaction reads `foot_boxes` unfiltered and ascending; Emitter and Lighting anchor on `pos_curr` | § 11.3 (reads, order, domain, `actor_local_*`); the consumers' behaviour is ADR-0027, ADR-0020, ADR-0017, ADR-0028 |
| TR-shipbody-046 | ship-body-and-interior-physics.md | CI performance gates are deterministic counts from instrumented `collide` and `section_at` wrappers; wall-clock P99 is a nightly benchmark failing at +25 % | § 13 (the bounds and the `CountingCollide` wrapper); the counting mechanism and lane P are ADR-0010 |
| TR-shipbody-048 | ship-body-and-interior-physics.md | Transfers change only `body_id`, `pos_curr`, `pos_prev` and `sub` and leave `controller.state` bytes identical; a same-tick Controller write after a pod hop must not displace the arrival cell | § 9 (field set, property test); that the Controller holds no Body write is ADR-0019 |
| TR-assembly-040 | modular-ship-assembly.md | Weld and cut conserve every actor's integer absolute position, use frame shifts that are multiples of 24 px, and keep `state_hash` identical regardless of zoom, seat count or MIRROR role | § 9, § 10 (conservation law, 24-px grid check, zoom independence by construction); replay level R1 is ADR-0005 |
| TR-horror-033 | horror-entities.md | Seat commands issued in tick `t` are applied by `body.integrate` in that tick and `horror.step` reads after it; pinned by a fixture; a latency above `INPUT_LATENCY_TICKS` forces `UNFOLD_LO` to 100 | § 12 (`TICKINDEX-1`, the data-edit rule and the S1 measurement hook); submit/apply/integrate indexing is ADR-0001, `horror.step` order is ADR-0023 |
| TR-shipbody-005 | ship-body-and-interior-physics.md | `body.state` is one SHARED SIM block: persist true, scope WORLD/SHARED, schema 1, supported `[1]`, `restore_after [layout.state]`, exactly one hook and one PROJECTOR subscriber | § 3 registration table (the values ADR-0003 marks Partial and leaves to the Body ADR) and the hook row; the descriptor mechanism is ADR-0003 § 2, and the `body.projector` subscriber is registered by ADR-0034 |
| TR-shipbody-011 | ship-body-and-interior-physics.md | `body.integrate` at divisor 1, offset 0, `after: [controller.step, layout.doors]`, `emitter.place` after it; every named hook id must exist or boot fails | § 3 hook row and "Hook order" (divisor, offset and the `after` set, ADR-0003's "left to Body ADR" residue); the ordering mechanism and `UNKNOWN_DEPENDENCY` are ADR-0003 and ADR-0001, and `emitter.place`'s edge is ADR-0017 |
| TR-shipbody-027 | ship-body-and-interior-physics.md | Ship motion applies no force, push, tilt, knockback or drift to any actor; docking and separation are P5 events carrying no simulation state | § 10 (inertial interior, `DETACH_DRIFT_PX_PER_TICK` locked at 0) and § 11.5 (three event kinds with ids only; **payload change:** `tick` and `seat` dropped, `frame_delta_px` split into two ints, `cause` added; recorded in the GDD/TR sync list); the event mechanism is ADR-0004 and ADR-0009 |

### Owned elsewhere (loaded or checked here only) (5)

| TR | Requirement (short) | Where it is decided; what this ADR contributes |
|---|---|---|
| TR-shipbody-008 | `snapshot()` is a five-key plain Dictionary; canonical bytes are the kernel's TLV; Body ships no byte writer | Mechanism: ADR-0006 and ADR-0005. This ADR fixes the five-key shape and the no-writer rule (§ 11.1) |
| TR-shipbody-009 | `restore` validates in order, rejects the whole block through `ctx.fault`, never clamps; round trip byte-identical | Two-phase contract: ADR-0006 § 2.2. This ADR fixes the `BDY_RESTORE_*` order, the Layout-liveness check and the idempotence argument (§ 11.2) |
| TR-shipbody-038 | Integer-semantics assumptions (`posmod`, truncation, float32 exactness to 2^23, `Performance.OBJECT_COUNT`, directory warning levels) are unverified and gated by a spike | `posmod` and truncation are ADR-0005 and ADR-0010 (editor probes repeated here as V2, release repeat is LINT-SPIKE-1); canvas float32 exactness is ADR-0008 (V3 repeated here); `OBJECT_COUNT` is ADR-0010; this ADR owns only the `WORLD_COORD_LIMIT` fallback (§ 10) |
| TR-shipbody-010 | `post_restore` sets `pos_prev := pos_curr`, preserves `sub`, runs the trapped-actor probe, does not rebuild the kernel seat binding; an overlapping foot box after restore is counted, never a load failure | The restore order and the cross-block verification are ADR-0016's. This ADR's contribution: § 8 and § 11.2 (`post_restore` contents, the diagnostic-only liveness re-check, the trapped-actor probe, `binding_source` instead of a Body-side rebuild, no clamp and no load failure on an overlapping box) |
| TR-shipbody-037 | The coupling `BROADPHASE_CELL_PX` against the foot box and the controller's top speed is boot-checked so a swept box touches at most 4 broadphase cells | ADR-0014's `BROADPHASE_CELL_COUPLING` (`BroadphaseCoupling.holds`, the stricter `max(FOOT_BOX_W, FOOT_BOX_H) + V_RUN` form) is the single check; this ADR drops its own `BODY_BROADPHASE_CELL` and cites it (§ 2) |

## Performance Implications
- **CPU**: One hook per tick, divisor 1. Per moving actor one `query_into` and about `3 * K` rectangle tests (K is the rectangles in the swept cells, far below `MAX_FOOTPRINT_RECTS_PER_BODY`); a stationary actor costs the residue update, one copy and a possible dirty mark. The cited budget is 0.05 ms per actor, 0.4 ms for eight, against a 2 ms P3 target, and it is the whole-hook ledger entry that includes the `solid_in` calls (§ 13): **pending S1** (no wall-clock figure is a gate until recorded on RM-DESKTOP and RM-DECK in a release export). The editor-binary probe (2.4 microseconds per actor-tick at K = 16, both axes) is indicative only.
- **Memory**: Persisted columns for eight actors and a few bodies: about 4 KB at worst; derived columns the same order; one `QueryBuffer` sized once at boot to `MAX_FOOTPRINT_RECTS_PER_BODY`; the snapshot copy is a few hundred integers. No per-tick allocation (`HOT-01`, `CNT-01` to `CNT-03`).
- **Load Time**: `restore` is a cold pass over a few hundred integers plus one liveness check per distinct body; negligible next to Layout's rebuild.
- **Network**: None at the MVP (`SHARED_SUMMARY` replication is VS-B; no client-side prediction).

## Migration Plan
Greenfield: no code exists. Implementation order for the Body epic: (0) commit `sweep_probe.gd` and `p2.gd` from the probe directory to `tests/support/probes/adr-0015/` (the evidence behind the analytic-clamp decision becomes reproducible); (1) `IntMath` golden tests and `LocomotionTables`; (2) `LocomotionStep` with `SweepOracle` and the `SWEEP-EQ` grid; (3) `BodyState`/`BodyDerived`, the two row functions and the alignment test; (4) snapshot/restore/`default_state` with `BODY-RT` and `GOLDEN-1`; (5) BM1/BM2, then BM6/BM7, then BM3/BM4/BM5/BM8 against ADR-0013's records and the real Layout double; (6) the hook, `TICKINDEX-1`, `INERTIAL-1`; (7) the read view and the MVP lints (§ 12; `BDY-07`, `BDY-08` and the bounds ledger land with their first triggering story); (8) spike S1 harness; (9) integration closure against the real kernel and Save/Load (AC-BDY-6 and AC-BDY-138 gate the epic). The GDD sync items (§ Related) land before stories are written from this ADR.

## Validation Criteria
- `SWEEP-EQ` passes with zero tolerance; the random-fixture soak with `BODY_SWEEP_CHECK` finds no difference; the analytic clamp is the only sweep in the release hook.
- `BODY-RT` (restore and re-snapshot at every tick of a 600-tick, three-phase run) is byte-identical; `GOLDEN-1` matches; `restore` rejects each named input with its named code and leaves the live block untouched.
- `MUT-CONS` holds I1, I2 and I5 after every call and I3 and I4 after every complete apply, over a generated sequence of Layout (LM2, LM3, LM4) and Body mutators including faulted, saturating and canonical-empty calls; a faulting call changes no byte of `snapshot()` and no dirty key; a BM3 case runs LM2 then BM3 against the real Layout double and asserts `sub` and `last_octant` byte-identical; a BM8 case follows LM4.
- `restore` rejects `2^31` and `2^32 + 5` in every int32 column (`BDY_RESTORE_COLUMN_RANGE`); a restore followed by a read through a pre-held `BodyView` returns the restored values; absent ids give `-1` and neutral getter answers at the first, middle, last and past-the-end positions; `LOCO-IDLE-1` passes on the hot form; `LocomotionTables` loads with three arrays of size 8.
- `INERTIAL-1` byte-identity; `TICKINDEX-1` ordering; the `moved` reader's registry-level `after` assertion.
- Count gates: at most one `query_into` per moving actor per tick, none for a stationary one, zero per-tick `origin_in_body` for a non-exempt actor; `CNT-01` unchanged over 600 ticks.
- S1: P99 of the whole hook (with `solid_in` time reported as a sub-counter of `body.integrate`) at most 90 % of the frame ceiling on RM-DESKTOP and RM-DECK in a release export; nightly benchmark within +25 % of the baseline; SoA versus array-of-records recorded (V5, V6).
- Release-export repeat of V2 and V3 (LINT-SPIKE-1 and ADR-0008's spike) agree with the editor probes.
- `BDY_*` clean-build counters: `PC_FOOTPRINT_OVERLAP`, `BDY_TRAPPED_ACTOR`, `BDY_REBASE_ACTOR_IN_CAP`, `BDY_TRANSFER_NO_SECTION`, `BDY_PLAN_MISMATCH` read 0 in every clean scenario.

## Related

- **Registry stances relied on**: see Context. This ADR proposes the registry entries listed in the workflow output (`state_ownership` for `body.state`; interfaces for the mutators, `LocomotionStep`, the read view, the ports and the events; the analytic-clamp and in-hook-pair API decisions; forbidden patterns for engine physics on a simulation path, actor displacement, SIM reads of `blocked`/`world_*`, and cached row indices). No registry file is edited here.
- **Foundation ADRs and the asks this ADR makes of them** (not worked around): ADR-0003 (state that an owner may keep the interpolated pair in its own hook with a no-op `commit_baseline`, and that the `key` of `snap_baseline` is ignored for `SHARED` blocks; the `can_*` naming divergence is settled by the CT-58 sentence in § 5, so no rename is pending; Body's row `body.integrate` 0.4 ms for eight actors in the P3 share ledger, CT-20); ADR-0004 § 7 (the illustrative quiet-block list names Body; Body declares `quiet_writes_allowed = false`); ADR-0006 § 2.2 (list `body.state` outside `REBASE_BLOCKS`, closed list with no Body member); ADR-0010 (register lints `BDY-01` to `BDY-08`, the `## @hot` list of § 13, the `body_readers.json` and `body_bounds.lock.json` config files, `src/sim/body/ports/` as a port directory; fold the further 2D-physics tokens of § 1 into `SIM-06` and `BDY-01`; an allow-list entry for the `body_state` snapshot and restore files, which use bare `Array` and `Variant` against `SIM-09`, § 11.1; **no `HOT-01` exemption is requested**, because no port method is named `bind`); ADR-0007 (the two `CONTENT_INVARIANT` names `BODY_FOOT_W_EVEN`, `BODY_COORD_LIMIT_COVERS_LAYOUT` and `BODY_CANVAS_BOUND`; the `[0, 0]` range of `DETACH_DRIFT_PX_PER_TICK`; a schema ceiling of 2^24 for `BODY_COORD_LIMIT`; the broadphase coupling is ADR-0014's `BROADPHASE_CELL_COUPLING`, not restated). **Withdrawn:** the open ask to ADR-0002 § 4 about the `binding_source` payload shape, answered by ADR-0016 § 6.2 (CT-25).
- **Sibling assumptions (cross-ADR)**:
  - ADR-0014 (a dependency, CT-19): `SolidQuery.solid_in(body_id, ax0, ay0, ax1, ay1, out: QueryBuffer) -> int` is consumed verbatim: SOLID rectangles in their current leaf state intersecting the half-open box, body frame, unique, ascending `rect_key`, allocation-free; `QueryBuffer` columns `keys` (`PackedInt64Array`), `spans` (`PackedInt32Array`, four ints per record: `x0, y0, x1 - 1, y1 - 1`, inclusive occupied span), `flags`, with `count`, `status` and `grew`, sized once to `MAX_FOOTPRINT_RECTS_PER_BODY`; the result reflects the post-cut set from P2 and the post-`layout.doors` leaf set from P3. Requested of ADR-0014: § 4 states that the span-to-half-open conversion happens in exactly one place (`RectSpans`, this ADR § 5.1); "`CollideSource` / `LayoutCollide` (ADR-0015) is the only permitted wrapper over `SolidQuery`"; its assumption line, diagram and Enables row read "Body via `LayoutCollide` (ADR-0015 `CollideSource` adapter, holding a `SolidQuery`)"; its Validation "Ports" line reads "`FakeSolidQuery extends SolidQuery` drives `LayoutCollide`; a contract test runs `LayoutCollide` against the real `LayoutSolidQuery`"; its section 11 and budget line replace "`solid_in` P99 at most 0.05 ms per actor, 8 actors at most 0.6 ms aggregate" with "`solid_in` is a share of ADR-0015's whole-hook 0.05 ms per actor (0.4 ms for eight); S1 reports it as a sub-counter of `body.integrate`"; `BROADPHASE_CELL_COUPLING` is the single check for TR-shipbody-037.
  - ADR-0013 (CT-02, CT-17, CT-22, CT-23): the reads owed to Core siblings, tagged `## @hot`: `body_is_live(body_id) -> bool` (`state == LIVE`), `body_state(body_id) -> int` (LIVE, DISSOLVED, UNKNOWN; Tier H, Tier S after restore) and `section_contains(section_id, local_x, local_y) -> bool` (own half-open cell set), plus `section_at`, `section_offset_x/_y` and `body_of`; `AttachResult` and `DetachResult` consumed verbatim (§ 9); **LM2 marks the absorbed body DISSOLVED before BM3 runs** (§ 9 BM3), and ADR-0013 § 9 LM4 notes that BM8 must follow it in the same apply; `can_attach` keeps section offsets inside `BODY_COORD_LIMIT` (must-repeat); Layout never reads `body.state`; Layout's adapters live in `src/sim/layout/` and Body's (`LayoutCollide`, the `BodyLayoutPort` implementation) in `src/sim/body/`.
  - ADR-0019 (CT-27, CT-53; **co-accepted**): `ControllerView.get_octant(player_id)`, `get_gait_effective(player_id)` and `has_row(player_id) -> bool` (`row_count` stays for the give-up solo test); **Body does not read `intent_octant`** (struck from the § 2.6 sentence; TR-shipbody-016 binds two fields); Body publishes the scalar getters `pos_x/pos_y`, `body_of_actor`, `seat_of_actor`, `actor_of_seat`, which the Controller reads (a `Vector2i` is built only at the Adapter boundary); the "pod-body test" and `home_body` reads are **ADR-0026's** (`is_pod_body(body_id) -> bool`, `home_body`; an assumption on ADR-0026), not Body's and not Layout's; the assumption row on this ADR reads "BM3 keeps `sub` and `last_octant`; BM4 (new side) and BM5 set `pos_prev := pos_curr` and zero `sub`; TR-pc-035 is read as covering a pod hop and a detach"; `controller.step` before `body.integrate`, `controller.noise` after it; the Controller clears only channel fields on a body edge and never writes Body.
  - ADR-0021 (not on disk; CT-22, CT-26): calls `can_rebase_attach` before LM2 and BM3 after it, `can_rebase_detach(views, old, shift_x, shift_y, cap_rects)` for **every** cut origin (PLAYER refused, SYSTEM deferred; TR-shipbody-039) so that no cut path reaches BM4 without it, and BM4 after the Graph and Layout mutators in one scope; checks each status and stops on the first negative; repeats the `docked` offer gate; passes Layout's records unchanged (including `AttachResult.moved_sections` to BM3); adopts the INV-4 wording of the ADR-0016 bullet below in its invariant bodies and the weld property test.
  - ADR-0026 (not on disk; CT-23, CT-27): Web Map calls BM1, BM5, BM6, BM7 and BM8 only inside the listed atomic applies, owns the `REORIGIN_IS_TRANSLATION` lint and the `WEB_COORD_BUDGET_PX` bound, and follows the discard order of § 9 BM8 (BM5 / BM2 on the actors, the Graph's mutator, LM4, then BM8); it owns `is_pod_body` and `home_body`.
  - ADR-0034 (CT-32; V8): `body.projector` subscribes to `body.state` (`BODY`/`ACT` keys, the three events), runs the stage-0 `BODY_TRANSFER` pass that reparents with `reparent(p, false)`, computes `canvas_offset_render` from `world` and `world_prev`, and owns the entity-id namespaces; the owning presenter rewrites the node position from `BodyView.pos_x/pos_y` in the same frame. Body publishes getters only.
  - ADR-0023: Horror constructs `LocomotionStep`, uses the same `CollideSource` (`LayoutCollide`) with its own body id (assumption), keeps its position in `horror.state`, and registers `horror.step` after `body.integrate`.
  - ADR-0016 (CT-25, CT-44): `restore_after [layout.state]`, the seat-binding rebuild from `binding_source` (payload per ADR-0016 § 6.2) and verifiers; the Assembly invariant "INV-4 one Body pose row per live body" reads "at most one Body pose row per body, and every row's body is live" (a live body with no row is the legal sparse zero), and ADR-0016 V12 quotes that wording; ADR-0021 adopts it before V12's fixture row lands.
  - ADR-0017, ADR-0018 (CT-24): cite `BODY_COORD_LIMIT` (`I-BOUNDS`, the section 11 int32 column-sufficiency statement) and the canvas bound `|world_a - world_b| + |pos| < 2^23` (the 0.5-lattice assumption); `pos_x/pos_y` are body-frame.
  - ADR-0027, ADR-0020, ADR-0017, ADR-0028: read contracts of § 11.3.
  - ADR-0033: the presentation-only debris exemption of `SIM-06` lives on a presentation path.
- **Design documents**: `design/gdd/ship-body-and-interior-physics.md` (owner), `design/gdd/ship-section-and-layout-data-model.md`, `design/gdd/player-controller-input.md`, `design/gdd/modular-ship-assembly.md`, `design/gdd/web-map-and-transit.md`, `design/gdd/horror-entities.md`, `design/gdd/pixel-perfect-render-pipeline-camera.md`, `design/gdd/command-simulation-layer.md`.
- **GDD sync (proposed; the GDD is updated to this ADR)**: BDY-5 gains `interpolated = true`, `gameplay = true`, `commit_baseline`, `snap_baseline`, `binding_source`; BDY-7 fixes `kind` values `SEAT = 1`, `AGENT = 2`; BDY-8 spells `after` as `{id, tolerates_stale}` dictionaries; BDY-15 names `CollideSource` (`query_into(body_id, ...)`, `origin_in_body(body_id, ...)`), `QueryBuffer` and the span decode and renames every `solid_in_into` to `solid_in` (CT-19); BDY-20/21 and the mutator table: `scope` first, `OK/REFUSED/-fault`, BM1 id through `PlaceResult`, shifts as two ints, `cap_rects` as a flat int array and as an argument of `can_rebase_detach`, BM3 also takes `moved_sections`, BM8 `release_body` and `can_release_body` are added, reasons versus fault codes, the sink as a port, new reason `BDY_PLACE_ID_EXHAUSTED`, `BDY_ARG_RANGE` and `BDY_RELEASE_*`, fault `BDY_APPLY_PRECONDITION` and `BDY_RESTORE_COLUMN_RANGE`; BDY-26/D.7 float32 bound `[UNV]` becomes "probed 2026-10-29, editor build" and gains `BODY_COORD_LIMIT` and the canvas bound; BDY-31 records the in-hook pair and the `commit_baseline` no-op; the events table loses `seat` and `tick` and splits `frame_delta` (TR-shipbody-027's payload text); D.12 loses any `BODY_BROADPHASE_CELL` wording and cites `BROADPHASE_CELL_COUPLING`; D.13 `ACT` and `BODY` marks with `quiet_writes_allowed = false`; Open Question 46 is answered (SoA, reversible by S1); TR-shipbody-020's text is amended for BM1; TR-pc-035's text is amended ("resets `sub`" covers a pod hop and a detach; a weld keeps `sub`); TR-shipbody-042's reader list is bound to `body_readers.json`.
- **Probe artefacts**: `sweep_probe.gd` and `p2.gd` (V4 and the V1 to V3 probes) are in `C:/Users/dasbl/AppData/Local/Temp/claude/adr-probes/0015/`, outside the repository; they are to be committed to `tests/support/probes/adr-0015/` (Migration Plan, task 0), after which this line cites the repository path. The engine review's probes are in `adr-probes/review-ADR-0015` (V9).

## Open Questions
- **OQ-1 (S1).** Does the whole hook, with the abstract-port call per query and the load/store of the `LocoState` scratch, fit the P99 budget on RM-DECK? If not, the fallbacks in order are: index-addressed `step_row` that bypasses the scratch copy (a second code path, so only with an equivalence test), an array-of-records or SoA re-layout (no schema change), then the GDExtension rung (ADR-0010 § 9, VS-A scope only). Owner: spike S1.
- **OQ-2 (closed 2026-10-29).** ADR-0014 returns coordinates, as inclusive `spans` (four ints per record: `x0, y0, x1 - 1, y1 - 1`); the conversion to half-open is `RectSpans` (§ 5.1).
- **OQ-3.** The deceleration ladder (`STOP_DECEL_TICKS > 0`) is dormant at the MVP; the provisional sub-case of the move octant (§ 6) is implemented but untested against the Controller until the knob is raised.
- **OQ-4.** Four-seat crowding in a one-tile corridor (actors do not block one another at the MVP) is a Vertical Slice question.
- **OQ-5.** Whether Save/Load's autosave guard (`trapped_into` non-empty) is accepted by ADR-0006 as a must-repeat or a Body ask.
- **OQ-6.** `cause` on `transfer_actor`: only `POD` is used at the MVP; whether a future non-pod cause exists (an airlock hop) is Web Map's.
- **OQ-7.** Whether `ACTOR_ID_KEY_MAX` should instead be enforced by letting `DirtyKey.make` mask with `low32` (ADR-0004's rule says the id must not exceed 32 bits, so the current refusal is stricter and cheaper).
- **OQ-8.** `BODY_COORD_LIMIT` needs a registry entity (derived from `MAX_SECTIONS * MAX_SECTION_EXTENT_PX`, ceiling 2^24, default well below the canvas bound), and `MAX_SEAT` should be tightened if ADR-0002's `PlayerId` has a narrower bound. Owner: the registry owner.
- **OQ-9.** Move `sweep_probe.gd` and `p2.gd` into `tests/support/probes/adr-0015/` (this revision may edit only this file, so the move is a task for the Body epic's task 0).

## Review Log

**2026-10-29, revision after the per-ADR engine and TD reviews and the Core cross-ADR adjudication (autonomous mode, "Revise flagged items"). Status stays Proposed.** `Date` stays at its original value; this log is the revision record.

Engine review (verdict CONCERNS; the core engine decisions held and needed no change: no engine physics on a sim path, id-sorted packed-array SoA, `@abstract` ports including the one-line form, the typed-return override rule, no reliance on the packed-array element-write change):
- **E1 (applied):** the stateful `bind(body_id)` is removed (the `bind_body` option of the review is superseded by CT-19); `CollideSource.query_into` and `origin_in_body` take `body_id` explicitly, `LocoState` carries `body_id`, and § 5.1, § 7, § 12, Key Interfaces and Alternative 11 say so; no `HOT-01` exemption is requested and no port method is named `bind`. `CountingCollide` wording updated (§ 13).
- **E2 (applied):** § 2 and Key Interfaces state `const NAME: PackedInt32Array = [..]` and forbid the constructor form in a `const`; read-only note; a one-line `LocomotionTables` load test is added to the golden test, the `Validation Criteria` and Guidelines. V9 records the probe.
- **E3 (applied):** § 4.1 and § 11.2 step 4 add the column-range rule with new fault `BDY_RESTORE_COLUMN_RANGE` (domain 5, local 33, append-only); the same bounds are mirrored in `can_place`, `can_transfer`, `can_rebase_attach` and `can_rebase_detach` through the new check-only reason `BDY_ARG_RANGE`, with `BDY_APPLY_PRECONDITION` as the apply-time twin; `BodyIds.INT32_MAX` and `MAX_SEAT` added; `BODY-RT` gains the `2^31` and `2^32 + 5` fixture.
- **E4 (applied):** new § 4.5 `_find_actor_row` / `_find_body_row` (`-1` unless `row < size and id matches`, `before = true`); getters return neutral values, mutators `BDY_UNKNOWN_ACTOR`; fixture `ROW-ABSENT-1` at first, middle, last and past-the-end; R10 and Guidelines updated.
- **E5 (applied):** § 7 states `a_blocked[i] = 1 if result.blocked else 0` and § 11.3 reads `a_blocked[row] != 0`; Guidelines bullet.
- **E6 (applied):** private port members (`_collide`, `_layout`, `_sink`, `_tuning`) in § 2 and Guidelines; underscore-prefixed unused parameters; § 8 shows `snap_baseline(_key: int, ctx)`.
- **E7 (applied):** § 11.2 phase 2 builds the scratch from fresh locals discarded after phase 2 and copies element by element (`resize` plus an index loop); reference replacement and a cached column alias in `BodyView`, the hook or `LayoutCollide` are forbidden (§ 4.2, Guidelines); `BODY-RT` includes a restore-then-read-through-a-pre-held-view case.
- **E8 (applied):** the twelve additional 2D-physics tokens are added to § 1 and the `BDY-01` entry of § 12 and to the ask to ADR-0010; the guarantee is stated as "no use", not "not instantiated".
- **E9 (applied):** § 11.1 and Alternative 10 reworded to the binary path only (JSON dump is diagnostics, ADR-0006 § 1.3; the alternative is not engine-blocked); the `SIM-09` allow-list entry for the `body_state` snapshot and restore files is added to the ADR-0010 ask (merged into the CT-54 amendment).
- **E10 (applied):** § 10 states that `canvas_offset_render` must floor before assigning to `Node2D.position` (ulp 0.5 px at 2^22, 1 px at 2^23) and that `>>` never receives a negative operand in `src/sim/body/`; V3 notes the reproduction; the release-export repeat stays planned.

TD review (verdict CONCERNS):
- **T1 (BLOCKING, applied; CT-22):** BM3's predicate is split by phase in § 9: `can_rebase_attach` (before LM2) requires both bodies LIVE; `rebase_on_attach` (after LM2) requires surviving LIVE, absorbed DISSOLVED per `body_state`, and `body_of(s) == surviving` for every section in `moved_sections` (a parameter added to the mutator). `body_is_live` is defined as `state == LIVE` in the `BodyLayoutPort`; `body_state` read added; a `MUT-CONS` case runs LM2 then BM3 against the real Layout double.
- **T2 (BLOCKING, applied; CT-23):** BM8 `release_body` and `can_release_body` added (refuses with `BDY_RELEASE_ACTORS_PRESENT` while actor rows remain, `BDY_RELEASE_BODY_NOT_DISSOLVED` unless DISSOLVED; faults 34 and 35); runs after LM4 in the same apply; discard sequence recorded as an assumption on ADR-0026; I3 restated as an invariant of the whole apply; LM2, LM3 and LM4 added to the `MUT-CONS` generator; R11 added.
- **T3 (BLOCKING, applied; CT-19):** § 5.1, § 5.3, § 5.5, the diagram, Key Interfaces, Related, R3, OQ-2 and Alternative 11 rewritten against ADR-0014 verbatim: `solid_in`, inclusive `spans`, one decode (`RectSpans`: `x1 = spans[4k+2] + 1`, `y1 = spans[4k+3] + 1`) shared by `AxisClamp`, `SweepOracle` and the doubles, a 1-px-wide leaf fixture, `QueryBuffer` sized to `MAX_FOOTPRINT_RECTS_PER_BODY`, `grow()` never inside `step_into`, a non-OK `status` or set `grew` a counted defect (`BDY_QUERY_DEFECT`). `LayoutCollide` holds an injected `SolidQuery`; Horror uses the same `CollideSource` (assumption on ADR-0023). OQ-2 closed ("spans"). ADR-0014's `@abstract` release fallback (LINT-SPIKE-1(a)) added to Engine Compatibility.
- **T4 (BLOCKING, applied; CT-27 alignment):** the TR-pc-035 row corrected: all three mutators set `pos_prev := pos_curr`; BM4 (new side) and BM5 zero `sub`; BM3 keeps `sub` and `last_octant`; "resets `sub`" is read as covering a pod hop and a detach, and a TR text amendment is requested; a BM3 property test asserts byte-identical `sub` and `last_octant`; the ADR-0019 bullet repeats the reading.
- **T5 (applied):** BM4 step (4) inserts the new pose row only if it is not canonical-empty, with a directed `MUT-CONS` case; the dead canonical-empty line in the § 7 freeze branch is replaced by a comment stating why no hook write can produce one.
- **T6 (applied):** the explicit step 2a in § 6 (`o == NONE`: `sub := 0`, `last_octant := NONE`, return before any table index); the tables stay at size 8; fixture `LOCO-IDLE-1` on the hot form.
- **T7 (applied; CT-20):** § 13 states the ledger entry is the whole `body.integrate` hook, 0.05 ms per actor P99 and 0.4 ms for eight, including the `solid_in` calls, the clamp, the row copy and the dirty marks; S1 reports the whole-hook figure with `solid_in` as a sub-counter; ADR-0014 is asked to restate its figures as a share.
- **T8 (applied; CT-24):** `BODY_COORD_LIMIT` added to `BodyTuning`, the bounds ledger and restore; boot invariants `BODY_COORD_LIMIT_COVERS_LAYOUT` and `BODY_CANVAS_BOUND`; the canvas bound `|world_a - world_b| + |pos| < 2^23` published in § 10; Layout's `can_attach` must repeat the limit (must-repeat for ADR-0013); the default is derived from `MAX_SECTIONS * MAX_SECTION_EXTENT_PX` (registry names) rather than asserted (OQ-8).
- **T9 (applied):** TR-shipbody-005, -011 and -027 added to the Shared table (with the payload change for -027), TR-shipbody-010 to "Owned elsewhere" with this ADR's contribution; TR-shipbody-037 also added there (CT-21). Section headings recounted (12 shared, 5 owned elsewhere).
- **T10 (applied; CT-21):** `BODY_BROADPHASE_CELL` deleted; ADR-0014's `BROADPHASE_CELL_COUPLING` is cited as the single check; `BODY_FOOT_W_EVEN` kept.
- **T11 (applied; CT-26):** `can_rebase_detach` takes `cap_rects` and returns `BDY_REBASE_ACTOR_IN_CAP`; the ADR-0021 assumption covers every cut origin (PLAYER refused, SYSTEM deferred; TR-shipbody-039).
- **T12 (applied in part):** an MVP-required list (SIM-06 plus BDY-01, BDY-02, BDY-05, SWEEP-EQ, BODY-RT, MUT-CONS) and the items that land with their first triggering story are in § 12 and the Migration Plan; the probe scripts are named for `tests/support/probes/adr-0015/` as task 0 and OQ-9. **Declined to move the files in this revision**: the revision may edit only this file.
- **T13 (applied):** AGENT rows are not stepped at the MVP (treated as an unbound seat, counted `BDY_AGENT_UNSTEPPED`); the ADR-0002 § 4 ask is replaced by a citation of ADR-0016 § 6.2 (CT-25); the pod-body and home-body reads are named as ADR-0026's.

Cross-ADR rulings:
- **CT-02 (aligned):** Body's adapters (`LayoutCollide`, the `BodyLayoutPort` implementation) live in `src/sim/body/`; Layout's `LayoutSolidQuery` is ADR-0013's. **CT-17 (aligned):** the reads owed are listed in the ADR-0013 bullet. **CT-19, CT-20, CT-21, CT-22, CT-23, CT-24, CT-25, CT-26 (owner, applied):** see T3, T7, T10, T1, T2, T8, T13 and T11. **CT-27 (aligned):** scalar getters, no `intent_octant`, pod-body and home-body reads on ADR-0026, the TR-pc-035 reading. **CT-32 (aligned):** V8 recorded as answered by the 2026-10-29 probe (`reparent(p, false)`, the owning presenter rewrites the position). **CT-44 (aligned):** the INV-4 wording in the ADR-0016 bullet. **CT-53 (applied):** Depends On adds ADR-0014 and the co-acceptance with ADR-0019, and the "forward reference" is removed from the Ordering Note. **CT-54, CT-58 (no Core change beyond the asks):** the `can_*` prefix is settled by ADR-0003's amendment; the ADR-0010 asks are listed in Related.
- **Declined / deviations:** the engine review's `bind_body` rename option (E1) is superseded by CT-19 (explicit `body_id`); no `HOT-01` carve-out. T6 chose the step-2a early return over 9-row tables. No finding was rejected as wrong; the rejected findings listed by the adjudication were not applied.

Not edited (outside this file): `architecture.md`, the registries, the GDDs, other ADRs, and the probe scripts' location; the GDD sync list and registry candidates are in the result.
- **2026-10-29 confirm residuals:** ADR-0016 moves from Enables to Related (ADR-0016 does not list this ADR under Depends On; both directions would close 0015 -> 0016 -> 0019 -> 0015), and ADR-0018 and ADR-0019 join Enables; BM4's `section_contains(s, local_x, local_y)` call is confirmed against ADR-0013's section-local row; the commit_baseline rationale names `moved_x(actor)` and `moved_y(actor)`.
- **2026-10-29 orchestrator rulings:** R3 (the binding-source payload and the `binding_source` callable row read `[(player_id, actor_id)]`, ascending `player_id`; at the MVP seat == PlayerId, matching ADR-0016 § 6.2 and ADR-0002 § 4), R10 (the `BDY-02` `body_readers.json` lint text lists `controller.noise` as registered after `body.integrate`). Dependency-cycle break: ADR-0019 moved from Depends On to a new `Co-accepted with` row (the pair's hard edge is ADR-0019 -> ADR-0015); the `Related, not Enables` sentence (ADR-0016) and the unwritten ADR-0020/0021/0023/0026/0027/0028 consumers moved out of Enables into a new `Soft references (not acceptance prerequisites)` row; Enables now lists ADR-0017, ADR-0018, ADR-0019 and ADR-0034, each of which lists this ADR under Depends On.
