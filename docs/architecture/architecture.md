# The Synaptic Sea — Master Architecture

## Document Status

- Version: 3 (v1 written 2026-10-28; v2 the same day after the TD-ARCHITECTURE and LP-FEASIBILITY reviews; v3 2026-10-29 syncs the text to Foundation ADR-0001..0010, all Proposed)
- Last Updated: 2026-10-29
- Engine: Godot 4.7.2 (stable, mono build installed; GDScript only), renderer Forward+ (provisional, with the Vulkan driver on Windows; ADR-0008, reversible at the lighting spike)
- Scope: the MVP (single-player loop test). Vertical Slice and Alpha systems appear only where an MVP decision has to leave room for them (co-op replication, procgen).
- GDDs Covered: all 26 approved MVP GDDs in `design/gdd/`: command-simulation-layer, life-support-graph, item-and-module-database, pixel-perfect-render-pipeline-camera, emitter-state-and-light-exposure-query, 2d-lighting-and-occlusion, player-controller-input, interaction-system, failure-and-trail-system, suit-and-equipment, ship-section-and-layout-data-model, ship-body-and-interior-physics, modular-ship-assembly, derelict-authoring-handmade-kits, boarding-and-salvage, web-map-and-transit, attention-system, horror-entities, sanity-and-hallucination-rules, perception-filter, hud-and-suit-layer, in-world-instruments-and-inspect-view, chart-table-web-map-and-menus, adaptive-audio-and-tells, save-load-and-versioned-format, debug-overlay-true-values
- Technical Requirements Baseline: `docs/architecture/tr-baseline.md` (794 requirements in the original extraction, registered by `/architecture-review`; the registry `docs/architecture/tr-registry.yaml` now holds 1138 after the completeness pass, see § Traceability coverage). **Known limit:** extraction capped most GDDs at about 30 requirements, so the baseline is a representative architecture-bearing set, not a complete one. `/architecture-review` must run a completeness pass (Acceptance Criteria sections at least) before it registers IDs, or requirements outside the baseline will have no TR-ID for stories to trace.
- ADRs Referenced: ADR-0001..ADR-0010 (the ten Foundation ADRs) are written, all Proposed, none Accepted; ADR-0011..ADR-0034 are planned (§ Required ADRs). Review: `docs/architecture/architecture-review-2026-10-28.md` (verdict FAIL, the expected state while the Core ADRs are unwritten and no ADR is Accepted). *(Amended 2026-10-29 — ADR sync)*
- TD-ARCHITECTURE: 2026-10-28 — REVISED after CONCERNS (v1 verdict CONCERNS: no structural flaw; Foundation ADRs unwritten and five gated by spikes; OQ-1 understated; hook-table stale markers; dependency direction unstated; ADR-0009 to split. v2 folds in items 1-9; the rest belong to the named ADRs.)
- LP-FEASIBILITY: 2026-10-28 — REVISED after CONCERNS (v1 verdict CONCERNS: implementable in GDScript by one programmer if performance is measured early; undefined interface types and FAULT_CODE table, injection vs autoload, snapshot hitch vs encode budget, no-rollback weld, unverifiable allocation and lint claims, three oversized ADRs. v2 records each against an ADR or open question and adopts the spike order.)

### How to read this document

The GDDs already fix most of the architecture: each one names its state block, its step hook (divisor, offset, `after:`), its mutators and its read view, and the kernel GDD fixes the tick phases and the write rules. This document does not re-decide any of that. It assembles those contracts into one map, names the layers and boundaries they imply, records the decisions that still belong in ADRs, and lists the conflicts left between GDDs. Where this document and a GDD disagree, the GDD is the source and this document is wrong; raise it with `/propagate-design-change`. Where two GDDs disagree, the **owner** GDD wins over the consumer (delegated rule G1: the registry `source:` GDD, else the publisher of the field, signal or hook). This document follows the owner and routes a fix to the consumer; that is how OQ-1 is handled, not this document overriding a GDD.

Every section has `##`/`###` headings so downstream skills can read it by section.

---

## Engine Knowledge Gap Summary

**Engine:** Godot 4.7.2. **LLM training covers:** up to about 4.3 (4.4 thinly). **Post-cutoff versions:** 4.4 (Mar 2025, verify), 4.5 (Sep 2025, HIGH), 4.6 (Jan 2026, HIGH), 4.7 (Jun 2026, HIGH). Always check `docs/engine-reference/godot/` before naming an API.

### HIGH risk domains (verify against the engine reference before deciding)

| Domain | Post-cutoff change that matters here | Where it bites |
|---|---|---|
| GDScript | 4.7: an override of a typed method inherits its return type and needs an explicit `return`. 4.7: writing one element of a packed array no longer calls the property setter. 4.5: `@abstract`, variadic args. | Every kernel-invoked owner callable ends `return ctx.complete()` (ADR-0003). Every block built on `PackedInt*Array` columns marks dirty keys explicitly inside its mutators, never through a setter (ADR-0004). `OccluderSource` may use `@abstract` (ADR-0014). |
| Rendering (2D) | 4.6: glow runs before tonemapping. 4.6: D3D12 is the Windows default (was Vulkan). 4.7: `CanvasItem` lines lose their antialiasing feather. 4.7: HDR output. 4.5: Shader Baker. 4.7: new projects default to `canvas_items` / `expand` stretch. | Emission glow threshold (Lighting LO-14), the renderer backend on RM-DESKTOP (ADR-0008), instrument needles and hairlines drawn as lines (Instruments, F&T), startup guard rails that pin the stretch block (Pipeline PR-28). |
| Input | 4.7: mouse and keyboard report `InputEvent.DEVICE_ID_MOUSE` / `DEVICE_ID_KEYBOARD`, not device 0. | The Input Adapter identifies devices by event type; `device == 0` is lint-forbidden (ADR-0019). |
| UI | 4.6: dual focus (mouse/touch focus separate from keyboard/gamepad focus). 4.5: AccessKit, `FoldableContainer`, recursive Control disable. 4.7: `accessibility_live` enum moved. | Chart table, pause panel and inspect card focus handling (ADR-0030). |
| SubViewport wiring | `size_2d_override`, `size_2d_override_stretch`, `use_hdr_2d` and Light2D behaviour under them are not sourceable in the pinned reference. | The integer-scale world viewport and lighting under zoom (ADR-0008, ADR-0028); spike steps 11 and 12 of the pipeline and steps 2-4 of the lighting spike. |
| Release-export performance | Every wall-clock budget is measured on a 4.7.2 release export on RM-DESKTOP and RM-DECK (delegated ruling R12). | All performance ACs; nothing measured in the editor or on the developer machine counts. |

### MEDIUM risk domains (verify key APIs)

| Domain | Change | Where it bites |
|---|---|---|
| Core / file I/O | 4.4: `FileAccess.store_*` return `bool`. | Save/Load checks every store return (SL-10). |
| Resources | 4.5: `duplicate_deep()`. | Snapshots are built as explicit deep copies; never rely on `duplicate()` for nested data. |
| Audio | 4.7: `AudioStreamPlayer2D.area_mask` defaults to 0; `AudioEffectSpectrumAnalyzer.tap_back_pos` removed. | Any `Area2D` audio bus override must set the mask explicitly (ADR-0029). |
| TileMapLayer / Y-sort | `TileMapLayer` (4.3) and its Y-sort tie rule under reparenting are not verified for 4.7. | PROJECTOR scene shape (ADR-0034, spike SP-YSORT-1). |
| Networking | No 4.5-4.7 breaks listed, but peer-ID assignment on 4.7.2 is not sourceable. | Dormant until VS-B (ADR-0032). |

### LOW risk domains

Core `Node` lifecycle, `RefCounted`, `HashingContext` (SHA-256; allowed only in `src/save`, `src/data`, `src/shared/det/hash64.gd` and `src/kernel/state_hasher.gd`, ADR-0010 IO-01), `Marshalls`, JSON, `Thread`, `Camera2D`, `Light2D` / `LightOccluder2D` base APIs, Godot Physics 2D (unchanged in 4.6/4.7 and not used on any simulation path anyway).

### Systems that touch HIGH or MEDIUM risk domains

| System | Domain | Risk |
|---|---|---|
| Command & Simulation Layer | GDScript typed returns, packed arrays, `_process` ordering | HIGH |
| Life-Support Graph, Ship Section & Layout, Ship Body, Emitter State, Suit, Sanity, Attention, Horror | GDScript packed-array element writes; `@warning_ignore` for integer division | HIGH |
| Pixel-Perfect Render Pipeline & Camera | SubViewport wiring, stretch, D3D12, HDR, frame_pre_draw canvas writes | HIGH |
| 2D Lighting & Occlusion | Glow before tonemapping, D3D12, Light2D under `size_2d_override` | HIGH |
| Player Controller & Input | 4.7 device IDs | HIGH |
| HUD, Chart Table, Instruments | 4.6 dual focus, 4.7 line drawing, AccessKit | HIGH |
| Save/Load | `FileAccess` bool returns, rename atomicity on Windows | MEDIUM |
| Adaptive Audio & Tells | `area_mask` default | MEDIUM |
| Ship Section & Layout PROJECTOR, Ship Body projector | TileMapLayer Y-sort under reparent | MEDIUM |

Decision recorded (autonomous mode, Phase 0d): proceed, and flag the HIGH-risk domains throughout. Every ADR in those domains carries an Engine Compatibility section that cites the engine-reference file it was checked against.

---

## System Layer Map

### The layering rule

The systems index orders systems by dependency (Foundation → Core → Feature → Presentation → Polish). The architecture layers below are **responsibility** layers. One test decides the side: **a system that registers a SIM state block or a step hook is simulation; a system that registers none and only subscribes to the kernel is presentation.** Static content that is loaded once and never changes at runtime is data. The kernel and Save/Load are foundation because everything else registers with them.

The simulation side never reads the presentation side. Presentation reads committed simulation state through read views and writes only engine-side nodes. The one presentation class that may change the simulation is `INPUT_SURFACE`, and it does so only by submitting commands.

### Which way dependencies may point

The stack is a responsibility stack, not a rule that dependencies only point down. The rules are:

1. **Simulation to simulation, any direction.** Inside the simulation (Core and Feature), a module may call another module's read view or published mutator in either direction, as long as the step-hook graph the kernel topo-sorts stays acyclic. Examples that exist today: the Controller Sim (Core) reads Suit's `locomotion_cap` (Feature); Layout and the Graph call each other (`instantiate_section` one way, room identity the other). A consumer cannot finish before its provider exists or is stubbed, so these edges are sprint-ordering facts.
2. **No compile-time cycles.** Cross-system references go through `class_name` and typed interface base classes at the seams. `preload` of a sibling system's script is forbidden (a `preload` cycle fails to parse) and lint-checked (ADR-0010).
3. **Foundation host services read Feature state only through the kernel.** Save/Load polls owner read views (Assembly `last_shape_change_T`, Interaction `last_salvage_T`, the Attention view, Suit vitals) obtained through the kernel registry, never by importing those modules. The Pipeline root reads `body.state` the same way.
4. **Simulation never reads presentation.** One existing case breaks this: the Emitter reads the Failure & Trail glow table (`GLOW_BASE_PM`, `SEV_GLOW_PM`), which Failure & Trail owns. The table is static data, so it moves to Content data under ADR-0007; Failure & Trail still authors the values (IC-12). Presentation reads simulation only through read views and notifications.

### Layer diagram

