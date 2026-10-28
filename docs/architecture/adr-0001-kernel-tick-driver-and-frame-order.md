# ADR-0001: Kernel Tick Driver and Frame Order

## Status
Proposed

## Date
2026-10-28

## Last Verified
2026-10-28 (against `design/gdd/command-simulation-layer.md` CR-1a, D.5 and `docs/engine-reference/godot/` 4.7). Engine behaviours marked UNVERIFIED below have not been run on 4.7.2.

## Decision Makers
User (delegated all remaining design calls, 2026-10-28); Claude (author); reviewed by godot-specialist (engine: no blocking issues) and technical-director (TD-ADR: CONCERNS, amendments A1-A6, B7-B8 and C9-C13 applied in this revision). Cross-ADR amendments TD-04 (batch), TD-03, TD-10, TD-11, TD-14, TD-20, TD-22 and TD-27 applied 2026-10-28 (see Review Log).

## Summary
The simulation kernel needs exactly one clock, and three approved design documents still describe two different ones (the kernel's `_process`-driven `TickDriver`, and an older physics-callback tick). This ADR fixes the kernel's source: one `TickDriver` driven from the `_process` of a single always-processing host node, with a fixed per-frame order set by named process-priority constants, `tick_alpha` as the only value presentation reads from the clock, and the kernel's services handed to each module as a narrow injected facet rather than through an autoload.

## Engine Compatibility

| Field | Value |
|-------|-------|
| **Engine** | Godot 4.7.2 (stable), GDScript |
| **Domain** | Core (frame loop, node processing order), with an Input boundary |
| **Knowledge Risk** | MEDIUM. Every API used predates the 4.3 cutoff, but the pinned engine reference is silent on cross-node `process_priority` order and on how `Engine.time_scale` scales `_process(delta)`, and 4.7 changed input device IDs. |
| **References Consulted** | `docs/engine-reference/godot/VERSION.md`, `breaking-changes.md` (4.6 → 4.7 GDScript and Input rows), `deprecated-apis.md`, `current-best-practices.md`, `modules/input.md`, `modules/physics.md` |
| **Post-Cutoff APIs Used** | None. `Node._process`, `Node.process_priority`, `Node.PROCESS_MODE_ALWAYS`, `RefCounted`, `Engine.time_scale` and `roundi()` all exist in 4.3. |
| **Verification Required** | (1) Spike KRN-LAT-02: with the Input Adapter, the host node and a presentation node in one scene, the recorded `_process` call order is adapter → driver → presentation, on a dev build and on a release export. (2) `Engine.time_scale` scaling of `delta` (the kernel GDD marks it UNVERIFIED). (3) The first-frame `delta` after a long block (scene load, window drag, focus loss, a debugger breakpoint) is finite and handled by the cap and forgiveness, not a crash; the boundary service discards the first frame after a load (CR-1a). (3a) Delta jitter at `physics_jitter_fix` 0.5 versus 0, with `application/run/delta_smoothing` on and off, and the cost of the empty 60 Hz physics tick in an empty-scene baseline; the decision is to leave all three at engine defaults and record the measured effect on the 0-tick/2-tick frame pattern and on latency, not to claim they have nothing to do with the tick (MainTimerSync may adjust the idle step as well as the physics step, UNVERIFIED). (3b) `_input` and `_unhandled_input` are delivered before the frame's `_process` calls (UNVERIFIED in the reference), checked in the same scene as KRN-LAT-02. (4) `RenderingServer.frame_pre_draw` ordering (owned by ADR-0008; recorded here because the canvas write depends on it): **windowed 4.7.2 editor: all node `_process` calls, then `frame_pre_draw`, then `frame_post_draw` (probed); release export, D3D12 and Steam Deck are UNVERIFIED (pipeline spike step 12).** The probe settles the editor run only. (5) Test KRN-FRAME-END (Validation Criteria): `deliver_frame_end` is reached exactly once on every exit path of `TickDriver.frame`. (6) The boot walk KRN-05 accepts exactly the known-node set (registered INPUT_SURFACE objects, `KernelHost`, `SaveHost`) below `PRESENTATION`. |

## ADR Dependencies

| Field | Value |
|-------|-------|
| **Depends On** | None (first ADR). The design source is `design/gdd/command-simulation-layer.md` CR-1, CR-1a and D.5. |
| **Enables** | ADR-0002 (command pipeline), ADR-0003 (block registry and hooks), ADR-0004 (notification and `tick_alpha` consumers), ADR-0006 (Save/Load boundary service timing), ADR-0005 (determinism, keyed RNG and hashing: the tick source and the single wall-clock conversion), ADR-0008 (render pipeline: `present_time`, camera write), ADR-0009 (presentation subscriber contract), ADR-0019 (Input Adapter placement) |
| **Blocks** | Every epic that submits commands or draws from the clock: Player Controller, Pixel-Perfect Pipeline, Suit, Modular Ship Assembly (stories that quote the tick source), Interaction, Lighting. |
| **Ordering Note** | Do not **Accept** until the stale tick-source text is fixed in four GDDs and two architecture artifacts (see § GDD sync), spike KRN-LAT-02 has run (it now also checks `_input` delivery order and `time_scale`), and the TD-ADR amendments recorded below have been re-reviewed. If the spike fails, the sequential-parent fallback changes the Decision and the ADR needs a revision, not just a status change. Authoring now is safe because this ADR follows the kernel GDD (the owner), not the stale consumers. **2026-10-28 amendments (TD-04 and siblings):** this ADR now also carries the `deliver_frame_end` wrapper, `PresentationClock.state` and `load_seq`, and `ProcessPriority.HOST_SERVICE`; ADR-0004, ADR-0006, ADR-0008, ADR-0009 and ADR-0010 cite them, so those drafts are re-read against this text before any of them is Accepted. References to ADR-0010 (SIM-02 allow-list, KRN-03, KRN-05) are forward references to the lint owner, not a dependency: ADR-0010 depends on this ADR and lists it. |

## Context

### Problem Statement
The game simulates at a fixed 60 Hz in integer ticks, and everything else (render, input, audio, camera) runs at the render rate. Something has to turn engine frames into whole ticks, say when input is read relative to those ticks, and say when presentation reads the result. The kernel GDD answers this in CR-1a: a `TickDriver` with an injected delta, driven from `_process`, with the physics step "never the clock". Three consumer GDDs (Player Controller, Suit, Pixel-Perfect Pipeline) and Modular Ship Assembly still describe the physics callback as the tick source, with `physics_jitter_fix` and `max_physics_steps_per_frame` couplings. Two accumulators drifting against each other is the failure that fixed-tick determinism cannot survive, so the choice has to be recorded once, in a place stories and the control manifest will cite.

### Constraints
- **Determinism (kernel H9, CR-13).** The kernel never reads wall-clock time or `delta`. Ticks are whole, and the same save bytes plus the same command stream must produce the same state hash at any render rate.
- **Replay and test seams.** `Kernel.step()` is a plain synchronous method. Tests call it directly or inject `delta_us` into the driver; nothing in the kernel sleeps.
- **Main thread only (CR-1a).** The kernel, the driver and every owner callable run on the main thread.
- **No physics body on any simulation path (Ship Body BDY-3).** The avatar has none, so no sim code needs the physics callback.
- **Latency (D.4, KRN-LAT-01/02).** Submit-to-applied within one tick on an empty queue; `L_first` median ≤ 33.3 ms, p95 ≤ 50 ms on RM-DESKTOP.
- **Frame budgets (ruling R12).** P99 frame ≤ 16.6 ms on RM-DESKTOP (14.9 ms with headroom) and ≤ 33.3 ms on RM-DECK (30.0 ms), release export.
- **Coding standard.** Dependencies are injected, not reached through singletons, so headless tests can build their own kernel.
- **Engine reference silent** on cross-node `process_priority` order and on `Engine.time_scale` and `delta`.

### Requirements
- One tick source, and the physics callback is not it.
- A fixed, lint-checkable order inside one rendered frame: input sample, then the tick loop, then presentation.
- A catch-up cap and a forgiveness rule so a slow frame degrades to slow motion, not a spiral.
- A single float handed to presentation that places a drawn pose between two committed ticks.
- No global `Kernel` name that sim or presentation code can reach without being given it.

## Decision

### 1. The tick source is `_process`, in one host node

A single `KernelHost` node (`src/game/kernel_host.gd`; `extends Node`, `process_mode = PROCESS_MODE_ALWAYS`) owns a `TickDriver` (`RefCounted`, no scene tree) and calls it from its own `_process(delta)`. The host does one thing with the engine's `delta`: it converts it once, `delta_us = maxi(0, roundi(delta * 1_000_000.0))`, and passes the integer to the driver. That conversion and the diagnostics timer are the only wall-clock reads **that reach the simulation** (kernel H9). Which files may read clocks at all is the SIM-02 allow-list owned by ADR-0010 (`kernel_host.gd`, `monotonic_clock.gd`, `system_wall_clock.gd` and the diagnostics timer; TD-11, TD-14); the injected `MonotonicClock` and the Save/Load-only `WallClock` port are not kernel inputs. Presentation has exactly one `_process`: the single presentation host's (the Pipeline root, `PipelineRoot`; ADR-0008 § 6.1, ADR-0009), at `ProcessPriority.PRESENTATION`, which advances `present_time` and drives the presenter stages (camera smoothing and cosmetic accumulators live in those stages; Pipeline PR-9; Chart Table CT-43). Presenters define no `_process`. Presentation may read the committed tick `T` where the sim also computes the value (for example `beacon_aim(T)`, LO-27); none of that feeds back into the simulation. The driver is exactly the one specified in kernel CR-1a and D.5: an integer accumulator in units of microseconds × `TICK_RATE` (`TICK_UNITS` = 1,000,000 per tick), a boundary service before any tick, `ticks_run = min(floor(acc / TICK_UNITS), MAX_TICKS_PER_FRAME)`, a re-read of the kernel state after every `step()`, lag forgiveness (`LAG_FORGIVE_TICKS` = 0), result flush, and the paused service. This ADR does not restate those rules; the GDD is their authority. It records the *placement* and *order* decisions below.

**`TickDriver.frame()` is a two-statement wrapper** (TD-04; ADR-0004 § 5): `_frame_body(delta_us)` runs the CR-1a steps and may return early (a pause, a restore request, a fault, a zero-tick frame), and then `_kernel.deliver_frame_end()` runs unconditionally, with no other statement after the body. Because GDScript has no `finally`, the early returns live in `_frame_body` and the delivery lives outside it, so no exit path can skip frame-end delivery. The per-frame result is written into a reused `TickFrameResult` that callers read from `driver.last_result` (so `frame()` returns nothing and has no statement after the delivery). Test KRN-FRAME-END (Validation Criteria) proves it.

**The physics callback is not the clock and is not used by any simulation or presentation code in the MVP.** No script in `src/kernel/`, `src/sim/` or `src/presentation/` defines `_physics_process` or `_integrate_forces`; `physics_interpolation` stays off. The kernel does not read `physics_jitter_fix`, `physics_ticks_per_second` or `max_physics_steps_per_frame`, and nothing is coupled to them, **but they can shape the `delta` the host receives** (the engine's main-loop timer sync produces both, UNVERIFIED), so the spike logs delta jitter at `physics_jitter_fix` 0.5 and at 0. Determinism does not depend on it, because replays inject `delta_us`. A boot check records `physics_ticks_per_second != TICK_RATE` in the diagnostics header as information, not as a failure. A later Vertical Slice physics projection (ADR-0033) runs in `_physics_process` as a read-only PROJECTOR and never steps the kernel.

### 2. Frame order is fixed by named priority constants

Godot runs `_process` in ascending `process_priority`, ties broken by tree order. Tree order is not a contract we want to depend on, so the order is fixed by four constants in one file, `src/kernel/process_priority.gd`:

```gdscript
class_name ProcessPriority extends RefCounted

## Lower values run first inside one rendered frame.
const INPUT_ADAPTER: int = -100   # INPUT_SURFACE objects only; samples input, calls submit()
const TICK_DRIVER: int = -50      # KernelHost: boundary service, tick loop, forgiveness, result flush, frame-end delivery
const HOST_SERVICE: int = -40     # SaveHost (ADR-0006) only: per-frame poll after the driver; observes the last committed tick boundary
const PRESENTATION: int = 0       # engine default; the single presentation host (PipelineRoot), so a node that forgets to set it still runs after the driver
```

The order is therefore **input adapter → tick driver → host service (SaveHost) → presentation**, matching the kernel GDD's priority table (its ordinals 0, 1 and 2 are order only; the values above are the implementation; `HOST_SERVICE` is the one addition and sits between the driver and presentation). `HOST_SERVICE` is named and valued once, here, and is used by `SaveHost` only (ADR-0006, TD-04, TD-10); there is no `PRESENTATION_LATE` constant (TD-03: the single presentation host removes the need). Choosing `PRESENTATION` as the engine default is deliberate: an unset node lands after the driver, which is the safe failure.

Rules, each of which becomes a lint (ADR-0010):
- A node whose `process_priority` is below `TICK_DRIVER` must be a registered INPUT_SURFACE object. Nothing else may run before the driver. Scene files (`.tscn`) must not set `process_priority` at all (the Inspector stores a literal); code sets it from `ProcessPriority`. A regex cannot see priorities set at runtime, so the authoritative check is a **boot-time tree walk** (ADR-0010 KRN-05): every node whose priority is below `TICK_DRIVER` is a registered INPUT_SURFACE object, only `KernelHost` uses `TICK_DRIVER`'s value, only `SaveHost` uses `HOST_SERVICE`'s value (the known-node set below `PRESENTATION` is the registered INPUT_SURFACE objects, `KernelHost` and `SaveHost`), and every INPUT_SURFACE, `KernelHost`, `SaveHost` and presentation node resolves to the main-thread `process_thread_group`, because a sub-thread group leaves the main-thread ordering and breaks the main-thread rule. `process_thread_group` defaults to `INHERIT`, so the walk resolves each node's **effective** group by following `INHERIT` up through its ancestors to the first explicit value (the root counts as `MAIN_THREAD`); reading the node's own property alone would pass an `INHERIT` node under a `SUB_THREAD` parent. *(Amended 2026-10-28 — architecture review.)* Exactly which rules the lint tool can enforce statically and which move to boot walks is ADR-0010's call (OQ-16).
- No node other than `KernelHost` may use `TICK_DRIVER`'s value, and no node other than `SaveHost` may use `HOST_SERVICE`'s value.
- A presentation node's `process_priority` is `PRESENTATION` or greater and is never set to a literal; it references the constant. The presentation host is the one `_process` in presentation (ADR-0008 § 6.1, ADR-0009); layer 3 and 4 UI Controls and registered INPUT_SURFACE objects keep the exemptions ADR-0009 lists.
- Code in `src/sim/` and `src/presentation/` does not `await get_tree().process_frame` (that signal fires outside the per-node ordering, UNVERIFIED); waiting is expressed in ticks or in `present_time`.

### 3. The Input Adapter samples and submits in its own `_process`

The Adapter (an INPUT_SURFACE, Player Controller GDD PC-1(a)) reads device state and calls `submit()` from its `_process` at `ProcessPriority.INPUT_ADAPTER`. It may latch raw events in `_input`/`_unhandled_input`, which the engine delivers before the frame's `_process` calls, but it submits only from `_process`. Its commands are therefore queued before the driver's boundary service and tick loop run, so an intent sampled in frame N sits in tick T's P1 batch, is applied in P2 and is integrated in P3 of the same tick: one tick of latency, never two. The Adapter never samples in `_physics_process`. Devices are identified by event type, not by `device == 0` (4.7 changed mouse and keyboard device IDs; ADR-0019).

In a frame that runs several ticks (catch-up), the Adapter has submitted once, so only the first tick of that frame sees new commands. Continuing movement survives because intent state lives in `controller.state` and commands are edge- or change-based, not per-tick.

### 4. `tick_alpha` is the only value presentation reads *from the driver*

At the end of each driver frame the driver publishes `tick_alpha = clampf(float(acc) / float(TICK_UNITS), 0.0, 1.0)` (identical to kernel D.3's `t_since_last_commit / tick_duration`; the casts are required, because `acc` and `TICK_UNITS` are both `int` and an unmarked `/` is integer division in GDScript, which would give 0 on every normal frame and fail the `integer_division` warning policy). It is a presentation-only float: it is not a state block, is never hashed or saved, and the simulation never reads it (kernel H9). Presentation reads it once per frame, through `PresentationClock`, in the single presentation host's `_process` (the driver stages fill `PresentFrame`; ADR-0009) and interpolates previous-to-current committed state with `pos = floor(prev + (curr - prev) * tick_alpha)` (Pipeline Section D, interpolation formula; restore case in its Edge Cases). Because forgiveness at `LAG_FORGIVE_TICKS` = 0 drops whole lag ticks at the end of every frame but keeps the sub-tick remainder, `acc < TICK_UNITS` after the driver runs in `RUNNING`, so `tick_alpha` is continuous and in [0, 1). **Two rules from kernel D.3 are binding and are restated here because they are easy to lose:** (1) **while the kernel is not `RUNNING` the driver holds `tick_alpha` at 1.0**, so a paused, loading or saving game draws the latest committed pose and never a frozen partway-back pose; (2) **on every transition into `RUNNING` the kernel calls `snap_baseline` on every `interpolated` block** (ADR-0003), so `previous == current` on resume and the first running frame, including a 0-tick frame, cannot interpolate across a discontinuity. **This kernel call is the only restore-continuity mechanism** (TD-22): ADR-0008 has no `previous := current` rule of its own in `post_restore`, and the camera's `DISCONTINUITY_RESTORE` CUT fires on a change of `PresentationClock.load_seq` (§ 5).

Presentation also keeps a render-rate cosmetic clock, `present_time`, owned by the Pipeline root (Pipeline PR-9, ADR-0008). The rule recorded here: **cosmetic phase (flicker, gutter, fades) uses `present_time`; anything the simulation also computes (beacon aim, poses, wane) uses the kernel tick; the tick counter is never used for cosmetic phase** (Lighting LO-4).

### 5. The kernel reaches modules by injection

There is no `Kernel` autoload and no static singleton for kernel or game state. The main scene's root script (the **composition root**, `src/game/game.gd`) constructs the kernel, the `TickDriver` and the `KernelHost`, then gives every module and subscriber **the narrowest facet it needs through `setup()`**, never the whole `Kernel`: a simulation module receives the registration and mutator-scope services ADR-0003 defines; a presentation subscriber receives `subscribe(...)`, the read views and a read-only **`PresentationClock`** (`tick_alpha`, `T`, `running`, `state`, `load_seq`, backed by the driver and kernel); the composition root issues the **`NotifyFacet`**, binding the subscriber class at issue (ADR-0004 § 2), so a holder cannot subscribe as a class it was not issued; and the right to submit or to call the control plane comes only from the named handles ADR-0002 defines, which the root issues to a few objects. The kernel-state need of presentation (ADR-0008) is met by `PresentationClock.state`, not by a separate `KernelStateView` type. **Restore marker:** `PresentationClock.load_seq` is incremented by the kernel on every `LOADING → RUNNING` transition (a restore and a new game), immediately before `deliver_all` (ADR-0004), and never on `PAUSED → RUNNING` or `SAVING → RUNNING`; presentation compares it with the last value it saw to detect a discontinuity (ADR-0008 camera CUT, ADR-0009 `on_discontinuity(RESTORE)`). `ChangeBatch` carries no load flag by design (ADR-0004 § 6). Handing the whole kernel to presentation would give it `submit()`, which kernel CR-10 limits to INPUT_SURFACE. Headless tests construct their own kernel, driver and facets and need no scene. In the code examples in `docs/architecture/architecture.md`, `Kernel.` reads as "the injected facet". There are no recorded exceptions.

### 6. Pause, load and the tree

The kernel's `PAUSED` state is the only pause the simulation knows (kernel CR-16). `KernelHost` is `PROCESS_MODE_ALWAYS`, so the driver keeps running its boundary service whether or not anything pauses the scene tree, and no simulation code reads `SceneTree.paused`. Whether the UI layers use `SceneTree.paused` or per-layer `process_mode` is **not decided here**: the Chart Table GDD routes it to its own ADR candidate (a) and the Debug Overlay GDD (E-DO-48, AC-DO-174) assumes `SceneTree.paused` can be true; ADR-0030 decides it, and it must keep `KernelHost`, `SaveHost` (ADR-0006) and the Input Adapter `PROCESS_MODE_ALWAYS`. Window focus loss, an OS-level stall, a debugger break or a long load simply produces one large `delta`: the cap (`MAX_TICKS_PER_FRAME` = 4) and forgiveness absorb it, and the accumulator is cleared on every transition into `RUNNING`, so loads, saves and pauses never cause catch-up bursts afterwards (kernel CR-1a).

### Architecture Diagram

```
Engine main loop, one rendered frame (_process in ascending process_priority)
│
├─ -100  Input Adapter           [INPUT_SURFACE]   reads input → surface.submit(...)  (SurfaceHandle, ADR-0002)           (queued only)
│
├─  -50  KernelHost._process(delta)     (src/game/kernel_host.gd)
│          delta_us = maxi(0, roundi(delta × 1e6))                  ← the only wall-clock read that reaches the sim
│          TickDriver.frame(delta_us) = two statements:
│           A _frame_body(delta_us)   (may return early: pause, restore request, fault, zero ticks)
│               1 boundary service (SAVING/LOADING/pause/resume; may clear acc, discard this frame's delta;
│                 on LOADING → RUNNING: load_seq += 1, then deliver_all; snap_baseline on entry to RUNNING)
│               2 repeat ticks_run times, re-reading kernel state after each:
│                    Kernel.step() = P0 begin · P1 intake · P2 dispatch · P3 step hooks · P4 commit · P5 notify · P6 boundary
│               3 forgiveness (drop whole lag ticks, keep the sub-tick remainder)
│               4 flush the result buffer
│               5 if PAUSED: paused service
│           B Kernel.deliver_frame_end()      (unconditional; nothing after it; KRN-FRAME-END)
│          tick_alpha = clampf(float(acc) / float(TICK_UNITS), 0, 1)   (1.0 while not RUNNING)
│
├─  -40  SaveHost._process               [HOST_SERVICE]    SaveService.poll_frame(); observes the last committed tick boundary (ADR-0006)
│
└─   0   PresentationHost = PipelineRoot [PRESENTATION]    ONE presentation _process (ADR-0008 § 6.1, ADR-0009):
           advance present_time → zoom cooldown → RESCALING re-derive if dirty → presentation driver.frame(delta, present_time)
           runs the presenter stages from committed views at tick_alpha (camera solve is stage 1; presenters define no _process);
           frame_pre_draw handler then only writes canvas_transform / parallax from the solved pose (ADR-0008)

_physics_process: not used by the kernel, simulation or presentation in the MVP.
```

### Key Interfaces

```gdscript
class_name TickDriver extends RefCounted
## Turns an injected wall-time delta into whole kernel ticks. No scene tree, no engine clock.
func _init(kernel: Kernel, params: TickParams) -> void
func frame(delta_us: int) -> void              # exactly two statements: _frame_body(delta_us) then _kernel.deliver_frame_end(); nothing after the body
func _frame_body(delta_us: int) -> void        # boundary service, tick loop, forgiveness, flush, paused service; may return early (pause, restore request, fault)
var last_result: TickFrameResult:              # reused object filled by _frame_body: ticks_run, dropped, tick_alpha
	get: return _result
var tick_alpha: float:                         # presentation-only; read once per frame
	get: return _tick_alpha

class_name KernelHost extends Node             # src/game/kernel_host.gd
func setup(driver: TickDriver) -> void         # called by the composition root; sets process_mode and process_priority = ProcessPriority.TICK_DRIVER
func _process(delta: float) -> void            # returns at once if setup() has not run (null driver); else delta_us = maxi(0, roundi(delta * 1_000_000.0)); driver.frame(delta_us)

class_name ProcessPriority extends RefCounted  # INPUT_ADAPTER -100, TICK_DRIVER -50, HOST_SERVICE -40, PRESENTATION 0 (Decision § 2)

# Issued by the composition root; the subscriber class is bound at issue (ADR-0004 § 2)
class_name NotifyFacet extends RefCounted      # subscribe only; signature and class binding owned by ADR-0004

# Every module and subscriber:
func setup(facet: ModuleFacet) -> void         # the narrowest facet for the module's class (ADR-0003 / ADR-0009); stored; never looked up through a global

class_name PresentationClock extends RefCounted  # read-only, backed by the TickDriver and the kernel
var tick_alpha: float:
	get: return _driver.tick_alpha                 # 1.0 while the kernel is not RUNNING
var T: int:
	get: return _kernel_clock_t()                  # last committed tick
var running: bool:
	get: return _kernel_running()
var state: int:
	get: return _kernel_state()                    # the kernel state enum value (as defined by the kernel GDD); replaces any KernelStateView type
var load_seq: int:
	get: return _kernel_load_seq()                 # +1 by the kernel on every LOADING -> RUNNING (restore and new game), immediately before deliver_all; never on PAUSED/SAVING -> RUNNING
```

### Implementation Guidelines
- **Must** drive `Kernel.step()` only from `TickDriver.frame()`, and call `TickDriver.frame()` only from `KernelHost._process`. Tests may call either directly.
- **Must** keep `TickDriver.frame()` to two statements, `_frame_body(...)` then `_kernel.deliver_frame_end()`; **must never** add a statement after the delivery or an early `return` in `frame()` itself.
- **Must** increment `load_seq` only on `LOADING → RUNNING`, once per transition, immediately before `deliver_all`; **must never** increment it on `PAUSED → RUNNING` or `SAVING → RUNNING`, and **must never** add a load flag to `ChangeBatch`.
- **Must** use `ProcessPriority.HOST_SERVICE` only in `SaveHost`.
- **Must never** define `_physics_process` or `_integrate_forces` in `src/kernel/`, `src/sim/` or `src/presentation/`.
- **Must** use the `ProcessPriority` constants; **must never** write a literal `process_priority` value.
- **Must** submit input only from the Input Adapter's `_process`; **must never** call `submit()` from `_input`, `_physics_process` or a signal handler.
- **Must** pass each module its facet with `setup()`; **must never** add an autoload, a static variable or a group lookup that exposes the kernel, a block or a read view, and **must never** give a presentation node the whole `Kernel`.
- **Must** read `tick_alpha` through `PresentationClock` in presentation only, never in a simulation callable, and **must never** use the tick counter for cosmetic animation phase.
- **Must** keep `Engine.time_scale = 1.0` in release builds; a boot check records `TIME_SCALE_NON_UNIT` in diagnostics otherwise. A boot assertion also requires `application/run/low_processor_mode` to be false (in low-processor mode the engine idles between events, the simulation would stall and the next `delta` would spike), and the diagnostics header records `Engine.max_fps`, the vsync mode and `physics_ticks_per_second`.
- **Must** keep the `delta_us` conversion in `KernelHost` and nowhere else. The forbidden clock and RNG token list and its allow-list are owned by ADR-0010 `SIM-02` (allow-list: `kernel_host.gd`, `monotonic_clock.gd`, `system_wall_clock.gd` and the diagnostics timer; TD-11, TD-14); this ADR keeps no list of its own. *(Amended 2026-10-28 — architecture review; list replaced by the pointer under TD-04/TD-14.)*
- **Should** test the driver with injected `delta_us` sequences (16,000 / 16,667 / 17,334 µs patterns, 100 ms hitches, zero-length frames) and the frame order with a recording scene (KRN-LAT-01, KRN-LAT-02).

## Alternatives Considered

### Alternative 1: Drive the tick from the physics callback
- **Description**: `Kernel.step()` runs in `_physics_process` at `physics_ticks_per_second` = 60, with `max_physics_steps_per_frame` set equal to `MAX_TICKS_PER_FRAME` and `physics_jitter_fix` set to 0. The Adapter samples input in the same callback. This is what the Player Controller, Suit, Pipeline and Assembly GDDs still describe.
- **Pros**: The engine already owns an accumulator and catch-up; Godot Physics 2D steps align with the tick if a physics projection is added later.
- **Cons**: Two accumulators (the engine's and the kernel's D.5) must agree, and `physics_jitter_fix` smooths the engine's delta, which silently changes tick spacing unless forced to 0. The coupling to `max_physics_steps_per_frame` is an engine setting a mis-edit can break without any test noticing. Input is sampled in a callback that may run zero or two times per rendered frame, making latency irregular. The reference is silent on the 4.7 behaviour of all three settings. The MVP has no physics body, so there is nothing for the physics step to align with.
- **Rejection Reason**: The kernel GDD rejected it in CR-1a for these reasons, and the kernel is the owner. It adds an unverified engine dependency to a layer whose whole point is that nothing outside it can change tick behaviour.

### Alternative 2: Own accumulator from `Time.get_ticks_usec()`
- **Description**: The driver reads the wall clock directly each frame and accumulates real elapsed microseconds, ignoring the engine's `delta`.
- **Pros**: Exact integer microseconds, no float conversion; unaffected by `Engine.time_scale`.
- **Cons**: A second wall-clock read, which breaks the "one conversion, one diagnostics timer" rule (H9) and makes the clock lint much harder; bypasses the engine's own frame pacing (a debugger break or breakpoint would produce a huge catch-up burst immediately); and removes the dev-build ability to scale simulated speed through `time_scale`.
- **Rejection Reason**: The float conversion of `delta` is exact enough (a double carries the full range of a frame to well under a microsecond) and keeps a single, injectable time input. Held as the fallback if the spike finds `delta` unusable.

### Alternative 3: A dedicated simulation thread
- **Description**: The kernel ticks on its own thread at a fixed rate and presentation reads snapshots.
- **Pros**: Simulation cost cannot stall rendering.
- **Cons**: Violates the main-thread rule (CR-1a), makes owner callables and read views thread-unsafe, and would need synchronisation across every block. The kernel budget (P3 ≤ 2.0 ms) does not need it.
- **Rejection Reason**: Complexity and determinism risk with no budget case for it.

### Alternative 4: Autoload `Kernel` singleton (for access)
- **Description**: Register the kernel as an autoload so any script can call `Kernel.submit(...)` or `Kernel.tick_alpha`.
- **Pros**: Least typing; the examples in the architecture document already read this way.
- **Cons**: Every module gains a hidden dependency; headless tests must reset a global between cases; two kernels (replay comparison, MIRROR in VS-B) cannot coexist in one process; contradicts the coding standard.
- **Rejection Reason**: Injection costs one `setup()` call per module and keeps the kernel constructible by tests and, later, a second instance for MIRROR.

## Consequences

### Positive
- One clock, one place it is read, and one stated order inside a frame; the tick-source disagreement across four GDDs is resolved in favour of the owner.
- Tests run the kernel and driver with no scene tree and an injected `delta_us`, so replay-at-any-frame-rate (TR-pxl-022) is a cheap, deterministic test.
- The sampled-input latency is bounded (one tick from submit to applied) and does not depend on how many physics steps happened to run, unlike a physics-callback sample. Frame-to-frame it is not perfectly regular: at 60 Hz render with vsync, rounding and jitter produce occasional 0-tick frames followed by 2-tick frames, so end-to-end `L_first` varies by up to a frame (the D.4 and KRN-LAT-01 bounds already account for it).
- Defaults are safe failures: an unset presentation priority runs after the driver; an unset physics setting is irrelevant.
- Injection lets a second kernel (MIRROR, replay comparison) exist in one process.

### Negative
- The order rests on `process_priority` behaving as documented across unrelated nodes, which the pinned reference does not state (spike KRN-LAT-02). If it does not, the fallback is a single parent node that calls the Adapter, driver and presentation in explicit sequence.
- Every module needs a `setup()` call with the right facet and the composition root grows with the module count; forgetting one fails at first use, so a boot check lists modules that were never given a facet.
- Float conversion of `delta` adds an unbiased rounding error of at most 0.5 µs per frame, which enters the integer accumulator (about 30 µs per second at 60 fps, negligible against a 16,667 µs tick); replay tests must inject `delta_us` rather than run real frames.
- A frame that runs several ticks submits input once, so only its first tick sees new commands; this is correct for change-based commands but must be remembered when writing tick-level tests.

## Risks
- **`process_priority` order across nodes behaves differently than assumed.** Mitigation: KRN-LAT-02 runs before acceptance, on dev and release builds; the sequential-parent fallback is described above.
- **`Engine.time_scale` scales `delta` differently in 4.7.2.** Mitigation: pinned to 1.0 in release, flagged `TIME_SCALE_NON_UNIT` otherwise; verified in the spike.
- **A consumer GDD's stale text is implemented instead of this ADR.** Mitigation: the four GDDs are fixed before acceptance (§ GDD sync) and a lint rejects `_physics_process` in `src/kernel`, `src/sim` and `src/presentation`.
- **Large `delta` after an OS stall produces one slow-motion burst.** Mitigation: cap and forgiveness bound it to four ticks, and the accumulator is cleared on every transition into `RUNNING`.
- **The Pipeline's canvas write in `frame_pre_draw` may precede some `_process` calls on a build other than the one probed.** The windowed 4.7.2 editor probe shows all node `_process`, then `frame_pre_draw`, then `frame_post_draw`; release export, D3D12 and Steam Deck are UNVERIFIED (pipeline spike step 12). Mitigation: owned by ADR-0008; the camera is solved inside the single presentation `_process` (stage CAMERA_SOLVE), so `frame_pre_draw` only writes `canvas_transform` and the parallax offsets from the solved pose; if the release-export spike fails, ADR-0008 moves that write to the end of `PipelineRoot._process`, after the presentation driver's `frame`. The render thread model is pinned by ADR-0008 § 5.1 (`rendering/driver/threads/thread_model` held at default 1, checked by effective value).
- **A frame exit path skips frame-end delivery.** Mitigation: `frame()` is a two-statement wrapper and KRN-FRAME-END covers a pause, a fault, a restore request and a zero-tick frame.

## GDD Requirements Addressed

| GDD System | Requirement | How This ADR Addresses It |
|------------|-------------|--------------------------|
| command-simulation-layer.md | CR-1, CR-1a: fixed 60 Hz tick; `TickDriver` with injected delta driven from `_process` by one always-processing node; physics step is not the clock (TR-cmd-001, TR-cmd-002) | Decision § 1 adopts it verbatim and names the host node and the one `delta` conversion |
| command-simulation-layer.md | CR-1a priority table: input adapter, then driver, then presentation, with a lint on `process_priority` (TR-cmd-004) | Decision § 2 fixes the constants and the three lint rules |
| command-simulation-layer.md | D.5: lag dropped at end of frame, accumulator cleared on every transition into `RUNNING` (TR-cmd-003) | Decision § 1 and § 6 reference it; the `tick_alpha` derivation in § 4 depends on its remainder rule |
| command-simulation-layer.md | CR-1a main thread only; no `Thread`, `WorkerThreadPool` or `call_deferred` into kernel code (TR-cmd-005) | Implementation Guidelines; Alternative 3 rejected |
| command-simulation-layer.md | CR-2: per-tick phases P0–P6 (TR-cmd-006) | Architecture Diagram shows where the phases sit in the frame |
| player-controller-input.md | PC-1(a), PC-15: the Adapter samples and submits before the kernel steps; one tick latency (TR-pc-004) | Decision § 3: Adapter samples in its own `_process` at `INPUT_ADAPTER`; the GDD's "physics callback" wording is stale (§ GDD sync) |
| pixel-perfect-render-pipeline-camera.md | PR-9 (`present_time`), the interpolation formula in Section D, and the tick-driver routing at L227 / L297 / Open Question 10: sim stays 60 Hz at any render rate; tick-before-render ordering; tick-driver ownership (TR-pxl-021, TR-pxl-022) | Decision § 1, § 4 and § 6; the `physics_jitter_fix` / `max_physics_steps_per_frame` clauses are removed by the GDD fix |
| 2d-lighting-and-occlusion.md | LO-4: kernel clock for what the sim also computes, `present_time` for cosmetic phase (TR-light-008) | Decision § 4 records the rule |
| hud-and-suit-layer.md, chart-table-web-map-and-menus.md | Presentation reads committed state at render rate and never causes a tick (TR-hud-010, TR-charttable-005) | Decision § 4 and the Architecture Diagram |
| derelict-authoring-handmade-kits.md | P3 budget wording tied to the frame step (TR-derelict-021) | The tick loop is the one place P3 cost is incurred; `STEP_BUDGET_MS` semantics are already settled by the owner (a 4 ms alarm threshold; the P3 target is 2.0 ms P99) and ADR-0003 records them, so IC-4 in the architecture document moves to ADR-0003 |

## Performance Implications
- **CPU**: The driver itself is an integer add, a divide and a loop; it is negligible against the kernel's budgets (P3 ≤ 2.0 ms P99; frame simulation ≤ 4.0 ms for one tick and ≤ 6.0 ms for four), which are measured on a release export on RM-DESKTOP and RM-DECK. At 30 fps (RM-DECK) a frame normally runs two ticks; the 4-tick cap means slow motion begins below 15 fps.
- **Memory**: One `TickDriver`, one `KernelHost` node, a few integers. No per-frame allocation in the driver (`TickFrameResult` is reused).
- **Load Time**: None. The accumulator is cleared when load finishes, so load time is not repaid as ticks.
- **Network**: None at MVP. In VS-B a MIRROR kernel uses the same driver with no authority over commands (ADR-0002, ADR-0032).

## Migration Plan
There is no code yet. The change is to documents: fix the stale tick-source text in the Player Controller, Suit, Pixel-Perfect Pipeline and Modular Ship Assembly GDDs (list below), and the two stale requirement rows and the architecture document's Data Flow 1, OQ-1 and OQ-13 text, before this ADR is Accepted. The architecture document's `Kernel.` global in examples is replaced by the injected facets when it is next revised.

## Validation Criteria
- KRN-LAT-01 passes: with an injected driver fed 16,000 / 16,667 / 17,334 µs frames, `L_first` never exceeds `2 × frame_duration + L_present`.
- KRN-LAT-02 passes on a dev build and on a release export: the recorded call order is adapter → driver → `SaveHost` (HOST_SERVICE) → presentation host, and the lint and the KRN-05 boot walk find no presentation node at or below `TICK_DRIVER` and no node other than the known set (INPUT_SURFACE objects, `KernelHost`, `SaveHost`) below `PRESENTATION`.
- A scripted 10-second replay produces an identical `state_hash` at `max_fps` 30, 60, 144 and 0 (TR-pxl-022) with no `physics_*` settings touched.
- A grep of `src/kernel`, `src/sim` and `src/presentation` finds no `_physics_process` and no read of `SceneTree.paused` in `src/kernel` or `src/sim`; no `.tscn` sets `process_priority`; no script sets it to a literal; and `project.godot`'s `[autoload]` section contains no kernel, block or read-view entry.
- Unit and integration tests inject `delta_us` into `TickDriver` or call `KernelHost._process(delta)` directly; they never rely on real frame pacing (a headless run has no vsync, so frames are as fast as possible, and `--quit-after N` counts iterations, not seconds). Before KRN-LAT-02 is written as a gdUnit4 scene test, confirm that gdUnit4's scene runner calls `_process` on the host (UNVERIFIED).
- **KRN-FRAME-END** passes: with a recording kernel, `deliver_frame_end` is reached exactly once in each of five frames: a pause requested at tick 3 of 4, a fault at tick 2, a restore request, a zero-tick frame, and an ordinary four-tick frame (control).
- **KRN-LOAD-SEQ** passes: `PresentationClock.load_seq` increases by exactly 1 on `LOADING → RUNNING` for a restore and for a new game, increases before the `deliver_all` batches are delivered, and is unchanged across `PAUSED → RUNNING` and `SAVING → RUNNING`.
- A 100 ms hitch produces four ticks, drops the rest and leaves `acc` below one tick; a load, save or pause does not produce catch-up ticks after returning to `RUNNING`.

## GDD sync (stale tick-source text)
**Status 2026-10-28: the consumer GDD passages below are corrected** (player-controller-input, suit-and-equipment, pixel-perfect-render-pipeline-camera, modular-ship-assembly); `tr-baseline.md` and `architecture.md` were corrected with this ADR. Acceptance still waits on spike KRN-LAT-02 and the TD-ADR re-review. The list records what was stale:
The kernel GDD (owner) is correct. These consumer passages still describe the physics callback and conflict with this ADR:
- `design/gdd/player-controller-input.md` L62 (PC-1(a)), L740 and L751 (latency terms), L1021 (Immediacy), L1077 (AC-PCI-003: "in the same physics callback")
- `design/gdd/suit-and-equipment.md` L1441
- `docs/architecture/tr-baseline.md` TR-pc-004 and TR-pxl-022 (stale requirement text that stories would embed; corrected in the same pass as this ADR)
- `docs/architecture/architecture.md` § Data Flow 1 (priority values and the boundary-service-before-add order), OQ-1, OQ-13 and the paragraph that defers kernel access to ADR-0001 (corrected in the same pass)
- `design/gdd/pixel-perfect-render-pipeline-camera.md` L227, L297, L1219 (`physics_jitter_fix`, `max_physics_steps_per_frame`, "two accumulators")
- `design/gdd/modular-ship-assembly.md` L1058, L1161, L1528, L1593, L1772, L1940 (the "when the tick is driven by the physics callback" conditions, the boot assertion, and open item 41)

**Added 2026-10-28 (cross-ADR amendments, for the GDD owners; this ADR does not edit them):**
- `design/gdd/command-simulation-layer.md` CR-1a priority table (L60): add the `HOST_SERVICE` slot between the driver and presentation (SaveHost), the `deliver_frame_end` last step of the driver frame, and the "that conversion plus the diagnostics timer are the only wall-clock reads in the project (H9)" clause, which now reads "reach the simulation" with the allow-list owned by ADR-0010 SIM-02; KRN-LAT-02 (L1088) adds `SaveHost` to the recorded order.
- `docs/architecture/architecture.md` Data Flow 1 (around L257-279): priority list gains `HOST_SERVICE (-40) SaveHost`, and the presentation half is redrawn to the single presentation host's stage list (TD-03, TD-22; shared with ADR-0008 and ADR-0009).

## Related
- `design/gdd/command-simulation-layer.md` (CR-1, CR-1a, CR-2, D.3, D.4, D.5; KRN-LAT-01..03; open question 2 closed by CR-1a)
- `docs/architecture/architecture.md` (§ Data Flow 1, OQ-1, OQ-13)
- ADR-0002, ADR-0003, ADR-0004, ADR-0005, ADR-0006, ADR-0008, ADR-0009, ADR-0010, ADR-0019 (to be written)
- Review rulings: delegated ruling G1 (owner wins)

## Review Log

### 2026-10-28 — cross-ADR amendments (autonomous mode, action "Revise flagged items")
Status stays **Proposed**. Only the amendments assigned to this ADR by the adjudicator were applied; the Decision's tick-source, accumulator and `tick_alpha` rules are unchanged. No earlier "review unavailable" text existed in this file.

| Ruling / finding id | Outcome in this ADR |
|---|---|
| TD-04 [R4, R14(ADR-0001), ADR-0004/X1, ADR-0004/R14(ADR-0001), ADR-0004/N2, ADR-0006/X3, ADR-0008/R4, ADR-0008/R14(state-accessor), ADR-0009/X3, ADR-0009/R4, ADR-0009/R14(ADR-0001), ADR-0010/X4] | Applied as one batch. (1) `TickDriver.frame()` is a two-statement wrapper, `_frame_body(...)` then an unconditional `Kernel.deliver_frame_end()`; test KRN-FRAME-END added. (2) Composition root issues `NotifyFacet`, binding the subscriber class at issue (ADR-0004 § 2). (3) `PresentationClock` gains read-only `state: int` and `load_seq: int`; the kernel increments `load_seq` on every `LOADING → RUNNING` immediately before `deliver_all`, never on `PAUSED`/`SAVING → RUNNING`; `ChangeBatch` has no load flag; no `KernelStateView` type. (4) `ProcessPriority.HOST_SERVICE = -40` added after `TICK_DRIVER` and before `PRESENTATION`, used by `SaveHost` only; KRN-05 known-node set adds `SaveHost`. (5) `KernelHost` path is `src/game/kernel_host.gd`; the wall-clock allow-list text is replaced by a pointer to ADR-0010 SIM-02. (6) Diagram and probe text per TD-03 and TD-20. `PRESENTATION_LATE` not added. |
| TD-03 (owner ADR-0008) | Diagram and § 1 show one presentation host (`PipelineRoot`) instead of per-node presenters; no `PRESENTATION_LATE`; the camera-write risk text now says the solve runs inside the host's `_process` and `frame_pre_draw` only writes the canvas, with the end-of-`_process` fallback. |
| TD-10 (owner ADR-0006) | `SaveHost` documented as the `HOST_SERVICE` user and kept `PROCESS_MODE_ALWAYS` (§ 6); its frame poll runs after `TICK_DRIVER`. The SaveHost design itself stays in ADR-0006. |
| TD-11 (owner ADR-0010) | SIM-02 allow-list is cited as `kernel_host.gd`, `monotonic_clock.gd`, `system_wall_clock.gd` and the diagnostics timer; wall time reaches no kernel input. This ADR keeps no list. |
| TD-14 (owner ADR-0010) | The clock-API token list in the Implementation Guidelines (old line 170) is removed in favour of a pointer to the unified SIM-02 list; the false claim that `OS.get_ticks_*` cannot be linted is dropped with it. |
| TD-20 (owner ADR-0008) | Verification Required item (4) and the risk text now read: windowed 4.7.2 editor, all node `_process`, then `frame_pre_draw`, then `frame_post_draw` (probed); release export, D3D12 and Deck UNVERIFIED (pipeline spike step 12); thread model pinned by ADR-0008 § 5.1. |
| TD-22 (owner ADR-0008) | § 4 states that `snap_baseline` on entry to `RUNNING` is the only restore-continuity mechanism and that the camera CUT fires on a `load_seq` change. |
| TD-27 (owner ADR-0010) | Enables gains ADR-0005. The ADR-0010 references are forward references to the lint owner, not a dependency, recorded in the Ordering Note. |

Open Questions: this ADR carried none, so none were removed.
Registry: `frame_order`, `presentation_clock` and `simulation_clock` entries are revised (see the returned registry candidates); the registry file itself is not edited here.
