# Game Concept: The Synaptic Sea

*Created: 2026-10-25*
*Status: Draft*

> **Creative Director Review (CD-PILLARS)**: APPROVED 2026-10-25 (after two CONCERNS rounds; lethal friendly harm recorded as a deliberate user decision, not blocking)
> **Art Director Review (AD-CONCEPT-VISUAL)**: DIRECTIONS DELIVERED 2026-10-25 — user selected a B-dominant hybrid (see Visual Identity Anchor)
> **Technical Director Review (TD-FEASIBILITY)**: CONCERNS (accepted) 2026-10-25 — HIGH RISK as first written; two scope changes accepted (host-owned save, single-player slice first)
> **Producer Review (PR-SCOPE)**: OPTIMISTIC → adjustments applied 2026-10-25 (VS split, spike phase, widened MVP estimate, 4–6 year horizon)

---

## Elevator Pitch

> It's a co-op survival sim where you keep a ship welded together from dead ships
> alive inside a vast cosmic web, and the deeper you salvage, the less you can
> trust what you see.

Wake in the cryo bay of a derelict. Repair a podcraft. Board the other wrecks
caught in the web, strip their working systems, weld them onto your own hull.
The web notices light and noise. Exposure erodes sanity, and sanity rewrites
what your eyes and instruments tell you.

---

## Core Identity

| Aspect | Detail |
| ---- | ---- |
| **Genre** | Survival sim / cosmic horror, with functional (not freeform) base-building |
| **Platform** | PC (Steam / Epic), Windows first |
| **Target Audience** | Systems-literate explorers who want co-op dread (see Player Profile) |
| **Player Count** | 1–4. Single-player first; online co-op gated behind the proven slice |
| **Session Length** | 60–120 minutes |
| **Monetization** | Premium, Early Access (provisional — not yet decided) |
| **Estimated Scope** | Large (48–72 months, solo) |
| **Comparable Titles** | Project Zomboid, Barotrauma, Oxygen Not Included |

---

## Core Fantasy

"The ship is alive because we keep it alive, minute by minute — and we're the
only ones awake, and we're starting to understand what that means."

Cooperative self-sufficiency under cosmic dread. The player's base is not a
house they found; it is a body they assembled from the corpses of other ships,
and every loop in it is something they personally keep breathing. The horror
is not that the web attacks the ship. It is that the web attacks the crew's
ability to know whether the ship is fine.

The emotional anchor comes directly from the designer's own touchstone: a
Project Zomboid base, built with friends, that becomes self-sufficient and
survivable. This game puts that feeling in a place where "survivable" is never
quite certain and the friends might be hallucinating.

---

## Unique Hook

Like Project Zomboid's long-tail survival loop, **AND ALSO** your base is a
life-support machine welded from other dead ships, and the sanity system is a
perception system: gauges, sprites, salvage and even your crewmates can lie to
you, and the only cure is to check — which costs time, risk or trust.

Why it is not a gimmick:
- It affects gameplay, not aesthetics. A false "fixed" state on a scrubber is
  a death you walked into.
- It is connected to the core fantasy. Co-op matters because crewmates are each
  other's instruments.
- It is novel. Sanity in survival games is usually a stat debuff. Here it is
  misinformation with a verification economy around it.

---

## Player Experience Analysis (MDA Framework)

### Target Aesthetics (What the player FEELS)

| Aesthetic | Priority | How We Deliver It |
| ---- | ---- | ---- |
| **Discovery** (exploration, secrets) | 1 | Procedural derelicts with one visible unknown each; web rules learned by observation; lore assembled from fragments; the center as a horizon |
| **Challenge** (obstacle course, mastery) | 2 | Visible life-support graph that fails in traceable ways; Attention management; breach recovery; learning horror tells |
| **Fellowship** (social connection) | 3 | Crew roles (boarder vs. ship-keeper); per-player sanity makes trust a mechanic; a shared home-ship |
| **Fantasy** (make-believe) | 4 | "We are the only ones awake"; a ship that is ours because we welded it |
| **Narrative** (drama, story arc) | 5 | Non-linear lore; the cryo-bay question (who else is in the pods, and why do they keep waking?) |
| **Sensation** (sensory pleasure) | 6 | HD pixel art under a light-as-resource rendering model; flow-coded loops glowing in the dark; adaptive audio (later tier) |
| **Expression** (self-expression) | 7 | Which modules to graft and where — functional, not cosmetic |
| **Submission** (relaxation) | N/A | Deliberately absent |