```
┌──────────────────────────────────────────────────────────────────────────────┐
│ PRESENTATION (no block, no hook; kernel subscribers)                          │
│  Perception Filter · 2D Lighting & Occlusion · Failure & Trail · HUD & Suit   │
│  Layer · In-World Instruments · Chart Table & Menus · Adaptive Audio & Tells  │
│  · Avatar Presenter (PC) · Layout PROJECTOR · body.projector · Debug Overlay  │
│  INPUT_SURFACE: Input Adapter (PC), Chart Table menus — submit commands only  │
├──────────────────────────────────────────────────────────────────────────────┤
│ FEATURE (SIM rules: blocks + hooks + commands)                                │
│  Interaction · Suit & Equipment · Modular Ship Assembly · Web Map & Transit   │
│  · Attention · Horror Entities · Sanity & Hallucination Rules                 │
│  (Boarding & Salvage: design rules only, realised by the systems above)       │
├──────────────────────────────────────────────────────────────────────────────┤
│ CORE (SIM world model)                                                        │
│  Life-Support Graph · Ship Section & Layout · Ship Body & Interior Physics ·   │
│  Emitter State & Light-Exposure Query · Player Controller (Controller Sim)    │
├──────────────────────────────────────────────────────────────────────────────┤
│ FOUNDATION                                                                    │
│  Command & Simulation kernel · Save/Load & Versioned Format · Content data    │
│  (Item & Module catalog, section templates, Derelict Authoring content +      │
│  lint, tuning tables) · Render Pipeline root & Camera                         │
├──────────────────────────────────────────────────────────────────────────────┤
│ PLATFORM                                                                      │
│  Godot 4.7.2 (SceneTree, RenderingServer Forward+ provisional and Vulkan on   │
│  Windows, ADR-0008; AudioServer, Input,                                       │
│  FileAccess/DirAccess, Thread, HashingContext) · Windows 10/11 · SteamOS      │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Layer assignments

| # | System | Layer | Side | Owns (exclusively) | Why this layer |
|---|---|---|---|---|---|
| 1 | Command & Simulation Layer | Foundation | Kernel | `kernel.clock`, command queue, registries, binding tables, command log, kernel state, diagnostics | Every simulation system registers with it; it owns no domain state (H1). |
| 25 | Save/Load & Versioned Format | Foundation | Host service | Files under `user://saves/`, `campaign.json`, `slot_epoch_a.bin` / `slot_epoch_b.bin` (the pair), `profile.sav` (deferred to the first PROFILE block, not before Alpha); **zero blocks, zero hooks** | It serialises what the kernel snapshots; it cannot roll back with the world. |
| 3 | Item & Module Database | Foundation (data) | Static data | Immutable `ItemCatalog` built at boot; `ItemInstance` value type | No block, command or hook; every system reads definitions from it. |
| 14 | Derelict Authoring (Handmade Kits) | Foundation (data) | Content + lint | Section/world JSON, content lint table, placement records | Owns no runtime API (DA-1); produces data the Layout, Graph and Emitter load. |
| 4 | Pixel-Perfect Render Pipeline & Camera | Foundation (presentation) | Presentation root | Viewport sizes, scales k and H, layer stack, `CameraSolver`, `present_time`, `world_to_hud` | Every visual system draws inside the frame it owns. |
| 2 | Life-Support Graph | Core | SIM | `graph.state`; hooks `graph.propagate.power/.atmo/.failure` | The truth of O₂, heat, power, coolant, fuel and integrity. |
| 11 | Ship Section & Layout Data Model | Core | SIM | `layout.state`; hook `layout.doors`; immutable templates | Single owner of geometry, sockets, rooms and ids. |
| 12 | Ship Body & Interior Physics | Core | SIM | `body.state`; hook `body.integrate`; `LocomotionStep` | Ship-local integer kinematics for every actor. |
| 5 | Emitter State & Light-Exposure Query | Core | SIM | `emitter.state`; hook `emitter.place` | Light as a simulation input that no hallucination can touch. |
| 7 | Player Controller & Input | Core (Controller Sim) + Presentation (Adapter, Presenter) | Split | `controller.state` (PER_PLAYER); hooks `controller.step`, `controller.noise` | The Controller Sim is world model; the Adapter and Presenter are presentation. |
| 8 | Interaction System | Feature | SIM | `interaction.state`; hook `interaction.step`; most verb commands | The verb layer and act-only-on-lit rule. |
| 10 | Suit & Equipment | Feature | SIM | `suit.state` (PER_PLAYER); hook `suit.vitals`; `equip_tool` | Personal survival state. |
| 13 | Modular Ship Assembly | Feature | SIM | `assembly.state`; hook `assembly.step`; five commands | The atomic weld/cut/install transaction. |
| 16 | Web Map & Transit | Feature | SIM | `web.state`; hook `transit.step`; pod commands; `init_world` | World model and the abstracted pod hop. |
| 17 | Attention System | Feature | SIM | `attention.state`; hook `attention.step`; `push_stimulus` | The web's attention and the breach state machine. |
| 18 | Horror Entities | Feature | SIM | `horror.state`; hook `horror.step` | One MVP behaviour. |
| 19 | Sanity & Hallucination Rules | Feature | SIM (+PERCEPTION_OUT) | `sanity.state`, `sanity.ledger`, `sanity.perception`; hook `sanity.drift` | Authoritative sanity; writes the only perception block. |
| 15 | Boarding & Salvage | Feature (design rules) | none | Design-time rules only (BS-1) | Realised by Interaction, Suit, Assembly and Web Map; no code module. |
| 20 | Perception Filter | Presentation | Query service | `perceived()`, decoy lists, truth-channel tags; no block (PF-1) | The only path from a true value to a corruptible display. |
| 6 | 2D Lighting & Occlusion | Presentation | PRESENTATION + PROJECTOR duty | Light2D/occluder node pools; modifier chain | Draws emitters; owns no simulation fact. |
| 9 | Failure & Trail System | Presentation | PRESENTATION, pure reader | `TrailSpec` resolver, trail nodes | Truth channel; immune to the Filter. |
| 21 | HUD & Suit Layer | Presentation | PRESENTATION | HUD node surfaces, `HudLayout`, `HudModel` | Reads Suit through the Filter. |
| 22 | In-World Instruments & Inspect View | Presentation | PRESENTATION | `InstrumentPresenter` per instrument | Where class (b) lies are drawn. |
| 23 | Chart Table (Web Map & Menus) | Presentation | PRESENTATION + INPUT_SURFACE | `ChartPick`, `PauseGate`, `SettingsStore` | Never lies; submits commands through the Adapter path. |
| 24 | Adaptive Audio & Tells | Presentation | PRESENTATION | Bus tree, voice pool, `cue_requests()` | Reads Graph, Attention, Horror state directly so audio can lead visuals. |
| 26 | Debug Overlay (True Values) | Presentation (tooling) | DIAGNOSTIC | Overlay model/view, observer log | Dev and playtest builds only. |

### Engine awareness: Foundation and Core

- **Kernel**: relies on `Node._process` ordering by `process_priority` and on `PROCESS_MODE_ALWAYS` (LOW, but unverified for cross-node order, spike KRN-LAT-02); typed callables returning a `CompletionToken` (HIGH, 4.7 typed-return override rule); packed arrays with explicit dirty marking (HIGH, 4.7). *Reference: `breaking-changes.md` § 4.6 → 4.7, GDScript rows.*
- **Save/Load**: `FileAccess.store_*` returns `bool` since 4.4 (MEDIUM; every return is checked); `DirAccess.rename` atomicity on Windows is unverified (LOW, flagged [UNV]). *Reference: `breaking-changes.md` § 4.3 → 4.4.*
- **Render Pipeline**: SubViewport stretch wiring, the 4.7 stretch defaults for new projects, D3D12 default on Windows, HDR output (HIGH). *Reference: `modules/rendering.md`, `modules/ui.md`, `current-best-practices.md` § UI and Input (4.7).*
- **Graph, Layout, Body, Emitter**: scene-tree free and integer-only, so no rendering or physics API risk. The shared HIGH risk is the 4.7 packed-array element-write change; all four mark dirty keys inside mutators. *Reference: `current-best-practices.md` § GDScript (4.7).*
- **Player Controller**: 4.7 device IDs (HIGH). *Reference: `modules/input.md` § 4.7 Changes.*

---

## Module Ownership

Ownership tables give, per module: what it **owns** (sole writer), what it **exposes** (others may read or call), what it **consumes**, and the **engine APIs** it calls directly. Names are exactly as in the GDDs; the GDD is the authority on signatures.

### Foundation

| Module | Owns | Exposes | Consumes | Engine APIs |
|---|---|---|---|---|
| Kernel (`src/kernel/`) | `kernel.clock` (T, world_seed; restore_epoch held read-only), queue, registries, binding table (peer_id → PlayerId; PlayerId → owned entities), command log, kernel state | `submit`, `submit_system`, `has_pending` (on the named `SurfaceHandle`, ADR-0002); registration (blocks, commands, hooks, binding sources, diagnostics channels, key spaces) on the module's `BlockRegistrar`; `subscribe(block_id, handler, name)` on a class-bound `NotifyFacet` (class fixed at issue, ADR-0004); read views; `block_hash`, `state_hash`, `canonical_encode`; `rng_derive`, `rng_derive_epoch`; control plane (`request_pause/resume/snapshot/restore/new_game/shutdown`); `tick_alpha`, `T`, `state`, `load_seq` through the read-only `PresentationClock` (no `Kernel` global, ADR-0001) | `delta` from the one TickDriver node; `restore_epoch` from Save/Load; owner callables from every block owner | `Node._process`, `process_priority`, `PROCESS_MODE_ALWAYS` (LOW); `RefCounted`; `Callable` (HIGH: 4.7 typed returns); `HashingContext` (LOW); `var_to_bytes(…)` for payload size checks (LOW) |
| Save/Load (`src/save/`) | Save files, `campaign.json`, `slot_epoch_a/b.bin`, `profile.sav`, worker job | Host-local reads `save_status`, `fail_escalated`, `SaveInfo`, `restore_reason`, `quit_held`, `quit_write_active`; calls `new_game`, Save & Quit, Continue, `cancel_held_close` | Kernel control plane, `state_hash`, snapshot payloads; owner reads for gates (`controller.state` loco_state, `attention.view`, Suit vitals, Assembly `last_shape_change_T`/`ragged_breach_open`, Interaction `last_salvage_T`, Web events) | `FileAccess` (MEDIUM: bool returns), `DirAccess.rename` (LOW [UNV]), `Thread` (LOW), `HashingContext`, `JSON` with depth bound, `Marshalls` |
| Content data (`src/data/`, `assets/data/`) | `ItemCatalog`, section templates, world records, verb tables, tuning tables, `ids.lock.json` | `def()`, `compatible_defs()`, `repair_part_for()`, `salvage_yield()`, template reads, boot-lint results | JSON files listed in committed manifests | `FileAccess`, `JSON` (numbers arrive as floats: int-cast), export `include_filter` (LOW) |
| Render Pipeline root (`src/presentation/pipeline/`) | `PipelineRoot`, the single `PresentationHost` (one `_process` at `ProcessPriority.PRESENTATION`, ADR-0008 § 6, ADR-0009 § 3); world SubViewport size, k, H, letterbox, layer stack, full-screen material slot, `CameraSolver` state, `present_time`, zoom rung; the `PresentationDriver` and its stage list | `RenderLayout` (read-only), `world_to_hud`, `mouse_to_world_aim`, `mouse_to_world_view`, published follow pose `f_q`, `present_time`, pipeline state | Avatar pose (`body.state`), local raw aim (Adapter), current vehicle, `BODY_TRANSFER` events, loaded body bounds (PROJECTOR) | `SubViewport`, `SubViewportContainer` (HIGH), `Window` content scale (MEDIUM), `Camera2D` / canvas_transform, `RenderingServer.frame_pre_draw` (canvas writes only; probed 4.7.2 editor, release export pending), `Parallax2D` (MEDIUM), `Viewport.get_render_info` |

### Core

| Module | Owns | Exposes | Consumes | Engine APIs |
|---|---|---|---|---|
| Life-Support Graph (`src/sim/graph/`) | `graph.state` (SoA packed arrays), three propagate hooks, `GraphTuning` | Class A mutators M1-M13, M19 with paired `check_*`; Class B M14-M18; `can_join`, `can_join_static`; read view (`compartment`, `node`, `edge`, `failures(filter)`, `served_pm`, `opening`, `seam_view`, `alarm_sections`, `true_value`, `power_use`, …); dirty keys `C/ B/ N/ F/ L/ S/ T/ SEC/` | Kernel WriteScope, hooks, `rng_derive`; Layout room identity, `V`, `exposed_tiles`; Item DB defs | `RefCounted`, `PackedInt64Array`/`PackedInt32Array`/`PackedByteArray` (HIGH: 4.7 element writes), (`HashingContext` is allowed only in `src/save`, `src/data`, `src/shared/det/hash64.gd` and `src/kernel/state_hasher.gd`; ADR-0010 IO-01) |
| Ship Section & Layout (`src/sim/layout/`) | `layout.state` (bodies, section instances, sockets, welds, counters), templates, broadphase caches, hook `layout.doors` | LM1 `build_section`, LM2 `attach_section`, LM3 `detach_section`, LM4 `discard_section_geometry` + `can_*`; `rects_in`, `solid_in`, `OccluderSource`, `version/static_version/leaf_version`, `room_at`, `section_at`, `socket_plan`, `detach_preview`, `leaf_would_overlap` | Graph `instantiate_section`, `opening`, `seam_view`; Item DB `compatible_defs` | none in the data module; the PROJECTOR uses `TileMapLayer` (MEDIUM), `Node2D` y-sort |
| Ship Body (`src/sim/body/`) | `body.state` (body poses/velocities, actor rows), hook `body.integrate`, `LocomotionStep` | BM1-BM8 + `can_*` (BM8 `release_body`) *(Amended 2026-10-29 — ADR-0015 sync)*; `pos_curr/pos_prev`, `foot_box(es)`, `actor_section`, `actors_overlapping`, `docked`, `seat_bindings()`; P5 events `BODY_TRANSFER`, `BODY_DOCK_CONTACT`, `BODY_HULL_SEPARATION` | `controller.state` octant and `v_eff`; Layout `solid_in`, `section_at`, `section_offset` | none in the sim module (no physics server, lint-enforced); `body.projector` uses `Node2D`, `Camera2D.limit_*` |
| Emitter State (`src/sim/emitter/`) | `emitter.state`, hook `emitter.place`, pair cache, per-tick read-view columns | E1-E10 mutators; `toggle_beam`, `set_switch`, `set_beacon` commands; `lux_pm`, `act_lux_pm`, `lit(ref)`, `shapes()`, `beam_spill`, `beam_bindings()`; dirty keys `E/<id>` | Controller chest point and aim; Body pose; Layout `OccluderSource`, `anchor_geometry`; Graph `served_pm`, `run_pm`, M14, `failures()`; F&T glow table | none (scene-tree free, lint-enforced) |
| Controller Sim (`src/sim/controller/`) | `controller.state` (PER_PLAYER intent, gait, aim, facing, loco_state, carry/channel refs, chest and hand points), hooks `controller.step`, `controller.noise` | Commands `set_move_intent`, `set_aim`, `give_up`, `stagger` (SYSTEM); Class A status-returning mutators `set/clear_channel_ref`, `set/clear_carry_ref` (each with a `check_*` twin; OK, REFUSED or -fault_code; ADR-0019 section 2.5) *(Amended 2026-10-29 — ADR-0019 sync)*; read view | Suit `locomotion_cap`; Body `moved`; kernel binding table | none in the sim module |

### Feature

