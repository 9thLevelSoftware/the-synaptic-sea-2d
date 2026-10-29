## Architecture Review Report
Date: 2026-10-28 (second run)
Engine: Godot 4.7.2 (GDScript)
Mode: full (autonomous; decisions logged in `production/session-logs/decision-log.md`)
GDDs Reviewed: 26 (MVP; plus `systems-index.md`; `game-concept.md` is not a system GDD)
ADRs Reviewed: 10 (ADR-0001..ADR-0010; all Proposed)

Reviewed-Content-Hash: docs/architecture/adr-0001-kernel-tick-driver-and-frame-order.md 1306b78153623bcd090ee762d74cadda31356005
Reviewed-Content-Hash: docs/architecture/adr-0002-authority-model-and-command-pipeline.md bc6771a22261e7e4b418eb9ba533af58aef3b188
Reviewed-Content-Hash: docs/architecture/adr-0003-state-block-registry-and-owner-contract.md 81afbbbcf077b5694779020f0cf8e081b86ef864
Reviewed-Content-Hash: docs/architecture/adr-0004-change-notification-and-dirty-keys.md 4005356a50b98eb9b162c60e8241c9f2c8a08cd9
Reviewed-Content-Hash: docs/architecture/adr-0005-determinism-keyed-rng-canonical-encoding-and-hashing.md 4ceff6f1b0b03c9db726ec3643dbfcf056d84093
Reviewed-Content-Hash: docs/architecture/adr-0006-save-container-generations-versioning-and-restore-epoch.md 0070cb74982427d5957f3797b69336293c46c327
Reviewed-Content-Hash: docs/architecture/adr-0007-static-content-data-pipeline-and-stable-type-ids.md 7b434e6b5540fa5062bd384566efe5f98aa02bfe
Reviewed-Content-Hash: docs/architecture/adr-0008-render-pipeline-wiring-and-integer-scaling.md 1a091a6d17a1593778c5b6fe43318e23f27cc1f6
Reviewed-Content-Hash: docs/architecture/adr-0009-presentation-subscriber-contract.md f315a75d1aac495045f222c0810fe2a06af8a1d4
Reviewed-Content-Hash: docs/architecture/adr-0010-source-layout-static-typing-and-lint-enforcement.md eba998fe9c21548b5b57951c739b08b425c05b37
Reviewed-Content-Hash: design/gdd/2d-lighting-and-occlusion.md b8f78b686f4b41f32fbd630652bb3e42eda0282d
Reviewed-Content-Hash: design/gdd/adaptive-audio-and-tells.md 775ae97fd730465f0f8f5c27bc70ecad94e9ade2
Reviewed-Content-Hash: design/gdd/attention-system.md 4c7fe0c451521028c3b80baab58a5b73e8f9e260
Reviewed-Content-Hash: design/gdd/boarding-and-salvage.md 22e123be63c10a3c2d6d922e80c888bcbd61bdcd
Reviewed-Content-Hash: design/gdd/chart-table-web-map-and-menus.md c1fe36ee3ec068241dbf7d0f2d00336833d86e24
Reviewed-Content-Hash: design/gdd/command-simulation-layer.md 38ca35df64dc812ca5e0da4e179b7bae6bf4fab5
Reviewed-Content-Hash: design/gdd/debug-overlay-true-values.md af12130eddb9e833e3d99f90defc5af39f31733a
Reviewed-Content-Hash: design/gdd/derelict-authoring-handmade-kits.md ff41356cd1adb9125e4cde5758f6ffa28ef7ba68
Reviewed-Content-Hash: design/gdd/emitter-state-and-light-exposure-query.md d225e091e43fc8263ca4e3930b561758d5f123b7
Reviewed-Content-Hash: design/gdd/failure-and-trail-system.md 1cf349407e301dc3590a5ec529fed88736deb2c9
Reviewed-Content-Hash: design/gdd/game-concept.md 8054ee21fb4d606442a905d3ceab01b506b99323
Reviewed-Content-Hash: design/gdd/horror-entities.md d37dd7bdd23a21819562f2afb26a4e9938ff5034
Reviewed-Content-Hash: design/gdd/hud-and-suit-layer.md 177a175e64ca64307c8d41f7b6214315392e9e03
Reviewed-Content-Hash: design/gdd/in-world-instruments-and-inspect-view.md 795c803cfac9c55ce6a805d6cfc18f53fb65957a
Reviewed-Content-Hash: design/gdd/interaction-system.md a4a1665f48a32b46527de86ca982d27fbada4281
Reviewed-Content-Hash: design/gdd/item-and-module-database.md 308c32eb30bd23e7c570db5951e2cddbeeec06ee
Reviewed-Content-Hash: design/gdd/life-support-graph.md 49857334548edd3486dd3cb243155fc468a60100
Reviewed-Content-Hash: design/gdd/modular-ship-assembly.md 89fbc7a66804f4f72cf7c7c04233f0bcb799870d
Reviewed-Content-Hash: design/gdd/perception-filter.md 7743c64ce653eaf74327c5ae5946e2604ebb03e2
Reviewed-Content-Hash: design/gdd/pixel-perfect-render-pipeline-camera.md b5b580bc5e3ccfd0a97ba3ae094c1cbb986b581b
Reviewed-Content-Hash: design/gdd/player-controller-input.md 7637d1e85fa0c06e3da04c3015737b882034bd95
Reviewed-Content-Hash: design/gdd/sanity-and-hallucination-rules.md 70011a813da50dc039986462b026a11163208f1c
Reviewed-Content-Hash: design/gdd/save-load-and-versioned-format.md a2a088f71ffbe6ca01bb469627060b9a5d043488
Reviewed-Content-Hash: design/gdd/ship-body-and-interior-physics.md 1d74491b210090f366c37fed7d0ec2a39559980f
Reviewed-Content-Hash: design/gdd/ship-section-and-layout-data-model.md 5a5a1294a77a2bb6ca74ebf85e3bf084cb6f2379
Reviewed-Content-Hash: design/gdd/suit-and-equipment.md 605988956a89d777006613eb514db819f98c0a5a
Reviewed-Content-Hash: design/gdd/systems-index.md c8d65ed5db901b2b0dbf10a53dfc9ac46a89bbc0
Reviewed-Content-Hash: design/gdd/web-map-and-transit.md 2d4fbb263e931e23167ca68cc0c7341fee5ad7da

