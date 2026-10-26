# Art Bible: The Synaptic Sea

> **Status**: Approved
> **Owned By**: art-director
> **Last Updated**: 2026-10-25
> **Art Director Sign-Off (AD-ART-BIBLE)**: APPROVED 2026-10-25 (after one CONCERNS round; 29 in-place revisions, all marked *Amended per …*)

`/art-bible` writes these nine sections in this order and replaces each
section's `[To be designed]` line as that section is approved. A section still
holding its placeholder is incomplete; a later run fills only those.

## 1. Visual Identity Statement

**Direction: *What You Light, You Can Read***

### The Rule

> **Light is a resource you spend. Inside it, every machine states its condition. Outside it, the dark is honest.**

**How to use it.** When two visual choices both seem fine, ask three questions in order. The first answer that separates them decides the choice.
1. Does one of them make the player *spend* something (light, power, attention) to learn what they see?
2. Does one of them put the information *in the world* (a moving part, a medium, a trail) instead of on top of it?
3. If the player is lied to here, will the lie leak the sane palette?

**Why this works as a tie-breaker.** It is one sentence with three clauses that map to three kinds of decision: lighting, state display, and truth. Almost every art question lands in one of those. It also keeps the art workload honest. The main cost sits in the 2D lighting pipeline rather than in sprite-state count, so the rule favors cheap, systemic answers (a light, a hue, a trail) over expensive bespoke ones (a new animation per state).

**Alternatives considered.**
- *"The Dark Is the Default"* alone. It gives strong mood but nothing to say about how a lit machine should look, so it would leave the Satisfactory-style legibility to chance.
- *"Every Pipe Is a Sentence"* alone. It gives strong legibility but treats light as free, which would erase Pillar 3 (The Web Watches).
- The hybrid keeps both. Legibility is the reward and darkness is the price.

### Principle 1: Darkness Is Honest; Light Is a Decision
**Serves Pillar 1, The World Never Lies** (Pillar 3 benefits secondarily).

Rooms are not fogged, they are *unlit*. Geometry always exists, and any source reveals it exactly as it is. The visible-world edge is the real edge of a real light, never a mask. The physical radius your light actually reveals is a fixed anchor and never lies. Even at low sanity, the hallucination layer may add things but may not move that boundary. Hard-shadow occlusion is guaranteed for the **six shadow-casting lights** of 8.11 flag 2 — the four player beams, the alarm beacon and one sixth slot (the nearest deployable in view, or a single fixture where none is) — and it is for those six that this principle is never cut; other fixtures are soft fills that cast no shadow. *(Amended per 8.11 flag 2)*

**Rationale.** Pillar 1 says the physical world is ground truth and only instruments and perception can be wrong. A vignette or fog-of-war mask is an interface lying about the world, and it would give the sanity system nothing to betray. True occlusion gives the player a reliable baseline. A hallucinated shadow with no occluder is then frightening *because* occlusion is normally trustworthy. It also makes light a decision, since each lamp is a visible cost with a visible reach.

**Design test.** When a vignette or fog-of-war mask competes with true 2D occlusion from light sources with hard falloff, choose the true occlusion.

**Alternative rejected.** A soft ambient-reveal radius around the player. It is cheaper to build and reads well, but it makes the player the light source for free, which erases the spend-light economy.

### Principle 2: Inside the Light, State Lives in the Sprite, Not the Icon
**Serves Pillar 2, Dread Before Chaos** (and Pillar 1, since the physical world carries the truth).

Every system has a visible moving part and a visible medium (a rotor, a fluid, a filament). Failure renders as a trail in the loop's own hue: puddle, frost, scorch, mist, drip. A trail is also a tell. Every breach is preceded by a perceivable escalation, and that escalation is drawn *in the world*, so a careful player can read it. Hue coding is always redundant with shape and pattern (flow chevrons, pipe diameter) so it stays readable for colorblind players.

**Rationale.** Pillar 2 needs tells a careful player could have caught. A banner is read once and dismissed, but a frosting elbow can be seen to worsen from across the room. This is the Satisfactory lesson: watching a machine work is its own reward, and failure can be diagnosed by following the eye along the pipe. Resource loops keep their hue everywhere they appear (working proposal: O₂ cyan, power amber, coolant ice-blue, fuel orange-red, integrity neutral grey), so a hue is a vocabulary the player learns once.

**Design test.** When a status LED or a "COOLANT LOW" banner competes with a turning rotor with visible fluid or a frosted elbow with a drip, choose the rotor and the elbow.

**Alternatives rejected.**
- A world-space icon layer on every system. It is cheap, but it reads as an interface and it gives the hallucination system an easy, un-physical target.
- Heavy per-state sprite sets. They are too expensive for a solo developer, so state should come from shared hue, trail and motion systems instead.

### Principle 3: Emission Is Exposure
**Serves Pillar 3, The Web Watches.**

Every running loop is visible as emission along its pipe in its own hue and casts light at its machinery nodes (8.3). With power off, the ship is lit only by those nodes and by the crew's own beams; every loop you run is a lantern the web can see. *(Amended per 8.11 flag 8)* The web itself has no hue of its own and appears only in your light, as oily iridescence under direct illumination. Deeper rings do not change hue. They lower ambient light and raise the iridescence.

**Rationale.** Pillar 3 says Attention is shown in the world, never as a hidden meter, and that restraint is a real strategy. Making light, power use and visibility the *same thing visually* means going dark is a legible tactical choice, not a menu setting. Tying the web's appearance to the player's own light also makes the web's presence feel like a consequence of the player's choices, which supports the "dread is feeling it come" hinge. Keeping hue constant across depth keeps the loop vocabulary from Principle 2 reliable at every ring. Depth is expressed only through ambient and iridescence.

**Design test.** When an on-screen attention meter competes with more filaments resolving in the player's light cone as Attention rises, choose the filaments.

**Alternative rejected.** Shifting the palette per depth ring. It gives strong depth identity, but it would break the flow-coded hue vocabulary, and it would compete with the violet and flesh-tone rule below.

### Corollary: When the Game Lies, the Lie Leaks (serves Pillar 4)

This is a binding constraint on Pillar 4's hallucination classes, not a fourth principle.

- **Flesh tones and violet exist nowhere in the sane world.** When either appears, it is the player's sanity and never the location. Every hallucination leaks them somewhere: a fringe, a flicker, a tint at the edge of the light. Subtle is allowed; absent is not.
- **Drift order.** Instruments first (needles disagree with the body of the ship), then the light lies (the cone seems to reach further than it does, shadows with no occluder, temperature shifts), then loop hues bleed violet at pipe edges and flesh tones enter.
- **Anchors that never shift.** Character sprites, hand tools, the HUD frame, and the true light radius. A crewmate's own light beam (colour and shape) also stays truthful, per Pillar 1's human channel.

**Design test.** A seamless hallucination vs. one that leaks the sane palette → the leak.

### Constraint Notes

- **Art style.** HD pixel art at 3/4 oblique top-down, with Godot 4.7.2 2D lighting. Readability under light comes from silhouette, hue and motion, not from texture noise. This keeps sprites cheap to author and keeps the lighting doing the visual work.
- **Pipeline consequence.** The lighting-budget spike (a lighting test on D3D12) is the *first* spike, because all three principles depend on 2D lights, occlusion and emission being affordable.
- **Open for the technical artist.** Whether true occlusion at hard falloff, many emissive loops, and a per-client hallucination filter all fit the lighting budget is unmeasured. If they don't, the first thing to cut is the number of simultaneous emitters, not the occlusion rule in Principle 1 as scoped to its six shadow-casting lights (8.11 flag 2). *(Amended per 8.11 flag 2)*

### Reference Touchstones

- **Project Zomboid.** Lived-in detail and a dark world that you move through at a patient pace.
- **Satisfactory.** Legible machinery and flows you can read at a glance.
- **Stellaris.** Dread delivered by systems, not cutscenes.
- **Barotrauma.** A 2D hostile vessel, with failure you can see happening.
- **Beyond the Aquila Rift.** A kind-looking lie, where the wrongness is in what leaks through.
- **Pandorum.** Waking into wrongness aboard a derelict.

### Checklist

For any new asset or visual feature, answer these before approval.
1. Does it make the player spend light to see it (Principle 1)?
2. Does its state live in a moving part or a trail, not an icon (Principle 2)?
3. If it glows, is that glow exposure, and does it carry the right hue (Principle 3)?
4. Does it avoid violet and flesh tones in the sane world (Corollary)?
5. If it is an anchor (character sprite, hand tool, HUD frame, light radius), does it stay unshifted under hallucination (Corollary)?

## 2. Mood & Atmosphere

### 2.0 How this section is read

The art-bible placeholder asks for "time of day". This game has none: the ship is inside a web with no star. The three axes that replace it are **power state** (what the crew is running), **depth ring** (ambient floor and iridescence) and **attention phase** (quiet → tell → breach). Every mood below is defined on those axes.

**Cost rule (solo developer).** Every state must be reachable with the *same* sprite set and a small lighting-parameter change: ambient level, number/temperature/direction of active sources, contrast (falloff hardness), one or two overlays, and which loop media are animating. No state may demand its own tileset or sprite variants. If a mood cannot be reached by lighting plus one carried element, the mood is wrong, not the budget. This is also a fidelity rule: the player should learn to read the ship's state from light alone — the one-line visual rule made literal.

**Reference frame for ambient.** "Ambient" is the scene's base light with no sources active, as a percentage of white. The home-ship's powered interior sits at about 10 %; true black is 0 %. Exact engine values belong to the lighting spike and Section 8; the *ordering* here is the contract.

**States.** The brief's list is kept with two changes: **the Tell** is added as its own state between boarding and breach (Pillar 2 is literally about this window and it has a distinct look; undefined, it would be implemented as "breach, but earlier"), and **crew loss** is split into a short transition beat plus **re-wake**, which is a variant of the cryo-bay wake rather than a new state (same room, different set dressing — a new state would cost a new location).

### 2.1 Discriminator table

| State | Ambient | Key light | Direction | Temperature | Contrast | Dominant hue | What moves | Carried element |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Cryo-bay wake / re-wake | 3 % | one dim tungsten emergency lamp, guttering | single overhead, hung low → short hard shadows down-screen (the sixth shadow slot, 8.11 flag 2) | cold base under one dim warm fixture; the pod's brighter warmth, briefly | high, hard | steel + dim tungsten; ice-blue only in the frost trail | condensation, one trail | frost trail from the one leak *(Amended per AD-ART-BIBLE review 2026-10-25)* |
| Home-ship stable | 10 % | many tungsten fixtures | overhead, multiple → soft, near-directionless | warm | low, soft | tungsten + all loop hues | every loop's medium | sight-glass fluid, turning rotor |
| Home-ship dark — conserved | 2 % | beams + the loops you kept | from the hand, forward in facing | neutral-cold, warm islands | high | the one loop you kept | only what you chose | a single running loop as a lantern |
| Home-ship dark — lost | 2 % | beams + alarm sweep | hand forward + rotating beacon → sweeping shadows | cold with red | high, pulsing | alarm red over trails | failure trails propagating | trail spreading along a seam |
| Pod transit | 0–1 % | pod running lights (tungsten) | outward from the pod | warm point in a black field | highest | steel + iridescence | filaments crossing the cone | the wreck's one residual light |
| Boarding (deliberate) | 1–3 % by ring | your beam only | from the hand, forward | your beam is the only warmth | high, hard | tungsten in a steel world | dust in the beam | dead gauges and dried trails |
| The Tell | as boarding | as boarding + iridescence rising | as boarding | as boarding | rising at the cone edge | iridescence at the rim | filaments, dust direction, real needle drift | filaments resolving at the cone edge |
| Breach (frantic) | as boarding | beams + alarm sweep + venting loops | hand + beacon + point sources from vents | cold with red and hot media | highest, moving | alarm red + hot loop hues | trails, sweep, venting | a trail you must outrun |
| Low-sanity overlay | unchanged | unchanged (may *appear* to lie) | unchanged | *appears* to drift warm | unchanged | violet / flesh at edges | things that should not | the leak at the edge of the light |
| Crew loss (transition) | collapsing to 0 % | sources go out one by one | — | warm → cold | — | steel | nothing, then condensation | the last running loop going dark |
| Menus & web map | flat, screen-space | none (unlit UI) | none | neutral graphite, tungsten text | medium | graphite + tungsten | filaments in the current ring, slowly | the map darkening toward centre |

Distinctness check: no two rows share both key light *and* dominant hue. Conserved and lost dark share ambient and key light and are separated by red and by motion (chosen vs uncontrolled); that is deliberate — the player should feel how close restraint and failure sit.

### 2.2 Cryo-bay wake (and re-wake)

**Emotional target.** *Waking into wrongness, gently.* Not fear — disorientation and a tenderness toward one's own body. The player should want to touch things to check they are real. (Pandorum's first ten minutes, not its last.)

**Lighting.** Cold. A single **dim Tungsten** emergency fixture hanging low — all human light is Tungsten (4.2), so the room's cold comes from the 3 % ambient on the Graphite base and from the frost trail, not from the lamp — **guttering**: irregular dips of 0.3–0.5 s no more often than every 10–20 s, never periodic and never the deployable's flicker (3.2, 5.5); shadows fall short and hard down-screen because this lamp holds the sixth shadow slot of 8.11 flag 2 (it is the only fixture in this state). Ambient near 3 %, so the far wall is suggestion only. Hard falloff: the pod row goes from legible to black in a few tiles. The one *bright* warm point — the lamp is dim — is the interior light of the pod you came out of, which dies within the first minute — the game's first lesson that warmth is spent, not owned. *(Amended per AD-ART-BIBLE review 2026-10-25 and 8.11 flag 2: an ice-blue lamp would spend a hue 4.0 keeps scarce, read as coolant emission beside the coolant trail it is meant to lead the eye to, and need a new 4.1/4.2/4.7 entry; "flicker" is reserved for deployables.)*

**Atmospheric descriptors.** Condensed. Thawing. Clinical. Hushed. Tilted (one failed loop has left the room slightly wrong — a frost line across the floor, a mist pooling in a corner).

**Energy.** Very low. The slowest the game ever is. Nothing moves faster than a drip.

**Carried element.** The frost trail from the single failing loop, running from the leak to the far wall. It is the tutorial: the eye follows it and finds the source. In the sane world it is ice-blue (coolant) or cyan mist (O₂), whichever loop the onboarding uses.

**Re-wake variant (failure contract).** Same room, same lamp, same ambient. The deltas are set dressing: more pods stand open and dark; the frost line runs a different path; recovered logs are already on the table. The returning player must feel "I know this room" before "something has changed", so lighting is identical and only objects differ. *Alternative considered*: a harsher, redder re-wake to mark failure. Rejected — it tells the player failure is punishment, and the contract (Pillar 5) says failure is education.

### 2.3 Home-ship stable (powered)

**Emotional target.** *Earned calm.* The ship breathes and you can see it breathing. The only state in which warmth dominates and the only state the player should want to linger in. Everything else in the game is measured by its distance from this.

**Lighting.** Warm. Many small overhead tungsten fixtures rather than one big light, so the ship reads domestic, not industrial; fixtures are soft, nearly directionless fills and cast no shadows — only the crew's own beams do here (8.11 flag 2) — so what shadow there is is graphite, not black. Ambient at the game's maximum (about 10 %). Low contrast. Every running loop is visible as emission along its pipe in its own hue and casts light at its machinery nodes (8.3), so the ship is lit by its lamps and *marked out* by its systems — *Emission is exposure* turned into comfort: the price of this cosiness is that the web can see every lantern (Pillar 3). *(Amended per 8.11 flags 2 and 8)*

**Atmospheric descriptors.** Humming. Lived-in. Layered (grafts of different dialects under one warm light). Steady. Occupied.

**Energy.** Low and rhythmic. Rotors turn, sight-glasses pulse, a bellows rises and falls. Continuous slow motion everywhere and nothing sudden.

**Carried element.** Moving loop media visible through a sight-glass — the fluid you can see is the proof the pipe is not lying (Pillar 1; principle 2).

*Alternative considered*: a clean cool "spaceship white" stable state for maximum contrast with the dark. Rejected — the cold base is the default, and a white interior would read as safe-by-location when safety here is always something the crew is doing.

### 2.4 Home-ship dark — conserved and lost

Listed as one state in the brief; kept together because they share ambient (about 2 %) and key light (hand beams plus whatever loops run), but they are two moods and the difference must be legible in one glance.

**Conserved (chosen).** *Emotional target: disciplined quiet — a held breath you chose to hold.* Lighting: no fixtures; light comes from the hand, forward, plus the one or two loops the crew left running — emission along their pipes, light only at their machinery nodes (8.3) — isolated lanterns in a black hull. *(Amended per 8.11 flag 8)* Hard falloff. Nothing is red. Nothing moves except what the crew chose. Descriptors: hushed, disciplined, lantern-lit, listening. Energy: low and deliberate. Carried element: a single running scrubber or cell glowing in its hue with the whole ship black around it — the image that says "we are small on purpose" (Pillar 3: restraint is a strategy).

**Lost (failure).** *Emotional target: exposed.* The same darkness, but happening to you. Alarm red sweeps from a rotating beacon (2–3 s period; never a strobe — Section 4), throwing shadows that move; failure trails propagate along seams in their loop hues — frost crawling, mist spreading, scorch blooming. Descriptors: cold-red, spreading, leaking, venting. Energy: medium and rising. Carried element: a failure trail spreading along a weld seam while you watch.

Two cues separate them: red (present only in lost) and motion (chosen vs uncontrolled). Both are redundant, so neither colorblindness nor a still frame breaks the read.

### 2.5 Pod transit

**Emotional target.** *Smallness.* The only exterior view in the game and the only time the web is seen as a thing rather than a texture on a wall. Awe with a cold floor under it.

**Lighting.** Ambient at or near 0 %: the void is black, not navy. The pod's running lights — Tungsten fixture, pointing outward — are the sole source; filaments appear only where that light touches them and show iridescence there and nowhere else. The game's highest static contrast. The loneliness comes from scale, not from a colder light: one small tungsten pool in a field of nothing.

**Atmospheric descriptors.** Suspended. Silent. Vast. Threaded (filaments crossing the cone like cables in headlights). Watched.

**Energy.** Low, with one spike — the derelict resolving out of the dark.