| Module | Owns | Exposes | Consumes | Engine APIs |
|---|---|---|---|---|
| Interaction (`src/sim/interaction/`) | `interaction.state` (channels per seat, partial-progress ledger, FULL-job parts, throws, loose items, stubs, counters), hook `interaction.step`, verb declaration table | Verb commands (`begin_patch`, `begin_cut`, `begin_cap`, `begin_crank`, `cancel_channel`, `set_breaker`, `set_valve`, `top_up`, `set_opening`, `pickup`, `drop`, `throw_glowstick`, `field_check`, `begin_harm`, …); Class B `open_channel`, `sync_section`, `place_loose`; `focus_view(seat)`, `channel_view`, `last_salvage_T()` | Emitter `lit`/`act_lux_pm`; Graph mutators M7-M13, M16; Suit mutators; Controller channel/carry refs; Sanity `actor_sanity_below_harm_threshold`, `record_falsification`, `record_harm_event`; Attention `push_stimulus` | none in the sim module |
| Suit & Equipment (`src/sim/suit/`) | `suit.state` (PER_PLAYER vitals, tools, stowed, carried, beam shape), hook `suit.vitals` | `equip_tool`; mutators (`add_stowed`, `set_carried`, `release_carried`, `consume_charge`, `apply_suit_damage`, `patch_integrity`, `revive`, …); reads `tool_in_hand`, `locomotion_cap`, `true_value(key, player_id)` | Graph `draw_air` (M15), `withdraw_store` (M16), compartment atmosphere; Item DB; Emitter E3 for glowsticks | none |
| Modular Ship Assembly (`src/sim/assembly/`) | `assembly.state` (per-seat pending, runtime last_result), hook `assembly.step` | Commands `begin_weld`, `begin_seam_cut`, `weld`, `seam_cut`, `install_module`; `weld_offer`, `seam_cut_offer`, `cut_mode`, `last_shape_change_T()`, `ragged_breach_open()`; verifier INV-1..INV-10 | Graph M2, M3, M5, `can_join`, `seam_view`; Layout LM2, LM3, `socket_plan`, `detach_preview`; Body BM3, BM4, `docked`, `actors_overlapping`; Interaction `open_channel`, `sync_section`; Suit `release_carried`; Controller `clear_carry_ref` | none |
| Web Map & Transit (`src/sim/web/`) | `web.state` (derelict records, transit machine), hook `transit.step`, `init_world` | `pod_hop`, `berth`, `unberth`; SYSTEM `pod_arrive`, `pod_settle`, `berth_step`, `web_reconcile`; `web.view`, `console_ready`, `locus_kind`, `ring_of`; events `POD_DEPARTED`, `POD_ARRIVED`, `DERELICT_LOADED`, `BERTHED`, … | Layout LM1/LM4; Graph M4, M16, M19; Body BM1, BM5-BM7; Emitter E1 | none |
| Attention (`src/sim/attention/`) | `attention.state`, hook `attention.step`, RNG stream `attention.tell` | Class B `push_stimulus(kind, source_ref, class)`; `attention.view` (phase, breach_phase, tell_active, tell_remaining, tier, level_pm, …); `exogenous_permitted` | Emitter exposure, Graph `power_use`, Controller `loco_state`, Suit, kernel `rng_derive_epoch` | none |
| Horror Entities (`src/sim/horror/`) | `horror.state`, hook `horror.step` | `horror.view` (entities, disturbances, marks); dormant `repel()` | Attention view, Emitter exposure, Body `LocomotionStep`, Graph M17 `deposit_trace`, Layout geometry | none |
| Sanity (`src/sim/sanity/`) | `sanity.state`, `sanity.ledger` (HOST_ONLY), `sanity.perception` (PERCEPTION_OUT), hook `sanity.drift` | `actor_sanity_below_harm_threshold(player_id)`, `sanity_view(player_id)`; Class B `record_falsification`, `record_harm_event` | Attention view (phase, tier, tell_outcome, tell_epoch only), Web Map ring, Horror distance, Emitter lit-entry, `assets/data/sanity/lie_targets.json` | none |

### Presentation

