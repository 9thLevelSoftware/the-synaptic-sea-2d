## Architecture Review Report
Date: 2026-10-28
Engine: Godot 4.7.2 (GDScript)
Mode: full (autonomous; decisions logged in `production/session-logs/decision-log.md`)
GDDs Reviewed: 26 (MVP; plus `systems-index.md`. `game-concept.md` is not a system GDD)
ADRs Reviewed: 3 (ADR-0001, ADR-0002, ADR-0003; all Proposed)

Reviewed-Content-Hash: docs/architecture/adr-0001-kernel-tick-driver-and-frame-order.md 4b384905c8726e6232456001e0099bc92c4c2fe1
Reviewed-Content-Hash: docs/architecture/adr-0002-authority-model-and-command-pipeline.md 6d4672527d9db57aa46c601a54d78558653e8ea1
Reviewed-Content-Hash: docs/architecture/adr-0003-state-block-registry-and-owner-contract.md 18fa93a4fb56a0b3a785b62e707a7506b947008d
Reviewed-Content-Hash: design/gdd/2d-lighting-and-occlusion.md b8f78b686f4b41f32fbd630652bb3e42eda0282d
Reviewed-Content-Hash: design/gdd/adaptive-audio-and-tells.md 775ae97fd730465f0f8f5c27bc70ecad94e9ade2
Reviewed-Content-Hash: design/gdd/attention-system.md 4c7fe0c451521028c3b80baab58a5b73e8f9e260
Reviewed-Content-Hash: design/gdd/boarding-and-salvage.md 22e123be63c10a3c2d6d922e80c888bcbd61bdcd
Reviewed-Content-Hash: design/gdd/chart-table-web-map-and-menus.md c1fe36ee3ec068241dbf7d0f2d00336833d86e24
Reviewed-Content-Hash: design/gdd/command-simulation-layer.md 96aeed103c8a55fcc8bbaf04c8e15f9493b8a668
Reviewed-Content-Hash: design/gdd/debug-overlay-true-values.md af12130eddb9e833e3d99f90defc5af39f31733a
Reviewed-Content-Hash: design/gdd/derelict-authoring-handmade-kits.md ff41356cd1adb9125e4cde5758f6ffa28ef7ba68
Reviewed-Content-Hash: design/gdd/emitter-state-and-light-exposure-query.md 2467e63766af81f069f82bbf219c5967d1fd3fcb
Reviewed-Content-Hash: design/gdd/failure-and-trail-system.md 0496221baa9e226b541fe805fc01f3d9edfc8992
Reviewed-Content-Hash: design/gdd/game-concept.md 8054ee21fb4d606442a905d3ceab01b506b99323
Reviewed-Content-Hash: design/gdd/horror-entities.md d37dd7bdd23a21819562f2afb26a4e9938ff5034
Reviewed-Content-Hash: design/gdd/hud-and-suit-layer.md 177a175e64ca64307c8d41f7b6214315392e9e03
Reviewed-Content-Hash: design/gdd/in-world-instruments-and-inspect-view.md 795c803cfac9c55ce6a805d6cfc18f53fb65957a
Reviewed-Content-Hash: design/gdd/interaction-system.md a4a1665f48a32b46527de86ca982d27fbada4281
Reviewed-Content-Hash: design/gdd/item-and-module-database.md 3fc4816117d0cfcdb32a965fdc6ea960a8103e3d
Reviewed-Content-Hash: design/gdd/life-support-graph.md 56932cfc0c70de71f28d8f9711b4fafa157b51c1
Reviewed-Content-Hash: design/gdd/modular-ship-assembly.md 89fbc7a66804f4f72cf7c7c04233f0bcb799870d
Reviewed-Content-Hash: design/gdd/perception-filter.md 7743c64ce653eaf74327c5ae5946e2604ebb03e2
Reviewed-Content-Hash: design/gdd/pixel-perfect-render-pipeline-camera.md b5b580bc5e3ccfd0a97ba3ae094c1cbb986b581b
Reviewed-Content-Hash: design/gdd/player-controller-input.md 7637d1e85fa0c06e3da04c3015737b882034bd95
Reviewed-Content-Hash: design/gdd/sanity-and-hallucination-rules.md f79049d0a2706e8d3f72fdbe7555a121723107b3
Reviewed-Content-Hash: design/gdd/save-load-and-versioned-format.md a2a088f71ffbe6ca01bb469627060b9a5d043488
Reviewed-Content-Hash: design/gdd/ship-body-and-interior-physics.md 1d74491b210090f366c37fed7d0ec2a39559980f
Reviewed-Content-Hash: design/gdd/ship-section-and-layout-data-model.md 7894fa30e9f5c6a0978e7b9c2c14bacd24590aec
Reviewed-Content-Hash: design/gdd/suit-and-equipment.md f8ca4822f151aa771916071529efef318a345027
Reviewed-Content-Hash: design/gdd/systems-index.md b54524ad50f80beeb03e316192843998c4d127b3
Reviewed-Content-Hash: design/gdd/web-map-and-transit.md 2d4fbb263e931e23167ca68cc0c7341fee5ad7da

