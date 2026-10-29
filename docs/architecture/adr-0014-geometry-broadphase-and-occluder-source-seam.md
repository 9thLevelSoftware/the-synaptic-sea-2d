# ADR-0014: Geometry Broadphase and OccluderSource Seam

## Status
Proposed

## Date
2026-10-29

## Last Verified
2026-10-29 (against `design/gdd/ship-section-and-layout-data-model.md` LY-12, D.9, D.22 and AC-39, AC-136, AC-137, AC-151, AC-164, AC-173, AC-182; `ship-body-and-interior-physics.md` BDY-15 and D.12; `emitter-state-and-light-exposure-query.md` EM-20, EM-23; `2d-lighting-and-occlusion.md` LO-11; ADR-0001 to ADR-0010, all Proposed; sibling drafts ADR-0011, ADR-0013, ADR-0019. Engine facts marked "probed 2026-10-29" were run on the 4.7.2 mono console binary, headless, from outside the repo)

## Decision Makers
Claude (author), running under modes.automation: autonomous (project.yaml); decisions logged for the user's review; the user has not yet reviewed this ADR.

## Summary
Layout's rectangles are read by four consumers (Body sweeps the SOLID set, the Emitter runs a segment test over the OCCLUDES_LIGHT set, Lighting builds occluders from it and the PROJECTOR draws from it), but no document fixes the query contract, the spatial index behind it or the change counters the consumers cache against. This ADR decides: **one** derived per-body rect table laid out in ascending `rect_key` order behind a dense uniform grid whose cells hold row numbers (door leaves and socket plates are in the grid at their maximum footprint, and `layout.doors` only overwrites a row's current extent); **two** `@abstract` ports (`SolidQuery`, `OccluderSource`) that take an explicit `body_id` and four half-open integers and fill a **caller-owned `QueryBuffer`** (public structure-of-arrays columns, overwrite semantics, `count` authoritative, growth only in a cold method); and **int64 per-section counters** (`version`, `static_version`, `leaf_version`) with a non-persisted `version_epoch` and a tick-stamped `changed_region`.

## Engine Compatibility

| Field | Value |
|-------|-------|
| **Engine** | Godot 4.7.2 (pinned 4.7; installed 4.7.2.stable.mono, `docs/engine-reference/godot/VERSION.md`) |
| **Domain** | Core (a pure-integer, scene-tree-free data structure and its read ports). Physics is deliberately not used: no engine collision, shape query, tile collision or navigation mesh is created (TR-section-011). The one presentation-facing surface is the `OccluderSource` port, read by Lighting's occluder builder and the PROJECTOR |
| **Knowledge Risk** | MEDIUM. No rendering, physics or scene API is used. The HIGH items touched are the 4.7 packed-array element-write change (nothing in this ADR relies on a packed-array property setter; `QueryBuffer` has none) and the 4.7 typed-return override rule (it applies to every implementation of the two `@abstract` ports; sourced from `breaking-changes.md` § 4.6 to 4.7) |
| **References Consulted** | `docs/engine-reference/godot/VERSION.md`; `breaking-changes.md` (4.6 to 4.7 GDScript rows; 4.4 to 4.5 `@abstract`); `deprecated-apis.md`; `current-best-practices.md` § GDScript (4.5+) and (4.7); `modules/physics.md` (read to confirm it documents no 2D query API, so an engine-physics alternative cannot be sourced here). **`docs/engine-reference/godot/modules/` has no core, scripting or tilemap file**, so every claim about packed-array methods and GDScript cost below is either probed (2026-10-29) or marked UNVERIFIED. Project ADRs 0001 to 0010 and the sibling drafts ADR-0011, ADR-0013, ADR-0019 (§ Related) |
| **Post-Cutoff APIs Used** | `@abstract` ports (4.5; chosen by ADR-0010 § 3); the 4.7 typed-return override rule on port implementations; packed-array element writes that skip the property setter (4.7). `PackedByteArray.find` is *not* used by the default path (it is the named rung 4 of the fallback ladder, § 10) |
| **Verification Required** | **V1 (probed 2026-10-29, done):** `PackedByteArray.find(1, from)` returns the next index or -1; `from >= size` returns -1, but a negative `from` is end-relative and can return a valid but wrong in-range index with no -1 sentinel (probed on 4.7.2 with `[0,1,0,1]`: `find(1, -1)` returns 3, `find(1, -3)` returns 1), so a caller must never pass a negative start (the rung-4 extraction carries a `pos >= 0` guard in code and in the corpus test); a local alias of a `RefCounted` member `PackedInt64Array` writes through to the member (`is_same` true; an element write through the member is visible through the alias); `PackedInt32Array.sort()` and `PackedInt64Array.sort()` exist; `resize` to a smaller size discards the tail (a later grow returns zeros); a key `(1048575 << 12) + 4095` decodes by `>> 12` and `& 4095`. **V2 (probed 2026-10-29, done; editor binary, debug-capable, headless script, synthetic 2,000-rect body with 4.9 grid entries per rect, not a release export):** § 11 table. **V3 (UNVERIFIED, S1 / SP-ENG-1):** every wall-clock figure on a release export on RM-DESKTOP and RM-DECK, including the adapter hop and a real MVP rect mix. **V4 (UNVERIFIED, S1 with ADR-0013 V3):** the P2 cost of `rebuild_body` at the largest body the MVP can weld (probe: about 11.7 ms for 2,000 rects including random generation and a 2,000-element `Array.sort`, so an upper bound for the table-and-grid build; editor binary). V4 also carries the **uncached `GEOM_OVERLAP` cost, both legs** (this ADR's `footprint_overlaps` rect leg plus ADR-0013's tile leg, section 7) at the largest body pair the MVP can weld, against ADR-0013's `can_attach` target of 2 ms or less; a worst-case estimate of the rect leg alone is about 6 ms for 2,000 probes at about 3 us each on the editor binary (UNVERIFIED; the MVP pair is expected to be far smaller). **V5 (UNVERIFIED):** `Performance.MEMORY_STATIC` is available on a release export (ADR-0010 CNT-02 treats it as a developer and CI-debug check only). **V6 (probed 2026-10-29 by the engine review, done; 4.7.2 editor binary, every relevant warning at 2; answers ADR-0010 LINT-SPIKE-1(b), reported to ADR-0010 § 1.2):** `UNUSED_PARAMETER` does not fire on the parameters of an `@abstract` method, and the one-line `@abstract class_name PortA extends RefCounted` form loads, so the ports keep plain parameter names. A concrete override that does not use every parameter fails (`The parameter "body_id" is never used in the function "solid_in()"`), so `src/` implementations prefix unused parameters with `_`; doubles under `tests/` are exempt via `directory_rules`. The 4.7 typed-return rule is confirmed: an override with a `pass` body fails with `Not all code paths return a value`, and one without `-> int` also trips the untyped-declaration warning. **V7 (probed 2026-10-29 by the engine review, done):** under ADR-0010's warning policy a parameter, local or member that shares a name with a method of the same class is a parse error treated as an error (`SHADOWED_VARIABLE=2` covers functions), a member and a method of one name are a parse error, `: ...` is not a valid function body, and `const OK: int = 0` is accepted. These shaped the Key Interfaces names (`initial_capacity`, the private `_version_col` columns) |

## ADR Dependencies