| Module | Owns | Exposes | Consumes | Engine APIs |
|---|---|---|---|---|
| Perception Filter (`src/presentation/perception/`) | Filter state per viewer, decoy lists, `PerceptionTag` contract | `perceived(ref, viewer) -> Perceived`; decoy list (read by Lighting); audio requests; `truth_channel_union` | `sanity.perception` (the only reader besides Debug Overlay and the future replication adapter), Graph/Suit true values | `CanvasItem`/`Material` properties only; ShaderMaterial fringe (counts toward Lighting's five-shader cap) |
| 2D Lighting (`src/presentation/lighting/`) | Light2D, LightOccluder2D, stand-in and decoy node pools; modifier chain; flash governor | nothing to the simulation | `emitter.state` view, `shapes()`, Layout `OccluderSource`, Attention/Horror modifier state, Filter decoys | `PointLight2D`, `LightOccluder2D`, `CanvasModulate`, ShaderMaterial (LOW API, cost [UNV]); glow Environment, `use_hdr_2d` (HIGH) |
| Failure & Trail (`src/presentation/trail/`) | Trail nodes, `TrailSpec` resolver, glow table data | glow table rows (read by Emitter as data) | Graph `failures(filter)`, Layout anchors | `Sprite2D`, `GPUParticles2D`/`CPUParticles2D`, `Material` |
| HUD (`src/presentation/hud/`) | HUD nodes on CanvasLayer 2, `HudLayout`, `HudModel` | `hud_debug_snapshot()` | Suit read view via `perceived()`, Interaction `focus_view`, pipeline `world_to_hud`, H | `CanvasLayer`, `Control`, `Theme` (HIGH: 4.6 dual focus) |
| Instruments (`src/presentation/instruments/`) | `InstrumentPresenter` per instrument, inspect card | `card_opened`/`card_closed` presentation signals | `perceived()` (sole caller for its ref), Graph descriptors through the Filter | `Sprite2D` pre-rendered frames, instance uniforms (HIGH [UNV]) |
| Chart Table & Menus (`src/presentation/chart/`) | `ChartPick`, `PauseGate`, `SettingsStore`, menu state machine | commands through the INPUT_SURFACE handle; settings reads | `web.view`, Save/Load host reads, Adapter MenuInput | `Control`, `CanvasLayer` 3 |
| Adaptive Audio (`src/presentation/audio/`) | Bus tree, voice pool, presentation memory | nothing to the simulation | Graph `failures()`, Attention view, Horror view, Filter audio requests | `AudioServer`, `AudioStreamPlayer(2D)` (MEDIUM: 4.7 `area_mask`) |
| Avatar Presenter + Input Adapter (`src/presentation/controller/`) | Adapter local state (last_acked, latches, context), presenter nodes | MenuInput stream; INPUT_SURFACE submissions | `controller.state`, `body.state`, Emitter beam pose | `Input`, `InputMap` (HIGH: 4.7 device IDs), `AnimatedSprite2D`, `Light2D.position/rotation` |
| Layout PROJECTOR + `body.projector` | Per-body root `Node2D`, `TileMapLayer`s, sprites, camera-limits rectangle (published to the Pipeline solver, never written to `Camera2D.limit_*`) | `attach_presenter(actor_id, node)` | `layout.state`, `body.state` | `TileMapLayer` (MEDIUM), `Node2D.y_sort_enabled` |
| Debug Overlay (`src/debug/`, `src/observer/`) | Overlay model/view (CanvasLayer 4), observer log | `register_channel(declaration)` | Every DIAGNOSTIC channel, true values, `sanity.ledger` | `Control`, `FileAccess` for the observer log |

### Dependency diagram

Arrows read "depends on (reads or calls)". Presentation arrows are reads only.

```
                         ┌──────────────── Kernel ───────────────┐
                         │ tick · commands · blocks · notify ·    │
                         │ rng · hash · control plane             │
                         └──▲───────────▲───────────────▲─────────┘
                            │ registers │               │ control plane
     ┌──────────────────────┼───────────┼───────┐   Save/Load ──► files
     │ CORE                 │           │       │       ▲ polls owner reads
     │  Layout ◄──── Graph ◄┘   Emitter ┘       │       │
     │    ▲  ▲         ▲  ▲       ▲  ▲          │
     │    │  └── Body ─┘  │       │  │          │
     │    │       ▲       │       │  │          │
     │    │  Controller ──┴───────┘  │          │
     └────┼───────▲──────────────────┼──────────┘
          │       │                  │
     ┌────┴───────┴──────────────────┴──────────────────────────────┐
     │ FEATURE  Interaction ──► Graph, Suit, Controller, Emitter,    │
     │                          Sanity (gate), Attention (push)      │
     │          Assembly ──► Graph, Layout, Body, Interaction, Suit  │
     │          Web/Transit ──► Layout, Graph, Body, Emitter         │
     │          Suit ──► Graph, Item DB, Emitter                     │
     │          Attention ──► Emitter, Graph, Controller             │
     │          Horror ──► Attention, Emitter, Body, Graph, Layout   │
     │          Sanity ──► Attention, Web, Horror, Emitter           │
     └───────────────────────────▲──────────────────────────────────┘
                                 │ read views + P5 notifications only
     ┌───────────────────────────┴──────────────────────────────────┐
     │ PRESENTATION  Filter ◄─ sanity.perception                     │
     │   HUD, Instruments ──► Filter ──► true values                 │
     │   Lighting ──► Emitter, Layout, Attention, Horror, Filter     │
     │   Trail, Audio ──► Graph, Attention, Horror (truth channels)  │
     │   Chart Table ──► Web view, Save/Load reads (INPUT_SURFACE)   │
     └──────────────────────────────────────────────────────────────┘
     All Core and Feature modules read Content data (Item DB, templates).
```

---

## Data Flow

### 1. Frame update path

One rendered frame. The tick is driven from `_process`, never from the physics callback (kernel CR-1a; see Open Question OQ-1 for three GDDs that still cite the physics callback).

```
Engine frame (_process order by process_priority)
│
├─ INPUT_ADAPTER (-100)  Input Adapter (INPUT_SURFACE; the only presentation-side node below the driver's priority, so the KRN-LAT-02 priority lint exempts INPUT_SURFACE nodes)
│     read InputMap actions → change-only intents (set_move_intent, set_aim ≤ every 3 ticks)
│     → surface.submit(type, payload, actor, correlation)           [sync call on the Adapter's SurfaceHandle (ADR-0002), queued]
│
├─ TICK_DRIVER (-50)  KernelHost → TickDriver (the only wall-clock read that reaches the sim)
│     boundary service first (SAVING / LOADING / pause / resume; a frame that ends in a transition into RUNNING clears the accumulator and discards its own delta)
│     then, while RUNNING: acc += delta_us × TICK_RATE ; ticks = min(floor(acc / 1e6), MAX_TICKS_PER_FRAME=4)
│     repeat ticks times: Kernel.step()  ──► P0 Begin
│                                           P1 Intake   (queue → this tick's batch, coalesce intents)
│                                           P2 Dispatch (authority check → validate → apply in WriteScope)
│                                           P3 Step     (hooks whose T mod divisor == offset, topo order)
│                                           P4 Commit
│                                           P5 Notify   (batches queued; PRESENTATION merged per frame)
│                                           P6 Boundary (snapshot/restore/pause requests)
│     drop lag (LAG_FORGIVE_TICKS 0) ; flush results ; compute tick_alpha ; Kernel.deliver_frame_end() (unconditional last step of the driver frame)
│
├─ HOST_SERVICE (-40)  SaveHost (src/game/save_host.gd; the only user of this priority, ADR-0006)
│     SaveService.poll_frame(): observes the last committed tick boundary; polls the worker; no presentation work
│
└─ PRESENTATION (0)  PipelineRoot._process = the single PresentationHost (ADR-0008 s6.1, ADR-0009 s3); presenters define no _process
      subscriber handlers (run in P5) only mark dirty: _dirty.absorb(batch); return ctx.complete()
      advance present_time → zoom cooldown → RESCALING re-derive if the layout is dirty → driver.frame(delta, present_time)
      driver.frame fills one PresentFrame (alpha read once, pos = floor(prev + (curr - prev) × frame.alpha)), then runs the stages in order:
        0 PROJECT (layout projector, body.projector, Lighting's occluder builder) → 1 CAMERA_SOLVE (CameraSolver: pose_q, f_q, jolt, parallax offsets, world_to_hud inputs; solved, not yet written)
        → 2 AVATAR (Avatar Presenter) → 3 FILTER (Perception Filter step, decoy list) → 4 INSTRUMENTS (perceived()) → 5 HUD (perceived(); world_to_hud with this frame's pose)
        → 6 LIGHTING → 7 TRAIL → 8 AUDIO → 9 DEBUG
      RenderingServer.frame_pre_draw (probed after every node _process, windowed 4.7.2 editor; release export unverified): writes canvas_transform and the parallax offsets only, from the already-solved pose
```

*(Amended 2026-10-29 — ADR-0001 / ADR-0008 / ADR-0009 sync)* Frame order is the named constants of ADR-0001 § 2: `INPUT_ADAPTER` -100, `TICK_DRIVER` -50, `HOST_SERVICE` -40, `PRESENTATION` 0 (no `PRESENTATION_LATE`); there is no "priority 2". The stage list is ADR-0009 § 3 (a convention beyond the three GDD-stated edges: camera solve before HUD, Filter after Avatar and before Lighting; TD-22); the single-host shape and the camera solve in `CAMERA_SOLVE` are ADR-0008 § 6 (TD-03). If `frame_pre_draw` fails the release-export spike (ADR-0008 pipeline spike step 12), the canvas write moves to the end of `PipelineRoot._process`.

**P3 hook schedule (MVP).** Divisor and offset decide which ticks a hook runs on; `after:` orders hooks that run on the same tick. An `after:` edge between hooks that do not always run on the same tick should declare `tolerates_stale` (CR-3); an undeclared one boots with a warning (ruling R-8), so the warning list must be empty at the end of every CI run.

| Hook | Owner | Divisor | Offset | After |
|---|---|---|---|---|
| `controller.step` | Controller | 1 | 0 | `suit.vitals` (stale-tolerant) |
| `layout.doors` | Layout | 1 | 0 | — |
| `body.integrate` | Body | 1 | 0 | `controller.step`, `layout.doors` |
| `controller.noise` | Controller | 1 | 0 | `body.integrate` |
| `emitter.place` | Emitter | 1 | 0 | `body.integrate`, `graph.propagate.failure` (stale-tolerant) |
| `interaction.step` | Interaction | 1 | 0 | `emitter.place` |
| `assembly.step` | Assembly | 1 | 0 | `interaction.step` |
| `transit.step` | Web Map | 1 | 0 | `body.integrate`, `graph.propagate.failure` (**undeclared** stale edge, IC-11) |
| `graph.propagate.power` | Graph | 6 | 0 | — |
| `graph.propagate.atmo` | Graph | 6 | 2 | `graph.propagate.power` (stale-tolerant) |
| `graph.propagate.failure` | Graph | 6 | 4 | `graph.propagate.atmo` (stale-tolerant) |
| `suit.vitals` | Suit | 6 | 2 | `graph.propagate.atmo` |
| `attention.step` | Attention | 6 | 0 | `graph.propagate.power`, `emitter.place` |
| `horror.step` | Horror | 5 | 0 | `attention.step` (stale-tolerant), `emitter.place` |
| `sanity.drift` | Sanity | 12 | 0 | `graph.propagate.failure` (stale-tolerant), `suit.vitals` (stale-tolerant) |

The kernel builds this order once in BOOTING; an unknown `after:` id or a cycle fails boot. ADR-0003 records the sort; each owning GDD remains the authority on its own row.

**Budgets (P99, release export, RM-DESKTOP / RM-DECK):** P3 ≤ 2.0 ms (alarm at `STEP_BUDGET_MS` 4 ms); frame simulation ≤ 4.0 ms for one tick and ≤ 6.0 ms for four; Graph hooks ≤ 0.5 ms on a hook tick; `emitter.place` + exposure ≤ 0.4 ms; Sanity median ≤ 0.5 ms; pipeline `_process` ≤ 0.5 ms; light CPU ≤ 1.0 ms steady; total frame ≤ 14.9 ms desktop and ≤ 30.0 ms Deck (90 % of the 16.6 / 33.3 ms ceilings, ruling R12).

### 2. Command path (a player verb)

```
Adapter (frame N)          Kernel tick T                         Owner
  surface.submit(         _admit (at submit): stamps issuer_peer, origin,
   "begin_patch",            seq, submission_id (ADR-0002)
   {target}, actor,       P1 intake: queued_tick = T-1
   correlation)  ───────► P2: origin ∈ allowed_origins?
                              actor owned by issuer's PlayerId? ──► Interaction.validate(cmd, views, ctx)
                              (Rejected(ACTOR_NOT_OWNED) else)       ctx.ok() / ctx.reject(reason)
                                                                 ──► Interaction.apply(cmd, scope, ctx)
                                                                       Controller.set_channel_ref(...)
                                                                       return ctx.complete()
                          P3: interaction.step advances the channel each tick
                          P4 commit ; P5 notify (interaction.state keys)
  result record ◄──────── one terminal result per command (Applied / Rejected(reason) /
   (reason code,            Dropped / Faulted / Superseded), flushed at P5 and once per frame
    correlation)
```

The Adapter never predicts an outcome except body motion and sound (pre-play keyed by `correlation`); light and effects change only on committed state.

### 3. Weld transaction (cross-system atomic write)

```
begin_weld (PLAYER) ── validate (static predicates only: can_join_static, socket_plan, docked)
   └─ apply: Interaction.open_channel(seat, ASSEMBLY, verb, target, WELD_CHANNEL_TICKS=360)
interaction.step × 360 ticks (re-checks lit and reach every LIT_RECHECK_TICKS)
assembly.step sees channel DONE ── submit_system("weld", {actor, target, cause})
next tick P2: weld.validate (authoritative site; full can_join) ── weld.apply, one WriteScope:
   Graph.weld_join (M2) → Layout.attach_section (LM2) → Body.rebase_on_attach (BM3)
   → Interaction.sync_section × 2 → assembly.state
   each call returns OK or FAULT_CODE; first non-OK → ctx.fault() → kernel FAULTED (no rollback)
P5: one notification batch (graph, layout, body, interaction, assembly keys) + BODY_TRANSFER
   → PROJECTOR, Lighting and Trail are dirty-marked and draw in the driver's stages; camera rebase
   (the Emitter is not a P5 subscriber: it is the simulation hook emitter.place, which re-derives body_id in W0 on the next tick or in the same tick's P3)
Save/Load polls Assembly.last_shape_change_T() → ASSEMBLY_WELD autosave (subject to gates)
```

Cut is the same shape with M3 → LM3 → BM4; install is a TAP command (Graph `check_install_module` → M5 → Suit `release_carried` → Controller `clear_carry_ref` → `sync_section`).

### 4. Event and notification path

- **Simulation → presentation:** P5 notification `(block_id, tick, change_keys)` with no state payload, delivered to a method Callable `(batch: ChangeBatch, ctx: NotifyContext) -> CompletionToken` that only accumulates (`_dirty.absorb(batch)`, then `return ctx.complete()`). Dirty keys are ints `(KIND << 32) | id`, namespaced per block. Past `MAX_DIRTY_KEYS` (256) a batch collapses to `ALL`. PRESENTATION, PROJECTOR and DIAGNOSTIC batches are merged across a multi-tick frame and delivered once; INPUT_SURFACE and REPLICATION get every tick.
- **Presentation → simulation:** only commands through an INPUT_SURFACE handle. There is no other path.
- **Simulation → simulation:** synchronous calls inside the tick: pure reads of another block's read view, and published mutators called inside the caller's WriteScope. A cross-block write is always a published mutator of the owning system (Class A checked or Class B clamped). Simulation code never subscribes to notifications (an object that owns a block or hook never subscribes, CR-10).
- **P5 events** (`BODY_TRANSFER`, `POD_ARRIVED`, `DERELICT_LOADED`, …) are presentation-subscriber notifications that carry no simulation state; Save/Load does not read them, it polls monotonic counters in owner read views (e.g. `web.state` `pod_arrived_seq` / `berthed_seq`) once per frame (ADR-0006 § 1.12). *(Amended 2026-10-29 — ADR-0006 sync)*
- **Engine signals** are used only inside presentation (e.g. `card_opened`, settings change). The kernel's observer registry is not built on engine signals (ADR-0004, Proposed; acceptance gated on KRN-LAT-02 notify cost). *(Amended 2026-10-29 — ADR-0004 sync)*

### 5. Perception path (the one place a lie can enter)

```
Sanity (SIM) ── sanity.drift ──► sanity.perception (PERCEPTION_OUT, per viewer, never persisted)
                                        │  (simulation reads of it return empty + violation)
Instrument / HUD presenter ── perceived(ref, viewer) ──► Perception Filter
                                        │ reads true value (Graph / Suit read view)
                                        │ reads sanity.perception entry for (ref, viewer)
                                        ▼
                               Perceived {displayed_value, motion_mode, leak_amplitude_pm, …}
Truth channels never pass through the Filter: Failure & Trail, true reveal radius (Lighting L0),
character sprites, crewmate beams, HUD frame, physical consequences in the Graph.
```

The Graph's `state_hash` is identical with and without perception content; the simulation never computes a perceived value.

### 6. Save path

```
Trigger (interval 10 800 ticks, ASSEMBLY_WELD/CUT, SALVAGE_CUT, QUIT, NEW_GAME)
 └─ gates (DOWNED, refractory, suit CRITICAL, owed warning, RAGGED breach) hold or defer ≤ 7 200 ticks
Save/Load ── request_snapshot ──► Kernel at P6: SAVING (one boundary service, synchronous, T fixed)
   every persist block: snapshot(ctx) → ctx.put(plain data, deep copy, sorted ids)
   ◄── payloads split by persist_scope: WORLD → generation, PROFILE → profile.sav
Main thread: encode + hand-off ≤ SAVE_ENCODE_BUDGET_MS 4 ms
Worker thread (immutable PackedByteArrays + paths only):
   write .tmp → flush → re-read → re-hash → rotate oldest → rename → verify → campaign.json
   any failure: previous generations untouched, save_status FAILED, bounded backoff
SaveHost (HOST_SERVICE -40, after the driver) polls once per frame: the job state, never a join (a live thread is never joined)
Save & Quit and the window-close request (NOTIFICATION_WM_CLOSE_REQUEST, auto_accept_quit false) take the same polled path:
   poll the QUIT job under SAVE_QUIT_TIMEOUT_MS 3 000, then request_shutdown on WRITTEN; on timeout report FAILED and offer "quit without saving"
```

*(Amended 2026-10-29 — ADR-0006 sync)* Save & Quit polls rather than joins, and a window close, Alt+F4 or OS shutdown is routed to the same polled quit flow instead of cutting a write in flight (ADR-0006 § 1.6).

### 7. Load / restore path

```
Continue / death stub ── pre-check (pure: bytes, registry view, build constants; no kernel)
   └─ pass: epoch staging (write of the older slot_epoch_a/b.bin file, polled per frame) → EPOCH_DURABLE
      (or SL_EPOCH_NOT_DURABLE, save_status FAILED, restore proceeds — listed exposure, R14)
Kernel request_restore(payload, restore_epoch, expected_state_hash: int = 0, has_expected_hash: bool = false) at P6: LOADING (one boundary service; ADR-0002 § 6)
   (has_expected_hash false: the K-4 compare is skipped and the outcome verdict is NONE; true: MATCH or MISMATCH, a diagnostic only)
   restore each WORLD block in restore_after order (two-phase: decode+validate into scratch, then copy the contents into the existing block object, never replacing it, ADR-0003 § 2; failures via ctx.fault)
   post_restore each block in the same order (derived caches, cross-block verification)
   rebuild binding table half (b) from binding sources (seat_bindings, beam_bindings)
   reproduction check: state_hash vs header (reported, never a fallback reason)
   → RUNNING; accumulator cleared; camera CUT; Adapter resends intents; first frame passes through the Filter
```

**restore_after order (MVP, ADR-0016 section 2.4):** `layout.state`, `graph.state`, `body.state`, `suit.state`, `controller.state`, `emitter.state`, `interaction.state`, `assembly.state`, `web.state`, `attention.state`, `horror.state`, `sanity.state`, `sanity.ledger`, then the non-persisted `sanity.perception`; the kernel topo-sorts the declared DAG with smallest registration index first among ready blocks, and `restore`, `post_restore`, `default_state` and `init_world` all follow it. *(Amended 2026-10-29 — ADR-0016 sync)*

**Threading.** Exactly one thread boundary exists in the MVP: the save writer (and its read-back). Everything else, kernel, owner callables and presentation, runs on the main thread; `Thread`, `WorkerThreadPool` and `call_deferred` into kernel code are lint-forbidden (CR-1a).

### 8. Initialisation order

```
1. Engine boot; project.godot guard rails asserted by effective value (stretch disabled, filters, snap,
   hdr_2d off on the root, physics_interpolation off, renderer and driver), ADR-0008 s5
2. Pipeline root builds the viewport stack (the Pipeline's own BOOTING → PRESENTING states, not the kernel's)
3. Content data loads before the kernel exists: the composition root calls ContentLoader.load_all() →
   ContentLoadResult (ADR-0007 § 1.7), in a fixed order: Graph type tables + GraphTuning → Item DB →
   section templates (manifest order) → world records, verb tables, tuning tables; boot lints and
   invariant assertions. On any error the composition root never starts the kernel (ADR-0007 § 2.3).
   ("Loaded in BOOTING" in a GDD means this start-up phase, not the kernel's BOOTING state.)
4. Kernel BOOTING: owners register blocks, command types, hooks, binding sources, diagnostics channels
   through their BlockRegistrar (Layout, Graph last of the Item DB → Layout → Graph chain; then Body,
   Controller, Emitter, Suit, Interaction, Assembly, Web, Attention, Horror, Sanity). Registration then
   freezes; hook topo-sort.
5. Presentation subscribers subscribe through their class-bound PresentationFacet (Lighting, Trail, HUD,
   Instruments, Audio, PROJECTORs, Debug) and register PresenterDescriptors with the PresentationDriver
6. Main menu (Chart Table) → New Game: Save/Load new_game(seed) → kernel request_new_game(seed, 0)
   → Web Map init_world inside one WriteScope: Layout build_section → Emitter instantiate_section_emitters
   → Interaction sync_section → Interaction place_loose (seeded loose ledger, Derelict DA-26)
   → Suit seat defaults: create_seat_beam (E9) then bind_beam (E4) (Suit SE-9)
   → PROJECTOR notification → initial NEW_GAME generation written
   or Continue: load path (§ 7)
```

### 9. Cross-system edges not drawn above

*(Amended 2026-10-29 — architecture-review document-coverage pass, `docs/architecture/review-data/res-archdoc.md` § 3)* Sections 1 to 8 show the frame, the command, the weld, notification, perception, save, load and boot paths. The edges below exist in the approved GDDs and are not on those paths; each names its source rule. None adds a dependency direction forbidden by the layering rule, and presentation edges are reads only. (Review items 15 and 21 were already covered and are not repeated.)

| # | Edge | How it travels | Source |
|---|---|---|---|
| 1 | Interaction / Controller salvage carry → Attention (`U_BURDEN`) | `controller.noise` pushes class `BURDEN` every tick a salvage carry is held aboard a non-home locus body, moving or standing, until dropped, installed or thrown | Player Controller PC-10, D.13; Boarding BS rows 6 and 8; Attention F-26 |
| 2 | Chart Table menu → Pipeline states `PAUSED_FROZEN` and `TRANSIT_HOLD` | `PauseGate` calls `request_pause` / `request_resume` on a control handle; the Pipeline reads kernel state through `PresentationClock.state`; the chart stays open during a leg | Chart Table dependency row 2, CT-15; Pipeline state table |
| 3 | Chart Table / Adapter console seam | host-local `ChartPick` injected into `pod_hop` / `berth` payloads; a press with no pick routes to "open chart" | Chart Table CT-10 |
| 4 | Suit seat defaults at `new_game` | `create_seat_beam` (Emitter E9) then `bind_beam` (E4) after the start kit; step 6 of § 8 | Suit SE-9; Save/Load SL-18 |
| 5 | Audio reads beyond Graph, Attention, Horror and the Filter | pull reads of Suit cues, Controller gait and `channel_view`, Interaction `ia_*` and `result_seq`, Assembly `last_result` and the seam-cut loop, Body `BODY_*` events, Web `POD_*` and `BERTHED`, Layout `anchor_geometry` / `hull_anchor`, Graph `opening()` transmission, Emitter `E/<id>`; event names verbatim | Audio AA-3, AA-8 and its dependency table |
| 6 | Restore → presentation (post-restore silence) | `PresentationClock.load_seq` changes on `LOADING → RUNNING`; the driver calls every presenter's `on_discontinuity(RESTORE)` before any `present`; sustained cues are cut, one-shots dropped | Audio AA-5 (notification row, CR-8); Save/Load SL-7; ADR-0009 § 6 |
| 7 | Instruments inputs | Layout / Derelict `subject_anchors(section_id)` registered through Interaction `sync_section`; authored instrument placement records; Item DB `sprite_family` and state tokens; Suit O2 `stock`; Web home fuel level against `HOP_FUEL_ML` | Instruments IW-21, IW-22, IW-31; Interaction IA-37; Web Map WM-20 |
| 8 | Controller Adapter → Instruments, Pipeline → Instruments | the Adapter's presentation-local `{context, inspect_ref}` signal and `inspect` / `cancel`; `present_time` from the `PresentFrame` | Instruments IW-24 and its Player Controller dependency row |
| 9 | Instruments → Interaction | an instrument is an OBJECT subject of class `INSTRUMENT` (lit, in reach); `verb_inspect` opens the card; the only upward signals are presentation `card_opened` / `card_closed` | Instruments IW-21, IW-24; Interaction IA-2 |
| 10 | Derelict content → Failure & Trail and Lighting | dried trails (`since_steps` at least `DRIED_AFTER_STEPS`), the heat-presentation step (`ASK-F&T-H1`); containment colours and shadow-slot limits for Lighting's lint | Derelict DA-14, DA-17, DA-31; Failure & Trail FT-3, FT-11, FT-18; Lighting LO-33 |
| 11 | Derelict content → Interaction | the seeded loose-item ledger placed by `place_loose` at world build (step 6 of § 8) | Derelict DA-26; Layout LY-22 |
| 12 | Derelict content → Web Map | `residual_lamp_predicate`, `pod_body`, `visible_unknown`, the world file `mvp_world.json` | Derelict DA-4, DA-27 |
| 13 | Derelict content → Ship Body | the pod stub, spawn table and `can_place`; lint `SPAWN_BLOCKED` | Derelict DA-3; Body BDY-25 |
| 14 | Derelict content → Attention, Horror | the beacon data row and the EMERGENCY cryo lamp; Layout `room_adjacency` and `route_length_px` consumed by Attention and Horror | Derelict DA-23, DA-24, DA-29 |
| 16 | Controller → Attention | the `controller.noise` hook (after `body.integrate`) calls `push_stimulus(NOISE, actor, class)` with WALK, RUN or CARRY from `loco_state` on each tick the actor moved | Player Controller dependency row 6; kernel CR-9 (a) |
| 17 | Camera framing on pause and transit hold | the camera is a window on player-owned inputs only (C1); `PAUSED_FROZEN` and `TRANSIT_HOLD` hold the last pose; a cut raises `notify_camera_cut`, and every presenter then gets `CAMERA_CUT` | Pipeline C1 and its camera states table; ADR-0009 § 3 |
| 18 | HUD ← Controller prompts and hand rail | HUD plates read Interaction `focus_view` (including `self_target`) and the Controller's prompt state through read views; no HUD write path | HUD HS-37, HS-39 and its Player Controller dependency row |
| 19 | Debug Overlay DIAGNOSTIC reads | Save/Load `save_status`, `state_hash` and last generation; Web `transit_stale_cmds`; Assembly verdict-drift counters, each as a registered diagnostics channel | Debug Overlay channel table and Save/Load row |
| 20 | Perception Filter ↔ Audio | audio requests `(viewer, ref or position, intent in {ON, LEVEL(pm), OFF}, class)`; the Filter owns the `FILTER_AUDIO_CLASSES` namespace; one corruptible family | Perception Filter PF-20; Audio AA-1, AA-2 |
| 22 | Failure & Trail ↔ Pipeline and Layout | Trail draws in the Pipeline's layer stack at scale `H`; it rebuilds anchors from Layout on `T` and `SEC/<id>` keys (mark, then poll in the stage) | Failure & Trail FT-15 and its dependency table |
| 23 | Settings and reduce-motion / reduce-flashing → flash governors | `SettingsStore` values reach Lighting's per-light governor, the HUD arbiter and the Chart Table through the Pipeline's frame-level invariants; one shared frame-mean limit (IC-7) | Pipeline PR-13, PR-14; Chart Table `SettingsStore` |
| 24 | Item DB ↔ Audio and Instruments | Audio takes no value-tiered pickup sounds (a design constraint, not a data edge); Instruments take `sprite_family` from the catalog; all modules read frozen Content data | Audio GDD rejected-patterns table (citing `IDB:991`); Instruments IW-21 |

#### Presentation-to-presentation lateral edges

Presentation systems read each other only through the stage order of ADR-0009 § 3: a presenter's `reads` entries must each be a strictly earlier stage, and `freeze()` fails the boot otherwise. The lateral edges that exist are:

- **Camera → HUD, Instruments, Lighting:** `CAMERA_SOLVE` (stage 1) publishes `pose_q`, `f_q` and the `world_to_hud` inputs before the HUD stage projects this frame's prompts (ADR-0008 § 3.3, § 6.1).
- **Avatar → Lighting:** the beam pose is published in stage 2 and read in stage 6.
- **Filter → Instruments, HUD, Lighting, Audio:** `perceived()` and the decoy list (stage 3) are read by later stages; the Filter has no write path to a camera or viewport property.
- **HUD ↔ Instruments:** a shared Theme and the honest-numeral grid (`HONEST_TICK_MS`, HUD HS-14; Instruments IW-6) and one flash arbiter; neither draws into the other's layer.
- **Lighting, HUD, Chart Table:** the three flash governors share the Pipeline's frame-mean limit (PR-13, IC-7).
- **Chart Table ↔ HUD ↔ Instruments:** a shared component grammar (`design/ux/interaction-patterns.md`, Chart Table dependency table); the Chart Table owns layers 3 and 4 and keeps running on engine time while the kernel is `PAUSED`.

---

## API Boundaries

Signatures are GDScript (typed). They restate the GDD contracts at the boundary; the GDD anchor in brackets is the authority.

### Kernel ingress and registration [command-simulation-layer CR-4..CR-10, CR-19]

```gdscript
# Ingress: a method of SurfaceHandle (ADR-0002), issued once at BOOTING to the Input Adapter by
# kernel.issue_surface_handle(&"input_adapter"). Fire-and-forget: returns nothing, calls nothing.
func submit(type: StringName, payload: Dictionary, actor: int = NO_ACTOR, correlation: int = 0) -> void
# SYSTEM-origin submission from a step hook or apply (dedupe optional); on HookContext and ApplyContext only.
func submit_system(type: StringName, payload: Dictionary, dedupe_key: Variant = null) -> void

# Registration — methods of the module's BlockRegistrar (ModuleFacet.registrar, ADR-0003); legal only in
# BOOTING, set to null at the freeze; violations fail boot.
func register_block(desc: BlockDescriptor) -> void       # block_id, owner, schema_version, class, persist,
                                                          # persist_scope, scope, gameplay, interpolated,
                                                          # repl_hint, restore_after, callables
func register_command(desc: CommandTypeDescriptor) -> void  # allowed_origins, actor_binding, actor_source,
                                                          # coalesce, max_age_ticks, allowed_when_paused, writes_blocks
func register_hook(desc: HookDescriptor) -> void          # hook_id, owner_name, divisor, offset,
                                                          # after: [{id, tolerates_stale}], callable
func register_key_space(desc: KeySpaceDescriptor) -> void   # dirty-key layout (ADR-0004 § 3)

# Subscription — a method of a class-bound facet, not of the kernel (ADR-0004): NotifyFacet is held by the
# composition root (the subscriber class is fixed at issue); presenters hold PresentationFacet (ADR-0009).
# handler(batch: ChangeBatch, ctx: NotifyContext) -> CompletionToken
func subscribe(block_id: StringName, handler: Callable, name: StringName) -> SubscriptionHandle
```

*(Amended 2026-10-29 — ADR-0002 / ADR-0003 / ADR-0004 sync)* There is no `Kernel` global (ADR-0001 § 5): the composition root (`src/game/game.gd`) hands each module the narrowest facet through `setup(facet)`. Control-plane requests (`request_pause`, `request_resume`, `request_snapshot`, `request_restore`, `request_new_game`, `request_shutdown`, seat binding) are methods of a `ControlHandle` issued to the named callers only (`save_load`, `pause_gate`, `game.gd`), not commands. `request_restore(payload: Dictionary, restore_epoch: int, expected_state_hash: int = 0, has_expected_hash: bool = false) -> void` returns a verdict `NONE` when `has_expected_hash` is `false` (ADR-0002 § 6).

**Invariants callers respect:** never construct a command with `issuer_peer`; payloads hold only whitelisted Variant types (finite floats, depth ≤ 8, ≤ 4 KiB, no Object/Callable/NodePath/RID); never submit from a PRESENTATION handler. **Kernel guarantees:** exactly one terminal result per command; apply order ascending `(queued_tick, host_order)`; subscribers see only post-P4 state.

### Owner-callable contract [CR-19, ruling R3]

```gdscript
# Every kernel-invoked owner callable receives a ctx and returns only ctx.complete().
func validate(cmd: Command, views: ReadViews, ctx: ApplyContext) -> CompletionToken:
	if not _target_exists(cmd.payload):
		ctx.reject(&"NO_SUCH_TARGET")
		return ctx.complete()
	ctx.ok()
	return ctx.complete()

func apply(cmd: Command, scope: WriteScope, ctx: ApplyContext) -> CompletionToken
func snapshot(ctx: SnapshotContext) -> CompletionToken           # ctx.put(plain_dictionary)
func restore(data: Dictionary, stored_version: int, ctx: SnapshotContext) -> CompletionToken
func post_restore(ctx: SnapshotContext) -> CompletionToken
func read_view(ctx: Object) -> CompletionToken
```

⚠️ `CompletionToken` return — Godot 4.7 (post-cutoff; HIGH only if owners are subclasses). The 4.7 rule (an override of a typed method inherits its return type and needs an explicit `return`) applies **only if** owner callables override a base class such as `BlockOwner`. ADR-0003 (Proposed) chose registered `Callable`s on plain (normally `RefCounted`) owners with no base class, so the rule is moot for owner methods and GDScript's ordinary "a typed function returns on every path" check applies; its context classes override nothing. *(Amended 2026-10-29 — ADR-0003 sync)* Either way the kernel checks the returned token's identity with a real `if`, never an `assert`, because `assert` is stripped from release exports (KRN-CTX-06). Verified against `docs/engine-reference/godot/breaking-changes.md` § 4.6 → 4.7 (GDScript). Behaviour confirmed: NEEDS VERIFICATION on a release export.

### Mutator contract [CR-9, LS-18, LY-27, BDY-21]

```gdscript
# Class A: pure check + infallible apply. The caller's validate runs the check.
func check_weld_join(plan: JoinPlan) -> StringName                # &"" = OK, else a reason
func weld_join(scope: WriteScope, plan: JoinPlan, out: WeldResult) -> int   # status OK / REFUSED / -code; fills caller-owned out {seam_id, opening_ids} (ADR-0003 § 5)
# Class B: clamps, never rejects, no-ops on stale ids; returns void (ADR-0003 § 5).
func push_stimulus(scope: WriteScope, kind: int, source_ref: int, cls: NoiseClass) -> void
# Cross-system Class A mutators called inside one apply return a status the caller must check.
func rebase_on_attach(scope: WriteScope, surviving: int, absorbed: int, frame_shift_px: Vector2i) -> int  # MutatorStatus.OK, REFUSED, or -fault_code (ADR-0003 § 5)
```

Mutators mark their own dirty keys explicitly (4.7 packed-array element writes call no setter). A mutator called outside a kernel WriteScope writes nothing, counts `WRITE_OUTSIDE_SCOPE`, and reports a violation.

### Read views [CR-12, LS-19, EM-25]

```gdscript
# Getter-only facade or deep copy; never a live container. Hot paths use typed scalar getters
# or caller-owned buffers and allocate nothing per call.
func compartment_p(cid: int) -> int
func failures(filter: FailureFilter, out: FailureSlice) -> void
func rects_in(body_id: int, ax0: int, ay0: int, ax1: int, ay1: int, out: QueryBuffer) -> int   # Layout OccluderSource (ADR-0014); spans are occupied (x0, y0, x1-1, y1-1)
func solid_in(body_id: int, ax0: int, ay0: int, ax1: int, ay1: int, out: QueryBuffer) -> int   # Layout SolidQuery (ADR-0014); wrapped only by ADR-0015 LayoutCollide
# *(Amended 2026-10-29 — ADR-0014 sync)*
func lit(ref: int) -> bool                                        # Emitter, hysteresis on act_lux_pm
```

Any simulation-side read of a `PERCEPTION_OUT` block returns an empty view and counts `PERCEPTION_READ_BLOCKED`.

### Pure simulation functions shared across systems

```gdscript
class_name LocomotionStep extends RefCounted   # Body BDY-10; reused by Horror
static func step(state: LocoState, intent: LocoIntent, collide: SolidQuery) -> LocoResult
static func step_into(state: LocoState, intent: LocoIntent, collide: SolidQuery, out: LocoResult) -> void

# Kernel D.10 — pure, golden-vector pinned
static func rng_derive(world_seed: int, stream_name: StringName, counter: int) -> int
static func rng_derive_epoch(world_seed: int, stream_name: StringName, counter: int, restore_epoch: int) -> int
```

### Perception Filter [PF-*]

```gdscript
# The only path from a true value to a corruptible display. Pull query; no block, no hook.
func perceived(ref: PerceptionRef, viewer: int) -> Perceived
# PerceptionRef kinds: GRAPH_NODE, GRAPH_COMPARTMENT, SUIT_KEY, SCANNER, HOUSING_GLOW, STATE_GLYPH, MACHINE_FRAME
```

**Guarantees:** a ref whose tag is a truth channel always returns the true value; with no active lie the result equals the true value; the Filter writes only `CanvasItem`, `Material` and `Light2D` properties and its decoy list.

### Presentation subscriber contract [PIPE PR-20, LO-1]

```gdscript
# Handler: a method Callable on a long-lived object (no lambda, no .bind()); never awaits (ADR-0004, ADR-0009 § 2).
func _on_changes(batch: ChangeBatch, ctx: NotifyContext) -> CompletionToken:  # marks dirty only; no draw, no read of other blocks
	_dirty.absorb(batch)                               # DirtyAccumulator (ADR-0004); a Dictionary.merge of a packed array was never valid
	return ctx.complete()                              # ADR-0003 callable contract, ADR-0009 § 2

# Presenters define no _process. PresentationDriver calls present(frame) once per frame, from PipelineRoot's single
# _process, in stage order (ADR-0009 § 3). Reads committed views at frame.alpha.
func present(frame: PresentFrame) -> void:
	if _dirty.is_empty() and not _animating():         # early-out only when both dirty and animating are empty (ADR-0009 § 2):
		return                                         # interpolation needs a draw on frames with no notification
	_redraw(frame.alpha)                               # alpha is frame.alpha, read once per frame; never Kernel.tick_alpha
```

**Forbidden:** writing a block, submitting (unless INPUT_SURFACE), moving a physics body, writing the camera or a viewport property (pipeline root only), reading `sanity.perception` (Filter and Debug only).

### Save/Load host API [SL-21]

```gdscript
func new_game(world_seed: int) -> void
func save_and_quit() -> void
func continue_newest() -> void
func cancel_held_close() -> void
var save_status: SaveStatus          # IDLE, WRITING, WRITTEN, FAILED (host-local read)
```

Save/Load registers no command; `save_now` and `load(generation)` exist only in dev builds.

### Content loader [ADR-0007 § 1.7, § 2.3]

```gdscript
# Composition root only, before the kernel exists (so before the kernel's BOOTING). The only entry; runs stages 1-4 of § 1.7.
class_name ContentLoader extends RefCounted
func _init(source: ContentSource, kernel_tick_rate: int) -> void
func load_all() -> ContentLoadResult

class_name ContentLoadResult extends RefCounted
var content: ContentSet            # null when errors is non-empty
var errors: Array[BootError]       # empty on success; all errors of the failing stage
var stage_failed: int              # 0 = none, else 1..4
```

*(Amended 2026-10-29 — ADR-0007 sync)* On any error the composition root never starts the kernel. `ContentSet` is frozen and read-only; catalogs reach modules as their own `setup()` arguments. Only `src/data/` may use `FileAccess`, `DirAccess` or `JSON` for content (lint, ADR-0010).

### Pipeline root and render layout [ADR-0008 § 3, § 6; PIPE PR-12, PR-20]

```gdscript
class_name PipelineRoot extends Node             # process_priority = ProcessPriority.PRESENTATION; the single PresentationHost
func setup(clock: PresentationClock, driver: PresentationDriver, tuning: RenderTuning,
		settings: DisplaySettings, hud_layout: HudLayout) -> void
func request_zoom_step(direction: int) -> void   # +1 in, -1 out; edge-triggered; rate-limited

class_name RenderLayout extends RefCounted       # pure value; written only by PipelineRoot; handed out read-only
func world_to_hud(body_pos: Vector2i, pose_q: Vector2i) -> Vector2i          # the only projection to HUD coordinates; floored to the HUD grid
func mouse_to_world_aim(cursor_px: Vector2i, f_q: Vector2i) -> Vector2       # open loop (D.14b); the only mapping that feeds beam aim
func mouse_to_world_view(cursor_px: Vector2i, pose_q: Vector2i) -> Vector2   # picking and pings; never beam aim (lint, ADR-0010)
```

*(Amended 2026-10-29 — ADR-0008 sync)* `PipelineRoot` is the only writer of viewport, container and layer geometry; the camera is solved in the `CAMERA_SOLVE` stage and `frame_pre_draw` only writes `canvas_transform` and the parallax offsets. Subscribers receive `RenderLayout` read-only and re-derive nothing.

### Interface types still to define

The signatures above use types that no GDD defines field by field. Each is defined in the named ADR before the first story that implements it; programmers must not invent them.

| Type | Defined in |
|---|---|
| `Command`, `BlockDescriptor` (full field list), `CommandTypeDescriptor`, `ReadViews`, `WriteScope`, `ApplyContext`, `SnapshotContext`, `HookContext`, `NotifyContext`, `CompletionToken` (who constructs it; per call or pooled) | ADR-0003 |
| `ChangeBatch` (kernel-held PackedInt64Array behind key(i)/has(key)), `NotifyEvent`, `DirtyAccumulator`, `KeyBuf`, `KeySpaceDescriptor`, `SubscriptionHandle`, `NotifyFacet` (src/kernel/facets/ except KeySpaceDescriptor) | ADR-0004 (Proposed) |
| `FAULT_CODE` table: one closed enumeration per owning system, the kernel's own codes, and who may raise which | ADR-0003 |
| Per-command payload schemas and the payload validator | ADR-0002 (each command's owner lists its schema) |
| `QueryBuffer`, `RegionBox`, `ChangedRegion`, `SolidQuery`, `OccluderSource` | ADR-0014 (Proposed) *(Amended 2026-10-29 — ADR-0014 sync)* |
| `JoinPlan`, `WeldResult`, `FailureFilter`, `FailureSlice` | ADR-0011, ADR-0012 |
| `LocoState`, `LocoIntent`, `LocoResult` | ADR-0015 |
| `PerceptionRef`, `Perceived`, `PerceptionTag` | ADR-0024 |

**How presentation reaches the kernel.** ADR-0001 (Proposed) decides it: no autoload and no `Kernel` global; the composition root hands each module the narrowest injected facet through `setup(facet)`. A simulation module gets a `ModuleFacet` (registrar, simulation views, fault domain, RNG facet and a read-only `ClockView`, ADR-0003); presentation gets a `PresentationFacet` (class-bound `subscribe`, `PresentationViews`, a read-only `PresentationClock` with `tick_alpha`, `T`, `running`, `state` and `load_seq`; ADR-0009 § 1), and per-frame alpha arrives as `frame.alpha` on the `PresentFrame`. Submit and control rights come only from the named handles of ADR-0002 (`SurfaceHandle`, `ControlHandle`). The examples in this document therefore use facets, not `Kernel.`; the remaining `Kernel.step()` and `Kernel.deliver_frame_end()` in the frame diagram are the `TickDriver`'s own calls on the kernel it was constructed with.

**How an interface is expressed.** GDScript has no interfaces. ADR-0010 (Proposed) picks `@abstract` ports for injected seams (probed on the 4.7.2 editor binary; fallback a plain base class whose methods `push_error`); kernel-invoked owner callables stay registered Callables (ADR-0003); duck-typed checking survives only as `ContractCheck` for recorded stubs. *(Amended 2026-10-29 — ADR-0010 sync)*

### Engine-type flags at the boundaries

| Type used in an interface | Version note | Status |
|---|---|---|
| `RefCounted` as block base (never `Resource`, never `Node`) | unchanged | LOW |
| `Callable` returning a typed `CompletionToken` | 4.7 typed-return override rule | HIGH, verify |
| `PackedInt32Array` / `PackedInt64Array` columns | 4.7 element writes skip setters | HIGH, explicit dirty marking |
| `@abstract class OccluderSource` and the other injected ports | 4.5 feature | HIGH; ADR-0010 chooses `@abstract` ports (probed on the 4.7.2 editor binary; release export checked by LINT-SPIKE-1); fallback: plain base class with a `push_error` stub |
| `Rect2i`, `Vector2i` in signatures (never in snapshots) | unchanged | LOW |
| `StringName` action names | unchanged | LOW |
| `InputEvent.DEVICE_ID_MOUSE` / `DEVICE_ID_KEYBOARD` | 4.7 | HIGH |

---

## ADR Audit

*(Amended 2026-10-29 — ADR sync)* The ten Foundation ADRs, ADR-0001 to ADR-0010, are written, all with status Proposed (none Accepted: stories that cite a Proposed ADR are blocked until it is). The full audit is `docs/architecture/architecture-review-2026-10-28.md` (verdict FAIL, the expected state: ADR-0011 to ADR-0034 are unwritten and no ADR is Accepted); this table records only the headline per ADR.

| ADR | Engine Compat | Version | GDD Linkage | Conflicts | Valid |
|---|---|---|---|---|---|
| ADR-0001 Kernel tick driver and frame order | Engine Compatibility section present; MEDIUM; spike KRN-LAT-02 gates Accept | 4.7.2 | cites the kernel GDD (CR-1a, D.5), Player Controller, Pipeline | stale tick-source text in four GDDs listed in its § GDD sync (OQ-1) | Proposed |
| ADR-0002 Authority model and command pipeline | present; MEDIUM | 4.7.2 | kernel CR-4..CR-7, CR-15..CR-17 | amends one kernel GDD sentence on control-plane outcomes | Proposed |
| ADR-0003 State-block registry and owner contract | present; HIGH (owner-callable contract); release-export check gates Accept | 4.7.2 | kernel GDD and every block owner | see the review's Cross-ADR Conflicts section | Proposed |
| ADR-0004 Change notification and dirty keys | present; MEDIUM; gated on KRN-LAT-02 notify cost | 4.7.2 | kernel CR-10, D.9 | see the review's Cross-ADR Conflicts section | Proposed |
| ADR-0005 Determinism, keyed RNG, encoding and hashing | present; MEDIUM | 4.7.2 | kernel D.10, D.11, CR-8 | golden vectors not yet written (OQ-8) | Proposed |
| ADR-0006 Save container, generations, versioning, restore epoch | present; MEDIUM; SAVE-WRK-1-ACCEPT gates Accept | 4.7.2 | Save/Load GDD | OQ-14 snapshot hitch ruling proposed | Proposed |
| ADR-0007 Static content data pipeline and stable type IDs | present; LOW (APIs); CONTENT-EXP-1 gates Accept | 4.7.2 | Item DB, Layout, Sanity, Derelict | see the review's Cross-ADR Conflicts section | Proposed |
| ADR-0008 Render pipeline wiring and integer scaling | present; HIGH; renderer and driver provisional (Forward+, Vulkan) pending the lighting spike | 4.7.2 | Pipeline, Lighting, HUD | see the review's Cross-ADR Conflicts section | Proposed |
| ADR-0009 Presentation subscriber contract | present; MEDIUM; gated only on the order half of KRN-LAT-02 | 4.7.2 | Pipeline, HUD, Filter, Lighting | stage order is a convention beyond three GDD-stated edges (TD-22) | Proposed |
| ADR-0010 Source layout, static typing and lint enforcement | present; HIGH; LINT-SPIKE-1 gates Accept | 4.7.2 | kernel CR-14 and the cross-GDD lints | see the review's Cross-ADR Conflicts section | Proposed |

### Traceability coverage

Current state per `docs/architecture/requirements-traceability.md` (last updated 2026-10-29, after the Foundation gap adjudication): 1138 requirements registered in `docs/architecture/tr-registry.yaml`; covered by an Accepted ADR 0; covered by a Proposed ADR 192 (16.9 %); partial 187 (16.4 %); gaps 759 (66.7 %). By planned layer: Foundation ADR-0001..0010 190 covered, 156 partial, 0 gap; Core 0 / 11 / 216; Feature and Presentation 2 / 8 / 522; Deferred 0 / 0 / 9; GDD-owned 0 / 12 / 12. The Required ADRs below are still the plan for the 24 unwritten ADRs. The 794-row table that follows is the original v2 baseline assignment (`docs/architecture/tr-baseline.md`, column **Planned ADR**); the registry has since grown to 1138 requirements, and the traceability index, not this table, is the live count.

| ADR | Title | Primary TRs | Shared TRs | Requirements (primary) |
|---|---|---|---|---|
| ADR-0001 | Kernel tick driver and frame order | 13 | 0 | charttable 005; cmd 001-006; derelict 021; hud 010; light 008; pc 004; pxl 021-022 |
| ADR-0002 | Authority model and command pipeline | 11 | 3 | cmd 008-010, 012, 027-028, 030-031, 034; interact 029; pc 024 |
| ADR-0003 | State-block registry, write discipline and owner-callable contract | 24 | 0 | assembly 008, 011; attention 001; charttable 023; cmd 007, 011, 014-015, 021-022, 026, 029; emitter 001; horror 001; interact 014; pc 002, 005, 019; sanity 001; section 001; shipbody 005, 011; suit 001; webmap 002 |
| ADR-0004 | Change notification and dirty keys | 13 | 0 | attention 026; cmd 023-024; emitter 026; ftrail 007; light 007; lsg 020; section 022; shipbody 022, 027; suit 013; webmap 023-024 |
| ADR-0005 | Determinism, keyed RNG, canonical encoding and hashing | 19 | 2 | assembly 016; attention 013; boarding 027; cmd 019-020, 032; debug 019; horror 016, 020; itemdb 012; lsg 021; pc 027; sanity 016-017; section 026; shipbody 031; suit 005, 030; webmap 026 |
| ADR-0006 | Save container, generations, versioning and restore epoch | 65 | 0 | assembly 012-013; attention 023-024, 027; boarding 016-018; charttable 020-021; cmd 016-018; derelict 009; emitter 007; horror 017-019; itemdb 011, 013, 020; lsg 022-023; pc 020-022; sanity 020-021; saveload 001-020, 022-023, 025-026; section 006-007; shipbody 007-009; suit 007-011, 021, 029; webmap 030 |
| ADR-0007 | Static content data pipeline and stable type IDs | 64 | 1 | attention 010; audio 011-012; boarding 014, 019, 025; charttable 013; derelict 001-004, 007-008, 013-017, 020, 023-025, 027-029; horror 006; hud 026; instr 007-009, 028; interact 030; itemdb 001-010, 015-019, 021-024, 026-028; lsg 030; pc 015; sanity 024-026; saveload 024; section 003, 029; suit 028; webmap 006 |
| ADR-0008 | Render pipeline wiring and integer scaling | 26 | 0 | charttable 017, 030; hud 002-003, 011; light 019; percfilter 030; pxl 001-004, 010-016, 019, 023-026, 028-030 |
| ADR-0009 | Presentation subscriber contract (v2: scene shape split out to ADR-0034; `/architecture-review` re-sorts these rows between the two) | 27 | 0 | assembly 030; boarding 026; ftrail 017; hud 022; instr 001-002, 005, 010, 012-014; light 001, 003; pc 026; percfilter 025; pxl 005-009, 017-018, 020; section 024-025; shipbody 032-033 |
| ADR-0010 | Source layout, static typing and lint enforcement | 6 | 0 | cmd 033; derelict 026; ftrail 033; section 002; shipbody 002; suit 006 |
| ADR-0011 | Life-Support Graph data model | 11 | 0 | instr 027; itemdb 014; lsg 001-006, 018-019, 024 |
| ADR-0012 | Graph propagation scheme | 17 | 0 | derelict 011-012; lsg 007-013, 015-017, 025-029 |
| ADR-0013 | Ship layout block, geometry and ID allocation | 14 | 0 | assembly 025; section 004-005, 010, 012-014, 018-021, 027-028, 030 |
| ADR-0014 | Geometry broadphase and OccluderSource seam | 5 | 1 | light 014; section 011, 015-017 |
| ADR-0015 | Hand-rolled integer kinematics and the ship body | 24 | 0 | pc 013-014; shipbody 001, 003-004, 006, 012-021, 023-026, 028-029, 034; webmap 008 |
| ADR-0016 | Restore ordering and cross-block verification | 15 | 0 | assembly 009; attention 025; boarding 005; cmd 013; emitter 008-009; interact 016-017; sanity 019; section 008-009; shipbody 010, 030; suit 012; webmap 022 |
| ADR-0017 | Emitter state and integer light-exposure query | 32 | 2 | boarding 013; derelict 018; emitter 002-006, 010-025, 027-030; horror 009-010; interact 022-023; webmap 028 |
| ADR-0018 | Shadow origin and beam occlusion consistency | 1 | 0 | light 015 |
| ADR-0019 | Player controller split and input contexts | 21 | 1 | boarding 007-008; charttable 009-010, 012; hud 029; pc 001, 003, 006-012, 016-018, 023, 025, 029 |
| ADR-0020 | Interaction channels and verb declaration table | 34 | 0 | assembly 017; boarding 001-004, 006, 011, 020-022, 028; derelict 019; instr 021; interact 001-013, 015, 018-020, 025-027; itemdb 025 |
| ADR-0021 | Atomic weld/cut/install transaction | 20 | 2 | assembly 001-007, 010, 014-015, 018-024, 026-027, 029 |
| ADR-0022 | Attention model, stimulus API and breach state machine | 28 | 0 | attention 002-009, 011-012, 014-021, 028-031, 034; boarding 015, 029; interact 024; lsg 014; webmap 027 |
| ADR-0023 | Horror entity simulation | 18 | 0 | horror 002-005, 007-008, 011-015, 021-023, 025, 027-029 |
| ADR-0024 | Perception isolation and the Perception Filter | 64 | 0 | attention 022; audio 010, 024; boarding 023; cmd 025; debug 012, 028; ftrail 002-004; horror 026; hud 004-006, 012; instr 003-004, 006, 011, 015-020, 022-024, 026, 029-030; light 021; percfilter 001-019, 021-023, 026-029; sanity 013, 018; saveload 021, 027; suit 023; webmap 029 |
| ADR-0025 | Sanity model and lie lifecycle | 19 | 0 | interact 028; sanity 002-012, 014-015, 022-023, 028-030 |
| ADR-0026 | World model, derelict loading and transit | 30 | 1 | assembly 031; boarding 009-010, 012; charttable 014; derelict 005-006, 010, 022, 030; section 023; webmap 001, 003-005, 007, 009-021, 025 |
| ADR-0027 | Suit vitals and equipment | 14 | 0 | suit 002-004, 014-020, 022, 024-025, 027 |
| ADR-0028 | 2D lighting budget and modifier chain | 52 | 1 | charttable 029; ftrail 001, 005-006, 008-016, 018-028, 030; horror 024, 030; instr 025; light 002, 004-006, 009-013, 016-018, 020, 022-030; percfilter 020; pxl 027 |
| ADR-0029 | Audio architecture | 30 | 0 | attention 032; audio 001-009, 013-023, 025-030; charttable 025; hud 021; percfilter 024 |
| ADR-0030 | UI framework | 39 | 0 | assembly 028; attention 033; boarding 024; charttable 001-004, 006-008, 011, 015-016, 018-019, 022, 024, 026-028; ftrail 029; hud 001, 007-009, 013-020, 023-025, 028, 030; interact 021 |
| ADR-0031 | Diagnostics, observer and dev-feature gating | 33 | 0 | debug 001-011, 013-018, 020-027, 029-030; ftrail 031; hud 027; instr 031; pc 030; sanity 027; shipbody 035 |
| ADR-0032 | Replication shape for co-op (VS-B) | 5 | 1 | assembly 032; attention 035; ftrail 032; pc 028; suit 026 |
| ADR-0033 | Presentation-only debris physics (VS) | 0 | 0 | — |
| ADR-0034 | PROJECTOR scene shape and Y-sort (split from ADR-0009 in v2) | 0 | 0 | rows move here from ADR-0009 at `/architecture-review` |
| **Total** | | **794** | 15 | |

---

## Required ADRs

Thirty-four decisions. Numbers are sequential; the GDD-named candidates each one absorbs are in brackets, so `/architecture-decision` can find the anchors. Several GDD candidates were the same decision under two names and are merged: ADR-LAY-03 and ADR-BODY-03 (scene projection); ADR-BODY-04 and the kernel tick ADR (tick source); ADR-LAY-02 and Emitter candidate 4 (OccluderSource); ADR-LAY-07 belongs to the kernel restore order. **v2 changes:** ADR-0009 is split so the subscriber contract every presentation system needs is Foundation while the Y-sort scene shape (gated by SP-YSORT-1) becomes Core ADR-0034; ADR-0018 folds into ADR-0028 if the lighting spike settles both at once; ADR-0006, ADR-0007 and ADR-0024 carry 64-65 requirements each and are written as sectioned ADRs with one numbered sub-decision per section (container / versioning / restore epoch; content pipeline / boot lints / ids; isolation / Filter query / truth-channel tags) so each part can be reviewed on its own.

### Must have before coding starts (Foundation)

*(Amended 2026-10-29 — ADR sync)* All ten are written (`docs/architecture/adr-0001-…` to `adr-0010-…`) and Proposed; none is Accepted. Their "Decides" cells below are the planning text; each ADR is the authority for what it decided.

| ADR | Title | Absorbs | Decides |
|---|---|---|---|
| ADR-0001 | Kernel tick driver and frame order | kernel tick ADR, ADR-BODY-04 | `_process` TickDriver, priority order, `tick_alpha`, `present_time`, where the Adapter samples input |
| ADR-0002 | Authority model and command pipeline | kernel authority-model ADR, kernel OQ 1, 4 | host authority, HOST/MIRROR, command anatomy, per-command payload schemas, binding table, PlayerId keying, control plane API (ADR-0003 owns the kernel state machine; each states the edge) |
| ADR-0003 | State-block registry, write discipline and owner-callable contract | kernel OQ 7, 10 | RefCounted blocks, owner base class or registered Callables, WriteScope, Class A/B mutators, `ctx.complete()`, the interface types and FAULT_CODE table, hook topo-sort, kernel state machine (incl. OQ-5) |
| ADR-0004 | Change notification and dirty keys | kernel OQ 8, ADR-LSG-06 | kernel observer registry vs signals (OQ-4), subscriber classes, int dirty keys, catch-up merge, a debug-build check that every changed block marked a dirty key |
| ADR-0005 | Determinism, keyed RNG, canonical encoding and hashing | kernel H64/mix64, ADR-LSG-07, K-1 | integer-only sim, `rng_derive(_epoch)`, TLV encoding, H64, cached block hashes, replay guarantees |
| ADR-0006 | Save container, generations, versioning and restore epoch | ADR-LSG-05, S-1, rulings R13/R14 | file container, ring, write protocol, worker thread, migration, 64-bit interchange, PROFILE split, `slot_epoch_a/b.bin` |
| ADR-0007 | Static content data pipeline and stable type IDs | ADR-LSG-08, ADR-LAY-04 | JSON under `assets/data/`, manifests, int-casting, immutable catalogs, append-only ids, boot lints, export filters |
| ADR-0008 | Render pipeline wiring and integer scaling | pipeline render-wiring ADR | SubViewport stack, k and H, RENDER_MODE, stretch pins, zoom by logical size, **renderer method (Forward+ vs Compatibility) and Windows driver (D3D12 vs Vulkan)**, the single draw-call budget table |
| ADR-0009 | Presentation subscriber contract | — | what presentation may read and write, dirty-mark-then-draw, integer placement at `tick_alpha`, `physics_interpolation` off, INPUT_SURFACE as the only submitting class |
| ADR-0010 | Source layout, static typing and lint enforcement | cross-GDD lints | `src/` split, dependency-direction rules (§ System Layer Map), interface mechanism, scene-tree-free sim, warnings as errors, the lint tool and which rules it can enforce (regex vs AST vs runtime counters), how zero-allocation claims are verified, gdUnit4 version pinned against 4.7.2, GDExtension escape-hatch policy |

### Must have before the relevant Core system is built

| ADR | Title | Absorbs | Decides |
|---|---|---|---|
| ADR-0011 | Life-Support Graph data model | ADR-LSG-01, -02, -04 | SoA packed arrays, fixed-point units, append-only topology identity |
| ADR-0012 | Graph propagation scheme | ADR-LSG-03 | three chained hooks, 12 phases, one Jacobi pass, fairness gate, dormant sections |
| ADR-0013 | Ship layout block, geometry and ID allocation | ADR-LAY-01, -05, -06 | instances by reference, integer rects, sockets, weld offsets, id split with the Graph |
| ADR-0014 | Geometry broadphase and OccluderSource seam | ADR-LAY-02, Emitter candidate 4 | one rect set, per-body grid, allocation-free queries, version counters |
| ADR-0015 | Hand-rolled integer kinematics and the ship body | ADR-BODY-01, -02 | no engine physics, `LocomotionStep`, swept clamp, `body.state` shape, rebase on weld/detach |
| ADR-0016 | Restore ordering and cross-block verification | ADR-LAY-07 | `restore_after` DAG, two-phase restores, `post_restore` order, verifiers, binding rebuild |
| ADR-0017 | Emitter state and integer light-exposure query | Emitter candidates 1-3, 6 | integer cone and occlusion test, pair cache, work-unit budget, lit hysteresis |
| ADR-0018 | Shadow origin and beam occlusion consistency | Emitter candidate 5, LO-10 | how drawn shadows match the foot-origin segment test (lighting spike) |
| ADR-0019 | Player controller split and input contexts | PC-1, PC-2, PC-14 | Adapter / Sim / Presenter split, intents, resync, contexts, 4.7 device IDs |
| ADR-0034 | PROJECTOR scene shape and Y-sort | ADR-LAY-03 ≡ ADR-BODY-03 | per-body roots, reparent vs rebuild, per-body vs per-section TileMapLayer, Y-sort key and entity_id namespaces, camera-limit rectangle (gated by SP-YSORT-1; needed before the PROJECTOR story, not before coding starts) |

### Should have before the relevant Feature or Presentation system is built

| ADR | Title | Decides |
|---|---|---|
| ADR-0020 | Interaction channels and verb declaration table | channel records, act-only-on-lit, verb table, harm gate call |
| ADR-0021 | Atomic weld/cut/install transaction | two-phase begin/complete, mutator order, status checks; **no-rollback vs restoring the four touched blocks on a fault** (OQ-15), the weld property test as an acceptance gate |
| ADR-0022 | Attention model, stimulus API and breach state machine | push_stimulus, one integrator, tell-window contract, exogenous permission |
| ADR-0023 | Horror entity simulation | horror.step, behaviour states, footprints, LocomotionStep reuse |
| ADR-0024 | Perception isolation and the Perception Filter | PERCEPTION_OUT, `perceived()`, truth-channel tags, decoys, audio lies |
| ADR-0025 | Sanity model and lie lifecycle | drift, stages, harm gate, guard rails, data-driven lie targets |
| ADR-0026 | World model, derelict loading and transit | `web.state`, `init_world`, construction order, dormant sections, pod legs |
| ADR-0027 | Suit vitals and equipment | `suit.state`, vitals hook, breath source, tools, carry, deployables |
| ADR-0028 | 2D lighting budget and modifier chain | node pools, shadow slots, demotion, modifier chain, glow/HDR, trail budget, the shared frame-mean flash rule (IC-7), 4.7 line-drawing change for trails and hairlines |
| ADR-0029 | Audio architecture | bus tree, voice pool, `cue_requests()`, whether audio can lie |
| ADR-0030 | UI framework | HUD layout, Theme, dual focus, menus state machine, localisation keys, 4.7 line-drawing change for instrument needles, AccessKit recorded as deferred to Alpha |
| ADR-0031 | Diagnostics, observer and dev-feature gating | overlay channels, feature tags, release exclusion by file |

### Can defer (not MVP)

| ADR | Title | When |
|---|---|---|
| ADR-0032 | Replication shape for co-op | before VS-B |
| ADR-0033 | Presentation-only debris physics | before VS (ADR-BODY-05) |

### Spikes that gate ADRs

*(Amended 2026-10-29 — ADR-0009 / ADR-0010 sync)* LINT-SPIKE-1 (ADR-0010 Engine Compatibility (a) to (j): `@abstract` on a release export, promoted warnings and `can_instantiate()` on the editor binary, lexical scanner corpus run, IntMath divisor guard, export pack scan) gates ADR-0010 and runs beside KRN-LAT-02.

ADR-0009 is gated only on KRN-LAT-02's order result (host `_process` after TickDriver) on a desktop release export; the former PRES-SPIKE-1 is story-level evidence: `frame_pre_draw` ordering goes to ADR-0008 pipeline spike steps 11 and 12, `speed_scale` freezing to the first story that freezes a particle or animation, and driver overhead to the Pipeline story's perf check on RM-DESKTOP and RM-DECK.

| Spike | Gates | Source |
|---|---|---|
| S0-a / S0-b (packed-array alias semantics; signed 64-bit literal; arithmetic `>>` and the masking logical shifts need; H64 and `mix64` golden vectors; S0-b no longer verifies an `integer_division` ignore, no Graph file carries one and division goes through `IntMath`, ADR-0011 section 2.3) *(Amended 2026-10-29 — ADR-0011 sync)* | ADR-0005, ADR-0010, ADR-0011 | life-support-graph |
| KRN-LAT-02 (cross-node `process_priority` order, `Engine.time_scale`), built as a headless kernel skeleton: `ctx.complete()` checks, hook topo-sort, WriteScope and observer-registry notify cost under gdUnit4 (also answers OQ-4); also records the adapter → driver → `SaveHost` (HOST_SERVICE) → presentation host call order on a dev build and a release export | ADR-0001, ADR-0003, ADR-0004, ADR-0009 (order half only), ADR-0016 (stub-level KRN-RST-05: recorded stub blocks under four linear extensions) *(Amended 2026-10-29 — ADR-0016 sync)* | command-simulation-layer |
| SP-YSORT-1 (Y-sort tie under reparent; per-body vs per-section TileMapLayer) | ADR-0034 | ship-section-and-layout-data-model |
| Pipeline spike steps 11 and 12 (SubViewport wiring, RENDER_MODE A/B, `RenderingServer.frame_pre_draw` writing only the two canvas writes from values CAMERA_SOLVE solved in `PipelineRoot._process`; probed 2026-10-28 on 4.7.2 editor, release-export re-run pending; no `process_priority` ordering) | ADR-0008 | pixel-perfect-render-pipeline-camera |
| Lighting spike steps 1-10 (art bible 8.12; LO-10 shadow origin). **Its renderer method and driver result gates Foundation ADR-0008, so it runs before Foundation coding**, not with the Feature tier | ADR-0008, ADR-0018, ADR-0028 | 2d-lighting-and-occlusion |
| S1 / SP-ENG-1 (GDScript hot-path cost on an RM-DESKTOP and RM-DECK release export: a stub Graph with 12 phases, `emitter.place`, the P3 loop at realistic counts, **accessor and `Callable.call` overhead per ms**, four-tick catch-up; needs an exported benchmark harness outside gdUnit4) | ADR-0012, ADR-0014, ADR-0015, ADR-0017; decides the GDExtension rung | graph, layout, body, emitter |
| Weld property test (generated section pairs: `can_join` true implies all four apply steps return OK; verifier INV-1..INV-10 after each) | ADR-0021 | modular-ship-assembly |
| SAVE-WRK-1-ACCEPT (a, b, c, d, e, h, i on a Windows `template_release`: `DirAccess.rename` over a locked or antivirus-held target, kill-during-rename-over, flush/close/re-read, quit during a write (polled, not joined), the `PROCESS_MODE_ALWAYS` poll at `HOST_SERVICE` and the window-close path, the `Mutex` status record, the kill loop; g on any Linux build: the `campaign.lock` mechanism; f with a synthetic 14-block fixture: snapshot hitch and type-walk timing). Probe results (rename over an unlocked target works on Windows but is probably not atomic; a READ-held target fails with `ERR_FAILED` and stays intact; a sleeping worker did not delay exit) are recorded in ADR-0006 § 1.5 to § 1.6 | ADR-0006 (gates Accept) | save-load-and-versioned-format |
| SAVE-WRK-1-SLICE (f on the real owner blocks with the kernel's real capture-time hash, on RM-DESKTOP and RM-DECK) | gates the Vertical Slice milestone, not Accept (decides rung 1 or 2 of ADR-0006 § 1.9) | save-load-and-versioned-format |
| CONTENT-EXP-1 (`template_release` export on Windows and Linux/Deck: manifest-listed `.json` read with `FileAccess.open` from the package; include-filter glob; `JSON.new().parse` number typing and the 2^53 boundary; BOM, duplicate key, trailing comma; content hashes equal to the editor run) | ADR-0007 (gates acceptance) | item-and-module-database, ship-section-and-layout-data-model, sanity-and-hallucination-rules |

**Spike order** (lead-programmer): S1 first, then the kernel skeleton (KRN-LAT-02), then S0-a/b, then LINT-SPIKE-1 (beside the kernel skeleton), then the weld property test, then SAVE-WRK-1-ACCEPT. The rendering spikes (pipeline steps 11 and 12, lighting, SP-YSORT-1) do not depend on the simulation and run in parallel once the first three are under way; the lighting spike must reach its renderer decision before Foundation coding.

---

## Architecture Principles

1. **The simulation is the truth, and it is pure data.** Every gameplay fact lives in a kernel-registered state block owned by exactly one system, built from `RefCounted` objects and packed arrays, with no `Node`, no `Resource` and no scene-tree dependency. It runs headless and is unit-testable. *(Pillar 1; TD-FEASIBILITY risk "graph entangled with the scene tree".)*
2. **Every change is a command, and the host is the authority, even in single-player.** Player intent enters only as a plain-data command stamped with its issuer; cross-system writes go only through the owner's published mutators inside a kernel WriteScope. Co-op is a transport added later, not a rewrite. *(TD-FEASIBILITY B2.)*
3. **Presentation reads, never writes; perception never reaches the simulation.** Presentation subscribes to change notifications and draws from read views. A hallucination is a presentation-side transform through `perceived()`; `PERCEPTION_OUT` is unreadable from the simulation, and truth channels (trails, real light radius, character sprites, crewmate beams, physical consequences) never pass through the Filter. *(Pillar 4 guard rails; "no lie is unfalsifiable".)*
4. **Deterministic by construction.** Fixed 60 Hz integer ticks, integer state, ascending-id iteration, keyed RNG streams with persisted counters, and one hash definition. Same save bytes plus the same restore epoch and command stream give the same state hash. Determinism is what makes replay tests, save verification and later co-op mirroring possible.
5. **Versioned, verified persistence from v1.** Saves serialise data models, never scenes; every block carries a schema version migrated by its owner; every file is self-verifying and written atomically; a bad save never faults the kernel and never overwrites a good one. *(TD-FEASIBILITY: "save format not versioned from v1".)*
6. **Budgets are measured, not assumed.** Every performance claim is a P99 on a Godot 4.7.2 release export on RM-DESKTOP and RM-DECK; hot paths allocate nothing per tick; work is bounded by construction (caps and counted work units, never wall-clock), so a budget miss is a tuning change, not a redesign.

---

## Open Questions

| ID | Question | Priority | Resolution path |
|---|---|---|---|
| OQ-1 | **Blocking for Player Controller, Pipeline and Suit stories.** Tick source still cited as the physics callback in three GDDs. The kernel (owner, rule G1) drives ticks from one `_process` TickDriver and says the physics step is never the clock (CR-1a). Player Controller (PC-1(a)/PC-15, ~L1021: "the Adapter samples intents in the physics callback immediately before `Kernel.step()`"), Suit (~L1441) and the Pipeline (PR-22 and ~L227/297/1219: physics_jitter_fix and max_physics_steps_per_frame "if the physics callback drives the tick") still describe the old source. Architecture follows the kernel: the Adapter samples in its own `_process` at `INPUT_ADAPTER` (-100), just before the TickDriver at `TICK_DRIVER` (-50); the single presentation host runs at `ProcessPriority.PRESENTATION` (0); `SaveHost` at `HOST_SERVICE` (-40) between the driver and presentation (ADR-0001). The consumer GDD passages also include Modular Ship Assembly (L1058, L1161, L1528, L1593, L1772, L1940). The Player Controller text was re-amended on 2026-10-28 (L62), and its acceptance criterion at ~L1077 asserts the ordering "in the same physics callback", so a story built from it would test the wrong thing. | High | Fix the three GDDs before any Controller, Pipeline or Suit story; ADR-0001 records the source |
| OQ-2 | **Renderer method and Windows driver.** The Pipeline GDD calls Forward+ "a working assumption, reversible at the lighting spike" and the Lighting GDD routes Forward+ vs Compatibility to an ADR, so Forward+ in this document's status line is provisional. 4.6+ also defaults to D3D12 on Windows; RM-DESKTOP's GTX 1050 Ti / RX 570 and the Lighting GDD's note ("binary may report vulkan") leave the driver undecided. `use_hdr_2d` and glow behave differently between methods, and the choice has Steam Deck cost consequences. | High | ADR-0008 (Proposed) pins Forward+ and the Vulkan driver on Windows provisionally, with silent driver fallback disabled, reversible at the lighting spike, which therefore still runs before Foundation coding |
| OQ-3 | **PROJECTOR scene shape** (per-body reparent vs rebuild; per-body vs per-section TileMapLayer) and the Y-sort tie rule under reparent are unverified on 4.7. | High | Spike SP-YSORT-1 → ADR-0034 (Layout requires it before the PROJECTOR story) |
| OQ-4 | **Notification transport**: kernel-owned observer registry (the kernel GDD's lean) vs engine signals. No spike was named for it in v1; it is now measured in the KRN-LAT-02 kernel skeleton, and ADR-0004 may decide on the GDD's lean if the skeleton shows no cost problem. *(Amended 2026-10-29 — ADR-0004 sync)* Decided on semantics (a kernel-owned observer registry, not engine signals; ADR-0004, Proposed); the cost gate is recorded: acceptance is gated on the KRN-LAT-02 notify-cost result. | Medium | ADR-0004 (Proposed; cost gate KRN-LAT-02) |
| OQ-5 | **Kernel FAULTED-from-BOOTING may re-enter LOADING.** Command & Simulation Layer L192/L196 need a qualifier (recommended item from the 2026-10-28 review round). | Medium | Kernel GDD fix; ADR-0003 states the kernel state machine |
| OQ-6 | **Attention AT-8 uses `rng_derive` where threat draws must use `rng_derive_epoch`** (ruling R1/R13). | Medium | Attention GDD fix; ADR-0005 |
| OQ-7 | **Seven kernel dependency edges missing from the systems index**, and the Co-op row lacks the R9 "undesigned — provisional contract" label. | Low | Systems-index pass |
| OQ-8 | **D.12 canonical hash spec ambiguity** (field order and TLV framing for packed arrays across owners). | Medium | ADR-0005 (golden `state_hash` vectors not yet written) |
| OQ-9 | **Coupled tuning ranges**: some Tuning Knobs ranges permit values that break each other's constraints; boot assertions catch it, but the documented ranges should agree. | Low | GDD tuning passes; ADR-0007 boot-invariant policy |
| OQ-10 | **Release-export performance is unmeasured.** Every budget is "pending spike" on RM-DESKTOP and RM-DECK; GDScript hot-path cost may force a GDExtension fallback for the Graph (rung 3, VS-A only). | High | Spike S1; ADR-0010 section 9 records the GDExtension escape hatch policy (Proposed) |
| OQ-11 | **Replication shape for co-op** (MultiplayerSynchronizer proxies vs custom change records; peer-ID behaviour on 4.7.2 not sourceable). | Low (VS-B) | ADR-0032 |
| OQ-12 | **What guests keep from a host-owned world** (game concept open question) — the PROFILE persist scope exists for it, but its contents are undecided. | Low (Alpha) | Host-Owned World & Guest Carry-over GDD |
| OQ-13 | **Kernel access from modules and presentation: injection vs autoload.** Proposed in ADR-0001: injection of narrow facets, no autoload; the examples in this document now use the injected facets (`SurfaceHandle`, `BlockRegistrar`, `NotifyFacet`, `PresentationFacet`, `PresentationClock`) instead of a `Kernel` global. | High | ADR-0001 (Proposed) |
| OQ-14 | **Snapshot hitch vs frame budget.** A 32-block snapshot may take up to 16.7 ms (TR-cmd-032) inside one synchronous boundary service, above the 14.9 ms desktop frame target; only the encode hand-off is capped (`SAVE_ENCODE_BUDGET_MS` 4 ms). Either split capture across frames (which the kernel's atomic-SAVING rule forbids today) or accept and document one over-budget frame per autosave. | Medium | ADR-0006 with the kernel owner; measured in the save worker spike |
| OQ-15 | **No-rollback weld.** A drift between a validate check and the four apply steps faults the whole kernel instead of rejecting one command. Mitigation chosen for v2: the weld property test and the verifier after every weld in test builds; the alternative (restore the four touched blocks on a fault) is a technical-director decision. | Medium | ADR-0021 |
| OQ-16 | **Unverifiable acceptance wording.** "Zero heap allocations" (TR-lsg-025, TR-cmd-032) cannot be measured directly in GDScript, and some "lint-enforced" rules (field-writer restrictions, writes outside a WriteScope, no floats on a query path) need an AST pass or runtime counters, not a regex. Restate allocation ACs as a flat `Performance.OBJECT_COUNT` over N ticks plus a static scan of hot files, and say per rule how it is enforced. | Medium | ADR-0010 section 7 restates the checks as four tiers (Proposed); GDD AC wording passes remain |

### Cross-GDD inconsistencies found while building the baseline

These came out of mapping the 794 requirements to ADRs. They are stated as the extracted requirements state them; check the GDD text before fixing. Each is small, and none changes a layer or an ownership decision above. The owner (rule G1) is named where one is clear.

| ID | Inconsistency | Owner / resolution |
|---|---|---|
| IC-1 | Interaction caps its subjects at 144 (`INTERACT_FIXTURE_BUDGET` 120 + `PLAYER_DROP_RESERVE` 24, TR-interact-023), but the largest loaded set S4 budgets 145 interaction subjects (TR-derelict-019). This is a real numeric conflict: the S4 set will fail its content lint. | Interaction owns the cap; Derelict Authoring trims one subject or the cap is re-derived. **Fix before any Derelict content story**, not left to ADR-0020. |
| IC-2 | The kernel has seven states (TR-cmd-029); the Debug Overlay says "all six" (TR-debug-010). | Kernel. Debug Overlay GDD fix. |
| IC-3 | The Player Controller counts ten `debug_overlay_*` actions (TR-pc-030); the Debug Overlay lists nine (TR-debug-026). The dev and playtest feature-tag names differ between Debug Overlay and Sanity (TR-debug-004, TR-sanity-027). | Debug Overlay owns the actions and tags. ADR-0031. |
| IC-4 | `STEP_BUDGET_MS` (4 ms) is an alarm threshold in the kernel (P3 target 2.0 ms, TR-cmd-032) but a target in Derelict Authoring and Web Map (TR-derelict-021, TR-webmap-025). | Kernel (owner). Two GDD wording fixes; ADR-0003 records the semantics. |
| IC-5 | `restore_epoch` is owned and persisted by Save/Load (TR-cmd-018); Horror describes it as exposed read-only in `kernel.clock` (TR-horror-017). Both hold if `kernel.clock`'s read view carries a non-persisted, non-hashed copy, which is what CR-8 says. | Kernel. ADR-0005 states it once. |
| IC-6 | Draw-call budgets are set per system (Y-sort ~300 fallback trigger, Trail ≤ 38, HUD ≤ 60, Chart Table ≤ 40; Pipeline D_world ≤ 350, D_root ≤ 63 (62 expected, including the D_composite = 1 term), sum ≤ 1000) with no single table. | Pipeline owns the frame budget. ADR-0008 carries one table. |
| IC-7 | Three flash governors (Lighting per light/region/frame mean, HUD arbiter, Chart Table F ≤ 2 under reduce flashing) share one frame-mean limit with no stated arbitration. | Pipeline owns the frame-mean clamp (PR-27). ADR-0028 defines the shared rule. |
| IC-8 | Pause is single-player only (TR-cmd-030); the Chart Table's VS-B OVERLAY mode makes no kernel call; HUD and Audio freeze on kernel PAUSED. Consistent for MVP; the co-op overlay path is unspecified. | ADR-0032 (VS-B). |
| IC-10 | GDDs name source roots `src/sim/graph/`, `src/sim/emitter/`, `src/debug/`, `src/observer/`, `src/data/items/`, but Sanity names `src/sanity/`. This document uses `src/sim/<system>/` for every simulation module. | Resolved by ADR-0010 section 1 (Proposed): `src/sim/sanity/`; Sanity GDD path fix in AC-SN-30, AC-SN-82, AC-SN-130. |
| IC-11 | `transit.step` (divisor 1) is ordered after `graph.propagate.failure` (divisor 6, offset 4), but Web Map declares no `tolerates_stale` for the edge; same-tick data is impossible across those divisors, so it boots with a warning. | Web Map owner declares the edge stale-tolerant (or justifies otherwise). GDD fix. |
| IC-12 | The Emitter (simulation) reads the Failure & Trail glow table, owned by a presentation module. | The table moves to Content data (ADR-0007); Failure & Trail keeps authoring it. GDD wording fix in both. |
| IC-9 | The hook order is stated piecemeal across Controller, Suit, Body, Emitter, Horror and Sanity. The table in § Data Flow is the merged view; boot-time topo-sort is the check. | Each owner for its row; ADR-0003. |