### Key Dynamics (Emergent player behaviors)

- Players will **go dark** deliberately — running fewer loops, cutting lights —
  to lower the web's attention before a deep sortie.
- Players will **cross-check each other**: "Is that scrubber really green on
  your screen?" becomes normal crew chatter.
- Players will **specialize**: one keeps the ship alive, one boards.
- Players will **weigh a module's value against the noise of taking it**.
- Players will **develop house rules for low-sanity crewmates** (disarm them,
  keep them away from the reactor) without the game prescribing it.
- Players will **tell stories about the time the loot was garbage**.

### Core Mechanics (Systems we build)

1. **Life-support graph** — O₂, heat and power as a continuously simulated,
   visible graph across welded sections; failures propagate along edges and
   render as physical trails (frost, mist, scorch, drip).
2. **Modular welding** — derelict sections are grafted onto the home-ship,
   joining their nodes to the graph and their collision to one composite body;
   sections can be cut away.
3. **Attention** — the web accumulates attention from light, noise, power use
   and depth (depth sets the baseline); shown in the world, never as a hidden
   meter; crossing thresholds flips boarding into breach.
4. **Sanity and hallucination** — per-player sanity degrades with exposure;
   low sanity drives a client-side presentation filter that corrupts perception
   (entities, system states, crewmates, salvage) while the simulation stays
   authoritative and at least one channel of truth remains.
5. **Procedural derelicts** — hand-authored room kits assembled by a generator
   from a seed, with salvage placed by rules and one externally visible
   unknown per wreck.
6. **Persistence and the failure contract** — host-owned, versioned saves of
   the composite ship, world seed plus deltas, and per-player progression;
   losing the ship or a body never loses knowledge.

---

## Player Motivation Profile

### Primary Psychological Needs Served

| Need | How This Game Satisfies It | Strength |
| ---- | ---- | ---- |
| **Autonomy** | Which derelict, which module, how deep, how loud, who boards and who stays | Core |
| **Competence** | Reading a failing loop, surviving a breach, learning what each horror does before it does it, learning when to doubt your eyes | Core |
| **Relatedness** | Crew roles; crewmates as each other's instruments; a shared ship; the cryo-bay mystery | Core (co-op), Supporting (solo via the ship-as-body) |

### Player Type Appeal (Bartle Taxonomy)

- [x] **Explorers** — Primary. Understanding the web's rules, the derelicts' stories, and their own perception.
- [x] **Socializers** — Secondary. Crew trust, verification chatter, shared stories of disaster.
- [x] **Achievers** — Light. Ship capability and depth rings as a ladder; no completion checklist.
- [ ] **Killers/Competitors** — Not served. No PvP, no leaderboards; horrors are avoided, not farmed.

### Flow State Design

- **Onboarding curve**: The cryo bay is a single sealed section with one
  failing loop. The first ten minutes teach "see the leak, follow it to the
  source, fix it, watch the gauge agree with the pipe." The first pod hop
  teaches the boarding rhythm; the first Attention threshold teaches breach.
- **Difficulty scaling**: Depth rings raise the Attention baseline, horror
  variety and salvage value together. Outer-ring depletion pushes inward.
- **Feedback clarity**: Every system has a visible moving part and medium.
  Failure is a trail the eye can follow. Attention is heard and seen before it
  acts. Sanity drift starts with instruments disagreeing with the world.