---

### Traceability Summary
Total requirements: 1136 (794 from `tr-baseline.md`, plus 342 from this review's completeness pass over 4,053 Acceptance Criteria in the 26 MVP GDDs)
✅ Covered (Accepted ADR): 0 (no ADR is Accepted)
🟡 Covered (Proposed ADR): 50
⚠️ Partial: 48. For 10 of them a Proposed ADR covers only part; the other 38 are planned for ADR-0001/0002/0003, which exist but do not address them.
❌ Gaps: 1038

Every 🟡 row rests on a `Proposed` ADR, so none of them counts as covered for `/create-control-manifest`, `/create-epics`, `/create-stories` or `/gate-check`. The way out is each ADR's own Ordering Note (spike KRN-LAT-02 for ADR-0001; the TD-ADR re-review for ADR-0002; spike KRN-CTX-06 and the TD-ADR re-review for ADR-0003), then `/architecture-decision accept ADR-NNNN`. A status flip on its own is not enough.

**How this review was scoped.** The GDDs total 14 MB (about 500 KB each), so Phase 2 did not re-extract them. It used `tr-baseline.md`, which `/create-architecture` extracted from the same 26 GDDs earlier today, and ran the completeness pass that file asks for before registration: 26 read-only Sonnet agents each read one GDD's Acceptance Criteria section against that system's existing rows and returned only new architectural requirements. Since the baseline was made, seven GDDs changed, all by the ADR GDD-sync edits (+267/−38 lines including the registry), and those edits are what the three ADRs cite. Two ADR rows without a TR-ID were matched by hand: ADR-0003's CR-18 row to TR-cmd-037, and ADR-0002's PlayerId row to TR-suit-003. Seven completeness rows whose text the ADR Decision sections plainly cover were also credited by hand: TR-cmd-042/043/044/048 (ADR-0002), TR-cmd-046/047 (ADR-0001) and TR-cmd-053 (ADR-0002 and 0003). Everything else comes from the ADRs' own `GDD Requirements Addressed` tables.

| System | TRs | 🟡 | ⚠️ | ❌ |
|---|---|---|---|---|
| Command & Simulation Layer | 53 | 25 | 8 | 20 |
| Life-Support Graph | 43 | 0 | 1 | 42 |
| Item & Module Database | 40 | 0 | 0 | 40 |
| Pixel-Perfect Render Pipeline & Camera | 37 | 2 | 0 | 35 |
| Emitter State & Light-Exposure Query | 46 | 1 | 2 | 43 |
| 2D Lighting & Occlusion | 43 | 1 | 1 | 41 |
| Player Controller & Input | 46 | 5 | 5 | 36 |
| Interaction System | 43 | 2 | 3 | 38 |
| Failure & Trail System | 44 | 0 | 1 | 43 |
| Suit & Equipment | 43 | 1 | 3 | 39 |
| Ship Section & Layout Data Model | 43 | 1 | 1 | 41 |
| Ship Body & Interior Physics | 50 | 2 | 1 | 47 |
| Modular Ship Assembly | 48 | 2 | 1 | 45 |
| Derelict Authoring (Handmade Kits) | 41 | 1 | 0 | 40 |
| Boarding & Salvage | 44 | 0 | 1 | 43 |
| Web Map & Transit | 46 | 1 | 1 | 44 |
| Attention System | 47 | 1 | 0 | 46 |
| Horror Entities | 43 | 1 | 1 | 41 |
| Sanity & Hallucination Rules | 40 | 1 | 2 | 37 |
| Perception Filter | 45 | 0 | 1 | 44 |
| HUD & Suit Layer | 42 | 1 | 1 | 40 |
| In-World Instruments & Inspect View | 41 | 0 | 0 | 41 |
| Chart Table (Web Map & Menus) | 38 | 2 | 3 | 33 |
| Adaptive Audio & Tells | 43 | 0 | 1 | 42 |
| Save/Load & Versioned Format | 42 | 0 | 5 | 37 |
| Debug Overlay (True Values) | 45 | 0 | 5 | 40 |

The full 1,136-row matrix is in `docs/architecture/requirements-traceability.md`.

### Coverage Gaps (no ADR exists)
All 1,038 gaps already have a planned ADR in `architecture.md` § Required ADRs; there is no requirement that no planned ADR would cover. Gaps are grouped by that ADR here; each gap's TR-IDs are listed under § Known Gaps in `requirements-traceability.md`.

| Planned ADR | Tier | Gaps | Partial | Domain | Engine risk |
|---|---|---|---|---|---|
| ADR-0004 Change notification and dirty keys | Foundation | 19 | 0 | Core | MEDIUM (4.7 packed-array element writes) |
| ADR-0005 Determinism, keyed RNG, canonical encoding, hashing | Foundation | 42 | 3 | Core | MEDIUM (S0-a/b spike) |
| ADR-0006 Save container, generations, versioning, restore epoch | Foundation | 93 | 4 | Save/Load | MEDIUM (FileAccess bool returns, Windows rename) |
| ADR-0007 Static content pipeline and stable type IDs | Foundation | 104 | 0 | Data | LOW |
| ADR-0008 Render pipeline wiring and integer scaling | Foundation | 48 | 0 | Rendering | HIGH (D3D12 default, SubViewport, glow order) |
| ADR-0009 Presentation subscriber contract | Foundation | 39 | 1 | Rendering | MEDIUM |
| ADR-0010 Source layout, static typing, lint enforcement | Foundation | 37 | 1 | Core | HIGH (4.7 GDScript rules) |
| ADR-0011 Life-Support Graph data model | Core | 18 | 0 | Core | MEDIUM |
| ADR-0012 Graph propagation scheme | Core | 27 | 0 | Performance | MEDIUM (S1 spike) |
| ADR-0013 Ship layout block, geometry, IDs | Core | 27 | 0 | Data | LOW |
| ADR-0014 Broadphase and OccluderSource seam | Core | 13 | 0 | Performance | MEDIUM |
| ADR-0015 Integer kinematics and the ship body | Core | 32 | 1 | Physics | LOW (no engine physics) |
| ADR-0016 Restore ordering and cross-block verification | Core | 22 | 1 | Save/Load | LOW |
| ADR-0017 Emitter state and light-exposure query | Core | 49 | 1 | Core | MEDIUM |
| ADR-0018 Shadow origin and beam occlusion | Core | 1 | 0 | Rendering | HIGH (lighting spike) |
| ADR-0019 Player controller split and input contexts | Core | 39 | 0 | Input | HIGH (4.7 device IDs) |
| ADR-0034 PROJECTOR scene shape and Y-sort | Core | 3 | 0 | Rendering | MEDIUM (SP-YSORT-1) |
| ADR-0020 Interaction channels and verb table | Feature | 47 | 3 | Core | LOW |
| ADR-0021 Atomic weld/cut/install transaction | Feature | 35 | 2 | Core | LOW |
| ADR-0022 Attention model and breach state machine | Feature | 35 | 1 | Core | LOW |
| ADR-0023 Horror entity simulation | Feature | 24 | 0 | Core | LOW |
| ADR-0024 Perception isolation and the Filter | Presentation | 86 | 1 | Core | LOW |
| ADR-0025 Sanity model and lie lifecycle | Feature | 26 | 1 | Core | LOW |
| ADR-0026 World model, derelict loading, transit | Feature | 45 | 1 | Core | LOW |
| ADR-0027 Suit vitals and equipment | Feature | 26 | 1 | Core | LOW |
| ADR-0028 2D lighting budget and modifier chain | Presentation | 76 | 0 | Rendering | HIGH (4.6 glow, 4.7 lines) |
| ADR-0029 Audio architecture | Presentation | 47 | 1 | Audio | MEDIUM (4.7 `area_mask` default) |
| ADR-0030 UI framework | Presentation | 53 | 2 | UI | HIGH (4.6 dual focus, 4.7 lines) |
| ADR-0031 Diagnostics and dev-feature gating | Presentation | 53 | 1 | Core | LOW |
| ADR-0032 Replication shape for co-op | Deferred (VS-B) | 9 | 0 | Networking | MEDIUM |

A row is counted under every ADR it names, so the Gaps column adds up to more than 1,038.

**The 38 partials planned for the written ADRs** are mostly detailed contracts that the completeness pass found in Acceptance Criteria: same-tick ordering conventions, binding changes taking effect at T+1 (TR-cmd-045), PAUSED freezes render-time timers (TR-pc-038, TR-ftrail-039), the boot refusal when TICK_RATE ≠ 60 (TR-assembly-042), the `after: [body.integrate]` lint (TR-shipbody-042), and per-system command and hook registrations. Most of them are realised by the system ADRs (0019–0027) on top of the kernel contract. Each is listed under § Partial Coverage in the traceability index, so the next revision of each ADR can either claim it or hand it to its system ADR.

### Cross-ADR Conflicts
**None found.** All three ADRs were read in full (Decision, interfaces, Guidelines), and the 0002 ↔ 0003 scope edge is consistent in the text:
- `register_command` sits on `BlockRegistrar` (ADR-0003 § Key Interfaces; ADR-0002 line 215 forwards to it).
- Result and control-outcome handlers are `(record, ctx: NotifyContext) -> CompletionToken` in both.
- `payload_schema` is a field of ADR-0003's complete `CommandTypeDescriptor`, and ADR-0002 owns its meaning.
- `snap_baseline` on `interpolated` blocks (ADR-0001 § 4) matches ADR-0003's descriptor, where it is mandatory iff `interpolated`.
- `ModuleFacet` (ADR-0001 § 5) is defined in ADR-0003 with the registrar, views and fault domain.

Known conflict-prone areas: `docs/consistency-failures.md` has no Architecture-domain entries yet. The recurring architecture-adjacent pattern is stale tick-source text in consumer GDDs (ADR-0001 § GDD sync). It is fixed for these ADRs per ADR-0001 § GDD sync, and no new instance was found. One registry leftover: `MAX_SCHEDULE_CYCLE_TICKS` is in the kernel GDD's Tuning Knobs but not yet in `design/registry/entities.yaml` (ADR-0003 § GDD sync says so itself).

### ADR Dependency Order
Output of `adr-dep-graph.sh`: 3 ADRs, edges 0002→0001, 0003→0001, 0003→0002, no `CYCLE`, no `NO_DEPS_SECTION`.

```
Foundation (no dependencies):
  1. ADR-0001: Kernel tick driver and frame order
Depends on Foundation:
  2. ADR-0002: Authority model and command pipeline (requires ADR-0001)
  3. ADR-0003: State-block registry and owner contract (requires ADR-0001, ADR-0002)
```
⚠️ ADR-0002 depends on ADR-0001, which is still Proposed. ⚠️ ADR-0003 depends on ADR-0001 and ADR-0002, both still Proposed. Accept them in this order. Each ADR's `Enables` row names ADRs that are not written yet (0004–0034); those are planned, not unresolved dependencies.

### GDD Revision Flags
| GDD | Assumption | Reality | Action |
|---|---|---|---|
| command-simulation-layer.md | `var_to_bytes(payload, false).size()` for `MAX_PAYLOAD` (CR-4 table row, Payload whitelist, CR-8, Tuning Knobs, Edge Cases; 7 places) | Godot 4.7.2: `var_to_bytes()` takes one argument. Two arguments are a parse error ("Too many arguments … Expected at most 1"), confirmed headless on the installed 4.7.2 editor. Objects are already excluded by the whitelist, so the fix is `var_to_bytes(payload)`. | Revise GDD |
| ship-section-and-layout-data-model.md | AC-162 (P3) tests `var_to_bytes(dict, false)` determinism | Same | Revise GDD |
| suit-and-equipment.md | AC-SE-96 measures `var_to_bytes(carried, false)` as 68 bytes | Same. Also re-check the 68-byte figure with the one-argument form. | Revise GDD |

The systems index is updated: these three rows are `Needs Revision` (autonomous mode, logged). The same two-argument form also appears in ADR-0002 (3 places), `architecture.md` (1) and `tr-baseline.md`/`tr-registry.yaml` TR-cmd-010. When the GDD is fixed, set TR-cmd-010's `revised` date.

### Engine Compatibility Issues
Engine: Godot 4.7.2. ADRs with an Engine Compatibility section: 3 / 3. All three pin 4.7.2; there are no stale version references and no post-cutoff API dependencies. Deprecated-API grep: clean. The `OS.get_ticks_*` and `device == 0` hits forbid those APIs rather than use them.

### Engine Specialist Findings
godot-specialist second opinion (read-only). Items marked **[probed]** were run on the installed 4.7.2 editor in a headless scratch probe; the rest are from the specialist's knowledge of the engine source.

| Sev | ADR | Finding | Status |
|---|---|---|---|
| HIGH | 0002 Engine Compat (1), Decision § 3 | `var_to_bytes(payload, false)` is the Godot 3 `var2bytes` shape; in 4.x it is a parse error. Use `var_to_bytes(payload)`. | **[probed] confirmed** |
| HIGH | 0003 Verification (3) | "A Callable bound to a RefCounted method keeps the object alive" is false. A method Callable holds only the ObjectID; when the last reference drops, the owner is freed and `is_valid()` turns false. The composition root must hold every owner in an explicit strong-reference list, and the teardown rule and `OBJECT_COUNT` test must start from "callables do not own". | **[probed] confirmed** (owner freed, callable invalid) |
| MEDIUM | 0003 § 3, Verification (1) | Fail-closed must not depend on a runtime error aborting the callee. In release templates many VM checks are compiled out, so a callee can log the error, carry on and still return `ctx.complete()`. KRN-CTX-06 should test both outcomes on a release export; the WriteScope close and the STRICT_DEBUG checks have to carry fail-closed. | Unverified (needs an export template) |
| MEDIUM | 0003 Guidelines | `await` inside an owner callable is not banned. A suspended call returns a coroutine state (caught as INCOMPLETE), but it can later resume against a re-armed context and scope and write through them. Add a lint against `await` in `src/sim/` callables; hold the call result as `Variant` and compare it with `is_same()`. | Design gap |
| MEDIUM | 0001 § 2 | `process_thread_group` defaults to INHERIT, so the boot walk must resolve each node's effective group through its ancestors, not read the node's own value. | Design gap |
| LOW | 0001 § 2 | `SceneTree.process_frame` fires before any `_process`, so a coroutine resumed by it runs ahead of the Input Adapter. This supports the ban. | Strengthens the ADR |
| LOW | 0001 Guidelines | `OS.get_unix_time*` does not exist in Godot 4. Lint the real APIs instead (`Time.get_unix_time_from_system`, `Time.get_datetime_dict_from_system`, `Time.get_time_dict_from_system`), and consider `Engine.get_frames_drawn` and `get_process_frames`. The Sanity GDD lint list (AC-SN-130) has the same stale name. | Lint list fix |
| LOW | 0001 Engine Compat (2)/(3a)/(3b) | `Engine.time_scale` scales `_process` delta (**[probed]**: the call exists, and the scaling is the specialist's claim); `delta_smoothing` and MainTimerSync do shape delta; `_input` arrives before `_process`. The spike should still record them. | Partly confirmed |
| LOW | 0002 § 3 | String and StringName dictionary keys are equal for lookup, so the `MALFORMED` mixed-key rule is a policy check (`typeof`), not an engine necessity. | **[probed] confirmed** (`{"k":1}.has(&"k")` is true) |
| LOW | 0002 Engine Compat (3) | Use `MultiplayerPeer.TARGET_PEER_SERVER` rather than the literal 1 for `HOST_PEER_ID`. | Advisory |
| LOW | 0003 Engine Compat | `Callable.get_argument_count()` exists, so remove the "if it exists" hedge. A typed function with a missing `return` path is rejected at parse time. | **[probed] confirmed** (method present) |
| LOW | 0002 Engine Compat (2) | `var_to_bytes` errors on a cyclic container. Walking the depth-8 cap first stays correct. | Supports the ADR |

These go back to the ADRs' authors as revisions before acceptance (`/architecture-decision` on ADR-0002 and ADR-0003). They are not cross-ADR conflicts, so they are not logged in `consistency-failures.md`.

### Architecture Document Coverage
`docs/architecture/architecture.md` exists and was checked.
- **Systems:** all 26 MVP systems in `systems-index.md` appear in § Layer assignments (Foundation 5, Core 6, Feature 8, Presentation 7, with the Player Controller split across Core and Presentation). The 16 Vertical Slice and Alpha systems are correctly absent, and Co-op (#33) has a deferred ADR-0032.
- **Orphaned architecture:** none. Every layered system has a GDD.
- **Data flow and API boundaries:** eight data flows and the API Boundaries section already went through TD-ARCHITECTURE and LP-FEASIBILITY (both CONCERNS → revised, v2). This review did not re-derive them. One stale item: the `weld_join` signature and the § Interface types still-to-define list point at ADRs that are not written yet, and that is expected.
- **One correction to carry:** `architecture.md` contains the two-argument `var_to_bytes` once (see GDD Revision Flags).

---

### Verdict: FAIL

**Why FAIL and not CONCERNS:** under Phase 7, critical gaps (requirements in the Foundation or Core layer with no ADR) mean FAIL. Seven of the ten Foundation ADRs (0004–0010) are not written, which leaves 382 Foundation-planned requirement links as gaps. In addition, no ADR is Accepted, so even the 50 covered rows are 🟡. This is the expected state partway through authoring the ADRs: `architecture.md` plans 34 ADRs and 3 exist. It is not a sign of a broken design. No cross-ADR conflicts were found, and the dependency graph is clean.

### Blocking Issues (must resolve before PASS)
1. Write the remaining Foundation ADRs 0004–0010 (`architecture.md` § Required ADRs), then the Core ADRs 0011–0019 and 0034 before their systems are built.
2. Revise ADR-0002 (`var_to_bytes` arity, HIGH) and ADR-0003 (Callable lifetime, HIGH; fail-closed without abort and an `await` ban, MEDIUM) before acceptance. ADR-0001 needs the `process_thread_group` effective-group walk and the corrected clock lint list.
3. Move ADR-0001 → 0002 → 0003 to Accepted through their spikes and re-reviews (`/architecture-decision accept`).
4. Revise the three flagged GDDs (two-argument `var_to_bytes`).

### Required ADRs
Unchanged from `architecture.md` § Required ADRs and its spike order. Most foundational first: ADR-0004 (change notification), ADR-0005 (determinism and hashing), ADR-0006 (save container), ADR-0007 (content pipeline), ADR-0008 (render wiring; gated by the lighting spike's renderer result), ADR-0009 (presentation subscriber contract), ADR-0010 (source layout and lints). After those come the Core ADRs 0011–0019 and 0034, then the Feature and Presentation ADRs 0020–0031.
