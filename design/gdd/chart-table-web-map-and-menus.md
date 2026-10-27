# Chart Table (Web Map & Menus)

> Authoring guidance: .claude/docs/templates/guidance/game-design-document-guide.md (load per-section as you author — do not read entirely).

> **Status**: Designed
> **Author**: user + ux-designer, ui-programmer, art-director, creative-director (via /design-system, autonomous; orchestrated run 2026-10-26)
> **Last Updated**: 2026-10-26
> **Last Verified**: 2026-10-26
> **Index position**: enumeration 23 / design order 22 (`systems-index.md`: Systems Enumeration row 23, Recommended Design Order row 22; the two numberings differ by one and neither is "corrected"). Category UI · Layer Presentation · Priority MVP · Effort S. Specialists: ux-designer (primary), ui-programmer and art-director (supporting); creative-director (Player Fantasy).
> **Implements Pillar**: P1 The World Never Lies (primary — the one layer of the game that is not an instrument and therefore cannot lie: it sits outside the Perception Filter and Sanity, carries no reading, no meter and no cost, and is the human channel the other pillars lean on, ART 7.0 row 3). **Serves**: P5 Deeper Is Richer, Loss Is Not Total (the chart records only what the crew lit and never reveals, rings 2-3 are the horizon and not content, no completion figure or win state exists on any surface); P3 The Web Watches (no Attention surface of any kind under any setting, no per-write autosave mark, filament drift is constant); P4 Sanity Rewrites Reality (the chart and the save UI are the channel of truth that always remains, and nothing a hallucination class can falsify ever enters the layer); P2 Dread Before Chaos (the commitment stays at the pod console, the pause is non-diegetic only and never softens the frozen world).
> **Creative Director Review (CD-GDD-ALIGN)**: CONCERNS → REVISED 2026-10-26 (4 items applied; skipped: No separate edit for the Pillar 5 scope note beyond the one CT-38 clause (the reviewer asked for no change).; design/gdd/systems-index.md and other GDDs were not touched, per file ownership. ASK-CT-10 still needs Web Map & Transit to respond to the neutral-ordinal reading of WM:1044.)

## Summary

The Chart Table is the game's one never-lying meta layer: a flat, unlit screen-space surface, opened by the `chart_table` input (Esc / Start), that holds the web map where the crew picks the next sortie target, the pause panel (Resume, Chart, Display & Comfort, Save & Quit), and the boot-time screens (first-launch photosensitivity notice, main menu with Continue / New campaign / Quit, and the load-failure messages). It exists because three player acts happen outside the fiction (choose where to go, stop the game, leave and come back safely) and because a game about lying instruments needs one surface the player can trust completely. It owns no simulation state, registers no block, hook, command or mutator, and shows nothing that a hallucination, the Web's Attention or the cost of a hop could colour. Read it if you build or review any screen, menu, pick, pause or settings behaviour; skip it for the world-model of the web (Web Map & Transit), the save format (Save/Load), the layer model (Render Pipeline), actions and bindings (Player Controller), prompts at the pod console (Interaction), the in-world gauges (In-World Instruments) and every sound (Adaptive Audio).

> **Quick reference** — Layer: `Presentation` · Priority: `MVP` · Key deps: `Web Map & Transit` (designed), `Pixel-Perfect Render Pipeline & Camera` (designed); also `Command & Simulation Layer`, `Save/Load & Versioned Format`, `Player Controller & Input` (designed; absent from the index row, see Dependencies)

## Overview

**What it is, in one sentence.** The Chart Table is the screen-space, unlit, never-lying **meta layer** (render pipeline layer 3, PR-3, PIPE:94-103; art bible 7.0 row "Chart table (meta)") through which the player chooses a sortie target, stops the game, leaves it safely and comes back to it, and through which the game makes its three one-time promises about comfort and honesty (the first-launch photosensitivity notice, the Display & Comfort page, and the plain ship-log wording of every message).

**The data layer.** As data the Chart Table is small and entirely **host-local**: it owns no simulation state and registers **no state block, no hook, no command, no mutator** (CT-27). It holds four things, none of them in a snapshot. (1) `ChartPick`, the player's pending choice of target (NONE or one `derelict_id`), which is host-local presentation state that is never simulation, never persisted, never a field of `web.state` or `web.view`, and never assumes the game is paused (WM-8, WM:203). (2) `PauseGate`, the single adapter between the layer and the kernel's host-local control plane (`request_pause`, `request_resume`, `request_shutdown`; KRN CR-15): in the solo build opening the layer pauses and closing it resumes, in the reserved co-op build (VS-B) the same layer opens as an overlay with the kernel `RUNNING`. (3) The `SettingsStore`, a per-install store of three comfort values and a "notice seen" flag that nothing in the simulation ever reads (CT-19). (4) The authored chart layout, a content file of node positions per record (CT-12). It **reads** only `web.view` (WM-36, provisional name), `SaveInfo`, `save_status`, `restore_reason` and Save/Load's `campaign_info()` / `list_saves()`; it calls only the kernel control plane and Save/Load's host-local API (SL-21: `new_game`, `continue_campaign`, `save_and_quit`, `retire_campaign`, `delete_campaign`). It is an `INPUT_SURFACE` (KRN:111, PIPE:188, PCI:147) that **submits zero commands at MVP**; the class is kept for VS-B guest pause and save requests. The one simulation write a chart-table decision ever causes, `pod_hop{target_id}`, is made by the **pod console press** (Interaction's CONSOLE subject), not by this layer (CT-10).

**The player-facing effect.** The player meets it as a quiet, flat, Ground-coloured plane that appears when they press Esc or Start from anywhere: a right-third **pause panel** laid over the world, which stays frozen exactly as it was lit (not dimmed, blurred or desaturated; PIPE:265, ART 7.4) with four items in a fixed order (Resume, Chart, Display & Comfort, Save & Quit) and one fact (the playtime of the newest record). "Chart" opens the full-bleed **web map**: a concentric dark-toward-the-centre field with slowly drifting grey filaments, the home-ship as one Tungsten point with a rectilinear silhouette, derelicts as grey silhouettes that carry only what the crew has lit (a single Tungsten tick marks the one thing seen from the pod), and a bracket the player snaps from node to node. Nothing sits beside a node: no distance, no fuel, no time, no name, no recommendation. The player picks a node, closes the layer, walks to the pod console and presses; the press is the commitment (WM:1225). Before the first frame of any session they meet the **boot-flow screens**: the one-time photosensitivity notice (PIPE PR-13(e)), the main menu (Continue with its plain facts, New campaign, Quit), and, when something has gone wrong, a plain ship-log line (SL UI-7, UI-8, UI-9). Saving is a single action, **Save & Quit**, that writes and exits; there is no manual save slot, no thumbnail, no spinner and no "progress saved".

**What the game loses without it.** There would be no way to start a sortie (the pod console press needs a target, WM-8, and a no-pick press needs somewhere to send the player), no Save & Quit (Save/Load's only manual save, SL-6), no Continue and no New campaign (SL-21 has no other caller), no home for the photosensitivity notice the pipeline makes mandatory (PR-13(e)) and no place for the comfort toggles an outside playtester needs before they meet the 2.8 breach sweep. It would also lose the one surface that carries the pillar P4 promise that "at least one channel of truth always remains": every other surface in the game is an instrument, a perception or a world, and each of them can be made to lie.

**Boundaries (what it deliberately is not).** It is not a flight model, not a route builder (WM-4 is a star: HOME to one derelict, or a derelict back to HOME), not a settings editor (the full PR-14 settings UI, rebinding, zoom and HUD scale belong to Settings & Input Rebinding, Alpha; zoom is never a setting, PIPE:232), not an inventory or loadout screen, not a ship schematic (none exists; In-World Instruments rejects one, IW-28), not a minimap (there is none in play, ART 7.0), not a codex, not a chat window and not a death screen. Rows reserved for later systems are listed in CT-38.

**Index wording, reconciled.** The enumeration row reads `Chart Table (Web Map & Menus) | UI | MVP`, its rationale bullet reads "The never-lying human instrument: web map, route planning, pause panel, menus. MVP: pick a target, pause, save" (IDX:40), and the routed thread for this stage reads "pick a target, pause, save/load". These are the same scope: "save" is Save & Quit (this layer's panel action), and "load" is Continue and New campaign (the main menu). "Route planning" is read as **picking one legal target on a star of at most one hop**, because the topology allows no multi-leg route and no waypoint (WM-4; guard rail G1 in Player Fantasy); it never means building, ranking, pricing or timing a route. The index lists this system's dependencies as Web Map & Transit and the Pixel-Perfect Render Pipeline only (IDX:95, IDX:201); the real edges are wider (the kernel's control plane, Save/Load's API, the Player Controller's context, plus soft edges to Suit & Equipment, Interaction and Adaptive Audio, and negative requirements from Attention, Horror, Boarding and Sanity), recorded in Dependencies and routed to `/map-systems` as an index correction (C5). The producer review that proposed folding this system into Web Map & Transit for MVP was declined by the user, so this GDD stands alone and **restates no Web Map value**: it cites `web.view` and WM's rules by name (IDX:143).

**The Open Question this GDD owes the kernel (answered here, verbatim).** KRN Open Question 6 asks the Chart Table to "confirm [it] does not assume 'pick a target' happens while paused, and define VS-B's pause-as-overlay" (KRN:961). The answer, which CT-4 makes a rule and AC-CT-2 (Acceptance Criteria) makes a test: **The Chart Table does not assume "pick a target" happens while paused.** The pick is valid with the kernel `RUNNING` or `PAUSED`; no chart, pick, panel or menu rule reads `kernel.state == PAUSED`; `PauseGate` is the single place the solo build and the co-op overlay differ.

**Glossary (resolves conflict C4 and the same-name risk).** *The chart table* in this GDD means the meta layer above; it is not a prop and has no place in the world. *The chart* is the full-bleed web-map screen drawn from `web.view`. *The panel* is the right-third pause panel. *The pick* (`ChartPick`) is the host-local pending target. *The pod console* is Interaction's CONSOLE subject anchored on the docked-podcraft prop in the dock room (WM-8): the only trigger of `pod_hop`, `berth` and `unberth`; it is not a chart widget and there is no chart-side confirm. The *chart-table register plate* is the opaque Ground plate on which Interaction draws the console prompt and refusal words (WM:1029); this GDD shares its grammar and owns none of its content. *Boot-flow screens* are the notice, the main menu, UI-5, UI-8 and UI-9, shown while the kernel is not `RUNNING`; they never request a pause. ART 7.0 row 1's lit "chart-table console" and ART 7.4's working proposal of an in-world ship-schematic console are rejected by In-World Instruments (IW-28) and do not exist (C4).

**MVP slice.** Ships at MVP: the pause panel (S1, S2), the spatial and compact chart with the pick and view-only mode over the WM roster of HOME plus derelicts A and B (S3-S5), Display & Comfort with exactly three controls (S6), Save & Quit with `UI3_PLATE` and the hold bar (S7, S8), the first-launch notice, the main menu, UI-5, UI-7/8/9 and the FAULTED return (S9-S15), `PauseGate` SOLO only. Later: OVERLAY and crew glyphs and chat (VS-B), Codex annotations and Onboarding (VS), post-loss UI and procedural rings (VS-A), HUD scale, full PR-14 settings, rebinding, the Records page and un-retire (Alpha); see CT-38.

## Player Fantasy

*Dependency status at authoring (Glob of `design/gdd/`, 2026-10-26/27): Web Map & Transit is on disk (`web-map-and-transit.md`, designed in this run), the Pixel-Perfect Render Pipeline is on disk and locked. `design/ux/` and `design/accessibility-requirements.md` do not exist.*

### Framing: the honest table

**Core emotion: sober footing.** The art bible already names the target: "*The chart table.* Sober, competent, outside the fiction's danger but inside its mood. The player is a navigator planning a sortie." (ART 2.11, ART:258). The Chart Table is the place the player stands when they are not standing in the dark. What they feel there is not safety and not relief; it is **footing**: a flat, quiet surface on which nothing has ever lied to them, from which they decide one thing and then go back.

**The fantasy in one sentence:** *"I am the one who decides where the ship's life goes next, at a table that has never lied to me and will not tell me what I want to know."* The second clause is the point. A chart that answered "how far" and "how much fuel" would make the player a **shopper** comparing prices. A chart that only remembers what the crew lit makes the player a **navigator**: someone who has to have walked to the sight-glass first.

Three properties make that feeling, and each is a rule, not a mood board:

| Property | What the player feels | Why it is not comfort | Rule that delivers it |
|---|---|---|---|
| **It is true.** | "Whatever else is wrong with me, this is real." | It is true about very little: where the wrecks are, which ones the crew has lit, where the pod is. It knows nothing about the ship's health, the Web or the player's mind | CT-2 (truth channel), CT-27 (reads only `web.view`, `SaveInfo`, `save_status`, `restore_reason`), CT-39 (narrow truth) |
| **It is still.** | Time stops (solo) or at least the world is out of the frame (full-bleed chart). "The only motion is a slow drift of filaments in the current ring, so the map remembers the web is alive." (ART 2.11) | The world behind the panel is frozen *exactly as lit*, lies and tells included; nothing is softened. The web is still drifting on the chart | CT-4, CT-13, CT-40; PIPE:265; ART 7.3 "Sober means almost nothing moves, and what moves is the web" |
| **It decides nothing for you.** | Authority: the player is the navigator | No recommendation, no cost, no time, no distance; every number the player wants is somewhere in the world, at a price | CT-14, CT-35, WM guard rail 7 |

**The crew member the player is here** is the navigator of WM's committed crossing (WM:59, "the navigator's authority and the passenger's exposure in one beat"). Web Map & Transit owns the exposure half (the leg). This GDD owns the authority half, and the authority is honest only because the table is mute about the price.

**Framing alternatives considered** (Category UI: direct framing).

| | Framing | What it centres | Verdict |
|---|---|---|---|
| **A** | **Direct: "The honest table."** A sober human instrument where the player picks, stops and leaves. Nothing on it lies; nothing on it helps more than the crew has earned | P1 truth by layer (ART 7.0 row 3), P5 inward composition, the navigator's authority (ART 2.11) | **Adopted** |
| B | "The navigator's bridge": plotting the web as a mastery fantasy; route planning as skill expression | route planning, comparative choice between targets | **Rejected as primary.** The MVP web is a star of at most one hop (WM-4); a plotting fantasy over three nodes either overpromises or pulls cost, distance and ETA data onto the chart to give the plotting something to chew on, which is guard rail G6 below. It also repeats the framing Web Map & Transit rejected for the same reason (WM:78). The *word* navigator is kept (ART 2.11); the *fantasy of plotting* is not |
| C | "The refuge": the menu layer as sanctuary, a warm decompression space between horrors | relief, emotional recovery | **Rejected.** ART 2.11 puts the table "inside its mood"; a refuge would dim or soften the world (ART 7.4 forbids it), invite a calming cue (a relief sting is a mood effect, and menu audio carries no sanity or Attention effect, AA:562) and in solo turn the pause into a sanity-recovery exploit by feel even though Sanity reads nothing from it. It contradicts P2: the tell is still there when you come back. It is also Submission, which the concept ranks as "deliberately absent" (CON:89) |
| D | Diegetic prop: a physical chart table the player walks to in the home-ship | a place to stand, embodiment | **Ruled out (CT-1).** A prop is a lit, lie-capable instrument (ART 7.0 row 1) and is unreachable on a sortie. Recorded so a later stage does not re-propose it as a fantasy upgrade; no in-world schematic console exists: the Controller's Adapter owns the `inspect_card` context and In-World Instruments owns the card's content and draws no schematic (IW-28, C4) |

*Why A carries the concept's MDA ranking (CON:82-89):* Discovery (1) is delivered by the chart **growing only by looking** (the Tungsten tick appears only once the unknown was seen from the pod, ART 7.4; WM:85); Challenge (2) by the chart **withholding the price**, so the player must read the Graph's witnesses; Fantasy (4) by the navigator role and the home silhouette growing with each absorbed section (Moment 4). **"And also" test at system scale:** *it is a galaxy map, and also it only remembers what your crew has lit, and it will never tell you what the trip costs.* The second clause sparks the right question ("then how do I know?") and the answer is the world.

### Pillars served (verbatim, then scope notes)

> **Pillar 1: The World Never Lies.** "Pipes leak, frost forms, air hisses; the physical world is ground truth. Only instruments and perception can be wrong. [...] *Design test*: a hidden stat vs. a visibly leaking pipe → the pipe." (CON:208-221)

Primary. The Chart Table serves P1 from the side most systems never touch: it is the one surface in the game that is **not part of the world and therefore cannot be an instrument** (ART 7.0 row 3: "Chart table (meta): nobody in the fiction ... can a reading lie? never"). P1 says only instruments and perception can be wrong; this layer is neither. Its job is to be true *and to say almost nothing*, so that the truth the player needs still has to be read off the pipe.

> **Pillar 5: Deeper Is Richer, Loss Is Not Total.** "Risk and reward map to depth in the web. [...] *Design test*: safe farming vs. scarcity that pushes inward → scarcity." (CON:272-279)

Served by composition, not data: the chart darkens toward the centre ("the horizon the game never lets you reach", ART 2.11 carried element) and carries no completion, percentage or win datum (WM guard rail 5, WM:144). **Scope note (carried from WM:53):** the Chart Table does **not** carry the failure contract's "never knowledge" half. The chart is drawn from `web.view`, which is WORLD-scope and rolls back on restore (WM-28); the pick resets on every restore (CT-8). Nothing in this GDD may be cited as evidence that knowledge survives a failure; the Knowledge Codex (VS) owns that, and its chart annotations are a reserved slot (CT-38).

> **Pillar 3: The Web Watches.** "Attention is shown in the world (creaking, flicker, filament movement), never as a hidden meter." (CON:231-237)

Served by absence. The pillar names "filament movement" as an in-world Attention sign, and the chart also draws moving filaments. That collision is the most likely way this layer becomes an Attention meter by accident, so it is a named guard (G5, CT-40): **the chart's filament drift is constant and is never modulated by Attention, depth, sanity, time in session or anything else.** "Depth rings are shown as place, not as attention" (AT:1373).

> **Pillar 4: Sanity Rewrites Reality.** "**No lie is unfalsifiable**: the truth is always available through the physical world, a second instrument, a saner crewmate, the human channel, or time — and checking costs time, risk or trust. At least one channel of truth always remains." (CON:261-265)

Served twice, and the two halves pull against each other (resolved in "Under hallucination" below): the layer **is** a channel of truth that always remains, and it must **not** become a free way to check a lie, because P4 requires that checking cost something.

> **Pillar 2: Dread Before Chaos.** "Every breach is preceded by a tell a careful player could have perceived (sound, light, gauge drift, hull stress)." (CON:223-229)

Served by keeping the commitment off the menu: the console press is the point of no return (WM:1225; CT-10), the pause is non-diegetic only (KRN CR-16), and the panel never dims, blurs or desaturates the world it freezes (ART 7.4; PIPE:265), so a tell that was on screen when the player pressed Esc is still on screen while they stand at the panel.

### Emotional arc across a session at the table

Session shape from CON:181-185. Final copy for every surface is the UX spec's and the narrative director's; the register is fixed under "Tone and language".

| When | Surface | Target feeling | Must never feel like |
|---|---|---|---|
| Boot | Photosensitivity notice (CT-20), then main menu (CT-23) | *Plain care.* A notice that asks once and never nags | A legal wall, a splash screen, a countdown |
| Session start | Continue line: playtime, crew count, kind, `RISKY`, `RING <n>` (SL UI-4) | *A ship's log, not a trophy case.* "This is where we were." | A save-slot gallery, a thumbnail, a "last played 3 days ago" nudge, a completion bar |
| Planning | Chart, the pick (CT-7, CT-9) | *Sober competence* (ART 2.11) | Shopping, routing software, a quest board |
| Mid-sortie pause | Panel over the frozen world (CT-3, CT-4) | *A held breath.* The world is paused in the middle of being itself | A safe room, a decompression chamber, a dimmed "you are safe now" scrim |
| Natural stop | Save & Quit (CT-21) | *Closing the log.* "Natural stop: the ship is stable again." (CON:184) | A reward, "progress saved!", an autosave flourish |
| Mid-crisis quit while DOWNED | Hold-to-confirm "quit without saving", "the newest autosave will be used" (SL UI-3) | *Plain honesty about the cost* | A death screen, "you died", a guilt line |

### Anchor moments

All four occur in the MVP slice (home-ship 5 / derelict A 4 / derelict B 5 compartments, LSG MVP slice; WM roster). Glyphs and colours are WM section 3 (WM:1036-1050) and ART 7.4, cited and not restated with new values.

**Moment 1, "The table remembers what you lit" (first sortie; the Discovery payoff).** The cryo bay is stable. The player walks to the home pod console and presses with no pick. The console does not refuse: the chart opens (CT-10, "a press with no valid pick opens the chart"). Full-bleed Ground; concentric steps darkening inward (CT-13); one Tungsten point with a rectilinear Foundry silhouette for HOME; two plain Graphite-lit rectangles, bare, no labels, for derelicts A and B (UNSEEN, WM:1041; on the spatial map, which is the layout at every MVP canvas row at `H_default`, and in the compact layout only the CT-44 neutral ordinal appears, identical for UNSEEN and SEEN, a slot mark and not a name). Focus starts on HOME, not on a target, so focus is never a recommendation. The player snaps the bracket to A (CT-6); nothing appears beside it: no distance, no fuel, no name (CT-14, CT-35). Esc, resume, press. After the leg and the residual lamp (WM Moment 1), the next time the chart opens A is a Foundry silhouette with **one 1-art-px Tungsten tick** (WM:1043). *What the player learns:* the chart grows only by looking. It is a record of the crew's own eyes.

**Moment 2, "Stopping the clock is not stopping the world" (solo pause mid-tell at low sanity; the P1 + P4 payoff).** In derelict B, sanity at stage 3, a scrubber gauge reads fixed while frost creeps along the floor (CON P4(b)). The 10-30 s tell has begun (P2). The player presses Esc. A short cut-fade (inside the ART 7.3 band) and the right-third panel is on Ground: Resume, Chart, Display & Comfort, Save & Quit, in Paper White, the same at stage 3 as at stage 0 (CT-2, HR-6). To the left, the world is frozen **undimmed**: the lying gauge still reads fixed, the frost is still frost, the violet fringe on the gauge rim is still there, held in one frame (PIPE:265). Nothing on the panel says anything about any of it. Resume: the tell continues from exactly where it stopped. *What the player learns:* the menu is a place to stand, not a way out. It holds the world still; it does not make it kinder.

**Moment 3, "The log, not the trophy" (next evening, the main menu; the P5 tone payoff).** Boot, main menu. Continue carries one line of SIM facts as words: playtime, crew count, kind, `RISKY` if it applies, `RING 1` (SL UI-4; CT-23). No thumbnail, no screenshot of the ship, no "you were here", no percentage. If the newest record was unreadable, one plain ship-log line says so and Continue proceeds from the one before (SL UI-7). *What the player learns:* the game keeps a log of where they were, the way a ship does, and it will not flatter them.

**Moment 4, "The ship is drawn from your history" (after the weld; the Fantasy / Investment payoff).** Derelict B was berthed and its sections welded. On the chart B first abuts the home silhouette with a solid 1-px Hull Grey seam (berthed, WM:1044), and once absorbed it is drawn **as part of the home silhouette**, no tick (WM:1045). The home mark is now a different shape from the one in Moment 1. Nothing announces it; there is no "ship expanded" banner. *What the player learns:* "a ship that is literally assembled from your history" (CON:200, Retention: Investment) is visible on the one surface that cannot lie about it.

**Moment-ordering tripwire.** A 30-minute MVP solo session (CON:483-485) reaches Moments 1 and 2 almost certainly, Moment 4 if the weld is reached, Moment 3 at the second launch. If playtest notes call the chart "empty" or "useless" before Moment 1's tick appears, the cure is **not** to add data to the chart; it is to check that the residual lamp and the tick are legible (WM, ART). Adding a label, cost or name is never one of the levers.

### Design tests

> **Primary (P1):** "A chart that tells you where to go and what it costs vs. a chart that only remembers what your crew lit → the one that remembers."
> **Secondary (pause, P1 + P2):** "A pause that softens the world vs. a pause that holds the world exactly as it was → the held world."
> **Tertiary (records, P5 + the no-win-state anti-pillar):** "A save screen that shows how far you have come vs. a log line that shows where you were → the log line."

| Test | Rejects | Checkable as |
|---|---|---|
| Primary | fuel cost, travel time, ETA, distance label, danger/loot rating, "recommended", a name on an UNSEEN node, an interpolated pod marker, a route reconstructed from the pick | review against WM:1036-1050 and CT-35; lint on strings (CT-28); (auto) the chart double reads only `web.view` and never `web.leg_timing()` (CT-27) |
| Secondary | dim, blur, desaturation, vignette, scrim, slow-motion, a calming cue, a sanity-relief effect, any element that changes with sanity stage | screenshots at stage 0 and forced stage 3 (AC-SN-83); pixel diff of the world region before Esc and while the panel is up equals zero |
| Tertiary | thumbnail, completion %, "rings charted", achievement row, wall-clock nudge, "you died", "game over" | screenshot of UI-4; SL vocabulary lint (SL AC-SL-86) |

Further tests this system adds, from the concept's own form: a **dimmed or blurred world behind the pause panel vs the world frozen exactly as lit → frozen**; a **death screen, "loading" indicator or tip vs black then control (ART 2.10, SL UI-6) → black**; a **minimap in play vs a chart you open (ART 7.0 "No minimap in play") → the chart you open**; a **confirm dialog on the hop vs the console press as the point of no return (WM:1225) → the press**.

### Under hallucination: how the fantasy deepens, never breaks

The layer has **no corruptible surface** (CT-2; PIPE PR-3/PR-10; PF:223; SN:1226). "Deepens" therefore cannot mean "the menu lies". It means three things, and the third is this system's one new rule.

1. **The contrast sharpens.** At stage 0 the panel and the world agree; at stage 3 the panel is the only thing on screen the player is sure of, and the world beside it is frozen mid-lie (Moment 2). The layer's value as an anchor rises exactly as the world's falls. Nothing on the layer changes for this to happen; the world changes around it. This is P4's "at least one channel of truth always remains", given a fixed place on screen (HR-6).
2. **The frozen lie is intended, and documented as intended.** `PAUSED_FROZEN` holds the last frame, including any hallucination, undimmed (PIPE:265). A tester or player at stage 3 will see a phantom held still beside a true menu. That image is the fantasy working, not a fault. It is an Edge Case and a QA note (Edge Cases, Acceptance Criteria), so nobody "fixes" it with a dim.
3. **Narrow truth (CT-39): the table is true, and it is not an oracle.** *The layer displays only facts that no hallucination class can lie about.* Every field it draws comes from `web.view`, `SaveInfo`, `save_status` or `restore_reason`, and none of them is touched by classes (a), (b), (c) or (d). Conversely, **a fact that any hallucination class can falsify (Graph state, entity presence or position, salvage contents or value, a crewmate's rendered form or location) never enters the layer**, because a true copy of it on a free, pausable menu would make that lie free to disprove, and P4 requires that "checking costs time, risk or trust" (CON:263). This is the positive form of the negative rules CT-31 to CT-37 and gives the review one test instead of seven lists: *if a hallucination could make the world disagree with this field, the field does not belong here.*

| Hallucination class | May it touch the Chart Table? | Why the fantasy holds |
|---|---|---|
| (a) phantom entities | Never. No entity, threat, proximity or "last seen" element exists on the layer (CT-32; HE:377, :1358) | the chart cannot confirm or deny a phantom, so checking one still costs a look into the dark |
| (b) false system states | Never. No Graph, module or suit datum exists on the layer (CT-34, CT-37; LSG:22 "never operates the Graph as a menu") | the scrubber that reads fixed must still be checked against the frost |
| (c) crewmates as horrors (VS-B) | Never wraps the layer. **Risk K1:** ART 7.4 / 7.5 reserve a crewmate beam-shape glyph on the chart as "one of exactly three ways to find a crewmate". Under OVERLAY that glyph is a zero-cost check against class (c) | the human channel is P1-inviolable, so the glyph is true; whether its *price* is right is VS-B's call (Open Questions) |
| (d) phantom salvage | Never. No loot value, contents or remaining-salvage count (CT-33; BS:334, AC-BS-47c) | the crate is still checked by weight, a second scanner or the trip home (CON P4(d)) |

**Negatives lifted into rules:** no gleam, glide, tremor or settle on any element (ART 7.3: those signatures are readings only); no violet or flesh at any stage (ART 4.6, HR-6); no change of filament drift, ring darkness, font, scale or layout with sanity stage; no sanity-stage-dependent audio on the UI bus (AA:562).

### Tone and language

**Register:** ship-log, low-affect, factual, few words (SL:127, SL:1008; WM:1319). The Chart Table speaks like a logbook kept by a tired, competent person. Final copy is UX's and the narrative director's (CT-28); this section fixes the register and the lexicon the copy must respect. Fixed item labels: Resume, Chart, Display & Comfort, Save & Quit (CT-3); Continue, New campaign, Quit (CT-23; SL:993).

| Use | Never use (on any surface of this layer) | Why |
|---|---|---|
| chart, record, sortie, ring (as place, `RING <n>`), pod, home | map completion, explored, discovered %, charted %, progress | P5 / no win state (WM guard rail 5) |
| Continue, Save & Quit, New campaign, newest record, autosave | load game, slot, checkpoint, respawn, save point, quicksave, progress saved (SL:993; AC-SL-86) | SL vocabulary is locked |
| *(no word at all beside a node)* | destination, objective, quest, target recommended, danger, safe, threat, loot, value, reward, distance, ETA, fuel, cost, time | P1 / P3 / guard rail 1 |
| `IN TRANSIT` (instrument face, iff `pod_at == IN_FLIGHT`, CT-15) | arriving in, departing, % complete, any countdown | WM:300, WM:1048 |
| *(silence)* | you died, game over, lost, failed run, the web took you, sanity, attention, the web is watching | SL:993; P3; P4; no authored story |

No exclamation marks, no emoji, no second-person exhortation ("Be careful out there"), no tips, no rhetorical questions (SL:127; no pause-time hint system). Sentences in messages state one fact each. Numbers appear only where Save/Load already supplies them (playtime, crew count) in the tabular instrument numerals (ART 7.1). *Illustrative register* (not final copy): acceptable, "Newest record unreadable. Continuing from the one before it." (SL UI-7 placeholder, SL:1003); rejected, "Oh no! Your save was corrupted, but don't worry, we found a backup!" (exclamation, affect, reassurance); rejected, a node line carrying a distance, a time and a fuel figure (every element is guard rail G6).

### Reference techniques: adopt and reject

| Reference | Technique | Verdict |
|---|---|---|
| *Project Zomboid* (designer touchstone) | a solo pause that freezes the world in place; a world map that records only what the character has seen | **Adopt** the plain freeze (solo, CT-4) and "the map shows what you saw" (WM-28). **Reject** player-drawn map annotations at MVP (a VS Codex question, CT-38) and any loot or zombie-density overlay |
| *Beyond the Aquila Rift* (concept inspiration) | the comforting false room the protagonist is kept in "for his own good" | **Reject, and invert.** The Chart Table is the anti-Aquila surface: it never offers the kind lie. No softened world behind the panel, no reassurance copy, no "everything is fine" summary. The kind lie belongs to Sanity's world filter, never to this layer |
| *Pandorum* (concept inspiration) | the crew's own light as the only reliable thing in a ship that lies about its state | **Adopt** the principle that the human thing is the trustworthy thing: the chart is drawn in human, rectilinear geometry over an inhuman shape (ART 3.4). Nothing visual from the film enters the layer |
| *FTL: Faster Than Light* | a node map with one click per jump; beacon hazard icons; the rebel-fleet advance line | **Adopt** the one-step pick on a small node graph. **Reject** hazard / distress / store icons, the fuel counter beside the map, and above all the **advancing fleet line** (a progress / threat datum on the map, G4 and G6) |
| *Sunless Sea* | a chart filled in only by sailing it; fog as default | **Adopt** "charted only by going and looking" (the Tungsten tick, Moment 1). **Reject** the Terror, fuel and supply counters beside its chart (P3: never a meter) |
| *Stellaris* (concept inspiration) | a galaxy of nodes and lanes as readable human geometry | **Adopt** legibility of the human layer (ART 3.4). **Reject** every overlay mode (resources, threat, intel level) and survey-reveal |
| *Outer Wilds* | the Ship Log rumour map: a record of what you learned, linked by inference | **Reject for MVP** (speculative links reveal; the chart never reveals, ART 7.4). **Reserve** the "record of what you learned" idea for the VS Knowledge Codex annotation slot (CT-38), with the IDX:47 condition that every annotation has a stated effect on play |
| *Alien: Isolation* | the menu does not stop the threat when it is diegetic (the save station) | **Reject the diegetic half** for this layer (CT-1: a prop could lie). **Adopt** its plainness of menu copy. The P2 commitment lives at the pod console, which is already diegetic (WM:1225) |
| *Darkwood* | a map with no "you are here" marker | **Reject.** `pod_at` is a truth the chart owes (WM:1047); disorientation as horror belongs in the world, not on the one honest layer |
| *Dead Space* | fully diegetic UI | **Reject for this layer** (ART 7.0 places menus, settings and the web map outside the fiction). Diegetic reading belongs to In-World Instruments |
| *Barotrauma* | the Status Monitor / nav terminal ship-overview screen | **Reject** (LSG:32 already rejects it): no ship-state overview on any meta screen |
| Edward Hopper (outside the medium) | a still, lit interior with dark around it; mood held without event | **Adopt as tonal reference only** for the pause image: a quiet, flat, well-lit plane beside an unresolved scene. Not a visual style instruction |

### Anti-pillar guard rails (stated here, enforced in Detailed Rules)

| # | Guard | Anti-pillar / pillar | What it forbids on this layer | Source |
|---|---|---|---|---|
| G1 | **Not a freeform builder** | anti-pillar 1 | route building, waypoints, multi-leg plans, pod or ship customisation screens, a ship-layout editor, an "arrange modules" view | WM-4 star topology; WM guard rail 3 |
| G2 | **Not a combat game** | anti-pillar 2 | any horror, threat bar, proximity, direction, kill or "encounter" record; any loadout or weapon screen | HE:377, :1358 (AC-HE-119); CT-32, CT-37 |
| G3 | **No intentional PvP** | anti-pillar 3 | any crew-facing action from the layer that can strand, harm, kick or disarm a crewmate; any per-player blame or incident log on the chart or menus | CON P4(c), anti-pillar 3 |
| G4 | **No authored story** | anti-pillar 4 | captions, flavour text on nodes, named derelicts with backstory blurbs, narrated loading or arrival lines, a pause-screen quote, tips | WM:1315; WM guard rail 4 |
| G5 | **No win state, no progress** | anti-pillar 5; P5 | completion %, rings charted, "deepest reached" stat, achievements row, a centre that can be selected, a depth-progress bar; **constant filament drift and constant ring darkness**, never modulated by Attention, depth, sanity or session time | WM guard rail 5; AT:1373; ART 2.11; CT-40 |
| G6 | **No meter, no reveal, no recommendation** | P1, P3 | fuel, travel time, ETA, distance, danger, loot estimate, "recommended", a name or dialect on an UNSEEN node, an interpolated pod position, a route rebuilt from the pick (**sole exception:** the CT-44 neutral ordinal in the compact layout, which is a slot mark and not a name, dialect, template word or status word, and is the identical form on UNSEEN and SEEN rows) | WM guard rail 7; CT-14; CT-15; CT-35; CT-44 |
| G7 | **No soft pause** | P1, P2 | dim, blur, desaturation, vignette, slowed sim, calming or relief audio, any sanity or Attention effect on the layer | ART 7.4; PIPE:265; AA:562 |
| G8 | **Narrow truth** | P4 | any field a hallucination class can falsify | CON:261-265; CT-39 |
| G9 | **No death theatre** | user decision ("death is a bitch. It happens."); P5 | death screen, "you died", "game over", a respawn button, a stats-of-your-run screen; MVP restore is Save/Load's (SL-7) with black then control (SL UI-6) | SL:993; ART 2.10; ART 7.4 "Loss" |
| G10 | **The commitment is not on the menu** | P2 | a "confirm sortie", "launch", "hop now" or resume-and-hop control anywhere on the layer | CT-10; WM:1225, WM:1301 |

### Validation: we will know this Player Fantasy was right if

At the MVP kill/continue playtest (at least three outside testers, a 30-plus-minute solo session, CON:483-485):

1. **No tester asks the chart for a number and gives up.** At least one tester asks "how far / how much fuel" aloud **and** later reads the answer off a world witness (sight-glass, drip, reactor timer; WM:66). If two or more testers stop starting sorties because they cannot price them, the fix is legibility of the Graph's witnesses (LSG, ART), never a number on the chart.
2. **The chart is found without instruction after the first sortie.** Every tester starts their second sortie without help. Owned observable: median time from the first chart open of a sortie to the console press, `CHART_TO_PRESS_S` `[PLACEHOLDER]` 20 s (flag above 40 s as the friction signal that Esc, Chart, pick, Esc, walk, press is too long; the reserved remedy is a separate chart action owned by the Player Controller, never data on the chart). It is a **playtest observable and a manual acceptance threshold**, not a gameplay constant and not registered.
3. **The pause reads as held, not safe.** No tester describes the pause or the chart as a "safe room" or uses it to "calm down"; at least one tester at stage 2 or above comments on the frozen world still being wrong behind the panel (Moment 2 felt). No tester reports the frozen hallucination as a menu bug once the QA note is in the build.
4. **The tick is noticed.** At least two of three testers mention unprompted that a wreck "changed" or "got a mark" on the chart after they saw its light (Moment 1 felt).
5. **The log is not a trophy.** No tester asks for a thumbnail or a completion figure on the main menu as a missing feature; requests for *more* log facts are input for the Codex (VS), not for this layer.
6. **The Web stays unmetered.** No tester reads the chart's filament drift or ring darkness as a danger signal (if one does, check G5 first).

### Risks raised by the Player Fantasy (routed, none decided here)

| # | Kind | Statement | Routing |
|---|---|---|---|
| K1 | Risk (VS-B) | The reserved crewmate beam-shape glyph on the chart is true (human channel, P1) and, under OVERLAY, is a zero-cost check against hallucination class (c): open the chart, see where your friend really is. That may make class (c) too cheap to falsify against P4's "checking costs time, risk or trust" | Not decided; the slot is reserved (CT-38). Route to Co-op Session and Sanity at VS-B: decide whether opening the chart in OVERLAY (attention away from a running world) is price enough or whether the glyph needs a condition |
| K2 | Risk (Sanity) | Solo `PAUSED_FROZEN` lets a player study a frozen frame for a palette leak at zero time cost. A one-frame temporal flicker may be caught or missed by the freeze, which is luck, not design | Accepted for the layer (a leak's job is to be noticed: "Subtle is allowed; absent is not", CON:215; pausing is also an accessibility good). Route to Sanity: confirm every class-(a)-(d) leak has a component visible in a single held frame, so the freeze is never the only way to catch one and never hides one |
| K3 | Risk (tone) | Moment 1 depends on the chart being visibly sparse at first; a "the chart is empty" reaction may push later stages to add labels | Tripwire in "Moment-ordering tripwire" and validation item 1: the lever is world legibility, never chart data |
| K4 | Note | "Navigator" is kept as a word (ART 2.11) while the plotting fantasy (framing B) is rejected; later copy must not drift into "plot a course" | Lexicon table above |

## Detailed Design

### Core Rules

Rules are grouped by topic (A-H); ids are stable, not sequential within a group (CT-42 to CT-44 sit with the topic they belong to). Rule ids are `CT-n`; formulas are `D.n` (Formulas); acceptance criteria `AC-CT-n` (Acceptance Criteria) and open questions `OQ-CT-n` (Open Questions). Each rule is stated as **Rule**, **Why**, **Source**. Where two specialist proposals differed and no earlier ruling decided it, the choice is marked "(orchestrator ruling: ...)". Names that Web Map & Transit marks provisional (`web.view`, `web.state`, `transit_state`) are used with that mark carried (WM-36). `HOME_ID` := the `derelict_id` of the `web.view.nodes[]` entry whose `kind` is the home ship (the WM-3 roster `kind`); it is a derived label, never a constant and never registered (WM defines no such constant). Godot-specific mechanisms are not designed here: each is marked "-> ADR candidate" (ADRs (a)-(f) are listed at the end of the section), and anything the engine reference cannot source is marked NOT SOURCEABLE. **Engine reference coverage:** `docs/engine-reference/godot/` has `modules/ui.md`, `modules/input.md`, `modules/rendering.md`, `VERSION.md` and `breaking-changes.md`; there is **no** `scripting.md` or `core.md`, and nothing on `ConfigFile`, `SceneTree.paused` / `process_mode`, `Window` display-mode APIs, `Input.mouse_mode`, key-echo detection, focus-out notifications, font-importer flags or AccessKit property names; those are NOT SOURCEABLE and are routed to the ADRs below.

#### A. What the layer is

**CT-1. A meta layer, not a place.**
*Rule.* The chart table is one PIPE layer-3 `CanvasLayer` in the **root viewport**, opened by the `chart_table` input action (Esc / Start) from the `gameplay` or `inspect_card` context (PCI:62: pause is reachable from both; in the pod, on a derelict, in the home-ship). It has no prop, no location, no lighting, no glow, no ambient, no occlusion and no camera jolt. It is scaled by the HUD scale `H` for the bitmap face and continuously for the reading face (PR-12).
*Why.* A prop would be a lit, lie-capable instrument (ART 7.0 row 1) and could lie; the meta layer is honest by definition (ART 7.0 row 3: "can a reading lie? never"). The chart must be reachable on a sortie, which a home-ship console could not be. An in-world "chart-table console" or ship schematic does not exist (IW-28, C4).
*Source.* PIPE:94-103 (PR-3); PCI:78; ART 7.0.

**CT-2. Truth channel.**
*Rule.* Layer 3 is never wrapped by the Perception Filter or by Sanity. No gleam, glide, tremor or settle occurs on any chart, panel or menu element (ART 7.3: the three motion signatures are readings only, and nothing on this layer is a reading). The layer is pixel-identical at sanity stage 0 and at forced stage 3, apart from the world visible beneath the panel. It never carries Violet or Flesh (ART 4.6), and Alarm Red appears on no element, because the layer draws no failing instrument (HR-4: red is never text). The only motion on the layer is the filament drift (CT-13, CT-40), the ART 7.3 cut/fade band and the ART 7.3 bracket ease-out, and the linear hold bar (CT-22).
*Why.* P1; ART HR-6 names menu, pause and settings layers as anchors; PF:223 (row 9, TRUTH: the pass sits below every `CanvasLayer`); SN:1226 (TRUTH anchor).
*Source.* PIPE PR-3 / PR-10; ART 7.3, 7.4, HR-4, HR-6. **Note.** The world frozen or running under the panel keeps whatever it showed, including a lie (CT-4, Player Fantasy "Under hallucination"); the panel does not.

**CT-3. Surface architecture: panel first, stable order.**
*Rule.* `chart_table` opens the **panel**, a right-third plate over the world (ART 7.4). The panel has exactly four items in this fixed order: **Resume, Chart, Display & Comfort, Save & Quit**. The order never changes with state. An item that cannot act is shown **disabled** (Graphite lit), never hidden. Default focus on open is Resume (non-destructive). Chart (and any message plate) replaces the panel in a cut (a replacement is not an appearance, so it takes no fade), except under reduce flashing, where it uses the long end of the ART 7.3 fade band (`CHART_FADE_MS`, CT-13). "Display & Comfort" opens its page inside the panel's column. The panel carries only item labels and the one-line newest-record fact (CT-17); every paragraph of copy lives on a full-bleed **message plate** on the chart-table register (CT-28).
*Why.* One entry point; the world stays visible under the panel as ART 7.4 specifies; the HUD (layer 2) stays visible and frozen behind the pause panel, except that an active warning mark is held ON solid (HUD HS-23; the HUD's rule, acknowledged here; the full-bleed chart covers layer 2 entirely); a stable order is a muscle-memory affordance and menu layers are anchors that must not shift (HR-6). The accepted cost is that the chart is two actions from a pause; CT-10 (a no-pick console press opens the chart) offsets it. A second `chart` action stays an Alpha playtest fallback owned by the Player Controller and is not designed here.
*Provisional (HR-13, ASK-CT-8).* Whether a player-initiated toggle of a third-of-screen plate counts under the 3 Hz ceiling is an art-director call. Until ruled, the visual open and close of the layer is a single transition each inside the ART 7.3 fade band and a repeated panel-to-chart-to-panel loop is not rate-limited; the scoped guard `CHART_REOPEN_MIN_MS` (334 ms, `[PLACEHOLDER]`, derived from the cited 3 Hz ceiling) is carried in Open Questions as the default form if ruled yes. It would apply to the Chart item only, never to Esc, never to the console-press entry, and close is never delayed. Not a rule until ruled.
*Source.* ART 7.4, HR-6, HR-13; PCI:78.

**CT-4. PauseGate (answers KRN Open Question 6).**
*Rule.* One adapter, `PauseGate`, with two modes. **SOLO (MVP):** opening the layer calls `request_pause`; closing it calls `request_resume` (KRN CR-15; PCI:147). **OVERLAY (VS-B, reserved):** no kernel call; the kernel stays `RUNNING`; panel and chart are identical. **No chart, pick, panel or menu rule may read `kernel.state == PAUSED`**; `PauseGate.mode` is the one place SOLO and OVERLAY differ. The single exception is inside `PauseGate` itself: it exposes one boolean, `can_save_quit`, true only when the mode is SOLO **and** the kernel reports `PAUSED` (Save/Load's `PAUSED -> SAVING`, SL-6, needs it); nothing else on the layer reads it except the Save & Quit item's enabled state. In OVERLAY `can_save_quit` is replaced by Session / Save-Load's form (`saveload.request_save`, reserved, SL:37, SL-17). Verbatim answer to KRN:961: **The Chart Table does not assume "pick a target" happens while paused.** The pick is a host-local selection valid with the kernel `RUNNING` or `PAUSED`; nothing depends on time standing still.
*Timing.* The layer enters PANEL on the press frame (presentation does not wait); the kernel honours the control-plane request at a tick boundary (P6, CR-15) and is `PAUSED` up to one tick later; the fade (`CHART_FADE_MS`, inside the ART 7.3 band) hides the gap, and `can_save_quit` is false until the kernel confirms. A same-boundary `request_pause` then `request_resume` (a double press) must leave no wedged half-open state: the layer keeps its own `open` boolean and `PauseGate` re-issues the latest desired state until the kernel matches (host-local, no commands). Whether the kernel coalesces or executes a same-boundary pair in order is routed (ASK-CT-9, kernel).
*Why.* KRN CR-16: pause is single-player only; in co-op a menu cannot stop a shared sim. Reading kernel state anywhere else is the divergence the brief's risk R4 warns about. The pause is available during a leg and during `TRANSIT_HOLD` and "changes nothing" there (WM:1268).
*Source.* KRN CR-15, CR-16, KRN:129, KRN:961; PCI:147; SL-6. **-> ADR candidate (a):** the relationship between kernel `PAUSED` and `SceneTree.paused` / `process_mode` of layer 3 is NOT SOURCEABLE in the engine reference. Behavioural requirement stated here: layer 3 handles input and animates on its own UI clock (CT-43) while the kernel is `PAUSED`.

**CT-5. Input depth: Esc closes, cancel steps back.**
*Rule.* In the `chart_table` context (PC-2 sends `set_move_intent {NONE}`, clears the run latch, **submits `cancel_channel`** for a hold-mode channel, and releases the cursor):
- `chart_table` (Esc / Start) is a toggle and **closes the whole layer from any depth** (SOLO then calls `request_resume`).
- `cancel` (X / B) **steps back one level**: Display & Comfort, Chart or a message plate returns to the panel; the panel closes (PCI:73, "closes the current UI").
- **Hold exception (orchestrator ruling: the stricter, safer reading of the rulings).** While a hold-to-confirm fill is running (CT-22), release, `cancel` **or** `chart_table` aborts the fill (the bar snaps to empty in the same frame) and **nothing else happens in that press**; a second `chart_table` press then closes the layer. The alternative (Esc during a fill aborts and also closes) was proposed and declined: an Esc that resumes the simulation while a destructive hold was in progress is a second effect from one press.
- On boot-flow screens (notice, main menu, UI-5, UI-8, UI-9) `chart_table` does nothing **except abort a running hold fill (TX-10)**, and `cancel` goes back only where a previous boot screen exists (UI-5 to the main menu; UI-8 and UI-9 have none).
- During a Save & Quit write (CT-21, state `SAVE_QUIT_WAIT`) `chart_table` and `cancel` are ignored so a half-written state is never abandoned.
- `zoom_in` / `zoom_out` are **not** actions of the `chart_table` context; a zoom press while the layer is open is discarded (PIPE:688; PCI:62). No zoom row, hint or glyph appears on any surface (zoom is never a setting, PIPE:232). The mouse wheel scrolls in this context (legal across contexts, PCI:699); nothing scrolls at MVP.
- Directional navigation is the same physical controls that move the body, re-read as four-way navigation: the four-way cardinal read of the stick is **this layer's own re-read** of the Controller's move vector (the Controller itself quantises to eight octants, PCI:84), gated by `MOVE_DEADZONE` / `MOVE_DEADZONE_EXIT` (PCI, not registered); accept is `interact` (E / LMB / A); back is `cancel`; close is `chart_table`. The Controller defines default bindings for the `gameplay` context only (PCI:64-80: `interact` :72, `cancel` :73, `chart_table` :78); no `chart_table`-context table exists, so directional navigation and accept are **(provisional — ASK-CT-2 to the Player Controller / Settings: reuse `interact` and Godot's built-in `ui_*` set, or add per-context actions)**. No new binding is introduced here. A rebind that removes the last path to the chart table on the active device is refused `BINDING_REQUIRED{chart_table}` by the Controller's side (PCI:699).
*Source.* PCI:62, :73-80, :699; PIPE:688, 232.

**CT-6. Focus is a shape; dual focus.**
*Rule.* Focus is the Tungsten bracket (left bar plus chamfer corners) **and** a one-art-px thickening of the focused bezel; never a Tungsten fill (ART 7.2, HR-11). The bracket relocates with a 100 ms ease-out (ART 7.3). The bracket is drawn from **keyboard / gamepad focus only**: it hides on mouse motion and reappears on key or pad navigation (PCI PC-13). Mouse hover turns the item's label Tungsten (ART 7.4) and never moves the bracket; a chart node carries no label, so hover changes nothing on a spatial node and the click's visible result is the route (CT-7) *(provisional — OQ-CT-9: whether a node needs a hover affordance)*. A mouse click on an item activates it and moves key/pad focus to it, so the next pad press navigates from where the mouse acted; the bracket stays hidden until key or pad input. Glyphs follow the last device with `DEVICE_SWITCH_HYSTERESIS` (PCI, registry; value in Tuning Knobs); mixed input is permitted. Every flow is gamepad-complete (HR-14). A click on an item is activated only when the pointer-up lands on the item that received the press.
*Why.* Godot 4.6 split mouse/touch focus from keyboard/gamepad focus (`modules/ui.md`, HIGH risk, "test with BOTH"): the bracket must be drawn from the keyboard/gamepad focus and not from hover or PC-13 cannot hold. Both paths are tested (Acceptance Criteria).
*Source.* ART 7.2, 7.3, 7.4; PCI:141 (PC-13); HR-11, HR-14; `modules/ui.md`, `modules/input.md`.

#### B. The pick and the sortie

**CT-7. The pick (`ChartPick`).**
*Rule.* `ChartPick` is NONE or one `derelict_id`. It is host-local presentation state of this layer: never simulation, never persisted, never a field of `web.state` or `web.view`, never assumed to depend on pause (WM-8, WM:1300). **Setting:** on the chart with `pod_at == HOME_ID` and `transit_state == IDLE`, accept (CT-5) on a legal target (CT-9) sets the pick, replacing any previous pick; accepting the already-picked node leaves it set (idempotent, so a double press cannot clear it by accident); accepting the HOME node (the origin) **clears** the pick; accepting a non-selectable node does nothing (**orchestrator ruling:** the ux-designer's rule; the ui-programmer's alternative, "accept on the picked node clears it, HOME is a no-op", was declined because a double press would clear the pick). Closing the chart or the layer **keeps** the pick. The pick is shown by a Tungsten dashed planned-sortie line from the HOME node to the picked node (WM:1048 "planned sortie"), plus the bracket wherever focus is: a shape difference (dashed versus absent), so no colour carries it. A focused but unpicked node shows the bracket only and never a previewed route (a previewed route is a recommendation by default). Nothing else changes on the node: no label, no number (CT-14).
*Why.* The pick is "the deliberate step" (WM:1225); the console press is the commitment (CT-10). Focus and pick are two states, so the player needs a non-colour way to see "picked" while the bracket sits elsewhere. The Tungsten dashed line **always means "the target the next console press carries"**: HOME to `ChartPick` when the pod is at HOME, the pod's node to HOME when it is at a derelict, drawn only while `transit_state == IDLE`. It is not a recommendation (from a derelict it is the only legal move; from HOME it appears only after the player's own pick), and this single definition closes OQ-CT-10 (the route is the pick mark; no second mark).
*Source.* WM-8 (WM:203), WM:1225, WM:1300.

**CT-8. Pick lifecycle.**
*Rule.* `ChartPick := NONE` on: (1) any `request_restore`, for every `restore_reason` (`NEW_GAME`, `LOAD_GAME`, `DEATH_STUB`, `FAULT_RECOVERY`; SL-21); (2) arrival (`POD_ARRIVED`, WM-31), detected equivalently as `pod_at` leaving `IN_FLIGHT` so a test double needs no event bus; (3) whenever the picked record leaves the legal-target mirror (ABSORBED, ORPHAN, `berthed`, or absent from `web.view`). The pick is revalidated against `web.view` on every chart open and every `web.view` change. It is **frozen** (not editable) while `transit_state != IDLE`. No rule reads it except the console seam (CT-10). If the node holding key/pad focus leaves the selectable set, focus moves to HOME; the bracket otherwise does not move and no message is shown.
*Why.* A stale pick is the only way the chart and the press can disagree. The sim still validates with Web Map & Transit's own reasons (`TARGET_UNKNOWN`, `TARGET_ABSORBED`, `TARGET_BERTHED`, `BERTH_PENDING`; WM-9), which this GDD cites and never re-lists as its own.
*Source.* WM-9 (WM:205-217), WM-31; SL-21.

**CT-9. Legal-target mirror.**
*Rule.* The chart computes `legal_targets` (D.4) purely from `web.view`, for **selectability only**: from HOME, derelicts whose `status` is not ABSORBED or ORPHAN and that are not `berthed`; from a derelict, only HOME; with `transit_state != IDLE` or `pod_at == IN_FLIGHT`, none. UNSEEN derelicts are selectable bare nodes (WM:1299; OQ-WM-4). ORPHAN is drawn but neither focusable nor selectable (WM:1046). The mirror is **not a validation and not a second copy of WM-9's reason order**; every refusal is Web Map & Transit's and Interaction's, shown as words on the register plate (not here). **The mirror never reflects `FUEL_LOW`, `BUSY`, `OUT_OF_RANGE`, `NOT_LIT`, `ARRIVAL_BLOCKED`, `CAPACITY` or `BERTH_PENDING`**; those refusals are words at the console only (CT-35). **Known gap, not hidden:** `BERTH_PENDING` (a hop to any other derelict while an unwelded one is berthed) is not mirrored; the node stays selectable and the console refuses in words (OQ-CT-3).
*Why.* WM-9 defines `TARGET_ABSORBED` (WM:217) as "a welded-in or orphaned record; the Chart Table never offers it", so that reason already covers ORPHAN, and WM:1046 draws ORPHAN not selectable; the mirror is what makes "never offers" true. Greying a node on low fuel would put a fuel meter on the truth layer, which is why the exclusion above is a named sentence (CT-35).
*Source.* WM-4 (WM:193); WM-9 (WM:205-217).

**CT-10. The console seam: the only simulation write.**
*Rule.* The chart submits nothing. The **pod console press** (Interaction's CONSOLE subject, `tool_required` NONE) is the only simulation write and the only commitment (WM:1225, WM:1301): no "confirm sortie", no resume-and-hop gesture, no confirmation dialog anywhere on this layer. Resolution at the press, in the Input Adapter's seam:
1. **From a derelict:** HOME is implied (WM-4, WM-8: "from a derelict the press carries HOME implicitly"). If the chart is opened there it shows the bracket on HOME as the only focusable node, and the planned-sortie line from the pod's node to HOME is drawn from `pod_at` (not from `ChartPick`), because that is the only legal move; the selection cannot move to another node. This implied line carries the same single meaning as CT-7's dashed line (the target the next console press carries), drawn only while `transit_state == IDLE`; it is not a recommendation.
2. **From HOME with a valid pick:** the press submits `pod_hop{target_id := ChartPick}`. The same pick supplies `derelict_id` to the home console's `berth{derelict_id}` (WM-21); `unberth` takes none. Which verb a press means is Interaction's, not this GDD's.
3. **From HOME with no valid pick:** the press **opens the chart** (host-local, no refusal, no simulation effect), **even when exactly one legal target exists**: an auto-implied hop from HOME would make the press both choice and commitment. Default focus on opening is the pod's own node (HOME), never a target, so focus is never a recommendation. In SOLO the open requests a pause as in CT-4.
The seam (the Adapter injects the host-local pick into the payload; a no-pick press routes to "open chart") is **provisional — ASK-CT-1 to Interaction and the Player Controller** (IA:131 lists `pod_hop` as "Web Map's"). Degraded if refused: the press refuses with a words-only reason from Interaction's table; this GDD adds no reason id.
*Why.* P2 (the commitment at the console, not on a menu); WM-8. *Source.* WM:203, WM:1225, WM:1301; IA:131. **-> ADR candidate (d):** the pick-to-console submit seam.

#### C. Drawing the web

**CT-11. Drawing `web.view` (the twelve rows, owned layout).**
*Rule.* The chart draws `web.view` exactly per WM section 3 (WM:1036-1050); this GDD owns the layout and Web Map & Transit owns the drawing grammar. The summary below restates the grammar only to bind the layout; colours and silhouettes are WM's and ART 4.6's.

| `web.view` fact | Drawn as | Focusable | Selectable |
|---|---|---|---|
| HOME node | Tungsten fixture point + 1-px rectilinear Foundry silhouette (`ui_glyph_node_home_24`) | yes (origin; start focus when `pod_at == HOME_ID`) | no (origin; accept clears the pick, CT-7) |
| UNSEEN derelict | plain Graphite-lit rectangle (`ui_glyph_node_wreck_plain_24`), no dialect, no tick, no label | yes | yes |
| SEEN / BOARDED | Graphite-lit rectilinear silhouette chosen by `template_id` (`ui_glyph_node_wreck_foundry_24` at MVP) | yes | yes |
| `unknown_seen` | one 1-art-px square Tungsten tick inside the silhouette, 1-px Ground clearing, no glow, no circle | n/a | n/a |
| `berthed` | silhouette abutting HOME at the socket edge, solid 1-px Hull Grey seam | no | no |
| ABSORBED | drawn as part of the HOME silhouette, no tick | no | no |
| ORPHAN | Graphite mid, detached, no tick | no | no |
| `pod_at` HOME / a derelict | small rectilinear pod marker (`ui_glyph_pod_24`) at that node | n/a | n/a |
| `pod_at` IN_FLIGHT | **no marker**; state word `IN TRANSIT` (CT-15) | n/a | n/a |
| planned sortie | Tungsten dashed line = "the target the next console press carries": HOME to the pick (CT-7), or pod node to HOME (CT-10); drawn only while `transit_state == IDLE` | n/a | n/a |

State is never carried by colour alone: every state has a distinct silhouette or mark (tick, seam, detachment, glyph), satisfying SL:1010 and HR-10 by construction. Routes, seams and rule lines are 1-art-px quads aligned to whole art px, never width-1 line draws on a fractional transform (4.7 `CanvasItem` line drawing has no antialiasing feather; -> ADR candidate (b)).
*Visibility risks carried, not decided here (art-director, Visual/Audio):* Graphite lit on the outer ground sits below the 3:1 glyph floor (HR-12) for silhouettes, and ORPHAN's Graphite mid is near-invisible on Ground; both are WM drawing rows (WM:1041-1046), routed to WM and the spike as a possible outline on the 24x24 glyph assets (a glyph change, not a WM row change).
*Source.* WM:1036-1050; ART 3.4, 4.6, 7.2, 7.4.

**CT-12. Layout data is authored, per record.**
*Rule.* A Chart-Table-owned content file (`assets/data/ui/chart_layout.json`, path provisional) holds authored node positions **per record (`derelict_id`) in chart-local art px** inside the chart safe frame (D.1b), the depth-ring radii, the filament texture reference and a stable per-record **ordinal**; the silhouette glyph is chosen **by `template_id`**. Nothing is derived from `world_slot`, leg data, ticks or distance (WM:1049, ASK-AD-7); `world_slot` is in `web.view` and is never read for layout. A missing record position is a content-lint failure; at runtime a deterministic fallback slot on the outer ring is used and a log line is emitted. Positions are authored cosmetic placements and are lint-checked so no authored distance correlates with any Web Map & Transit cost value.
*Why.* Two derelicts of one template must not overlap; a derived layout is a distance read, and a distance read is a cost number by proxy.
*Layout coordinate space (closed by checkpoint ruling Q7; OQ-CT-11 is closed).* Positions are authored in art px inside a **fixed chart frame** `CHART_FRAME` 288 x 162 art px (`[PLACEHOLDER]`, derived from the smallest spatial canvas, D.1b) and drawn at an integer **chart multiplier** `m` applied to positions only (D.1b); glyphs stay 24x24 at `H`. The **compact layout** (CT-44) applies iff `m < 1`. The ui-programmer's normalised (u, v) coordinates were declined.
*Source.* WM:1049, ASK-AD-7 (WM:1180).

**CT-13. Depth rings, filaments and the only motion.**
*Rule.* The chart field is drawn as concentric **ring bands** darkening toward the centre, ending in a centre disc that is exactly Void Steel `#0B1018`; `CHART_RING_STEPS` (3, `[PLACEHOLDER]`) counts the ring bands. All MVP nodes sit in ring 1, the outermost band, which is Ground `#1C2128` itself; rings 2 and 3 are the horizon, not content, and carry no node, label or tick (WM:1050). The darkening is an **ordering contract** (outer lightest, each step darker than the one outside it by at least `RING_STEP_MIN_CHANNEL` `[PLACEHOLDER]` per channel, centre disc darkest; ring 1 is Ground itself, the disc is Void Steel itself, and the tones between are the D.6 interpolation), not a percentage restated as a value: ART 7.4 and WM:1050 say "about 4 % darker", but the art-director found that reading cannot be satisfied together with "the inner ring reaches Void Steel" and stays near the visibility threshold (flag F-6; wording clarification routed to the art-bible owner; exact tones are derived values in D.6 and Visual/Audio, never new palette entries). `depth_label` and ring numbering carry no progress semantics (WM:1309). Filaments are Hull Grey, 1 art px, branching, non-repeating, no right angles (ART 3.4), **authored texture layers** (not procedural), drifting by whole-art-px offsets on the layer's own UI clock (CT-43) at no more than the ART 7.3 ceiling, **in ring 1 only**; the filaments of rings 2 and 3 are static and denser toward the centre. **Reduced motion** stops the drift and holds its current whole-pixel offset (no snap-back) and turns the bracket relocation into a cut; it does **not** turn fades into cuts (a fade is not spatial motion) *(orchestrator ruling: the art-director's reading, over the ux-designer's "fades become cuts")*. **Reduce flashing** (a separate toggle, PR-13(c)) uses the long end of the ART 7.3 fade band and forbids a hard cut of a large plate. The two toggles together are the union. Everything human on the chart is static.
*Why.* The composition must pull inward (ART 2.11 carried element). `present_time` is frozen under pause (PIPE:265), so the drift cannot use it.
*Source.* ART 2.11, 3.4, 7.3, 7.4, HR-8; WM:1050; PIPE:265; PIPE PR-13(c).

**CT-14. No label of any kind beside a node or the selection.**
*Rule.* No distance, travel time, fuel figure, danger rating, loot estimate, "recommended" marker, name or number sits beside any node or the selection, **except the CT-44 neutral ordinal in the compact layout, which is not a name** (a slot mark with no identity: it never varies with status, `template_id`, kind or depth, and UNSEEN and SEEN rows render the identical key form; AC-CT-112). The only text on the chart is the state word `IN TRANSIT` (CT-15), the layer's own control hints (keycap glyphs) and, in the compact layout only, those neutral ordinals. The spatial map carries no text beside any node, UNSEEN or SEEN.
*Why.* A distance invites "closer = cheaper", a cost number by proxy (P1; WM guard rail 7; OQ-WM-34 default: "No label is required; none is derived"). The art bible's selection row mentions "label and distance beside it"; that wording is superseded by this rule and routed as a clarification to the art-bible owner.
*Source.* WM:1049; OQ-WM-34.

**CT-15. The chart while `transit_state != IDLE` (LEG, ARRIVING, BERTHING; the Pipeline's `TRANSIT_HOLD` included).**
*Rule.* The layer stays openable and is **view-only**. The pick is frozen (CT-8); nothing on the map is focusable or activatable; focus rests on the layer's Back hint (a text-labelled control). The state word `IN TRANSIT` (instrument face) is shown **iff `pod_at == IN_FLIGHT`**: a direct `web.view` read, no other condition. It sits at a **fixed position** inside the chamfered border at the top-left, anchored to the picture rect with the 5 % safe margin (PIPE:88), and is never placed near the origin node, the target node or along the route, because position would read as direction or progress that `web.view` structurally lacks. It appears and disappears in a cut with the data; no ellipsis animation, no dots, no blink. BERTHING shows **no** state word and no progress element (WM:1307: a berth is witnessed by the world). No pod marker is interpolated (WM:1048). **Nothing is drawn from `ChartPick` while non-IDLE**: the chart must never reconstruct the target from its own pick, because the pick resets on restore, is host-local and is wrong in co-op. The planned-sortie route in flight is drawn **only from `web.view.leg_target_id` and `leg_dir`** (WM-36; closes C12 and ASK-CT-3): on an OUT leg the dashed Tungsten line runs HOME to `leg_target_id`; on a BACK leg (`leg_target_id` is HOME, the origin is not published) no route is drawn and the `IN TRANSIT` word stands alone. The chart never calls `web.leg_timing()` (Audio-only, WM:300).
*Source.* WM:1048, :1268, :1307; WM-36; ASK-AD-7.

**CT-16. Layout contract across every reachable canvas.**
*Rule.* The layer is asserted at every reachable HUD canvas `picture / H`, with `H` an integer in `[H_default, 2 * H_default]`, `H_default = max(k, 3)` and `k` the integer **world scale** (PIPE D.4; **zoom never changes the HUD canvas**, PR-12: "H is independent of k and of zoom"). The required canvas rows are: 1080p / 1440p / 2160p `keep` at `H_default`, 640 x 360; 1080p at `2 * H_default`, 320 x 180; Deck 1280x800 `expand` at H 3, 426.67 x 266.67, and at H 6, 213.33 x 133.33; 720p at H 3, 426.67 x 240, and at H 6, **213.33 x 120, the smallest reachable canvas**. Layout is anchor-relative only (PIPE:88, 5 % safe margin) and **every layer-3 rectangle is positioned by anchors 0 or 1 plus integer HUD-px offsets**; a centred element is permitted only if its on-screen width is an even number of screen pixels. The panel is therefore `floor(canvas_w / 3)` HUD px wide, hung from a right anchor of 1, so every edge is an integer screen pixel at every `H` and "a third" holds to within one screen pixel (orchestrator ruling: the ui-programmer's integer rule over the ux-designer's fractional `canvas_w / 3`; D.1 in Formulas). Where the chart multiplier `m` (D.1b) is below 1 the chart switches to the compact layout (CT-44). Nothing scrolls at MVP's node and item count. At MVP only `H_default` is player-reachable (HUD scale is Settings', Alpha), but the contract and its acceptance criteria cover the full range through a forced-`H` test hook.
*Why.* The reflexion log (2026-10-26, "invariant — inner box unsatisfiable at zoom-in fields"): an invariant written against the default must be re-checked whenever a later decision adds reachable states.
*Source.* PIPE PR-12 (PIPE:224), D.4 (PIPE:369-386); `docs/consistency-failures.md`.

#### D. The panel, settings and the notice

**CT-17. The pause panel's content.**
*Rule.* The panel shows the four items of CT-3 and **exactly one save fact**: the **newest-record line**, the `SaveInfo.playtime` of the newest generation, read **once on panel open**, formatted by D.7 and static while the panel is open: no countdown, no "time since", no gate state, no per-write mark (SL UI-2). The panel has **no title and no state word**: it never says "Paused" (in OVERLAY the sim is running, and a word that is false in one mode is a lie on the truth layer). `SaveInfo.depth_label` does **not** appear on the panel (it is on the main-menu Continue line only; SL UI-2 names one recency fact, SL UI-4 names the depth label; the Web Map & Transit row at WM:1309 saying "pause panel / save list" is satisfied by the save list, routed as C13). Panel plate: opaque Ground (HR-2), one chamfered Graphite-lit border, one rivet mark per corner (ART 7.4). The world beneath is frozen exactly as lit, not dimmed, blurred or desaturated (PIPE:265); a click on that world does nothing (CT-41). Panel item labels are in the reading face (ART 7.1 assigns settings and long menu copy to the reading face; panel item labels follow it as menu copy, orchestrator ruling closing ASK-UP-3; a clarification row to the art-bible owner); the playtime numerals are the instrument face.
*Source.* SL UI-2 (SL:998); ART 7.4; PIPE:265; WM:1309.

**CT-18. Display & Comfort page: exactly three controls.**
*Rule.* One page inside the panel's column with three controls: **display mode** (windowed / fullscreen / borderless), **reduced motion** and **reduce flashing**. Defaults are the pipeline hooks' defaults (PR-13; PIPE:301 "MVP exposes the hooks with defaults"). Display mode is a three-choice list whose current choice is marked by a shape plus a word (not colour alone); the two booleans are chamfered switches (ART 7.4) with an on/off **word**. Changes apply immediately and persist (CT-19); there is **no Apply button and no revert countdown** (a countdown would be a timer surface; any revert-on-failure safety is Settings' / an ADR). After a display-mode change the page reads back and shows the **applied** mode (a failed fullscreen request shows windowed); a mode change resizes the window, so `k`, `H_default`, the canvas and the panel width recompute live, the page stays open and focus is retained **by item id**, the bracket does not nudge and no animation plays on the relayout. The page carries only labels, switches and their words; it has no explanatory paragraphs. **Never on any page:** zoom (PIPE:232), rebinding, resolution list, world scale, **HUD scale** (deferred to Settings & Input Rebinding, Alpha: `H_default` already meets the 21-px floor on every display by construction), frame cap, V-sync, audio, the HR-15 ambient step (Lighting's), any true-value overlay (never an accessibility option, ART 7.4). Master volume is OQ-CT-1 (default: none at MVP).
*Why.* The three controls are those an outside playtester needs and that PR-13(c)/(e) makes load-bearing; each is a boolean or enum hook the pipeline already defines. `Window` display-mode behaviour and failure reporting are NOT SOURCEABLE in the engine reference.
*Source.* PIPE PR-13, PR-14 (PIPE:228, 232), PIPE:301; ART 7.4.

**CT-19. Settings store.**
*Rule.* The three comfort values and the `notice_seen` flag live in a host-local, per-install `SettingsStore` seam: **not a kernel block, not a command, not WORLD or PROFILE scope** (PROFILE is per-campaign, SL-9; settings survive New campaign). Nothing in the simulation reads it (HR-6: "no game state may read or alter" anchors, PCI:845). Consumers (Lighting, the Filter, Audio, the Camera) read `reduced_motion` and `reduce_flashing` through the store's read interface and a presentation-only change signal. A missing or unreadable store is treated as first launch with the pipeline's defaults and the notice (CT-20) is shown again (the safe direction); a write failure keeps the values in memory and retries at the next flush, with no player message at MVP. Writes are debounced and flushed on page close, on Save & Quit and on exit (`SETTINGS_WRITE_DEBOUNCE_MS` `[PLACEHOLDER]`, Tuning Knobs); the few-hundred-byte write is on the UI thread and no thread is introduced. The Chart Table registers zero blocks, hooks, commands and mutators (CT-27).
*Source.* KRN CR-16; PCI:845; PIPE:301. **-> ADR candidate (c):** the store mechanism (a `ConfigFile` at `user://settings.cfg` is the working proposal; NOT SOURCEABLE in the engine reference) and its Alpha replacement.

**CT-20. First-launch photosensitivity notice.**
*Rule.* Shown once, before the main menu, on the chart-table register, in the reading face (PIPE PR-13(e), PIPE:228). It offers the two toggles (reduce flashing, reduced motion) inline and one dismiss action; no countdown, no auto-dismiss, no hold, no animated element (the notice is not itself a photosensitive stimulus), and default focus on the first toggle. The dismiss label must not be "Continue" (Save/Load's campaign verb, SL:993); it is a `ct.*` key. Dismissal writes `notice_seen` to the store. Copy is the UX spec's and the narrative director's; this GDD fixes only the behaviour. If the flag is absent the notice appears again; a second showing is identical (no state difference).
*Source.* PIPE PR-13(e); CON outside-playtester gate (CON:483).

#### E. Saving, campaigns and failures

**CT-21. Save & Quit (UI-2 and UI-3 behaviour, verbatim).**
*Rule.* One action calls Save/Load's `save_and_quit()`. **Save/Load's `save_and_quit` itself calls `request_shutdown` after `WRITTEN`, or after an unchanged-state-hash skip that writes nothing (SL-6); the layer calls nothing on that path and never waits on `WRITTEN` as an exit guard.** The item is enabled iff `PauseGate.can_save_quit` (CT-4); otherwise it is disabled (Graphite lit). While the write runs (state `SAVE_QUIT_WAIT`, this GDD's own label and not a Save/Load name) all items are disabled, `chart_table` and `cancel` are ignored, and **nothing is added to the screen**: no spinner, no ring, no bar, no percentage, no toast, no text change (SL UI-2; SL H5; SL-24). If any bound seat is `DOWNED` or `DEAD` (the `DEAD` case is unreachable in the solo MVP, SL-6), nothing is written and the result `SL_QUIT_NO_WRITE_DOWNED` opens **`UI3_PLATE`**, the UI-3 message plate: a string stating that the newest autosave will be used (Continue restores the newest verified generation), **Back** (default focus, non-destructive) and **Quit without saving** (the destructive item, under hold-to-confirm, CT-22). A held or failed write (`SAVE_QUIT_TIMEOUT_MS`, SL, not registered) lands in **`SAVE_FAILED`, which is the panel itself** with the plain word `FAILED` (a word, never colour, never Alarm Red: a save is not an instrument, HR-4) in the newest-record line position: the four items stay in CT-3 order, and Resume, Chart, Display & Comfort and `chart_table` all work (SL:785: "the pause panel stays"). Accepting Save & Quit there opens `UI3_PLATE` with **Retry** (default focus, non-destructive; to `SAVE_QUIT_WAIT`) and **Quit without saving** (hold). Completing the UI-3 hold calls `request_shutdown` once (CR-15) with no write; further input is ignored. `save_status` is a sticky read the layer never clears. `UI3_PLATE` is a full-bleed **message plate** (CT-28) in place of the panel *(orchestrator ruling: UI-3's string is a paragraph and the panel is too narrow for the 45-75 character reading measure at 4 of 6 canvases, ui-programmer finding X1 option (a); OQ-CT-12)*. No death language anywhere (SL:993 list).
*Source.* SL-6 (SL:195); SL UI-2, UI-3 (SL:998-999); SL H5, SL-24.

**CT-22. Hold-to-confirm gesture (closes SL OQ-32 by citation).**
*Rule.* Destructive actions (UI-3 quit without saving; UI-5 delete) fill a **linear bar with a needle-mark**, never a radial wipe (ART 3.4, 7.3), over `CONFIRM_HOLD_S` (a Player Controller-owned registry constant; value and range in the Tuning Knobs "cited, not ours" rows; the two-tap option (PCI:835) is an Alpha accessibility option *(provisional — Accessibility Settings undesigned; no id is coined here)*); the action fires on the first frame the held time reaches it, not on release. Default focus on a screen with a destructive action is its non-destructive item. Release, `cancel`, `chart_table`, the pointer leaving the item, or window focus loss snaps the bar to empty in the same frame and **nothing else happens**; a new gesture needs a new press edge. The fill is a monotonic fill on the layer's UI clock (CT-43), not a flash (HR-13), and it is function rather than decoration so reduced motion does not remove it. Formula D.2.
*Clarification.* PCI:656 says "hold timers (confirm bar, name reveal) are frozen with the pause". That sentence concerns the Adapter's **in-world** channel timers; the UI-3 bar must run while the kernel is `PAUSED`, because the Chart Table is the only thing awake then. Routed to the Player Controller as a wording clarification (C15).
*Source.* SL UI-3, UI-5; ART 7.3, 7.4, HR-14; registry `CONFIRM_HOLD_S`; PCI:835.

**CT-23. Main menu (UI-4).**
*Rule.* The boot menu has three items in fixed order: **Continue, New campaign, Quit**. Continue is shown disabled (not hidden) when `campaign_info()` returns nothing; default focus is Continue if enabled, else New campaign. Continue shows exactly Save/Load UI-4's SIM facts as words: playtime (D.7), crew count, kind (`AUTO` / `EVENT` / `QUIT`), `RISKY` (as a word when set) and `depth_label` (`RING <n>`). **`RING <n>` is the ring of the newest record's pod / home position at the time that record was written; it is never a maximum, a historical "deepest reached" or any running aggregate over the campaign** (G5; a later record at a shallower ring shows the shallower ring; AC-CT-113). No thumbnail, no completion figure, no wall-clock "last played" beyond file metadata. The UI-7 line is shown under Continue on menu entry iff `campaign_info()` reports the newest generation failed verification (ASK-CT-5). One Continue serves the one logical autosave: the generation chain is invisible and never chosen (UI-4's own wording uses another noun; the stale-token lint allow-list carries it). Vocabulary is Save/Load's only (SL:993). `new_game(world_seed)` takes its seed from Save/Load's boot rule; the menu has no seed field (OQ-CT-6). The backdrop reuses the depth-ring plate and the ring-1 filament sheet with **no nodes** (a node would imply a save state or leak a Continue fact); fallback plain Ground. No title art, logo animation or tagline.
*Source.* SL UI-4 (SL:1000), SL-21 (SL:261); ART 2.11.

**CT-24. New campaign, UI-5, no beam picker.**
*Rule.* New campaign with no existing campaign calls `new_game(world_seed)` directly (via CT-42). With an existing campaign it opens the **UI-5 screen** itself: default action **Retire** (reversible; the file stays on disk), **Delete** as a separate explicit action on the same screen under hold-to-confirm (CT-22), and Back. Retire and delete call `retire_campaign(campaign_id)` / `delete_campaign(campaign_id)`; the retired state `RETIRED` is caused by this layer (SL T-9, SL:430). The destructive item looks like every other list item (no red, no warning mark) and is distinguished by the bar only. No campaign list, Records page or un-retire screen at MVP (OQ-CT-4: "reversible" is by file only until an Alpha screen). No beam-shape picker: New campaign calls nothing beyond `new_game`; seat defaults are Suit & Equipment's (SE-9: seat 1 `BEAM_WIDE`); a `NewCampaignOptions` seam is reserved for the VS-B picker (SE:189 calls it optional).
*Source.* SL UI-5 (SL:1001); SE-9 (SE:189).

**CT-25. Load-failure screens (boot-flow).**
*Rule.* UI-8 (`SL_LOAD_NO_LOADABLE_SAVE`: a plain statement that no readable record exists, with New campaign and Quit) and UI-9 (incompatible build: message, Quit and New campaign, never overwrite or repair) are **boot-flow screens**: no pause is requested because the kernel is not `RUNNING` when the failure is reported (SL:1004 says "Chart Table issues the pause"; that wording is stale, C7). UI-7 (corrupt newest record) is one plain ship-log line under the Continue entry **iff** `campaign_info()` reports that the newest generation failed verification (that field is provisional, ASK-CT-5); otherwise the line is HUD & Suit Layer's on the first `RUNNING` frame (SL:1003 lists both owners; this GDD owns the main-menu instance only). New campaign from UI-9 calls `new_game(world_seed)` through CT-42 without opening UI-5; Save/Load creates a different directory and the `READ_ONLY` campaign is never retired, deleted or written (SL:752, SL T-9). Placeholder strings are Save/Load's (SL:1003-1005); final copy is this GDD's `ct.*` table with UX and narrative sign-off. No icon, no red, no dialog chrome beyond the standard Ground plate.
*Source.* SL UI-7..UI-9; SL:770.

**CT-26. FAULTED in release.**
*Rule.* The destination is the boot-flow **main menu** (UI-4) with Continue offered, focus on Continue: in `FAULTED` the kernel never writes or autosaves over an existing save (KRN:151), so the newest verified generation is intact. Whether a one-line notice accompanies it is OQ-CT-5 (routed to the kernel and Save/Load); this GDD invents no string. Dev builds halt with the Debug Overlay (not ours). PIPE:269 says release "returns to the chart table" and KRN:151 says "returns to the menu"; the wording is reconciled by this rule and routed (C14).
*Source.* KRN:151, :940; PIPE:269.

**CT-42. The load sequence: black before work.**
*Rule.* Accepting Continue, New campaign, Retire (then New) or Delete (then New): (1) all items are disabled and the layer's plate is removed in the same frame; (2) layer 1 black is shown (PIPE row 1: shown in `TRANSIT_HOLD` and `BOOTING/LOADING`); (3) at least one frame at full black is presented; (4) only then is `continue_campaign()` / `new_game(world_seed)` called; (5) the first `RUNNING` frame is a camera `CUT` (PIPE:264). Nothing is drawn between (2) and (5): no text, no tip, no bar, no spinner (SL UI-6). If the call returns UI-7, UI-8 or UI-9, the layer re-enables itself on the boot screen with the plain message.
*Why.* A restore called with a visible, focused menu would freeze that menu for the whole restore, and a long non-pumping call can mark the window "not responding" on Windows (NOT SOURCEABLE). Whether the restore is one blocking call or chunked across frames is **ASK-CT-7 to Save/Load**; the rule only assumes one presented black frame, never progress.
*Source.* SL UI-6; PIPE:99, :264, :278.

#### F. Registrations, strings and access

**CT-27. Zero registrations; submits nothing.**
*Rule.* The chart table is an `INPUT_SURFACE` (KRN:111; PIPE:188; PCI:147) that **submits zero commands at MVP**; the class is kept for VS-B guest pause / save requests (KRN CR-15). It registers **no state block, no hook, no command, no mutator**, and introduces no `allowed_when_paused` type (so KRN CR-16's paused-allowed list is unchanged). It reads only `web.view`, `SaveInfo`, `save_status`, `restore_reason`, `campaign_info()` / `list_saves()` and its own host-local state; never `web.leg_timing()`. It calls only the kernel control plane (`request_pause`, `request_resume`, `request_shutdown`) and Save/Load's host-local API (`new_game`, `continue_campaign`, `save_and_quit`, `retire_campaign`, `delete_campaign`). KRN:715 offers "a chart-table command with a path array" as a `MAX_PAYLOAD` example: that example is stale (C3) because this layer submits no command.
*Source.* KRN CR-15; SL-21; WM:300; SL-17 (the zero-block precedent).

**CT-28. Strings, faces and fit.**
*Rule.* All user-visible strings are keys in a closed `ct.*` table owned here (`tr()`), ship-log register, no composed fragments (a value token such as a time is allowed inside one whole string), no exclamation marks, no emoji, Save/Load's vocabulary only. **Instrument face** (bitmap, integer `H`): a closed string set, declared in the `ct.*` table, of `IN TRANSIT`, numerals, keycap letters and short state words; **reading face** (vector, continuous scale): panel item labels, settings copy, the notice, messages and every sentence (ART 7.1). Type and contrast floors are ART 7.1's, by name (menu body, section heading, title, label and caption floors at 1080p / 1440p; text 4.5:1, glyphs 3:1). Each key carries a **fit class**: `LABEL_1` (one line at every canvas row), `LABEL_2` (up to two lines) or `PARAGRAPH` (message plates only, full-bleed Ground sized to the text with the ART 7.1 line-length measure). A lint measures each key with the real faces at each canvas row of CT-16 and fails a `LABEL_1` that overflows; a menu label is never truncated or ellipsised (a clipped "Save & Qu" on the truth layer would read as a lie) and never wraps to a third line.
*Source.* SL:127, :993, :1008; ART 7.1.

**CT-29. Audio: requests only.**
*Rule.* This layer owns and specifies no sound. It requests from Adaptive Audio a UI-bus focus-move cue and a confirm cue (AA:562: "menu and map sounds are UI-bus, ducked out of the world mix on open; pause suspends all sim-derived sounds"); nothing for open, save or load. Ducking on open and suspension of sim sounds on pause are Adaptive Audio's. Silent menus are acceptable at MVP. No sanity or Attention effect on menu audio.
*Source.* AA:562, AA:1086; HR-6. *(provisional — Adaptive Audio owns the cues.)*

**CT-30. Accessibility at MVP (structural).**
*Rule.* Now: opaque Ground plates (HR-2), focus as a shape (HR-11), ART 7.1 type and contrast floors (HR-12), gamepad completeness of every flow including the chart snap and every destructive action (HR-14), no colour-only state (SL:1010), a text label on every interactive control. At Alpha (Accessibility Settings / Settings): AccessKit wiring, hold-duration and two-tap, HUD opacity floor, reading-face scale, HUD scale. `design/accessibility-requirements.md` **does not exist**; the project's accessibility tier is unrecorded (OQ-CT-8).
*Source.* ART HR-2, HR-11, HR-12, HR-14; SL:1010.

#### G. Negative requirements (each its own rule, each with an acceptance criterion in Acceptance Criteria)

**CT-31. No Attention surface.** *Rule.* No Attention indicator, meter, tier, hint or glyph on any chart, panel or menu surface, under any setting; depth rings are shown as place, not as attention; no Chart Table surface displays or sonifies the Attention level, tell progress or a Horror's proximity (AA:1515); no per-write autosave mark (its absence would be a hidden Attention meter, SL:965). *Source.* AT:1373, AC-AT-61, AC-AT-66; AA:1515 (Adaptive Audio UI Requirements, "depth rings are place, not attention"); ART 7.4.
**CT-32. No horror element (Pillar 3).** *Rule.* No horror, horror position, threat bar, proximity indicator, direction, off-screen arrow, countdown, "last seen" pin or target ring on any chart or menu surface, **under any accessibility setting**. *Source.* HE:377, :1353, :1358, AC-HE-119; Pillar 3 (Attention is shown in the world, never as a meter).
**CT-33. No loot value, no remaining-salvage count.** *Rule.* The Chart Table picks the target and shows only what the crew lit, and never reveals; this confirms Boarding's provisional contract (BS:334, Dependencies row 17b); no loot value, no remaining-salvage count (BS:334; BS:1204's wording "and the fuel the hop will cost" is overruled by CT-35, C1). *Source.* BS:334, AC-BS-47c (BS:1427), WM-28, ART 7.4.
**CT-34. No sanity indicator.** *Rule.* No sanity meter, bar, ring, number, icon or colour on any surface of this layer, including under accessibility settings; save and pause surfaces are never a hallucination class. *Source.* SN:1350, :1357, AC-SN-83, SL H6.
**CT-35. No fuel, time, ETA, danger or "recommended".** *Rule.* No fuel figure, travel time, ETA, danger rating or "recommended" marker anywhere on the chart; `web.view` structurally lacks the data and the chart never reads `web.leg_timing()`. Boarding's row BS:1204 ("the Chart Table shows what the crew lit and the fuel the hop will cost") conflicts and **Web Map & Transit governs** (WM:1049, WM-35, AB-5; routed to Boarding to reword). *Source.* WM:1049, :1299, :1305; WM-36; C1.
**CT-36. No save decoration.** *Rule.* No per-write autosave mark, thumbnail, spinner, percentage, death screen, "progress saved" toast or loading text or tip. Load is layer-1 black via `TRANSIT_HOLD` (UI-6), then a camera `CUT`. *Source.* SL H5, SL:965, SL-24 (SL:267); SL UI-6; ART 2.10.
**CT-37. No inventory, weight or loadout screen.** *Rule.* No inventory, weight readout or loadout screen. The Sanity evidence list (SN:1350, SN:1357) names "the inventory" and "the death/restore screen"; neither exists (IDB R8; SE:1289; SL; ART), routed to Sanity (C8). *Source.* IDB R8 (IDB:1065, :1320); SE:1289, :1317.

#### H. Reserved slots and the rules added by the specialists

**CT-38. Reserved slots (no stub art, no layout space at MVP).**

| Slot | Owner and phase | Constraint carried |
|---|---|---|
| Crew beam-shape glyphs on the chart; left-edge text chat | Co-op Session / Chat (VS-B) | ART 7.4, 7.5; risk K1 |
| Knowledge Codex annotations | Codex (VS) | each annotation needs a stated effect on play (IDX:47) and is never a lore dump on the truth layer; PROFILE scope |
| Post-loss UI read at the chart table | Failure Contract & Re-wake (VS-A) | ART 7.4 "Loss" |
| Onboarding flow after the notice | Onboarding (VS) | `POD_NOT_READY` reserved in Web Map & Transit |
| Beam-shape picker | VS-B | `NewCampaignOptions` seam |
| Records page, un-retire | Alpha | OQ-CT-4 |
| Full PR-14 settings UI, rebinding, HUD scale | Settings & Input Rebinding (Alpha) | MVP hooks with defaults |

**CT-39. Narrow truth.** *Rule.* The layer displays only facts that no hallucination class can lie about. Every field it draws is listed with its source (`web.view`, `SaveInfo`, `save_status`, `restore_reason`, `campaign_info()`, the `SettingsStore`) and with the statement that no hallucination class touches it; a fact that any class can falsify (Graph state, entity presence or position, salvage contents or value, a crewmate's rendered form or location) never enters the layer. *Why.* A true copy of a falsifiable fact on a free, pausable menu would make that lie free to disprove, and P4 requires that checking cost time, risk or trust. Review test: *if a hallucination could make the world disagree with this field, the field does not belong here.* *Source.* CON:261-265; Player Fantasy "Under hallucination".

**CT-40. Constant drift, constant depth.** *Rule.* The filament drift step and the ring tones are independent of every simulation read and of Attention, depth, sanity and session time; the only inputs to the drift are the layer's UI clock and the reduced-motion toggle. *Why.* P3 names "filament movement" as an in-world Attention sign; a chart filament that modulated would be a meter. *Source.* AT:1373; ART 2.11.

**CT-41. Input hygiene around the irreversible press.**
*Rule.* (1) **Fresh press edge.** When the layer is opened by a console press, every activation on it requires a press edge that begins after the open frame; the open press's release, its auto-repeat and any carried-over held state are ignored. (2) **The closing activation is consumed.** When `interact` on Resume, `chart_table` or `cancel` at depth 1 closes the layer, the layer swallows that press and its release; gameplay sees no `interact` edge until a fresh press. (3) **Echo ignored.** The Esc / Start toggle ignores key echo, so a held Esc does not flicker the layer. (4) **No click-outside.** A click on the frozen world outside the panel does nothing (the world is a held frame, and a click there would read as an in-world action). (5) **Focus loss.** Window focus loss aborts any hold fill and clears navigation state; on return the layer stays open with the previous focus and the kernel state is unchanged.
*Why.* The next `interact` edge at the pod console is `pod_hop`, the irreversible act (WM:1225). PCI:656 resends intents on resume and keeps a hold-mode channel only if the button is physically down, so this is a requirement on the layer, not a Controller change. Key-echo detection and focus-out notifications are NOT SOURCEABLE in the engine reference (ADR candidate (e)).
*Source.* WM:1225; PCI:656-658.

**CT-43. The layer's UI clock.**
*Rule.* Layer 3 keeps its own presentation-time accumulators driven by the render delta, independent of the kernel state, of `present_time` (frozen under pause, PIPE:265) and of `SceneTree.paused` (ADR candidate (a)): one per hold gesture (starts at press, resets on release, abort or focus loss) and one monotonic drift accumulator that persists across opens, so filaments do not rewind on re-open. Every delta is clamped to `UI_DT_MAX_MS` `[PLACEHOLDER]` so a hitch cannot complete a hold or jump the drift. The clock is presentation time (KRN CR-11) and never enters the simulation; no wall clock enters any rule except these presentation timers.
*Source.* KRN CR-11; PIPE:265.

**CT-44. The compact layout and stable order.**
*Rule.* The layout is **compact iff the chart multiplier `m` (D.1b) is below 1**, i.e. the canvas is narrower than `COMPACT_LAYOUT_MIN_CANVAS_W` (320 HUD px, `[PLACEHOLDER]`, **derived** as `CHART_FRAME` width / 0.9, with the height twin 180 = 162 / 0.9, so it moves with the frame) or shorter than its twin; it is chosen at open and on canvas change, never mid-animation. Then the spatial map is replaced by a **vertical node list** in the ART 7.4 menu grammar: one column of chamfered-rectangle items, Ground interiors, 1-px Graphite-lit borders, the bracket and bezel thickening for focus, each item carrying its 24x24 node glyph at left (the unknown tick drawn inside the silhouette exactly as on the map) and a **neutral positional ordinal** at right (a slot mark, below). The ring plate's Ground, border and rivets remain; rings and filaments are not drawn. The planned-sortie relation is a Tungsten dashed rule under the picked row (no new asset, no fill). The `IN TRANSIT` word keeps its fixed top-left position. **The compact label is a neutral positional ordinal and carries no identity.** It is one whole `ct.*` key per authored layout slot (`ct.compact.slot_<n>`, `n` = the record's position in the layout file, HOME included; no composed fragment such as a word plus a number, CT-28), rendered as the plain ordinal for that slot. It must not be a name, a dialect, a template word or a status word, and it never varies with `status`, `kind`, `template_id`, depth ring or map distance: **an UNSEEN row and a SEEN row render the identical key form**, and a record's key does not change when it becomes SEEN, BOARDED, `berthed` or ORPHAN. The key is bound to the layout slot and is not renumbered when an ABSORBED record leaves the list (a gap in the ordinals is a fact the list already shows, and a stable slot mark lets the player remember which row is which). The ordinal order is lint-checked like positions (no correlation with any Web Map & Transit cost; AC-CT-112). This is the single permitted exception to G6 and CT-14 ("except the CT-44 neutral ordinal in the compact layout, which is not a name"); it is **not** precedent for any further label. WM:1044 (UNSEEN "no dialect, no tick, no label beyond what selection shows") is read as forbidding identity, which a slot mark does not carry (text-label rule of CT-30; **provisional — ASK-CT-10 to Web Map & Transit**). **Order is a ranking proxy and is therefore fixed:** the list is in stable authored order, HOME first, derelicts in layout-file order; it is never sorted by status, distance, fuel, danger, recency or seen/unseen; an ABSORBED record leaves the list (it is part of HOME now), an ORPHAN stays in its slot, non-selectable; the relative order of the remaining rows never changes. Rows are never truncated; if the list cannot fit, the canvas minimum was mis-set.
*Source.* ART 7.4; WM:1044, :1050; CT-12, CT-16.

**Snap-to-node (the chart's own navigation, D.3).** On the spatial map, a direction press from the focused node moves focus to the nearest focusable node whose bearing falls inside the pressed direction's sector (`SNAP_SECTOR_DEG` `[PLACEHOLDER]`), widening to a half-plane if the sector is empty, else staying; no wrap, no shake, no flash; every focusable node is reachable from every other; focus order is geometry only, never ranked, and default focus is HOME. Distances used for ordering are layout distances and are never displayed (WM:1049 and ASK-AD-7 untouched). Formula, variable table and worked example in D.3 (Formulas). Stick navigation is edge-triggered at the Controller's deadzone with auto-repeat (`UI_NAV_REPEAT_MS` / `UI_NAV_REPEAT_RATE_MS` `[PLACEHOLDER]`).

**ADR candidates (not decided here).** (a) pause / UI process mode: kernel `PAUSED` versus `SceneTree.paused` and layer-3 `process_mode`; (b) layer-3 ownership, the shared component library (theme, bracket, keycap, Ground plate), the two-scale mechanism for the bitmap and reading faces, and textured-quad drawing of routes and seams; (c) the host-local settings store and its Alpha replacement; (d) the pick-to-console submit seam; (e) menu input actions for the `chart_table` context, key-echo and focus-loss handling; (f) black-before-work load sequencing with Save/Load.

### States and Transitions

The layer is a small state machine on **layer 3**, plus a second axis, the `PauseGate` mode. It owns no simulation state: every state below is host-local presentation state (CT-27), none is in a snapshot, and after any `request_restore` the machine is rebuilt from CLOSED (TX-27). The kernel column is shown for SOLO; in OVERLAY every in-session cell reads `RUNNING`. **No guard in either table reads the kernel column**; guards read `web.view`, `save_status`, `SaveInfo` / `campaign_info()`, `PauseGate` (`mode`, `can_save_quit`) or host-local state (CT-4).

#### States

| State | Entry | Exit | Transitions (see the table below) | Who may cause them |
|---|---|---|---|---|
| `BOOT_NOTICE` | boot with no `notice_seen` in the store, or an unreadable store (CT-19, CT-20) | dismiss | TX-1 | the player (dismiss, or the two toggles inline); the store (flag absent) |
| `MAIN_MENU` | boot with the notice seen; dismiss; FAULTED in release (CT-26); `cancel` from UI-5 | Continue; New campaign; Quit; a load-failure result | TX-3..TX-6, TX-11, TX-12, TX-30 | the player; Save/Load (failure results); the kernel (FAULTED return) |
| `UI5_NEW_OVER_EXISTING` | New campaign with an existing campaign (CT-24) | Retire; Delete (hold); `cancel` | TX-7, TX-8, TX-12 | the player |
| `UI8_NO_LOADABLE` | `continue_campaign()` returns `SL_LOAD_NO_LOADABLE_SAVE` (CT-25) | New campaign; Quit | TX-5, TX-6 (when a campaign exists), TX-30 | the player; Save/Load (result) |
| `UI9_INCOMPATIBLE` | `continue_campaign()` returns the incompatible-build result (CT-25) | New campaign (TX-5, never via UI-5); Quit (TX-30) | TX-5, TX-30 | the player; Save/Load (result) |
| `LOAD_HOLD` | the load sequence (CT-42) step 1 | the first `RUNNING` frame (a camera `CUT`) or a failure result | TX-4, TX-5, TX-7, TX-9, TX-11 | the player (the accept that started it); Save/Load (result); not this layer's content: layer 1 black is the pipeline's, with no text |
| `CLOSED` | in-session `chart_table`/Resume/`cancel` at depth 1; the first `RUNNING` frame; any restore | `chart_table`; a no-pick console press | TX-13, TX-14, TX-18 | the player (`chart_table`); the Input Adapter's console seam (CT-10); a restore closes any other state into it (TX-27) |
| `PANEL` | `chart_table` from CLOSED; `cancel` from CHART, DISPLAY_COMFORT or a message plate | Chart; Display & Comfort; Save & Quit; Resume; `chart_table`; `cancel` | TX-15..TX-17, TX-19..TX-21, TX-24 | the player |
| `CHART` | Chart from PANEL with `transit_state == IDLE`; a no-pick console press (TX-18) | `cancel`; `chart_table`; `transit_state` leaving IDLE | TX-15, TX-16, TX-18, TX-19, TX-28 | the player; `web.view` (a `transit_state` change) |
| `CHART_VIEW_ONLY` | Chart from PANEL with `transit_state != IDLE`; TX-28 | `cancel`; `chart_table`; `transit_state` returning to IDLE while open (TX-29, to CHART with focus on HOME) | TX-15, TX-19, TX-29 | the player; `web.view` |
| `DISPLAY_COMFORT` | Display & Comfort from PANEL | `cancel`; `chart_table` | TX-15, TX-19 | the player; the store (applied-mode read-back; live relayout, focus kept by item id) |
| `SAVE_QUIT_WAIT` (this GDD's label) | Save & Quit accepted, `can_save_quit`, and the call not refused; Retry on `UI3_PLATE` (TX-32) | Save/Load's `save_and_quit` ends the process (SL-6), or `save_status` `FAILED` / the held-write timeout | TX-22, TX-23 | Save/Load (`save_and_quit`, `save_status`); the player cannot cause or abort it (input ignored, CT-21) |
| `SAVE_FAILED` | `save_status == FAILED` or the held-write timeout (TX-23). It is the PANEL with the plain word `FAILED` in the newest-record line position; the four items stay in CT-3 order | Resume; Chart; Display & Comfort; `chart_table`; Save & Quit (to `UI3_PLATE`) | TX-15..TX-17, TX-20, TX-25 | the player; Save/Load (`save_status`) |
| `UI3_PLATE` | Save & Quit refused with `SL_QUIT_NO_WRITE_DOWNED` (TX-24); Save & Quit accepted from `SAVE_FAILED` (TX-25) | `cancel` to PANEL (TX-19); `chart_table` to CLOSED (TX-15, SOLO resume); Retry (where offered) to `SAVE_QUIT_WAIT` (TX-32); press-and-hold Quit without saving to `HOLD_CONFIRM` (TX-31) | TX-15, TX-19, TX-25, TX-31, TX-32 | the player |
| `HOLD_CONFIRM` (sub-state) | press-and-hold on the destructive item of `UI3_PLATE` (TX-31) or `UI5_NEW_OVER_EXISTING` (TX-8) | fill reaches 1 (TX-9, TX-26); release / `cancel` / `chart_table` / focus loss returns to the screen that owns the item (TX-10) | TX-8..TX-10, TX-26, TX-31 | the player only (the layer's UI clock advances the fill, CT-43) |

The `PauseGate` axis: **SOLO (MVP)** calls `request_pause` on CLOSED to PANEL or CHART (TX-13, TX-18) and `request_resume` on any in-session state to CLOSED (TX-15). **OVERLAY (VS-B, reserved)** makes no kernel call (TX-14); `can_save_quit` is Session / Save-Load's. Every other row is identical in both modes.

#### Transitions

| ID | From | To | Trigger | Guard (never reads kernel state) | Effect |
|---|---|---|---|---|---|
| TX-1 | `BOOT_NOTICE` | `MAIN_MENU` | dismiss | none | write `notice_seen` to the store |
| TX-2 | (boot) | `BOOT_NOTICE` | boot | store has no `notice_seen`, or is unreadable | show the notice |
| TX-3 | (boot) / FAULTED in release | `MAIN_MENU` | boot, or the kernel returning to the menu | notice seen | CT-26; focus on Continue if enabled |
| TX-4 | `MAIN_MENU` | `LOAD_HOLD` | Continue | `campaign_info()` non-empty | CT-42 sequence, then `continue_campaign()` |
| TX-5 | `MAIN_MENU`, `UI8`, `UI9` | `LOAD_HOLD` | New campaign | from `MAIN_MENU` or `UI8`: no existing campaign (otherwise TX-6); from `UI9`: always (the `READ_ONLY` campaign is never offered for retire or delete; New campaign creates a different directory, SL:752, SL T-9) | CT-42 sequence, then `new_game(world_seed)` |
| TX-6 | `MAIN_MENU`, `UI8` | `UI5_NEW_OVER_EXISTING` | New campaign | an existing campaign | CT-24 |
| TX-7 | `UI5` | `LOAD_HOLD` | Retire | none | `retire_campaign(campaign_id)`, then the TX-5 sequence |
| TX-8 | `UI5` | `HOLD_CONFIRM` | press-and-hold Delete | none | D.2 starts on the UI clock |
| TX-9 | `HOLD_CONFIRM` (UI-5) | `LOAD_HOLD` | fill reaches 1 | none | `delete_campaign(campaign_id)`, then the TX-5 sequence |
| TX-10 | `HOLD_CONFIRM` | the screen that owns the destructive item (`UI3_PLATE` or `UI5`) | release, `cancel`, `chart_table` (CT-5), focus loss | fill < 1 | bar to empty in the same frame; nothing else |
| TX-11 | `MAIN_MENU` | `UI8` / `UI9` | `continue_campaign()` returns `SL_LOAD_NO_LOADABLE_SAVE` / the incompatible result | none | boot-flow screen; no pause requested |
| TX-12 | `UI5` | `MAIN_MENU` | `cancel` | none | focus returns to New campaign |
| TX-13 | `CLOSED` | `PANEL` | `chart_table` | `PauseGate.mode == SOLO` | `request_pause`; the panel appears at once; default focus Resume |
| TX-14 | `CLOSED` | `PANEL` | `chart_table` | `PauseGate.mode == OVERLAY` | no kernel call (reserved) |
| TX-15 | `PANEL` (`SAVE_FAILED` included), `CHART`, `CHART_VIEW_ONLY`, `DISPLAY_COMFORT`, `UI3_PLATE` | `CLOSED` | `chart_table`, or Resume | not `HOLD_CONFIRM` (TX-10) and not `SAVE_QUIT_WAIT` | SOLO: `request_resume`; pick kept (CT-7); the closing press and its release are consumed (CT-41) |
| TX-16 | `PANEL` | `CHART` | Chart | `view.transit_state == IDLE` | revalidate pick (CT-8); focus on the pod's node |
| TX-17 | `PANEL` | `CHART_VIEW_ONLY` | Chart | `view.transit_state != IDLE` | pick frozen; nothing drawn from it; focus on Back |
| TX-18 | `CLOSED` | `CHART` | pod console press | `view.pod_at == HOME_ID`, `view.transit_state == IDLE`, `ChartPick` NONE or invalid | open the chart; focus on HOME; SOLO requests the pause as in TX-13; no refusal, no command (CT-10) |
| TX-19 | `CHART`, `CHART_VIEW_ONLY`, `DISPLAY_COMFORT`, `UI3_PLATE` | `PANEL` | `cancel` | not mid-fill | none |
| TX-20 | `PANEL` | `DISPLAY_COMFORT` | Display & Comfort | none | focus on the first control |
| TX-21 | `PANEL` | `SAVE_QUIT_WAIT` | Save & Quit | `PauseGate.can_save_quit` and `save_and_quit()` did not return `SL_QUIT_NO_WRITE_DOWNED` | all items disabled; nothing added to the screen |
| TX-22 | `SAVE_QUIT_WAIT` | (exit) | Save/Load's `save_and_quit` ends the process (SL-6) | none | Save/Load's `save_and_quit` calls `request_shutdown` after `WRITTEN` or an unchanged-hash skip (SL-6); the layer calls nothing |
| TX-23 | `SAVE_QUIT_WAIT` | `SAVE_FAILED` | `save_status == FAILED`, or the write held past `SAVE_QUIT_TIMEOUT_MS` (SL, not registered) | none | the panel returns with the plain word `FAILED` in the newest-record line position; the four items unchanged; Resume, Chart, Display & Comfort and `chart_table` work |
| TX-24 | `PANEL` | `UI3_PLATE` | Save & Quit refused with `SL_QUIT_NO_WRITE_DOWNED` | `save_and_quit()` returned `SL_QUIT_NO_WRITE_DOWNED` | the UI-3 message plate: the string states the newest autosave will be used; Back (default focus) and Quit without saving (hold) |
| TX-25 | `SAVE_FAILED` | `UI3_PLATE` | Save & Quit accepted | none | the UI-3 message plate with Retry (default focus) and Quit without saving (hold) |
| TX-26 | `HOLD_CONFIRM` (entered from `UI3_PLATE`) | (exit) | fill reaches 1 | none | the layer calls `request_shutdown` once (CR-15); no write (the only shutdown call on the Save & Quit surface) |
| TX-27 | any in-session state | `CLOSED` | any `request_restore` | none | `ChartPick := NONE` (CT-8); the layer closes; no animation |
| TX-28 | `CHART` | `CHART_VIEW_ONLY` | `view.transit_state` leaves IDLE while open | none | pick frozen; focus to Back |
| TX-29 | `CHART_VIEW_ONLY` | `CHART` | `view.transit_state` returns to IDLE while open | none | pick revalidated (it was reset on arrival, CT-8); focus on HOME |
| TX-30 | `UI8`, `UI9`, `MAIN_MENU` | (exit) | Quit | none | on the FAULTED-return menu `request_shutdown` is legal and is called; on a boot-flow screen the kernel is not in a state that lists `request_shutdown` (KRN lists RUNNING, PAUSED, FAULTED), so the Quit path is ASK-CT-11 (degraded: boot code ends the process with no kernel call) |
| TX-31 | `UI3_PLATE` | `HOLD_CONFIRM` | press-and-hold Quit without saving | none | D.2 starts on the UI clock |
| TX-32 | `UI3_PLATE` | `SAVE_QUIT_WAIT` | Retry | the plate was entered from `SAVE_FAILED` and `PauseGate.can_save_quit` | as TX-21 |

**Reachability and closure checks (for the acceptance criteria).** Every state has an exit that does not depend on a hold: `HOLD_CONFIRM` exits by release; `UI3_PLATE` exits by `cancel`, `chart_table`, Retry or Back; `SAVE_QUIT_WAIT` exits when Save/Load's `save_and_quit` ends the process (SL-6) or by `FAILED` / the timeout of TX-23 (a Save/Load guarantee); the boot-flow screens each offer Quit. There is no state with no player input and no external exit except `SAVE_QUIT_WAIT` and `LOAD_HOLD`, both bounded by Save/Load results. A restore from any state lands in CLOSED (TX-27), and the first `RUNNING` frame after `LOADING` is a camera `CUT` (PIPE:264).

**Order of presentation on boot.** TX-2 (notice) precedes TX-3 (menu); the menu precedes any session; the notice is never shown mid-session. FAULTED in release (TX-3) skips the notice when it has been seen.

### Interactions with Other Systems

Status words follow the Glob of `design/gdd/` at authoring: **designed** = a GDD file exists headed Designed; **in-flight** = a file exists in draft or a scratch directory exists with no contract; **undesigned-provisional** = no file. Line numbers are as read on 2026-10-26/27 and drift while sibling agents edit; rule and AC ids are given with them. **Downstream dependents among the systems designed in this run: none.** Settings & Input Rebinding (Alpha) depends on this GDD (IDX:210) and is undesigned. Each row quotes the neighbour's own row where one exists; "Consistent" means this GDD's behaviour matches the quote, and any mismatch is a numbered conflict in the Cross-References table or Open Questions.

#### Upstream: data this layer reads, calls it makes

| Neighbour (status) | Data in | Data out | Interface (exact names) | Interface owner | The neighbour's row, quoted | Consistent? |
|---|---|---|---|---|---|---|
| **Web Map & Transit** (designed, this run; names marked provisional by WM) | `web.view` (WM-36): `nodes[]` per record {`derelict_id`, `template_id`, `kind`, `ring`, `world_slot`, `status`, `unknown_seen`, `loaded`, `berthed`}, `pod_at` (a record id or `IN_FLIGHT`), `transit_state` (IDLE, LEG, ARRIVING, BERTHING), `destination_ready`. `leg_target_id` and `leg_dir` (OUT, BACK; the persisted target_id and leg_dir while `transit_state` is LEG or ARRIVING, else NONE; endpoint identity only, never progress; on a BACK leg `leg_target_id` is HOME and the origin is not published). `world_slot` is carried but never read for layout (CT-12). `SaveInfo.depth_label` as a string only *(provisional, registry `SaveInfo`)* | none. The pick reaches WM only through the console press (CT-10); the chart never writes | `web.view` (read-only); never `web.leg_timing()` (Audio-only, WM:300); `pod_hop{target_id}`, `berth{derelict_id}`, `unberth` are submitted by the console press, not by this layer | WM owns `web.view`, the drawing grammar (WM:1036-1050) and the reasons (`TARGET_UNKNOWN`, `TARGET_ABSORBED`, `TARGET_BERTHED`, `BERTH_PENDING`); this GDD owns layout, pick, snap and route drawing | WM:420 (Interactions row 4): "It **displays only**; it writes no simulation state; the pick is host-local UI, never simulation, never persisted; the confirm is the console press after resume, not a paused simulation write (KRN CR-16, KRN OQ 6); it never reveals (ART 7.4), never shows Attention or sanity, never assumes the pick happens while paused. Snap-to-node and route drawing are its. *(provisional — Chart Table undesigned)*". WM:888 (Dependencies row): "Hard (its whole content) ... owes: displays only, never reveals, never assumes pause (IDX:95)". WM:1319: "the chart's gamepad snap is node to node (HR-14), Chart Table's." | Yes. The former gap (C12, ASK-CT-3) is closed: `web.view` carries `leg_target_id` and `leg_dir` (WM-36), used by CT-15 for the in-flight route. WM:1309 places `depth_label` on the "pause panel / save list"; ruled to the main-menu save list only (**C13**, ASK-CT-4, CT-17) |
| **Save/Load & Versioned Format** (designed) | `campaign_info()` / `list_saves()` → `SaveInfo` {campaign id, generation, kind (`AUTO` / `EVENT` / `QUIT`), `T`, playtime (`T div TICK_RATE`, whole seconds, Save/Load's), `game_version`, crew count, `RISKY`, depth label}; `save_status` ∈ {`IDLE`, `WRITING`, `WRITTEN`, `FAILED`}; `restore_reason` ∈ {`NEW_GAME`, `LOAD_GAME`, `DEATH_STUB`, `FAULT_RECOVERY`}; results `SL_QUIT_NO_WRITE_DOWNED`, `SL_QUIT_DEDUPED` (SL:784), `SL_LOAD_NO_LOADABLE_SAVE`, the incompatible-build result | `new_game(world_seed)`, `continue_campaign()`, `save_and_quit()`, `retire_campaign(campaign_id)`, `delete_campaign(campaign_id)`; no `save_now()`, no `load(generation)` outside a dev build | SL-21 (SL:261); SL-6 (SL:195); UI-2..UI-9 (SL:998-1005) | Save/Load owns the format, generations, the gate, `RISKY`, the death stub, `SaveInfo` computation and the vocabulary; this GDD owns the screens, the copy, the hold gesture | SL:477 (Interactions row): "Calls the host-local API of SL-21; reads `save_status`, `SaveInfo`, `restore_reason`; destructive actions are hold-to-confirm (HR-14); owns the pause panel's look, the load/continue screen and the message copy". SL:993: "Save/Load draws nothing itself. Every surface below is drawn and laid out by another system ... The player-facing vocabulary is fixed: **Continue**, **Save & Quit**, **New campaign**". SL:1012 (UX Flag): "Stories that reference UI should cite `design/ux/[screen].md`, not the GDD directly" | Yes, with wording to refresh: UI-8 says "Chart Table issues the pause" (SL:1004); the kernel is not `RUNNING` there, so UI-8 is a boot-flow screen (**C7**, CT-25). UI-7 is shown here only if `campaign_info()` reports a verification failure, else HUD's (ASK-CT-5) |
| **Pixel-Perfect Render Pipeline & Camera** (designed) | `H`, `H_default = max(k, 3)` (k = integer world scale, not the zoom step), the HUD canvas `picture / H`, the 5 % safe margin; `PAUSED_FROZEN` (the last frame held, undimmed, `present_time` frozen); `TRANSIT_HOLD` (layer-1 black); camera `CUT` | none (it requests the kernel pause, not the pipeline) | PR-3 (layer 3 root-viewport `CanvasLayer`), PR-12 (HUD scale), PR-13(c)/(e), PR-14, D.4 | Pipeline owns the layer model, the frozen world and its non-treatment, the step sets; this GDD owns the panels | PIPE:296 (Interactions row 7): "layer-3 Controls; the pause panel over the frozen, undimmed world; the settings UI of PR-14; the first-launch photosensitivity notice ... The chart table is an `INPUT_SURFACE`; it must not assume 'pick a target' is paused (kernel Open Q 6) — `TRANSIT_HOLD` works either way. Reading face is the one continuous scale". PIPE:776: "Chart Table (Web Map & Transit) | Hard | The root-viewport layer; the paused-frozen rule ... | Nothing". PIPE:301 (row 12): "Alpha-tier systems; MVP exposes the hooks with defaults" | Partly: row 7 assigns "the settings UI of PR-14" here; narrowed to the three-item Display & Comfort page (**C2**, CT-18); the full PR-14 UI is Settings'. PIPE:269 "returns to the chart table" is reconciled with KRN:151 in CT-26 (**C14**) |
| **Command & Simulation Layer** (designed) | none read from kernel state by any rule (CT-4); `PauseGate.can_save_quit` reads it inside the gate only | `request_pause`, `request_resume`, `request_shutdown` (CR-15); no commands | KRN CR-15, CR-16, CR-11 | Kernel owns the control plane, pause semantics and the paused service cycle | KRN:111 (subscriber classes): "`INPUT_SURFACE` | As `PRESENTATION`, plus **submit** commands as the local seat | Player input adapter, Chart Table, menus". KRN CR-16 (KRN:129): "... **Pause is single-player only**: in co-op a menu cannot stop a shared sim — VS-B treats pause as a UI overlay with the kernel `RUNNING`. The Chart Table GDD must not assume 'pick a target' happens while paused." KRN:961 (Open Question 6): "Pause is single-player only — confirm the Chart Table GDD does not assume 'pick a target' happens while paused, and define VS-B's pause-as-overlay" | Yes: the question is answered in CT-4. The `MAX_PAYLOAD` example at KRN:715 ("a chart-table command with a path array") is stale because this layer submits no command (**C3**) |
| **Player Controller & Input** (designed) | the `chart_table` context; actions `chart_table` (Esc / Start), `cancel` (X / B), `interact` (E, alias LMB / A); directional navigation; device-last-used glyphs (`DEVICE_SWITCH_HYSTERESIS`, PCI, registry); `CONFIRM_HOLD_S`; wheel as scroll | none | PC-2 (contexts), PC-13 (bracket and glyphs), PC-16, the `gameplay`-context default binding table (PCI:64-80; no `chart_table`-context table exists, ASK-CT-2), `BINDING_REQUIRED{chart_table}` | Controller owns actions, contexts, rebinding, glyph switching and `CONFIRM_HOLD_S`; this GDD owns the context's contents and the snap | PCI:217 (Interactions row 17): "Transit → Controller; Adapter → Chart Table | `pod_hop` → `IN_POD` → arrival body transfer; no pod steering at MVP; `chart_table` context, snap-to-node is Chart Table's". PCI:78: "`chart_table` | Esc | Start | press | `request_pause` solo (control plane via the menu); overlay only in co-op". PCI:147 (PC-16): "`chart_table` calls `request_pause` (CR-15) through the menu in single-player and opens an overlay with the kernel `RUNNING` in co-op (CR-16)". PCI:699: "If a rebind would remove the last path to the chart table on the active device: refused with `BINDING_REQUIRED{chart_table}`" | Yes, with two asks: the context has no navigation or accept action in the table (**ASK-CT-2**), and PCI:656 "hold timers (confirm bar, name reveal) are frozen with the pause" concerns the Adapter's in-world timers, not the UI-3 bar (**C15**, CT-22) |
| **Interaction System** (designed) | the console press and its refusal words, drawn by Interaction on the chart-table-register plate (WM:1029) | the pick, injected at the seam (CT-10, **ASK-CT-1**) | `pod_hop` is "Web Map's" (IA:131); CONSOLE subject, `tool_required` NONE; `console_ready` | Interaction owns the prompt, the verb choice and the refusal reasons and copy | IA:131 (declaration row): "`pod_hop` | — | Web Map's | **Web Map** (undesigned) | PLAYER | Web Map's". WM:1301: "Prompt only; no Web Map & Transit element. The console is the only trigger for `pod_hop` / `berth` / `unberth`. The plate sits on the meta-register Ground" | Consistent; the seam is provisional (degraded: a words-only refusal from Interaction's table) |
| **Suit & Equipment** (designed, soft) | none | `new_game` seat defaults: `create_seat_beam` (E9), then `bind_beam` (E4) with `BEAM_WIDE` for seat 1 | SE-9 (SE:189); SE:476 | Suit owns the beam and the loadout | SE:189: "An optional new-campaign picker is the Chart Table's; the choice persists in `suit.state`." SE:476: "host-local boot call: `create_seat_beam` (E9) then `bind_beam` (E4) | *(provisional.)* **Must repeat:** ... the optional beam-shape picker" | Yes: no picker at MVP, the `NewCampaignOptions` seam is reserved (CT-24); this GDD repeats the picker clause |
| **Adaptive Audio** (designed, soft) | none | cue requests (a UI-bus focus-move cue and a confirm cue) | AA:562 (row "Chart Table / menus"); AA:1086 (Dependencies row) | Audio owns every sound | AA:562: "menu and map sounds are UI-bus, ducked out of the world mix on open; pause suspends all sim-derived sounds | no sanity or Attention effect on menu audio | Chart Table | HR-6". AA:1086: "Chart Table / menus | both | Soft | UI bus; pause behaviour | yes, `CT:393`, `CT:524`, `CT:570` | in-flight" | Yes (CT-29); the AA Dependencies row now names this GDD (its `CT:nnn` line cites go stale as this file grows, which is Audio's to refresh) |
| **Attention System** (designed) | none | none | negative requirement | Attention owns the meter that must not be shown | AT:1373: "*Neighbouring UI that must not absorb Attention:* ... Chart Table (depth rings are shown as place, not as attention). Each of those GDDs must repeat this rule or surface a CONFLICT (AC-AT-61, AC-AT-66)." | Yes: repeated as CT-31 |
| **Horror Entities** (designed) | none | none | negative requirement | Horror owns the entities | HE:377: "**HUD & Suit Layer, In-World Instruments, Chart Table** (undesigned) | No element shows a horror, a threat bar, a proximity indicator or a direction | none". HE:1358: "No HUD, instrument, chart-table or web-map element shows a horror, a threat bar, a proximity indicator, a direction, an off-screen arrow, a countdown to a strike, a 'last seen' pin or a target ring, **under any accessibility setting** (AC-HE-119)." | Yes: CT-32 |
| **Boarding & Salvage** (designed) | none | none | negative requirement | Boarding owns salvage | BS:334: "**Chart Table** | picks the target; shows only what the crew lit; never shows loot value or remaining salvage count (ART 7.4) | none". BS:1204: "the Chart Table shows what the crew lit and the fuel the hop will cost" | **Conflict on BS:1204**; WM governs (CT-35, **C1**, routed to Boarding to reword). BS:334 is consistent (CT-33) |
| **Sanity & Hallucination Rules** (designed) | none | none | negative requirement; truth anchor | Sanity owns the stages and the forced-stage hook | SN:1350: "HUD, crew strip, inventory, pause, settings, chat, death/restore screen | ... | **Nothing about sanity.**" SN:1357: "At stage 0 and at forced stage 3 (`force_stage`): the HUD, the crew strip, the inventory, the pause menu ... and the death/restore screen; retained in `production/qa/evidence/` (AC-SN-67, 83, 147)." | Yes (CT-34), except the stale "inventory" and "death/restore screen" entries (**C8**, CT-37) |
| **Perception Filter** (designed) | none | none | truth classification | Filter owns the wrap | PF:223 (row 9): "HUD frame, menu, pause, settings, chart table, text chat, Debug Overlay | TRUTH (separate viewport; the pass sits below every `CanvasLayer`; the frame is fed nothing) | PIPE PR-3, PR-10; HR-6, HR-13; SN-14" | Yes (CT-2) |
| **Item & Module Database / Derelict Authoring / Life-Support Graph** (designed) | none | none | boundary only | n/a | IDB R8 and SE:1289: no inventory screen or weight readout; DA:688: authors no chart-table prop; LSG:22: "the player never operates the Graph as a menu" (no Chart Table row in the Graph GDD) | Consistent (CT-1, CT-37); the Graph has no row naming this system and needs none |

#### Lateral and downstream (undesigned or reserved)

| Neighbour (status) | Data in | Data out | Interface | Owner | Contract carried by this GDD | Notes |
|---|---|---|---|---|---|---|
| **HUD & Suit Layer** (in design (file on disk, Status In Design; its rows HS-36, HUD:300, HUD:341, HUD:402 and HUD:459 name this GDD and agree)) | none | shared component grammar via `design/ux/interaction-patterns.md` (keycap glyphs, Ground plate, chamfered bezel, bracket) | HUD:304 states the HUD's UX spec lands first and defines `design/ux/interaction-patterns.md`; this layer cites it | HUD owns layer 2: the critical strip, suit panel, hand rail, prompts and the failed-write notice (SL UI-10); this layer owns nothing on layer 2; behind the pause panel the HUD stays visible and frozen, except that an active warning mark is held ON solid (HS-23); the pause re-toggle interval the HUD asks for is declined here (OQ-CT-27) | *(provisional — HUD in design)* If the HUD never defines the shared components, this layer defines the ART 7.4 menu components and HUD reuses them. UI-7 may appear on the first `RUNNING` frame as HUD's line |
| **In-World Instruments & Inspect View** (in design (file on disk, Status In Design; IWI:126 rejects the lit chart-table console, consistent with C4)) | none | none | the Controller's Adapter owns the `inspect_card` context (PC-2, PCI:62; IW-2, IWI:531); In-World Instruments owns the card's content and draws no schematic (IW-28, IWI:126); no lit chart-table console exists | Instruments (readings only) | C4 terminology only; this layer designs no prop |
| **Debug Overlay** (designed (`debug-overlay-true-values.md`); its Interactions B Chart Table row DO:416, Dependencies row DO:776 and Cross-References row DO:1029 name this GDD and agree: layer 4 above layer 3 (DO-5), no debug view here, the FAULTED hand-off; world debug draws stay on layer 4 (DO-26)) | may read chart state | none | layer 4, DIAGNOSTIC, reads the true channel | Debug Overlay | this layer has no debug view; the true-value overlay is never an accessibility option (ART 7.4) |
| **Settings & Input Rebinding** (undesigned, Alpha; depends on this GDD, IDX:210, Hard) | `SettingsStore` interface and the Display & Comfort values | the Display & Comfort page is replaced; the seam stays | PIPE:301 "MVP exposes the hooks with defaults"; PCI:213, :845 | Settings (Alpha) | *(provisional — Settings undesigned)* `BINDING_REQUIRED{chart_table}` stays the Controller's; zoom is never a setting |
| **Accessibility Settings** (undesigned, Alpha) | none | hold-duration option and two-tap, reading-face scale, HUD opacity floor, HUD scale | the hold-duration option of `CONFIRM_HOLD_S` (PCI's range) | Accessibility (Alpha) | *(provisional)* `design/accessibility-requirements.md` does not exist |
| **Co-op Session & Replication / Chat / Pings** (undesigned, VS-B) | a guest's pause or save request (a command defined by Session / Save-Load, KRN CR-15) | `PauseGate` OVERLAY; crew glyphs and a left-edge text chat are reserved slots | KRN CR-15, CR-16 | Session | *(provisional)* risk K1 (crew glyph as a zero-cost check against class (c)); the opaque panel covers the hand rail bottom-right in OVERLAY while the sim runs (reserved risk for HUD and Session) |
| **Knowledge Codex & Logs** (undesigned, VS) | annotation data, PROFILE scope | none | IDX:47 | Codex | *(provisional)* each annotation needs a stated effect on play |
| **Failure Contract & Re-wake** (undesigned, VS-A) | none | none | ART 7.4 "Loss": read at the chart table after re-wake | Failure Contract | *(provisional)* the MVP death stub has no screen (SL-7) |
| **Onboarding** (undesigned, VS) | none | none | a flow after the notice | Onboarding | *(provisional)* `POD_NOT_READY` is reserved in Web Map & Transit |
| **Procedural Derelict Generation** (undesigned, VS-A) | a candidate for generating rings 2-3 and the layout | none | n/a | Generation | the MVP layout is authored (CT-12, CT-13) |

**Bidirectional-consistency summary.** Web Map & Transit, Save/Load, the Pipeline, the Controller, the kernel, Suit & Equipment, Adaptive Audio, Attention, Horror, Boarding and Sanity each already name this system (quoted above); the designed Debug Overlay GDD (`debug-overlay-true-values.md`) also names it (DO:416, DO:776, DO:1029) and agrees. The Graph, Item DB, Derelict Authoring and Interaction name it only as "Web Map's" or not at all (Interaction's `pod_hop` row points at WM, not at this layer; the seam is ASK-CT-1). The systems index lists only Web Map & Transit and the Pipeline (IDX:95, :201): corrections routed to `/map-systems` (C5). The registry carries four "pending referenced_by" comments naming `design/gdd/chart-table.md` (line numbers not cited; the registry lines drift), a path that differs from this file; the merge stage rewrites them and drops the `HOP_FUEL_ML` pending reference or keeps it only for a negative sentence (C6).

## Formulas

**Scope and determinism statement.** The Chart Table owns no simulation state (CT-27), so none of the formulas below is hashed (kernel D.12 `state_hash`), snapshotted, replicated or fed back into the simulation, and none uses `rng_derive`. They are **host-local presentation arithmetic** on the layer's own UI clock (CT-43): floating-point is therefore permitted, and wherever a formula is integer-only (D.1, D.1b, D.2's accumulator, D.6, D.7, D.9) that is for **pixel exactness and test repeatability**, not for the kernel's determinism rule. The only simulation-derived inputs are read-only integers or enums from `web.view` and `SaveInfo`. Every formula below is evaluated at presentation time and may be called from a test double with no kernel running (CT-4, AC-CT-2). Dependency formulas are cited by registry or owner name, never redefined: the HUD scale `H` and `H_default = max(k, 3)` (Pixel-Perfect Render Pipeline D.4, registry `hud_multiplier`), `CONFIRM_HOLD_S` (Player Controller, registry), `HUD_MIN_SCALE` (Pipeline, registry) and the art-bible numbers of section 7 (ART 7.3, 7.4). Notation: `a div b` is integer (floor) division of non-negative integers; `ceil_div(a, b) = (a + b - 1) div b` is integer ceiling division (registry `safe_margin`); `P_w`, `P_h` are the picture rectangle in screen px (Pipeline D.2/D.3); the HUD canvas is `P_w / H` by `P_h / H` HUD px and may be fractional (the Deck is 426.67 x 266.67 at `H = 3`). The six canvas rows are the rows of CT-16, numbered 1-6 in the order they are listed there.

### D.1 Panel fit

The panel fit formula is defined as:

`W_panel = P_w div (3 * H); safe_x = ceil_div((5 * P_w + 99) div 100, H); safe_y = ceil_div((5 * P_h + 99) div 100, H); content_w = W_panel - PANEL_INNER_PAD_PX - safe_x; V_need = 4 * PANEL_ITEM_H_HUD + 3 * PANEL_ITEM_GAP_HUD + PANEL_RECORD_LINE_H_HUD + n_wrap * PANEL_LINE_EXTRA_HUD; V_avail = (P_h div H) - 2 * safe_y; the panel fits iff V_need <= V_avail`

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `P_w`, `P_h` | int, screen px | 1280-3840 / 720-2160 reachable | Pipeline D.2/D.3 (picture rect) | the visible picture, after `keep` or `expand` |
| `H` | int | `H_default .. 2 * H_default` | Pipeline PR-12, D.4 (cited; `H_default = max(k, 3)`, `k` the integer world scale, never the zoom step) | the HUD integer scale; at MVP only `H_default` is player-reachable (CT-18) and the rest is reached by the forced-`H` test hook (CT-16) |
| `W_panel` | int, HUD px | 71-213 over the six rows | this GDD (ART 7.4 "right third", integer rule of CT-16) | panel width, hung from a right anchor of 1; `W_panel * H` is an integer screen px by construction |
| `safe_x`, `safe_y` | int, HUD px | 11-32 / 6-18 | registry `safe_margin` (source HUD & Suit Layer; the 5 % is the art bible's, ab:754; Pipeline PR-2 cites it), cited, not owned here | the safe margin in whole HUD px, rounded **up** per registry `safe_margin` (never below 5 %); the frame fit of D.1b is re-asserted against the larger margin (AC-CT-26). *(Registry value, not an orchestrator ruling; the earlier floor form is retired.)* |
| `PANEL_INNER_PAD_PX` | int, HUD px | 2-8, default 4 `[PLACEHOLDER]` | this GDD, checkpoint ruling Q18 | inset between the panel's left edge and its content (chamfer border and rivet clearance); `PANEL_PAD_HUD` is retired |
| `content_w` | int, HUD px | 56-177 over the six rows | calculated | the width a panel label may use; lint-checked by D.8 |
| `PANEL_ITEM_H_HUD`, `PANEL_ITEM_GAP_HUD`, `PANEL_RECORD_LINE_H_HUD`, `PANEL_LINE_EXTRA_HUD` | int, HUD px | 12-20 / 2-6 / 8-12 / 6-10; defaults 16 / 3 / 10 / 8 `[PLACEHOLDER]` | this GDD | item height, gap, newest-record line height, and the extra height of one wrapped line |
| `n_wrap` | int | 0-4 | the `ct.*` fit classes (CT-28) | total extra wrapped lines across the four items (`LABEL_2` items; a third line is a lint failure) |
| `V_need`, `V_avail` | int, HUD px | 83-115 / 108-324 | calculated | stacked height needed and the vertical space inside the safe margin |

**Output Range:** `W_panel` 71 to 213 HUD px, i.e. 426 to 639 screen px; `content_w` 56 to 177 HUD px (336 to 531 screen px); the world strip left of the panel is `P_w - W_panel * H`, never below 854 screen px at 1280 wide. The panel fits at all six rows with `n_wrap = 0`; at the smallest canvas (row 6) it tolerates at most `floor((108 - 83) / 8) = 3` wrapped lines in total.

| Row | Picture, `k`, `H` | Canvas (HUD px) | `W_panel` (screen px) | `safe_x` / `safe_y` | `content_w` (screen px) | `V_avail` | Chart multiplier `m` (D.1b) | Chart layout |
|---|---|---|---|---|---|---|---|---|
| 1 | 1920x1080 keep, 3, 3 (also 2560x1440 at `H` 4 and 3840x2160 at `H` 6: same canvas) | 640 x 360 | 213 (639) | 32 / 18 | 177 (531) | 324 | 2 | spatial |
| 2 | 1920x1080 keep, 3, 6 | 320 x 180 | 106 (636) | 16 / 9 | 86 (516) | 162 | 1 | spatial (exact fit) |
| 3 | 1280x800 expand (Deck), 2, 3 | 426.67 x 266.67 | 142 (426) | 22 / 14 | 116 (348) | 238 | 1 | spatial |
| 4 | 1280x800 expand (Deck), 2, 6 | 213.33 x 133.33 | 71 (426) | 11 / 7 | 56 (336) | 119 | 0 | **compact** |
| 5 | 1280x720 keep, 2, 3 | 426.67 x 240 | 142 (426) | 22 / 12 | 116 (348) | 216 | 1 | spatial |
| 6 | 1280x720 keep, 2, 6 | **213.33 x 120 (smallest)** | 71 (426) | 11 / 6 | 56 (336) | 108 | 0 | **compact** |

**Example (row 3, then row 6).** Row 3: `W_panel = 1280 div 9 = 142`, so the panel is `142 * 3 = 426` screen px and its left edge is at `1280 - 426 = 854`, an integer; a literal one-third anchor would put the edge at 853.33 and make the 1-px bezel and seam one screen pixel thicker on one side under `gui_snap_controls_to_pixels` (the engine's rounding rule is NOT SOURCEABLE, so the layout does not depend on it). `safe_x = ceil_div((5 * 1280 + 99) div 100, 3) = ceil_div(64, 3) = 22`; `content_w = 142 - 4 - 22 = 116` HUD px = 348 screen px. Row 6: `W_panel = 1280 div 18 = 71`; `safe_y = ceil_div((5 * 720 + 99) div 100, 6) = ceil_div(36, 6) = 6`; `V_avail = 120 - 12 = 108`; `V_need = 4 * 16 + 3 * 3 + 10 + 0 = 83`, slack 25, so the panel fits; with all four items wrapped once (`n_wrap = 4`) `V_need = 83 + 32 = 115 > 108` and the lint fails the string set. Row 1: `W_panel = 1920 div 9 = 213` (639 screen px, one screen pixel short of a true third, invisible).

### D.1b Chart frame, chart multiplier and the compact breakpoint

The chart frame formula is defined as:

`MIN_W = (10 * CHART_FRAME_W) div 9; MIN_H = (10 * CHART_FRAME_H) div 9; m = min(P_w div (MIN_W * H), P_h div (MIN_H * H)); compact iff m < 1; x0 = ((P_w div H) - CHART_FRAME_W * m) div 2; y0 = ((P_h div H) - CHART_FRAME_H * m) div 2; node_hud(i) = (x0 + m * pos_x(i), y0 + m * pos_y(i)); S_MIN = 24 + 2 * BRACKET_PAD_HUD; layout lint: for every pair i != j, |pos(i) - pos(j)| >= S_MIN`

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `CHART_FRAME_W`, `CHART_FRAME_H` | int, art px (= HUD px at multiplier 1) | 288 / 162 `[PLACEHOLDER]`, derived: the smallest spatial canvas (320 x 180, row 2) reduced by the 5 % margin on each axis | this GDD (checkpoint ruling Q7) | the fixed frame in which node positions are authored (CT-12) |
| `MIN_W`, `MIN_H` | int, HUD px | 320 / 180, **derived** | this GDD (checkpoint ruling Q8) | `MIN_W` **is** `COMPACT_LAYOUT_MIN_CANVAS_W`; `MIN_H` is its height twin; both move with the frame and are never edited independently |
| `m` | int | 0-2 reachable (H >= k bounds it at 2) | calculated | the chart multiplier, applied to positions only; glyphs stay 24x24 at `H` (CT-12) |
| `x0`, `y0` | int, HUD px | `>= safe_x`, `>= safe_y` when `m >= 1` | calculated | frame origin, whole HUD px; rings and filaments extend full-bleed outside the frame |
| `pos_x(i)`, `pos_y(i)` | int, art px | `0 .. CHART_FRAME_W - 1`, `0 .. CHART_FRAME_H - 1` | `chart_layout.json` (CT-12), authored per `derelict_id` | node centre in frame-local px; never derived from `world_slot`, legs, ticks or distance |
| `BRACKET_PAD_HUD` | int, HUD px | 2-6, default 4 `[PLACEHOLDER]` | this GDD | clearance between a 24x24 glyph and its focus bracket |
| `S_MIN` | int, HUD px | 28-36, default 32, derived | calculated | the minimum centre-to-centre distance so brackets never overlap neighbours |

**Output Range:** `m` is 0, 1 or 2 over every reachable canvas (`H >= k` bounds it at 2); the layout is spatial iff `m >= 1`. With the margins rounded up (D.1, registry `safe_margin`) the fit is **not** derived by construction (a canvas one or two HUD px above a `320 * m` boundary would fail it); it is asserted per reachable canvas row instead: `CHART_FRAME_W * m + 2 * safe_x <= P_w div H` and `CHART_FRAME_H * m + 2 * safe_y <= P_h div H` hold at every CT-16 row with `m >= 1` (row 2 exactly, since 16 and 9 are exactly 5 % of 320 and 180), so the frame never leaves the safe area (AC-CT-26). **Why the integer form:** `0.9 * 320` is not exactly representable in binary floating point and the 1080p `H = 6` canvas (exactly 320 x 180) sits on the boundary.

**Example.** Row 2 (1920x1080, `H = 6`): `m = min(1920 div 1920, 1080 div 1080) = 1`, spatial (the boundary case), `x0 = (320 - 288) div 2 = 16 = safe_x`, `y0 = (180 - 162) div 2 = 9 = safe_y`. Row 3 (Deck, `H = 3`): `m = min(1280 div 960, 800 div 540) = min(1, 1) = 1`, `x0 = (426 - 288) div 2 = 69`, `y0 = (266 - 162) div 2 = 52`; an illustrative node at frame position (210, 60) is drawn at HUD px (279, 112). Row 1: `m = 2`, `x0 = 32`, `y0 = 18`, the same node at (452, 138). Row 6 (720p, `H = 6`): `m = min(1280 div 1920, 720 div 1080) = 0`, **compact**. Separation lint with illustrative positions HOME (144, 100), A (210, 60), B (80, 40) (not authored content): HOME-A 77.2, HOME-B 87.7, A-B 131.5, all >= 32. Content-lint corollaries: every record has a position and a stable ordinal, no position is read from `world_slot`, and no authored distance correlates with any Web Map & Transit cost (CT-12).

### D.2 Hold-to-confirm fill

The hold-to-confirm fill formula is defined as:

`t_us(n+1) = t_us(n) + min(dt_us, UI_DT_MAX_MS * 1000) while the activation is physically down and the pointer or focus stays on the item; fill = clamp(t_us / (CONFIRM_HOLD_S * 1000000), 0, 1); needle_px = min(floor(fill * L_BAR_HUD), L_BAR_HUD - 1); the action fires on the first frame with t_us >= CONFIRM_HOLD_S * 1000000; release, cancel, chart_table, pointer exit or focus loss sets t_us = 0 in the same frame`

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `t_us` | int, microseconds of UI process time | 0 to the threshold | this layer (CT-43); **not** simulation time, runs while the kernel is `PAUSED` | the held time on the layer's own clock |
| `dt_us` | int, microseconds | 0 to unbounded | the render delta | the frame delta before clamping |
| `UI_DT_MAX_MS` | int, ms | 33-100, default 50 `[PLACEHOLDER]` | this GDD | the per-frame delta clamp (CT-43) |
| `CONFIRM_HOLD_S` | float, seconds | per the registry (Tuning Knobs, cited rows) | Player Controller, registry constant (cited, not retuned) | the hold duration; the same constant that closes SL OQ-32 |
| `L_BAR_HUD` | int, HUD px | 32-96, default 64 `[PLACEHOLDER]`; capped at `P_w div H - 2 * safe_x` | this GDD | the bar length; the bar sits on a full-bleed message plate (`UI3_PLATE`, UI-5), not inside the 71-px panel |
| `fill`, `needle_px` | float 0-1, int HUD px | 0-1, 0 to `L_BAR_HUD - 1` | calculated | the linear bar and its needle-mark (ART 7.3: linear, never radial) |

**Output Range:** `fill` in [0, 1]; the action fires at most once per gesture; a new gesture needs a new press edge (CT-22, CT-41).

**Example** (using the registry value of `CONFIRM_HOLD_S` at authoring, 0.8 s, so the threshold is 800,000 microseconds; the number is illustrative and the constant is the Controller's). At 60 fps, `dt_us = 16,667`. Frame 24: `t_us = 400,008`, `fill = 0.500`, `needle_px = floor(0.5 * 64) = 32`. Frame 47: `t_us = 783,349`, not yet. Frame 48: `t_us = 800,016 >= 800,000`, **fires**. Release at frame 30: `t_us = 0`, no action. Hitch: one 400 ms frame is clamped to 50 ms, so `fill` rises by 0.0625 and a stall can never complete a hold. At 10 fps (`dt_us = 100,000` clamped to 50,000) the bar advances at half real time and the hold takes 1.6 s: accepted, because a layer below 20 fps is already a failed frame budget. At either end of the Controller's range the same arithmetic holds with the threshold replaced.

### D.3 Snap-to-node

The snap-to-node formula is defined as:

`d = (x_c - x_f, y_f - y_c); phi = atan2(d.y, d.x) in degrees; delta = ((phi - theta + 540) mod 360) - 180 in [-180, 180); S1 = { c in F : -SNAP_SECTOR_DEG/2 <= delta(c) < +SNAP_SECTOR_DEG/2 }; S2 = { c in F : |delta(c)| < 90 }; chosen = argmin over (dist(f, c), |delta(c)|, ordinal(c)) lexicographically, over S1 if S1 is non-empty, else over S2 if S2 is non-empty, else the focus stays`

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `f` | node | the focused node | this layer | where focus is now |
| `F` | set of nodes | the focusable nodes except `f` | D.4 plus HOME (CT-11 table) | the candidates; ORPHAN, berthed and absorbed records are not in it |
| `(x, y)` | int, frame art px, screen y down | frame-local | `chart_layout.json` (CT-12) | authored positions; `d.y` flips so that up is positive |
| `theta` | deg | 0 (right), 90 (up), 180 (left), 270 (down) | the pressed direction | cardinal navigation (CT-5); a stick press is the layer's own four-way read of the move vector |
| `SNAP_SECTOR_DEG` | float, deg | 60-120, default 90 `[PLACEHOLDER]` | this GDD | the width of the primary sector |
| `dist` | float, layout px | >= 0 | calculated | used **only for ordering and never displayed** (WM:1049, ASK-AD-7) |
| `ordinal(c)` | int | 1..N | `chart_layout.json` | stable authored ordinal, the last tie-break |

**Output Range:** a node in `F` or "stay"; no wrap, no shake, no flash. The half-open interval `[-half, +half)` puts a node exactly on a diagonal in **exactly one** sector (45 degrees belongs to up, not right; 315 degrees belongs to right, not down), so the result is deterministic. `delta` is rounded to four decimals before the comparison so a floating-point 44.99999999 does not decide a diagonal. Content-lint invariant: the snap graph over the authored layout is strongly connected (every focusable node reachable from every other in at most `N - 1` presses).

**Example** (illustrative data, not authored content; screen coordinates, y down): HOME (144, 100), A (210, 60), B (80, 40), `SNAP_SECTOR_DEG = 90`. Focus HOME, press right (theta 0): A has d = (66, 40), phi = 31.2, delta = 31.2, in `S1`; B has d = (-64, 60), phi = 136.9, delta = 136.9, out; result **A**. Press up (theta 90): A delta = -58.8, B delta = +46.9, both outside [-45, 45), so `S1` is empty; `S2` admits both (|delta| < 90); distances HOME-A 77.2 and HOME-B 87.7; result **A** (this is why the fallback exists: with three authored points, up from HOME would otherwise be a dead control). Press left (theta 180): B delta = -43.1, in `S1`; result **B**. Press down (theta 270): A delta = 121.2, B delta = -133.1, both outside 90, `S2` is empty: **focus stays** (no wrap). From A, press left: HOME delta = 31.2, result HOME. Stick navigation is edge-triggered at the Controller's deadzone with the repeat schedule of D.9.

### D.4 Legal-target mirror

The legal-target mirror formula is defined as:

`legal_targets(view) = {} if view.transit_state != IDLE or view.pod_at == IN_FLIGHT; { n.derelict_id : n in view.nodes, n.kind == DERELICT, n.status not in {ABSORBED, ORPHAN}, not n.berthed } if view.pod_at == HOME_ID; { HOME_ID } if view.pod_at is a derelict id`

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `view` | `web.view` | read-only | Web Map & Transit WM-36 (provisional name) | the only input; no other read |
| `view.nodes[].kind` | enum | HOME, DERELICT | WM-3 | the roster kind |
| `view.nodes[].status` | enum | UNSEEN, SEEN, BOARDED, ABSORBED, ORPHAN | WM-3 | the record status |
| `view.nodes[].berthed` | bool | | WM-36 | derived by WM |
| `view.pod_at` | record id or `IN_FLIGHT` | | WM-36 | where the pod is |
| `view.transit_state` | enum | IDLE, LEG, ARRIVING, BERTHING | WM-36 | the machine state |
| `HOME_ID` | `derelict_id` | | derived: the `derelict_id` of the `nodes[]` entry whose `kind` is HOME | a derived label, never a constant, never registered |

**Output Range:** a set of 0 to N - 1 record ids (the HOME singleton from a derelict). It is WM-4 evaluated for **selectability only**; it is not a validation and not a second copy of WM-9's reason order. **The mirror never reflects `FUEL_LOW`, `BUSY`, `OUT_OF_RANGE`, `NOT_LIT`, `ARRIVAL_BLOCKED`, `CAPACITY` or `BERTH_PENDING`** (CT-9): none of them is a `web.view` field, and greying a node on low fuel would put a fuel meter on the truth layer. A pure function of one `web.view` snapshot: a test double feeds two views that differ only in fuel and expects identical output (AC-CT-24).

**Example.** Pod at HOME, `transit_state` IDLE, A UNSEEN, B BOARDED and not berthed: `{A, B}` (UNSEEN is selectable, bare; WM:1299). B `berthed`: `{A}`; the console still refuses a hop to A in words (`BERTH_PENDING`, OQ-CT-3), because the mirror does not carry it. B ABSORBED and A ORPHAN: `{}`. Pod at A: `{HOME_ID}`. `pod_at == IN_FLIGHT`, or `transit_state == LEG`: `{}`. The roster {A SEEN, B UNSEEN, C ABSORBED, D SEEN and berthed} with the pod at HOME gives `{A, B}`.

### D.5 Filament drift

The filament drift formula is defined as:

`r = FILAMENT_DRIFT_FRACTION * V_DRIFT_CEIL; phi(n+1) = phi(n) + r_eff * min(dt, UI_DT_MAX_MS / 1000), with r_eff = 0 under reduced motion and r otherwise; offset_px = FILAMENT_DRIFT_STEP_PX * floor(phi / FILAMENT_DRIFT_STEP_PX) mod P_f; the offset is applied to the ring-1 filament sheet only`

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `V_DRIFT_CEIL` | float, art px per second | the art-bible bound | ART 7.3 and 2.11 (cited, not restated with a value, not registered under a new name) | the ceiling "at most 1 art px per second, current ring only" |
| `FILAMENT_DRIFT_FRACTION` | float | 0.25-1.0, default 0.5 `[PLACEHOLDER]` | this GDD | the fraction of the ceiling actually used *(orchestrator ruling: the ux-designer's fraction knob over the ui-programmer's raw rate, because a raw number would be a second owner of an art-bible value, rulings R28; default 0.5 over the ux-designer's 1.0 so the layer sits visibly below the ceiling and "very low energy" holds, ART 2.11)* |
| `FILAMENT_DRIFT_STEP_PX` | int, art px | 1-2, default 1 `[PLACEHOLDER]` | this GDD | the quantum, so the texture never sits on a half pixel |
| `phi` | float, art px | 0 up; persists across opens | this layer (CT-43) | the drift accumulator; unlike `t * r` it does not jump when reduced motion toggles |
| `dt` | float, seconds | | render delta | clamped by `UI_DT_MAX_MS` |
| `P_f` | int, art px | content-defined, longer than the largest canvas plus one wrap period | art-director (asset A6) | the repeat length of the ring-1 sheet |

**Output Range:** `offset_px` in `[0, P_f)`, whole art px; the step period is `FILAMENT_DRIFT_STEP_PX / r` seconds. Reduced motion holds `phi` (no snap-back, no animation of the stop). Inputs are the UI clock and the reduced-motion toggle only (CT-40).

**Example.** Defaults (`FILAMENT_DRIFT_FRACTION` 0.5 of a ceiling of 1 art px/s, so `r = 0.5`), step 1, `P_f = 720` (illustrative). At 3.9 s `phi = 1.95`, offset 1; at 4.0 s `phi = 2.0`, offset 2; the sheet wraps after 1,440 s. Reduced motion switched on at 100 s (`phi = 50`, offset 50): the sheet holds at 50 for as long as the toggle stays on; switched off at 130 s it resumes from `phi = 50` and reaches offset 51 at 132 s. A 400 ms hitch adds `0.5 * 0.05 = 0.025` art px, not 0.2.

### D.6 Depth-ring ground steps

The depth-ring tone formula is defined as:

`tone(i, c) = G(c) - ((2 * (G(c) - V(c)) * i + CHART_RING_STEPS) div (2 * CHART_RING_STEPS)) for i in 0 .. CHART_RING_STEPS and each channel c; ring i + 1 is band i for i < CHART_RING_STEPS; the centre disc is i = CHART_RING_STEPS; ordering contract: for every i and c, tone(i, c) - tone(i + 1, c) >= RING_STEP_MIN_CHANNEL`

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `G(c)` | int, 8-bit channel | (28, 33, 40) = Ground `#1C2128` | ART 4.1, 4.6 (cited) | ring 1, where every MVP node sits |
| `V(c)` | int, 8-bit channel | (11, 16, 24) = Void Steel `#0B1018` | ART 4.1, 4.6 (cited) | the centre disc, exactly |
| `CHART_RING_STEPS` | int | 1-5 by the contract (the span of 16 units allows `floor(16 / 3) = 5`); default 3, MVP content ceiling 3 `[PLACEHOLDER]` | this GDD | the number of ring bands drawn |
| `RING_STEP_MIN_CHANNEL` | int, 8-bit units | 2-6, default 3 `[PLACEHOLDER]` | this GDD (art-director ruling 5.1, checkpoint ruling Q5) | the minimum per-channel step so adjacent steps survive 8-bit quantisation and a poor panel |

**Output Range:** each tone lies between Ground and Void Steel and the sequence is strictly darker inward; the disc equals Void Steel exactly. The formula is the **ordering contract made concrete**: it restates no percentage (ART 7.4 and WM:1050 say "about 4 %", which cannot be satisfied together with "reaches Void Steel"; conflict row C16, routed to the art-bible owner). The tones are derived values, never new palette entries and never registered (R16, R28).

**Example** (`CHART_RING_STEPS = 3`, Ground (28, 33, 40), Void Steel (11, 16, 24)): ring 1 (i = 0) = (28, 33, 40) `#1C2128`; ring 2 (i = 1) = (22, 27, 35) `#161B23`; ring 3 (i = 2) = (17, 22, 29) `#11161D`; centre disc (i = 3) = (11, 16, 24) `#0B1018`. Per-channel steps are 6/6/5, 5/5/6 and 6/6/5, minimum 5 >= `RING_STEP_MIN_CHANNEL` 3, so the contract holds with margin. The art-director's illustrative tones (24, 29, 36), (20, 25, 32), (15, 20, 28) in the art-director's proposal are one evenly-divided reading over four intervals; the checkpoint ruling (Q5) fixed ring 1 = Ground exactly, so this interpolation governs and the exact on-screen values are verified in the spike (banding check by pixel sampling, AC-CT-47).

### D.7 Newest-record line

The newest-record line formula is defined as:

`HH = playtime div 3600; MM = (playtime mod 3600) div 60; SS = playtime mod 60; text = zero-pad(HH, 2) + ":" + zero-pad(MM, 2) + ":" + zero-pad(SS, 2)`

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `playtime` | int, seconds | >= 0 | Save/Load `SaveInfo.playtime` (`T div TICK_RATE`, whole seconds; read as given, never recomputed here) | the playtime of the newest generation, read once on panel open |

**Output Range:** `HH` grows past two digits without wrapping ("100:00:00" for 360,000 s); the string is set in the instrument face with tabular numerals; it is static while the panel is open (no tick, no "time since"); no wall clock enters.

**Example.** `playtime = 9641`: `HH = 2`, `MM = (9641 mod 3600) div 60 = 2441 div 60 = 40`, `SS = 41`, text `02:40:41`. `playtime = 0` gives `00:00:00`. `playtime = 59` gives `00:00:59`.

### D.8 Label fit

The label fit formula is defined as:

`W_text(key, row) = ceil(n_chars(key) * ADV_EM * em_px * TEXT_EXPANSION_TEST); a LABEL_1 key passes iff W_text <= content_w(row) * H(row); a LABEL_2 key passes iff W_text <= 2 * content_w(row) * H(row); a key that would need three lines fails`

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `n_chars` | int | 1-80 | the `ct.*` key's string | the rendered character count of the whole key |
| `ADV_EM` | float | 0.5 (estimated) | **NOT SOURCEABLE** until the real reading face is imported | the average glyph advance as a fraction of the font size; the lint replaces `n_chars * ADV_EM * em_px` with the measured width of the real face at each row |
| `em_px` | int, screen px | the ART 7.1 menu-body floor | ART 7.1 (cited, 1080p and 1440p only; sub-1080p floors are ASK-UP-2 / OQ-CT-13) | the font size of panel labels (reading face, CT-17) |
| `TEXT_EXPANSION_TEST` | float | 1.2-1.5, default 1.3 `[PLACEHOLDER]` | this GDD | the localisation growth factor the lint applies to every key |
| `content_w`, `H` | int | D.1 | D.1 | the width available and the HUD scale of the row |

**Output Range:** pass or fail per key per row; a failing `LABEL_1` fails the build (a label is never truncated or ellipsised at runtime, CT-28).

**Example** (estimated advance, flagged `[UNV]`). "Display & Comfort", 17 characters, at 20 px: `ceil(17 * 0.5 * 20 * 1.3) = 221 <= 336` (row 6 content width in screen px, registry `safe_margin` rounded up), so it fits one line at every row; "Save & Quit" gives 143. A 36-character label gives `468 > 336`: it fails `LABEL_1` at rows 3 to 6 (content widths 348 and 336 screen px) and passes `LABEL_2` (`468 <= 672`). At 1440p (row 1 canvas, `H = 4`, `em_px` 26): `W_panel = 2560 div 12 = 213`, `content_w = 213 - 4 - 32 = 177` HUD px = 708 screen px, and "Display & Comfort" needs `ceil(17 * 0.5 * 26 * 1.3) = 288`.

### D.9 Directional-navigation repeat

The navigation repeat formula is defined as:

`n_moves(t_held) = 1 if t_held < UI_NAV_REPEAT_MS else 1 + floor((t_held - UI_NAV_REPEAT_MS) / UI_NAV_REPEAT_RATE_MS) + 1; a move past the last node in the pressed direction is a no-op (no wrap); a direction edge is taken only after the stick passes MOVE_DEADZONE and is released only below MOVE_DEADZONE_EXIT`

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `t_held` | int, ms of UI process time | >= 0 | this layer (CT-43) | how long the direction has been held since its edge |
| `UI_NAV_REPEAT_MS` | int, ms | 250-500, default 350 `[PLACEHOLDER]` | this GDD | the delay before auto-repeat starts |
| `UI_NAV_REPEAT_RATE_MS` | int, ms | 80-200, default 120 `[PLACEHOLDER]` | this GDD | the interval between repeats |
| `MOVE_DEADZONE`, `MOVE_DEADZONE_EXIT` | float | Controller values | PCI, not registered (cited) | the hysteresis that stops a stick on the boundary from chattering |

**Output Range:** `n_moves` >= 1 once the edge is taken; a key or d-pad press uses the same schedule.

**Example.** Held for 1,000 ms with 350 / 120: moves at 0, 350, 470, 590, 710, 830, 950 ms, so `n_moves = 1 + floor(650 / 120) + 1 = 1 + 5 + 1 = 7`. Held for 300 ms: 1.

### D.10 Layer-3 draw-call estimate

The layer-3 draw-call estimate formula is defined as:

`D_L3 = (CHART_RING_STEPS + 1) + D_filament + 2 * N_nodes + D_pod + D_route + D_seam + D_frame + D_text + D_hints; D_L3 <= LAYER3_DRAW_CALL_BUDGET`

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `D_filament` | int | 3 | art-director assets A6 and A7 | the drifting sheet (ring 1) plus the two static sheets |
| `D_L3` | int | about 15-30 on the chart | calculated | layer-3 draw calls while the chart is open |
| `CHART_RING_STEPS` | int | 1-5, default 3 | D.6, Tuning Knobs | ring bands; `+ 1` adds the centre disc |
| `N_nodes` | int | 3 at MVP | `web.view` | HOME plus two derelicts; each costs a glyph and, at most, a tick (2 calls) |
| `D_pod`, `D_route`, `D_seam`, `D_frame` | int | 0-1 each | CT-11 | pod marker, dashed route (one tiled quad), berth seam, border and rivets |
| `D_text`, `D_hints` | int | 0-1 / 0-4 | CT-15, CT-28 | the `IN TRANSIT` word; control hints (keycaps and labels) |
| `LAYER3_DRAW_CALL_BUDGET` | int | at most the Pipeline's `D_hud` upper bound while open `[PLACEHOLDER]` (layer-3 term ownership: ASK-CT-12) | this GDD, Pipeline D.8 (cited) | the cap |

**Output Range:** about 15 to 30 calls on the chart (19 in the MVP example); fewer on the panel (a plate, four items, a bracket, a line). **Example:** 4 ring fills + 3 sheets + 6 node calls + 1 pod + 1 route + 0 seam + 1 frame + 0 text + 3 hints = 19 <= 60. This is an **estimate**: batching can lower it, the Pipeline measures the real figure per viewport with `Viewport.get_render_info(RENDER_INFO_TYPE_CANVAS, ...)` (PIPE D.8, AC-CT-45), and the layer-2 HUD beneath is still drawn unless the Pipeline hides it (OQ-CT-16). PIPE D.8's `D_root = D_hud + D_filter + D_hold` has no layer-3 term, so who owns the open-chart draw cost is ASK-CT-12; until ruled the budget stays "at most `D_hud`'s upper bound" and AC-CT-45 reports the layer-3 figure as a separate series.

### Values this GDD cites and does not own

Names and owners only; the numbers sit in the Tuning Knobs "cited" rows and nowhere in rule text (guard rail 5).

| Value | Owner | Used in |
|---|---|---|
| `CONFIRM_HOLD_S` | Player Controller (registry) | D.2, CT-22 |
| `H_default = max(k, 3)`, `HUD_MIN_SCALE`, `hud_multiplier` (registry); `cap_px = 7 * H` (PIPE D.4 variable, carried in `hud_multiplier`'s output, not a registry entry); the 5 % safe margin (registry `safe_margin`, source HUD & Suit Layer, rounded up per axis; PIPE:88, PR-2 Deck safe area, art bible 7.0 cite it) | Pixel-Perfect Render Pipeline PR-12, D.4; HUD & Suit Layer (`safe_margin`) | D.1, D.1b, D.8 |
| fade band, bracket ease-out, drift ceiling | art bible 7.3 | CT-3, CT-13, D.5 |
| ring-step wording ("about 4 %", not restated), the right-third panel | art bible 7.4, WM:1050 | D.1, D.6, C16 |
| type floors, line length, glyph sizes, 3 Hz ceiling | art bible 7.1, 7.2, HR-13 | D.8, CT-28 |
| Ground, Void Steel and the rest of the UI palette | art bible 4.1, 4.6 | D.6 |
| `DEVICE_SWITCH_HYSTERESIS` | Player Controller (registry) | CT-6, D.3, D.9 |
| `MOVE_DEADZONE`, `MOVE_DEADZONE_EXIT` | Player Controller (not registered) | CT-6, D.3, D.9 |
| `SAVE_QUIT_TIMEOUT_MS` | Save/Load (not registered) | CT-21 |
| `HOP_FUEL_ML`, `BERTH_FUEL_ML`, `HOP_FUEL_FLOOR_ML`, `RETURN_HOP_FUEL_ML`, `HOP_LEG_TICKS`, `ARRIVE_SETTLE_TICKS` | Web Map & Transit (registry) | **named only in negative sentences; the layer displays none (CT-35)** |
| `TICK_RATE` | Command & Simulation Layer (registry) | D.7 (through `SaveInfo.playtime`) |

## Edge Cases

Format: **If [condition]**: [exact outcome]. [rationale]. Ids are `EC-CT-n` and are cited by the acceptance criteria. "SOLO" and "OVERLAY" are the two `PauseGate` modes (CT-4); a sentence that names neither holds in both. Where an outcome depends on a neighbour that has not decided, it is marked provisional and points at the ask or open question.

### A. The pick and the target

- **EC-CT-1. If the picked record becomes ABSORBED, ORPHAN or `berthed`, or disappears from `web.view`, while the chart is open**: `ChartPick := NONE` on that `web.view` change, the dashed line disappears in the same frame, the bracket stays where it is, and if the node holding key or pad focus left the focusable set, focus moves to HOME; no message is shown. A stale pick is the only way the chart and the console press can disagree (CT-8).
- **EC-CT-2. If a `request_restore` completes while any state of the layer is open (any `restore_reason`)**: the layer closes into `CLOSED` in the same frame with no animation (TX-27), `ChartPick := NONE`, and the machine is rebuilt from CLOSED. The restored world may not contain the record that was picked, and nothing about the layer is in a snapshot (CT-27).
- **EC-CT-3. If the pod console is pressed at HOME with no valid pick and exactly one legal target exists**: the chart opens with focus on HOME; no hop, no refusal, no simulation effect (SOLO also requests the pause, CT-4). An auto-implied hop would make one press both the choice and the commitment (CT-10; WM:1225).
- **EC-CT-4. If the pod console is pressed while the pod is at a derelict**: HOME is implied and the hop proceeds with no chart involvement; the pick is not read. If the chart is open instead, HOME is the only focusable node, the dashed line runs from the pod's node to HOME, accept on HOME does nothing and `ChartPick` stays NONE. The only legal move needs no choice (WM-4, WM-8; CT-10).
- **EC-CT-5. If accept lands on the already-picked node, on HOME while the pod is at HOME, or on a non-selectable node**: the picked node stays picked (idempotent); HOME clears the pick; a non-selectable node (ORPHAN, `berthed`) does nothing. A double press must never clear a pick by accident (CT-7).
- **EC-CT-6. If the picked record changes status on the same presentation frame as the console press**: the Adapter injects whatever the pick held when the press was processed, and Web Map & Transit's own reasons decide (`TARGET_ABSORBED`, `TARGET_BERTHED`, `TARGET_UNKNOWN`), shown as words on the register plate by Interaction. The chart never re-validates a command; it only mirrors selectability (CT-9).
- **EC-CT-7. If every derelict is ABSORBED or ORPHAN (the legal set is empty)**: the chart opens with HOME as the only focusable node, no message, no "all charted" line; a no-pick console press still opens the chart. There is no completion state and no win state (anti-pillar; WM guard rail 5).
- **EC-CT-8. If an unwelded derelict B is `berthed` and the player picks a different derelict A**: A stays selectable and the pick is accepted; the console refuses the hop with `BERTH_PENDING` in words. The mirror does not copy WM-9's reason order (CT-9; OQ-CT-3).
- **EC-CT-9. If a fuel level, a hop timer or any other Graph value changes while the chart is open**: selectability, focus and drawing are unchanged. The mirror never reads `FUEL_LOW`, `BUSY`, `OUT_OF_RANGE`, `NOT_LIT`, `ARRIVAL_BLOCKED`, `CAPACITY` or `BERTH_PENDING`; a fuel-dependent greying would be a fuel meter on the truth layer (CT-9, CT-35; AC-CT-24).

### B. Pause, input and focus

- **EC-CT-10. If Esc (or Start) is held and key echo arrives**: only the first non-echo press toggles the layer; echoes are ignored. A held Esc must not flicker the layer open and shut (CT-41(3); key-echo detection is NOT SOURCEABLE in the engine reference, ADR candidate (e)).
- **EC-CT-11. If `chart_table`, `cancel`, release, pointer exit or window focus loss occurs during a hold-to-confirm fill**: the bar snaps to empty in the same frame and nothing else happens; a second `chart_table` press then closes the layer. One press must not abort a destructive gesture and also resume the simulation (CT-5, CT-22, TX-10).
- **EC-CT-12. If `chart_table` and `cancel` arrive on the same frame**: they are processed in that order, `chart_table` closes the whole layer, and the `cancel` is consumed with nothing to step back to. One frame, one effect (CT-5).
- **EC-CT-13. If `chart_table` is pressed twice inside one tick boundary (SOLO)**: the layer's own `open` boolean toggles twice and `PauseGate` re-issues the latest desired state until the kernel state matches, so the layer is never wedged half-open. Whether the kernel coalesces or executes a same-boundary `request_pause` and `request_resume` in order is ASK-CT-9; the layer needs only that it converges (CT-4).
- **EC-CT-14. If the last-used device switches mid-navigation**: glyphs swap after `DEVICE_SWITCH_HYSTERESIS`, key and pad focus is retained, the bracket hides on mouse motion and reappears on the next key or pad navigation (CT-6; PC-13). Mixed input is permitted and must not lose the player's place.
- **EC-CT-15. If the mouse hovers item X while the pad focuses item Y**: X's label is Tungsten, the bracket is on Y, pad accept activates Y, and a click activates X and moves key and pad focus to X while the bracket stays hidden until the next key or pad input. Godot 4.6 split the two focus kinds, so both are tested (CT-6; `modules/ui.md`).
- **EC-CT-16. If a click begins on one item and the button is released on another (or off all items)**: nothing activates and no fill remains. A click is an activation only when pointer-up lands on the item that received the press (CT-6).
- **EC-CT-17. If the window loses focus with the layer open (mid-hold, mid-navigation, or idle)**: any hold fill goes to empty, navigation state is cleared, the layer stays open with the previous focus on return, and the kernel state is unchanged (no auto-resume). Focus-out notifications are NOT SOURCEABLE in the engine reference (CT-41(5); ADR candidate (e)).
- **EC-CT-18. If a zoom press (wheel, `=`, `-`, d-pad up or down) arrives while the layer is open**: it is discarded; the wheel scrolls only a focused scrolling control, and none exists at MVP; no hint or glyph is shown. Zoom is not an action of the `chart_table` context and never a setting (CT-5; PIPE:688, PIPE:232).
- **EC-CT-19. If the layer was opened by a console press and that press releases or auto-repeats**: nothing on the layer activates; every activation needs a press edge that begins after the open frame. Otherwise the LMB release that follows a console press could activate whatever is under the cursor (CT-41(1)).
- **EC-CT-20. If `interact` on Resume closes the layer while the player stands at the pod console**: the closing press and its release are consumed and gameplay sees no `interact` edge until a fresh press, so no `pod_hop` is submitted by the same finger. The next `interact` edge at the console is the irreversible act (CT-41(2); WM:1225).
- **EC-CT-21. If the stick rests on the deadzone boundary during chart navigation**: the exit threshold stops chatter, one move is taken per direction edge, and repeats follow D.9; a held direction never wraps past the last node. The four-way cardinal read is this layer's own (CT-5).
- **EC-CT-22. If a gamepad disconnects while the layer is open**: the layer stays open, glyphs switch to the last remaining device (keyboard if none), and no pause or close is forced. A disconnect with the layer closed is the Controller's (PCI), not this layer's.
- **EC-CT-23. If the player clicks the frozen world outside the panel**: nothing happens. The world is a held frame, and a click there would read as an in-world action (CT-41(4)).

### C. Transit and the chart

- **EC-CT-24. If the chart opens during LEG or ARRIVING**: it opens as `CHART_VIEW_ONLY` with focus on Back; nothing is focusable or activatable; the `IN TRANSIT` word shows iff `pod_at == IN_FLIGHT`; on an OUT leg the route HOME to `web.view.leg_target_id` is drawn (never from the pick); on a BACK leg no route is drawn (the origin is not published); `web.leg_timing()` is never called. The chart never reconstructs a target from its own pick (CT-15).
- **EC-CT-25. If the chart opens during BERTHING**: no state word and no progress element are shown, the pick is frozen and nothing is drawn from it. A berth is witnessed by the world (WM:1307).
- **EC-CT-26. If the layer is opened during the Pipeline's `TRANSIT_HOLD` (layer-1 black)**: the panel appears over the black hold; in SOLO the pause freezes `T` as always (CR-16), so the leg does not advance while the panel is up; Resume returns to the hold. The pause is available and "changes nothing" the player can use (WM:1268; PIPE:264).
- **EC-CT-27. If `transit_state` leaves IDLE while the chart is open (OVERLAY only; in SOLO `T` is frozen)**: `CHART` becomes `CHART_VIEW_ONLY` (TX-28), focus moves to Back and the pick is frozen; on return to IDLE the pick has been reset by arrival and the state returns to `CHART` with focus on HOME (TX-29). No message in either direction.
- **EC-CT-28. If `web.view` carries `leg_target_id` and `leg_dir` in flight (WM-36)**: on an OUT leg the route HOME to `leg_target_id` is drawn only from them, never from the pick (CT-15); on a BACK leg, or if `leg_target_id` reads NONE, no route is drawn and the `IN TRANSIT` word stands alone.

### D. Saving, quitting and failures

- **EC-CT-29. If Save & Quit is accepted while any bound seat is `DOWNED` or `DEAD`** (the `DEAD` case is unreachable in the solo MVP, SL-6): nothing is written, `UI3_PLATE` opens with Back as the default focus, the string states that the newest autosave will be used, and Quit without saving needs the full hold. This is Save/Load's gate (`SL_QUIT_NO_WRITE_DOWNED`); the layer adds no death language (CT-21; SL UI-3).
- **EC-CT-30. If Save & Quit is accepted in the first tick after opening, before the kernel confirms `PAUSED`**: the item is disabled (Graphite lit) because `can_save_quit` is false until the kernel confirms, so no request is made. The fade (`CHART_FADE_MS`, inside the ART 7.3 band) hides the gap (CT-4).
- **EC-CT-31. If the write is held past `SAVE_QUIT_TIMEOUT_MS` or `save_status` reads `FAILED`**: the layer is the panel with the plain word `FAILED` in the newest-record line position; the four items stay in CT-3 order; Resume, Chart, Display & Comfort and `chart_table` work. The word is never colour alone and never Alarm Red (CT-21; HR-4; SL:785).
- **EC-CT-32. If Retry fails again**: the panel returns with `FAILED`, with no counter, no escalation text and no changed colour. Accepting Save & Quit opens `UI3_PLATE` each time, so Quit without saving stays one hold away. A save is not an instrument and the layer adds no affect (CT-21).
- **EC-CT-33. If `chart_table` or `cancel` is pressed while the write runs (`SAVE_QUIT_WAIT`)**: ignored; nothing is added to the screen (no spinner, ring, bar, percentage or text change). A half-written state is never abandoned, and SL H5 forbids a write indicator (CT-5, CT-21).
- **EC-CT-34. If Save/Load's `save_and_quit` finds the state hash unchanged** (`SL_QUIT_DEDUPED`, SL:784): it writes nothing and calls `request_shutdown` itself (SL-6); the layer shows nothing different and calls nothing. The layer therefore cannot tell a written quit from a skipped one, and must not try to (CT-21).
- **EC-CT-35. If the UI-3 hold completes**: the layer calls `request_shutdown` once with no write, and further input is ignored for the frames before the process ends. The newest autosave stands (CT-21; TX-26).
- **EC-CT-36. If the kernel enters `FAULTED` in a release build with the layer open**: the layer closes and the boot-flow main menu opens with Continue offered and focused; no pause is requested and no string is invented (OQ-CT-5). In `FAULTED` the kernel never autosaves over an existing save, so the newest verified generation is intact (CT-26; KRN:151).
- **EC-CT-37. If Quit is accepted on a boot-flow screen**: the kernel is not in a state that lists `request_shutdown` (it lists RUNNING, PAUSED and FAULTED), so the process ends through boot code with no kernel call (provisional — ASK-CT-11, kernel and Save/Load); Quit on the FAULTED-return menu keeps `request_shutdown` (TX-30).
- **EC-CT-38. If the operating system's window-close request arrives**: the layer calls nothing and adds nothing to the screen. With the layer closed (kernel `RUNNING`), Save/Load's own handler runs `save_and_quit()` from `RUNNING` and writes a `QUIT` generation, or dedupes (`SL_QUIT_DEDUPED`) or writes nothing while a seat is `DOWNED`, then calls `request_shutdown` (SL-6, AC-SL-44). With the layer open in SOLO (kernel `PAUSED`), SL-6 does not name the path: provisional, folded into ASK-CT-11 to Save/Load; degraded: the layer closes and defers to Save/Load's handler. The layer is never a second save path (SL-24).

### E. Boot, campaigns and settings

- **EC-CT-39. If Continue finds the newest generation corrupt**: Save/Load falls back to the one before it, and the UI-7 line "Newest record unreadable. Continuing from the one before it." (placeholder copy) is shown under the Continue entry iff `campaign_info()` reports the failed verification (provisional — ASK-CT-5), otherwise it is the HUD & Suit Layer's on the first `RUNNING` frame. No repair is offered (CT-25).
- **EC-CT-40. If no readable record exists (`SL_LOAD_NO_LOADABLE_SAVE`)**: the boot-flow UI-8 plate shows New campaign (default focus) and Quit; no pause is requested; the layer re-enabled itself from `LOAD_HOLD`. Nothing is offered that would imply a save exists (CT-25; C7).
- **EC-CT-41. If the save is from an incompatible build**: the UI-9 plate shows the message with Quit and New campaign; nothing is overwritten or repaired, and New campaign starts a fresh campaign without touching the incompatible file (SL:752). The player's old record is kept for a build that can read it (CT-25).
- **EC-CT-42. If New campaign is chosen with an existing campaign**: UI-5 opens with Retire as the default focus; Delete is a separate item under the hold gesture; `cancel` returns to the main menu with focus on New campaign. The destructive item looks like every other item and is distinguished only by the bar (CT-24, CT-22).
- **EC-CT-43. If a Delete hold is released early**: the bar is empty, nothing is deleted, and a new gesture needs a new press edge (CT-22). Retire stays reversible by file only; there is no un-retire screen at MVP (OQ-CT-4).
- **EC-CT-44. If `continue_campaign()` or `new_game(world_seed)` returns a failure after the black frame**: the layer re-enables itself on the boot screen it came from, with Save/Load's plain message where one exists (UI-7, UI-8, UI-9); no message string exists for a `new_game` failure at MVP, and the layer invents none (provisional — OQ-CT-17). Nothing is drawn between the black frame and the first `RUNNING` frame otherwise (CT-42).
- **EC-CT-45. If the first launch is repeated (the settings store lost or unreadable)**: the photosensitivity notice appears again, identical, with the two toggles at the pipeline defaults; nothing else differs. Showing it again is the safe direction (CT-19, CT-20).
- **EC-CT-46. If the settings file is missing, corrupt or read-only**: defaults load; a write failure keeps the values in memory and retries at the next flush with no player message at MVP; a corrupt file is rewritten at the next flush. Nothing in the simulation reads the store (CT-19).
- **EC-CT-47. If a requested display mode fails (a fullscreen request is refused)**: the page reads back and shows the applied mode (windowed), the store records the applied mode, and no error text appears. `Window` mode failure reporting is NOT SOURCEABLE; the behaviour required is the read-back (CT-18).
- **EC-CT-48. If a display-mode change resizes the window while the page is open**: `k`, `H_default`, the canvas and `W_panel` recompute live; the page stays open; focus is retained by item id; the bracket does not nudge; no animation plays. A resize that moves the chart multiplier across 1 switches spatial and compact at the resize event, never mid-fade, keeping the pick and the focused record by `derelict_id` (CT-18, CT-44).
- **EC-CT-49. If reduced motion is toggled while the chart is open**: the drift accumulator stops at its current whole-pixel offset (no snap-back), bracket moves become cuts at once, fades are unchanged, and no animation plays for the change itself. Toggled off, the drift resumes from the held accumulator with no jump (CT-13; D.5).
- **EC-CT-50. If reduce flashing is toggled on while a plate is mid-transition**: the transition in progress completes as it was; later replacements use the long end of the fade band and never a hard cut of a large plate. Nothing is retroactive (CT-3, CT-13).

### F. Layout, display and strings

- **EC-CT-51. If the canvas is the smallest (213.33 x 120)**: the chart is the compact list (`m = 0`), the panel is 71 HUD px wide, panel strings follow their fit classes, the panel's stacked height is 83 of 108 HUD px, and the hold bar is not constrained because it sits on a full-bleed plate. The layer must be complete here, not degraded (D.1, D.1b, CT-16).
- **EC-CT-52. If the canvas is fractional (426.67 x 266.67)**: every layer-3 rectangle is anchor 0 or 1 plus integer HUD-px offsets, the panel hangs from the right anchor at `W_panel = 142`, and the screenshot shows no 1-px seam or bezel inconsistency (CT-16, D.1).
- **EC-CT-53. If the canvas is exactly 320 x 180 (1080p at `2 * H_default`)**: `m = 1`, the layout is spatial, and the frame fits the safe margin with no slack (`x0 = 16`, `y0 = 9`). The integer form of D.1b is what keeps this boundary stable (a floating `0.9 * 320` need not be exactly 288).
- **EC-CT-54. If `chart_layout.json` has no entry for a record**: at runtime the record takes the deterministic outer-ring fallback slot `(ordinal - 1) mod N_FALLBACK_SLOTS`, a log line is emitted, and the content lint fails the build. A missing position must never hide a node (CT-12).
- **EC-CT-55. If a future roster has more nodes than the MVP's three and the compact list cannot fit**: nothing scrolls at MVP, so the layout lint fails the build (the canvas minimum was mis-set); paging or scrolling of the list is OQ-CT-14, a Vertical Slice question. Row order never changes to make it fit (CT-44).
- **EC-CT-56. If a localised string overflows its fit class**: the build lint fails; at runtime the string wraps within its cap and is never truncated or ellipsised. A clipped "Save & Qu" on the truth layer would read as a lie (CT-28, D.8).
- **EC-CT-57. If `SaveInfo.playtime` is 0, or 100 hours or more**: the line reads `00:00:00` or `100:00:00` (hours grow past two digits without wrapping); no wall clock is read (D.7).
- **EC-CT-58. If one frame delta is zero or very large (a window drag, a display-mode change)**: the delta is clamped to `UI_DT_MAX_MS`, so a stall cannot complete a hold or jump the drift, and a zero delta changes nothing (CT-43; D.2, D.5).

### G. Hallucination and the truth channel

- **EC-CT-59. If a hallucination is on screen when the panel opens (SOLO)**: the world stays frozen and undimmed exactly as lit, including the lie (a phantom, a gauge reading fixed, a violet fringe); the panel is true. This is intended (`PAUSED_FROZEN`, PIPE:265), documented here and in the QA note so a tester never reads a frozen phantom as a menu fault and never "fixes" it with a dim (CT-2; R17; C11).
- **EC-CT-60. If sanity is forced to stage 3 (`force_stage`) with the layer open**: layer-3 pixels are identical to stage 0 apart from the world visible beneath the panel; no gleam, glide, tremor or settle appears on any element; no Violet or Flesh appears (CT-2; AC-SN-83).
- **EC-CT-61. If a false system state (class (b)) is showing on a world instrument when the player pauses**: the panel carries nothing about systems, so the false reading is unchanged and checking it still costs a look or a risk. A true copy of a falsifiable fact on a free, pausable menu would make the lie free to disprove (CT-39).
- **EC-CT-62. If a phantom entity (class (a)) or a crewmate rendered as a horror (class (c)) is on screen when the layer opens**: no chart or menu element can confirm or deny it (no entity, proximity, direction or crew glyph exists at MVP); the reserved crew glyph (K1) is a VS-B decision, not an MVP edge (CT-32, CT-38).
- **EC-CT-63. If the player opens the chart hoping to verify salvage or a loot value (class (d))**: the chart shows what the crew lit and nothing about contents, value or remaining count; the crate is still checked by weight, a second scanner or the trip home (CT-33).
- **EC-CT-64. If the layer is open with a world instrument lying beneath (OVERLAY)**: the world keeps running and keeps lying under the opaque panel; the panel is not a refuge and not an oracle. In OVERLAY the panel also hides the bottom-right hand rail (reserved risk R-AD-6, routed to the HUD and Session).

### H. Save, restore and determinism

- **EC-CT-65. If an autosave or snapshot occurs while the layer is open**: the layer's open state, focus, pick and settings are in no snapshot; a restore rebuilds the machine from `CLOSED` with `ChartPick = NONE`. Opening or closing the layer is therefore invisible to the save format (CT-27; TX-27).
- **EC-CT-66. If the same campaign is replayed with the layer opened, picked and closed at arbitrary frames (SOLO) and again with none**: `state_hash` at every `T` is identical. The layer registers nothing and submits nothing; only the console press (Interaction's) changes the simulation (CT-27; kernel D.12).
- **EC-CT-67. If Continue is accepted twice in a row (a double click)**: the second accept finds all items disabled (CT-42 step 1) and does nothing; one restore runs.

### I. Degenerate strategies

- **EC-CT-68. If the player mashes Esc to read the frozen world repeatedly**: SOLO pause stays free and unrationed (risk K2 accepted); there is no cooldown on `chart_table`; the open and close of the plate is a single transition each inside the ART 7.3 fade band, and a rate limit for the Chart item only (`CHART_REOPEN_MIN_MS`) is the scoped fallback of ASK-CT-8 and is not a rule at MVP. In OVERLAY the sim runs under the plate, so there is no free scan (CT-3, CT-4).
- **EC-CT-69. If the player tries to price a hop from the length of the dashed line or the node layout**: the layout is authored cosmetic placement with no correlation to any Web Map & Transit cost (content lint), no label is drawn, and the dashed line is the pick mark and nothing else. Fuel is witnessed by the Graph's world instruments (CT-12, CT-14, CT-35).
- **EC-CT-70. If the player uses Save & Quit, Continue, or Quit without saving to reroll an outcome**: Continue restores the stored state, and every random stream is derived from the world seed and a counter held in state (kernel D.10), so a reload reproduces rather than rerolls; the menu offers no seed field (OQ-CT-6). What a reload costs in time is Save/Load's cadence, not this layer's.
- **EC-CT-71. If the player treats the pause as a safe room**: the pause holds the world exactly as lit and carries no relief, no recovery and no change to sanity (`T` is frozen; Sanity reads nothing from the layer); the layer says nothing reassuring. The validation test "the pause reads as held, not safe" is a playtest check (Player Fantasy, validation 3).
- **EC-CT-72. If the player retires and restarts a campaign to avoid a failure cost**: the retired file stays on disk (reversible by file only), New campaign starts a fresh world, and P5's cost of failure (ship and body, never knowledge) is Failure & Trail's and Save/Load's to enforce, not a menu rule. The layer does not judge the choice and adds no warning chrome (CT-24).

### J. Co-op hooks (reserved, VS-B)

- **EC-CT-73. If `chart_table` is pressed in OVERLAY (reserved)**: the same panel and chart open with the kernel `RUNNING` and no kernel call (TX-14); only `PauseGate` differs; `can_save_quit` is replaced by Session / Save-Load's form. No chart, pick, panel or menu rule reads the kernel's `PAUSED`, which is the reason this case needs no other change (CT-4; AC-CT-2).
- **EC-CT-74. If two seats have the layer open in OVERLAY**: each seat's `ChartPick` is local to that seat's layer; only the seat that presses the console injects its own pick into the command; a guest's pick never reaches the host. The pick is host-local presentation state, not a replicated field (CT-7, CT-10; WM-8).
- **EC-CT-75. If a restore arrives while a guest has the layer open (OVERLAY)**: every open layer closes into `CLOSED` and resets its pick (TX-27); no seat is left on a stale chart.
- **EC-CT-76. If the pod arrives (`POD_ARRIVED`) while a seat has the chart open in OVERLAY**: `pod_at` leaves `IN_FLIGHT`, the pick is reset, selectability is recomputed from the new `web.view`, and `CHART_VIEW_ONLY` returns to `CHART` with focus on HOME (EC-CT-27; CT-8).

## Dependencies

Direction is stated from this GDD's side: **upstream** = this layer depends on it; **downstream** = it depends on this layer; **lateral** = a shared boundary with no data dependency. Strength: **Hard** = the layer cannot function (or the neighbour cannot function) without it; **Soft** = degraded behaviour exists and is named. Status words: **designed** (a GDD file exists headed Designed), **in-flight** (a scratch directory or draft exists with no contract), **undesigned-provisional** (no file; assumptions are marked provisional wherever they appear). Status is as of the Glob at authoring; files designed in this run may change while sibling agents work.

| System | Direction | Strength | Interface | Status | Degraded behaviour / note |
|---|---|---|---|---|---|
| Web Map & Transit | upstream | Hard (the chart's whole content) | `web.view` read-only (WM-36); `pod_hop{target_id}`, `berth{derelict_id}` are submitted by the console press, not by this layer | designed (this run; `web.view`, `web.state`, `transit_state` are marked provisional by WM) | a rename changes only the cite; the in-flight route is drawn only from `web.view.leg_target_id` and `leg_dir` (OUT leg; none on a BACK leg), else `IN TRANSIT` stands alone (CT-15) |
| Pixel-Perfect Render Pipeline & Camera | upstream | Hard | layer 3 in the root viewport (PR-3), `H` / `H_default` (PR-12, D.4), `PAUSED_FROZEN`, `TRANSIT_HOLD`, the reduced-motion and reduce-flashing hooks (PR-13), the 5 % safe margin | designed | none; the layer cannot exist without the root-viewport layer and the paused-frozen rule |
| Command & Simulation Layer | upstream | Hard | host-local control plane `request_pause` / `request_resume` / `request_shutdown` (CR-15); pause semantics (CR-16); presentation clock (CR-11); `INPUT_SURFACE` (KRN:111) | designed | not listed in the index row (C5); VS-B overlay removes the pause call (OVERLAY mode) |
| Save/Load & Versioned Format | upstream | Hard | SL-21 API, `SaveInfo`, `save_status`, `restore_reason`, SL-6, UI-2..UI-9 (SL:998-1005) | designed | not listed in the index row (C5); without it there is no Continue, New campaign or Save & Quit |
| Player Controller & Input | upstream | Hard | the `chart_table` context (PC-2), `chart_table` / `cancel` / `interact`, PC-13, PC-16, `CONFIRM_HOLD_S`, `BINDING_REQUIRED{chart_table}` | designed | navigation and accept actions are provisional (ASK-CT-2); not listed in the index row (C5) |
| Interaction System | upstream | Soft | the console seam: the Adapter injects the host-local pick into `pod_hop` and routes a no-pick press to "open chart" (CT-10); refusal words on the register plate | designed (seam provisional, ASK-CT-1) | the press refuses with a words-only reason from Interaction's table; no simulation difference |
| Suit & Equipment | upstream | Soft | `new_game` seat defaults (`create_seat_beam`, `bind_beam` with `BEAM_WIDE` for seat 1; SE-9) | designed | no picker at MVP; the seam is reserved |
| Adaptive Audio | upstream | Soft | UI-bus cues requested (AA:562) | designed | menus silent at MVP; acceptable |
| Attention System, Horror Entities, Boarding & Salvage, Sanity & Hallucination Rules, Perception Filter | upstream (negative requirements) | Soft | no data; the negative rules CT-31..CT-35, CT-37, CT-2 | designed | each names this GDD and demands the rule be repeated or a CONFLICT raised (AT:1373, HE:377/:1358, BS:334, SN:1350/:1357, PF:223); one conflict (BS:1204, C1) is resolved in WM's favour |
| Item & Module Database, Derelict Authoring, Life-Support Graph, Ship Section & Layout, Body, Assembly, Emitter State, 2D Lighting, Failure & Trail | none (boundary only) | n/a | no interface; no inventory or loadout screen, no authored chart prop, no Graph overview, no assembly UI (CT-37, CT-1) | designed | listed so a reviewer does not look for an edge; Assembly's weld prompt, Lighting's HR-15 ambient step and Failure's trail are world surfaces, not meta |
| Settings & Input Rebinding | downstream | Hard (IDX:210: depends on this GDD) | `SettingsStore` interface, the Display & Comfort page it replaces, the `chart_table` rebind guard | undesigned-provisional (Alpha) | MVP settings stay a three-control page; full PR-14 UI, rebinding, HUD scale and zoom-exclusion are Settings' |
| Accessibility Settings | downstream | Soft | hold-duration option and two-tap, reading-face scale, HUD opacity floor, AccessKit wiring | undesigned-provisional (Alpha) | structural compliance only at MVP (CT-30) |
| Co-op Session & Replication; Chat / Voice / Pings | downstream | Soft | `PauseGate` OVERLAY, a guest's pause or save request (KRN CR-15), reserved crew glyphs and left-edge chat | undesigned-provisional (VS-B) | MVP unaffected; risk K1 routed |
| Knowledge Codex & Logs; Failure Contract & Re-wake; Onboarding; Procedural Derelict Generation | downstream | Soft | reserved slots (CT-38) | undesigned-provisional (VS / VS-A) | none at MVP |
| HUD & Suit Layer; In-World Instruments & Inspect View; Debug Overlay | lateral | Soft | shared component grammar (`design/ux/interaction-patterns.md`); boundary of ownership (CT-1, C4) | HUD: in design (file on disk, Status In Design; its rows HS-36, HUD:300, HUD:341, HUD:402 and HUD:459 name this GDD and agree); Instruments: in design (file on disk, Status In Design; IWI:126 rejects the lit chart-table console, consistent with C4, and IWI:531 corrects the context owner to the Controller's Adapter); Debug Overlay: designed (`debug-overlay-true-values.md`; its rows DO:416, DO:776 and DO:1029 name this GDD and agree; layer 4 per DO-5, DO-26) | HUD:304 states the HUD's UX spec lands first and defines `design/ux/interaction-patterns.md`; this layer cites it |
| Systems index (`systems-index.md`) | documentation | n/a | rows IDX:40, :95, :201, :210, :239 | designed | needs correction (C5): add the kernel, Save/Load, Controller (hard) and Suit, Interaction, Adaptive Audio (soft) edges |

**Bidirectional requirement.** The designed GDDs that must name this system and already do: Web Map & Transit (WM:420, WM:888), Save/Load (SL:477, SL:993, SL:1012), the Pipeline (PIPE:296, PIPE:776), the Controller (PCI:78, :147, :217, :699), the kernel (KRN:111, :129, :961), Suit & Equipment (SE:189, :476), Adaptive Audio (AA:562, :1086), Attention (AT:1373), Horror (HE:377, :1358), Boarding (BS:334, :1204, AC-BS-47c), Sanity (SN:1350, :1357), Perception Filter (PF:223); HUD & Suit Layer (HS-36, HUD:300, :341, :402, :459; Status In Design), In-World Instruments (IWI:126, IWI:161; Status In Design) and Debug Overlay (DO:416, DO:776, DO:1029; designed, `debug-overlay-true-values.md`). Not yet naming it, and routed: Interaction (its `pod_hop` row points at Web Map; the seam is ASK-CT-1) and the systems index (C5). Rows that need wording refreshes in neighbours, none edited here: BS:1204 (C1), PIPE row 7 and PIPE:269 (C2, C14), KRN:715 (C3), SL:1004 and SL:770 (C7), SN:1350 / :1357 (C8), WM:1309 (C13), PCI:656 (C15), the registry's four `design/gdd/chart-table.md` comments (C6).

**Dependency cycle check.** This layer depends on Web Map & Transit and the Pipeline (and on foundation systems below them); neither depends back on this layer except Web Map & Transit's *documentation* of what it owes (WM:888) and the Pipeline's Hard "needs the root-viewport layer" row (PIPE:776), both one-directional data dependencies. Settings & Input Rebinding depends on this GDD and nothing in this GDD depends on it, so no cycle exists.

## Tuning Knobs

**How to read this section.** The Chart Table has few gameplay numbers (it owns no simulation state, CT-27), so its knobs are layout, timing-of-presentation and content-lint values. Every owned value is `[PLACEHOLDER]` until a playtest and a spike on the real faces and display path. A **★** marks a registry candidate, meaning a constant that another document may need to cite by name; the merge stage writes registry entries from the final text, not from this draft (reflexion log, 2026-10-26), and only the owned names are registered: **no art-bible number is registered under a new name** (a second owner of an art-bible number is the restated-constant failure; rulings R28). Cited rows belong to another document, are listed so a reader can find the source of truth, and are **not retuned here**; registry constants are cited by name, never by registry line. Values that are named in Web Map & Transit's negative sentences (fuel and leg timing) are deliberately not repeated: the chart displays none of them (CT-35). Nothing on this layer is tuned by sanity stage, Attention, depth or session time (CT-40).

### Owned knobs

| Knob | Default | Unit | Safe range | Too high | Too low | Interacts with | Owner and status |
|---|---|---|---|---|---|---|---|
| ★ `CHART_FADE_MS` | 100 | ms | 80-120 (the ART 7.3 band) | the panel feels sluggish; a longer fade is a longer luminance change on a third of the screen | abrupt cut; a hard cut of a large plate is what reduce flashing forbids | CT-3, CT-13; reduce flashing uses the long end of the band; HR-13 | this GDD, `[PLACEHOLDER]`, inside a cited band |
| ★ `COMPACT_LAYOUT_MIN_CANVAS_W` | 320 (derived) | HUD px | derived only: `(10 * CHART_FRAME_W) div 9`; never edited alone | compact list used on canvases where the map fits | the map clips the safe margin | `CHART_FRAME`, D.1b, CT-44 | this GDD, derived from the frame (checkpoint ruling Q8) |
| ★ `CHART_FRAME` (W x H) | 288 x 162 (derived) | art px | derived: 0.9 of the smallest spatial canvas; any change re-derives the breakpoint and its height twin 180 | fewer canvases stay spatial; compact appears earlier | node room shrinks; `S_MIN` lint failures | `COMPACT_LAYOUT_MIN_CANVAS_W`, D.1b, CT-12 | this GDD, derived (checkpoint ruling Q7) |
| ★ `SNAP_SECTOR_DEG` | 90 | degrees | 60-120 | primary sectors overlap; ambiguous diagonals | gaps between sectors; the half-plane fallback does all the work | D.3, CT-44 | this GDD, `[PLACEHOLDER]` |
| ★ `FILAMENT_DRIFT_STEP_PX` | 1 | art px | 1-2 | visible hops of the filament sheet | n/a (1 is the finest whole-pixel quantum) | D.5; `FILAMENT_DRIFT_FRACTION` | this GDD, `[PLACEHOLDER]` |
| ★ `FILAMENT_DRIFT_FRACTION` | 0.5 | fraction of the cited drift ceiling | 0.25-1.0 (never above 1.0, which is the ART 7.3 ceiling) | the drift reads as motion with intent; at 1.0 it sits on the art bible's limit | the chart reads dead and static (ART 2.11 needs "a slow drift") | D.5; reduced motion sets it to 0; CT-40 | this GDD, `[PLACEHOLDER]` (ruling: fraction over a raw rate) |
| ★ `CHART_RING_STEPS` | 3 | ring bands | 1-5 by the contract; MVP content ceiling 3 | rings 2-3 and beyond read as content (WM:1050 calls them horizon) | no inward pull (ART 2.11 carried element) | D.6, `RING_STEP_MIN_CHANNEL`, content in VS | this GDD, `[PLACEHOLDER]` |
| ★ `RING_STEP_MIN_CHANNEL` | 3 | 8-bit units per channel | 2-6 (bounded by the 16-unit Ground-to-Void-Steel span) | caps the number of rings; steps start to read as bands | steps vanish after 8-bit quantisation or on a poor panel (flag F-6, C16) | D.6, `CHART_RING_STEPS`, AC-CT-47 | this GDD, `[PLACEHOLDER]` (ordering contract) |
| ★ `LAYER3_DRAW_CALL_BUDGET` | at most the Pipeline's `D_hud` upper bound while open | draw calls | 15-60 | frame cost on the Deck | n/a | D.10, Pipeline D.8, OQ-CT-16, ASK-CT-12 | this GDD, `[PLACEHOLDER]`, measured in the spike |
| ★ `PANEL_INNER_PAD_PX` | 4 | HUD px | 2-8 | steals width at the 71-px panel; fewer characters per line | text touches the chamfer or rivets | D.1, D.8, CT-28 | this GDD, `[PLACEHOLDER]` (`PANEL_PAD_HUD` retired) |
| ★ `UI_DT_MAX_MS` | 50 | ms | 33-100 | a stall can nearly complete a hold or jump the drift | the hold bar slows below 20 fps | D.2, D.5, CT-43 | this GDD, `[PLACEHOLDER]` |
| ★ `SETTINGS_WRITE_DEBOUNCE_MS` | 500 | ms | 200-2000 | the last toggle is lost on a crash | many small writes | CT-19 | this GDD, `[PLACEHOLDER]` |
| ★ `UI_NAV_REPEAT_MS` | 350 | ms | 250-500 | sluggish first repeat | accidental double steps | D.9, `UI_NAV_REPEAT_RATE_MS` | this GDD, `[PLACEHOLDER]` |
| ★ `UI_NAV_REPEAT_RATE_MS` | 120 | ms | 80-200 | sluggish repeat | skips past the target node | D.9 | this GDD, `[PLACEHOLDER]` |
| `L_BAR_HUD` | 64 | HUD px | 32-96, capped at `P_w div H - 2 * safe_x` | the bar crowds the plate | needle steps too coarse | D.2, CT-22 | this GDD, `[PLACEHOLDER]` |
| `PANEL_ITEM_H_HUD` | 16 | HUD px | 12-20 | row 6 vertical fit fails | cramped hit areas | D.1 | this GDD, `[PLACEHOLDER]` |
| `PANEL_ITEM_GAP_HUD` | 3 | HUD px | 2-6 | vertical fit | items read as one block | D.1 | this GDD, `[PLACEHOLDER]` |
| `PANEL_RECORD_LINE_H_HUD` | 10 | HUD px | 8-12 | vertical fit | the newest-record numerals clip | D.1, D.7 | this GDD, `[PLACEHOLDER]` |
| `PANEL_LINE_EXTRA_HUD` | 8 | HUD px | 6-10 | the wrapped-line allowance shrinks | wrapped labels collide | D.1 (`n_wrap`) | this GDD, `[PLACEHOLDER]` |
| `BRACKET_PAD_HUD` | 4 | HUD px | 2-6 | `S_MIN` grows; nodes cannot all fit | the bracket crosses the glyph | D.1b | this GDD, `[PLACEHOLDER]` |
| `S_MIN` (derived) | 32 | HUD px | 28-36 | layout-lint failures at the smallest frame | brackets overlap neighbours | D.1b, CT-12 | derived: `24 + 2 * BRACKET_PAD_HUD` |
| `TEXT_EXPANSION_TEST` | 1.3 | factor | 1.2-1.5 | false lint failures | the lint is too lax for localisation | D.8, CT-28 | this GDD, `[PLACEHOLDER]` |
| `N_FALLBACK_SLOTS` | 6 | slots | 3-8 | n/a | fallback collisions between unplaced records | EC-CT-54, CT-12 | this GDD, `[PLACEHOLDER]` |
| `ROUTE_DASH` (on / off) | 2 / 2 | art px | 1-3 | a coarse dash reads as a solid rule on a short route | a fine dash aliases | CT-7, CT-11; the dash pattern is the shape backup for the Tungsten colour | art-director, `[PLACEHOLDER]` (route and dash grammar; asset A16 of the art-director's list) |
| `CHART_REOPEN_MIN_MS` | 334 (derived: `ceil(1000 / 3)` from the cited 3 Hz ceiling) | ms | 250-500 | the Chart item feels unresponsive when mashed | repeat toggling exceeds 3 Hz | CT-3, ASK-CT-8, HR-13 | **not a rule at MVP**; carried as the default form if the art-director rules yes; not registered |
| `PANEL_ITEM_COUNT` | 4 | items | fixed | scrolling at small canvases | n/a | CT-3, D.1 | this GDD, fixed |
| `DISPLAY_COMFORT_CONTROL_COUNT` | 3 | controls | fixed at MVP | scope creep into the Alpha settings UI | the notice has nothing to point at | CT-18 | this GDD, fixed |
| `CHART_TO_PRESS_S` | 20 | seconds, median | playtest observable; flag above 40 | n/a | n/a | Player Fantasy validation 2 | **a playtest observable, not a gameplay constant, not registered** |

### Cited, not ours

| Value | Default (at authoring) | Owner | Why the layer depends on it | Interacts with |
|---|---|---|---|---|
| `CONFIRM_HOLD_S` (registry) | 0.8 s, range 0.6-1.2, two-tap option in Alpha | Player Controller | the hold bar's duration; closes SL OQ-32 by citation | D.2, CT-22 |
| `H_default = max(k, 3)`, `H` range | 3 at the lowest | Pixel-Perfect Render Pipeline PR-12 (registry `hud_multiplier`) | the HUD scale, independent of zoom | D.1 |
| `HUD_MIN_SCALE` (registry) | 3 | Pixel-Perfect Render Pipeline | the floor of `H_default` | D.1 |
| cap height `7 * H` | 7 per H | Pixel-Perfect Render Pipeline PR-12 | the instrument face height | D.8 |
| the safe margin (registry `safe_margin`) | 5 % of the picture, rounded up to whole HUD px per axis | HUD & Suit Layer (registry `safe_margin`; Pipeline PR-2 cites it) | anchors and the frame fit | D.1, D.1b |
| fade band | 80-120 ms | art bible 7.3 | `CHART_FADE_MS` lives inside it | CT-3 |
| bracket ease-out | 100 ms | art bible 7.3 | focus relocation | CT-6 |
| drift ceiling | at most 1 art px per second, current ring only | art bible 7.3 and 2.11 | `FILAMENT_DRIFT_FRACTION` is a fraction of it | D.5 |
| ring-step wording | "about 4 %", not restated; replaced here by an ordering contract | art bible 7.4, WM:1050 (clarification C16) | D.6 | `RING_STEP_MIN_CHANNEL` |
| panel width | a right third | art bible 7.4 | D.1 | CT-16 |
| type floors and line length | menu body 20 / 26, heading 24 / 32, title 32 / 42, label and caption 15 / 20 (1080p / 1440p), 45-75 characters | art bible 7.1 | D.8, CT-28 | OQ-CT-13 |
| glyph sizes and node asset class | 16 and 24 | art bible 7.2 | D.1b | CT-11 |
| flash ceiling | 3 Hz | art bible HR-13 | `CHART_REOPEN_MIN_MS` derivation | CT-3 |
| `DEVICE_SWITCH_HYSTERESIS` | 250 ms | Player Controller (PCI, registry) | glyph swap | CT-6 |
| `MOVE_DEADZONE`, `MOVE_DEADZONE_EXIT` | Controller values | Player Controller (PCI, not registered) | the stick edge for navigation | D.9 |
| `SAVE_QUIT_TIMEOUT_MS` | per Save/Load | Save/Load (SL, not registered) | when `FAILED` shows | CT-21 |
| `TICK_RATE` | 60 | Command & Simulation Layer (registry) | `SaveInfo.playtime = T div TICK_RATE` | D.7 |
| `HOP_FUEL_ML`, `BERTH_FUEL_ML`, `HOP_FUEL_FLOOR_ML`, `RETURN_HOP_FUEL_ML`, `HOP_LEG_TICKS`, `ARRIVE_SETTLE_TICKS` | not repeated | Web Map & Transit (registry) | **named only in negative sentences; the chart displays none** | CT-35 |

### Deferred to Alpha (named so nothing is lost)

HUD scale (by cap height), reading-face scale, HUD opacity floor, the two-tap hold option, rebinding and full PR-14 settings, master volume (OQ-CT-1) and the Records page belong to Settings & Input Rebinding and Accessibility Settings (undesigned-provisional). Zoom is never a setting (PIPE:232).

### Interaction notes for the playtest

- **The three knobs most likely to move** are `FILAMENT_DRIFT_FRACTION` (does the chart read alive or dead), `RING_STEP_MIN_CHANNEL` with `CHART_RING_STEPS` (are the rings visible on a real panel, flag F-6) and `PANEL_INNER_PAD_PX` (does every label fit the 71-px panel in the real face). None is a balance lever: a wrong value costs legibility, not fairness.
- **Linked pairs.** `CHART_FRAME` and `COMPACT_LAYOUT_MIN_CANVAS_W` move together (the second is derived). `BRACKET_PAD_HUD` and `S_MIN` move together. `UI_DT_MAX_MS` limits both the hold bar (D.2) and the drift (D.5), so lowering it slows both at low frame rates.
- **Not a lever, ever.** No knob above is allowed to depend on a simulation value, and no lever for "making the chart more informative" exists: when playtest says the chart is empty, the cure is world legibility (Player Fantasy, tripwire).

## Visual/Audio Requirements

*Source: the art-director's proposal, reconciled with the checkpoint rulings (Q5, Q9, Q13) and Part 1 of this GDD. Requirement ids are `CT-VA-n`; visual rows are `V1`..`V25` (emitted by this layer) and `X1`..`X5` (not ours). Every number is either cited with its owner (art bible `ART`, Pixel-Perfect Render Pipeline `PIPE`, Player Controller `PCI`) or is one of this GDD's `[PLACEHOLDER]` knobs from Tuning Knobs; no art-bible number is restated under a new name (rulings R28). Implementation patterns are marked "-> ADR candidate" and not designed. Class legend follows WM:1009: **T** = truth channel (never wrapped by the Perception Filter or Sanity), **M** = meta register, **L** = lie-capable. Every visual this layer emits is T and M; none is L.*

### Governing statement

**CT-VA-1. Sobriety made checkable.** The chart table is the one layer that cannot lie, so its art job is flat, unlit, Ground-plated, one accent, one motion (the web drifting). Everything on it is drawn from a closed kit (ART 7.4 furniture, ART 7.2 glyphs, the WM section 3 chart rows). Anything not in that kit is a defect, not a design choice. The closed kit is enumerated by the element whitelist of AC-CT-107, so "not in the kit" is a machine-checkable statement.

| ART ref | Applies as |
|---|---|
| 1 Corollary, Principle 1 | The world under the panel stays lit exactly as it was; no dim, blur, desaturation (7.4 Pause) |
| 2.11 | Emotional target (Charted, Quiet, Inward-pulling), energy very low, carried element: concentric darkening toward the centre |
| 3.4 | Frame heavy and static; human objects rectilinear; the web branching with no right angles; UI curves only on dial arcs and needles (none exist on this layer) |
| 4.4, 4.6 | Violet and Flesh are never UI colours; Paper White appears only in UI; Alarm Red only on a failing instrument |
| 4.5, 4.7 | Luminance ladder: hue is never the only channel |
| 7.0 row 3 | Chart table (meta): nobody in the fiction; a reading never lies; the whole layer is honest by definition |
| 7.1 | Two faces, type floors, tabular numerals, 45-75 character reading lines |
| 7.2 | 24x24 menu and map glyph class, keycap grammar, focus is a shape |
| 7.3 | The "Menus and chart table" motion paragraph; the three reading motions (settle, tremor, glide) are absent |
| 7.4 | Register, web-map components, menu components, Pause, Loss, accessibility surface |
| 7.7 | HR-2 opaque ground, HR-4 Alarm Red never text, HR-6 anchors, HR-8 reduced motion, HR-10 no hue remap, HR-11 focus is a shape, HR-12 minimums, HR-13 3 Hz ceiling, HR-14 gamepad completeness |
| 8.2, 8.5 | HUD glyph sizes 16 / 24 and 8x8 marks; file grammar `<cat>[_<dialect>]_<subject>[_<variant tokens>]_<map>.png` (no `map` suffix on `ui_` and `font_`) |

### Truth-channel classification

**CT-VA-2. Nothing here is corruptible.** ART 7.0 row 3 puts the layer outside "who produced the information" (nobody in the fiction), unlit, and "can a reading lie? never"; ART 7.4 states "nothing on them is a reading, so nothing on them can glide or gleam". A corruptible element needs a glass for the stage-1 gleam to sit on (ART 4.6, last paragraph) and a reading to detach; this layer has neither. It is drawn after the Perception Filter slot (PF:223; PIPE PR-3, PR-10) so no per-client filter reaches it, and it is identical at every sanity stage including forced stage 3 (CT-2).

| # | Visual emitted | Source datum | Class | Why it cannot be a lie | Redundant (non-colour) channel |
|---|---|---|---|---|---|
| V1 | Ground full-bleed plate, chamfered Graphite-lit border, four corner rivets | none (furniture) | T / M | decoration of a meta surface; no state | n/a |
| V2 | Depth steps (3 concentric Ground bands plus the centre disc) | none: authored layout; `ring_of` is not read | T / M | authored place, not attention (AT:1373) | step edges are geometry |
| V3 | Filaments (Hull Grey, 1 art px) | none (authored texture) | T / M | decoration; carries no datum; drift is constant (CT-40) | n/a (never encodes state) |
| V4 | HOME node: Tungsten point, rectilinear Foundry silhouette, `ui_glyph_node_home_24` | `web.view.nodes[]` HOME | T | pure read of `web.view` (WM-36) | glyph shape |
| V5 | UNSEEN derelict: plain Graphite-lit rectangle | `status` UNSEEN | T | reveals nothing beyond status (WM-28) | plain rectangle, no tick |
| V6 | SEEN / BOARDED derelict silhouette (Foundry at MVP) | `status`, `template_id` | T | records what the crew lit | dialect silhouette |
| V7 | Unknown tick: 1 art px Tungsten square plus 1 px Ground clearing | `unknown_seen` | T | only once seen from the pod | square shape; word-free |
| V8 | `berthed`: abutting silhouette plus a 1-px Hull Grey seam | `berthed` | T | direct read | seam geometry |
| V9 | ABSORBED: merged into the home silhouette | `status` ABSORBED | T | direct read | silhouette extent |
| V10 | ORPHAN: detached Graphite-mid silhouette, not selectable | `status` ORPHAN | T | direct read | detachment (see R-AD-5 for the dashed outline) |
| V11 | Pod marker `ui_glyph_pod_24` | `pod_at` | T | direct read, never interpolated | glyph |
| V12 | `IN TRANSIT` word (instrument face) | `pod_at == IN_FLIGHT` only (CT-15) | T | a state word, not a progress datum | the word |
| V13 | Planned-sortie route (Tungsten, dashed) | host-local `ChartPick` or `pod_at` (CT-7, CT-10) | T | a plan, not progress | dash pattern |
| V14 | Focus bracket plus 1-art-px bezel thickening | keyboard / gamepad focus | T | UI state, no simulation | bracket and thickening (HR-11) |
| V15 | Pause panel (right-third Ground plate, chamfered list items) | none | T / M | menu furniture | list structure |
| V16 | Newest-record line (`SaveInfo.playtime`, instrument face, static) | `SaveInfo` | T | host-local read, static while open | numerals |
| V17 | Display & Comfort page (chamfered switches, enum row) | `SettingsStore` | T / M | settings are anchors (HR-6) | switch position plus a word |
| V18 | Hold-to-confirm linear bar plus needle-mark | local UI timer | T / M | local input timer | bar fill and needle protrusion |
| V19 | `FAILED` word | `save_status` | T | direct read | the word |
| V20 | Main menu Continue line (playtime, crew count, kind, `RISKY`, `RING <n>`) | `SaveInfo` | T | SIM facts only (SL UI-4) | all as words |
| V21 | UI-5 (New campaign over existing) | `campaign_info()` | T / M | plain sentence plus actions | text |
| V22 | UI-7 / UI-8 / UI-9 lines | Save/Load result ids | T / M | plain ship-log lines | text |
| V23 | First-launch photosensitivity notice | install flag | T / M | static reading-face text | text |
| V24 | Keycap glyphs (device-following) | last input device | T / M | meta | keycap letter |
| V25 | Compact node list (small canvases) | `web.view` | T | the same data in list form | list structure |

**Rows that are not this layer's** (listed so a tester does not mis-file a defect):

| # | Visual | Owner | Class | Note |
|---|---|---|---|---|
| X1 | **The frozen world under the panel** | Perception Filter and the world layers, held by `PAUSED_FROZEN` (PIPE:265) | **L as captured** | Keeps its Filter output, including any hallucination, undimmed. A frozen lie beside a true panel is the layer working (CT-2, EC-CT-59, QA note AC-CT-79), not a defect |
| X2 | Pod-console prompt and refusal plate | Interaction (WM:1029) | M | same register (opaque Ground plate, instrument and reading faces) but drawn on the HUD layer at the console's screen position, only when lit and in reach; not drawn by the chart table |
| X3 | Failed-write notice (UI-10) | HUD & Suit Layer | M | not here (SL:1006) |
| X4 | Black `TRANSIT_HOLD` | Pipeline layer 1 | n/a | the layer may open over it; it supplies no content |
| X5 | Any suit reading, loop glyph, gauge | HUD and the world | L | the chart emits none; no dial arc exists anywhere on this layer (ART 7.4) |

### Palette rules

**CT-VA-3. Allowed palette (ART 4.1, 4.6, by role).** Hexes are the art bible's, cited so the palette-scan AC (AC-CT-69) is writable without opening another document; none is a new palette entry.

| Role | Colour and hex | Used for on this layer |
|---|---|---|
| Ground | Graphite `#1C2128` | full-bleed plate, list interiors, panel, bar interior, ring 1 |
| Frame / bezel | Graphite lit `#5C6670` | chamfered border, list item borders, keycap border, disabled labels, UNSEEN / SEEN / BOARDED silhouettes |
| Highlight | Hull Grey `#8C949C` | filaments (1 art px), berth seam (1 px), section labels |
| Text | Paper White `#DDE4EA` | all text, hold-bar fill, keycap letters |
| Accent / selection | Tungsten fixture `#FFCC66` | focus bracket, hover label, HOME point, planned-sortie route, unknown tick, hold needle-mark |
| Deepest | Void Steel `#0B1018` | centre disc of the depth steps only |
| Dim | Graphite mid `#2A303A` | ORPHAN silhouette only (WM:1046) |

**CT-VA-4. Excluded in every state.** (1) **Violet** (`#8A3FD1` core, `#B07BE6` fringe; hue 270-330 degrees, saturation above 25 %) and **Flesh** (`#D9A58C`, `#B5705C`; hue 5-30 degrees, saturation 20-60 %, value 45-85 %) are absent at sanity stage 0 and forced stage 3 alike (ART 4.4, HR-6, SN:1226, AC-SN-83). The single violet that reaches screen space is the stage-1 gleam on a lying reading's glass on the HUD; this layer has no glass. (2) **Alarm Red** `#E0102A` appears on no element and on no text: no failing instrument exists here, so `FAILED`, `RISKY`, `IN TRANSIT`, the destructive hold bar and UI-9 are Paper White words or shapes (HR-4, ART 4.6). (3) **Loop hues** (O2 Cyan, Power Amber, Coolant Ice, Fuel Ember) and **Chemlight** `#86B896` are absent; at MVP both `hue_class` values are Tungsten (WM:1043); a loop-hue unknown tick is a post-MVP possibility and enters the ART 4.7 table with a named shape backup first. (4) **Iridescence** is absent; filaments are plain Hull Grey (ART 4.6, 7.4). (5) No global hue remap or daltonisation (HR-10): colour-blind support is shape. (6) No tint reacts to Attention, sanity, depth, time of day, fuel or integrity; depth steps change **value only** (ART 4.0 principle 3).

**CT-VA-5. Contrast ladder (ART 4.5; ratios from the art bible's own table and recomputed 2026-10-27 from the sRGB hexes; approximate, WCAG relative luminance).**

| Pair | Ratio | Verdict |
|---|---|---|
| Paper White on Ground | about 12.6 to 13 : 1 | passes the 4.5:1 text floor |
| Tungsten fixture on Ground | about 10.8 to 11 : 1 | passes |
| Paper White on Tungsten | about 1.2 : 1 | **never put Paper White on a Tungsten plane**; focus is a shape |
| Hull Grey on Ground | about 5.3 : 1 | passes |
| Graphite lit on Ground | about 2.8 : 1 on ring 1 (Ground itself) | **below the 3:1 glyph floor (HR-12)** for silhouettes and list borders: risk R-AD-4 |
| Graphite mid on Ground | about 1.2 : 1 | effectively invisible: risk R-AD-5 |

Achromatopsia check (ART 4.5): Tungsten (L* about 85-92) and Paper White (about 90) are one grey; the pair is never the only distinction (hover is accent-only; focus is bracket plus thickening; the planned route is told from any other route by dash pattern, not by hue alone).

### Per-event visual feedback

Timings are cited, never restated as new constants: the 80-120 ms fade band and the 100 ms bracket ease-out are ART 7.3; `CHART_FADE_MS` is the one owned `[PLACEHOLDER]` inside the band. "Cut" is a one-frame change. Art px are at HUD scale `H` (layer 3 is a root-viewport layer, PIPE PR-12), so a 1-art-px stroke is `H` screen px (H of 3 or more meets the 2-screen-px stroke floor, HR-12).

**CT-VA-6. Layer and panel events.**

| Event | Visual response | Motion | Constraints |
|---|---|---|---|
| `chart_table` pressed, layer opens (SOLO) | right-third Ground panel over the world, which stays lit exactly as it was; default focus bracket on Resume | `CHART_FADE_MS` within the band, or a cut: the plates only | no slide, bounce, dim, blur, desaturation or vignette on the world (ART 7.4, 7.3); panel opaque (HR-2) |
| Layer closes | panel plates removed; the world continues | same fade or cut | no afterimage |
| Layer opens over `TRANSIT_HOLD` black | panel or chart on Ground over the black; the HUD (layer 2) is covered where layer 3 is opaque | same | works either way (PIPE:296) |
| Chart chosen from the panel | the panel plate is replaced by the full-bleed chart (Ground, border, rivets) | cut (a replacement is not an appearance); under reduce flashing the long end of the band (CT-3) | the HUD below is covered (layer 3 above layer 2) |
| `cancel` one level back | chart, panel, closed; the same transitions in reverse | cut or fade | CT-5 |
| Rapid re-toggle of Esc | each toggle is honoured | each luminance change is one transition of at most 120 ms | whether a player-initiated toggle of a third-of-screen plate counts under HR-13 is the art-director's call; no guard is a rule at MVP (ASK-CT-8, OQ-CT-15); the 334 ms guard `CHART_REOPEN_MIN_MS` is the default form if ruled yes |
| Panel open in OVERLAY (VS-B, reserved) | identical panel; the world keeps moving beneath | none beyond the above | the panel hides the hand rail (bottom-right); the critical strip and suit panel stay visible: risk R-AD-6 |

**CT-VA-7. Focus, hover, disabled, activation.**

| Event | Visual response | Motion | Notes |
|---|---|---|---|
| Focus moves (key or pad) | Tungsten bracket (left bar plus chamfer corners) **and** a 1-art-px thickening of the focused bezel relocate to the new item or node | 100 ms ease-out (ART 7.3); a cut under reduced motion | the primary channel is the geometry change; never a Tungsten fill (ART 7.4) |
| Mouse motion | the bracket hides (PC-13); hover only | cut | the bracket is drawn from key and pad focus only (CT-6) |
| Mouse hover | the item's label turns Tungsten (ART 7.4) | cut | accent-only and luminance-neutral (R-AD-8): the cursor position is the real channel; hover never moves the bracket or thickens a bezel; a chart node carries no label, so hover changes nothing on a spatial node (OQ-CT-9) |
| Item disabled | label and border in Graphite lit `#5C6670` | none | never hidden (CT-3); still reads as a chamfered list item; the reason, if any, is a `ct.*` reading-face line, never an icon |
| Press / activate | the label holds Paper White for the press frame | cut | no flash, no scale, no squash (ART 7.3) |
| Device switch (`DEVICE_SWITCH_HYSTERESIS`, PCI, registry) | all keycap glyphs swap to the last device's set in one frame | cut | **keycap slots are fixed-width** (the widest glyph across devices) so a swap never shifts a label or a frame; gamepad face buttons are chamfered keycaps, not circles (ART 7.2); PlayStation symbols sit inside the keycap at 16 px |

**CT-VA-8. Chart events (one row per WM section 3 drawing row, plus the pick).**

| `web.view` fact or event | Visual (WM:1038-1050) | Forbidden here |
|---|---|---|
| HOME node | Tungsten fixture point `#FFCC66`, 1-px rectilinear Foundry silhouette, `ui_glyph_node_home_24`; the only warm mark besides a planned sortie and the unknown tick | glow, bloom, pulse, ring halo |
| UNSEEN derelict | plain Graphite-lit rectangle; **no label, tick or dialect** | any hint of kind, loot, danger |
| SEEN / BOARDED | Graphite-lit rectilinear silhouette in the donor dialect (MVP: Foundry, worn), `ui_glyph_node_wreck_foundry_24` | colour-coding by visit count |
| `unknown_seen` | one 1-art-px Tungsten square inside the silhouette plus a 1-px Ground clearing (a drawn pixel, not an asset) | circle, glow, loop hue, animation, count |
| `berthed` | silhouette abutting HOME at the socket edge; the join is a solid 1-px Hull Grey seam | a Tungsten seam line |
| ABSORBED | drawn as part of the home silhouette; no tick | a separate node, a "complete" mark |
| ORPHAN | Graphite-mid silhouette, detached, not selectable, no tick (R-AD-5) | a selectability cue, the bracket |
| `pod_at` HOME or a derelict | `ui_glyph_pod_24` at that node | trail, previous-position ghost |
| `pod_at == IN_FLIGHT` | **no pod marker**; the `IN TRANSIT` word (CT-VA-17); the planned route from `web.view.leg_target_id` and `leg_dir` on an OUT leg only (WM-36; none on a BACK leg), with the word alone otherwise | interpolated marker, progress, ETA, distance covered, speed lines |
| Chart opened during LEG, ARRIVING, BERTHING | the same drawing; the pick is frozen and nothing is drawn from it (CT-15); BERTHING has **no** word and no element | any "arriving" cue |
| Pick set (accept on a legal node) | the planned-sortie route is drawn HOME to that node, Tungsten, 1 art px, dashed; the bracket stays as focus | label, distance, cost, "recommended" |
| Pick invalidated (CT-8) | the route is removed in a cut; the bracket is unaffected | animation, red, toast, shake; no explanatory line (the route vanishing is the feedback and the player is looking at the chart) |
| Focus on a node (not picked) | the bracket only; no route | a previewed route (a preview is a recommendation by default) |
| Accept on a non-selectable node | the bracket cannot land there (snap skips it) | a "denied" shake or red |
| Restore with the chart open | the layer closes (TX-27); on reopen the chart is a pure function of `web.view` | a resume-in-place animation |

**Pick versus focus (the visual separation).** Focus is the bracket (transient, follows input); the pick is the dashed Tungsten route (persistent until reset). The two coincide when focus rests on the picked node. This is the only way to tell them apart without a new mark, and it is how **OQ-AD-1 (does activation or focus alone pick?) and the "second bracket" option are closed**: accept picks, focus never does, and the route is the pick mark (CT-7; checkpoint ruling Q13; OQ-CT-10 closed). The dashed line always means "the target the next console press carries".

**CT-VA-9. Save, quit, menus and notices.**

| Event | Visual | Motion | Constraints |
|---|---|---|---|
| Panel with the newest-record line (UI-2) | one instrument-face line (D.7 text, Paper White, tabular numerals, cap height at least the ART 7.1 floor) | none; **static while open**, never ticks | no countdown, no "time since", no per-write mark (SL H5), no spinner |
| Hold-to-confirm at 0 % | empty linear bar (Ground interior, 1-px Graphite-lit chamfered border); the needle-mark at the left end | none | never radial (ART 3.4, 7.3) |
| Hold at 50 % | Paper White fill to half; the needle-mark at the leading edge | linear fill, whole-art-px steps | CT-VA-14 |
| Hold at 100 % | bar full, needle at the right end; the action fires on the same frame | none | |
| Hold aborted (release, `cancel`, `chart_table`, pointer exit, focus loss) | the bar snaps to empty in one frame | cut | no bounce-back, no flash |
| `SAVE_QUIT_WAIT` (the `QUIT` write in progress) | **nothing is added.** No spinner, "saving" text, ring, percentage or sound (SL-24). The only change is the four items entering their disabled styling; nothing is focusable so the bracket is hidden (orchestrator ruling: Q-C, bounding the AC-CT-50 pixel diff) | none | if the write is slow the panel stays; absence is correct |
| `FAILED` (held or failed write) | the plain word `FAILED` replaces the playtime numerals in the newest-record line position (orchestrator ruling: Q-F; CT-21 names the position), Paper White, instrument face | cut | **not Alarm Red** (HR-4: a save is not an instrument) and no red glyph; the word is the channel |
| DOWNED Save & Quit, `UI3_PLATE` | message plate: the string, Back, Quit without saving (hold) | as the hold bar | no death language (SL:993) |
| Main menu (UI-4) | Ground full-bleed with border and rivets; a left navigation rail (Continue, New campaign, Quit); Continue's `SaveInfo` shown as words in the content column | none besides the drift | no thumbnail, no completion figure, no wall-clock "last played" |
| `RISKY` flag | the Paper White word `RISKY` in the Continue line | none | a word, never a red mark or a colour state (SL:1010, HR-4) |
| Depth label | `RING <n>` as a plain word | none | no bar, no fraction, no "n of 3" (WM guard rail 5) |
| UI-5 (New campaign over existing) | a plain reading-face sentence; list items Retire (default focus) and Delete, plus Back; Delete carries the hold bar | as the hold bar | the destructive item looks like every other item (no red, no warning mark) and is told apart by the bar only |
| UI-7 line | one plain reading-face line under Continue, Paper White | cut | no icon, no red |
| UI-8 / UI-9 | boot-flow screen: a reading-face message plus New campaign and Quit | cut | no dialog chrome beyond the standard Ground plate |
| First-launch notice (PIPE PR-13(e)) | reading face on Ground; two chamfered-switch toggles (reduce flashing, reduced motion) and one dismiss item | cut in | static, no flashing sample, no animation of the warning itself (the notice must not be a photosensitive stimulus); default focus on the first toggle |
| FAULTED return in release (CT-26) | the boot-flow main menu appears as at boot | cut | no new visual, no error colour |
| Reduced motion or reduce flashing toggled while open | takes effect on the next frame; the drift stops at its current whole-pixel offset and holds | | CT-VA-13 |
| Compact layout engaged | the node list replaces the spatial map (CT-VA-18) | cut | chosen at open and on canvas change, never mid-animation |

### Motion constraints

**CT-VA-10. Authority.** ART 7.3, "Menus and chart table", quoted: *"Cuts and short fades (80-120 ms), no slides, no bounces, no parallax. The focus bracket snaps with a 100 ms ease-out. The web map's filaments drift at <= 1 art px s^-1 in the current ring only (2.11); everything human on the map is static. Hold-to-confirm (HR-14) fills a linear bar with a needle-mark, never a radial wipe (3.4). Sober means almost nothing moves, and what moves is the web."*

**CT-VA-11. What moves (the complete list).** Everything else is static: nodes, silhouettes, glyphs, routes, seams, ring steps, rivets and text.

| Moving thing | Rule | Clock |
|---|---|---|
| Filaments, **ring 1 only** | whole-art-px offsets (`FILAMENT_DRIFT_STEP_PX`), direction fixed per sheet, no oscillation, at `FILAMENT_DRIFT_FRACTION` of the cited ceiling (D.5) | the layer's own UI clock (CT-43); `present_time` is frozen under pause (PIPE:265) |
| Focus bracket | 100 ms ease-out between items (ART 7.3) | UI clock |
| Layer and panel fade | `CHART_FADE_MS` within 80-120 ms | UI clock |
| Hold-to-confirm fill | linear in time over `CONFIRM_HOLD_S` (cited, PCI) | UI clock |

**CT-VA-12. What never appears.** (1) **Settle, tremor, glide** (ART 7.3's three signatures) are readings only; a needle-like wander on any chart element would be read as the class-(b) glide (HR-5) and teach the player that the one honest layer lies. (2) No parallax between the ring plate and the filaments (ART 7.3). (3) No pulse on HOME, the unknown tick or the pod marker: a pulsing Tungsten point would be an Attention-style signal (P3) and an ETA proxy. (4) No scrolling drift of the whole map: the drift is a texture offset inside ring 1, not a camera. (5) No camera jolt (PR-3), no screen shake, no blur, no scale change under any state (HR-13). (6) The main menu applies the same drift rule to any backdrop filaments; it is the only motion on that screen.

**CT-VA-13. Reduced motion and reduce flashing (separate toggles, PIPE PR-13(c), HR-8).**

| Toggle | Effect on this layer |
|---|---|
| Reduced motion ON | filament drift stops and holds its current whole-pixel offset (no snap-back); bracket relocation becomes a cut; fades are unchanged (a fade is not spatial motion; CT-13); the hold bar keeps filling |
| Reduce flashing ON | panel and plate transitions use the long end of the ART 7.3 fade band and never a hard cut of a large plate; the hold bar is unchanged |
| Both | the union |
| Neither | CT-VA-11 as written |

*Why separate:* a hard cut of a third-of-screen plate is a bigger luminance step than a 120 ms fade, so "less motion" and "less flashing" pull opposite ways for the panel; one toggle cannot serve both.

**CT-VA-14. Hold bar motion.** Linear fill, no easing, no overshoot, whole-art-px steps (a `L_BAR_HUD` 64 bar over `CONFIRM_HOLD_S` (0.8 s, the Player Controller's registry value at authoring, illustrative) advances 80 art px a second, about 1.3 art px a frame at 60 Hz: no stepping problem). The fill is a monotonic luminance rise confined to the bar, never full-frame, and runs once per hold, so it is well inside the 3 Hz ceiling (HR-13). The abort snap-to-empty is a single transition. The bar is **function, not decoration**, so reduced motion does not remove it (HR-8 spirit: keep information, drop ornament).

**CT-VA-15. Flicker ceiling (HR-13, 3 Hz).** Every luminance transition on this layer is single-shot and bounded by 120 ms. The only way to exceed 3 Hz is repeated toggling of the whole panel (an Esc mash) or rapid focus movement (the bracket changes a small area, below any photosensitive extent threshold, and is bounded by its 100 ms ease). The first-launch notice and the main menu contain no animated element other than the drift.

### Art rulings carried into the GDD

**CT-VA-16. Depth steps are an ordering contract, not a percentage (clarification C16).** ART 7.4 and WM:1050 say each step is "about 4 % darker toward centre; the inner ring's centre reaches Void Steel". The arithmetic does not close: Ground `#1C2128` is (28, 33, 40) and Void Steel `#0B1018` is (11, 16, 24), a span of 17 / 17 / 16 units; 4 % relative is about 1.5 units a step (near-invisible after 8-bit quantisation, ART 8.11 flag 1, and it never reaches Void Steel), and 4 points of white overshoots below Void Steel. The governing reading is the **ordering contract of D.6**: ring 1 is Ground itself (the lightest, where every MVP node sits and every contrast check is measured), each ring band is darker than the one outside it by at least `RING_STEP_MIN_CHANNEL` per channel, and the centre disc is exactly Void Steel. **The D.6 interpolation governs: (28, 33, 40), (22, 27, 35), (17, 22, 29), (11, 16, 24).** The art-director's first illustrative tones (24, 29, 36), (20, 25, 32), (15, 20, 28) assumed ring 1 below Ground; checkpoint ruling Q5 fixed ring 1 = Ground exactly, so they are superseded and not live values. Exact on-screen values are verified by pixel sampling in the spike (AC-CT-47, banding check at 1080p and the Deck canvas). The "about 4 %" wording is not restated as a value in this GDD; the clarification is routed to the art-bible owner (C16). The tones are derived values, never new palette entries and never registered.

**CT-VA-17. The `IN TRANSIT` word.** Inline on the full-bleed Ground, **no plate** (ART 7.4's register is already Ground). Instrument face (bitmap, 7 art px cap height), Paper White, tabular letterspacing of one art px (ART 7.1). **Fixed frame position**: inside the chamfered border at the top-left, anchored to the picture rect with the 5 % safe margin (PIPE:88); it is a status word like a label on a bezel and is never placed near the origin node, the target node or along the route, because position would read as direction or progress that `web.view` structurally lacks (WM-36). Shown iff `pod_at == IN_FLIGHT` (CT-15); it appears and disappears in a cut with the data; no ellipsis animation ("IN TRANSIT..."), no blinking, no dots. BERTHING shows **no word and no element** (WM:1307). The word is the only text on the chart during flight: there is no "arriving in" and no counter.

**CT-VA-18. Compact layout (look).** Triggered iff `m < 1` (CT-44, D.1b); the worst reachable canvas is 213.33 x 120 HUD px. Menu-component grammar (ART 7.4): one column of chamfered-rectangle list items with Ground interiors and 1-px Graphite-lit borders; the bracket and bezel thickening as in CT-VA-7; each item carries its 24x24 node glyph at left (`ui_glyph_node_home_24`, `ui_glyph_node_wreck_plain_24`, `ui_glyph_node_wreck_foundry_24`) with the unknown tick drawn inside the silhouette exactly as on the map, and its plain label at right. **No rings and no filaments** in compact mode (ring steps would be unreadable at 120 art px of height and add draw cost); the Ground plate, border and rivets remain so the layer still reads as the same register. The planned-sortie relation is a Tungsten dashed rule beneath the picked row (1 art px, no new asset, no fill, no new colour). The `IN TRANSIT` word keeps its fixed top-left position. **List order is a ranking proxy** (a vertical list invites "top is best"), so it is the stable authored order of CT-44: HOME first, derelicts in layout-file order, never sorted; this is acceptance criterion AC-CT-31. Rows are never truncated; if a list cannot fit, the canvas minimum was mis-set.

**CT-VA-19. Filaments and rings: authored (D12), three bands (D20).** (1) Authored, not procedural: ART 3.4 requires branching, non-repeating, no right angles, density rising inward, which is hand-drawn art direction; procedural generation is a VS-A candidate to be revisited only if ring 2-3 content arrives. For MVP: one hand-authored filament sheet for ring 1 (drifting, scrolling by whole-art-px offset with a seamless wrap that creates no visible seam or repeat at any reachable canvas; the sheet is longer than the largest canvas in the drift axis plus one wrap period) and static sheets for the horizon bands. (2) Three bands plus the Void Steel centre disc give the "inward-pulling" composition (ART 2.11) and the Depth Rings of the Vertical Slice then need no layout change. Rings 2 and 3 are horizon only, with static filaments denser toward the centre (ART 3.4), no nodes, no labels, no ticks. (3) Ring 1 is the sparsest band, so the drifting filaments are the sparsest: correct for "very low energy" (ART 2.11).

**CT-VA-20. Route and dash grammar.** The planned sortie is Tungsten, dashed, 1 art px, dash 2 on / 2 off art px (`ROUTE_DASH`, `[PLACEHOLDER]`), rectilinear segments only (human geometry over inhuman, ART 3.4). ART 7.4 also lists "routes dashed Hull Grey", but the MVP star topology has no standing routes, so the planned sortie is the only route drawn; if non-planned routes ever land (Hull Grey) they must use a different dash pattern, because the Tungsten versus Hull Grey distinction alone is weak in greyscale (shape backup, ART 4.0 principle 2). C12 is closed by Web Map & Transit (`leg_target_id`, `leg_dir`): the in-flight route is drawn on an OUT leg only, from those fields; on a BACK leg it is absent and the word `IN TRANSIT` remains alone (CT-15). Routes, seams and rivet lines are **textured quads of a 1-art-px tile**, not width-1 line draws (the 4.7 `CanvasItem` line draw has no antialiasing feather) -> ADR candidate (b), the layer-3 component library.

**CT-VA-21. Main-menu backdrop.** The main menu and the boot-flow screens reuse the depth-step plate and the ring-1 filament sheet as their Ground backdrop with **no nodes** (a node would imply a save state or leak a Continue fact), the menu column sitting left of the centre disc so the composition still pulls the eye inward behind the list. Fallback if the playtest finds it noisy: plain Ground. No title art, no logo animation, no tagline (anti-pillar: no authored story). A game title, if shown, is a reading-face title (title floor of ART 7.1), Paper White.

### Type and layout

**CT-VA-22. Faces and localisation.** **Instrument face** (bitmap, 7 px and 5 px cap, integer `H`): labels, `IN TRANSIT`, the newest-record numerals, keycap letters (no node labels at MVP). **Reading face** (vector humanist sans; regular plus one heavier weight, no light weights): settings copy, notices, messages, list item labels in menus, titles. Floors are cited from ART 7.1, not restated (Tuning Knobs, cited rows). Both faces carry tabular numerals, a slashed or dotted zero and a distinct 1 / l / I. The bitmap set is a **closed list** (`IN TRANSIT`, digits, `RISKY`, `FAILED`, short labels; CT-28): every sentence and any string likely to localise goes in the reading face, otherwise each locale needs hand-drawn bitmap glyphs (R-AD-10). The reading-face scale is continuous (the only non-integer text), so plates are sized to text with a minimum, not text fitted to a fixed plate; a reading-face line is 45-75 characters (ART 7.1); all text sits on an opaque Ground plate (HR-2). Chamfers only, no rounded corners (ART 3.4); one rivet per corner of the border, never as texture; dial arcs absent. Layout is anchored to the picture rect with the 5 % safe margin at every reachable canvas (CT-16). The right-third panel is `W_panel` HUD px wide (D.1); at row 6 it is 71 art px, so labels must fit by reading-face scale or compact behaviour applies: no label is truncated and none carries an ellipsis (CT-28).

### Audio

**CT-VA-23. Audio: requests only; the mood the visuals set.** The layer owns and specifies no sound (CT-29, rulings R26). It requests from Adaptive Audio a UI-bus focus-move cue and a confirm cue (AA:562); nothing for open, save or load; no request depends on sanity stage or Attention (AA:562 "no sanity or Attention effect on menu audio", HR-6). Silent menus are acceptable at MVP. The visual mood the audio should match: **hush**. The chart table is the quietest room in the game, a place where nothing is wrong and nothing is being measured; menu and map audio should sit outside the fiction's danger. The one living thing is the slow drift of the web: if audio acknowledges it at all it is as a faint, constant, non-reactive bed, never a response to a chart action. Layers, ducking, fades and triggers remain the audio-director's (provisional — Adaptive Audio owns the cues).

### Risks and findings carried (art-director)

| ID | Finding | Severity | Routed to |
|---|---|---|---|
| R-AD-1 / C16 | ART 7.4 and WM:1050 "about 4 %" arithmetic does not close; ruled as an ordering contract (CT-VA-16) | Medium | art-bible owner (wording) |
| R-AD-2 / C17 | ART 7.4 selection row "label and distance beside it" versus CT-14 / OQ-WM-34 (no label, none derived); the rulings stand | Low | art-bible owner |
| R-AD-3 | the planned-route length is a layout distance and reads as "far / near" even without a label. Mitigation: layout positions are authored cosmetic placements, MVP fuel is flat, and the layout lint (AC-CT-33) rejects any authored distance that correlates with a Web Map & Transit cost value; screenshot review at ASK-AD-7 | Low-Medium | Web Map & Transit, layout review |
| R-AD-4 | Graphite lit on Ground is about 2.8:1, under the HR-12 glyph floor of 3:1; every UNSEEN / SEEN / BOARDED silhouette and list border sits there. Options: accept (WM's drawing row is locked; the bracket and label carry selection), add a 1-px Hull Grey (about 5.3:1) outline inside the 24x24 glyph assets (a glyph change, not a WM row change), or lighten the band. Recommended: the glyph outline, only if the spike measures under 3:1 | Medium | spike, art bible, WM |
| R-AD-5 | ORPHAN in Graphite mid is about 1.2:1 and effectively invisible, so "a thing you cut loose" disappears from the chart (P5). Recommended: a 1-px dashed Graphite-lit outline (dashed means severed, a shape not a colour), which does not violate WM's "Graphite mid, detached, not selectable" | Medium | WM drawing-row clarification, spike |
| R-AD-6 | in OVERLAY (VS-B) the opaque panel covers the hand rail while the sim runs; reserved, not an MVP defect | Low (VS-B) | HUD, Session |
| R-AD-7 | a panel re-toggle can exceed 3 Hz on a third-of-screen plate (HR-13); the 334 ms guard is `CHART_REOPEN_MIN_MS`, **not a rule at MVP** (ASK-CT-8, OQ-CT-15) | Medium | art-director, Controller |
| R-AD-8 | hover turns the label Tungsten but Paper White versus Tungsten is about 1.2:1: hover is invisible in greyscale. Accepted: the cursor is visible and hover duplicates no information | Low | none |
| R-AD-9 | pick versus focus has no distinct mark except the dashed route; closed by CT-7 (accept picks, focus does not) | Low | closed |
| R-AD-10 | instrument-face localisation: the bitmap set stays a closed list (CT-VA-22) | Low | UX, localisation later |
| R-AD-11 | the reading-face hinting default changed in 4.7 (`ResourceImporterDynamicFont.hinting`): re-check legibility at the floor sizes | Low | technical-artist |
| R-AD-12 | no `on` / `off` token in ART 8.5's `state` list for toggle assets; ask the art bible to add them (`closed` / `open` would misname) | Low | art-bible owner |

No CONFLICT with a locked fact or ruling was found by the art-director; the two wording defects (C16, C17) are in a visual source, not in a locked gameplay value.

### Asset list for a later `/asset-spec`

Naming follows ART 8.5 (`<cat>[_<dialect>]_<subject>[_<variant tokens>]_<map>.png`; no `map` suffix on `ui_` and `font_`; size tokens `16`, `24`, `08`; indices `01`-`99`), which supersedes the generic template pattern; sizes follow ART 8.2. All are `ui_` or `font_` assets: flat, one-colour or Ground-plate, unlit, no normal or specular sheets.

| # | Asset | File / slice name (provisional) | Size / spec | Source |
|---|---|---|---|---|
| A1 | Home node glyph | `ui_glyph_node_home_24` | 24x24, 1-px stroke, flat | WM A10 (WM:1155), name kept verbatim |
| A2 | Unseen / plain wreck glyph | `ui_glyph_node_wreck_plain_24` | 24x24 | WM A10 |
| A3 | Foundry wreck glyph | `ui_glyph_node_wreck_foundry_24` | 24x24 | WM A10 |
| A4 | Pod marker | `ui_glyph_pod_24` | 24x24 | WM A10 |
| A5 | Depth step plates | `ui_chart_depth_step_01` to `_03`, `ui_chart_depth_centre` | stretchable bands; ordering contract (CT-VA-16) | ART 7.4, 2.11 |
| A6 | Ring-1 filament sheet (drifting) | `ui_chart_filament_01` | tileable on the drift axis, longer than the largest canvas plus one wrap period; Hull Grey 1 art px, branching, non-repeating | ART 3.4, 7.4 |
| A7 | Ring 2-3 filament sheets (static, denser) | `ui_chart_filament_02`, `_03` | static | ART 7.4 |
| A8 | Chamfered border and corner rivet | `ui_frame_chart` (9-slice), `ui_frame_panel` | Graphite-lit 1 art px, one rivet per corner | ART 7.4 |
| A9 | List item | `ui_item_menu` (9-slice) | Ground interior, 1-px Graphite-lit border; states normal, disabled | ART 7.4 |
| A10 | Focus bracket | `ui_focus_bracket` (9-slice) and `ui_item_menu_focus` | Tungsten, left bar plus chamfer corners | ART 7.2, HR-11 |
| A11 | Keycap base and pad face keycaps | `ui_keycap_base`; letters in the instrument face; `ui_glyph_pad_cross_16` etc. | chamfered square; symbols 16 px | ART 7.2, 8.5 |
| A12 | Hold-to-confirm bar | `ui_bar_hold` (9-slice) | linear, Ground interior, Paper White fill | ART 7.3, HR-14 |
| A13 | Needle-mark | `ui_mark_needle_08` | 8x8, Tungsten, protrudes above and below the bar | ART 8.2 |
| A14 | Chamfered toggle switch | `ui_toggle_switch` (on / off tokens pending R-AD-12) | position and a word carry state | ART 7.4 |
| A16 | Route dash tile | `ui_chart_dash_01` | 1 art px, 2 on / 2 off, Tungsten, textured quad | ART 3.4 |
| A17 | Seam tile | `ui_chart_seam_01` | 1 art px Hull Grey, solid | WM:1044 |
| A18 | Instrument face | `font_instrument_7px.png`, `font_instrument_5px.png` | bitmap, tabular numerals, slashed or dotted zero | ART 7.1, 8.5 |
| A19 | Reading face | `font_reading.ttf` (regular plus heavier) | vector humanist sans, tabular numerals, no light weights | ART 7.1, 8.5 |

A15 (slider) is not needed at MVP (no sliders in Display & Comfort); A20 (dismiss marks) is none (words only, CT-30). **Drawn, not assets:** the unknown tick, the berth seam join where the A17 tile is not used, the Void Steel centre disc (a flat fill). **Not assets:** loop glyphs (`ui_glyph_o2_*` etc., HUD's), the warning mark `ui_glyph_warn_16` (HUD's; never used here), HUD frames, any thumbnail, spinner, dial arc, title art or logo animation. Atlas organisation is decided at `/asset-spec` (ART 8.6, the `ui` page).

### Acceptance evidence the art side requires

Retained in `production/qa/evidence/chart-table/` (BLOCKING per the coding-standards UI and Visual rows); the criteria themselves are in Acceptance Criteria, groups D, F and G.

1. The chart at each `web.view` state, with no cost, progress, red, violet or distance label (AC-CT-25).
2. The depth-step banding check at 1080p and the Deck canvas, adjacent steps at least `RING_STEP_MIN_CHANNEL` per channel by pixel sampling (AC-CT-40, AC-CT-47).
3. A palette scan over the layer-3 screenshot set: zero Violet-band, zero Flesh-band, zero Alarm Red, zero loop-hue pixels (AC-CT-69).
4. The panel over a frozen world: world pixels outside the panel bit-identical to the pre-pause frame, layer 2 excluded (an active warning mark is held ON per HUD HS-23) (AC-CT-48).
5. Focus on both input paths (AC-CT-14); the hold bar at 0 / 50 / 100 % readable in a greyscale capture (AC-CT-37); the compact layout at 213.33 x 120 (AC-CT-30); the main menu, notice, UI-5 and UI-7 / 8 / 9 (AC-CT-78).
6. Motion: drift at or below the ceiling over 10 s, stopped under reduced motion, and no element changing luminance faster than 3 Hz in a 60 s capture including an Esc mash (AC-CT-39, AC-CT-82, AC-CT-103).
7. A greyscale pass over the chart and the panel: bracket, pick route, hold bar and `IN TRANSIT` distinguishable (AC-CT-75).

> **📌 Asset Spec** - Visual/Audio requirements are defined. After the art bible is approved, run `/asset-spec system:chart-table-web-map-and-menus` to produce per-asset visual descriptions, dimensions, and AI generation prompts from this section.

## Game Feel

*Requirement ids are `CT-GF-n`. No specialist wrote this section, so it is built from numbers already fixed elsewhere in this file or cited from their owners; every figure below names its source. Hold-timing figures use the registry value of `CONFIRM_HOLD_S` at authoring (0.8 s, owner: Player Controller) as an illustration in the manner of D.2, and are recomputed from the constant, never copied from this section. A UI layer has no "weight" in the combat sense; here weight is **stillness**: the layer feels like a heavy, static frame on which almost nothing moves, and the feel targets below are about response, restraint and the absence of ornament.*

### Feel statement

**CT-GF-1. The feel in one line.** Instant to answer, slow to move, never excited. Every input is answered in the frame it arrives or in the one after; nothing on the layer overshoots, bounces, pulses, shakes, glints or counts down (the three reading motions of ART 7.3 are absent, CT-2); the only autonomous motion is a web drifting at a fraction of one art pixel a second. The pause is a held breath, not a refuge (Player Fantasy, Moment 2): the panel appears without softening the world and disappears without ceremony.

**CT-GF-2. Feel pillars for this layer.** (1) *Answered*: no input is ever swallowed without a visible or documented reason (a disabled item is shown disabled; `SAVE_QUIT_WAIT` ignores input by rule, CT-21). (2) *Held*: the world beneath is the same frame (PIPE:265), so the panel's arrival changes nothing it does not cover. (3) *Deliberate*: the irreversible acts (UI-3 quit, UI-5 delete) cost a full hold on a linear bar, and the sortie commitment is off this layer entirely (the console press, CT-10). (4) *Plain*: no easing except the 100 ms bracket, no sound beyond a focus-move and a confirm cue.

### Responsiveness: the input-to-feedback chain

**CT-GF-3. Latency targets.** Frames are presentation frames of the root viewport; ticks are simulation ticks (`TICK_RATE` 60, so one tick is 16.67 ms). A target of "the same frame" means the first frame presented after the input event is processed.

| Input | First visible feedback | Target latency | Source and note |
|---|---|---|---|
| `chart_table` press, CLOSED to PANEL (SOLO) | panel plates begin their `CHART_FADE_MS` fade (or appear in a cut); bracket on Resume | **0 frames beyond the input frame** (presentation does not wait on the kernel) | CT-4; the fade is 80-120 ms (ART 7.3), default 100 ms |
| Kernel confirms `PAUSED` after that press | none visible; `can_save_quit` becomes true | **at most 1 tick (16.67 ms)** after the request | KRN CR-15 (honoured at a tick boundary, P6); the fade hides the gap; Save & Quit stays disabled until then (EC-CT-30) |
| `chart_table` or Resume, panel to CLOSED | plates fade out (or cut); the closing press and its release are consumed | 0 frames beyond the input frame; world already live under SOLO on the next kernel tick (`request_resume`) | CT-41 (2) |
| Chart item accepted (panel to chart) | the panel is replaced by the full-bleed chart in one frame | **0 intermediate frames** (a cut); under reduce flashing, the long end of the fade band (120 ms) | CT-3 |
| Direction press on the chart or panel | the bracket and bezel thickening start moving to the new target | 0 frames to start; **100 ms ease-out** to arrive; 0 ms (a cut) under reduced motion | ART 7.3; CT-6; the target position is final on the first frame, only the draw eases |
| Accept on a legal node | the dashed route appears, HOME to the node | **same frame** (no fade, a cut) | CT-7, CT-VA-8 |
| Accept on the picked node, on HOME at HOME, or on a non-selectable node | nothing changes (idempotent or inert) | same frame | EC-CT-5; there is deliberately no error feedback |
| Hold press edge on a destructive item | the fill starts from 0 | same frame | D.2, CT-22 |
| Hold reaches `CONFIRM_HOLD_S` | the action fires on the first frame at or past the threshold (not on release) | at 60 fps: **frame 48** (800,016 microseconds against 800,000); at 30 fps: frame 25; at 10 fps (clamped to `UI_DT_MAX_MS` per frame): frame 16, i.e. 1.6 s of wall time | D.2; illustrative at the registry value |
| Hold released, cancelled, pointer exit, focus loss | the bar snaps to empty | **same frame**, no bounce | CT-22, TX-10 |
| Toggle on Display & Comfort (reduced motion, reduce flashing, display mode) | the switch moves and its on / off word changes; the effect applies on the next frame | same frame for the control; **next frame** for the consumer; display mode needs the OS (the applied mode is read back and shown) | CT-18; the store write is debounced 500 ms (`SETTINGS_WRITE_DEBOUNCE_MS`) and invisible |
| Mouse hover | the item's label turns Tungsten; the bracket hides | same frame | CT-6 |
| Mouse click | activates on **pointer-up on the same item** that received the press; focus moves to it | same frame as the release | CT-6, EC-CT-16 |
| Device switch | all keycap glyphs swap to the new device's set | one frame, after `DEVICE_SWITCH_HYSTERESIS` (PCI, registry) | CT-6; keycap slots are fixed-width so nothing shifts |
| Save & Quit accepted | the four items enter their disabled styling; **nothing else** | same frame | CT-21; no spinner, bar or text |
| Continue / New campaign accepted | the items disable and the plate is removed in the same frame; layer-1 black; at least one full-black frame; then the restore call | 0 frames to black request; the black's own fade belongs to the Pipeline (`TRANSIT_HOLD`, PIPE:264) | CT-42; nothing is drawn between black and the first `RUNNING` frame |
| Panel or chart open to first presented frame, Deck class hardware | the first frame of the fade | one frame; no hitch above one frame on opening the chart (the layout is parsed at boot) | AC-CT-81, ADVISORY until the script budget is set |

**CT-GF-4. Mash tolerance.** Every press is honoured; there is **no cooldown** on `chart_table` and none on any panel item at MVP (EC-CT-68, AC-CT-103). The visual transition of the layer is a single 80-120 ms fade each way, so ten presses in a second produce ten honoured toggles, each a bounded single-shot luminance change; whether that needs a 334 ms guard on the Chart item is the art-director's open call (`CHART_REOPEN_MIN_MS`, ASK-CT-8, OQ-CT-15) and is not a rule. A same-boundary press pair converges (`PauseGate` re-issues the latest desired state, EC-CT-13); `chart_table` held with key echo toggles exactly once (EC-CT-10).

**CT-GF-5. Input hygiene as feel.** The irreversible press is protected by feel rules, not by dialogs: a console press that opens the chart cannot also activate something (fresh press edge, CT-41 (1)); the `interact` that closes the layer on Resume is swallowed so the player standing at the pod console does not hop the instant they resume (CT-41 (2)); a click on the frozen world does nothing (CT-41 (4)); a held Esc does not flicker (CT-41 (3)). The player should never feel that the layer "did something I did not ask".

### Timing and motion

**CT-GF-6. Timing table.** The only timed behaviours on the layer.

| Behaviour | Duration or rate | Easing | Clock | Owner of the number |
|---|---|---|---|---|
| Layer and panel fade, in and out | `CHART_FADE_MS`, default 100 ms, inside the 80-120 ms band; 120 ms under reduce flashing | linear (a fade; no easing specified by ART) | UI clock | band: ART 7.3; knob: this GDD, `[PLACEHOLDER]` |
| Panel to chart replacement | 0 ms (a cut); 120 ms fade under reduce flashing | none | UI clock | CT-3 |
| Bracket relocation | 100 ms | ease-out | UI clock | ART 7.3 (cited) |
| Bracket relocation under reduced motion | 0 ms | cut | n/a | HR-8; CT-13 |
| Hold-to-confirm fill | `CONFIRM_HOLD_S` (0.8 s at authoring) | **linear**, no overshoot | UI clock, per-frame delta clamped to `UI_DT_MAX_MS` | Player Controller (registry) |
| Hold abort | 0 ms | cut to empty | UI clock | CT-22 |
| Filament drift | `FILAMENT_DRIFT_FRACTION` (0.5) of the cited ceiling of 1 art px a second = **0.5 art px a second**, in whole-art-px steps of `FILAMENT_DRIFT_STEP_PX` (1): one visible step every 2 s | none (stepwise) | UI clock accumulator `phi` | ceiling: ART 7.3; fraction and step: this GDD |
| Directional-navigation repeat | first repeat after `UI_NAV_REPEAT_MS` (350 ms), then every `UI_NAV_REPEAT_RATE_MS` (120 ms); held 1,000 ms gives 7 moves; no wrap | none | UI clock | this GDD, `[PLACEHOLDER]` |
| Settings write | one write 500 ms after the last change; flushed at once on page close, Save & Quit, exit | n/a | UI clock | `SETTINGS_WRITE_DEBOUNCE_MS`, `[PLACEHOLDER]` |
| Load black | at least one presented full-black frame before the restore call | n/a | Pipeline | PIPE:264, CT-42 |

**CT-GF-7. Snap targets (positions are always whole pixels).** (1) Every layer-3 rectangle is positioned by anchors 0 or 1 plus integer HUD-px offsets (CT-16), so every bezel, bracket and rivet edge lies on a screen-pixel boundary at every `H`; a centred element is allowed only at an even on-screen width. (2) Panel edge: `W_panel` HUD px, `W_panel * H` screen px, an integer by construction, so the panel's left edge `P_w - W_panel * H` is an integer screen pixel (D.1: 1281, 1284, 854, 854, 854, 854 for the six canvas rows). (3) Node positions: authored integers in the 288 x 162 frame, scaled by the integer `m` (D.1b); a 24x24 glyph stays 24x24 at `H`. (4) The bracket relocates between integer positions; during the 100 ms ease its interpolated draw positions are rounded to whole HUD px so the bracket never sits on a half pixel. (5) Hold bar needle: `needle_px = min(floor(fill * L_BAR_HUD), L_BAR_HUD - 1)` whole HUD px (D.2); a `L_BAR_HUD` 64 bar over `CONFIRM_HOLD_S` (0.8 s, the Player Controller's registry value at authoring, illustrative) advances 80 art px a second, about 1.3 px a frame at 60 fps. (6) Filaments: whole-art-px offsets only, so the texture never rests on a half pixel (D.5). (7) Routes, seams and rule lines: 1-art-px quads aligned to whole art px, never width-1 line draws on a fractional transform (CT-11).

**CT-GF-8. Snap-to-node feel (gamepad and keyboard).** A direction press moves focus to the nearest focusable node whose bearing falls in the pressed direction's sector (`SNAP_SECTOR_DEG` 90 degrees), widening to the half-plane if the sector is empty, else staying (D.3). Feel targets: with three nodes, **no direction press is a dead control when another node exists on that side or in the half-plane** (the fallback exists for this); a press with no candidate leaves focus where it is with **no shake, no flash, no sound** (staying is silence, not an error); the bracket never wraps from the last node to the first; the same press always lands on the same node (deterministic, the diagonal assignment is exact, D.3); and every node is reachable from every other in at most N - 1 presses. Mouse and pad are equivalent in outcome: a click on a node is focus plus accept (CT-6). A stick resting on the deadzone boundary produces one move per edge, not a chatter (D.9, `MOVE_DEADZONE_EXIT`).

### Weight, juice and what is deliberately absent

**CT-GF-9. No juice, by rule.** No squash, scale pop, overshoot, bounce, wobble, shake, flash, glow, pulse, particle, screen-space effect, parallax, slide or wipe on any element of this layer (ART 7.3, rulings R16, CT-2). The press acknowledgement is a one-frame Paper White hold on the label (CT-VA-7). A destructive action looks like every other item until its bar fills. A completed hold fires on the same frame the bar is full; there is no celebratory beat. This is not austerity for its own sake: ornament on the truth layer reads as the layer reacting, and a reacting layer reads as an instrument that can be fooled.

**CT-GF-10. Stillness as weight.** The panel is a heavy opaque plate, chamfered and riveted (ART 7.4), that does not move once placed; the world beside it is a held frame, undimmed. The contrast (a still true plate beside a world frozen mid-lie at stage 3) is the feel the Player Fantasy aims at (Moment 2). The only life on the chart is the filament drift, kept deliberately below the art-bible ceiling so the screen reads "very low energy" (ART 2.11) and never "something is happening".

**CT-GF-11. Silence and absence of reward.** No sound on open, save or load; the only cues requested are a UI-bus focus-move cue and a confirm cue (CT-29), owned by Adaptive Audio. No "progress saved", no autosave mark, no completion flourish, no achievement toast, no tip on the black between menu and world (CT-36). Save & Quit's feel is *closing the log*: the items dim, the process ends.

### Feel risks and playtest observables

**CT-GF-12. What the playtest measures (all advisory, none a gameplay constant).**
- `CHART_TO_PRESS_S`: the median seconds from the first chart open of a sortie to the console press; target 20 s, flag above 40 s as the friction signal that Esc, Chart, pick, Esc, walk, press is too long. The reserved remedy is a separate chart action owned by the Player Controller, **never** data on the chart (Player Fantasy, validation 2; AC-CT-80).
- Whether the 100 ms bracket ease reads as responsive or laggy on a gamepad at the Deck's 60 Hz (the knob is cited from ART 7.3, not retuned here; a complaint is routed to the art bible).
- Whether `FILAMENT_DRIFT_FRACTION` 0.5 reads alive or dead (the first knob likely to move; Tuning Knobs, interaction notes).
- Whether the pause reads "held" and not "safe" (validation 3; AC-CT-106).
- Whether the hold at the registry value reads deliberate or sticky (a Controller-owned constant and the Alpha two-tap option (PCI:835)).

**CT-GF-13. Frame-time feel.** The layer must never be the cause of a visible hitch: the layout file is parsed at boot, not on open; the settings flush completes inside one frame (a few hundred bytes on the UI thread); per-frame work is the bracket tween, one accumulator or two and a handful of draw calls (D.10 estimates 19 against a budget of at most the Pipeline's `D_hud` upper bound). The numeric script-time budget is `[PLACEHOLDER]` and measured in a spike (AC-CT-45, AC-CT-81, ADVISORY until set). A large frame delta (a window drag, a display-mode change) is clamped to `UI_DT_MAX_MS` (50 ms) so no hold completes and no drift jumps from a hitch (EC-CT-58, D.2 example: a 400 ms frame advances a hold by 50 ms only).

**CT-GF-14. Cross-checks.** The feel numbers above are consistent with: ART 7.3 (fade band, bracket, drift ceiling, linear bar); HR-8 and HR-13 (reduced motion, 3 Hz); D.2, D.5, D.9 (hold, drift, repeat); CT-43 (the UI clock, independent of the kernel state so the bar and the drift run while the kernel is `PAUSED`); PCI:656 as clarified in C15.

## UI Requirements

*This system is a UI system: every surface listed here is its own. Requirement ids are `CT-UI-n`. The surfaces are layer-3 Controls in the root viewport (CT-1); the spec split follows rulings R30. `design/ux/` and `design/accessibility-requirements.md` do not exist yet, so every status below reads "not yet authored" and the shared-component grammar is defined by the HUD & Suit Layer's UX spec, which lands first (HS-48; `design/ux/interaction-patterns.md`); this layer cites it.*

### Surface inventory

**CT-UI-1. Surfaces owned by this layer.** One row per screen, with the state that shows it (States and Transitions), the data it reads, its inputs, the spec that will own its pixels and the evidence it owes. Input columns use the actions of CT-5: navigate (direction), accept (`interact`), back (`cancel`), close (`chart_table`).

| # | Surface | State(s) | Reads | Player-facing content | Inputs | Spec file (to be authored) | Evidence owed |
|---|---|---|---|---|---|---|---|
| S1 | **Pause panel** (right third over the frozen world) | `PANEL` | `SaveInfo.playtime` (newest), `PauseGate.can_save_quit`, `save_status` | four items in fixed order, the newest-record line (D.7), keycap hints | navigate, accept, back, close | `design/ux/pause-panel.md` | panel at canvas rows 1, 3, 6 with the world visible and undimmed, stage 0 and forced stage 3 (AC-CT-48, AC-CT-78) |
| S2 | **Pause panel, `FAILED` variant** | `SAVE_FAILED` | `save_status == FAILED` or the held-write timeout | the same four items; `FAILED` replaces the numerals in the newest-record position (Q-F ruling) | as S1 | `pause-panel.md` | capture at both stages (AC-CT-53) |
| S3 | **Chart, spatial** | `CHART` | `web.view` | ring bands, filaments, HOME, derelicts, the pod marker, the dashed route, bracket, control hints | navigate (snap, D.3), accept (pick), back, close | `design/ux/chart-table.md` | one capture per `web.view` state (AC-CT-25) |
| S4 | **Chart, compact list** | `CHART` with `m < 1` | `web.view`, `chart_layout.json` ordinals | one column of node rows with glyph and CT-44 neutral ordinal (one whole key), the dashed rule under the picked row | as S3 | `chart-table.md` | capture at 213.33 x 120 (AC-CT-30) |
| S5 | **Chart, view-only** | `CHART_VIEW_ONLY` | `web.view` | the same drawing without a pick or a route; `IN TRANSIT` iff `pod_at == IN_FLIGHT`; focus on Back | back, close | `chart-table.md` | captures for IN_FLIGHT and BERTHING (AC-CT-25, AC-CT-38) |
| S6 | **Display & Comfort page** (inside the panel's column) | `DISPLAY_COMFORT` | `SettingsStore` | display mode (windowed / fullscreen / borderless), reduced motion, reduce flashing | navigate, accept, back, close | `pause-panel.md` | capture plus a resize-with-focus check (AC-CT-42) |
| S7 | **`UI3_PLATE`** (full-bleed message plate: DOWNED quit refusal, or Retry plate) | `UI3_PLATE`, `HOLD_CONFIRM` | `SL_QUIT_NO_WRITE_DOWNED`, `save_status` | the sentence, Back or Retry (default focus), Quit without saving (hold bar) | navigate, accept, hold, back, close | `pause-panel.md` | captures for both entries (AC-CT-52, AC-CT-53) |
| S8 | **Hold-to-confirm bar** (component) | `HOLD_CONFIRM` | the UI clock (D.2) | linear bar plus needle-mark | press-and-hold | `pause-panel.md`; the component in `interaction-patterns.md` | 0 / 50 / 100 % captures (AC-CT-37, AC-CT-78) |
| S9 | **First-launch notice** | `BOOT_NOTICE` | `SettingsStore.notice_seen` | the photosensitivity text, two toggles inline, one dismiss item | navigate, accept | `design/ux/main-menu.md` | one capture (boot-flow, AC-CT-44) |
| S10 | **Main menu** | `MAIN_MENU` | `campaign_info()` | Continue (with its SIM-facts line), New campaign, Quit; backdrop of ring plate and filaments with no nodes | navigate, accept | `main-menu.md` | capture, with and without a campaign (AC-CT-56) |
| S11 | **UI-5, New campaign over existing** | `UI5_NEW_OVER_EXISTING`, `HOLD_CONFIRM` | `campaign_info()` | a sentence, Retire (default), Delete (hold), Back | navigate, accept, hold, back | `main-menu.md` | capture (AC-CT-57) |
| S12 | **UI-7 line** (main-menu instance) | `MAIN_MENU` | `campaign_info()` verification-failure field (ASK-CT-5, provisional) | one ship-log line under Continue | none | `main-menu.md` | capture (AC-CT-58) |
| S13 | **UI-8, no loadable save** | `UI8_NO_LOADABLE` | result `SL_LOAD_NO_LOADABLE_SAVE` | a plain statement, New campaign (default), Quit | navigate, accept | `main-menu.md` | capture (AC-CT-58) |
| S14 | **UI-9, incompatible build** | `UI9_INCOMPATIBLE` | the incompatible-build result | a message, Quit, New campaign | navigate, accept | `main-menu.md` | capture (AC-CT-58) |
| S15 | **FAULTED-return menu** | `MAIN_MENU` after a release `FAULTED` (CT-26) | `campaign_info()` | the main menu with Continue focused; no extra string (OQ-CT-5) | navigate, accept | `main-menu.md` | capture with `force_stage` 3 active before the fault (AC-CT-59, AC-CT-90) |
| S16 | **Load hold** | `LOAD_HOLD` | none | layer-1 black owned by the Pipeline; **this layer draws nothing** | none (second accept ignored) | none | the frame-log trace (AC-CT-60) |

**CT-UI-2. Surfaces that are not this layer's.** (1) The pod-console prompt and its refusal words: Interaction, on the register plate (WM:1029); the layer only shares the grammar. (2) The failed-write notice (UI-10): HUD & Suit Layer (SL:1006). (3) Any suit, loop or gauge reading: HUD and In-World Instruments. (4) There is no ship-schematic console (IW-28, C4). (5) The first-`RUNNING`-frame instance of UI-7: HUD & Suit Layer. (6) Sounds: Adaptive Audio. (7) The full settings UI, rebinding, HUD scale, audio and accessibility settings: Settings & Input Rebinding and Accessibility Settings (Alpha).

### Shared components

**CT-UI-3. The component grammar this layer needs** (ART 7.2, 7.4; names provisional until `interaction-patterns.md` exists). (1) **Ground plate** with a chamfered Graphite-lit border and one rivet per corner. (2) **List item** (chamfered rectangle, Ground interior, normal and disabled). (3) **Focus bracket** plus bezel thickening (HR-11). (4) **Keycap** (chamfered, fixed-width slot, device-following glyph). (5) **Switch** (chamfered, position plus an on / off word). (6) **Three-choice list** (display mode: the current choice marked by shape plus a word). (7) **Hold-to-confirm bar** (linear, needle-mark). (8) **Message plate** (full-bleed Ground sized to text, 45-75 character reading measure, up to three items). (9) **Newest-record line** (instrument face, tabular numerals). (10) **Node glyph set and pod marker** (24x24). (11) **Back hint** (the text-labelled control that holds focus in view-only chart mode). Components 1, 2, 3, 4, 7 are shared with the HUD & Suit Layer's prompts and the Interaction hold bar; they are defined by the HUD & Suit Layer's UX spec, which lands first (HS-48); this layer cites them. The hold bar shares the HUD component's look only; on this layer an abort snaps to empty (CT-22, rulings R19), not the HUD's drain (`CONFIRM_DRAIN_RATE`, HS-42); a clarification row (C19) is routed to HUD / UX.

### Input and focus requirements

**CT-UI-4. Input map for the `chart_table` context.** Accept is `interact` (E / LMB / A); back is `cancel` (X / B); close is `chart_table` (Esc / Start); directional navigation re-reads the move vector as four-way (CT-5), repeat per D.9. The Controller defines default bindings for the `gameplay` context only (PCI:64-80: `interact` :72, `cancel` :73, `chart_table` :78); no `chart_table`-context table exists, so directional navigation and accept are **(provisional — ASK-CT-2 to the Player Controller / Settings)**. Mouse: the cursor is released (PC-2), hover and click as CT-6, the wheel scrolls but nothing scrolls at MVP. Zoom is not an action here (CT-5). Every flow completes on a gamepad alone (HR-14), including the chart snap and each destructive hold; every interactive control has a text label (CT-30).

**CT-UI-5. Default focus per surface.**

| Surface | Default focus | Why |
|---|---|---|
| S1 panel | Resume | non-destructive (CT-3) |
| S3 / S4 chart (spatial or compact) | HOME's node | never a target, so focus is never a recommendation (CT-10) |
| S5 chart, view-only | Back | nothing on the map is actionable (CT-15) |
| S6 Display & Comfort | the first control | TX-20 |
| S7 `UI3_PLATE` | Back (DOWNED entry) or Retry (`SAVE_FAILED` entry) | the non-destructive item (CT-21) |
| S9 notice | the first toggle | CT-20 |
| S10 main menu | Continue if enabled, else New campaign | CT-23 |
| S11 UI-5 | Retire | reversible (CT-24) |
| S13 UI-8 | New campaign | the one productive action |
| S14 UI-9 | Quit or New campaign (UX decides; neither destroys data) | CT-25; never an overwrite |
| S15 FAULTED-return menu | Continue | the newest verified generation is intact (CT-26) |

**CT-UI-6. Focus loss, device and window behaviour.** Focus is retained by item id across a live relayout (display-mode change, window resize); the bracket does not nudge and no animation plays (CT-18). A window focus loss aborts any hold fill, clears navigation state and keeps the layer open (CT-41 (5)). A gamepad disconnect keeps the layer open and switches glyphs to the remaining device (EC-CT-22).

### Layout and scaling requirements

**CT-UI-7. Canvases.** The layer is laid out against the six reachable canvas rows (CT-16, D.1): 640 x 360, 320 x 180, 426.67 x 266.67, 213.33 x 133.33, 426.67 x 240 and 213.33 x 120 HUD px (the smallest). Anchor-relative only, 5 % safe margin rounded up to whole HUD px (registry `safe_margin`), panel `W_panel = P_w div (3 * H)`, chart frame 288 x 162 at multiplier `m`, compact iff `m < 1` (rows 4 and 6). Nothing scrolls at MVP. Panel labels are never truncated or ellipsised (CT-28); the fit lint runs against the real faces.

**CT-UI-8. Two scales on one layer.** The instrument face and all glyphs scale by the integer `H`; the reading face scales continuously (the only non-integer text, ART 7.1), so plates that carry reading-face text are sized to the text with a minimum and the full-bleed message plate carries every paragraph (CT-3, CT-21). -> ADR candidate (b): the two-scale mechanism and the shared theme.

**CT-UI-9. Strings.** Every user-visible string is a key in the closed `ct.*` table with a fit class (`LABEL_1`, `LABEL_2`, `PARAGRAPH`), owned by this layer, ship-log register, Save/Load's vocabulary only (CT-28); the lexicon of Player Fantasy "Tone and language" applies. Final copy is the UX spec's and the narrative director's; stale-token lint with a signed allow-list (AC-CT-68).

### Accessibility requirements (MVP, structural)

**CT-UI-10. What is required now (CT-30, rulings R27).** Opaque Ground plates (HR-2); focus as a shape (HR-11); ART 7.1 type and contrast floors (HR-12), with the two signed exceptions R-AD-4 and R-AD-5 until the spike reports; gamepad completeness (HR-14); no colour-only state (SL:1010; every state in CT-11 has a shape or word); a text label on every control; reduced motion and reduce flashing available from the first launch (the notice, the Display & Comfort page); no true-value overlay as an accessibility option (ART 7.4); no Attention, sanity, horror or threat element under any setting (CT-31, CT-32, CT-34). **Deferred to Alpha** (Accessibility Settings / Settings): AccessKit wiring (NOT SOURCEABLE in the engine reference), hold-duration and two-tap option, HUD opacity floor, reading-face scale, HUD scale. The project's accessibility tier is unrecorded (OQ-CT-8).

### Cross-surface rules

**CT-UI-11. What no surface may contain** (each a CT rule with an acceptance criterion): an Attention element (CT-31), a horror element (CT-32), a loot or salvage value (CT-33), a sanity element (CT-34), a fuel, time, ETA, danger or recommendation (CT-35), a save decoration (CT-36), an inventory (CT-37), title art, a tagline or a logo animation (CT-23; a plain reading-face game title is allowed, CT-VA-21), a thumbnail, a completion figure or a wall-clock "last played" (CT-23), a spinner, tip or percentage at load (CT-36, CT-42), and the word "Paused" (CT-17).

**CT-UI-12. Reserved slots (CT-38).** Crew glyphs and left-edge chat (VS-B), Codex annotations (VS), post-loss UI (VS-A), Onboarding, the beam picker, the Records page and un-retire (Alpha): no stub art and no layout space at MVP; each is one row in CT-38 and one review check (AC-CT-74).

> **📌 UX Flag — Chart Table (Web Map & Menus)**: This system has UI requirements. In Phase 4 (Pre-Production), run `/ux-design` to create a UX spec for each screen or HUD element this system contributes to **before** writing epics: `design/ux/chart-table.md` (the spatial and compact chart, the pick, view-only mode), `design/ux/pause-panel.md` (the panel, Display & Comfort, Save & Quit, `UI3_PLATE`, the hold-to-confirm bar) and `design/ux/main-menu.md` (the first-launch notice, Continue / New campaign including UI-5, UI-7 / UI-8 / UI-9 and the FAULTED return), with the shared components (Ground plate, list item, bracket, keycap, switch, hold bar, message plate) in `design/ux/interaction-patterns.md`, defined by the HUD & Suit Layer's UX spec, which lands first (HS-48); this layer cites them. Stories that reference UI should cite `design/ux/[screen].md`, not the GDD directly. Retained screenshots of each screen, at sanity stage 0 and forced stage 3 wherever the screen is reachable in a session, in `production/qa/evidence/chart-table/` are the evidence gate (AC-CT-25, AC-CT-30, AC-CT-78).

## Cross-References

Every explicit dependence on another document's specific element, as read by `/review-all-gdds` Phase 2c. *Nature* is one of Data dependency (we consume their output), State trigger (their state change triggers our behaviour), Rule dependency (our rule assumes theirs is also true) or Ownership handoff (we hand a value to them). A number this GDD cites and does not own is listed with its owner; this GDD states no second value for any of them. The Formulas, Edge Cases, Tuning Knobs, Acceptance Criteria, Visual/Audio, Game Feel and UI Requirements sections introduce no neighbour dependency beyond the tables below and cites the same targets by the same names; the additional sources it reads are: the UI palette hexes (art bible 4.1, 4.6, inlined in Visual/Audio and the AC fixtures), ART 4.0 principles 2 and 3 and 8.11 flag 1 (depth steps), HR-5 and HR-8 (reduced motion), the `modules/ui.md` and `modules/input.md` engine references (dual focus, key echo NOT SOURCEABLE), the registry constants listed in Tuning Knobs, and the qa-lead's flags F-A to F-G (Acceptance Criteria, "Gaps").

#### GDD and registry references

| This document references | Target | Specific element | Nature |
|---|---|---|---|
| The chart is drawn from the web's true values | `design/gdd/web-map-and-transit.md` | `web.view` (WM-36, WM:291-299; provisional name; registry `web.view`): `nodes[]`, `pod_at`, `transit_state`, `destination_ready`; no progress, fuel, Attention or sanity field (WM:300) | Data dependency |
| The legal-target mirror | `web-map-and-transit.md` | WM-4 (star topology: from HOME the derelicts not ABSORBED / ORPHAN and not berthed; from a derelict only HOME); WM-9 reasons `TARGET_UNKNOWN`, `TARGET_ABSORBED`, `TARGET_BERTHED`, `BERTH_PENDING` (WM:205-217) | Rule dependency |
| The pick and the console press | `web-map-and-transit.md` | WM-8 (WM:203): host-local pick, `pod_hop{target_id}`, HOME implied from a derelict; WM:1225 (the press is the commitment, no confirm); WM:1301 (the console is the only trigger); WM-21 (`berth{derelict_id}`) | Rule dependency, Ownership handoff |
| Drawing rules for the twelve chart rows | `web-map-and-transit.md` | Section 3 "The chart" (WM:1036-1050), glyph set A10 (WM:1155), ASK-AD-7 (WM:1180), evidence row WM:1209 | Rule dependency |
| UNSEEN derelicts selectable, bare | `web-map-and-transit.md` | OQ-WM-4 (WM:1299); WM-28 (chart records what the crew lit, never reveals); record statuses UNSEEN, SEEN, BOARDED, ABSORBED, ORPHAN (WM-3, WM:176) | Rule dependency |
| `IN TRANSIT` shown iff `pod_at == IN_FLIGHT`; BERTHING shows nothing | `web-map-and-transit.md` | WM:1048 (no interpolated marker; state word); WM:1307 (a berth is witnessed by the world); WM:1268 (the pause button is available but changes nothing); `transit_state` IDLE, LEG, ARRIVING, BERTHING (WM:310; registry `transit_state`) | Rule dependency |
| Arrival resets the pick | `web-map-and-transit.md` | WM-31 `POD_ARRIVED` | State trigger |
| `web.leg_timing()` is never called | `web-map-and-transit.md` | WM:300 (Audio-only `{t0, n}`, not part of `web.view`) | Rule dependency |
| No distance label, no cost beside the selection | `web-map-and-transit.md` | WM:1049; guard rails 5 and 7 (WM:138-146); OQ-WM-34 (WM:1653) | Rule dependency |
| Fuel and timing values are never displayed | `web-map-and-transit.md`, `design/registry/entities.yaml` | `HOP_FUEL_ML`, `BERTH_FUEL_ML`, `HOP_FUEL_FLOOR_ML`, `RETURN_HOP_FUEL_ML` (registry entries by name; WM-20), `HOP_LEG_TICKS`, `ARRIVE_SETTLE_TICKS` (registry entries by name; WM-14): named in negative sentences only | Rule dependency |
| `depth_label` on the main-menu Continue line only | `web-map-and-transit.md`, `save-load-and-versioned-format.md` | WM-27 / WM:1309 / `ring_of` (registry); SL UI-2 vs UI-4 (`depth_label` provisional in the registry) | Rule dependency (conflict C13) |
| In-flight route needs leg endpoints | `web-map-and-transit.md` | WM:1048 vs WM-36 (`leg_target_id` and `leg_dir` now in `web.view`; BACK-leg origin not published) | Data dependency (C12 closed, ASK-CT-3 answered) |
| Compact-list neutral ordinals (CT-44) | `web-map-and-transit.md` | WM:1044 UNSEEN "no label beyond what selection shows" | Rule dependency (ASK-CT-10) |
| Gamepad snap is node to node | `web-map-and-transit.md` | WM:1319 | Ownership handoff |
| Layer 3 in the root viewport; not wrapped by the Filter | `design/gdd/pixel-perfect-render-pipeline-camera.md` | PR-3 (PIPE:94-103), PR-10, PR-8 (PIPE:188, `INPUT_SURFACE`), row 1 (PIPE:99, black hold shown in `TRANSIT_HOLD` and `BOOTING/LOADING`) | Rule dependency |
| HUD canvas, `H`, `H_default`, safe margin | `pixel-perfect-render-pipeline-camera.md` | PR-12 (PIPE:224), D.4 (PIPE:369-386), `H_default = max(k, 3)`, `HUD_MIN_SCALE`, `hud_multiplier` (registry, by name); `cap_px = 7 * H` (PIPE D.4 variable, PIPE:373, :381, carried in `hud_multiplier`'s output, not a registry entry); `k` is the integer world scale; the 5 % safe margin (registry `safe_margin`, source `hud-and-suit-layer.md`; PIPE:88, PR-2 Deck safe area) | Data dependency |
| The world is not dimmed under a menu | `pixel-perfect-render-pipeline-camera.md` | `PAUSED_FROZEN` (PIPE:265); `present_time` frozen under pause | State trigger |
| Load and transit hold | `pixel-perfect-render-pipeline-camera.md` | `TRANSIT_HOLD` (PIPE:264); camera `CUT` (PIPE:278); SL UI-6 | State trigger |
| Zoom press discarded while the layer is open | `pixel-perfect-render-pipeline-camera.md` | PIPE:688; PR-14 / PIPE:232 (zoom is never a setting) | Rule dependency |
| Photosensitivity notice; reduce flashing; reduced motion | `pixel-perfect-render-pipeline-camera.md` | PR-13(c) and (e) (PIPE:228); PR-14 (PIPE:232); PIPE:301 (hooks with defaults) | Ownership handoff |
| Settings UI row narrowed | `pixel-perfect-render-pipeline-camera.md` | Interactions row 7 (PIPE:296); Dependencies row (PIPE:776); FAULTED wording (PIPE:269) | Rule dependency (C2, C14) |
| Layer 3 draw-call budget | `pixel-perfect-render-pipeline-camera.md` | D.8 (PIPE:610-624), `D_hud`, `D_root` | Data dependency (Formulas) |
| Pause and resume | `design/gdd/command-simulation-layer.md` | CR-15 (control plane: `request_pause`, `request_resume`, `request_snapshot`, `request_restore`, `request_shutdown`), CR-16 (pause semantics, KRN:129), CR-11 (presentation clock) | State trigger |
| `INPUT_SURFACE` class; submits nothing at MVP | `command-simulation-layer.md` | subscriber classes (KRN:111); `allowed_when_paused` (not introduced) | Rule dependency |
| KRN Open Question 6 | `command-simulation-layer.md` | KRN:961 | Ownership handoff (answered by CT-4) |
| FAULTED in release | `command-simulation-layer.md` | KRN:151, KRN:940 | State trigger |
| Stale payload example | `command-simulation-layer.md` | `MAX_PAYLOAD` (KRN:715) | Rule dependency (C3) |
| Determinism: no wall clock in any rule | `command-simulation-layer.md` | CR-11; the kernel's forbidden time and RNG calls | Rule dependency |
| Host-local API and reads | `design/gdd/save-load-and-versioned-format.md` | SL-21 (SL:261): `new_game(world_seed)`, `continue_campaign()`, `save_and_quit()`, `retire_campaign`, `delete_campaign`, `campaign_info()`, `list_saves()`; `save_status`, `SaveInfo` (registry, by name), `restore_reason` | Data dependency, State trigger |
| Save & Quit and its refusals | `save-load-and-versioned-format.md` | SL-6 (SL:195); UI-2, UI-3 (SL:998-999); `SL_QUIT_NO_WRITE_DOWNED`; `SAVE_QUIT_TIMEOUT_MS` (SL, not registered); SL:781, SL:785 | State trigger |
| Main menu, campaigns, failure screens | `save-load-and-versioned-format.md` | UI-4, UI-5, UI-7, UI-8, UI-9 (SL:1000-1005); `RETIRED` (SL T-9, SL:430); SL-11; `SL_LOAD_NO_LOADABLE_SAVE` | Rule dependency |
| No save decoration | `save-load-and-versioned-format.md` | SL H5, SL H6, SL:965, SL-24 (SL:267), UI-1 retired, UI-6 (SL:1002) | Rule dependency |
| Fixed vocabulary and tone | `save-load-and-versioned-format.md` | SL:993, SL:127, SL:1008, AC-SL-86 (stale-token lint) | Rule dependency |
| Zero-registration precedent | `save-load-and-versioned-format.md` | SL-17 (SL:253); the VS-B `saveload.request_save` reserved (SL:37) | Rule dependency |
| Hold duration | `save-load-and-versioned-format.md`, `player-controller-input.md`, registry | SL OQ-32 (SL:1329) closed by citation of the registry constant `CONFIRM_HOLD_S` (PCI:835); value in Tuning Knobs, cited rows | Ownership handoff |
| Settings are per install, not PROFILE | `save-load-and-versioned-format.md` | SL-9 (SL:201) | Rule dependency |
| Contexts, actions, bindings | `design/gdd/player-controller-input.md` | PC-2 (PCI:62), `gameplay`-context default binding table (PCI:64-80; no `chart_table`-context table, ASK-CT-2), PC-13 (PCI:141), PC-16 (PCI:147), `BINDING_REQUIRED{chart_table}` (PCI:699), Interactions row 17 (PCI:217), row 13 (PCI:213), PCI:845 (Settings), PCI:656-658 (pause and focus-loss edge cases), `DEVICE_SWITCH_HYSTERESIS`, `MOVE_DEADZONE`, `MOVE_DEADZONE_EXIT` (all three PCI; `DEVICE_SWITCH_HYSTERESIS` registry, the two deadzones not registered) | Rule dependency |
| The console press | `design/gdd/interaction-system.md` | IA:131 (`pod_hop` declaration "Web Map's"), CONSOLE subject, `tool_required` NONE, ASK-I1 / ASK-I2 (as cited by WM) | Rule dependency (seam ASK-CT-1) |
| Seat defaults at New campaign | `design/gdd/suit-and-equipment.md` | SE-9 (SE:189), `create_seat_beam` (E9), `bind_beam` (E4), `BEAM_WIDE`; SE:476; SE:1289, SE:1317 (no weight readout) | Rule dependency, Ownership handoff |
| UI-bus cues | `design/gdd/adaptive-audio-and-tells.md` | AA:562, AA:1086 | Ownership handoff |
| No Attention surface | `design/gdd/attention-system.md` | AT:1373; AC-AT-61, AC-AT-66; AT-17, AT-27 | Rule dependency |
| No horror element | `design/gdd/horror-entities.md` | HE:377, HE:1358; AC-HE-119 | Rule dependency |
| No loot value or salvage count; fuel clause overruled | `design/gdd/boarding-and-salvage.md` | BS:334; BS:1204 (conflict C1); AC-BS-47c (BS:1427); BS-16 | Rule dependency |
| No sanity indicator; screenshots at stage 0 and forced stage 3 | `design/gdd/sanity-and-hallucination-rules.md` | SN:1226 (TRUTH anchor), SN:1269, SN:1350, SN:1357; AC-SN-83, `force_stage` | Rule dependency |
| Layer 3 is TRUTH | `design/gdd/perception-filter.md` | PF:223 (row 9); PF-20 | Rule dependency |
| No inventory screen | `design/gdd/item-and-module-database.md` | IDB R8 (IDB:1065, :1320) | Rule dependency |
| No authored chart prop | `design/gdd/derelict-authoring-handmade-kits.md` | DA:688 | Rule dependency |
| The player never operates the Graph as a menu | `design/gdd/life-support-graph.md` | LSG:22, LSG:32 | Rule dependency |
| Index rows and corrections | `design/gdd/systems-index.md` | IDX:40, :47 (Knowledge Codex), :95, :143 (PR-SCOPE declined), :201, :210, :239; C5 | Rule dependency |
| Pillars | `design/gdd/game-concept.md` | P1 CON:208-221, P2 CON:223-229, P3 CON:231-237, P4 CON:261-265, P5 CON:272-279; MDA CON:82-89; session shape CON:181-185; MVP CON:461-485; outside-playtester gate CON:483 | Rule dependency |
| Reflexion log | `docs/consistency-failures.md` | 2026-10-26 entries: UI grammar (linear bar, no ring), invariant under reachable states, placeholder in a table of record | Rule dependency |
| Engine reference | `docs/engine-reference/godot/` | `VERSION.md`, `breaking-changes.md`, `modules/ui.md` (dual focus, `focus_neighbor_*`, `tr()`), `modules/input.md` (device identification by event type), `modules/rendering.md`; no `scripting.md` or `core.md` exists | Rule dependency |

#### Art-bible references (`design/art/art-bible.md`)

| This document references | Specific element | Nature |
|---|---|---|
| Register, emotional target, descriptors, carried element | 2.11 (ART:256-266) | Rule dependency |
| Web-map grammar: no right angles, everything human rectilinear, curves only on dial arcs | 3.4 (ART:330-338) | Rule dependency |
| Palette roles (Ground, Graphite lit and mid, Hull Grey, Paper White, Tungsten, Void Steel), Violet and Flesh never UI, Alarm Red only on a failing instrument | 4.1, 4.4, 4.5, 4.6, 4.7 (ART:366-389, 455-469) | Rule dependency |
| Three layers by who produced the information; "Chart table (meta)" row; no minimap in play; the lit "chart-table console" row | 7.0 (ART:726-745) | Rule dependency (C4) |
| Two faces, type floors, tabular numerals, reading line length | 7.1 (ART:764-798) | Rule dependency |
| Glyph sizes 16 and 24, keycaps, focus is a shape | 7.2 (ART:799-826) | Rule dependency |
| Fades 80-120 ms, bracket 100 ms ease-out, drift at most 1 art px per second, hold bar linear with needle-mark | 7.3 (ART:849) | Rule dependency |
| Register, components, right-third panel, "Loss: no death screen", accessibility surface, ship-schematic console | 7.4 (ART:853-888) | Rule dependency |
| Crewmates on the map (VS-B) | 7.5 (ART:915, 919) | Rule dependency |
| Hard requirements HR-2, HR-4, HR-6, HR-8, HR-10, HR-11, HR-12, HR-13, HR-14, HR-15 | 7.7 (ART:968-987) | Rule dependency |
| Death screen, load black | 2.10 | Rule dependency |
| Asset naming and sizes | 8.2, 8.5 | Rule dependency |

#### Conflict and clarification rows raised or carried by this GDD (none edited here)

| # | Between | Statement | Resolution in this GDD |
|---|---|---|---|
| C1 | BS:1204 vs WM:1049 / WM-35 / AB-5 | "the Chart Table shows what the crew lit **and the fuel the hop will cost**" vs "no travel time and no fuel cost may sit beside [the selection]" | **WM governs** (structural: `web.view` has no fuel field); CT-35; route to Boarding to reword to "shows what the crew lit" |
| C2 | PIPE:296 row 7 vs the routed settings thread | Pipeline assigns "the settings UI of PR-14"; settings belong to Settings (Alpha) | CT-18 (three controls); route a narrowing of row 7 to the Pipeline owner |
| C3 | KRN:715 vs CT-27 | the `MAX_PAYLOAD` example names a "chart-table command with a path array"; this layer submits no command | stale example; route to the kernel to replace it |
| C4 | ART 7.0 row 1 vs row 3 | one name, two objects (the lit "chart-table console" vs the meta layer) | glossary in Overview; this GDD owns the meta layer only; the Controller's Adapter owns the `inspect_card` context, In-World Instruments owns the card's content and draws no schematic (IW-28) |
| C5 | IDX:95, :201 vs real edges | the index lists only Web Map & Transit and the Pipeline | Dependencies table; route to `/map-systems` |
| C6 | the four registry `referenced_by` comments (cited by count, not by line) | four pending `referenced_by` comments name `design/gdd/chart-table.md` | merge stage rewrites the path; drop or negative-sentence the `HOP_FUEL_ML` reference |
| C7 | SL:770 / SL UI-8 vs the kernel | "Chart Table issues the pause" where the kernel is not `RUNNING` | UI-8 and UI-9 are boot-flow screens (CT-25); route wording refresh to Save/Load |
| C8 | SN:1350 / :1357 vs IDB R8, SE, SL, ART | the evidence list names "the inventory" and "the death/restore screen"; neither exists | CT-37; route to Sanity |
| C9 | PIPE:688 vs PCI | the pipeline discards a zoom press while paused; the Controller binds zoom only in `gameplay` and `inspect_card` | consistent; stated in CT-5 |
| C10 | WM:203 "never assuming pause" vs "single-player pauses" | reconciled by CT-4: this layer pauses because `chart_table` calls `request_pause` in SOLO, but no logic depends on `PAUSED` | no change needed |
| C11 | PIPE:265 | the frozen world keeps its filtered lie under a true panel | intended; Edge Case and QA note (Edge Cases, Acceptance Criteria) |
| C12 | WM:1048 vs WM-36 | resolved by WM-36: `web.view` now publishes `leg_target_id` and `leg_dir` (endpoint identity only; the BACK-leg origin is not published) | CT-15 draws the OUT-leg route from them; ASK-CT-3 answered |
| C13 | WM:1309 vs SL UI-2 | `depth_label` "on the pause panel / save list" vs UI-2's single recency fact | main menu only (CT-17, CT-23); ASK-CT-4 to WM |
| C14 | KRN:151 vs PIPE:269 vs KRN:940 | FAULTED release returns "to the menu" vs "to the chart table" | CT-26: the boot-flow main menu; the notice string is OQ-CT-5 |
| C15 | PCI:656 vs CT-22 | "hold timers (confirm bar, name reveal) are frozen with the pause" vs a UI-3 bar that must run while the kernel is `PAUSED` | the PCI sentence concerns the Adapter's in-world timers; route a wording clarification to the Controller |
| C16 | ART 7.4 / WM:1050 | "each about 4 % darker; the inner ring reaches Void Steel" cannot both hold and stays near the visibility threshold | CT-13 ordering contract; route a wording clarification to the art-bible owner (flag F-6) |
| C17 | ART 7.4 selection row vs OQ-WM-34 | "label and distance beside it" vs "no label is required; none is derived" | CT-14; route a wording clarification to the art-bible owner |
| C18 | PIPE:188 (repeated at IWI:161) vs CT-27 | PR-8 reads "`INPUT_SURFACE` if it also issues commands - the HUD's prompts do not; the chart table does" vs this layer submitting zero commands at MVP | CT-27 governs at MVP; route to both owners (Pipeline for PIPE:188, Instruments for IWI:161) with the wording "holds the class for VS-B; submits nothing at MVP" |
| C19 | HS-42 / HS-48 vs CT-22 | HS-42 calls its hold bar "the same bar" and drains it on release at `CONFIRM_DRAIN_RATE`; this layer's UI-3 bar snaps to empty on abort (CT-22, rulings R19) | the shared component is the look only; route a clarification to the HUD & Suit Layer and the `interaction-patterns.md` UX spec: "the Chart Table's destructive-action bar snaps to empty on abort; the drain is the HUD's own behaviour" |

#### Asks raised (provisional seams, none decided by a neighbour yet)

| Id | To | Question | Degraded if refused |
|---|---|---|---|
| ASK-CT-1 | Interaction, Player Controller | the Adapter injects the host-local pick into `pod_hop`'s payload and routes a no-pick press to "open chart" | a words-only refusal from Interaction's table |
| ASK-CT-2 | Player Controller, Settings | the `chart_table` context's navigation and accept actions (reuse `interact` and `ui_*`, or add per-context actions) | this GDD cites them as provisional |
| ASK-CT-3 | Web Map & Transit | a non-progress leg field in `web.view` | answered: `leg_target_id`, `leg_dir` (WM-36); BACK-leg origin not published |
| ASK-CT-4 | Web Map & Transit | clarify WM:1309 (save list = main menu) | as ruled in CT-17 |
| ASK-CT-5 | Save/Load | a verification-failed flag on `campaign_info()` | UI-7 line is HUD's on the first `RUNNING` frame |
| ASK-CT-7 | Save/Load, Pipeline | blocking or chunked restore; the minimum presented black frame | CT-42 assumes one presented black frame |
| ASK-CT-8 | art-director | does HR-13 cover a player-initiated full-bleed toggle? | no guard; `CHART_REOPEN_MIN_MS` stays in Open Questions |
| ASK-CT-9 | Kernel | same-boundary `request_pause` + `request_resume`: coalesced or in order | layer-side desired-state re-issue (CT-4) |
| ASK-CT-10 | Web Map & Transit | compact-list neutral ordinals (CT-44; the single exception named in G6 and CT-14) vs WM:1044 "no label" | the list shows no label text, which breaks the text-label rule (CT-30); the ordinal carries no identity either way |
| ASK-CT-11 | Kernel, Save/Load | what is the kernel state while the main menu is shown, and what is the Quit path there (the kernel's table lists `request_shutdown` only from RUNNING, PAUSED and FAULTED, none from BOOTING)? And the window-close path while the kernel is `PAUSED` under the open layer (EC-CT-38)? | boot-flow Quit ends the process through boot code with no kernel call (no SIM state exists); FAULTED-return Quit keeps `request_shutdown` (TX-30); a window close under the open layer closes the layer and defers to Save/Load's handler (EC-CT-38) |
| ASK-CT-12 | Pixel-Perfect Render Pipeline | add a layer-3 term (`D_L3`, open-layer only) to PIPE D.8 `D_root`, or rule that layer 3 shares `D_hud` while open | the budget stays "at most `D_hud`'s upper bound" and AC-CT-45 reports it as a separate series |

(The id ASK-CT-6 is intentionally unused: the FAULTED notice is OQ-CT-5.)

#### Registry candidates (to be written by the merge stage from the final text, not from this draft)

Owned `[PLACEHOLDER]` constants: `CHART_FADE_MS`, `COMPACT_LAYOUT_MIN_CANVAS_W`, `SNAP_SECTOR_DEG`, `FILAMENT_DRIFT_STEP_PX`, `CHART_RING_STEPS`, `LAYER3_DRAW_CALL_BUDGET`, and the further layout knobs listed in Tuning Knobs (`PANEL_INNER_PAD_PX`, `FILAMENT_DRIFT_FRACTION`, `UI_DT_MAX_MS`, `SETTINGS_WRITE_DEBOUNCE_MS`, `UI_NAV_REPEAT_MS`, `UI_NAV_REPEAT_RATE_MS`, `RING_STEP_MIN_CHANNEL`, and the derived `CHART_FRAME`); host-local / formula entries `chart_pick` and `legal_targets`, explicitly "not a state block". **Not registered under new names:** every art-bible number (fades 80-120 ms, bracket 100 ms, drift ceiling, ring-step wording, type floors, glyph sizes, panel third) and every locked value cited above (`CONFIRM_HOLD_S`, `H_default`, `HUD_MIN_SCALE`, `cap_px`). `CHART_TO_PRESS_S` is a playtest observable and is not registered.

## Acceptance Criteria

*Ids `AC-CT-1` to `AC-CT-111` (plus 47b, 75b-75d). Numbering is append-only: criteria cited by number in rule and formula text (D.1b to AC-CT-26, D.4 and EC-CT-9 to AC-CT-24, D.6 to AC-CT-47, D.10 to AC-CT-45, CT-4 and EC-CT-73 to AC-CT-2) keep their number; the qa-lead's corrections were applied to the criteria in place, and new criteria take 82 onward. Each criterion names the rule, formula, transition or edge case it traces to; a coverage matrix closes the section. "Without opening another document" is the standard: every term a tester needs is defined in the glossary or the fixtures below.*

### Gate convention, tags and story types

**Tags.** `(auto)` = automated unit or integration test with a test double, deterministic, no wall clock. `(lint)` = a static or content lint run in CI. `(review)` = a named reviewer reads a retained artefact against a checklist. `(manual)` = a person plays or inspects, with retained evidence. `(spike)` = measured once on the real display path and real faces, result retained. `(integration)` = run against the real neighbour modules. `(playtest)` = outside testers. Each tag carries its **story type** after a comma (Logic, Integration, UI, Visual, Config), which is how `/story-readiness` resolves the gate level from the coding-standards table.

**Gate levels (applied once, here).** `(auto)` Logic criteria and `(integration)` criteria are **BLOCKING**. `(lint)` criteria are BLOCKING once their CI job exists; **a lint AC marked "CI job to be built" is not verifiable until that job exists and needs its own story** (the source scans, fit-class measurement, layout-JSON lint, string scans, asset-name scan and scene-id whitelist are all new). `(manual)` UI and Visual criteria marked BLOCKING need a **retained screenshot** in `production/qa/evidence/chart-table/`: a screenshot captured and discarded is an assertion, not evidence. `(review)` criteria are BLOCKING when they gate a UI or Visual story, and say so. `(spike)` and `(playtest)` are **ADVISORY** unless stated.

**Sanity stages.** Screens reachable in a session (panel, chart, Display & Comfort, `UI3_PLATE`, the `FAILED` panel) are captured at **sanity stage 0 and forced stage 3** (`force_stage`, AC-SN-83). **Boot-flow screens** (notice, main menu, UI-5, UI-7 / UI-8 / UI-9, the load hold) exist when the kernel is not `RUNNING`, so no session, no Sanity and no `force_stage` exist: they are captured **once** (stage "n/a"), plus a structural lint that boot-flow scenes reference no Sanity, Filter or Attention symbol and are not children of the Filter's viewport (AC-CT-46). The FAULTED-return menu is reachable after a session and is captured with `force_stage` 3 active before the fault. *(orchestrator ruling: qa-lead R3; the draft's "both stages" for pre-session screens was physically unsatisfiable. Whether `force_stage` should work pre-session is OQ-CT-18.)*

### Glossary (terms the criteria use)

`HOME` = the home-ship node; `HOME_ID` = the `derelict_id` of the `web.view.nodes[]` entry whose `kind` is the home ship, never a constant (K16). `ChartPick` = the layer's host-local pending target, NONE or one `derelict_id` (CT-7). **Legal target** = a member of `legal_targets(view)` (D.4). `PANEL`, `CHART`, `CHART_VIEW_ONLY`, `DISPLAY_COMFORT`, `SAVE_QUIT_WAIT`, `SAVE_FAILED`, `UI3_PLATE`, `HOLD_CONFIRM`, `UI5_NEW_OVER_EXISTING`, `MAIN_MENU`, `BOOT_NOTICE`, `LOAD_HOLD`, `CLOSED` = the layer's states (States and Transitions). `UI-3`, `UI-5`, `UI-7`, `UI-8`, `UI-9` = the Save/Load surfaces of SL:998-1005 as this layer draws them (UI-3 = `UI3_PLATE`; UI-5 = `UI5_NEW_OVER_EXISTING`). `can_save_quit` = `PauseGate`'s one boolean (CT-4). `PauseGate` = the single adapter to the kernel control plane. `H` = HUD integer scale; `k` = integer world scale (never the zoom step); "row 1 to 6" = the canvas rows of the table below. "WM-36 list" = the `web.view` field list: `nodes[]` {`derelict_id`, `template_id`, `kind`, `ring`, `world_slot`, `status`, `unknown_seen`, `loaded`, `berthed`}, `pod_at`, `transit_state`, `destination_ready`. `IN_FLIGHT` = the `pod_at` value while the pod is between nodes. `[UNV]` = unverified against the engine or real faces.

**The six canvas rows (D.1, D.1b), inlined.** `P_w x P_h` is the picture in screen px; the HUD canvas is `P_w / H` by `P_h / H`.

| Row | Picture, mode, `k`, `H` | Canvas (HUD px) | `W_panel` | `safe_x` / `safe_y` | `content_w` | `V_avail` | `m` | Layout |
|---|---|---|---|---|---|---|---|---|
| 1 | 1920x1080 keep, 3, 3 (same canvas: 2560x1440 at `H` 4, 3840x2160 at `H` 6) | 640 x 360 | 213 | 32 / 18 | 177 | 324 | 2 | spatial |
| 2 | 1920x1080 keep, 3, 6 | 320 x 180 | 106 | 16 / 9 | 86 | 162 | 1 | spatial |
| 3 | 1280x800 expand (Deck), 2, 3 | 426.67 x 266.67 | 142 | 22 / 14 | 116 | 238 | 1 | spatial |
| 4 | 1280x800 expand, 2, 6 | 213.33 x 133.33 | 71 | 11 / 7 | 56 | 119 | 0 | compact |
| 5 | 1280x720 keep, 2, 3 | 426.67 x 240 | 142 | 22 / 12 | 116 | 216 | 1 | spatial |
| 6 | 1280x720 keep, 2, 6 | 213.33 x 120 (smallest) | 71 | 11 / 6 | 56 | 108 | 0 | compact |

**Fixture constants** (cited from their owners so no criterion needs another document; none is a new value). `CONFIRM_HOLD_S` 0.8 s (Player Controller, registry; range 0.6-1.2). Drift ceiling 1 art px a second (ART 7.3). `DEVICE_SWITCH_HYSTERESIS` 250 ms (PCI, registry). Fade band 80-120 ms and bracket ease-out 100 ms (ART 7.3); `CHART_FADE_MS` 100. `UI_DT_MAX_MS` 50; `UI_NAV_REPEAT_MS` 350 / `UI_NAV_REPEAT_RATE_MS` 120; `RING_STEP_MIN_CHANNEL` 3; `S_MIN` 32; `SETTINGS_WRITE_DEBOUNCE_MS` 500; `L_BAR_HUD` 64; `CHART_FRAME` 288 x 162. Colours (ART 4.1, 4.6): Ground `#1C2128`, Void Steel `#0B1018`, Graphite lit `#5C6670`, Graphite mid `#2A303A`, Hull Grey `#8C949C`, Paper White `#DDE4EA`, Tungsten fixture `#FFCC66`, Alarm Red `#E0102A`; Violet hue 270-330 degrees with saturation above 25 %; Flesh hue 5-30 degrees, saturation 20-60 %, value 45-85 %.

**Test doubles assumed.** `ViewDouble` (feeds a `web.view` snapshot and records every call made on it, so a call to `web.leg_timing()` or a read of a progress field is observable). `KernelDouble` (records `request_pause`, `request_resume`, `request_shutdown`; exposes a settable state RUNNING / PAUSED / FAULTED / BOOTING and a count of `submit` calls). `SaveDouble` (scripted `SaveInfo`, `save_status`, `campaign_info()`, `restore_reason`, and the results `SL_QUIT_NO_WRITE_DOWNED`, `SL_QUIT_DEDUPED`, `SL_LOAD_NO_LOADABLE_SAVE`, incompatible-build; a variant that never returns, for the held write). `ClockDouble` (drives the layer's UI delta in integer microseconds). `StoreDouble` (the `SettingsStore` seam; missing, corrupt or read-only). `WindowDouble` (display-mode requests, refusing fullscreen on demand). `InteractionDouble` (the console seam). Every double used in a cross-system criterion is kept honest by AC-CT-109 (contract tests).

### A. Architecture, registrations and the pause answer

- **AC-CT-1** (auto, Logic) [CT-27] GIVEN the project's registries after boot with the layer loaded, WHEN the state-block, hook, command and mutator registries are enumerated, THEN no entry has the Chart Table as owner and no type introduced by the layer carries `allowed_when_paused` (the paused-allowed list of KRN CR-16 is unchanged).
- **AC-CT-2** (auto + lint CI job to be built, Logic) [CT-4, KRN Open Question 6, checkpoint Q17] GIVEN a `KernelDouble` fixed at RUNNING (never PAUSED), WHEN the full chart flow runs (open, move focus across every node, pick, close, open again with the pick kept), THEN every step behaves identically to the same flow with the kernel PAUSED, **and** a lint over the layer's sources finds no reference to `Kernel.state`, `State.PAUSED` or `is_paused()` outside `pause_gate.gd` (symbol names to be confirmed by the lead-programmer). Pinned wording: **no rule reads `PAUSED`; the chart is functional with the kernel RUNNING.**
- **AC-CT-3** (lint CI job to be built + auto, Logic) [CT-27, WM AC-WM-88 / AC-WM-109] GIVEN the layer's sources, WHEN scanned for the kernel's command-submission call (symbol name to be confirmed by the lead-programmer), THEN no submit path originates in the layer; **and** GIVEN a `KernelDouble`, WHEN a full session through every state of the machine runs, THEN the submit count is 0 and the only kernel calls are `request_pause`, `request_resume` and (TX-26, TX-30 on the FAULTED-return menu) `request_shutdown`. The layer cannot cause a command except through the console press, which is Interaction's.
- **AC-CT-4** (auto, Logic) [CT-15, CT-27, CT-35, WM:300] GIVEN a `ViewDouble` that records calls, WHEN every chart state is exercised (HOME, UNSEEN, SEEN, BOARDED, `unknown_seen`, `berthed`, ABSORBED, ORPHAN, `pod_at` HOME / derelict / IN_FLIGHT, LEG, ARRIVING, BERTHING), THEN `web.leg_timing()` is never called and every field read belongs to the WM-36 list; **and** GIVEN a view with an injected field named like a progress datum (elapsed ticks, remaining ticks, `t0`, `n`), THEN no drawn element changes.
- **AC-CT-5** (auto, Logic) [CT-4, TX-13, TX-14] GIVEN mode SOLO and the layer CLOSED, WHEN `chart_table` is pressed, THEN the layer is in PANEL on that frame, `request_pause` is called exactly once and default focus is Resume; GIVEN mode OVERLAY, THEN the same panel opens with **zero** kernel calls.
- **AC-CT-6** (auto, Logic) [CT-4, EC-CT-13, ASK-CT-9] GIVEN SOLO, WHEN `chart_table` is pressed twice inside one tick boundary (a `KernelDouble` that coalesces, then one that executes in order), THEN in both cases the layer's `open` boolean equals the kernel's final pause state within two ticks and no half-open state persists (`PauseGate` re-issues the latest desired state). *(provisional — ASK-CT-9, kernel: proven against the real kernel in AC-CT-109.)*
- **AC-CT-7** (auto, Logic) [CT-3, R2; qa-lead R1; Q-B ruling] GIVEN every combination of {`can_save_quit` true or false} x {`save_status` IDLE / WRITTEN / FAILED} x {`transit_state` IDLE / LEG / ARRIVING / BERTHING} x {a `SaveDouble` that returns `SL_QUIT_NO_WRITE_DOWNED` or not} x {state PANEL or `SAVE_QUIT_WAIT`}, WHEN the panel is drawn, THEN the items are Resume, Chart, Display & Comfort, Save & Quit in that order and none is absent; Resume, Chart and Display & Comfort are enabled in PANEL for every combination (**Chart is enabled while `transit_state != IDLE`: it opens view-only, TX-17**; *orchestrator ruling Q-B*); Save & Quit is enabled iff `can_save_quit` is true; in `SAVE_QUIT_WAIT` all four are disabled; a disabled item is drawn in Graphite lit `#5C6670` and activating it does nothing.
- **AC-CT-8** (auto, Logic) [CT-1, PCI:62] GIVEN the Controller double in the `gameplay` context and then in `inspect_card`, WHEN `chart_table` is pressed, THEN the layer opens in both; GIVEN the `chat` context, THEN it does not.
- **AC-CT-9** (auto, Logic) [CT-5, TX-15, TX-19] GIVEN the layer at depth 2 (Chart, Display & Comfort, or a message plate), WHEN `cancel` is pressed, THEN the layer is PANEL; GIVEN PANEL, WHEN `cancel` is pressed, THEN it is CLOSED; GIVEN any depth, WHEN `chart_table` is pressed, THEN it is CLOSED (SOLO: one `request_resume`).
- **AC-CT-10** (auto + manual, Logic) [CT-5, CT-22, TX-10, EC-CT-11] GIVEN a hold fill at 50 %, WHEN each of release, `cancel`, `chart_table`, pointer exit and window focus loss occurs in turn, THEN in each case the bar is empty in the same frame, the state returns to the screen that owns the item (`UI3_PLATE` or `UI5_NEW_OVER_EXISTING`), no other state changes, and the layer is still open; a second `chart_table` then closes it.
- **AC-CT-11** (auto, Logic) [CT-5, EC-CT-18, PIPE:688] GIVEN the layer open, WHEN a wheel step, `=`, `-` and d-pad up / down are delivered, THEN none changes any state, no zoom hint or glyph exists on any surface, and the camera's zoom step is unchanged on close.
- **AC-CT-12** (auto, Logic) [CT-41(1)(2), EC-CT-19, EC-CT-20] GIVEN a console press that opens the chart, and then its release and auto-repeat, THEN no control activates; GIVEN `interact` on Resume at depth 1 with the player standing at the pod console, WHEN the layer closes, THEN gameplay receives no `interact` edge and the Adapter submits no `pod_hop` until a fresh press.
- **AC-CT-13** (auto, Logic) [CT-41(3), EC-CT-10] GIVEN `chart_table` held for 2 s with key-echo events, THEN the layer toggles exactly once. *(engine behaviour for echo is NOT SOURCEABLE: re-check on the real engine, gap F-F.)*
- **AC-CT-14** (auto + manual, UI, BLOCKING) [CT-6, EC-CT-14, EC-CT-15; `modules/ui.md` dual focus; qa-lead R12] **(auto half)** GIVEN a scripted pointer-motion event followed by `ui_down`, THEN key / pad focus stays on the retained control and moves from there. **(manual half)** GIVEN mouse hover on item X and gamepad focus on item Y, THEN X's label is Tungsten and the bracket and bezel thickening are on Y; pad accept activates Y; a click activates X and moves key / pad focus to X; WHEN the mouse moves, THEN the bracket is hidden on that motion (PC-13) and key / pad focus is retained; keycap glyphs swap after `DEVICE_SWITCH_HYSTERESIS` (250 ms); WHEN a pad direction is pressed, THEN the bracket reappears on the retained focus. Run on both input paths and retain screenshots of each state (a clip may supplement but does not replace them).
- **AC-CT-15** (auto, Logic) [CT-6, EC-CT-16] GIVEN a pointer-down on item A and a pointer-up on item B (and again off all items), THEN no item activates and no fill remains.

### B. The pick, the mirror and the console seam

- **AC-CT-16** (auto, Logic) [CT-7, EC-CT-5] GIVEN the pod at HOME, `transit_state` IDLE and legal targets {A, B}, WHEN accept lands on A, THEN `ChartPick = A`; accepting A again leaves it A; accepting B replaces it; accepting HOME clears it; accepting an ORPHAN or `berthed` node changes nothing.
- **AC-CT-17** (auto, Logic) [CT-8, EC-CT-1, EC-CT-2] GIVEN `ChartPick = A`, WHEN (1) a `request_restore` occurs for each of `NEW_GAME`, `LOAD_GAME`, `DEATH_STUB`, `FAULT_RECOVERY`, (2) `pod_at` leaves IN_FLIGHT (arrival), (3) A becomes ABSORBED, (4) A becomes ORPHAN, (5) A becomes `berthed`, (6) A is absent from `web.view`, THEN `ChartPick = NONE` in each case on that change; **and** the bracket does not move unless the focused node itself left the focusable set (then focus is on HOME); **and** no message is shown.
- **AC-CT-18** (auto, Logic) [CT-8, CT-15, EC-CT-24] GIVEN `transit_state` in {LEG, ARRIVING, BERTHING} (and separately `pod_at == IN_FLIGHT`), WHEN the chart opens or is already open, THEN the layer is `CHART_VIEW_ONLY`, no node is focusable or activatable, focus is on Back, the pick cannot change, and **no element is drawn from `ChartPick`** (render the chart with the pick set before the departure and again with it NONE: the two renders are identical).
- **AC-CT-19** (auto with an `InteractionDouble`, Integration) [CT-10, ASK-CT-1, EC-CT-3, EC-CT-4] GIVEN the pod at HOME: WHEN the console is pressed with a valid pick, THEN the injected command is `pod_hop{target_id := ChartPick}`; WHEN pressed with no valid pick and exactly one legal target, THEN the chart opens with focus on HOME, no command is submitted and no refusal is produced; GIVEN the pod at a derelict, WHEN pressed, THEN the command carries HOME and the pick is not read. *(provisional — ASK-CT-1: if refused, the degraded outcome is a words-only refusal from Interaction's table and this criterion then reads "the chart does not open".)*
- **AC-CT-20** (auto + review, Logic) [CT-7, CT-10, checkpoint Q13] GIVEN a node that is focused but not picked, THEN no dashed line is drawn; GIVEN a pick from HOME, THEN the Tungsten dashed line runs HOME to the pick; GIVEN the pod at a derelict, THEN it runs from the pod's node to HOME; GIVEN `transit_state != IDLE`, THEN no dashed line is drawn. A reviewer confirms the line is the only pick mark and that its meaning is "the target the next console press carries".
- **AC-CT-21** (auto, Logic) [D.4, CT-9] GIVEN the D.4 examples, THEN `legal_targets` equals: {A, B} (A UNSEEN, B BOARDED not berthed, pod at HOME); {A} (B berthed); {} (B ABSORBED, A ORPHAN); {HOME_ID} (pod at A); {} (`pod_at == IN_FLIGHT`); {} (`transit_state == LEG`); {A, B} for the roster {A SEEN, B UNSEEN, C ABSORBED, D SEEN and berthed}.
- **AC-CT-22** (auto, Logic) [CT-9, CT-11, WM:1046] GIVEN an ORPHAN record, THEN it is drawn Graphite mid, detached, with no tick, is neither focusable nor selectable, and snap skips it; GIVEN an ABSORBED record, THEN it is drawn only as part of the HOME silhouette and has no node of its own.
- **AC-CT-23** (auto + review, Logic) [CT-9, CT-14, WM:1299] GIVEN an UNSEEN derelict, THEN it is a selectable node drawn as the plain Graphite-lit rectangle with no dialect, no tick and no label of any kind in the spatial layout (the compact layout's CT-44 neutral ordinal is the sole exception and is the identical key form as a SEEN row's, AC-CT-112); and rings 2 and 3 carry no node, tick or label.
- **AC-CT-24** (auto, Logic) [CT-9, CT-35, checkpoint K6] GIVEN two `ViewDouble` snapshots that differ only in the Graph's fuel level (and separately in hop-timer state, `destination_ready` and a `BERTH_PENDING` condition), WHEN `legal_targets` and the chart render are computed for each, THEN the selectable set, the focus order and the rendered pixels are identical. A fuel change must leave selectability unchanged.
- **AC-CT-25** (manual, Visual, BLOCKING) [CT-11, CT-2, WM:1209, AC-WM-60] GIVEN the chart, WHEN captured for each state UNSEEN, SEEN, BOARDED, `unknown_seen`, `berthed`, ABSORBED, ORPHAN, `pod_at` HOME, `pod_at` derelict, `pod_at` IN_FLIGHT (the `IN TRANSIT` word at its fixed position and no marker) and BERTHING (no word), at sanity stage 0 and forced stage 3, THEN each shows no cost, no progress datum, no red, no violet and no distance label, and every state is distinguishable without colour (silhouette, tick, seam, detachment or glyph); retained in `production/qa/evidence/chart-table/`.
- **AC-CT-26** (auto, Logic) [D.1b, CT-12, CT-16; qa-lead R13] GIVEN the six canvas rows, WHEN `m`, `x0`, `y0` are computed, THEN `m` = 2, 1, 1, 0, 1, 0 for rows 1-6; `x0` = 32, 16, 69 and `y0` = 18, 9, 52 for rows 1, 2, 3; for row 5 `x0` = 69 and `y0` = 39; and for every row with `m >= 1` the invariants `CHART_FRAME_W * m + 2 * safe_x <= P_w div H` and `CHART_FRAME_H * m + 2 * safe_y <= P_h div H` hold (576 + 64 <= 640 and 324 + 36 <= 360 at row 1; 288 + 32 <= 320 and 162 + 18 <= 180 at row 2; 288 + 44 <= 426 and 162 + 28 <= 266 at row 3; 288 + 44 <= 426 and 162 + 24 <= 240 at row 5), with `safe_x` / `safe_y` from registry `safe_margin` (rounded up). *(the draft's seventh canvas, 3440x1440, has no source row in CT-16 and was dropped.)*
- **AC-CT-27** (auto, Logic) [D.1, CT-16; qa-lead R13] GIVEN the six rows, THEN `W_panel` = 213, 106, 142, 71, 142, 71; `safe_x` = 32, 16, 22, 11, 22, 11; `safe_y` = 18, 9, 14, 7, 12, 6 (registry `safe_margin`, rounded up); `content_w` = 177, 86, 116, 56, 116, 56; `V_avail` = 324, 162, 238, 119, 216, 108; and the panel's left edge in screen px (`P_w - W_panel * H`) is an integer at every row (1281, 1284, 854, 854, 854, 854).

### C. Layout, fit and compact layout

- **AC-CT-28** (auto + lint CI job to be built, Logic) [D.1, CT-16] GIVEN row 6 (213.33 x 120), THEN `V_need = 83 <= V_avail = 108` with `n_wrap = 0`; WHEN the `ct.*` fit-class assignment yields `n_wrap > 3` at row 6, THEN the lint fails.
- **AC-CT-29** (lint CI job to be built, Config) [CT-16, CT-11; qa-lead R11] GIVEN every layer-3 scene, THEN every anchor is 0 or 1 (or belongs to a centred element whose on-screen width is an even number of screen pixels at every reachable `H`) and every offset is an integer HUD px; **and** every route and seam quad has whole-art-px position and size, and the layer's draw code contains no `draw_line` of width 1 on a transformed canvas item.
- **AC-CT-30** (auto + manual, UI, BLOCKING) [CT-44, D.1b] GIVEN rows 1-6, THEN the chart is spatial at rows 1, 2, 3, 5 and compact at rows 4 and 6; a forced-`H` test hook reproduces each row; the compact capture at 213.33 x 120 is retained at both sanity stages and shows one column of items, each with its glyph and neutral ordinal (one whole key), no rings and no filaments, the border and rivets present, the `IN TRANSIT` word (when in flight) at its fixed top-left position and no row truncated.
- **AC-CT-31** (auto, Logic) [CT-44; qa-lead R10] GIVEN the compact list with records in layout-file order, WHEN a record becomes SEEN, BOARDED, `berthed`, ORPHAN or ABSORBED, THEN the relative order of the remaining rows never changes, an ORPHAN row stays in its slot (non-selectable), an ABSORBED record leaves the list, and an UNSEEN record's label has the same form as a SEEN record's. The order is never sorted by status, distance, fuel, danger, recency or seen / unseen. In the compact list the planned-sortie relation is a Tungsten dashed rule under the picked row and no other mark. A record's ordinal key is unchanged by every one of those status changes (AC-CT-112).
- **AC-CT-32** (lint CI job to be built, Config) [D.8, CT-28] GIVEN every `ct.*` key, each of the six rows and the real faces (the estimate `ADV_EM` is replaced by measured widths; **NOT SOURCEABLE until the reading face is imported**), THEN every `LABEL_1` fits one line, every `LABEL_2` at most two, with `TEXT_EXPANSION_TEST` applied; no label is truncated or ellipsised at runtime; the closed set of instrument-face strings contains only `IN TRANSIT`, numerals, keycap letters and short state words.
- **AC-CT-33** (lint CI job to be built, Config) [CT-12, D.1b, D.3, EC-CT-54, EC-CT-55] GIVEN `chart_layout.json`, THEN every record in the roster has a position and a stable ordinal; no position is computed from `world_slot`, legs, ticks or distance; every pairwise centre distance is `>= S_MIN` (32); the authored distances show no correlation with any Web Map & Transit cost value; the snap graph over the layout is strongly connected; WHEN an entry is removed, THEN the lint fails the build and the runtime falls back to slot `(ordinal - 1) mod N_FALLBACK_SLOTS` with a log line; WHEN a roster cannot fit the compact list, THEN the lint fails the build and the row order is not changed to fit.
- **AC-CT-34** (auto, Logic) [D.3, CT-44] GIVEN the D.3 example layout (HOME (144, 100), A (210, 60), B (80, 40)) and `SNAP_SECTOR_DEG` 90, THEN from HOME: right gives A, up gives A (fallback), left gives B, down stays; from A: left gives HOME; a node exactly on a diagonal belongs to exactly one sector; the result is independent of the order candidates are listed in; no direction wraps.
- **AC-CT-35** (auto, Logic) [D.9] GIVEN a direction held for 300 ms and 1,000 ms with 350 / 120, THEN `n_moves` is 1 and 7; a held direction at the last node produces no wrap; a stick on the deadzone boundary produces one move per edge.

### D. Time, hold, motion and colour

- **AC-CT-36** (auto, Logic) [D.2, CT-22, CT-43; qa-lead R4] GIVEN a `ClockDouble` and `CONFIRM_HOLD_S` at its registry value 0.8 s (threshold 800,000 microseconds), THEN (a) at `dt_us` 16,667 the action fires on frame 48 and not frame 47 (`t_us` 800,016 against 783,349); (b) at `dt_us` 33,333 it fires on frame 25 (frame 24 = 799,992); (c) at `dt_us` 100,000 (10 fps) each frame adds 50,000 microseconds (clamped), so it fires on frame 16, i.e. 1.6 s of wall time, twice the nominal hold (accepted by D.2); (d) repeating (a) with `CONFIRM_HOLD_S` 0.6 s fires at frame 36 and with 1.2 s at frame 72; (e) one 400 ms frame advances `t_us` by exactly 50,000; (f) with the `KernelDouble` PAUSED the bar still advances (the UI clock runs while the kernel is paused).
- **AC-CT-37** (review + manual, Visual, BLOCKING) [CT-22, ART 7.3] GIVEN the hold bar captured at 0 %, 50 % and 100 %, THEN it is a linear bar with a needle-mark, never radial, with no easing, no overshoot and no colour change beyond the fill, the needle protrudes beyond the bar edges and the bar reads in a greyscale capture; the destructive item looks like every other item (no red, no warning mark) until the bar fills; and reduced motion on does not stop the fill.
- **AC-CT-38** (auto + manual, UI, BLOCKING) [CT-15, WM:1048, WM:1307] GIVEN `pod_at == IN_FLIGHT`, THEN the `IN TRANSIT` word is drawn at its fixed top-left position, appears and disappears in a cut with the data, and no pod marker is drawn; GIVEN BERTHING with `pod_at` not IN_FLIGHT, THEN no word and no progress element appears; the word never changes position across LEG and ARRIVING.
- **AC-CT-39** (auto, Logic) [D.5, CT-13; qa-lead R5] GIVEN the ceiling fixture of 1 art px a second (ART 7.3), THEN at defaults (`FILAMENT_DRIFT_FRACTION` 0.5, step 1) `offset_px` is 1 at 3.9 s and 2 at 4.0 s; at fraction 1.0 the rate is exactly 1 px a second and never above; a configured fraction above 1.0 or below 0.25 is clamped or rejected with a log line (AC-CT-102); one 400 ms frame advances `phi` by exactly 0.025 art px at defaults (not 0.2); reduced motion switched on at `phi` = 50 holds offset 50 for 30 s of UI time with no snap-back, and switched off resumes from 50 and reaches 51 after 2 s; the offset wraps modulo `P_f`; the filaments of rings 2 and 3 never change offset.
- **AC-CT-40** (auto, Logic) [D.6, CT-13] GIVEN Ground (28, 33, 40), Void Steel (11, 16, 24) and `CHART_RING_STEPS` 3, THEN the tones are (28, 33, 40), (22, 27, 35), (17, 22, 29), (11, 16, 24); every per-channel step is at least `RING_STEP_MIN_CHANNEL` (3); the disc equals Void Steel exactly; every MVP node sits on the ring-1 tone.
- **AC-CT-41** (auto, Logic) [CT-40] GIVEN the same chart rendered with every simulation input varied (sanity stage 0-3, any Attention value, any depth ring, session time 0 and 10 hours), THEN the drift offset sequence (for a fixed UI clock) and every ring tone are identical.
- **AC-CT-42** (auto + manual, UI, BLOCKING) [CT-18, CT-19, TX-20; qa-lead R16] GIVEN the Display & Comfort page, THEN it holds exactly three controls (display mode, reduced motion, reduce flashing), default focus is on the first control, and none of zoom, rebinding, resolution, world scale, HUD scale, frame cap, V-sync, audio, ambient step or any true-value overlay exists on it; a change applies immediately with no Apply button and no revert countdown; GIVEN a `WindowDouble` that refuses fullscreen, THEN the page shows windowed after read-back and the store holds windowed; WHEN the window resizes, THEN the page re-lays-out with focus retained by item id, the bracket rectangle's origin in HUD px relative to its item is identical before and after, and no animation plays. Retained capture counts toward AC-CT-78.
- **AC-CT-43** (auto + lint CI job to be built, Logic) [CT-19, EC-CT-45, EC-CT-46] GIVEN a `StoreDouble` that is missing, corrupt or read-only, WHEN the game boots, THEN defaults load, the photosensitivity notice is shown (the safe direction), a write failure keeps values in memory without a player message and a corrupt file is rewritten at the next flush; **and** a lint finds that nothing in the simulation reads the store, and that the store is not a kernel block, command, hook or mutator.
- **AC-CT-44** (auto + lint + manual, UI, BLOCKING) [CT-20, PIPE PR-13(e); qa-lead R3, R8] GIVEN a first launch, THEN the notice is shown before the main menu with the two toggles inline and one dismiss action, default focus is the first toggle; it has no countdown, no auto-dismiss, no hold and no animated element; the dismiss label is a `ct.*` key that is not "Continue"; dismissal writes `notice_seen` and a second launch skips it; GIVEN a session with `notice_seen` unset, WHEN `chart_table` is pressed, THEN the panel opens and the notice does not (it never appears mid-session); a missing store shows the notice again (AC-CT-43). The capture is retained **once** (a boot-flow screen: no sanity stage exists).
- **AC-CT-45** (spike, ADVISORY until `LAYER3_DRAW_CALL_BUDGET` is set) [D.10, LAYER3_DRAW_CALL_BUDGET, Pipeline D.8; qa-lead R14] GIVEN the chart open with three nodes and a route on the real display path at 1080p and at the Deck canvas, WHEN `Viewport.get_render_info` canvas draw calls are read on the root viewport for 300 frames, THEN `D_L3` is reported against `LAYER3_DRAW_CALL_BUDGET` and the root total against the Pipeline's root budget, as a separate series; the result states whether layer 2 draws beneath the open layer (OQ-CT-16). Reported, not failed, until the budget is written to the registry; the CSV is retained.
- **AC-CT-46** (auto + retained diff images + lint, Visual, BLOCKING) [CT-2, CT-34, AC-SN-83; qa-lead R3] GIVEN each in-session layer-3 screen (panel, chart, Display & Comfort, `UI3_PLATE`, the `FAILED` panel), WHEN rendered at sanity stage 0 and at forced stage 3, THEN the layer-3 rectangle is pixel-identical apart from the world visible beneath the panel, no gleam, glide, tremor or settle exists on any element, and the diff images are retained in `production/qa/evidence/chart-table/`; GIVEN the boot-flow scenes, THEN a lint finds no reference to a Sanity, Filter or Attention symbol and no scene is a child of the Filter's viewport.
- **AC-CT-47** (auto, Visual) [D.6, flag F-6, C16; qa-lead R6] GIVEN the chart captured at 1080p and at the Deck canvas, WHEN the ring bands are pixel-sampled, THEN adjacent bands differ by at least `RING_STEP_MIN_CHANNEL` (3) in every channel, the sampled tones equal D.6 ((28, 33, 40), (22, 27, 35), (17, 22, 29), (11, 16, 24)) and the disc is exactly `#0B1018`. **AC-CT-47b** (review, Visual, ADVISORY) a named reviewer records whether the inward darkening is visible on two real panels (one IPS, one OLED or the Deck screen); the outcome is recorded, not gated, and the remedy list (a raised `RING_STEP_MIN_CHANNEL` or a 1-px boundary line, never a percentage) stays in OQ-CT-19.
- **AC-CT-48** (auto, Visual) [CT-17, PIPE:265] GIVEN SOLO and the world rendered, WHEN the panel opens, THEN the pixels of the world region (left of the panel) are identical to the frame before `chart_table` was pressed: no dim, blur, desaturation, vignette, scrim or slow-motion is applied (layer-2 pixels excluded from the compare: an active HUD warning mark held ON solid per HUD HS-23 is not a defect, CT-3).
- **AC-CT-49** (auto, Logic) [CT-22, CT-24, CT-21] GIVEN every screen that carries a destructive action (`UI3_PLATE`, UI-5), WHEN it opens, THEN default focus is on the non-destructive item (Back, Retry or Retire), the destructive item never fires on a single press edge or on release, and a new gesture needs a new press edge after an abort.

### E. Saving, quitting and campaigns

- **AC-CT-50** (auto, Logic) [CT-21, TX-21, TX-22, K1; qa-lead R2; Q-C ruling] GIVEN `can_save_quit` true and a `SaveDouble` that does not return `SL_QUIT_NO_WRITE_DOWNED`, WHEN Save & Quit is accepted, THEN `save_and_quit()` is called once; all four items are disabled; `chart_table` and `cancel` are ignored; the pixel difference between the frame before the accept and the first frame after is **confined to the four item rectangles** (their disabled styling and the hidden bracket: with every item disabled nothing is focusable, *orchestrator ruling Q-C*); no pixel outside those rectangles changes, and no element, glyph, text run or animation exists that was not in the PANEL frame (no spinner, ring, bar, percentage, toast or text change); and the layer makes **no** `request_shutdown` call on this path (a `SaveDouble` that ends the process after `WRITTEN` and one that skips an unchanged-hash write produce the same layer behaviour).
- **AC-CT-51** (auto, Logic) [CT-21, CT-4, EC-CT-30] GIVEN `can_save_quit` false (the first tick after opening, OVERLAY without Session's form, or a confirmation pending), THEN Save & Quit is disabled (it stays focusable but inert) and `save_and_quit()` is never called.
- **AC-CT-52** (auto + manual, UI, BLOCKING) [CT-21, TX-24, EC-CT-29] GIVEN a `SaveDouble` that returns `SL_QUIT_NO_WRITE_DOWNED`, THEN nothing is written, `UI3_PLATE` opens with Back as default focus and the string stating that the newest autosave will be used, `cancel` returns to PANEL, `chart_table` closes (SOLO resume); and the string set contains no death language (SL:993).
- **AC-CT-53** (auto + manual, UI, BLOCKING) [CT-21, TX-23, TX-25, TX-32, K3; qa-lead R19; Q-F and Q-G rulings] GIVEN `save_status == FAILED`, or a `SaveDouble` that never returns while the `ClockDouble` advances past the Save/Load-configured timeout (`SAVE_QUIT_TIMEOUT_MS`, SL, not registered: gap F-D), THEN the layer is the PANEL with the plain word `FAILED` **replacing the playtime numerals** in the newest-record line position (*orchestrator ruling Q-F*; a word, not colour alone, not Alarm Red), the four items are in CT-3 order, Resume, Chart, Display & Comfort and `chart_table` work; accepting Save & Quit opens `UI3_PLATE` with Retry (default focus) and Quit without saving (hold); Retry returns to `SAVE_QUIT_WAIT`; repeated failure shows no counter and no escalation text; GIVEN `can_save_quit` false on that plate, THEN Retry is shown **disabled, not hidden** (*orchestrator ruling Q-G*) and no `save_and_quit()` call is made.
- **AC-CT-54** (auto, Logic) [CT-21, TX-26, EC-CT-35] GIVEN `UI3_PLATE`, WHEN the Quit-without-saving hold completes, THEN `request_shutdown` is called exactly once, nothing is written, and input after the call is ignored.
- **AC-CT-55** (auto, Logic) [CT-17, D.7, SL UI-2, C13; qa-lead R20] GIVEN `SaveInfo.playtime` of 9641, 0, 59 and 360000, THEN the newest-record line reads `02:40:41`, `00:00:00`, `00:00:59` and `100:00:00`; it is read once on panel open (WHEN the `SaveInfo` double's playtime changes while the panel is open, THEN the line does not change until the panel is reopened); the panel shows exactly one save fact and never `depth_label`, a "time since", a countdown, a gate state or a per-write mark.
- **AC-CT-56** (auto + manual, UI, BLOCKING) [CT-23, SL UI-4; qa-lead R9] GIVEN the main menu, THEN the items are Continue, New campaign, Quit in that order; Continue is disabled (not hidden) when `campaign_info()` is empty, and default focus is Continue if enabled else New campaign; the Continue line contains exactly: playtime, crew count, kind word, the `RISKY` word iff set, and `RING <n>` (the newest record's ring, never a maximum; AC-CT-113); the screen has no thumbnail, no completion figure, no wall-clock "last played", no title art, no logo animation, no tagline, no seed field and no generation or chain chooser; the backdrop is the depth-ring plate and ring-1 filament sheet with zero node glyphs (fallback plain Ground). Captured once (boot-flow), with and without a campaign.
- **AC-CT-57** (auto + manual, UI, BLOCKING) [CT-24, TX-5..TX-9] GIVEN no existing campaign, WHEN New campaign is accepted, THEN `new_game(world_seed)` is called after the CT-42 sequence and nothing else is called; GIVEN an existing campaign, THEN UI-5 opens with Retire as default focus, Delete as a separate hold-gated item, and Back; Retire calls `retire_campaign`, Delete (hold complete) calls `delete_campaign`, `cancel` returns to the main menu with focus on New campaign; no beam-shape picker exists and no call other than `new_game` is made at New campaign. Captured once (boot-flow).
- **AC-CT-58** (auto + manual, UI, BLOCKING) [CT-25, EC-CT-39..41; qa-lead R3] GIVEN the results UI-7, UI-8, UI-9, THEN no `request_pause` is made (boot-flow); UI-8 shows New campaign (default focus) and Quit; UI-9 shows the message, Quit and New campaign, and never overwrites or repairs the incompatible file; UI-7 is shown under Continue iff `campaign_info()` reports the verification failure *(provisional — ASK-CT-5)*, else it is the HUD's on the first `RUNNING` frame; each screen is captured once (boot-flow). GIVEN UI-9, WHEN New campaign is accepted, THEN UI-5 does not open, neither `retire_campaign` nor `delete_campaign` is called, and `new_game` is called after the CT-42 sequence.
- **AC-CT-59** (auto, Logic) [CT-26, TX-3, EC-CT-36; qa-lead R21] GIVEN the `KernelDouble` entering FAULTED in a release build with the layer open, THEN the layer closes and the main menu opens with Continue enabled and focused, no `request_pause` is made and no string beyond the `ct.*` table is shown (OQ-CT-5); Quit from that menu calls `request_shutdown`; from a boot-flow screen the layer makes no kernel call (the boot-code path of ASK-CT-11, tested only as "no kernel call").
- **AC-CT-60** (auto trace + manual, UI, BLOCKING) [CT-42, SL UI-6, PIPE:264; qa-lead R18] GIVEN Continue, New campaign, Retire-then-New or Delete-then-New accepted, THEN, observed through a frame-capture hook (or the Pipeline's frame log) that records for each presented frame the layer-3 draw list and the layer-1 state: all items are disabled and the plate is removed in one frame; at least one frame shows layer-1 black **and** an empty layer-3 draw list **before** the timestamp of the `continue_campaign()` or `new_game()` call; the first `RUNNING` frame is a camera `CUT`; nothing is drawn between black and that frame (no text, tip, bar, spinner); a second accept during the sequence does nothing (EC-CT-67).

### F. Negative requirements (one criterion per rule, each citing its source)

Criteria AC-CT-61 to AC-CT-64 and AC-CT-67 are checked in two parts: the **element whitelist** of AC-CT-107 proves the layer's scenes contain no member of the forbidden class (so a reviewer's silence is not the only evidence), and the **palette scan** of AC-CT-69 proves no Violet, Flesh or Alarm Red pixel; a reviewer then signs the retained captures.

- **AC-CT-61** (auto whitelist + review, UI, BLOCKING) [CT-31, AT:1373, AC-AT-61, AC-AT-66; qa-lead R7] GIVEN every screen of the layer at both sanity stages (in-session screens) and with every accessibility toggle, THEN no Attention indicator, meter, tier, hint or glyph appears; no legend, number, tier word, colour change or animation is attached to any ring and the tones are constant across Attention values 0 to max (AC-CT-41); there is no per-write autosave mark.
- **AC-CT-62** (auto whitelist + review, UI, BLOCKING) [CT-32, HE:377, HE:1358, AC-HE-119] GIVEN every screen under every accessibility setting, THEN no horror, threat bar, proximity indicator, direction, off-screen arrow, countdown, "last seen" pin or target ring is in the whitelist inventory or the captures, and this criterion is listed in the UX specs' own criteria.
- **AC-CT-63** (auto whitelist + review, UI, BLOCKING) [CT-33, BS:334, AC-BS-47c] GIVEN the chart with an UNSEEN, a SEEN and a BOARDED derelict, THEN no loot value and no remaining-salvage count appears anywhere on the layer.
- **AC-CT-64** (auto whitelist + review, UI, BLOCKING) [CT-34, SN:1350, SN:1357, AC-SN-83] GIVEN every in-session screen at stage 0 and forced stage 3, THEN no sanity meter, bar, ring, number, icon or colour is in the whitelist inventory or the captures, including under accessibility settings; the captures are retained.
- **AC-CT-65** (auto + review, UI, BLOCKING) [CT-35, CT-14, WM:1049, C1] GIVEN the string table and every captured chart state, THEN no fuel figure, travel time, ETA, danger rating, distance label, name beside a node or "recommended" marker exists; the only chart text is `IN TRANSIT`, control hints and (compact only) the CT-44 neutral ordinals; the chart is not changed by any fuel input (AC-CT-24).
- **AC-CT-66** (review + lint CI job to be built, Config) [CT-36, SL H5, SL:965, SL-24] GIVEN the layer, THEN no per-write autosave mark, thumbnail, spinner, percentage, "progress saved" toast, loading text, tip or death screen exists, and no asset by those names is in the layer's asset list.
- **AC-CT-67** (auto whitelist + review, UI, BLOCKING) [CT-37, IDB R8, SE:1289] GIVEN the layer's screens and the panel items, THEN no inventory, weight readout or loadout screen exists or is reachable.
- **AC-CT-68** (lint CI job to be built, with a signed allow-list, Config) [CT-28, SL AC-SL-86; qa-lead gap G5] GIVEN the `ct.*` table and the rendered strings, THEN none contains "load game", "slot", "checkpoint", "respawn", "save point", "game over", "you died", "progress saved", "quicksave", "recommended", "ETA", "cost", "spinner", "percentage" or "paused", nor "ring" outside `RING <n>` and the depth-ring negations; the word "Continue" is used only for the main-menu verb; no string has an exclamation mark, an emoji or a composed fragment; every allow-list entry is signed.
- **AC-CT-69** (auto pixel palette scan, Visual, BLOCKING) [CT-2, ART 4.6, HR-4] GIVEN every layer-3 screenshot in the evidence set, THEN no pixel falls in the Violet band (hue 270-330 degrees, saturation above 25 %) or the Flesh band (hue 5-30 degrees, saturation 20-60 %, value 45-85 %), no pixel on any element or text is Alarm Red `#E0102A`, and no pixel is a loop hue or Chemlight.
- **AC-CT-70** (review, UI, BLOCKING) [CT-39] GIVEN the field list of the layer (each drawn field with its source), THEN every field comes from `web.view`, `SaveInfo`, `save_status`, `restore_reason`, `campaign_info()` or the `SettingsStore`, and the reviewer records for each that no hallucination class (a) to (d) can falsify it; any field outside that list fails.

### G. Determinism, strings, accessibility, evidence and performance

- **AC-CT-71** (auto, Logic) [CT-27, EC-CT-65, EC-CT-66] GIVEN two runs of the same seeded campaign, one with the layer opened, picked, changed and closed at arbitrary frames (SOLO) and one without, THEN `state_hash` at every `T` is identical; the snapshot taken with the layer open contains no layer state; and a restore rebuilds the machine from CLOSED with `ChartPick = NONE`.
- **AC-CT-72** (auto, Logic) [CT-4, EC-CT-73, EC-CT-74] GIVEN OVERLAY with a fake `PauseGate`, THEN the panel, chart and plates behave identically to SOLO apart from kernel calls and `can_save_quit`; each seat's pick is local; only the pressing seat's pick enters its console command. *(provisional — VS-B reserved.)*
- **AC-CT-73** (lint CI job to be built, Config) [CT-28] GIVEN the `ct.*` table, THEN it is closed (every user-visible string is a key), no key is composed from fragments, every key has a fit class, and the instrument-face set is the closed list of AC-CT-32.
- **AC-CT-74** (review, UI) [CT-38] GIVEN the layer's layout and assets, THEN no stub art and no layout space exists for crew glyphs, chat, Codex annotations, post-loss UI, the beam picker, the Records page or the full settings UI; each reserved slot has its row in CT-38.
- **AC-CT-75** (manual, UI, BLOCKING) [CT-30, HR-14; qa-lead R17] GIVEN the whole layer, THEN every flow (including the chart snap and every destructive action) completes on a gamepad alone, recorded as a dated checklist with the device named. **AC-CT-75b** (lint CI job to be built, Config) [CT-30] every interactive control has a non-empty text label. **AC-CT-75c** (auto, Logic) [HR-12] text against background contrast is at least 4.5:1 and glyph contrast at least 3:1, computed from the palette for every text and glyph pair drawn, except the two signed measurements R-AD-4 (silhouette contrast) and R-AD-5 (ORPHAN), which the spike reports. **AC-CT-75d** (review, UI, BLOCKING) [HR-11, HR-2] focus is a shape (bracket and bezel thickening), never a Tungsten fill, and every plate is opaque Ground.
- **AC-CT-76** (auto, Logic) [CT-29, AA:562] GIVEN the audio double, THEN the layer requests only a UI-bus focus-move cue and a confirm cue, nothing for open, save or load, and no request depends on sanity stage or Attention.
- **AC-CT-77** (auto, Logic) [CT-43, EC-CT-58] GIVEN a `ClockDouble` delivering one 400 ms frame, THEN every layer accumulator advances by no more than `UI_DT_MAX_MS` (50 ms); a zero delta changes nothing.
- **AC-CT-78** (manual, UI + Visual, BLOCKING) [coding-standards UI and Visual rows; qa-lead R3, F-G] GIVEN the retained set `production/qa/evidence/chart-table/`, THEN a file listing shows (stage token `s0` = sanity stage 0, `s3` = forced stage 3, `boot` = boot-flow screen captured once): `panel_row1_s0|s3.png`, `panel_row3_s0|s3.png`, `panel_row6_s0|s3.png` (undimmed world visible beside the panel); `panel_failed_s0|s3.png`; `display_comfort_s0|s3.png`; `ui3_plate_downed_s0|s3.png`, `ui3_plate_retry_s0|s3.png`; `hold_bar_000.png`, `hold_bar_050.png`, `hold_bar_100.png`; `chart_<state>_s0|s3.png` for each of the eleven states of AC-CT-25; `chart_compact_row6_s0|s3.png`; `chart_row3_fractional_s0|s3.png` (426.67 x 266.67, no seam inconsistency); the pixel-grid crop (AC-CT-111) `panel_row3_grid_crop_8x.png` of the panel's left edge and one node; `main_menu_nocampaign_boot.png`, `main_menu_campaign_boot.png`, `main_menu_faulted_s3.png`; `notice_boot.png`; `ui5_boot.png`; `ui7_boot.png`, `ui8_boot.png`, `ui9_boot.png`.
- **AC-CT-79** (review, UI) [EC-CT-59, R17, C11] GIVEN the QA test plan, THEN it contains the note that a frozen hallucination visible beside the true panel is intended (`PAUSED_FROZEN`), so a tester does not file it as a menu fault, and an Edge Case check that the panel never dims.
- **AC-CT-80** (playtest, ADVISORY) [Player Fantasy validation 2, `CHART_TO_PRESS_S`; qa-lead R15] GIVEN at least three outside testers in a 30-plus-minute solo session, THEN the median `CHART_TO_PRESS_S` is recorded per tester (target 20 s; flag above 40 s as the friction signal; the remedy is never data on the chart) and each tester's free-text comment about the chart is recorded verbatim. The result feeds Open Questions and never gates a story.
- **AC-CT-81** (spike, ADVISORY until the numbers are set) [performance; qa-lead R14] GIVEN the layer open on the Steam Deck class target, WHEN the panel opens, the chart opens, and the bracket relocates, THEN the open to first presented frame is one frame, the layer's script time per frame is reported against `[PLACEHOLDER: 0.5 ms]` (an unset budget, not invented here), no hitch above one frame occurs on opening the chart (the layout file is parsed at boot), and the `SettingsStore` flush completes inside one frame. The numbers are retained with the capture.

### H. Added by the qa-lead review (AC-CT-82 onward)

- **AC-CT-82** (auto, Logic, BLOCKING) [CT-3, CT-13, EC-CT-50, `CHART_FADE_MS`] GIVEN a `ClockDouble` and the layer open at PANEL, WHEN Chart is accepted, THEN the panel is replaced by the chart in the same frame (a cut, zero intermediate frames); GIVEN reduce flashing on, THEN the replacement completes over the long end of the band (120 ms) and no frame shows a hard cut of the plate; GIVEN reduce flashing toggled on while a transition is mid-way, THEN the running transition completes as it was and later ones use the long end; the open and the close of the layer are one 80-120 ms transition each; reduced motion alone changes no fade.
- **AC-CT-83** (auto, Logic, BLOCKING) [CT-5, TX-10, EC-CT-12] GIVEN each boot-flow state (notice, main menu, UI-5, UI-8, UI-9), WHEN `chart_table` is pressed, THEN nothing changes (except aborting a running hold fill); WHEN `cancel` is pressed, THEN only UI-5 goes back (to the main menu, focus on New campaign) and the notice, main menu, UI-8 and UI-9 ignore it. GIVEN `chart_table` and `cancel` arriving in one frame in an in-session panel, THEN `chart_table` closes the whole layer and the `cancel` is consumed (one frame, one effect).
- **AC-CT-84** (auto + manual, UI, BLOCKING) [CT-6, CT-13, EC-CT-49] GIVEN key or pad navigation between two items, THEN the bracket reaches its new position over 100 ms with an ease-out, evidenced by a position table at t = 0, 50 and 100 ms (a mid-ease screenshot is not required); GIVEN reduced motion on, THEN the bracket relocates in one frame, fades are unchanged, and toggling reduced motion while the chart is open plays no animation for the change.
- **AC-CT-85** (auto + review, UI, BLOCKING) [CT-17] GIVEN the panel in every state, THEN it has no title element and no text equal to "Paused" or any state word other than `FAILED` (in the newest-record position); the plate is opaque Ground `#1C2128` with one Graphite-lit chamfered border and one rivet per corner; item labels are in the reading face and the playtime numerals in the instrument face.
- **AC-CT-86** (auto, Logic, BLOCKING) [CT-19, `SETTINGS_WRITE_DEBOUNCE_MS`] GIVEN a `StoreDouble` and a `ClockDouble`, WHEN a toggle changes, THEN one write occurs 500 ms later (not before; several changes inside the window produce one write containing the last values); the store is flushed at once on page close, on Save & Quit accept and on exit; a New campaign, Retire or Delete leaves the stored values and `notice_seen` unchanged; a change signal fires exactly once per changed value and a consumer double reading after the signal sees the new value.
- **AC-CT-87** (auto, Logic, BLOCKING) [CT-41(4)(5), EC-CT-17, EC-CT-23] GIVEN the layer open (SOLO) in PANEL and in CHART, WHEN the pointer clicks a screen position outside the panel, THEN no state, pick or kernel call changes; GIVEN a window-focus-lost then focus-gained event while idle, THEN the layer is still open on the same state and item, no kernel call was made, navigation state is cleared and the bracket is where it was.
- **AC-CT-88** (auto, Logic, BLOCKING) [CT-43, D.5] GIVEN the drift accumulator `phi` = 12.0, WHEN the layer closes and reopens after any interval, THEN `phi` is unchanged by the closed interval (no rewind, no jump); each hold accumulator is 0 on every new gesture and after release, abort or focus loss.
- **AC-CT-89** (auto, Logic, BLOCKING) [CT-21, TX-25, TX-32] GIVEN `save_status` FAILED, WHEN the layer closes and reopens, THEN `FAILED` is still shown (the layer never clears it) until Save/Load reports otherwise; GIVEN `UI3_PLATE` entered by `SL_QUIT_NO_WRITE_DOWNED`, THEN Retry is not offered; GIVEN `UI3_PLATE` entered from `SAVE_FAILED` with `can_save_quit` false, THEN Retry is disabled and no `save_and_quit()` call is made (Q-G ruling).
- **AC-CT-90** (auto + manual, UI, BLOCKING) [TX-3, CT-26, CT-23] GIVEN a store with `notice_seen`, WHEN the kernel enters FAULTED in release, THEN the main menu opens (no notice); with a campaign Continue is enabled and focused; with no campaign Continue is disabled and focus is on New campaign (CT-23 governs); no `request_pause` is made.
- **AC-CT-91** (auto, Logic, BLOCKING) [TX-28, TX-29, EC-CT-26, EC-CT-27, EC-CT-76] GIVEN OVERLAY and CHART open with a pick set, WHEN `view.transit_state` leaves IDLE, THEN the state becomes `CHART_VIEW_ONLY`, focus is on Back and the dashed line disappears; WHEN it returns to IDLE, THEN the state is `CHART`, the pick is NONE (arrival reset), focus is on HOME and no message appears in either direction. GIVEN SOLO during the Pipeline's `TRANSIT_HOLD` black, WHEN `chart_table` is pressed, THEN the panel appears over the black, `T` does not advance, and Resume returns to the hold.
- **AC-CT-92** (auto, Logic, BLOCKING) [TX-27, EC-CT-2, EC-CT-75] GIVEN the layer in each in-session state (PANEL, CHART, CHART_VIEW_ONLY, DISPLAY_COMFORT, UI3_PLATE, SAVE_FAILED, SAVE_QUIT_WAIT, HOLD_CONFIRM) in turn, WHEN a `request_restore` completes (each of the four reasons for at least two states), THEN the layer is CLOSED in the same frame with no transition frames, any hold fill is empty, and a following `chart_table` opens the PANEL with default focus Resume.
- **AC-CT-93** (auto, Logic, BLOCKING) [TX reachability] GIVEN the transition table as data, WHEN a walker explores from every state, THEN every state has a path to CLOSED or to an exit or boot state whose first edge is not a hold; `SAVE_QUIT_WAIT` and `LOAD_HOLD` are the only states with no player-caused exit, both bounded by Save/Load results; every `request_*` call appears on exactly the edges named in the table.
- **AC-CT-94** (auto, Logic, BLOCKING) [CT-9, EC-CT-4, EC-CT-7, EC-CT-8] GIVEN B `berthed` and A SEEN, WHEN accept lands on A, THEN `ChartPick = A`, A stays selectable, and no refusal or warning text exists on the layer; GIVEN every derelict ABSORBED or ORPHAN, THEN HOME is the only focusable node, the chart shows no text, no completion marker and no empty-state message, and a no-pick console press still opens the chart; GIVEN the pod at a derelict and the chart open, THEN HOME is the only focusable node, accept on HOME changes nothing and `ChartPick` stays NONE.
- **AC-CT-95** (auto with an `InteractionDouble`, Integration, BLOCKING) [CT-10(2), WM-21] GIVEN the pod at HOME with pick A, WHEN the home console `berth` press occurs, THEN the injected payload is `berth{derelict_id := A}`; `unberth` carries no id; with no pick it follows AC-CT-19's no-pick rule. *(provisional — ASK-CT-1.)*
- **AC-CT-96** (auto, Logic, BLOCKING) [D.2, CT-22] GIVEN `dt_us` 16,667, `CONFIRM_HOLD_S` 0.8 and `L_BAR_HUD` 64, THEN at frame 24 `fill` is 0.500 and `needle_px` 32; at the frame where `t_us` reaches the threshold `fill` is 1.0 and `needle_px` is 63 (capped at `L_BAR_HUD - 1`); the action fires once per gesture; reduced motion on does not stop the fill advancing.
- **AC-CT-97** (auto, Logic, BLOCKING) [D.3] GIVEN a node at exactly 45 degrees up-right of the focus, THEN it is chosen by "up" and not by "right"; a node at exactly 315 degrees by "right" and not "down"; a `delta` of 44.99999999 is rounded to 45.0000 and belongs to up; two equidistant candidates in one sector are broken by smaller |delta|, then by smaller ordinal; the half-plane fallback runs only when the primary sector is empty; `SNAP_SECTOR_DEG` 60 and 120 pass the same cases.
- **AC-CT-98** (auto, Logic, BLOCKING) [D.1b, CT-12, EC-CT-48] GIVEN pos (210, 60), THEN the HUD px are (452, 138) at row 1, (226, 69) at row 2 (`x0` 16, `y0` 9) and (279, 112) at row 3; glyph size is 24x24 at every `m`; a `pos` outside `0..287` or `0..161` fails the lint; WHEN a resize moves `m` from 1 to 0 with a pick on A and focus on B, THEN the layer switches to compact at the resize event (no animation), the pick is still A and focus is still B (by `derelict_id`).
- **AC-CT-99** (auto, Logic, BLOCKING) [D.8] GIVEN `ADV_EM` 0.5 `[UNV]`, `TEXT_EXPANSION_TEST` 1.3 and `em_px` 20, THEN `W_text("Display & Comfort")` = 221, `W_text("Save & Quit")` = 143, a 36-character label = 468 (fails `LABEL_1`, passes `LABEL_2` at content widths 336 and 348 screen px); at `em_px` 26 the 17-character label gives 288; a key needing three lines fails.
- **AC-CT-100** (auto, Logic, BLOCKING) [D.10] GIVEN `CHART_RING_STEPS` 3, `N_nodes` 3, one pod marker, one route, no seam, one frame, no text and three hints, THEN `D_L3` = 19.
- **AC-CT-101** (lint CI job to be built, Config) [D.6, `CHART_RING_STEPS`, `RING_STEP_MIN_CHANNEL`] GIVEN the sweep `CHART_RING_STEPS` 1 to 5 against `RING_STEP_MIN_CHANNEL` 2 to 6, THEN the lint passes iff every adjacent per-channel step is at least the minimum with Ground (28, 33, 40) and Void Steel (11, 16, 24) (for example, 5 steps with a minimum of 4 fails; 3 steps with a minimum of 3 passes, the smallest step being 5).
- **AC-CT-102** (lint CI job to be built + auto, Config, BLOCKING) [Tuning Knobs, coding standard] GIVEN the layer's sources, THEN no owned knob value is a literal in script (all read from the config resource); a value outside its safe range is clamped or rejected at load with a log line; derived knobs (`COMPACT_LAYOUT_MIN_CANVAS_W`, `MIN_H`, `S_MIN`) are computed from their sources and not independently settable; every ★ knob has a registry entry with the identical name.
- **AC-CT-103** (auto, Logic, BLOCKING) [EC-CT-22, EC-CT-68, EC-CT-38] GIVEN a gamepad-disconnect event with the layer open, THEN the layer stays open, glyphs switch to the last remaining device, no pause or close occurs; GIVEN `chart_table` pressed 10 times in 1 s, THEN every press is honoured (no cooldown; `CHART_REOPEN_MIN_MS` is not active); GIVEN an OS window-close request, THEN the layer makes no `save_and_quit()` or `request_shutdown` call of its own; any write is Save/Load's handler's (AC-SL-44).
- **AC-CT-104** (auto + integration, Integration, BLOCKING) [EC-CT-44, TX-4, TX-5, EC-CT-67] GIVEN `continue_campaign()` returns each failure result after the black frame, THEN the layer re-enables itself on the screen it came from with only Save/Load's message; GIVEN `new_game` fails, THEN the same screen returns with **no** new string (OQ-CT-17); GIVEN a second accept during `LOAD_HOLD`, THEN exactly one restore call is made.
- **AC-CT-105** (auto, Logic, BLOCKING) [CT-12, CT-13, CT-11; Q-A default] GIVEN `HOME_ID` for views whose HOME record id is 0, 7 and 1000, THEN the chart behaves the same (HOME is found by `kind`, never by a constant); the glyph asset follows `template_id`; rings 2 and 3 carry no node, tick or label; the ring-1 filament layer is the only layer with a changing offset; GIVEN a `template_id` with no glyph asset, THEN the plain wreck glyph is drawn and a log line is emitted (*default pending OQ-CT-20*), and the layout lint fails the build for it.
- **AC-CT-106** (playtest, ADVISORY) [EC-CT-71, Player Fantasy validation 3] GIVEN at least three outside testers, WHEN asked after a pause during a tense moment, THEN each answer to "did the pause feel safe or held" is recorded verbatim; no gate.
- **AC-CT-107** (auto + review, UI, BLOCKING) [CT-31..34, CT-37, CT-39; qa-lead R7] GIVEN a dump of every node and asset id in the layer-3 scenes, THEN each id is in a closed whitelist (plate, border, rivet, item, bracket, keycap, `glyph_node_*`, `glyph_pod`, filament sheets, ring bands, route quad, seam quad, fill bar, `IN TRANSIT` label, `ct.*` labels, switches); any id outside it fails, and a reviewer confirms none belongs to the Attention, sanity, threat, loot or inventory classes.
- **AC-CT-108** (auto, Logic, BLOCKING) [CT-1, CT-2, PF:223] GIVEN the scene tree, THEN the layer is a `CanvasLayer` in the root viewport and none of its nodes is a descendant of the world SubViewport or the Perception Filter's pass; the Filter receives no layer-3 texture.
- **AC-CT-109** (integration, Integration, BLOCKING) [Web Map & Transit, Kernel, Controller, Save/Load] Contract tests keep the doubles honest: the `ViewDouble` is schema-checked against WM-36 (a renamed field fails the build); `KernelDouble` calls match the real control-plane signatures; `SaveDouble` results enumerate the real Save/Load result enum and API signatures. Integration against the real modules: (a) a full sortie IDLE, LEG, ARRIVING, BERTHING, IDLE with the chart opened at each stage (TX-17, TX-28, TX-29; `IN TRANSIT` iff `pod_at == IN_FLIGHT`); (b) `request_pause` then `PAUSED` within one tick, Save & Quit disabled until then (EC-CT-30), and a same-boundary double press converges (ASK-CT-9); (c) opening the layer cancels a hold-mode channel and zeroes movement, and the closing `interact` produces no `pod_hop`; (d) Continue, New campaign, Retire and Delete against the real Save/Load on a temporary save directory. Evidence: an integration test run, or a documented playtest with the run log under `production/qa/evidence/chart-table/`.
- **AC-CT-110** (lint CI job to be built, Config) [CT-9, CT-35] GIVEN the `ct.*` table, THEN it contains no WM-9 reason id (`BUSY`, `IN_TRANSIT`, `OUT_OF_RANGE`, `NOT_LIT`, `NOT_DOCKED_HERE`, `TARGET_UNKNOWN`, `TARGET_ABSORBED`, `TARGET_HERE`, `TARGET_BERTHED`, `BERTH_PENDING`, `FUEL_LOW`, `ARRIVAL_BLOCKED`, `CAPACITY`, `POD_NOT_READY`; the full WM-9 order, AC-WM-10), no refusal sentence and no "paused"; the state word `IN TRANSIT` (with a space, CT-15) is the one allow-listed near match.
- **AC-CT-111** (spike, Visual, BLOCKING for the first UI story that ships layer 3) [CT-16, CT-28, CT-UI-8, D.1, D.1b, EC-CT-52; rulings §4] GIVEN canvas row 3 (1280x800 `expand`, `H` 3, canvas 426.67 x 266.67) on the real display path and the real faces, WHEN the panel, the spatial chart (bracket, bezel thickening, route quad, berth seam, node glyphs) and one reading-face message plate are rendered, THEN every bitmap-face glyph, node glyph, bezel, seam and route edge lies on a whole screen-pixel boundary, every 1-art-px stroke is exactly `H` screen px thick everywhere on the plate (no stroke one pixel thicker on one side), the reading-face text renders at its continuous scale with no clipping inside a plate sized to it, and the panel's left edge is at screen x 854; the captures `chart_row3_fractional_s0|s3.png`, `panel_row3_s0|s3.png` and an 8x pixel-grid crop of the panel's left edge and one node (pixel-grid crop, AC-CT-111) are retained in `production/qa/evidence/chart-table/`. Re-run at row 4 (213.33 x 133.33, `H` 6).
- **AC-CT-112** (lint CI job to be built + auto, Config, BLOCKING) [CT-44, G6, CT-14, ASK-CT-10] GIVEN the layout file and the `ct.*` table, THEN every compact-layout label is one whole `ct.compact.slot_<n>` key keyed to the authored layout slot; no compact label string, and no key it resolves through, varies with `status`, `template_id`, `kind`, depth ring or map distance (the lint runs the same label resolver over every status x template_id x kind x ring combination and fails on any pair of rows whose strings differ other than by slot); the string for a slot contains no name, dialect, template word or status word (checked against the closed `ct.*` and Web Map & Transit vocabulary lists); an UNSEEN record and a SEEN record in the same slot resolve to byte-identical strings; a record changing status leaves its key unchanged; and the `ct.*` table contains no other label key reachable from a node (the exception is not extensible).
- **AC-CT-113** (auto + review, Logic, BLOCKING) [CT-23, G5, SL UI-4] GIVEN a campaign whose newest record has its pod / home position at ring 2 after an earlier record at ring 3, THEN the Continue line shows `RING 2`; GIVEN any sequence of records, THEN `RING <n>` equals the ring of the newest record's pod / home position and no code path computes a maximum, minimum or running aggregate over records or sessions; the string table contains no "deepest", "furthest", "max" or "best" depth wording.

### Gaps / untestable as written

| # | Gap | Effect on the criteria | Owner |
|---|---|---|---|
| F-B | Provisional seams: ASK-CT-1 (console pick injection), ASK-CT-5 (`campaign_info()` verification field), ASK-CT-9 (kernel coalescing), ASK-CT-11 (boot-flow Quit), ASK-CT-2 (menu navigation and accept actions) | AC-CT-6, 19, 58, 59, 75, 83, 95 change text if a neighbour refuses; each carries its "(provisional)" mark | neighbours |
| F-C | Unset placeholders: `LAYER3_DRAW_CALL_BUDGET`, the per-frame script budget `[PLACEHOLDER: 0.5 ms]`; `ADV_EM` is **NOT SOURCEABLE** until the reading face is imported; sub-1080p type floors (OQ-CT-13) | AC-CT-45, 81 are ADVISORY; AC-CT-32 and 99 cannot run on real widths; AC-CT-32 at rows 3-6 waits for OQ-CT-13 | spike, art bible |
| F-D | `SAVE_QUIT_TIMEOUT_MS` is Save/Load's and unregistered | AC-CT-53 drives it through the `SaveDouble` and the `ClockDouble` | Save/Load |
| F-E | New CI lints to be built, each its own story: source scans (AC-CT-2, 3), fit-class measurement (AC-CT-28, 32), layout-JSON lint (AC-CT-33), string scans (AC-CT-68, 73, 110), asset-name scan (AC-CT-66), scene-id whitelist (AC-CT-107), knob/registry compare (AC-CT-102), ring sweep (AC-CT-101); the pixel-grid crop of AC-CT-111 is a manual spike capture, not a lint | lint criteria are not BLOCKING-verifiable until their job exists | lead-programmer, qa-lead |
| F-F | The engine reference has no coverage for `ConfigFile`, `Window` display-mode failure, key echo, focus-out, AccessKit (NOT SOURCEABLE, ADRs (a) to (f)) | AC-CT-13, 42, 43, 87 must be re-checked (and the two-scale layer measured on the real display path by the pixel-grid crop of AC-CT-111, since ADR candidate (b) is NOT SOURCEABLE) against the actual 4.7.2 behaviour when the ADRs land; engine behaviour is verified on the real engine, not a double | technical-director |
| Q-A | What the chart draws for a `template_id` with no glyph asset (only Foundry exists at MVP) | AC-CT-105 states a default (plain wreck glyph, log line, lint failure); OQ-CT-20 | agent, WM |
| Q-D | Whether `force_stage` is available before a session exists | boot-flow screens are captured once; OQ-CT-18 | Sanity |
| Q-I | Whether the pod can rest at a `berthed` derelict (D.4 would return `{HOME_ID}`) | no criterion; OQ-CT-21 | Web Map & Transit |
| R-AD-4, R-AD-5 | Graphite lit (about 2.8:1) and Graphite mid (about 1.2:1) fall below the glyph floor | AC-CT-75c excepts them by name; the spike reports | spike |

(F-A, the missing Tungsten, Violet, Flesh and Alarm Red hexes, is closed by inlining them in the fixtures above. F-G, the missing evidence file names, is closed in AC-CT-78. The qa-lead's questions Q-B, Q-C, Q-F and Q-G are answered as orchestrator rulings in AC-CT-7, 50 and 53; Q-H, the unsourced 3440x1440 canvas, is dropped in AC-CT-26.)

### Coverage matrix (rule, formula, transition and edge case to criteria)

| Item | Acceptance criteria |
|---|---|
| CT-1 | AC-CT-8, 108 |
| CT-2 | AC-CT-25, 46, 69, 108 |
| CT-3 | AC-CT-5, 7, 9, 82 |
| CT-4 | AC-CT-2, 5, 6, 51, 72 |
| CT-5 | AC-CT-9, 10, 11, 83 |
| CT-6 | AC-CT-14, 15, 84 |
| CT-7 | AC-CT-16, 20 |
| CT-8 | AC-CT-17, 18 |
| CT-9 | AC-CT-21, 22, 23, 24, 94 |
| CT-10 | AC-CT-19, 20, 95 |
| CT-11 | AC-CT-22, 25 |
| CT-12 | AC-CT-26, 33, 98, 105 |
| CT-13 | AC-CT-39, 40, 47, 82, 84 |
| CT-14 | AC-CT-23, 65, 112 |
| CT-15 | AC-CT-4, 18, 38, 91 |
| CT-16 | AC-CT-26 to 29, 111 |
| CT-17 | AC-CT-48, 55, 85 |
| CT-18, CT-19 | AC-CT-42, 43, 86 |
| CT-20 | AC-CT-44, 90 |
| CT-21 | AC-CT-50 to 54, 89 |
| CT-22 | AC-CT-10, 36, 37, 96 |
| CT-23 | AC-CT-56, 90, 113 |
| CT-24 | AC-CT-57 |
| CT-25 | AC-CT-58 |
| CT-26 | AC-CT-59, 90 |
| CT-27 | AC-CT-1, 3, 71 |
| CT-28 | AC-CT-32, 68, 73, 99, 111 |
| CT-29 | AC-CT-76 |
| CT-30 | AC-CT-75, 75b, 75c, 75d |
| CT-31 to CT-37 | AC-CT-61 to 67 (CT-31 to 61, CT-32 to 62, CT-33 to 63, CT-34 to 64, CT-35 to 65, CT-36 to 66, CT-37 to 67), AC-CT-107 |
| CT-38 | AC-CT-74 |
| CT-39 | AC-CT-70, 107 |
| CT-40 | AC-CT-41 |
| CT-41 | AC-CT-12, 13, 87 |
| CT-42 | AC-CT-60, 104 |
| CT-43 | AC-CT-36, 77, 88 |
| CT-44 | AC-CT-30, 31, 34, 112 |
| D.1 | AC-CT-27, 28, 111 |
| D.1b | AC-CT-26, 30, 33, 98, 111 |
| D.2 | AC-CT-36, 96 |
| D.3 | AC-CT-34, 97 |
| D.4 | AC-CT-21, 24 |
| D.5 | AC-CT-39, 88 |
| D.6 | AC-CT-40, 47, 101 |
| D.7 | AC-CT-55 |
| D.8 | AC-CT-32, 99 |
| D.9 | AC-CT-35 |
| D.10 | AC-CT-45, 81, 100 |
| TX-1, TX-2 | AC-CT-44 |
| TX-3 | AC-CT-59, 90 |
| TX-4 to TX-9, TX-11, TX-12 | AC-CT-56 to 58, 60 |
| TX-10 | AC-CT-10, 83 |
| TX-13, TX-14 | AC-CT-5 |
| TX-15, TX-19 | AC-CT-9 |
| TX-16, TX-17 | AC-CT-18 |
| TX-18 | AC-CT-19 |
| TX-20 | AC-CT-42 |
| TX-21 to TX-26, TX-31, TX-32 | AC-CT-50 to 54, 89 |
| TX-27 | AC-CT-17, 71, 92 |
| TX-28, TX-29 | AC-CT-91 |
| TX-30 | AC-CT-59 |
| TX closure | AC-CT-93 |
| Tuning knobs | AC-CT-102 |
| Cross-system integration | AC-CT-108, 109 |
| EC-CT-1 to 5, 6, 9 | AC-CT-16, 17, 19, 24 |
| EC-CT-7, 8 | AC-CT-94 |
| EC-CT-10 to 16, 18 to 21 | AC-CT-13, 10, 6, 14, 15, 11, 12, 35 |
| EC-CT-12 | AC-CT-83 |
| EC-CT-17, 23 | AC-CT-87 |
| EC-CT-22, 38, 68 | AC-CT-103 |
| EC-CT-24 to 28 | AC-CT-18, 38, 91 |
| EC-CT-29 to 35 | AC-CT-50 to 54, 89 |
| EC-CT-36, 37 | AC-CT-59, 90 |
| EC-CT-39 to 43 | AC-CT-57, 58 |
| EC-CT-44, 67 | AC-CT-60, 104 |
| EC-CT-45 to 50 | AC-CT-43, 42, 82, 84 |
| EC-CT-51 to 58 | AC-CT-30, 78, 26, 33, 104, 32, 55, 77, 111 |
| EC-CT-59 to 64 | AC-CT-79, 46, 70, 62, 63, 72 |
| EC-CT-65, 66 | AC-CT-71 |
| EC-CT-69, 70 | AC-CT-33, 56 |
| EC-CT-71 | AC-CT-106 |
| EC-CT-72 | AC-CT-57 |
| EC-CT-73 to 76 | AC-CT-72, 91, 92 |

## Open Questions

*Owner is `user` (a product decision nobody else may take), `agent` (a specialist, neighbour or lead can answer) or `GDD` (this document answers it in a later revision). Target is `playtest`, `next GDD` (a named neighbour) or `ADR`. Each row states the **default this GDD already assumes**, so nothing here blocks implementation. User-decision rows come first. The ASK-CT rows (provisional seams to neighbours) live in Cross-References and are not repeated; rows closed by a ruling are listed last so their ids are not reused.*

| # | Question (default assumed) | Owner | Target |
|---|---|---|---|
| OQ-CT-1 | Should the Display & Comfort page carry a master volume at MVP? *Default: none* (audio is Adaptive Audio's and Settings'; the page stays at three controls, CT-18) | user | next GDD (Settings & Input Rebinding, Alpha) |
| OQ-CT-4 | Is a Records page (a list of campaigns) and an un-retire screen wanted before Alpha? *Default: no.* At MVP "reversible" retire is reversible by file only (CT-24) | user | next GDD (Settings / Save-Load UI, Alpha) |
| OQ-CT-6 | Where does `world_seed` come from at New campaign, and does the menu ever show a seed field? *Default: Save/Load's boot rule supplies it; no seed field, no generation chooser* (CT-23) | user | next GDD (Save/Load, Procedural Generation) |
| OQ-CT-7 | What is the first-launch default of reduce flashing: the pipeline's hook default, or on until the player turns it off (the safe direction)? *Default: the pipeline's hook default (PR-13); the notice asks once* | user | playtest |
| OQ-CT-8 | What accessibility tier does the project commit to? `design/accessibility-requirements.md` does not exist; MVP is structural compliance only (CT-30) | user | next GDD (Accessibility Settings) |
| OQ-CT-2 | On a gamepad-only first launch, does dismissing the photosensitivity notice need anything beyond a single accept? *Default: no* (no hold, no countdown, CT-20) | agent (ux-designer, Controller) | ADR (menu input actions, ADR e) |
| OQ-CT-3 | Should a node be greyed when a hop to it would be refused with `BERTH_PENDING`? *Default: no.* The mirror does not carry it (it is not a `web.view` field) and the console refuses in words; a greyed node would leak a rule state onto the chart (CT-9) | agent (Web Map & Transit) | next GDD (WM revision) |
| OQ-CT-5 | Does the FAULTED return in release carry a one-line notice? *Default: none; this GDD invents no string* (CT-26) | agent (kernel, Save/Load) | next GDD |
| OQ-CT-9 | Does a label-less chart node need a hover affordance? *Default: none* (hover changes nothing on a node; the click's result is the route, CT-6) | agent (ux-designer, art-director) | playtest |
| OQ-CT-12 | Is `UI3_PLATE` as a full-bleed message plate acceptable to the art-director, given that the right-third panel is below the ART 7.1 45-75 character measure at four of six canvases (finding F-7)? *Default: yes; the panel carries labels and one line only, every paragraph is a message plate* (CT-21, CT-28) | agent (art-director) | next GDD (art bible) |
| OQ-CT-13 | What are the absolute type floors below 1080p (ASK-UP-2)? ART 7.1 states floors at 1080p and 1440p only. *Default: the lint uses the 1080p floor in screen px and flags rows 3-6* | agent (art-director) | next GDD (art bible) |
| OQ-CT-14 | Does the compact list page or scroll when a roster exceeds what fits? *Default: neither at MVP; the layout lint fails the build instead* (EC-CT-55, CT-44) | GDD | next GDD (Vertical Slice revision) |
| OQ-CT-15 | Does the 3 Hz ceiling (HR-13) cover a player-initiated toggle of a third-of-screen plate? If yes, the scoped guard `CHART_REOPEN_MIN_MS` (334 ms, derived from the ceiling) applies to the Chart item only, never to Esc, never to the console entry, with close never delayed. *Default: no guard* (ASK-CT-8, R-AD-7) | agent (art-director) | playtest |
| OQ-CT-16 | Does layer 2 (the HUD) draw beneath the open layer 3, which sets the real draw-call figure? *Default: yes, still drawn* (D.10, AC-CT-45) | agent (Pixel-Perfect Render Pipeline owner) | ADR (layer-3 ownership, ADR b) |
| OQ-CT-17 | Is a message string wanted when `new_game` fails after the black frame? *Default: none; the same screen returns* (EC-CT-44, AC-CT-104) | agent (Save/Load) | next GDD |
| OQ-CT-18 | Should `force_stage` work before a session exists, so boot-flow screens can be captured at forced stage 3? *Default: no; boot-flow screens are captured once plus a structural lint* (Acceptance Criteria, gate convention) | agent (Sanity owner) | next GDD |
| OQ-CT-19 | If the depth-step darkening is not visible on a real panel (flag F-6, AC-CT-47b), which remedy: a larger `RING_STEP_MIN_CHANNEL` (range 2-6) or a 1-px boundary line? Never a percentage restated in this GDD. *Default: neither until the spike measures* | agent (art-director) | playtest |
| OQ-CT-20 | What does the chart draw for a `template_id` with no glyph asset? *Default: the plain wreck glyph, a log line and a content-lint failure; only Foundry exists at MVP* (AC-CT-105) | agent (art-director, Web Map & Transit) | next GDD |
| OQ-CT-21 | Can the pod rest at a `berthed` derelict? D.4 would return `{HOME_ID}` from there. *Default: the mirror follows WM-4 as written* | agent (Web Map & Transit) | next GDD |
| OQ-CT-22 | **Risk K1 (VS-B).** The reserved crewmate beam-shape glyph on the chart is true (human channel) and, under OVERLAY, a zero-cost check against hallucination class (c). Is opening the chart price enough, or does the glyph need a condition? *Default: slot reserved, nothing designed* (CT-38) | agent (Co-op Session, Sanity) | next GDD (VS-B) |
| OQ-CT-23 | **Risk K2.** Solo `PAUSED_FROZEN` lets a player study a frozen frame for a palette leak at no time cost. Does every class-(a) to (d) leak have a component visible in a single held frame, so the freeze is never the only way to catch one nor hides one? *Default: accepted (pause is an accessibility good and a Project Zomboid precedent). **Routing to Sanity is binding:** every leak must have a single-frame-visible component, and no leak may be designed to be catchable only under the freeze. Stays open until Sanity answers* | agent (Sanity owner) | next GDD |
| OQ-CT-24 | **R-AD-4 and R-AD-5.** Do UNSEEN / SEEN / BOARDED silhouettes (about 2.8:1) and ORPHAN (about 1.2:1) need a 1-px outline inside the 24x24 glyph assets (Hull Grey for the first, dashed Graphite lit for the second)? *Default: measure first; add the outline only if the spike reads under 3:1 or the ORPHAN is not perceptible* | agent (art-director, Web Map & Transit) | next GDD (spike) |
| OQ-CT-25 | **R-AD-6 (VS-B).** In OVERLAY the opaque panel covers the hand rail while the sim runs. Is that acceptable? *Default: reserved, not an MVP defect* | agent (HUD & Suit Layer, Co-op Session) | next GDD (VS-B) |
| OQ-CT-26 | **R-AD-12.** Add `on` and `off` tokens to the ART 8.5 `state` list for the toggle asset? *Default: use `ui_toggle_switch` with provisional tokens until the art bible adds them* | agent (art-director) | next GDD (art bible) |
| OQ-CT-27 | **Pause re-toggle interval (HUD OQ-HS-62, HS-23).** The HUD asks for a minimum interval between pause toggles (proposed at least 1 s on process time) so rapid toggling cannot exceed `FLASH_MAX_PER_S` through warning-mark leading edges; the HUD does not debounce the pause. This contradicts EC-CT-68 ("there is no cooldown on `chart_table`"), CT-3's provisional (a rate limit applies "never to Esc ... and close is never delayed") and CT-4's double-press handling. *Default: EC-CT-68 stands (no pause cooldown); the HUD's held-ON mark under the panel (CT-3) is carried. Options: a pause cooldown here, an edge suppression on the HUD side, or accept the risk* | user (with HUD & Suit Layer) | next consistency pass |

**Closed by a ruling (ids not reused).** OQ-CT-10 (is the route or a second mark the pick?) is closed by CT-7 and checkpoint ruling Q13: the dashed Tungsten route is the pick mark and always means "the target the next console press carries". OQ-CT-11 (layout coordinate space) is closed by checkpoint ruling Q7: art-px frame `CHART_FRAME`, integer chart multiplier `m`, normalised coordinates declined (CT-12). The art-director's OQ-AD-1 (does activation or focus alone pick?) is closed by CT-7 and Visual/Audio: accept picks, focus never does.

**Routed to neighbours (not questions for this GDD).** Conflict and clarification rows C1 to C19 and asks ASK-CT-1 to ASK-CT-12 (Cross-References) are carried by their owners: Boarding (C1), the Pipeline owner (C2, C14, C18, C16 via the art bible, ASK-CT-12), the kernel (C3, ASK-CT-9, ASK-CT-11), `/map-systems` (C5), the registry merge stage (C6), Save/Load (C7, ASK-CT-5, ASK-CT-7), Sanity (C8), Web Map & Transit (C12, C13, ASK-CT-3, ASK-CT-4, ASK-CT-10), the Controller (C15, ASK-CT-1, ASK-CT-2), the HUD & Suit Layer (C19) and the art-bible owner (C16, C17).