- **Recovery from failure**: Breach is survivable by retreat to the pod. Losing
  the ship or a crewmate loses ship and body, never knowledge: the next crew
  wakes in the cryo bay knowing more. Failure is educational by contract.

---

## Core Loop

### Moment-to-Moment (30 seconds) — two rhythms

- **Boarding rhythm** (deliberate, Zomboid-paced): cut a hatch, listen, sweep a
  light, read a dead ship's gauges, decide whether a module is worth the noise
  of taking it. Every door is a decision.
- **Breach rhythm** (frantic, Barotrauma-paced): the web has noticed. A hull
  section fails, power drops, something wakes. Patch, re-route, retreat.
- **The hinge** is Attention. Light, noise, power spikes and depth raise it; it
  is shown in the world; crossing a line flips the rhythm. Dread is feeling it
  come.

### Short-Term (5–15 minutes) — the sortie

Fly the pod to a derelict → board → find one salvageable system (scrubber,
reactor cell, hull plate, data core) → extract it under rising Attention →
return → weld it in. Every wreck shows one visible unknown from outside; that
is the "one more sortie" pull.

### Session-Level (60–120 minutes)

Pick a target on the web map. Prep (O₂, tools, suppressants). Run 2–4 sorties.
Come home, integrate modules, re-stabilize the graph, read recovered logs.
Natural stop: the ship is stable again. The hook: the log names a deeper wreck.

### Long-Term Progression (days/weeks)

- **Ship capability** — life-support margin, range, hull.
- **Knowledge** — the web's rules, horror tells, decoded lore. Persists across
  crew loss. Some rules shift with depth so the web never becomes solvable.
- **Depth** — rings of the web, each with a higher Attention baseline.
- **Sanity scars** — permanent quirks a character carries.
- There is no win state. The center is a horizon.

### Retention Hooks

- **Curiosity**: the unknown on every wreck; the log that names a deeper one;
  who else is in the cryo pods.
- **Investment**: a ship that is literally assembled from your history.
- **Social**: a crew that needs you to be the sane one tonight.
- **Mastery**: reading the graph faster; knowing a tell; knowing when to doubt.

---

## Game Pillars

### Pillar 1: The World Never Lies
Pipes leak, frost forms, air hisses; the physical world is ground truth. Only
instruments and perception can be wrong. Three channels are **inviolable** —
no hallucination may ever alter them:
- **Physical consequences.** A pipe may *look* fixed, but the room still gets
  cold, the air still thins, the pressure still drops.
- **The sane palette.** Flesh tones and violet never appear in the sane world,
  and every hallucination leaks them somewhere — a fringe, a flicker, a tint at
  the edge of the light. Subtle is allowed; absent is not.
- **The human channel.** Voice/proximity chat and a crewmate's own light beam
  (colour and shape) are never faked. A monster that speaks in your friend's
  voice, or carries their torch, is your friend.

*Design test*: a hidden stat vs. a visibly leaking pipe → the pipe.

### Pillar 2: Dread Before Chaos
Every breach is preceded by a tell a careful player could have perceived
(sound, light, gauge drift, hull stress). This pillar owns tempo and
presentation; Pillar 3 owns causality.

*Design test*: breach fires at a hidden threshold vs. breach preceded by a
10–30 second escalating tell → the tell.

### Pillar 3: The Web Watches
The players' light, noise, power use and restraint raise or lower its
attention. Attention is shown in the world (creaking, flicker, filament
movement), never as a hidden meter. Depth sets the baseline; player action
moves it from there. Restraint and stealth are real strategies.

*Design test*: a flat spawn timer vs. attention-driven escalation → attention.

### Pillar 4: Sanity Rewrites Reality (and Crewmates Are Instruments)
Low sanity changes what the player perceives, not just their stats. Four
hallucination classes, each with its guard rail:
- **(a) Phantom entities.** Monsters that are not there; at depth, real ones
  hidden. A hidden real entity still leaves its P1/P2 footprint (sound, frost,
  hull stress, light disturbance) — the hallucination hides the body, never the
  trail.
