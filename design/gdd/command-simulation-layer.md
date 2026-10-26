# Command & Simulation Layer

> Authoring guidance: .claude/docs/templates/guidance/game-design-document-guide.md (load per-section as you author — do not read entirely).

> **Status**: Designed
> **Author**: user + systems-designer, gameplay-programmer, engine-programmer, creative-director, qa-lead (via /design-system, autonomous)
> **Last Updated**: 2026-10-25
> **Last Verified**: 2026-10-25
> **Implements Pillar**: The World Never Lies (P1 — one authoritative truth); Sanity Rewrites Reality (P4 — perception can never write simulation)
> **Creative Director Review (CD-GDD-ALIGN)**: APPROVED 2026-10-25 (after one CONCERNS round: kernel over-claim on rejection withdrawn; `persist_scope` WORLD/PROFILE added for Pillar 5; five notes applied)

## Summary

The Command & Simulation Layer is the thin kernel every other system runs inside: a fixed 60 Hz tick, a single door through which every change to the game's state passes as an authority-checked, peer-stamped **command**, a registry of **state blocks** that domain systems own and the kernel never does, and a once-per-tick **change notification** that presentation reads but can never write back through. The player never sees it; what they get from it is a ship that cannot lie — gauges and sprites may be hallucinated, but there is exactly one authoritative truth beneath them, and every command is judged against it. The kernel does not make that truth visible — how, when and at what cost the player learns why a command failed is priced by Interaction, Sanity and the HUD under Pillar 4's verification rule. It exists so that *The World Never Lies* and *Sanity Rewrites Reality* are structural rather than aspirational, and so that co-op can be added later without a rewrite: single-player is simply the host with one local peer.

> **Quick reference** — Layer: `Foundation` · Priority: `MVP` · Key deps: `None` (upstream); 16 downstream dependents, all MVP/VS systems register with it

## Overview

The Command & Simulation Layer is the thin kernel every other system of The Synaptic Sea runs inside. It advances the simulation at a fixed tick, accepts every change to the game's state as a **command** carrying the identity of the peer that issued it, checks that peer's authority, validates the command against current state, applies it or rejects it with a reason, and then tells presentation what changed. It owns no domain state of its own: the Life-Support Graph, the ship layout, suit vitals, Attention, sanity and the rest each register their own **state block** with the kernel's registry and define their own command handlers. The kernel's job is to be the one door through which truth changes.

The player never sees this system; they interact with it only through the systems built on it — every weld, every valve turned, every step taken is a command the kernel admitted. It is automatic from the player's point of view and invisible when it works.

It exists for two reasons that are both pillars. First, **The World Never Lies** needs one authoritative truth: if any system could mutate state directly, two readers could disagree about whether a pipe is leaking, and the honest-world promise would be unenforceable. Second, **Sanity Rewrites Reality** needs a wall: the Perception Filter and every other presentation script may *read* simulation state and write only what is drawn, never what is true. The kernel is that wall made structural rather than aspirational, and its registry is also the hook the Save/Load contract and the true-value Debug Overlay hang from. The game would lose, without it, the ability to add co-op later without a rewrite (TD-FEASIBILITY: authority discipline from day one — in single-player the local peer simply *is* the host) and the guarantee that a hallucination can never corrupt the ship.

**Design-level only.** Whether the kernel is an autoload, how commands are encoded, and how change notification is delivered are implementation choices → **becomes an ADR** (authority model). *(Framing: technical infrastructure, no ADR yet to cite, no direct player fantasy — selected as the recommended options for a `Core`/`Foundation` system under `autonomous` automation.)*

## Player Fantasy

*Indirect — the player never meets this system; they meet a promise it keeps.*

**There is only one ship.** A gauge in the scrubber bay can read green while frost climbs the elbow beside it, and the player has to decide which to believe. The answer is always the same, and it is always available: the gauge and the frost both read the same state, and only one of them passed through the player's mind on the way to the screen. Nothing the player sees, hears or imagines can reach back and change what the ship is doing.

The anchor moment is in the cryo bay or on the first sortie. The player has "fixed" a scrubber and the needle has settled. Then their breath fogs, and they realise the room is still getting colder. The horror is personal: *I am the thing that is broken.* The comfort is structural: *the ship is not.* Each time a player checks and the world turns out to be right, they learn to trust the world more and their own eyes less. That trust is the only thing this system gives the player, and one physical consequence that disagrees with what the ship is actually doing would spend it.

In co-op, the same rule gives every act a name. A sane crewmate cannot fake the web's violence, and the beam in the corridor belongs to whoever's beam it is.

**Pillars served.** *The World Never Lies*: "Pipes leak, frost forms, air hisses; the physical world is ground truth. Only instruments and perception can be wrong." *Sanity Rewrites Reality*: "Broken reads as fixed, fixed as broken; the lie extends from the gauge to the sprite. The physical consequence stays honest." The concept's own resolution of their tension — "the physical ship is ground truth; instruments and perception are what can be wrong. The tension is the game" — is what this kernel makes structurally true.

**Design tests.**
- If we are debating whether a presentation-side effect (a hallucination, a light flicker, a UI animation) may adjust simulation state for convenience, or must be routed through a command: route it, or don't do it.
- The fantasy has failed if any player-visible physical consequence ever disagrees with the authoritative state. That is a Pillar 1 violation, not a cosmetic bug — and the reason the Debug Overlay exists.

*What this section does not claim*: the kernel does not make the truth visible. Failure & Trail, Lighting's true reveal radius and the human channel do that. The kernel is why there is only one truth for them to show. (Alternatives considered: a co-op-first "identity behind every act" framing — true but untestable before VS-B and over-credits the kernel; a "steady pulse" fairness framing — honest about the tick but mostly the Graph's and Attention's fantasy.)

## Detailed Design

### Core Rules

*Specialists consulted: systems-designer (primary), gameplay-programmer (feel/feasibility), engine-programmer (Godot 4.7.2 integration). Their one disagreement — tick rate — is resolved in CR-1. Items tagged **→ ADR** are engine-integration choices deliberately left out of this behaviour spec.*

