# Phase 6: architecture document coverage (read-only)

Sources: docs/architecture/architecture.md (layer map L117-146, ownership L158-206, data flow L250-418, API L421-571, ADR audit L575-623, Required ADRs L627-699), design/gdd/systems-index.md rows 1-42, 26 MVP GDD Dependencies sections (sampled: audio, derelict authoring, debug overlay, boarding, instruments, chart table), gdd-sync-check.json.

## 1 Missing systems

None for MVP. All 26 MVP index rows (#1-#26) appear in architecture.md "Layer assignments" (L119-146), one row each. Boarding & Salvage (#15) appears as "Feature (design rules), no code module" (L138), which is deliberate.

Not in architecture.md at all (expected, non-MVP; 16 rows): #27 Procedural Derelict Generation, #28 Depth Rings & Scarcity, #29 Resource Depletion & Wear, #30 Knowledge Codex & Logs, #31 Failure Contract & Re-wake, #32 Environmental Storytelling Placement, #33 Co-op Session & Replication (only as ADR-0032 deferred), #34 Chat Voice & Pings, #35 Crew Strip HUD, #36 Onboarding, #37 Sanity Scars, #38 Ship Capability Progression, #39 Steam Lobby & NAT, #40 Host-Owned World, #41 Accessibility Settings, #42 Settings & Input Rebinding (MVP SettingsStore lives under Chart Table, L144, no #42 row mapping).

Layer-label disagreements between index Dependency Map and architecture (labels differ by design: dependency order vs responsibility, L72-74; flag only): Save/Load index Core vs arch Foundation; Item DB index Feature vs arch Foundation(data); Emitter index Feature vs arch Core; Lighting/Perception Filter/Failure & Trail index Feature vs arch Presentation; Debug Overlay index Polish vs arch Presentation(tooling).

## 2 Orphaned architecture

No orphan with a GDD gap. Every module in Module Ownership maps to an index row.

Infrastructure with no GDD/index row (listed separately):
- Composition root `src/game/game.gd` (ADR-0001 5; architecture mentions only as a reference to ADR-0001, no ownership row).
- `KernelHost` / `TickDriver` / `SaveHost` host nodes (ADR-0001; arch names only TICK_DRIVER). They belong to the Kernel/Save GDDs.
- `PipelineRoot` / `PresentationDriver` single presentation host (ADR-0009/0008). Belongs to Pipeline GDD.
- Lint tool + `ProcessPriority` constants + `@abstract` ports (ADR-0010). No GDD.
- Content data module `src/data/` composite: ItemCatalog (#3) + Derelict content (#14) + verb/tuning tables. Mapped to #3/#14, but "tuning tables" and `ids.lock.json` have no single GDD owner.
- Observer log `src/observer/` (Debug Overlay #26 owns it).
- Layout PROJECTOR / `body.projector` (belong to #11/#12 GDDs; ADR-0034 is the planned ADR; architecture row L205 shows them under Presentation without an index row of their own).
- ADR-0033 presentation-only debris physics (ADR-BODY-05): no GDD row, VS-deferred, 0 TRs.
- Layout "Avatar Presenter + Input Adapter" are halves of #7 (split by design).

## 3 Data flow gaps

architecture.md Data Flow sections 1-8 cover: frame/tick, command path, weld, notifications, perception, save, load, init. Sampled GDD edges absent (grep count 0 unless noted) from Data Flow / API Boundaries, most significant first (24 listed, under the cap):

1. Boarding/Interaction carry burden -> Attention (`U_BURDEN`, per-tick BURDEN push while salvage held aboard non-home locus body; PC-10/D.13, BS row 6/8, Attention F-26). "burden" appears 0 times. Edge is not in Interaction's Consumes (L185) or Attention's (L189).
2. Chart Table menu -> Pipeline states `PAUSED_FROZEN` / `TRANSIT_HOLD` (CT dependency row 2; PR-13). 0 hits; arch only shows PauseGate and `request_pause` once.
3. Chart Table / Adapter console seam: host-local pick injected into `pod_hop`/`berth`; no-pick press routes to "open chart" (CT-10). Only `ChartPick` name appears (L144); no flow.
4. Suit new_game seat defaults: `create_seat_beam`, `bind_beam` (SE-9 -> Chart Table/new_game). 0 hits; init order (section 8) lacks the seat-beam step.
5. Audio hard/soft reads beyond Graph/Attention/Horror/Filter (arch L203): Suit cues, Controller gait/`channel_view`, Interaction `ia_*`/`result_seq`, Assembly `last_result`/seam-cut loop, Body `BODY_*`, Web `POD_*`/`BERTHED`, Layout `anchor_geometry`/`hull_anchor`, Graph `opening()` transmission, Emitter `E/<id>`, Save/Load post-restore silence rule (SL-7). Arch Audio row lists only four consumes.
6. Audio post-restore notification / restore event (SL-7 no-sound rule). 0 hits for restore-event delivery to presentation.
7. In-World Instruments: Layout/Derelict `subject_anchors(section_id)`, Item DB `sprite_family`/state tokens, Suit O2 sight-glass `stock`, Web home fuel tank (WM-20). 0 hits for `subject_anchors`.
8. Instruments <- Controller `{context, inspect_ref}` Adapter signal and Pipeline `present_time`. Arch Instruments row consumes only Filter and Graph descriptors.
9. Instruments -> Interaction `INSTRUMENT` subjects / `verb_inspect` (only `card_opened` signals listed).
10. Derelict Authoring -> Failure & Trail dried trails (`DRIED_AFTER_STEPS`), heat-presentation step (ASK-F&T-H1); Derelict -> Lighting (`SHADOW_SLOTS_MAX`, `CONTAIN_COLOURS`). 0 hits for "dried".
11. Derelict Authoring -> Interaction `place_loose` seeded loose ledger (init order step 6 lists `sync_section`, not `place_loose`).
12. Derelict Authoring -> Web Map `residual_lamp_predicate`, `pod_body`, `mvp_world.json`, `visible_unknown`. 0 hits.
13. Derelict Authoring -> Ship Body spawn table / `can_place` / `pod` template (Body consumes list L177 lacks it).
14. Derelict Authoring -> Attention beacon data row (DA-24) and EMERGENCY cryo lamp (DA-23); `room_adjacency` / `route_length_px` consumed by Attention and Horror (Attention consumes L189 omits Layout).
15. Boarding -> Save/Load autosave soft inputs and polled `last_salvage_T` (covered: L167, L185). Not a gap, listed for completeness.
16. Controller `controller.noise` hook -> Attention stimulus (hook listed L289; the push edge to Attention is not drawn; Attention consumes "Controller loco_state" only, L189).
17. Camera framing "never-shift anchor" for Perception Filter / Pipeline `PAUSED`/`TRANSIT_HOLD` camera CUT: only "camera CUT" at load (L394).
18. HUD <- Controller prompts / hand rail (PCI soft edge); arch HUD row consumes Suit, Interaction `focus_view`, pipeline only.
19. Debug Overlay reads Save/Load `save_status` + `state_hash`, Web `transit_stale_cmds`, Assembly verdict-drift counters: generic "every DIAGNOSTIC channel" (L206), not edge-listed. Acceptable but undrawn.
20. Perception Filter <-> Audio request `(viewer, ref|position, intent, class)` and `FILTER_AUDIO_CLASSES`: covered at L197/L203 by name only; no flow diagram section.
21. Web Map -> Layout/Graph/Body/Emitter covered. Web -> Attention "Web Map ring" for Sanity covered (L191). Web `pod_hop` fuel -> Graph M16 covered. Not gaps.
22. Failure & Trail -> Pipeline layer stack / `H` and F&T <- Layout anchors: partly (L199). F&T `deposit_trace` Horror -> Graph M17 covered (once).
23. Settings (MVP four-control page) / reduced-motion + reduce-flashing hooks (Pipeline PR-13) -> Lighting flash governor / HUD arbiter: `SettingsStore` named twice, no flow.
24. Item DB <-> Audio constraint (IDB:991) and Item DB -> Instruments: Item DB "all Core and Feature modules read Content data" (L245) covers generically.

Data-flow structure gap: no section for the "Presentation reads from presentation" lateral edges (HUD <-> Instruments shared Theme/flash arbiter `HONEST_TICK_MS`; Chart Table shared component grammar). Note also OQ-1 (three GDDs still cite the physics callback) is documented (L254).

## 4 API boundary / stale text

Count of known-stale architecture.md entries in gdd-sync-check.json: 22 entries for path docs/architecture/architecture.md: 19 OLD_PRESENT (stale), 3 old_absent (entries 0, 9, 10: anchor text no longer present, need re-anchoring before apply). By ADR: ADR-0001 1 stale (+1 old_absent), ADR-0004 4, ADR-0009 3 (+1 old_absent), ADR-0006 0 (+1 old_absent), ADR-0010 6, ADR-0008 5.

Sample confirmed still present by exact-string grep in architecture.md (each count = 1): (1) "presentation runs at the engine default 0" (ADR-0001), (2) "func subscribe(cls: SubscriberClass, handler: Callable)" (ADR-0004; ADR-0004 now says `subscribe(block_id, handler, name)`), (3) "_redraw(Kernel.tick_alpha)" (ADR-0009 PresentationDriver replaces it), (4) "ADR-0010 records the GDExtension escape hatch policy" (ADR-0010 section 9 now Proposed), (5) "RenderingServer Forward+, AudioServer" (ADR-0008 provisional wording), plus (6) "priority 2  Presentation subscribers". All 6 verified stale.

Additional stale text not in the sync JSON (found by reading):
- L389 `request_restore(payload, restore_epoch, expected_state_hash)`: ADR-0002 L203 is `request_restore(payload, restore_epoch, expected_state_hash: int = 0, has_expected_hash: bool = false)`, verdict NONE when false (TD-05). Stale.
- L263-264 / frame diagram: no HOST_SERVICE (-40) SaveHost between driver (-50) and presentation (JSON entry 0 old_absent: block text changed already or anchor differs). L275 "priority 2" vs ADR-0001 PRESENTATION 0.
- L577-581 "ADR Audit: No ADRs exist yet" and "All 794 uncovered": stale, ten Proposed ADRs exist.
- L557 mentions ADR-0001 Proposed facets generically, but API examples still use global `Kernel.tick_alpha` and `PresentationClock` is not in signature blocks; ADR-0001 names `NotifyFacet`, `ModuleFacet`, `setup(facet)`.
- L441, L519: subscribe/_on_kernel_changes signatures (ADR-0004/0009), and `owner callable returns ctx.complete()` rule extended to subscribers (ADR-0009: `_on_changes(batch, ctx) -> CompletionToken`).
- L402-405, L413: init order says "Pipeline root builds viewport stack (BOOTING -> PRESENTING)" then "Content data loads" step 3 then "Kernel BOOTING" step 4. Step 3 text already says composition root loads content before the kernel (ADR-0007 section 1.7 / 2.3), consistent. The remaining BOOTING reference (L405) "BOOTING -> PRESENTING" is the Pipeline state name, not the kernel. Needs only a disambiguation note; ADR-0007 L418 explicitly rules "loaded in BOOTING" means start-up before the kernel for GDD wording (Layout LY-2/LY-30 and IDB-30), architecture L302/L409 wording is fine.
- Interface types table (L546-555) points to ADR-0003/0004/0011-0015/0024 as defining; ADR-0004 now defines ChangeBatch; others remain planned.
- P3 hook schedule (L284-300) matches (no sync entry).
Integration requirement support: boundaries cover kernel ingress, owner callables, mutators, read views, Filter, subscriber, Save/Load. Missing boundaries: Content data loader API (`ContentLoadResult load_all()` per ADR-0007 L246), Pipeline root API (`world_to_hud`, `mouse_to_world_*`) in a signature block, Avatar Presenter/Adapter MenuInput stream, Audio `cue_requests()`, Chart Table PauseGate/`SettingsStore`.

## 5 Required ADR layer table

Document labels (architecture.md L631, L646, L661, L678): "Must have before coding starts (Foundation)"; "Must have before the relevant Core system is built"; "Should have before the relevant Feature or Presentation system is built"; "Can defer (not MVP)". TR count = "Primary TRs" in the ADR table (L587-623); shared in brackets. Total 794 primary, 15 shared.

Foundation (ADR-0001..0010, all ten are Foundation):

| ADR | title | layer | TR count |
|---|---|---|---|
| ADR-0001 | Kernel tick driver and frame order | Foundation | 13 |
| ADR-0002 | Authority model and command pipeline | Foundation | 11 (+3 shared) |
| ADR-0003 | State-block registry, write discipline and owner-callable contract | Foundation | 24 |
| ADR-0004 | Change notification and dirty keys | Foundation | 13 |
| ADR-0005 | Determinism, keyed RNG, canonical encoding and hashing | Foundation | 19 (+2) |
| ADR-0006 | Save container, generations, versioning and restore epoch | Foundation | 65 |
| ADR-0007 | Static content data pipeline and stable type IDs | Foundation | 64 (+1) |
| ADR-0008 | Render pipeline wiring and integer scaling | Foundation | 26 |
| ADR-0009 | Presentation subscriber contract | Foundation | 27 (v2; rows re-sorted vs ADR-0034 at review) |
| ADR-0010 | Source layout, static typing and lint enforcement | Foundation | 6 |

Planned ADR-0011..0034:

| ADR | title | layer | TR count |
|---|---|---|---|
| ADR-0011 | Life-Support Graph data model | Core | 11 |
| ADR-0012 | Graph propagation scheme | Core | 17 |
| ADR-0013 | Ship layout block, geometry and ID allocation | Core | 14 |
| ADR-0014 | Geometry broadphase and OccluderSource seam | Core | 5 (+1) |
| ADR-0015 | Hand-rolled integer kinematics and the ship body | Core | 24 |
| ADR-0016 | Restore ordering and cross-block verification | Core | 15 |
| ADR-0017 | Emitter state and integer light-exposure query | Core | 32 (+2) |
| ADR-0018 | Shadow origin and beam occlusion consistency | Core | 1 (folds into ADR-0028 if lighting spike settles both) |
| ADR-0019 | Player controller split and input contexts | Core | 21 (+1) |
| ADR-0020 | Interaction channels and verb declaration table | Feature/Presentation | 34 |
| ADR-0021 | Atomic weld/cut/install transaction | Feature/Presentation | 20 (+2) |
| ADR-0022 | Attention model, stimulus API and breach state machine | Feature/Presentation | 28 |
| ADR-0023 | Horror entity simulation | Feature/Presentation | 18 |
| ADR-0024 | Perception isolation and the Perception Filter | Feature/Presentation | 64 |
| ADR-0025 | Sanity model and lie lifecycle | Feature/Presentation | 19 |
| ADR-0026 | World model, derelict loading and transit | Feature/Presentation | 30 (+1) |
| ADR-0027 | Suit vitals and equipment | Feature/Presentation | 14 |
| ADR-0028 | 2D lighting budget and modifier chain | Feature/Presentation | 52 (+1) |
| ADR-0029 | Audio architecture | Feature/Presentation | 30 |
| ADR-0030 | UI framework | Feature/Presentation | 39 |
| ADR-0031 | Diagnostics, observer and dev-feature gating | Feature/Presentation | 33 |
| ADR-0032 | Replication shape for co-op (VS-B) | Can defer (not MVP; before VS-B) | 5 (+1) |
| ADR-0033 | Presentation-only debris physics (VS) | Can defer (not MVP; before VS) | 0 |
| ADR-0034 | PROJECTOR scene shape and Y-sort (split from ADR-0009 in v2) | Core (listed in Core table L659; needed before the PROJECTOR story; gated by SP-YSORT-1) | 0 (rows move from ADR-0009 at review) |

Notes: the document does not split "Feature" from "Presentation" for ADR-0020..0031 (single tier). Natural reading: Feature = 0020-0023, 0025-0027; Presentation = 0024, 0028-0031 (inference, not in the doc). Section header "Thirty-four decisions" is consistent with 0001..0034. Sectioned multi-decision ADRs: 0006, 0007, 0024.