- **(b) False system states.** Broken reads as fixed, fixed as broken; the lie
  extends from the gauge to the sprite. The physical consequence (P1) stays
  honest.
- **(c) Crewmates as horrors.** A teammate renders as a monster, and friendly
  harm — up to and including death — is possible. **System-gated**: harm to a
  crewmate is only possible while the *attacker's* sanity is verifiably below
  threshold, so a sane player cannot fake it. The victim sees a tell (the
  attacker's palette leak or a visible sanity sign). The game attributes the
  incident to the web — logs, scars — not to the player. Death here is like
  Project Zomboid's: it happens, and the failure contract (P5) is what makes it
  survivable for the crew.
- **(d) Phantom salvage.** A crate of treasure is garbage once it is home. At
  least one cheaper field check exists (carried weight, a second scanner, the
  palette leak); the trip home is the fallback, not the only reveal.

**No lie is unfalsifiable**: the truth is always available through the
physical world, a second instrument, a saner crewmate, the human channel, or
time — and checking costs time, risk or trust. At least one channel of truth
always remains. Solo play is supported through the first two checks; co-op adds
the human ones.

*Design test*: a hallucination that can never be disproved vs. one that costs
time, risk or trust to disprove → the costly one.
*Secondary test*: −10% move speed vs. a gauge showing a false value → the
false gauge.

### Pillar 5: Deeper Is Richer, Loss Is Not Total
Risk and reward map to depth in the web. The outer ring is kept unsafe by
depletion and wear (scrubbers degrade, salvage runs out), so there is no safe
place to farm forever. **Failure contract**: losing the home-ship or a crewmate
costs ship and body, never knowledge — decoded rules and lore persist, and the
next crew wakes knowing more.

*Design test*: safe farming vs. scarcity that pushes inward → scarcity.

**Intended tension**: Pillar 1 (legible) vs. Pillar 4 (lying). Resolved inside
the game world: the physical ship is ground truth; instruments and perception
are what can be wrong. The tension is the game.

### Anti-Pillars (What This Game Is NOT)

- **NOT a freeform ship-building sandbox**: welding is functional and modules
  are pre-shaped. Creative construction would dilute Dread Before Chaos and
  Deeper Is Richer.
- **NOT a combat game**: no weapon tree. Tools may repel or distract (flares,
  torches, decoys); nothing levels up from killing — including crewmates.
  Horrors are avoided and survived, not farmed. Protects The Web Watches.
- **NOT intentional PvP**: harm to a crewmate is only possible while the
  attacker's sanity is system-verified as low, is never rewarded, and is
  attributed to the web. Protects relatedness.
- **NOT an authored linear story**: no chapters or cutscenes; lore is found and
  assembled. Protects the long tail.
- **NOT a game you win**: no "escape the web" ending in v1.

---

## Visual Identity Anchor

**Direction name**: *What You Light, You Can Read* — a B-dominant hybrid of the
art-director's "The Dark Is the Default" and "Every Pipe Is a Sentence".

**One-line visual rule**: Light is a resource you spend; inside your light,
every pipe is a sentence; outside it, the dark is honest.

**Supporting visual principles**

1. *Darkness is honest; light is a decision.* Rooms are not fogged, they are
   unlit; geometry is there and any source reveals it.
   *Test*: a vignette / fog-of-war mask vs. true 2D occlusion from light
   sources with hard falloff → true occlusion.
2. *Inside the light, state lives in the sprite, not the icon.* Every system
   has a visible moving part and medium; failure renders as a trail (puddle,
   frost, scorch, mist) in the loop's hue.
   *Test*: a status LED vs. a turning rotor with visible fluid → rotor and
   fluid; a "COOLANT LOW" banner vs. a frosted elbow with a drip → the elbow.
3. *Emission is exposure.* Running loops glow in their own hue; with power off
   the ship is lit only by what is running. Every loop you run is a lantern the
   web can see; the web itself is seen only in your own light.
   *Test*: an attention meter vs. more filaments resolving in your light cone
   as attention rises → the filaments.

**Color philosophy**: Near-monochrome cold base (blue-black steel, graphite).
Human light is tungsten amber; alarm is red. Resource loops are flow-coded and
keep their hue everywhere they appear (working proposal: O₂ cyan, power amber,
coolant ice-blue, fuel orange-red, integrity neutral grey). The web has an oily
iridescence under direct light and otherwise no hue of its own. Deeper rings
do not change hue; they lower ambient and raise the iridescence. **Flesh tones
and violet appear nowhere in the sane world**, so when they appear, it is the
player's sanity, never the location.

**Sanity drift**: instruments first (needles disagree with the body of the
ship), then the light lies (cone reaches further than it does, shadows with no
occluder, temperature shifts), then loop hues bleed violet at the pipe edges
and flesh tones enter. **Anchors that never shift**: character sprites, hand
tools, the HUD frame, and the physical radius your light actually reveals when
you walk into it.

**Accessibility coordination** (for `/ux-design`): hue coding must be
redundant with shape and pattern (flow chevrons, pipe diameter) for colorblind
players.

**Pipeline note**: this direction puts the main art workload into the 2D
lighting pipeline rather than sprite-state count. The lighting-budget spike is
therefore the first spike, not the last.

---

## Inspiration and References

| Reference | What We Take From It | What We Do Differently | Why It Matters |
| ---- | ---- | ---- | ---- |
| Project Zomboid | Long-tail survival, lived-in detail, "one more day", base-with-friends | The base is mobile, modular, and made of other ships; no neighbourhoods | Validates patience for slow systemic survival and a decade-long EA model |
| Satisfactory | Legible machinery; flows you can read at a glance | The "factory" is a life-support system under eldritch pressure; no scale-up | Validates that watching a machine work is its own reward |
| Stellaris | Slow-reveal galactic horror; a universe that gets worse the more you know | Scale is one ship, not an empire; horror is personal and perceptual | Validates appetite for cosmic dread delivered by systems, not cutscenes |
| Barotrauma | Co-op crew under pressure in a hostile medium; breach rhythm | Boarding rhythm is slow and quiet; sanity is misinformation, not a stat | Validates co-op dread in a 2D vessel |
| Oxygen Not Included | Visible gas/heat/power simulation as the core puzzle | Far smaller graph, in service of dread rather than optimization | Validates a visible life-support graph as a core mechanic |

**Non-game inspirations**: *Beyond the Aquila Rift* (Love, Death & Robots) —
a ship trapped in a web, reality decaying, the kindness of the lie. *Pandorum*
— waking into wrongness aboard a derelict, not knowing how long you slept.

---

## Target Player Profile

| Attribute | Detail |
| ---- | ---- |
| **Age range** | 20–40 |
| **Gaming experience** | Mid-core to hardcore; comfortable with systems and slow pacing |
| **Time availability** | 1–2 hour evening sessions, often scheduled with friends; longer on weekends |
| **Platform preference** | PC, Steam |
| **Current games they play** | Project Zomboid, Barotrauma, Oxygen Not Included, Lethal Company |
| **What they're looking for** | Co-op dread with honest systems; a base that is theirs; a mystery that systems reveal, not cutscenes |
| **What would turn them away** | Jump-scare horror; fast onboarding expectations; freeform creative building; authored endings; PvP |

**Who this is NOT for**: competitors (no PvP or leaderboards), players who
want authored narrative payoff, freeform builders, anyone who needs a fast
start.

---

## Technical Considerations

| Consideration | Assessment |
| ---- | ---- |
| **Recommended Engine** | Godot 4.7.2, GDScript (pinned by `/setup-engine`). TD-FEASIBILITY: nothing argues for another engine; high-level multiplayer fits a small listen-server co-op |
| **Key Technical Challenges** | Composite welded ship (one physics body, sections reparented on weld); life-support graph as pure data separate from the scene tree; server-authoritative design from day one even in single-player; per-client hallucination presentation filter that never touches the simulation; readable procgen from room kits; 2D lighting budget on Vulkan (ADR-0008 § 1 pinned driver) *(Amended 2026-10-29 — ADR-0008 sync)*; versioned saves from v1 |
| **Art Style** | HD pixel art (high-resolution, detailed pixels), 3/4 oblique top-down, light-as-resource rendering |
| **Art Pipeline Complexity** | High — lighting pipeline is the main workload; modular derelict sections are the largest asset sink |
| **Audio Needs** | Adaptive (later tier). Attention and breach tells are audio-first; the ship must breathe |
| **Networking** | Listen-server (host-authoritative) via Godot high-level multiplayer. Host-owned saves, no live host migration. Steam lobby / NAT solution requires a technology evaluation before co-op work. Co-op gated behind the single-player slice |
| **Content Volume** | Full vision: 3 depth rings; several hand-authored room kits; dozens of module types; 4+ horror types; 4 hallucination classes; lore fragment set. Open-ended play; no fixed hours |
| **Procedural Systems** | Derelict interiors from kits + grammar, seeded; salvage placement by rules; host sends seed + deltas |

---

## Risks and Open Questions

### Design Risks
- The two-rhythm loop may not be tense and absorbing for 30+ minutes solo
  (the MVP's single question).
- Intense hallucinations may read as cheap deaths if the cost-to-verify is
  tuned wrong. "No lie is unfalsifiable" must hold in practice, not just on
  paper.
- Crewmates-as-monsters is lethal by design (user decision; CD accepted as
  non-blocking). The guard rails (sanity-gated harm, victim tell, web
  attribution) must hold in practice or the crew's distrust turns personal.
  Watch for players lowering their own sanity deliberately to unlock friendly
  harm: crossing the threshold must be slow and costly (scars, rising
  Attention). Playtest signal: friendly-fire deaths told as stories, and crews
  that keep playing together afterward. If groups split up, revert to a
  down-only cap.
- Persistent knowledge could make the web solvable; some rules must shift
  with depth.
- Deep resource simulation could swallow the dread (the Hull Pressure risk).

### Technical Risks (from TD-FEASIBILITY, CONCERNS)
- Authority discipline lapsing during the single-player phase → co-op becomes
  a rewrite. Enforce via ADR and control manifest; periodically run two local
  instances.
- Life-support graph entangled with the scene tree → untestable, unserializable.
- Save format not versioned from v1 → breaks every time a module changes.
- 2D lighting cost unmeasured for 4.7 on the pinned Vulkan driver (ADR-0008 § 1); Forward+ provisional vs. Compatibility *(Amended 2026-10-29 — ADR-0008 sync)*
  renderer is reversible and should be decided from spike data.
- Hallucination filter touching simulation state → endless desync bugs. Rule:
  every client receives the true value; corruption is presentation-only, with
  at least one channel of truth remaining, and a true-value debug overlay.
- Steam lobby / NAT traversal is a separate milestone from LAN co-op.
- 4.5–4.7 APIs are past the model's training cutoff; verify against
  `docs/engine-reference/godot/`.

### Market Risks
- Cosmic-horror survival is a niche within a niche; premium EA depends on a
  strong demo and word of mouth.
- Co-op arriving late may disappoint the audience most drawn to the pitch.

### Scope Risks (from PR-SCOPE, OPTIMISTIC → adjusted)
- First game, first engine: 1–3 months of learning folded into the MVP clock.
- HD pixel art throughput likely exceeds code time.
- No velocity data exists; every estimate is low-confidence until the spike
  phase produces some.
- Motivation over a 4–6 year solo horizon, not a deadline, is the real risk.

### Open Questions
- **What do guests keep** from a host-owned world (knowledge? sanity scars?
  nothing?) — for game-designer and creative-director; the save schema waits on
  it.
- **Who is in the other cryo pods, and why do they keep waking?** — for
  narrative-director; it is the failure contract's fiction.
- **Renderer choice** (Forward+ vs. Compatibility) — answered by the lighting
  spike.
- **Steam lobby / NAT solution** — technology evaluation before VS-B.

---

## MVP Definition

**Core hypothesis**: The boarding→breach two-rhythm loop, over a visible
life-support graph, is tense and absorbing for a 30+ minute solo session.

**Required for MVP** (single-player):
1. One handmade spawn derelict and one handmade target derelict, with an
   abstracted pod hop between them.
2. Visible O₂ / heat / power graph with propagating, physically rendered
   failures.
3. One weld, one player seam cut, and one install: attach one section and join
   its nodes to the graph; cut a section free at a seam; install a salvaged
   module. Co-op is still not in MVP. *(Amended 2026-10-27 — user rulings: X1 / D52, "Resolved 2026-10-27 — user ruling (design-review decisions)": weld + player seam cut + install are all MVP.)*
4. Attention with one threshold and one horror behaviour.
5. Sanity as a single hallucination class (false system state), falsifiable by
   the physical world.
6. Save/load v1 with a format version.
7. All state changes through a single command/simulation layer (no networking
   code, no replication).

**Explicitly NOT in MVP**: procedural generation, co-op, multiple horrors,
other hallucination classes, lore, Steam, art at final fidelity, adaptive
audio.

**Kill/continue gate at the end of MVP**: playtest with at least three people
outside the project. If a 30+ minute solo session is not tense and absorbing,
iterate on the loop before any VS-A work. No procgen on an unproven loop.

### Scope Tiers (if budget/time shrinks)

| Tier | Content | Features | Timeline (solo; low confidence until spikes produce velocity) |
| ---- | ---- | ---- | ---- |
| **Spike phase** | Throwaway prototypes | Weld on a moving ship (tilemap + collision + graph merge); graph with spreading failures; save round-trip with version; 2D lighting budget on Vulkan (ADR-0008 § 1 pinned driver) *(Amended 2026-10-29 — ADR-0008 sync)* | 4–8 weeks |
| **MVP** | 2 handmade derelicts | Core loop only (list above) + playtest gate | 6–9 months full-time / 10–16 part-time |
| **VS-A** (single-player, **the shippable fallback**) | Outer ring of the web map; one procgen room kit | Procgen derelicts; 2nd horror type; all 4 hallucination classes with falsification checks; one derelict kit at target HD-pixel fidelity under the lighting model | +8–12 months |
| **VS-B** (co-op) | Same content | 2-player LAN listen-server; authority model proven with two local instances | +4–6 months, only after VS-A is complete |
| **Alpha** | 3 depth rings; full kit set; lore fragments | Steam lobbies / NAT; crew scars and knowledge persistence (failure contract); versioned saves; host-owned world + guest carry-over; true-value debug overlay | +12–18 months |
| **Full Vision** | Early Access launch | Polish; adaptive audio; accessibility; onboarding | ~4–6 years total |

**What ships if time runs out**: VS-A, as a small single-player survival-horror
game. Co-op and depth rings are cut; the product is whole.

---

## Next Steps

- [x] Concept reviewed by creative-director (CD-PILLARS APPROVE)
- [x] Engine configured (`/setup-engine` — Godot 4.7.2, GDScript)
- [ ] `/art-bible` — from the Visual Identity Anchor above; required before Technical Setup
- [ ] `/design-review design/gdd/game-concept.md`
- [ ] `/prototype` — the spike phase (weld, graph, save, lighting budget)
- [ ] `/map-systems` — decompose into systems
- [ ] `/design-system [system]` — per-system GDDs
- [ ] `/create-architecture`, then ADRs for: authority model, life-support graph data model, hallucination presentation layer (with the one-channel-of-truth rule), save ownership
- [ ] `/architecture-review`
- [ ] `/gate-check`
- [ ] `/vertical-slice` in Pre-Production; `/playtest-report` at the MVP gate
