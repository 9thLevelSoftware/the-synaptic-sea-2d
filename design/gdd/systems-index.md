# Systems Index: The Synaptic Sea

> **Status**: Draft
> **Created**: 2026-10-25
> **Last Updated**: 2026-10-25
> **Source Concept**: design/gdd/game-concept.md

---

## Overview

The Synaptic Sea needs 42 systems. Its mechanical heart is a simulated life-support graph (O2, power, coolant, fuel, integrity) carried by a composite ship that the player grows by welding salvaged sections onto it; around that sit the two rhythms of the core loop (the deliberate boarding rhythm and the frantic breach rhythm), joined by an Attention system the web uses to watch the players, and a Sanity system that rewrites what the players perceive rather than how they perform. The game pillars decide the shape of the list: The World Never Lies puts the graph, trails and lighting in the foundation; Dread Before Chaos and The Web Watches make Attention and the audio tells MVP systems; Sanity Rewrites Reality isolates the Perception Filter from the simulation so a hallucination can never corrupt state; Deeper Is Richer, Loss Is Not Total adds depth rings, depletion and the failure contract at the Vertical Slice.

The tiers follow the producer-adjusted scope in the concept. **MVP** is the single-player loop test (two handmade derelicts, one weld, one Attention threshold, one horror behaviour, one hallucination class, save v1, plus a kill/continue playtest). **Vertical Slice** is split in the concept into VS-A (single-player, procedural derelicts, the shippable fallback) and VS-B (2-player LAN co-op, built only after VS-A is complete); systems marked **(VS-B)** carry the Vertical Slice tier here but must not start before VS-A. **Alpha** adds the depth rings' full content, Steam lobbies, host-owned world carry-over and accessibility settings. The template's Priority Tiers are fixed, so 'Spike phase' (weld, graph, save, lighting budget) is not a tier: it is a prototype pass over four MVP systems (Modular Ship Assembly, Life-Support Graph, Save/Load, 2D Lighting) and is listed under High-Risk Systems.

**System-by-system rationale.** Fixed-column tables cannot carry prose, so each system's scope and the reason for its tier follow here in enumeration order.