**Carried element.** The target wreck's *one visible unknown*, rendered as a **residual human light** — a cell still running, a fixture still lit on a dead ship — in Tungsten or a loop hue. It is the only lit thing in the exterior the player did not pay to light, which should feel slightly wrong because it is: somebody else's lantern, and the web can see it too. (The web itself is never visible except in the player's own light — principle 3 — so the unknown is never an iridescent gleam.)

*Alternative considered*: a dim nebula backdrop for visual interest. Rejected — it breaks *darkness is honest* and makes the web a place instead of a presence.

### 2.6 Boarding rhythm (deliberate)

**Emotional target.** *Forensic curiosity under a held breath.* Reading a dead ship's story by torchlight. Zomboid's "every door is a decision", delivered by a light cone.

**Lighting.** The beam is the only light, from the hand, forward in the facing direction. Ambient is the depth ring's floor (3 % outer, 1 % or less inner). Hard-edged cone with soft interior falloff; everything outside it is black and *real* — walk into it and it is there (principle 1). The beam is the only warmth in frame: the player carries the game's only comfort into a cold place.

**Atmospheric descriptors.** Dusty. Still. Cold-dead. Archaeological (the dead ship's own old failure trails, dried and frosted). Interrupted (a meal, a repair, a hatch half-cut).

**Energy.** Low and deliberate. Dust drifts in the beam; nothing else moves unless the player moves it.

**Carried element.** Dead gauges and dried trails: a scorch with no fire, a frost fan with no leak, a puddle long evaporated to a stain. They teach the loop grammar on a ship that cannot lie because it has no power to lie with.

### 2.7 The Tell (added state)

**Why added.** Pillar 2 owns a 10–30-second window in which a careful player could have perceived what is coming. Boarding and breach each have a mood; the hinge between them is the game's signature image — dread *arriving* — and needs its own.

**Emotional target.** *It noticed.* The small-animal feeling of being looked at. Not yet fear — attention.

**Lighting.** Identical to boarding: ambient, beam radius, beam temperature and direction do *not* change (the beam is a truth anchor, and attention is not sanity). What changes is at the **edge** of the cone: filaments that were texture resolve into strands; the web's iridescence brightens where the beam grazes it; dust in the beam changes direction as if something displaced the air. Contrast rises because the rim gets busier while the interior stays the same.

**Atmospheric descriptors.** Prickling. Resolving. Leaning-in. Creaking (visually: hull-stress hairlines; a needle that drifts on a *real* gauge).

**Energy.** Rising across the window — few filaments, then many; drift, then movement.

**Carried element.** Filaments resolving at the cone edge: *Emission is exposure* and Pillar 3 in one sprite — the more you light, the more of it you see looking back.

Boundary: this defines what the player *sees* as attention rises. What raises attention, the thresholds and the audio belong to the game-designer and audio-director.

### 2.8 Breach rhythm (frantic)

**Emotional target.** *Controlled panic.* Barotrauma-paced. The player must be frightened and still able to read a pipe — the hardest test of *inside the light, every pipe is a sentence*.

**Lighting.** Beams, plus Alarm Red sweeping from a rotating beacon, plus the hot light of loops venting — fuel ember, power arcing, cyan mist from a cut O₂ line — as point sources where they burst. Contrast is the game's highest and it moves. Ambient is unchanged from boarding; the chaos is entirely in the sources.

**Readability rule.** Alarm Red is a rotating sweep with a 2–3 s period and never a full-frame flash; failure trails render at full loop-hue saturation so they stay readable through red light; the beam's cone stays the brightest thing in frame so the player always knows where they are looking. The frantic feeling comes from *more things moving*, not from the screen fighting the player.

**Atmospheric descriptors.** Venting. Sweeping. Hot-and-cold (fuel ember beside coolant frost). Buckling. Crowded (everything moving at once).

**Energy.** High. Everything that can move does.

**Carried element.** A failure trail propagating along a graph edge faster than you can walk — the image that teaches "retreat to the pod".

*Alternative considered*: a red full-screen vignette on breach. Rejected on three counts — it is a status LED in disguise (principle 2), a photosensitivity risk, and it dilutes the one colour that must stay rare.

### 2.9 Low-sanity overlay (modifier on any state)

**Emotional target.** *Doubt, not horror.* The player should feel "that's not right" before "that's a monster". The overlay never adds a mood of its own; it corrupts the mood the player is in.

**Lighting (apparent, never actual).** The true scene is unchanged — ambient, sources and the real beam radius are anchors. Stage 1: instruments disagree (a needle, a readout) and each lying instrument carries a violet gleam on its glass, because every lie must leak (Pillar 1). Stage 2: the light *appears* to lie — the cone seems to reach further than the geometry it reveals, shadows appear with no occluder, apparent temperature drifts warm; the leak is a violet fringe at the cone rim and violet-black shadows where the sane world's are steel-black. Stage 3: loop hues bleed violet at pipe silhouettes and flesh warmth enters surfaces — steel takes a skin tone, pipes read as veins. The ship-as-body fantasy is literalised against the player.

**Atmospheric descriptors.** Wrong. Warm where it should not be. Fringed. Over-reaching (light that promises more than it delivers). Veined.

**Energy.** Unchanged from the host state — the overlay must not announce itself through tempo, or the player reads sanity off the screen's energy instead of off the world.

**Carried element.** The leak at the edge of the light. The player learns to look at edges — cone rim, pipe silhouette, gauge glass — which is where the truth test lives.

**Seen from outside.** A crewmate below threshold, as seen by a saner viewer, carries a violet fringe around their sprite outline and along their beam's edge. Sprite pixels, beam colour and beam shape are untouched (human channel). Specified in 3.2 and 4.4.

### 2.10 Crew loss (transition beat)

**Emotional target.** *The last light going out.* Grief as a lighting event, lasting seconds.

**Lighting.** Sources go dark one by one, warmest first — fixtures, then loops, then beams — until ambient collapses to 0 %. No red; the alarm has already lost. No violet; this is real.

**Descriptors.** Extinguishing. Settling. Honest.

**Energy.** Falling to zero.

**Carried element.** The last running loop dimming to black, then a cut to the cryo-bay lamp guttering on (2.2, re-wake). The transition makes the failure contract visible: the dark is honest, and the next frame is a room you know. *(Amended per AD-ART-BIBLE review 2026-10-25)*

*Alternative considered*: a death screen with text. Rejected — authored UI at the moment of loss contradicts "no cutscenes", and the lighting already says it.

### 2.11 Menus & web map

**Emotional target.** *The chart table.* Sober, competent, outside the fiction's danger but inside its mood. The player is a navigator planning a sortie.

**Lighting.** None — UI is flat and unlit (4.6). Graphite ground, paper-white text, tungsten accents. The web map draws filaments as neutral grey lines whose density rises toward the centre; depth rings are concentric steps of darker ground, so the map encodes "deeper = darker" the same way the world does. The home-ship is a tungsten point; known wrecks are grey silhouettes. The map never shows iridescence or violet; it is a human instrument.

**Descriptors.** Charted. Quiet. Inward-pulling (composition should pull the eye to the dark centre).

**Energy.** Very low. The only motion is a slow drift of filaments in the current ring, so the map remembers the web is alive.

**Carried element.** Concentric darkening toward the centre — the horizon the game never lets you reach (Pillar 5).

## 3. Shape Language

### 3.0 Governing principle

**Humans build in rectangles and rings; the web grows in branches and never repeats.** Everything in this section follows from that sentence. It serves the one-line visual rule — *inside your light, every pipe is a sentence* requires human things to be grammatical (modular, repeating, orthogonal) so that a break in grammar is readable — and it serves Pillar 1 (The World Never Lies): the physical world is ground truth, so its shapes must be unambiguous enough to trust when a colour cannot be.

Three Gestalt tools do most of the work: **figure–ground** (the beam cuts figure out of black ground — the lighting *is* the composition), **similarity** (every object from one manufacturer shares a dialect, so a graft from another reads as foreign), and **continuity** (pipes are continuous lines the eye follows from symptom to source). Where a rule below seems arbitrary it is usually serving one of these.

### 3.1 Readability contract (Pillar 1)

Every character and hero object must be identifiable at **25 % scale** — a thumbnail with no outline and no colour, silhouette alone. Every loop must be identifiable by **pipe section alone** (diameter, fitting, pattern) with hue removed. Both tests run on every new asset *before* it is coloured, because colour is the channel the game is allowed to corrupt.

### 3.2 Character silhouette philosophy (Pillars 1 and 4)

**Crewmates: compact biped, circle on rectangle.** In 3/4 oblique the camera sees head, shoulders and hands; legs are short and low-information. The crewmate silhouette is a **round helmet** over a **square life-support pack**, hands and tool breaking the outline to the side. Symmetrical, upright, closed shapes. This is the only place in the world where a perfect circle sits on a perfect rectangle: the human shape is the human construction grammar wearing itself. It reads *competent and fragile* — a tool-using thing in a shell.

**Crewmate identity is carried by beam shape, not sprite hue.** Pillar 1's human channel says a crewmate's own beam — colour *and shape* — is never faked, which makes the beam the identity object. Up to four players, so **four directional beam shapes**: wide cone, narrow cone, split (twin) beam, and broad fan (short and wide). All four stay inside the Tungsten family (Section 4) with at most a small temperature offset; the differentiator is shape because shape survives colorblindness and four beam hues would eat palette the loops need. Half-angle, reach and edge width per shape are tabulated in 8.3 (provisional until 8.12 step 2). *(Amended per AD-ART-BIBLE review 2026-10-25)* Helmet trim carries a matching stripe count (1–4) for the still frame. *Alternative considered*: per-player suit colour, the co-op default. Rejected — it spends four hues in a game that must keep hue scarce, and a suit colour is a sprite property a hallucination could plausibly be imagined to fake, whereas the beam is a light the engine renders honestly.

**Deployable light is never a beam shape.** Flares, torches and decoys are player tools (concept, Anti-Pillars). They stay in the Tungsten family but are **omnidirectional and flicker** — a pool, never a cone — so a horror standing beside a dropped flare cannot be mistaken for a crewmate carrying a light. The identity rule is therefore precise: **anything carrying one of the four player beam shapes is a crewmate.** Omnidirectional light is a tool; no light at all is the web.

**Roles are not silhouettes.** Boarder vs ship-keeper is emergent, so it is carried by the tool in hand (cutter, scanner, welder, suppressant) breaking the outline, not by body type. One trait per archetype: the crewmate's trait is *pack and beam*.

**Horrors: the negation of the crewmate.** Horrors are built from the opposite of every crewmate rule: elongated not compact, asymmetrical not symmetrical, **open and branching** not closed, limb counts that are not four, and **no closed circle anywhere in the silhouette** — the helmet-circle is reserved. They **emit nothing**: a horror is only ever seen by the player's light, and under it shows the web's iridescence (Section 4) rather than any hue of its own. Horror variety (4+ types) is built by varying branch count, length and asymmetry inside those rules, so every new horror is cheap to read as a horror and expensive to mistake for a friend. Emotionally the contrast says *made vs grown*: the web's shapes are the shapes of things never designed for anyone.

**Hallucinated crewmate-as-horror (class c).** The crewmate's sprite is replaced by a horror silhouette — but the beam is an anchor and the human channel is inviolable, so the "horror" is carrying a Tungsten beam of one of the four player shapes. The design test: *a horror with a player's beam is your friend.* The hallucination must also leak violet or flesh at its fringe. Silhouette rule, light rule and palette leak are redundant with one another; a player who learns any one of the three is protected.

**The low-sanity crewmate seen by a saner viewer.** Character sprites never shift, so the sanity sign is placed **outside the sprite's pixels**: a violet fringe around the silhouette's outer edge and along the edge of their beam, like a halo of chromatic error. Sprite, beam colour and beam shape are untouched. This keeps the human channel honest while giving the class (c) victim the tell Pillar 4 promises.

**Hidden real entities at depth (class a) — open question.** The pillar says the hallucination hides the body, never the trail. Whether a hidden entity still occludes the beam (a shadow with no visible occluder) is for the game-designer and technical-artist. Recommendation: it should *not*, because "a shadow with no occluder" is a stage-2 sanity symptom and must stay an unambiguous sanity tell; a hidden entity's presence is carried by its P1 trails (frost, hull stress, dust displacement). Not decided here.

### 3.3 Environment geometry (Pillars 1, 3 and 5)

**Human and mechanical: orthogonal, modular, repeating.** Walls, floors, bulkheads and pipes sit on a grid with right angles and regular rhythms. In 3/4 oblique, walls show a front face and a top; the front face is where the pipe grammar lives, at camera eye height. Repetition is a feature: a run of identical panels is the ground against which a leak, a scorch or a graft becomes figure (Pillar 1).

**The web: branching, continuous, non-repeating (Pillar 3).** No right angles, no identical segments, no closed circles. Filaments taper and fork like roots or nerves and never align to the tile grid, so wherever the web has entered a hull the grid breaks. Under the beam they show the iridescent sheen; unlit they are not drawn (principle 1). The web runs *through* human geometry rather than over it — a filament passes behind a panel and emerges beyond it — which turns continuity against the player: the eye follows it into the dark.

**Loop pipe grammar (shape redundancy for hue; Pillar 1 and principle 2).** Each loop has a distinct diameter, fitting and flow pattern so it reads with colour removed. This is the colorblind backbone Section 4 relies on, and *every pipe is a sentence* made literal: the pipe's shape is its noun, the pattern its verb.

| Loop | Pipe | Fitting silhouette | Flow pattern | Failure trail |
| --- | --- | --- | --- | --- |
| O₂ | thin round | ring clamps at regular pitch | small close-pitched chevrons in flow direction | soft drifting mist |
| Power | flat ribbon / bus bar, not a tube | square junction boxes | dash-pulse travelling along the bar | arc scorch (sharp, dark, radial) |
| Coolant | medium round, **lagged** (wrapped, segmented) | padded collars | wide slow chevrons | crystalline frost fan, static |
| Fuel | thick round | **bolted flanges**; diagonal hazard banding | no chevrons; a slow fill-level sight-glass | glossy puddle with ember edge |
| Integrity | the hull itself: ribs, beams, plates | weld seams, rivet lines | none | hairline cracks, buckling, bent ribs |

Mist drifts, frost is static and crystalline, scorch is sharp and radial, a puddle is glossy and flat, a crack is a hairline. The five trail *behaviours* are distinct at a glance in greyscale.

**Salvage dialects: grafts must read as grafts (Pillar 5; Pillar 1).** The ship is assembled from other manufacturers' wrecks, so it should look stitched together — cosy under the stable light, a hull of scars in the dark. The ship's shape *is* its history (Pillar 5: loss is not total because the ship carries what you learned), and a graft that still looks like its donor is an honest physical record (Pillar 1). Three manufacturer dialects for the full vision, each defined on the same five axes so new kit is cheap to author and consistently different:

| Axis | Dialect A — "Foundry" | Dialect B — "Lattice" | Dialect C — "Civil" |
| --- | --- | --- | --- |
| Corner radius | heavy rounded | sharp chamfers | flat, square |
| Fastener rhythm | big rivets, wide pitch | exposed bolts on trusses | hidden; seams only |
| Panel ratio | squat, wide | tall, narrow | square |
| Pipe routing | surface-mounted, chunky brackets | exposed in open trusswork | recessed in channels, covered |
| Ports / hatches | round | octagonal | rectangular |

The home-ship's original hull is one dialect (recommend Foundry — sturdy and domestic). Every graft keeps its donor dialect: a Lattice scrubber bolted into a Foundry corridor is visibly foreign, and the **weld seam** between them is a raised bead in Hull Grey that is never hidden. The seam is a hero shape (3.5): where the graph joins, where integrity fails first, and where the player's history is written.

*Alternative considered*: a unified ship style with grafts marked by a colour tag. Rejected — a tag is an icon (principle 2), it spends a hue, and it discards the free storytelling of mismatched manufacturers.

### 3.4 UI shape grammar (Pillars 1 and 4)

**The HUD is a human instrument.** It uses the home-ship's own dialect — chamfered rectangles, bezels, rivet-line detailing — never organic curves. The only curves permitted in UI are **dial arcs and needles**, the one instrument shape this world has.

**Frame vs reading.** The HUD *frame* is a never-shift anchor; the *readings* inside it (needle positions, numerals, state glyphs) are instruments and may lie under class (b). The grammar makes the boundary visible: frames are solid, heavy, static; readings are thin, light, moving marks inside them. A player can always tell "this is the instrument" from "this is what it is saying", which makes the stage-1 drift readable rather than merely confusing.

**Icons follow pipe grammar.** Where UI needs a loop glyph it uses the loop's fitting silhouette (ring clamp, square box, padded collar, bolted flange, rivet line) rather than an abstract symbol, so a colorblind player who has learned the ship has learned the UI.

**Web map.** Filaments drawn as branching, non-repeating lines with no right angles; everything human on it — ship, wrecks, routes — is rectilinear. The chart is a human object laid over an inhuman shape.

### 3.5 Hero vs supporting shapes (Pillars 3, 1, 5, 2 in order)

**Rule: in human spaces, *round and moving* is reserved for things that matter.** Supporting shapes (panels, plates, bulkheads, clutter) are rectangular, static and low-contrast — the honest ground. Static circles exist in the grammar (Foundry ports, ring clamps) but never move; hero shapes get the circles *in motion* and the highest local contrast, so "round and moving" is the visual word for *look here*.

Hero shapes, in priority order:

1. **The beam cone** (Pillar 3 — emission is exposure). The single most important shape in the game: hard outer edge, soft interior falloff, always the brightest thing in frame. The player's eye, hand and identity.
2. **Loop machinery with visible moving media** (Pillar 1) — rotor, bellows, sight-glass, pump. Circular and in motion; the proof that *state lives in the sprite*.
3. **The weld seam** (Pillar 5). The ship's history and its first failure point.
4. **The derelict's one visible unknown** (Pillar 5 — deeper is richer). The only lit thing the player did not pay to light.
5. **Failure trails** (Pillar 2). Not objects but the game's most important drawings: a trail must always out-contrast the panel it crosses.

Everything else is supporting. The hierarchy maps onto the core loop — beam (perceive) → machinery (read) → seam (integrate) → unknown (desire) → trail (fail) — and a screen that violates it (a decorative circle, a moving background element) teaches a false "look here". Emotionally it keeps attention on the player's agency: the brightest shape is theirs, the moving shapes are the ship they keep alive, and the web earns no hero shape at all. It is seen only in borrowed light, which is the point.

## 4. Color System

### 4.0 Principles

1. **Hue is scarce and semantic.** The sane world is near-monochrome; every chromatic colour is a sentence about a system or a person. If a colour has no meaning, it is grey.
2. **Hue is never the only channel.** Every semantic colour is redundant with Section 3's shape/pattern grammar and most with a sound cue whose design belongs to the audio-director.
3. **Depth lowers ambient and raises iridescence; it never shifts hue.** The place is never the signal. A hue change always means a system or a mind.
4. **Flesh and violet are reserved.** Nowhere in the sane world; when they appear it is the player's sanity.
5. **Light is colour.** Most colour in frame is emission (loops, beams, alarm) landing on grey steel. Surface albedo stays cold and desaturated so emission reads cleanly.

Values are sRGB and approximate; the lighting spike tunes them under the real renderer (Forward+ HDR 2D with glow is the working assumption and is reversible). The *relationships* — hue angle, saturation band, value ordering — are the contract, not the hex.

### 4.1 Primary palette (7)

| # | Name | Hex (approx.) | Role | Why this colour means this here |
| --- | --- | --- | --- | --- |
| 1 | **Void Steel** | `#0B1018` | Base dark: unlit hull (material floor tint, 8.11 flag 1), shadow, void | Blue-black rather than pure black so true black (0 % ambient at depth) is still *darker* than the hull — the dark has a floor the player can learn. Cold because the default is cold. "Darker than true black" is implemented by the material-level hull floor of 8.11 flag 1, because an ambient multiply alone quantises both to black in 8-bit output. *(Amended per 8.11 flag 1)* |
| 2 | **Graphite** | `#2A303A` (mid) / `#5C6670` (lit) | Human construction: walls, plates, suits | Desaturated cool grey takes emission cleanly and never competes with it. The ship is a surface for light to write on. All large planes are Graphite (see 4.4 test). |
| 3 | **Tungsten** | `#FFE2A8` (beam core) / `#FFCC66` (fixture) | Human light: beams, fixtures, deployables, UI accent | Incandescence is the colour of a *made* light; the only warmth the sane world has, and it belongs to people. Hue ≈ 40°, yellow-amber (deliberately not orange — see 4.4), high value. |
| 4 | **O₂ Cyan** | `#3FD6E8` | Oxygen loop | Breath and clinical air — the colour of oxygen readouts. Near-complement of Fuel Ember so the two loops most likely to vent together are maximally separated. |
| 5 | **Power Amber** | `#F2C230` | Power loop | Power here is incandescent — the same lineage as the human light, because the ship's power is the crew's own warmth. Separated from Tungsten by saturation (80 % vs 34 %), shape and behaviour, not by hue (4.2). |
| 6 | **Alarm Red** | `#E0102A` | Alarm only | Universal, and kept *exclusively* for alarm so rarity is the signal: red appears in no loop and no UI accent — only the breach sweep and a failing instrument's warning mark. |
| 7 | **Hull Grey** | `#8C949C` | Integrity loop; weld seams, ribs, cracks | The hull is ground truth and the most honest thing in the world; it is given no hue because it has nothing to lie with. Thin-element only (seams, ribs) — never a plane. |

**Extended loop hues (same rules, subordinate):**

| Name | Hex | Role | Why |
| --- | --- | --- | --- |
| **Coolant Ice** | `#C8E6F5` | Coolant loop | Frost: very pale, low saturation, highest value in the loop set. Deliberately *not* a second blue — near-white with a cold cast, so against O₂ Cyan the pair differs in saturation (20 % vs 70 %) and value, not just hue. |
| **Fuel Ember** | `#F06A28` | Fuel loop | Combustion. Orange-red at high saturation (> 80 %), which keeps it out of the flesh band (4.4) and distinct from Alarm Red (hue ≈ 352°) by hue and behaviour — fuel is a steady fill, alarm is a sweep. |

**Web iridescence (no hue of its own).** Under direct light the web shows a thin-film sheen cycling **teal → green → bronze → gold** at low saturation, applied as a specular *band*, never a fill. **The magenta–violet arc (hue ≈ 270–330°) is excluded**, although real thin-film iridescence passes through it: iridescence is highest at depth, and if it carried violet, deep rings would leak the sanity hue to sane players and break Pillar 1's sane-palette channel. Unlit, the web has no colour and is not drawn.

*Alternative considered* for the base: warm sepia-grey steel (the "worn freighter" look). Rejected — warm steel absorbs Tungsten and crowds the flesh band; the cold base is what makes one warm light legible and flesh impossible to reach by accident.

### 4.2 Semantic usage

| Meaning | Colour | Shape / pattern redundancy | Where it may appear |
| --- | --- | --- | --- |
| A crewmate | Tungsten | one of four directional player beam shapes | beams only |
| Human light (not identity) | Tungsten | fixtures: static housings; deployables: omnidirectional pool, flickering | fixtures, flares, torches, decoys, UI accent |
| A loop is running | its hue, emissive along the pipe | flow pattern animating | pipes, machinery media |
| A loop is dead / unpowered | its pipe in Graphite, no emission | pattern static | derelicts, conserved dark |
| A loop has failed | its hue in the **trail** medium | trail behaviour (drift / static crystal / radial / gloss / hairline) | floors, seams, elbows |
| Alarm | Alarm Red, rotating sweep, 2–3 s | sweep motion; warning mark on gauge | breach, lost dark |
| Attention rising | iridescence brightening, more filaments resolving | filament count and movement at the cone edge | Tell, breach |
| Sanity (own) | Violet, then Flesh (4.4) | always at an *edge*: cone rim, pipe silhouette, gauge glass, sprite fringe | hallucinated content only |
| Sanity (a crewmate's, seen by you) | Violet fringe | around their outline and beam edge; sprite pixels untouched | class (c) tell |
| Integrity / history | Hull Grey | raised seam, rivet line, hairline crack | seams, ribs |

**Safety and danger are not colours here.** This game has no "safe" colour because safety is never a state of the place; it is something the crew is doing. The nearest thing to "safe" is the *stable composition* of 2.3 — many warm sources, every loop glowing, nothing red. The nearest thing to "danger" is the absence or corruption of that picture. Reward is not a colour either: salvage is Graphite plus its loop hue, and phantom salvage (class d) betrays itself with a flesh or violet leak — the cheap field check Pillar 4 requires.

**The beam/power collision, resolved.** Section 1 gives both the human light and the power loop the word "amber". Shared, a false-system-state hallucination on a power conduit could resemble a crewmate's light and erode the human channel. At the chosen hexes their hues are only ~5° apart, so **hue is not a separating axis**; they are separated on three others: *saturation and value* (beam core is near-white, `#FFE2A8`, S 34 %; Power Amber is deep, `#F2C230`, S 80 %), *shape* (a beam is a hard-edged cone with soft interior falloff that moves with a body; power is static emission along a flat bar), and *behaviour* (a beam never pulses; power always carries a travelling dash). Fixture Tungsten (`#FFCC66`) sits between them in saturation, which is acceptable because fixtures are static housings and the human channel protects *beams*, not fixtures.

### 4.3 Depth rings: temperature and ambient

Depth never changes hue. It changes two things. The table is an **ordering contract** — outer > middle > inner — whose values are spike-written per 8.12 step 7 (8.11 flag 1); the percentages are targets, not settings. *(Amended per 8.11 flag 1)*

| Ring | Ambient floor (of white) | Iridescence intensity | What the player sees |
| --- | --- | --- | --- |
| Outer (1) | ≈ 3 % | low — a faint sheen only within a few tiles of a direct beam | The hull's far wall is a suggestion; the web is a texture you can almost ignore |
| Middle (2) | ≈ 1–2 % | medium — the band is visible across the whole cone; filaments look wet | Outside the cone is black; filaments catch the light and hold it |
| Inner (3) | 0–1 % | high — filaments throw highlights back and appear lit from within | The beam is the whole world; the web glitters at its edge and seems to lean in |

Temperature holds constant: Graphite stays cool-neutral, Tungsten stays tungsten. A warmer or cooler ring would make the place the signal and collide with the stage-2 drift ("temperature shifts"), which must remain unambiguously a mind's error. Home-ship ambient (10 % stable, 2 % dark) is independent of ring: your own ship is lit by what you run, not by where you are.

### 4.4 The sanity palette

**Reserved colours.**

| Name | Hex (approx.) | Definition (a gamut band, not a value) |
| --- | --- | --- |
| **Violet** | `#8A3FD1` (core), `#B07BE6` (fringe) | hue 270–330°, saturation above 25 % |
| **Flesh** | `#D9A58C` (light), `#B5705C` (deep) | hue 5–30°, saturation 20–60 %, value 45–85 % |

**Why the flesh band is defined by hue first.** Warm light on grey steel inevitably produces a mid-saturation, mid-value beige — and *desaturating* the light moves lit steel further *into* that band, not out of it. So the control is hue: Tungsten is held at **hue ≥ 38°** (yellow-amber), flesh is **≤ 30°** (orange-pink), and lit steel lands at 40–45°, which reads as sand or brass, not skin. Fuel Ember (hue 20°) sits in the flesh hue range but at S > 80 %, outside by saturation; Alarm Red (352°) is outside by hue. **Asset test:** sample Hull Grey and lit Graphite under every Tungsten source; the result's hue must stay ≥ 36°. If a light has drifted toward orange, pull it back toward yellow — never desaturate it. **Known near-case:** Hull Grey under fixture Tungsten computes to roughly hue 43°, S 56 %, V 55 % — outside the band on hue, but it is the closest sane-world colour to flesh, which is why Hull Grey is restricted to thin elements (seams, ribs, cracks) and large planes are Graphite (lit value ≤ 40 %, below the band). The asset reviewer checks warm-lit Hull Grey first.

**Why these two hues.** Violet is the one hue neither the human light (warm) nor the cosmic dark (blue-black) can produce — the hue the sane world structurally lacks, so its presence is unexplainable by the world. Flesh is the body intruding: the crew live sealed in suits, the ship is "a body we assembled", and when horror comes from inside the mind it comes as skin. Together they say *this is you, not the place*.

**Drift order and what each stage colours.** Every stage leaks; subtle is allowed, absent is not (Pillar 1).

| Stage | What lies | Where the palette leaks | Anchors untouched |
| --- | --- | --- | --- |
| 1 — Instruments | needles, readouts, state glyphs disagree with the ship | a violet gleam on the lying instrument's glass or needle edge | HUD frame, sprites, tools, true beam radius |
| 2 — The light | cone *appears* to reach further; shadows with no occluder; apparent warmth | violet fringe at the cone rim; shadows go violet-black instead of steel-black; the false reach is tinted | the geometry the real radius reveals — walk into it and it is honest |
| 3 — The world's hue | loop hues bleed at pipe silhouettes; surfaces warm toward flesh; pipes read as veins; phantom entities, salvage, crewmates-as-horrors | violet at every pipe edge; flesh on steel and phantom content; a horror-shaped crewmate carries a violet fringe over a human beam | sprites (own and others'), hand tools, HUD frame, beam colour and shape |

Violet comes before flesh because violet is a *cool* wrongness that sits beside the cold base and reads as an error of the eye, while flesh is a *warm* wrongness that reads as a change in the world; the drift moves from "my eyes are off" to "the ship is alive in the wrong way". *Alternative considered*: desaturating the world as sanity drops (the survival-horror convention). Rejected — it removes information the player needs to run the ship, is a stat-like global filter rather than a falsifiable lie, and has no leak to catch.

**The crewmate mark.** A crewmate below threshold, seen by a saner player, shows the violet fringe (`#B07BE6`) around their outline and along their beam's edge. Sprite pixels, beam colour and beam shape are untouched. The mark is on the *edge* like every other leak, so "look at edges" is the same lesson everywhere.

### 4.5 Luminance ladder (greyscale and achromatopsia)

Approximate perceptual lightness (L*, from the sRGB hexes), highest first. Pairs marked ≈ are the greyscale collisions; each is carried by shape in 4.7.

Tungsten beam core (≈ 92) ≈ Coolant Ice (≈ 91) > Power Amber (≈ 80) ≈ O₂ Cyan (≈ 79) > Hull Grey (≈ 61) ≈ Fuel Ember (≈ 61) > Alarm Red (≈ 47) ≈ Violet core (≈ 43) ≈ Graphite lit (≈ 43) > Graphite mid (≈ 20) > Void Steel (≈ 5).

Two usable gaps: Fuel Ember vs Alarm Red (≈ 61 vs 47) is a secondary backup for the protan/deutan pair; Tungsten vs Power Amber (≈ 92 vs 80) backs the beam/power separation. The ladder is a backstop, not the primary channel.

### 4.6 UI palette and where it diverges

| UI role | Colour | Note |
| --- | --- | --- |
| Ground | Graphite `#1C2128` | darker than world mid-grey so world emission never competes with UI |
| Frame / bezel | Graphite lit `#5C6670`, Hull Grey `#8C949C` highlights | the HUD is the home-ship's dialect |
| Text | Paper White `#DDE4EA` | cool, not cream — cream drifts toward the flesh band |
| Accent / selection | Tungsten fixture `#FFCC66` | the UI is a human instrument |
| Warning mark | Alarm Red `#E0102A` | only on a failing instrument; never decorative |
| Loop glyphs, gauge faces | the loop's hue with its fitting silhouette | the UI teaches the same grammar as the ship |
| Map filaments | Hull Grey, thin | no iridescence on the map; the chart is human |
| Map depth rings | ground stepped darker toward centre | the only "deeper = darker" rule, carried into UI |

**Divergences from the world palette.** The UI is flat and unlit (no falloff, bloom or occlusion); it never shows iridescence; it is the only place Paper White appears; and **Violet and Flesh are never UI colours** — the HUD frame is an anchor and sanity is not a meter. The single exception is not a UI colour but a lie: a class (b) *reading* inside the frame carries the stage-1 violet gleam on its glass (4.4). That gleam is the only violet that ever reaches the screen-space layer, and it marks the reading, never the frame.

### 4.7 Colorblind safety table

Pairs chosen by proximity under protan/deutan (red–green), tritan (blue–yellow) and achromatopsia. Each names the backup already in the shape grammar; "sound cue" means a distinct cue is required and its design is routed to the audio-director.

| Pair | Fails for | Shape / pattern backup | Other backup |
| --- | --- | --- | --- |
| O₂ Cyan vs Coolant Ice | tritan; weak for everyone | thin round + ring clamps + close chevrons vs medium lagged + padded collars + wide slow chevrons; trail: drifting mist vs static crystalline frost | saturation gap (70 % vs 20 %); sound cue |
| Tungsten vs Coolant Ice | achromatopsia (L* 92 vs 91) | moving hard-edged cone vs a static pipe / static crystalline trail | Tungsten is a light with falloff; Ice is a fill |
| Power Amber vs Fuel Ember | protan, deutan | flat bus bar + square boxes + dash-pulse vs thick round + bolted flanges + hazard banding + fill-level glass; trail: radial scorch vs glossy puddle | L* gap (80 vs 61); sound cue |
| Fuel Ember vs Alarm Red | protan, deutan | fuel is a static pipe and steady fill; alarm is a rotating 2–3 s sweep with a gauge warning mark | motion is primary; L* gap (61 vs 47); sound cue |
| Tungsten vs Power Amber | protan, deutan; near-shared hue for all | moving cone (one of four player shapes) vs static flat bar with travelling dash; beam never pulses, power always does | saturation gap (34 % vs 80 %); human channel is shape-protected by design |
| Hull Grey vs Fuel Ember | achromatopsia (L* 61 vs 61) | raised seam / rib / hairline vs thick flanged pipe / glossy puddle | — |
| Violet vs Void Steel / blue shadow | deutan (violet reads blue), tritan | violet only at *edges* — cone rim, pipe silhouette, gauge glass, sprite fringe — never a fill; a sane shadow has an occluder, a lying one does not | physical falsification (walk into the light's claimed radius); second instrument; saner crewmate |
| Flesh vs Tungsten-lit steel | achromatopsia; mild for all | flesh appears on *surfaces* and phantom content; Tungsten is a light with a cone; lit steel holds hue ≥ 36° (4.4 test) | physical consequence stays honest (P1) |
| Hull Grey vs Graphite | achromatopsia | seam is a raised bead; crack is a hairline — geometry, not colour | none needed; designed to be near |
| Iridescence vs loop hues | none significant | moving specular *band* on branching geometry vs fills on orthogonal pipes | — |

**Pairs needing no backup**: Hull Grey (achromatic by design), Void Steel vs everything (value), Alarm Red vs O₂ Cyan (complementary and behaviourally distinct). **Rule for new colours**: any new semantic hue is entered in this table with a named shape backup before it is used in an asset.

---

**Consistency notes.** Six tensions were resolved across the three sections. (1) *Beam vs power amber*: Section 1 used one word for both; at the chosen hexes their hues are only ~5° apart, so 4.2 separates them on saturation/value, shape and behaviour — not hue — and 3.2 makes beam *shape* the player's identity so the human channel never depends on colour. (2) *Deployable light vs the horror rule*: flares, torches and decoys are human lights a horror could stand beside; 3.2 restricts the identity rule to the four directional player beam shapes and makes all deployables omnidirectional and flickering, and the fourth beam shape was changed from a ring lantern to a broad fan for the same reason. (3) *Pod light and the unknown*: the pod's lights are Tungsten fixture (no neutral white exists in the palette), and the wreck's one visible unknown is a residual *human* light, not an iridescent gleam, because the web is seen only in the player's own light (principle 3). (4) *Iridescence vs the sanity hue*: physical thin-film colour passes through magenta; 4.1 excludes the 270–330° arc so deep rings (highest iridescence, 2.5/4.3) cannot leak violet to a sane player. (5) *Warm light vs flesh*: warm light on grey steel produces beige, and desaturating the light makes it worse; 4.4 defines flesh by hue first (≤ 30°), holds Tungsten at ≥ 38°, restricts Hull Grey to thin elements, and gives a sampling test with the known near-case named. (6) *Hidden entities vs the stage-2 light lie*: both could produce "a shadow with no occluder"; 3.2 recommends hidden entities do not occlude, leaving that image unambiguously a sanity tell — flagged as an open question for the game-designer, not decided here. Downstream flags for later sections: the flesh exclusion implies crewmates show no bare skin in the sane world (helmets on, visors reflective) — a Section 5 consequence; all ambient percentages and hex values are relationships to be tuned under the real renderer in the lighting spike and recorded in Section 8; and sound cues named in 4.7 are requirements routed to the audio-director, not designs.

## 5. Character Design Direction

### 5.0 Governing principle

**The crewmate sprite is the one instrument in the game that cannot lie.** Section 1's Corollary and 4.4 name four anchors that never shift under hallucination — character sprites, hand tools, the HUD frame, the true beam radius — and of those, the sprite is the only one that walks, works, gets cold and can be looked at by a friend. That gives character design two jobs the rest of the world does not have. It must be **maximally honest**: anything drawn on a crewmate is true on every client, so the sprite can carry real body state the way a pipe carries loop state (principle 2). And it must be **maximally cheap**: one body shared by up to four players, with identity, state and history carried by light, palette and overlays rather than by frame variants — 2.0's cost rule applied to people.

**How the anchor rule reconciles with class (c).** 3.2 says a hallucinated crewmate-as-horror *replaces* the sprite with a horror silhouette; 4.4 says sprites are never touched. Both hold if the rule is read as: **the hallucination layer never edits a crewmate's pixels — it may only substitute the whole body.** A crewmate is rendered either exactly as drawn or as a horror rig at the same position and velocity, carrying the crewmate's true beam and the violet fringe. The beam's origin stays at the crewmate's **true hand point** whichever body is drawn — the horror rig has no hand, and the light is anchored to the player, not to the sprite (an anchor rule for the technical-artist). There is no half-transformed crewmate, no warped visor, no tinted suit, no transition frame. This is a budget rule (no transition art is ever authored) and a learnable rule for the player: if a crewmate looks *slightly* wrong, it is light or cold or a scar, never sanity — sanity on bodies is all-or-nothing.

### 5.1 The player archetype: the Sealed Technician

**Build.** Per 3.2: a round helmet over a square life-support pack, compact and symmetrical, hands and tool breaking the outline to the side. Proportion targets in 3/4 oblique: helmet ≈ 30 % of the visible figure height, torso-and-pack ≈ 45 %, legs ≈ 25 %. The helmet is a true circle in silhouette from every facing; the pack is a true rectangle wider than the torso and visible over the shoulders from every facing, so the circle-on-rectangle read survives rotation. Exact pixel dimensions belong to Section 8; the working ratio is a crewmate ≈ 1.1 floor tiles tall, against a wall front face of 1.5 tiles (6.2), so the helmet sits below the wall's top cap and the pipe band crosses at the crewmate's head-and-shoulder height — a pipe is always read *past* the figure, never *behind* its helmet.

**Material and value.** The suit is soft-pressure fabric in Graphite mid with hard shell plates (helmet, pack, forearms, knees) in Graphite lit. A one-pixel Void Steel contour runs around the whole figure at sprite resolution. The contour is the figure–ground device of 3.0 made explicit: characters and hero objects have a contour, supporting environment never does (6.6). Suit albedo is flat and unshaded, lit by the engine (6.2), with a normal map so the helmet sphere catches the beam direction honestly.

**No skin, anywhere.** This is the Section 4 downstream flag honoured: flesh is reserved (4.4), so no crewmate shows bare skin in the sane world. The helmet is sealed. The visor is dark reflective glass — Void Steel with a single tight Tungsten specular glint that follows the strongest light in the scene. *Why reflective rather than opaque or transparent:* a transparent visor needs a face (the flesh band, and face frames per facing); an opaque robotic faceplate loses "competent and fragile" (3.2) and makes the crew read as machines, which undercuts ship-as-body-made-by-people. A mirror keeps the body unseen and turns the face into a tiny honest instrument: the glint shows where the light is, which is Pillar 1 in a few pixels. The rule extends past the player: pod sleepers and the crews of dead ships are also suited and sealed (6.5) — flagged for the narrative-director, since the cryo-pod question is theirs; the visual constraint is simply that no body is ever shown.

**Identity.** The beam shape is identity (3.2) and this section adds nothing to that. For the still frame, the helmet carries **four trim bands** as geometry on every helmet, always drawn, so mirroring and animation are free; the player's count (1–4) renders in Hull Grey (bare metal that catches light) and the remainder in Graphite mid. That is a palette swap, costs zero frames, and a *count* of bright bands survives greyscale (4.5). Bands are **non-emissive**: a glowing helmet would be a lantern (principle 3) and would compete with the beam as the identity object. *Near-case:* 4.4 names warm-lit Hull Grey as the sane world's closest colour to flesh and tells the reviewer to check it first; bands are thin elements (legal per 4.1) but are bound to the 4.4 sampling test under every Tungsten source, including another player's beam landing on the helmet. If they fail, bands fall back to Graphite lit and lose contrast; they are never warmed.

**Tool in hand.** A separate sprite layer. The tool is the role (3.2) — cutter, welder, scanner, suppressant, deployable light, carried module — and hand tools are anchors (Corollary), so tools are drawn once, plainly, and never styled per dialect.

### 5.2 Expression and pose style: stiff-weighted

Against the brief's four targets, the crewmate is **stiff** (with weight) — not expressive, not realistic, not exaggerated.

**Why stiff.** (1) The face is sealed, so there is no facial channel; expression lives in posture, in the tool, and above all in **where the beam points** — the beam is the eye (3.5, hero #1), and "looking at something" is lighting it. (2) Section 2 sets each state's energy with lighting and media (2.2: "nothing moves faster than a drip"), and 2.9 forbids the sanity overlay from changing tempo; a body with squash-and-stretch announces energy the lighting did not set and breaks that ladder. (3) Project Zomboid (Section 1 touchstone): patient, procedural bodies that read as competent people doing a job. (4) Budget: stiff motion needs fewer frames to look finished.

**Rules.** No squash-and-stretch, no smear frames, no overshoot on the silhouette. Idle is a 2–3 px rise and fall of the pack — the pack breathes, because the body is sealed. Walk is an 8-frame deliberate cycle with no bounce in the helmet. Run is 8 frames with a forward lean of the whole figure, helmet circle preserved. The work loop is 6 frames, two-handed, tool forward. Carry is a walk variant with the module held at chest height, breaking the outline upward.

**Where exaggeration is permitted.** Only in reaction frames — stagger, recoil from a venting loop, collapse — because those are the honest body states that must be read across a room in a sweeping red light (2.8). Truth earns emphasis; comfort does not. Even there the helmet stays a circle and the figure stays upright-ish; a crewmate must never curl, elongate or splay into the horror grammar of 3.2.

**Alternatives rejected.** *Expressive / cartoon*: squash breaks the 25 % silhouette test (3.1) in motion, and bodies that out-move the ship invert the 3.5 hierarchy (the moving shapes are supposed to be the ship the crew keeps alive). *Realistic / rotoscoped*: frame count a solo developer cannot carry, and realism keeps asking for skin. *Exaggerated*: a crewmate exaggerating a stagger reads as a horror's unfold — a direct collision with the negation rule in 3.2.

### 5.3 Honest body states and sanity scars

**Vital overlays — the sprite as a channel of truth.** Because the sprite is an anchor, anything drawn on it is true on every client. This section uses that deliberately with two overlays and one animation: **rime** on the shell plates (pale, desaturated, static crystalline — the same physical medium as the coolant trail of 3.3, because it *is* cold; it sits only on plates, never at a fitting, so it cannot be read as a coolant leak) means the crewmate is genuinely cold; **visor fog** (condensation inside the glass, dimming the glint) means genuinely bad air; the **stagger** animation means genuinely failing. A low-sanity player who cannot trust the O₂ gauge can still see a crewmate's fogged visor in their beam — "you look cold" becomes a cross-check that costs only light. Which thresholds trigger each overlay is the game-designer's; the visual rule is that overlays are always true and never used for anything the simulation does not confirm.

**Sanity is never drawn on the body.** Not on the player's own sprite (there is no self-tell; the lie is seamless to the liar except where the world leaks), and not in a crewmate's pixels (the tell sits outside the pixels — 5.5).

**Sanity scars: a scar is a weld seam on a person.** The concept lists permanent quirks a character carries. Visually they are **suit repairs in the ship's own language** (3.3, 3.5): a Graphite patch on the pack with a Hull Grey bead at its edge (the bead is the only Hull Grey — thin element, 4.1), a pack clip or forearm fitting replaced from another dialect (an octagonal Lattice clip on a Foundry pack), a sealed crack line across the visor, a lagged wrap on one forearm. The helmet slot holds only the visor crack or a crown patch and **never touches the four trim bands**, so a scar can never change or confuse the band count that carries identity (5.1). The crewmate accrues grafts the way the ship does, and for the same Pillar 5 reason — loss is not total; the history is worn. Three visual slots (helmet, pack, limb), each with two or three options, authored as overlays following a per-frame offset table, never as body variants (2.0). Scars are always Graphite or Hull Grey and never violet or flesh: they are real. They sit **inside** the silhouette, grey and static — in every way the opposite of the sanity tell, which is outside, violet, and belongs to the viewer. How many scars exist, how they are earned, and whether guests keep them (the concept's open question) are game-designer and creative-director decisions; this section fixes only how a scar is drawn.

### 5.4 Horrors: what they may and may not borrow from the human silhouette

3.2 defines horrors as the negation of the crewmate. This section turns that into an allowed/forbidden list and a construction method.

**May borrow.**
- *The height band.* A horror's visible mass sits between 1× and 3× crewmate height (elongation may exceed that along a filament), so camera scale holds and doorways still frame them.
- *Reach.* The gesture of a limb entering the cone before the body does is the signature image; it borrows the human idea of reaching toward, with nothing else human about the limb.
- *Human geometry as a surface.* A horror may drape over or thread through machinery, a doorway or a pod row — the web runs through human geometry, not over it (3.3) — and may stand beside a dropped deployable light (omnidirectional, 3.2), which is exactly why deployables are never beam-shaped.
- *Physical honesty.* Horrors are real: they occlude the beam, cast shadows, and displace dust.

**May not borrow.** A closed circle anywhere in the silhouette (the helmet circle is reserved); bilateral symmetry; four limbs in two pairs; an upright compact stance on a vertical axis; anything held *in a hand*; any of the four player beam shapes; any Tungsten, or any emission of its own; any worn human part — no helmet shells, no pack fragments, no band count; Graphite plating or any albedo hue. Under the beam a horror shows Void Steel mass with the iridescence band of 4.1 and nothing else.

**Variety matrix (4+ types).** Behaviours belong to the game-designer; the visual roster is built on four axes inside the 3.2 rules — **branch count** (3 / 5 / 7+), **elongation** (1.5× / 3× / 6× crewmate height), **attachment** (floor / wall-and-ceiling / filament-rider), **motion signature** (creep / unfold / snap). Rule: every type differs from every other on at least two axes, so each new horror is cheap to read as a horror and expensive to mistake for a neighbour. Example combinations, as working targets only: a 3-branch, 1.5×, floor, *creep* type for the outer ring; a 7+-branch, 6×, filament-rider, *unfold* type for the inner.

**Construction: segment rigs, not frame sheets.** Each horror is a core mass plus N limbs of three or four tapering segment sprites, assembled and animated procedurally, stepped at a fixed low frame rate so the motion keeps a pixel cadence, with the iridescence band applied per segment as a shader. *Why:* branch-count variation is free; 3.3 says web shapes **never repeat**, and a hand-animated loop is repetition by definition, whereas a rig with per-instance variance never plays the same reach twice; and the roster cost drops from roughly 80–100 frames per type to 10–15 segment sprites per type. *Risks, routed to the technical-artist:* rotating pixel segments blur sub-pixel, and rigs can look like puppets. Requirements, not implementation: a grid-snapped look (segment rotation quantised or redrawn at key angles), stepped playback, joint overlap so segments never show gaps, and the 3.1 silhouette test run on **three canonical rig poses per type** — rest, reach, move — since a rig has no canonical frame. 2.0 applies to horrors too: one rig per type serves every lighting state. *Alternative rejected:* frame-animated horrors — ~400 frames for four types, visible loops, and no cheap path to variety.

Horrors are never seen unlit (principle 1) and never glow (principle 3). The player's beam is the only way to see one, which is the point: looking is spending.

### 5.5 Telling them apart at a glance

| | Crewmate | Horror (real) | Hallucinated crewmate-as-horror (class c) | Hidden real entity (class a, at depth) |
| --- | --- | --- | --- | --- |
| Silhouette | circle on rectangle, symmetric, closed, upright | branching, asymmetric, open, no circle | horror silhouette | **nothing drawn** |
| Light carried | one of four Tungsten beam shapes, from the hand | none — seen only in your light | **one of four Tungsten beam shapes** (the crewmate's true beam) | none |
| Palette on it | Graphite, Hull Grey bands, no hue | Void Steel + iridescence band (4.1) | iridescence **plus a violet fringe** (`#B07BE6`) on outline and beam edge | none; footprints are hue-less |
| Motion | stiff, deliberate (5.2) | creep / unfold / snap | the rig is driven by the crewmate's real velocity — a horror that walks like a technician | footprints only (below) |
| What tells you | beam shape; bands; voice | silhouette; no beam; no circle | beam shape; fringe; voice (human channel) — three redundant tells, any one suffices | dust, filaments, smear, rime, hairlines moving with nothing visible |
| Falsifies it | nothing needed | a second crewmate sees it too; it leaves honest trails | the beam (always); a saner crewmate; the human channel | a saner crewmate sees the body; time |

**The victim's tell (class c), specified.** Per 3.2 and 4.4 the low-sanity crewmate seen by a saner player carries a violet fringe `#B07BE6`; this section fixes its rendering so the technical-artist can build it. Width: 2 px at sprite resolution, drawn **outside** the Void Steel contour so the sprite's own pixels, including the contour, are untouched. Extent: the full outer silhouette and both edges of the beam cone. Behaviour: a slow breathing of 4–6 s period — mandatory, since Section 7 (HR-7) makes fringe motion a required channel alongside hue — never a flicker (flicker is the deployable light's word, 3.2) and never a tempo change (2.9). Nature: it is **per-viewer presentation**, so it cannot be baked into any sheet (the same crewmate is clean on another client); it renders in the unlit pass so it is visible even when the crewmate stands in darkness; it **casts no light and raises no Attention** — it is not there, and treating it as emission would collide with principle 3. No sound belongs to the fringe itself: it is per-viewer presentation and silent. The P1 proximity cue that Section 7 (7.5, HR-7) adds is a different object — a physically real sound from the low-sanity crewmate's suit, honest by construction and audible on every client (whether its own source hears it is the game-designer's and audio-director's call). A player never sees their own fringe. The honest sprite is also a tell in itself: when a low-sanity attacker swings at what they see as a horror, the victim sees a fringed crewmate performing the work loop toward them.

**Hidden-entity footprints, specified.** Pillar 4(a) says the hallucination hides the body, never the trail, and lists sound, frost, hull stress and light disturbance. Two of those words — frost and hull stress — belong to loop trails in 3.3 (coolant's static crystalline fan; integrity's hairlines along ribs and seams), and a footprint that could be read as a loop failure would give Pillar 1 a false positive. So footprints are drawn in **the web's vocabulary, never the loops'**: branching, iridescent where lit, and carrying no loop hue. The set, shared with the Tell of 2.7 where noted: (i) dust in the beam changes direction (shared); (ii) filaments twitch (shared); (iii) a wet iridescent smear on floor or wall in a branching pattern, with no filament attached; (iv) **branching rime** — white-grey, forking like a filament, never Coolant Ice's static fan and never at a coolant fitting; (v) hull-stress hairlines that **radiate from a contact point** rather than running along a rib or seam the way integrity failure does. Whether a hidden entity occludes the beam remains the open question 3.2 leaves to the game-designer and technical-artist; the recommendation there — that it should not, so "a shadow with no occluder" stays an unambiguous stage-2 sanity tell — is restated, not decided.

### 5.6 Facings, LOD and what survives at camera distance

**Facings.** Eight directions, five drawn (S, SE, E, NE, N), three mirrored. The design is mirror-safe on purpose: bands are rings, the pack is symmetric, scar overlays flip with their offsets; only the tool changes hands on a mirror, which is accepted — the hand is not a channel and the tool is a role, not an identity. The beam is engine light anchored to a per-facing hand point and rotates freely, so the sprite is 8-way while the light is 360°; the two never fight because the hand point moves with the facing. Horror rigs have no facings.

**LOD philosophy: silhouette first, marks second, texture never.** Detail that does not survive the 3.1 test is removed from the sprite, not compensated with more detail. Three tiers:

| Tier | Must survive at | Carries |
| --- | --- | --- |
| 1 — Silhouette | 25 % scale, no colour, no contour (3.1) | helmet circle, pack rectangle, tool breaking the outline; for horrors, branching and asymmetry |
| 2 — Marks | game camera at 100 % and at 50 % | band count, scar present (not which), tool type, rime and visor fog, the violet fringe |
| 3 — Surface | 100 % only; may be lost | fabric seams, plate edges, visor glint detail, which scar |

Rules: minimum feature 2×2 px at sprite resolution; no single-pixel noise (Section 1 constraint: readability from silhouette, hue and motion, not texture); flat albedo with a normal map (6.2) so the beam direction does the shading. If the camera offers zoom, the crewmate is authored for the default and the 50 % check covers zoomed-out; there is no second sprite set (2.0).

### 5.7 Sprite-sheet budget per archetype

Counts are drawn images. Each sheet also carries a painted height/material pass of 2–4 grey levels at roughly 20–30 % of its albedo drawing time, from which the normal and specular sheets are generated (6.2, 8.8); the generated sheets add no images, but the height pass is drawing time. *(Amended per 8.11 flags 3 and 5)*

| Archetype | What is drawn | Count |
| --- | --- | --- |
| Crewmate body (one, shared by all players) | 5 facings × (idle 6 + walk 8 + run 8 + work 6 + carry-walk 8 + stagger 4 + down 6 = 46) | 230 |
| Tool layer | 6 tools × 5 facings × 2 states (held, in use) | 60 |
| Scar overlays | 3 slots × 3 options × 5 facings, static, offset-tabled | 45 |
| Vital overlays | rime, visor fog × 5 facings (rime as 2-frame overlay or as a shader mask — technical-artist's call) | ≈ 10 |
| Band count | palette swap | 0 |
| Violet fringe | shader | 0 |
| Horror types (4) | 10–15 segment sprites each + 3 test poses | ≈ 50–60 |
| Hallucinated crewmate-as-horror | reuses a horror rig + the true beam + the fringe | 0 |
| Hidden-entity footprints | 5 small VFX loops of 4–6 frames; two shared with the Tell | ≈ 25 |
| Pod sleepers | one suited silhouette behind frosted glass, 2 states (frosted, clearing) | 2 |
| Dead crews | no bodies; empty suits are Section 6 storytelling props | 0 here |
| Beam edge masks (fringe) | 4 small unshaded edge-mask sprites, one per beam shape, so the 5.5 fringe can follow a beam edge a `Light2D` cannot outline (8.10) *(Amended per 8.11 flag 3)* | 4 |
| **Total, full vision** | | **≈ 424–434** |

For comparison, per-player suit colours, frame-animated horrors and baked band variants would put the same roster past 1,200 images. MVP ships the crewmate body at reduced fidelity ("art not at final fidelity") with one horror rig; nothing in this table is MVP-blocking.

## 6. Environment Design Language

### 6.0 Governing principle

**The environment is the text the beam reads, and it was written by three hands in a fixed order.** Manufacturers built it (the dialects of 3.3); a crew altered it (welds, grafts, repairs, cuts); the web entered it (intrusion). Each hand has its own grammar — orthogonal-and-repeating, Hull Grey beads and off-rhythm patches, branching-and-never-repeating — and the player reads history as the order the layers were laid down: dialect under weld under web. Every room must answer three questions in one beam-sweep: who built this, what did people do to it, has the web come in. Gestalt does the work (3.0): similarity makes a dialect, a break in similarity makes a graft, continuity makes a pipe worth following, and figure–ground is the beam itself.

### 6.1 Architecture: dialects, grafts and the web

**Dialects.** The three manufacturer dialects and their five axes are defined in 3.3 and are not restated. This section adds what each implies for how a space is *read* — visual implications only; the cultures and histories behind them belong to the narrative-director and are flagged there.

- *Foundry* is the most legible dialect: surface-mounted pipes on chunky brackets put the whole sentence on the wall face. Big domestic volumes, built to be repaired by hand. The recommended home-ship origin (3.3), which means the player learns the loop grammar in its clearest form.
- *Lattice* is the most exposed: open trusswork, nothing hidden, everything reachable — built to be worked in by people in suits. Pipes read through the truss from odd angles, so Lattice rooms are where the beam's angle matters most.
- *Civil* is the least legible: recessed, covered services, built to be lived in by people who never saw the pipes. The readability rule for Civil is **covered, not hidden**: a channel cover carries its loop's diameter as channel width, shows the fitting silhouette (3.3) on the cover face, and has a sight slot showing the medium. A Civil wreck makes the player spend more light and more time per sentence (Rule question 1), which is its character — not a Pillar 1 exemption.

**Grafts and the crew's hand.** The weld seam is hero #3 (3.5): a raised Hull Grey bead, never hidden, drawn on floor and wall wherever two sections join, in straight, corner and T pieces with a *fresh* (bright, clean) and *old* (dulled, scuffed) state. A graft keeps its donor dialect in full; the seam is the only thing that is the crew's. Field repairs are a fourth, deliberately ungrammatical vocabulary: bolted patch plates — the plate body in Graphite lit, since Hull Grey is never a plane (4.1), with Hull Grey only on the bolt rhythm and the bead at the edge — that stay on the tile grid (3.3 — human geometry is orthogonal) but **straddle panel boundaries**, breaking the fastener rhythm of whatever dialect they sit on. A patch reads as "we did this in a hurry" because it is on-grid and off-rhythm. A section the crew cut away leaves a seam with a cap plate and nothing beyond it.

**Wear and age.** Each dialect wears along its own axis so age reads without rust (6.2): Foundry scuffs to bare metal at its rounded corners (Hull Grey edges); Lattice loses bolts, leaving empty holes in the truss; Civil's covers hang open. A derelict's age is dust depth (matte Graphite mid) and the dried trails of 2.6.

**The web.** 3.3 fixes its shape. This section fixes where it may be: only in **breached geometry** (6.4), entering from the hull side, passing behind panels and emerging beyond them, never aligned to the grid, never on the home-ship unless a hull section is itself breached, never as a hero shape (3.5), never drawn unlit (principle 1).

### 6.2 Texture philosophy: unshaded albedo, lit by the engine

**Decision.** Stylised flat-albedo HD pixel art with normal (and specular) maps, lit entirely by Godot's 2D lights. The sprite stores *what a surface is*; the engine decides how it looks under the lights that are actually on.

**Why — the Pillar 1 argument.** Hand-shaded pixel art bakes a light direction into every tile. The beam has another direction, and it moves. A pipe shaded from the top-left while the player's beam comes from the right is a lie drawn into the ground truth — principle 1 violated at the pixel level — and the player would learn, correctly, that the drawing and the light disagree, which is the one lesson this game must never teach by accident. Baked shading also fails 2.0: one tileset must serve every lighting state, from 10 % stable warmth to a single hard cone at 0 % ambient, and a fixed shadow direction cannot. Unshaded albedo plus normals is the only option in which the 2.x states are literally "the same tiles under different light".

**Alternatives rejected.** *Hand-shaded pixel art with baked light* — the above; beautiful, and dishonest here. *PBR-ish photographic texture* — surface noise fights silhouette legibility (Section 1 constraint notes), it is not pixel art, and it drags in a texturing pipeline a solo developer does not need. *Flat pixel with no normals* — honest but blind: the beam would reveal shape only by colour, and a round pipe would not read as round under a moving light; normals are what let the light *draw*.

**Rules.**
- *Baked shading allowed:* direction-agnostic only — crevice darkening (seams, under lips, inside grates) at no more than one value step; and the Void Steel contour on hero objects and characters. No baked shading on floor tiles. No cast shadows in any sprite.
- *Large planes:* at most two Graphite values plus normal-map detail. Detail lives at **edges and fasteners** — which is where 3.3's five dialect axes already live.
- *Normal maps required on* wall faces, pipes and fittings, machinery, seams, characters; *optional on* floors. *Specular:* metals tight and bright, lagging none, glass a single tight highlight, the web's iridescence is a shader band (4.1), never a texture. Generation tooling is the technical-artist's; Section 8 records formats and sheet pairing.
- *Permitted materials* (all cold, all outside the flesh band under warm light, 4.4): steel and alloy in Graphite; bare metal in Hull Grey, thin elements only (4.1); glass as Void Steel with a highlight; white-grey lagging and insulation; black rubber hose and gasket; cold grey-blue fabric and padding (Civil interiors); frost in Coolant Ice.
- *Excluded materials*, because they sit in or beside the flesh band (hue 5–30°, mid saturation) and warm light pushes them in: rust and oxidation of any kind; copper, bronze and brass; wood; leather; cardboard; cream or tan paper; tan canvas; warm plastics. There is no rust in this world — a vacuum has none, and the palette forbids it. Paper White is UI-only (4.6): in-world documents are dark log slates or backlit Graphite screens with Tungsten-fixture text, never paper.
- *Weathering vocabulary* in place of rust: scuff-to-bare (Hull Grey), scorch (Void Steel), dust (matte Graphite mid), frost (Coolant Ice), and the dried trail stains of 2.6.
- *Proportions* (exact sizes to Section 8): floor tiles square in screen space; a wall shows a top cap and a front face **1.5 tiles tall** — taller than the 1.1-tile crewmate of 5.1, so a figure never breaks the wall's top edge; the pipe band — "eye height" in 3.3 — runs at 0.4–0.8 of the face height (≈ 0.6–1.2 tiles), which is never hidden by floor props and crosses the crewmate at head-and-shoulder height.

### 6.3 The room kit: what one kit must contain

Environments are assembled from hand-authored kits — by hand at MVP, by the generator from VS-A. A **kit** is one dialect's tiles and props plus the shared sets every dialect uses. Defining the minimum keeps the "largest asset sink" (concept) bounded.

**Shared sets (dialect-free, authored once).**
- Pipes and fittings: 5 loops × (straight, elbow, T, terminal) × 2 orientations. Pipe identity — diameter, fitting, flow pattern — is constant across dialects (3.3); only the mount varies, so pipes are one set.
- Failure trails: 5 behaviours × 3 sizes, plus the propagation animation along a seam (2.4, 2.8).
- Weld seams: straight, corner, T × fresh, old (6.1).
- Web intrusion: 6–8 branch pieces, 2 entry breaks, 3 endpoints; iridescence by shader.
- Hidden-entity footprints and Tell VFX (5.5, 2.7).
- Deployable lights (omnidirectional, flickering — 3.2) and the generic salvage crate.

**Per dialect.**
- Floor: base, one variant (grate or plated), threshold.
- Wall: straight face, inner corner, outer corner, top cap, end cap.
- Door: frame in the dialect's port shape (round / octagonal / rectangular, 3.3) and door leaf in four states — closed, open, cut, welded (6.5).
- Pipe mount: bracket / truss / channel cover, in straight and elbow.
- Light fixture: one housing; the unlit state is the same sprite with the light off (free).
- Machinery nodes: one per flowing loop (four — O₂, power, coolant, fuel), each with its visible moving medium (hero #2, 3.5), plus a **hull plate** module for integrity, which per 3.3 is the hull itself and has no medium and no machine. All five are the salvageable modules and carry their donor dialect onto the home-ship.
- Props: eight to ten (crate, locker, bunk, table, console, chair, tool rack, canister, cover plate).
- Storytelling set: five or six "interrupted" props (meal tray, tool left in a fitting, open locker, log slate, half-cut hatch) and the **empty suit** in three poses (slumped, prone, seated), visor opaque or frosted, never a body.
- Breached variants: no more than 30 % of wall and floor pieces get a torn, buckled or holed state.

**Counts.** Roughly 70–90 unique pieces per dialect plus about 120 shared. Sized to the concept's tiers: **MVP** — Foundry only, handmade, three machinery nodes (O₂, heat/coolant, power), one or two breached pieces, no procgen: ≈ 50 dialect pieces + ≈ 60 shared. **VS-A** — the full Foundry kit under the generator. **Alpha** — three dialects.

**The three-piece test.** A 5×5 room in which only the wall face, the door frame and the pipe mount are swapped must be identifiable as its dialect at a glance, in greyscale, in one beam. These are the highest-frequency pieces, so Gestalt similarity does the work, and all five 3.3 axes live on them: corner radius, fastener rhythm and panel ratio on the wall face; port shape on the door; routing on the mount. Everything else — props, fixtures, floors — may be shared across dialects with a trim swap. If a kit needs more than three unique pieces to read, its wall face is under-designed; add rhythm to the face, not pieces to the kit.

**Visual rules for the generator** (implemented by the game-designer and technical-artist; the art-director owns the rule, not the code).
1. *Continuity:* pipes run continuously between rooms and never dead-end without a terminal fitting — the eye must be able to follow symptom to source (3.0).
2. *One trail per source:* a trail begins at a fitting or seam and runs along the floor or seam line; no orphan trails.
3. *Hero cap:* at most one moving medium per room that is not a machine room (3.5).
4. *One residual light per wreck* — the visible unknown of 2.5, in Tungsten or a loop hue, never iridescent.
5. *Seams only at joins;* grafts keep donor dialect.
6. *Intrusion only in breached rooms*, entering from the hull side.
7. *The pipe band stays clear:* no prop may overlap the 0.4–0.8 face band.
8. *A clear path* at least one tile wide through every room, so a breach can be run from (2.8).

### 6.4 Prop density by area type

Density is measured, not felt: **wall props per 10 tiles of front face**, **floor coverage** (non-walkable set dressing as a percentage of floor), **moving media** (machinery with visible medium — the only permitted motion in human spaces apart from loop flow patterns and trails, 3.5), and **trails**. Two caps apply everywhere: a trail must out-contrast whatever it crosses (3.5), so no floor prop may exceed Graphite lit against the floor where a trail could run; and nothing decorative moves.

| Area | Wall props / 10 | Floor coverage | Moving media | Trails | Rhythm | Why |
| --- | --- | --- | --- | --- | --- | --- |
| Cryo bay | 1–2 (the pods *are* the wall) | ≤ 10 % | 1 (the leak and its condensation) | 1 | strictly ordered pod row; disorder only in which pods stand open | 2.2 — the slowest state; one line for the eye to follow |
| Corridor | ≤ 1 | ≤ 5 % | 0 | as the graph dictates | the pipe band is the only dense element | corridors are sentences (Rule); escape routes must stay readable (2.8) |
| Machine room | 3–4 | 20–30 % | 3–6 | originate here | a clear cross-path; machinery on the walls, media facing the room | where state lives (principle 2, hero #2) |
| Breached section (derelict) | as its original type | as its original type, **displaced** | 0 (dead) | 2–4, dried | ≤ 30 % breached pieces; props thrown off-grid — the one place human objects may sit off-grid; 1–2 web entries breaking the grid (3.3) | density by disorder, not count |
| Web exterior | 0 | — | 0 | 0 | filament count per cone set by ring (4.3); the pod, the wreck silhouette, one residual light | 2.5 — smallness; no backdrop |
| Home-ship (grafted rooms) | by room type | room type + 10 % crew clutter over time | by room type | live, in loop hue | seams at every join; foreign dialects under one warm light | 2.3 — lived-in, layered |

**Breached geometry is not the breach state.** 2.0 forbids any *state* from demanding its own tiles and 2.8 makes breach a lighting-and-trail event. So "breached section" above is **permanent dead-ship geometry** placed by the hand or the generator. A *live* breach on the home-ship swaps no tiles: damage renders as integrity trails on unchanged geometry (hairline → crack → bent rib, the three trail sizes of 3.3), plus the sweep, the venting and the propagating trails of 2.8. If a hull module is destroyed outright, that is the module's own destroyed state — a sprite state on one object, not a tileset — and whether that exists is the game-designer's.

### 6.5 Environmental storytelling: the verdict grammar

No text, no cutscenes (Anti-Pillars) and **no bodies** (4.4; 5.1). Each derelict delivers a verdict on what happened to its crew in three sentences the beam can read, and the player must spend light to assemble them.

**Vocabulary.**
1. *Dried trails — the mechanical cause* (2.6). The dead ship's own grammar names the failure: a frost fan at a lagged elbow is coolant; a radial scorch at a junction box is a power arc; an ember-edged stain is fuel; branching hairlines through ribs is integrity. O₂ failure is the one that leaves no trail — mist evaporates — so it is read from absence: a dead scrubber, undisturbed dust, and the hatches.
2. *Hatches and pods — the crew's verb.* Door leaf states: **closed**, **open** (powered down neatly), **cut** (ragged, scorched — someone went through in a hurry, and the scorch side says from which direction), **welded** (a Hull Grey bead across the frame — someone wanted it to stay shut). Pod states: closed-and-frosted (occupied; a suited silhouette behind the frost — never skin), open-and-dark, smashed. A machine room welded shut from inside says *they hid*; every airlock open and every suit missing says *they left*.
3. *Interrupted props — what they were doing.* A meal tray, a tool left in a fitting, a locker open with its suit gone, a log slate on a table. One or two per room, never a diorama.
4. *What is missing — the absence.* Suits, pods, a module cut out cleanly by someone with the right tool. Empty suits in the three poses of 6.3, visors opaque or frosted, are the only "people" ever found; where a body should be there is a suit or nothing. The absence is the horror (Pandorum, Section 1) and it is also the Pillar 5 fiction's doorway — flagged for the narrative-director, who owns who was in the pods and why.

**Composition rules.**
- *Spend light to read it* (Rule question 1): the three sentences are distributed — the cause in one room, the crew's verb in another, the absence in a third — never all inside one cone.
- *Eye-line:* trail → source → hatch. A trail always leads to the fitting that failed, and the fitting is near the door the crew used.
- *One cause, one verb, one absence per wreck*: the "three sentences" rule, which the generator can place as three tagged slots.
- *Dialect flavours the telling, not the story:* Lattice crews work outside, so their verbs are airlocks; Civil crews hide behind covers, so their verbs are covers and welds; Foundry crews fix things, so their verbs are patches and seams. What actually happened to each crew is the narrative-director's and the generator's.
- *The home-ship tells its own story* in the same grammar: seams, foreign-dialect modules, cap plates where sections were cut away, and the crew's scars (5.3). The re-wake variant of 2.2 is this grammar applied to the cryo bay.

### 6.6 Legibility: characters against environments

The brief requires a character silhouette to read against the environment texture density this section defines, under the lighting of Sections 1–2. Rules, then the test.

1. **Frequency separation.** Environment planes are low-frequency: at most two Graphite values with detail only at edges and fasteners (6.2). Characters are closed mid-frequency shapes with a Void Steel contour (5.1). No environment tile carries a contour; hero objects (machinery media, seams) may. Contour means *figure*.
2. **The reserved composition.** No prop arrangement may form a compact closed-circle-on-rectangle stack at crewmate scale — no round canister on a crate, no round gauge over a square console at body height. 3.2 reserves that shape. Static circles exist in the grammar (Foundry ports, ring clamps, 3.5) but at door scale or fitting scale, never at helmet scale. Round floor features (drains, hatches) must be smaller than a third of the helmet or larger than twice it.
3. **Round and moving.** The helmet is round and it moves, which 3.5 reserves for hero shapes. This is consistent, not an exception: the crewmate is the *source* of hero #1, the beam, and is read as part of it — a moving circle with a cone attached. The crewmate is not a sixth hero; the hierarchy stands.
4. **Floors are the character's ground.** In 3/4 oblique the figure overlaps floor more than wall, so floors are the strictest planes: two values, pattern contrast of one step, no normal-map detail that produces hard highlights under a beam, grate patterns at a pitch that never matches helmet or pack width.
5. **Value and light.** Inside the beam, suit plates (Graphite lit) and a lit wall face can approach the same value; separation comes from the contour, from the normal-mapped helmet sphere's highlight, and from the bright beam origin at the hand — which is why the hand point is always inside the figure's silhouette. Horror against filament: both use web grammar, intentionally — a horror resolving out of filaments *is* the Tell (2.7). They separate by motion and by mass: a filament never has a core, and a horror's branches converge.
6. **The test.** Run the 3.1 silhouette test on a **lit frame**, not on the sheet: one Tungsten beam of each of the four player shapes from the crewmate's hand, at each ring's ambient floor (4.3), against each dialect's wall face and floor, with the crewmate at the cone centre and again at the soft rim. Pass at 25 % means helmet-and-pack read as figure; at 50 % the band count reads; in the achromatic pass (4.5) all of the above hold. Run it before colour (3.1) and again under lighting, on every new wall face and floor tile — the environment is tested against the character, not only the character against the environment.

---

**Consistency notes.** Eight tensions were resolved across these two sections. (1) *Anchor vs class (c)*: Section 1 and 4.4 say sprites never shift, 3.2 says class (c) replaces the sprite; 5.0 reconciles them as "never edited, only wholly substituted" — human or horror, never a blend — which also removes transition art from the budget. (2) *Hidden-entity footprints vs loop trails*: Pillar 4(a)'s "frost" and "hull stress" are loop-trail words in 3.3; 5.5 draws footprints in the web's grammar (branching, iridescent, hue-less) so a footprint can never be read as a coolant or integrity failure, and leaves the occlusion question open as 3.2 does. (3) *Helmet bands in Hull Grey*: legal as thin elements (4.1) but 4.4's named near-flesh case; 5.1 binds them to the 4.4 sampling test with Graphite lit as the fallback. (4) *Breached geometry vs the breach state*: 2.0 forbids per-state tilesets; 6.4 splits permanent dead-ship breached geometry (kit variants) from a live breach (trails and light on unchanged tiles). (5) *Warm materials vs the flesh band*: rust, copper, wood, leather, paper and tan fabrics all sit in or beside 4.4's band under Tungsten; 6.2 excludes them and replaces rust with a cold weathering vocabulary; Paper White stays UI-only per 4.6. (6) *Horror rigs vs pixel purity and 3.1*: rigs satisfy 3.3's never-repeat rule and the budget, at the cost of rotation blur; 5.4 states the requirements (grid-snapped look, stepped playback, three test poses) and routes implementation to the technical-artist. (7) *The helmet as round-and-moving*: 3.5 reserves that for heroes; 6.6 reads the crewmate as the source of hero #1 rather than a sixth hero. (8) *Mirrored facings swap the tool hand*: accepted, because the hand is not a channel. Two smaller corrections were made against the locked text while drafting: integrity has no machinery node, because 3.3 makes the hull itself the loop with no medium — the kit carries four nodes plus a hull-plate module; and Hull Grey is never a plane (4.1), so patch plates and scar patches are Graphite bodies with Hull Grey only on bead and bolt rhythm. Downstream flags: **Section 7** inherits the Foundry dialect for the HUD frame (3.4), should draw tool glyphs from the hand-tool sprites, and must show no sanity meter — the only sanity signal is the world and the crewmate fringe; **Section 8** owns tile size, frame dimensions, the crewmate's pixel height, normal/specular sheet pairing and formats, and naming for overlay, segment and variant sprites; **technical-artist**: the fringe shader in the unlit pass with no light contribution, rig stepping and rotation quantisation, normal/specular generation, the iridescence band, and whether rime is an overlay or a shader mask; **narrative-director**: no bodies anywhere in the sane world, pod occupants are suited silhouettes, dialect cultures are offered as visual implications only; **game-designer**: scar count and earning, vital-overlay thresholds (the overlays are a channel of truth and must track simulation state exactly), hidden-entity occlusion, destroyed hull-module states, and the generator's eight visual rules; **audio-director**: nothing is specified here beyond the mood the visuals set — a stiff, sealed crew and a ship that creaks before it fails.

## 7. UI/HUD Visual Direction

### 7.0 Governing rule: the HUD is a suit, the ship is an instrument

**Who produced the number decides where it is drawn.** Three layers, one question each.

| Layer | Who produced the information | Drawn where | Lit? | Can a reading lie (class b)? | Anchor? |
| --- | --- | --- | --- | --- | --- |
| **In-world instruments** | the ship, the pod, a derelict | sprites in the world layer: gauges, sight-glasses, dials on panels, the pod dash, the chart-table console | yes — unreadable until lit | yes; this is class (b)'s primary home ("from the gauge to the sprite") | the instrument housing is physical geometry (P1); its reading is not |
| **Suit layer** | the player's own suit | screen-space `CanvasLayer`, 3.4 frame/reading grammar, 4.6 palette | no — the visor is backlit | readings yes, frame never | the frame is the "HUD frame" anchor of Section 1's Corollary |
| **Chart table (meta)** | nobody in the fiction: keybinds, menus, settings, the web map | screen-space, 2.11 register | no | **never** — it is not an instrument (R-4) | whole layer is honest by definition |

**The deciding rule, applied with Section 1's three questions.** (1) *Does it make the player spend light?* Anything the ship reports about itself goes in the world so it must be lit to read — a dead loop's gauge is dark until the beam finds it, and a running loop's gauge is faintly lit by its own emission (Principle 3: a lit gauge is a lantern). (2) *Is it in the world or on top of it?* The suit is the one thing that is always with the player and always powered, so suit telemetry is the only *state* permitted on the screen-space layer. (3) *If the player is lied to here, does the lie leak?* Readings in both the world and the suit layer are drawn in the thin, light, moving "reading" register of 3.4, so every lie-capable mark is visually declared as such, and the stage-1 gleam (4.4) has a glass to sit on. Meta elements are drawn in a register that never carries the gleam because they never carry a lie.

**The screen-space set (HR-3).** At all times — home-ship, pod transit and boarding alike — the suit layer carries: suit O₂ (*breath*), suit integrity, suit-sensed local temperature and pressure (*warmth*), the equipped tool and its state (*the hand*), beam and tool charge, and the Alarm warning mark. The suit does not know it is in a pod, so nothing changes in transit (the pod's own heading reticle and fuel sight-glass are lit by its running lights, 2.5, and live in the world). Which of these readings can lie is the game-designer's; the visual rule is that any of them *may* be drawn lying, so all of them sit in the reading register. **The honest cross-check for a suit reading is a crewmate's body**: 5.3 makes rime, visor fog and the stagger animation anchors that are true on every client, so a player who doubts their own O₂ gauge can look at a friend's visor in their beam — "you look cold" costs only light.

**Not on the suit layer, ever:**
- **No sanity meter and no Attention indicator — anywhere, including under accessibility settings** (user decision; Principle 3's design test; 4.6). Attention is the world and the audio: filaments resolving at the cone edge (2.7), dust changing direction, and the audio-director's Tells (HR-9). The only sanity that reaches screen-space is the violet gleam on a *reading* that is lying. The accessibility consequence is recorded in 7.6.
- **No ship state.** "COOLANT LOW" is the banner Principle 2 rejected. The ship tells you its state along its pipes; the suit tells you about you.
- **No minimap in play.** A minimap is a fog-of-war mask drawn small (Principle 1's rejected alternative). The web map (7.4) is a chart you open.

**Interaction prompts — the one apparent exception.** Principle 2 rejected a world-space icon layer, but that rejection was about *state*. A prompt shows a keybind and a verb (`E · Cut`), never the object's condition, so it is meta, not an instrument. It is drawn on the HUD layer in the chart-table register, on its own opaque Ground plate (HR-2), anchored to the object's screen position, and **is drawn only when the object is inside the lit area** (the technical-artist sets the illumination threshold). Nothing is prompted in the dark, so the prompt teaches the rule rather than breaking it. Whether the *action itself* is possible on an unlit object is the game-designer's call; this is a drawing rule only.

**The inspect view (= HR-14's mirror card).** In-world gauges carry no numerals at game-camera distance (3.1: readable by needle angle and zone band alone). When the player needs precision, or is on gamepad (HR-14: proximity focus, one contextual action), they *inspect* the instrument: a screen-space card at HUD size, framed in the ship's own bezel dialect, carrying the same reading with numerals in the instrument face, Paper White (it is screen-space, so 6.2's Tungsten-text rule for in-world screens does not apply). Two rules: the card is drawn only for an instrument that is currently lit (whether inspection is *permitted* unlit is the game-designer's; visually there is nothing to magnify), and the card is **the same instrument**, not a second one — it lies exactly when the gauge on the wall lies, with the same HR-5 tri-modal tell. Pillar 4's "second instrument" check is a *different* gauge elsewhere on the ship, never the card.

**Composition (user decision 1).** The beam cone is the brightest shape in frame and the eye's home (3.5), so the HUD is small, cool and at the edges — with one deliberate exception for what the player must read while running:

- **Critical strip, centre-bottom.** Suit O₂ as a dial arc *with its numeral beside it* (HR-12), and the Alarm warning mark. One opaque Ground plate, no wider than 12 % of frame width and no taller than 5 % of frame height at 100 % UI scale (at 1080p ≈ 230 × 54 px), sitting on the bottom safe margin; at a higher UI scale the plate grows with the integer step and the cap is knowingly exceeded — the player's explicit accessibility trade (7.1). Why centre: the camera holds the crewmate near frame centre, so the strip sits in the near-periphery of the beam origin and is readable in breach without an eye movement to a corner (2.8's hardest test: frightened, still reading). *Accepted and bounded cost:* a south-facing beam crosses this plate, and because the plate is opaque (HR-1/HR-2) the cone passes behind it; the size caps keep the cone's hard outer edge and its hero read (3.5 #1) legible past the plate, and nothing else on the HUD may ever sit between the player and the lower frame edge.
- **Full suit panel, bottom-left.** Integrity, warmth, charge, the detailed O₂ reading. The lower corners hold a glanceable reading in peripheral vision without pulling the eye off the cone, and keeping the top of the frame clear leaves wall front-faces — where the pipe grammar lives (3.3, 6.2) — unobstructed.
- **Hand rail, bottom-right** (tool glyphs with keybinds); **crew strip, top-left** (7.5); **prompts at their objects.**
- Everything inside a 5 % safe margin. Every reading and every glyph sits on an opaque Ground plate (`#1C2128`), never over live world pixels and never on a bezel fill (HR-2). The HUD's non-Ground pixel area at rest is capped at roughly 1 % of the frame (tunable): the HUD layer is *exempt* from 3.5's "beam brightest" rule by living on its own layer (HR-1), and the area cap is the discipline that keeps the exemption from being abused.

**The HUD's dialect.** The frame is the home-ship's (3.4), which 3.3 and 6.1 make Foundry. It inherits Foundry's **fastener rhythm** (big rivets, wide pitch — one rivet per bezel corner, never a texture) and **panel ratio** (squat, wide plates), but not its corner radius: 3.4 reserves UI curves for dial arcs and needles, so every HUD corner is a chamfer. The HUD is Foundry rendered in the flat UI grammar — recorded as a resolved tension in 7.6.

**Alternatives considered.**
- *Fully diegetic (Dead Space model).* Rejected. At 3/4 oblique HD pixel scale a suit-mounted readout is two or three art pixels; the player would spend light to read their own breath; and the "HUD frame" anchor would have nothing to anchor.
- *Fully screen-space (survival-genre default): every loop as a bar or icon.* Rejected. It violates Principle 2, erases spend-light, and gives class (b) the dullest possible target — an icon flipping — instead of a needle disagreeing with the body of the ship.
- *All vitals in one corner cluster* (the first draft). Rejected by the user in favour of the centre-bottom critical strip: in breach the eye cannot afford the corner trip.
- *Pod HUD as a separate screen-space mode.* Rejected; the pod is a vehicle with a lit dash.

### 7.1 Typography

**Personality: stamped nameplate and engraved dial.** Industrial, humanist, un-ornamented, slightly condensed. Type on the HUD should look punched into a Foundry plate (3.3): capitals for frame labels, numerals that look machined. Never futuristic-angular (a cleaner civilisation than this salvage hull), never a terminal/monospace "hacker" face (that says computer; this HUD says instrument).

**Two faces, not one.**

| Face | Used for | Why this face |
| --- | --- | --- |
| **Instrument face** — a bitmap pixel font on the art grid | suit readings, frame labels, inspect-card numerals, keybind glyphs, map labels | the HUD is the ship's dialect (3.4) and must sit on the ship's pixel grid; an antialiased vector glyph beside a pixel bezel reads as a browser overlay; the integer grid makes a needle's one-pixel wander legible (7.3) |
| **Reading face** — a vector humanist sans, rendered by Godot's font system | lore fragments, recovered logs, settings, long menu copy, tooltips | long-form reading in a pixel font is fatiguing and resists accessibility scaling; the chart table is outside the lit world (2.11) and may use the one face that is not pixel-bound |

Both faces must have **tabular (fixed-width) numerals, a slashed or dotted zero, and distinct 1 / l / I**. Tabular numerals are load-bearing: a reading that changes — or lies — must never shift the frame around it. The frame is an anchor; a proportional "1" to "7" change that nudges a bezel would move the one thing the Corollary says never moves.

**Weight.** The instrument face ships in one weight. Readings are *light and thin*; frame labels are the same face in Hull Grey, which reads as engraved because of colour, not weight. The reading face uses regular and one heavier weight for headings; no light weights (thin vector text on Ground at these sizes falls under the contrast floor).

**Size hierarchy and floors (provisional until the accessibility-specialist sets the tier; HR-12).** Expressed in screen pixels first, because floors are about the eye, then converted to art pixels at an assumed integer scale. *Assumption, to be confirmed in Section 8:* a 640 × 360 base, scaled 3× at 1080p and 4× at 1440p. The screen-pixel floors are the contract; the art-pixel column re-derives if the base changes. Two observations for Section 8, which owns the choice: 640 × 360 lands on an integer at 1080p, 1440p and 2160p (3×, 4×, 6×), while 960 × 540 gives 2.67× at 1440p, which would soften every pixel in the game, not just the type. If "HD pixel art" is taken to mean sprites authored at or near 1× native, the instrument face becomes a crisp bitmap font at the floor sizes and the two-face split holds for its other reasons. Typography does not decide the base.

| Level | Where | 1080p (screen px, cap height) | 1440p | Art px at 3× / 4× |
| --- | --- | --- | --- | --- |
| Reading numerals | critical strip, suit panel, inspect card, map distance | ≥ 21 | ≥ 28 | 7 |
| Label | gauge names, hand-rail tool names, crew names | ≥ 15 | ≥ 20 | 5 |
| Caption floor | the smallest glyph anywhere | 15 | 20 | 5 |
| Menu body (reading face) | settings, logs | ≥ 20 px font size | ≥ 26 | — |
| Lore long-form | recovered logs read in full | ≥ 22 | ≥ 29 | — |
| Section heading | menu groups | ≥ 24 | ≥ 32 | — |
| Title | screen names | ≥ 32 | ≥ 42 | — |

*Alternative considered:* the ux-designer's HR-12 proposal of ≥ 24 px for critical readings and ≥ 18 px body at 1080p. The user kept 21 / 15 provisionally; the tier is the accessibility-specialist's to set, and whichever number wins, the 3×/4× art-pixel derivation is one integer step away (8 and 6 art px). The two proposals may not share a unit: this table is in **cap height**, and 24 / 18 reads as *font size* (cap height ≈ 70 % of it, so 24 px font ≈ 17 px caps) — under which 21 / 15 is the larger floor, not the smaller. The accessibility-specialist reconciles units before choosing. The remaining HR-12 items are adopted unchanged: **needle and glyph strokes ≥ 2 screen px** (one art pixel at 3× is 3 px, so the floor is met by construction), **text ≥ 4.5 : 1 and glyphs ≥ 3 : 1** against their own Ground plate, **UI scale 100–200 % in integer steps** of the art scale for the instrument face, and **every critical needle carries a numeral beside it**.

Nothing falls below the caption floor; if a label does not fit at 5 art px cap height, the frame is too small, not the type. Reading-face line length 45–75 characters. Instrument-face letterspacing one art pixel, never tighter.

**UI scale.** Integer steps for the instrument face; a separate continuous reading-face scale. The frame grows with the integer step and the beam does not, so a player who scales the HUD up spends frame area the beam would otherwise own — their accessibility trade, made explicit. Typography, scale and accessibility settings are anchors (HR-6): no state of the game may change them.

**Alternatives considered.** *One pixel font everywhere* — rejected: lore and settings become hard to read, and every size step is an integer jump. *One vector font everywhere* — rejected: a smooth face on pixel bezels breaks 3.4, and sub-pixel edges hide a one-pixel needle wander. *A pixel font with several hand-drawn sizes* — viable but costly for a solo developer; revisit only if the two-face split fails a readability test.

### 7.2 Iconography

**Flat silhouettes, one colour, on the art grid.** No outlines, no gradients, no illustration, no drop shadows. A flat stamped silhouette is what a plate on this ship would carry; an outlined icon adds a second line colour and reads as web UI; an illustrated icon is a picture of a thing rather than the thing's grammar.

**Loop icons are the fitting table.** Per 3.4, loop glyphs are the loop's fitting silhouette plus its flow mark, so shape carries meaning with hue removed (4.7):

| Loop | Glyph | Flow mark inside the glyph |
| --- | --- | --- |
| O₂ | thin ring clamp | three close-pitched chevrons |
| Power | square junction box | a dash-pulse (two dashes) |
| Coolant | padded collar | one wide chevron |
| Fuel | bolted flange | a diagonal hazard band |
| Integrity | rivet line (three rivets) | a hairline |

**Hue follows 4.2, not the icon.** A glyph is drawn in its loop hue only while that loop is *running*; a dead or unpowered loop's glyph is Graphite lit; a *failed* loop's glyph stays in hue because the failure is a trail, and the Alarm warning mark appears beside it, never over it. Icons never swap to express state. State is a reading — a needle, a fill level, a number — inside or beside the glyph. The icon set stays tiny and the lie stays where 3.4 puts it: in the reading, never in the frame or the glyph.

**Tool glyphs are drawn from the hand-tool sprites.** 5.1 makes the tool layer an anchor drawn once and never styled per dialect; the hand-rail glyph for each tool (cutter, welder, scanner, suppressant, deployable light, carried module) is that sprite's profile silhouette reduced to the 16 px grid — the same shape that breaks the crewmate's outline (3.2), so the rail teaches the in-world read and there is no second tool vocabulary to author. Tools are Paper White; the equipped tool is Tungsten fixture; a tool's *state* (charge, suppressant remaining) is a reading beside it.

**The warning mark — defined once.** A chamfered square filled Alarm Red carrying a Paper White exclamation stroke (≈ 3.8 : 1, a glyph not text — HR-4). It lives on the critical strip (7.0) and fires in two cases, both beside a reading: a **critical suit reading** (the mark sits beside that reading's glyph), and a **ship alarm the suit receives** — the suit's alarm receiver is itself an instrument, so this too is a reading and 4.6's "only on a failing instrument" holds. It pulses in beacon phase (7.3) and is always paired with the reading's own shape change and the audio-director's cue. It is the HUD glyph HR-8 relies on when the world's sweep goes static under reduced motion. Alarm Red appears nowhere else on the UI.

**Two canonical sizes.** 16 × 16 art px (HUD, inspect card) and 24 × 24 (menus, map legend), each with a 1-art-px stroke and 1-px inner clearance so the glyph survives beside a bezel. Nothing smaller; a 12-px glyph cannot hold a flow mark.

**Keybind and gamepad glyphs.** Keys are chamfered-square keycaps, Ground interior, Graphite lit border, Paper White letter in the instrument face. Gamepad face buttons are drawn the **same way** — a chamfered keycap carrying the button's letter (A/B/X/Y) — not as circles. The one grammar exception is a controller whose buttons are named by symbols (the PlayStation set): the symbol goes inside the keycap at glyph size, a platform convention the player already owns, not a shape this UI chose. 3.4 reserves the only UI curves for dial arcs and needles, and a round button glyph would be the one closed circle on screen that is not an instrument; a keycap in the ship's dialect also lets keyboard and gamepad prompts share one component. (The ux-designer owns which prompts exist and when they switch device; this is the visual grammar only.)

**Focus is a shape (HR-11).** Keyboard and gamepad focus, menu selection and map selection are all the same component: a **Tungsten bracket** (left bar plus chamfer corners) *and* a one-art-px thickening of the focused bezel. Paper White against Tungsten fixture is ≈ 1.2 : 1, so a colour-only selection is invisible to an achromat; the geometry change is the primary channel and the Tungsten is the accent.

**Alternatives considered.** *Abstract symbols* (a drop for coolant, a flame for fuel) — rejected: a second vocabulary with no shape backup the player already knows. *Outlined icons with hue fill* — rejected: two colours per icon spends hue, and the outline competes with the bezel. *Bespoke tool icons* — rejected by the Section 6 flag: the sprite already exists and is an anchor.

### 7.3 UI animation: three motions

The HUD is a set of physical instruments drawn by a human hand, so it moves the way instruments move. There are exactly three motion signatures, and the player must be able to tell them apart without colour, because 2.7 already uses one of them as an honest Tell.

| Signature | What it means | Motion | Rests? | Palette |
| --- | --- | --- | --- | --- |
| **Settle** (honest change) | the ship changed; the instrument caught up | damped spring: ease-out over 250–400 ms, overshoot ≤ 5 %, then still | yes | none |
| **Tremor** (honest stress — the Tell, 2.7) | something physical is loading the hull or the loop | ± 1 art px, irregular, 6–10 Hz, around a *true* value | returns to rest when the stress ends | none — no gleam, because nothing is lying |
| **Glide** (the lie — class b, stage 1) | the reading has detached from the ship | smooth, uncaused drift at a slow constant rate (0.5–2 ° s⁻¹ on a dial arc), with a slow micro-wander (< 0.5 Hz) that never fully rests | **never** | violet gleam on the glass or needle edge — fringe `#B07BE6`, never core (4.4, HR-5) |

**The honest/lying distinction is cause, mass and rest.** Honest motion starts with a cause the player could have seen in the world (a valve turned, a seam cracked), has mass (it eases and settles) and then rests. A tremor is honest because it is *fast, small and centred* — the shaking of a real needle on a stressed ship — and it stops when the stress stops. A glide is the one motion no real instrument makes: slow, smooth, directional, with nothing in the ship to explain it, never quite at rest. A player who learns "needles that wander are lying; needles that shake are warning" has learned Pillar 2 and Pillar 4 from the same gauge.

**This is HR-5's motion channel.** HR-5 requires every class-(b) reading on the suit layer or an inspect card to carry a motion signature and a sound signature besides the gleam, so the lie survives hue removal and audio removal in turn. The glide *is* the motion signature; HR-5's own example ("a needle that settles with none of the micro-jitter honest needles show") was illustrative, and the bible's rule runs the other way — honest needles are still at rest and shake only under real stress; the liar never rests. The sound signature is the audio-director's (HR-5, HR-9); the visual requirement is only that it exists and begins no later than the glide.

**Digital readouts.** Honest numerals **tick** — one step per real change, never a tween. A lying readout ticks *continuously*, every few hundred milliseconds, as it tracks the glide, or holds stale while the ship audibly and visibly changes. The mechanism is always honest; only the value lies.

**What never moves (HR-6, HR-13).** The HUD `CanvasLayer` is excluded from camera shake, from world glow, ambient and occlusion, and from the per-client hallucination filter. The frame's geometry, position and scale; input mapping; typography, UI scale and accessibility settings; the menu, pause and settings layers (no violet or flesh on them, ever); the speaker indicator and text chat — all anchors. No HUD element shakes, blurs, shows chromatic aberration or changes scale under any state. Readings move inside a still frame; that stillness is what makes stage-1 drift readable rather than merely unsettling (3.4).

**Pulse and flash (HR-13).** The warning mark pulses **in phase with the beacon sweep** (2–3 s period, ~50 % duty), never faster — one rhythm for alarm everywhere. Nothing on the UI changes luminance faster than 3 Hz (the tremor is positional, not luminous, so it is exempt — flagged for the accessibility-specialist to confirm alongside 2.8's sweep). No full-frame flashes, ever.

**Reduced motion (HR-8).** The toggle must not strip the backbone 4.7 leans on. On the UI: flow marks inside loop glyphs become static chevrons and dashes (direction kept); the warning mark stops pulsing and holds solid, with its sound and the reading's shape change carrying the alarm; *settle* becomes a step; *glide* slows but survives — what remains of the lie is **never-rest** (the needle is always a pixel from where it was a second ago) and the continuous digit tick, so a lying instrument stays distinguishable from an honest one at rest; *tremor* is the one signature that may need a static substitute (a held stress mark on the bezel), routed to the accessibility-specialist. World-side reduced motion (the sweep, trail propagation) is Section 2's and the technical-artist's; HR-8 forbids suppressing trail propagation.

**Menus and chart table.** Cuts and short fades (80–120 ms), no slides, no bounces, no parallax. The focus bracket snaps with a 100 ms ease-out. The web map's filaments drift at ≤ 1 art px s⁻¹ in the current ring only (2.11); everything human on the map is static. Hold-to-confirm (HR-14) fills a **linear bar with a needle-mark**, never a radial wipe (3.4). Sober means almost nothing moves, and what moves is the web.

**Alternatives considered.** *Sci-fi scanline/glitch for the lie* — rejected: a screen effect, not an instrument behaviour; it says "the HUD is broken" when the pillar says "the gauge is wrong". *Red flashing for alarm* — rejected on photosensitivity and 2.8. *Juicy UI (bounce, squash, scale-pop)* — rejected: it puts energy into the HUD, and 2.9 says sanity must never be read off the screen's energy.

### 7.4 The web map and menus

**Register.** 2.11 defines the mood; this defines the furniture. Ground `#1C2128` full-bleed; one chamfered Graphite-lit border; one rivet mark per corner (Foundry rhythm, 7.0). Text Paper White, instrument face for labels and distances, reading face for descriptions. The map and menus are a human instrument that never lies (R-4): nothing on them is a reading, so nothing on them can glide or gleam.

**Web map components.**

| Element | Drawing | Note |
| --- | --- | --- |
| Depth rings | concentric steps of Ground, each ≈ 4 % darker toward centre; the inner ring's centre reaches Void Steel | the only "deeper = darker" rule carried into UI (4.6); composition pulls inward (2.11) |
| Filaments | Hull Grey, 1 art px, branching, no right angles, density rising inward | the current ring's filaments drift; others are static |
| Home-ship | a Tungsten fixture point with a 1-px rectilinear Foundry silhouette | the only warm mark on the chart |
| Known wrecks | Graphite-lit rectilinear silhouettes in their donor dialect (3.3) | a visited wreck gains its dialect; an unvisited one is a plain rectangle |
| The one visible unknown | a single 1-art-px square tick inside the wreck silhouette in Tungsten or its loop hue (no circle — 3.4), **only once seen from the pod** (2.5) | the chart records what the crew lit; it never reveals |
| Routes | dashed rectilinear lines, Hull Grey; the planned sortie in Tungsten | human geometry over inhuman geometry (3.4) |
| Crewmates | their beam-shape glyph (7.5) at their position | identity is beam shape, nowhere a colour; this is one of the three permitted ways to find a crewmate (7.5) |
| Selection / focus | the 7.2 bracket around the wreck, label and distance beside it; on gamepad the bracket **snaps node to node** (HR-14), never free-cursors over off-grid filaments | never a filled block |

The map **never** shows iridescence, violet, flesh, Attention or sanity.

**Ship schematic (working proposal).** If the game-designer wants a graph view of the composite ship, it is an **in-world chart-table console** opened into the inspect-card register — lit, instrumented, lie-capable — not a chart-table (meta) screen. The ship reporting on itself is always an instrument.

**Menu components.** Left navigation rail; content on the right; one column of chamfered-rectangle list items with Ground interiors and 1-px Graphite-lit borders. Selection and focus are the 7.2 bracket plus bezel thickening (HR-11), never a Tungsten fill — a warm plane on screen would be a lantern the UI lit for free. Hover turns the label Tungsten; disabled is Graphite lit (≈ 2.8 : 1, inactive and exempt by design). Sliders are a bar with a needle-mark; toggles a chamfered switch; destructive actions hold-to-confirm on a linear bar (HR-14); **dial arcs appear only on instruments**, never as menu decoration.

**Pause.** A right-third side panel on Ground over the frozen world. **The world is not dimmed, blurred or desaturated** under a menu: dimming is a mask on the world (Principle 1), blur is a lens lie, and desaturation is the sanity convention 4.4 rejected. The world simply stops, lit exactly as it was.

**Loss.** No death screen (2.10). The last light goes out, then the cryo-bay lamp. Any post-loss UI (logs recovered, knowledge kept) is read at the chart table after re-wake, as furniture in the room the player knows.

**Accessibility surface (visual side; the ux-designer and `design/accessibility-requirements.md` own the requirement).**
- UI scale in integer steps; reading-face scale continuous (7.1).
- HUD opacity with a floor (recommend 70 % — below it readings fall under 4.5 : 1 on their plates).
- Pattern strength: thickens flow marks on glyphs and the direction shapes 4.7 relies on. **No global hue remap** (HR-10): colour-blind support is shape and pattern emphasis plus the per-element alternates already in 4.7; a full-frame daltonization shader would move the sane palette and give the violet leak nothing stable to leak against.
- Reduced motion per 7.3 (HR-8).
- **Low-vision ambient floor (HR-15, user decision 6).** A per-client option raising the scene's ambient floor to **the measured ring floor plus a bounded number of steps** (provisional: floor + 2–3 steps of the 8-bit frame, set from 8.12 step 7; always below the 10 % stable state of 2.3 so "powered" still reads as the warmest state — this replaces the earlier absolute "≤ 6 %", which 8.11 flag 1 showed is not a meaningful unit before the floor is measured *(Amended per AD-ART-BIBLE review 2026-10-25)*), revealing **hull geometry only** — the web, horrors and filaments remain visible only in direct light, exactly as before — and touching **no simulation**: Attention is unchanged, because the raised ambient is not a light the crew is running. This is a bounded exception to Principle 1's "rooms are unlit", decided for accessibility and recorded with its costs in 7.6. The technical-artist owns the rendering (raised ambient on hull albedo that does not count as illumination for web-grammar content).
- **Never an accessibility option:** the technical director's true-value debug overlay (R-7). An honest overlay on top of a lying HUD would make Pillar 4 switchable off, and the pillar is the game.
- **No Attention indicator** under any setting (user decision 2; 7.6).

### 7.5 Co-op: crewmates on the HUD

*Designed now so the single-player HUD leaves room for it; required at VS-B, not MVP.*

**The crew strip** sits top-left: one tile per crewmate, stacked. Each tile is a chamfered frame (anchor) holding:

| Part | Register | Content |
| --- | --- | --- |
| **Identity glyph** | anchor — the human channel (Pillar 1, 3.2) | the crewmate's beam shape (wide cone, narrow cone, split, fan) with their **trim-band count** (1–4, the Hull Grey bands of 5.1), in Tungsten fixture |
| **Name** | anchor | instrument face, Paper White |
| **Voice (speaker indicator)** | anchor — the human channel is never faked (HR-6) | while their voice is active the identity glyph's interior fills, outline → solid, in the same Tungsten fixture; nothing else changes. The audio-director owns voice itself. *Alternative considered:* stepping the glyph to Tungsten beam core `#FFE2A8` — rejected by the user; beam core is not a 4.6 colour and the palette stays locked |
| **Location** | reading | a small glyph: Foundry hull (home-ship), pod silhouette, grey wreck rectangle — one of the three permitted ways to find a crewmate (below) |
| **Beam state** | reading | glyph lit = beam on, Graphite = beam off (emission is exposure, carried to the strip) |
| **Vitals** | — | **none by default** (user decision 5). The honest vitals are on the body: rime, visor fog and stagger are anchors on every client (5.3), so the way to check a crewmate's air is to light their visor. If the game-designer later relays suit telemetry, it enters the tile as a reading, lie-capable like any other |

**Sanity: a world mark with redundancy, no roster flag (user decision 5; HR-7).** Nothing on the tile ever shows a crewmate's sanity — 4.6 forbids violet as a UI colour and Pillar 4 places the class (c) tell in the world. The tell is tri-modal and lives on the crewmate as seen by a saner viewer:
1. **Hue** — the violet fringe `#B07BE6` exactly as 5.5 specifies: 2 px at sprite resolution, outside the Void Steel contour, along the full outer silhouette and both edges of the beam cone, per-viewer presentation rendered in the unlit pass, casting no light and raising no Attention.
2. **Motion** — the fringe **breathes** at the 4–6 s period 5.5 already permits. Under this decision breathing is no longer optional: it is the motion channel that carries the tell for a deutan or tritan viewer who sees the violet as blue. Never a flicker (the deployable light's word, 3.2) and never a tempo change (2.9).
3. **Sound** — a **P1 proximity cue owned by the audio-director**: physically real, so it plays on every client, unlike the fringe, which is not there. The fringe itself stays silent, as 5.5 says; the cue is a separate honest thing near the person.

*Consequential edits to 5.5 — applied in 5.5:* breathing is mandatory, and the sound clause distinguishes the silent fringe from the P1 proximity cue. *(Amended per AD-ART-BIBLE review 2026-10-25)* *Open question, routed to game-designer and audio-director:* a P1 sound audible on every client is audible to the low-sanity player too, and 5.3 says there is no self-tell — whether the cue is heard by its own source, and what it is in fiction, is theirs.

**Why beam shape and not colour.** 3.2 already spent the decision: four player hues would eat the loop palette, and a suit colour is a sprite property a hallucination could plausibly fake, whereas the beam is a light the engine renders honestly. The tile's identity glyph is therefore the same shape the player sees in the world. Under class (c) — a crewmate wholly substituted by a horror rig (5.0) — the tile still shows a human beam glyph beside a horror carrying that very beam. Silhouette rule, light rule, voice and tile agree; any one suffices (5.5).

**Crew loss on the strip.** When a crewmate dies, their identity glyph goes **dark** — Tungsten to Graphite — and the frame stays. The tile is 2.10 in miniature: the last light going out, honestly, with no red and no violet. Visual recommendation: the dark tile stays in the strip for the rest of the sortie, so the loss is carried as an absence the frame still holds; when it is removed is a ux-designer/game-designer condition.

**Finding a crewmate — closed (user decision 4).** There are exactly three ways to know where a crewmate is, and no fourth will be added: the **web map** (7.4), the **crew-strip location glyph**, and **their beam sweeping the dark**. No off-screen arrows, no edge marks, no voice-gated locators of any kind. The first draft offered a voice-gated edge mark as a least-damaging compromise; the user closed it: a locator of any kind shows something that is not lit (Principle 1) and is a world-space icon in disguise (Principle 2), and a beam crossing the black is a better in-fiction locator than any arrow.

**In-world identity.** No name tags over heads by default — a floating label is a world-space icon and a lit label in the dark breaks Principle 1. In the world a crewmate *is* their beam shape and band count (3.2, 5.1). A hold-to-reveal name is the ux-designer's call; if offered, it is drawn in the chart-table register and only for crewmates inside the viewer's light.

**Text chat (HR-6 anchor).** Drawn in the chart-table register: reading face, Paper White on an opaque Ground plate, at the left edge below the crew strip, fading in on a new line and never scrolling the world. It is never lit, never a reading, and carries no gleam — it is the human channel in letters.

**Pings (R-5, HR-6) — not decided here.** HR-6 lists pings among the human-channel anchors and R-5 asks for a world-space ping for deaf/HoH and no-mic crews. Whether pings exist is the ux-designer's and game-designer's; decision 4 bounds them: a ping may **mark a thing**, never **locate a crewmate**, and whether it may be drawn on an unlit object is the Principle 1 question those owners must answer. If it exists it is a chamfered Tungsten-fixture mark in the chart-table register, is an anchor, and is never a hero shape (3.5).

### 7.6 Art direction vs readability: conflicts and how each was decided

Approximate WCAG contrast ratios of the 4.6 palette on the two surfaces text could sit on (from the published hexes; re-check after Section 8 and the lighting spike):

| Colour | on Ground `#1C2128` | on Graphite lit `#5C6670` (bezel) | Verdict |
| --- | --- | --- | --- |
| Paper White `#DDE4EA` | ≈ 12.6 : 1 | ≈ 4.6 : 1 | body text on Ground; marginal on bezel |
| Hull Grey `#8C949C` | ≈ 5.3 : 1 | ≈ 1.9 : 1 | labels on Ground only |
| Tungsten fixture `#FFCC66` | ≈ 10.8 : 1 | ≈ 3.9 : 1 | accent text on Ground; large-only on bezel |
| Paper White vs Tungsten fixture | ≈ 1.2 : 1 | — | colour alone cannot mark selection (HR-11) |
| Alarm Red `#E0102A` | ≈ 3.3 : 1 | ≈ 1.2 : 1 | **fails body text everywhere**; mark only (HR-4) |
| Paper White on Alarm Red | ≈ 3.8 : 1 | — | the warning-mark stroke; glyph floor met |
| O₂ Cyan / Power Amber / Coolant Ice | ≈ 9–12 : 1 | — | glyphs and readings fine on Ground |
| Fuel Ember `#F06A28` | ≈ 5.2 : 1 | — | glyph fine; not small text |
| Graphite lit `#5C6670` (as disabled text) | ≈ 2.8 : 1 | — | inactive only; exempt by design |
| Violet fringe `#B07BE6` | ≈ 5.3 : 1 | — | the gleam. The ux check's ≈ 2.9 : 1 is violet *core* `#8A3FD1`, which is why HR-5 says fringe, never core |

1. **Dark HUD on a dark game vs legibility.** Decided for legibility by surface, not brightness: *every reading and glyph sits on an opaque Ground plate* (HR-2). Bezels are borders, never fills behind type. Hull Grey on a lit-Graphite bezel fails at ≈ 1.9 : 1; on Ground it passes at ≈ 5.3 : 1.
2. **Alarm Red as text.** Fails body-text contrast on every UI surface. Decided: Alarm Red is a *mark* — the 7.2 glyph beside a failing reading — and never a word (HR-4). This also keeps red rare (4.1).
3. **"The beam is the brightest thing in frame" (3.5) vs Paper White text.** Decided by *layer and area*, not by dimming text: the HUD layer is exempt from 3.5 and from world glow, ambient and occlusion (HR-1); opaque plates mean text never sits over the beam core, which is how HR-1's test (≥ 4.5 : 1 at 0 % ambient and over the beam core) is met by construction; the ~1 % non-Ground area cap and the critical strip's size caps keep the exemption from eroding hero #1.
4. **Corner vitals vs breach readability.** Decided by the user for a centre-bottom critical strip (O₂ with numeral, warning mark), full panel at the edge. Accepted cost: a south-facing beam passes behind an opaque plate; bounded by the 12 % × 5 % size caps (7.0).
5. **No numerals on in-world gauges (3.1) vs precision and gamepad reach.** Decided for both: needle-and-zone in the world, numerals on the inspect card — drawn only for a lit instrument, lying exactly when the wall gauge lies, and serving as HR-14's mirror card (7.0).
6. **Pixel font vs accessibility scaling.** Decided by splitting the problem: pixel face for instruments (integer steps), vector face for reading (continuous). Text floors 21 / 15 px at 1080p stand provisionally over the ux-designer's 24 / 18; the accessibility-specialist sets the tier (7.1).
7. **Interaction prompts vs Principle 2's rejected world-icon layer.** Decided by content and gating: keybind and verb only, never state, drawn only for lit objects (7.0).
8. **A pause menu that reads the world vs dimming the world.** Decided for Principle 1: the world freezes undimmed (7.4).
9. **Lying needle drift vs the honest Tell drift of 2.7.** Decided by motion signature: tremor is honest, glide is the lie, and only the lie carries the gleam; the glide is HR-5's motion channel (7.3).
10. **Foundry's rounded corners (3.3) vs 3.4's chamfer-only UI.** Decided for 3.4: the HUD inherits Foundry's fastener rhythm and panel ratio, not its corner radius, because UI curves are reserved for dial arcs and needles (7.0).
11. **Co-op identity readability vs the hue budget.** Already decided in 3.2 for beam shape; the HUD inherits it (7.5).
12. **Off-screen crewmate locators.** **Decided by the user: none of any kind.** Map, crew-strip location glyph and beam sweep only (7.5).
13. **Crewmate sanity on the roster vs the human channel.** Decided by the user for a world mark with redundancy — fringe + breathing + P1 proximity sound — and no roster flag (7.5). The two consequential edits to 5.5 are applied in 5.5. *(Amended per AD-ART-BIBLE review 2026-10-25)*
14. **Voice-activity cue colour.** Decided by the user inside the locked 4.6 palette: Tungsten fixture fill; beam core not added (7.5).
15. **Attention indicator for accessibility.** **Decided by the user: no Attention indicator anywhere, including under accessibility settings.** Attention is world and audio only. *Consequence, stated plainly:* for a low-vision player who cannot resolve filaments and dust at the cone edge, the audio-director's audio-first Tells (HR-9) become the **only** non-visual channel for Attention; for a deaf or hard-of-hearing player, the visual world is the only channel, and the HR-15 ambient floor does not help, because filaments stay visible only in direct light. Routed to the accessibility-specialist to assess, with the ux-designer's off-by-default filament-count glyph recorded as the alternative the user declined.
16. **Principle 1 ("rooms are unlit") vs low vision.** Decided for a *bounded* exception (HR-15): a per-client ambient floor of the measured ring floor + 2–3 steps (7.4; formerly "≤ 6 %" *(Amended per AD-ART-BIBLE review 2026-10-25)*), hull geometry only, no simulation effect. Costs accepted: the floor exceeds pod transit (0–1 %), boarding (1–3 %) and the dark home-ship (2 %), so the 2.1 mood ladder compresses for that player; "powered stable" at 10 % still reads warmest. The web-grammar content stays beam-only, so Pillar 3's "looking is spending" survives (7.4).
17. **The stage-1 gleam as the only leak vs colour-blind and low-vision players.** 4.0 principle 2 says hue is never the only channel; the first draft leaned on the gleam. Decided for tri-modal lies (HR-5) and tri-modal class-(c) tells (HR-7): hue, motion, sound, each sufficient (7.3, 7.5).

**Hand-offs.**
- *ux-designer*: which prompts exist and when they switch device; hold-to-reveal names; whether pings exist (bounded by decision 4); the gamepad-complete vs keyboard-only flow enumeration at MVP (HR-14; text chat the only permitted keyboard-only item in the core loop); the accessibility options list in `design/accessibility-requirements.md`; menu flows.
- *technical-artist / ui-programmer*: confirm the base resolution and integer scaling at 1080p/1440p/2160p; the illumination threshold that enables prompt and inspect-card drawing; a single Godot `Theme` carrying the 4.6 palette, both faces and nine-slice pixel bezels; the HUD `CanvasLayer` excluded from camera shake, glow, ambient, occlusion and the per-client hallucination filter, with readings fed from perceived state and frames from nothing at all; the HR-15 hull-only ambient raise; mirror-card feasibility; and, with Section 8, the **trail-contrast number** the ux check's first row asked for — world-side, quantifying 3.5's "a trail must out-contrast whatever it crosses" so breach stays readable through red light (2.8).
- *game-designer*: which suit readings exist and which can lie; whether unlit objects can be acted on or inspected; whether crew vitals are ever relayed; whether a ship-schematic console exists; from the ux check — is "heat" the coolant loop or a room-temperature P1 consequence; does a boarder receive home-ship telemetry on a derelict, and is it an instrument or an anchor; are world sounds (including phantom-entity audio) ground truth or hallucinable; are scanner and other hand-tool readings instruments (Pillar 4(d), R-3); is carried weight an honest physical check or an instrument numeral; is the P1 proximity cue heard by its own source (7.5).
- *audio-director*: HR-9 — all five loops with distinct running and failing sounds, every Tell with an audio component starting no later than its visual; HR-5 and HR-7 sound signatures; the P1 proximity cue of decision 5; the tick, tremor, glide and beacon-phase pulse are defined here as motion only; whether any has a sound is theirs; and, with the game-designer, R-6 — if audio can lie, it needs a leak equivalent to violet, and if it cannot, captions must name sources precisely (blocking for deaf/HoH support).
- *accessibility-specialist*: set the HR-12 tier (21 / 15 vs 24 / 18); confirm HR-13's 3 Hz ceiling, the positional tremor and 2.8's sweep against photosensitivity; the reduced-motion substitute for tremor; the Attention-channel gap of item 15.
- *Closed since the ux check* *(the creative-director hand-off on the two 5.5 edits is dropped — they are applied in 5.5; AD-ART-BIBLE review 2026-10-25)*: R-2 (sprite animation as an anchor) is answered by 5.0 and 5.3; the art-director items (typography face under integer scale, HR-7 fringe motion look) are answered in 7.1 and 7.5.

**Open questions.** Whether suit readings are lie-capable at all in MVP (class (b) is MVP's only hallucination; the concept's "gauge to the sprite" suggests ship instruments first). Whether the inspect card exists in MVP or whether 3.1's needle-and-zone read is sufficient for two handmade derelicts on keyboard and mouse.

### 7.7 Hard requirements (from the UX alignment check, as adopted)

Numbered as the ux-designer numbered them; test lines kept; annotations in italics record where a user decision or a Section 7 rule modifies the requirement.

1. **HR-1 HUD exemption.** The HUD renders on its own layer, excluded from world glow, ambient, occlusion and 3.5's "beam brightest" rule; it never dims or fades with world light, Attention or sanity. *Test:* at 0 % ambient and over the beam core, HUD text ≥ 4.5 : 1 against its own ground. *Met by HR-2's opaque plates; the ~1 % area cap of 7.0 is the art-direction discipline that accompanies the exemption.*
2. **HR-2 Opaque ground.** Every HUD reading and text sits on an opaque UI-ground plate (`#1C2128`); never over live world pixels or on the bezel. *Test:* no Hull Grey glyph on `#5C6670`.
3. **HR-3 Screen-space set.** Suit O₂, suit integrity, equipped tool and state, the Alarm warning mark, and suit-sensed local temperature/pressure are screen-space at all times, pod transit and boarding included. *Laid out per 7.0: critical strip centre-bottom, full panel bottom-left.*
4. **HR-4 Alarm Red is never text.** UI: warning-mark glyph only (≥ 3 : 1), paired with shape and sound. *Glyph defined in 7.2.*
5. **HR-5 Stage-1 lie is tri-modal.** Any class-(b) reading on the suit layer or an inspect card carries, besides the fringe-violet gleam (`#B07BE6`, never core), one defined motion signature and one sound signature. Subtle allowed, absent not. *Test:* with hue removed and with audio removed in turn, a trained tester identifies the lying instrument above chance. *The motion signature is the glide of 7.3; HR-5's example was illustrative. Sound → audio-director.*
6. **HR-6 Anchor list extended.** Never shift under hallucination: HUD frame geometry, position and scale; input mapping; UI typography, scale and accessibility settings; menu, pause and settings layers (no violet or flesh, ever); speaker indicator, text chat, pings; haptics driven by P1 events. *Pings: whether they exist is open and bounded by decision 4 (7.5); if they exist they are anchors.*
7. **HR-7 Class-(c) victim tell is tri-modal.** The low-sanity crewmate mark carries motion in the fringe (outside sprite and beam pixels) and an audible P1 cue in proximity, besides violet. *Test:* under achromatopsia simulation a tester identifies the below-threshold crewmate before contact. *Adopted per decision 5: 5.5's fringe, breathing made mandatory, P1 proximity cue → audio-director; both 5.5 edits applied in 5.5.* *(Amended per AD-ART-BIBLE review 2026-10-25)*
8. **HR-8 Reduced motion preserves the backbone.** Flow direction remains as static chevrons and dashes; the Alarm sweep becomes static mark + sound + HUD glyph; trail propagation is never suppressed; HR-5 motion signatures slow but do not vanish. *UI side specified in 7.3 (never-rest and continuous tick survive; tremor substitute → accessibility-specialist).*
9. **HR-9 Loop audio is complete.** All five loops have distinct running and failing sounds; every Tell has an audio component starting no later than its visual. → audio-director. *Under decision 2 this is the only non-visual Attention channel (7.6 item 15).*
10. **HR-10 No global hue remap.** Colour-blind support is shape and pattern emphasis plus per-element alternates in 4.7; no full-frame daltonization shader.
11. **HR-11 Focus is a shape.** Keyboard and gamepad focus changes bezel geometry (bracket, thicker chamfer) in addition to the Tungsten accent. *Component defined in 7.2.*
12. **HR-12 Minimums (provisional; 1080p at 100 %).** *As adopted:* critical readings ≥ 21 px and labels ≥ 15 px **cap height** (user decision 3; the ux-designer's 24 / 18 recorded as the alternative — note that 24 / 18 reads as font size, under which 21 px cap height is the *larger* floor, so the accessibility-specialist reconciles units before setting the tier); needle and glyph stroke ≥ 2 px; text ≥ 4.5 : 1, glyphs ≥ 3 : 1; UI scale 100–200 % in integer steps for the bitmap face; every critical needle has a numeral beside it.
13. **HR-13 HUD flicker ceiling.** No HUD element flashes above 3 Hz; no HUD shake, blur, chromatic aberration or scale change under any state. *The 7.3 tremor is positional, not luminous — flagged for the accessibility-specialist to confirm.*
14. **HR-14 Gamepad completeness.** Enumerate gamepad-complete vs keyboard-only flows at MVP (text chat the only permitted keyboard-only item in the core loop). Diegetic instruments: proximity focus, one contextual action, a mirror card at HUD size carrying the same reading and the HR-5 tells. Web map: snap-to-node. Destructive actions: hold-to-confirm. *The mirror card is the inspect view of 7.0; snap-to-node is the 7.2 bracket jumping node to node; hold-to-confirm is a linear bar, never a radial wipe (7.3). Enumeration → ux-designer.*
15. **HR-15 Ambient-floor option is bounded.** The low-vision ambient option raises the floor to the measured ring floor + 2–3 steps (provisional, per 7.4 and 8.12 step 7; below the 10 % stable state *(Amended per AD-ART-BIBLE review 2026-10-25)*), reveals hull geometry only (web, horrors and filaments still only in direct light), does not touch the Attention simulation, and is per-client. *Adopted per decision 6; costs recorded in 7.6 item 16; rendering → technical-artist.*

## 8. Asset Standards

### 8.0 Governing principle

**Everything drawn lives on one pixel grid; light and shadow do not; nothing is exported that the engine must repair.** The sprite stores what a surface *is* (6.2) and the engine decides how it looks, so this section's job is to fix the grid, the sheets, the names and the tests, and then get out of the renderer's way. Two earlier rules do most of the deciding: 2.0 (no state may demand its own tileset or sprite variant) keeps the sheet count low, and 6.2 (unshaded albedo, normal and specular maps, engine-lit) makes every drawn asset a *pair or set* of sheets rather than one image. Where an art preference and a technical limit may disagree, this section records the preference and the **IDEAL** explicitly, leaves a **CONSTRAINT** slot for the technical-artist, and names the fallback the art-director will accept — the placeholder's "record both and the trade-off chosen" is honoured in 8.10.

The 3D vocabulary of the placeholder is mapped to 2D equivalents: *poly budget* → unique textures and draw batches per frame; *texture memory* → VRAM of the uncompressed sheet set; *material slots* → shader materials (fringe, iridescence, medium fill, rig stepping); plus the one budget Section 1 already names as first to cut — simultaneous normal-mapped light sources.

### 8.1 Base resolution

**Decision: 640×360 base, integer-scaled — 2× at 720p, 3× at 1080p, 4× at 1440p, 6× at 2160p.** This is the base 7.1 assumed for its typography table and asked Section 8 to confirm; it is confirmed, so the art-pixel column of 7.1 (7 and 5 px cap heights) stands as written and nothing upstream moves.

**Why this base and not the others.**

| Option | 1080p | 1440p | 2160p | Verdict |
| --- | --- | --- | --- | --- |
| **640×360** | 3× | 4× | 6× | The only common base that divides 720, 1080, 1440 and 2160 by an integer. Every Steam monitor gets square, unblurred pixels. |
| 960×540 | 2× | **2.667×** | 4× | Twice the detail per tile, four times the drawing area per piece, and 1440p — a common PC monitor — gets non-integer pixels (uneven pixel widths, shimmer on movement) or black bars; 7.1 makes the same observation. Art throughput is the concept's named scope risk; 4× area per asset is the wrong direction. |
| 1×-native "HD" (sprites drawn at 1080p density) | 1× | 1.333× | 2× | Not pixel art — the concept's Art Style line says pixel art. ~9× the pixel area of 640×360 per asset, every non-1080p display scales non-integer anyway, and bilinear-filtered normals on 1×-native sprites read as vector art under the lights. Barotrauma's look, at a cost a solo developer cannot carry. |
| 480×270 (retro) | 4× | **5.333×** | 8× | Fails 1440p and gives too few pixels for the helmet bands (8.2). |

Ultrawide (3440×1440) and Steam Deck (1280×800) are the two non-16:9 cases. The art preference is `aspect = expand` — the viewport widens to ~860 base px at 4× on ultrawide and gains 40 base rows at 2× on the Deck — because more *screen* is not more *information* when the dark is honest (Principle 1): seeing more black is not an advantage. The technical-artist's spike architecture (8.4) starts from `aspect = keep` (letterboxed) because the world lives in a fixed 640×360 `SubViewport`; `expand` is recorded in 8.10 as a post-spike ask, not an A/B item. Either way the HUD anchors to edges per 7.0's layout, so a safe-area margin is assumed rather than a fixed frame (flagged to the ux-designer).

**Camera zoom.** If the game-designer wants zoom, it is **integer steps only**: at 1080p the single step out is 2× (sprites at 67 %), at 1440p and 2160p it is 2× and 3× respectively (sprites at 50 %). 5.6 tier 2 already requires marks to survive 50 %, so the zoomed-out view costs no new art and is tested by the existing gate (8.7). Non-integer zoom is forbidden: it breaks the grid this section exists to protect. The HUD's own integer scale steps (7.1, HR-12) are independent of camera zoom — the frame grows, the world does not.

### 8.2 The pixel-size table

**Tile: 48×48 base px.** The tile is derived from the tightest constraint in the bible, which is the helmet, not the room:

1. 5.1 puts four trim bands on every helmet; 5.6 fixes the minimum feature at 2×2 px and requires the band *count* to survive 50 % scale (zoom is integer per 8.1, so the issue is not sub-pixel loss but 5.6's floor: a 1 px band is below the 2×2 minimum feature and is not guaranteed to read at the zoom-out step). So bands are 2 px with 1 px separators: 4×2 + 3×1 = **11 px**. *(Amended per AD-ART-BIBLE review 2026-10-25)*
2. Add a visor strip of ≥ 4 px (the glint of 5.1 needs a 2×2 feature inside it) and a 1 px crown: helmet ≥ **16 px**.
3. 5.1: helmet ≈ 30 % of figure height → figure ≥ **53 px**.
4. 5.1: crewmate ≈ 1.1 tiles → tile ≥ **48 px**.

Everything else follows. All values are base px; multiply by 3 for 1080p.

| Element | Size (base px) | Derivation / rule |
| --- | --- | --- |
| Floor tile | 48×48 | square in screen space (6.2) |
| Visible field | 13.3 × 7.5 tiles (640×360) | see the framing trade-off below |
| Crewmate figure | 53 tall (55 with contour), ≈ 24 wide at the pack | 1.1 tiles; helmet 16 / torso-and-pack 24 / legs 13 (30/45/25 %, 5.1) |
| Helmet | 16 px circle | 11 px band zone + 4 px visor + 1 px crown |
| Trim bands | 2 px each, 1 px separators | 5.6 min feature; count survives 50 % |
| Contour | 1 px Void Steel, outside the figure | 5.1; the one permitted 1 px element besides band separators |
| Violet fringe clearance | 2 px outside the contour, kept clear in the cell | 5.5 — the fringe is a shader; the cell must leave it room (8.6) |
| Wall front face | 72 px (1.5 tiles) | 6.2; 55 px crewmate incl. contour stays below the cap |
| Wall top cap | **24 px (0.5 tile) — set here, no earlier section fixed it** | makes a wall piece a clean 48×96 (1×2-tile) cell, authored as one piece and sliced into a 48×72 face tile and a 48×24 cap tile so the cap layer can draw above actors (draw-call shape, 8.6) |
| Pipe band | rows 29–58 above the floor line (0.4–0.8 of 72) | 6.2; crewmate shoulders ≈ row 37, helmet rows 37–53, so pipes cross at head-and-shoulder height (5.1) and no prop enters the band (6.3 rule 7) |
| Pipe diameters | O₂ 6 · Power 10 (flat bar) · Coolant 14 (10 core + lagging) · Fuel 18 | 4 px steps so diameters stay distinct at 50 % (3.1 pipe-section test); fuel flange 24, power junction box 20, all inside the 30 px band |
| Weld seam bead | 4 px Hull Grey | thin element (4.1), hero #3 (3.5) |
| Machinery node | 96×96 cell (2×2 tiles incl. cap); moving-medium region ≥ 16×16 | hero #2 must read at 50 % |
| Door frame + leaf | 96×96 cell | 2 tiles wide in every dialect's port shape (3.3) |
| Props | 48×48, 48×96, 96×48 | 1×1, 1×2 tall, 2×1 wide; empty suits use the character cell |
| Crewmate / tool / overlay cell | 80×80, shared origin | fits carry (module above helmet), tool breaking the outline, prone *down* pose, plus margins |
| Horror | core 64×64 or 96×96; segments 32×32, 48×32, 64×32 | height band 1–3× crewmate = 53–159 px; elongation to 6× = 318 px (5.4) |
| Trails | 48×48 / 96×96 / 144×144 | the three sizes of 3.3 and 6.3 |
| Web pieces | 96×96, 144×144, 192×96; endpoints 48×48 | placed off-grid (3.3) |
| Footprint / Tell VFX | 48×48, 4–6 frames | 5.5 |
| HUD glyphs | **16×16** (HUD, inspect card) and **24×24** (menus, map legend) | 7.2's two canonical sizes; 1 px stroke, 1 px inner clearance |
| HUD state marks | **8×8** | a Section 8 addition below 7.2's glyph floor — see below |
| HUD text (instrument face) | 7 px and 5 px cap height | 7.1 / HR-12: 21 and 15 px cap height at 1080p = 7 and 5 art px at 3× |

**Projection convention (3/4 oblique), stated once.** Horizontal surfaces are seen from directly above and vertical faces at full height, so one base pixel of floor depth, one of width and one of height are each one screen pixel — **1:1, no foreshortening**. Floor tiles are therefore 48×48 squares (6.2); a wall's **footprint is 24 px deep** on the floor grid and is drawn as its 24 px top cap, with its 72 px front face hanging below the cap at full height (a wall strip occupies 96 screen rows but only half a tile of floor). **A prop's top cap equals its footprint depth**, drawn above its front face in the same way; **door frames share the wall's 24 px depth** and sit in the wall line. Front faces occlude what lies screen-south of their footprint and sort against actors on the Y-sort layer (8.6); caps never do. This is the camera 6.2's "top cap and front face" assumes, written here so every kit piece is drawn to it. *(Amended per AD-ART-BIBLE review 2026-10-25)*

**Glyph sizes, reconciled with 7.2.** 7.2 fixes two canonical *glyph* sizes — 16×16 for the HUD and inspect card, 24×24 for menus and the map legend — and rules that nothing smaller can hold a flow mark. Section 8 keeps both and adds one cell that is not a glyph in 7.2's sense: the **8×8 state mark**, a presence/absence or lock mark that sits beside 5 px labels and carries no flow mark, no fitting silhouette and no loop hue. **Rule: an 8 px mark never identifies a loop.** A loop is named only by a 16 or 24 px glyph carrying its fitting silhouette, because that silhouette is the colorblind backup 3.4 and 4.7 rely on; the Alarm warning mark of 7.2 is a 16 px glyph, not an 8 px mark. The HUD renders on the same 640×360 grid as the world, integer-scaled, as 7.1 requires for the instrument face: 3.4 calls the HUD a human instrument in the home-ship's dialect, and an instrument drawn at a finer pixel density than the ship it is bolted to would read as a foreign object. The reading face (7.1) is the one deliberate exception — vector text on the UI layer, never in the world.

**The framing trade-off (for the user).** 48 px buys the helmet and costs the frame: 13.3 tiles across and 7.5 down, and once a back wall takes its 2-tile strip, about 5.5 tiles of floor depth are in view. Pillar 2 wants "a frosting elbow seen to worsen from across the room"; a machine room 8 wide × 6 deep does not fully fit at default zoom. The alternative is a **40 px tile** (16×9 tiles exactly; crewmate 44 px; helmet 13 px), which fails the band arithmetic by 3 px unless the bands drop to 1 px separators *and* the visor shrinks to 2 px — i.e. unless 5.6's minimum feature is relaxed for the one mark that carries identity. Mitigations that keep 48: the integer zoom-out step (8.1), and room kits that favour wide-and-shallow volumes, which is already Foundry's "squat, wide" panel ratio (3.3). **Decided (logged): 48 px** — the identity mark is an anchor and the frame is a camera setting; the user chose difficulty over comfort in 7.6 and a tight frame serves dread. The 40 px alternative is recorded in 8.10 as the fallback not taken.

### 8.3 File formats and sheet pairing

**One drawn asset is a set of sheets with one stem.** Every sheet in a set has identical pixel dimensions and an identical cell grid, so any pairing mechanism the technical-artist chooses (`CanvasTexture`, a TileSet atlas source backed by one, or a material) works without per-sheet offsets.

| Suffix | Content | Format | Required on |
| --- | --- | --- | --- |
| `_alb` | albedo — flat, unshaded, palette-locked (6.2) | PNG, RGBA8, binary alpha | everything drawn |
| `_nrm` | normal map, screen-space (Z toward the viewer; flat = 128,128,255) | PNG, RGB8, no alpha | wall faces, pipes and fittings, machinery, seams, characters, web, props with relief; **optional on floors** (6.2), and if present low-frequency only (6.6 rule 4) |
| `_spc` | specular: metals tight and bright, lagging none, glass a single highlight (6.2) | PNG, greyscale L8 (imports at 1 B/px `[UNV-4.3]`; budgeted at 4 B/px worst case) | everything with `_nrm`; the visor glint lives here |
| `_msk` | medium mask — **R** = loop medium region the engine fills with hue and emission; **G** = flow-phase ramp the travelling pattern scrolls along; **B** reserved for the technical-artist (iridescence band region is the candidate) | PNG, RGBA8 | pipes, fittings, machinery media, trails, fixtures, web |

**Why a fourth sheet.** 4.2 says a running loop is its hue, emissive along the pipe, and a dead loop is the same pipe in Graphite with no emission; 2.0 says no state may cost a sprite variant. Baking hue into the albedo breaks one or the other. Two routes were weighed: (a) **two albedo states per piece** — doubles the pipe set (40 → 80), the trail set (fresh/dried, 2.6), machinery media and fixtures, and still cannot show a loop at half power or a failing loop that flickers; (b) **the mask** — one shader, zero extra drawing, continuous states for free (a dimming loop is a Pillar 2 tell no extra art ever paid for), and fixture lit/unlit stays "the same sprite with the light off" as 6.3 promises. **Route (b) is the art preference** and the sheet set is a quartet where media exist. The chevrons and dash marks of 3.3 remain *geometry* in `_alb`/`_nrm` — they are the colorblind shape backup and must be visible when the loop is dead ("pattern static", 4.2; HR-8 in 7.7 requires the same under reduced motion); the mask only drives the travelling glow. The same mechanism serves 7.2's rule that a loop glyph is drawn in hue only while the loop runs. **Decided (logged): route (b).** Implementation shape from the technical-artist: the filled medium renders as an unshaded additive layer (`CanvasItemMaterial.LIGHT_MODE_UNSHADED` `[BIN]`) whose luminance exceeds the glow threshold so glow carries the "lantern" read, with the flow pattern scrolled by UV in the shader rather than by frames (which is why pipes count as single frames in every budget here). **Pipes glow but do not illuminate**: only machinery nodes cast light, as at most 8 non-shadow `PointLight2D` inside the lighting budget of 8.10. Principle 3's "every loop you run is a lantern" is therefore literal at the nodes and visual along the pipes.

**Categories that break the rule, deliberately.**
- **Light textures** (`light_`) are not pixel art. Greyscale 8-bit PNG, soft alpha permitted, authored at 2048×2048 for the four player beam shapes so a 7-tile beam (336 base px) is still sharp at 6× if lights render at window resolution (pass B, 8.4); under pass A the importer's `process/size_limit = 1024` for this category (8.4) is the right ceiling, since a base-resolution light never samples more. 512² for deployable pools, 1024² for fixtures. The hard outer edge of hero #1 (3.5) lives in this alpha: a 1–2 screen-px edge, then the soft interior falloff. Linear filtering, and mipmaps **on under pass B only** (8.4's adopted preset keeps them off under pass A, where a base-resolution light never minifies) — these scale, sprites never do.
- **UI** (`ui_`): albedo only, no normals, no specular, no mask, never lit (4.6; HR-1 in 7.7). Binary alpha. Paper White exists only here.
- **Fonts** (`font_`): the instrument face (7.1) is a bitmap pixel font at 7 px and 5 px cap height, imported with antialiasing off, hinting off, no subpixel positioning, no oversampling, no MSDF. The reading face (7.1) is a vector humanist sans (TTF/OTF) rendered by Godot's font system with antialiasing on — the one non-pixel text in the game, confined to menus, logs and tooltips.
- **Particle textures** (dust in the beam, 2.7/5.5): 4×4 and 8×8 albedo, nearest-filtered; soft alpha not permitted — dust is pixels, not fog.

**The four beam shapes and the other lights — numbers (provisional — confirmed in 8.12 step 2).** Referenced from 3.2. Values are chosen so the four player shapes light roughly equal areas (∝ half-angle × reach²), so no shape dominates; the game-designer tunes from here. Edge width is the hard outer edge of hero #1 (3.5): under pass A it is one base pixel by construction; under pass B it is a 1–2 px edge in the light texture, scaled. The narrow cone and the beacon at 8 tiles exceed what a 2048² light texture covers at 6× (≈ 7.1 tiles); under pass B at 2160p either the texture ceiling rises for those two or their rim softens slightly — accepted either way. *(Amended per AD-ART-BIBLE review 2026-10-25)*

| Light | Half-angle | Reach (tiles / base px) | Outer edge at 3× | Geometry | Shadow-casting |
| --- | --- | --- | --- | --- | --- |
| Wide cone | 30° | 6 / 288 | 3 screen px, then soft interior falloff | single cone | yes |
| Narrow cone | 12° | 8 / 384 | 3 screen px | single cone | yes |
| Split (twin) | 10° each | 7 / 336 | 3 screen px | two cones with axes ±18° off the facing (36° centre-to-centre); the dark wedge between them is ≥ 16° and never closes | yes |
| Broad fan | 60° | 3.5 / 168 | 3 screen px | one short wide cone; width at the rim ≈ 2× its reach | yes |
| Deployable pool (flare, torch, decoy) | 360° | radius 2.5 / 120 | soft — no hard edge; flickers (3.2) | omnidirectional | the nearest in view only (the sixth slot, 8.11 flag 2) |
| Fixture | 360° | radius 3 / 144 | soft — no hard edge; never flickers | static housing | none (8.11 flag 2); the cryo-bay lamp is the exception via the sixth slot (2.2) |
| Alarm beacon | 20° | 8 / 384 | 3 screen px | rotating sweep, 2–3 s period (4.2) | yes |

**Alpha.** Binary (0 or 255) on every drawn sprite, including mist and trails: a soft-alpha mist is not pixel art and it is also the one place a fill could drift toward a hue it has no right to. Soft alpha only on `light_` textures.

**Colour.** sRGB, 8 bits per channel, no embedded ICC or gamma chunk. Albedo uses only the Section 4 hexes plus the two Graphite plane values of 6.2; the palette ships as one file (8.5) and the automated scan in 8.9 enforces it. **The 4.4 sampling test is run on engine screenshots at target scale, never on source PNGs** — Section 4 says the hexes are tuned under the real renderer, and HDR 2D with glow and a tonemapper can move the hue of a lit surface a few degrees, which is exactly the margin 4.4 polices.

**Lossless is a requirement, not a preference.** VRAM-compressed formats work in 4×4 texel blocks and smear hard edges; at 3× every smeared texel is a 3×3 screen smudge. Compression noise in a normal map becomes *lighting* noise under a moving beam, which is the texture noise Section 1 forbids. And the 4.4 test needs exact hexes to survive to the GPU. Cost is quantified in 8.6.

### 8.4 Godot import and project settings

Option names, types and defaults below carry the technical-artist's sourcing legend: `[BIN]` = read from the installed 4.7.2 editor's class dump (existence, types, defaults and enum values only — never behaviour or cost); `[REF]` = the pinned `docs/engine-reference/godot/` (which covers neither 2D texture import nor 2D lighting; the `[BIN]` facts here are worth folding into it); `[UNV-4.3]` = training-era knowledge, used only where a spike test is named alongside it; `NOT SOURCEABLE` = measured by the spike (8.12), never guessed. Note for anyone arriving from Godot 3: texture *filtering* is no longer an import flag — it is a `CanvasItem` property with a project default — so "filter off" is set once at project level and overridden only on the `light_` category.

**Project settings — written explicitly into `project.godot`, never inherited.** Two defaults the pinned reference describes differ from what the installed binary reports — `display/window/stretch/mode` default `disabled` and `aspect` `keep` `[BIN]` against REF's `canvas_items` / `expand`, and `rendering/rendering_device/driver.windows` default `vulkan` `[BIN]` against REF's D3D12 — and `project.godot` currently has no `[display]` block, so the stretch block is written out by hand (8.11, flag 7). Base `display/window/size/viewport_width/height = 640×360` (binary default 1152×648 `[BIN]`); `stretch/mode = canvas_items`; `stretch/scale_mode = integer` (`Window.CONTENT_SCALE_STRETCH_INTEGER = 1` `[BIN]`); `stretch/aspect = keep` for the spike (8.1); `rendering/textures/canvas_textures/default_texture_filter = 0` **NEAREST** — the binary default is `1` = LINEAR `[BIN]`, so the project is blurry until this is set; `rendering/2d/snap/snap_2d_transforms_to_pixel` and `snap_2d_vertices_to_pixel = true` (both default `false` `[BIN]`); `rendering/viewport/hdr_2d = true` and `use_debanding = true` (both default `false` `[BIN]` — the glow the emissive media need, and the dark-floor remedy of 8.11 flag 1); `display/window/hdr/request_hdr_output` stays `false` `[BIN]` through the spike; Shader Baker on (4.5, `[REF]`). The world renders inside a `SubViewport` and the HUD on a `CanvasLayer` in the root viewport (8.11, flag 3), which is how HR-1's exemption from world light and glow is met by construction. `CanvasModulate` `[BIN]` carries the ambient on the world canvas; a HUD on its own canvas is unaffected `[UNV-4.3]`.

**Texture import preset (project Import Defaults for `texture`, then per-folder overrides) — adopted from the technical-artist.** Defaults are `[BIN]`; the *meaning* of each integer (that `compress/mode 0` is Lossless, that `detect_3d/compress_to 0` is Disabled) is `[UNV-4.3]` and is confirmed in the Import dock during the spike (8.12, step 6).

| Option | `_alb`, `_msk`, `vfx_` | `_nrm` | `_spc` | `light_` | `ui_` |
| --- | --- | --- | --- | --- | --- |
| `compress/mode` | lossless (`0` `[UNV-4.3]`) | lossless | lossless | lossless | lossless |
| `mipmaps/generate` | false | false | false | false (see note) | false |
| `process/fix_alpha_border` | true | false | false | true | true |
| `process/premult_alpha` | false | false | false | false | false |
| `process/normal_map_invert_y` | false | **spike-determined** | false | false | false |
| `detect_3d/compress_to` | disabled (a stray 3D use never VRAM-compresses) | disabled | disabled | disabled | disabled |
| `process/size_limit` | 0 | 0 | 0 | 1024 under pass A; 2048 if pass B wins | 0 |
| Filter (project default / per node) | NEAREST | inherits through `CanvasTexture` | inherits | LINEAR permitted on `PointLight2D.texture` only (`[UNV-4.3]`: sampled by the light, not the item) | NEAREST |

Notes. *Light textures:* the art preference of 8.3 (linear, mipmapped, 2048²) applies only where it changes the picture — under pass B; the technical-artist's `mipmaps = false` and 1024 ceiling are the default until then. *CanvasTexture:* `CanvasTexture.texture_filter` defaults to `PARENT_NODE` `[BIN]` and inherits the project NEAREST; set it explicitly to NEAREST only on pages that must never inherit a different parent. *Atlas importer:* `ResourceImporterTextureAtlas` is **never used** — its `trim_alpha_border_from_region = true` `[BIN]` would eat the fringe padding and would pack albedo independently of normal and specular (8.6). *Lossless storage:* `.ctex` lossless is WebP-lossless internally `[UNV-4.3]`, no quality loss either way; `rendering/textures/lossless_compression/force_png` stays `false` `[BIN]`. *Normal-map Y convention:* NOT SOURCEABLE; fixed once in the lighting spike by lighting the crewmate's helmet sphere from each screen direction and checking the highlight sits toward the light — if it does not, the preset sets `process/normal_map_invert_y = true` for `_nrm`; the verified value is written into this table and into the normal-generation tool, and no sheet ever carries a different convention. *Lines:* 4.7 draws `CanvasItem` lines without an antialiasing feather `[REF]`, so hairlines, filaments and seams are always sprites, never runtime line draws.

**Render mode (an art call with a technical price — recorded as IDEAL plus fallback; the spike's pass A/B settles it).** The IDEAL is **sprites on the grid, light off the grid**: every sprite integer-scaled with nearest sampling while lights, occluder shadows and glow render at window resolution. The beam is engine light, not a drawing (6.2's logic applied to hero #1), and its soft interior falloff should be smooth; rendering it at base resolution steps it into 3 px bands at 1080p and makes the light "look drawn", folding hero #1 into the sprite layer. The technical-artist's architecture makes both passes one scene: the root viewport, with the stretch settings above, carries only the HUD `CanvasLayer` and a `SubViewportContainer` (`stretch = true` `[BIN]`); the world sits in a `SubViewport` with `use_hdr_2d`, `use_debanding`, `snap_2d_transforms_to_pixel` and `canvas_item_default_texture_filter = NEAREST` `[BIN]`. **Pass A**: the `SubViewport` at 640×360, scaled by the container — lights at base resolution, every cone edge, shadow edge and rotated segment on the pixel grid, one ninth of the lit pixels. **Pass B**: the `SubViewport` at window size with `size_2d_override = 640×360` and `size_2d_override_stretch = true` `[BIN]` — logical base coordinates, sprites integer-scaled, lights and shadows at window resolution. Whether `Light2D` resolves at the viewport's render size and whether `size_2d_override` behaves this way are both `[UNV-4.3]`; 8.12 step 2 is the test, and the cost difference is the 9× lever of 8.11 flag 4. **The fallback the art-director accepts is pass A**: a 3-px stair-step on the cone edge keeps 3.5's "hard outer edge" and loses only the smoothness of the interior falloff, which debanding partly recovers. Consequence under pass B: horror rig segments rotating under nearest sampling produce 3×3 blocks at an angle — the same pixel density, off the grid. **This is tolerated for horrors and web only** (3.3: the web never aligns to the grid) and **forbidden for anything human-built**, which is never rotated at runtime; if the off-grid look fails review, the technical-artist's remedies are pre-rotated segment sets (24 angles, a tool step) or a base-resolution sub-viewport for horrors alone. Sub-pixel blur is forbidden everywhere; 5.4's quantised rotation (15°) and stepped playback (12 fps) stand.

### 8.5 Naming convention

Consistent with `project.yaml` (`naming.files: snake_case`): lowercase ASCII, underscores, two-digit indices, no version numbers or dates in file names (version control holds history). **The file unit is the sheet or atlas, not the frame**; piece, loop, state and facing names live inside the sheet as Aseprite tags and slices, TileSet tile names and `SpriteFrames` animation names, using the *same* token grammar, so a name means the same thing in the file system, the source file and the engine.

**File grammar.** `<cat>[_<dialect>]_<subject>[_<variant tokens>]_<map>.png`

This grammar deliberately deviates from the project's generic `[category]_[name]_[variant]_[size].[ext]` convention: sizes here are fixed per category (8.2), so the trailing slot carries the **map suffix** instead — which every sheet of a set needs and the generic form has no place for; the order category → name → variants is otherwise preserved, and `size` survives as an ordinary variant token where a category genuinely has sizes (trails, UI glyphs). *(Amended per AD-ART-BIBLE review 2026-10-25)*

Variant tokens, in this fixed order when present: *loop* → *piece / orientation* → *state* → *facing* → *size* → *index*.

| Token class | Values |
| --- | --- |
| `cat` | `env` (floor, wall, door, mount, fixture) · `mach` · `prop` · `story` (interrupted props, empty suits) · `pipe` · `seam` · `trail` · `web` · `char` · `tool` · `ovl` (scar and vital overlays) · `horror` · `vfx` · `light` · `ui` · `font` |
| `dialect` | `foundry` · `lattice` · `civil`; omitted on shared sets (6.3) |
| `loop` | `o2` · `pwr` · `cool` · `fuel` · `hull` |
| piece / orientation | `straight` · `elbow` · `t` · `term` · `h` · `v` · `inner` · `outer` · `cap` · `end` |
| `state` | `closed` · `open` · `cut` · `welded` (doors, 6.5) · `fresh` · `old` (seams, 6.1) · `frosted` · `clearing` (pods) · `breached` · `held` · `use` (tools) |
| `facing` | `s` · `se` · `e` · `ne` · `n` (5.6; mirrored facings are never files) |
| animation loop | `idle` · `walk` · `run` · `work` · `carry` · `stagger` · `down` (5.2) |
| `size` | `s` · `m` · `l` (trails) · `16` · `24` · `08` (UI glyphs and marks, 7.2 / 8.2) |
| `index` | `01`–`99` |
| `map` | `alb` · `nrm` · `spc` · `msk`; absent on `ui_`, `light_`, `font_` |

**Examples.**
`env_foundry_tiles_alb.png` (one kit atlas; tile names inside: `floor_base`, `floor_grate`, `wall_straight`, `wall_inner`, `wall_cap`, `door_round_cut`, `mount_bracket_elbow`) · `pipe_shared_alb.png` with slices `o2_elbow_h`, `fuel_t_v` · `trail_cool_frost_m_alb.png` · `seam_straight_old_alb.png` · `web_branch_03_alb.png` · `char_crew_body_alb.png` with tags `walk_s`, `idle_n`, `down_e` · `tool_cutter_alb.png` with tags `held_s`, `use_s` · `ovl_scar_pack_02_alb.png` with slices `s`, `se`, `e`, `ne`, `n` · `ovl_vital_rime_alb.png` · `horror_t01_alb.png` with slices `core`, `seg_01`…`seg_12` · `vfx_footprint_smear_alb.png` · `light_beam_wide.png`, `light_beam_narrow.png`, `light_beam_split.png`, `light_beam_fan.png`, `light_pool_flare.png` · `ui_glyph_o2_16.png`, `ui_glyph_o2_24.png`, `ui_glyph_warn_16.png`, `ui_mark_lock_08.png`, `ui_frame_gauge.png`, `ui_keycap_base.png` · `font_instrument_7px.png`, `font_instrument_5px.png`, `font_reading.ttf`.

**Placeholders.** Horror types are unnamed until the game-designer names them: `horror_t01`…`t04` are placeholders that are **renamed, never aliased** — a file never carries both.

**Sidecars.** Every sheet exports its Aseprite JSON under the same stem (`char_crew_body.json`): frame rectangles, tags, slices. Anchors are Aseprite slices with pivots inside the same source, named `anchor_<point>_<facing>` — `anchor_hand_s` (beam origin per facing, 5.6), `anchor_pack_s`, `anchor_helmet_s`, `anchor_limb_s` (scar slots, 5.3) — so the per-frame offset table 5.3 asks for is exported, not typed. Source `.aseprite` files keep the stem and live outside the imported asset tree. The palette ships once as `palette_synaptic_sea.gpl` (and an `.aseprite` swatch) containing exactly the Section 4 hexes and the two Graphite plane values.

**One layout per page.** The technical-artist's `layout.json` — one layout shared by every sheet of an atlas page, so the normal, specular and mask are sampled with the albedo's UVs — is the same artefact as this JSON sidecar: the Aseprite export *is* the layout file, emitted once per page, read by the packer and by the engine alike, and never regenerated per sheet. The technical-artist's working names (`crew_body_01_albedo.png`, `beam_<shape>.png`, `horror_<type>_seg_<n>.png`) are superseded by the grammar above; `_alb` / `_nrm` / `_spc` / `_msk` is canonical.

### 8.6 Texture-atlas organisation

**Rule: one atlas set per group of things that are drawn together, lit together and share a material.** Mixing a lit, normal-mapped kit with unlit UI in one atlas costs a material switch per draw; splitting a kit across many small atlases costs batches. Godot 4 has no sprite-atlas packer for non-tile art, so **atlases are authored as atlases**: the kit is one Aseprite file on a 48 px grid, the crewmate is one file of 80×80 cells. Power-of-two is not required; the **cap is 2048×2048** per sheet, with 4096² permitted only if a full Alpha dialect kit with breached variants will not fit 2048² — which the counts in 6.3 say it will.

| Atlas set | Contents | Sheets | Size (ideal) |
| --- | --- | --- | --- |
| `env_<dialect>_tiles` | floor, wall, door, mount, fixture, machinery, props, storytelling set, breached variants for one dialect (6.3) | alb · nrm · spc · msk | 1024² MVP → 2048² Alpha |
| `pipe_shared` + `seam_shared` | 5 loops × 4 pieces × 2 orientations; seams × fresh/old | alb · nrm · spc · msk | 1024² |
| `trail_shared` | 5 behaviours × 3 sizes + propagation frames | alb · msk (nrm/spc only on frost and puddle) | 1024² |
| `web_shared` | branches, entries, endpoints | alb · nrm · spc · msk | 1024² |
| `char_crew_body` | 230 frames at 80×80 (5.7) | alb · nrm · spc | 2048×1024 |
| `tool_shared` | 60 cells | alb · nrm · spc | 1024² |
| `ovl_shared` | 45 scar + ≈ 10 vital cells | alb · nrm · spc | 1024² |
| `horror_t0n` | one per type: core + segments | alb · nrm · spc | 1024² each |
| `vfx_shared` | footprints, Tell, particle textures | alb · msk | 1024² |
| `ui_hud` | frame pieces, keycaps, focus bracket, glyphs at 16 and 24, marks at 8 (7.2) | alb only | 1024² |
| `light_*` | individual textures | greyscale | 2048² beams, 1024² fixtures, 512² pools |

**Cells and margins.** Grid tiles fill their cells edge to edge; the TileSet atlas source's own texture padding handles bleed (technical-artist confirms). Every non-tile cell (characters, tools, overlays, props, horror segments, VFX) keeps a **≥ 4 px clear margin** on all sides: 2 px for the fringe shader to draw in (5.5 puts it outside the contour; the outline shader samples alpha at ±1 and ±2 texels, so 2 px is the technical-artist's hard minimum) and the rest against nearest-sampling bleed on rotated segments — 4 px is confirmed as the standard (8.11, flag 6). The padding must survive import: `ResourceImporterTextureAtlas` is forbidden (8.4), sheets are packed externally once with one layout for every sheet of a page (8.5), and sheet dimensions are multiples of the cell size. UI glyph cells keep 7.2's 1 px inner clearance. Cells are **fixed-size and never trimmed** at export, with the origin (foot-centre for characters) at the same cell coordinate across body, tool and overlay sheets, so an overlay's offset is one vector per frame and a tool's registration is free.

**VRAM (uncompressed, the lossless requirement's price) — two estimates, both recorded.** *Art-side worst case:* per pixel a quartet costs ≈ 13 bytes (alb 4, nrm 4, spc 1, msk 4) and a triplet ≈ 9; MVP (one Foundry kit at 1024², shared sets, crewmate, tools, one horror, VFX, UI, lights) ≈ **100 MB**; full vision (three dialect kits at 2048², four horrors, everything above) ≈ **350 MB**. *Technical-artist's arithmetic* (`MB ≈ cells × cell_px × 12 / 1 048 576`: three RGBA8 sheets at 4 B/px, 2T×2T character cells, +25 % atlas waste, ≤ 20 MB light textures, HDR-2D render targets of 2–3 MB at 640×360 or 22 MB at 1080p, shadow atlas ≤ 16 MB with its texel format NOT SOURCEABLE): full vision at T = 48 ≈ **190 MB**, plus ≈ 35 MB for the `_msk` sheets this section adds ≈ **225 MB**; MVP environment ≈ 18 MB. Both fit any plausible discrete GPU and the project's 4 GB memory ceiling; no min-spec GPU or VRAM figure exists in `project.yaml` (NOT SOURCEABLE). The technical-artist's conclusion is adopted: **VRAM decides nothing here** — lit-pixel count (8.4) and drawing time do. 8.12 step 10 records `RENDERING_INFO_TEXTURE_MEM_USED` and `VIDEO_MEM_USED` `[BIN]` against both numbers.

**Draw-call hygiene (art's share).** No piece is duplicated across atlases "for convenience" — a Foundry locker that also appears in a Civil wreck is a graft (6.1) and is drawn from the Foundry atlas. Unique atlases visible in a typical frame should stay around a dozen (every page is three or four textures and a batch break; batching merges consecutive items that share texture, material, blend mode and light state `[UNV-4.3]`, `rendering/2d/batching/item_buffer_size = 16384` `[BIN]`); if the technical-artist reports batching pressure, the first merge is `pipe_shared` + `seam_shared` + `trail_shared` into one 2048² set, never the kits.

**Draw-call shape (technical-artist's target against the 1000-draws/frame budget; numbers are hypotheses for 8.12 step 5).** Floor `TileMapLayer` ≈ 4 draws (`rendering_quadrant_size = 16` `[BIN]`, one batch per quadrant per texture `[UNV-4.3]`); floor trails and stains 4–20; **one Y-sorted `Node2D`** holding the wall-face `TileMapLayer`, props, actors and horror rigs — the single largest multiplier, 50–300 measured, because Y-sort is known to break quadrant batching `[UNV-4.3]` and is NOT SOURCEABLE as a 4.7.2 number; wall caps on their own `TileMapLayer` with `z_index` above actors ≈ 4; pipes plus the unshaded emissive overlay 10–40; lights 0 extra draws, shadow passes extra; fringe and hallucination ≤ 10; `GPUParticles2D` ≤ 12 live emitters (dust-in-beam ≤ 300 total, mist ≤ 64 per trail, venting ≤ 128 per vent, `amount_ratio` `[BIN]` scaled by distance from the beam); HUD 20–60. Art consequences: **floors never Y-sort** (6.6 rule 4 already makes them the character's ground, not its competitor); wall caps are their own tile layer (8.2); and if the sorted layer exceeds ~300 draws the remedy is per-row `z_index` bands for wall faces, never fewer props.

### 8.7 LOD expectations

**There is no LOD art.** 5.6 fixes the philosophy — silhouette first, marks second, texture never — and 2.0 forbids a second sprite set. Mipmaps are off because a mipmap *is* a blurred LOD chain and the grid is sacred. What "LOD" means in this project is a **set of review gates at three scales**, run on every asset, on which the asset is cut down until it passes rather than compensated with detail:

| Tier | Scale | Must survive | Applies to |
| --- | --- | --- | --- |
| 1 — Silhouette | 25 %, no colour, no contour (3.1) | helmet circle, pack rectangle, tool breaking the outline; branching and asymmetry on horrors; pipe diameter and fitting with hue removed; dialect at the three-piece test (6.3) | every character, hero object, pipe piece, wall face, door, mount |
| 2 — Marks | 100 % and 50 % (the integer zoom-out step, 8.1) | band count, scar present, tool type, rime and visor fog, the fringe; the moving medium of a machinery node; trail behaviour distinct from the four others; seam fresh vs old | characters, overlays, machinery, trails, seams |
| 3 — Surface | 100 % only; may be lost | fabric seams, plate edges, glint detail, which scar, fastener rhythm detail | everything |

Rules carried from 5.6 and 6.6: minimum feature 2×2 px; no single-pixel noise; floors have no normal detail that throws hard highlights under a beam; the lit-frame test (6.6 rule 6) is the tier-1/2 test run *under lighting*, at each ring's ambient floor (4.3), on every new wall face and floor as well as every character. Horror rigs are tested on the three canonical poses of 5.4 because a rig has no canonical frame. UI glyphs are tested at 16 and 24 only (7.2); the HUD's integer scale steps (7.1) enlarge, never shrink, so no smaller gate exists. Light textures are the only assets that may legitimately own a mip chain — and only under pass B (8.4).

### 8.8 Export settings philosophy

**Export what the engine samples; nothing more, nothing it must undo.**

- **1× only.** Never export at 2× or 3× "for convenience"; the engine scales. Never resample, never antialias, never apply a colour transform.
- **Fixed, untrimmed cells**, origin at a recorded cell coordinate (8.6). Trimming breaks every offset table and anchor.
- **Binary alpha**, transparent background, no matte colour.
- **Palette-locked albedo**: the sprite file's palette *is* `palette_synaptic_sea.gpl`; an off-palette pixel is an export error, not a style choice. The two sanity bands (4.4) are not in the palette and cannot be reached by accident; hallucination content that *needs* them is a shader tint or a separate `ovl_` sheet marked as such, never a sane-world albedo. UI sheets add Paper White and UI Ground (4.6) and nothing else.
- **No baked direction.** 6.2's allowances only: crevice darkening of at most one value step at seams, under lips and inside grates; the 1 px Void Steel contour on characters and hero objects. Nothing on floors; no cast shadows anywhere. Operational test: the albedo flipped horizontally must look exactly as correct as the original — if the flip "changes where the light comes from", a direction was baked.
- **Normal and specular are generated from a painted height pass, never painted as normals** (amending 5.7's "add sheets, not drawing time" — 8.11, flag 5). A flat albedo has no height; a Sobel on albedo yields garbage. Each sheet carries a height/material layer of **2–4 grey levels** in the source file, budgeted at **20–30 % of the albedo drawing time**, from which the technical-artist's tool generates `_nrm` and `_spc`, so a redraw of the albedo regenerates its partners and the sheets can never drift apart. The height pass obeys 6.2: relief at edges and fasteners, planes flat, floors near-flat (6.6 rule 4). Painted normals are permitted only for hero-object correction passes and are then kept as a layer in the same source.
- **Sidecar JSON on every export** (8.5). An export without its JSON is incomplete.
- **Screenshots, not source files, for colour approval** (8.3): the 4.4 test and the 4.5 greyscale pass are run on 1080p captures at 3×; HUD contrast floors (HR-12) are measured the same way.

### 8.9 Per-asset acceptance checklist

An asset is accepted when every line that applies to it passes. Lines marked **(auto)** are scriptable and the technical-artist should script them; the rest are the art-director's review.

1. **Silhouette (3.1).** Reads at 25 %, no colour, no contour, *before* colouring. Pipe pieces: identifiable by section alone with hue removed. Dialect pieces: the three-piece test (6.3) in greyscale.
2. **Marks (5.6 tier 2).** At 50 % the band count, scar presence, tool type, vital overlays and medium motion still read.
3. **Flesh sampling (4.4).** On a 1080p screenshot, sample Hull Grey and lit Graphite under every Tungsten source present (fixture, each of the four beam shapes, deployable pool); hue ≥ 36°. **Check warm-lit Hull Grey first**; helmet bands fall back to Graphite lit if they fail, never warmed (5.1).
4. **Reserved-hue scan (auto).** No pixel in a sane-world `_alb` or `ui_` sheet falls in hue 270–330° at S > 25 %, or hue 5–30° at S 20–60 % and V 45–85 % (4.4; HR-6). Excluded materials of 6.2 (rust, copper, wood, leather, paper, tan) cannot pass this line by construction.
5. **Palette lock (auto).** Every `_alb` pixel is a palette hex or fully transparent; alpha is 0 or 255.
6. **No baked directional shading (6.2).** Flip test passes; crevice darkening ≤ 1 step; floor tiles carry none; no cast shadows.
7. **Contour rules (5.1, 6.6).** Characters and hero objects: exactly one 1 px Void Steel contour outside the figure, with ≥ 2 px clear beyond it in the cell. Environment tiles and supporting props: no contour. UI glyphs: no outline at all (7.2).
8. **Minimum feature (5.6).** 2×2 px, except the contour, band separators and 7.2's 1 px glyph strokes; no single-pixel noise. (auto: isolated-pixel count = 0 outside those elements.)
9. **Plane rule (6.2) (auto).** Large planes use at most two Graphite values; Hull Grey appears only in thin elements (4.1).
10. **Reserved composition (6.6 rule 2).** No circle-on-rectangle stack at crewmate scale anywhere in a kit; round floor features smaller than a third of the helmet (≤ 5 px) or larger than twice it (≥ 32 px).
11. **Sheet set complete (auto).** Every required partner sheet exists with the same stem and identical dimensions; flat regions of `_nrm` are (128,128,255); `_msk` R covers exactly the medium.
12. **State question (2.0).** Does this asset exist only because of a lighting, power, depth, attention or sanity *state*? If yes, reject and route the state to light, hue, mask or overlay. (UI corollary, 7.2: a glyph never swaps to express state.)
13. **Trail contrast (3.5, 6.4).** A trail out-contrasts every floor and wall it can cross in greyscale (4.5); no floor prop exceeds Graphite lit where a trail could run.
14. **Horror rules (5.4).** No closed circle, no bilateral symmetry, no four-limbs-in-pairs, no held object, no Tungsten or hue, no human worn part; three canonical poses pass line 1.
15. **Lit-frame test (6.6 rule 6)** for every new wall face, floor and character: four beam shapes, three ring ambients, cone centre and soft rim; achromatic pass holds.
16. **Naming, cell and import (auto).** Name parses against 8.5; cells on grid with margins per 8.6; import flags per 8.4; JSON sidecar present.

### 8.10 Ideal / constraint register

Both halves in one table. IDEAL, Why and Fallback are the art-director's; the CONSTRAINT column is the technical-artist's engine half, tagged with the sourcing legend of 8.4. Numbers marked as hypotheses are replaced by the lighting spike (8.12). Four rows are decided (logged): base resolution, tile size, loop media and render mode (as IDEAL plus fallback).

| Topic | IDEAL (art) | Why | CONSTRAINT (technical-artist) | Fallback the art-director accepts |
| --- | --- | --- | --- | --- |
| Base resolution | 640×360, integer scale at all four common sizes | the only integer-everywhere base (8.1); confirmed to 7.1 | Integer-scale table (arithmetic): 640×360 → 3× / 4× / 6× at 1080p / 1440p / 2160p; 960×540 → 2× / **2.67×** / 4×, fails 1440p; 1920×1080 → 1× / **1.33×** / 2×, fails 1440p and costs 9× the pixels to draw and light. **Decided: 640×360.** | none — not negotiable without reopening 7.1 |
| Tile size | 48 px | helmet-band arithmetic (8.2) | A 2048² page holds 441 cells of 96×96 (the TA's 2T cell) or ≈ 650 of the 80×80 cell of 8.2; the 230 body frames of 5.7 fit one 2048×1024 page. **Decided: 48 px.** | 40 px only if the user relaxed 5.6 for the bands (not taken) |
| Render mode | sprites on grid, lights at window resolution | hero #1's falloff is light, not drawing (8.4) | Pass A and pass B are one scene via `SubViewport.size_2d_override` / `size_2d_override_stretch` `[BIN]`; where `Light2D` resolves is `[UNV-4.3]`; light cost ∝ lit pixels × overlapping lights `[UNV-4.3]`, so pass B is up to 9× pass A (8.11, flag 4). **Recorded as IDEAL pass B with fallback pass A; 8.12 step 2 decides.** | pass A: lights at 640×360, cone edge a 3-px stair-step |
| Loop media | `_msk` fill shader, one pipe set | 2.0 + 4.2 without doubling 6.3 (8.3) | Emission = unshaded additive overlay (`LIGHT_MODE_UNSHADED` `[BIN]`) above `glow_hdr_threshold` (default `1.0` `[BIN]`), flow by UV scroll; pipes never illuminate; illuminating loops ≤ 8 non-shadow `PointLight2D` at machinery nodes inside the 24 cap below; glow ≤ 3 of `MAX_GLOW_LEVELS = 7` `[BIN]`. **Decided.** | two albedo states per piece (not taken) |
| Lighting budget and normal-mapped surfaces | normals on everything 6.2 lists plus low-frequency floors; true occlusion per Principle 1 | the beam draws shape (6.2) | Budget shape (hypotheses): shadow-casting `PointLight2D` ≤ 6 (4 beams + beacon + 1 deployable or fixture); non-shadow ≤ 24; `DirectionalLight2D` 0 (hard cap 8 `[BIN]`); occluder polygons in view ≤ 120 at ≤ 12 vertices, merging wall runs into one `LightOccluder2D` per room edge if `TileMapLayer` does not merge (NOT SOURCEABLE); `shadow_filter` `PCF5` on beams, `NONE` elsewhere; shadow atlas 2048 `[BIN]`. Lights cost no draw calls, per-item light cap 16, shadow cost ∝ shadow lights × occluder vertices — all `[UNV-4.3]`, numbers NOT SOURCEABLE. `CanvasTexture` carries normal and specular `[BIN]`; whether `Light2D` consumes them per pixel via `height` is `[UNV-4.3]` (8.12 step 6). Compatibility-renderer 2D parity NOT SOURCEABLE (step 9). | drop floor normals first; emitters before occlusion (Section 1); PCF13 never assumed |
| Ambient floors | 4.3's ring floors (≈ 3 / 1–2 / 0–1 %) with unlit hull still darker than true black (4.1) | the dark has a floor the player can learn | `CanvasModulate` multiply `[BIN]`: Graphite mid `#2A303A` × 0.02 ≈ 0.8/255 and Void Steel `#0B1018` × 0.02 ≈ 0.2/255 — both quantise to black in 8-bit SDR; ≈ 6–8 % needed for a 2-step gap; HDR 2D lifts darks a step or two `[UNV-4.3]`; `request_hdr_output` stays off. 8.12 step 7 histogram decides. | material-level hull floor + HDR 2D + debanding (8.11, flag 1) |
| Fixture shadows | 2.3's many warm fixtures under true occlusion | Principle 1 | shadow-casting lights are the scarce resource (row above); 2.3 + 2.8 are the worst cases | fixtures non-shadow; shadows for ≤ 4 beams + beacon + 1 deployable (8.11, flag 2) |
| Texture format | lossless, all sheets | hard edges, exact hexes, normal noise = lighting noise (8.3) | `compress/mode` default `0` `[BIN]`, meaning Lossless `[UNV-4.3]`; RGBA8 at 4 B/px, specular 1 B/px `[UNV-4.3]`. VRAM ≈ 190–225 MB full vision by the TA's arithmetic, ≈ 350 MB art worst case, ≈ 100 MB MVP (8.6); 4 GB memory ceiling; min-spec VRAM NOT SOURCEABLE. 8.12 step 10 measures. | none for `_alb` / `_nrm`; `_spc` may be packed into `_msk` B |
| Atlas cap | 2048² (4096² only for a full Alpha kit) | batching vs authoring (8.6) | No page above 4096² regardless (GPU maximum NOT SOURCEABLE; 16384 is `[UNV-4.3]`); every page is three or four textures and a batch break; `item_buffer_size = 16384` `[BIN]`; 1000 draws/frame budget; the Y-sorted layer is the multiplier, 50–300 (8.12 step 5). | merge the three shared sets; per-row `z_index` bands on wall faces |
| Light textures | 2048² greyscale, linear, mipmapped | 7-tile beam sharp at 6× | `PointLight2D.texture` is a `Texture2D` `[BIN]`, so the four beam shapes are four textures; LINEAR on the light texture `[UNV-4.3]`; preset `size_limit` 1024 under pass A (8.4) | 1024², no mipmaps, under pass A |
| Horror rig rotation | nearest; off-grid tolerated for web and horror only | 3.3 + 5.4 (8.4) | Manual rig: `Node2D` per limb chain with `Sprite2D` segments, GDScript animator stepped at 12 fps (every 5th tick at 60 Hz) with 15° quantisation, `z_index` ordering for joint overlap, iridescence as a per-segment `ShaderMaterial` with `set_instance_shader_parameter` `[BIN]`. `Skeleton2D` and the `SkeletonModification2D*` set exist `[BIN]` but are built for `Polygon2D` skinning, which deforms textures — reserved for the filament-rider type at most, quantised after solving; no `SkeletonModifier2D` in 4.7.2 `[BIN]`. Under pass B: pre-rotated 24-angle segment sets (a tool step; ≈ 160 MB at T = 48) or a base-resolution sub-viewport for horrors. | pre-rotated angle sets |
| Fringe (5.5) | 2 px shader outside the contour, unlit pass, 4 px cell margin | 5.5; HR-7 | Feasible: `LIGHT_MODE_UNSHADED` `[BIN]` receives and emits no light; per-viewer via `set_instance_shader_parameter` `[BIN]` from a presentation-only script (8.11, manifest rule); alpha sampled at ±1 / ±2 texels → ≥ 2 px padding; `show_behind_parent` `[BIN]` so it never covers sprite pixels; a `Light2D` has no outline, so the beam-edge fringe is four edge-mask sprites (one per beam shape); under pass B the shader takes the integer scale as a uniform; luminance below `glow_hdr_threshold` (8.12 step 8). | expanded quad if margins cannot be guaranteed |
| Materials | ≤ 5 | material count is the 2D "material slots" | TA count: fringe, per-segment iridescence, full-screen sanity filter (a `canvas_item` shader on the `SubViewportContainer`, or `BackBufferCopy` `[BIN]` without one), pipe-flow scroll (= the `_msk` fill), ambient-floor hull material — five; Shader Baker on `[REF]`. | — |
| Aspect | `expand` for ultrawide and Steam Deck | 8.1 | spike architecture starts from `aspect = keep` with a fixed 640×360 `SubViewport` | `keep` with letterbox until measured; `expand` is a post-spike ask |

### 8.11 Technical-artist flags and resolutions

Eight items — seven trade-offs the technical-artist raised and one the AD-ART-BIBLE review (2026-10-25) added — each answered here. ACCEPT means the art direction changes as stated; the section amended is named so the edit can be made.

1. **Ambient floors of 0–3 % quantise to black in 8-bit SDR output.** — **ACCEPT-WITH-CONDITION.** Adopt the technical-artist's recommendation: a material-level hull floor, `max(albedo × ambient, floor_tint)`, plus `hdr_2d` and debanding. Conditions: (i) the floor tint is Void Steel `#0B1018` at the ring's floor value and applies to **human-built hull geometry and the marks on it** — floors, walls, props, machinery, seams, and trails (a trail is a mark on a hull surface; an unlit trail must never render *below* the hull it sits on, or a dark stain would read without spending light) — never to web, filaments or horrors, which stay "not drawn unlit" (Principle 1; 6.1). (ii) The contract is the *gap*, not the percentage: unlit hull must sit 1–2 steps above true black in the final 8-bit frame at every ring (4.1's "the dark has a floor the player can learn"), measured by 8.12 step 7's histogram. (iii) **Amends 4.3**: its percentages become targets for the *ordering* (outer > middle > inner; home-ship 10 % stable / 2 % dark independent of ring, as 2.0 already says the ordering is the contract), and the value recorded per ring is whatever step 7 yields, written back into 4.3's table. (iv) This is the same mechanism HR-15's bounded low-vision option (measured floor + 2–3 steps, hull-only, per-client — 7.4) raises, so HR-15 costs no second path. Amends 4.1 (how "darker than true black" is implemented) and closes 2.0's "exact engine values belong to the lighting spike and Section 8".
2. **Many shadow-casting fixtures (2.3) are the state most likely to break the frame.** — **ACCEPT.** Fixtures are non-shadow lights; shadow-casting is reserved for the four player beams, the alarm beacon and **one sixth slot** — the nearest deployable in view, or, where no deployable is in view, a single fixture. **Amends 2.3** (fixture light is a soft, directionless fill and casts no shadow — which 2.3 already describes as "soft and nearly directionless") and **2.2** (the cryo bay's one emergency lamp with its "short, hard" shadows keeps them by taking the sixth slot — it is the only fixture in that state, so nothing is lost there). It **narrows the scope of Section 1's "true occlusion"** to the player's instruments of truth: the beams (the human channel), the beacon (so the 2.4 / 2.8 sweep still throws turning shadows), and the sixth slot; Section 1's rule that emitters are cut before occlusion stands unchanged for those six. A second deployable in view is a non-shadow pool. Consequence: a fixture can never produce "a shadow with no occluder", so the stage-2 tell (4.4) is always an unshaded dark polygon against a *beam's* light, which keeps that image an unambiguous sanity tell — consistent with 3.2's open recommendation. If even six is over budget, emitters are cut before occlusion (Section 1).
3. **Glow is per-viewport; the fringe must stay below the glow threshold; the HUD exemption needs the world in a `SubViewport`.** — **ACCEPT.** World in a `SubViewport`, HUD on a `CanvasLayer` in the root viewport (8.4). Conditions: the root viewport's integer stretch still governs the HUD, so 7.1's "instrument face on the art grid" holds — the HUD is integer-scaled at window resolution, never fractional. The fringe is non-emissive by 5.5 ("casts no light and raises no Attention"), so "below `glow_hdr_threshold`" is that rule made measurable; if `#B07BE6` at full alpha blooms under HDR 2D, the fringe's *alpha* is lowered, never its hue. The beam-edge fringe becomes four small unshaded edge-mask sprites — **amends 5.7** by +4 images, accepted. HR-1 is met by construction.
4. **Base resolution is a 9× lighting-cost lever.** — **ACCEPT, as decided.** IDEAL pass B, fallback pass A (8.4, 8.10); 8.12 step 2 is the A/B, judged on the criteria written there. No art rule depends on which wins: the grid, the sheets and the tests are identical in both.
5. **Normal maps need a painted height pass, 20–30 % of albedo time.** — **ACCEPT.** **Amends 5.7** ("normal maps are generated from them and add sheets, not drawing time" → each drawn sheet carries a 2–4-level height/material pass at 20–30 % of its albedo time, so the ≈ 424–434-image budget of 5.7 carries roughly +25 % drawing time — flagged to the producer) and **8.8** (reworded). Conditions: 2–4 grey levels only; relief at edges and fasteners (6.2); floor height passes near-flat (6.6 rule 4); never a painted normal except hero-object correction.
6. **The 2-px fringe needs 2 px of transparent padding per cell that survives import.** — **ACCEPT, confirmed at 4 px.** The art standard (8.6) is ≥ 4 px on every non-tile cell — 2 px for the fringe, 2 px against nearest-sampling bleed — which exceeds the technical-artist's 2 px minimum. `ResourceImporterTextureAtlas` is forbidden (8.4); sheets are packed externally once with one layout per page (8.5); sheet dimensions are multiples of the cell.
7. **Two REF-vs-BIN default discrepancies (Windows driver, stretch defaults).** — **ACCEPT.** Not an art decision; the stretch block is written explicitly into `project.godot` (8.4) and 8.12 step 1 prints the active driver. Editing `project.godot` is a configuration change and follows the normal approval path (lead programmer / technical-artist); it is not Section 8's to write.
8. **Principle 3, 2.3 and 2.4 said the ship is "lit only by what is still running" and loops glow "as isolated lanterns", while 8.3 makes pipes emissive without illuminating and gives light only to machinery nodes (≤ 8 non-shadow `PointLight2D`).** — **ACCEPT** (added per AD-ART-BIBLE review 2026-10-25). Principle 3 is reworded to "every running loop is visible as emission along its pipe and casts light at its machinery nodes"; 2.3 and 2.4 are reworded to match. Nothing the player reads changes — a pipe still glows in its hue and a node still throws light on the floor — and Pillar 3's exposure rule is intact: emission along the pipe is exactly what the web sees, and the budget shape of 8.10 is what makes 2.3's "every loop glowing" affordable at all. **Amends Section 1 Principle 3, 2.3 and 2.4.**

**Rule for the control manifest (from the technical-artist, adopted).** Presentation scripts — fringe, sanity filter, ambient-floor material, emissive fill, rig animator, HUD readings — **may read simulation state and may write only `CanvasItem`, `Material` and `Light2D` properties; they never call into gameplay or networking.** A player never sees their own fringe because the local script skips the local avatar. This is the concept's "corruption is presentation-only" rule stated for the renderer; routed to the lead programmer for `docs/architecture/control-manifest.md`.

### 8.12 Lighting spike plan

The technical-artist's ten-step measurement plan, adopted as written (it is the first spike per Section 1), with the art-director's pass criterion beside each step.

1. Print `RenderingServer.get_current_rendering_driver_name()` and `get_current_rendering_method()` `[BIN]` — settles flag 7a. *Art:* none; record the driver in 8.4.
2. One room (20×11 tiles, Foundry kit placeholders, `CanvasTexture` floor and walls) in the 8.4 `SubViewport` architecture. **Pass A**: `SubViewport` 640×360 scaled ×3. **Pass B**: `SubViewport` at window size with `size_2d_override = 640×360`, `size_2d_override_stretch = true` `[BIN]`. Confirm where `Light2D` resolves (flag 4); compare beam-edge look and frame time. *Art judges, in order:* the cone's outer edge (3.5: hard), the interior falloff (smooth vs 3-px bands, with debanding on), the fringe at exactly 2 base px, a rotating horror segment (grid-stepped vs off-grid), and whether Graphite planes stay two values under the beam (6.2). Pass B wins only if it fits 16.6 ms with the budget shape of 8.10; otherwise pass A is the recorded choice.
3. Sweep shadow lights 0→8 with `PCF5` and occluders 0→300 (per-tile vs merged); record frame time and `RENDERING_INFO_TOTAL_DRAW_CALLS_IN_FRAME` `[BIN]` at each step; find the knee. *Art:* the 2.8 breach composition (4 beams + beacon + 1 deployable) must sit below the knee; if not, fixtures are already non-shadow (flag 2) and the next cut is the deployable's shadow.
4. Sweep non-shadow lights 0→32 overlapping the floor; record per-item light-cap behaviour. *Art:* the 2.3 stable composition ("many warm sources, every loop glowing") must survive at ≤ 24 non-shadow lights with no visible light dropping out on a lit item.
5. Y-sort on/off on the wall layer; record draw calls. *Art:* the draw-call shape of 8.6 holds, or wall faces move to per-row `z_index` bands.
6. Normal-map Y and `CanvasTexture`-over-region / TileSet sampling checks; confirm importer enum meanings in the Import dock. *Art:* the helmet sphere's highlight tracks the beam from all four screen directions; a normal-mapped tile placed through a TileSet and an `AtlasTexture` region sprite light identically to a plain `Sprite2D`; the verified Y setting is written into 8.4.
7. Ambient floors 0 / 1 / 2 / 3 / 8 % with and without `hdr_2d` and debanding, SDR output; screenshot and histogram Graphite vs Void Steel (flag 1). *Art:* unlit hull 1–2 steps above true black at every ring with the material floor on; the values that achieve it are written into 4.3.
8. Fringe shader on one crewmate, glow on; confirm no bloom at `glow_hdr_threshold = 1.0` (the 4.6 glow rework `[REF]` — no older tuning transfers). *Art:* the fringe reads as an edge error, not a lantern (5.5); the running-loop overlay *does* bloom (8.3); the 4.4 sampling test is run on this capture.
9. Compatibility-renderer branch: repeat 2–4 once; record which features are missing (NOT SOURCEABLE until run). *Art:* the reversibility Section 4 assumes is either confirmed or struck.
10. Record `RENDERING_INFO_TEXTURE_MEM_USED` and `VIDEO_MEM_USED` `[BIN]` against the 8.6 estimates; enable the Shader Baker `[REF]` and note first-run hitching before and after. *Art:* none; both VRAM numbers in 8.6 are replaced by the measurement.

**Downstream flags.** *ux-designer / Section 7 maintenance*: a Steam Deck (1280×800, 2×) safe-area margin for the 7.0 layout; the 8×8 state-mark cell added beside 7.2's 16/24 glyph sizes, with the rule that it never names a loop. *Technical-artist*: the spike plan above; the `_msk` channel contract; the four beam edge-mask sprites; the ambient-floor hull material; TileSet padding; the auto-scriptable checklist lines of 8.9; folding the `[BIN]` facts of 8.4 and 8.10 into `docs/engine-reference/godot/modules/rendering.md`. *Lead programmer*: the control-manifest rule of 8.11; the explicit `[display]` block in `project.godot` (flag 7). *Creative-director*: the amendments to Section 1 (scope of true occlusion; Principle 3 wording), 2.2, 2.3, 2.4, 4.1, 4.3 and 5.7 recorded in 8.11 — none alters a pillar, but 4.3's values are now spike-written, fixtures outside the sixth shadow slot no longer occlude, and the crewmate budget carries a height pass. *Accessibility-specialist*: if HR-12's tier moves the cap-height floors from 21/15 to the next integer step, 7.1 already derives 8/6 art px; no Section 8 size changes, since the instrument face scales by integer steps independently of the world grid; HR-15's low-vision floor is the 8.11 flag-1 material. *Game-designer*: integer zoom steps only; room-kit proportions under the 13.3×7.5-tile frame; horror type names to replace `t01`–`t04`. *Producer*: the height pass of 8.11 flag 5 adds roughly 20–30 % to per-sheet drawing time against 5.7's image count; the `_msk` sheet and the four edge masks are the only other additions.

### 8.13 Post-approval maintenance notes (AD-ART-BIBLE, 2026-10-25)

Non-blocking items recorded by the sign-off reviewer for the next maintenance pass:

- **Vent bursts (2.8) vs 8.11 flag 8.** 2.8 describes venting loops as point sources where they burst; flag 8 says only machinery nodes cast light. A vent burst is a transient failure event: treat it as a transient non-shadow `PointLight2D` inside the ≤ 24 cap, or as purely emissive glow with no light — whichever 8.12 supports. Routed to technical-artist.
- **8.3 fixture row** says "never flickers"; the cryo-bay lamp gutters (2.2) and is the one exception, via the sixth shadow slot.

- **User decision 2026-10-26 — chest-mounted lamp (Pandorum), supersedes "hand point" as the beam origin in 3.2, 5.1, 5.5, 5.6 and 8.3.** The player's lamp is mounted on the suit's chest and is hands-free: the beam origin is a per-facing **chest point** near the centreline (so a mirror barely moves it), the tool layer carries no light, and the beam is available while carrying and channelling. Carry frames must keep the chest lamp unobstructed (module held below it). The four beam *shapes* remain the identity rule unchanged. Art-director to rewrite the five passages in place at the next maintenance pass.
- **User decision 2026-10-26 — throwable glowsticks** join the deployable-light family (3.2, 8.3): thrown on a short arc, landing as an omnidirectional world light the web can see. Current deployables are specified as flickering Tungsten pools; a glowstick's steady chemical light needs a colour decision inside the palette — it must not borrow a loop hue or the sanity hues (4.4). Art-director to specify (and whether the deployable "flicker" word stays for flares only).

## 9. Reference Direction

### 9.0 How this section is read

Section 1 lists six touchstones with one line each. That list says *what the game is near*; this section says *what to take from each neighbour and where to stop*. The rules:

- **One reference owns one bible decision.** Each of the five references below is pinned to a single decision Sections 1–8 already made, and is cited for that decision only. A reference that could also justify a second decision does not; the second decision has its own reference. This is what "additive" means here, and it is auditable in the table below.
- **Every reference carries a *Take* and an *Avoid*.** The *Take* is a technique, colour choice or compositional rule, stated so a reviewer can check an asset against it. The *Avoid* is the part of the source that contradicts a bible rule, named precisely, so "it looks like X" is a legitimate critique only when the asset has picked up something from X's *Avoid* column.
- **Touchstones are not references.** The user's and the designer's comparables are honoured in a register (9.6) with one kept technique and one refused one each. They set the neighbourhood; they do not set the rules.
- **In review**, cite a reference by number ("9.3 says the needle is the reading, the bezel is the frame"), never by name alone. A mood board for any asset category draws from at most two references, and the board states which two.

| # | Reference | Bible decision it owns | The one thing taken |
| --- | --- | --- | --- |
| 9.1 | Joseph Wright of Derby (1760s–70s candlelight paintings) | Principle 1 and 3; 2.1; 3.5 #1 | the light source sits *inside* the frame, occluded by what it lights; nothing outside its reach is drawn |
| 9.2 | *Graveyard Keeper* (Lazy Bear Games, 2018) | 6.2 pipeline | unshaded pixel albedo plus normal maps, lit at runtime by moving engine lights |
| 9.3 | Apollo Command Module main display console (NASA, 1960s) | 3.4, 7.0, 7.3 instrument grammar | the needle-and-zone-band meter: the panel is the frame, the needle is the reading, the number lives elsewhere |
| 9.4 | *Annihilation* (Alex Garland, 2018) — the Shimmer | 4.1 web iridescence; 4.3 depth | thin-film sheen as the alien's only signature, carried on highlights and moving with the viewing angle |
| 9.5 | *Beyond the Aquila Rift* (*Love, Death & Robots*, 2019) | 4.4 sanity palette; Section 1 Corollary | the lie is drawn *kinder and warmer* than the truth, and what leaks through it is flesh |

No two rows share a decision. 9.1 and 9.2 are the nearest pair (both are about light) and are separated by level: 9.1 is composition, what the frame contains; 9.2 is construction, how a sprite is built so the engine can light it. 9.4 and 9.5 are the other near pair (both are about the inhuman) and are separated by channel: 9.4 is the web's *sane* appearance, the iridescent band every player sees; 9.5 is the *insane* palette only a failing mind sees.

### 9.1 Joseph Wright of Derby — the source inside the frame

**Source.** The candlelight and furnace paintings: *A Philosopher Lecturing on the Orrery* (c. 1766), *An Experiment on a Bird in the Air Pump* (1768), *The Alchemist Discovering Phosphorus* (1771), *An Iron Forge* (1772). Enlightenment machinery, lit by one artificial source, watched by competent people.

**Take: the brightest object is hidden behind the thing it lights.** In the *Orrery* the lamp that stands in for the sun is occluded by the instrument and a child's head; in the *Air Pump* the candle is behind the glass bowl; in the *Forge* the white-hot bar is the source and the subject at once. Three consequences, each already a bible rule, and this is the reference that shows them working together on one canvas:

1. *Nothing outside the source's reach is drawn.* Wright's backgrounds are not detailed-then-darkened; they are undrawn. The room exists (a window, a column edge catches a rim) but the eye is given nothing to do there. This is Principle 1's "unlit, not fogged" as a picture: the geometry is real, the darkness is honest, and the edge of the lit world is where the light actually stops. For 2.6 boarding and the 4.3 inner ring this is the composition target — the beam is the whole picture and the rest of the frame is Void Steel with, at most, one caught rim.
2. *The lit thing is the machine, and the people are lit by looking at it.* The orrery, the air pump and the forge are the hero shapes; the faces are secondary and turned toward them. That is 3.5's hierarchy — beam, then loop machinery with visible media — and Principle 3's lantern: a running machine lights the crew that tends it. For 2.3 home-ship stable, the fixture-lit sight-glass with a crewmate bent over it is a Wright composition at 3/4 oblique.
3. *Hard falloff from near-source to black inside one frame, no fill light.* Wright does not soften the dark to help the viewer. The 2.1 rows that read "high, hard" (cryo-wake, conserved dark, boarding) are this contrast ratio.

**Avoid.**
- *Umber shadows.* Wright's darks are warm brown; the bible's are blue-black (Void Steel, 4.1). An asset whose shadow side has drifted warm has picked up the wrong half of Wright. Shadow is cold here because the dark is the void, not the room.
- *Skin.* The paintings are dominated by lit faces and hands, which is this game's forbidden band (4.4). The composition is taken; its dominant colour is excluded. Crewmates are sealed (5.1); the only "face" in the lit area is a visor with one highlight.
- *The painted glow.* Wright softens the air around a source into a halo. 3.5 #1 specifies a hard outer edge with soft interior falloff. The cone's rim is a line, not a bloom.
- *The candle's hue.* A candle flame in oil paint reads orange-brown, which is inside the flesh hue range. Tungsten is held at hue ≥ 38° (4.4); when a Wright board is used, the light is pulled toward yellow-amber, never toward his flame.
- *Theatrical expression.* The figures gasp, wonder, weep. 5.2 is stiff-weighted; feeling is in the pose and the tool, not the face.

**Why additive.** This is the only reference about *where the light is in the picture*. 9.2 says nothing about composition; 9.3's panels are lit flat; 9.4 and 9.5 are about colour.

### 9.2 *Graveyard Keeper* — pixel albedo lit by the engine

**Source.** Lazy Bear Games' 2018 management sim, and specifically its lead programmer's published breakdown of the rendering — "Graveyard Keeper: how the graphics effects are made" (Svyatoslav Cherkasov, 2018; <https://slava.lazybeargames.com/graveyard-keeper-how-the-graphics-effects-are-made>, also on gamedeveloper.com) — covering normal maps on pixel sprites, dynamic light sources (torches, ovens, lamps) and a light-distribution scheme so that top-facing surfaces of 3/4 geometry are lit as tops. Named as the proof that the 6.2 decision ships: HD pixel art stays pixel art while a moving light draws its volume.

**Take: the sprite stores what a surface *is*; the light decides how it looks.** The specific techniques to lift, each mapped to a 6.2 rule:

- *Normal maps on sprites so that a round thing reads as round under a light from any direction.* This is the single reason 6.2 rejected hand-shaded pixel art: a pipe shaded top-left lies when the beam comes from the right. Graveyard Keeper's lamps move around its sprites and the sprites answer correctly. The bible's acceptance test for any normal-mapped asset is a Graveyard Keeper test: rotate one Tungsten point light around the sprite at four cardinal positions; the lit side must follow the light with no residual baked direction.
- *Top faces and front faces are lit differently.* Their scheme lights the top of 3/4 geometry as a horizontal surface and the front as a vertical one. 6.2's wall — top cap plus a 1.5-tile front face — needs exactly this, so an overhead fixture (2.3) lights caps and a hand-held beam (2.6) lights faces.
- *Point sources with physical reach.* Each of their lamps is a positioned light with falloff, not a screen tint. Every Tungsten fixture, deployable and running loop in this game is the same kind of object (Principle 3).

**Avoid.**
- *The time-of-day LUT and the blue-tinted night.* Graveyard Keeper's night is a colour grade applied over a fully visible scene; the whole world stays readable, just bluer. That is a global filter, the thing 4.4 rejected for sanity and Principle 1 rejects for darkness. This game's night is 2 % ambient: outside the light there is nothing to grade.
- *Residual baked shading.* Their sprites still carry hand-painted highlights and ambient occlusion under the normal map. 6.2 permits only direction-agnostic crevice darkening at one value step and the Void Steel contour on hero objects; no baked highlights, no baked cast shadows.
- *Shadow sprites.* They draw a shadow sprite per object and move it with the light. 6.2 forbids cast shadows in any sprite; shadows here come from the engine's occluders (Principle 1's true occlusion) or not at all.
- *The warm mid-value palette and chibi proportion.* Their world is brown, green and cosy, with big-headed figures. 4.1's base is cold Graphite; 5.1's crewmate is 1.1 tiles, compact, not cute. Nothing of the look is taken; only the construction.

**Why additive.** This is the only reference about *how a sprite is built*. It is a pipeline reference, and the mood is deliberately the opposite of this game's, which is the cleanest way to keep it from being copied.

### 9.3 Apollo Command Module main display console — frame, reading, number

**Source.** The main display console of the Apollo Command Module (NASA, 1960s): the grey panel field of round and vertical-tape meters, guarded toggle switches, engraved labels, the caution-and-warning annunciator block, and the DSKY (the numeric keyboard and display of the guidance computer). Photographs of the flown and museum consoles are the board material.

**Take: a meter is read by needle position against its scale at a glance; precision is called up on a separate display.** Three parts, each pinned to a bible rule:

1. *The needle is the reading; the panel is the frame.* An Apollo meter is a thin white needle (or tape edge) moving inside a static, engraved, mid-grey housing, with a scale and a marked zone for the safe band. The viewer separates "what the instrument is" from "what it is saying" at a glance, which is 3.4's frame-versus-reading grammar made physical and 7.0's answer to where a lie can sit (on the thin moving mark, never on the housing). 3.1's rule that in-world gauges are readable by needle angle and zone band alone is this meter with its printed scale numerals removed — at HD pixel scale they would be unreadable anyway, and the zone band carries the read.
2. *Precision lives on the DSKY, not on the meters.* The console keeps exact values on one dedicated numeric display, called up by the crew. That is 7.0's inspect card: the same instrument, magnified, carrying the numeral in Paper White — and lying exactly when the wall gauge lies.
3. *One annunciator block for every alarm.* The caution-and-warning panel is a single field of lamps, and nothing else on the console lights up to shout. 4.1 keeps Alarm Red for alarm only, and 7.3 gives alarm one rhythm (the beacon sweep) everywhere; the console shows how little alarm needs once it has a reserved place.

The secondary take is material: engraved white-on-grey labels (Paper White on Graphite, 4.6), no decoration, every mark functional. The HUD's Foundry rivets (7.0) are its only ornament and they are fasteners.

**Avoid.**
- *Switch density.* The console carries several hundred switches and dozens of meters across the whole field of view. 7.0 caps the HUD's non-Ground pixel area at roughly 1 % of the frame and keeps the top edge clear. The Apollo reference justifies the *shape* of one meter, never the *count*.
- *The panel plane as HUD fill.* The console's grey field is a continuous surface. This HUD is a few small opaque Ground plates (`#1C2128`) on a black-margin frame; the world shows between them.
- *Integral panel lighting in a cool hue.* Apollo's panel markings were lit from within in a cool cast (confirm the exact hue against photographs before boarding it). Any cool instrument light collides with O₂ Cyan or Coolant Ice (4.1); the bible's in-world instrument light is Tungsten fixture, and screen-space text is unlit Paper White.
- *Nostalgia.* The console is period: mission badges, cursive manufacturer plates, the whole "cassette futurism" reading. Nothing on this HUD should date it. A player should see an instrument, not an era.
- *Toggle-switch iconography.* Where UI needs a loop glyph, 3.4 uses the loop's own fitting silhouette; it does not borrow the switch.

**Why additive.** This is the only reference about *instruments as drawn objects*. 9.1's machines are lit but their dials are not read; 9.2 has no instruments at all.

### 9.4 *Annihilation* — the Shimmer as a surface property

**Source.** Alex Garland's 2018 film: the Shimmer's thin-film refraction, and the final lighthouse interior where the organism is near-black and its colour rides only on highlights. The sequence to board is the lighthouse, not the meadow.

**Take: the alien has no colour of its own; it has a sheen that depends on the angle to the light.** The film's device is that the Shimmer refracts what is already there, so it reads as a surface property rather than a pigment, and it moves when the camera or the light moves. 4.1 specifies exactly this for the web: iridescence applied as a specular *band*, never a fill, visible only under direct light, with no hue of its own; 5.4 extends it to horrors (Void Steel mass with the band and nothing else). Three specifics to lift:

- *The band moves with the angle.* As the player's beam sweeps a filament, the teal → green → bronze → gold band should slide along it. A static rainbow stripe is a texture; a sliding one is a thin film. This is the behaviour that makes "filaments look wet" (4.3 middle ring) and "throw highlights back" (inner ring).
- *The sheen is strongest on the darkest surface.* The lighthouse organism is almost black, and that is why its highlights carry so much. The web's unlit mass is not drawn at all (Principle 1); its lit mass is Void Steel; the band sits on top. Dark base, bright band, nothing between.
- *Continuity carries dread.* The Shimmer's sheen is unbroken across every surface it has touched, so the eye follows it. 3.3 wants the web to run *through* human geometry and lead the eye into the dark; the band is what the eye follows.

**Avoid.**
- *The full spectrum.* The Shimmer is prismatic and passes through magenta and violet. 4.1 excludes the 270–330° arc on purpose: iridescence is highest at depth, and a violet web would leak the sanity hue to sane players and break Pillar 1's sane-palette channel. The board for the web is the film's bronze-green-gold range only, and a reviewer samples any iridescence shader for hue before anything else.
- *Volumetric haze and bloom.* The Shimmer is a fog that tints the air. The web has no volume; unlit it is undrawn, and the band is a surface highlight with no bloom. 3.5's hard-edged beam is the only light shape, and it does not glow into air: 2.1's "dust in the beam" is particles the cone catches, not a haze the cone tints.
- *Floral beauty and the organic-garden register.* The film's growths are flowers, antlers, coral. The web is roots and nerves — tapering, forking, no closed circles (3.3), and it is never pretty in the flower sense. Beauty here is the band on something that should not have one.
- *Mirrored and mutated human forms.* The film's horror is bodies copied and blended. 5.4 forbids horrors any worn human part, bilateral symmetry or paired limbs. The web does not imitate; it negates.
- *Daylight.* Every Shimmer exterior is bright. 2.1 has no daylight state and no lit exterior; the one exterior, pod transit, is a warm point in a 0–1 % black field.

**Why additive.** This is the only reference about the web's *sane* appearance, the material every player sees. 9.5 is the material only a failing mind sees, and the two must never share a hue.

### 9.5 *Beyond the Aquila Rift* — the kind lie and the flesh that leaks through it

**Source.** The *Love, Death & Robots* episode (2019, from the Alastair Reynolds story): a crew wakes at a station run by a familiar face; the comfort is a construct; the truth is seen in a single cold hard light.

**Take: the hallucination is rendered *more* comfortable than reality, and its leak is flesh.** The episode's design decision is that the lie is warm, soft and domestic — bedroom light, skin, a face that is almost right — while the truth is cold and hard. That is the bible's drift order (4.4) in one image: stage 2 "apparent warmth", stage 3 "surfaces warm toward flesh; pipes read as veins". Three specifics:

- *The lie's temperature.* A low-sanity scene is not darker or more saturated than a sane one; it is *warmer*, and warm in the wrong hue — toward orange-pink (flesh, hue 5–30°) rather than Tungsten's yellow-amber. The episode's bedroom is the colour target for what a lying beam seems to bring into a room.
- *The tell is in the almost-right.* The face is correct until it is not, and the viewer senses the wrongness before locating it. Section 1's Corollary says subtle is allowed, absent is not: the leak lives at a fringe, a flicker, a tint at the edge of the light — found by a player who looks at edges, missed by one who does not.
- *The truth is seen by a single cold light.* The reveal is lit the way 2.6 boarding is lit: one hard source, steel, no warmth. The sane world's coldness is the proof that the warm one was a lie, which is why 4.1's base is cold and 4.3 holds temperature constant across depth. Warmth that was not paid for with a Tungsten source is a symptom.

**Avoid.**
- *The reveal.* The episode resolves into a full, explicit shot of the truth. This bible defines no reveal register: the leak at the edge is the only sanity signal it draws, and the low-sanity overlay (2.9) has no "cut to the truth" state. Whether the game ever confirms a hallucination to the player is the game-designer's; if it does, that moment will need its own visual rule, and it will not be a shot of a monster.
- *Photoreal rendering and skin as a subject.* The episode is skin-forward. Flesh here is a hue band that leaks onto steel and phantom content (4.4), never a drawn body; crewmate sprites are sealed anchors and never shift.
- *The creature.* The thing at the end is arachnid: bilateral, paired limbs, a head. 5.4 forbids all three. Horrors are built from the web's negation rules, not from any animal.
- *The real station's grade.* The truth-side environment is rust, grime and brown-green. 6.2 excludes rust, oxidation and every warm material; the sane world is cold Graphite and Hull Grey.
- *Eroticism as the lure.* The bible's lure *palette* is unpaid warmth on steel — the stable home-ship of 2.3 seeming to return under a beam that cannot have brought it. What a hallucination shows (phantom salvage, entities, crewmates-as-horrors) is Pillar 4's and the game-designer's; this section fixes only the colour of comfort, not its content.

**Why additive.** This is the only reference about the *palette of the lie*. 9.4 owns the web's honest sheen; 9.1 owns honest warmth (Wright's Tungsten is paid for by a visible source). This reference owns warmth that is not paid for.

### 9.6 Touchstone register

The user's six touchstones and the designer's four comparables (Barotrauma is on both lists, so nine rows), plus one near-miss worth naming, each with the one technique kept and the one thing refused. These calibrate the neighbourhood; where a kept item overlaps a numbered reference, the reference governs.

| Touchstone | Keep (one technique, and the rule it serves) | Do not keep |
| --- | --- | --- |
| **Project Zomboid** | the room as a record: props at 3/4 scale placed so a reader can reconstruct what happened (6.5 verdict grammar); the patient pace of one figure moving through a dark interior | daylit suburban palette; baked-shaded tiles (6.2); night as a blue tint over fully drawn geometry; the line-of-sight mask that hides tiles the player cannot see, which is a perception mask and Principle 1 rejects masks |
| **Satisfactory** | machinery with a visible moving part and a visible medium, and flow marks along the carrier so a fault is found by following the eye along the line (Principle 2; 3.3 pipe grammar) | the saturated daylight exterior; the corporate orange, which sits next to Fuel Ember and the flesh band; the world-space status icon on every machine (Principle 2's rejected icon layer); scale as spectacle |
| **Stellaris** | dread delivered as a sober chart and a line of text, never a picture: the chart-table register of 2.11 and 7.4, where the map is calm and the threat is read | the whole visual register: space-opera palette, hyperlane glow, bloom on the galaxy map, any colour on the chart that is not Graphite, Tungsten text and Paper White |
| **Beyond the Aquila Rift** | governed by 9.5 | governed by 9.5 |
| **Pandorum** | ship size measured by what the beam fails to reach: a far wall that does not resolve is the scale cue (4.3 outer ring, 2.6); waking into a ship whose condition must be read from its surfaces (2.2) | the grade: green-brown, wet, rusted, grimy, every material 6.2 excludes; the creatures (feral humans — flesh, bilateral, 5.4 forbids); handheld camera energy, which 2.9 and 7.3 forbid as a screen effect |
| **Barotrauma** | failure as a thing you watch arrive through the hull — water rising, fire spreading compartment to compartment — so a breach is a trail with a direction (Pillar 2; 2.8 "a trail you must outrun") | the side-on cross-section; the heavy-outline sprite style; status icons floating over every device; a hull interior that is fully visible at all times |
| **Cosmoteer** | a ship assembled from modules where the join between parts is visible and the ship's shape is its history (3.3 grafts; the weld seam as hero shape 3.5 #3) | flat top-down lighting; modules identified by colour tag (3.3 rejected the tag); the clean, cheerful palette |
| **Oxygen Not Included** | one hue per loop, learned once and reliable everywhere, with gases and liquids drawn as visible media (4.2; Principle 2) | overlay modes — a toggled world-space diagram layer is the icon layer Principle 2 rejected; the cartoon register; the cutaway side view; bright ambient everywhere |
| **Lethal Company** | co-op horror where a friend is identified by the light they carry and looking is the risk (3.2 beam identity; Principle 3 "emission is exposure"); the derelict's one residual light (3.5 #4) | the low-poly, VHS-grain, fisheye register; chromatic aberration and screen distortion (7.3 forbids them on the HUD and 2.9 forbids reading sanity off screen energy); the comedic tone |
| *Darkwood* (not a touchstone; named as the nearest miss) | light as a resource with a hard outer edge, and night as the whole game | the vision-cone perception mask: things outside the character's facing are hidden even when lit. Principle 1's edge is the edge of a real light, not of attention |

### 9.7 What this bible is not trying to look like

Each item is a look Sections 1–8 have already rejected, listed so a mood board or an asset can be refused by name.

- **A fog-of-war game.** No vignette, no reveal radius, no unexplored-grey, no minimap. The dark is unlit geometry (Principle 1; 7.0).
- **Hand-shaded pixel art with a baked light direction.** Beautiful, and a lie once the beam moves (6.2).
- **A colour-graded night.** No blue tint, no global LUT for darkness or for sanity; the sane world is not desaturated as the mind fails (4.4; 9.2 *Avoid*).
- **A rusted, worn freighter.** No rust, brass, copper, wood, leather, tan or sepia steel (4.1 alternative rejected; 6.2 excluded materials). Weathering is scuff, scorch, dust and frost.
- **A per-depth palette.** Deeper rings are darker and more iridescent, never warmer, cooler or differently hued (4.3).
- **A prismatic alien.** No violet or magenta in any iridescence; no bloom; no haze (4.1; 9.4 *Avoid*).
- **A creature feature.** No animal, insect or mutated-human horror; horrors are built from the web's negation rules (5.4).
- **A fully diegetic HUD** (the *Dead Space* model) or a **fully screen-space survival HUD** of bars and icons (7.0 alternatives rejected). The suit tells you about you; the ship tells you about itself along its pipes.
- **A dense retro cockpit.** One meter's grammar from 9.3, never its switch count (7.0's area cap).
- **A juicy or glitchy interface.** No bounce, scale-pop, scanlines, chromatic aberration or shake on the HUD; a lying instrument glides, it does not break (7.3).
- **A candlelit period painting.** Wright of Derby's composition without his umber shadows, his skin or his orange flame (9.1 *Avoid*).
- **A cutscene reveal.** The lie leaks at edges and is never confirmed by a shot of the truth (9.5 *Avoid*).