**Vocabulary.** *Tick*: one fixed-step iteration of the authoritative simulation; tick number `T` is a monotonically increasing integer and the only clock the simulation knows. *State block*: a named, versioned, single-owner container of domain state registered with the kernel. *Command*: a serializable request to change state, carrying issuer identity. *Owner*: the domain system that is the only writer of a block. *Read view*: the read-only accessor an owner publishes. *Step hook*: owner-registered code that runs in the propagation phase. *Mode*: `HOST` (authoritative; the only mode in MVP) or `MIRROR` (VS-B guest: receives state, runs no authoritative tick) — the public API is identical in both. *PlayerId*: a stable seat identity (1 in single-player); *peer_id*: the transport identity of a connection (host = 1). Persistent per-player state is keyed by PlayerId, never by peer_id, because peer IDs are reassigned per connection (this corrects the systems index's "keyed by peer ID" for Sanity and Suit).

**CR-1. Fixed tick at `TICK_RATE` = 60 Hz, with per-hook step divisors.** The simulation advances in steps of exactly `1/TICK_RATE` seconds of *simulated* time, independent of render frame time; durations everywhere are in ticks (the Attention tell window of 10–30 s is 600–1800 ticks). The kernel never reads wall-clock time or `delta`. Input- and physics-adjacent hooks run every tick; slow systems run at a divisor (Graph 6 → 10 Hz, Attention 6, Suit vitals 6, Sanity drift 12 → 5 Hz).
*Rationale and alternatives.* The gameplay-programmer argued for a 20 Hz kernel (slow systems don't need more; worst-case 50 ms latency is invisible behind immediate local acknowledgement) but set a hard floor of 15 Hz and would push back at 10. The engine-programmer argued for tick = physics rate so the moving ship body and the characters inside it share one timeline. 60 Hz + divisors satisfies both: one clock, ≤17 ms input-to-effect, and the cost profile of a 10 Hz graph. A variable timestep was rejected (breaks replay, tests and save equivalence); a 30 Hz kernel sits at an awkward ratio to Godot's 60 Hz physics step. The tick *source* (physics callback vs accumulator) → ADR; the recommendation is a single node's physics callback driving a plain `step()` method that tests and a future server call directly. Changing `TICK_RATE` later invalidates every tuned value expressed in ticks.

**CR-2. Tick phase order.** Each tick executes these phases to completion, in order, with no interleaving:

| # | Phase | What happens | Who may write blocks |
|---|-------|--------------|----------------------|
| P0 | Begin | Tick counter increments; the intake queue is *frozen* (commands arriving now wait for T+1) | nobody |
| P1 | Intake | Frozen queue sorted by the ordering key (CR-13), truncated to `MAX_CMDS_PER_TICK`; overflow stays queued in order | nobody |
| P2 | Dispatch | Per command in order: authority check → validate → apply or reject (CR-5) | the handler, inside `apply` only |
| P3 | Step | Step hooks in resolved order (CR-3); a hook runs when `T mod divisor == offset` | the hook's own block, plus owner-published mutators of other blocks (CR-9) |
| P4 | Commit | Dirty sets finalised; "tick committed" marker advances; interpolation baseline captured | nobody |
| P5 | Notify | Batched change notifications delivered (CR-10) | nobody — writes disabled |
| P6 | Boundary services | Pending snapshot (Saving), state-change requests, command-log append | the kernel only |

P2 commands see earlier P2 effects in the same tick; P3 sees all of P2; subscribers and presentation only ever see post-P4 state, never a half-applied tick.

**CR-3. Step hooks and ordering.** Hooks register `(hook_id, owner, divisor, offset, after: [hook_ids])`. The kernel resolves a total order by topological sort of `after`; ties break by registration order, then `hook_id`. A cycle or unknown `hook_id` fails boot (CR-6). Sim-to-sim data flow within a tick is "reader declares `after` writer" (Suit vitals `after: graph.propagate`). The order is data: testable, and printable by the Debug Overlay. Hooks may not submit `PLAYER`-origin commands; they may submit `SYSTEM`-origin ones (CR-4).

**CR-4. Command anatomy, origin and intents.** A command is plain serializable data:

| Field | Meaning |
|-------|---------|
| `type` | Registered command type ID, owned by a domain system |
| `issuer_peer` | Stamped by the kernel's ingress — never taken from the payload |
| `origin` | `PLAYER` or `SYSTEM`, stamped by ingress |
| `actor` | The entity the command acts as (opaque ID), or none |
| `seq` | Per-issuer monotonically increasing integer |
| `payload` | Primitives, arrays, dictionaries, packed arrays only — no object references, node paths or callables; ≤ `MAX_PAYLOAD` |
| `queued_tick` | Stamped on enqueue |

*Discrete commands vs continuous intents.* If the sim must log, validate, save or replay it, it is a command (`begin_cut`, `begin_weld`, `begin_patch`, `cancel_channel`, `install_module`, `remove_module`, `toggle_beam`, `interact`, `pickup`, `drop`, `use_item`, `pod_hop`, `set_move_intent`, `set_aim`). What the stick is doing *right now* is an intent: intents are commands too, but a type may declare `coalesce = LATEST_PER_ACTOR_PER_TICK` — coalescing happens at admission over the unfrozen queue, keyed `(type, actor, issuer_peer)`, the older command becoming `Superseded` — and the input adapter sends one only when the quantised value changes — a few per second, never one per tick. Edge-triggered inputs (press, release, confirm) must declare `coalesce = NONE` or taps are lost. Movement integration itself runs on the physics side reading the current intent from the Controller's block; the sim sees position as a read (ownership of position is settled between Player Controller and Ship Body — see C.3). In debug builds every command is round-tripped through serialization before dispatch, even locally; one that cannot survive is a defect. **Submit is fire-and-forget**: the only return is a result event keyed by `(issuer_peer, seq)` — a remote issuer could never get a synchronous answer, so the local one doesn't either. Immediate *acknowledgement* (a tool sound, a hand animation, a progress ring starting) is presentation's job and happens in the same frame; the sim confirms on the next tick.

**CR-5. Command lifecycle and the exactly-one-result guarantee.** Every command reaches exactly one terminal state and emits exactly one result event to its issuer. Stages: *Issued* → *Admission* (kernel state accepts commands; type registered; queue not full; `seq` not a duplicate — gaps are accepted and counted, not rejected; issuer rate limit) → *Queued* → *Authority check* in P2 (origin allowed for the type; if the type declares `actor_binding = OWNED_BY_ISSUER`, the binding table says the issuer controls `actor`; age ≤ the type's `max_age_ticks`) → *Validate* (owner's side-effect-free `validate(cmd, read_views)` returns OK or a domain reason) → *Apply* (`apply(cmd, write_scope)`; infallible by contract once validate passed, CR-9) → terminal:

| Terminal | Meaning |
|----------|---------|
| `Applied` | Effects are in state as of tick T |
| `Rejected(reason)` | No effect |
| `Dropped(reason)` | Never evaluated: a reload or shutdown flushed it — or `Dropped(FAULTED)`: its `apply` violated the infallibility contract and faulted the kernel (added so G5 holds on the defect path) |
| `Superseded` | Coalesced away by a newer command of a coalescing type |

Kernel reject reasons are a closed set: `NOT_ACCEPTING`, `PAUSED`, `UNKNOWN_TYPE`, `QUEUE_FULL`, `RATE_LIMITED`, `DUPLICATE_SEQ`, `ORIGIN_NOT_ALLOWED`, `ACTOR_NOT_OWNED`, `STALE`, `MALFORMED`, `VALIDATION_FAILED(domain_code)` — domain codes are namespaced and owned by the registering system (Interaction's set, e.g. `OUT_OF_RANGE`, `INVALID_TARGET`, `PRECONDITION`, `BUSY`, is its GDD's). The result event carries `seq`, `issuer_peer`, `ok`, `reason`, `subject` (the target the rejection refers to), an optional `detail` dictionary of numbers, and `tick` — **never display strings**. Presentation maps reasons to sound, shake or tooltip. **Reason codes are computed from true state and the kernel never filters them.** That is the whole of the kernel's claim. Whether, when and how cheaply a reason reaches the player is *not* the kernel's: Interaction, Sanity and the HUD own it and must obey Pillar 4's cost rule (a check costs time, risk or trust) — for example surfacing a `PRECONDITION` only after channel time has been spent, or only through diegetic feedback rather than an instant tooltip. *Design test (CD-GDD-ALIGN):* if a rejection lets the player falsify a hallucination at zero time, risk or trust cost, the presentation of that rejection is wrong — not the kernel. A command *aborted mid-channel* (power drops during a cut) is reported by the channel owner through state change, not through the command result. The kernel never retries.

**CR-6. Registration at boot, then frozen.** Block, command-type and hook registration are legal only in `BOOTING`; after `BOOTING → LOADING` the registries are immutable until shutdown. Registration is an explicit declaration by the composition root, not discovery via the scene tree. Duplicate IDs, unknown types, missing owners and hook cycles fail boot loudly in every build.

**CR-7. Authority model (behaviour).** The host is the sole authority: only a `HOST` kernel ticks and applies; a `MIRROR` never applies. Each command type's owner declares `allowed_origins ⊆ {PLAYER, SYSTEM}` and `actor_binding ∈ {NONE, OWNED_BY_ISSUER}`. `SYSTEM` commands never arrive from a network; remote ingress always stamps `PLAYER`. The kernel holds a **binding table** — `peer_id → PlayerId` and `PlayerId → {controlled entity IDs}` — as identity data, not domain state; it is mutated only through a host-only API called by the session owner (a boot step in MVP; Co-op Session in VS-B) and is not saved, since sessions rebuild it. *(Alternative: Player Controller owns the binding as a block — rejected because the kernel could then not authority-check without calling a domain system. → ADR.)* **Single-player is not a special case**: local input is `PLAYER`-origin peer 1 acting on an entity peer 1 is bound to; the authority stage runs on every command from day one, so TD-FEASIBILITY's "authority discipline lapsing" risk is structural, not disciplinary. The kernel checks *who*; owners validate *whether the world allows it*; the kernel encodes no game rules. For friendly harm the kernel checks only issuer→actor ownership; the "attacker below the sanity threshold" gate is Interaction's `validate` calling Sanity's query on the actor's authoritative sanity level (a `SIM` block), never on anyone's perception.

**CR-8. State-block registry contract.** An owner registers: `block_id` (unique, namespaced, e.g. `graph.state`); `owner`; `schema_version` (bumped on any serialized-shape change); `class` ∈ {`SIM`, `PERCEPTION_OUT`} (CR-12); `persist` (false = derived, rebuilt after load) and, when true, `persist_scope` ∈ {`WORLD`, `PROFILE`} — `WORLD` blocks live in the world snapshot and roll back on `request_restore`; `PROFILE` blocks (decoded rules, lore, scars: Pillar 5's "never knowledge") live outside it and are never rolled back by a world restore; `scope` ∈ {`SHARED`, `PER_PLAYER` keyed by PlayerId}; `repl_hint` ∈ {`HOST_ONLY`, `SHARED_SUMMARY`, `OWNER_PRIVATE`} — mandatory now though unused until VS-B, so no block ever ships without the decision; `snapshot()` (plain data, canonical state only, deterministic ordering, entity references by stable ID, no derived caches); `restore(data, stored_version)` (whole-block replace; the owner migrates from `stored_version`; an unsupported version is a reported failure, never a silent default); `default_state()` (when a save lacks the block); `post_restore()` (after *all* blocks restore: resolve cross-block IDs, rebuild caches, check invariants); `read_view()`. Restore order follows declared `restore_after`. Entity IDs are allocated by the owning block from a counter stored inside it; the kernel allocates none. **No global RNG in simulation**: each system draws from a named stream whose counter lives in its block; the kernel supplies a pure derivation helper from `(world_seed, stream_name, counter)`. The kernel's own clock (`kernel.clock`: tick counter + world seed) is its one registered block and contains no domain facts. Blocks are typed data objects with a canonical `to_dict`/`from_dict` interchange form of JSON-safe primitives (→ ADR for Resource vs RefCounted); every block has a round-trip test. **4.7 gotcha (sourced)**: writing one element of a packed array no longer fires the property setter, so dirty-tracking must be explicit in mutation methods, never in setters.

**CR-9. Write discipline and atomicity.** A block is written only by its owner's mutators, and only inside a `WriteScope` the kernel issues during P2 apply or P3 step; outside those phases mutators assert in debug and are rejected in release. **Cross-block commands** (weld, cut, module install, pod hop) call *public mutators* other owners explicitly publish; the kernel guarantees the whole `apply` is atomic against every other command, hook and snapshot, followed by one notification batch. **Failure model: validate-then-apply, no rollback.** `validate` proves everything through read-only queries (for weld, the Graph's pure join-validation query); `apply` is infallible by contract; a failure inside `apply` is a defect that moves the kernel to `FAULTED` (C.2). *(Copy-on-write rollback rejected for MVP cost; event sourcing rejected as primary store — the kernel keeps a bounded command log for diagnostics and replay tests only.)* Step hooks affect other systems by (a) a published unconditional mutator (a horror writing a footprint into the Graph's footprint channel) or (b) a `SYSTEM`-origin command processed at T+1 when the target may reject. The kernel enforces *phase*; "only declared callers call declared mutators" is enforced by lint and the control manifest. → ADR

**CR-10. Change notification contract.** Notifications are emitted once per tick in P5, after commit, batched per block as `(block_id, tick, change_keys)`; if more than `MAX_DIRTY_KEYS` changed, `change_keys` collapses to `ALL`. **Payloads carry no state**; subscribers fetch through read views, so notifications are identical whether state arrived by command (`HOST`) or by replication (`MIRROR`) and presentation never changes between modes. Subscriber classes are declared at subscribe time:

| Class | May do | Examples |
|-------|--------|----------|
| `PRESENTATION` | Read `SIM` and `PERCEPTION_OUT` views; write only CanvasItem / Material / Light2D / particle / audio nodes | Failure & Trail, Instruments, HUD, Lighting, Perception Filter, Audio |
| `INPUT_SURFACE` | As `PRESENTATION`, plus **submit** commands as the local seat | Player input adapter, Chart Table, menus |
| `PROJECTOR` | Same reads; write engine-side *derived* objects owned by its block owner (physics shapes, bodies, scene content); never writes blocks | Layout → physics projection; derelict content build |
| `REPLICATION` | Read views marked for replication; send | Co-op adapter (VS-B) |
| `DIAGNOSTIC` | Read everything, including kernel internals | Debug Overlay |
| *(none for SIM)* | Simulation systems never subscribe; sim-to-sim is ordered step reads or commands | — |

Subscriber handlers run with writes disabled. Render `PRESENTATION` gets no submit API; `INPUT_SURFACE` is the one class that may submit, which is how "presentation never writes simulation" coexists with menus that issue commands. *(The `INPUT_SURFACE` and `PROJECTOR` readings of the TD rule are provisional pending a technical-director ruling — Open Questions.)* **Sim never calls presentation**: the flow is always state → notification → presentation. Horror light disturbance and Attention flicker are state that Lighting reads, not calls into Lighting. There are two consumer channels: true values (Debug Overlay only) and filtered values (the Perception Filter reads true state and publishes what a client perceives; presentation nodes read the filtered channel). The kernel exposes `tick_alpha` for render interpolation; owners of moving state expose previous and current committed values in their read views. → ADR for transport (signals vs registry).

**CR-11. Time model for presentation.** Presentation runs at render rate and reads last-committed state. It never causes a tick; pausing presentation effects (reduced motion) never pauses the sim.

**CR-12. Perception isolation, enforced by block class.** `SIM` blocks are readable by simulation code and presentation. `PERCEPTION_OUT` blocks (Sanity's per-player "what lies to show" output) are *write-only to simulation*: no sim code — including the owner's own hooks — obtains a read view; presentation and the replication adapter do. Sanity's drift and the friendly-harm gate read the sanity *level* (`SIM`), never the perception block. A sim-side read of a `PERCEPTION_OUT` view asserts in debug and returns nothing in release. Perception state is `persist = false` by default (reproducible from sanity level + seed) and `OWNER_PRIVATE` in VS-B.

**CR-13. Determinism and ordering guarantees** (`HOST` mode, same build and platform). **G1 Order**: commands apply in ascending `(queued_tick, host_order)`; in single-player that is submission order. **G2 Single timeline**: every command applies at exactly one tick. **G3 Replay**: from an initial snapshot, the same command stream and the same *recorded physics observations* yield identical block state at tick N (canonical snapshot hash). **G4 Save equivalence**: snapshot at T, restore, continue ≡ uninterrupted play for `persist` blocks (hash-compared) — the key regression test for CR-8. **G5** exactly-one-result (CR-5). **G6 Atomic ticks** (CR-2). Physics-derived facts (positions after collision, contacts, raycasts) are *inputs* from the engine: Godot Physics 2D makes no determinism claim, so G3 either records and replays them or excludes physics-coupled blocks. This is a stated limit, not a defect; host-authoritative co-op needs host truth, not cross-machine lockstep. → ADR

**CR-14. Hard rules** — each becomes a lint, a test, or a control-manifest line. **H1** No domain state in the kernel (kernel-owned data is exactly: `kernel.clock`, the command queue, the registries, the binding tables, the command log, kernel state, diagnostics). **H2** No kernel-defined domain commands; its control plane (CR-15) is a host-local API, not commands. **H3** One owner per block; writes only via owner mutators inside a `WriteScope`. **H4** Presentation never writes simulation; render presentation has no submit API. **H5** No simulation system reads perception state. **H6** Peer identity on every command, kernel-stamped, never payload-borne; a command constructor without `issuer_peer` must not exist. **H7** Everything crossing the kernel boundary is plain data. **H8** No networking code in MVP: the only ingress is local input, the only egress local presentation. **H9** Simulation never reads wall-clock or global RNG. **H10** Kernel logic is scene-tree independent — runs headless, testable without a scene; a `Kernel.step()` call advances the sim synchronously in tests. Autoload vs node → ADR; if an autoload exists it is a thin holder of an instance that code receives as a parameter, so tests never share global state.

**CR-15. Control plane** (host-local, not commands): `request_pause`, `request_resume`, `request_snapshot`, `request_restore`, `request_shutdown`. Callers: Save/Load, the pause/Chart Table UI (host-local call), boot code. Honoured at tick boundaries (P6). In VS-B a guest's pause or save *request* is a command defined by Session or Save/Load, validated on the host, which then calls the control API.

**CR-16. Pause semantics.** `PAUSED` stops the tick but not rendering. While paused, submit accepts only types flagged `allowed_when_paused` by their owner, and **only non-diegetic actions may carry the flag** (save, settings, resume, menu navigation) — no gameplay verb that changes `SIM` state is allowed while paused, so a solo player cannot equip, repair or re-route with time stopped mid-tell (Dread Before Chaos), and solo pause behaves like co-op's overlay-only pause; a paused-allowed command may write only blocks their owner registers as non-gameplay (settings, menu, save-request state), never a `SIM` gameplay block; everything else is rejected `PAUSED` immediately, never queued, so there is no burst on resume. Admitted paused-allowed commands are dispatched by a **paused service cycle** that runs P1, P2, P4, P5 and P6 with no P3 and with `T` unchanged, so a save or a menu action takes effect without advancing simulated time. Commands already queued when the pause is honoured are neither dispatched by the service cycle nor rejected: they wait, in order, for the first `RUNNING` tick after resume. **Pause is single-player only**: in co-op a menu cannot stop a shared sim — VS-B treats pause as a UI overlay with the kernel `RUNNING`. The Chart Table GDD must not assume "pick a target" happens while paused. Relationship to the scene tree's pause → ADR.

**CR-17. Bounds and overload behaviour.** At most `MAX_CMDS_PER_TICK` processed per tick, the rest waiting FIFO. `QUEUE_CAPACITY` total and `PER_ISSUER_CAPACITY` per issuer; beyond that `QUEUE_FULL` at admission — never silent drop, never unbounded growth. Catch-up runs at most `MAX_TICKS_PER_FRAME` ticks per frame, then lets simulated time lag rather than spiral; a lag counter is exposed to diagnostics. P3 exceeding `STEP_BUDGET_MS` logs the slowest hooks but the tick completes. A command older than its type's `max_age_ticks` is rejected `STALE`, so a hitch cannot unleash a second of stale intents. At the proposed defaults `STALE` is unreachable from local ingress (the queue can hold a command at most 8 ticks and `T` does not advance while paused or saving); it is a VS-B seam for remote-stamped ticks, guarded locally by a "never stale" invariant test (Section E).

**CR-18. Snapshot semantics.** A snapshot is taken only at a tick boundary (P6), covers all `persist` blocks at the same T, and records `(T, per-block schema_version)`. Capture is synchronous and in memory with the tick halted (`SAVING`); file I/O is Save/Load's and off the critical path → ADR. Restore replaces all `WORLD`-scoped persisted blocks atomically and leaves `PROFILE`-scoped blocks untouched (Pillar 5: losing the ship or a body never loses knowledge); partial restore within a scope is unsupported; failure aborts to `FAULTED` with pre-restore state intact where possible. With join-in-progress deferred, the same snapshot/restore contract is VS-B's guest transfer at session start, so it must be complete and cheap.

**Velocity rule (gameplay-programmer, adopted as an acceptance criterion).** Adding a new command verb touches at most two files: one handler and one registration line. If the kernel ever makes a feature cost more than that, the layer will be bypassed — which is the exact lapse TD-FEASIBILITY named. Hence: commands are plain data records (no class per verb), state blocks are coarse (one per domain system, not per entity), and replay/undo/rollback/middleware are deferred.

### States and Transitions

**Kernel states**

| State | Meaning | Tick advancing | Accepts submit |
|-------|---------|----------------|----------------|
| `BOOTING` | Registries being populated (CR-6); nothing loaded | No | No (`NOT_ACCEPTING`) |
| `LOADING` | Restoring a snapshot or building new-game state; also entered mid-session for reload, including the MVP death stub (Save/Load) | No | No (`NOT_ACCEPTING`) |
| `RUNNING` | Normal ticking | Yes | Yes |
| `PAUSED` | Single-player pause (CR-16) | No | Only `allowed_when_paused` types |
| `SAVING` | Snapshot capture at a tick boundary; transient, returns to the prior state | No | Queues normally; no tick runs until capture completes |
| `FAULTED` | Invariant violation, apply failure, restore failure, or boot failure after registration. Never requested | No | No |
| `SHUTTING_DOWN` | Teardown | No | No |

`FAULTED` exists because CR-9's "apply cannot fail" contract needs somewhere for a violation to go other than undefined behaviour. In `FAULTED` the kernel never writes or autosaves over an existing save; a dev build halts with diagnostics, release returns to the menu.

**Transitions**

| From | Trigger | To | Notes |
|------|---------|----|-------|
| (start) | Process start | `BOOTING` | |
| `BOOTING` | Registries valid and frozen | `LOADING` | A new game also passes through `LOADING`, populating blocks from `default_state()` |
| `BOOTING` | Duplicate ID, hook cycle, unknown dependency | `FAULTED` | |
| `LOADING` | All blocks restored and `post_restore` passes | `RUNNING` | Or `PAUSED` if the caller requested it |
| `LOADING` | Restore or `post_restore` fails | `FAULTED` | Previous state preserved if possible |
| `RUNNING` | `request_pause` | `PAUSED` | At the next tick boundary |
| `PAUSED` | `request_resume` | `RUNNING` | |
| `RUNNING` / `PAUSED` | `request_snapshot` | `SAVING` | Remembers the return state |
| `SAVING` | Capture complete | prior state | |
| `SAVING` | Capture error | `FAULTED` | |
| `RUNNING` / `PAUSED` | `request_restore` (load, reload, death stub) | `LOADING` | Queue flushed; every flushed command gets `Dropped(RELOAD)` |
| `RUNNING` / `PAUSED` / `FAULTED` | `request_shutdown` | `SHUTTING_DOWN` | Queue flushed with `Dropped(SHUTDOWN)` |
| `RUNNING` | Apply failure or step invariant failure | `FAULTED` | |
| `FAULTED` | `request_restore` (explicit user action) | `LOADING` | The only exit other than shutdown |
| `SHUTTING_DOWN` | Teardown complete | (end) | |

Every other `(state, trigger)` pair is invalid: the kernel ignores it and reports it to diagnostics, never silently changing state. No transition leaves `SAVING` except to its return state or `FAULTED`; a restore requested during `SAVING` waits for the boundary; there is no `BOOTING → RUNNING`.

**Per-command states**

| State | Entered when | Next |
|-------|--------------|------|
| `ISSUED` | submit called | `ADMITTED` or `REJECTED` |
| `ADMITTED` | passes admission | `QUEUED` |
| `QUEUED` | enqueued | `AUTHORIZING`; or `DROPPED` on flush; or `SUPERSEDED` by a coalescing successor |
| `AUTHORIZING` | reached in P2 | `VALIDATING` or `REJECTED` |
| `VALIDATING` | authority passed | `APPLYING` or `REJECTED` |
| `APPLYING` | validate OK | `APPLIED` (an apply failure faults the kernel) |
| `APPLIED` · `REJECTED(reason)` · `DROPPED(reason)` · `SUPERSEDED` | terminal | exactly one result event emitted (CR-5) |

### Interactions with Other Systems

**All interfaces below are provisional — no consumer has a GDD yet.** The kernel owns the mechanism (registry, dispatcher, phases, subscriber classes); each consumer owns what it registers (block schemas, command types, hooks). Three rules for every consumer, carried into the control manifest: a system that wants state to persist registers a block and never talks to Save/Load directly; a system that wants to change another's state publishes a command or a mutator and never reaches into another's block; a system that wants to *show* state subscribes and reads and never calls into the simulation.

| # | System | Flows into the kernel | Flows out of the kernel | Interface owner | Provisional notes |
|---|--------|-----------------------|-------------------------|-----------------|-------------------|
| 1 | **Life-Support Graph** | Blocks: topology, flow/failure/atmosphere state (`persist`, `SIM`, `SHARED_SUMMARY`); propagation hook at divisor 6; published mutators for the footprint channel and for Assembly's weld/cut | Tick base; ordered step; dirty keys to trails, instruments, audio; `read_view` for Suit and Attention | Kernel: mechanism · Graph: schemas, mutators | Dirty-key volume on large graphs sets `MAX_DIRTY_KEYS`. The Graph's index edge on Save/Load is satisfied by the registry contract (index correction candidate). Must stay scene-tree free (H10) |
| 2 | **Player Controller & Input** | `PLAYER`-origin intent commands (`set_move_intent`, `set_aim`, facing; `coalesce = LATEST_PER_ACTOR_PER_TICK`) and edge-triggered actions (`coalesce = NONE`); registers its own types | Result events; character-state read views; `tick_alpha` | Controller: types · Kernel: submit, ordering | **Open boundary**: who owns character position. Proposal — Ship Body owns ship-local kinematic position, Controller owns intent and facing; movement integrates on the physics side from the current intent so the sprite never waits a tick |
| 3 | **Save/Load & Versioned Format** | Control-plane calls `request_snapshot` / `request_restore`; owns file format, header, format version, migration orchestration, the MVP death stub (reload last save) | Snapshot `(T, per-block data + versions)`; dependency-ordered restore + `post_restore`; `Dropped(RELOAD)` for flushed commands | Kernel: registry contract and snapshot semantics · Save/Load: everything on disk | Domain systems depend on the contract, not on Save/Load. Save/Load decides policy for unknown blocks in a save and uses `default_state()` for missing ones. **The MVP death stub (reload last save) is a placeholder, not Pillar 5's failure contract.** Cross-run knowledge (decoded web rules, lore, scars) must survive `request_restore`: blocks declare `persist_scope` `WORLD` (in the world snapshot, rolled back on restore) or `PROFILE` (outside it, never rolled back by a world restore; Save/Load owns the profile store). Open Question 13 |
| 4 | **Suit & Equipment** | `PER_PLAYER` block keyed by PlayerId; equip/loadout/consume commands; vitals hook `after: graph.propagate`; pushes noise/power stimuli to Attention | Ordered Graph read; notifications for HUD | Suit: block, commands · Attention: stimulus intake API | Stimulus push uses CR-9 path (a) or (b); Attention's intake shape is undesigned. Deployables-as-emitters go through Emitter State's commands |
| 5 | **Modular Ship Assembly** | Weld, cut, install, uninstall as single commands whose `apply` calls published mutators on Graph, Layout and Body | Atomicity (G2, CR-9), one notification batch, validate-then-apply | Assembly: commands · Graph/Layout/Body: mutators | **The stress test for the kernel.** Join validation must be a pure query. Detach allocates a new body ID from the Layout block. Physics reprojection after a weld is a `PROJECTOR` |
| 6 | **Ship Body & Interior Physics** | Hook sampling physics results into its block each tick; `PROJECTOR` for layout → physics; ship-local frame | Tick alignment with the physics step; body create/destroy notifications | Ship Body | The determinism limit (G3) bites here; the tick-to-physics-step relationship and the `PROJECTOR` class need a TD ruling → ADR |
| 7 | **Emitter State & Light-Exposure Query** | Emitter block (`SIM`, `persist`) and its set/toggle/aim commands (Interaction, Suit and deployables submit *those*; nobody writes emitters directly); exposure query as a pure read | Read views for Lighting; time base | Emitter State | The query touches physics space (raycasts): callable from validate, P3 and presentation prompts, never from notification handlers that decide state. Per-tick query budget → Section D. Lighting never writes emitters |
| 8 | **Web Map & Transit** | World-model block (derelict list, seeds + deltas, loaded set) and the `pod_hop` command; Transit advances via a hook | Atomic apply of the hop's data change | Web Map & Transit | Content load for a target may exceed a tick: the hop applies *data* atomically and sets Transit to `ARRIVING`; a `PROJECTOR` builds scene content and reports ready; the Transit hook completes arrival. No multi-tick transaction in the kernel |
| 9 | **Horror Entities** | Entity block (`SIM`); hook `after` Attention and Emitter exposure; footprints via the Graph's mutator; light disturbance exposed as state Lighting reads | Tick base (tell windows in ticks); seeded RNG stream helper | Horror | `SYSTEM`-origin commands where a target may reject |
| 10 | **Sanity & Hallucination Rules** | Sanity level per player (`SIM`, `PER_PLAYER`, `persist`) and perception output (`PERCEPTION_OUT`, not persisted); drift hook at divisor 12; falsification checks arrive as commands from Interaction/Suit | Class-enforced isolation (CR-12); `OWNER_PRIVATE` replication hint | Sanity | Keyed by PlayerId, not peer ID. Friendly-harm gate: Interaction's `validate` calls a Sanity query on `SIM` data (CR-7). A falsification result any sim system depends on must be written to a `SIM` block. **Replication (VS-B)**: only `PERCEPTION_OUT` is `OWNER_PRIVATE`; the sanity-level block is at least `SHARED_SUMMARY` for its threshold flag, because P4(c) requires the *victim* to see the attacker's tell. Sanity also prices how a reason code reaches a hallucinating player (P4 cost rule) |
| 11 | **Perception Filter / Debug Overlay** | Subscriptions only: Filter as `PRESENTATION`, Overlay as `DIAGNOSTIC` | Read views (`SIM` + `PERCEPTION_OUT`); diagnostics: tick, kernel state, queue depth, last-N command log with results, hook order and timings, rejection counters, state hash | Kernel: views and diagnostics surface · Filter/Overlay: drawing | Neither gets submit or mutators. Overlay absent from release builds (never an accessibility feature). Filter writes only CanvasItem/Material/Light2D |
| 12 | **Interaction System** | Verb commands (`begin_cut`, `begin_weld`, `interact`, field checks, friendly-harm act); as `INPUT_SURFACE` for prompts | Result events with honest reasons (CR-5) | Interaction: command types and domain reason codes | **P4 cost constraint**: Interaction decides when and how a true reason reaches the player; an instant honest rejection is a free lie-detector and is forbidden. Open: what an `INPUT_SURFACE` submits when the player targets a phantom (P4(a)/(d)) that has no `SIM` ID, and what the result teaches — an instant `INVALID_TARGET` is the same free oracle (Open Question 14). The exact reason-code set is Interaction's GDD |
| 13 | **Attention System** | Attention block (`SIM`, `persist`), stimulus intake, breach state machine hook at divisor 6 | Ordered reads of Emitter exposure, Graph and Suit; tick base for the tell window | Attention | Missing from the index's kernel edges (index correction candidate) |
| 14 | **Co-op Session & Replication (VS-B)** | Ingress adapter (the only place a network sender ID is read; stamps `issuer_peer` and `origin = PLAYER`; never trusts payload); binding-table maintenance; result return to issuers; `REPLICATION` subscriber | Identical API in `MIRROR` mode: `submit` forwards to the host; notifications fire when replicated state is applied; `scope`/`repl_hint` metadata; full snapshot as the initial guest transfer | Co-op Session | Metadata and seams only in MVP. Replication of data blocks (not nodes) likely uses a thin proxy node or custom messaging rather than MultiplayerSynchronizer property sync → ADR. The listen-server host's local player must behave exactly like single-player |

**Engine-reference coverage (Godot 4.7.2, pinned).** `networking.md` covers the host-authoritative RPC pattern, MultiplayerSpawner/Synchronizer and `SceneMultiplayer.auth_callback`, and lists no 4.5–4.7 breaks; the reference is **silent** on `_process` vs `_physics_process` semantics, 2D physics interpolation, Resource/RefCounted serialization, autoload behaviour and signals (NOT SOURCEABLE — ADRs must verify). Sourced breaking changes that constrain the implementation: 4.7 typed-return overrides need an explicit `return` on every path (bites `StateBlock`/handler base classes); 4.5 `@abstract` is usable for those bases but untested on RefCounted in the reference; variadic lambdas (`...args`) can adapt a signal of any arity; lambdas capture locals by value; packed-array element writes bypass setters (CR-8); `duplicate()` is not deep — use `duplicate_deep()`; `FileAccess.store_*` return `bool` (Save/Load must check); gdUnit4 6.2.x runs on 4.7.2 and reports leaked orphan nodes as exit 101, which a Node-free kernel avoids by construction.

## Formulas

*Scope note.* This is an infrastructure kernel, so every formula below is timing, capacity, ordering or identity math; nothing here is a balance curve. Every numeric default is the proposal from Section C's numeric quantities and is **provisional until the kernel spike** unless marked otherwise. Source column vocabulary: `constant` = compiled into the kernel, changing it is a code change; `data file` = kernel config or registration data, read at boot; `calculated` = derived at runtime. All "ticks" are the integer tick counter `T` of CR-1; all "seconds" are simulated seconds unless prefixed *wall*.

**D.1 Tick duration and duration conversion**

The `tick_duration` formula is defined as:

`tick_duration = 1 / TICK_RATE`

**Variables:**
| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| TICK_RATE | int | 15–120 (default **60**) | constant (set once at project start, CR-1) | Authoritative ticks per simulated second |
| tick_duration | float | 0.00833–0.0667 s | calculated | Simulated seconds advanced by one tick |

**Output Range:** 16.667 ms at the default; 66.7 ms at the gameplay-programmer's hard floor of 15 Hz; 8.33 ms at 120 Hz. Never zero because `TICK_RATE ≥ 15` is a boot assertion. The kernel never multiplies by wall-clock `delta` (H9); `tick_duration` is the only place seconds and ticks meet.
**Example:** `TICK_RATE = 60` → `tick_duration = 1/60 = 0.016667 s`.

The `ticks_for_seconds` formula is defined as:

`ticks_for_seconds(s) = round(s × TICK_RATE)` *(adopted — decision log 2026-10-25; rationale: `round` half-up for durations; owners that need a guaranteed-minimum wait use `ceil` and say so)*

**Variables:**
| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| s | float | 0–unbounded | data file (the owning system's tuning) | A designer-facing duration in simulated seconds |
| TICK_RATE | int | 60 | constant | As above |
| ticks_for_seconds | int | 0–unbounded | calculated | The same duration expressed in ticks — the only unit the sim stores |

**Output Range:** 0 for any `s < 1/(2·TICK_RATE)` = 8.3 ms — a duration shorter than half a tick rounds to zero ticks and happens "immediately" on the next tick; owners must not rely on sub-tick durations. Unbounded above; a 64-bit int never overflows in practice (see D.14). Every tuned value stored in ticks is invalidated if `TICK_RATE` changes; the conversion is done once at data-load, never per tick.
**Example:** Attention tell window 10–30 s → `round(10 × 60) = 600` to `round(30 × 60) = 1800` ticks. A hook at divisor 6 sees that window as 100–300 of its own runs.

**D.2 Hook scheduling**

The `hook_runs_this_tick` formula is defined as:

`hook_runs_this_tick(h, T) = (T mod divisor_h) == offset_h`

**Variables:**
| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| T | int | 1–2^63−1 | calculated (kernel.clock) | Current tick number |
| divisor_h | int | 1–TICK_RATE (defaults: Graph **6**, Attention **6**, Suit vitals **6**, Sanity drift **12**, input/physics-adjacent **1**) | data file (hook registration, frozen at boot) | Run every `divisor_h` ticks |
| offset_h | int | 0–divisor_h−1 (default 0) | data file (hook registration) | Phase within the divisor cycle; staggers load |
| hook_runs_this_tick | bool | {false, true} | calculated | Whether hook `h` executes in P3 of tick `T` |

**Output Range:** Boolean. `divisor_h = 0` is a boot failure (division by zero; `BOOTING → FAULTED`), as is `offset_h ≥ divisor_h` or any negative value. `divisor_h = 1` is true every tick.
**Example:** Graph `(6, 0)` runs at T = 6, 12, 18 …; Sanity drift `(12, 0)` at T = 12, 24 …; at T = 12 both run and `after` ordering decides who goes first.

The `effective_hook_rate` formula is defined as:

`effective_hook_rate(h) = TICK_RATE / divisor_h`

**Variables:**
| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| TICK_RATE, divisor_h | int | as above | constant / data file | — |
| effective_hook_rate | float | 1–60 Hz | calculated | Runs per simulated second |

**Output Range:** 60 Hz at divisor 1 down to 1 Hz at divisor 60. Integer only when `divisor_h` divides `TICK_RATE`; 60 is divisible by 1, 2, 3, 4, 5, 6, 10, 12, 15, 20, 30, 60 — choosing a divisor outside that set is legal but produces a non-integer rate that reads badly in tuning tables.
**Example:** Graph `60/6 = 10 Hz`; Sanity drift `60/12 = 5 Hz` — exactly the CR-1 claims.

The `hooks_running_at` formula (per-tick load) is defined as:

`hooks_running_at(T) = |{ h : hook_runs_this_tick(h, T) }|` and the worst-case tick is `T* = lcm(all divisors) × k` where every zero-offset hook coincides.

**Variables:**
| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| hooks_running_at | int | (#divisor-1 hooks)–(#all hooks) | calculated | Hooks executing this tick |

**Output Range:** With all offsets 0 and divisors {1, 6, 6, 6, 12}, every 12th tick runs all five hooks and ticks 6 mod 12 run four; the P3 cost profile is spiky. Staggering offsets (Graph 0, Attention 2, Suit 4 — see the `after` caveat in Section E) flattens it to at most three per tick. This is what `STEP_BUDGET_MS` (D.6) is measured against.
**Example:** Defaults, offsets all 0: T = 12 → 5 hooks; T = 6 → 4; T = 7 → 1.

**D.3 Render interpolation**

The `tick_alpha` formula is defined as:

`tick_alpha = clamp(t_since_last_commit / tick_duration, 0, 1)`

**Variables:**
| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| t_since_last_commit | float | 0–unbounded wall-seconds | calculated (by the tick *driver*, not the sim) | Wall time since the last P4 commit |
| tick_duration | float | 0.0167 s | calculated | D.1 |
| tick_alpha | float | 0.0–1.0 | calculated | Fraction of the way from the previous committed state to the current one |

**Output Range:** 0 at the instant of commit, 1 when a full tick has elapsed; clamped at 1 so that during a hitch or catch-up lag presentation holds the latest committed pose rather than extrapolating past it (extrapolation would show the ship doing something the sim has not yet done — a Pillar 1 hazard). The sim never reads this value (H9); only the driver computes it and only `PRESENTATION`/`PROJECTOR` consume it.
**Example:** 7 ms after commit at 60 Hz → `7/16.67 = 0.42`; a position read view exposes `prev + (curr − prev) × 0.42`.

**D.4 Input-to-effect latency**

The `apply_tick` formula is defined as:

`apply_tick(c) = queued_tick_c + 1 + floor(q_pos_c / MAX_CMDS_PER_TICK)`

**Variables:**
| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| queued_tick_c | int | ≤ T | calculated (stamped at enqueue) | Tick during (or after) which the command was admitted |
| q_pos_c | int | 0–QUEUE_CAPACITY−1 | calculated | Position of `c` among pending commands in ordering-key order at the moment of the next P0 freeze |
| MAX_CMDS_PER_TICK | int | 128 | data file | CR-17 per-tick processing cap |
| apply_tick | int | queued_tick+1 – queued_tick+8 | calculated | The one tick at which `c` is dispatched (G2) |

**Output Range:** Minimum `queued_tick + 1` (an empty or lightly loaded queue — the normal case, matching CR-1's "≤ 17 ms" which is submit → applied in *simulated* time); maximum `queued_tick + ceil(QUEUE_CAPACITY / MAX_CMDS_PER_TICK) = queued_tick + 8` when the queue is full. Commands never apply at `queued_tick` itself because P0 freezes the queue before P1.
**Example:** Command admitted while T = 1000 with 300 commands ahead of it → `1000 + 1 + floor(300/128) = 1003`.

The `input_to_effect_latency` formula (wall-clock, to the pixel) is defined as:

`L_pixel = L_wait + (apply_tick − queued_tick) × tick_duration_wall + L_render`, where `L_wait ∈ [0, tick_interval_wall)` and `L_render ∈ [0, frame_duration)`

**Variables:**
| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| L_wait | float | 0–16.7 ms | calculated | Wall time from submit to the next P0 (depends where in the frame the submit happened) |
| tick_duration_wall | float | 16.7 ms at steady state; 0 during catch-up | calculated | Wall time between successive ticks |
| frame_duration | float | 16.7 ms at 60 fps; 33.3 at 30 fps | calculated (engine) | Render frame period |
| L_render | float | 0–frame_duration | calculated | Wait from P5 notify to the frame that draws it |
| L_pixel | float | see below | calculated | Submit → drawn (excludes display scan-out) |

**Output Range (60 Hz tick, 60 fps render, tick driven by the physics callback before `_process` — conditional on the ADR):** **Best case ≈ a few ms** — input polled at frame start, tick runs in the same frame, P5 notifies, `_process` reads the committed state and the same frame draws it (`L_wait ≈ 0`, one tick of sim time, `L_render ≈ 0`). **Worst case ≈ 33.3 ms** — submitted just after this frame's tick (e.g. from an `INPUT_SURFACE` notification handler in P5), waits for next frame's tick (16.7), applied, drawn that frame (+16.7). With a backlogged queue add `floor(q_pos/128) × 16.7 ms`, up to +116.7 ms at a full queue. At 30 fps render the worst case is 50 ms. Immediate *acknowledgement* (CR-4) is presentation's and is 0 ticks by construction. The engine-reference is silent on `_physics_process`/`_process` ordering and on 2D interpolation, so the "same frame" best case must be verified by the ADR spike.
**Example:** Press at 2 ms into frame k; tick k runs at 4 ms (applied), P5 at 6 ms, `_process` reads at 7 ms, frame presented at 16.7 ms → `L_pixel ≈ 15 ms`. Press from a P5 handler at 6 ms of frame k → next tick at frame k+1 (≈ 20.7 ms), drawn at 33.3 ms → `L_pixel ≈ 27 ms`.

**D.5 Catch-up and simulated-time lag**

The `ticks_run_this_frame` formula is defined as:

`ticks_due = floor(accumulator / tick_duration)`; `ticks_run = min(ticks_due, MAX_TICKS_PER_FRAME)`; `accumulator ← accumulator − ticks_run × tick_duration`

**Variables:**
| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| accumulator | float | 0–unbounded wall-seconds | calculated (tick driver) | Un-simulated wall time |
| ticks_due | int | 0–unbounded | calculated | Ticks that would keep sim time equal to wall time |
| MAX_TICKS_PER_FRAME | int | 1–8 (default **4**) | data file | CR-17 catch-up cap |
| ticks_run | int | 0–4 | calculated | Ticks actually stepped this frame |

**Output Range:** 0 (a frame shorter than a tick) to `MAX_TICKS_PER_FRAME`. **If the tick source is Godot's physics callback (CR-1's recommendation) the engine's `physics/common/max_physics_steps_per_frame` (engine default 8) is the effective cap and must be set to the same value; whether the engine then discards or accumulates the excess is not sourced in the engine reference and the ADR must verify it.** The kernel's own `step()` has no accumulator; this formula lives in the driver.
**Example:** A 100 ms frame at 60 Hz: `ticks_due = 6`, `ticks_run = 4`, 2 ticks (33 ms) of lag remain.

The `lag_ticks` formula is defined as:

`lag_ticks = floor(accumulator / tick_duration)` after the frame's ticks have run; `simulated_time_lag_s = lag_ticks × tick_duration`; per-frame growth `Δlag = max(0, ticks_due − MAX_TICKS_PER_FRAME)`; repayment per frame `= MAX_TICKS_PER_FRAME − ticks_due_at_steady_state` (3 per frame at 60 fps)

**Variables:**
| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| lag_ticks | int | 0–unbounded | calculated | Exposed to diagnostics (CR-17) |
| simulated_time_lag_s | float | 0–unbounded | calculated | How far the ship's clock is behind the player's |

**Output Range:** Converges to 0 only while `frame_time < MAX_TICKS_PER_FRAME × tick_duration` = 66.7 ms (15 fps) at defaults; below 15 fps lag grows without bound and the game runs in slow motion rather than spiralling. During repayment the sim runs at `MAX_TICKS_PER_FRAME×` real time (4×) — a feel hazard for a horror game (see Section E and the `LAG_FORGIVE_TICKS` knob in Section G). Because the sim never reads wall time, lag is invisible to every domain system; its only effects are slow-motion and the diagnostics counter.
**Example:** A 1 s hitch → `lag_ticks = 60 − 4 = 56` after the first frame; repaid at 3/frame → 19 frames (≈ 0.3 s) of 4× fast-forward.

**D.6 Step budget**

The `step_over_budget` formula is defined as:

`step_over_budget(T) = (Σ_h∈running t_hook(h, T)) > STEP_BUDGET_MS`

**Variables:**
| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| t_hook | float | 0–unbounded ms | calculated (diagnostics timer, not visible to the sim) | Wall time one hook took |
| STEP_BUDGET_MS | float | 1–8 (default **4**) | data file | CR-17 logging threshold for P3 — **a threshold, not a cap**; the tick always completes |
| step_over_budget | bool | — | calculated | Logs the slowest hooks when true |

**Output Range:** Boolean. The budget is sized so that a full catch-up frame stays inside a frame: `MAX_TICKS_PER_FRAME × (STEP_BUDGET_MS + t_dispatch + t_notify) ≤ frame_duration` → at defaults `4 × 4 = 16 ms` for P3 alone already consumes a 60 fps frame, so the real target is a typical tick well under 2 ms and the 4 ms figure is the alarm.
**Example:** T = 12 runs five hooks at 1.2 + 0.9 + 0.4 + 0.3 + 0.1 = 2.9 ms → under budget; the Graph propagating a 2,000-node derelict at 4.5 ms alone → over budget, logged as slowest.

**D.7 Ordering, staleness and the STALE finding**

The `ordering_key` formula is defined as:

`ordering_key(c) = (queued_tick_c, host_order_c)`, compared lexicographically; `host_order` is a kernel-stamped counter incremented on every *admission*, never reused *(adopted — decision log 2026-10-25; rationale: Section C names `host_order` but does not say who stamps it)*

**Variables:**
| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| queued_tick_c | int | ≤ T | calculated | Tick stamped at enqueue |
| host_order_c | int | 1–2^63−1 | calculated | Admission sequence number, unique per kernel instance per session |
| ordering_key | tuple(int, int) | — | calculated | G1 total order |

**Output Range:** A strict total order: `host_order` is unique, so no two commands compare equal and no tie-break is ever needed. In single-player it reduces to submission order (G1). The pair rather than `host_order` alone is kept so a VS-B ingress can legitimately stamp an earlier `queued_tick` and still sort deterministically. P1 takes the first `MAX_CMDS_PER_TICK` by this key; overflow keeps its key and position.
**Example:** `(1000, 5817) < (1000, 5818) < (1001, 5816)` — the third was admitted earlier but carries a later `queued_tick` (only possible via a VS-B ingress; impossible locally).

The `is_stale` formula is defined as:

`is_stale(c, T) = (T − queued_tick_c) > max_age_ticks(type_c)`, evaluated at P2 authority check

**Variables:**
| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| T | int | — | calculated | Dispatch tick |
| queued_tick_c | int | ≤ T | calculated | Enqueue tick |
| max_age_ticks | int | 1–600 (defaults: intents **30** = 0.5 s, actions **120** = 2 s) | data file (per command type, owner-declared) | Oldest acceptable age |
| is_stale | bool | — | calculated | Rejects `STALE` when true |

**Output Range:** Age at dispatch is `apply_tick − queued_tick ∈ [1, 8]` locally (D.4), because `T` does not advance in `PAUSED` or `SAVING` and a full queue drains in 8 ticks. **Finding: with the proposed defaults `is_stale` is unreachable in HOST mode with local ingress — 8 < 30 < 120.** This is not a bug if the intent is "local commands never go stale; STALE guards a remote ingress", but Section C implies it also guards hitches, which it cannot (hitches stop frames and ticks together; wall time is invisible). *(Proposed — decided (decision log 2026-10-25).)* Options: **(a, recommended)** keep the predicate; declare the invariant `max_age_ticks(any type) ≥ ceil(QUEUE_CAPACITY / MAX_CMDS_PER_TICK)` as a boot assertion and acceptance test ("a local command is never STALE"); have the VS-B ingress adapter stamp `queued_tick = clamp(issuer_observed_tick, T − max_age − 1, T)` so remote lateness is measurable — which is what the 30/120 values are really for. **(b)** lower defaults so STALE is reachable locally — rejected by me because a local command that waited 9 ticks in a full queue is still the player's genuine latest input. **(c)** delete STALE from MVP's reason set and reintroduce in VS-B — honest but loses the seam.
**Example:** Intent queued at T = 1000, queue full → dispatched T = 1008, age 8 ≤ 30 → not stale. VS-B guest intent carrying observed tick 1000, arriving at T = 1040 → age 40 > 30 → `STALE`.

**D.8 Queue admission and rate limit**

The `admitted` formula is defined as:

`admitted(c) = accepts(kernel_state, type_c) ∧ registered(type_c) ∧ |queue| < QUEUE_CAPACITY ∧ |queue_issuer(c)| < PER_ISSUER_CAPACITY ∧ seq_c > last_seq(issuer_c, origin_c) ∧ ¬rate_limited(issuer_c) ∧ size(payload_c) ≤ MAX_PAYLOAD ∧ plain(payload_c)`; the first failing term in that order is the reject reason (`NOT_ACCEPTING`/`PAUSED`, `UNKNOWN_TYPE`, `QUEUE_FULL`, `QUEUE_FULL`, `DUPLICATE_SEQ`, `RATE_LIMITED`, `MALFORMED`, `MALFORMED`)

**Variables:**
| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| QUEUE_CAPACITY | int | 256–4096 (default **1024**) | data file | Total pending commands, all issuers |
| PER_ISSUER_CAPACITY | int | 64–QUEUE_CAPACITY (default **256**) | data file | Pending commands per `(issuer_peer, origin)` |
| last_seq | int | 0–2^63−1 | calculated (per issuer, reset when the issuer's binding is created) | Highest admitted `seq` |
| MAX_PAYLOAD | int | 1–64 KiB (default **4 KiB**) | data file | Serialized payload bound |
| admitted | bool | — | calculated | Enters `QUEUED` when true |

**Output Range:** Boolean; the order of terms is normative so the same bad command always gets the same reason (G5 determinism of results). Coalescing (Section E) runs *before* the capacity terms, so a superseding intent never fails `QUEUE_FULL` because of the intent it replaces. Queue memory bound: `QUEUE_CAPACITY × (MAX_PAYLOAD + ~200 B overhead) ≈ 4.2 MiB` worst case.
**Example:** Local player (issuer 1, 3 pending), queue at 1023, SYSTEM issuer at 255 pending: player's next command → admitted (1024th slot); the Horror hook's next `SYSTEM` command → `QUEUE_FULL` (global).

The `rate_limited` formula is defined as:

`rate_limited(i) = remote(i) ∧ (submits_in_window(i) ≥ REMOTE_RATE_LIMIT × RATE_WINDOW_TICKS / TICK_RATE)`, window measured in ticks `[T − RATE_WINDOW_TICKS, T]`

**Variables:**
| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| remote(i) | bool | always false in MVP (H8) | calculated | Issuer arrived via a network ingress |
| REMOTE_RATE_LIMIT | int | 30–600 per s (default **120/s**) | data file | Admissions per second per remote issuer |
| RATE_WINDOW_TICKS | int | 6–120 (default **60**) | data file | Sliding window length; tick-based because the kernel has no wall clock (H9) |
| rate_limited | bool | — | calculated | Rejects `RATE_LIMITED` when true |

**Output Range:** Boolean; in MVP the predicate is a seam that is always false. At defaults the budget is 120 admissions per 60-tick window — bursts up to 120 pass, sustained input above 2/tick is cut. Because the window is tick-based, during `PAUSED`/`SAVING` the window does not slide and a remote cannot be rate-limited more harshly than 120 per paused period — acceptable since VS-B has no pause.
**Example:** A guest sending 2 intents + 1 tap per frame at 60 fps = 180/s → the 121st in any 1 s window is `RATE_LIMITED`.

The `queue_drain_ticks` formula is defined as:

`queue_drain_ticks = ceil(QUEUE_CAPACITY / MAX_CMDS_PER_TICK)`

**Variables:** as above. `queue_drain_ticks` int, 1–unbounded.
**Output Range:** 8 ticks (133 ms) at defaults — the worst wait for the last-admitted command, and the bound D.7's invariant compares against.
**Example:** `ceil(1024/128) = 8`.

**D.9 Dirty-key collapse**

The `change_keys` formula is defined as:

`change_keys(b, T) = dirty(b, T) if |dirty(b, T)| ≤ MAX_DIRTY_KEYS else ALL`

**Variables:**
| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| dirty(b, T) | set of key | 0–|keys(b)| | calculated (explicit marks in mutators — CR-8's packed-array gotcha) | Keys of block `b` written during P2/P3 of tick `T` |
| MAX_DIRTY_KEYS | int | 32–4096 (default **256**) | data file | Collapse threshold |
| change_keys | set of key ∪ {ALL} | — | calculated | Delivered in P5 as `(block_id, T, change_keys)` |

**Output Range:** A set of at most 256 keys, or the sentinel `ALL`. Collapse is worth it when `|dirty| × c_key > c_full` — the per-key subscriber refresh cost times count exceeds a full-block refresh — so the threshold should sit near `c_full / c_key`; the Graph (largest key volume) sets it, per Interactions row 1. A block with no dirty keys emits nothing. Notification payload is therefore bounded at `MAX_DIRTY_KEYS × key_size` per block per tick.
**Example:** A weld dirties 14 Graph keys → delivered as 14 keys; a hull breach cascading through 300 nodes → `ALL`; Failure & Trail does one full re-read.

**D.10 RNG stream derivation**

The `rng_derive` formula is defined as:

`rng_derive(world_seed, stream_name, counter) = mix64( world_seed ⊕ mix64(fnv64(stream_name)) ⊕ (counter × 0x9E3779B97F4A7C15) )`; `unit = (rng_derive >> 11) / 2^53`; `rng_int(lo, hi) = lo + floor(unit × (hi − lo + 1))`

**Variables:**
| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| world_seed | int (u64) | 0–2^64−1 | data file (kernel.clock block; set at new game, persisted) | Per-world seed |
| stream_name | string | registered, e.g. `horror.tell` | data file (owner declares) | Names an independent stream |
| counter | int | 0–2^63−1 | calculated (lives in the owner's block, incremented per draw, persisted with it) | Draw index |
| mix64 | fn | bijective u64→u64 with full avalanche (SplitMix64 finaliser or xxHash64 — → ADR) | constant | Mixing function |
| rng_derive | int (u64) | 0–2^64−1 | calculated | Deterministic raw value |
| unit | float | [0, 1) | calculated | 53-bit uniform |
| rng_int | int | [lo, hi] | calculated | Uniform integer in an inclusive range |

**Output Range:** `unit` is in `[0, 1)` exactly (the top 53 bits of a u64 divided by 2^53 can never reach 1). `rng_int` is inclusive both ends and free of modulo bias. Pure: the same `(seed, name, counter)` always gives the same value regardless of call order or which other streams drew — which is what lets G3 replay and G4 save-equivalence hold with no RNG state outside blocks. A stream's counter restores with its block, so a save mid-tell reproduces the same tell.
**Example (hash output illustrative until the ADR picks `mix64`):** `world_seed = 0x5EED_0000_0000_2026`, `stream_name = "horror.tell"`, `counter = 17` → say `rng_derive = 0x3C6E_F372_FE94_F82B` → `unit ≈ 0.2361` → tell window `rng_int(600, 1800) = 600 + floor(0.2361 × 1201) = 883` ticks ≈ 14.7 s.

**D.11 Snapshot size and capture budget**

The `snapshot_bytes` formula is defined as:

`snapshot_bytes = header_bytes + Σ_{b ∈ persist} (block_header_bytes + canonical_bytes(b))`

**Variables:**
| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| header_bytes | int | ~64 | calculated | `T`, `world_seed`, `TICK_RATE` (proposed), block count |
| block_header_bytes | int | ~48 | calculated | `block_id`, `schema_version`, byte length |
| canonical_bytes(b) | int | 0–unbounded | calculated (owner's `snapshot()`) | Size of the block's canonical form |
| persist | set | ≤ MAX_BLOCKS (**128** registry cap; **32** expected) | data file | Blocks with `persist = true` |
| snapshot_bytes | int | — | calculated | In-memory snapshot size handed to Save/Load |

**Output Range:** Unbounded in principle; Save/Load owns the on-disk form. Expectation for MVP: ≈ 100–300 KiB, dominated by the Graph. *(Provisional until spike.)*
**Example:** Graph 2,000 nodes × 6 fields × 8 B ≈ 96 KiB; 31 other blocks at ~2 KiB ≈ 62 KiB; headers ≈ 1.6 KiB → ≈ 160 KiB.

The `capture_time_ms` formula is defined as:

`capture_time_ms = Σ_{b ∈ persist} t_snapshot(b) + t_hash(snapshot_bytes)`; budget `capture_time_ms ≤ SNAPSHOT_BUDGET_MS`; per-block guideline `t_snapshot(b) ≤ SNAPSHOT_BUDGET_MS / |persist|`

**Variables:**
| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| t_snapshot(b) | float | ms | calculated (diagnostics timer) | Owner's `snapshot()` wall time |
| t_hash | float | ms | calculated | Hashing the canonical bytes (debug/test builds only unless Save/Load wants integrity) |
| SNAPSHOT_BUDGET_MS | float | 8–33 (default **16.7** = one 60 fps frame) | data file | CR-18 "cheap" target; the visible hitch at autosave |
| capture_time_ms | float | — | calculated | Length of the `SAVING` state in wall time |

**Output Range:** The budget is a diagnostics threshold like `STEP_BUDGET_MS`, not a cap — a snapshot is never truncated. Per-block guideline ≈ `16.7 / 32 ≈ 0.5 ms`. GDScript builds Dictionaries at roughly 0.5–1 M entries/s (unverified, spike), so a 0.5 ms block has a budget of ~250–500 scalar fields; any block larger than that must snapshot as `Packed*Array` copies (memcpy-speed), which is the Graph's design constraint. VS-B reuses this as the guest's initial transfer, so the same budget applies there. *(All numbers provisional until spike.)*
**Example:** 31 small blocks at 0.1 ms + Graph at 3 ms (packed arrays) + hash 1.5 ms ≈ 7.6 ms → under budget; the same Graph as nested Dictionaries at ~25 ms → over, logged.

**D.12 State hash (G3/G4)**

The `state_hash` formula is defined as:

`state_hash(scope, T) = H64( T ‖ world_seed ‖ ⨁_{b ∈ scope, sorted by block_id} ( block_id ‖ schema_version ‖ canonical_bytes(b) ) )`, where `canonical_bytes` serialises dictionaries with keys sorted bytewise, arrays in order, ints as 64-bit little-endian, floats as their IEEE-754 bit pattern with `−0.0 → +0.0` and all NaNs → one canonical NaN, strings as UTF-8 length-prefixed, booleans as one byte

**Variables:**
| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| scope | enum | {`PERSIST` (G4), `ALL_SIM` (G3)} | calculated (test chooses) | Which blocks are hashed; `PERCEPTION_OUT` is never in either |
| H64 | fn | xxHash64 / FNV-1a 64 (→ ADR; SHA-256 if Save/Load wants integrity) | constant | Hash function |
| canonical_bytes | bytes | — | calculated | Byte form that is identical for equal state regardless of insertion order (Godot `Dictionary` is insertion-ordered, so sorting is mandatory) |
| state_hash | int (u64) | 0–2^64−1 | calculated | Shown by the Debug Overlay; compared by G3/G4 tests |

**Output Range:** Uniform over u64; collision probability 2^−64 per comparison, negligible for tests. Equal hashes with unequal state would be a false pass, so G4's test also diffs on mismatch rather than only comparing hashes. Physics-coupled blocks are excluded from `ALL_SIM` in G3 unless physics observations were recorded (CR-13's stated limit).
**Example:** Snapshot at T = 5000, restore, run to 5600 → `state_hash(PERSIST, 5600)`; uninterrupted run to 5600 → same value, e.g. `0x9F1C_2B77_0A54_E3D1` on both sides; any difference names the first differing `block_id` and key.

**D.13 Command log coverage**

The `log_coverage_ticks` formula is defined as:

`log_coverage_ticks = COMMAND_LOG_CAPACITY / cmds_per_tick_actual`; worst case `COMMAND_LOG_CAPACITY / MAX_CMDS_PER_TICK`

**Variables:**
| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| COMMAND_LOG_CAPACITY | int | 256–8192 (default **1024**) | data file | Ring buffer of commands with results (CR-9, diagnostics and replay tests only) |
| cmds_per_tick_actual | float | 0–128 | calculated | Observed admissions per tick |
| log_coverage_ticks | float | ≥ 8 | calculated | How far back the Debug Overlay's "last-N commands" reaches |

**Output Range:** At least 8 ticks (133 ms) at saturation; at a realistic 0.1 commands/tick (~6/s: a few intents and a tap) ≈ 10,240 ticks ≈ 2.8 minutes. Log entries store the command plus its result and `tick`, ≤ `MAX_PAYLOAD + ~100 B` each → ≤ ~4.3 MiB.
**Example:** `1024 / 0.1 = 10,240` ticks ≈ 171 s.

**D.14 Tick counter bound**

The `ticks_to_overflow` formula is defined as:

`ticks_to_overflow = (2^(bits−1) − 1) / TICK_RATE` seconds

**Variables:**
| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| bits | int | 64 (Godot `int`) | constant | Width of `T`, `seq`, `host_order`, counters |
| ticks_to_overflow | float | — | calculated | Simulated seconds before wrap |

**Output Range:** 64-bit → 1.5 × 10^17 s ≈ 4.9 billion years: overflow is impossible and the kernel does no wrap handling; a boot assertion that `T`, `seq` and `host_order` are 64-bit is the whole mitigation. (For contrast, 32-bit signed would wrap after 414 days of continuous play — the reason the width is asserted rather than assumed.)
**Example:** `(2^63 − 1) / 60 ≈ 1.54 × 10^17 s`.

## Edge Cases

Every outcome below is the exact kernel behaviour; items marked *(adopted — see decision log 2026-10-25)* fill a gap in Section C and require the user's call before being treated as spec. "Result event" always means the single CR-5 result keyed by `(issuer_peer, seq)`.

**Submission timing and kernel state**

- **If `submit` is called during P2 or P5 of tick T (or any phase P0–P6)**: the intake queue has been frozen since P0, so the command is admitted into the *unfrozen* pending set with `queued_tick = T` and is eligible no earlier than tick T+1 (`apply_tick ≥ T+1`, D.4). No result event yet; it is `QUEUED`. From a P5 `INPUT_SURFACE` handler this is the normal and intended path (a menu confirming a selection). From P2/P3 only `SYSTEM`-origin submits exist (CR-3/CR-9); see "a hook submits a PLAYER command" below.
- **If `submit` is called while `SAVING`**: admitted and queued normally with `queued_tick = T` (the tick just captured). The pending queue is kernel data, not a block, so the snapshot never contains it; the command is dispatched at T+1 once the kernel returns to `RUNNING`. If the save is later restored, that command's effects exist in the live world but not in the save — correct, because it applied *after* T.
- **If `submit` is called while `LOADING`, `BOOTING`, `FAULTED` or `SHUTTING_DOWN`**: rejected `NOT_ACCEPTING` at admission; the result event is emitted during the `submit` call through the normal result channel (fire-and-forget still holds — no return value — but admission rejections never wait for a tick). Nothing is queued.
- **If `submit` is called while `PAUSED` with a type not flagged `allowed_when_paused`**: rejected `PAUSED` immediately, never queued (CR-16), so resume produces no burst.
- **If `submit` is called while `PAUSED` with an `allowed_when_paused` type**: *(adopted — decision log 2026-10-25; rationale: Section C admits these commands but no tick runs in `PAUSED`, so CR-16 has no dispatch path)*. Proposal: while `PAUSED` the driver runs a **paused service cycle** once per render frame consisting of P1, P2, P4, P5, P6 — **no P0 (T does not increment) and no P3 (no hooks)**. Commands dispatch at the current `T` with their ordinary `(queued_tick, host_order)` key; several commits can therefore occur at the same `T`, each logged with its `host_order` so G3 replay reproduces them in order. Alternative: make every paused action a control-plane call or pure presentation and drop `allowed_when_paused` from CR-16 — simpler, but then (reason withdrawn: CR-16 now forbids any `SIM` gameplay write while paused).
- **If `request_resume` arrives after a long pause**: the driver's accumulator is cleared on the `PAUSED → RUNNING` transition, so `ticks_due = 0` on the first running frame; a pause never produces catch-up ticks.
- **If a control-plane request (`request_pause`, `request_snapshot`, binding change) is made mid-tick (from any phase)**: it is recorded and honoured at this tick's P6, never mid-phase (CR-15). Two conflicting requests in one tick (pause and restore) are honoured in arrival order at P6; the second may become an invalid `(state, trigger)` pair and is then ignored and reported to diagnostics per the transitions table.

**Ordering, coalescing and sequence numbers**

- **If two commands target the same actor in one tick**: both dispatch in `ordering_key` order (D.7); the second sees the first's applied effects in its `validate` and `apply` (CR-2). No kernel coalescing unless the type declares it.
- **If a second command of a `coalesce = LATEST_PER_ACTOR_PER_TICK` type arrives for the same `(type, actor, issuer_peer)` while the first is still in the *unfrozen* pending set**: *(adopted — decision log 2026-10-25; rationale: Section C does not say where coalescing happens)* the earlier one is removed, becomes `SUPERSEDED` and emits its result immediately; the new one is admitted with its own fresh `host_order` and `queued_tick`. Coalescing runs before the capacity terms of `admitted` (D.8), so it never fails `QUEUE_FULL` because of the command it replaces. The key includes `issuer_peer` because admission precedes the authority check — without it a remote issuer could supersede another seat's intent. A P1 batch therefore never contains two commands of the same coalesce key.
- **If the earlier intent has already been frozen into the current tick's P1 batch when the newer one arrives**: the frozen one is *not* superseded — it applies at T, the newer applies at T+1. Supersession only ever reaches into the unfrozen pending set; the batch is immutable after P1.
- **If a `coalesce = NONE` tap (e.g. `begin_weld`) is queued between two intents of the same key** (`[intent A, tap, intent B]`): A is superseded; the queue becomes `[tap, intent B]` in `host_order`. B keeps its own (newer) `host_order` and never inherits A's position — the kernel never reorders. The tap therefore validates against the movement state *before* B, which is the issuer's actual chronology.
- **If a `NONE` type is submitted twice in one inter-tick window**: both are queued and both apply; losing one would lose a tap (CR-4). Idempotence of a double tap is the owner's `validate` (`BUSY`).
- **If `seq` is less than or equal to the issuer's `last_seq`**: rejected `DUPLICATE_SEQ`; nothing else changes (replays and reorders are both caught by this one rule).
- **If `seq` skips values (gap)**: *(adopted — decision log 2026-10-25; rationale: CR-5 has `DUPLICATE_SEQ` but no gap reason)* admitted; `last_seq` jumps forward; the gap size is added to a per-issuer diagnostics counter. Gaps are expected under VS-B loss and the kernel never retries, so rejecting them would strand every later command.
- **If an issuer's binding is (re)created (new session, VS-B reconnect)**: that issuer's `last_seq` resets to 0 and its rate window clears; `host_order` never resets within a session.
- **If a `SYSTEM`-origin command is submitted by a hook or handler**: `issuer_peer = 1` (host), `origin = SYSTEM`, and `seq` is drawn from a kernel-held `SYSTEM` sequence (owners never manage it); its result event is delivered to the submitting owner by `(1, seq)` at the next P5 *(proposed)*. Seq spaces are per `(issuer_peer, origin)` so a SYSTEM command can never collide with the host player's own `seq`.

**Dispatch and apply**

- **If a command's `actor` was destroyed earlier in the same tick (by an earlier command or, for a hook-submitted command, by last tick's P3)**: the kernel's authority check consults only the binding table, which is identity data and is not touched by domain destruction, so authority *passes* if the issuer is still bound to that entity ID; the owner's `validate` then fails on a missing entity and the result is `VALIDATION_FAILED(domain_code)` (Interaction's `INVALID_TARGET` or equivalent). The kernel never checks entity existence (H1). If the session owner has also unbound the actor (honoured at P6, so earliest next tick), the result is `ACTOR_NOT_OWNED` instead.
- **If a type declares `actor_binding = OWNED_BY_ISSUER` and the command's `actor` is none**: `ACTOR_NOT_OWNED`.
- **If a type declares `allowed_origins = {SYSTEM}` and a `PLAYER` command arrives for it**: `ORIGIN_NOT_ALLOWED` at P2 (not admission — the type is registered, so admission passes).
- **If `validate` passes but a published mutator called inside `apply` would violate another block's invariant** (e.g. a weld whose join query said OK but the Graph's mutator asserts): the mutator's assertion is an *apply failure* — a defect (CR-9). The kernel: aborts the current `apply` at the assertion; does not reach P4 (nothing is committed, notified, logged as applied or saveable); writes already made by this `apply` remain in the live blocks (no rollback) but are unreachable to presentation because P5 never runs; transitions `RUNNING → FAULTED`; emits `Dropped(FAULTED)` *(proposed addition to the CR-5 closed set — decided (decision log 2026-10-25); G5 requires some terminal and the set has none)* for the failing command, every remaining command in the batch, and every pending command; records `(T, type, issuer, host_order, block_id, assertion)` in diagnostics. Dev build halts with the diagnostic; release returns to the menu; no autosave is written (FAULTED rule).
- **If `validate` itself throws or has a side effect**: a thrown error in `validate` is treated identically to an apply failure (defect → `FAULTED`); side effects in `validate` are not detectable by the kernel at runtime and are a lint/test obligation on the owner.
- **If a handler calls a mutator outside a `WriteScope`** (from a notification handler, from presentation, from `validate`): the mutator asserts in debug; in release the write is refused, no dirty key is marked, and a `WRITE_OUTSIDE_SCOPE` diagnostics counter increments. The kernel state does not change — this is not an apply failure because no `apply` is running.

**Hooks**

- **If hook `after` lists contain a cycle or reference an unknown `hook_id`**: `BOOTING → FAULTED` with the cycle path (or the unknown ID and its referrer) in diagnostics, in every build (CR-6). No partial order is ever used.
- **If two hooks have no `after` relation**: ordered by registration index, then `hook_id` (CR-3); the resolved order is printed by the Debug Overlay and is identical on every boot with the same composition root.
- **If hook A declares `after: B` but their `(divisor, offset)` mean they never run in the same tick** (Suit `(6, 4)` after Graph `(6, 0)`): the order is still well-defined on the ticks where both run (never), and A reads B's state as committed by B's most recent run. *(Proposed — decided (decision log 2026-10-25))*: boot emits a **warning** (not a failure) listing the pair, because staggering is legitimate when a reader tolerates up-to-`divisor`-tick-old data; if the owner needs same-tick data, offsets must match and the divisors must be equal or one divide the other.
- **If a hook submits a `PLAYER`-origin command**: there is no API path — hooks receive only `submit_system()`, which stamps `SYSTEM` (H6). If a hook nevertheless reaches the local input ingress while the kernel's phase is P2 or P3, the ingress asserts in debug; in release the command is rejected `ORIGIN_NOT_ALLOWED` at admission and a diagnostics counter increments. It is never silently re-stamped `SYSTEM`.
- **If two or more hooks share a `(divisor, offset)`**: this is normal (Graph, Attention and Suit all at `(6, 0)` by default); they run in the same tick in resolved order. The only cost is a load spike every `divisor` ticks (D.2 `hooks_running_at`).
- **If a hook registers `divisor = 0`, `offset < 0`, or `offset ≥ divisor`**: `BOOTING → FAULTED` naming the hook.
- **If P3 exceeds `STEP_BUDGET_MS`**: the tick completes in full; the per-hook timings for that tick are logged with the slowest first (CR-17). Nothing is skipped or deferred.
- **If a hook throws or trips a step invariant**: `RUNNING → FAULTED` exactly as an apply failure; the pending queue gets `Dropped(FAULTED)`.

**Time and overload**

- **If the catch-up cap `MAX_TICKS_PER_FRAME` is reached on one frame**: `ticks_run = 4`, `lag_ticks` grows by `ticks_due − 4`, and the counter is exposed to diagnostics. The sim is correct and merely behind.
- **If the cap is reached on consecutive frames** (sustained frame time > 66.7 ms at defaults): lag grows every frame without bound; the game runs in slow motion; no spiral because work per frame is capped. When frames recover, lag is repaid at 3 ticks/frame — the sim runs at **4× real time** until caught up. *(Proposed — decided (decision log 2026-10-25) by gameplay-programmer / creative-director, not this agent)*: add `LAG_FORGIVE_TICKS` (suggest 30 = 0.5 s) — if `lag_ticks` exceeds it after a frame, the driver discards the deficit instead of fast-forwarding, because a horror game should never run 4× while the player reacts at 1×, and since no domain system reads wall time, discarded lag is unobservable except in diagnostics.
- **If the tick is driven by Godot's physics callback**: `MAX_TICKS_PER_FRAME` must equal `physics/common/max_physics_steps_per_frame`; the ADR must verify whether the engine discards or accumulates the excess (the engine reference is silent). A mismatch is a boot assertion.
- **If the queue holds more than `MAX_CMDS_PER_TICK` commands at P0**: P1 takes the first 128 by `ordering_key`; the rest remain queued with their keys and are first in line at T+1 (CR-17). The 129th command's `apply_tick` is `queued_tick + 2`.
- **If a command reaches P2 with age `T − queued_tick > max_age_ticks`**: rejected `STALE`. At the proposed defaults this cannot happen for a local issuer (D.7 finding); it is the VS-B remote-lateness guard.
- **If a coalesced (`LATEST_PER_ACTOR_PER_TICK`) intent is rejected `STALE`**: the actor's intent block retains the *previously applied* intent — a "stop" that went stale leaves the character moving. Consumer rule carried to the Player Controller GDD: an issuer receiving `STALE` for a coalescing type must immediately resubmit its current value (the input adapter's quantised-change filter must be bypassed for one send). The kernel never retries.
- **If a queued command's `max_age` would expire while the kernel is `PAUSED` or `SAVING`**: it cannot — `T` does not advance in either state, so age is frozen with it.

**Queue capacity**

- **If one issuer's pending count reaches `PER_ISSUER_CAPACITY` (256)**: that issuer's next submit is rejected `QUEUE_FULL`; all other issuers are unaffected. Supersession still works for that issuer (it frees a slot before the capacity check).
- **If the global pending count reaches `QUEUE_CAPACITY` (1024)**: every issuer's next submit, including the local player's and `SYSTEM` submits from hooks, is rejected `QUEUE_FULL`; the hook receives the result event at the next P5 and must tolerate rejection (CR-9 path (b) targets may reject anyway). In MVP this is unreachable (`2 issuers × 256 = 512 < 1024`); in VS-B it is reachable (Section G). *(Option, not spec)*: a `SYSTEM_RESERVE` of slots only `SYSTEM` may use so remote floods cannot starve the host's own systems.
- **If a submitted payload exceeds `MAX_PAYLOAD` (4 KiB) or contains an object reference, node path, callable or any non-plain type**: rejected `MALFORMED` at admission. In debug builds the round-trip test (CR-4) catches non-plain payloads that fit the size bound; in release `MALFORMED` is only raised by the size and type-tag check.

**Notification**

- **If a block's dirty set exceeds `MAX_DIRTY_KEYS` (256) in one tick**: `change_keys` collapses to `ALL` for that block only; other blocks deliver their key sets normally. The kernel does not retain the individual keys. Every subscriber must treat `ALL` as "re-read everything you use from this block".
- **If a block is written but every write restores the original value**: it is still dirty (the kernel marks writes, not diffs); the subscriber may re-read and find nothing changed.
- **If a `PRESENTATION` subscriber attempts a block write from its handler**: writes are disabled in P5 — the mutator asserts in debug, refuses in release, increments `WRITE_OUTSIDE_SCOPE`. The notification batch completes for all other subscribers.
- **If a `PRESENTATION` subscriber attempts to submit**: it has no submit handle — the handle is issued only to `INPUT_SURFACE` subscribers at subscribe time and is bound to the local seat (H4). If a `PRESENTATION` script obtains an `INPUT_SURFACE` handle by reference-passing, the kernel cannot distinguish the call and the control manifest/lint is the enforcement; the kernel-level guarantee is that no handle is ever *issued* to `PRESENTATION`.
- **If a subscriber handler throws**: the exception is caught by the batch loop, logged with the subscriber ID, and the remaining subscribers still receive the batch; the tick is already committed, so the sim is unaffected. It is a presentation defect, not a kernel fault.
- **If sim code (any call during P2 or P3, or any registered owner) requests a `PERCEPTION_OUT` read view**: debug assert naming the caller; release returns an empty view (no data, not null — callers should not crash) and increments `PERCEPTION_READ_BLOCKED`. The owner's own hooks are included (CR-12). A view obtained legitimately outside tick phases and cached for use inside one is a lint obligation.

**Snapshot and restore**

- **If `request_snapshot` is issued while `PAUSED`**: `PAUSED → SAVING` at the (idle) boundary; capture runs at the current `T` with no tick; return state is `PAUSED`.
- **If `request_snapshot` is issued during a cross-block `apply` or any mid-tick phase**: it is recorded and honoured at this tick's P6, after P4 commit; a snapshot can never observe a half-applied command or a mid-P3 state (CR-18, G6).
- **If a block's `snapshot()` throws or returns non-plain data**: capture error → `SAVING → FAULTED`; the partial snapshot is discarded and nothing is handed to Save/Load; the existing save file is untouched. The pending queue gets `Dropped(FAULTED)`.
- **If a restore payload lacks a registered `persist` block**: `default_state()` is used for that block, a diagnostics note is written, and restore proceeds (CR-8; Save/Load decides whether a missing block is acceptable before calling).
- **If a restore payload contains a `block_id` the registry does not know**: the kernel's **pre-check** (run over the whole payload before any `restore()` is called) fails with `RESTORE_UNKNOWN_BLOCK`; state is fully intact (nothing was touched) and the kernel goes `LOADING → FAULTED` per the transitions table. Save/Load is expected to strip or refuse unknown blocks first, so reaching this is a Save/Load defect.
- **If a block's stored `schema_version` is newer than the code's**: the owner reports unsupported (never a silent default) — caught in pre-check if owners declare `supported_versions`, otherwise at that block's `restore()`; either way `LOADING → FAULTED`. State is intact only if caught in pre-check.
- **If a block's stored `schema_version` is older and the owner has no migration for it**: same path and outcome as "newer".
- **If the snapshot header's `TICK_RATE` differs from the running constant**: *(adopted — decision log 2026-10-25; rationale: record `TICK_RATE` in the header)* pre-check fails `TICK_RATE_MISMATCH` → `FAULTED`; migrating every tick-expressed value in every block is impractical, so a save is bound to the tick rate that wrote it.
- **If any block's `post_restore()` fails (cross-block ID unresolved, invariant violated)**: `LOADING → FAULTED` with the block and invariant named. Because restore is whole-block replace and all blocks have been replaced by the time `post_restore` runs, **pre-restore state is not intact here** — "where possible" in CR-18 means pre-check failures only. *(Option)*: capture an in-memory snapshot of the live state before a mid-session restore so diagnostics can diff; automatic revert to it is out of scope (would need a second restore pass that could itself fail).
- **If `request_restore` arrives while `SAVING`**: it waits for the capture to finish (the kernel returns to its prior state at P6 and then honours the restore at that same boundary); the queue is flushed with `Dropped(RELOAD)` at that point.
- **If a restore completes while the pending queue is non-empty**: every pending command has already received `Dropped(RELOAD)` at the `→ LOADING` transition; the first tick after `RUNNING` sees an empty queue.

**Authority, classes and identity**

- **If an `INPUT_SURFACE` submits with an `actor` its seat's PlayerId does not control**: for `OWNED_BY_ISSUER` types → `ACTOR_NOT_OWNED` at P2; for `actor_binding = NONE` types the kernel does not check `actor` at all and the owner's `validate` decides. The surface can never choose its `issuer_peer` (stamped by the handle — H6), so "submitting as another seat" is unrepresentable.
- **If the binding table is changed during a tick**: the change is a control-plane request honoured at P6, so every authority check within one tick sees one consistent table; a command admitted before the change and dispatched after it is checked against the *new* table (authority is checked at P2, not at admission).
- **If a `PlayerId` is unbound from an entity while that entity has pending `OWNED_BY_ISSUER` commands**: those commands reach P2 and are rejected `ACTOR_NOT_OWNED`; they are not dropped early.
- **If a peer is bound to a PlayerId that already has persistent `PER_PLAYER` blocks**: nothing in the kernel changes — persistent state is keyed by PlayerId, never peer_id (Vocabulary); the binding only maps the new peer to the existing seat.

**Lifecycle, mode and tests**

- **If the kernel is `SAVING` when an invariant check (inside a block's `snapshot()`) fails**: see capture error above — `FAULTED`, snapshot discarded, existing save untouched.
- **If `FAULTED` is entered while a notification batch is mid-delivery**: impossible — faults originate in P2/P3 (apply, hook) or `SAVING`/`LOADING`; P5 handlers cannot fault the kernel (their exceptions are caught).
- **If `request_shutdown` arrives with queued commands**: every pending command receives `Dropped(SHUTDOWN)` synchronously during the `→ SHUTTING_DOWN` transition, then teardown; no further ticks, no snapshot (the FAULTED/shutdown rule: the kernel never autosaves on the way out).
- **If the tick counter would overflow**: it cannot — `T`, `seq` and `host_order` are 64-bit (D.14); the boot assertion on width is the only handling.
- **If anything attempts to construct or switch the kernel into `MIRROR` mode in MVP**: `BOOTING → FAULTED` with `MODE_UNSUPPORTED`; the `MIRROR` enum value exists so the API is mode-agnostic, but no MIRROR ingress, replication subscriber or constructor is compiled in MVP (H8). An acceptance test asserts that a MIRROR submit is impossible by asserting the construction fails.
- **If two kernel instances exist in one process (a test constructing two)**: each has its own registries, queue, clock, binding table and log; a block object registered with a second kernel while bound to a first fails that kernel's boot (`BLOCK_ALREADY_BOUND`) because owners are instance-scoped; an autoload, if one exists, is a thin holder of *one* instance and kernel code never reads it (H10), so the second instance is unaffected by whatever the autoload holds. gdUnit4's orphan-node check (exit 101) is avoided because neither instance is a Node.
- **If `TICK_RATE` is changed in code between two play sessions without a save in between**: nothing breaks at the kernel level, but every tick-expressed tuned value in every data file is now wrong by the ratio of the rates; this is why `TICK_RATE` is a set-once constant and why Section G lists every knob it invalidates.

## Dependencies

**Upstream (this system depends on): none.** The kernel is the Foundation layer. It depends on the engine's fixed-step callback and on Godot Physics 2D as a source of *observations* (CR-13), but on no game system.

**Downstream (systems that depend on this one).** Every row is a registrant: it registers blocks, command types or hooks, or subscribes. Hard = cannot function without the kernel; there are no soft dependents — a system that could work without the kernel would be writing state outside it, which H3/H4 forbid.

| System | Nature | Interface (kernel side) | Interface (consumer side) | In systems index? |
|--------|--------|-------------------------|---------------------------|-------------------|
| Life-Support Graph | Hard | Registry, step hooks, WriteScope, dirty keys | Blocks, propagation hook, published mutators | Yes |
| Save/Load & Versioned Format | Hard | Control plane (`request_snapshot`/`request_restore`), snapshot/restore semantics, `Dropped(RELOAD)` | File format, migrations, death stub | Yes |
| Player Controller & Input | Hard | Submit (as `INPUT_SURFACE`), coalescing, result events, `tick_alpha` | Intent and action command types | Yes |
| Suit & Equipment | Hard | `PER_PLAYER` blocks keyed by PlayerId, ordered step | Vitals/loadout block and commands | Yes |
| Modular Ship Assembly | Hard | Atomic cross-block `apply`, validate-then-apply | Weld/cut/install commands | Yes |
| Ship Body & Interior Physics | Hard | Tick alignment, `PROJECTOR` class | Physics-sampling hook, projection | Yes |
| Emitter State & Light-Exposure Query | Hard | Registry, pure-read query rules | Emitter block and commands | Yes |
| Web Map & Transit | Hard | Atomic data apply, hook | World block, `pod_hop` | Yes |
| Horror Entities | Hard | Ordered step, RNG streams, `SYSTEM` commands | Entity block, footprint mutator calls | Yes |
| Sanity & Hallucination Rules | Hard | `SIM`/`PERCEPTION_OUT` class isolation | Sanity level + perception blocks, drift hook | Yes |
| Perception Filter | Hard | `PRESENTATION` subscription, filtered channel | Reads only | **No — add edge** |
| Debug Overlay (True Values) | Hard | `DIAGNOSTIC` subscription | Reads only | Yes |
| Interaction System | Hard | Submit, domain reason codes, friendly-harm gate order | Verb command types | **No — add edge** |
| Attention System | Hard | Registry, step hook (breach state machine) | Attention block, stimulus intake | **No — add edge** |
| Ship Section & Layout Data Model | Hard | Registry (layout block), `PROJECTOR` | Layout block schema | **No — add edge** |
| Co-op Session & Replication (VS-B) | Hard | `MIRROR` mode, ingress adapter, binding table API, `REPLICATION` class, `scope`/`repl_hint` | Transport, replication | Yes |

**Bidirectional consistency.** The systems index lists 12 dependents; this GDD finds 16 — Interaction, Attention, Ship Section & Layout Data Model and Perception Filter all register with or subscribe to the kernel without listing it in `Depends On`. These are flagged for the systems-index update in Phase 5d (no GDD exists yet for any of them, so no other document needs correction). Conversely, the index's edge *Life-Support Graph → Save/Load* is satisfied by this kernel's registry contract and is a candidate for removal; it is left to the Graph GDD to confirm. The Item & Module Database is static definitions and needs no kernel edge.

**Data the kernel takes from consumers, and what it refuses.** It takes registrations (frozen at boot), commands (plain data), and subscriptions. It refuses: object references in any payload (H7), domain state in itself (H1), domain command types (H2), reads of `PERCEPTION_OUT` from sim code (H5), and any write outside a `WriteScope` (H3).

## Tuning Knobs

All knobs below are **provisional until the kernel spike**. Every knob traces to a Section D symbol. "Tunability": **set-once** = fixed at project start, changing it is a code/data migration; **boot** = registration data frozen at `BOOTING → LOADING`, changeable between runs; **runtime** = may be changed by a dev console between ticks (honoured at P6) without invalidating any stored value.

| Knob | Default | Safe range | What it affects (D ref) | Breaks too low | Breaks too high | Interactions | Tunability |
|------|---------|------------|-------------------------|----------------|-----------------|--------------|------------|
| `TICK_RATE` | 60 Hz | 15–120; practically {30, 60, 120} to align with Godot physics | `tick_duration`, `ticks_for_seconds`, every tick-expressed value (D.1), latency (D.4), all rates (D.2) | < 15 Hz fails the gameplay-programmer's floor: ≥ 67 ms sim latency, visible stepping; 30 Hz misaligns with 60 Hz physics (CR-1) | > 60 Hz doubles P2/P3 cost for no gain; the Graph still needs divisor rework to hold 10 Hz | **Invalidates every divisor, offset, `max_age_ticks`, `RATE_WINDOW_TICKS`, `LAG_FORGIVE_TICKS`, every owner's tick-stored duration and every save (`TICK_RATE_MISMATCH`)**. Must equal the physics step if the tick is physics-driven | **Set-once** |
| Hook `divisor` (per hook) | Graph 6, Attention 6, Suit 6, Sanity 12, input/physics 1 | 1–60; prefer divisors of 60 | `hook_runs_this_tick`, `effective_hook_rate`, `hooks_running_at` (D.2) | 1 for a slow system wastes P3 budget (Graph at 60 Hz ≈ 6× its cost) | Graph > 12 (< 5 Hz) makes atmosphere feedback visibly late; Sanity > 60 (< 1 Hz) makes drift stepwise | Changes the meaning of any `after` with a different-rate partner (same-tick data only when offsets coincide and divisors nest); `STEP_BUDGET_MS` spikes at `lcm(divisors)` | Boot (registration) |
| Hook `offset` (per hook) | 0 | 0–divisor−1 | `hook_runs_this_tick`, `hooks_running_at` (D.2) | All 0 → all slow hooks fire on the same tick: worst P3 spike | n/a (bounded by divisor) | Non-equal offsets between an `after` pair break same-tick data flow (Section E warning); use to flatten P3 load | Boot (registration) |
| `MAX_CMDS_PER_TICK` | 128 | 32–512 | `apply_tick`, `queue_drain_ticks` (D.4, D.8), per-tick P2 cost | < ~32: a weld burst or 4 seats' intents overflow, adding whole ticks of latency | > 512: a flood can stall P2 past the frame; no latency benefit beyond queue depth | `queue_drain_ticks = ceil(QUEUE_CAPACITY / this)` must stay below every `max_age_ticks` (D.7 invariant); log coverage worst case = `COMMAND_LOG_CAPACITY / this` | Runtime |
| `QUEUE_CAPACITY` | 1024 | 256–4096 | `admitted`, `queue_drain_ticks`, memory bound (D.8) | < issuers × `PER_ISSUER_CAPACITY` → one flooding issuer triggers *global* `QUEUE_FULL` and starves `SYSTEM` submits | > 4096: ≥ 17 MiB worst-case payload memory; drain time > 32 ticks → local STALE becomes reachable | MVP: 2 × 256 = 512 < 1024 (safe). VS-B 4 seats + SYSTEM: 5 × 256 = 1280 > 1024 → raise to ≥ 1280 or add `SYSTEM_RESERVE`. Must satisfy `ceil(this / MAX_CMDS_PER_TICK) ≤ min(max_age_ticks)` | Runtime (between sessions) |
| `PER_ISSUER_CAPACITY` | 256 | 64–QUEUE_CAPACITY | `admitted` (D.8) | < ~64: a legitimate local burst (menu + intents during a hitch frame) hits `QUEUE_FULL` | = `QUEUE_CAPACITY`: one issuer can take the whole queue | Should be ≤ `QUEUE_CAPACITY / max_issuers`; unreachable by a rate-limited remote (120/s vs 7,680/s drain) — only local or `SYSTEM` issuers can hit it | Runtime (between sessions) |
| `max_age_ticks` (per type) | intents 30 (0.5 s); actions 120 (2 s) | 8–600 | `is_stale` (D.7) | < `queue_drain_ticks` (8): a local command can go STALE purely from queue depth — the player's real input discarded | > 600: in VS-B a 10 s-late intent still applies, teleporting a guest's avatar | **Unreachable locally at defaults (D.7 finding)**; the remote clamp window in the VS-B ingress is `this + 1`; all values in ticks → invalidated by `TICK_RATE` | Boot (type registration) |
| `REMOTE_RATE_LIMIT` | 120 /s | 30–600 | `rate_limited` (D.8); dormant in MVP (H8) | < 30: a guest at 60 fps sending intent + tap is throttled; feels like dropped input | > 600: a guest can push 10 commands/tick, enough to monopolise `MAX_CMDS_PER_TICK` with 4 seats | Budget per window = `this × RATE_WINDOW_TICKS / TICK_RATE`; should be ≤ `PER_ISSUER_CAPACITY` per window so the rate limit, not capacity, is the first wall | Runtime |
| `RATE_WINDOW_TICKS` | 60 (1 s) | 6–120 | `rate_limited` (D.8) | 6: bursts of > 12 (a menu confirm with several sub-commands) are refused | 120: a 240-command burst passes in one frame | Tick-based (H9); scales with `TICK_RATE` | Runtime |
| `MAX_PAYLOAD` | 4 KiB | 1–64 KiB | `admitted`, queue memory bound, log size (D.8, D.13) | < 1 KiB: a module-install or chart-table command with a path array is `MALFORMED` | > 16 KiB: queue worst case ≥ 17 MiB; a remote can waste bandwidth | Memory bound = `QUEUE_CAPACITY × this`; log bound = `COMMAND_LOG_CAPACITY × this` | Boot |
| `MAX_DIRTY_KEYS` | 256 | 32–4096 | `change_keys` collapse (D.9) | < 32: routine Graph propagation collapses to `ALL` every run, forcing full redraws of trails/instruments at 10 Hz | > 4096: a breach cascade delivers thousands of keys and the subscriber's per-key loop costs more than a full refresh | Set near `c_full / c_key` of the heaviest subscriber; the Graph's largest derelict sets it (Interactions row 1) | Runtime |
| `MAX_TICKS_PER_FRAME` | 4 | 1–8 | `ticks_run`, `lag_ticks`, convergence floor (D.5) | 1: lag never repays — any frame > 16.7 ms permanently slows the sim | 8: a 133 ms frame runs 8 ticks (≈ 8 × tick cost) next frame → spiral risk; 8× fast-forward | `this × typical_tick_ms` must be well under `frame_duration` (`STEP_BUDGET_MS` is a log threshold, not a cap: 4 × 4 ms already fills a 60 fps frame); convergence floor = `TICK_RATE / this` fps (15 fps at defaults); must equal Godot `max_physics_steps_per_frame` if physics-driven | Boot |
| `LAG_FORGIVE_TICKS` *(adopted 2026-10-25)* | 30 (0.5 s) | 15–120, or ∞ (never forgive) | Lag discard in D.5 | 15: brief stutters discard sim time the player would not have noticed being replayed | ∞: a 1 s hitch is followed by 0.3 s at 4× — a feel/fairness hazard needing a gameplay-programmer / creative-director ruling | Only meaningful if the driver accumulates (not discards) excess — ADR-dependent; in ticks → `TICK_RATE`-bound | Runtime |
| `STEP_BUDGET_MS` | 4 ms | 1–8 ms | `step_over_budget` (D.6) — logging only | < 1 ms: every Graph tick logs; noise hides regressions | > 8 ms: a tick that fills half a frame goes unlogged | Diagnostic only; the real constraint is `MAX_TICKS_PER_FRAME × tick_cost < frame_duration` | Runtime |
| `SNAPSHOT_BUDGET_MS` | 16.7 ms (one frame) | 8–33 ms | `capture_time_ms` (D.11) — logging only | 8: every autosave on a large derelict logs | 33: a two-frame autosave hitch passes silently; VS-B guest transfer creeps | Per-block guideline = `this / |persist|` (≈ 0.5 ms at 32 blocks); drives the Graph's packed-array snapshot design | Runtime |
| `COMMAND_LOG_CAPACITY` | 1024 | 256–8192 | `log_coverage_ticks` (D.13) | 256: 2 ticks of history at saturation; a replay test cannot reach back past a weld burst | 8192: ≈ 34 MiB worst case with 4 KiB payloads | Memory = `this × (MAX_PAYLOAD + ~100 B)`; coverage worst case = `this / MAX_CMDS_PER_TICK` | Boot |
| `MAX_BLOCKS` | 128 (32 expected) | 32–256 | Registry capacity; `snapshot_bytes`, per-block budget (D.11) | < actual block count: boot fails | > 256: per-block snapshot budget drops below 0.07 ms — only trivially small blocks fit | Per-block budget = `SNAPSHOT_BUDGET_MS / |persist|`; `restore_after` topo-sort cost is negligible at this size | Boot |
| `SYSTEM_RESERVE` *(option, VS-B)* | 0 (MVP) / 128 (VS-B suggestion) | 0–PER_ISSUER_CAPACITY | `admitted` global term (D.8) | 0 in VS-B: a 4-seat flood starves `SYSTEM` submits (Horror, Transit) | ≥ `PER_ISSUER_CAPACITY`: wasted queue | Effective player capacity = `QUEUE_CAPACITY − this` | Runtime |
| `H64` / `mix64` choice | xxHash64 / SplitMix64 finaliser (→ ADR) | n/a | `state_hash`, `rng_derive` (D.10, D.12) | A weak mix correlates streams (`horror.tell` and `graph.failure` draw the same pattern) | n/a | **Set-once**: changing either invalidates every recorded replay hash and every seeded world; not a tuning knob — listed so nobody treats it as one | **Set-once** |

**Coupled constraints (boot assertions — each is one acceptance test):**
1. `max_age_ticks(t) ≥ ceil(QUEUE_CAPACITY / MAX_CMDS_PER_TICK)` for every type `t` — "a local command is never STALE" (D.7, option a).
2. `QUEUE_CAPACITY ≥ max_issuers × PER_ISSUER_CAPACITY + SYSTEM_RESERVE` — "one issuer cannot cause global `QUEUE_FULL`" (MVP: `max_issuers = 2`).
3. `0 ≤ offset_h < divisor_h` and `divisor_h ≥ 1` for every hook (D.2).
4. `MAX_TICKS_PER_FRAME == physics max_physics_steps_per_frame` when physics-driven (D.5, ADR-dependent).
5. `TICK_RATE` in the snapshot header equals the running constant (proposed, Section E).
6. `T`, `seq`, `host_order`, RNG counters are 64-bit (D.14).

**Set-once versus runtime, summarised.** Set-once: `TICK_RATE`, `H64`/`mix64`. Changing `TICK_RATE` after any data file or save exists requires rescaling every tick-expressed value by `new/old` and invalidates every existing save; treat it as a project-start decision confirmed by the kernel spike. Boot-frozen (registration data; change between runs, never mid-session): divisors, offsets, `max_age_ticks`, `MAX_PAYLOAD`, `MAX_TICKS_PER_FRAME`, `COMMAND_LOG_CAPACITY`, `MAX_BLOCKS`. Runtime (dev console, honoured at P6, no stored value depends on them): `MAX_CMDS_PER_TICK`, `QUEUE_CAPACITY`, `PER_ISSUER_CAPACITY`, `REMOTE_RATE_LIMIT`, `RATE_WINDOW_TICKS`, `MAX_DIRTY_KEYS`, `LAG_FORGIVE_TICKS`, `STEP_BUDGET_MS`, `SNAPSHOT_BUDGET_MS`, `SYSTEM_RESERVE`.


> **Adopted (CD-GDD-ALIGN, 2026-10-25)**: `LAG_FORGIVE_TICKS` is ON with the proposed 30-tick starting value, final value from the spike — lost time is *dropped*, never fast-forwarded, because replaying a 10–30 s tell window at 4× while the player reacts at 1× compresses the tell and reads as unfair; since no domain system reads wall-clock time the drop is observable only in diagnostics. `allowed_when_paused` is restricted to non-diegetic actions (CR-16), so the equip-while-paused edge case above resolves as `Rejected(PAUSED)`.

## Visual/Audio Requirements

None of its own — the kernel renders nothing and plays nothing (H4, CR-11). What it *requires* of visual and audio systems is structural: every presentation element subscribes as `PRESENTATION` (or `INPUT_SURFACE`), reads last-committed state through read views, and interpolates with `tick_alpha` where motion must be smooth. Two consequences for the art bible's rules are worth stating here so they are not rediscovered per system: (1) the art bible's truth-channel / corruptible tag contract (Perception Filter) maps onto the kernel's two consumer channels — truth-channel elements read `SIM` views directly, corruptible elements read the Filter's `PERCEPTION_OUT`-derived channel; (2) the "nothing fails silently" rule of Pillar 1 is served by the kernel only in that every state change produces a notification — *what* is drawn for it is the owning system's Visual/Audio section. No asset-spec flag: there are no assets.

## Game Feel

Optional for a `Core` system; one target is recorded because it is the kernel's doing. **Input-to-effect latency is bounded at one tick plus one render frame (≤ 33 ms at 60 Hz / 60 fps), and acknowledgement is immediate.** The gameplay-programmer's feel bar: a press should produce a local response in the same frame (tool sound, hand animation start, progress ring beginning), with the sim confirming on the next tick; the only visible effect of the command path is a ring that begins a tick later. Movement never waits for a tick because integration runs on the physics side from the current intent (CR-4). Two anti-targets: no synchronous local fast path for results (it would not survive a remote issuer), and no client-side prediction in MVP (the `seq`/`queued_tick` fields and determinism guarantees are the hooks for it later). Reduced-motion settings never alter the tick (CR-11), so feel and accessibility settings cannot change simulation outcomes.

## UI Requirements

No player-facing UI. The kernel exposes one **developer-only** surface, consumed by the Debug Overlay (True Values) as a `DIAGNOSTIC` subscriber: current tick and kernel state; queue depth and per-issuer depth; the last-N command log with results and reject reasons; resolved hook order with per-hook timings and the `STEP_BUDGET_MS` overrun flag; rejection counters by reason; catch-up lag counter; `state_hash` of the committed tick. It must be absent from release builds and is never an accessibility feature (art bible R-7, concept risks). Player-facing *consequences* of kernel behaviour — rejection feedback, honest pending-vs-confirmed states on channels — are UI requirements of Interaction, HUD & Suit Layer and In-World Instruments, which must map reason codes to feedback without display strings ever entering the kernel (CR-5). *(No UX flag: nothing here is a player screen.)*

## Cross-References

No other GDD exists yet; every reference below is to the systems index entry and the contract this document imposes. When each GDD is authored it must cite this one for the mechanism it uses.

| Referenced system | What this GDD references | Where | Target GDD |
|-------------------|--------------------------|-------|------------|
| Life-Support Graph | Pure join-validation query; footprint-channel mutator; propagation hook at divisor 6 | CR-3, CR-9, C.3 #1 | `design/gdd/life-support-graph.md` (not yet written) |
| Save/Load & Versioned Format | Snapshot/restore semantics; death stub; file format ownership | CR-8, CR-18, C.3 #3 | not yet written |
| Player Controller & Input | Intent coalescing; position ownership question | CR-4, C.3 #2 | not yet written |
| Ship Body & Interior Physics | `PROJECTOR`; tick/physics alignment; G3 limit | CR-10, CR-13, C.3 #6 | not yet written |
| Ship Section & Layout Data Model | Layout block; ID allocation on detach | CR-8, C.3 #5 | not yet written |
| Modular Ship Assembly | Atomic cross-block apply | CR-9, C.3 #5 | not yet written |
| Emitter State & Light-Exposure Query | Query callable only from validate/P3/prompts | C.3 #7 | not yet written |
| Attention System | Stimulus intake API; breach state machine hook; tell window in ticks | CR-1, C.3 #13 | not yet written |
| Sanity & Hallucination Rules | `PERCEPTION_OUT` class; PlayerId keying; friendly-harm gate on `SIM` sanity level | CR-7, CR-12, C.3 #10 | not yet written |
| Interaction System | Domain reason codes; honest rejections; friendly-harm act | CR-5, CR-7, C.3 #12 | not yet written |
| Perception Filter / Debug Overlay | Two consumer channels; `DIAGNOSTIC` surface | CR-10, UI Requirements | not yet written |
| Co-op Session & Replication (VS-B) | `MIRROR` mode; ingress adapter; binding table; `repl_hint` | CR-7, CR-10, C.3 #14 | not yet written |
| Game concept | Pillars 1 and 4; TD-FEASIBILITY constraints (authority from day one, hallucinations never touch simulation, host-owned saves) | Overview, Player Fantasy | `design/gdd/game-concept.md` |
| Art bible | Truth-channel / corruptible tag contract; R-7 (debug overlay never an accessibility feature); 8.11 control-manifest rule | Visual/Audio, UI Requirements | `design/art/art-bible.md` §5.5, §7.6, §8.11 |
| Engine reference | networking.md (RPC pattern, Spawner/Synchronizer); breaking-changes.md (typed-return overrides, packed-array setters, `duplicate_deep`, `FileAccess` bools); gdUnit4 notes | C.3 coverage note | `docs/engine-reference/godot/` |

## Acceptance Criteria

*Every criterion is pass/fail without reading the rest of this document. Tags in brackets name the Core Rule, guarantee, hard rule or Section D formula the criterion discharges. Evidence class per criterion: **(auto)** gdUnit4 test under `tests/unit/kernel/`, run headless by `commands.test` (exit 0 = pass; 100 = failures; 101 = passed but leaked orphan Nodes, which counts as a FAIL for this system); **(lint)** a static scan run in CI; **(review)** a code-review checkbox; **(manual)** an instrumented play on a dev build with retained evidence in `production/qa/evidence/`. Items marked *(adopted — decision log 2026-10-25)* rest on a Section E/D proposal that is not yet spec; they become binding when the 2026-10-25 decision log adopts them, and must be deleted — not quietly kept — if it does not. All numeric defaults are the Section G values and are provisional until the kernel spike.*

### Tick & phases

- **GIVEN** a `HOST` kernel in `RUNNING` with `TICK_RATE = 60`, **WHEN** `Kernel.step()` is called 60 times — once in a tight loop, once with a 50 ms wall sleep between calls, **THEN** in both runs `kernel.clock.T` increases by exactly 60 and the simulated time derived from it by exactly 1.000 s; the two runs' `kernel.clock` blocks are byte-identical. (auto) [CR-1; `tick_duration`]
- **GIVEN** `TICK_RATE = 60`, **WHEN** `ticks_for_seconds(s)` is evaluated for `s` = 10.0, 30.0, 0.004, 0.0084 and 0.0083, **THEN** the results are 600, 1800, 0, 1 and 0 (round half-up; 0.0083 s < half a tick). (auto) [D.1 `ticks_for_seconds`] *(adopted)*
- **GIVEN** the kernel source directory and every registered block's source, **WHEN** a lint scans for `Time.get_ticks_msec`, `Time.get_ticks_usec`, `Time.get_unix_time`, `get_process_delta_time`, `get_physics_process_delta_time`, `randi`, `randf`, `randomize`, `seed(` and `RandomNumberGenerator`, **THEN** zero matches outside the tick driver and the diagnostics timer (the only two allow-listed files). (lint) [CR-1, CR-14 H9]
- **GIVEN** a `RUNNING` kernel with one `PRESENTATION` subscriber and one queued command whose `apply` writes key `k` of block `B`, **WHEN** one tick runs, **THEN** the diagnostics phase trace for that tick reads exactly `P0, P1, P2, P3, P4, P5, P6` with each phase's end timestamp ≤ the next phase's start, and the subscriber's handler runs exactly once, during P5, and reads the *new* value of `k`. (auto) [CR-2, G6]
- **GIVEN** a `RUNNING` kernel at tick `T`, **WHEN** a command is submitted from inside a P2 `apply` (as `SYSTEM`) and another from inside a P5 `INPUT_SURFACE` handler, **THEN** both are admitted with `queued_tick == T`, neither appears in tick `T`'s P1 batch, and both are dispatched at `T+1`. (auto) [CR-2; D.4 `apply_tick`]
- **GIVEN** two commands `A` then `B` on the same actor queued for one tick, where `A.apply` sets key `x = 5`, **WHEN** the tick runs, **THEN** `B.validate` and `B.apply` observe `x == 5` through their read views, and a P3 hook on the same block observes every write made in P2. (auto) [CR-2]
- **GIVEN** hooks registered in the order `Z (after: [Y])`, `Y (after: [X])`, `X (after: [])`, plus `P` and `Q` with no `after` relation registered `P` then `Q`, **WHEN** boot completes, **THEN** the diagnostics hook order lists `X` before `Y` before `Z` and `P` before `Q`; re-registering `Q` before `P` yields `Q` before `P`; the order is identical on 10 consecutive boots. (auto) [CR-3]
- **GIVEN** hooks `A (after: [B])` and `B (after: [A])`, **WHEN** boot runs, **THEN** kernel state is `FAULTED` and diagnostics names the cycle path (`A → B → A`); **GIVEN** a hook whose `after` names an unregistered `hook_id`, **THEN** `FAULTED` naming the unknown ID and its referrer. Both outcomes hold in a release export as well as a debug run. (auto; release case manual) [CR-3, CR-6]
- **GIVEN** hooks `graph (6, 0)`, `sanity (12, 0)`, `input (1, 0)` and `stagger (6, 2)`, **WHEN** ticks 1–12 run, **THEN** per-hook run counters read `input = 12`, `graph = 2` (T = 6, 12), `sanity = 1` (T = 12), `stagger = 2` (T = 2, 8). (auto) [CR-1, CR-3; D.2 `hook_runs_this_tick`]
- **GIVEN** `TICK_RATE = 60` and hooks at divisors 1, 6 and 12, **WHEN** the diagnostics surface reports `effective_hook_rate`, **THEN** it reads 60.0, 10.0 and 5.0 Hz respectively. (auto) [D.2 `effective_hook_rate`]
- **GIVEN** the default hook set (divisors `{1, 6, 6, 6, 12}`, all offsets 0), **WHEN** diagnostics reports `hooks_running_at(T)` for `T` = 7, 6 and 12, **THEN** the values are 1, 4 and 5. (auto) [D.2 `hooks_running_at`]
- **GIVEN** a hook registered with `divisor = 0`, or `offset = -1`, or `offset = 6` with `divisor = 6`, **WHEN** boot runs, **THEN** `BOOTING → FAULTED` and diagnostics names that hook. (auto) [CR-6; D.2; Section G coupled constraint 3]
- **GIVEN** hook `suit (6, 4) after: [graph]` and `graph (6, 0)`, **WHEN** boot runs, **THEN** boot succeeds (`LOADING` is reached) and diagnostics contains one warning naming the pair `suit`/`graph`. (auto) [CR-3] *(adopted — decision log 2026-10-25)*
- **GIVEN** a step hook that calls `submit_system()`, **WHEN** the submitted command is dispatched, **THEN** its `origin == SYSTEM`, `issuer_peer == 1`, and its `seq` comes from the kernel's `SYSTEM` sequence; **GIVEN** a step hook that instead reaches the local input ingress during P3, **THEN** in a debug build an assertion fires, and in release the command is `Rejected(ORIGIN_NOT_ALLOWED)` at admission and the corresponding diagnostics counter increments by 1 — it is never re-stamped `SYSTEM`. (auto) [CR-3, CR-4, CR-14 H6]
- **GIVEN** a `RUNNING` kernel, **WHEN** 1,000 read-view reads, 1,000 `tick_alpha` reads and 100 subscriber notifications occur with no `step()` call, **THEN** `T` is unchanged. (auto) [CR-11]
- **GIVEN** `tick_duration = 1/60`, **WHEN** the driver computes `tick_alpha` at `t_since_last_commit` = 0, 0.007, 0.01667 and 0.050 s, **THEN** the values are 0.0, 0.42 (± 0.01), 1.0 and 1.0 (clamped, never > 1). (auto) [CR-10; D.3 `tick_alpha`]
- **GIVEN** a block exposing a moving value, **WHEN** two ticks commit values 10 then 12, **THEN** the read view returns `previous == 10` and `current == 12`, and the interpolated accessor at `tick_alpha = 0.5` returns 11. (auto) [CR-10]
- **GIVEN** `PAUSED` at `T = 500` and a command type flagged `allowed_when_paused` whose `apply` writes block `M` (a non-gameplay menu/settings block, per CR-16), **WHEN** the command is submitted and one paused service cycle runs, **THEN** the result is `Applied` with `tick == 500`, `M`'s notification is delivered with `tick == 500`, `T` is still 500, and every hook run counter is unchanged (no P3). (auto) [CR-16] *(adopted — decision log 2026-10-25)*
- **GIVEN** `PAUSED`, **WHEN** a command whose type is not `allowed_when_paused` is submitted, **THEN** `Rejected(PAUSED)` is emitted during the `submit` call and queue depth is unchanged; **WHEN** `request_resume` is then called, **THEN** the first running tick's P1 batch contains zero commands from the paused period. (auto) [CR-16]
- **GIVEN** the kernel has been `PAUSED` for 10 s of wall time, **WHEN** `request_resume` is honoured, **THEN** the driver's accumulator reads 0 and the first running frame runs at most the ticks due to that frame's own delta (1 at 60 fps), never a catch-up burst. (auto) [CR-16; D.5]
- **GIVEN** `RUNNING` at tick `T`, **WHEN** `request_pause` is called from inside a P2 `apply`, **THEN** the kernel state observed by every P5 subscriber of tick `T` is still `RUNNING`, and the state after P6 is `PAUSED`. (auto) [CR-15]
- **GIVEN** the kernel in `BOOTING`, **WHEN** `request_pause` is called, **THEN** the state is unchanged and diagnostics records one invalid-transition entry naming `(BOOTING, request_pause)`; the same holds for `(SAVING, request_pause)` and `(FAULTED, request_resume)`. (auto) [States and Transitions]
- **GIVEN** `FAULTED`, **WHEN** `request_restore` is called with a valid snapshot, **THEN** the state becomes `LOADING` then `RUNNING`; **WHEN** `request_shutdown` is called instead, **THEN** `SHUTTING_DOWN`; no other trigger changes the state. (auto) [States and Transitions]

### Commands & authority

- **GIVEN** a `RUNNING` kernel, **WHEN** commands are submitted whose payloads contain, respectively, an `Object` reference, a `NodePath`, a `Callable`, and a 4,097-byte serialized form at `MAX_PAYLOAD = 4 KiB`, **THEN** each is `Rejected(MALFORMED)` at admission and the queue depth is unchanged. (auto) [CR-4, CR-14 H7; D.8 `admitted`]
- **GIVEN** a debug build, **WHEN** any command is dispatched, **THEN** the handler receives an instance that is value-equal to but not reference-identical with the submitted one, and the diagnostics counter `CMD_ROUNDTRIPPED` equals the number of dispatched commands. (auto) [CR-4]
- **GIVEN** any command, **WHEN** `submit` is called, **THEN** the call returns nothing (`void`/`null`) and the only outcome is one result event keyed `(issuer_peer, seq)`. (auto) [CR-4]
- **GIVEN** intent `A` (`set_move_intent`, `coalesce = LATEST_PER_ACTOR_PER_TICK`, actor 7, peer 1) in the unfrozen pending set, **WHEN** intent `B` with the same `(type, actor, issuer_peer)` is submitted before the next P0, **THEN** `A`'s result `Superseded` is emitted during `B`'s `submit` call, the pending set holds `B` and not `A`, and `B.host_order > A.host_order`. (auto) [CR-4] *(adopted)*
- **GIVEN** the sequence `[intent A, tap begin_weld (coalesce = NONE), intent B]` submitted in that order within one inter-tick window, where `A` and `B` share a coalesce key, **WHEN** the next tick runs, **THEN** exactly three result events are emitted (`A = Superseded`, tap = `Applied` or a domain rejection, `B = Applied`), the command log shows the tap dispatched before `B`, and the tap's `validate` observed the intent state as it was *before* `B`. (auto) [CR-4, G1] *(adopted — decision log 2026-10-25)*
- **GIVEN** intent `A` already frozen into tick `T`'s P1 batch, **WHEN** same-key intent `B` is submitted during P2 of `T`, **THEN** `A` is `Applied` at `T`, `B` is `Applied` at `T+1`, and no `Superseded` result is emitted. (auto) [CR-4, CR-2]
- **GIVEN** two `coalesce = NONE` `interact` commands submitted in one inter-tick window, **WHEN** the tick runs, **THEN** both are dispatched and two result events are emitted. (auto) [CR-4]
- **GIVEN** `LATEST_PER_ACTOR_PER_TICK` intents from peer 1 and from a test peer 2 on the same actor and type, **WHEN** both are pending, **THEN** neither supersedes the other (coalesce key includes `issuer_peer`). (auto) [CR-4]
- **GIVEN** 1,000 commands covering every terminal (`Applied`; every kernel reject reason reachable locally; `Dropped(RELOAD)`, `Dropped(SHUTDOWN)`, `Dropped(FAULTED)`; `Superseded`), **WHEN** the run ends, **THEN** exactly 1,000 result events were emitted and the multiset of `(issuer_peer, seq)` in results equals the submitted set — no duplicates, no omissions. (auto) [CR-5, G5]
- **GIVEN** a command that fails two admission terms at once (unregistered type *and* full queue), **WHEN** submitted, **THEN** the reason is `UNKNOWN_TYPE` (first failing term in D.8's order), and the identical command resubmitted gets the identical reason. (auto) [CR-5; D.8 `admitted`]
- **GIVEN** any result event, **WHEN** inspected, **THEN** it has exactly the fields `seq`, `issuer_peer`, `ok`, `reason`, `subject`, `detail`, `tick`; `reason` is an enum/int code, and a recursive walk of `detail` finds no `String` values. (auto) [CR-5]
- **GIVEN** a fixture block whose `SIM` state marks a device `working` and a `PERCEPTION_OUT` fixture that marks it `broken`, **WHEN** a repair command on it is dispatched, **THEN** the result is `Rejected(VALIDATION_FAILED(PRECONDITION))` — computed from `SIM`, so the rejection reports the truth. (auto) [CR-5, CR-12]
- **GIVEN** a command that was `Rejected`, **WHEN** 100 further ticks run, **THEN** the command log contains exactly one entry for its `(issuer_peer, seq)`. (auto) [CR-5]
- **GIVEN** a command type with `allowed_origins = {SYSTEM}`, **WHEN** a `PLAYER` command of that type is submitted, **THEN** it is admitted (type is registered) and `Rejected(ORIGIN_NOT_ALLOWED)` at P2 with `result.tick == apply_tick`. (auto) [CR-7]
- **GIVEN** a type with `actor_binding = OWNED_BY_ISSUER`, peer 1 bound to entity 7 only, **WHEN** peer 1 submits with `actor = 9`, `actor = none`, and `actor = 7`, **THEN** the first two are `Rejected(ACTOR_NOT_OWNED)` and the third passes authority (reaches `validate`). (auto) [CR-7]
- **GIVEN** a single-player boot, **WHEN** the local input adapter submits one command, **THEN** the command carries `issuer_peer == 1`, `origin == PLAYER`, and the diagnostics authority-check counter increments by exactly 1 — the authority stage ran. (auto) [CR-7]
- **GIVEN** a binding change requested during P2 of tick `T` and an `OWNED_BY_ISSUER` command queued for `T` after it, **WHEN** `T` completes, **THEN** every authority check in `T` used the pre-change table and the first check in `T+1` uses the new table; **GIVEN** a `PlayerId` unbound while it has pending `OWNED_BY_ISSUER` commands, **THEN** those commands reach P2 and are `Rejected(ACTOR_NOT_OWNED)`, not dropped. (auto) [CR-7]
- **GIVEN** a snapshot taken with a populated binding table, **WHEN** its payload is inspected, **THEN** it contains no binding data (only `persist` block entries plus the header). (auto) [CR-7]
- **GIVEN** an MVP build, **WHEN** a kernel is constructed in `MIRROR` mode, **THEN** `BOOTING → FAULTED` with `MODE_UNSUPPORTED` and no `submit` is possible. (auto) [CR-7, CR-14 H8]
- **GIVEN** the command constructor/factory, **WHEN** called without `issuer_peer`, **THEN** construction is refused (debug assertion; release returns an error, no instance); **GIVEN** an `INPUT_SURFACE` handle bound to seat 1, **WHEN** it submits a payload containing `{"issuer_peer": 2}`, **THEN** the admitted command's `issuer_peer == 1` and the payload key has no effect. (auto) [CR-14 H6]
- **GIVEN** issuer 1 with `last_seq = 3`, **WHEN** it submits `seq = 3`, `seq = 2`, `seq = 5`, **THEN** the first two are `Rejected(DUPLICATE_SEQ)`; the third is admitted, `last_seq` becomes 5 and the per-issuer gap counter increments by 1. (auto) [CR-5] *(adopted)*
- **GIVEN** issuer 1 with `last_seq = 40`, **WHEN** its binding is destroyed and recreated, **THEN** `last_seq` reads 0 and `seq = 1` is admitted, while `host_order` continues from its previous value. (auto) [CR-5; D.7]
- **GIVEN** the host player and a step hook both at `seq = 1`, **WHEN** both submit, **THEN** both are admitted (seq spaces are per `(issuer_peer, origin)`). (auto) [CR-4]
- **GIVEN** a block mutator, **WHEN** it is called from a notification handler, from `validate`, and from a presentation script, **THEN** in debug an assertion fires each time; in release the stored value is unchanged, no dirty key is marked, `WRITE_OUTSIDE_SCOPE` increments by 3 and the kernel state is unchanged (not `FAULTED`). (auto) [CR-9, CR-14 H3]
- **GIVEN** a weld-fixture command whose `apply` calls published mutators on blocks `X` and `Y`, **WHEN** it is applied, **THEN** P5 delivers exactly one batch containing both `X` and `Y` entries with the same `tick`, and a subscriber reading both views in that batch sees both writes. (auto) [CR-9, G6]
- **GIVEN** a fixture command whose `apply` writes block `X` and then asserts, two further commands in the same P1 batch and three more pending, **WHEN** the tick runs, **THEN** the kernel is `FAULTED`; no P4/P5 ran (no subscriber invoked, tick-committed marker unchanged); all six commands receive `Dropped(FAULTED)`; diagnostics holds `(T, type, issuer, host_order, block_id, assertion)`; the command log contains no `Applied` entry for the faulting command. A `validate` that throws produces the same outcome. (auto) [CR-9, CR-5] *(adopted)*
- **GIVEN** every registered command type, **WHEN** `validate` is called with a fixture command on a `RUNNING` kernel outside P2, **THEN** `state_hash(ALL_SIM)` is identical before and after and no dirty key is marked — `validate` has no side effects. (auto) [CR-9]
- **GIVEN** `FAULTED`, **WHEN** `request_snapshot` is called or Save/Load's autosave timer fires, **THEN** no snapshot payload is handed to Save/Load and the call is recorded as an invalid `(FAULTED, request_snapshot)` pair. (auto) [States and Transitions — FAULTED rule] The consequence that an existing save file's bytes stay unchanged is Save/Load's criterion, not the kernel's.

### Registry & snapshots

- **GIVEN** a kernel in `LOADING` or `RUNNING`, **WHEN** `register_block`, `register_command_type` or `register_hook` is called, **THEN** the call is refused with an error and every registry count is unchanged. (auto) [CR-6]
- **GIVEN** `BOOTING`, **WHEN** two blocks register the same `block_id`, or a command type names an owner that registered no block, **THEN** `BOOTING → FAULTED` naming the duplicate ID or the missing owner — in a release export as well. (auto; release case manual) [CR-6]
- **GIVEN** every registered block, **WHEN** its registration is inspected, **THEN** `block_id` contains a namespace separator, `schema_version ≥ 1`, `class ∈ {SIM, PERCEPTION_OUT}`, `persist` is a bool, `scope ∈ {SHARED, PER_PLAYER}`, `repl_hint ∈ {HOST_ONLY, SHARED_SUMMARY, OWNER_PRIVATE}`; a fixture registration omitting `repl_hint` fails boot. (auto) [CR-8]
- **GIVEN** every registered block constructed with no `SceneTree` present, **WHEN** `snapshot() → to_dict() → JSON.stringify → JSON.parse → from_dict() → snapshot()` runs, **THEN** the final canonical bytes equal the first, no block `is Node`, and the gdUnit4 run exits 0 — not 101 (leaked orphan Nodes). (auto) [CR-8, CR-14 H7, H10]
- **GIVEN** a restore payload where one block's `schema_version` is newer than the code's, and another where it is older with no migration, **WHEN** `request_restore` runs, **THEN** `LOADING → FAULTED` in both cases, the failing block is named, and `default_state()` was never substituted. (auto) [CR-8, CR-18]
- **GIVEN** a restore payload lacking one registered `persist` block, **WHEN** restored, **THEN** that block equals `default_state()`, a diagnostics note names it, and the kernel reaches `RUNNING`. (auto) [CR-8]
- **GIVEN** a restore payload containing a `block_id` the registry does not know, **WHEN** `request_restore` runs, **THEN** the pre-check fails `RESTORE_UNKNOWN_BLOCK`, no block's `restore()` was called, `state_hash(PERSIST)` is unchanged from before the request, and the state is `FAULTED`. (auto) [CR-8, CR-18]
- **GIVEN** blocks `B (restore_after: [A])`, **WHEN** a restore runs, **THEN** `A.restore` completes before `B.restore` begins, and every `post_restore()` runs only after every `restore()`; **GIVEN** `B.post_restore()` fails, **THEN** `LOADING → FAULTED` naming `B` and the violated invariant. (auto) [CR-8]
- **GIVEN** a booted kernel with no domain registrations, **WHEN** the registry is listed, **THEN** it holds exactly one block, `kernel.clock`, whose canonical form has exactly the keys `T` and `world_seed`. (auto) [CR-8, CR-14 H1]
- **GIVEN** a fixture block that allocates entity IDs 1–3 from its own counter, **WHEN** it is snapshotted, restored into a fresh kernel and allocates again, **THEN** the next ID is 4 and the kernel allocated none (kernel exposes no ID allocator). (auto) [CR-8]
- **GIVEN** `world_seed = S`, **WHEN** `rng_derive(S, "horror.tell", 17)` is evaluated 1,000 times in arbitrary interleaving with draws from `"graph.failure"`, **THEN** every evaluation returns the same u64; `unit` is in `[0, 1)` for 100,000 draws; `rng_int(0, 1)` over 1,000 draws yields both 0 and 1 and nothing else; `rng_int(600, 1800)` over 10,000 draws stays within `[600, 1800]`; and streams `"horror.tell"` and `"graph.failure"` differ at ≥ 99 of their first 100 counters. (auto) [CR-8; D.10 `rng_derive`]
- **GIVEN** `request_snapshot` called from inside a P2 `apply` at tick `T`, **WHEN** the tick completes, **THEN** the snapshot's header `T` equals the tick's `T`, the snapshot's hash equals `state_hash(PERSIST, T)` computed after P4, every `persist` block appears with its `schema_version`, and no `persist = false` block appears. (auto) [CR-18, G6; D.12]
- **GIVEN** `RUNNING`, **WHEN** `request_snapshot` is honoured, **THEN** the state sequence observed by diagnostics is `RUNNING → SAVING → RUNNING`; from `PAUSED` it is `PAUSED → SAVING → PAUSED` with `T` unchanged throughout; a command submitted while `SAVING` is admitted with `queued_tick == T`, is absent from the snapshot, and is dispatched at `T+1`. (auto) [CR-18, CR-16]
- **GIVEN** a block whose `snapshot()` throws or returns a non-plain value, **WHEN** capture runs, **THEN** `SAVING → FAULTED`, nothing is handed to Save/Load, and every pending command receives `Dropped(FAULTED)`. (auto) [CR-18]
- **GIVEN** `RUNNING` with 50 pending commands, **WHEN** `request_restore` is honoured, **THEN** all 50 receive `Dropped(RELOAD)` at the `→ LOADING` transition and the first `RUNNING` tick's P1 batch is empty; **GIVEN** `request_restore` arrives while `SAVING`, **THEN** the capture completes first and the restore is honoured at that same boundary. (auto) [CR-18, States]
- **GIVEN** a snapshot whose header `TICK_RATE` is 30 while the running constant is 60, **WHEN** restored, **THEN** the pre-check fails `TICK_RATE_MISMATCH` and the state is `FAULTED` with pre-restore state intact. (auto) [CR-18; Section G constraint 5] *(adopted — decision log 2026-10-25)*
- **GIVEN** 32 fixture `persist` blocks, **WHEN** a snapshot is captured, **THEN** diagnostics reports `snapshot_bytes` equal to `header_bytes + Σ(block_header_bytes + canonical_bytes(b))` as independently summed by the test over the same blocks. (auto) [D.11 `snapshot_bytes`]
- **GIVEN** `SNAPSHOT_BUDGET_MS = 16.7` and the diagnostics timer stubbed to report per-block times summing to 25 ms, **WHEN** a snapshot is captured, **THEN** the snapshot is complete (every `persist` block present), the over-budget flag is set and the blocks are logged slowest-first; with stubbed times summing to 7.6 ms the flag is clear. (auto) [CR-18; D.11 `capture_time_ms`]

- **GIVEN** a `PROFILE`-scoped block holding three decoded rules and a `WORLD`-scoped block, **WHEN** the world is snapshotted at `T = 300`, both blocks are mutated, and `request_restore` to the `T = 300` snapshot is honoured, **THEN** the `WORLD` block equals its `T = 300` state and the `PROFILE` block still holds all post-300 mutations — losing the world never loses knowledge (Pillar 5). (auto) [CR-8, CR-18 `persist_scope`]
- **GIVEN** the G4 save-equivalence fixture, **WHEN** `state_hash(PERSIST)` is computed, **THEN** it covers `WORLD`-scoped blocks only, and a `PROFILE` block changed between the two runs does not break equality. (auto) [CR-13 G4, CR-18]

### Notifications & isolation

- **GIVEN** block `B` written at keys `a`, `b`, `b` during P2/P3 of tick `T` and block `C` not written, **WHEN** P5 runs, **THEN** exactly one notification `(B, T, {a, b})` is delivered, none for `C`, and the notification record carries only `block_id`, `tick`, `change_keys` — no state values. (auto) [CR-10]
- **GIVEN** a block key written to its existing value, **WHEN** P5 runs, **THEN** the key is in `change_keys` (writes are marked, not diffed). (auto) [CR-10; D.9]
- **GIVEN** `MAX_DIRTY_KEYS = 256`, **WHEN** block `B` dirties exactly 256 keys and sibling block `C` dirties 3 in one tick, **THEN** `B`'s notification lists 256 keys and `C`'s lists 3; **WHEN** `B` dirties 257, **THEN** `B`'s `change_keys == ALL` and `C`'s still lists 3. (auto) [CR-10; D.9 `change_keys`]
- **GIVEN** a `MAX_DIRTY_KEYS` change requested via the runtime knob during P2 of tick `T`, **WHEN** `T`'s P5 runs, **THEN** it uses the old value, and `T+1`'s P5 uses the new. (auto) [Section G runtime knobs]
- **GIVEN** subscriptions as `PRESENTATION`, `INPUT_SURFACE`, `PROJECTOR`, `DIAGNOSTIC`, **WHEN** each subscribe call returns, **THEN** only `INPUT_SURFACE` receives a submit handle; the handle stamps `issuer_peer` to the local seat regardless of arguments; `PRESENTATION` and `PROJECTOR` have no submit path; a subscribe attempted by a registered block owner is refused (sim never subscribes). (auto) [CR-10, CR-14 H4]
- **GIVEN** a `PRESENTATION` subscriber that calls a block mutator from its handler, and two other subscribers on the same block, **WHEN** P5 runs, **THEN** in debug an assertion fires; in release the write is refused, `WRITE_OUTSIDE_SCOPE` increments by 1, and both other subscribers still receive the batch. (auto) [CR-10, CR-14 H4]
- **GIVEN** the presentation source directories (per the code-root convention), **WHEN** a lint scans them for calls to block mutators, `WriteScope` acquisition, or `submit(` outside files registered as `INPUT_SURFACE`, **THEN** zero matches. (lint) [CR-14 H4]
- **GIVEN** a subscriber whose handler throws, **WHEN** P5 runs, **THEN** the remaining subscribers receive the batch, diagnostics logs the subscriber ID, and the kernel remains `RUNNING` with `T` advanced. (auto) [CR-10]
- **GIVEN** a `PERCEPTION_OUT` block owned by a fixture "sanity" system, **WHEN** its read view is requested by (i) the owner's own step hook, (ii) another type's `validate`, (iii) a `PRESENTATION` subscriber, **THEN** in debug (i) and (ii) fire an assertion naming the caller; in release (i) and (ii) receive an empty view (not `null`) and `PERCEPTION_READ_BLOCKED` increments by 2; (iii) receives the populated view in both builds. (auto) [CR-12, CR-14 H5]
- **GIVEN** a `PERCEPTION_OUT` block, **WHEN** its registration defaults are inspected and its contents are changed between two hash computations, **THEN** `persist == false` by default and `state_hash(PERSIST)` and `state_hash(ALL_SIM)` are both unchanged by the edit. (auto) [CR-12; D.12]
- **GIVEN** the kernel source and every `SIM` block's source, **WHEN** a lint scans for references to presentation scripts or signal emissions targeting them, **THEN** zero matches (state → notification → presentation is the only direction). (lint) [CR-10]

### Determinism

- **GIVEN** a `HOST` single-player kernel, **WHEN** 200 commands are submitted across 20 ticks in a known order, **THEN** the command log's dispatch order equals submission order and the `(queued_tick, host_order)` keys are strictly increasing; no two logged commands share a `host_order`. (auto) [CR-13 G1; D.7 `ordering_key`]
- **GIVEN** three commands keyed `(1000, 5817)`, `(1000, 5818)`, `(1001, 5816)` injected via a test ingress, **WHEN** P1 sorts them, **THEN** the dispatch order is exactly that sequence (lexicographic, `queued_tick` first). (auto) [D.7 `ordering_key`] *(adopted)*
- **GIVEN** any run of 1,000 ticks, **WHEN** the command log is checked, **THEN** every `Applied` command appears at exactly one `tick`, and every such `tick ≥ queued_tick + 1`. (auto) [CR-13 G2; D.4]
- **GIVEN** a kernel with only two registered blocks standing in for the Life-Support Graph and Attention — each satisfying the full CR-8 contract, each with a step hook at divisor 6 that mutates its block, and each drawing at least once per run from a named RNG stream — no `SceneTree`, and a fixed `world_seed` plus a recorded 1,000-tick command stream, **WHEN** two independent kernel instances each run `Kernel.step()` 1,000 times under `--headless`, **THEN** `state_hash(ALL_SIM, 1000)` is identical on both and the gdUnit4 process exits 0. (auto) [CR-13 G3, CR-14 H10; D.12 `state_hash`]
- **GIVEN** the same fixture, **WHEN** a snapshot is taken at `T = 500`, restored into a fresh kernel, and the remaining command stream is replayed to `T = 1,100`, while a control kernel plays uninterrupted to 1,100, **THEN** `state_hash(PERSIST, 1100)` is equal on both; on a mismatch the test output names the first differing `block_id` and key. (auto) [CR-13 G4, CR-8, CR-18; D.12]

G5 (exactly one result) and G6 (atomic ticks) are discharged by the criteria tagged `G5` under Commands & authority and `G6` under Tick & phases; they are not repeated here.

- **GIVEN** two blocks whose dictionaries hold the same key/value pairs inserted in different orders, one holding `-0.0` where the other holds `+0.0`, and two distinct NaN bit patterns, **WHEN** `state_hash` is computed, **THEN** the hashes are equal; changing one integer by 1 changes the hash. (auto) [D.12 `state_hash`]
- **GIVEN** a paused service cycle that committed twice at the same `T` *(adopted — decision log 2026-10-25)*, **WHEN** the command log is replayed, **THEN** both commits are reproduced in `host_order` and the post-replay `state_hash(PERSIST)` matches the original. (auto) [CR-16, G3]
- **GIVEN** the G3 fixture, **WHEN** one block is marked physics-coupled, **THEN** it is excluded from `ALL_SIM` unless recorded physics observations are supplied — pending Open Question 5. (auto) [CR-13 limit]

### Bounds & overload

- **GIVEN** `MAX_CMDS_PER_TICK = 128` and 300 commands pending at the P0 of `T = 1001`, all with `queued_tick = 1000` and contributed by both issuers (150 each, under the default `PER_ISSUER_CAPACITY = 256`), **WHEN** ticks 1001–1003 run, **THEN** 128, 128 and 44 commands dispatch respectively, in `ordering_key` order with no reordering across the boundary; the 129th command's `apply_tick == queued_tick + 2` and the 300th's `== 1003`. (auto) [CR-17; D.4 `apply_tick`]
- **GIVEN** `QUEUE_CAPACITY = 1024`, `MAX_CMDS_PER_TICK = 128`, and the test-only configuration `PER_ISSUER_CAPACITY = 512` with two issuers so the queue can actually be filled, **WHEN** diagnostics reports `queue_drain_ticks` and the full queue drains with no new submits, **THEN** the value is 8 and the queue is empty after exactly 8 ticks. (auto) [CR-17; D.8 `queue_drain_ticks`]
- **GIVEN** issuer `A` with 256 pending (`PER_ISSUER_CAPACITY`) and issuer `B` with 3, **WHEN** each submits once more, **THEN** `A` gets `Rejected(QUEUE_FULL)` and `B` is admitted; **WHEN** `A` submits a superseding intent, **THEN** it is admitted (the superseded one frees a slot before the capacity term). (auto) [CR-17; D.8 `admitted`]
- **GIVEN** coupled constraint 2 deliberately disabled by a test-only override, `QUEUE_CAPACITY = 300`, `PER_ISSUER_CAPACITY = 256`, issuer `A` at 256 pending and issuer `B` at 44, **WHEN** `B` submits and a step hook submits a `SYSTEM` command, **THEN** both get `Rejected(QUEUE_FULL)` — `B` despite being under its per-issuer cap, the hook via a result event at the next P5. This exercises the global-capacity branch, which is unreachable in any configuration that passes constraint 2 (see Gaps). (auto) [CR-17; D.8]
- **GIVEN** MVP defaults, **WHEN** boot runs, **THEN** the assertion `QUEUE_CAPACITY ≥ max_issuers × PER_ISSUER_CAPACITY + SYSTEM_RESERVE` (1024 ≥ 2 × 256 + 0) passes; **GIVEN** `QUEUE_CAPACITY = 500` with the same issuers, **THEN** `BOOTING → FAULTED`. (auto) [Section G coupled constraint 2]
- **GIVEN** `REMOTE_RATE_LIMIT = 120`, `RATE_WINDOW_TICKS = 60`, **WHEN** the local issuer submits 1,000 commands within 60 ticks, **THEN** none is `RATE_LIMITED` (`remote(i)` is false in MVP); **GIVEN** a test issuer flagged `remote`, **WHEN** it submits 121 within 60 ticks, **THEN** the 121st is `Rejected(RATE_LIMITED)`; **GIVEN** that remote issuer at 120 admitted in the current window, **WHEN** the kernel is `PAUSED`, 100 paused service cycles run and it submits once more, **THEN** that submit is still `Rejected(RATE_LIMITED)` — the tick-based window did not slide. (auto) [CR-5, CR-14 H8; D.8 `rate_limited`]
- **GIVEN** a command type registered with `max_age_ticks = 4` while `ceil(QUEUE_CAPACITY / MAX_CMDS_PER_TICK) = 8`, **WHEN** boot runs, **THEN** `BOOTING → FAULTED` naming the type; with every type ≥ 8, boot passes. (auto) [CR-17; D.7; Section G coupled constraint 1] *(adopted)*
- **GIVEN** the test-only configuration `QUEUE_CAPACITY = 1024`, `PER_ISSUER_CAPACITY = 512`, two issuers (passes coupled constraint 2: 1024 ≥ 2 × 512), and the queue filled to 1,024 commands of a type with `max_age_ticks = 30`, **WHEN** it drains over 8 ticks, **THEN** zero results are `STALE` — the local never-stale invariant. (auto) [CR-17; D.7 `is_stale`]
- **GIVEN** a test ingress that stamps `queued_tick = T − 40` on an intent with `max_age_ticks = 30`, **WHEN** it reaches P2, **THEN** `Rejected(STALE)`; with `queued_tick = T − 8`, it passes the age check. (auto) [D.7 `is_stale`]
- **GIVEN** a coalesced intent whose previous value `v0` was applied, **WHEN** a newer same-key intent is `Rejected(STALE)` via the test ingress, **THEN** the intent block still reads `v0`. (auto) [CR-17]
- **GIVEN** a command admitted at `T = 100` with age 0, **WHEN** `request_snapshot` is honoured at `T = 100`'s P6 and the kernel returns to `RUNNING`, **THEN** the command is dispatched at `T = 101` with age 1 — age did not advance during `SAVING`. (auto) [CR-17, CR-18]
- **GIVEN** the tick driver with `MAX_TICKS_PER_FRAME = 4` and `accumulator = 0.100 s`, **WHEN** one frame runs, **THEN** `ticks_run == 4`, `lag_ticks == 2`, the diagnostics lag counter reads 2 and the accumulator reads 0.0333 s (± 1 µs). (auto) [CR-17; D.5 `ticks_run_this_frame`, `lag_ticks`]
- **GIVEN** the driver after a 1.000 s frame, **WHEN** that frame and the following frames run at 16.7 ms each, **THEN** `lag_ticks` reads 56 after the first frame and decreases by 3 per subsequent frame until 0 (19 frames); in no frame does `ticks_run` exceed 4. (auto) [CR-17; D.5]
- **GIVEN** the tick is driven by Godot's physics callback (conditional on the tick-source ADR), **WHEN** `physics/common/max_physics_steps_per_frame ≠ MAX_TICKS_PER_FRAME`, **THEN** boot fails the coupled-constraint assertion naming both values. (auto, ADR-conditional) [D.5; Section G coupled constraint 4]
- **GIVEN** `STEP_BUDGET_MS = 4` and the diagnostics timer stubbed to report hook times 1.2, 0.9, 0.4, 0.3, 0.1 ms, **WHEN** the tick runs, **THEN** `step_over_budget == false` and nothing is logged; stubbed to 4.5 ms for one hook, **THEN** `step_over_budget == true`, the tick completes (every scheduled hook ran, `T` advanced), and the log lists hooks slowest-first. (auto) [CR-17; D.6 `step_over_budget`]
- **GIVEN** `COMMAND_LOG_CAPACITY = 1024` and 128 commands per tick for 20 ticks, **WHEN** the log is read, **THEN** it holds exactly 1,024 entries covering ticks 13–20 (worst-case coverage 8 ticks), each with command, result and `tick`; the oldest were evicted first. (auto) [CR-9, CR-17; D.13 `log_coverage_ticks`]
- **GIVEN** a test hook that sets `kernel.clock.T = 2^53`, `last_seq = 2^53` and `host_order = 2^53`, **WHEN** one tick runs and one command is admitted, **THEN** the values read `2^53 + 1` with no wrap, and the boot assertion on 64-bit width passes. (auto) [D.14 `ticks_to_overflow`; Section G coupled constraint 6]

- **GIVEN** `LAG_FORGIVE_TICKS = 30` and `lag_ticks = 56` after a 1.000 s frame, **WHEN** the next frame runs, **THEN** at most `MAX_TICKS_PER_FRAME` ticks run, `lag_ticks` reads 0 afterwards (the excess is dropped, not repaid), the diagnostics forgive counter increments by 1, and no block observes the dropped time. (auto) [CR-17; Tuning Knobs `LAG_FORGIVE_TICKS`]

### Developer velocity & diagnostics

- **GIVEN** a kernel with one existing domain system, **WHEN** a developer adds a new command verb (e.g. `toggle_beam`) and opens the PR, **THEN** the diff touches at most two files — one handler and one registration line in the owner's registration — and the reviewer ticks a "verb cost ≤ 2 files" checkbox; a third touched file is a blocking review finding to be raised with the kernel owner. (review) [Velocity rule]
- **GIVEN** the kernel source directory, **WHEN** a lint lists its files, **THEN** no file is a per-verb command class (no `*_command.gd` or `class_name *Command` per verb) and no domain command type is registered by kernel code: a boot with zero domain registrations reports zero registered command types. (lint, auto) [Velocity rule, CR-14 H2]
- **GIVEN** a booted kernel, **WHEN** its instance properties are enumerated by reflection, **THEN** the set is exactly `{clock, command_queue, registries, binding_tables, command_log, kernel_state, diagnostics}` (names per the implementation, one property per item). (auto) [CR-14 H1]
- **GIVEN** the kernel source directory in MVP, **WHEN** a lint scans for `MultiplayerAPI`, `MultiplayerPeer`, `ENet`, `@rpc`, `rpc(`, `WebSocket`, `PacketPeer`, **THEN** zero matches. (lint) [CR-14 H8]
- **GIVEN** two kernel instances constructed in one test process, **WHEN** each boots with its own blocks and runs 10 ticks, **THEN** their `T`, queues and logs are independent; a block object registered with both fails the second boot with `BLOCK_ALREADY_BOUND`; the suite exits 0, not 101. (auto) [CR-14 H10]
- **GIVEN** a `DIAGNOSTIC` subscriber, **WHEN** one tick runs, **THEN** it can read each of: `T`, kernel state, total and per-issuer queue depth, last-N command log with results and reasons, resolved hook order with per-hook timings and the over-budget flag, rejection counters by reason, lag counter, and `state_hash` of the committed tick — and each value changed or was refreshed for that tick. (auto) [CR-10, UI Requirements]
- **GIVEN** a release export of the project, **WHEN** its file list is inspected, **THEN** the Debug Overlay scene and the `DIAGNOSTIC` subscription entry point are absent. (manual; retained export file list) [UI Requirements]
- **GIVEN** a debug build with a frame-stamp overlay and input-event logging, **WHEN** a press is performed 100 times at 60 Hz tick / 60 fps render, **THEN** the 95th-percentile submit-to-first-changed-frame latency is ≤ 33.3 ms and the acknowledgement effect appears in the same frame as the press; results retained as a CSV in `production/qa/evidence/`. (manual; spike-conditional) [Game Feel; D.4 `input_to_effect_latency`]

### Performance budget

Measured by the diagnostics timer on the reference dev machine, provisional until the kernel spike confirms the numbers; a miss logs and is raised to the technical-director — it does not fail the suite until the spike has confirmed the figure.

- **GIVEN** 128 commands with no-op handlers, 32 registered blocks and 10 subscribers, **WHEN** one tick runs, **THEN** kernel overhead for P0–P6 excluding handler and hook bodies is ≤ 0.5 ms (P99 over 1,000 ticks). [CR-17; D.6]
- **GIVEN** 32 `persist` blocks including a Graph stand-in of 2,000 nodes × 6 fields stored as `Packed*Array`, **WHEN** a snapshot is captured, **THEN** `capture_time_ms ≤ 16.7` (P99 over 100 captures). [CR-18; D.11]
- **GIVEN** the default hook set on representative fixture data, **WHEN** 1,000 ticks run, **THEN** P3 wall time is ≤ 2.0 ms at P99 (the 4 ms `STEP_BUDGET_MS` is the alarm, not the target). [D.6]
- **GIVEN** the queue filled to `QUEUE_CAPACITY` with `MAX_PAYLOAD`-sized commands, **WHEN** process memory is sampled before and after, **THEN** the queue's footprint is ≤ 4.2 MiB. [D.8]

### Gaps / untestable as written

- **Formula count.** Section D names 23 formulas, not 21 (D.1 two, D.2 three, D.3 one, D.4 two, D.5 two, D.6 one, D.7 two, D.8 three, D.9 one, D.10 one, D.11 two, D.12 one, D.13 one, D.14 one). Every one of the 23 has a criterion above; the brief's "21" should be corrected in whichever document carries it.
- **`L_pixel` best case (D.4).** The "same-frame" path depends on `_physics_process`/`_process` ordering the engine reference is silent on; only the ≤ 33 ms worst-case bound is written as a (manual, spike-conditional) criterion. The best-case figure needs the ADR spike.
- **`H64` / `mix64` unchosen (D.10, D.12).** The GDD's example hash and RNG values are illustrative, so no golden-value test is possible yet; the criteria above are property-based (purity, range, stream independence). Golden values can be added once the ADR picks the functions — and must then be frozen as set-once.
- **`PRESENTATION` script obtaining an `INPUT_SURFACE` handle by reference passing.** The kernel cannot detect it; the only enforcement is the control manifest and lint (criterion written as a lint on `submit(` outside `INPUT_SURFACE` files). The control-manifest format does not yet exist.
- **"Only declared callers call declared mutators" (CR-9).** Same: control-manifest/lint obligation with no manifest format to lint against.
- **Commands queued before `request_pause` is honoured.** Decided in CR-16 (they wait, in order, for resume) and tested by the criterion at the end of Tick & phases.
- **Engine behaviour when physics steps exceed the cap (D.5).** Whether Godot discards or accumulates the excess is unsourced; the `lag_ticks` criteria assume the driver's own accumulator. If the ADR chooses the physics callback, those criteria must be re-expressed against the engine's behaviour.
- **`LAG_FORGIVE_TICKS`.** Adopted 2026-10-25 (see Tuning Knobs); criterion added under Bounds & overload.
- **Open Question 5 (G3 scope).** The physics-coupled exclusion criterion stands only until a record-and-replay decision; if recording is chosen, a criterion for observation capture/replay is needed.
- **Open Question 1 (subscriber-class reading of H4).** The `INPUT_SURFACE`/`PROJECTOR` criteria encode the GDD's provisional reading; a different TD ruling changes them.
- **Release-build reactions ("returns to the menu", "dev build halts").** The menu return is Save/Load/UI behaviour, not kernel-observable; only "no snapshot handed, state `FAULTED`" is tested.
- **Post-`restore` failure leaves state replaced.** CR-18's "pre-restore state intact where possible" is testable only for pre-check failures (criterion written); the `post_restore` failure path is by design not revertible and has no intact-state criterion.
- **D.11 size expectation (100–300 KiB).** No real blocks exist; the `snapshot_bytes` criterion checks the sum identity, not the magnitude.
- **Wall-clock figures.** All Performance-budget numbers are machine-dependent and spike-provisional, as stated in that block.
- **`CMD_ROUNDTRIPPED` counter.** The debug round-trip is only observable by instance identity; the criterion assumes a diagnostics counter the GDD does not name — the kernel owner should confirm or the criterion falls back to identity alone.
- **Determinism is "same build and platform" only (CR-13).** No cross-machine or cross-version hash equality is claimed or tested.
- **`MIRROR` behaviour.** Only "construction fails in MVP" is testable; the identical-API claim is VS-B scope.
- **Global `QUEUE_FULL` for an under-cap issuer is unreachable in MVP by design.** Coupled constraint 2 guarantees it; the criterion above disables the constraint to cover the code branch, not a live state. In VS-B (5 issuers × 256 > 1024) it becomes reachable and the criterion should be re-expressed against real configuration.
- **Headless determinism test (a) runs on stand-in blocks.** The Life-Support Graph and Attention GDDs do not exist yet; when they land, the test must be re-pointed at the real blocks and the fixture retired.

- **GIVEN** `RUNNING` with 20 non-`allowed_when_paused` commands queued, **WHEN** `request_pause` is honoured and five paused service cycles run, **THEN** none of the 20 is dispatched or rejected, queue depth is still 20, and all 20 dispatch in order on the first `RUNNING` tick after `request_resume` (they were queued before the pause, so there is no resume burst beyond what was already pending). (auto) [CR-16 — queued-before-pause rule]

## Open Questions

| # | Question | Owner | Resolve by |
|---|----------|-------|------------|
| 1 | **TD ruling on the subscriber-class reading of "presentation never writes"**: is `INPUT_SURFACE` (may submit commands as the local seat, may not mutate) and `PROJECTOR` (writes engine-side derived objects, never blocks) the intended interpretation of the control-manifest rule? | technical-director | Before `/create-architecture`; recorded in the authority-model ADR |
| 2 | **Tick source**: physics callback driving `step()` (recommended) vs custom accumulator; relationship to Godot's own physics-step cap and to `SceneTree.paused` | technical-director, engine-programmer | Authority-model / tick ADR, confirmed in the weld spike |
| 3 | **Who owns character position** — Ship Body (ship-local kinematic position) or Player Controller? Proposal: Body owns position, Controller owns intent and facing | game-designer, technical-director | Before the Player Controller and Ship Body GDDs (design order 3 and 13) |
| 4 | **Binding table ownership**: kernel-held (chosen) vs a Player Controller block | technical-director | Authority-model ADR |
| 5 | **G3 scope**: record-and-replay physics observations, or exclude physics-coupled blocks from the replay test? | technical-director, qa-lead | Before `/test-setup` |
| 6 | **Pause is single-player only** — confirm the Chart Table GDD does not assume "pick a target" happens while paused, and define VS-B's pause-as-overlay | game-designer, ux-designer | Before the Chart Table GDD (design order 22) |
| 7 | **Block base type and interchange form** (RefCounted vs Resource; JSON vs `var_to_bytes`) and the save-file trust model (a `.tres` can embed a script path — never load untrusted saves that way) | technical-director, lead-programmer | Save/Load GDD and its ADR |
| 8 | **Notification transport** (signals vs kernel-owned observer registry) and how read-only views are enforced in GDScript, which has no const references (debug copies vs frozen wrappers) | engine-programmer | ADR; measured in the spike |
| 9 | **Replication shape for data blocks** (VS-B): thin proxy nodes for MultiplayerSynchronizer vs custom change-record messaging; peer-ID assignment behaviour on 4.7.2 is NOT SOURCEABLE in the pinned reference and must be verified | network-programmer, technical-director | Before VS-B; co-op ADR |
| 10 | **Step-hook cross-write paths**: keep both (a) published unconditional mutator and (b) T+1 `SYSTEM` command, or collapse to one? The gameplay-programmer may want one | systems-designer, gameplay-programmer | After the first two consumer GDDs (Graph, Attention) show which they need |
| 11 | **Systems-index corrections** from this GDD: Sanity and Suit "keyed by peer ID" → PlayerId; add kernel edges for Interaction, Attention, Ship Section & Layout Data Model, Perception Filter; reconsider Graph → Save/Load | /map-systems (Phase 5d of this run) | This session |
| 13 | **Profile-scoped persistence** (Pillar 5): confirm `persist_scope: PROFILE` as the mechanism by which decoded rules, lore and scars survive `request_restore`, and that Save/Load owns a separate profile store; define what happens to `PROFILE` blocks on a *new game* vs a *new crew in the same world* | Save/Load owner, technical-director, game-designer | Save/Load GDD (design order 4) — before any Codex or Failure Contract design |
| 14 | **Commands aimed at phantoms** (P4(a) entities, P4(d) salvage) that have no `SIM` ID: what does an `INPUT_SURFACE` submit, and what does the result teach the player without becoming a free oracle? | Interaction, Sanity | Interaction GDD (design order 10) |
| 12 | **Exact `TICK_RATE`** (60 vs a 30 Hz fallback) is confirmed by the weld-on-moving-ship spike; every tick-expressed value is provisional until then | technical-director | End of spike phase |