- **1. Command & Simulation Layer** (Core, MVP, inferred). Thin kernel only: fixed tick, command dispatch with authority check, a registry of state blocks owned by domain systems, change notification to presentation, and peer/authority identity on every command from day one (single-player is peer 1). Owns no domain state and defines no other system's commands. ADR-first (authority model), with a short technical spec rather than a full player-facing GDD. *Why this tier:* TD-FEASIBILITY B2: authority discipline from day one is what keeps co-op an addition rather than a rewrite. No networking in MVP; this is only the command/state separation.
- **2. Life-Support Graph** (Core, MVP, explicit). Pure-data model of O2, power, coolant, fuel and integrity loops across welded sections: node and port types, edges (seams are edges whose properties Assembly sets at weld time), flows, failure state and propagation, and per-compartment atmosphere (temperature, pressure, O2 — the P1 consequences). Separate from the scene tree so it can be serialized, tested headless and synced by summary. Stays at five loops. *Why this tier:* The core of Pillar 1; MVP hypothesis cannot be tested without it. Everything that fails, glows, leaks or is salvaged reads or writes it.
- **3. Item & Module Database** (Economy, MVP, inferred). Definitions of salvageable modules (scrubber, reactor cell, hull plate, coolant unit, data core), tools, consumables and their dialect and weights; references graph port-type IDs that the Life-Support Graph GDD defines. *Why this tier:* Salvage and welding have nothing to act on without definitions; also feeds save format.
- **4. Pixel-Perfect Render Pipeline & Camera** (Core, MVP, inferred). 640x360 SubViewport world at integer scale, HUD in the root viewport, nearest filtering, snapping, Y-sort layer shape, camera. Art bible 8.1/8.4. *Why this tier:* Every visual system sits on it; the lighting spike (8.12) starts here.
- **5. Emitter State & Light-Exposure Query** (Core, MVP, inferred). Simulation-side record of every light emitter (fixtures, beams on/off and cone shape, machinery-node lanterns, deployables, power draw), written through commands, plus a geometric light-exposure query (cone test and occluder raycasts on physics shapes) that Attention, Horror Entities and Interaction use. Runs headless; never reads Light2D. *Why this tier:* TD-SYSTEM-BOUNDARY: keeps light a simulation input no hallucination can touch, and lets Attention run headless and on the host for remote players.
- **6. 2D Lighting & Occlusion** (Core, MVP, explicit). Rendering only: reads emitter state and draws Light2D, occluders, ambient floors, the material-level hull floor, the six shadow-casting slots, emission glow and light textures. Owns the final Light2D values through a modifier contract: Attention flicker, horror light disturbance and Perception Filter decoys submit layered modifiers; the Filter may add decoy visuals but never changes the real reveal radius (a pillar anchor). *Why this tier:* Identity-defining and the art bible's first spike: if lighting does not fit the frame budget the whole direction needs a trade-off.
- **7. Player Controller & Input** (Core, MVP, inferred). Movement, 5-facing sprite with mirrored facings, per-facing chest point for the beam (user decision 2026-10-26: chest-mounted lamp), gamepad/keyboard input mapping, stiff-weighted animation states. *Why this tier:* Nothing is playable without a body to light.
- **8. Interaction System** (Gameplay, MVP, inferred). Inspect, cut, weld, open, carry, run a field check; the act-only-on-lit rule uses the light-exposure query (prompts drawn only for lit objects is the UI side, 7.0); hold-to-confirm on destructive actions; proximity focus for gamepad. Falsification checks reach Sanity as commands from here. Also owns the friendly-harm act: tool use against a crewmate is an Interaction command accepted only when Sanity's gate reports the actor below threshold (no dedicated damage or combat system — Anti-Pillar), with no reward path. *Why this tier:* The verb layer for boarding; encodes the 'spend light to act' rule.
- **9. Failure & Trail System** (Gameplay, MVP, explicit). Pure reader: maps graph failure state and footprint sources to physical trail visuals (frost, mist, scorch, drip, hairlines) in the three sizes, writing only CanvasItem/Material/particles. Owns no propagation and no atmosphere — both are Graph state. Tagged as a truth channel: immune to the Perception Filter. *Why this tier:* Pillar 1 made visible; the honest channel every hallucination is checked against.
- **10. Suit & Equipment** (Gameplay, MVP, inferred). Suit vitals (O2, integrity, local temperature and pressure read from the Graph's compartment atmosphere) as a PER_PLAYER block keyed by PlayerId, loadout and tool slots, the chest-mounted lamp (hands-free; the four beam shapes) and deployables — including throwable glowsticks (user decision 2026-10-26) — as emitters, carried modules. Each tool's effect is defined by the system it acts on (cutter/welder: Interaction and Assembly; scanner: Sanity falsification; suppressant: Attention/Horror). *Why this tier:* Personal survival state and the beam are the player's two instruments; the suit layer HUD reads from here.
- **11. Ship Section & Layout Data Model** (Core, MVP, inferred). The runtime section schema shared by authoring, procgen, assembly and save: tile layer reference, collision shapes, sockets, seams, graph-node references, salvage/module slots, and the pure-data ship layout (sections, transforms, welds) that is projected onto the scene tree. *Why this tier:* TD-SYSTEM-BOUNDARY: without a single owner three GDDs would each define a socket; needed before the weld spike.
- **12. Ship Body & Interior Physics** (Core, MVP, inferred). The composite ship as one physics body with a ship-local reference frame; characters and props moving inside a moving body (the moving-platform problem); detach spawns a new body. *Why this tier:* TD-SYSTEM-BOUNDARY: the spike is weld-on-a-moving-ship and nothing owned the motion; needed before the weld spike.
- **13. Modular Ship Assembly** (Gameplay, MVP, explicit). Weld and cut sections as atomic one-tick commands across graph join and layout change: join validation is a pure Graph query; seams set edge properties; cap plates; detach into a new body. Also owns module install/uninstall (cut a scrubber free, install it at home) as a graph mutation distinct from section welding. *Why this tier:* The game's hook and its riskiest build task (producer: the weld is the hardest MVP item); first spike.
- **14. Derelict Authoring (Handmade Kits)** (Gameplay, MVP, explicit). Hand-built Foundry-dialect derelict and spawn ship from the room kit (6.3): tagged sockets, graph nodes, salvage slots, one visible unknown per wreck. *Why this tier:* MVP uses two handmade derelicts; the kit definition is also what procgen later assembles.
- **15. Boarding & Salvage** (Gameplay, MVP, explicit). The boarding rhythm: enter, listen, light, decide, cut a module free, carry it home under rising Attention; field checks for suspect salvage. *Why this tier:* The 5-minute loop. Without it there is no sortie.
- **16. Web Map & Transit** (Gameplay, MVP, explicit). The web world model (derelict list, seeds plus deltas, what is loaded) and the abstracted pod hop between home-ship and target (state 2.5), docking, fuel cost. Full flight is out of scope. Depth Rings extends it at VS; the Chart Table only displays it. *Why this tier:* Connects the two ends of the MVP sortie at minimum cost.
- **17. Attention System** (Gameplay, MVP, explicit). The web's attention. Owns a stimulus-input API (light via the exposure query; noise and power stimuli pushed in by Graph, Suit, Interaction, Assembly and Horrors — never read from Audio), baseline set by depth, and the BREACH STATE MACHINE (calm → tell window → breach → subsiding). Owns the tell-window contract — every breach is preceded by a 10–30 s escalating tell the player could perceive — which Horror Entities, Graph failure injection, Lighting modifiers and Audio must all honour; the contract is a testable acceptance criterion written before Horror and Audio are designed. Its presentation (filaments in the light cone, creaks, flicker as a Lighting modifier) reads attention level plus emitter state. Pillars 2 and 3. *Why this tier:* The hinge between the two rhythms; core hypothesis cannot be tested without it.
- **18. Horror Entities** (Gameplay, MVP, explicit). Segment-rig horrors with creep/unfold/snap behaviours, real occlusion and shadow; writes hidden-entity footprints (branching rime, radial hairlines, light disturbance) into the Graph footprint channel and submits light-disturbance modifiers to Lighting. MVP: one behaviour. *Why this tier:* Gives Attention consequences; MVP minimum is one behaviour.
- **19. Sanity & Hallucination Rules** (Gameplay, MVP, explicit). Per-player sanity, its drift order, the four hallucination classes and their guard rails (inviolable channels, no unfalsifiable lie, system-gated friendly harm, verification cost). Authoritative; writes per-player perception state keyed by a stable PlayerId (seat), never by transport peer ID, from MVP; receives falsification checks as commands from Interaction/Suit. No simulation system ever reads perception state — the friendly-harm gate reads the sanity level, not perception. *Why this tier:* Pillar 4 and the hook. MVP: class (b) false system state only.
- **20. Perception Filter** (Gameplay, MVP, explicit). Per-client perceived-value query/decorator service (displayed value of X for local peer P) plus screen-wide effects: stage-1/2/3 drift, the violet fringe, palette leak, Light2D decoy modifiers. Owns no instrument or HUD nodes — Instruments and HUD ask it and draw the result. Defines the truth-channel/corruptible tag contract on presentation nodes (trails, true reveal radius, crewmate beams, character sprites, HUD frame immune; gauges and machine sprites corruptible). Owns the class-(c) victim tell as a truth-channel element: the below-threshold crewmate's violet fringe with mandatory breathing and the P1 proximity cue (art bible 5.5), rendered for every saner viewer and attributed to the web in logs and scars. Reads simulation state, writes only CanvasItem/Material/Light2D. *Why this tier:* Isolating it keeps hallucinations from touching the simulation (TD rule) and gives the debug overlay something to contrast.
- **21. HUD & Suit Layer** (UI, MVP, explicit). Centre-bottom critical strip, bottom-left suit panel, hand rail, warning mark, tri-modal lying-reading signatures, HR-1..15. *Why this tier:* How the player reads their own body; Section 7 of the art bible specifies it.
- **22. In-World Instruments & Inspect View** (UI, MVP, explicit). Owns the gauge, sight-glass and dial nodes as lit sprites and the inspect view (the same instrument magnified); asks the Perception Filter for the displayed value and draws it with the settle/tremor/glide motion signatures. *Why this tier:* Where class (b) lies live; pairs with the Failure & Trail System as the truth check.
- **23. Chart Table (Web Map & Menus)** (UI, MVP, explicit). The never-lying human instrument: web map, route planning, pause panel, menus. MVP: pick a target, pause, save. *Why this tier:* Needed to start a sortie; the web map grows in later tiers.
- **24. Adaptive Audio & Tells** (Audio, MVP, explicit). Distinct running and failing sound per loop read directly from Graph state (so audio tells can lead visuals), horror and Attention cues, the P1 proximity cue. Must state whether audio can lie: if it can, audio corruption joins the Perception Filter's write set under the same rule; the voice channel is never faked either way. Audio is the only non-visual Attention channel. *Why this tier:* Pillar 2 and the accessibility consequence of 'no Attention indicator': sound carries the weight.
- **25. Save/Load & Versioned Format** (Persistence, MVP, explicit). A versioned state-block registration contract: each domain system registers its serializable state block through the Command kernel's registry, and Save/Load serializes and migrates the set (format version from v1; host-owned). Domain systems depend on this contract, not the reverse. Also owns the MVP death stub: on crew death the game reloads the last save (placeholder, replaced by Failure Contract & Re-wake at VS-A) so no MVP GDD improvises its own. *Why this tier:* A no-win-state game that breaks saves on every module change cannot be played long; versioning is cheapest on day one.
- **26. Debug Overlay (True Values)** (Meta, MVP, inferred). Developer-only overlay showing authoritative values beside the hallucinated ones, to tell deliberate corruption from real desync. Never exposed as an accessibility feature (R-7). *Why this tier:* TD-FEASIBILITY: needed from the start; also the QA instrument for the sanity system.
- **27. Procedural Derelict Generation** (Gameplay, Vertical Slice, explicit). Seeded generator assembling hand-authored kit chunks by grammar: continuity, one trail per source, hero cap, clear path (6.3 rules), salvage placement, one visible unknown. Host sends seed plus deltas. *Why this tier:* VS-A: the point where it becomes a repeatable game rather than a prototype; deliberately after the loop is proven.
- **28. Depth Rings & Scarcity** (Progression, Vertical Slice, explicit). Rings of the web: Attention baseline, ambient and iridescence per ring, salvage value, horror roster. Pillar 5. VS: one ring beyond the spawn ring. Ring access in VS-A is gated by ship capability (pod range, life-support margin) read from the Graph and Assembly, so the shipped fallback has an implicit capability ladder; Ship Capability Progression formalises it at Alpha. *Why this tier:* Gives 'deeper is richer' a mechanism.
- **29. Resource Depletion & Wear** (Economy, Vertical Slice, explicit). Scrubbers and plates degrade; outer-ring salvage runs out, so there is no safe place to farm forever. *Why this tier:* The enforcement mechanism for Pillar 5 the creative-director asked for.
- **30. Knowledge Codex & Logs** (Progression, Vertical Slice, explicit). Decoded web rules, horror tells and recovered logs that persist across crew loss; some rules shift with depth so the web never becomes solvable. Knowledge must change play, not just be stored: a decoded tell becomes recognisable earlier, a learned rule is annotated on the Chart Table, and a recovered log names a deeper wreck on the Web Map (the session hook). Acceptance criterion: every codex entry has a stated effect on play. *Why this tier:* The currency of the failure contract and the Discovery aesthetic.
- **31. Failure Contract & Re-wake** (Progression, Vertical Slice, explicit). Losing the ship or a body costs ship and body, never knowledge; the next crew wakes in the cryo bay knowing more (state 2.10). Owns the cryo-bay fiction as a throughline: re-wake presents the mystery (pod count, pod states, what changes each wake, who else is in the pods and why they keep waking), co-authored with narrative-director, with lore fragments fed through the Knowledge Codex. Includes lethal friendly harm's consequences. The GDD's Player Fantasy section opens with the cryo bay, not a respawn rule. *Why this tier:* Pillar 5's contract; prevents the death spiral.
- **32. Environmental Storytelling Placement** (Narrative, Vertical Slice, explicit). The cause/verb/absence three-sentence verdict per wreck, placed by tagged slots; no bodies, only empty suits. *Why this tier:* Delivers lore without text or cutscenes; makes procgen derelicts read as places.
- **33. Co-op Session & Replication (VS-B)** (Core, Vertical Slice, explicit). Listen-server host-authoritative 2-player LAN: spawner/synchronizers, request validation, summary-state sync of the graph. Built only after VS-A is complete. *Why this tier:* VS-B; gated behind the single-player slice per TD-FEASIBILITY.
- **34. Chat, Voice & Pings (VS-B)** (Core, Vertical Slice, explicit). Voice/proximity, text chat and world pings: the human channel, never hallucinated, with a speaker indicator. *Why this tier:* The costly-check channel Pillar 4 relies on; deaf/HoH and no-mic crews need text and pings.
- **35. Crew Strip HUD (VS-B)** (UI, Vertical Slice, explicit). Identity glyph (beam shape plus stripe count), name, voice activity, location, beam state; no sanity, no vitals, no locators. *Why this tier:* Co-op readability without breaking the human-channel rule.
- **36. Onboarding (Cryo Bay)** (Meta, Vertical Slice, explicit). The first ten minutes: one sealed bay, one failing loop, follow the trail, fix it, watch the gauge agree. Teaches the boarding rhythm and the first Attention threshold. *Why this tier:* A slow-start game dies without it; MVP playtesters get a hand-led version.
- **37. Sanity Scars & Character History** (Progression, Alpha, explicit). Permanent suit-repair-style scars in three slots; how earned, how many, whether guests keep them (open question). *Why this tier:* Gives sanity lasting weight; depends on the failure contract and the guest-carry-over decision.
- **38. Ship Capability Progression** (Progression, Alpha, explicit). Life-support margin, range, hull as the ladder players climb as they graft. *Why this tier:* The Achiever ladder; not needed to test the loop.
- **39. Steam Lobby & NAT** (Core, Alpha, explicit). Steam lobbies and NAT traversal so friends connect over the internet. Needs a technology evaluation first. *Why this tier:* A separate milestone from LAN co-op (TD).
- **40. Host-Owned World & Guest Carry-over** (Persistence, Alpha, explicit). Whose save owns the shared ship; what guests keep; copying a save to let a guest host. Open design question for game-designer and creative-director. *Why this tier:* B1 decision made structural: no live host migration.
- **41. Accessibility Settings** (Meta, Alpha, explicit). UI scale, reading-face scale, HUD opacity floor, reduced motion preserving the backbone, bounded low-vision ambient floor, pattern strength, captions policy (R-6). *Why this tier:* Design starts early because several settings change what other systems must expose.
- **42. Settings & Input Rebinding** (Meta, Alpha, inferred). Graphics, audio, controls, key and button rebinding, gamepad completeness enumeration (HR-14). *Why this tier:* Expected on PC; not needed to test the loop.

**Relationship analysis.** The heaviest dependency roots are listed in the data line below; they are the project's bottlenecks. Command & Simulation Layer sits under almost everything because TD-FEASIBILITY made 'every mutation is an authority-checked request' the rule that keeps co-op additive. The Life-Support Graph is the data every visible system reads. The Item & Module Database is a quiet bottleneck: salvage, welding, save format, depletion and depth rings all need its definitions, so it should be designed early despite looking like content. Lighting is both a Foundation-adjacent technical risk and the carrier of three pillars. The Perception Filter is deliberately a query service with many readers and no simulation writers: it is the isolation that makes Sanity safe. The Emitter State & Light-Exposure Query exists for the mirror-image reason: light must stay a simulation input the Filter cannot touch. UI and Audio systems depend on gameplay systems, never the reverse, so they can be designed after the rules they display.

**Relationship analysis (data).** The four heaviest dependency roots are Command & Simulation Layer (16 dependents), Life-Support Graph (14 dependents), Pixel-Perfect Render Pipeline & Camera (11 dependents), Item & Module Database (9 dependents).

> **Technical Director Review (TD-SYSTEM-BOUNDARY)**: CONCERNS, revisions applied 2026-10-25 — emitter state split from light rendering; Ship Section & Layout Data Model and Ship Body & Interior Physics added; Save/Load inverted into a state-block contract; Failure & Trail made a pure reader; Item DB sequenced after the Graph; Web Map & Transit owns the world model. GDD-level notes are carried in each system's rationale above.

---

## Systems Enumeration

| # | System Name | Category | Priority | Status | Design Doc | Depends On |
|---|-------------|----------|----------|--------|------------|------------|
| 1 | Command & Simulation Layer (inferred) | Core | MVP | Needs Revision | design/gdd/command-simulation-layer.md | — |
| 2 | Life-Support Graph | Core | MVP | Needs Revision | design/gdd/life-support-graph.md | Command & Simulation Layer |
| 3 | Item & Module Database (inferred) | Economy | MVP | Needs Revision | design/gdd/item-and-module-database.md | Life-Support Graph |
| 4 | Pixel-Perfect Render Pipeline & Camera (inferred) | Core | MVP | Needs Revision | design/gdd/pixel-perfect-render-pipeline-camera.md | — |
| 5 | Emitter State & Light-Exposure Query (inferred) | Core | MVP | Needs Revision | design/gdd/emitter-state-and-light-exposure-query.md | Command & Simulation Layer, Life-Support Graph, Player Controller & Input |
| 6 | 2D Lighting & Occlusion | Core | MVP | Needs Revision | design/gdd/2d-lighting-and-occlusion.md | Pixel-Perfect Render Pipeline & Camera, Emitter State & Light-Exposure Query |
| 7 | Player Controller & Input (inferred) | Core | MVP | Needs Revision | design/gdd/player-controller-input.md | Command & Simulation Layer, Pixel-Perfect Render Pipeline & Camera |
| 8 | Interaction System (inferred) | Gameplay | MVP | Needs Revision | design/gdd/interaction-system.md | Command & Simulation Layer, Player Controller & Input, Emitter State & Light-Exposure Query, Pixel-Perfect Render Pipeline & Camera |
| 9 | Failure & Trail System | Gameplay | MVP | Needs Revision | design/gdd/failure-and-trail-system.md | Life-Support Graph, Pixel-Perfect Render Pipeline & Camera |
| 10 | Suit & Equipment (inferred) | Gameplay | MVP | Needs Revision | design/gdd/suit-and-equipment.md | Command & Simulation Layer, Life-Support Graph, Item & Module Database, Player Controller & Input, Emitter State & Light-Exposure Query, Save/Load & Versioned Format |
| 11 | Ship Section & Layout Data Model (inferred) | Core | MVP | Approved | design/gdd/ship-section-and-layout-data-model.md | Command & Simulation Layer, Life-Support Graph, Item & Module Database |
| 12 | Ship Body & Interior Physics (inferred) | Core | MVP | Needs Revision | design/gdd/ship-body-and-interior-physics.md | Command & Simulation Layer, Ship Section & Layout Data Model, Player Controller & Input, Pixel-Perfect Render Pipeline & Camera |
| 13 | Modular Ship Assembly | Gameplay | MVP | Needs Revision | design/gdd/modular-ship-assembly.md | Command & Simulation Layer, Life-Support Graph, Item & Module Database, Interaction System, Ship Section & Layout Data Model, Ship Body & Interior Physics, Save/Load & Versioned Format |
| 14 | Derelict Authoring (Handmade Kits) | Gameplay | MVP | Needs Revision | design/gdd/derelict-authoring-handmade-kits.md | Ship Section & Layout Data Model, Item & Module Database, Emitter State & Light-Exposure Query |
| 15 | Boarding & Salvage | Gameplay | MVP | Needs Revision | design/gdd/boarding-and-salvage.md | Interaction System, Suit & Equipment, Modular Ship Assembly, Derelict Authoring (Handmade Kits), Item & Module Database |
| 16 | Web Map & Transit | Gameplay | MVP | Needs Revision | design/gdd/web-map-and-transit.md | Command & Simulation Layer, Derelict Authoring (Handmade Kits), Save/Load & Versioned Format |
| 17 | Attention System | Gameplay | MVP | Needs Revision | design/gdd/attention-system.md | Command & Simulation Layer, Life-Support Graph, Emitter State & Light-Exposure Query, Suit & Equipment, Player Controller & Input, Save/Load & Versioned Format |
| 18 | Horror Entities | Gameplay | MVP | Needs Revision | design/gdd/horror-entities.md | Command & Simulation Layer, Life-Support Graph, Emitter State & Light-Exposure Query, Attention System |
| 19 | Sanity & Hallucination Rules | Gameplay | MVP | Approved | design/gdd/sanity-and-hallucination-rules.md | Command & Simulation Layer, Attention System, Interaction System, Suit & Equipment, Save/Load & Versioned Format |
| 20 | Perception Filter | Gameplay | MVP | Needs Revision | design/gdd/perception-filter.md | Command & Simulation Layer, Pixel-Perfect Render Pipeline & Camera, 2D Lighting & Occlusion, Sanity & Hallucination Rules |
| 21 | HUD & Suit Layer | UI | MVP | Needs Revision | design/gdd/hud-and-suit-layer.md | Pixel-Perfect Render Pipeline & Camera, Suit & Equipment, Perception Filter, Player Controller & Input |
| 22 | In-World Instruments & Inspect View | UI | MVP | Needs Revision | design/gdd/in-world-instruments-and-inspect-view.md | Life-Support Graph, 2D Lighting & Occlusion, Perception Filter, Interaction System |
| 23 | Chart Table (Web Map & Menus) | UI | MVP | Needs Revision | design/gdd/chart-table-web-map-and-menus.md | Web Map & Transit, Pixel-Perfect Render Pipeline & Camera |
| 24 | Adaptive Audio & Tells | Audio | MVP | Needs Revision | design/gdd/adaptive-audio-and-tells.md | Life-Support Graph, Horror Entities, Attention System, Sanity & Hallucination Rules |
| 25 | Save/Load & Versioned Format | Persistence | MVP | Needs Revision | design/gdd/save-load-and-versioned-format.md | Command & Simulation Layer |
| 26 | Debug Overlay (True Values) (inferred) | Meta | MVP | Needs Revision | design/gdd/debug-overlay-true-values.md | Command & Simulation Layer, Perception Filter, Pixel-Perfect Render Pipeline & Camera |
| 27 | Procedural Derelict Generation | Gameplay | Vertical Slice | Not Started | — | Derelict Authoring (Handmade Kits), Ship Section & Layout Data Model, Item & Module Database, Web Map & Transit |
| 28 | Depth Rings & Scarcity | Progression | Vertical Slice | Not Started | — | Attention System, Item & Module Database, Procedural Derelict Generation, Web Map & Transit, Modular Ship Assembly |
| 29 | Resource Depletion & Wear | Economy | Vertical Slice | Not Started | — | Life-Support Graph, Item & Module Database, Depth Rings & Scarcity |
| 30 | Knowledge Codex & Logs | Progression | Vertical Slice | Not Started | — | Save/Load & Versioned Format, Item & Module Database, Attention System, Horror Entities, Depth Rings & Scarcity, Web Map & Transit |
| 31 | Failure Contract & Re-wake | Progression | Vertical Slice | Not Started | — | Save/Load & Versioned Format, Knowledge Codex & Logs, Life-Support Graph, Suit & Equipment, Sanity & Hallucination Rules |
| 32 | Environmental Storytelling Placement | Narrative | Vertical Slice | Not Started | — | Procedural Derelict Generation, Knowledge Codex & Logs |
| 33 | Co-op Session & Replication (VS-B) | Core | Vertical Slice | Not Started | — | Command & Simulation Layer, Save/Load & Versioned Format, Life-Support Graph, Player Controller & Input |
| 34 | Chat, Voice & Pings (VS-B) | Core | Vertical Slice | Not Started | — | Co-op Session & Replication (VS-B) |
| 35 | Crew Strip HUD (VS-B) | UI | Vertical Slice | Not Started | — | Co-op Session & Replication (VS-B), HUD & Suit Layer, Chat, Voice & Pings (VS-B) |
| 36 | Onboarding (Cryo Bay) | Meta | Vertical Slice | Not Started | — | Life-Support Graph, Interaction System, Failure & Trail System, HUD & Suit Layer |
| 37 | Sanity Scars & Character History | Progression | Alpha | Not Started | — | Sanity & Hallucination Rules, Failure Contract & Re-wake |
| 38 | Ship Capability Progression | Progression | Alpha | Not Started | — | Modular Ship Assembly, Depth Rings & Scarcity |
| 39 | Steam Lobby & NAT | Core | Alpha | Not Started | — | Co-op Session & Replication (VS-B) |
| 40 | Host-Owned World & Guest Carry-over | Persistence | Alpha | Not Started | — | Save/Load & Versioned Format, Co-op Session & Replication (VS-B), Sanity Scars & Character History |
| 41 | Accessibility Settings | Meta | Alpha | Not Started | — | HUD & Suit Layer, In-World Instruments & Inspect View, Adaptive Audio & Tells, Perception Filter, Pixel-Perfect Render Pipeline & Camera |
| 42 | Settings & Input Rebinding (inferred) | Meta | Alpha | Not Started | — | Player Controller & Input, Chart Table (Web Map & Menus), Pixel-Perfect Render Pipeline & Camera |

---

## Categories

| Category | Description | Typical Systems |
|----------|-------------|-----------------|
| **Core** | Foundation systems everything depends on | Command layer, render pipeline, lighting, controller, co-op session |
| **Gameplay** | The systems that make the game fun | Boarding, welding, Attention, horrors, sanity, procgen |
| **Progression** | How the player grows over time | Depth rings, knowledge codex, failure contract, scars |
| **Economy** | Resource creation and consumption | Module database, depletion and wear |
| **Persistence** | Save state and continuity | Save/load, host-owned world |
| **UI** | Player-facing information displays | Suit HUD, in-world instruments, chart table, crew strip |
| **Audio** | Sound systems | Loop sounds, audio-first tells |
| **Narrative** | Story delivery | Environmental storytelling placement |
| **Meta** | Systems outside the core loop | Debug overlay, onboarding, accessibility, settings |

---

## Priority Tiers

| Tier | Definition | Target Milestone | Design Urgency |
|------|------------|------------------|----------------|
| **MVP** | Required for the core loop to function. Without these, you can't test "is this fun?" | First playable prototype (single-player; follows the 4-8 week spike phase) | Design FIRST |
| **Vertical Slice** | Required for one complete, polished area. VS-A single-player; VS-B co-op only after VS-A | Vertical slice / demo | Design SECOND |
| **Alpha** | All features present in rough form. Complete mechanical scope, placeholder content OK. | Alpha milestone | Design THIRD |
| **Full Vision** | Polish, edge cases, nice-to-haves, and content-complete features. | Beta / Release | Design as needed |

> **Producer Review (PR-SCOPE)**: OPTIMISTIC 2026-10-25 — adjustments declined by the user. The producer estimated the 26 MVP systems as full GDDs at ~64–81 design sessions (7 L, 17 M, 2 S) plus ~35–55 build weeks, against a 6–9 month full-time MVP, and recommended: (1) thin technical specs/ADRs instead of full GDDs for the eight infrastructure systems (Command Layer, Render Pipeline, Emitter State, Section Data Model, Ship Body, Save contract, Item DB, Debug Overlay); (2) MVP-slice GDDs for Horror, Sanity and Attention with VS-A addenda; (3) folding Chart Table into Web Map & Transit for MVP; (4) a thin Settings slice in VS-A so the fallback is releasable. All four were declined; the full GDD pass stands. Consequence recorded: plan on the 9-month full-time / 16-month part-time end with 12 months full-time as the pessimistic case, and re-estimate after the spike phase. Producer's advice retained: design the four spike systems and their prerequisites (Command Layer, Section Data Model, Ship Body, Graph, Save, Lighting, Assembly) first, run the 4–8 week spike, then design the rest.
> **Creative Director Review (CD-SYSTEMS)**: CONCERNS, revisions applied 2026-10-25 — (MVP) Attention now owns the breach state machine and the tell-window contract; (MVP) Save/Load owns the MVP death stub (reload last save) until the Failure Contract lands; (Vertical Slice) Failure Contract & Re-wake owns the cryo-pod mystery throughline with narrative-director; (Vertical Slice) Knowledge Codex gains Attention/Horror/Depth Rings/Web Map dependencies and an actionable-knowledge acceptance criterion; (Vertical Slice) Depth Rings gates ring access by ship capability so VS-A has an implicit ladder; (Vertical Slice, VS-B) Interaction owns the friendly-harm act under Sanity's gate and the Perception Filter owns the victim tell. Verdict otherwise: every system traces to a pillar, none breaks an anti-pillar, MVP is the right test of the core fantasy.
> **Design note (command-simulation-layer GDD, 2026-10-25)**: the kernel GDD found four dependents missing their kernel edge (Interaction, Attention, Ship Section & Layout Data Model, Perception Filter — added), one redundant edge (Life-Support Graph → Save/Load, satisfied by the registry contract — removed), and corrected per-player keying from peer ID to a stable PlayerId (Sanity, Suit). Design order is unchanged.
> **Design note (pixel-perfect-render-pipeline-camera GDD, 2026-10-26)**: the pipeline GDD found four dependents missing their edge (Ship Body hard; Interaction, Debug Overlay, Settings & Input Rebinding and Accessibility Settings soft — added). It also routes: an art-bible 8.2 projection amendment (faces occlude screen-north), camera framing as a never-shift anchor (Perception Filter), Player Controller speeds in whole base px per tick, and a UX flag for the settings screen and HUD layer (`/ux-design` before epics). Design order unchanged.
> **Design note (player-controller-input GDD, 2026-10-26)**: three dependents gained their controller edge (Attention — noise stimuli, soft; Emitter State — beam toggle/aim; HUD — hand rail/prompts, soft). User decisions recorded: chest-mounted hands-free lamp (beam origin = chest point; art bible 3.2/5.1/5.5/5.6/8.3 amendment routed), throwable glowsticks under Suit & Equipment. Routed corrections: render pipeline speeds 120/180 and the inner-box inequality; kernel CR-4 linear bar, facing is derived not an intent. UX flag: prompts/channel bar, hand rail, rebinding (`/ux-design` before epics). Design order unchanged.
> **Design note (life-support-graph GDD, 2026-10-26)**: one `graph.state` block and three chained hooks at divisor 6 (`graph.propagate.power` → `.atmo` → `.failure`); five loops with atmosphere per compartment; fairness gate `W_MIN_WARNING_S` 10 s on endogenous losses; seam sub-edges start disconnected (welding moves nothing). Three dependents lack the Graph edge in this index — Interaction (hard: verbs call Graph mutators), Perception Filter (hard: wraps `true_value`), Sanity & Hallucination Rules (soft: falsification table) — add at the next index pass; the Save/Load edge points the other way (kernel registry contract). Kernel row 1, CR-3 and Suit-row hook ids fixed in place at the consistency check (one block, three chained hooks `graph.propagate.power/.atmo/.failure`, confirms MAX_DIRTY_KEYS); still routed: D.10 counter clarification, K-1 per-block hash amendment (ADR); Save/Load S-1 binary interchange for 64-bit values; Layout ownership clause (identity/volume vs atmosphere/opening/valve/breaker); Suit reads band constants from the registry; Item DB references port-type IDs; Perception/Sanity/F&T never suppress a true descriptor channel. USER questions: MVP death = restart from autosave (recommended); time-band tuning at playtest.
> **Ordering rule**: VS-B (co-op) systems are listed after every VS-A system and must not start before VS-A is complete; VS-A is ordered so that a stop after Onboarding (Cryo Bay) still leaves a shippable single-player game.

---

## Dependency Map

### Foundation Layer (no dependencies)

1. Command & Simulation Layer — depends on: none **[bottleneck: 16 dependents]**
2. Pixel-Perfect Render Pipeline & Camera — depends on: none **[bottleneck: 11 dependents]**

### Core Layer (depends on foundation)

1. Life-Support Graph — depends on: Command & Simulation Layer **[bottleneck: 14 dependents]**
2. Player Controller & Input — depends on: Command & Simulation Layer, Pixel-Perfect Render Pipeline & Camera **[bottleneck: 8 dependents]**
3. Save/Load & Versioned Format — depends on: Command & Simulation Layer **[bottleneck: 9 dependents]**

### Feature Layer (depends on core)

1. Item & Module Database — depends on: Life-Support Graph **[bottleneck: 9 dependents]**
2. Emitter State & Light-Exposure Query — depends on: Command & Simulation Layer, Life-Support Graph, Player Controller & Input **[bottleneck: 6 dependents]**
3. 2D Lighting & Occlusion — depends on: Pixel-Perfect Render Pipeline & Camera, Emitter State & Light-Exposure Query
4. Interaction System — depends on: Command & Simulation Layer, Player Controller & Input, Emitter State & Light-Exposure Query, Pixel-Perfect Render Pipeline & Camera **[bottleneck: 5 dependents]**
5. Failure & Trail System — depends on: Life-Support Graph, Pixel-Perfect Render Pipeline & Camera
6. Suit & Equipment — depends on: Command & Simulation Layer, Life-Support Graph, Item & Module Database, Player Controller & Input, Emitter State & Light-Exposure Query, Save/Load & Versioned Format **[bottleneck: 5 dependents]**
7. Ship Section & Layout Data Model — depends on: Command & Simulation Layer, Life-Support Graph, Item & Module Database
8. Ship Body & Interior Physics — depends on: Command & Simulation Layer, Ship Section & Layout Data Model, Player Controller & Input, Pixel-Perfect Render Pipeline & Camera
9. Modular Ship Assembly — depends on: Command & Simulation Layer, Life-Support Graph, Item & Module Database, Interaction System, Ship Section & Layout Data Model, Ship Body & Interior Physics, Save/Load & Versioned Format
10. Derelict Authoring (Handmade Kits) — depends on: Ship Section & Layout Data Model, Item & Module Database, Emitter State & Light-Exposure Query
11. Boarding & Salvage — depends on: Interaction System, Suit & Equipment, Modular Ship Assembly, Derelict Authoring (Handmade Kits), Item & Module Database
12. Web Map & Transit — depends on: Command & Simulation Layer, Derelict Authoring (Handmade Kits), Save/Load & Versioned Format
13. Attention System — depends on: Command & Simulation Layer, Life-Support Graph, Emitter State & Light-Exposure Query, Suit & Equipment, Player Controller & Input, Save/Load & Versioned Format **[bottleneck: 5 dependents]**
14. Horror Entities — depends on: Command & Simulation Layer, Life-Support Graph, Emitter State & Light-Exposure Query, Attention System
15. Sanity & Hallucination Rules — depends on: Command & Simulation Layer, Attention System, Interaction System, Suit & Equipment, Save/Load & Versioned Format
16. Perception Filter — depends on: Command & Simulation Layer, Pixel-Perfect Render Pipeline & Camera, 2D Lighting & Occlusion, Sanity & Hallucination Rules
17. Procedural Derelict Generation — depends on: Derelict Authoring (Handmade Kits), Ship Section & Layout Data Model, Item & Module Database, Web Map & Transit
18. Depth Rings & Scarcity — depends on: Attention System, Item & Module Database, Procedural Derelict Generation, Web Map & Transit, Modular Ship Assembly
19. Resource Depletion & Wear — depends on: Life-Support Graph, Item & Module Database, Depth Rings & Scarcity
20. Knowledge Codex & Logs — depends on: Save/Load & Versioned Format, Item & Module Database, Attention System, Horror Entities, Depth Rings & Scarcity, Web Map & Transit
21. Failure Contract & Re-wake — depends on: Save/Load & Versioned Format, Knowledge Codex & Logs, Life-Support Graph, Suit & Equipment, Sanity & Hallucination Rules
22. Environmental Storytelling Placement — depends on: Procedural Derelict Generation, Knowledge Codex & Logs
23. Co-op Session & Replication (VS-B) — depends on: Command & Simulation Layer, Save/Load & Versioned Format, Life-Support Graph, Player Controller & Input
24. Chat, Voice & Pings (VS-B) — depends on: Co-op Session & Replication (VS-B)
25. Sanity Scars & Character History — depends on: Sanity & Hallucination Rules, Failure Contract & Re-wake
26. Ship Capability Progression — depends on: Modular Ship Assembly, Depth Rings & Scarcity
27. Steam Lobby & NAT — depends on: Co-op Session & Replication (VS-B)
28. Host-Owned World & Guest Carry-over — depends on: Save/Load & Versioned Format, Co-op Session & Replication (VS-B), Sanity Scars & Character History

### Presentation Layer (depends on features)

1. HUD & Suit Layer — depends on: Pixel-Perfect Render Pipeline & Camera, Suit & Equipment, Perception Filter, Player Controller & Input
2. In-World Instruments & Inspect View — depends on: Life-Support Graph, 2D Lighting & Occlusion, Perception Filter, Interaction System
3. Chart Table (Web Map & Menus) — depends on: Web Map & Transit, Pixel-Perfect Render Pipeline & Camera
4. Adaptive Audio & Tells — depends on: Life-Support Graph, Horror Entities, Attention System, Sanity & Hallucination Rules
5. Crew Strip HUD (VS-B) — depends on: Co-op Session & Replication (VS-B), HUD & Suit Layer, Chat, Voice & Pings (VS-B)

### Polish Layer (depends on everything)

1. Debug Overlay (True Values) — depends on: Command & Simulation Layer, Perception Filter, Pixel-Perfect Render Pipeline & Camera
2. Onboarding (Cryo Bay) — depends on: Life-Support Graph, Interaction System, Failure & Trail System, HUD & Suit Layer
3. Accessibility Settings — depends on: HUD & Suit Layer, In-World Instruments & Inspect View, Adaptive Audio & Tells, Perception Filter, Pixel-Perfect Render Pipeline & Camera
4. Settings & Input Rebinding — depends on: Player Controller & Input, Chart Table (Web Map & Menus), Pixel-Perfect Render Pipeline & Camera

---

## Recommended Design Order

| Order | System | Priority | Layer | Agent(s) | Est. Effort |
|-------|--------|----------|-------|----------|-------------|
| 1 | Command & Simulation Layer | MVP | Foundation | technical-director, lead-programmer | L |
| 2 | Pixel-Perfect Render Pipeline & Camera | MVP | Foundation | technical-artist, engine-programmer | M |
| 3 | Life-Support Graph | MVP | Core | systems-designer, technical-director | L |
| 4 | Player Controller & Input | MVP | Core | gameplay-programmer, game-designer | M |
| 5 | Save/Load & Versioned Format | MVP | Core | technical-director, lead-programmer | M |
| 6 | Item & Module Database | MVP | Feature | game-designer, economy-designer | M |
| 7 | Emitter State & Light-Exposure Query | MVP | Feature | technical-director, systems-designer | M |
| 8 | Failure & Trail System | MVP | Feature | systems-designer, technical-artist | M |
| 9 | 2D Lighting & Occlusion | MVP | Feature | technical-artist, godot-shader-specialist | L |
| 10 | Interaction System | MVP | Feature | game-designer, ui-programmer | M |
| 11 | Suit & Equipment | MVP | Feature | game-designer | M |
| 12 | Ship Section & Layout Data Model | MVP | Feature | technical-director, systems-designer | M |
| 13 | Ship Body & Interior Physics | MVP | Feature | engine-programmer, technical-director | M |
| 14 | Derelict Authoring (Handmade Kits) | MVP | Feature | level-designer | M |
| 15 | Attention System | MVP | Feature | systems-designer, game-designer | L |
| 16 | Modular Ship Assembly | MVP | Feature | systems-designer, technical-director | L |
| 17 | Web Map & Transit | MVP | Feature | game-designer | M |
| 18 | Horror Entities | MVP | Feature | ai-programmer, game-designer | L |
| 19 | Sanity & Hallucination Rules | MVP | Feature | systems-designer, game-designer | L |
| 20 | Boarding & Salvage | MVP | Feature | game-designer | M |
| 21 | Perception Filter | MVP | Feature | technical-artist, godot-shader-specialist | M |
| 22 | Chart Table (Web Map & Menus) | MVP | Presentation | ux-designer | S |
| 23 | Adaptive Audio & Tells | MVP | Presentation | audio-director, sound-designer | M |
| 24 | HUD & Suit Layer | MVP | Presentation | ux-designer, ui-programmer | M |
| 25 | In-World Instruments & Inspect View | MVP | Presentation | ux-designer, ui-programmer | M |
| 26 | Debug Overlay (True Values) | MVP | Polish | tools-programmer, qa-lead | S |
| 27 | Procedural Derelict Generation | Vertical Slice | Feature | level-designer, technical-director | L |
| 28 | Knowledge Codex & Logs | Vertical Slice | Feature | narrative-director, game-designer | M |
| 29 | Failure Contract & Re-wake | Vertical Slice | Feature | game-designer, narrative-director | M |
| 30 | Onboarding (Cryo Bay) | Vertical Slice | Polish | ux-designer, game-designer | M |
| 31 | Depth Rings & Scarcity | Vertical Slice | Feature | game-designer, economy-designer | M |
| 32 | Resource Depletion & Wear | Vertical Slice | Feature | economy-designer | M |
| 33 | Environmental Storytelling Placement | Vertical Slice | Feature | narrative-director, level-designer | M |
| 34 | Co-op Session & Replication (VS-B) | Vertical Slice | Feature | network-programmer, technical-director | L |
| 35 | Chat, Voice & Pings (VS-B) | Vertical Slice | Feature | network-programmer, ux-designer | M |
| 36 | Crew Strip HUD (VS-B) | Vertical Slice | Presentation | ux-designer | S |
| 37 | Steam Lobby & NAT | Alpha | Feature | network-programmer | L |
| 38 | Settings & Input Rebinding | Alpha | Polish | ux-designer | S |
| 39 | Ship Capability Progression | Alpha | Feature | game-designer, economy-designer | M |
| 40 | Accessibility Settings | Alpha | Polish | accessibility-specialist, ux-designer | M |
| 41 | Sanity Scars & Character History | Alpha | Feature | game-designer | M |
| 42 | Host-Owned World & Guest Carry-over | Alpha | Feature | game-designer, technical-director | M |

---

## Circular Dependencies

- **None found** in the hard dependency graph (checked programmatically at generation: every system resolves to a finite depth).
- *Soft cycle — Attention <-> Lighting.* Light raises Attention, and Attention makes web filaments visible in the light cone. Resolution: one-way data flow. Lighting exposes an emitter registry; Attention reads it and publishes an attention level; the filament renderer (part of Attention's presentation) reads both. Lighting never reads Attention.
- *Soft cycle — Sanity <-> Perception Filter.* Sanity decides what is perceived; the Filter draws it. Resolution: Sanity is authoritative and writes only perception state; the Filter reads that state and writes only CanvasItem/Material/Light2D properties (control-manifest rule from TD-FEASIBILITY). The Filter never calls back.
- *Soft cycle — Life-Support Graph <-> Modular Ship Assembly.* Assembly changes the graph's shape; the graph decides what assembly may join. Resolution: the graph is pure data; Assembly mutates it only through the Command layer.

---

## High-Risk Systems

| System | Risk Type | Risk Description | Mitigation |
|--------|-----------|-----------------|------------|
| Ship Body & Interior Physics | Technical | Characters and props inside a moving composite body (the moving-platform problem) is a known Godot problem area and the base of the weld. | Spike together with the weld; one physics body, ship-local frame. |
| Modular Ship Assembly | Technical | Merging a section into a moving ship's collision, tilemap and graph is the hardest MVP task (PR-SCOPE). | Spike first: weld and detach two sections mid-flight with graph recompute. |
| 2D Lighting & Occlusion | Technical | Cost of six shadow lights, many emissive loops and true occlusion on D3D12 is unmeasured; ambient floors may quantise to black. | Lighting spike (art bible 8.12) before committing to the look; cut emitters before occlusion. |
| Life-Support Graph | Design / Technical | Deep simulation could swallow the dread; must be pure data, serializable and syncable by summary. | Spike graph with spreading failures; keep it small (5 loops); data model ADR. |
| Sanity & Hallucination Rules | Design | Lethal friendly harm and verification cost must feel fair; no unfalsifiable lies. | Playtest signal: friendly-fire deaths told as stories and crews keep playing together; revert to down-only if not. |
| Perception Filter | Technical | Per-client divergence is indistinguishable from desync without tooling. | Presentation-only rule; Debug Overlay (true values) built with it. |
| Procedural Derelict Generation | Design / Scope | Readable procedural interiors take longer than the generator; art volume is the biggest sink. | Kit-based grammar, one kit prototyped in oblique view first, seed plus deltas. |
| Co-op Session & Replication (VS-B) | Technical / Scope | Netcode on a solo first project; NAT is a separate milestone. | Gated behind VS-A; authority discipline from day one; two-local-instance checks; technology evaluation before Steam lobbies. |
| Save/Load & Versioned Format | Technical | Saving a composite ship with no win state breaks on every data change. | Version from v1; serialize data models, not scene trees. |
| Host-Owned World & Guest Carry-over | Design | What guests keep is undecided and the save schema waits on it. | Decide with creative-director before Alpha; MVP and VS-A are unaffected. |

---

## Progress Tracker

| Metric | Count |
|--------|-------|
| Total systems identified | 42 |
| Design docs started | 26 |
| Design docs reviewed | 26 (2026-10-27; Boarding & Salvage: major revision) |
| Design docs approved | 2 |
| MVP systems designed | 26/26 |
| Vertical Slice systems designed | 0/10 |

---

## Next Steps

- [ ] Review and approve this systems enumeration
- [ ] Design MVP-tier systems first (use `/design-system [system-name]`)
- [ ] Run `/design-review` on each completed GDD
- [ ] Run `/gate-check technical-setup` when MVP systems are designed
- [ ] Validate the highest-risk systems with `/vertical-slice` before committing to Production