---

> This report replaces the earlier 2026-10-28 review (verdict FAIL, 3 ADRs). That version is in git at `f059a74`. This run is the full re-run after Foundation ADR-0004..0010 were written (`329b59f`). Phase 1a freshness: ADR-0001/0002/0003 CHANGED, ADR-0004..0010 NEW, 7 GDDs CHANGED (ADR syncs only).

### Traceability Summary
Total requirements: 1138 (1136 registered + 2 new)
✅ Covered (Accepted ADR): 0. All ten ADRs are Proposed.
🟡 Covered (Proposed): 181 (15.9%). Previous run: 50.
⚠️ Partial: 187 (16.4%). Previous run: 48.
❌ Gaps: 770 (67.7%). Previous run: 1038.

| Layer of planned ADR | 🟡 Covered (Proposed) | ⚠️ Partial | ❌ Gap |
|---|---|---|---|
| Foundation (ADR-0001..0010) | 181 | 170 | 89 |
| Core (ADR-0011..0019, 0034) | 0 | 3 | 200 |
| Feature / Presentation (ADR-0020..0031) | 0 | 2 | 469 |
| Deferred (ADR-0032/0033) | 0 | 0 | 9 |
| GDD-owned (ADR-0007 owned-elsewhere table) | 0 | 12 | 3 |

**How coverage was graded.** One Sonnet reviewer per written ADR graded COVERED, PARTIAL or NOT for every requirement planned for that ADR or cited by it: 461 requirements and 517 (requirement, ADR) pairs. A requirement takes its best grade across ADRs. The 675 requirements planned only for ADR-0011..0034 were not graded against the written ADRs; they are Gap by construction. Two reviewers (ADR-0005, ADR-0010) graded mainly from the ADR's own GDD Requirements Addressed table and rule tables, because whole-section reads were refused by the 40 KB read guard. Those covered grades rest on the ADR's own claims.

Phase 3b (story and test linkage) was skipped: `production/epics/` does not exist yet.

**Per written ADR** (grades over its candidates; full rows in `requirements-traceability.md`):

| ADR | Candidates | Covered | Partial | Not addressed |
|---|---|---|---|---|
| ADR-0001 tick driver | 30 | 7 | 11 | 12 |
| ADR-0002 command pipeline | 27 | 5 | 17 | 5 |
| ADR-0003 block registry | 44 | 8 | 22 | 14 |
| ADR-0004 notification | 19 | 5 | 12 | 2 |
| ADR-0005 determinism | 48 | 28 | 20 | 0 |
| ADR-0006 save container | 104 | 44 | 36 | 24 |
| ADR-0007 content pipeline | 104 (+2 new) | 30 (+1) | 47 (+1) | 27 |
| ADR-0008 render pipeline | 57 | 20 | 18 | 19 |
| ADR-0009 presentation | 42 | 14 | 18 | 10 |
| ADR-0010 layout and lint | 42 | 20 | 22 | 0 |

