# ADR-0013: Ship Layout Block, Geometry and ID Allocation

## Status
Proposed

## Date
2026-10-29

## Last Verified
2026-10-29 (against `design/gdd/ship-section-and-layout-data-model.md` LY-1 to LY-35 and D.5, D.6, D.9, D.10, D.12, D.21; ADR-0001 to ADR-0010, all Proposed; the engine facts marked "probed 2026-10-29" were run on the 4.7.2 mono console binary)

## Decision Makers
Claude (author), running under modes.automation: autonomous (project.yaml); decisions logged for the user's review; the user has not yet reviewed this ADR.

## Summary
Ship Section & Layout is the one owner of rooms, rectangles, sockets, welds, bodies and three id spaces, and every Core and Feature system reads or calls it, but no document fixes how its block is shaped, how its rectangles are keyed, how a weld moves a body or which side of the Layout/Graph boundary allocates which id. This ADR decides: `layout.state` stores **instances by reference** to immutable catalog templates as **dense per-column packed arrays** (rows addressed by `id - 1`, tombstones kept, a third persisted `structure_hash` added to the GDD's two); geometry is **integer half-open rectangles** with a flags byte in a reserved, never-renumbered `idx` space; sockets are **sparse terminal-state rows** changed only inside two mutators; weld offsets are computed on the **lower-`body_id` survivor** with a canonicalised `welds[]`; and ids are **split** so that Layout allocates `section_id`, `body_id` and `room_id` from persisted counters while the Graph allocates node, edge, compartment and seam ids and Layout only stores the per-instance bases.

## Engine Compatibility

| Field | Value |
|-------|-------|
| **Engine** | Godot 4.7.2 (pinned 4.7; installed 4.7.2.stable.mono, `docs/engine-reference/godot/VERSION.md`) |
| **Domain** | Core (simulation data model; scene-tree free, integer-only; the PROJECTOR that draws it is ADR-0034's) |
| **Knowledge Risk** | MEDIUM. The module uses no rendering, physics or scene API. The two HIGH items it touches are the 4.7 packed-array element-write change (every dirty mark here is explicit inside a mutator) and the 4.7 typed-return override rule (it applies to the `@abstract` port of § 8; the rule is sourced from `breaking-changes.md` § 4.6 to 4.7) |
| **References Consulted** | `docs/engine-reference/godot/VERSION.md`; `breaking-changes.md` (4.6 to 4.7 GDScript rows, 4.3 `TileMapLayer` row); `deprecated-apis.md`; `current-best-practices.md` § GDScript (4.7). **`docs/engine-reference/godot/modules/` has no core, scripting or tilemap file** (it holds animation, audio, input, navigation, networking, physics, rendering and ui), so every claim about `Array[String]` encoding, packed-array methods and GDScript cost below is either probed or marked UNVERIFIED. Project ADRs: 0001 to 0010 (§ Related) |
| **Post-Cutoff APIs Used** | Packed-array element writes that skip the property setter (4.7); `@abstract` ports (4.5, chosen by ADR-0010) for `LayoutGraphPort`; typed `Array[String]` as a persisted column. No `TileMapLayer`, `Node` or `Resource` API is used by this ADR |
| **Verification Required** | **V1 (probed 2026-10-29, done):** `PackedInt32Array.bsearch(v)` returns the insertion index and, for ties, the first index with `before = true` and one past the last with `before = false`; `PackedInt32Array.insert` and `remove_at` keep order; a value `0xDEADBEEF` stored in a `PackedInt32Array` reads back as `-559038737` and masks to `3735928559`; `var_to_bytes` then `bytes_to_var` kept a typed `Array[String]` nested in a `Dictionary` typed (`is_typed` true) in this probe, also through `duplicate(true)`, and a `PackedInt64Array` is exact to 2^40 (**this ADR does not rely on type preservation**: ADR-0006 does not assert it, so restore re-types the column with `Array[String].assign`, § 10); `PackedStringArray` also round-trips (but ADR-0006 § 1.2 does not allow it in a save, so it is not used); `duplicate()` of a packed array is independent of later element writes, whereas plain assignment `var t := s` **aliases** (`t[0] = 99` changes `s[0]`); element writes into a narrow packed column **truncate silently** (`PackedByteArray[0] = 300` reads 44; `PackedInt32Array[0] = 0xDEADBEEF` reads `-559038737`), which this ADR uses deliberately for the three hashes and guards everywhere else (§ 10 step 1); `var x: int = d["k"]` with a `String` value, or a missing key, raises a script error and aborts the function (so restore uses `has()` and `typeof()` before any typed assignment, § 10). The idiomatic 4.7.2 port header (`@abstract class_name X extends RefCounted` on one line, then `@abstract func ... -> T` with a trailing comment) was loaded; a concrete override that omits a return fails with "Not all code paths return a value"; the `@abstract` declarations with unused parameters load cleanly under ADR-0010's promoted warnings, concrete implementations do not (`_`-prefix rule, Guidelines). **V2 (UNVERIFIED, spike S1 / SP-ENG-1):** the GDScript cost of the `layout.doors` loop at `MAX_DOORS_LOADED` doors on a release export on RM-DESKTOP and RM-DECK against its 0.2 ms line (TR-section-042). Plausibility baseline for S1 (review probe, not a release figure): a throwaway loop shaped like the hook (48 doors, one virtual port call with a caller-owned `out` object, clamp, `floor_div` through a static call, compare and store) ran at about 19 microseconds per 48 doors on the 4.7.2 mono editor binary, headless, debug, roughly 10x under the line; it omits section flagging and key marking, so V2 stays UNVERIFIED. The hook is dominated by the port call (one per door), not by the integer math, so the ADR-0010 § 9 ladder tries the Graph change-counter early-out (OQ-5) before any native rung. **V3 (UNVERIFIED, S1 with ADR-0014):** the P2 cost of `attach_section` and `detach_section` including the per-body broadphase rebuild at `MAX_FOOTPRINT_RECTS_PER_BODY`, and the uncached `GEOM_OVERLAP` line with **both legs** (ADR-0014 `footprint_overlaps` and Layout's own tile-cell leg, § 6) at the largest MVP body pair against the 2 ms target. **V3b (UNVERIFIED, attached to S1):** whether a reused packed array that is `clear()`ed or `resize(0)`d each call reallocates (the engine's copy-on-write buffer design suggests it does); the ADR therefore specifies fill-in-place arrays with a separate `count` (§ 7, Guidelines), which is allocation-free by construction and needs no figure. **V4 (UNVERIFIED, after ADR-0005 is Accepted):** golden vectors for `geometry_hash`, `visual_hash` and `structure_hash` of the four MVP templates, equal on the editor and on the `template_release` export (ADR-0007 CONTENT-EXP-1). **V5 (UNVERIFIED):** `Array[String]` column cost in `snapshot()` and in the type walk for the largest long-campaign shape of § 12 (tens of rows; expected negligible, measured in SAVE-WRK-1 f). **Accept gating:** Accept is **not** gated on V2 or V3 (they are cost lines with ADR-0010's fallback ladder, and no data shape depends on them); **V4** golden vectors gate the template-hash story. The S1 / SP-ENG-1 row of `architecture.md` should list ADR-0013 V2/V3/V3b (routed to `/architecture-review`, OQ-2) |

## ADR Dependencies

| Field | Value |
|-------|-------|
| **Depends On** | ADR-0003 (block descriptor, Class A/B mutators, `WriteScope`, fault domains, owner-callable contract); ADR-0004 (dirty-key layout and `KeySpaceDescriptor`); ADR-0005 (canonical encoding, `Hash64.of_value`, `IntMath`, `low32`, block hash cache); ADR-0006 (snapshot types allowed in a save, two-phase `restore`, `ctx.fault`); ADR-0007 (template catalog, manifests, `section_templates` ledger, boot tiers); ADR-0010 (`src/sim/layout/`, ports, lint tiers, `## @hot`); ADR-0001 and ADR-0002 (P2/P3 phases, hook registration, no Layout command); ADR-0011 (the Graph's `RoomSource` port and the Graph side of `LayoutGraphPort`; the one hard edge of the pair, declared here because this ADR is the later-numbered). None is Accepted yet, so this ADR cannot be Accepted before them. |
| **Co-accepted with** | ADR-0011 (mutual ports: `LayoutGraphPort`, which the Graph's `GraphLayoutAdapter` implements, and `RoomSource`, which Layout's `LayoutRoomSource` implements; neither may be Accepted alone; ADR-0011 does not list this ADR under Depends On) |
| **Soft references (not acceptance prerequisites)** | ADR-0016 (reads Layout's `restore`/`post_restore` and verifies this block, as a draft; restore ordering is not an acceptance prerequisite of either ADR). **Forward references (unwritten ADRs; each will list this ADR under Depends On when authored):** ADR-0021 (the weld transaction calls LM2/LM3), ADR-0026 (`init_world` calls LM1), ADR-0020, ADR-0023 and ADR-0027 (read views). |
| **Enables** | ADR-0014 (rect set, broadphase, `OccluderSource`), ADR-0015 (Body consumes `frame_shift`, `moved_sections`, `rects[]`, `section_at`), ADR-0017 (Emitter reads `version`/`static_version`, `anchor_geometry`), ADR-0034 (PROJECTOR inputs and `entity_id` fields). |
| **Blocks** | The Layout epic (template compile and hashes, `layout.state`, LM1 to LM4, `layout.doors`, read views); every Feature story that calls `build_section`, `attach_section`, `detach_section` or reads `rects_in`/`room_at`/`door_passage` |
| **Ordering Note** | Wave A. ADR-0011 (Graph data model, same wave) must supply the Graph side of § 3 and § 8: the `instantiate_section` return and its contiguity guarantees, the allocation-free `opening` read and the `LayoutGraphPort` implementation, which is ADR-0011's dedicated adapter `GraphLayoutAdapter` (`src/sim/graph/graph_layout_adapter.gd`, exposed as `graph.layout_port()`); in return this ADR implements ADR-0011's `RoomSource` (`LayoutRoomSource`, § 11). The setup cycle (Layout needs the Graph port, the Graph needs `RoomSource`) is broken by `LayoutModule.bind_graph` (§ 1). § Related lists every assumption. ADR-0014, ADR-0015, ADR-0016 and ADR-0034 are written afterwards and build on the records fixed here. Where this ADR and the Layout GDD differ (§ Related, "GDD sync"), the ADR is the proposal and the GDD is updated to it, not the reverse |

## Context

### Problem Statement

Layout is the project's single answer to "where is it?". Its GDD (LY-1 to LY-35) fixes behaviour in detail but defers seven implementation questions to ADRs: the block shape against immutable templates (ADR-LAY-01), the id split with the Graph (ADR-LAY-05), the field order of the two template-local byte strings (ADR-LAY-06), and four more that belong to siblings. Three of the deferrals have since been overtaken by Foundation ADRs and no longer mean what the GDD says: ADR-0005 forbids owner-written byte layouts, ADR-0006 limits what a snapshot may contain, and ADR-0003 fixes the shape of a Class A mutator. Without one decision, the Layout epic would invent a snapshot shape the type walk may reject, an id scheme that the Graph's `next_*` counters can desynchronise from, and a weld routine whose result depends on argument order.

The decision is needed now because Body, Assembly, Web Map, Emitter and the PROJECTOR all take ids, offsets and rectangle keys from Layout, and a different assumption in any of them (a sparse section id, a body frame origin, a renumbered `idx`) is a save-compatibility defect once the first build leaves the project.

### Constraints

- **The simulation is pure data** (architecture principle 1): `layout.state` is a `RefCounted` of typed fields and packed arrays; no `Node`, no `Resource`, no float anywhere in the module (LY-3 a, LY-31; ADR-0010 `SIM-01`, `SIM-03`, `SIM-10`).
- **Only plain data reaches a save** (ADR-0006 § 1.2 item 3): `null`, `bool`, `int` (64-bit), finite `float`, `String`, `Array`, `Dictionary` with `String` or `int` keys, and `PackedByteArray`/`PackedInt32Array`/`PackedInt64Array`. `PackedStringArray`, `Vector2i`, `Rect2i` and `StringName` are rejected by the walk, although ADR-0005's encoder would accept some of them.
- **One encoder, no owner byte writers** (ADR-0005 § 3.1): Layout supplies the shape of a payload and nothing else; `layout_hash()` is the kernel's block hash.
- **Every persisted id is a stable string key, a ledger integer or a Graph type id** (ADR-0007 § 3.1); never an array position of another block, never position-derived (registry `position_derived_persisted_id`).
- **Content is loaded before the kernel exists** (ADR-0007 § 1.7): templates reach `setup()` as a frozen, get-only `SectionTemplates` catalog; packed arrays are never returned from a catalog (§ 1.5).
- **Mutators are Class A with status returns and caller-owned `out` objects** (ADR-0003 § 5); dirty keys are marked explicitly because 4.7 packed-array element writes no longer call a property setter.
- **P2 applies commands, P3 runs hooks** (ADR-0001): a hook sees the same tick's P2 state, and the Graph's door `area` and `door_state` change only in P2 (LY-10).
- **Capacity ceilings** are fixed by the GDD's Tuning Knobs table and registered constants; this ADR enforces them and does not retype their values.
- **Pre-release saves are disposable** (SL-14, ADR-0006 § 2.6); the migration obligation starts when the first build leaves the project, the same moment ADR-0007's ledgers flip to `SHIPPED`.

### Requirements

- Store instances by reference to templates, with a restore policy that refuses a template edit that would corrupt an instance with history (TR-section-004, -007).
- Allocate and never reuse or renumber `section_id`, `body_id`, `room_id`; keep tombstones; persist `graph_ref_base` per instance (TR-section-005, -037).
- Express geometry as half-open integer rectangles with a flags byte on the 24-px grid, translation only (TR-section-010).
- Keep the door leaf as one rectangle derived from the Graph's `area` in the P3 `layout.doors` hook, with correct seeding, unknown-opening and clamp behaviour (TR-section-012, -013, -014, -034, -035, -042).
- Make sockets terminal-state records changed only by LM2/LM3, with a closed first-failure-wins mating predicate (TR-section-018, -019, TR-assembly-025).
- Provide exactly four Class A mutators with pure checks, and weld offsets that do not depend on argument order (TR-section-020, -021).
- Own room identity, `V` and `exposed_tiles`, own no loose-item table, no `visited` bit, no pose (TR-section-027, -028).
- Enforce capacity ceilings and the block-size diagnostic (TR-section-030, TR-derelict-040, TR-webmap-036).
- Publish read views with fixed field names and deterministic order, including CONSOLE anchor records (TR-section-039, TR-derelict-034, TR-horror-031, -032).

### Binding stances respected (registry check, `docs/registry/architecture.yaml`)

| Registered stance | How this ADR conforms |
|---|---|
| `owner_callable_contract`, `block_registration`, `mutator_status` (ADR-0003) | `layout.state`, `layout.doors`, the fault domain and the key space are registered in `BOOTING` through `ModuleFacet.registrar`; the four mutators are Class A, statuses `OK`/`REFUSED`/`-code`, results through `out`. The `can_*` names diverge from the `check_*` example in the registry; the shape does not (§ 9) |
| `dirty_key_marking`, `change_notification` (ADR-0004) | Keys are `(KIND << 32) | id` with kinds SEC 1, BODY 2, SEAM 3 as LY-32 and **one added kind, LEAF 4** (§ 8); `quiet_writes_allowed` is false |
| `canonical_encode_and_hash`, `block_persistence_payload` (ADR-0005, ADR-0006) | The snapshot is a `Dictionary` of named packed arrays; no int-keyed dictionary, no `PackedStringArray`; template hashes are `Hash64.of_value` of three sub-payloads whose shape § 4 fixes |
| `static_content_catalogs`, `id_ledgers` (ADR-0007) | Templates arrive as a frozen `SectionTemplates` through `setup()`; `section_templates` ledger nesting is decided in § 4 |
| `position_derived_persisted_id`, `block_as_resource_or_node`, `live_reference_in_snapshot_payload`, `packed_array_exposed_or_setter_tracked` (forbidden patterns) | No id derives from a position; the block is a `RefCounted`; payload arrays are duplicates; dirty marking is explicit |
| `fault_domain_table` | Layout is static domain 4 |

No registered stance is contradicted. Three Foundation texts are read as follows and recorded as asks rather than worked around: ADR-0003 § 5 names the pure check `check_*` (this ADR keeps the GDD's published `can_*` names and the same shape), ADR-0005 § 3.2 lists `PackedStringArray` as encodable while ADR-0006 § 1.2 rejects it in a save (this ADR avoids it), and ADR-0007 § 3.2 describes one id range per ledger space (§ 4 uses one range and a class split by key shape).

## Decision

### Architecture Diagram

```
 content (before the kernel exists)                       kernel BOOTING (ADR-0003)
 ---------------------------------                        ------------------------
 assets/data/sections/*.json  --ADR-0007 stage 3-->  SectionTemplates (frozen, get-only)
   + ids.lock.json (flat pairs)                         |  compiled rect table, rooms, sockets, doors,
   + geometry/visual/structure hashes (once)            |  anchors, slots, 3 hashes per template
                                                        v
                                         LayoutModule.setup(facet, templates, tuning)   [graph port bound later: bind_graph]
                                           registers: block layout.state  hook layout.doors
                                                      fault domain 4  key space {SEC,BODY,SEAM,LEAF}
 ---------------------------------------------------------------------------------------------------
 RUNNING                             owner object LayoutModule (src/sim/layout/)
                                     +------------------------------+     +----------------------------+
   Web Map init_world (INIT) ------> | LayoutState  (persisted)     |     | LayoutDerived (RAM only)   |
     LM1 build_section               |  counters, bodies, instances,| --> |  row lookups, socket refs, |
   Assembly weld apply (P2) -------> |  sockets, welds   (SoA)      |     |  leaf cache, versions,     |
     Graph.weld_join -> LM2          +------------------------------+     |  opening_leaves, caps,     |
   Assembly cut apply (P2) -------> LM3        ^ snapshot/restore           |  broadphase handle (0014)  |
   Web Map discard -----------------> LM4      | (copy into existing object) +----------------------------+
   P3 hook layout.doors  <-- Graph.opening(edge) (LayoutGraphPort) : leaf_w = width - floor(width*area/area_max)
                                                   |
   reads: room_at, section_at, door_passage, anchor_geometry, ...  --> Body, Emitter, Horror, Interaction,
          rects_in / solid_in (ADR-0014)                                Failure & Trail, PROJECTOR (ADR-0034)

 id split:   Layout   section_id  body_id  room_id      (counters in layout.state, first id 1, dense, never reused)
             Graph    node_id  edge_id  compartment_id  seam_id   (counters in graph.state)
             bridge   per instance  graph_ref_base {node_base, edge_base, comp_base}  (stored by Layout, returned by Graph M1)
```

### 1. The block: one `layout.state`, instances by reference, derived state kept apart

Layout registers exactly one block, `layout.state`, with the descriptor values of LY-1 and the fields ADR-0003 requires:

| `BlockDescriptor` field | Value |
|---|---|
| `block_id` / `owner_name` | `&"layout.state"` / `&"layout"` |
| `schema_version` / `supported_versions` | `LAYOUT_BLOCK_SCHEMA_VERSION` (GDD Tuning Knobs, 1) / `[1]` |
| `block_class` / `persist` / `persist_scope` / `scope` | `SIM` / true / `WORLD` / `SHARED` |
| `gameplay` / `interpolated` | true / false |
| `repl_hint` | `SHARED_SUMMARY`: the summary is the instance table plus the three hashes; geometry never travels (templates are identical on every peer by `catalogue_version`) |
| `restore_after` | `[]`: Layout is the root of the restore order and reads no other block in `restore()`; ADR-0016 owns the DAG |
| `snapshot`, `restore`, `default_state`, `post_restore`, `read_view`, `empty_view` | six registered `Callable`s, each ending `return ctx.complete()`; `commit_baseline`/`snap_baseline` absent (not interpolated); `init_world`/`binding_source` absent (Web Map's `init_world` calls LM1; Layout binds nothing). M1's only caller is Layout LM1 through `LayoutGraphPort`; LM1's callers are those § 9 names (Web Map `init_world` in the `INIT` scope at the MVP) |

**Instances by reference (ADR-LAY-01).** A template is authored, immutable and lives in the frozen `SectionTemplates` catalog; an instance is a row in `layout.state` that names a template by its `template_id` string and carries only what can differ between two instances of one template: its ids, body, offset, id bases, drift hashes and socket/weld history. Full geometry in the block would multiply rectangles by sections, duplicate authored data, defeat the Vertical Slice "seed plus deltas" model and let an exporter bug corrupt the world permanently; a second `layout.templates` block would add a second restore root for nothing. Both are rejected (§ Alternatives 1 and 2).

**Two objects, one owner.** The module owns two plain `RefCounted` objects:

- **`LayoutState`** holds the persisted tables of § 2 as typed fields. It is the object the block descriptor's callables read and write; `restore` copies into it and never replaces it (ADR-0003 § 2), because the read-view facade `LayoutViews` wraps it for the life of the process.
- **`LayoutDerived`** holds everything that is rebuilt from `LayoutState`, the templates and the Graph and is **never snapshotted or hashed**: the `section_id`-indexed lookup columns, the socket-ref index, the live-count caps, the leaf cache and `opening_leaves` map (§ 8), the three version counters and `changed_region` stamps (§ 7), `mutation_count`, and the handle to ADR-0014's broadphase. The split is what makes "derived caches are never persisted" (TR-section-004) checkable: a test snapshots `LayoutState` only, and the allow-list test of AC-130 walks only that payload.

**Construction and injection (resolves the setup cycle).** `LayoutViews` extends `RefCounted` only (GDScript has single inheritance, so it cannot extend several ports). It is constructed inside `LayoutModule.setup(facet, templates, tuning)` and lives for the whole process. Each port is satisfied by a small adapter that delegates to it, with the port's method names: `LayoutRoomSource extends RoomSource` (`room_volume_tiles` wraps `V`, `room_exposed_tiles` wraps `exposed_tiles`, both return -1 for an unknown room), `LayoutSolidQuery extends SolidQuery` and `LayoutOccluderSource extends OccluderSource` (ADR-0014), exposed as `layout.solid_query()` and `layout.occluder_source()`. The composition root (a) calls `LayoutModule.setup` (no Graph argument), (b) passes `layout.room_source()` to `GraphModule.setup(facet, tuning, catalog, rooms, strict_numeric, work)` (the signature is ADR-0011's), (c) calls `LayoutModule.bind_graph(port: LayoutGraphPort)` with `graph.layout_port()` after `GraphModule.setup` and **before the registry freeze**. A port call made before `bind_graph` is an asserted defect. The `architecture.md` § 8 step 4 note is updated accordingly (routed to `/architecture-review`).

**What the block never holds.** Floats; geometry; the section `kind` (the Graph's section record owns it, LS-17); a `visited` bit (the block is WORLD-scoped and rolls back on `request_restore`); a loose-item table (Interaction's `interaction.state` holds it and lists `layout.state` in its own `restore_after`); a door `area` or `door_state`; atmosphere; pose, velocity or any `body.state` field; a stored `layout_hash` or `mutation_count` (Save/Load's `block_state_hash` owns the stored copy, LY D.10, OQ-36). This is TR-section-028 and the first half of TR-section-013.

**Source layout (ADR-0010).** `src/sim/layout/`: `layout_module.gd` (owner, registration, callables), `layout_state.gd`, `layout_derived.gd`, `layout_ids.gd`, `socket_ref.gd`, `layout_faults.gd`, `layout_mutators.gd` (LM1 to LM4 and checks), `weld_math.gd` and `detach_planner.gd` (pure), `door_leaves.gd` (the hook), `layout_restore.gd` (validation), `rect_flags.gd` (the `RectFlags` bit values and `ALL` mask, § 5), `slot_feed.gd` (§ 11), `layout_room_source.gd` (`LayoutRoomSource extends RoomSource`), `layout_solid_query.gd` and `layout_occluder_source.gd` (ADR-0014 adapters), `views/` (`LayoutViews` and the record classes of § 11, including `SectionBases`, `OpeningRead`, `SeamOpenings`, `SectionCounts`), `ports/layout_graph_port.gd` (`@abstract`, § 8). Body's adapters (`LayoutCollide`, the `BodyLayoutPort` implementation) live in `src/sim/body/` (ADR-0015). `socket_ref.gd` and `layout_ids.gd` join the `load_all` parse gate (ADR-0010 § 5.2) as the first Layout stories. The PROJECTOR is `src/presentation/projector/` and is never imported from here. The pure compile of authored runs into the rect table (`TemplateCompiler`) lives under `src/data/sections/` beside the loader that calls it, owned by Layout by review, not by import (ADR-0007 § 1.9).

### 2. The snapshot payload: dense parallel columns, ascending ids, tombstones kept

`snapshot(ctx)` calls `ctx.put(payload)` with a `Dictionary` whose keys are `String`s. Every table is a set of **parallel columns** (TR-section-006): no `Dictionary` is keyed by an id, no `Vector2i`/`Rect2i`/`Color`/`Resource`/`StringName`, no `PackedStringArray`, and every packed array in the payload is a `duplicate()` (ADR-0003 snapshot contract; registry `live_reference_in_snapshot_payload`). The key set is closed; the AC-130 allow-list file is regenerated from this table.

| Payload key | Type | Contents (row order) |
|---|---|---|
| `schema_version` | `int` | `LAYOUT_BLOCK_SCHEMA_VERSION` |
| `counters` | `Dictionary` of three `int`s | `next_section_id`, `next_body_id`, `next_room_id` (first allocated id is 1, so `default_state` sets all three to 1) |
| `bodies` | `Dictionary` | `body_id` (`PackedInt32Array`), `origin_section_id` (`PackedInt32Array`), `state` (`PackedByteArray`: 0 `LIVE`, 1 `DISSOLVED`); ascending `body_id` |
| `instances` | `Dictionary` of 14 columns | `section_id`, `body_id`, `offset_x`, `offset_y`, `room_id_base`, `node_base`, `edge_base`, `comp_base`, `geometry_hash`, `visual_hash`, `structure_hash` (all `PackedInt32Array`); `rot_q`, `discarded` (`PackedByteArray`); `template_id` (`Array[String]`); ascending `section_id` |
| `sockets` | `Dictionary` | `section_id` (`PackedInt32Array`), `socket_index` (`PackedByteArray`), `state` (`PackedByteArray`: 1 `WELDED`, 2 `CAPPED`, 3 `CUT_OPEN`; `FREE` = 0 is never stored), `seam_id` (`PackedInt32Array`); ascending `(section_id, socket_index)`; only non-`FREE` sockets have a row |
| `welds` | `Dictionary` | `seam_id`, `section_a`, `section_b` (`PackedInt32Array`), `socket_a`, `socket_b`, `state` (`PackedByteArray`; `state`: 0 `ACTIVE`, 1 `CUT_SEALED`, 2 `CUT_RAGGED`), `welded_at_T`, `cut_at_T` (`PackedInt64Array`); ascending `seam_id` |

Decisions inside the shape:

- **Ids are dense ordinals: `section_id == row + 1` and `body_id == row + 1`.** A section or body id is consumed only by the mutator that appends its row, in the same call, so no id exists without a row. Row lookup is then an index, not a map: no `Dictionary` on the read path, no `bsearch` for the common query. Restore verifies the equality (`LAYOUT_RESTORE_COUNTER`). The cost is that a future "reserve an id then maybe not use it" idea is forbidden by this ADR, which is the right pressure (§ Risks R1).
- **`template_id` is a typed `Array[String]` column** (TR-section-004). The alternatives were the ledger integer (rejected: the GDD allow-list, LY-6 b and every restore reason name the string; four templates; saves are disposable pre-release; a human-readable dump matters more than 4 bytes a row) and a `PackedStringArray` (rejected: not on ADR-0006's allow-list). ADR-0006 does not assert that a typed array survives the codec, so **restore never relies on it**: it checks each element with `typeof(v) == TYPE_STRING` (`LAYOUT_RESTORE_SCHEMA` on failure) and fills the typed column with `Array[String].assign(src)` (§ 10).
- **64-bit times are native `PackedInt64Array` columns** (TR-section-006). ADR-0006 § 1.2 item 3 allows full-width `int` and § 1.3 limits the halves convention to values that can reach JSON; `welded_at_T`/`cut_at_T` reach JSON only through the dev `dump-save`, which renders any `int` of magnitude 2^53 or more as a decimal string and `T` is bounded below 2^53. The GDD's two-halves form stays legal and is not used.
- **The three hashes are stored as signed int32 and compared as `value & 0xFFFFFFFF`** (`IntMath.low32`, ADR-0005 P9; probed 2026-10-29). A comparison without the mask is a lint-visible defect (AC-172 c).
- **`rot_q` is a byte column that must be 0.** It is stored so the field exists in schema v1 and a later rotation feature is not a layout change; any nonzero value is `LAYOUT_ROT_UNSUPPORTED` at restore and a boot error in a template that asks for it.
- **`socket_index` is the template's declaration ordinal** of that socket, a `uint8` (MAX_SOCKETS_PER_SECTION never exceeds 8 and is lint-bounded by `MAX_SEAMS_PER_SECTION`). A runtime socket handle is the packed int `SocketRef = (section_id << 4) | socket_index` (`socket_ref.gd`, 24 bits at the section-id ceiling), used as an argument, a memo key and a sort key; it is never persisted.
- **Seam ids are the Graph's.** A `welds` row's `seam_id` is whatever `weld_join` returned; Layout allocates none and treats it as an opaque positive `int32`. After a cut the two sockets keep their `seam_id` (state `CAPPED` or `CUT_OPEN`) because the scar's bead and cap plate are drawn from that history (Pillar 5).
- **The payload size rule holds.** The largest realistic shape (D.21's long campaign) is tens of KB, far under `MAX_BLOCK_BYTES`; `snapshot()` computes the D.21 estimate from row counts and raises the one-shot diagnostic `layout_block_soft_bytes` when it exceeds `LAYOUT_BLOCK_SOFT_BYTES` (GDD Tuning Knobs); the hard ceiling is Save/Load's (TR-section-030).

**Tombstones.** A discarded section keeps its row with `discarded = 1` and its `body_id`; the body it sat on becomes `DISSOLVED`; its `socket` and `welds` rows stay (a surviving mate keeps its `CAPPED`/`CUT_OPEN` history). Ids stay consumed, so nothing is ever reused or renumbered (TR-section-005).

### 3. ID allocation and the split with the Graph (ADR-LAY-05)

| Id | Allocator | Counter and home | Range and rule |
|---|---|---|---|
| `section_id` | **Layout** | `next_section_id` in `layout.state` | first 1; `<= 2^20 - 1` so `rect_key` fits 32 bits (D.9); `can_build` returns `CAPACITY` before consuming `2^20` |
| `body_id` | **Layout** | `next_body_id` in `layout.state` | first 1; `<= 2^31 - 1` (int32 column); one new id per LM1 and per LM3 |
| `room_id` | **Layout** | `next_room_id` in `layout.state` | first 1; `<= 2^31 - 1`; **contiguous per instance**: `room_id = room_id_base + room_index`, `room_index` the room's declaration order in the template |
| `node_id`, `edge_id`, compartment id | **Graph** | `next_node_id`, `next_edge_id` in `graph.state` (there is no compartment counter: a compartment id is a COMPARTMENT node id taken from `next_node_id`; `comp_base == node_base`, ADR-0011 § 4.4) | returned by M1 as `SectionBases {node_base, edge_base, comp_base}` and stored by Layout per instance as `graph_ref_base` |
| `seam_id` | **Graph** | `next_seam_id` in `graph.state` | given to Layout in LM2's arguments |
| opening ids | **Graph** | `next_edge_id` (an opening is an edge) | read back through `seam_openings_into` (`opening_ids` in `socket_plan.openings` order) |
| `section_id` in `graph.state` | follows Layout | `next_section_id` in `graph.state` | M1 advances it to `max(section_id) + 1`; Layout never reads it except in the post-restore verification |

**Mapping rules (all derived, none persisted).**

- Template-local index to Graph id: `node_id = node_base + node_ref_index`, `edge_id = edge_base + edge_ref_index`; template-local room to compartment: `comp_id = comp_base + room_index`; room to id: `room_id = room_id_base + room_index`. Hence `compartment_of_room(room_id) = comp_base(section) + (room_id - room_id_base(section))` and the inverse.
- Reverse lookup (a Graph id or a `room_id` to its section) is a `bsearch` over the ascending base column (`node_base`, `edge_base`, `comp_base`, `room_id_base` are all ascending because every allocator is monotone and rows append). **The mapping covers template-range ids only** (`[node_base, node_base + template.node_count)` and the matching edge range). A section whose range does not contain the id answers `NONE` (a discarded section's `anchor_geometry` answers nothing, without an error, FT:703). **Stub nodes** (the `HULL_SEGMENT` rows a RAGGED `cut_detach` appends with fresh high ids, flagged STUB, outside the section's contiguous M1 range, ADR-0011 §§ 3.6, 4.1) and seam edges resolve through `GraphView.section_of(id)` plus the seam's stub list, never through this `bsearch`; `anchor_geometry(stub)` answers **the cut socket's hatch span** (the hatch rects of the cut seam, section-local to the cut section), so Failure & Trail and the Emitter's fault glow can place a leaking stub.
- These rules require the Graph to return **contiguous** node, edge and compartment ranges **in template order** from M1 (cross-ADR assumption on ADR-0011) and to size each range from the template alone. The post-restore check (§ 10 (d)) compares the **template-range** counts only; stubs and seam edges are excluded.

**When ids are consumed.** LM1 reads (does not yet advance) `next_section_id`, `next_body_id` and `next_room_id`, calls the Graph's M1 through the port with the section id (M1's only caller is this call), and only on an `OK` status appends the rows and advances the counters. So a Graph defect leaves Layout's counters and tables untouched (fail-closed per ADR-0003 § 5; the kernel is `FAULTED` anyway). `can_build` and `can_detach` test every counter ceiling **before** any id is consumed or the Graph is called, so no counter ever wraps (TR-section-037).

**Why Layout owns the section id.** The Graph's M1 takes "a caller section id" (LS-15, LS-19), the Graph's `next_section_id` then follows Layout's, and Layout needs the section id to exist before the Graph does (it keys the dirty key and the body row). The alternative of the Graph allocating it (§ Alternatives 4) would make Layout's `build_section` depend on a return value to name its own row and would put the `rect_key` bound (2^20) in the Graph's hands.

### 4. Template identity, the ledger nesting (OQ-B) and the three hashes (ADR-LAY-04, -06)

**Ledger nesting (ADR-0007 OQ-B).** `assets/data/sections/ids.lock.json` is one `section_templates` space with one flat `pairs[]`, exactly the shape ADR-0007 § 3.2 defines, so `LedgerCheck.verify` stays generic and the append-only CI gate needs no second format. Two classes share the space, told apart by **key shape**: a template pair has key `<template_id>`, a room pair has key `<template_id>/<room_key>` (the `/` is illegal inside a snake_case key, so the split is unambiguous). Template ids come from the low part of the declared range and room ids from above a fixed boundary (a `SectionLedgerCheck` in `src/data/sections/`, owned by Layout by review, checks the boundary, that every room pair's prefix is a template pair in the same ledger, and that retiring or dormanting a template does the same to its room pairs: lint `LEDGER_ROOM_ORPHAN`). One entry per `room_key` rather than a nested list keeps `dormant` and `retired_ids` per room, so removing one room from a template is a per-pair `dormant` flip and a renamed room is a visible new pair, which is the diff a migration step needs. The integers are **not persisted in `layout.state`** (the instance persists the `template_id` string, § 2); they exist so a rename or removal after `SHIPPED` is detected and migrated (ADR-0007 § 3.3), and so later blocks can reference a room by integer if they must.

**The three hashes (the shape of ADR-LAY-06).** ADR-0005 § 3.1 forbids an owner-written `[u32 tag][u32 length][payload]` byte string, and § 4.6 already defines the template hashes as `Hash64.of_value` of a post-cast sub-payload "whose shape is Layout's and ADR-0013's". So ADR-LAY-06 becomes the **`Dictionary` shape of three sub-payloads**, hashed once by `SectionTemplates` at catalog freeze and stored as `low32` in the template row (never in the template file, ADR-0007 § 3.5):

| Hash | Payload (keys sorted by the encoder, so order in source is irrelevant; every family is one flat `PackedInt32Array` in the field order of its LY-4 row, strings replaced by template-local indices, variable-length members as a CSR `*_off` array plus a data array) | Changes when |
|---|---|---|
| `geometry_hash` | `size_tiles`; `rooms` (origin, flags, tile runs); `wall_runs` (orientation, start, length, kind, flanking rooms, `face_h_px`, `face_variant`); the compiled `rects` (stride 7: `x0, y0, x1, y1, flags, owner_kind, owner_ref`, in `idx` order, leaf and variant slots at their closed or full extent); `doors`; `sockets` with their `lanes`; `hardpoints`; `plate_positions`; `loose_slots`; `story_slots`; `anchors` (polyline CSR); `fixture_anchors`; `instrument_placements` (geometry fields only) | anything that decides where a rectangle, tile, socket, door or anchor is |
| `visual_hash` | `dialect`; kit-piece ids and layers; art references; labels; advisory hints (`socket_tag`); flags no query reads | only presentation changes |
| `structure_hash` | the **ordered** key sequences: `room_key[]`, `socket_key[]`, `door_key[]`, `hp_key[]`, `node_ref[]`, `edge_ref[]`, `anchor_key[]`, `slot_id[]`; each room's sorted tile list | any reorder, add, remove or rename of an identity-bearing element, or any tile-set change |

**Why a third hash.** LY-26 e classifies a template change as "structural" when the *sets* of keys or a room's tile set differ, but the block persists only two hashes, so the check has nothing to compare against except Graph counts. A pure **reordering** (rooms `[a, b]` to `[b, a]`) keeps every set equal and every count equal, yet `room_id = room_id_base + room_index`, `socket_index` and `comp_base + room_index` all index the *current* template by position, so an existing save would silently attach its Graph compartments and welded sockets to the wrong rooms and sockets. LY-6 e's sentence "a template that merely reorders rooms changes ids only for NEW instances" is therefore wrong as written. The remedy is to persist what the check needs: `structure_hash` is one more `int32` per instance (§ 2), costs four bytes a row, and makes the structural rule exact: **a `structure_hash` mismatch is `LAYOUT_RESTORE_TEMPLATE_CHANGED` for every instance, history or not**, because every instance has Graph data built from the old order. `geometry_hash` keeps the GDD's history-dependent rule (§ 10). The third hash is not redundant with `geometry_hash`: because `rooms` are encoded in declaration order, a reorder already changes `geometry_hash`, which on an instance **with history** is refused by the GDD's own rule; the marginal value of `structure_hash` is the **never-welded `FREE` instance**, where the history-dependent rule would accept the change as `geometry_changed_fresh` and silently re-point `room_id_base + room_index`, `comp_base + room_index` and `socket_index` to the wrong rooms and sockets. This is a deliberate deviation from the TR text "13-int section-instance rows" (the row is 14 columns, one of them the string); it is listed first among the decisions for the user's review.

**Two more template rules.** (a) Every coordinate of a footprint rectangle, socket `line_px`/`start_px`, room `origin_px` and wall run lies on the 24-px grid (`WALL_FOOTPRINT_DEPTH`) except a door leaf's moving edge (LY-3 a, LY-7); `GRID_24` is a **BOOT invariant** (`CONTENT_INVARIANT`, ADR-0007 § 2.4), not only a CI lint, because the closure of the weld offset on the grid (§ 7) and the broadphase cell alignment depend on it; a 12-px shift in a socket fails the load (TR-derelict-033 Layout half). (b) `area_max <= 0` on a door is the one named structural invariant D.5 needs (it divides by it) and refuses the load; the published `area_max` range is guidance (ADR-0007 § 2.5). (c) `area_max <= width_px` (the 96-px door cell) is a second `CONTENT_INVARIANT` (ruling CT-14). It is a numeric bound between `area_max` in flow units (ADR-0011 § 2.1: 64 = one standard door) and `width_px` in pixels, with no unit conversion implied: the leaf gap `floor_div(width_px * area, area_max)` is a dimensionless ratio that never exceeds `width_px` (the leaf is never wider than the doorway it closes), and the bound guarantees that one flow unit opens at least one pixel, so `open_extent_px > 0` exactly when `area > 0` (ADR-0034 OQ-E) and the jamb/leaf extents stay inside the doorway.

### 5. Rectangles, units and frames: the record, the flags and the reserved `idx` space

ADR-0014 owns the broadphase, `QueryBuffer`, `OccluderSource` and the query signatures. This ADR owns **what a rectangle is, how it is keyed and which ones can exist**.

- **The record (TR-section-010).** A rect is `(x0, y0, x1, y1)` in section-local base px, **half-open** `[x0, x1) x [y0, y1)` over pixel cells, with one flags byte: `SOLID`, `OCCLUDES_LIGHT`, `DOOR_LEAF`, `PROP`, `MACHINE`, `HULL` (registry `layout_rect_flags`); the bit values and the `ALL` mask live in `src/sim/layout/rect_flags.gd` (`RectFlags`), which ADR-0014 and ADR-0034 import. The legal flag combinations are LY-7's table and a boot error otherwise (`RECT_FLAGS_ILLEGAL`). A rect with `x1 <= x0` or `y1 <= y0` is omitted from every result but keeps its `idx`.
- **Units and frames.** The unit is the base pixel; `TILE_PX` is the floor tile and `WALL_FOOTPRINT_DEPTH` the wall depth. There is no world frame in Layout. Reads answer **section-local** except those that answer for one body (`rects_in`, `solid_in`, `clip_to_footprints`, `leaf_would_overlap`, `section_at`, `weld_offset`, `detach_preview`), which answer in the **body frame** = section-local + `offset_px(section_id)` (LY-3 c). **The one section-keyed exception** is ADR-0014's `section_rects_into(section_id, mask, out)`: it is keyed by section but its rows are in the **body frame** (occupied spans, width = span + 1); a section-scene consumer subtracts `section_offset_x/_y(section_id)`, and a row whose current extent is empty is omitted (the consumer hides that sprite). The kind of a row (JAMB, PLATE, LEAF and so on) is derived from the reserved `idx` ranges below.
- **Transforms are integer translation only.** No rotation, no mirror, no zoom value appears anywhere in Layout (LY-3 d, e). Every px-to-tile or px-to-cell conversion is **floor** division through `IntMath.floor_div` (body-frame coordinates may be negative after a detach re-base); the bare `/` and `%` operators appear nowhere in `src/sim/layout/` (ADR-0010 `SIM-05`).
- **The `idx` space is compiled once per template and never changes** (D.9): static rects sorted by `(y0, x0, y1, x1, flags)` are `0 .. N_static - 1`; one reserved leaf slot per door follows in door declaration order; two reserved variant slots (PLATE, JAMB) per **hatched** socket follow, `idx = N_static + N_doors + 2h + v`. The key `(section_id, idx)` is therefore stable across attach, detach, restore and animation: a state change only switches slots between active and inactive; an inactive or zero-length slot is omitted from results and keeps its `idx` (TR-section-020's "renumbers nothing" half).
- **Rect keys.** `rect_key = section_id * RECT_KEY_STRIDE + idx` with the stride `MAX_RECTS_PER_SECTION` (GDD Tuning Knobs, a power of two) and `idx < stride`; the key reaches `2^32 - 1`, so every key column is `PackedInt64Array` (ADR-0014 owns the buffer). Boot refuses a template whose `N_static + N_doors + 2 N_hatched` exceeds the stride (`RECT_STRIDE_OVERFLOW`).
- **The compiled table is catalog data.** `SectionTemplate` exposes rect `idx`-indexed accessors (`rect_count()`, `rect_x0(i)`, `rect_y0(i)`, `rect_x1(i)`, `rect_y1(i)`, `rect_flags(i)`, `rect_owner_kind(i)`, `rect_owner_ref(i)`), never a packed array (ADR-0007 § 1.5). The bulk read that builds a body's grid at build, attach, detach and restore is ADR-0014's; those events are cold, so accessor calls are acceptable there (V3 measures them).
- **Static footprint set versus query set.** `rects_in` returns `OCCLUDES_LIGHT` rects and `solid_in` returns `SOLID` rects in their *current leaf state*; the mating overlap test of § 6 uses the **static** set with all flags, so a door animating can never flip a verdict. Both sets are views of one rect set (Pillar 1: collision, light and drawing cannot disagree); neither is an engine collision shape, and the avatar has no physics body.

### 6. Sockets, seams and the mating predicate

- **A socket is a pre-declared attachment site** `{socket_key, facing N|E|S|W, line_px, start_px, width_tiles, class_id, lanes[], hatch | none}` in the template (LY-13). `pos_px` is `(line_px, start_px)` for E/W facings and `(start_px, line_px)` for N/S. The four-facing footprint, LOW-end, PLATE/JAMB/LEAF table of LY-13 is normative and is not restated.
- **States (TR-section-018, TR-assembly-025).** `FREE -> WELDED(seam_id) -> CAPPED` (after a SEALED cut) or `CUT_OPEN` (after a RAGGED cut); `CAPPED` and `CUT_OPEN` never return to `FREE` in MVP, which supports exactly one weld-then-cut cycle per socket. **A socket's state is written only inside LM2 and LM3.** There is no `install_cap_plate`, no `set_socket_state` and no setter on `LayoutState.sockets`; `socket_state` writes are private to `weld_math`/`detach_planner` helpers called by those two mutators, and a lint (`LAY-SOCKET-01`, tier E1, ADR-0010 § 6) rejects any other write to the `sockets` columns.
- **`can_attach` is a pure, first-failure-wins reason chain** (TR-section-019), cheap before expensive, in this fixed order: `SOCKET_UNKNOWN`, `SOCKET_NOT_FREE`, `SAME_BODY`, `FACING_MISMATCH`, `WIDTH_MISMATCH`, `CLASS_MISMATCH`, `LANE_MISMATCH`, `COORD_LIMIT`, `GEOM_OVERLAP`. `COORD_LIMIT` (ruling CT-24) refuses a weld whose resulting body-frame extent (the survivor's sections plus the absorbed sections at `offset_b`, from `weld_math`, no geometry test) would exceed `BODY_COORD_LIMIT`, the bound ADR-0015 defines in `BodyTuning` and its bounds ledger (at most 2^24) and which Layout reads from the same `ScalarTable` value through `LayoutTuning`, never from `body.state`; it is cheap integer arithmetic, so it precedes `GEOM_OVERLAP`. `SAME_BODY` being a hard reject and every weld merging two bodies is what makes the weld relation a **forest** and a cut always split a body into exactly two components.
- **`GEOM_OVERLAP`.** The test set is the static footprint of both bodies after the D.6 translation: every static rect with all flags, each PLATE slot at its full footprint, no leaf and no JAMB slot, plus the tile cells of every room (rect against tile and tile against tile, so a wall over the other body's floor and two floors on one cell both fail); abutting is legal. **The rect-vs-rect leg is ADR-0014's `footprint_overlaps`; the tile-cell leg (rect-vs-tile and tile-vs-tile) is Layout's own `floor_tile_at`-based check, run after `footprint_overlaps` returns no rect hit** (ruling CT-16). The test runs over each body's broadphase handle (ADR-0014) and is **memoised on `(socket_ref_a, socket_ref_b, static_version_a, static_version_b)`**; uncached cost target <= 2 ms (TR-section-019), a pending-spike figure (V3). The memo is a small fixed-capacity ring in `LayoutDerived`, reset on restore.
- **Slot activation is a pure function** of `(this socket's state, this socket's hatch, the mate's hatch, W)` evaluated by LM1 to LM4 and `post_restore` (the LY-15 matrix); no persisted field is needed because the mate's hatch is known from the two templates. A hatched socket whose mate is bare is `hatch_one_sided`: legal, behaves bulkhead-only, its PLATE stays on while `WELDED`.
- **The `can_attach` preview needs more than a reason**, and a Class A check returns a bare `StringName` (ADR-0003 § 5). So the payload `{reason, hatch_one_sided, openings_count}` is its own pure read, `preview_attach(socket_ref_a, socket_ref_b, out: AttachPreview) -> void`, and `can_attach(...)` is `preview_attach` followed by returning `out.reason` (`&""` = OK). One implementation, two shapes.
- **`socket_plan(socket_ref_a, socket_ref_b, out: WeldPlan)`** is a pure read that is **canonicalised survivor-first**: it relabels the two arguments through `weld_math.survivor_of` (the survivor rule of § 7, Alternative 9) before filling anything, so the `JoinPlan` the Graph's `weld_join` receives, the `opening_ids` order and the Graph's seam row (`section_a`/`section_b`) are argument-order independent (ruling CT-13; ADR-0011 notes that `weld_join` row order follows the canonical `JoinPlan`). It fills the Graph's weld-plan DATA verbatim (one lane pair per lane in `(loop, lane_index)` order, one opening per mated hatch pair `(comp_a, comp_b, area_max)`, `L_seam = width_tiles`). The field layout of `WeldPlan` follows ADR-0011's `JoinPlan`; Layout fills it, Assembly adds `q`, the Graph's `can_join` validates, and both verdicts must pass before apply.

### 7. Weld offsets, bodies and the detach re-base

**Bodies.** `bodies[]` is Layout's: `{body_id, origin_section_id, state}`. **Every built section starts on its own new body at offset (0, 0), and every multi-section body is produced by welds only** (LY-25): there is no "place on an existing body" path, so the forest property holds by construction. A body's frame origin is the origin of its `origin_section_id`.

**Weld offset (D.6, TR-section-021).** `offset_b = offset_a + pos_a - pos_b`, where `a` is the **survivor-side** socket and `b` the absorbed-side socket, assigned **by body, never by argument order**: the body with the **lower `body_id`** survives (the older body; the home ship in the MVP). `attach_section` re-labels the two arguments internally, so `attach_section(x, y, ...)` and `attach_section(y, x, ...)` produce the same survivor, the same `offset_b`, the same `frame_shift_px`, and a **byte-identical** `layout.state`: the `welds[]` row always records the survivor-side section and socket as `section_a`/`socket_a` (canonicalisation). `weld_math.gd` is a pure module of integer helpers (`survivor_of`, `offset_b`, `frame_shift`) with no state, which `weld_offset(...)` (the pure read), `can_attach` and LM2 all call, so prediction and apply cannot drift.

**24-grid closure.** `pos_a`, `pos_b` and `offset_a` lie on the 24-px grid (`GRID_24` is a boot invariant, § 4), and `offset_a` does by induction from (0, 0), so `offset_b` does; an off-grid `offset_b` from a validated plan is therefore a **defect**, not a reason code: `apply` raises `LAYOUT_OFFSET_OFF_GRID` (domain 4) and returns its negative.

**LM2 effects, in order** (all inside the caller's single P2 apply, after the Graph's `weld_join`): append the `welds[]` row (`ACTIVE`); upsert both socket rows to `WELDED(seam_id)` keeping ascending order; shift every section of the absorbed body by `frame_shift_px = offset_b - offset_b_old` (the vector; the records carry it as `frame_shift_x` and `frame_shift_y`) and move it to the survivor `body_id`; mark the absorbed body `DISSOLVED`; switch the PLATE/JAMB/LEAF slots of both sockets (§ 6); **refresh the three stored hashes** (`geometry_hash`, `visual_hash`, `structure_hash`) of both socket-host sections from the current template if the instance had no history before this weld (it gets history now; § 10 explains why this, and not `restore`, is where a stored hash is rewritten); seed each newly active LEAF **closed** with its last-projected cache (§ 8); rebuild the affected broadphase handles (ADR-0014); bump `static_version` and `version` of **every section whose socket state changed (including the host section that does not move) and every moved section**; stamp `changed_region` (REGION for the two socket-host sections, FRAME for the other moved sections); cache the mated hatch's opening ids from `seam_openings_into(seam_id, out)` (the port, § 8) into `opening_leaves`; mark `S/<seam_id>`, `BODY/<survivor>`, `BODY/<absorbed>` and `SEC/<id>` for each affected section (5 keys for the MVP weld of two single-section bodies). The returned `AttachResult` carries `{surviving_body_id, absorbed_body_id, moved_sections[], frame_shift_x, frame_shift_y}`; `body_of` of every moved section is the survivor in the same apply and is readable by any ordered reader in the same tick. LM2 marks the absorbed body `DISSOLVED` **before** Body's `rebase_on_attach` runs (ADR-0015 BM3 therefore checks `body_state(absorbed) == DISSOLVED` at apply time, § 11).

**Result and traversal arrays are filled in place, never aliased.** Packed arrays are shared by reference in GDScript (`var t := s` aliases; only `duplicate()` separates). `AttachResult.moved_sections`, `DetachResult.moved_sections` and `rects` are caller-owned reused arrays, and the traversal stack and visited columns live in `LayoutDerived`. Rules: (1) a result array is sized once at setup (`resize`) and written by index with a separate `count` int; **an implementation never assigns another array (a `LayoutDerived` column or a local) to a result field**, or the caller would alias Layout's scratch and the next apply would silently rewrite it; (2) traversal stacks are fixed-length arrays with an explicit top index, not `clear()` / `push_back()` (`clear()` and `resize(0)` may release the buffer; V3b); (3) a result is valid until the next Layout call, and any consumer that retains one across calls (Assembly's transaction log, events, Body's re-base bookkeeping) must `duplicate()` it.

**LM3 and the detach re-base (D.12).** After the Graph's `cut_detach`, LM3 marks the weld row `CUT_SEALED` or `CUT_RAGGED` with `cut_at_T`; sets both sockets `CAPPED` or `CUT_OPEN` (keeping `seam_id`); then **enumerates the connected components of the `ACTIVE` weld forest minus the cut seam, in ascending `section_id`**. The component containing the body's `origin_section_id` keeps its `body_id`; the other receives a new `body_id` from `next_body_id`, its `origin_section_id` is `m`, the minimum `section_id` of that component, and its offsets are re-based by `frame_shift = offset_old(m)` (`offset_new(s) = offset_old(s) - frame_shift`). The traversal uses preallocated fixed-length `PackedInt32Array` stack and visited columns in `LayoutDerived` with an explicit top index, bounded by `MAX_SECTIONS`, so no container is built in the apply. `DetachPlanner.plan(...)` is the single implementation behind both `detach_preview(seam_id)` and LM3, so the preview and the apply cannot drift (Assembly's `MSA_SHIFT_DRIFT`). A cut that does not yield exactly two components is a defect (`LAYOUT_COMPONENT_COUNT`), unreachable by the forest argument. The `DetachResult` is `{new_body_id, moved_sections[], frame_shift_x, frame_shift_y, rects[]}` with `rects[]` the cut seam's two hatch rectangles in the pre-cut body frame, **computed before the sockets change**, exactly equal to `detach_preview(seam_id).rects`. Layout performs **no actor check** (it reads nothing from `body.state`): the CHANNEL path is vetoed by Assembly's completion validate, and Body's `rebase_on_detach` is the net behind it (LY-17, BDY-24 iii).

**Why not argument order or "the larger body".** Argument order makes the dissolved body depend on a caller's mixup (the home ship could dissolve) and makes the block bytes differ between two equal worlds, which breaks AC-172 (equal blocks by different histories hash equal). "Larger body survives" would make survival depend on a count that changes with every weld. The lower `body_id` is a persisted, immutable, total order.

### 8. The door leaf, the `layout.doors` hook and the Graph port

**One rectangle per leaf.** A door or hatch has one leaf rect. With `width_px` (the 96-px door cell), the Graph opening's `area` clamped to `[0, area_max]` and `area_max > 0`: `gap = floor_div(width_px * area, area_max)`, `leaf_w = width_px - gap`, and the rect is anchored to the hinge side with depth `WALL_FOOTPRINT_DEPTH` (D.5; hinge LOW for every socket hatch so both leaves of a mated hatch retract the same way). `leaf_w = 0` returns no rect and keeps its `idx`. A mated hatch is two leaves (one per section) driven by one `area`.

**The port.** Layout reads and calls the Graph only through `LayoutGraphPort` (`src/sim/layout/ports/layout_graph_port.gd`, `@abstract`, ADR-0010 § 3); this ADR owns the signature and the four caller-owned records (§ 11). It is implemented by ADR-0011's dedicated adapter `GraphLayoutAdapter` (`src/sim/graph/graph_layout_adapter.gd`, outside `ports/` because ADR-0010 ARCH-05 requires every file under `ports/` to be `@abstract`; it holds `GraphMutators` and `GraphView`, so `GraphView` keeps its pure getter-only contract) and injected through `LayoutModule.bind_graph(port)` (§ 1), not through `setup()`:

```gdscript
@abstract class_name LayoutGraphPort extends RefCounted
@abstract func check_instantiate_section(views: SimViews, section_id: int, template_id: String, kind: int) -> StringName   # pure Class A check: LS-18 reasons plus CAPACITY and ONSET_UNKEYED (called by can_build(views, template_id, kind) with section_id = the pending next_section_id)
@abstract func instantiate_section(scope: WriteScope, section_id: int, template_id: String, kind: int, out: SectionBases) -> int
@abstract func opening_into(edge_id: int, out: OpeningRead) -> bool   # false = unknown edge; fills area, area_max, door_state
@abstract func seam_openings_into(seam_id: int, out: SeamOpenings) -> bool   # opening_ids[] in socket_plan.openings order; valid after restore
@abstract func section_counts_into(section_id: int, out: SectionCounts) -> bool   # template-range counts, stub_count, tombstoned, next_section_id (persisted columns only, Tier S; verification)
```

The 4.7 typed-return rule applies to any override of a typed-return abstract method, so each implementation returns explicitly on every path (a stub that omits a return is a parse error or a defect found by V-lint). The reads are allocation-free (caller-owned `out`) and valid in the APPLY (P2 after the Graph's own writes), INIT, HOOK (P3) and `post_restore` scopes; the writer `instantiate_section` is valid in APPLY and INIT only (the pure check `check_instantiate_section` runs wherever `can_build` runs, in validate). `opening_into` is valid in INIT because LM1 seeds leaves in the `INIT` scope of `init_world`. A port call before `bind_graph` is an asserted defect. In an implementation, unused parameters are `_`-prefixed (`_views`, `_out`) (Guidelines).

**The hook (TR-section-012).** `layout.doors`: `hook_id &"layout.doors"`, owner Layout, `divisor 1`, `offset 0`, `after []`, running in P3. Body's `body.integrate` is registered `after: [controller.step, layout.doors]` (BDY-8), which orders the Emitter after it transitively, so every P3 reader sees same-tick leaves. The hook loops over the `LayoutDerived` leaf cache, a dense list of at most `MAX_DOORS_LOADED` slots in ascending `(section_id, door_index)` (deterministic bump order):

1. `graph.opening_into(edge, op)`. If it returns false (unknown opening) the leaf stays at its last projected value (closed at the template default), `area` is never invented, `leaf_version` is not bumped, and the diagnostic `opening_unknown` is raised **once per edge** (a bit in the slot's flags, cleared when the read succeeds again) (TR-section-035). A `area_max <= 0` read is treated the same way and counted `opening_invalid`.
2. `area` is clamped to `[0, area_max]`; a value above `area_max` raises the diagnostic `area_clamped` and uses `area_max` (TR-section-035).
3. `leaf_w` is computed (integer, `IntMath.floor_div`), compared with the slot's last-projected value, and if different the leaf rect is recomputed and the slot's section is flagged.
4. After the loop, for each flagged section in ascending id: `leaf_version += 1`, `version += 1`, never `static_version`; `changed_region` is stamped REGION(union of the old and new leaf rectangles). **At most one bump per section per tick** (TR-section-042).
5. If any section was flagged, the hook opens `layout.state` once (`scope.open_for`) and marks one **`LEAF/<section_id>`** key per flagged section.

**Why the hook marks a key, and why a new kind.** `layout.doors` changes only derived caches, yet the PROJECTOR must redraw a leaf whose rect changed and the door leaf sprite extent must equal the leaf rect with no tween (TR-section-013). ADR-0009 presenters draw from dirty batches; polling every section's counter every frame would add a read per frame for an event that happens a few times per play. Marking is legal (ADR-0004 § 7: a key marked for an unchanged value is not a violation; the HOOK scope may open any block). Its cost is that a granted `open_for` advances `touch_serial`, invalidating the block hash cache (ADR-0005 § 5.2) on those ticks; under the binary door model that is a few ticks per play, and the re-encode is bounded by the block size (tens of KB). LY-32's closed list is SEC, BODY, SEAM; a leaf change under `SEC/<id>` would also wake Failure & Trail's anchor rebuild and Lighting's static rebuild on every door toggle, so the key family **`LEAF` (kind 4)** is added to Layout's own key space (ADR-0004 § 3 leaves unnumbered families to the owner). Subscribers that care (the PROJECTOR, Lighting's dynamic leaf patch) test `batch.has(key)`; the rest ignore it.

**The correctness condition.** Every `area` write of a template door or mated hatch lands in **P2** by command (a SYSTEM-origin `set_opening` re-submitted each tick by Interaction's `attempts[]` for a `CLOSING_BLOCKED` retry); no Graph phase writes `area` or `door_state` of such an opening (the Graph's virtual breach and seam-weep openings have no leaf and are neither read nor projected). Layout stores, mirrors and writes neither value (TR-section-013); `layout_hash` does not cover door state. This is a cross-ADR assumption on ADR-0011/0012/0020 and a test (AC "closed leaf blocks and an open one does not in the same tick its state changes").

**Seeding.** LM1 reads the just-created openings through the port and derives each leaf immediately (the Graph holds authored initial areas), so there is no tick in which a built door is wrong. **LM2 does not read the Graph's `area`:** it initialises each newly active LEAF **closed** (`leaf_w = width_px`) and seeds the last-projected cache with that value in the same apply, because `weld_join` starts the new opening at `area` 0; reads between the P2 apply and the first P3 hook therefore see a solid span, raise no `stale_leaf`, and the hook bumps `leaf_version` only if its `area` read differs (TR-section-034). `post_restore` derives all leaves in one `derive_doors` pass (§ 10).

**Pure leaf queries.** `leaf_would_overlap(opening_edge_id, area, boxes: PackedInt32Array) -> bool` and `leaf_obstruction_into(opening_edge_id, area, boxes, out: LeafObstruction) -> bool` evaluate **every** leaf projected from that opening (`opening_leaves`: both sections of a mated hatch) at the new `area`, in the body frame, against every caller-supplied foot box (`boxes` is a flat half-open `x0, y0, x1, y1` array the caller owns); Layout reads nothing from `body.state` (TR-section-014). `door_passage_into(section_id, via_id, out: DoorPassage) -> bool` returns `{orientation, sill_rect, gap_rect, open_extent_px}` from the leaf value `layout.doors` last projected (so collision, light and passage cannot disagree and `opening` is not re-read); an unknown or discarded section or `via_id` returns false with no error; before `post_restore` it reports `open_extent_px` 0 and raises `stale_leaf`. The single passability rule is a pure helper `DoorPassage.passable(open_extent_px, orientation, foot_extent_ew, foot_extent_ns)`: passable iff `open_extent_px` is at least the foot extent across the doorway (the Body foot box, BDY-14: 16 across an east-west run, 8 across a north-south run, supplied by Body as constants, not owned here) (TR-horror-032).

**Cost.** The hook is an integer multiply and divide per door plus one port call and compare; the S1 probe shows it is dominated by the port call (one per door), not by the integer math (V2). Target <= 0.2 ms at `MAX_DOORS_LOADED` animating doors (5 percent of `STEP_BUDGET_MS`) on a release export, a pending-spike figure (V2). The fallback ladder is ADR-0010 § 9: cache accessors in locals, then early-out on an unchanged Graph `touch` counter if ADR-0011 publishes one, before any native rung.

### 9. Mutators, checks, results, faults and dirty keys

Layout has **exactly four mutators, all Class A**, and no Class B (it holds no quantity that can sensibly be clamped) (TR-section-020). Each has a pure check; `apply` is infallible once its check passed and a failure inside it is a defect that returns `-fault_code` after `scope.fault`.

| # | Check (pure) | Mutator | Caller (one) | Results (caller-owned `out`) | Dirty keys |
|---|---|---|---|---|---|
| LM1 | `can_build(views, template_id, kind) -> StringName` | `build_section(scope, template_id, kind, out: BuildResult) -> int` | Web Map `init_world` in the `INIT` scope; a pod hop builds nothing | `section_id`, `body_id` | `SEC/<id>`, `BODY/<id>` |
| LM2 | `can_attach(views, socket_ref_a, socket_ref_b) -> StringName` | `attach_section(scope, socket_ref_a, socket_ref_b, seam_id, welded_at_T, out: AttachResult) -> int` | Assembly `weld` apply, after the Graph's `weld_join` | § 7 | `S/<seam_id>`, 2 x `BODY`, `SEC` per affected section |
| LM3 | `can_detach(views, seam_id) -> StringName` | `detach_section(scope, seam_id, boundary_mode, out: DetachResult) -> int` | Assembly `seam_cut` apply, after the Graph's `cut_detach` | § 7 | `S/<seam_id>`, `BODY/<kept>`, `BODY/<new>`, `SEC` per affected section (6 in D.12's example) |
| LM4 | `can_discard(views, section_id) -> StringName` | `discard_section_geometry(scope, section_id) -> int` | Web Map only, after the Graph's M4; Assembly never discards | none | `SEC/<id>`, `BODY/<id>` |

**LM4 and versions.** LM4 does not bump the three counters; it calls `LayoutVersions.invalidate(section_id)` (all three counters set to 0, ADR-0014 § 6), so a consumer's cached value can never equal a live one. **LM4 and Body (ruling CT-23):** discarding a section can dissolve its body, and Body has no implicit cleanup, so in the discard apply BM8 `release_body` must follow LM4 (assumption on ADR-0026, the discard apply order: BM5/BM2 move or remove actors, then Graph M4, then LM4, then BM8). **Records.** `BuildResult`, `AttachResult` and `DetachResult` are caller-owned and reused; each resets itself through a `reset()` method (not an `_init` re-run) before every Layout fill (§ 11, Key Interfaces).

**Naming divergence, recorded.** ADR-0003 § 5 and the registry give the check as `check_x(views, args) -> StringName`. The GDD, 19 TR rows and four consumer GDDs publish `can_build`, `can_attach`, `can_detach`, `can_discard`. This ADR keeps the **published names** and the **exact shape** (pure, takes `views` and scalars, returns `&""` for OK and a reason otherwise); the divergence is a prefix and is listed as an ask to ADR-0003 (§ Related). The GDD's "returns `{section_id, body_id}`" becomes an `out` record, as ADR-0003 requires for every mutator that produces data.

**Reasons (closed sets, StringNames in `LayoutReasons`).** `can_build`: `TEMPLATE_UNKNOWN`, `CAPACITY`; a refusal by the Graph's `check_instantiate_section` is passed through as `layout.GRAPH_<REASON>` (CAPACITY maps to `CAPACITY`; the LS-18 reasons and `ONSET_UNKEYED` surface as `GRAPH_<REASON>`, which is unreachable at runtime for a validated template because the ADR-0007 template validator calls `graph_rules.onset_valid` at boot). `can_attach`: the nine of § 6. `can_detach`: `SEAM_UNKNOWN`, `SEAM_NOT_ACTIVE`, `CAPACITY`. `can_discard`: `SECTION_UNKNOWN`, `SECTION_WELDED`. They surface as `VALIDATION_FAILED(layout.<REASON>)` in the kernel's existing channel. `BODY_FULL` and `OFFSET_OFF_GRID` are not added (§ Open Questions OQ-1).

**Capacity enforcement (TR-section-030, -037).** `can_build` returns `CAPACITY` when any of the following holds, evaluated before any id is consumed or the Graph is called: live sections `>= MAX_SECTIONS` (tombstones do not count); live bodies `>= MAX_BODIES`; `next_section_id > 2^20 - 1`; `next_body_id > 2^31 - 1`; `next_room_id + rooms - 1 > 2^31 - 1`; the template's rects exceed `MAX_FOOTPRINT_RECTS_PER_BODY` or its doors plus loaded doors exceed `MAX_DOORS_LOADED`; rooms exceed `MAX_ROOMS_PER_SECTION`; or the Graph's `check_instantiate_section` refuses (see Reasons). `can_detach` returns `CAPACITY` when a body cannot be allocated: live bodies `>= MAX_BODIES` or `next_body_id > 2^31 - 1`, so a cut at the ceiling is a clean reject in validate, never a `FAULTED` apply. `MAX_BODIES` is counted over `LIVE` bodies, with a welded home ship plus a welded derelict B counting as one (TR-derelict-040 Layout half). `BODIES_LIVE_MAX` (Lighting's) is **not** a Layout reject; beyond it Lighting degrades lights to stand-ins and the world lint `BODIES_LIVE_MARGIN` reports the zero MVP margin. The per-section and per-world limits (`MAX_DOORS_PER_SECTION`, `MAX_SOCKETS_PER_SECTION`, `MAX_HARDPOINTS_PER_SECTION`, `MAX_ANCHORS_PER_SECTION`, `MAX_LOOSE_SLOTS_PER_SECTION`) are boot invariants over a template (`CONTENT_INVARIANT`); the world tile ceiling `TILES_WORLD_MVP` and the Graph node/edge ceilings are lints (Layout supplies `V`).

**Fault domain 4.** `LayoutFaults` (`src/sim/layout/layout_faults.gd`), pre-packed `(4 << 16) | local`, names appended and never renumbered:

| Local | Name | Raised by |
|---|---|---|
| 1 | `LAYOUT_RESTORE_SCHEMA` | restore: unsupported `stored_version`, missing or extra key, mismatched column lengths, wrong type (including a non-`String` `template_id` element), bad enum or an out-of-range byte value |
| 2 | `LAYOUT_RESTORE_UNKNOWN_TEMPLATE` | restore: a `template_id` not in the catalog |
| 3 | `LAYOUT_RESTORE_TEMPLATE_CHANGED` | restore: `structure_hash` mismatch, or `geometry_hash` mismatch on an instance with history |
| 4 | `LAYOUT_ROT_UNSUPPORTED` | restore (and boot): `rot_q != 0` |
| 5 | `LAYOUT_RESTORE_COUNTER` | restore: a counter not equal to the dense count + 1, a non-dense id column |
| 6 | `LAYOUT_RESTORE_DANGLING` | restore and post-restore: a weld, socket or body reference that does not resolve; an `area` above `area_max` found in `post_restore` (no clamp there; the clamp-with-`area_clamped` is the `layout.doors` hook's, TR-section-035) |
| 7 | `LAYOUT_OFFSET_OFF_GRID` | defect in LM2 apply |
| 8 | `LAYOUT_COMPONENT_COUNT` | defect in LM3 apply |
| 9 | `LAYOUT_APPLY_PRECONDITION` | defect: an apply reached without its check having passed |

Content boot errors stay in domain 2 (ADR-0007 § 2.3); Layout raises none of them from its own faults file. Diagnostics (counters, not faults): `opening_unknown`, `opening_invalid`, `area_clamped`, `stale_leaf`, `query_box_empty`, `geometry_changed_fresh`, `visual_changed`, `hatch_one_sided`, `layout_block_soft_bytes`, registered as one diagnostics channel (field list ADR-0031's).

**Key space.** `KeySpaceDescriptor` for `layout.state`: kinds `[1, 2, 3, 4]` named `[SEC, BODY, SEAM, LEAF]`, no events, `quiet_writes_allowed = false` (every discrete change marks a key; the per-mutator fixture asserts the specific keys). Keys are integers `(KIND << 32) | id` through `DirtyKey.make`; the worst MVP count (6) is far under `MAX_DIRTY_KEYS`.

**Order of writes inside a mutator.** Every mutator begins `scope.open_for(&"layout.state")`; on `false` it writes nothing and returns `REFUSED`. It then computes the full result (ids, offsets, components, slot sets) into locals and the `out` object **before** its first write to `LayoutState`, writes the columns, updates `LayoutDerived`, bumps counters, and marks keys last. A packed-array element write never relies on a setter (4.7).

**Hash and cadence.** `layout_hash()` calls the kernel `block_hash(&"layout.state")` (ADR-0005 § 5.3), memoised on `mutation_count`, a non-persisted counter bumped by LM1 to LM4 and reset to 0 by `restore()` with the memo invalidated; it is never computed per tick. The comparison with Save/Load's stored `block_state_hash` after restore is a diagnostic only.

### 10. Restore, drift and `post_restore`

**`restore(data, stored_version, ctx)` is two-phase** (ADR-0003 § 2, ADR-0006 § 2.2): decode and validate into a `LayoutScratch` (cold path, allocation allowed); on any failure call `ctx.fault(code, detail)` and write nothing; on success **copy the contents into the existing `LayoutState` object** (each column assigned from the scratch's own arrays, none aliasing `data`), then rebuild the static `LayoutDerived` (lookups, socket index, caps, `LayoutVersions.new_epoch()` (ADR-0014 § 6: `version_epoch` advances and is the authoritative restore signal for every rebuilding consumer; `default_state` calls it too), `mutation_count` 0, memo cleared, broadphase handles rebuilt by ADR-0014's call). A restore swaps whole tables only on success (TR-section-007).

**Validation, in order** (first failure wins, with a named detail): (1) `stored_version` in `supported_versions` and the key set equals the § 2 table, every column has the declared type and equal row counts per table (`LAYOUT_RESTORE_SCHEMA`), checked with the **typed-assignment idiom below** and the **byte-range checks** below; (2) every `template_id` resolves in the catalog (`LAYOUT_RESTORE_UNKNOWN_TEMPLATE`); (3) `rot_q` all 0 (`LAYOUT_ROT_UNSUPPORTED`); (4) the drift policy below (`LAYOUT_RESTORE_TEMPLATE_CHANGED`); (5) dense ids: `section_id[i] == i + 1`, `body_id[i] == i + 1`, `room_id_base[i + 1] == room_id_base[i] + rooms(template[i])`, counters equal the dense count + 1 and below the ceilings (`LAYOUT_RESTORE_COUNTER`); (6) references: every non-discarded instance sits on a `LIVE` body, every `LIVE` body has an in-range `origin_section_id` that belongs to it, socket rows are strictly ascending with `socket_index < template.socket_count` and a resolving `seam_id`, weld rows ascend and agree with their socket states (`ACTIVE` with two `WELDED`, `CUT_SEALED` with two `CAPPED`, `CUT_RAGGED` with two `CUT_OPEN`), `ACTIVE` welds form a forest whose components are exactly the `LIVE` bodies' section sets, and every `ACTIVE` weld's offsets satisfy `offset_b = offset_a + pos_a - pos_b` on the 24 grid (`LAYOUT_RESTORE_DANGLING`). An unsupported `stored_version` is a reported failure, never a silent default.

**The decode idiom (no engine script error can replace a named fault).** `data` is a `Dictionary` of `Variant`s, and in GDScript a typed assignment from a wrongly typed value (`var x: int = d["k"]` holding a `String`) or a missing key raises a script error that aborts the function, so `return ctx.complete()` would never be reached and no `ctx.fault` raised. Step (1) therefore (a) tests `has()` for **every** key, then (b) tests `typeof(v) == TYPE_PACKED_INT32_ARRAY` / `TYPE_PACKED_BYTE_ARRAY` / `TYPE_PACKED_INT64_ARRAY` / `TYPE_ARRAY` / `TYPE_DICTIONARY` / `TYPE_INT` (the ADR-0005 P20 typed-assignment idiom, never `as`) **before any typed assignment**, (c) validates each `template_id` element with `typeof(e) == TYPE_STRING` and fills the typed `Array[String]` column with `Array[String].assign(src)` (never plain assignment, since ADR-0006 does not assert that the codec preserves the type and forbids assigning an untyped `Array` to a typed one; a failure is `LAYOUT_RESTORE_SCHEMA` with detail `LAYOUT_RESTORE_BAD_TYPE`), and (d) copies packed columns into the scratch with `duplicate()` (never aliasing `data`). **Every early return path is `ctx.fault(...)` followed by `return ctx.complete()`.**

**Byte-range checks.** Element writes into narrow packed columns truncate silently (probed: a narrowing write of 300 into a byte column reads back as 44, so 259 would read back as 3), so a corrupted byte column could decode as a legal enum after a round trip. Byte columns are therefore range-checked **on the decoded value, before any narrowing write**: body `state` in 0..1; socket `state` in 1..3; weld `state` in 0..2; `discarded` in 0..1; `rot_q` = 0 (step 3); `socket_index < template.socket_count` (step 6). An `int32` column read from a packed source cannot exceed int32; an `int` that reaches a column from outside a packed array (a counter, `stored_version`) is range-checked against its ceiling before it is narrowed.

**Drift policy (TR-section-007).** The current template's three hashes are compared with the stored ones after `& 0xFFFFFFFF`. A **`structure_hash`** difference is `LAYOUT_RESTORE_TEMPLATE_CHANGED` for every instance (§ 4). A **`geometry_hash`** difference is `LAYOUT_RESTORE_TEMPLATE_CHANGED` when the instance carries history, meaning any non-`FREE` socket, any `welds[]` row naming it, or a body shared with another section, because a moved socket or wall edge under a welded or cut instance restores with broken abutment; on a never-welded `FREE` instance alone it is accepted with the diagnostic `geometry_changed_fresh`, and the rects are re-derived from the current template (they are always derived). A **`visual_hash`** difference alone is accepted with `visual_changed`. **`restore()` is byte-identical and never rewrites a stored hash** (ruling CT-11): the stored `geometry_hash`/`visual_hash`/`structure_hash` stay in `LayoutState` exactly as saved, and both diagnostics are re-raised on each load until the instance changes. The hashes are refreshed from the current template in **LM2 `attach_section`**, at the moment an instance first gets history (§ 7); the history-dependent geometry rule then compares against the template that history was made under, so a later weld cannot cause a false refusal. A hash collision cannot hide a structural change that also changes a tile set or key sequence in a way the validation of step (6) detects; a 2^-32 collision on `structure_hash` is accepted and recorded (§ Risks R6). Pre-release saves are disposable; after the first outside build the migration obligation of LY-6 e applies. Because restore is byte-identical, `snapshot(restore(x)) == x` holds after a visual-only or a fresh-geometry template edit, and the K-4 `state_hash` compare verifies `EXACT`/`MATCH` (a rewrite inside restore would have produced `NOT_REPRODUCED`, since ADR-0006's five REBASED causes are a closed set that does not include owner-side content drift; ADR-0016 § 9.3 reads "layout.state restore is byte-identical; content drift is diagnostic only").

**Query validity after restore.** Class 1 (static, valid when `restore()` returns): `room_at`, `V`, `exposed_tiles`, `room_surface`, `socket_plan`, `anchor_geometry`, static rect membership, `section_offset`, `body_of`, `section_at`. Class 2 (leaf-dependent, valid after `post_restore()`): `rects_in`/`solid_in` door leaves, `clip_to_footprints`, `leaf_would_overlap`, `door_passage_into`. Between the two they report the template default (every leaf closed) and raise `stale_leaf` if read; `LayoutDerived.leaves_valid` is the flag. The kernel guarantees every `restore()` completes before any `post_restore()` begins.

**`post_restore(ctx)`** (a) runs one `derive_doors` pass over the leaf cache using `opening_into` (an `area` above `area_max` found *here* is `LAYOUT_RESTORE_DANGLING` via `ctx.fault`, with no clamp, TR-section-035; the clamp with `area_clamped` belongs to the `layout.doors` hook; an unknown opening leaves the leaf closed); (b) rebuilds `opening_leaves` from the restored `welds[]` and `seam_openings_into`; (c) bumps `version` and `leaf_version` (never `static_version`) for each section whose derived leaf set differs from the all-closed default (the self-heal for an open leaf); rebuilding consumers do **not** rely on this bump, because the authoritative restore signal is `version_epoch` (`new_epoch()` in `restore` and `default_state`, ADR-0014 § 6); (d) runs cross-block **verification** through `section_counts_into`: every `graph_ref_base` resolves through `section_counts_into`, defined precisely: for each **non-discarded** instance, the rows in `[node_base, node_base + template.node_count)` and the matching edge range resolve to section `s` and are not tombstoned, and compartment `k` equals `comp_base + k`; for a **discarded** instance the same ranges resolve to tombstones of `s` (`out.tombstoned`). **Rows outside the template range (STUB rows a RAGGED cut appended, seam edges) are excluded**; `out.stub_count` is informational. Every `welds[]` `seam_id` resolves to a Graph seam record, and `graph.next_section_id == layout.next_section_id`. Failures use the restore codes. Verification order and the "installed `def_id` at a hardpoint resolves" diagnostic belong to ADR-0016's verifier list; this ADR supplies the checks Layout's side can run. After a restore the kernel delivers one `ALL` batch (ADR-0003 § 4); a `changed_region` ALL stamp per section after a restore is a non-guaranteed hint only (ADR-0014 § 6: the stamp may not be seen).

**Policy line (TR-derelict-008 Layout half).** A save restores from its **saved values**: templates are never mutated at runtime, a restore never re-templates an instance, and no Authoring content lint runs at restore.

### 11. Read views: records, tiers, order and frames

Reads are pure, side-effect free, deterministic and identical before a snapshot and after restoring it (idempotent), and have **no "perceived" variant** (Pillar 1, TR-section-039). They are exposed through the long-lived `LayoutViews` facade (ADR-0003 § 7); no live container is ever returned. Two tiers:

- **Tier H (hot; allocation-free; tagged `## @hot` for ADR-0010 `HOT-01`; scalar results or caller-owned `out`):** `room_at(section_id, x, y) -> int` (`0` = NONE; a sill px answers its `sill_room`, so no walkable point answers NONE), `floor_tile_at(...) -> int` (`-1` = NONE), `section_at(body_id, x, y) -> int` (`0` = NONE; FLOOR division), `section_offset_x/_y(section_id)`, `body_of(section_id)`, `version`, `static_version`, `leaf_version`, `changed_region_kind/_box_into`, `door_passage_into`, `leaf_would_overlap`, `leaf_obstruction_into`, `V(room_id)`, `exposed_tiles(room_id)`, `room_volume_tiles(room_id) -> int` and `room_exposed_tiles(room_id) -> int` (the `RoomSource` names, **-1 for an unknown room**; `V` and `exposed_tiles` answer 0 for unknown, the adapter maps that to -1), `body_is_live`, `body_state`, `section_contains`, `node_base`/`edge_base`/`comp_base` (below), and ADR-0014's reads, all additive to this section (ADR-0014 § 5 and § 7): `rects_in` and `section_rects_into`, `section_of_key` and `idx_of_key` on `OccluderSource`; `solid_in` on `SolidQuery`; `solid_at(body_id, x, y)` on `LayoutViews`; and the Layout-internal `footprint_overlaps` (`## @hot`) and `section_footprints_into` (untagged, `can_attach` validation only; not ports) (sentinels for an unknown or discarded section are `0`/false, never an error).
- **Tier C (cold; may allocate; returns a freshly built immutable record or a typed `Array[Record]`):** `room_surface`, `room_boundary`, `anchor_geometry`, `subject_anchors`, `socket_plan`, `weld_offset`, `detach_preview`, `room_adjacency`, `hull_segments_near`, `loose_slots`, `hardpoints_of`, `segment_at`, `plate_positions`, `sections_of`, `route_length_px`, `south_band`, `wall_face_rows`, `template_hash`.

**Fixed field names, orders and bounds.** Records live in `src/sim/layout/views/` as `RefCounted` classes whose fields are **plain typed `var`s**: GDScript has no friend or internal access, a get-only property cannot be written by another class, and a property getter is a function call per read in a `## @hot` path. Two kinds. (a) **`out` records** (`DoorPassage`, `LeafObstruction`, `AttachPreview`, `WeldPlan`, `SectionBases`, `OpeningRead`, `SeamOpenings`, `SectionCounts`, `BuildResult`, `AttachResult`, `DetachResult`) are caller-owned and filled by Layout: "written only by Layout, read by the caller"; the write restriction is enforced by an ADR-0010 lint row (`LAY-REC-01`, a new LAY-* row routed to the ADR-0010 owner), because the language cannot. (b) **Tier C returned records** (`AnchorGeometry`, `RoomSurface`, `RoomAdjacencyRow`, `SubjectAnchor`) are set in `_init(...)` (or a static `make`) and read by convention; a getter is used only where the cold path does not matter. Records: `AnchorGeometry {kind, host_face_id, base_px_polyline (PackedInt32Array pairs), sort_origin_y, section_id}`, `RoomSurface {floor_tiles (PackedInt32Array pairs, tile id ascending), wall_faces}` (the Failure & Trail contract, section-local), `DoorPassage` (D.5 table), `RoomAdjacencyRow {room_a, room_b_or_exterior, via (DOOR|HATCH), via_id, sill_room}`, `SubjectAnchor {local_key, kind_hint, anchor_px, room_id, console_role}`. Ordering contracts: `room_adjacency` by `(room_a, via_id)`; `subject_anchors` and the slot reads by key ascending; `plate_positions` by HULL_SEGMENT node id ascending; `room_surface` by tile id; `sections_of` by `section_id`; `detach_preview` by `section_id`. `wall_face_rows` returns at most 16 rows (the Failure & Trail bound `WALL_ROWS_MAX`). Cold reads that depend only on a template (`room_adjacency`, `south_band`, `plate_positions`) are precomputed once per template in the catalog row, so a read is a copy, not a recomputation.

**The four port records (ruling CT-02),** caller-owned in `src/sim/layout/views/`, following ADR-0014's `QueryBuffer` precedent: `SectionBases {node_base, edge_base, comp_base}`; `OpeningRead {area, area_max, door_state}`; `SeamOpenings {count, opening_ids: PackedInt32Array preallocated to the template maximum}`; `SectionCounts {nodes, edges, comps, stub_count, tombstoned: bool, next_section_id}` (meaning fixed by § 10 (d)). `InstantiateResult` does not exist (ruling CT-01: the port owner's name `SectionBases` wins).

**Reads owed to Core siblings (ruling CT-17).** Siblings have already built on these; each Tier H read is tagged `## @hot`.

| Read | Tier | Answers | For |
|---|---|---|---|
| `body_is_live(body_id) -> bool` | H (S after restore) | `state == LIVE` | ADR-0015 |
| `body_state(body_id) -> int` | H (S after restore) | `LIVE` / `DISSOLVED` / `UNKNOWN` | ADR-0015 BM3 (`rebase_on_attach` requires the absorbed body `DISSOLVED` after LM2) |
| `section_contains(section_id, local_x, local_y) -> bool` | H | **section-local** base-px point (the frame of every geometry read in this ADR, never the body frame; the caller subtracts the section offset, as ADR-0015 BM4 does with the pre-cut `offset + shift`) inside the section's own half-open tile cell set (its floor area, never a bounding box); false for unknown or discarded | ADR-0015 |
| `node_base(section_id) -> int`, `edge_base(section_id) -> int`, `comp_base(section_id) -> int` | H, Class 1 (valid after `restore`) | `0` = NONE | ADR-0017 E1 |
| `SectionTemplate.emitter_placement_count(template_id)` and indexed scalar getters `placement_def_id`, `placement_anchor_x`, `placement_anchor_y`, `placement_node_ordinal`, `placement_switch_on`, `placement_subkind` | frozen catalog (cold) | the template's emitter placements, loaded at stage 1 with the template schema and validated against ItemCatalog `emits` | ADR-0017 E1 |
| `RectFlags` (`src/sim/layout/rect_flags.gd`: bit values and `ALL` mask) | constants | § 5 | ADR-0014, ADR-0034 |
| `SlotFeed` (`slot_count`, `slot_footprint`, `slot_extent`) | H | the reserved socket variant slots (PLATE, JAMB) and their current extents, section-local | ADR-0014 |
| `room_volume_tiles`, `room_exposed_tiles` | H | CT-02 | ADR-0011 `RoomSource` |

**CONSOLE anchors (TR-derelict-034, re-tagged 2026-10-29).** `subject_anchors(section_id)` returns `SubjectAnchor` rows with `kind_hint` in `{DOOR_FACE, HARDPOINT, SOCKET, SLOT, CONSOLE, INSTRUMENT}`; `console_role` is `POD` or `DOCK` and is **present iff `kind_hint` is `CONSOLE`**. The structural lint `CONSOLE_ROLE_UNSET` (ERROR, boot-tier invariant) rejects a CONSOLE anchor without a role and a role on any other kind. The counts "each dock room holds exactly one dock status glow on its docked-podcraft prop" and "the world holds exactly four CONSOLE anchors" are **Derelict Authoring content lints** (`POD_CONSOLE_MISSING`, `DOCK_CONSOLE_MISSING`, the world-level count) evaluated over this read; Layout asserts no placement and counts nothing at runtime.

**Frame rule (TR-section-039, LY-3 c).** Every geometry read answers section-local base px; the caller composes `section_offset`. Reads that answer for one body answer in the body frame. A read never mixes frames. The one section-keyed read that answers in the body frame is `section_rects_into` (§ 5), documented as such.

### 12. Capacity, size and world-build budgets

- **Ceilings** are the registered constants and GDD knobs named in § 9, loaded from the `LayoutTuning` `ScalarTable` in the default-config data (ADR-0007 § 1.5); the module source contains no literal for any of them, and a start-up test runs the module at each knob's published minimum and maximum. Published ranges are guidance: an out-of-range value appears in the ADVISORY range report and is never clamped or asserted at boot; only schema bounds (`CONTENT_FIELD_RANGE`) and named structural invariants (`CONTENT_INVARIANT`: `area_max > 0`, `area_max <= width_px` (§ 4 (c): a numeric bound, not a unit conversion), `GRID_24`, `RECT_STRIDE_OVERFLOW`, `RECT_FLAGS_ILLEGAL`, `CONSOLE_ROLE_UNSET`, rotation 0) refuse a load (TR-section-033 Layout side).
- **Block size.** `bytes ~ 4 * (14 I + 10 K + 4 S + 3 B + 3)` over instance, weld, socket and body rows (D.21 with the 14th column), plus strings; `LAYOUT_BLOCK_SOFT_BYTES` raises the one-shot diagnostic `layout_block_soft_bytes`; `MAX_BLOCK_BYTES` is Save/Load's hard ceiling. The long-campaign shape (24 live and 200 tombstoned sections) is tens of KB.
- **World ceilings (TR-webmap-036 Layout half).** Layout supplies `V(room)` and the rect and door counts; the tile ceiling `TILES_WORLD_MVP`, the S0 to S4 loaded-set tile counts and `INTERACT_FIXTURE_BUDGET` are lints and tests owned by Derelict Authoring and Web Map. A test builds the S4 loaded set through `build_section` (not a hand sum) and asserts the real `V` totals and rect counts against them. `BUILD_TILES_PER_FRAME` (GDD Tuning Knobs) is the PROJECTOR's chunk size (ADR-0034); Layout's contribution is the per-section tile count.

### 13. Scene boundary and the PROJECTOR's inputs

The Layout data module references no `Node`, `Node2D`, `TileMapLayer`, `SceneTree`, `Resource` subclass or engine physics type (AC-120); a headless run leaves the orphan-node count unchanged. The PROJECTOR (`src/presentation/projector/`, ADR-0034) is a separate subscriber that never writes a block and never creates `Light2D` or `LightOccluder2D`, builds door, prop, machinery and bead/plate/scar sprites from the rect table and `welds[]`/socket state, and redraws on `SEC`, `BODY`, `SEAM` and `LEAF` keys. This ADR guarantees the three inputs of the sprite `entity_id` are stable: `section_id` (never reused, below 2^20), a closed `kind_ordinal` vocabulary (door leaf, jamb, prop, machinery, bead, plate, scar; ordinal values are assigned append-only by ADR-0034 § 4), and `template_local_idx` (the rect `idx` or the template placement index, below the rect stride and so below 2^16). The packing function and the namespace byte `LAYOUT_ID_NS` belong to ADR-0034 and the Pipeline (OQ-33); Layout exports no packing helper and tiles carry no `entity_id`.

### Key Interfaces

```gdscript
# src/sim/layout/layout_ids.gd  (constants only; structural limits, not tuning)
class_name LayoutIds extends RefCounted
const NONE: int = 0
const SECTION_ID_MAX: int = (1 << 20) - 1        # rect_key < 2^32 (D.9)
const ID31_MAX: int = 0x7FFFFFFF                 # int32 columns: body_id, room_id

# src/sim/layout/socket_ref.gd
class_name SocketRef extends RefCounted          # static helpers only
const SOCKET_INDEX_BITS: int = 4                 # private to the packing; declared here, not in LayoutIds
static func make(section_id: int, socket_index: int) -> int: return (section_id << SOCKET_INDEX_BITS) | socket_index
static func section_of(ref: int) -> int: return ref >> SOCKET_INDEX_BITS
static func index_of(ref: int) -> int: return ref & ((1 << SOCKET_INDEX_BITS) - 1)   # probed: make(5,3)=83, section_of(83)=5, index_of(83)=3

# src/sim/layout/layout_module.gd  (the owner; setup is called by the composition root)
class_name LayoutModule extends RefCounted
func setup(facet: ModuleFacet, templates: SectionTemplates, tuning: LayoutTuning) -> void      # constructs LayoutViews and the adapters; no Graph argument
func room_source() -> RoomSource                                  # the LayoutRoomSource, passed by the composition root to GraphModule.setup
func solid_query() -> SolidQuery                                  # the LayoutSolidQuery (ADR-0014 § 5), passed by the composition root to Body, Horror and the PROJECTOR
func occluder_source() -> OccluderSource                          # the LayoutOccluderSource (ADR-0014 § 5), passed to the Emitter, the Lighting builder and the PROJECTOR
func bind_graph(port: LayoutGraphPort) -> void                    # after GraphModule.setup, before the registry freeze; a port call earlier is an asserted defect

# Class A checks (pure; &"" = OK) and mutators (status int: OK 0, REFUSED 1, -fault_code)
func can_build(views: SimViews, template_id: String, kind: int) -> StringName   # reads the pending section id (counters.next_section_id) itself and passes it, with template_id and kind, to the Graph port's check_instantiate_section, so the Graph's reasons pass through
func build_section(scope: WriteScope, template_id: String, kind: int, out: BuildResult) -> int
func can_attach(views: SimViews, socket_ref_a: int, socket_ref_b: int) -> StringName
func preview_attach(socket_ref_a: int, socket_ref_b: int, out: AttachPreview) -> void   # reason, hatch_one_sided, openings_count
func attach_section(scope: WriteScope, socket_ref_a: int, socket_ref_b: int, seam_id: int, welded_at_T: int, out: AttachResult) -> int
func can_detach(views: SimViews, seam_id: int) -> StringName
func detach_section(scope: WriteScope, seam_id: int, boundary_mode: int, out: DetachResult) -> int
func can_discard(views: SimViews, section_id: int) -> StringName
func discard_section_geometry(scope: WriteScope, section_id: int) -> int

# Result records (caller-owned, reused; src/sim/layout/views/)
# Each record is a three-or-more-line file (no trailing colon after the superclass), every field has an explicit default (ADR-0010 sets
# ENUM/UNTYPED warnings to errors), and each record resets itself through reset() (not an _init re-run) before every Layout fill.
# Fields are plain typed vars, written only by Layout (lint LAY-REC-01), read by the caller (§ 11). Array fields are filled in place by
# index with a separate count; an implementation never assigns another array to a result field (§ 7).
class_name BuildResult extends RefCounted
var section_id: int = 0
var body_id: int = 0
func reset() -> void:
	section_id = 0
	body_id = 0

class_name AttachResult extends RefCounted
var surviving_body_id: int = 0
var absorbed_body_id: int = 0
var frame_shift_x: int = 0
var frame_shift_y: int = 0
var moved_count: int = 0
var moved_sections: PackedInt32Array = PackedInt32Array()   # sized once at setup; valid [0, moved_count); valid until the next Layout call
func reset() -> void:   # sets counts and scalars to 0; never reassigns moved_sections
	surviving_body_id = 0
	absorbed_body_id = 0
	frame_shift_x = 0
	frame_shift_y = 0
	moved_count = 0

class_name DetachResult extends RefCounted
var new_body_id: int = 0
var frame_shift_x: int = 0
var frame_shift_y: int = 0
var moved_count: int = 0
var moved_sections: PackedInt32Array = PackedInt32Array()
var rect_count: int = 0
var rects: PackedInt32Array = PackedInt32Array()            # 4 ints per rect, half-open, pre-cut body frame
func reset() -> void:
	new_body_id = 0
	frame_shift_x = 0
	frame_shift_y = 0
	moved_count = 0
	rect_count = 0

# Port records (src/sim/layout/views/; same conventions): SectionBases {node_base, edge_base, comp_base}; OpeningRead {area, area_max, door_state};
# SeamOpenings {count, opening_ids}; SectionCounts {nodes, edges, comps, stub_count, tombstoned, next_section_id}

# Adapters (each delegates to LayoutViews with the port's method names; src/sim/layout/)
# class_name LayoutRoomSource extends RoomSource: room_volume_tiles(room_id) -> int (wraps V), room_exposed_tiles(room_id) -> int (wraps exposed_tiles); -1 = unknown room

# Hook (registered once; HookDescriptor divisor 1, offset 0, after [])
func _hook_doors(views: SimViews, scope: WriteScope, ctx: HookContext) -> CompletionToken

# Hot, allocation-free reads (representative; full list in § 11)
func room_at(section_id: int, x: int, y: int) -> int
func section_at(body_id: int, x: int, y: int) -> int
func body_of(section_id: int) -> int
func door_passage_into(section_id: int, via_id: int, out: DoorPassage) -> bool
func leaf_would_overlap(opening_edge_id: int, area: int, boxes: PackedInt32Array) -> bool

# src/sim/layout/layout_faults.gd  (domain 4; pre-packed, append-only)
class_name LayoutFaults extends RefCounted
const DOMAIN_ID: int = 4
const LAYOUT_RESTORE_SCHEMA: int = (DOMAIN_ID << 16) | 1
const LAYOUT_RESTORE_UNKNOWN_TEMPLATE: int = (DOMAIN_ID << 16) | 2
const LAYOUT_RESTORE_TEMPLATE_CHANGED: int = (DOMAIN_ID << 16) | 3
const LAYOUT_ROT_UNSUPPORTED: int = (DOMAIN_ID << 16) | 4
const LAYOUT_RESTORE_COUNTER: int = (DOMAIN_ID << 16) | 5
const LAYOUT_RESTORE_DANGLING: int = (DOMAIN_ID << 16) | 6
const LAYOUT_OFFSET_OFF_GRID: int = (DOMAIN_ID << 16) | 7
const LAYOUT_COMPONENT_COUNT: int = (DOMAIN_ID << 16) | 8
const LAYOUT_APPLY_PRECONDITION: int = (DOMAIN_ID << 16) | 9
```

(Each `class_name` is its own file. Signatures are the contract; bodies are the Layout epic's. In an implementation, `@abstract` port declarations may keep unused parameters, but the concrete override uses `_`-prefixed names for unused ones, e.g. `_views`, `_out`.)

### Implementation Guidelines

- **Must** register exactly one block (`layout.state`), one hook (`layout.doors`), one fault domain (4), one key space `{SEC 1, BODY 2, SEAM 3, LEAF 4}` and one diagnostics channel, only through `ModuleFacet.registrar` in `BOOTING`; **must never** register a second Layout block or a `layout.templates` block.
- **Must** store instances by reference (`template_id` plus drift hashes plus runtime fields); **must never** put geometry, a rect, a leaf, a tile list, a `kind`, a `visited` bit, a loose-item row, a door `area` or `door_state`, a float, a `layout_hash` or `mutation_count` in `layout.state`.
- **Must** snapshot as the § 2 closed key set of parallel columns, ascending ids, duplicated packed arrays; **must never** use an int-keyed `Dictionary`, `PackedStringArray`, `Vector2i`, `Rect2i`, `StringName` or `Resource` in the payload, and **must never** put a key in the payload that is not on the regenerated allow-list.
- **Must** keep `section_id == row + 1` and `body_id == row + 1`: an id is consumed only by the call that appends its row; **must never** reuse, renumber, skip or reserve an id, and **must never** let a counter wrap (the `can_*` check fails first).
- **Must** pass the Graph the section id and store the returned `SectionBases {node_base, edge_base, comp_base}`; **must never** allocate a node, edge, compartment or seam id in Layout, and **must never** read the Graph's `next_*` counters except in the post-restore verification.
- **Must** compare stored hashes as `value & 0xFFFFFFFF` (`IntMath.low32`); **must never** compare a `PackedInt32Array`-read hash unmasked.
- **Must** compute the three template hashes once at catalog freeze with `Hash64.of_value` over the § 4 sub-payloads; **must never** write owner byte encoders for them (ADR-0005 `owner_canonical_byte_writer`) and **must never** store them in a template file.
- **Must** write a socket's state only inside LM2 and LM3 and choose the weld survivor by lower `body_id`; **must never** add a `set_socket_state` or `install_cap_plate` mutator, and **must never** let the result of a weld depend on argument order.
- **Must** begin every mutator with `scope.open_for(&"layout.state")`, check every Class A status, mark keys explicitly (`DirtyKey.make`) after the writes; **must never** rely on a packed-array property setter for dirty tracking (4.7), and **must never** call a Graph mutator from a `can_*` check.
- **Must** make LM2 seed each new LEAF closed with its last-projected cache and **must never** read the Graph's `area` inside LM2; **must** make `layout.doors` bump a section's `leaf_version` at most once per tick and **never** `static_version`.
- **Must** treat every door `area` write as a P2 write by command; **must never** call `set_opening_area` from a hook, and **must never** store, mirror or write `area` or `door_state` in Layout.
- **Must** use `IntMath.floor_div` for every px-to-cell, px-to-tile and leaf computation and avoid `/`, `%`, `/=`, `%=` in `src/sim/layout/`; **must never** use `float`, `Vector2`, `Rect2` or any engine geometry type in the module.
- **Must** take foot boxes as caller arguments; **must never** read `body.state` from Layout (acyclic restore, `restore_after: []`).
- **Must** make `restore` validate into scratch and copy into the existing `LayoutState`; **must never** replace the object a facade wraps, alias `data`, or return a partial restore; **must** reject `rot_q != 0` and any `structure_hash` mismatch.
- **Must** tag the hot reads of § 11 `## @hot` (the Tier H list is the Layout tag list for ADR-0010 § 6.6 `HOT-01`); **must never** allocate in `room_at`, `section_at`, `body_of`, `door_passage_into`, `leaf_would_overlap`, `room_volume_tiles`, `room_exposed_tiles` or the hook. Allocation-freedom is enforced by `HOT-01` (a lexical scan of the `## @hot` bodies, which bans `.resize(`, `.duplicate(`, literals and `PackedXArray(`) plus `CNT-01` for leaked Objects only; `Performance.OBJECT_COUNT` counts Objects only and `MEMORY_STATIC` is blind to transient allocation, so neither proves that no packed array, `Array`, `Dictionary` or string was allocated.
- **Must** fill result and traversal arrays in place (size once, write by index, separate `count`); **must never** assign another array to a result field, use `clear()`/`push_back()` on a reused result or traversal array, or retain a result across Layout calls without `duplicate()` (§ 7; packed arrays alias on assignment).
- **Must** write every record field as a plain typed `var` with an explicit default and reset a reused record through `reset()`; **must never** write a record's fields from outside Layout (`LAY-REC-01`); **must never** use a get-only property on an `out` record.
- **Must** use `_`-prefixed names for unused parameters in concrete implementations of `@abstract` ports and checks (`can_build/can_attach/can_detach/can_discard(_views, ...)` when they read Layout's own tables; `_out`), and name a local that mirrors a view method (`version`, `static_version`, `leaf_version`, `V`, `exposed_tiles`, `section_at`, `rects_in`) differently (`ver`, `leaf_ver`, `new_version`), because a same-named local or parameter shadows the method. `V` is the GDD name and stays; the exact adapter-facing method is `room_volume_tiles`.
- **Must** make `restore` use `has()` then `typeof()` before every typed assignment, `Array[String].assign` for `template_id`, `duplicate()` for packed columns, byte-range checks before narrowing writes, and end every path with `return ctx.complete()` after `ctx.fault(...)`; **must never** rewrite a stored hash in `restore` (§ 10).
- **Must** call `bind_graph` before the registry freeze and make every port call after it; **must** implement `RoomSource` through `LayoutRoomSource`; **must never** extend more than one port from one class.
- **Must** keep `src/sim/layout/` scene-tree free and `src/data/` free of `src/sim/` imports; the PROJECTOR script **must never** live under `src/sim/layout/`.
- **Must** enforce capacity ceilings in `can_build`/`can_detach` before any id is consumed or the Graph is called.
- **Must** load every Layout knob from `LayoutTuning`; **must never** put a literal for a knob in module source.

## Alternatives Considered

### Alternative 1: Full geometry in the block
- **Description**: Each instance stores its rooms, rect table, sockets and doors in `layout.state`.
- **Pros**: A save is self-contained; a template edit cannot affect an old save; no catalog dependency at restore.
- **Cons**: Rectangles times sections times snapshot cadence bloat the block (hundreds of rects per ship against tens of rows); authored data is duplicated into every save; the Vertical Slice "seed plus deltas" model is defeated; an exporter bug is persisted forever; the hash covers authored data that is already hashed at catalog freeze.
- **Rejection Reason**: Instances by reference with drift hashes keeps the block at tens of KB, and the drift policy already turns a template edit into a loud restore failure where it matters.

### Alternative 2: A second block `layout.templates`
- **Description**: Templates become a persisted block of their own so the save carries what it was built from.
- **Pros**: Restore never consults the catalog.
- **Cons**: A second restore root and a second hash for data that never changes at runtime; it contradicts ADR-0007's "catalogs are not state blocks"; it duplicates the content pipeline's integrity checks.
- **Rejection Reason**: Static data belongs to the catalog; the block stores references.

### Alternative 3: Array-of-`Dictionary` or int-keyed `Dictionary` tables in the snapshot
- **Description**: One row per instance as a `Dictionary`, or a table keyed by id.
- **Pros**: Readable dumps; trivial lookup by id.
- **Cons**: JSON turns int keys into strings and numbers into floats (LY-26 a); a per-row `Dictionary` allocates and sorts per row in the encoder; ADR-0005 § 3.6 asks blocks with more than about 250 scalar fields to use packed arrays; two equal worlds reached by different histories could order differently.
- **Rejection Reason**: Dense parallel columns in ascending id order are the shape ADR-0005 and ADR-0006 are built for, and `id - 1` row addressing removes the lookup map.

### Alternative 4: The Graph allocates `section_id` (and Layout asks for it)
- **Description**: `build_section` calls M1 first and takes the section id from the return.
- **Pros**: One fewer counter in Layout.
- **Cons**: Layout cannot name its own row, dirty key or body before the Graph runs; the `rect_key` bound (2^20) becomes the Graph's responsibility; Layout's `can_build` could not test the section-id ceiling before calling the Graph; two persisted counters would both claim to be the section counter.
- **Rejection Reason**: The GDD and the Graph (LS-15) already treat the section id as a caller input to M1. Layout allocates; the Graph's counter follows.

### Alternative 5: Layout also allocates the Graph's node, edge and compartment ids
- **Description**: Layout hands the Graph explicit id ranges.
- **Pros**: `graph_ref_base` becomes Layout's alone.
- **Cons**: Layout would have to know the Graph's per-template node and edge counts and its capacity rules; two owners could both write the Graph's counters.
- **Rejection Reason**: One writer per fact; the Graph returns the bases it allocated.

### Alternative 6: A nested ledger (one entry per template with a `room_keys` list), or two ledger spaces
- **Description**: `pairs[]` holds templates, each with a nested list of room keys; or `section_templates` and `section_rooms` are separate spaces.
- **Pros**: Mirrors the data's shape.
- **Cons**: A nested form needs a second `verify` and a second append-only gate (the generic one compares pairs); a per-template list makes "remove one room" a list edit instead of a per-pair `dormant` flip; two spaces mean two files and a cross-file foreign-key check.
- **Rejection Reason**: One flat space with compound keys reuses ADR-0007 unchanged and keeps the CI diff a list of added pairs.

### Alternative 7: Owner-written TLV for the template hashes (ADR-LAY-06 as the GDD wrote it)
- **Description**: `canonical_geometry_bytes` as `[u32 tag][u32 length][payload]` in a documented field order.
- **Pros**: Independent of the kernel encoder.
- **Cons**: ADR-0005 § 3.1 forbids owner byte writers and ships golden vectors for the kernel's TLV only; a second encoder is a second injectivity argument.
- **Rejection Reason**: ADR-LAY-06 becomes the payload shape; the byte form is ADR-0005's.

### Alternative 8: Two hashes only, with sets compared at restore
- **Description**: Keep the GDD's `geometry_hash` and `visual_hash`; classify "structural" by comparing key sets.
- **Pros**: The row stays 13 fields.
- **Cons**: There is nothing persisted to compare key sets against; a pure reorder passes every count and set and silently re-points ids to the wrong rooms and sockets.
- **Rejection Reason**: § 4. The third hash costs four bytes a row and makes the check exact.

### Alternative 9: Weld survivor by argument order, or by larger body
- **Description**: The first argument's body survives; or the body with more sections.
- **Pros**: Simple call site.
- **Cons**: Caller mixups can dissolve the home ship and make two equal worlds hash differently; the larger-body rule changes with every weld.
- **Rejection Reason**: The lower `body_id` is persisted, immutable and total.

### Alternative 10: Derive leaves lazily on read (the GDD fallback)
- **Description**: No hook; a leaf is derived when a query asks, stamped by the Graph's write state.
- **Pros**: No P3 cost.
- **Cons**: A read mutates a derived cache; a refresh in P2 can precede a later same-P2 change to `area`; every consumer pays the check.
- **Rejection Reason**: The hook is ordered (`body.integrate after layout.doors`) and bounded; the fallback stays recorded but unused.

### Alternative 11: Persist all sockets densely (including `FREE`)
- **Description**: A row per declared socket per instance.
- **Pros**: Fixed-size table; no sorted insert.
- **Cons**: Most rows are `FREE` forever in MVP; a template that adds a socket changes the table shape (a structural change even for an unwelded instance).
- **Rejection Reason**: Sparse non-`FREE` rows are smaller and a template that adds a socket does not touch old saves except through the structure hash, which is already exact.

## Consequences

### Positive
- One block, one owner, one closed payload shape that the type walk, the encoder and the allow-list test all agree on.
- Row addressing by `id - 1` removes id maps from every hot read and makes the dense-id check a restore invariant.
- A weld or cut gives the same bytes for the same world regardless of argument order (for `layout.state` by canonicalised `welds[]`, and for `graph.state` by the survivor-first `socket_plan` / `JoinPlan`), so whole-world `state_hash` equality and replay hold.
- The structural check is exact (a reorder is refused), where the GDD's wording would have corrupted old saves.
- Capacity ceilings fail in validate with a named reason, never as a `FAULTED` apply, and no counter can wrap.
- The Layout/Graph id boundary is a table; each side can be tested with the other stubbed through `LayoutGraphPort`.
- ADR-0014, ADR-0015, ADR-0016 and ADR-0034 build on records and rules that no longer move.

### Negative
- The instance row grows to 14 columns, deviating from the TR text; the GDD, TR-section-004, D.21 and the AC-130 allow-list are updated.
- Dense ids forbid reserving an id; a future procgen that wants sparse or hashed ids must supersede this ADR.
- `layout.doors` marks a key and therefore advances `touch_serial` on door-change ticks, invalidating the block hash cache on those ticks.
- Hash refresh moves to LM2 (a weld rewrites an instance's stored hashes the first time it gets history), so `restore` stays byte-identical but a weld changes bytes it did not otherwise touch (deterministic, argument-order independent).
- Record fields are plain `var`s because the language has no friend access; the write restriction is a lint (`LAY-REC-01`), not a compile-time guarantee.
- A new key family (LEAF) is added to a key space the GDD listed as closed.
- A template reorder now refuses every old save of that template (correctly, but harshly); authors must append rooms, not reorder them, once the first build is outside the project.
- Layout depends on a port whose Graph-side implementation does not exist yet, and ADR-0013 and ADR-0011 are mutually dependent (co-accepted; neither may be Accepted alone).

## Risks
- **R1: dense-id invariant broken by a future feature.** A mutator that consumes an id without a row breaks `id == row + 1`. Mitigation: restore asserts it; a unit test builds, discards and restores and checks it; a lint `LAY-ID-01` forbids writes to the counters outside LM1 and LM3.
- **R2: the third hash is a deviation.** A reviewer may prefer the GDD's two. Mitigation: the reasoning is in § 4 with the failing scenario; reverting is a one-column change before any save exists.
- **R3: hook-marked keys and hash cache churn.** If animating doors become continuous (a stepped slide, OQ-14 c), a key per tick advances `touch_serial` every tick. Mitigation: binary doors in MVP; the cost is bounded by block size; revisit with ADR-0005 § 5.4 if continuous doors are introduced.
- **R4: GDScript cost of the hook and of attach/detach.** All figures are pending-spike (V2, V3). Mitigation: the fallback ladder of ADR-0010 § 9; hook early-out on a Graph change counter; attach/detach are rare P2 events.
- **R5: element writes without a setter (4.7).** A missed dirty mark is a stale display. Mitigation: explicit marks inside mutators, the ADR-0004 audit (`quiet_writes_allowed` false), and a per-mutator key fixture.
- **R6: hash collisions.** `low32` of a 64-bit hash collides at 2^-32 per pair. Mitigation: hashes are diagnostic gates, restore step (6) re-validates structure against the current template independently, and a missed geometry change on an instance with history would at worst restore with a wrong abutment that the weld-offset check of step (6) catches.
- **R7: the Graph port is unwritten.** If ADR-0011 returns non-contiguous ranges, an allocating `opening` read, or counts that include stub rows in `section_counts_into`, §§ 3, 8 and 10 change. Mitigation: § Related lists the exact assumptions; ADR-0011 and this ADR are reviewed together.
- **R8: attach can exceed the per-body rect ceiling.** `can_attach` has no `BODY_FULL` reason. Mitigation: the MVP world is a few hundred rects against `MAX_FOOTPRINT_RECTS_PER_BODY`; a world lint bounds the sum; OQ-1.
- **R9: `Array[String]` column in the type walk.** Mitigation: V5; the fallback is a ledger-integer column with the string recovered from the catalog, which is a schema-version bump.
- **R10: template edits after the first outside build.** Authors can break old saves. Mitigation: the ledger flips to `SHIPPED` at the same moment; CI append-only gates; the drift policy is loud, not silent.

## GDD Requirements Addressed

TR text is `docs/architecture/tr-registry.yaml`. Section numbers are this ADR's.

### Decided here (20)

| TR-ID | Requirement | How this ADR addresses it |
|---|---|---|
| TR-section-004 | `layout.state` stores instances by reference (bodies, 13-int rows, sparse non-FREE sockets, `welds[]`, counters); derived caches never persisted | § 1, § 2: `LayoutState` vs `LayoutDerived`; closed key set; 14 columns (13 plus `structure_hash`), the one deviation |
| TR-section-005 | Ids allocated from persisted counters in template order, never reused or renumbered; tombstones; `graph_ref_base` persisted | § 2 tombstones, § 3 allocation table and rules, dense `id == row + 1` |
| TR-section-010 | Half-open integer rects with a flags byte on the 24-px grid; floor division; translation only | § 5: record, flags, frames, `GRID_24` as a boot invariant (§ 4) |
| TR-section-012 | `layout.doors` (divisor 1, `after: []`) in P3 re-derives leaf rects from `opening(edge_id)`; one bump per changed section; `body.integrate` after it | § 8: hook algorithm, port, bump rule |
| TR-section-013 | Layout never stores, mirrors or writes `area`/`door_state`; every `area` write in P2; leaf sprite equals the leaf rect | § 1 never-holds, § 8 correctness condition and the LEAF key |
| TR-section-014 | `leaf_would_overlap`/`leaf_obstruction` evaluate every leaf of the opening; foot boxes are arguments | § 8 pure leaf queries |
| TR-section-018 | Sockets FREE -> WELDED -> CAPPED/CUT_OPEN, terminal; changed only inside LM2/LM3 | § 6 states and the write lint |
| TR-section-019 | `can_attach` fixed reason order; `GEOM_OVERLAP` over static footprints and tile cells; memoised; <= 2 ms uncached | § 6 chain, memo key, pending-spike target |
| TR-section-020 | Exactly four Class A mutators with pure checks; LM2/LM3 inside Assembly's atomic apply after the Graph | § 9 table, Class A conformance, naming divergence recorded |
| TR-section-021 | `offset_b = offset_a + pos_a - pos_b` on the lower-id survivor; canonical `welds[]`; forest; detach allocates `body_id` and re-bases to the min-id section | § 7 |
| TR-section-027 | Layout owns room identity, tiles, `V`, `exposed_tiles`, unchanged while a room exists; Graph caches them, `graph.state restore_after: [layout.state]` | § 3 room ids and mapping, § 11 `V`/`exposed_tiles` reads, § 1 `restore_after: []`, § 10 post-restore |
| TR-section-028 | No loose-item table, no `visited` bit, no pose/velocity, no atmosphere; item positions are `(section_id, section-local px)` | § 1 never-holds; § 11 frame rule |
| TR-section-030 | Ceilings enforced by `can_build`/`can_detach`; block-size diagnostic at `LAYOUT_BLOCK_SOFT_BYTES`; under `MAX_BLOCK_BYTES` | § 9 capacity, § 12 |
| TR-section-035 | Unknown `opening(edge_id)` leaves the leaf closed, no `area` invented, no bump, `opening_unknown` once per edge; `area > area_max` clamped with a diagnostic; same fault in `post_restore` is `LAYOUT_RESTORE_DANGLING` | § 8 steps 1 and 2, § 10 post_restore (a) |
| TR-section-037 | `next_body_id` and `next_room_id` capped at 2^31-1; `CAPACITY` before an id is consumed or the Graph is called; no wrap | § 3 when ids are consumed, § 9 capacity |
| TR-section-039 | LY-28 read views: fixed field names, deterministic order, `wall_face_rows` <= 16, idempotent and identical after restore | § 11 |
| TR-section-042 | `layout.doors` at `MAX_DOORS_LOADED` <= 0.2 ms; <= 1 `leaf_version` bump per section per tick; never `static_version` | § 8 hook and cost (V2 pending) |
| TR-assembly-025 | Sockets terminal; one weld-then-cut cycle; changed only inside LM2/LM3 | § 6 |
| TR-horror-032 | Door passability via `door_passage`: passable iff `open_extent_px` >= foot extent across the run | § 8 `DoorPassage.passable`, `door_passage_into` |
| TR-derelict-034 | CONSOLE anchor records with `console_role` POD or DOCK; one dock glow per dock room; four CONSOLE anchors | § 11: record schema and `CONSOLE_ROLE_UNSET` here; the counts are Derelict Authoring content lints over this read |

### Shared: this ADR decides its half (12)

| TR-ID | Requirement | This ADR's half | Other half |
|---|---|---|---|
| TR-section-006 | Dense parallel packed arrays, ascending id, no int-keyed dictionary, 64-bit values as halves or strings, deep copies, restore from scratch | § 2 layout, native int64 columns, no forbidden types | ADR-0006 § 1.2, § 1.3, § 2.2 (allowed types, deep copies) |
| TR-section-007 | Restore swaps tables from scratch only on success; rejects unsupported version, unknown `template_id`, structural/geometry change; `rot_q != 0` is `LAYOUT_ROT_UNSUPPORTED` | § 10 validation order, drift policy, fault codes | ADR-0006 § 2.2 (two-phase), ADR-0016 (order across blocks) |
| TR-section-034 | A newly attached hatch LEAF is seeded closed; no `stale_leaf`; no `leaf_version` bump unless the P3 read differs | § 8 seeding, § 7 LM2 | ADR-0021 (the weld transaction order that makes P2 then P3) |
| TR-section-044 | `template_id` and `room_key` pinned by an append-only ledger; nesting is Layout's | § 4: one flat `pairs[]`, compound keys, `SectionLedgerCheck` | ADR-0007 § 3.2 (ledger mechanism, gate) |
| TR-derelict-007 | One authored value changes the hash its edit class touches (Derelict AC-DA-97, 2026-10-29) | § 3 and § 4: a geometry edit changes `geometry_hash`, a kit-piece or label edit changes `visual_hash`, a key reorder, rename, add or remove changes `structure_hash` (restore refuses `LAYOUT_RESTORE_TEMPLATE_CHANGED`); a Graph initial-value edit changes only `state_hash` | ADR-0005 (hash and golden vectors), Derelict Authoring DA-35 (the `default_state` diff) |
| TR-derelict-008 | Templates never mutated at runtime; restore from saved values; no Authoring lint at restore | § 10 policy line | ADR-0016 (restore ordering) |
| TR-derelict-033 | Socket offsets on the 24-px lattice (a 12-px shift fails); weld-lane junction nodes carry `salvageable: false` | § 4 `GRID_24` as a boot invariant | ADR-0021 / ADR-0011 (`salvageable` flag, `can_join` PORT_ABSENT) |
| TR-derelict-040 | Every reachable loaded set holds at most `MAX_BODIES` bodies; welded home plus B counts as one | § 9: `MAX_BODIES` over `LIVE` bodies, `CAPACITY` in `can_build`/`can_detach` | ADR-0026 (loaded-set enumeration) |
| TR-webmap-036 | Loaded set stays within tile and subject ceilings, asserted against a real Layout build at S4 | § 12: `V` and rect supply, S4 test via `build_section` | ADR-0026 (loaded set), Web Map/Authoring (ceilings as lints) |
| TR-boarding-035 | A loose item stores a section id and a section-local position, so it moves with its section on weld and detach | § 1 (no item table), § 7 (weld and cut change only `offset` and `body_id`), § 11 `floor_tile_at`/`loose_slots` | ADR-0021, Interaction (the item holder) |
| TR-interact-035 | An item moves losslessly between holders; loose positions are section-local and unchanged by weld or detach | § 7, § 11 frame rule | ADR-0027 (Suit item moves), Interaction `place_loose` |
| TR-horror-031 | Horror reads Layout only through published queries with per-run budgets and treats NONE/UNKNOWN as neither unlit nor open | § 11: every read pure, idempotent, explicit NONE sentinels; template-only reads precomputed | ADR-0014 (`OccluderSource`, `solid_in`), ADR-0023 (Horror's budgets) |

### Owned elsewhere (loaded or checked here only) (3)

| TR-ID | Owner | What this ADR does |
|---|---|---|
| TR-section-024 | ADR-0009 / ADR-0034 | PROJECTOR class rights are ADR-0009's; the `entity_id` packing, namespace byte and the 453-tile build rate are ADR-0034's. This ADR guarantees the three `entity_id` inputs and the per-section tile count (§ 13) |
| TR-section-033 | ADR-0007 | Boot-tier policy for knob ranges is ADR-0007 § 2.5; this ADR loads every Layout knob from `LayoutTuning`, starts at each minimum and maximum, and names the structural invariants that refuse a load (§ 12) |
| TR-derelict-003 | ADR-0007 / Derelict Authoring | The closed content-field set is Authoring's; Layout's schema carries `socket_tag` on a hardpoint as an advisory visual-class field (§ 4 visual payload) and adds no other field |

## Performance Implications
- **CPU**: `layout.doors` <= 0.2 ms at `MAX_DOORS_LOADED` on a release export (V2). Reads on the hot tier are index arithmetic and one `bsearch` at most; none allocates. LM1 to LM4 and restore are cold P2 or boundary events with a per-body grid rebuild (V3, ADR-0014); `can_attach` uncached <= 2 ms, memoised. `layout_hash()` is never per tick.
- **Memory**: the block is tens of bytes to tens of KB; `LayoutDerived` adds per-section columns of a few int64s, the socket-ref index, a bounded leaf cache (`MAX_DOORS_LOADED` slots), a small memo ring and preallocated traversal stacks sized by `MAX_SECTIONS`. Catalog templates hold the compiled rect table once per template.
- **Load Time**: three `Hash64.of_value` passes per template at catalog freeze (four templates in MVP), once. Restore validates and rebuilds in one cold pass.
- **Network**: none in MVP; `SHARED_SUMMARY` replication is the instance table plus three hashes (VS-B, ADR-0032).

## Migration Plan
There is no Layout code yet, so nothing migrates. Story order for the epic: (0) `layout_ids.gd` and `socket_ref.gd` with the `load_all` parse gate; (1) `TemplateCompiler` and the three hashes in `SectionTemplates`, with the ledger check; (2) `LayoutState`, `snapshot`/`restore`/`default_state` and the allow-list test; (3) LM1 with the `LayoutGraphPort` stub, `LayoutRoomSource` and `bind_graph`; (4) `layout.doors` and the leaf queries; (5) LM2/LM3 with the weld property tests (ADR-0021 shares them); (6) LM4 and the read views; (7) `post_restore` and verification (with ADR-0016). **GDD sync** (listed in the result and applied by the orchestrator): TR-section-004 and AC-130 to the 14-column payload and key names; LY-6 e and LY-26 e from sets to ordered sequences and the third hash; D.10, OQ-10 and the ADR-LAY-06 notes from owner TLV to payload shape; LY-32 and the key-space table to add LEAF; LY-14, LY-16, LY-17 and LY-27 for `out` records and `preview_attach`; D.21 for 14 columns; LY-26 a for native int64 times; LY-4 for the ledger nesting; LY-12 d for `LayoutVersions.invalidate` on discard and `version_epoch` / `new_epoch()` on restore (replacing "reset to 1 on `restore`" and "after a restore every section reports ALL once"); LY-26 for the byte-identical restore and the LM2 hash refresh; the Graph-port names (`check_instantiate_section`, `SectionBases`) wherever the GDD says `check_instantiate` / `graph_ref_base`.

## Validation Criteria
- The AC-130 allow-list test (regenerated) walks the MVP snapshot and finds exactly the § 2 keys.
- Round trip: `snapshot(restore(snapshot()))` is canonical-byte identical, including after a visual-only template edit and after a fresh-geometry template edit (the load verifies `EXACT`/`MATCH`; `restore` never rewrites a stored hash); a weld then refreshes the instance's three stored hashes from the current template in LM2; equal worlds built by different histories (build, weld, cut versus restore) hash equal; `mutation_count` appears in no payload (AC-172).
- Weld symmetry: `attach_section(a, b)` and `attach_section(b, a)` produce a byte-identical block and equal `AttachResult`; the lower `body_id` always survives; with the survivor-first `socket_plan`, **whole-world `state_hash` equality** (layout and graph state, including the seam row and `opening_ids` order) holds for reversed arguments (assumption on ADR-0021: its weld property test asserts whole-world `state_hash` equality for reversed arguments).
- Detach: `detach_preview(seam)` equals LM3's `moved_sections`, `frame_shift_px` and `rects`; exactly two components; the property test over generated section pairs (ADR-0021) holds.
- Dense ids: build, discard and restore keep `section_id == row + 1`; a hand-edited non-dense payload fails `LAYOUT_RESTORE_COUNTER`; the section ceiling returns `CAPACITY` before any id is consumed, and counters at `2^31 - 1` never wrap.
- Drift: a room reorder fails `LAYOUT_RESTORE_TEMPLATE_CHANGED` on a fresh instance; a wall-edge edit fails on a welded instance and succeeds with `geometry_changed_fresh` on a fresh one; a visual-only edit succeeds with `visual_changed`; `rot_q = 1` fails `LAYOUT_ROT_UNSUPPORTED`; a hash at or above 2^31 compares equal after the mask; both diagnostics (`geometry_changed_fresh`, `visual_changed`) re-raise on every load until the instance changes.
- Leaves: a closed leaf blocks and an open one does not in the tick its state changes (P2 write, P3 read); LM2 leaves the span solid with no `stale_leaf` and no `leaf_version` bump at the first P3; an unknown opening leaves the leaf closed with one `opening_unknown`; `area > area_max` clamps with a diagnostic; at most one bump per section per tick; `static_version` never bumps from the hook; the hook's cost line (V2).
- Restore fixtures (each yields a named fault, writes nothing, and reaches `return ctx.complete()` with no engine script error): a `template_id` column holding an `int`; a payload with the `welds` key missing; a socket `state` column carrying 259 (or any value outside 1..3); a `discarded` byte of 2; a non-`PackedInt32Array` hash column. `template_id` is restored as a typed `Array[String]` via `assign` even when the incoming array is untyped.
- Graph port: a RAGGED cut, save, restore: the verifier passes (template-range counts only, stubs and seam edges excluded) and `anchor_geometry(stub)` is the cut socket's hatch span; a discarded section reports `tombstoned` through `section_counts_into`. A port call before `bind_graph` asserts; the composition root order is `setup`, `GraphModule.setup(room_source())`, `bind_graph`, freeze.
- `can_attach`: `GEOM_OVERLAP` exercises both legs (the ADR-0014 rect leg, then Layout's tile-cell leg after `footprint_overlaps` returns no rect hit); `COORD_LIMIT` refuses an extent beyond `BODY_COORD_LIMIT` before any geometry test; an uncached run at the largest MVP body pair is reported against the 2 ms line (S1).
- Result arrays: an `AttachResult`/`DetachResult` reused across calls keeps independent contents (the caller that retained one `duplicate()`d it); no implementation assigns an array to a result field (code review plus a test that mutates `LayoutDerived` scratch and asserts the result is unchanged).
- Allocation-freedom of the Tier H reads is enforced by `HOT-01` (lexical scan of the `## @hot` bodies) plus `CNT-01` for leaked Objects only; `Performance.OBJECT_COUNT` is not claimed to prove that no packed array, `Array`, `Dictionary` or string was allocated. The explicit Tier H function list (§ 11) is added to the ADR-0010 `HOT-01` tag list.
- Dirty keys: the per-mutator fixture asserts the specific keys (LM2 5, LM3 6 in D.12's example, `LEAF` from the hook) and the ADR-0004 audit raises no `MISSED_DIRTY_MARK`.
- Static scans: no `Node`/`Resource`/float/`/` in `src/sim/layout/`; no write to `sockets` columns or counters outside the named mutators; no unmasked hash compare; no write to a record field outside Layout (`LAY-REC-01`); `socket_ref.gd` and `layout_ids.gd` pass the `load_all` parse gate (a regression for the unqualified `SOCKET_INDEX_BITS` defect).
- S4 loaded-set test through real `build_section` calls: tile and rect totals within the ceilings.
- Golden vectors for the three template hashes of the four MVP templates, equal on editor and export (V4).

## Related
- ADR-0001 (tick, P2/P3), ADR-0002 (no Layout command; Web Map `init_world`), ADR-0003 (block, mutators, faults), ADR-0004 (keys, audit), ADR-0005 (encoding, hashes, `IntMath`), ADR-0006 (types in a save, two-phase restore), ADR-0007 (catalog, ledger, boot tiers), ADR-0009 (presentation contract), ADR-0010 (layout, ports, lints), and `design/gdd/ship-section-and-layout-data-model.md` (owner).
- Sibling assumptions (cross-ADR): ADR-0011 provides contiguous M1 ranges in template order, the allocation-free `opening_into`, `seam_openings_into`, `section_counts_into` (template-range counts, `stub_count`, `tombstoned`) and `check_instantiate_section`, through the dedicated adapter `GraphLayoutAdapter extends LayoutGraphPort` (CT-01), notes that `weld_join` row order follows the canonical `JoinPlan`, and takes `RoomSource` from `LayoutRoomSource` (CT-02, co-acceptance CT-53); ADR-0012 leaves template-door `area` untouched in P3; ADR-0014 builds the per-body broadphase from the compiled rect accessors, owns `QueryBuffer`/`OccluderSource` signatures and exposes the three counters whose bump rules are § 7 to § 8; ADR-0015 consumes `frame_shift_px`, `moved_sections`, `rects[]` and `section_at`, reads `body_is_live`/`body_state`/`section_contains` (§ 11), defines `BODY_COORD_LIMIT` (repeated by `can_attach`), requires the absorbed body `DISSOLVED` in `rebase_on_attach` after LM2, and runs BM8 `release_body` after LM4 (CT-22, CT-23, CT-24); ADR-0016 orders restores and runs the verifiers; ADR-0021 orders Graph, Layout, Body inside one P2 apply; ADR-0026 calls LM1 in the `INIT` scope and enumerates loaded sets; ADR-0034 packs `entity_id`, assigns the `kind_ordinal` values append-only (JAMB included) and chooses the redraw path on `LEAF`; ADR-0014 owns `LayoutVersions.invalidate` / `new_epoch()` and `idx_of_key` (CT-15, CT-18); ADR-0017 reads `node_base` and the emitter-placement accessors (CT-17).
- Asks to Foundation ADRs: see the structured result (ADR-0003 check prefix; ADR-0004 key family table; ADR-0005 § 4.6 and ADR-0006 § 1.2 on `PackedStringArray` and the third hash; ADR-0007 ledger class split; ADR-0010 `LAY-*` lint rows including `LAY-REC-01` and the Layout `HOT-01` tag list; ADR-0003 one sentence admitting the `can_*` prefix with the `check_*` shape (CT-58, Foundation change)). `architecture.md` § 8 step 4 (construction order, `bind_graph`) and the S1 / SP-ENG-1 row (add ADR-0013 V2/V3/V3b) are updated by `/architecture-review`; this ADR may not edit them.

## Open Questions
- **OQ-1: per-body section cap.** `can_attach` has no `BODY_FULL` reason; a merged body could exceed `MAX_FOOTPRINT_RECTS_PER_BODY`. Recommended: keep it out of MVP (the roster is a few hundred rects) and add a world lint; reopen with a `MAX_SECTIONS_PER_BODY` knob at the Vertical Slice.
- **OQ-2: traceability file.** `requirements-traceability.md` line 1367 still lists TR-derelict-034 as `ADR-0013 + ADR-0007` while row 724 says it was re-tagged to ADR-0013 alone (2026-10-29). The summary table needs the same fix; this ADR is not allowed to edit it.
- **OQ-3: `structure_hash` approval.** The deviation from "13-int rows" needs the user's confirmation; the alternative is the GDD's two hashes with the reorder defect documented as an authoring rule. The hash is not redundant with `geometry_hash`: a reorder is already caught by `geometry_hash` on an instance with history, so the third hash exists for the never-welded `FREE` instance, where `geometry_changed_fresh` would otherwise accept a reorder and silently re-point room ids, compartments and socket indices.
- **OQ-4: continuous doors.** If a stepped slide (Option B) is introduced, the hook's key-per-tick cost (R3) and the early-out need re-measuring.
- **OQ-5: Graph change counter.** Whether ADR-0011 can publish a cheap "openings changed" counter to let the hook early-out; not required for correctness. It is the first rung to try in the ADR-0010 § 9 ladder, because the hook is dominated by the per-door port call.
- **OQ-6: `COORD_LIMIT` as a ninth `can_attach` reason.** Ruling CT-24 requires `can_attach` to repeat `BODY_COORD_LIMIT`; this ADR adds the reason `COORD_LIMIT` (before `GEOM_OVERLAP`) to the closed set, which the GDD (LY-19) and Assembly's reason mapping must then list. Alternative: fold it into `GEOM_OVERLAP`'s reason (rejected: a different cause, a different player message).
- **OQ-7: Accept ordering.** ADR-0013 is co-accepted with ADR-0011; ADR-0016 § 9.3 and ADR-0034 must adopt the byte-identical-restore and ordinal wording before either is Accepted.

## Review Log

**2026-10-29, revision after the per-ADR engine and TD reviews and the Core cross-ADR adjudication (autonomous mode, "Revise flagged items"). Status stays Proposed.** The Core ADR `Date` and `Last Verified` stay at their original values; this log is the revision record.

Engine review (verdict CONCERNS; Every V1 probe reproduced on 4.7.2):
- **E1 (BLOCKING, applied):** `SOCKET_INDEX_BITS` is now declared inside `SocketRef` (the second option; it is private to the packing) and removed from `LayoutIds`; probed values `make(5,3)=83` noted; `socket_ref.gd` and `layout_ids.gd` added as the first Layout stories and to the `load_all` parse gate (§ 1, Key Interfaces, Migration Plan, Validation).
- **E2 (applied):** `BuildResult`, `AttachResult`, `DetachResult` rewritten as valid multi-line files with explicit field defaults and a `reset()` method; stated once in Key Interfaces and § 9.
- **E3 (applied):** "get-only" removed from § 11. `out` records are plain typed `var`s written only by Layout, enforced by a new lint `LAY-REC-01` (routed to ADR-0010); Tier C returned records are set in `_init`/`make`.
- **E4 (applied):** § 7 and Guidelines: result and traversal arrays are filled in place with a separate `count`, never aliased, valid until the next Layout call (consumers `duplicate()`), fixed-length traversal stacks with a top index; the no-realloc property is marked UNVERIFIED as V3b attached to S1. `moved_count`/`rect_count` added to the records.
- **E5 (applied):** § 10 decode idiom: `has()` for every key, `typeof()` before every typed assignment, per-element `TYPE_STRING` check then `Array[String].assign`, packed columns copied with `duplicate()`, every early path `ctx.fault` then `return ctx.complete()`; fixtures for an int `template_id` and a missing `welds` key (Validation).
- **E6 (applied):** Guidelines bullet for `_`-prefixed unused parameters (`_views`, `_out`) and for locals that shadow view method names (`ver`, `leaf_ver`, `new_version`); `V` kept as the GDD name with `room_volume_tiles` as the exact adapter-facing method.
- **E7 (applied):** § 10 byte-range checks on the decoded value before narrowing writes (body `state` 0..1, socket `state` 1..3, weld `state` 0..2, `discarded` 0..1); corrupt-column fixture (socket `state` 259) in Validation.
- **E8 (applied):** the probe is recorded as the S1 baseline in V2; hook cost note says it is dominated by the port call; OQ-5 early-out is the first ladder rung. V2 stays UNVERIFIED.
- **E9 (applied):** Guidelines and Validation say allocation-freedom is enforced by `HOT-01` plus `CNT-01` (leaked Objects only), not by `OBJECT_COUNT`; the Tier H list is the Layout `HOT-01` tag list (ADR-0010 § 6.6).

TD review (verdict CONCERNS):
- **T1 (BLOCKING, applied; ruling CT-11):** `restore()` is byte-identical and never rewrites a stored hash; diagnostics re-raise on each load; LM2 `attach_section` refreshes the three stored hashes when an instance first gets history; "as designed" struck; Validation criterion: after a visual-only and after a fresh-geometry edit the load verifies `EXACT`/`MATCH`.
- **T2 (BLOCKING, applied; ruling CT-03):** § 3 states that the `bsearch` mapping covers template-range ids only and stubs resolve through `GraphView.section_of` plus the seam stub list, with `anchor_geometry(stub)` answering the cut socket's hatch span; § 10 (d) defined precisely (template-range rows, `comp_base + k`, discarded ranges are tombstones, stubs and seam edges excluded); RAGGED-cut fixture added.
- **T3 (BLOCKING, applied; ruling CT-02):** `LayoutViews` constructed inside `setup`; `LayoutRoomSource` implements ADR-0011's `RoomSource` (`room_volume_tiles`/`room_exposed_tiles` in Tier H with -1 for unknown); `setup` drops the Graph argument; `bind_graph` after `GraphModule.setup` and before the freeze; early port call is an asserted defect.
- **T4 (applied; ruling CT-01):** port renamed to `check_instantiate_section`, M1's out type is `SectionBases` (`InstantiateResult` does not exist); § 3 compartment row corrected (`comp_base == node_base`, no comp counter, opening ids are edge ids); both `seam_view` references replaced by `seam_openings_into`; INIT added to the valid scopes; `can_build` passes Graph reasons as `layout.GRAPH_<REASON>` (unreachable for validated templates, `onset_valid` at boot).
- **T5 (applied; ruling CT-17, CT-18):** "Reads owed to Core siblings" table in § 11 (`body_is_live`, `body_state`, `section_contains`, `node_base`/`edge_base`/`comp_base`, emitter-placement accessors, `RectFlags`, `SlotFeed`); `section_rects_into` recorded in § 5 and § 11 as the one section-keyed body-frame read.
- **T6 (applied; ruling CT-14):** JAMB added to the closed `kind_ordinal` vocabulary (values are ADR-0034's, append-only); template invariant `area_max <= doorway width in px` added (§ 4, § 12).
- **T7 (applied; ruling CT-15):** LM4 uses `LayoutVersions.invalidate` (all three counters 0); restore and `default_state` call `new_epoch()`; `version_epoch` is the authoritative restore signal; "ALL after restore" demoted to a non-guaranteed hint; GDD sync item for LY-12 d.
- **T8 (applied):** `post_restore` an `area` above `area_max` is `LAYOUT_RESTORE_DANGLING` with no clamp; the clamp with `area_clamped` is the hook's.
- **T9 (applied; ruling CT-13):** `socket_plan` and the `JoinPlan` are canonicalised survivor-first; whole-world `state_hash` equality added to the weld-symmetry criterion; assumption recorded on ADR-0021.
- **T10 (applied):** Engine Compatibility states Accept is not gated on V2/V3 and V4 gates the template-hash story; the S1 row and summary line 1367 are routed to `/architecture-review` (OQ-2, § Related).

Cross-ADR rulings:
- **CT-01 (applied as aligned):** port record `SectionBases`, adapter `GraphLayoutAdapter` outside `ports/`, M1's only caller is LM1 (§ 1 table row, § 3, § 8, § 11). **CT-02 (owner, applied):** see T3 and the four records in § 11. **CT-03 (applied as aligned):** see T2. **CT-11 (owner, applied):** see T1. **CT-12 (owner, applied):** the "survives still typed" claim removed from § 2 and V1; `Array[String].assign` after a `typeof` check. **CT-13 (owner, applied):** see T9. **CT-14 (owner, applied):** see T6. **CT-15 (owner, applied):** see T7. **CT-16 (owner, applied):** the tile-cell leg is Layout's own `floor_tile_at` check after `footprint_overlaps` (§ 6); V3 gains the uncached both-legs line. **CT-17 (owner, applied):** see T5. **CT-18 (aligned):** the body-frame exception stated in § 5. **CT-20 (aligned):** the hook's cost line is a ledger row owned by ADR-0003's P3 share ledger (0.2 ms, pending S1). **CT-22, CT-23 (aligned):** § 7 and § 9 record that LM2 dissolves the absorbed body before `rebase_on_attach` and that BM8 follows LM4. **CT-24 (applied):** `COORD_LIMIT` check in `can_attach`, using `BODY_COORD_LIMIT` from ADR-0015 via `LayoutTuning` (OQ-6). **CT-46 (no change):** `section_at(body_id, x, y)` stays as is. **CT-53 (owner, applied):** Depends On and Consequences record co-acceptance with ADR-0011. **CT-54, CT-58 (aligned):** HOT-01 tag list cited; the `can_*` prefix is admitted by a one-sentence ADR-0003 change.
- **Declined:** none. The rejected findings (the ADR-0006 `Array[String]` upgrade, `InstantiateResult` alias, the `ports/` placement of the adapter) were not applied.

Not edited (outside this file): `architecture.md` § 8 step 4, the S1 / SP-ENG-1 row, `requirements-traceability.md` line 1367, the registries, the GDDs and other ADRs; the GDD sync list is in the result.
- **2026-10-29 confirm residuals:** `section_contains` is stated section-local (`local_x`, `local_y`; the caller subtracts the offset, as ADR-0015 BM4 does); `can_build(views, template_id, kind)` reads the pending `next_section_id` and passes it to `check_instantiate_section`; the composition-root list quotes `GraphModule.setup(..., strict_numeric, work)`; `LayoutModule` declares `solid_query()` and `occluder_source()` and the Tier H list names ADR-0014's `solid_at`, `section_rects_into`, `section_of_key`, `idx_of_key` and the Layout-internal `footprint_overlaps` and `section_footprints_into`; the AttachResult/DetachResult prose uses `frame_shift_x`/`frame_shift_y`; the `area_max <= width_px` invariant is restated as a numeric bound with the unit reasoning; Enables gains ADR-0011.
- **2026-10-29 orchestrator rulings:** no R-id touches this ADR. Dependency-cycle break only: ADR-0011 is now the hard edge of the ADR-0011/ADR-0013 pair and is also declared in a new `Co-accepted with` row (removed from the Depends On cell prose of both directions); ADR-0011 and ADR-0016 removed from Enables (ADR-0011 does not list this ADR under Depends On; ADR-0016 reads this ADR as a draft); ADR-0016 and the unwritten ADR-0020/0021/0023/0026/0027 moved to a new `Soft references (not acceptance prerequisites)` row.