| Field | Value |
|-------|-------|
| **Depends On** | ADR-0013 (the rect record, flags byte, reserved `idx` space, `rect_key`, the `layout.doors` hook that rewrites leaves, LM1 to LM4 call points, restore and the read-view tiers; a Proposed draft); ADR-0003 (`ModuleFacet`, read-only `ClockView.tick()`, hot-tier conventions, fault domain 4); ADR-0005 (`IntMath.floor_div`; the `/` and `%` operators are banned); ADR-0007 (`LayoutTuning`, BOOT-tier invariants); ADR-0010 (`@abstract` ports under `ports/`, `## @hot` and HOT-01, CNT-01 to CNT-03); ADR-0001 (P2 before P3, hook order, composition order). None is Accepted yet, so this ADR cannot be Accepted before them |
| **Soft references (not acceptance prerequisites)** | ADR-0016 (restore ordering over a derived cache that must be rebuilt) and ADR-0034 (the PROJECTOR reads rect extents through the same port; this ADR is a soft reference of ADR-0034). **Forward references (unwritten ADRs; each will list this ADR under Depends On when authored):** ADR-0028 (Lighting's occluder builder, reading `spans` as occupied spans, section 4), ADR-0021 (the weld's `GEOM_OVERLAP` probe uses `footprint_overlaps`), ADR-0020 (`clip_to_footprints` is built on `solid_at`), ADR-0023 (Horror's per-run `solid_in` reads). |
| **Enables** | ADR-0015 (Body via `LayoutCollide`, ADR-0015's `CollideSource` adapter, which holds an injected `SolidQuery` and passes the body id explicitly; `LocomotionStep` takes the `CollideSource`), ADR-0017 (the Emitter's occlusion test and static-layer rebuilds read `OccluderSource`), ADR-0018 (shadow origin and beam occlusion consistency, reading `spans` as occupied spans, section 4). |
| **Blocks** | The Layout epic's broadphase story; the Body sweep story; the Emitter static-layer story; the Lighting occluder-builder story; every story that constructs a `QueryBuffer` |
| **Ordering Note** | Wave B. ADR-0013 (wave A) is the input: where ADR-0013 left something to this ADR it is named in § 7 (rebuild call points, the `RectFlags` bit names, `solid_at`, `footprint_overlaps`). Acceptance is gated on S1 (SP-ENG-1) for the three wall-clock lines of § 11 and on the ADR-0010 exemption of § Foundation asks (the `QueryBuffer` columns). The signatures proposed here replace the provisional `rects_in`/`solid_in` signatures in `architecture.md` § Read views and the placeholder `solid_in(_rect: Rect2i, _out: QueryBuffer) -> int` in ADR-0010 § Key Interfaces (a Foundation ask); the sync list in § Migration Plan carries the edits |

## Context

### Problem Statement
Four systems read the same rectangles within one tick and must never disagree about a wall: Body clamps a 16 x 8 px foot box against SOLID rectangles (one query per moving actor per tick), the Emitter tests segments against OCCLUDES_LIGHT rectangles and rebuilds a static layer when a section changes, Lighting builds `LightOccluder2D` polygons from the same set, and the PROJECTOR draws door leaves whose extent must equal the leaf rectangle. The Layout GDD (LY-12) states the contract in prose (out-parameter buffers, ascending key order, a per-body uniform grid, three counters), the Emitter GDD defers its "candidate 4" (where the spatial index lives) to an ADR, and `architecture.md` lists `QueryBuffer`, `SolidQuery` and `OccluderSource` as types "still to define". Nothing yet fixes: what the buffer is and who owns it; what a query's boundary conventions, ordering and failure answers are; what the index stores and when it is rebuilt; whether door leaves live in the grid; how a consumer detects that geometry changed; and how all of that coexists with ADR-0010's allocation rules (HOT-01, CNT-01 to CNT-03), which forbid a `Packed*Array` returned from an accessor.

### Constraints
- **Integer-only, scene-tree-free, deterministic** (ADR-0010 SIM-01 to SIM-05, ADR-0005): no float, no `/` or `%` (so `IntMath.floor_div` for every px-to-cell conversion), no `Node`, `TileMapLayer`, `PhysicsServer2D`, `Resource`. Layout's rectangle reads are pure functions of `layout.state`, the frozen templates and the Graph-driven leaf projection (ADR-0013 § 11).
- **One rect set** (TR-section-011, Pillar 1): collision, light occlusion and drawing read the same rectangles; there is no "perceived" or phantom variant of any query, no engine tile collision and no navigation mesh.
- **Half-open input boxes, occupied-span output** (LY-12 b): every box passed in is `[ax0, ax1) x [ay0, ay1)`; every rect handed out is `(x0, y0, x1 - 1, y1 - 1)`. Output is in the **body frame** and in ascending `rect_key = section_id * RECT_KEY_STRIDE + idx` (D.9); the key is 64-bit.
- **Allocation-free steady state** (TR-section-015): zero growth over 9,000 calls after warm-up. ADR-0010 § 7 restates that as HOT-01 (no `.resize(`, `.append_array(`, array literals, `.new(` in a `## @hot` function), CNT-01 (`OBJECT_COUNT` flat over 600 ticks) and CNT-02 (`MEMORY_STATIC` within tolerance, debug binary only).
- **Budgets, all pending S1 on RM-DESKTOP and RM-DECK** (TR-section-016): `solid_in` is a share of ADR-0015's whole-hook `body.integrate` ledger entry of 0.05 ms per actor P99 on a 2,000-rect body (0.4 ms for eight actors); S1 reports it as a sub-counter of `body.integrate` and this ADR holds no separate aggregate line (ruling CT-20); an Emitter 288 x 288 px query at most 0.3 ms, charged inside ADR-0017's `EXPOSURE_BUDGET_MS`, not added to the hook ledger.
- **Derived and never persisted**: the grid and the counters live in `LayoutDerived` (ADR-0013 § 1), are rebuilt at build, attach, detach and restore, and never enter `layout.state` or a hash.
- **Same-tick truth**: Body must read the post-cut, post-leaf rectangle set of the tick in `body.integrate` (TR-shipbody-036); the hook order `layout.doors` before `body.integrate` before `emitter.place` (architecture.md § Data Flow 1) and the rule that every `area` write is a P2 command make that achievable.
- **Coupling**: `BROADPHASE_CELL_PX >= FOOT_BOX_W + 3` so a swept foot box touches at most 4 cells (TR-shipbody-037).

### Requirements
- A caller-owned result buffer that two consumers can use in an interleaved order within one tick without aliasing (AC-151).
- A query that returns each rectangle once, in ascending key order, with an explicit answer for an empty or inverted box and for an unknown body (AC-173, TR-horror-031).
- Counters a consumer can cache against, with the restore hazard (a reset to 1 can equal a stale cached value) closed by construction, not by convention.
- Ports that let Body, the Emitter and Lighting be tested with hand-written doubles (ADR-0010 § 3.5) and that give each consumer only what it needs (Body cannot see versions; Lighting cannot see `solid_in`).
- A named fallback ladder for the GDScript cost, ending at the GDExtension rung of ADR-0010 § 9, with the port as the seam.

### Binding stances respected (registry check, `docs/registry/architecture.yaml`)
- `interface_mechanism` (ADR-0010): the injected seams `SolidQuery` and `OccluderSource` are `@abstract` ports under `src/sim/layout/ports/`; implementations restate the full signature including `-> T`; consumers type `setup()` parameters as the port.
- `allocation_verification` (ADR-0010 § 7): "zero allocation" means HOT-01 plus CNT-01 plus CNT-02; the spans, keys and flags are written by index, never appended.
- `packed_array_exposed_or_setter_tracked` (ADR-0010): accessors return read-only `Array[T]`, never a `Packed*Array`, and no `Packed*Array` property has a `set(`. **`QueryBuffer` is not an accessor result**: it is a caller-owned out record, constructed and held by the consumer, with public columns and no setter. It still sits outside the prose of ADR-0010 § 7 CNT-03 ("the only `Packed*` fields allowed in `src/` are private to their owner"); this ADR therefore asks ADR-0010 for an explicit exemption (§ Foundation asks) instead of working around the rule.
- `integer_division_convention` / `integer_division_outside_int_math` (ADR-0005, ADR-0010): `IntMath.floor_div` for every cell computation; the operators `/`, `%`, `/=`, `%=` appear nowhere in the module.
- `physics_callback_tick_source`, `scene_tree_or_engine_server_in_sim_layers`, `float_or_wall_clock_in_sim`: no `_physics_process`, no `PhysicsServer2D`/`PhysicsDirectSpaceState2D`, no float, no `Time.*` in the module; timing is measured only by the nightly benchmark lane.
- `native_code_policy` (ADR-0010 § 9): the GDExtension rung, if S1 demands it, replaces `RectIndex` behind the same ports; this ADR does not enter it.
- `block_as_resource_or_node`, `live_reference_in_snapshot_payload`: the grid is a plain `RefCounted` derived cache and never appears in a snapshot.
- Frame order (ADR-0001, `frame_order`) and `presentation_render_callback` (ADR-0009): presentation reads the port only inside the frame's presenter stages, after the TickDriver, so it sees committed state.

## Decision

### Architecture Diagram

```
 layout.state (persisted, ADR-0013) + frozen SectionTemplates (ADR-0007)          Graph opening(edge) (ADR-0013 § 8)
            |                                                                               |
            |  rebuild events (LM1..LM4, restore): RectIndex.rebuild_body / release_body    |
            v                                                                               v
 +----------------------------- LayoutDerived (never persisted, never hashed) ------------------------------+
 |  RectIndex                                    LayoutVersions                                              |
 |   BodyGrid[body_id - 1]   (one per LIVE body)   version / static_version / leaf_version  (int64, by section)|
 |    rows, ascending (section_id, idx):           version_epoch  (int, +1 per whole-block replacement)       |
 |     key | flags | class | cur extent | fp       changed_region  (kind, box, tick stamp)                      |
 |    dense CSR grid: cell -> row numbers          ^                                                          |
 |    stamp[] + cand[] scratch                     | bump_static / bump_leaf / stamp_*  (Layout only)         |
 |   ^ set_extent(row, ...)  <- layout.doors (P3)  |                                                          |
 +---------------------------------------------------------------------------------------------------------+
        |  query(body, box, mask, out)                      |  version(...), changed_region_*(...)
        v                                                   v
 LayoutSolidQuery extends SolidQuery               LayoutOccluderSource extends OccluderSource
   solid_in(body_id, ax0, ay0, ax1, ay1, out)        rects_in(body_id, ax0, ay0, ax1, ay1, out)
        |                                            section_rects_into(section_id, mask, out) / section_of_key / idx_of_key
        |                                            version / static_version / leaf_version / version_epoch
        |                                            changed_region_kind / changed_region_box_into
        v                                                   v
 Body (ADR-0015, via LayoutCollide        Emitter (ADR-0017)          Lighting builder (ADR-0028)    PROJECTOR (ADR-0034)
 (LayoutCollide holds a SolidQuery)
 Horror (ADR-0023, <= 5 per run)          each owns its QueryBuffer(s)    (PRESENTATION, PROJECT stage; reads after the tick)
```

### 1. One rect set: the body table

**One structure, three consumers' views.** For every LIVE body `RectIndex` owns one `BodyGrid`. Its core is a **row table** with one row per rect slot of every section on the body: every static rect, every door-leaf slot and every socket PLATE/JAMB slot of ADR-0013 § 5, **including slots that are currently inactive or zero-length** (they keep their row, exactly as they keep their `idx`). `rects_in` and `solid_in` are not two indexes: they are **two flag masks over this one table** (`OCCLUDES_LIGHT` and `SOLID`, registry `layout_rect_flags`). A third reader, the PROJECTOR, reads the same rows through `section_rects_into` (§ 5). That single table is what TR-section-011 means by "one rect set": no consumer has a private copy to drift, and no query has a perceived or phantom variant.

**Rows are in key order by construction.** The table is built by walking the body's sections in ascending `section_id` (`sections_of(body_id)` already answers ascending) and, inside a section, `idx` ascending (`0 .. rect_count() - 1`). Because `rect_key = section_id * RECT_KEY_STRIDE + idx` and `idx < RECT_KEY_STRIDE`, **row order is key order**: row `r < s` implies `key[r] < key[s]`, keys are strictly ascending and unique, and no sort is ever needed to produce ascending-key output. This is a rule, not an accident of one probe: the build asserts it (a debug diagnostic `rows_not_ascending`) and a unit test walks every fixture body.

**Columns** (all derived, all `RefCounted`-private to `BodyGrid`; the Packed fields are private to their owner, so they are inside ADR-0010 CNT-03 as written):

| Column | Type | Meaning |
|---|---|---|
| `key` | `PackedInt64Array` | `rect_key`; strictly ascending |
| `flags` | `PackedByteArray` | the LY-7 flags byte, copied from the template |
| `cls` | `PackedByteArray` | row class bits: `STATIC_MEMBER` (in the mating static set, § 7), `DYNAMIC` (a leaf or a PLATE/JAMB slot whose current extent can change) |
| `cx0, cy0, cx1, cy1` | four `PackedInt32Array` | the **current extent**, half-open, body frame (section-local rect plus `section_offset`); the only columns a query tests |
| `fx0, fy0, fx1, fy1` | four `PackedInt32Array` | the **maximum footprint**, half-open, body frame; fixed between rebuilds; used to place the row in cells and by `footprint_overlaps` |
| `stamp` | `PackedInt64Array` | per-row generation stamp for de-duplication; reset to 0 at rebuild |

**The empty extent is a sentinel, not a flag.** A slot with no current rectangle (an inactive PLATE or JAMB, a zero-width JAMB at `width_tiles` 2, a retracted leaf with `leaf_w` 0) has `cx0 = cy0 = EXTENT_EMPTY_LO` and `cx1 = cy1 = EXTENT_EMPTY_HI` (`LO = 0x3FFFFFFF`, `HI = -0x3FFFFFFF`, both inside `int32` and far outside `WORLD_COORD_LIMIT`). The overlap test `cx0 < ax1 and ax0 < cx1 and cy0 < ay1 and ay0 < cy1` is false for it on every query box, so the hot loop needs no "active" check and an inactive slot is "omitted exactly like a `leaf_w` 0 rect" (LY-12 a) by arithmetic.

**Dynamic slots are in the grid at their maximum footprint (replaces D.22's linear dynamic list).** The GDD's cost model scans a separate dynamic leaf and variant list on every query (`N_dyn x t_bbox` plus a merge of its hits into the key-ordered output). That is unnecessary: a door leaf's current rectangle is always inside its closed footprint (hinge side, `leaf_w <= width_px`, depth `WALL_FOOTPRINT_DEPTH`; ADR-0013 § 8), and PLATE and JAMB slots never move inside a build. Every dynamic slot is therefore inserted into the grid cells of its **maximum** footprint at rebuild, and `layout.doors` thereafter writes **only the four `cx*`/`cy*` cells of the row** through `set_extent`. The grid never changes while a door animates; a retracted leaf is rejected by the per-candidate box test against its (smaller or empty) current extent; the `N_dyn` and `t_merge_dyn` terms vanish from the cost model; and order is preserved because the leaf is an ordinary row. Consequence: a `set_extent` outside the row's footprint is a defect (debug diagnostic `slot_extent_outside_footprint`, covered by the corpus test), because the row would be invisible to cells it now overlaps.

**Flag masks.** `RectFlags` (bit names from the registry; the bit values and the file are ADR-0013's Layout epic, § Cross-ADR assumptions): `SolidQuery.solid_in` uses `RectFlags.SOLID`; `OccluderSource.rects_in` uses `RectFlags.OCCLUDES_LIGHT`; `section_rects_into` takes the caller's mask. Actors and horrors have no row (Layout holds no actor), so **they can never occlude or block through this index** (TR-light-035, TR-emitter-018 c); a prop is in the OCCLUDES_LIGHT set only when its template flags say so (LY-7).

### 2. The grid: dense, uniform, CSR

**Shape.** Per body: the cell size is `BROADPHASE_CELL_PX` (`LayoutTuning`; GDD Tuning Knobs, unregistered in `entities.yaml`, § Open Questions); the origin cell is `(floor_div(min_fx0, c), floor_div(min_fy0, c))` over the body's non-empty footprints, so a body whose frame went negative after a detach re-base is handled; the grid is `gw x gh` cells covering the footprints' bounding box. It is stored in compressed-sparse-row form: `cell_start: PackedInt32Array` of `gw * gh + 1` offsets and `cell_items: PackedInt32Array` of row numbers. The build is two passes over the rows in ascending row order (count, then fill), so **row numbers are ascending inside every cell** without a sort. A row is inserted in every cell its maximum footprint overlaps; a row with an empty footprint (a zero-length slot that can never have a rectangle) is not inserted.

**Why dense and not a hash.** A `Dictionary` keyed by cell has no deterministic iteration cost, allocates on insert, and its iteration order is not part of the project's determinism contract (ADR-0005 bans iterating dictionaries for canonical order); a dense CSR is index arithmetic and two array reads per cell. Its memory is bounded by the body's bounding box: a long chain of sections is wide but shallow (about 3,000 cells for a 14,000 x 450 px body at a 48-px cell), and a debug diagnostic `broadphase_oversize` fires when `gw * gh` exceeds eight times the row count, which would indicate a sparse, far-flung body that needs a different structure.

**Rebuild events (TR-section-016: "build, attach, detach, restore").** `RectIndex.rebuild_body(body_id, feed)` is called, synchronously and as the **last step of the mutator's apply**, by LM1 (the new section's own body), LM2 (the surviving body; the absorbed body is released), LM3 (both bodies), LM4 (the discarded section's body, or released if it has no live section) and `restore` (every LIVE body, with every leaf closed, then `post_restore` re-projects leaves with `set_extent`). `feed` is a Layout-internal helper (`SlotFeed`, not a port) that answers `slot_count(section_id)`, `slot_footprint(section_id, idx)` and `slot_extent(section_id, idx)` from the template accessors, the section offsets, the socket states and the leaf cache. Rebuilds are cold P2 or boundary events: they allocate (a new `BodyGrid` and arrays), and they are the only place a `.resize(` appears. A rebuild never runs from a hook, a query or presentation. An incremental update of the survivor's grid on a weld is rejected for the MVP (§ Alternatives 8); V4 measures the full rebuild and the ladder (§ 10) names the incremental rung.

**Cell coupling (TR-shipbody-037).** The swept foot box of one actor-tick (`FOOT_BOX_W + |step_x|` wide, `FOOT_BOX_H + |step_y|` high, Body D.12) has width at most `FOOT_BOX_W + V_RUN`. A half-open interval of width `w <= c` overlaps at most two cells whatever its offset, so the 4-cell bound holds iff `BROADPHASE_CELL_PX >= max(FOOT_BOX_W, FOOT_BOX_H) + V_RUN`. The TR text writes the constant 3; it is `V_RUN` (registry, base px per tick), so the check reads the name. It is a **BOOT-tier named structural invariant `BROADPHASE_CELL_COUPLING`** (a `CONTENT_INVARIANT`, ADR-0007 § 2.5/2.1, domain 2), not an advisory range: the pure helper `BroadphaseCoupling.holds(cell_px, foot_box_w, foot_box_h, v_run) -> bool` lives in `src/shared/layout/broadphase_coupling.gd` (ADR-0007 § 1.9: shared pure functions that a data loader and a sim module both need), the `ContentLoader` calls it at stage 4 over the loaded `LayoutTuning`, Body tuning and Controller tuning rows, and the same helper is the subject of the bound test ("a configuration violating it must fail the bound test"). Cross-table citation is a Foundation ask on ADR-0007 (§ Foundation asks). Layout's `rects_in` and `solid_in` are correct for **any** positive cell size (the corpus test runs at the GDD's published minimum and maximum); the invariant only protects the 4-cell *cost* bound. `BROADPHASE_CELL_COUPLING` is the single owner of TR-shipbody-037: ADR-0015 drops its own `BODY_BROADPHASE_CELL` (`FOOT_BOX_W + VEL_STEP_MAX`, weaker than this one) and cites this invariant (ruling CT-21).

### 3. The query

One internal function serves every box query (`RectIndex.query(body_id, ax0, ay0, ax1, ay1, mask, out) -> int`), tagged `## @hot`. `LayoutSolidQuery.solid_in` and `LayoutOccluderSource.rects_in` are one-line wrappers that pass `RectFlags.SOLID` and `RectFlags.OCCLUDES_LIGHT`. In order:

1. `out.count = 0`, `out.status = QueryBuffer.OK`, `out.cells_touched = 0`, `out.candidates_tested = 0`. **A query overwrites; it never appends.** The caller's old contents are gone.
2. **Empty-box guard (D.9, AC-173).** If `ax1 <= ax0` or `ay1 <= ay0` the box touches zero cells: `out.status = EMPTY_BOX`, the diagnostic `query_box_empty` is raised, and the call returns 0. The guard comes first so an inverted box (`ax1 = ax0 - 200`) can never produce a positive product of two negative cell counts.
3. **Unknown body.** If `body_id` is outside `1 .. size` or has no grid (dissolved, never built), `out.status = UNKNOWN_BODY`, `query_unknown_body` is raised, and the call returns 0. It is **not an error and not a fault** (ADR-0013 § 11: sentinels, never errors), but the status lets a caller distinguish "no rectangles here" from "no such body" (Horror's rule that UNKNOWN is neither lit nor open, TR-horror-031).
4. **Cells.** `cx0 = floor_div(ax0, c)`, `cx1 = floor_div(ax1 - 1, c)` (likewise `cy`), `out.cells_touched = (cx1 - cx0 + 1) * (cy1 - cy0 + 1)` (the D.9 formula, **before** clamping, so the closed-form assertions of AC-39 and AC-173 hold). The ranges are then clamped to the body's grid; a box wholly outside the grid returns 0 with status `OK` (a legal answer: no rectangles).
5. **Collect.** `gen = grid.next_gen()`. For each cell in row-major order, for each row number `r` in `cell_items[cell_start[cell] .. cell_start[cell + 1])`: if `stamp[r] != gen`, set `stamp[r] = gen`, increment `candidates_tested`, and if `(flags[r] & mask) != 0` and the half-open overlap test against the **current extent** passes, insert `r` into the scratch array `cand[0 .. n)` at its sorted position (insertion by row number). The stamp makes a rect that spans several cells be tested once; the insertion makes `cand` ascending; ascending row number is ascending key (§ 1).
6. **Emit.** If `n > out.capacity`, call `out.grow(n)` (cold, untagged, counted, § 4). Take local aliases of the three columns **after** that check (a style rule that keeps the loop on locals, not a correctness requirement: probed, a later `resize` through the member is also visible through the old alias), then for `k` in `0 .. n`: `key[k] = row key`, `spans[4k .. 4k + 3] = (cx0, cy0, cx1 - 1, cy1 - 1)`, `flags[k] = row flags`; set `out.count = n` and return it.

**Why stamp plus insertion for rung 0.** It needs no helper call beyond array indexing, has no failure mode if an invariant is broken (a bad stamp costs a duplicate test, not an out-of-range read), and its measured cost equals the alternative on the small boxes that dominate the 8-actor aggregate (§ 11). Its cost is quadratic in the result count; the probe shows that is invisible at the Emitter's box (59 results: 34 us) and only matters beyond several hundred results (443 results: 348 us versus 249 us for mark-and-find). The linear alternative is the named rung 4 of the ladder, pre-validated by the same corpus, so switching is a local change.

**Cost shape, replacing D.22's model.** `cost = t_call + t_adapter + cells * t_cell + entries * t_stamp + c * (t_test + t_insert) + c * t_emit`, where `entries` is the cell-list entries read (a rect spanning k cells is read k times, stamped once), `c` the unique candidates tested. The old `N_dyn * t_bbox + t_merge_dyn` terms are gone (§ 1). Every `t_` is UNVERIFIED on a release export (V3).

**`solid_at` and `footprint_overlaps`** use the same traversal (§ 7). Neither fills a buffer.

### 4. `QueryBuffer`: the caller-owned out record

`QueryBuffer` (`src/sim/layout/views/query_buffer.gd`, `RefCounted`) is one of the project's out records (the family of `AttachResult`, `DoorPassage`, `FailureSlice`). Contract:

- **Columns (TR-section-015).** `keys: PackedInt64Array` (the 64-bit `rect_key`), `spans: PackedInt32Array` (four ints per record: `x0, y0, x1 - 1, y1 - 1`, the occupied span in the body frame) and `flags: PackedByteArray`. They are **public vars with no `set(` accessor**: a consumer reads `out.keys[i]` or hoists `var k: PackedInt64Array = out.keys` into a local. Probed 2026-10-29: a local alias of a member packed array writes through and reads through (`is_same` true), so Layout also hoists the three columns once per call and writes by index.
- **Span convention and the one conversion.** `solid_in` is the canonical name; `solid_in_into` in ADR-0015 and the GDDs is synced to it (GDD sync). The input box is half-open but `spans` are **occupied (closed) spans**. Every consumer that works on half-open boxes converts in exactly one inline place: `x0 = spans[4k]`, `y0 = spans[4k + 1]`, `x1 = spans[4k + 2] + 1`, `y1 = spans[4k + 3] + 1`. Reading `spans[4k + 2]` as a half-open `x1` shrinks every rect by 1 px on its high edges: an actor could enter a wall by one pixel and two abutting hull rects would leave a 1-px seam. `CollideSource` / `LayoutCollide` (ADR-0015) is the only permitted wrapper over `SolidQuery`: it holds an injected `SolidQuery`, takes `body_id` explicitly (there is no stateful bind) and hands Body's `AxisClamp` records decoded through that one helper. Occluder consumers (ADR-0017 D8.2, ADR-0018, ADR-0028) read `spans` the same way: a polygon corner is at `(x0, y0)` and `(x1_occ + 1, y1_occ + 1)`; **Layout applies no shift**.
- **`count` is the only valid length.** The packed arrays have a *capacity* at or above `count`; entries at and after `count` are stale and must never be read, and `keys.size()` is not the record count. (Using a separate `count` instead of `resize(n)` per call is what keeps the steady state free of reallocation: probed, shrinking then growing discards the tail, so a `resize`-to-`count` design would reallocate on every size change.)
- **Ownership and lifetime.** Each consumer constructs and owns its buffers; one buffer per concurrently live result. A result is valid until the *same buffer's* next query or `clear()`. Two consumers (or two queries of one consumer that must both stay live) use two buffers, so an interleaved A-box1, B-box2, A-box3, B-box4 sequence (AC-151) cannot alias. A consumer **must never** store `out.keys` (or `spans`, `flags`) in a member field beyond the call that filled it, and **must never** pass the same buffer to two callees that both expect to keep it.
- **Diagnostics fields.** `status` (`OK`, `EMPTY_BOX`, `UNKNOWN_BODY`), `cells_touched` (D.9, pre-clamp), `candidates_tested` (unique rows tested) and `grew` (times the capacity grew). The two counts are the deterministic work counters the budget tests assert (the closed form of CNT-04 for this module) without an injected `WorkCounter`.
- **Capacity and growth.** The constructor takes `initial_capacity` in records (the parameter is not named `capacity`, which is a method of the class). **A consumer sizes its buffer for the worst case the query can return**: no query returns more rows than the body has, and `MAX_FOOTPRINT_RECTS_PER_BODY` bounds the body, so a buffer of that capacity (about 25 bytes per record: 8 + 16 + 1) cannot overflow at the MVP and costs roughly 50 KB. A consumer that knows a smaller bound (Body's swept box returns a handful) may use less. Overflow does **not** truncate (a truncated SOLID result would let an actor walk through a wall): the query calls `out.grow(n)`, an untagged cold method that `resize`s the three columns to `maxi(min_capacity, capacity() * 2)` (geometric growth, so a run of slightly larger results does not reallocate each time), increments `grew` and raises `query_buffer_grew`. "Steady state" is defined as **`grew` stays 0 after warm-up**; a test asserts it over the 9,000-call run (AC-137). Because HOT-01 scans the *tagged* function only, `grow` sits outside the tag; the `.resize(` token therefore never appears inside a `## @hot` body.
- **Helpers (cold, not `@hot`).** `clear()` (sets `count = 0`), `capacity() -> int`, `grow(min_capacity)`, `append_record(key, x0, y0, xm1, ym1, flags)` (for test doubles; grows as needed) and `is_ascending_unique() -> bool` (a linear scan of `keys[0 .. count)`, used by tests and by Body's debug-only `diag.solid_in_unordered` check, which stays Body's, ADR-0015).
- **Decoding a key.** A consumer that needs the section of a record calls `OccluderSource.section_of_key(key)`; the stride is Layout's tuning, so the buffer does not know it.
- **Not a persisted or hashed object**; never enters a snapshot (`live_reference_in_snapshot_payload`).

`RegionBox` (`src/sim/layout/views/region_box.gd`) is the same kind of record for `changed_region`: `x0, y0, x1, y1` half-open in the section's body frame, caller-owned and reused.

### 5. The two ports

Both are `@abstract` classes in `src/sim/layout/ports/` (ADR-0010 § 3.1 names that path for `OccluderSource`; `SolidQuery` sits beside it because Body and Horror, both sim modules, may import `sim/layout/ports`). Fallback if LINT-SPIKE-1(a) shows `@abstract` does not survive a release export: plain base classes whose methods `push_error("ABSTRACT_CALLED")` and return a safe default (`0`/`false`, empty status), call sites unchanged.

**Why two ports and one index.** Least privilege and testability: Body (through ADR-0015's `LayoutCollide`) and Horror need exactly one query and must not see versions or change stamps; the Emitter, Lighting and the PROJECTOR need the OCCLUDES_LIGHT set and the counters and never `solid_in`. A test double per consumer (`FakeSolidQuery extends SolidQuery`, `FakeOccluderSource extends OccluderSource`, both in `tests/support/`; the name `FakeSolidQuery` keeps it distinct from ADR-0015's `CollideSource` doubles) stays one method wide for Body's adapter. GDScript has single inheritance, so one facade could not extend both ports anyway: `LayoutSolidQuery extends SolidQuery` and `LayoutOccluderSource extends OccluderSource` are two thin adapters over the same `RectIndex` and `LayoutVersions`. The extra hop is measured: a six-argument call through an `@abstract` port-typed variable cost 0.195 us versus 0.188 us typed directly (probed 2026-10-29), so the hop is noise against a 3 us query, and ADR-0010 § 3.4's concern about ports in a hot loop is closed for this seam (S1 re-measures on a release export).

**Explicit `body_id`, no per-body handle.** The GDDs write `occluder_source(body_id) -> OccluderSource` and `solid_in(body_id, aabb)`. This ADR makes `body_id` an argument of both ports and keeps **one** `OccluderSource` instance for the whole world. A per-body bound handle would need an object per body (bodies are created at every detach and dissolved at every weld), would force the Emitter to cache and refresh handles across a weld, and would turn a stale cached handle into a silent wrong-body read. With `body_id` as an argument, the Emitter reads `body_of(section_id)` (re-read in W0 when `static_version` changes, TR-emitter-021) and passes it, and the two ports have the same shape. `version(section_id)` takes a `section_id` because the counters are per section.

**Box arguments are four integers, not `Rect2i`.** `architecture.md` shows `aabb: Rect2i` and ADR-0010's placeholder sample shows `_rect: Rect2i`; Rect2i is legal in a signature but wrong here: (a) ADR-0013's guideline bans engine geometry types from the module, (b) Rect2i is `position + size`, so an inverted box (`ax1 = ax0 - 200`, the D.9 guard case) is a negative size whose member behaviour is not part of the project's contract, (c) `Rect2i` helper methods (`grow`, `merge`, `has_point`) are integer-division-adjacent code SIM-04 cannot see, and (d) every other Layout hot read already takes scalars (`room_at(section_id, x, y)`, ADR-0013 § 11). The box is `[ax0, ax1) x [ay0, ay1)`.

**Return type is `int` (the record count, equal to `out.count`).** `architecture.md` writes `-> void`; ADR-0010's sample and LY-12 e write the count. The count is returned so a caller can branch without a field read; `out.count` stays authoritative. `RETURN_VALUE_DISCARDED` is 0 in the warning policy, so a caller that ignores it is legal.

```gdscript
# src/sim/layout/ports/solid_query.gd   (Body via LayoutCollide (ADR-0015), Horror)
@abstract class_name SolidQuery extends RefCounted
## SOLID rectangles of one body overlapping the half-open box, current leaf state, body frame, ascending rect_key,
## each once. Overwrites `out`. Unknown body or empty box: returns 0 and sets out.status.
@abstract func solid_in(body_id: int, ax0: int, ay0: int, ax1: int, ay1: int, out: QueryBuffer) -> int

# src/sim/layout/ports/occluder_source.gd   (Emitter, Lighting builder, PROJECTOR)
@abstract class_name OccluderSource extends RefCounted
@abstract func rects_in(body_id: int, ax0: int, ay0: int, ax1: int, ay1: int, out: QueryBuffer) -> int   # OCCLUDES_LIGHT set
@abstract func section_rects_into(section_id: int, mask: int, out: QueryBuffer) -> int   # bulk read for builders; § below
@abstract func version(section_id: int) -> int            # 0 = no such live section
@abstract func static_version(section_id: int) -> int
@abstract func leaf_version(section_id: int) -> int
@abstract func version_epoch() -> int
@abstract func changed_region_kind(section_id: int) -> int           # ChangedRegion.NONE / ALL / REGION / FRAME
@abstract func changed_region_box_into(section_id: int, out: RegionBox) -> bool   # true iff kind is REGION
@abstract func section_of_key(key: int) -> int
@abstract func idx_of_key(key: int) -> int                 # IntMath against RECT_KEY_STRIDE; kind is derived from ADR-0013 section 5's reserved idx ranges
```

Implementations restate the full signature including `-> T` and return explicitly on every path (4.7 typed-return rule; a stub that omits it is a defect found by ADR-0010 `REFLECT-SIG`). Ports keep plain parameter names; `src/` implementations prefix any parameter they do not use with `_` (V6; the two Layout adapters use all of theirs, a Horror or Body implementation may not).

**`section_rects_into(section_id, mask, out)`** is the bulk read the occluder builder and the PROJECTOR need and that no GDD named: all rows of one section whose flags intersect `mask` and whose current extent is non-empty, in the body frame, ascending key, same record layout. It walks the section's contiguous row block (no grid, no stamp), so its cost is linear in the section's rect count and it is meant for change-driven rebuilds, not per-frame polling. It lets Lighting rebuild "section s" without guessing a bounding box for `rects_in` (which would also return a neighbouring section's abutting rects) and lets the PROJECTOR size a door-leaf sprite from the same row the simulation tests, which is the "leaf sprite extent equals the leaf rect" half of TR-section-013 and the "drawing" leg of TR-section-011. Unknown or discarded section: returns 0 with `out.status = UNKNOWN_SECTION`. Rows are **occupied spans in the body frame** (width = span + 1; a deliberate exception to section-local frames, stated in ADR-0013 section 5's frame rule), so a section-scene consumer subtracts `section_offset_x/_y(section_id)`. A row whose current extent is empty is omitted, so a fully retracted leaf has no row and the consumer hides that sprite. The consumer derives the kind (JAMB, PLATE, LEAF and so on) from the reserved idx ranges of ADR-0013 section 5 using `idx_of_key(key)`, implemented with `IntMath` against the stride (the stride is Layout tuning and `/` and `%` are banned outside `IntMath`); `section_of_key` and `idx_of_key` are the only sanctioned key decoders.

**Validity classes (ADR-0013 § 10).** Door-leaf rows are Class 2: between `restore()` and `post_restore()` every leaf row holds the template default (closed) and `stale_leaf` is raised if read; the rest of the table is valid when `restore()` returns.

**Injection.** The composition root builds the two adapters from the Layout module after `setup` (`layout.solid_query()`, `layout.occluder_source()`) and passes them to Body, Horror, the Emitter and, as a `setup()` parameter, to the Lighting builder and the PROJECTOR. Presentation names only `sim/layout/ports/**` and `sim/layout/views/**` (ADR-0010 § 2.2 matrix); because the kernel cannot tick while presentation runs (ADR-0009 coherence argument), a presenter's read inside its stage sees committed state. Presentation must key on `(version_epoch, version)` and the dirty batch, **never on `changed_region`** (§ 6).

### 6. Version counters, `version_epoch` and `changed_region`

**Storage.** `LayoutVersions` (in `LayoutDerived`, ADR-0013 § 1) holds, indexed by `section_id`, three private `PackedInt64Array` columns `_version_col`, `_static_col`, `_leaf_col` (prefixed because a member may not share a name with a method of the same class, and `version`, `static_version` and `leaf_version` are the port's method names). The getters `version_of(section_id) -> int`, `static_version_of` and `leaf_version_of` are tagged `## @hot` and each begins with an explicit bounds guard `if section_id < 0 or section_id >= _version_col.size(): return 0`, because an out-of-range packed-array index is an engine script error and not a 0; the adapter's `version`, `static_version` and `leaf_version` delegate to them. The GDD says "u32"; 64-bit columns remove the wrap question (a door that animated every tick would take about 2.2 years of play to pass 2^32) and cost nothing; consumers compare for **inequality** with their cached value, never for ordering. `0` is the sentinel for "no such live section" and is never a real value (a built section starts at 1).

**Bump API (Layout-internal; nothing else writes these columns).** `bump_static(section_id)` adds 1 to `static_version` and to `version`; `bump_leaf(section_id)` adds 1 to `leaf_version` and to `version`; `invalidate(section_id)` sets all three to 0 (discard). The closed list of callers is ADR-0013 § 7/§ 8/§ 10 and the GDD's LY-12 d: `build_section` (0 to 1), `attach_section` and `detach_section` (`bump_static` on **every section whose socket state changes, including the host that does not move, and every moved section**), `discard_section_geometry` (**`invalidate`**: a cached consumer sees a change to 0, and since ids are never reused 0 cannot alias a later section; **this is an explicit deviation: it replaces the "bump on discard" of Layout GDD LY-12 d and ADR-0013 § 7/§ 10 with a value no live section can hold** (a GDD sync item and a cross-ADR assumption carry it; ADR-0013 sections 7 and 10 adopt `LayoutVersions.invalidate` in place of the bump and cite `version_epoch` as the restore signal, ruling CT-15), and it makes `version()` agree with ADR-0013 § 11's "0 for an unknown or discarded section"), `layout.doors` (`bump_leaf`, at most once per section per tick, never `static_version`) and `post_restore` (`bump_leaf` for each section whose derived leaf set is not all-closed, so a restored open door is a 1 to 2 transition). A weld of two single-section bodies therefore bumps both sections' `static_version`; a door animating bumps `leaf_version` and `version` only (AC-182's matrix is the test).

**`version_epoch` (additive).** The GDD resets all counters to 1 on `restore` and then tells consumers to remember to rebuild unconditionally, "because a post-restore `version` can numerically equal a pre-restore cached value". That is a convention a future consumer can forget. `LayoutVersions.version_epoch` (member `_version_epoch`, getter `version_epoch() -> int`) is an `int`, 1 at boot, **incremented by every whole-block replacement** (`restore` and `default_state`, the two calls that reset the counters), never persisted, never hashed, never read by a block. `new_epoch()` also resets every `changed_region` `stamp_tick` to -1, so a stamp from the pre-restore timeline whose tick equals the restored `T` can never be read. At restore every **live** section's three counters are set to 1 and every other section (discarded or never built) reads 0. A consumer caches `(version_epoch, version)`; an epoch change is "rebuild everything" by definition, with no ordering assumption between `post_restore` calls. Consumers may *also* use the kernel's restore signals (`PresentationClock.load_seq`, the `ALL` batch, the Emitter's own `post_restore`); the epoch makes the port self-sufficient. The name is chosen so it is not confused with `restore_epoch` (ADR-0003/ADR-0005, the RNG epoch), which this ADR does not read.

**`changed_region` (simulation-side, current tick only).** Per section: `kind` (`ChangedRegion.NONE 0, ALL 1, REGION 2, FRAME 3`), a `REGION` box (four `int32`, body frame) and a `stamp_tick`. The writers are the same mutators as the bumps, through `stamp_all`, `stamp_region(box)` and `stamp_frame`. **Validity is lazy, not cleared:** a read returns NONE unless `stamp_tick == ClockView.tick()`. `T` advances only in the tick driver's boundary (ADR-0003 § 10), so a stamp written in P2 of tick `T` is read by `emitter.place` in P3 of the same `T`, and goes silent at `T + 1` with no clearing pass. **Merge lattice when one section is stamped twice in a tick** (a socket-host section in LM2 in P2, then its leaf in the P3 hook): `ALL` absorbs everything; `REGION` with `REGION` is the bounding union; `FRAME` with `REGION` is `REGION` (the reader detects a translation by comparing `section_offset`, as the GDD says); `FRAME` with `FRAME` is `FRAME`. The values are the GDD's (build, discard, `post_restore` = ALL; a leaf change = REGION(union of old and new leaf rects); a socket-host section = REGION(union of that socket's PLATE, JAMB and LEAF slot footprints); every other moved section = FRAME). **Presentation must not read it**: a rendered frame can follow up to four ticks, so "this tick" is meaningless there; presentation uses the dirty batch and `version`. ADR-0016 section 1.1 (ruling CT-43) fixes the clock: during R3 and R4 `ClockView.tick()` is the last completed tick recorded in `kernel.clock` and the first RUNNING tick is one more (kernel GDD; ADR-0001's driver boundary). A stamp written by `post_restore` therefore does not match the first RUNNING tick's `T`, so **`changed_region` after a restore is advisory and `version_epoch` is authoritative**, and a consumer must not rely on `changed_region == ALL` after a restore (OQ-1, closed).

### 7. Maintaining the table; other clients of the same grid

**Who writes what.** `RectIndex` columns are written in exactly two places: the rebuild (`rebuild_body`, `release_body`, `rebuild_all`) and `set_extent(section_id, idx, x0, y0, x1, y1)` / `set_empty(section_id, idx)` (both `## @hot`: they run every tick from `layout.doors`), which resolves the row as `row_base_of_section[section_id] + idx` (a `LayoutDerived` column rewritten at every rebuild, so no cached row index can go stale) and writes the four `cx*`/`cy*` cells. The callers of `set_extent` are the `layout.doors` hook (one write per changed leaf, then `bump_leaf`) and `post_restore`'s `derive_doors`. LM2 and LM3 do **not** call `set_extent`: they end in `rebuild_body`, which reads the freshly seeded leaves (LM2 seeds each new LEAF closed, ADR-0013 § 8) and the switched PLATE/JAMB slots from the feed.

**Same-tick truth (TR-shipbody-036).** A weld, cut or discard happens in a P2 apply and ends by rebuilding the affected grids, so every P3 reader sees the new set. A leaf command (`set_opening`) also lands in P2; `layout.doors` (P3, `after: []`) projects it and calls `set_extent` before `body.integrate` (`after: [controller.step, layout.doors]`), so Body reads the **post-cut, post-leaf** set of that tick, and the Emitter, ordered after Body, reads it too. The ordering is ADR-0013/ADR-0001's; this ADR's contribution is that the writes are synchronous and the reads see them with no staging.

**`solid_at(body_id, x, y) -> bool` (new hot read on `LayoutViews`, additive to ADR-0013 § 11).** One cell lookup: `cell = (floor_div(x, c), floor_div(y, c))`, scan that cell's rows, true iff some row with `RectFlags.SOLID` has a current extent containing the cell `(cx0 <= x < cx1, cy0 <= y < cy1)`. Unknown body or a point outside the grid: false. It exists because `clip_to_footprints(body_id, from, to)` (listed by ADR-0013 § 11 as a Class 2 read, used by Interaction's throw and `free(p)`) needs a per-cell test and has no owner for the algorithm. **Decision:** the supercover walk is Interaction's rule (Interaction D.4: every cell the segment touches, both cells at a corner crossing), built on `solid_at`; the return record and the budget belong to ADR-0020 (open question 3). Walking with a bounding-box `solid_in` per step is rejected: one point lookup is cheaper and has no ordering concern.

**`footprint_overlaps(body_id, ax0, ay0, ax1, ay1) -> bool` and `section_footprints_into(section_id, out)` (Layout-internal, not ports).** ADR-0013 § 6's `GEOM_OVERLAP` tests the **static footprint** set (every static rect with all flags, each PLATE slot at its full footprint, no leaf, no JAMB) so a door animating can never flip a mating verdict. That is a *different set* from the query set. The rows carry the `STATIC_MEMBER` class bit and the `fx*/fy*` columns for it: `footprint_overlaps` runs the §3 traversal over the maximum footprints of `STATIC_MEMBER` rows and returns on the first hit; `section_footprints_into` enumerates one section's `STATIC_MEMBER` rows (body frame) so `can_attach` can translate section B's footprints by the candidate offset and probe body A's grid (`n_B x cells` probes, the D.22 figure). Both are pure reads, memoised by `can_attach` on `(socket_a, socket_b, static_version_a, static_version_b)` (ADR-0013 § 6).

**The tile leg of `GEOM_OVERLAP` is not this ADR's (option (a) chosen, ruling CT-16).** ADR-0013 section 6's `GEOM_OVERLAP` set also includes the tile cells of every room, tested rect-vs-tile and tile-vs-tile (two floors on one cell both fail). Floor tiles are not rows in this table and this ADR adds neither a tile row class nor a per-body floor-cell bitmap: `footprint_overlaps` and `section_footprints_into` cover only the `STATIC_MEMBER` rect rows, and the tile-cell leg is Layout's own `floor_tile_at`-based check, run after `footprint_overlaps` returns no rect hit (ADR-0013 section 6). The uncached cost of both legs at the largest MVP body pair is an S1 line (V4) against ADR-0013's `can_attach` target of 2 ms or less; if the rect leg misses, a bulk footprint probe is the first rung to add to section 10.

**What this ADR does not own.** `layout.doors` and the leaf algorithm (ADR-0013 § 8); the rect record, flags and key (ADR-0013 § 5); the integer segment-vs-rectangle test, its doubled-coordinate convention and the Emitter's subject index (ADR-0017); the occluder polygon merge and caps (ADR-0028); the PROJECTOR scene (ADR-0034); Body's swept-clamp (ADR-0015).

### 8. Determinism, derived-state rules and diagnostics

- **Pure function of persisted state.** For a fixed `layout.state`, template catalog and leaf cache, every query result is independent of the grid's cell size, of the order rebuilds happened in and of the history that produced the state (a body built by `build, attach, detach` and the same body restored must answer identically). The corpus test compares the grid against a brute-force scan (`TestBruteScan`, `src/sim/layout/testing/`, ADR-0010 § 4.3) after random `set_extent` sequences and after `rebuild_all`; the ADR-0011/0013 zero-and-diff pattern applies: zeroing every `RectIndex` column and calling `rebuild_all` must leave `state_hash` and every query result unchanged.
- **Nothing here is persisted or hashed.** `RectIndex`, `LayoutVersions`, `version_epoch` and the stamps are in `LayoutDerived`; the allow-list test of ADR-0013 (AC-130) walks only `LayoutState`.
- **Single-threaded.** The only thread in the project is the save worker, which handles a serialised payload; it never touches `RectIndex` (IO-02).
- **Coordinate range.** The grid columns are `PackedInt32Array` and the empty-extent sentinels (`0x3FFFFFFF`) sit near the int32 edge; a packed element write truncates silently (probed 2026-10-29: writing `0x1_0000_0005` stored 5, no error or warning), so a corrupted coordinate could match the sentinel or a wrong cell. `WORLD_COORD_LIMIT` keeps legal coordinates far from the sentinels, and this is now checked: a debug-build diagnostic `coord_out_of_range` tests `abs(v) <= WORLD_COORD_LIMIT` on every `rebuild_body` input and on every `set_extent`, and a **BOOT invariant `WORLD_COORD_LIMIT_BELOW_SENTINEL`** (`WORLD_COORD_LIMIT < EXTENT_EMPTY_LO`, a `CONTENT_INVARIANT` evaluated at ContentLoader stage 4 through `BroadphaseCoupling.sentinel_clear`; registration is an ADR-0007 ask, like `BROADPHASE_CELL_COUPLING`) fails the load otherwise.
- **Diagnostics** (Layout's diagnostics channel, ADR-0013 Implementation Guidelines; counters, never faults, never behaviour): `query_box_empty`, `query_unknown_body`, `query_unknown_section`, `query_buffer_grew`, `broadphase_oversize`, `rows_not_ascending`, `slot_extent_outside_footprint`, `coord_out_of_range`, and ADR-0013's `stale_leaf`. None changes a result.

### 9. How each claim is proven (ADR-0010 § 7 wording)

| Claim | Check |
|---|---|
| "Allocates nothing per call" (TR-section-015) | HOT-01 clean on `RectIndex.query`, `section_rects`, `set_extent`, `set_empty`, `solid_at`, `footprint_overlaps`, both adapters' hot methods and the `LayoutVersions` getters; CNT-01 `OBJECT_COUNT` flat over 600 ticks of queries; CNT-02 `MEMORY_STATIC` within tolerance over 100 warm-up plus 9,000 calls (debug binary); `out.grew == 0` after warm-up; a code review that finds no `.new(`, `PackedXArray(`, literal or `.resize(` inside a tagged function. CNT-01/CNT-02 see only retained growth: `OBJECT_COUNT` and `MEMORY_STATIC` are blind to transient allocation (ADR-0010 P7), so "allocates nothing per call" rests on the HOT-01 token scan (`.slice(`, `.sort_custom(` and `PackedXArray(` are on its list) |
| Ascending key, each once | `QueryBuffer.is_ascending_unique()` over every corpus result; the build-time `rows_not_ascending` check |
| Equal to brute force | AC-39 corpus (1,000 boxes from `SEED_AC39`: negative coordinates, boxes on cell edges, boxes spanning three or more cells) at `BROADPHASE_CELL_PX` minimum and maximum, both masks, after random leaf updates and after `rebuild_all` |
| Cell bound | the bound test with `BroadphaseCoupling` at `cell = max(FOOT_BOX_W, FOOT_BOX_H) + V_RUN` (passes) and one less (fails), over every phase offset of the cell grid |
| Budgets | `cells_touched` and `candidates_tested` closed forms (deterministic, per commit); wall-clock P99 only in the nightly release-export lane (+25 percent regression rule, TR-shipbody-046), never a per-commit gate |

### 10. Fallback ladder (cheapest first; each is a data-layout step behind the same ports)

0. As designed. 1. Tune `BROADPHASE_CELL_PX` within the coupling. 2. Replace the four `IntMath.floor_div` calls per query (0.19 us each, probed) with a per-body lookup table from body-frame px to cell index. 3. Fold the two adapters into the index class to remove the hop; hoist more columns into locals. 4. Replace the stamp-and-insertion de-duplication by mark-and-find for large boxes only: a per-body `PackedByteArray` marking rows, then ascending extraction with `find(1, pos)` **and a `pos >= 0` guard in code and in the corpus test** (a negative start is end-relative and would return a valid wrong index, V1) (probed: byte-identical results to the brute-force scan on the same 2,000-box corpus; 249 us versus 348 us at 443 results). 5. Incremental update of the survivor's grid on a weld instead of a full rebuild (only if V4 misses). 6. Merge collinear abutting wall rects at template compile (fewer rows, same occupied cells; an ADR-0007/0013 compile change, keys change, so a template hash changes). 7. Consumers cache results keyed by `(version_epoch, static_version)` (the Emitter's static layer already does). 8. GDExtension, only after S1 evidence and an accepted amendment, replacing `RectIndex` behind the same ports (ADR-0010 § 9; the GDScript implementation stays the golden reference).

### 11. Probed figures (2026-10-29, 4.7.2 editor binary, headless script, dev machine; NOT a release export and NOT the budget pass)

A synthetic body of 2,000 rects (24-px-thick walls of 48 to 384 px, 7 percent non-SOLID, a few inactive slots, cell 48, 4.9 grid entries per rect, 73 x 41 cells), deterministic LCG placement. Results versus a brute-force scan: 2,000 mixed boxes (22 x 14, 19 x 11, 288 x 288, 96 x 48, 1 x 1, 700 x 700, including boxes partly outside the grid) gave **0 mismatches** for both candidate algorithms (86,989 results).

| Box | Avg results | Stamp + insertion (this ADR) | Mark + find (rung 4) | Brute force |
|---|---|---|---|---|
| 22 x 14 (Body-sized) | 3.7 | 3.1 us | 3.3 us | 104.5 us |
| 288 x 288 (Emitter) | 59 | 34.2 us | 32.2 us | 138.6 us |
| 700 x 700 | 202 | 131 us | 113 us | 169 us |
| 1500 x 1500 | 443 | 348 us | 249 us | 251 us |

`MEMORY_STATIC` delta: 0 bytes over 20,000 small and 2,000 Emitter-sized queries after buffer warm-up, which shows **no retained growth** and nothing more (the counter is blind to transient allocation, ADR-0010 P7). During the 1500 x 1500 run it rose by 38,400 bytes, which is 1,536 records at 25 B per record, more than the 443-result average and so consistent with the probe's `grow` allocating the largest result seen (the growth policy was unspecified then; section 4 now fixes it as geometric). The `stamp` and `cand` scratch arrays are allocated at rebuild, before the baseline, and a query does not grow them; the probe did not separate them from the buffer, so the attribution to the three output columns is UNVERIFIED and S1 reports `MEMORY_STATIC` per structure. Other figures: `IntMath`-style `floor_div` 0.19 us per call; `find` over a 2,000-byte zero array 0.40 us; port call 0.195 us versus direct 0.188 us; table-and-grid build 11.7 ms for 2,000 rects (includes generating and sorting them). Read against the lines, these are **margin evidence, not passes**: 3 us as a share of the 50 us whole-hook per-actor ledger entry (400 us for eight actors, ADR-0015), 8 x 3 us as a share of that 400 us, 34 us against 300 us for the Emitter box, all on a debug-capable binary and a synthetic mix, with the sweep, the adapter hop and the real density excluded. The release-export wall-clock lines stay UNVERIFIED until S1.

### Key Interfaces

```gdscript
# src/sim/layout/views/query_buffer.gd
class_name QueryBuffer extends RefCounted
const OK: int = 0
const EMPTY_BOX: int = 1
const UNKNOWN_BODY: int = 2
const UNKNOWN_SECTION: int = 3
var count: int = 0                      # the only valid length
var status: int = OK
var keys: PackedInt64Array = PackedInt64Array()    # rect_key; capacity >= count
var spans: PackedInt32Array = PackedInt32Array()   # 4 per record: x0, y0, x1 - 1, y1 - 1 (occupied span, body frame)
var flags: PackedByteArray = PackedByteArray()
var cells_touched: int = 0              # D.9 formula, before clamping
var candidates_tested: int = 0          # unique rows tested
var grew: int = 0
func _init(initial_capacity: int) -> void: ...
func clear() -> void: count = 0
func capacity() -> int: ...
func grow(min_capacity: int) -> void: ...            # cold; not @hot; resizes to maxi(min_capacity, capacity() * 2)
func append_record(key: int, x0: int, y0: int, xm1: int, ym1: int, rect_flags: int) -> void: ...   # tests and doubles
func is_ascending_unique() -> bool: ...              # tests and debug assertions

# src/sim/layout/views/region_box.gd
class_name RegionBox extends RefCounted
var x0: int = 0
var y0: int = 0
var x1: int = 0
var y1: int = 0

# src/sim/layout/views/changed_region.gd   (constants only)
class_name ChangedRegion extends RefCounted
const NONE: int = 0
const ALL: int = 1
const REGION: int = 2
const FRAME: int = 3

# src/sim/layout/rect_index.gd   (internal; derived; not a port)
class_name RectIndex extends RefCounted
func setup(cell_px: int) -> void: ...
func rebuild_body(body_id: int, feed: SlotFeed) -> void: ...        # cold
func release_body(body_id: int) -> void: ...                        # cold
func rebuild_all(feed: SlotFeed) -> void: ...                       # cold; restore and tests
## @hot
func set_extent(section_id: int, idx: int, x0: int, y0: int, x1: int, y1: int) -> void: ...   # layout.doors, post_restore; debug coord_out_of_range check
## @hot
func set_empty(section_id: int, idx: int) -> void: ...
## @hot
func query(body_id: int, ax0: int, ay0: int, ax1: int, ay1: int, mask: int, out: QueryBuffer) -> int: ...
## @hot
func section_rects(section_id: int, mask: int, out: QueryBuffer) -> int: ...
## @hot
func solid_at(body_id: int, x: int, y: int) -> bool: ...
## @hot
func footprint_overlaps(body_id: int, ax0: int, ay0: int, ax1: int, ay1: int) -> bool: ...
func section_footprints_into(section_id: int, out: QueryBuffer) -> int: ...   # STATIC_MEMBER rows, fx*/fy*

# src/sim/layout/layout_versions.gd   (internal; writers are Layout only)
class_name LayoutVersions extends RefCounted
var _version_col: PackedInt64Array = PackedInt64Array()   # private prefix: version/static_version/leaf_version are method names
var _static_col: PackedInt64Array = PackedInt64Array()
var _leaf_col: PackedInt64Array = PackedInt64Array()
var _version_epoch: int = 1
## @hot
func version_of(section_id: int) -> int: ...          # guard: section_id < 0 or >= _version_col.size() -> 0
## @hot
func static_version_of(section_id: int) -> int: ...   # same guard
## @hot
func leaf_version_of(section_id: int) -> int: ...      # same guard
## @hot
func version_epoch() -> int: ...
## @hot
func changed_region_kind_of(section_id: int, tick: int) -> int: ...      # NONE unless stamp_tick == tick
## @hot
func changed_region_box_into(section_id: int, tick: int, out: RegionBox) -> bool: ...
func bump_static(section_id: int) -> void: ...
func bump_leaf(section_id: int) -> void: ...
func invalidate(section_id: int) -> void: ...
func stamp_all(section_id: int, tick: int) -> void: ...
func stamp_region(section_id: int, x0: int, y0: int, x1: int, y1: int, tick: int) -> void: ...
func stamp_frame(section_id: int, tick: int) -> void: ...
func new_epoch() -> void: ...                       # restore() and default_state(); _version_epoch += 1 and every stamp_tick = -1

# src/sim/layout/layout_solid_query.gd / layout_occluder_source.gd   (the adapters; one-line wrappers)
class_name LayoutSolidQuery extends SolidQuery
class_name LayoutOccluderSource extends OccluderSource

# src/shared/layout/broadphase_coupling.gd   (pure; ADR-0007 § 1.9)
class_name BroadphaseCoupling extends RefCounted
const EXTENT_EMPTY_LO: int = 0x3FFFFFFF
const EXTENT_EMPTY_HI: int = -0x3FFFFFFF
static func holds(cell_px: int, foot_box_w: int, foot_box_h: int, v_run: int) -> bool: return cell_px >= maxi(foot_box_w, foot_box_h) + v_run
static func sentinel_clear(world_coord_limit: int) -> bool: return world_coord_limit < EXTENT_EMPTY_LO   # WORLD_COORD_LIMIT_BELOW_SENTINEL
```

(Each `class_name` is its own file. Signatures are the contract; bodies are the Layout epic's. `SlotFeed` is a Layout-internal helper, defined with the Layout epic's rebuild story.)

**Notes on this block.** (1) It is pseudo-code: `: ...` marks an elided body and is a parse error if pasted; stories write `pass` or a `push_error` stub. The `@abstract func` declarations without bodies are valid. (2) Parameter, local and member names in these classes must not shadow any method of the same class (`SHADOWED_VARIABLE=2` includes functions): hence `initial_capacity` next to `capacity()`, the private `_version_col`/`_static_col`/`_leaf_col` next to the port methods `version`/`static_version`/`leaf_version`, and no local named `version`, `count` or `capacity` inside the adapters or `LayoutVersions`. The other listed classes were checked for the same pattern. (3) `grow`, `clear`, `capacity`, `count`, `status`, `keys`, `query`, `setup`, `set_extent` and `set_empty` do not collide with `Object` or `RefCounted` methods (engine review).

### Implementation Guidelines

- **Must** keep exactly one row table per LIVE body, ordered by ascending `(section_id, idx)`, with the SOLID and OCCLUDES_LIGHT queries as flag masks over it; **must never** keep a second rect copy, a per-consumer index, a "perceived" or phantom variant of any query, or an engine collision, tile-collision, navigation or occluder-layer substitute (no `PhysicsServer2D`, `PhysicsDirectSpaceState2D`, `TileMapLayer`, `TileSet` physics/occlusion/navigation layer or `LightOccluder2D` as a rect source; the occluder nodes are Lighting's and are built *from* this set).
- **Must** treat `spans` as occupied spans and convert to half-open in exactly one place (`x1 = spans[4k + 2] + 1`, `y1 = spans[4k + 3] + 1`), and put `SolidQuery` behind `CollideSource` / `LayoutCollide` only; polygon builders place corners at `(x0, y0)` and `(x1_occ + 1, y1_occ + 1)` with no shift from Layout; **must never** read `spans[4k + 2]` or `spans[4k + 3]` as a half-open bound.
- **Must** take `body_id` and four half-open integers on both ports and return the record count; **must never** take or return `Rect2i`, `Rect2`, `Vector2`, an `Array` or a `Packed*Array` from a query, and **must never** return a shared or reused buffer.
- **Must** overwrite the caller's `QueryBuffer` on every query, treat `count` as the only valid length, take local aliases of the three columns after the capacity check and write by index; **must never** `append`, `resize`, `append_array` or construct an array, `Dictionary` or object inside a `## @hot` function, and **must never** truncate a result: overflow goes to `QueryBuffer.grow`, which is cold, counted (`grew`, `query_buffer_grew`) and expected to stay at 0 after warm-up.
- **Must** size each consumer's buffer for the worst result it can receive (default `MAX_FOOTPRINT_RECTS_PER_BODY` records); **must never** keep `out.keys`, `out.spans` or `out.flags` beyond the call that filled them, read at or beyond `count`, or share one buffer between two results that must both stay live.
- **Must** return an empty or inverted box as 0 with `status = EMPTY_BOX`, an unknown body as 0 with `status = UNKNOWN_BODY` and an unknown section from `section_rects_into` as 0 with `UNKNOWN_SECTION`; **must never** raise a fault or throw for any of them, and callers **must** read `status` when "no rectangles" and "no such body" mean different things (Horror, TR-horror-031).
- **Must** compute every cell index with `IntMath.floor_div` (`ax1 - 1` for the high end) and clamp to the grid after counting `cells_touched`; **must never** use `/`, `%`, `/=`, `%=`, a float or an engine geometry type in `src/sim/layout/`.
- **Must** insert every row (including leaf, PLATE and JAMB slots) at its maximum footprint at rebuild and change a leaf only through `RectIndex.set_extent` / `set_empty`; **must never** write a `cx*`/`cy*`/`fx*`/`fy*` cell, `cell_start` or `cell_items` outside `rebuild_body`, `release_body`, `rebuild_all`, `set_extent` and `set_empty`, never set an extent outside the row's footprint, and never keep a separate dynamic list.
- **Must** represent "no current rectangle" as the `EXTENT_EMPTY_LO/HI` sentinel; **must never** add an `active` branch to the hot loop.
- **Must** call `rebuild_body` / `release_body` as the last step of LM1, LM2, LM3, LM4 and `restore`, inside the same apply, so P3 readers see the new set; **must never** rebuild from a hook, a query, a notification handler or presentation, and never rebuild lazily on first query.
- **Must** bump counters only through `LayoutVersions` from the closed caller list of § 6, bump `static_version` for every section whose socket state changes including the unmoved host and for every moved section, bump `leaf_version` at most once per section per tick and never `static_version` from `layout.doors`, and `invalidate` (all three to 0) on discard; **must never** persist, hash or snapshot a counter, a stamp or `version_epoch`, and never reuse the value 0 for a live section.
- **Must** increment `version_epoch` on `restore` and `default_state`; consumers **must** cache `(version_epoch, version)` and rebuild when the epoch differs; presentation **must never** read `changed_region` and the simulation **must never** read it outside the tick it was stamped (the read returns NONE otherwise).
- **Must** merge two stamps of one section in one tick by the lattice of § 6 (ALL absorbs; REGION union; FRAME with REGION is REGION); **must never** clear stamps with a pass.
- **Must** evaluate `BROADPHASE_CELL_COUPLING` (`BroadphaseCoupling.holds`) at BOOT and fail the load with a `CONTENT_INVARIANT`; **must never** treat it as an advisory range or clamp the cell size.
- **Must** express both ports as `@abstract` classes in `src/sim/layout/ports/`, implement them as the two adapters, type every consumer's `setup()` parameter as the port and double them with hand-written classes in `tests/support/`; **must never** let one class extend both ports, name `RectIndex` outside `src/sim/layout/`, or inject a Layout internal where a port is expected.
- **Must** tag `RectIndex.query`, `section_rects`, `set_extent`, `set_empty`, `solid_at`, `footprint_overlaps`, the adapters' hot methods and the `LayoutVersions` getters (`version_of`, `static_version_of`, `leaf_version_of`, `version_epoch`, `changed_region_kind_of`, `changed_region_box_into`) `## @hot`; `section_footprints_into` stays untagged (called only from `can_attach` validation); **must never** claim "zero allocation" without HOT-01, CNT-01 and CNT-02.
- **Must** let a `version_epoch` change trigger only a cache rebuild whose result is a pure function of persisted state (an epoch change gates only result-neutral cache rebuilds); **must never** feed it into a work budget, a persisted value or an RNG draw, so a restored run cannot diverge from an uninterrupted one (ADR-0017 D11.4 records the cold rebuild at `post_restore`, ruling CT-45).
- **Must** keep parameter, local and member names from shadowing any method of the same class, and name `LayoutVersions` columns with the private prefix; **must never** write a constant negative operand to `>>` or `<<` (a parse error: `Invalid operands for bit shifting. Only positive operands are supported`). A power-of-two cell-size shortcut for negative body-frame coordinates (fallback rung 2) uses a variable operand or `IntMath.floor_div`, not a literal; key decode `>> 12` is safe because keys are positive.
- **Must** treat `PackedXArray` locals and members of script classes as shared by reference (`QueryBuffer` and `BodyGrid` are script classes, so the aliasing is safe) and know that they copy on assignment from native properties; **must never** pass `out.keys`, `out.spans` or `out.flags` to a native method that takes the array by value inside a hot function.
- **Must** keep `RectIndex`, `LayoutVersions` and `QueryBuffer` out of every snapshot, hash and replication summary.

## Alternatives Considered

### Alternative 1: Engine physics queries (`PhysicsDirectSpaceState2D`, shape casts, `CharacterBody2D`, `Area2D`)
- **Description**: Make Layout rectangles collision shapes and ask the physics server for overlaps and rays.
- **Pros**: No index to write; the engine owns the broadphase.
- **Cons**: The 2D physics engine makes no determinism claim (CR-13 G3), so a physics-derived `lit` or a clamped position would flow into persisted blocks as an unrecorded observation; a direct-space-state query is valid only during the physics step while the kernel allows these reads from presentation at render rate and the tick is driven from `_process`; the simulation must stay scene-tree free and headless (SIM-01, SIM-06); the pinned engine reference carries no 2D query API (`modules/physics.md` has only a 3D example), so the call shapes cannot be sourced.
- **Rejection Reason**: Violates determinism, tree-freedom and the registry's `physics_callback_tick_source`/`scene_tree_or_engine_server_in_sim_layers`; TR-section-011 forbids it.

### Alternative 2: `TileMapLayer`/`TileSet` physics, occlusion or navigation layers as the rect source
- **Description**: Bake walls into the PROJECTOR's tile layers and read collision, light occluders and navigation from them.
- **Pros**: The renderer already has the tiles; occluders come free from the TileSet.
- **Cons**: A `TileSet` occluder layer carries one light mask for the whole layer (the per-node masks and the per-body mask lend of LO-24 need per-node properties, UNVERIFIED); the data would live in the scene tree, outside the headless simulation; door leaves with arbitrary moving edges are not tiles; two representations of one wall can disagree.
- **Rejection Reason**: TR-section-011 and TR-light-035 forbid it; the PROJECTOR and the occluder builder draw *from* the rect set.

### Alternative 3: One grid per section
- **Description**: Index each section independently in section-local coordinates.
- **Pros**: A weld or cut rebuilds only a section; no body-frame translation in the index.
- **Cons**: A body query then fans out to every section it may touch and merges their key-ordered results, and each result needs the section offset applied; Body's single swept box would routinely straddle two sections at a hatch; the merge returns the cost this ADR removes.
- **Rejection Reason**: Per-body is the unit consumers query (`solid_in(body_id, ...)`); one table gives one ordered pass.

### Alternative 4: Room-gated broadphase (candidate 4's other option)
- **Description**: Index rects by room and answer a query from the rooms its endpoints or box lie in.
- **Pros**: Naturally small candidate sets inside a room.
- **Cons**: A segment test crosses rooms and doors; hull walls and door sills are not inside a room's tile set; an actor in a doorway belongs to two rooms or none (`room_at` answers a sill's `sill_room`, so no walkable point is NONE only because of a special rule); light and collision consumers would need different gating; the gate depends on tile data that restore rebuilds later than the rects.
- **Rejection Reason**: A uniform grid over rectangles is geometry-only and identical for every consumer; the Emitter GDD itself leaned to an index in Layout.

### Alternative 5: An Emitter-owned spatial index
- **Description**: The Emitter copies rects it cares about into its own structure.
- **Pros**: Tuned to its segment tests.
- **Cons**: A second copy that can drift from the one Body collides with (two truths about a wall, a Pillar 1 violation), double memory, a second rebuild on every weld; the Emitter's *subject* index (which emitters and subjects to test) is a different thing and stays the Emitter's (ADR-0017).
- **Rejection Reason**: One rect set (TR-section-011); the Layout GDD (LY-12 c) already rejected this.

### Alternative 6: A BVH, quadtree or sweep-and-prune instead of a uniform grid
- **Description**: A tree or sorted-axis structure over rects.
- **Pros**: Adapts to non-uniform density.
- **Cons**: Rects are 24-px-thick walls on a 24-px lattice, close to uniform; tree traversal in GDScript is pointer chasing through arrays with a high per-node constant; building and ordering deterministically is harder; a grid query is index arithmetic.
- **Rejection Reason**: No density problem the grid cannot meet; cost is measured, not assumed (§ 11).

### Alternative 7: Dynamic leaf and variant slots in a separate linear list (the GDD D.22 model)
- **Description**: Keep the static rects in the grid and scan a short list of leaf, PLATE and JAMB slots on every query, merging hits into the key-ordered output.
- **Pros**: The grid never contains a rect that can be absent.
- **Cons**: A scan and a merge on every query (`N_dyn x t_bbox + t_merge_dyn`), up to `MAX_DOORS_LOADED` entries as a stress bound; a second ordering path that must interleave with the grid's.
- **Rejection Reason**: Putting the slots in the grid at their maximum footprint costs one row each and nothing per animation tick, keeps order structural and deletes the terms.

### Alternative 8: Incrementally update the survivor's grid on a weld or cut
- **Description**: Insert the absorbed body's rows into the survivor and remove a detached component's rows without a full rebuild.
- **Pros**: A smaller P2 cost for a large body.
- **Cons**: Rows must be renumbered to stay in key order (a merge of two ascending sequences changes every later row's number) and every cell list rewritten; the code path is rare and hard to test against the rebuild oracle.
- **Rejection Reason**: Welds are rare P2 events; the full rebuild is the simplest correct path and the oracle for any later incremental one. It is rung 5 of the ladder if V4 misses.

### Alternative 9: Value-returning queries, or a shared reused buffer returned by Layout
- **Description**: `rects_in(...) -> PackedInt32Array` (or an `Array` of rect objects), or one internal buffer that every caller receives a reference to.
- **Pros**: A simple call site.
- **Cons**: A value return allocates per call; a shared returned buffer aliases between Body, the Emitter, Lighting and Interaction within one tick, and cannot be both allocation-free and safe under copy-on-write (LY-12 e).
- **Rejection Reason**: Caller-owned out-parameters (AC-151). A value-returning copy may exist for tools and tests only.

### Alternative 10: A per-body bound `OccluderSource` handle (the GDD's `occluder_source(body_id)`)
- **Description**: `occluder_source(body_id)` returns an object bound to that body.
- **Pros**: The call sites match the GDD wording; `rects_in(aabb)` needs no `body_id`.
- **Cons**: An object per body (created at every detach, dissolved at every weld), consumers that cache handles across a weld, and a stale handle that silently reads a dissolved body; it makes the two ports different shapes.
- **Rejection Reason**: Explicit `body_id` is stateless and the same on both ports; the GDD wording is synchronised (§ Migration Plan).

### Alternative 11: `Rect2i` boxes
- **Description**: Keep `aabb: Rect2i` as in `architecture.md`.
- **Pros**: One argument; matches the architecture text.
- **Cons**: See § 5: an inverted box is a negative size with no project contract, engine geometry type in a module that bans them, SIM-04 blind to integer-division inside helpers, and inconsistent with every other Layout hot read.
- **Rejection Reason**: Four scalars are cheaper to read in GDScript, unambiguous for the inverted-box guard and consistent; `architecture.md` is synchronised.

### Alternative 12: One fat port (`LayoutGeometry`) or one facade extending both
- **Description**: All queries and counters in a single abstract class.
- **Pros**: One injection.
- **Cons**: Body would see versions and change stamps, doubles grow to the whole surface, and GDScript single inheritance prevents a facade from extending two ports in any case.
- **Rejection Reason**: Least privilege and small doubles; two thin adapters.

### Alternative 13: Order the output by sorting (`sort_custom`, per-call native sort of candidate keys)
- **Description**: Collect candidates in cell order and sort them by key.
- **Pros**: Simple to state.
- **Cons**: `.sort_custom(` is banned in `## @hot` (HOT-01) and by the DET rules in `src/sim`; native `sort()` sorts a whole array, so it needs an exact-size array and therefore a `resize` per call; row order already equals key order.
- **Rejection Reason**: Order is structural (§ 1); only the unique-candidate insertion remains.

### Alternative 14: Persist the grid or the counters in `layout.state`
- **Description**: Snapshot the broadphase and versions so a restore need not rebuild.
- **Pros**: No rebuild at restore.
- **Cons**: Derived data in the persisted block (TR-section-004, TR-section-017), a block-size and hash-cache cost on every door tick, and a fixed cell size baked into saves.
- **Rejection Reason**: Rebuild at restore is cold; counters reset to 1 and the epoch advances.

## Consequences

### Positive
- One rectangle set, one ordered result shape and one counter contract for four consumers; collision, light and drawing cannot disagree (Pillar 1) and no query has a perceived variant.
- Door animation costs one four-cell write per changed leaf and no grid change; the dynamic-list scan and merge of the GDD's cost model are gone.
- A caller-owned buffer with a `count`, a `status` and two deterministic work counters makes the allocation, ordering, empty-box and unknown-body behaviours testable without wall-clock or an injected counter.
- `version_epoch` closes the restore aliasing hazard in the type, not in a convention; unknown sections read 0, so a discard is a visible change.
- The two ports are small, hand-doubleable and carry the GDExtension seam of ADR-0010 § 9 unchanged.

### Negative
- Public `Packed*Array` columns on `QueryBuffer` need an explicit ADR-0010 exemption (§ Foundation asks); until it lands, ARCH review may flag them under the prose of CNT-03.
- The signatures differ from the GDDs and `architecture.md` (explicit `body_id`, four ints, `-> int`); five documents need synchronising (§ Migration Plan).
- A full table-and-grid rebuild at every weld, cut and restore: cold, but proportional to the body's rect count (V4).
- Insertion-based ordering is quadratic in the result count; acceptable at the measured sizes, with a named linear rung.
- Two counters (`version_epoch` and the kernel's restore signals) can both be used; the port is self-sufficient but the redundancy must be documented in each consumer.
- Footprint and extent columns double the per-row integer memory (about 50 bytes per row; about 100 KB for a 2,000-row body).

## Risks
- **R1 GDScript cost on a release export and RM-DECK (V3).** Mitigation: S1 gates acceptance; the ladder of § 10; the probe margin is about an order of magnitude on the per-actor line.
- **R2 Weld P2 hitch at a large body (V4).** A 2,000-row rebuild probed at about 11.7 ms on a debug-capable binary against `STEP_BUDGET_MS` as an alarm. Mitigation: the MVP loaded set is a few hundred rects per body; ADR-0013 OQ-1 recommends a world lint and later `MAX_SECTIONS_PER_BODY`; rung 5 is incremental update.
- **R3 A leaf update path is missed** (LM2 seeding, `post_restore`, a future mutator) and the table disagrees with the leaf cache: a see-through-but-solid wall. Mitigation: a single write point (`set_extent`), the brute-force equivalence corpus run after random update sequences, and the guideline that LM2/LM3 end in a rebuild rather than patching.
- **R4 Cell-coupling violated by a retune** (a larger `FOOT_BOX_W`, a smaller cell). Mitigation: the BOOT invariant and the bound test; correctness does not depend on it, only the cost bound.
- **R5 Consumer misuse of the buffer** (reading past `count`, keeping a column across calls, sharing a buffer). Mitigation: documentation here, `FakeSolidQuery`/fixture tests in each consumer ADR, and a lint proposal (§ Foundation asks); a lexical check cannot see aliasing, so the corpus interleave test (AC-151) is the backstop.
- **R6 Presentation reads `changed_region`** and misses a bump across a multi-tick frame. Mitigation: the port documents it as simulation-side and returns NONE outside the stamped tick; Lighting and the PROJECTOR key on `(version_epoch, version)`. Because the same port instance is handed to presentation (split by consumer need, not by layer), the rule is backed by a requested ADR-0010 READ-03 token ban on `changed_region_` under `src/presentation/` (§ Foundation asks in the structured result).
- **R7 `version_epoch` mistaken for `restore_epoch`.** Mitigation: the name, the explicit statement that it is never read by a block or the RNG, and the registry entry.
- **R8 Grid bounding-box blow-up for a sparse body** (a far-flung section). Mitigation: `broadphase_oversize` diagnostic and the world lint; not reachable in the MVP roster.
- **R9 Insertion cost on very large boxes** (hundreds of results). Mitigation: rung 4; no consumer in the MVP queries such a box through the grid (Lighting uses `section_rects_into`).
- **R10 The first post-restore step may not observe the stamped `T`**, so `changed_region == ALL` after restore is not guaranteed (OQ-1, closed: advisory after restore per ADR-0016 section 1.1). Mitigation: consumers use `version_epoch`.

## GDD Requirements Addressed

TR text is `docs/architecture/tr-registry.yaml`. Section numbers are this ADR's.

### Decided here (5)

| TR-ID | Requirement | How this ADR addresses it |
|---|---|---|
| TR-section-011 | One rect set serves collision, light occlusion and drawing; no perceived or phantom variant; no physics body, tile collision or nav mesh | § 1 one row table with `SOLID`/`OCCLUDES_LIGHT` as masks and `section_rects_into` for drawing; Alternatives 1, 2, 5; Implementation Guidelines first bullet |
| TR-section-015 | `rects_in`/`solid_in` fill a caller-owned `QueryBuffer` (int64 keys, int32 occupied spans, flags byte), ascending key, allocation-free over 9,000 calls | § 3 query, § 4 `QueryBuffer` (public columns, `count`, grow outside the tag), § 9 HOT-01 + CNT-01 + CNT-02 + `grew == 0`; probe: no retained MEMORY_STATIC growth (allocation-free rests on HOT-01) |
| TR-section-016 | Per-body derived uniform-grid broadphase, never persisted, rebuilt at build/attach/detach/restore; P99 lines | § 2 dense CSR grid, rebuild events and call order, § 8 derived rules; pass lines pending S1 (V3); probed margin in § 11 |
| TR-section-017 | `version`/`static_version`/`leaf_version` monotone, non-persisted, reset to 1 on restore, bumped for both sections of a weld/cut and every moved section; `changed_region` valid only for the current tick | § 6: int64 columns, closed bump list, `invalidate` on discard, `version_epoch`, lazy tick-stamped `changed_region` with a merge lattice |
| TR-shipbody-037 | `BROADPHASE_CELL_PX >= FOOT_BOX_W + 3` boot-checked so a swept foot box touches at most 4 cells | § 2 cell coupling: the `+ 3` is `V_RUN`; BOOT invariant `BROADPHASE_CELL_COUPLING`, `BroadphaseCoupling.holds`, the bound test |

### Shared: this ADR decides its half (7)

| TR-ID | Requirement | This ADR's half | Other half |
|---|---|---|---|
| TR-shipbody-036 | `solid_in` returns unique rects in ascending key order; the sweep tolerates empty/duplicate/unordered input with a debug-only `diag.solid_in_unordered`; the set read in `body.integrate` is post-cut, post-leaf | § 1 and § 3 structural order and de-duplication; `is_ascending_unique()`; § 7 same-tick truth | ADR-0015 (tolerant sweep, the debug signal, one query per actor-tick) |
| TR-light-014 | The occluder builder builds `LightOccluder2D` from `OccluderSource` rects with `version(section_id)` change detection and an unconditional rebuild on restore, within `OCCLUDER_POLYS_VIEW_MAX`, `OCCLUDER_VERTS_MAX`, `OCCLUDER_SYNC_BUDGET_US` | § 5 `section_rects_into` and the counters, § 6 `(version_epoch, version)` as the unconditional-rebuild signal | ADR-0028 (polygon merge, caps, the budget, scheduling) |
| TR-light-035 | Door occluders are derived continuously from `area`/`area_max`; only `OCCLUDES_LIGHT` rects produce occluders (never actors or horrors); occluders are not a TileSet layer | § 1 flag mask, no actor rows, partial leaf rects returned as they are (no threshold), Alternative 2, Guideline 1 | ADR-0028 / ADR-0034 (no TileSet occluder layer in the scene shape); ADR-0013 § 8 (continuous leaf width) |
| TR-derelict-035 | Merged `OCCLUDES_LIGHT` polygons stay within `OCCLUDER_POLYS_VIEW_MAX` (S4 80) at at most `OCCLUDER_VERTS_MAX` vertices; at most `TRAIL_LIGHT_REQUESTS_MAX` glowing faults per compartment | § 5: the exact `OCCLUDES_LIGHT` set in ascending key order is the deterministic merge input | ADR-0028 (merge, caps), Failure & Trail (glow requests), Derelict lints |
| TR-emitter-018 | Occlusion is an integer segment-vs-rectangle test over Layout opaque footprints behind an `OccluderSource` seam (`rects_in`, `version`); no physics backend; actors never occlude; door state from Layout's projection | § 5 the seam, occupied-span output, `OCCLUDES_LIGHT` mask, no actor rows, leaf state from `layout.doors` | ADR-0017 (the doubled-coordinate closed-box segment test, anchor clearance) |
| TR-emitter-046 | Exposure evaluation uses broadphase and radius-survivor rejection, not a linear subject scan | § 2 and § 3: the **rect** index is Layout's (answers the Emitter's candidate 4 first half); `cells_touched`/`candidates_tested` make the `rects_in` cost a closed form | ADR-0017 (the subject index, radius survivors, work-unit budget B, the D.14 static-rebuild policy and its rejected alternative (b)) |
| TR-horror-031 | Horror reads Layout only through published queries with per-run budgets and treats NONE or UNKNOWN as neither unlit nor open | § 3 and § 4: pure idempotent queries, explicit `status` (`UNKNOWN_BODY`, `EMPTY_BOX`) so "no rectangles" is distinguishable from "no such body" | ADR-0013 § 11 (other reads), ADR-0023 (the per-run budgets of 5/1) |

### Owned elsewhere (loaded or checked here only) (10)

| TR-ID | Owner | What this ADR does |
|---|---|---|
| TR-emitter-044 | ADR-0017 (re-tagged 2026-10-29 from ADR-0007 + ADR-0014) | Nothing: anchor clearance behaviour is Emitter runtime semantics. The summary table row at `requirements-traceability.md` line 1343 still reads `ADR-0007 + ADR-0014` (§ Open Questions 5) |
| TR-shipbody-018 | ADR-0015 | One `solid_in` per moving actor per tick is Body's rule; the query is built so one call with the swept box is the cheap path (§ 2 cell bound) |
| TR-emitter-021 | ADR-0017 | Body frame per `body_of`, re-read when `static_version` changes: the counters and `changed_region` are supplied (§ 6) |
| TR-emitter-040 | ADR-0017 | Weld and cut call no Emitter mutator: `static_version` bump plus `changed_region` supply the signal (§ 6) |
| TR-section-010 | ADR-0013 | The rect record, flags byte and `idx` space are consumed as defined (§ 1) |
| TR-section-012 | ADR-0013 | `layout.doors` re-derives leaves; this ADR supplies `set_extent` as its single write into the table (§ 7) |
| TR-section-021 | ADR-0013 | Weld offsets and the detach re-base are consumed; the rebuild after them is this ADR's (§ 2) |
| TR-section-034 | ADR-0013 | LM2 seeds the new LEAF closed; the rebuild reads that seed (§ 7) |
| TR-section-042 | ADR-0013 | `layout.doors` cost and one-bump rule: `bump_leaf` is the counter write (§ 6) |
| TR-interact-027 | ADR-0020 | `clip_to_footprints` return record and Interaction's supercover walk; this ADR supplies `solid_at` (§ 7) |

## Performance Implications
- **CPU**: query cost is `t_call + t_adapter + cells * t_cell + entries * t_stamp + c * (t_test + t_insert + t_emit)`; probed 3.1 us for a Body-sized box and 34 us for an Emitter-sized box on a debug-capable binary (V2); lines pending S1 on release export: `solid_in` is a share of ADR-0015's whole-hook `body.integrate` entry (0.05 ms per actor P99, 0.4 ms for eight; S1 reports it as a sub-counter), and the Emitter box (0.3 ms) is charged inside ADR-0017's `EXPOSURE_BUDGET_MS`. `layout.doors` adds one `set_extent` (four array writes) per changed leaf. Rebuild: cold, linear in the body's row count and entries (about 11.7 ms at 2,000 rows, an upper bound; V4).
- **Memory**: about 50 bytes per row for the table plus the cell lists (about 5 entries per rect at 4 bytes) plus the scratch arrays (`cand`, `stamp`): roughly 160 KB for a 2,000-row body; `MAX_BODIES` bodies share the world's rows, not 8x that. Each consumer's `QueryBuffer` is about 25 bytes per record of capacity (about 50 KB at `MAX_FOOTPRINT_RECTS_PER_BODY`).
- **Load Time**: one `rebuild_all` at restore or new game, once.
- **Network**: none; derived data is not replicated (`SHARED_SUMMARY` carries the instance table, ADR-0013).

## Migration Plan
There is no Layout code yet. Story order: (1) `QueryBuffer`, `RegionBox`, `ChangedRegion`, the two ports, `FakeSolidQuery`/`FakeOccluderSource` and `TestBruteScan`; (2) `BodyGrid`/`RectIndex` rebuild and the box query with the AC-39 corpus and the equivalence oracle; (3) `LayoutVersions` with the bump matrix and `version_epoch`; (4) `set_extent` wired to `layout.doors` and `post_restore`, LM1 to LM4 ending in `rebuild_body`; (5) `solid_at`, `footprint_overlaps`, `section_footprints_into` and `section_rects_into`; (6) `BroadphaseCoupling` and the BOOT invariant; (7) the S1 benchmark harness lines (query, aggregate, Emitter box, rebuild).

**GDD and architecture sync** (listed in the structured result; applied by the orchestrator): `architecture.md` § Read views (the two signatures) and § Interface types (add `RegionBox`, `ChangedRegion`); Layout GDD LY-12 a (one `OccluderSource` with explicit `body_id`), LY-12 d (`version_epoch`, `invalidate` on discard, tick-stamped `changed_region`), LY-12 e (overwrite, `count`, `status`, `cells_touched`, `candidates_tested`), D.22 (dynamic slots in the grid; the swept box is Body's D.12 box, not `[px - 8 - v, px + 8 + v)`), the Interactions rows for `occluder_source(body_id)` / `solid_in` and `clip_to_footprints`; Emitter EM-20 and EM-23, Lighting LO-11 and Body BDY-15 wording (`occluder_source(body_id)`, `rects_in(aabb)`; every `solid_in_into` becomes `solid_in`, with the occupied-span conversion of section 4 in ADR-0015); Body Tuning Knobs and AC-BDY-75 (`+ V_RUN`, not `+ 3`); `design/registry/entities.yaml` (the `FOOT_BOX_W + 3` coupling comment, line 9774: `+ V_RUN`); `tr-registry.yaml` TR-shipbody-037 and TR-section-016 text; `requirements-traceability.md` line 1343.

## Validation Criteria
- **Equivalence**: the AC-39 corpus (1,000 boxes from `SEED_AC39`, asserted by count first) returns exactly the brute-force result for both masks at `BROADPHASE_CELL_PX` published minimum and maximum, including negative coordinates, cell-edge boxes and boxes spanning three or more cells; again after random `set_extent` sequences, after `rebuild_all`, and after a build-attach-detach history versus the same body restored.
- **Empty/inverted/outside**: AC-173's three boxes return 0 with `EMPTY_BOX`, `cells_touched` 0; a box outside the grid returns 0 with `OK`; an unknown or dissolved body returns 0 with `UNKNOWN_BODY` and raises one diagnostic.
- **Order and uniqueness**: `is_ascending_unique()` holds on every corpus result; a rect spanning several cells appears once.
- **Interleaving**: AC-151 with two consumers, two buffers, interleaved boxes: each buffer equals its isolated result.
- **Allocation**: HOT-01 clean on the tagged functions; `OBJECT_COUNT` flat over 600 ticks of queries and leaf updates; `MEMORY_STATIC` within tolerance over 100 warm-up plus 9,000 calls (AC-137), `grew == 0` after warm-up; a deliberately small buffer shows `grew > 0` and an unchanged result (no truncation).
- **Counters**: the bump matrix (weld: both seam sections and every moved section `static_version` up; cut likewise; leaf change: `leaf_version` and `version` only, `static_version` 0 times; discard: all three 0; restore: every LIVE section's three counters 1, every other section 0, `version_epoch` + 1, every `changed_region` `stamp_tick` -1; a restored open door is 1 to 2); `changed_region` values and the merge lattice (AC-182); a stamp from tick `T` reads NONE at `T + 1`; an unknown section reads 0/NONE.
- **Same-tick truth**: a leaf closed by a P2 command is solid for `body.integrate` in that tick; a weld applied in P2 is visible to P3 readers of that tick (integration, with ADR-0015/0017/0021).
- **Coupling**: `BroadphaseCoupling.holds` passes at `max(FOOT_BOX_W, FOOT_BOX_H) + V_RUN` and fails one below; a swept foot box at every `|step| <= V_RUN` and every cell-grid phase touches at most 4 cells.
- **Ports**: `REFLECT-SIG` over both doubles; `FakeSolidQuery extends SolidQuery` drives ADR-0015's `LayoutCollide`, and a contract test runs `LayoutCollide` against the real `LayoutSolidQuery` (including a 1-px-wide leaf fixture that proves the `+ 1` span conversion); no script under `src/sim/layout/` names a `Node`, `TileMapLayer`, `TileSet`, `Rect2i` or a float.
- **Coordinate range**: a `rebuild_body` or `set_extent` input beyond `WORLD_COORD_LIMIT` raises `coord_out_of_range` in a debug build; `BroadphaseCoupling.sentinel_clear` fails when the limit reaches the sentinel.
- **S1 (gates acceptance)**: release-export P99 on RM-DESKTOP and RM-DECK for the three lines (`solid_in` as a sub-counter of `body.integrate`, the Emitter box, the rebuild) and the uncached `GEOM_OVERLAP` cost, both legs, at the largest MVP body pair against ADR-0013's `can_attach` 2 ms target; if a line misses, the ladder is walked before a budget is renegotiated.

## Related
- ADR-0001 (P2/P3, hook order, composition order), ADR-0003 (`ModuleFacet`, `ClockView`, hot-tier rules), ADR-0004 (no new key family here; `SEC`/`BODY`/`SEAM`/`LEAF` are ADR-0013's), ADR-0005 (`IntMath`), ADR-0007 (`LayoutTuning`, BOOT invariants, shared pure helpers), ADR-0009 (presentation reads committed state through injected ports), ADR-0010 (ports, HOT-01, CNT-01 to CNT-03, ladder), and `design/gdd/ship-section-and-layout-data-model.md` (owner), `ship-body-and-interior-physics.md`, `emitter-state-and-light-exposure-query.md`, `2d-lighting-and-occlusion.md`, `horror-entities.md`.
- Siblings: ADR-0013 (rect record, keys, leaf hook, LM1 to LM4, read-view tiers), ADR-0015 (`LayoutCollide`, its `CollideSource` adapter over `SolidQuery`, feeds `LocomotionStep`; this ADR assumes ADR-0023's Horror uses the same `CollideSource`), ADR-0017 (Emitter; its D8.2 is the occupied-span convention), ADR-0018 and ADR-0028 (Lighting: both read `spans` as occupied spans; a polygon corner is at `(x0, y0)` and `(x1_occ + 1, y1_occ + 1)`; Layout applies no shift), ADR-0034 (PROJECTOR: `section_rects_into` rows are body-frame occupied spans, `idx = idx_of_key(key)`), ADR-0016 (restore order), ADR-0020 (`clip_to_footprints`), ADR-0021 (weld transaction), ADR-0023 (Horror). ADR-0011 and ADR-0019 do not constrain this ADR.
- Asks to Foundation ADRs: see the structured result (ADR-0010 exemption for out-record columns and lint rows; ADR-0007 cross-table BOOT invariant; ADR-0009 sim ports as presenter `setup()` parameters).

## Open Questions
- **OQ-1 (closed): first post-restore `T`.** ADR-0016 section 1.1 (ruling CT-43) states `ClockView.tick()` during R3 and R4 (the last completed tick in `kernel.clock`) and the first RUNNING tick's `T` (one more). `changed_region` after a restore is advisory; `version_epoch` is authoritative (section 6).
- **OQ-2: `BROADPHASE_CELL_PX` is unregistered** in `design/registry/entities.yaml` (the Body GDD calls it "unregistered; derived, never persisted"). It is a `LayoutTuning` key; registering it (with `RECT_KEY_STRIDE`) would let the coupling be sourced from the registry.
- **OQ-3: `clip_to_footprints` record.** ADR-0013 § 11 lists it; this ADR supplies `solid_at`; ADR-0020 must fix the return record, the supercover walk's cell order and its budget.
- **OQ-4: per-body row cap.** ADR-0013 OQ-1: a merged body can exceed `MAX_FOOTPRINT_RECTS_PER_BODY`; the grid has no hard limit (buffers grow), but the budgets do not hold past the cap. Needs the world lint or `MAX_SECTIONS_PER_BODY`.
- **OQ-5: stale summary row.** `requirements-traceability.md` line 1343 lists TR-emitter-044 as `ADR-0007 + ADR-0014` though row 272 re-tags it to ADR-0017 alone; this ADR cannot edit the file.
- **OQ-6: `RectFlags`.** ADR-0013 names the six flags (registry `layout_rect_flags`) but not the bit values or the file; this ADR assumes `RectFlags` constants in `src/sim/layout/views/rect_flags.gd` (bits for `SOLID`, `OCCLUDES_LIGHT`, `DOOR_LEAF`, `PROP`, `MACHINE`, `HULL`, and an all-bits mask).
- **OQ-7: swept-box definition.** Layout D.22 uses `[px - 8 - v, px + 8 + v)` (22 px wide at `v` 3); Body D.12 uses `FOOT_BOX_W + |step|` (19 px). This ADR sizes the coupling on Body's box (the TR-shipbody-037 text); D.22 should adopt it.
- **OQ-8: Lighting's bulk read.** ADR-0028 may find `section_rects_into` insufficient (for example per-compartment rects) and ask for another bulk read; the port is additive.

## Review Log
Review Log 2026-10-29 (engine review E1-E8, TD-ADR review T1-T9, Core cross-ADR adjudication; autonomous mode "Revise flagged items"; Status stays Proposed).

**BLOCKING findings, applied**
- E1: `QueryBuffer._init(capacity)` renamed `initial_capacity` (it shadowed `capacity()` and the class would not load under SHADOWED_VARIABLE=2); Key Interfaces note added that no parameter, local or member may shadow a method of its class; the other classes were checked (LayoutVersions columns fixed under E2); V7 records the probes.
- T1: section 4 fixes the conversion every half-open consumer uses (`x1 = spans[4k+2] + 1`, `y1 = spans[4k+3] + 1`), states `solid_in` is canonical (`solid_in_into` synced), names `CollideSource`/`LayoutCollide` (ADR-0015) as the only wrapper over `SolidQuery`; Enables, diagram, port comment, Related and Validation Ports line corrected; the double is renamed `FakeSolidQuery`; Guidelines bullet added. ADR-0015 side is CT-19.
- T2 with CT-31: Enables and Related now state that ADR-0018 and ADR-0028 read `spans` as occupied spans, corners at `(x0, y0)` and `(x1_occ + 1, y1_occ + 1)`, no shift from Layout (the old text had no ADR-0018 line); ADR-0018's own wording is its ruling.
- T3 with CT-18: `idx_of_key` added to `OccluderSource` (IntMath against the stride); section 5 states rows are body-frame occupied spans, consumers subtract `section_offset_x/_y`, empty-extent rows are omitted so a retracted leaf has no row, and kind comes from the reserved idx ranges. ADR-0034's side is its own.
- T4 with CT-16: section 7 records option (a): the tile leg of `GEOM_OVERLAP` stays ADR-0013's (`floor_tile_at` check after `footprint_overlaps` returns no rect hit); V4/S1/Validation gain the uncached both-legs `GEOM_OVERLAP` line at the largest MVP pair against the 2 ms `can_attach` target, with the 6 ms worst-case rect-leg estimate stated as UNVERIFIED.

**Non-blocking findings, applied**
- E2: `LayoutVersions` columns renamed `_version_col`, `_static_col`, `_leaf_col`; getters `version_of`, `static_version_of`, `leaf_version_of` added to Key Interfaces with an explicit out-of-range guard returning 0, tagged `## @hot`; epoch member `_version_epoch` with getter `version_epoch()`; `changed_region` getters added with an explicit tick argument. The OOB headless hang stays UNVERIFIED (not restated as a claim).
- E3: V1 rewritten (negative `from` is end-relative and can return a valid wrong index; `from >= size` returns -1); rung 4 requires a `pos >= 0` guard in code and in the corpus test.
- E4: note that `: ...` is an elided body and a parse error if pasted.
- E5: V6 recorded as done (editor binary): no UNUSED_PARAMETER on `@abstract`, one-line `@abstract class_name` form valid, typed-return rule confirmed; section 5 sentence added (ports plain names, `src/` implementations prefix unused parameters with `_`); reported to ADR-0010 section 1.2 as an ask.
- E6: debug check `coord_out_of_range` (every `rebuild_body` input and `set_extent`), BOOT invariant `WORLD_COORD_LIMIT_BELOW_SENTINEL` with `BroadphaseCoupling.sentinel_clear` (ADR-0007 registration ask); `set_extent` and `set_empty` tagged `## @hot` and listed in Guidelines and section 9.
- E7: `grow` policy fixed as geometric (`maxi(min_capacity, capacity() * 2)`); MEMORY_STATIC wording is now "no retained growth"; the 38,400 B attribution is stated as UNVERIFIED (buffer versus scratch not separated; S1 reports per structure); step 6 alias rule marked a style rule; section 9 notes the transient-allocation blind spot (HOT-01 carries it). S1 caveats kept.
- E8 (a) and (b): added as Guidelines bullets. (c) to (e): no change needed.
- T5: declared deviation kept (`invalidate`, all three to 0); ADR-0013 ratifies it (CT-15), noted in section 6.
- T6: restore wording now "every LIVE section's three counters 1, every other section 0"; `new_epoch()` resets every `changed_region` `stamp_tick` to -1 (section 6, Key Interfaces, Validation).
- T7: `set_extent` and `set_empty` added to the `## @hot` list (Key Interfaces, Guidelines, section 9); `section_footprints_into` stays untagged by design.
- T8 with CT-45: Guideline added that a `version_epoch` change gates only result-neutral cache rebuilds and never feeds a budget, persisted value or RNG draw.
- T9: `design/registry/entities.yaml` (line 9774 `+ 3` comment) added to the Migration Plan sync list.

**Rulings aligned (owned elsewhere)**
- CT-15 (ADR-0013): section 6 cites ADR-0013's adoption of `invalidate` and `version_epoch`.
- CT-19 (ADR-0015): `CollideSource`/`LayoutCollide` only wrapper over `SolidQuery`; ADR-0023 assumed to use the same `CollideSource`; Validation Ports line reworded.
- CT-20 (ADR-0015): line 48, section 11 and Performance now say `solid_in` is a share of ADR-0015's whole-hook 0.05 ms per actor (0.4 ms for eight), reported by S1 as a sub-counter; the 0.6 ms aggregate is removed; the Emitter box is charged inside ADR-0017's `EXPOSURE_BUDGET_MS`.
- CT-21 (ADR-0015): section 2 states `BROADPHASE_CELL_COUPLING` is the single owner of TR-shipbody-037.
- CT-43 (ADR-0016): OQ-1 closed; `changed_region` after a restore is advisory, `version_epoch` is authoritative.
- CT-54 (ADR-0010): no Core text beyond the above; Acceptance stays conditional on the CNT-03 exemption.

**Declined:** none. No rejected finding applied.

- **2026-10-29 confirm residuals:** Enables row no longer lists ADR-0016 or ADR-0034 (neither has ADR-0014 as a hard Depends On; both are recorded as Related), and the injection paragraph's `layout.solid_query()` and `layout.occluder_source()` now exist in ADR-0013's `LayoutModule` Key Interfaces.
- **2026-10-29 orchestrator rulings:** no R-id touches this ADR. Dependency-cycle break only: the Depends On row is unchanged; the `Related, not Enables` sentence (ADR-0016, ADR-0034) and the unwritten ADR-0020/0021/0023/0028 consumers moved out of Enables into a new `Soft references (not acceptance prerequisites)` row; Enables now lists ADR-0015, ADR-0017 and ADR-0018, each of which lists this ADR under Depends On.