Most PARTIAL grades share one pattern: the Foundation ADR supplies the mechanism (block descriptor, content loader, subscriber contract, lint engine), and the system-specific values are left to the module ADR (ADR-0011..0031) or to the GDD. They close when those ADRs are written; they are not defects in the Foundation set.

### Coverage Gaps

**Gaps a written Foundation ADR was expected to decide but does not (89).** These are the actionable Foundation items, by planned owner:
- ADR-0001 (11): TR-pc-038, TR-ftrail-039, TR-suit-038, TR-assembly-042, TR-boarding-040, TR-sanity-036, TR-percfilter-035, TR-hud-040, TR-audio-035, TR-debug-032, TR-debug-036.
- ADR-0002 (5): TR-emitter-035, TR-light-033, TR-interact-031, TR-suit-034, TR-webmap-034.
- ADR-0003 (8): TR-cmd-015, TR-cmd-053, TR-pc-034, TR-pc-037, TR-shipbody-042, TR-webmap-025, TR-sanity-035, TR-charttable-035.
- ADR-0004 (2): TR-ftrail-040 (one Graph step's keys applied in a single render callback), TR-assembly-033 (per-block key footprints, never-collapse-to-ALL budget).
- ADR-0006 (19): TR-lsg-036, TR-lsg-038, TR-itemdb-011, TR-itemdb-013, TR-itemdb-038, TR-emitter-007, TR-pc-021, TR-suit-007, TR-suit-008, TR-suit-010, TR-shipbody-040, TR-shipbody-041, TR-assembly-012, TR-attention-023, TR-horror-018, TR-horror-036, TR-sanity-021, TR-sanity-037 (and TR-pc-037).
- ADR-0007 (27, of which 3 are GDD-owned rules): TR-lsg-043, TR-itemdb-026/031/034, TR-emitter-043/044, TR-light-041, TR-pc-046, TR-ftrail-036, TR-suit-042, TR-derelict-034, TR-boarding-036, TR-webmap-035, TR-attention-043, TR-percfilter-045, TR-hud-035, TR-instr-009/028/037/038/039, TR-charttable-031, TR-audio-040; GDD-owned: TR-itemdb-015, TR-boarding-019, TR-derelict-001 (TR-itemdb-016 re-tagged to ADR-0011).
- ADR-0008 (15 graded not addressed; TR-pxl-006 also below): TR-pxl-027, TR-pxl-033, TR-emitter-041, TR-light-038/039/040, TR-interact-042, TR-derelict-041, TR-webmap-038, TR-percfilter-032/034, TR-charttable-036/037, TR-audio-034, TR-debug-042.
- ADR-0009 (9): TR-pxl-006/007/008, TR-ftrail-017, TR-section-025, TR-shipbody-032, TR-boarding-026, TR-instr-010, TR-instr-013.

Many of these are acceptance-criteria-derived rows (the 342 added by the previous run's AC pass): per-system lint lists, per-system perf checks, per-system save fields. Either the owning Foundation ADR adds a row for each, or the row is re-tagged to the module ADR that will hold it. `/architecture-decision` on the owner decides which.

**Gaps whose planned ADR is unwritten (675)**, most foundational first:

| Planned ADR | Layer | Gaps | Suggested command | Engine risk |
|---|---|---|---|---|
| ADR-0011 Life-Support Graph data model | Core | 17 | `/architecture-decision life-support graph data model` | LOW |
| ADR-0012 Graph propagation scheme | Core | 26 | `/architecture-decision graph propagation` | LOW |
| ADR-0013 Ship layout block, geometry, ID allocation | Core | 27 | `/architecture-decision ship layout block` | MEDIUM (packed arrays, JSON ints) |
| ADR-0014 Geometry broadphase and OccluderSource | Core | 13 | `/architecture-decision geometry broadphase` | MEDIUM |
| ADR-0015 Integer kinematics and the ship body | Core | 30 | `/architecture-decision ship body kinematics` | LOW |
| ADR-0016 Restore ordering and cross-block verification | Core | 19 | `/architecture-decision restore ordering` | LOW |
| ADR-0017 Emitter state and light-exposure query | Core | 50 | `/architecture-decision emitter state` | MEDIUM |
| ADR-0018 Shadow origin and beam occlusion | Core | 1 | (may fold into ADR-0028) | HIGH (Light2D) |
| ADR-0019 Player controller split and input contexts | Core | 36 | `/architecture-decision player controller and input` | HIGH (4.7 input device IDs) |
| ADR-0034 PROJECTOR scene shape and Y-sort | Core | 2 | `/architecture-decision projector scene shape` | MEDIUM (SP-YSORT-1) |
| ADR-0020..0031 | Feature / Presentation | 469 | one per ADR; largest: ADR-0024 Perception Filter 86, ADR-0028 lighting 72, ADR-0030 UI 52, ADR-0017 above, ADR-0020 interaction 49, ADR-0031 diagnostics 48 | ADR-0028 HIGH, ADR-0030 MEDIUM, others LOW |
| ADR-0032 co-op replication | Deferred | 9 | before VS-B | MEDIUM |

### Cross-ADR Conflicts

**Known conflict-prone areas** (from the run-2 residual list; `docs/consistency-failures.md` has no Architecture or Engine domain entries): signature drift between a facet's definer and its forwarders (`subscribe`, `request_restore`); lint token lists that lag the ADR naming the rule; Depends On / Enables tables written in parallel.

**Run-2 residuals re-checked (13):** 2 FIXED (registry merged and matching ADR-0001/0002/0003/0009; ADR-0005 Variant pass-throughs allow-listed by ADR-0010:227). 11 STILL OPEN:

| # | ADRs | Severity | Open item |
|---|---|---|---|
| 2 | 0005 / 0010 | MEDIUM | ADR-0010 is internally inconsistent: § 5.3 (l.233) says `sorted_int_keys`/`sorted_string_keys` in `canonical_encoder.gd`, but the SIM-09 row (l.292) still allow-lists `sorted_keys.gd`, the DET row (l.311) says `sorted_text_keys`, and OQ-N(2) (l.679) still asks ADR-0005 to name them. The lint config will be built from SIM-09. |
| 4 | 0006 / 0003 | LOW | `SaveService.setup(...)` (ADR-0006 l.212) takes no `HostRegistrarFacet`, so nothing wires fault domain 1; ADR-0006 l.422 still calls this an unresolved assumption. |
| 5 | 0008 / 0009 | LOW | ADR-0009 l.430 says snap flags snap in base-pixel space "only under pass B"; ADR-0008 l.132/854 says "in both passes". |
| 6 | 0009 / 0004 | LOW | ADR-0009 l.124 forwards to `NotifyFacet.subscribe(cls, block_id, handler, name)`; ADR-0004 l.133 defines `subscribe(block_id, handler, name)` (class fixed at issue). |
| 7 | 0010 / 0009 | LOW | PRES-03 (ADR-0010 l.314) lacks `set_transform(`, `set_global_transform(`, `set("position")`, `global_transform =`, `Sprite2D.offset`, which ADR-0009 l.305 requires. |
| 8 | 0010 | LOW | ADR-0010 l.137 lists `NotifyFacet` among classes presentation may name, then restricts it to `src/game/`; `edges.json` could be generated wrongly. |
| 9 | 0005 / 0010 | INFO | `IntMath` at `src/shared/int_math.gd`, outside `det/` (TD-12 wording); both ADRs agree. |
| 10 | 0001..0007, 0010 | LOW | Depends On / Enables tables not symmetric (ADR-0001 Enables omits 0007/0010; 0002 omits 0004/0009/0010; 0003 omits 0007/0010; 0004 Enables 0006 which calls it "Related"). |
| 11 | 0001 | LOW | ADR-0001 l.27 Verification Required (1) still says adapter → driver → presentation; l.84/270 include SaveHost at HOST_SERVICE. |
| 12 | 0008 / 0009 | LOW | ADR-0009 l.188: PipelineRoot "owns" the PresentationDriver; ADR-0008 l.241: the composition root constructs and injects it. |
| 13 | 0010 / 0008 / 0009 | LOW | PRES-01 `_process` exemption set omits debug layer 5, which ADR-0008 l.567 puts on engine time. |

**New conflicts (2):**

#### Conflict: ADR-0006 vs ADR-0001
Type: Integration (phantom owner). Severity: MEDIUM.
ADR-0006 claims: Depends On (l.36) cites "ADR-0001 (... quit transition ...)"; l.153 "ADR-0001 owns the quit transition".
ADR-0001 claims: nothing about quit, shutdown or window close. `request_shutdown` is an ADR-0002 control call (l.205) and `SHUTTING_DOWN` is ADR-0003's state machine (l.269).
Impact: the SaveHost window-close path ("write a QUIT generation, then quit") has no kernel-side contract in the ADR it cites.
Resolution options:
  1. Re-attribute the quit transition in ADR-0006 to ADR-0003 (state machine) and ADR-0002 (`request_shutdown`).
  2. Add a short Quit/shutdown paragraph to ADR-0001 (SaveHost handles `NOTIFICATION_WM_CLOSE_REQUEST`; the kernel reaches SHUTTING_DOWN only through `request_shutdown`).

#### Conflict: ADR-0006 vs ADR-0008
Type: Performance budget. Severity: LOW.
ADR-0006 claims (l.554): the capture frame may reach `SNAPSHOT_BUDGET_MS` 16.7 ms; one over-budget frame per autosave is accepted pending SAVE-WRK-1 (f).
ADR-0008 claims (l.296): total frame CPU P99 ≤ 14.9 ms (RM-DESKTOP) / 30.0 ms (RM-DECK) at every ladder field, with no autosave exclusion.
Impact: a pipeline perf test whose window contains an autosave can fail the ADR-0008 row.
Resolution options:
  1. Qualify ADR-0008's row: "excludes SAVING-state frames".
  2. Cap or amortise capture cost (SAVE-WRK-1 f) so a capture frame fits 14.9 ms.

No HIGH (🔴) cross-ADR conflict. Contracts checked and consistent across the set: `ProcessPriority` values (-100 / -50 / -40 / 0), fault domain ids (0 kernel, 1 save_load, 2 content, 3 graph), `request_restore` / `request_new_game` signatures (ADR-0002 vs ADR-0006 vs registry), `WriteScope` / `BlockRegistrar` signatures, `restore_epoch` ownership (Save/Load owns, read-only copy in `kernel.clock`), `RESTORE_KEY_STRIDE` 2^20, `STEP_BUDGET_MS` semantics, the `HashingContext` file list.

### ADR Dependency Order

`adr-dep-graph.sh`: 10 ADRs, 34 edges, no `NO_DEPS_SECTION`. It reports **4 cycles**:
```
CYCLE: adr-0004 -> adr-0010
CYCLE: adr-0005 -> adr-0004 -> adr-0010
CYCLE: adr-0007 -> adr-0005
CYCLE: adr-0010 -> adr-0007
BLOCKED_BY_CYCLE: adr-0006, adr-0008, adr-0009
```
**Judgement: no real acceptance deadlock.** Every cycle runs through an edge its own ADR marks soft or post-acceptance. ADR-0004 (l.34) and ADR-0005 list ADR-0010 as a "soft dependency" ("Proposed is enough"). ADR-0007 (l.33) lists ADR-0005 for "golden content-hash vectors only, not an authoring dependency", and l.444 puts it after acceptance. Without those three edges the hard acceptance prerequisites (quoted from each ADR's Ordering Note) form a DAG. The script cannot tell soft edges from hard ones, so it will keep reporting the cycles until the soft references move out of the Depends On cell (to Related, or to a Validation follow-up row). That is a required text fix before acceptance (LOW).

**Recommended ADR acceptance order (hard prerequisites only):**
```
Foundation, no ADR prerequisite:
  1. ADR-0001 kernel tick driver and frame order   (gate: KRN-LAT-02, GDD tick-source sync)
Depends on 0001:
  2. ADR-0002 authority model and command pipeline
Depends on 0001, 0002:
  3. ADR-0003 state-block registry and owner contract   (fix E1 first, below)
Depends on 0001-0003:
  4. ADR-0004 change notification and dirty keys
  5. ADR-0007 static content data pipeline   (gate: CONTENT-EXP-1, passed)
Depends on the above:
  6. ADR-0005 determinism, keyed RNG, hashing   (0001, 0003, 0004; spikes S0-a..e)
  7. ADR-0010 source layout, typing and lint   (0001-0003, 0007; LINT-SPIKE-1)
Depends on the above:
  8. ADR-0006 save container   (0001-0003, 0005, 0007, 0010; SAVE-WRK-1-ACCEPT)
  9. ADR-0009 presentation subscriber contract   (0001-0004, 0010; KRN-LAT-02 order half)
Co-accepted with 0009:
 10. ADR-0008 render pipeline wiring   (0001, 0007, 0009, 0010; pipeline spike)
```
⚠️ Every edge is unresolved, because every ADR is still Proposed. No ADR can be safely implemented until its prerequisites are Accepted.

### GDD Revision Flags

Engine-reality check (Phase 5b): **no GDD revision flags.** The one HIGH engine finding (E1, `PresentationViews.get`) is ADR text only; no GDD names it. E2 (`Engine.time_scale` scales `_process` delta, probed) is consistent with the only GDD mention (`debug-overlay-true-values.md:499`, tick-driven). The systems index is unchanged.

**GDD text contradicted by Proposed ADRs: GDD sync required before acceptance.** These are not revision flags in the skill's sense, because no ADR is Accepted. The ADR has deliberately changed the design and the GDD (and its registry row) has not followed. Each must be synced, or the ADR revised, before that ADR is Accepted. 31 requirements, ⛔ in the matrix:

| ADR | Requirements | What the ADR decided |
|---|---|---|
| ADR-0003, ADR-0006 | TR-sanity-001, TR-saveload-033, TR-lsg-023, TR-suit-009, TR-shipbody-009, TR-attention-024 | owner `restore()` returns `CompletionToken` and reports failure through `ctx.fault(...)`, never `(ok, reason, detail)` |
| ADR-0005 | TR-cmd-040, TR-lsg-021, TR-emitter-038, TR-debug-033 | the kernel encodes and hashes every persist block; hash cache validated by ADR-0004's touch serial, not `mark_dirty`; no owner-written `canonical_bytes()` / `block_hash()` |
| ADR-0006 | TR-cmd-018, TR-saveload-020 | epoch in two alternating files `slot_epoch_a/b.bin` with `seq`, plus a read-only recovery copy in the container header |
| ADR-0006 | TR-lsg-022, TR-saveload-017, TR-sanity-020, TR-saveload-003 | payloads stored with `var_to_bytes`, native int64 allowed in payloads; `MAX_DECODE_NODES` provisional 100 000; owner raw-bytes form dropped |
| ADR-0007 | TR-section-033, TR-hud-026, TR-debug-044 | published Tuning Knob ranges are guidance, never boot assertions; out-of-schema values refuse (`CONTENT_FIELD_RANGE`), never clamp |
| ADR-0007 | TR-lsg-030, TR-itemdb-001, TR-section-003, TR-section-040 | content loads at start-up before the kernel's BOOTING. **Registry text revised in this review** (GDDs already synced in `1fefb7e`). |
| ADR-0008 | TR-pxl-002, TR-pxl-014, TR-pxl-015, TR-pxl-025, TR-pxl-030 | menus on layers 3-4 and debug on 5; stretch disabled (no inverse cursor transform); kernel `snap_baseline` + camera CUT on `load_seq`; pass B sized to the picture; `D_root` ≤ 63 |
| ADR-0009 | TR-pc-039 | per-frame placements vary with alpha; byte identity across 30/60/120/144 fps holds for per-tick samples only |
| ADR-0010 | TR-emitter-032, TR-pc-027 | work counter injected (`NullWorkCounter` in shipped builds), not compiled under a flag; SIM-02 allow-list grants no render-timer exemption |
| ADR-0001 | TR-cmd-004 (wording) | priorities are -100 / -50 / -40 HOST_SERVICE / 0, not 0 / 1 / 2 |

The queued sync list from the Foundation ADR runs (`foundation-gdd-sync.json`, 132 items) still has **96 items whose old text is present** in the target file: 19 in `architecture.md`, 20 in the Pixel-Perfect Pipeline GDD, 16 in Save/Load, 14 in the kernel GDD, 13 in `tr-baseline.md`, 9 in Ship Body, and the rest spread across 12 files. 31 items no longer match (applied or re-worded; they need re-anchoring) and 5 have no single target file. Applying this list is the "GDD sync" acceptance gate each ADR names.

### Engine Compatibility Issues

**Engine Audit Results**
Engine: Godot 4.7.2 (GDScript)
ADRs with an Engine Compatibility section: 10 / 10
Deprecated API references: none. The grep hits (`OS.get_ticks_*` in ADR-0005 l.165 and ADR-0010 l.209/744; `duplicate()` in ADR-0003/0005/0006) are ban lists or valid shallow ownership copies of packed arrays, confirmed by the specialist.
Stale version references: none. All ten name 4.7.2.
Post-cutoff API conflicts: none between ADRs. `@abstract` (4.5) is used by ADR-0010 for ports only and avoided by ADR-0003 for owner callables (consistent: ADR-0010 states there is no `BlockOwner` port). Typed `Dictionary[K, V]` (4.4) in ADR-0004/0007/0010 versus the untyped payload boundary in ADR-0002 is reconciled by the `dictionary_variant.json` allow-list, which is not yet written (E5). `duplicate_deep()` (4.5) in ADR-0005/0006; `FileAccess.store_*` returning `bool` (4.4) in ADR-0006.

### Engine Specialist Findings
godot-specialist (Sonnet), with four headless probes on the installed 4.7.2 binary:

| Id | Severity | ADR | Finding | Confidence |
|---|---|---|---|---|
| E1 | **HIGH** | ADR-0003 l.340; ADR-0009 l.119/570; ADR-0010 OQ-K | `PresentationViews.get(block_id)` overrides native `Object.get()`. On 4.7.2 this is a parse error ("overrides a method from native class Object", warning treated as error) and the script fails to load. ADR-0010 l.146 itself bans accessors named `get` (`NATIVE_METHOD_OVERRIDE`). Fix: rename to `get_view` or `lookup` in ADR-0003 § 7 and ADR-0009. | HIGH, probed |
| E2 | MEDIUM | ADR-0001 l.27/51/238 | `Engine.time_scale` does scale `_process` delta (headless probe: 1.006 / 0.502 / 2.010 s per 1.0 s at scale 1 / 0.5 / 2). The ADR's mitigation (pin 1.0, `TIME_SCALE_NON_UNIT`) is right; mark item (2) probed (headless), release export still open. | HIGH, probed |
| E4 | MEDIUM | ADR-0010 IO-02 l.309/391/878 | "handed over by value" is wrong for packed arrays, which are shared by reference (ADR-0006 probe 16). It should say "ownership transferred (no retained alias) or `.duplicate()`d". | HIGH |
| E5 | MEDIUM | ADR-0010 SIM-09 / OQ-H vs ADR-0002 | SIM-09 bans bare `Dictionary` in sim layers; the payload boundary needs untyped `Dictionary`. Reconciled only by the unwritten `dictionary_variant.json`; also `Dictionary[K,V].keys()` returns untyped `Array` under the bare-`Array` ban. | HIGH (reading) |
| E3 | LOW | ADR-0006 probe (1) | Windows rename-over-existing is likely remove-then-rename; the ADR already claims no atomicity. Keep it UNVERIFIED until a kill-between test. | MED |
| E6 | LOW | engine reference | The reference says D3D12 is the Windows default; `driver.windows` returns `vulkan` on 4.7.2 (probed). Doc defect in `docs/engine-reference/godot/`; ADR-0008 already pins and asserts. | HIGH |
| E7 | LOW | ADR-0003 l.346 | `const OK: int = 0` shadows the global `OK`; it loads with `shadowed_global_identifier=2`; add a LINT-SPIKE-1 fixture under the real § 5.1 warning set. | MED |
| E8 | LOW | ADR-0006 l.92 vs ADR-0010 HOT-01 | `run.bind(...)` on the save worker is outside the `## @hot` scope; confirm `SaveService` is never marked hot. | — |

Open probes still UNVERIFIED (known, not new): KRN-LAT-02 on a release export and `_input` ordering (ADR-0001); `frame_pre_draw` ordering on release / D3D12 / Steam Deck (ADR-0001/0008/0009); S0-a..d on `template_release` and Linux / Deck (ADR-0005); Windows rename, `Thread` exit and invalid UTF-8 (ADR-0006); pipeline spike steps 11-12, display scaling 125 / 150 % (ADR-0008); driver overhead on RM-DECK (ADR-0009); LINT-SPIKE-1 (a)(c) (ADR-0010). CONTENT-EXP-1 (ADR-0007) passed.

### Architecture Document Coverage

- **Missing systems:** none. All 26 MVP systems in `systems-index.md` appear in `architecture.md`'s layer table; the 16 non-MVP rows are absent by design.
- **Orphaned architecture:** none. Infrastructure without a GDD (composition root, KernelHost / TickDriver / SaveHost, PipelineRoot / PresentationDriver, the ADR-0010 lint tool and ports, tuning tables, `ids.lock.json`, ADR-0033 debris physics) is accounted for.
- **Data-flow gaps (24 GDD edges not drawn):** the largest are the carry-burden → Attention push (`U_BURDEN`); Chart Table → Pipeline `PAUSED_FROZEN` / `TRANSIT_HOLD` and the console-pick seam; Suit `create_seat_beam` / `bind_beam` at new_game (missing from init order § 8); Audio's reads beyond Graph / Attention / Horror / Filter; the restore-event no-sound rule; Instruments `subject_anchors`, `inspect_ref` and `INSTRUMENT` subjects; Derelict Authoring → F&T dried trails, Lighting, Web Map, Body spawn table, Attention beacon; `controller.noise` → Attention. There is no section for presentation-to-presentation lateral edges (HUD ↔ Instruments flash arbiter, Chart Table component grammar).
- **Stale API boundaries:** 19 queued sync items still unapplied in `architecture.md`, plus: `request_restore` lacks `has_expected_hash` (l.389); the frame diagram lacks HOST_SERVICE -40 and shows presentation at "priority 2" (ADR-0001: 0); § ADR Audit still says "No ADRs exist yet / all 794 uncovered" (l.577-581); API examples use a global `Kernel.tick_alpha` rather than injected facets; `subscribe` / `_on_kernel_changes` signatures predate ADR-0004/0009. Missing boundaries: the ContentLoader API (ADR-0007 `load_all()`) and the PipelineRoot API (`world_to_hud`, `mouse_to_world_*`).

---

### Verdict: FAIL

The rule applied is the skill's literal FAIL trigger, "Critical gaps (Foundation/Core layer requirements uncovered)". The **Foundation set is complete**: ten of ten written, with 351 of 440 Foundation-planned requirements covered or partial. **Core is not**: ADR-0011..0019 and ADR-0034 are unwritten, leaving 200 Core-layer requirements as gaps. There is also one HIGH engine finding (E1) that will not load on the pinned engine.

This is the expected state for this point in Technical Setup, and markedly better than the previous run (gaps 1038 → 770; Proposed coverage 50 → 181). Without the Core-layer rule, the remaining findings would cap the verdict at CONCERNS: every ADR is Proposed, and there is no blocking cross-ADR conflict.

### Blocking Issues (must resolve before PASS)
1. **Write the Core ADRs** ADR-0011..0019 and ADR-0034 (200 Core requirements).
2. **E1:** rename `PresentationViews.get` (ADR-0003 § 7, ADR-0009 l.119/570) before ADR-0003 is Accepted. It is a load failure on 4.7.2.
3. **Accept the Foundation ADRs** in the order above (`/architecture-decision accept ADR-NNNN`), each after its own gate: GDD sync (96 queued items + the 31 ⛔ rows), its spikes, and the residual fixes below.
4. Before acceptance, also fix these text issues:
   - residual 2 (ADR-0010 sorted-key file and names);
   - the ADR-0006 → ADR-0001 quit-transition owner;
   - move the soft dependency edges out of the Depends On cells so `adr-dep-graph.sh` reports no cycle;
   - E4 ("by value" wording);
   - E5 (write `dictionary_variant.json` contents, OQ-H).

### Required ADRs
1. ADR-0011 Life-Support Graph data model (Core, 17)
2. ADR-0012 Graph propagation scheme (Core, 26)
3. ADR-0013 Ship layout block, geometry and ID allocation (Core, 27)
4. ADR-0016 Restore ordering and cross-block verification (Core, 19)
5. ADR-0015 Integer kinematics and the ship body (Core, 30)
6. ADR-0014 Geometry broadphase and OccluderSource seam (Core, 13)
7. ADR-0017 Emitter state and light-exposure query (Core, 50)
8. ADR-0019 Player controller split and input contexts (Core, 36)
9. ADR-0034 PROJECTOR scene shape and Y-sort (Core; move its rows from ADR-0009)
10. ADR-0018 Shadow origin and beam occlusion (Core, 1; may fold into ADR-0028)
11. Then Feature / Presentation ADR-0020..0031 (469), largest first: ADR-0024 Perception Filter (86), ADR-0028 lighting (72), ADR-0030 UI (52), ADR-0020 interaction (49), ADR-0031 diagnostics (48), ADR-0026 world model (46), ADR-0029 audio (44).

---

### Post-review fixes (2026-10-28, same session, user chose "Fix the ADR issues")

Applied after the verdict above; the verdict and counts are unchanged. The receipt lines above were re-stamped for the seven edited ADRs, so the next freshness check treats these wording fixes as reviewed.

| Finding | Fix | Files |
|---|---|---|
| E1 (HIGH) `PresentationViews.get` overrides `Object.get()` | accessor renamed `get_view(block_id)` on `SimViews` and `PresentationViews` | ADR-0003 § 7, Key Interfaces, Validation Criteria; ADR-0009 l.570; `docs/registry/architecture.yaml` stance |
| Residual 2 (MEDIUM) ADR-0010 sorted-key file and names | SIM-09 / DET rows, the `src/shared/` contents row and OQ-N(2) aligned to § 5.3 (`CanonicalEncoder.sorted_int_keys` / `sorted_string_keys` in `canonical_encoder.gd`) | ADR-0010 |
| New conflict (MEDIUM) ADR-0006 cites an ADR-0001 quit transition | re-attributed to ADR-0003 `SHUTTING_DOWN`, entered through ADR-0002 `request_shutdown` | ADR-0006 Depends On and window-close paragraph |
| Dependency cycles (soft edges) | ADR-0010 moved to a "Soft references" row in ADR-0004 and ADR-0005; ADR-0005 moved to a "Validation follow-up" row in ADR-0007 (and its Validation Criteria wording) | ADR-0004, ADR-0005, ADR-0007, ADR-0010 Related |

`adr-dep-graph.sh` after the fix: 10 ADRs, no cycle, order 0001, 0002, 0003, 0004, 0007, 0005, 0010, 0006, 0009, 0008. Registry validate clean; `yaml.safe_load` parses.

Still open from Blocking Issues: the Core ADRs (1), acceptance with its gates (3), E4 "by value" wording and E5 `dictionary_variant.json` contents, the 10 LOW residuals, the ADR-0006 vs ADR-0008 frame-budget row, and the GDD sync (96 queued items + 31 ⛔ rows).
