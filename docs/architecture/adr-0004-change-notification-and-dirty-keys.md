# ADR-0004: Change Notification and Dirty Keys

## Status
Proposed

## Date
2026-10-28

## Last Verified
2026-10-28 (against `design/gdd/command-simulation-layer.md` CR-9, CR-10, CR-12, D.9, D.12, Open Question 8 and the Notifications and isolation acceptance criteria; `life-support-graph.md` LS-20; `ship-section-and-layout-data-model.md` LY-32; `ship-body-and-interior-physics.md` BDY-21 and BDY-28; `web-map-and-transit.md` WM-2 and WM-31; `suit-and-equipment.md` SE-34 and SE-38; `failure-and-trail-system.md` FT-15; `emitter-state-and-light-exposure-query.md` EM-26; `2d-lighting-and-occlusion.md` LO-3; `docs/architecture/tr-baseline.md` rows citing ADR-0004; ADR-0001, ADR-0002, ADR-0003 and `docs/engine-reference/godot/` `VERSION.md`, `breaking-changes.md`, `deprecated-apis.md`, `current-best-practices.md`). No game code exists. Seven engine facts (a to g) were probed on the installed 4.7.2 editor (`--headless --script`, an editor build, not a release export) and are marked "probed 2026-10-28"; every timing figure is indicative only and is re-measured by KRN-LAT-02 on a release export. Revised 2026-10-28 after the joint review (engine findings E1 to E9, TD-ADR findings T1 to T12, cross-ADR rulings TD-01, TD-04, TD-07, TD-08, TD-09, TD-16, TD-24, TD-27, TD-31, TD-32, TD-33).

## Decision Makers
Claude (author), running under `modes.automation: autonomous` (`project.yaml`); decisions logged for the user's review; the user has not yet reviewed this ADR. Reviewed 2026-10-28 by the godot-specialist (engine verdict NOTES, findings E1 to E9) and the technical-director (TD-ADR verdict CONCERNS, findings T1 to T12, three BLOCKING), plus the joint cross-ADR rulings; every finding was applied except one optional item declined with a reason. See Review Log. Status stays Proposed.

## Summary
About fifteen systems change state blocks inside kernel write scopes, and a dozen presentation subscribers must learn which parts changed without the kernel ever handing out state or depending on the scene tree. This ADR fixes the whole path: a kernel-owned observer registry (not engine signals) delivering `(block, tick, keys, events)` batches to five subscriber classes through a class-bound `NotifyFacet` issued only to the composition root, 64-bit integer dirty keys `(KIND << 32) | id` collected allocation-free in the write scope with collapse to `ALL` at a latched cap, a per-frame merge for presentation classes (sorted again at frame end), one `ALL` delivery after every load, and an audit, gated on the profile that runs the write tripwire, that catches a changed block that marked no key.

## Engine Compatibility

| Field | Value |
|-------|-------|
| **Engine** | Godot 4.7.2 (stable), GDScript |
| **Domain** | Core (scripting: `Callable` dispatch, packed arrays, `RefCounted` object pools) |
| **Knowledge Risk** | MEDIUM. The one HIGH-risk 4.7 row that bears on this ADR (writing one element of a packed array no longer calls the property setter, `breaking-changes.md` 4.6 to 4.7) is avoided by design: dirty keys are marked only by `scope.mark_dirty`, never from a setter. The `Callable`, `Signal` and `PackedInt64Array` APIs used here predate 4.4 and are not listed in `breaking-changes.md` or `deprecated-apis.md`. The pinned reference has no core or scripting module, so every claim about `PackedInt64Array.has/sort/bsearch/fill`, `Callable.is_standard/get_object/is_valid` and signal behaviour rests on training knowledge plus the probes below. |
| **References Consulted** | `docs/engine-reference/godot/VERSION.md`; `breaking-changes.md` (4.6 to 4.7 GDScript rows: typed-return overrides, packed-array element writes); `deprecated-apis.md` (string `connect`, untyped containers); `current-best-practices.md` (GDScript 4.5+ and 4.7: lambda capture by value, rest parameters, packed-array element writes). No module file exists for this domain. |
| **Post-Cutoff APIs Used** | Typed `Dictionary[StringName, int]` (Dictionary typing, 4.4, thinly covered per `VERSION.md`) for the block-id to index map of § 4, read with `get(key, -1)` and frozen with `make_read_only()`; a plain `Dictionary` read the same way is the fallback with no other change. Variadic `...args` (4.5) is deliberately not used for `emit_event`. |
| **Verification Required** | **Probed 2026-10-28 on the 4.7.2 editor build (`--headless --script`):** (a) a `set` on a `PackedInt64Array` property ran once for a whole-array assignment and **not** for `h.col[1] = 7` (confirms the 4.7 row; setter calls 1 and 1). (b) `const NO_KEY = 0x7FFFFFFFFFFFFFFF` equals the literal `9223372036854775807`; `PackedInt64Array.fill`, `has`, `sort` (sentinels land at the tail) and `bsearch` behave as assumed; `(0x7FFFFFFF << 32) \| 0xFFFFFFFF` is exactly INT64_MAX, hence the kind cap in § 3. (c) `Callable(obj, "m").is_standard()` is true and `get_object()` is the object; `.bind(...)` gives `is_custom()` true; a **lambda** has `is_custom()` true and `get_method()` `"<anonymous lambda>"`; a lambda that does not capture `self` reports the script as its `get_object()`, a self-capturing lambda reports the creator, and both are `is_custom()` and are refused by check 1 of § 2; a lambda stays `is_valid()` after its creator is released. A method `Callable` is `is_valid() == false` after its object is freed. (d) The engine removes a signal connection when its receiver is freed (the connection count drops from 1 to 0 at the free, before any emit; no error). (e) 5,000 clear-and-100-marks cycles (500,000 marks) left `OBJECT_COUNT` unchanged. (f) Indicative editor-build cost: `mark` plus amortised sort and clear 160 to 230 ns per mark (38 distinct keys: 8.8 us per tick; 256 distinct keys: 52 us per tick); `Callable.call` with an `is_valid()` guard 155 ns per call versus signal `emit` 157 ns per call at 10 subscribers. (g) `Callable.get_argument_count()` (subscribe check 1): 2 for a two-parameter method Callable, 2 for a two-parameter lambda, 1 for the same method Callable after `.bind(1)` (whose `is_standard()` is false); the arity check therefore rejects a one-argument bind result as well. **Still UNVERIFIED, settled by KRN-LAT-02 on a release export (RM-DESKTOP and RM-DECK):** every cost above; that signals emit in connection order (training knowledge, never relied on here); that `PackedInt64Array.has` is a single native linear scan on a release template; release-build behaviour of a subscriber that aborts on a script error (the same open point as ADR-0003 Verification 1); the `d[k]` miss behaviour of a typed `Dictionary` (the design never relies on it: lookups use `get(k, -1)`); that a handler containing `await` returns a coroutine state object instead of the token (known GDScript behaviour, not probed; guarded by the ADR-0010 lint and the token identity check of § 2); the cost of sorting and scanning the full capacity rather than `count` (E9, a KRN-LAT-02 measurement target). |
| **Engine Review** | godot-specialist 2026-10-28 against 4.7.2: no typed-return override (the only override is `_init(...) -> void`; handlers, `ChangeBatch` and `KeyBuf` override nothing), no const-of-object (only `const NO_KEY: int` and `const MAX_KIND: int`), no Object-method name clash (`key`, `has`, `event`, `count`, `kind`, `tick`, `buf`, `clear`, `subscribe` are all fine; a parameter named `key` next to a method `key()` compiled). Packed arrays are plain members of `RefCounted` classes and never exposed; the 4.7 element-write change is irrelevant to `buf[count] = key` because no setter exists. Typed `Dictionary[StringName,int]` and `Array[StringName]` are valid on 4.7.2; nothing deprecated is used. |

## ADR Dependencies

| Field | Value |
|-------|-------|
| **Depends On** | ADR-0001 (injected facets; the driver wrapper calls the frame-end delivery; Proposed), ADR-0002 (handles are independent of subscription; Proposed), ADR-0003 (`WriteScope`, `NotifyContext`, `BlockRegistrar`, violation set, `kernel.clock`; Proposed). |
| **Soft references (not acceptance prerequisites)** | ADR-0010 (source layout `src/kernel/facets/` and `src/presentation/`, the ARCH-08 and SIM-13 lint enforcement that this ADR's type placement and transport rule rely on; Proposed). Kept out of Depends On so the dependency graph has no ADR-0004 / ADR-0010 cycle. |
| **Enables** | ADR-0009 (presentation subscriber contract consumes `ChangeBatch`, `DirtyAccumulator` and the class-bound facet; ADR-0009 adds ADR-0004 to its Depends On), ADR-0005 (cache validity by the touch serial of § 7, adopted by its § 5.2), ADR-0006 (related, not a dependency: Save/Load observes autosave triggers by polling owner read views, with no subscriber class; it closes its OQ-F; ADR-0006 does not list this ADR under Depends On), ADR-0031 (counters, the `DIAGNOSTIC` gate), ADR-0032 (REPLICATION cadence), and every system ADR that registers a block (ADR-0011, 0013, 0015, 0017, 0019, 0020 to 0027: each declares a key space). |
| **Blocks** | Epics: Command & Simulation Layer (P5 notify), and every story that registers a block or subscribes to one. |
| **Ordering Note** | Forward references, none blocking authorship: ADR-0005 (`block_hash_any`, H64, the hash cache that reads the touch serial), ADR-0009 (which keys fire when, handler discipline, which facets carry `perception_access`), ADR-0010 (lint mechanisms), ADR-0031, ADR-0032. **Scope edges.** With ADR-0003: the dirty key was an opaque `int` there; this ADR fixes its layout and adds small things to ADR-0003's surface (`WriteScope.emit_event`, `BlockRegistrar.register_key_space`, five violation names, `touch_serial` in the H1 registries group), all listed in § Cross-ADR amendments. With ADR-0001: `TickDriver.frame` becomes a thin wrapper that always ends in `deliver_frame_end`. With ADR-0009: this ADR decides transport, types, placement and delivery; ADR-0009 decides what presentation may read and write, which keys fire when, and which facets carry `perception_access`. **Acceptance requires** ADR-0001, 0002 and 0003 Accepted **with the Cross-ADR amendments of this ADR applied** (ADR-0001: `TickDriver.frame` as a two-statement wrapper whose last step is `deliver_frame_end` on every exit path, KRN-FRAME-END, and `NotifyFacet` issued by the composition root in Key Interfaces; ADR-0003: `emit_event`, `register_key_space`, the five violation names, `UNKNOWN_DEPENDENCY` widened, `touch_serial` under the H1 registries group, the § 4 missed-mark sentence refined to § 7, the opaque-int comment pointing to § 3, the two-face `PresentationViews`), and KRN-LAT-02 passed for notify cost. |

## Context

### Problem Statement
The kernel GDD (CR-10, D.9) specifies change notification in prose: one batch per block per tick, `(block_id, tick, change_keys)`, no state in the payload, five subscriber classes, a collapse to `ALL` past `MAX_DIRTY_KEYS`, a merged delivery for presentation in a multi-tick frame, and subscriber lifetime rules. It leaves open the transport (Open Question 8, architecture OQ-4: kernel registry or engine signals), and ADR-0003 left the dirty key an opaque `int`. Eleven owner GDDs already write `(KIND << 32) | id` keys and rely on explicit marking, because 4.7 packed-array element writes call no setter. Missing is the code-level shape: what a key is and who may mint one, how a mutator marks a key without allocating, what a subscriber receives and may keep, how delivery stays cheap on a catch-up frame, how a restore reaches every subscriber, how P5 events (`BODY_TRANSFER`, `POD_ARRIVED`) travel without carrying state, and how a forgotten mark is caught before it becomes a stale display. Without one answer each of twelve presentation systems would invent its own dirty set and its own handling of `ALL`.

### Constraints
- **Payloads carry no state** and notifications are identical in HOST and MIRROR (CR-10). A key is an identifier; a subscriber fetches values from a read view.
- **4.7 packed-array element writes call no setter.** Dirty tracking cannot ride a setter; marks are explicit inside mutators (ADR-0003 § 4, registry `write_outside_writescope_or_setter_dirty`).
- **A quiet tick allocates nothing in the kernel** (ADR-0003 Performance; KRN-PERF-01: 32 blocks, 10 subscribers, no dirty block, P5 delivers 0 batches and the kernel allocates 0 containers). `OBJECT_COUNT` counts only Objects, not arrays, so identity and `is_read_only` style checks accompany it (ADR-0007).
- **`MAX_DIRTY_KEYS` 256** (knob 32 to 4096; never below 128 while the Graph is registered, Graph D.10); the worst designed cascade is 38 keys at MVP and 135 at VS-A.
- **Handlers run with writes disabled**; a subscriber handler is a kernel-invoked callable, so ADR-0003's rule applies: `-> CompletionToken`, `return ctx.complete()` last (`NotifyContext`), and a missing token is a recorded `INCOMPLETE`, never a fault (kernel AC, Notifications and isolation).
- **Catch-up merge is ratified** (R-12): PRESENTATION, PROJECTOR and DIAGNOSTIC get one batch per frame; INPUT_SURFACE and REPLICATION get every tick; ticks stay atomic.
- **An object that owns a block or hook never subscribes** (CR-10), per object not per system.
- **Subscribing confers no submit right** (ADR-0002, registry `presentation_submit_without_handle`).
- **Determinism.** Delivery order, key order and merge results are pure functions of registration order and the tick's marks; the kernel never iterates a `Dictionary` to deliver (CR-13).
- **Scene-tree free kernel** (H10). Engine signals would work on a `RefCounted`, so this is not decisive for § 1, but nothing here needs the tree.
- **Dev-only observers** (Debug Overlay) are absent from release builds.

### Requirements
- One transport, with the subscribe, deliver, prune and dispose rules of CR-10 testable with stub subscribers and no scene.
- One key layout every owner uses, bounded, printable by name in diagnostics, with no `String` built per key.
- Marking allocation-free, deduplicated, with exact collapse (256 distinct keys list; 257 give `ALL`).
- A batch type and a union helper that ADR-0009 can build on without copying kernel buffers.
- Events as ints, delivered with their block's batch, never merged away in a catch-up frame.
- Exactly one `ALL` per block after a load or new game; nothing from `INIT` and `RESTORE` scopes leaks as keys.
- A debug check that a changed block marked a key, honest about blocks whose GDD forbids a key per write.

## Decision

Seven numbered sub-decisions, each reviewable alone: **§ 1 transport**, **§ 2 subscriber classes and lifetime**, **§ 3 the key layout and key spaces**, **§ 4 collection in the write scope**, **§ 5 batch, event and accumulator types**, **§ 6 delivery: per tick, catch-up merge, restore**, **§ 7 the missed-mark audit and the touch serial**.

### Architecture Diagram

```
BOOTING   owner.setup(facet): registrar.register_block(...) + registrar.register_key_space(KeySpaceDescriptor)   (§ 3)
          composition root: nf = kernel.issue_notify_facet(cls, perception_access)   (class fixed at issue, § 2)
          presentation.setup(presentation_facet): presentation_facet.subscribe(block_id, handler, name)        (ADR-0009 wraps nf)
freeze    resolve subscription block ids (unknown id -> boot fault UNKNOWN_DEPENDENCY); re-evaluate checks 2 and 6;
          size key buffers (capacity = MAX_DIRTY_KEYS at the freeze), event pool and subscriber index arrays

Per tick T (inside Kernel.step)
  P0  latch MAX_DIRTY_KEYS knob (a change made during tick T-1 takes effect now): set_cap(k) on every tick and frame KeyBuf
  P2/P3  mutator: scope.open_for(block) -> touch_serial[block] = scope serial
                  scope.mark_dirty(block, key)  -> tick_buf[block]  (has() dedupe; distinct key number cap+1 -> is_all)
                  scope.emit_event(block, kind, a0..a5) -> global event pool
         (INIT / RESTORE scopes: touch recorded, keys and events discarded)
  P4  commit_baseline; close: for each dirty block ascending: sort tick_buf, finalise count / is_all
  P5  apply deferred subscribe/unsubscribe; deliver tick batches to PER_TICK classes (INPUT_SURFACE, REPLICATION)
      then fold tick_buf into frame_buf (empty: swap; else has()-union, past cap -> ALL; event indices appended); clear tick_buf; flush results
TickDriver.frame():  _frame_body(...)  ;  _kernel.deliver_frame_end()      (two statements, the second unconditional)
  deliver_frame_end: apply deferred changes; close() every frame_buf that received a union (sort); deliver frame_buf batches
                     to FRAME_MERGED classes (PRESENTATION, PROJECTOR, DIAGNOSTIC); clear
LOADING -> RUNNING   Kernel.deliver_all(): clear tick_buf, frame_buf, events; one ALL batch per block to EVERY class, tick = restored T

Subscriber handler (kernel-invoked, writes disabled, never awaits):
  func _on_x(batch: ChangeBatch, ctx: NotifyContext) -> CompletionToken:
      _dirty.absorb(batch)          # DirtyAccumulator: marks dirty only (ADR-0009 decides the rest)
      return ctx.complete()
```

### 1. Transport: a kernel-owned observer registry, not engine signals

The kernel keeps its own subscription registry and calls each subscriber's registered method `Callable` directly (`Callable.call(batch, ctx)`). It declares no engine `signal` for change notification, and no subscriber ever calls `connect` on a kernel object. Engine signals remain the right tool inside presentation (`card_opened`, a settings change), as the architecture's event path already states.

**The reason is semantics, not cost.** A signal connection returns nothing, carries no subscriber class and no block filter, and offers no hook for the contract the kernel GDD makes testable:

1. The handler must end in `return ctx.complete()` so the kernel can record `INCOMPLETE` per subscriber and still deliver to the rest (ADR-0003 § 3, kernel AC). A signal discards handler return values.
2. Delivery has **two cadences** (per tick and per merged frame) selected by subscriber class. Signals would need one declared signal per cadence per block (up to 128 blocks), or one signal and filtering in every handler, and would call every subscriber for every block.
3. CR-10's snapshot rule and `DEAD_SUBSCRIBER` counter need the kernel to see the subscription list. The engine removes the connection when the receiver is freed, with no error (probed 2026-10-28: the connection count drops from 1 to 0 at the free, before any emit), so the count the Debug Overlay shows would be lost.
4. The subscribe-time checks of § 2 (not an owner, standard method Callable, class gate, block exists) need the Callable at subscribe time, in one place, with one counter.
5. Block-scoped delivery makes P5 cost O(dirty blocks x their subscriptions) and a quiet tick zero calls.

**Cost does not separate the options** (probed 2026-10-28, editor build, 10 subscribers: `Callable.call` loop with an `is_valid()` guard 155 ns per call, signal `emit` 157 ns per call). That answers OQ-4 provisionally: the kernel GDD's lean (registry) is decided here on semantics. **Acceptance gate:** the KRN-LAT-02 skeleton measures registry notify cost on a release export against the proposal in Performance Implications (a proposal, not a GDD number: the GDD defines `t_P5` but no budget). If it fails, this ADR is reopened; there is no automatic fallback to signals, because the rejection reasons above are not cost reasons. Fix cost inside the registry first (fewer subscriptions per block, a count-bounded sort, the open-addressed set of Alternative 6). The transport rule is enforced by ADR-0010 SIM-13 (an active rule citing this section: kernel observer registry, no engine signals for kernel notification).

### 2. Subscriber classes, subscription and lifetime

Five classes, closed set (CR-10). This ADR fixes their **delivery** properties; ADR-0009 fixes what each may read and write (the CR-10 table is its starting point) and ADR-0002 fixes who may submit.

| Class | Cadence | Gets events | Gets `ALL` on load | Gate | Examples |
|---|---|---|---|---|---|
| `PRESENTATION` | `FRAME_MERGED` | yes | yes | none | Trail, Instruments, HUD, Lighting, Perception Filter, Audio, Avatar Presenter |
| `INPUT_SURFACE` | `PER_TICK` | yes | yes | none beyond § 2 checks | Input Adapter, Chart Table menus |
| `PROJECTOR` | `FRAME_MERGED` | yes | yes | none | Layout scene projector, occluder builder, body projector |
| `REPLICATION` | `PER_TICK` | yes | yes | refused unless `replication_enabled` (false at MVP; tests may enable) | Co-op adapter (VS-B, ADR-0032) |
| `DIAGNOSTIC` | `FRAME_MERGED` | yes | yes | refused unless the dev predicate holds (feature `dev` or `editor`, set by the composition root, ADR-0031) | Debug Overlay; the only class that may use the wildcard block `&"*"` |

**Subscription is per block.** One subscription names one block (Layout GDD LY-32: "subscribers that need both subscribe to both"). A subscriber that needs two blocks subscribes twice with the same handler. This makes the work proportional to dirty blocks and keeps a quiet tick at zero calls.

**Who holds what.** The subscriber class is **not** an argument a subscriber chooses. The kernel issues a `NotifyFacet` to the composition root with the class and a `perception_access` flag fixed at issue (`Kernel.issue_notify_facet(cls, perception_access)`); the composition root alone holds `NotifyFacet`. Presentation code receives ADR-0009's `PresentationFacet`, the class-bound wrapper the composition root builds around that facet, with `subscribe(block_id, handler, name)`. A presenter therefore cannot pick its own class, and no presentation type names the kernel-side facet (ADR-0010 ARCH-08 allows `NotifyFacet` only in `src/game/` and in `PresentationFacet`). `perception_access` is `true` only on the Perception Filter's facet and on `DIAGNOSTIC` facets (ADR-0009 § 1 lists them) and `false` everywhere else.

```gdscript
# src/kernel/facets/notify_facet.gd
class_name NotifyFacet extends RefCounted     # issued by the kernel to the composition root; never handed to a presenter
var key_capacity: int                          # the sized MAX_DIRTY_KEYS; DirtyAccumulator capacity (PresentationFacet forwards it)
func subscribe(block_id: StringName, handler: Callable, name: StringName) -> SubscriptionHandle   # class and perception_access fixed at issue
```

**Subscribe-time checks.** Each failure returns an inactive handle, counts `SUBSCRIBE_REFUSED`, calls the `STRICT_DEBUG` violation handler, and never faults the kernel (except the freeze-time rule of checks 2 and 6, below):

1. `handler.is_valid()`, `handler.is_standard()` (a **method** Callable: lambdas and `.bind()` results are `is_custom()`, probed; a lambda that captures `self` reports its creator as `get_object()` while one that does not reports the script, so check 2 could not rely on `get_object()` for lambdas, and `is_standard()` rejects every lambda first; a lambda that captures `self` also creates the cycle ADR-0002 already forbids for result handlers) and `handler.get_argument_count() == 2` (probed: a bind result with one bound argument reports 1).
2. `handler.get_object()` is not an object that owns a block or a hook: the kernel keeps the set of instance ids of every registered descriptor Callable's object. (This is the per-object rule of CR-10; the Input Adapter of the same system is a different object and is accepted.) Presentation `setup()` may run before every owner has registered, so, like checks 4 and 6, this check is **re-evaluated at the registry freeze** for subscriptions made earlier. A hit there is treated like the unknown-block case of check 4: under `STRICT_DEBUG` it is a **boot failure** reported with code `SUBSCRIBE_REFUSED` (the same reasoning applies, a subscription that silently never fires is the worse failure); under `LENIENT_RELEASE` it removes the subscription and counts `SUBSCRIBE_REFUSED`.
3. The class gate of the table above.
4. The block id exists. Before the registry freeze the name is stored and resolved at the freeze; an unknown name then is a **boot failure** with the existing kernel code `UNKNOWN_DEPENDENCY` (its meaning widens to "unknown subscription block"), because a typo that silently never fires is the worse failure. After the freeze an unknown id is refused as above.
5. The registry holds fewer than `MAX_SUBSCRIPTIONS` (96).
6. **PERCEPTION_OUT gate.** A subscription to a block whose descriptor read class is `PERCEPTION_OUT` is refused with `SUBSCRIBE_REFUSED` unless the issuing facet carries `perception_access = true`. Without it a presentation subscriber could learn the dirty-key ids, and so the timing and identity, of perception changes even though ADR-0009 blocks the read. Evaluated at subscribe time, and at the freeze for earlier subscriptions with the freeze-time rule of check 2. The wildcard block `&"*"` (a `DIAGNOSTIC` facet, which carries the flag) therefore includes `PERCEPTION_OUT` blocks.

**Handler contract.** `func handler(batch: ChangeBatch, ctx: NotifyContext) -> CompletionToken`, last statement `return ctx.complete()`. **A handler must return the armed token synchronously and must never `await`**: a handler containing `await` returns a coroutine state object, not the token, and its continuation would resume after P5 holding a poisoned `ChangeBatch` and a re-armed `NotifyContext`. A handler that returns anything other than the armed token (nothing, `null`, a stale token from an earlier delivery, or a suspended coroutine state) is recorded `INCOMPLETE` against its subscription name by the kernel's token identity and epoch check (the runtime backstop; ADR-0010's PRES rule bans `await` lexically in any script registered through `PresenterDescriptor` or calling `.subscribe(`). Other subscribers still receive the batch, the kernel state does not change. Subscribing as `INPUT_SURFACE` confers no submit right (ADR-0002); `PROJECTOR` and `PRESENTATION` handlers run in P5 with every scope `CLOSED`, so a mutator called from a handler writes nothing, counts `WRITE_OUTSIDE_SCOPE` and does not stop delivery to the other subscribers.

**Lifetime.**
- `SubscriptionHandle.unsubscribe()` is idempotent; `is_active()` reports state.
- The kernel holds only Callables, which do not keep their object alive (ADR-0003 Verification 3). Subscribers are held by the composition root; a freed subscriber is detected by `handler.is_valid()` just before its call, pruned, and counted `DEAD_SUBSCRIBER` (not an error, not `INCOMPLETE`, no engine error).
- **Deferred application replaces the per-tick list copy.** CR-10 says P5 iterates a snapshot of the subscriber list taken before P5. Copying a list every tick would allocate. The kernel instead queues `subscribe`, `unsubscribe` and prune requests and applies them at the start of each delivery boundary (before the per-tick delivery of a tick and before the frame-end delivery), rebuilding the per-block subscriber arrays only when a request is pending. **The rebuild allocates nothing:** the subscription table is a fixed array of `MAX_SUBSCRIPTIONS` slots, each per-block subscriber list is a fixed-capacity `PackedInt32Array` of slot indices (capacity `MAX_SUBSCRIPTIONS`, with a count) rebuilt in place, and the pending-request queue is a preallocated packed array of slot indices and operation flags, all created at the freeze. A `DEAD_SUBSCRIBER` prune therefore may fall on a quiet-tick boundary without breaking KRN-PERF-01. (The `subscribe()` call itself creates a `SubscriptionHandle` and is a setup-time operation outside the zero-allocation guarantee.) Observable behaviour is the snapshot behaviour: a handler that subscribes or unsubscribes during delivery affects the next boundary only, and an unsubscribed peer still receives the batch being delivered (KRN-NOT-02).
- `Kernel.dispose()` (idempotent) clears the registry, the key buffers, the event pool and every handle, so a test kernel is freed deterministically (KRN-NOT-03).

### 3. Dirty keys: a 64-bit integer, `(KIND << 32) | id`, declared per block

Every owner GDD already writes this encoding (Graph LS-20, Layout LY-32, Body BDY-21, Emitter EM-26). This ADR fixes it.

| Bits | Field | Range | Notes |
|---|---|---|---|
| 63 | sign | always 0 | a key is positive |
| 32 to 62 | `KIND` | 1 to `0x7FFF` by rule | bits 32 to 46 used, 47 to 62 reserved; kind 0 is reserved (key 0 is invalid); the cap keeps every key below 2^47, exact as a `float` (floats are exact to 2^53, so the cap is stricter than that argument needs), and strictly below `NO_KEY` |
| 0 to 31 | `id` | 0 to `0xFFFFFFFF` | 0 for block-singleton families (`T`, `WEB/T`, `WEB/POD`, `ATT/L`, `ATT/P`) |

- **The key is block-local.** `(layout.state, SEC/3)` and `(graph.state, SEC/3)` are different keys; the batch names the block. Kinds are declared per block, so two blocks may reuse a number. The shared family names in the GDDs (`S/<seam_id>`, `SEC/<id>`) are documentation, not a shared namespace.
- **The `ALL` sentinel is a batch flag**, not a key value (§ 5), so no key value is reserved for it.
- **`NO_KEY = 0x7FFFFFFFFFFFFFFF`** (INT64_MAX, probed) is the empty-slot sentinel of the buffers in § 4. It can never be a valid key.
- **Always integers.** No `String`, `StringName`, `float` or `Variant` key is ever built (Graph LS-20: no String per key per run). Composite ids that are naturally pairs (Graph `L/<loop>/<anchor>`) are packed by the owner into the 32-bit id (for example loop in the high 16 bits and anchor in the low 16) and range-checked under `STRICT_DEBUG`.
- **Packed arrays holding keys are `PackedInt64Array`.** (ADR-0003 used `PackedInt32Array` for the hook schedule; a key needs 47 bits.)

```gdscript
class_name DirtyKey extends RefCounted          # static helpers only, never instantiated
const NO_KEY: int = 0x7FFFFFFFFFFFFFFF
const MAX_KIND: int = 0x7FFF
static func make(kind: int, id: int) -> int:   # STRICT_DEBUG range-checks; release trusts the caller
	return (kind << 32) | id
static func kind_of(key: int) -> int:
	return key >> 32
static func id_of(key: int) -> int:
	return key & 0xFFFFFFFF
```

**Key spaces are declared at `BOOTING`**, one per block, with every other descriptor field (a missing field fails boot, CR-6). This is how the debug audit and the printers know the vocabulary, and how a block says whether a write without a key is legal (§ 7).

```gdscript
class_name KeySpaceDescriptor extends RefCounted
var block_id: StringName
var kind_ids: PackedInt32Array            # e.g. [1, 2, 3]
var kind_names: Array[StringName]         # e.g. [&"SEC", &"BODY", &"SEAM"], same order; for diagnostics only
var event_ids: PackedInt32Array           # event kinds this block may emit (may be empty)
var event_names: Array[StringName]
var quiet_writes_allowed: bool            # true: a write that changes the block may legitimately mark no key (quantised families)
# BlockRegistrar (ADR-0003) gains:  func register_key_space(desc: KeySpaceDescriptor) -> void
```

MVP families named by the GDDs (the numeric kind of a family the GDD did not number is the owner's ADR to fix; only distinctness inside a block is required here):

| Block | Families | Kinds fixed by the GDD |
|---|---|---|
| `graph.state` | `C/<cid>`, `B/<cid>`, `N/<id>`, `F/<id>`, `L/<loop>/<anchor>`, `S/<seam_id>`, `T`, `SEC/<id>` | COMP 1, BAND 2, NODE 3, EDGE 4, DESC 5, LOOP 6, SEAM 7, TOPO 8, SECTION 9 (LS-20) |
| `layout.state` | `SEC/<section_id>`, `BODY/<body_id>`, `S/<seam_id>` | SEC 1, BODY 2, SEAM 3 (LY-32) |
| `body.state` | `BODY/<body_id>`, `ACT/<actor_id>` | BODY 2, ACT 4 (BDY-21) |
| `emitter.state` | `E/<emitter id>` | owner |
| `suit.state` | `S/<PlayerId>` | owner |
| `web.state` | `WEB/<derelict_id>`, `WEB/T`, `WEB/POD` | owner |
| `attention.state` | `ATT/L`, `ATT/P` | owner |

**`MAX_DIRTY_KEYS`** (kernel knob, 256, range 32 to 4096, at least 128 while `graph.state` is registered). Buffers are **allocated** at the sized capacity `buf.size()` fixed at the registry freeze (the knob value then) and never resized. The **active collapse threshold** is a separate field, `KeyBuf.cap`. The knob may be **lowered** at run time: at the next tick's P0 the kernel latches the new value and calls `set_cap(k)` on every tick and every frame `KeyBuf` (a change during P2 of `T` uses the old value in `T`'s P5, KRN AC); raising it above the sized capacity is refused with the existing `KNOB_REFUSED` violation. `set_cap` is allocation-free; if a frame buffer already holds more than the new cap (mid-frame), it collapses to `is_all`, which is the conservative direction (over-delivering is legal). A `DirtyAccumulator` keeps the sized capacity (§ 5), the one place the lowered knob does not apply, deliberately.

### 4. Collection in the write scope: a fixed `PackedInt64Array` per block, `has()` dedupe, exact collapse

The GDD criteria require deduplication (marks `a, b, b` give `{a, b}`; writing a key to its existing value still marks) and a collapse on the **distinct** count. A `Dictionary` used as a set allocates on growth and would be iterated; a GDScript linear scan costs up to 256 element reads per mark. A preallocated native-scanned array does neither.

```gdscript
# src/kernel/facets/key_buf.gd   (facets, so DirtyAccumulator has no kernel-core edge)
class_name KeyBuf extends RefCounted             # kernel-internal use: one per block for the tick, one per block for the frame
var buf: PackedInt64Array                        # size = sized capacity (MAX_DIRTY_KEYS at the freeze); filled with NO_KEY; never resized
var cap: int                                     # active collapse threshold, 1 .. buf.size(); set by set_cap at the P0 latch
var count: int                                   # distinct keys held (valid while not is_all)
var is_all: bool                                 # true once a distinct key arrived with count == cap
var unioned: bool                                # true once a fold appended keys by union (not by swap): needs close() before delivery
func set_cap(k: int) -> void:                    # k <= buf.size(); allocation-free
	cap = k
	if count > cap:
		is_all = true                            # holds more than the new cap: collapse (conservative)
		count = 0
func mark(key: int) -> void:
	if is_all or buf.has(key):                   # native scan over the full capacity (<= 4096)
		return
	if count == cap:                             # cap, not buf.size(): a lowered knob takes effect here
		is_all = true                            # distinct key number cap + 1 (the 257th at 256): collapse, drop keys
		count = 0
		return
	buf[count] = key
	count += 1
func union_from(other: KeyBuf) -> void:          # the § 6 fold when the frame buffer is not empty
	if is_all:
		return
	if other.is_all:
		is_all = true
		count = 0
		return
	for i in other.count:
		mark(other.buf[i])                       # honours cap; may set is_all
	unioned = true                               # appended keys are not in sorted order any more
func close() -> void:                            # P4 for tick buffers; frame end for a frame buffer with unioned == true
	if not is_all and count > 1:
		buf.sort()                               # native sort puts NO_KEY at the tail; keys are ascending, unique
func clear() -> void:
	buf.fill(DirtyKey.NO_KEY)
	count = 0
	is_all = false
	unioned = false
```

- **`scope.mark_dirty(block_id, key)`** (ADR-0003 § 4 signature, unchanged) does, in order: (1) if the scope is not currently open for `block_id` (no successful `open_for` for that block in this scope use) it writes nothing and counts `WRITE_OUTSIDE_SCOPE` (not counted again once the scope has faulted); (2) if `key <= 0` or `key >= NO_KEY` it counts `DIRTY_KEY_INVALID` and returns (always on: two compares); (3) in `INIT` and `RESTORE` phases it returns (those keys are discarded, ADR-0003 § 4; § 6 delivers `ALL` instead); (4) under `STRICT_DEBUG` it checks the key's kind against the block's key space and counts `DIRTY_KEY_KIND_UNDECLARED`; (5) `tick_buf[block].mark(key)`.
- **Resolving `block_id` to a block index** is one typed `Dictionary[StringName, int]` lookup written `index_of.get(block_id, -1)`, never `index_of[block_id]`: `-1` means an unknown block and is treated as `WRITE_OUTSIDE_SCOPE` (counted, nothing written), so an unknown id counts a violation instead of raising a script error on the hot path. The map is built at the freeze and frozen with `index_of.make_read_only()` (the kernel reads it by key, never iterates it). `open_for` caches the index in the scope so `mark_dirty` for the block just opened does not look it up again.
- **A mark is not a diff.** A key marked for a write of an unchanged value is delivered (KRN AC). Over-marking is legal; under-marking is a defect (§ 7).
- **Exactness.** Capacity is the latched cap (the knob value, 256 by default), so exactly `cap` distinct keys are listed and the next distinct key gives `ALL`; a key marked 1,000 times is one key (probed 2026-10-28, `EXACT256`: `is_all` false, count 256; `OVERFLOW` at 257: `is_all` true).
- **Dirty-block bookkeeping.** The kernel keeps one flag per block (a `PackedByteArray` of `MAX_BLOCKS` 128) set on the first mark or event of a tick; P4 scans it in ascending block index, so block order is deterministic and a quiet tick costs one scan of at most 128 bytes.
- **No setter, no automatic tracking.** Marks come only from the mutator (registry `write_outside_writescope_or_setter_dirty`).

### 5. Types: `ChangeBatch`, `NotifyEvent`, `DirtyAccumulator`

**`ChangeBatch`** is a reused facade over the kernel's closed buffer. It is not a copy and not a container the subscriber may keep.

```gdscript
class_name ChangeBatch extends RefCounted
var block_id: StringName
var tick: int                          # the last committed tick this batch covers
var first_tick: int                    # equals tick for PER_TICK batches; first tick of the frame for merged batches
var is_all: bool                       # true: re-read everything you use from this block; count is 0
var count: int                         # number of distinct keys (0 when is_all)
var event_count: int
func key(i: int) -> int                # 0 <= i < count; keys are ascending and unique
func has(key: int) -> bool             # var i := buf.bsearch(key); return i < count and buf[i] == key   (false while is_all)
func event(i: int) -> NotifyEvent      # emission order; ticks ascending in a merged batch
```

- **The container type of `change_keys` is the kernel-held `PackedInt64Array` behind `key(i)` / `has(key)`.** The array itself is never handed out (packed arrays pass by reference and cannot be made read-only, ADR-0007 review B2).
- **Valid only during the handler call.** One `ChangeBatch` object exists per block per cadence and is re-armed for every delivery; under `STRICT_DEBUG` it is poisoned (`tick = -1`) after the call so a retained reference fails visibly. A subscriber copies what it needs, normally into a `DirtyAccumulator`.
- **Order.** Keys ascending and unique on **every delivered batch** (canonical, so HOST and MIRROR batches compare equal and tests compare arrays); the frame-end sort of § 6 restores this for a merged batch rather than weakening it. Subscribers must still treat the batch as a set. `has` is a `bsearch` over the sorted buffer, so it **relies on that order**. `PackedInt64Array.bsearch(value)` returns an insertion index and never -1 (probed 2026-10-28: `bsearch(6)` on `[1,5,9,MAX,MAX]` returned 2, `bsearch(100)` returned 3), so `bsearch(k) >= 0` is always true and is wrong; the contract is `var i := buf.bsearch(key); return i < count and buf[i] == key`, bounded by `count`, not `buf.size()`, so a `NO_KEY` probe can never match.
- **`is_all`** replaces the old `ALL` sentinel. A block with no keys and no events produces no batch at all.

**`DirtyAccumulator`** is the subscriber-side union that Failure & Trail FT-15 and every PRESENTATION system needs: store the keys since the last render callback, collapse on overflow, expose `has`.

```gdscript
# src/kernel/facets/dirty_accumulator.gd   (self-contained: built on KeyBuf in the same folder; no kernel-core edge)
class_name DirtyAccumulator extends RefCounted
func _init(capacity: int) -> void              # facet.key_capacity: the SIZED capacity
var is_all: bool
var count: int
func absorb(batch: ChangeBatch) -> void        # is_all if batch.is_all or the union would exceed capacity; otherwise has()-union
func has(key: int) -> bool                     # true for every key while is_all
func key(i: int) -> int
func clear() -> void                           # call in the render callback after consuming
```

It uses `KeyBuf` (`src/kernel/facets/key_buf.gd`) and allocates nothing after `_init`. **Its capacity is the sized capacity, not the latched cap.** A lowered knob makes the kernel collapse earlier; an accumulator that collapses later never loses a key (it holds a finer set that is a subset of what a full re-read covers, and any `is_all` batch still propagates), so keeping the sized capacity is safe and avoids a kernel-core edge from a presentation-side type. The cost is only that the accumulator can be finer than the kernel; the knob-lowering test (Validation Criteria) covers the kernel side.

**`NotifyEvent`** (P5 events: `BODY_TRANSFER_WELD`, `BODY_TRANSFER_DETACH`, `BODY_TRANSFER_POD`, `BODY_DOCK_CONTACT`, `BODY_HULL_SEPARATION`, `POD_DEPARTED`, `POD_ARRIVED`, `DERELICT_LOADED`, `DERELICT_UNLOADED`, `BERTHED`, `UNBERTHED`, `FIRST_BOARDED`). An event is a kind and up to six integers, never state:

```gdscript
class_name NotifyEvent extends RefCounted      # pooled; the kernel preallocates MAX_TICKS_PER_FRAME x 64 at the freeze
var kind: int                                  # declared in the emitting block's KeySpaceDescriptor.event_ids
var tick: int
var a0: int
var a1: int
var a2: int
var a3: int
var a4: int
var a5: int
# WriteScope (ADR-0003) gains:
# func emit_event(block_id: StringName, kind: int, a0: int = 0, a1: int = 0, a2: int = 0, a3: int = 0, a4: int = 0, a5: int = 0) -> void
```

- The GDD payloads fit: `BODY_TRANSFER{weld | detach | pod}` is three kinds with `{actor_id, seat, old_body, new_body, frame_delta_px.x, frame_delta_px.y}` (six ints, the sub-kind folded into the event kind); `POD_DEPARTED` is five ints; the tick travels in the event, not the args.
- **An event belongs to the block that emitted it** and is delivered in that block's batch to every subscription on that block, any class (a batch may therefore have events and no keys). It is not a second channel and needs no second subscription.
- `emit_event` follows the same scope rules as `mark_dirty` (open block required; discarded in `INIT`, `RESTORE`, `CLOSED`). **The limits are global, not per block:** `MAX_EVENTS_PER_TICK` (64) across all blocks in one tick, and `MAX_TICKS_PER_FRAME` x 64 across all blocks in one frame; overflow drops the newest and counts `EVENT_OVERFLOW`.
- **Storage, so no event tick allocates.** One global pool of `MAX_TICKS_PER_FRAME` x 64 `NotifyEvent` objects is preallocated at the freeze and bump-allocated through the frame, with a per-tick watermark (a faulted tick resets to its watermark, releasing its events). Each block whose key space declares `event_ids` owns two fixed-size `PackedInt32Array` index lists into that pool (one for the tick, capacity 64; one for the frame, capacity `MAX_TICKS_PER_FRAME` x 64; each with a count), preallocated at the freeze; a block that declares no events has none. The fold appends indices and copies no event. Events are **not** unioned: a merged batch carries every event of every committed tick of the frame, in order. Under `STRICT_DEBUG` the pooled `NotifyEvent`s are poisoned (`tick = -1`) after delivery together with the batch. The call's integer arguments are evaluated at the call site only.

### 6. Delivery: per tick, catch-up merge, early exit, restore

Each block has two `KeyBuf`s (tick and frame) and, if it declares events, two event index lists (§ 5).

**P4 close.** After `commit_baseline`, for each dirty block in ascending index: `tick_buf.close()` (native sort). If the tick **faulted**, every tick buffer and event list is cleared and nothing is delivered or merged for it (KRN-NOT-01: a faulting tick contributes nothing; P4 never ran).

**P5, per tick.** The kernel applies deferred subscription changes, then for each dirty block in ascending index and each `PER_TICK` subscription of that block in subscription order, arms the shared `NotifyContext`, calls the handler with the block's tick batch, checks the returned token (identity and epoch, ADR-0003 § 3), and records `INCOMPLETE` or `DEAD_SUBSCRIBER` as needed. Then each tick buffer is folded into its frame buffer: if the frame buffer is empty the two `KeyBuf` objects are swapped (O(1); the swapped-in buffer was closed at P4, so it is already sorted); otherwise `frame_buf.union_from(tick_buf)` runs (a `has()` union, a union beyond the latched cap sets `is_all`) and the frame buffer is marked `unioned`; event indices are appended. The tick buffer is cleared. The result-buffer flush follows the change batch, as in the kernel GDD.

**Frame end.** `TickDriver.frame` is a **two-statement wrapper** (ADR-0001): `_frame_body(delta_us)`, which holds the boundary service, the tick loop, forgiveness, the result flush and the paused service and **may `return` early** on a pause, a restore request or a fault, followed unconditionally by `_kernel.deliver_frame_end()`, with no statement after it and no other statement besides the body. GDScript has no try/finally, so "on every exit path" is achieved structurally, not by a trailing statement in a body that can return: `frame()` has exactly one `return` (or none), checked by a lint or test (owner ADR-0001; KRN-FRAME-END). An error inside a callee does not abort the caller in a release export (the call returns `null` and execution continues); a debug build attached to a debugger breaks instead, which is acceptable. A runtime script error inside `frame()`'s own two statements is not covered and is a defect, not a path to design for.

`deliver_frame_end()` applies deferred subscription changes and then, for each dirty frame buffer in ascending block index: **if the buffer received a union (a multi-tick frame), it is closed first (native sort, `NO_KEY` at the tail) before `FRAME_MERGED` delivery, so keys are ascending and unique on every delivered batch** (the union appends new keys after the swapped-in sorted ones, so without this step a merged batch would not be ascending and `ChangeBatch.has` as a `bsearch` would silently return wrong answers; HOST/MIRROR batch equality and the tests depend on the canonical order, so it is restored, not weakened). A one-tick frame keeps the O(1) swap and needs no sort. It then delivers the batch to the block's `FRAME_MERGED` subscriptions with `tick` = the last committed tick and `first_tick` = the first, and clears the frame buffers. A zero-tick frame delivers nothing, and still reaches `deliver_frame_end` exactly once.

All keys a block marked in one tick or merged frame are delivered in one `ChangeBatch` per subscription, never split across batches or across `deliver_frame_end` calls, and a presenter drains its `DirtyAccumulator` once per `present` step (ADR-0009), so every key of one Graph step (for example `N/` with `B/`, or a LEAK despawn with its `PATCH_WEEP` spawn) is applied in a single render callback and no frame shows a half-applied step (Failure and Trail FT-40, TR-ftrail-040).

- **Early exit** (a P6 pause, a restore request, a fault): the loop stops, the frame buffer already holds only committed ticks, and `deliver_frame_end()` delivers it after the state change is visible to the driver (KRN-NOT-01: pause at tick 3 of 4 gives the union of ticks 1 to 3 with `tick == 3`; a fault at tick 2 gives tick 1 only). KRN-FRAME-END: a pause at tick 3 of 4, a fault at tick 2, a restore request and a zero-tick frame each reach `deliver_frame_end` exactly once.
- **Paused service cycle** (P1, P2, P4, P5, P6, no P3, `T` unchanged, once per rendered frame while `PAUSED`): its P4 and P5 follow the same rules with `tick = T`; its marks come from `PAUSED_APPLY` scopes and reach `PER_TICK` classes immediately and merged classes at that frame's `deliver_frame_end()`.
- **Restore and new game.** ADR-0003 discards keys marked in `INIT` and `RESTORE` scopes. On the transition `LOADING -> RUNNING` the kernel calls `deliver_all()`: it first clears every tick buffer, frame buffer and event list (the world was replaced; any pending merged union is superseded), then delivers **one `ALL` batch per registered block to every subscription of every class**, with `tick` = the restored `T`, before the first tick of the run. This is the GDD's "one `ALL`, then silence"; owners seed their quantised last-emitted buckets in `post_restore` so no key follows (Graph LS-20, Suit SE-34). Events emitted during `INIT` and `RESTORE` are discarded.
- **No load marker, by design.** `ChangeBatch` carries no load flag: a load-time `is_all` batch cannot be told apart from an overflow `is_all` here on purpose. A presenter that must react differently to a restore reads `PresentationClock.load_seq` (the kernel increments it on every `LOADING -> RUNNING` transition immediately before `deliver_all`, ADR-0001 and ADR-0009).
- **`SAVING`** delivers nothing (snapshot is a read). **`FAULTED`** delivers only what `deliver_frame_end()` still holds from committed ticks; blocks half-applied by the fault are visible through read views without a notification, which is acceptable because the fault screen replaces the play view.
- **Batches for a `PERCEPTION_OUT` block** follow the same delivery; who may subscribe to one is subscribe-time check 6 of § 2.

### 7. The missed-mark audit and the touch serial

The architecture calls for "a debug-build check that every changed block marked a dirty key". A literal check is wrong for blocks whose GDD forbids a key per write: the Graph fires a key only when a quantum bucket changes (LS-20), the Emitter's pose never dirties (EM-26), Web Map marks zero keys per tick during a leg (WM-2). So the audit is built on two separate facts.

**The touch serial (always on, also in release).** The kernel keeps `touch_serial[block]`, a `PackedInt64Array` of `MAX_BLOCKS`, written only by a granted `open_for`. Every successful `scope.open_for(block)` stores the scope's serial into the block's slot (one store; the serial is a kernel counter incremented at each scope arming, so it also advances for same-`T` paused cycles). Because every mutator begins with `open_for` (ADR-0003), "this block may have changed" is derived by the kernel without the owner's help and is independent of whether the owner marked a presentation key. The kernel exposes it as `Kernel.touch_serial(block_index)`. **ADR-0005 § 5.2 adopts it** as the validity input of the per-block hash cache (`BlockHashCache`), because a dirty key cannot serve: a quantised Graph write that crosses no bucket changes the hash and marks no key. A dirty key can therefore never gate caching (ADR-0005 § 5.4: capture is always uncached). The serial is a registry member under the H1 allow-list (registries group, ADR-0003 amendment).

**The audit (runs wherever `SIM_WRITE_TRIPWIRE` runs).** It is gated on the injected `KernelProfile`, not on build names: it runs in the `STRICT_DEBUG` profile and in the test profile, exactly where the tripwire runs (kernel GDD CR-19(e), ADR-0003 § 7). Whether a playtest export carries `STRICT_DEBUG` is **UNVERIFIED** (an export-profile decision, Open Question 9). It reuses the per-block hash the tripwire already takes around P5; it adds no hash pass. It reads the tripwire's **kernel-internal uncached results**, `block_hash_any(block_index) -> int` (ADR-0005 § 5.3, which resolves its OQ-11), not the public `block_hash` API: the public call refuses a block the active scope has opened (`BLOCK_HASH_IN_WRITE`) and a `persist == false` block, which are exactly the blocks the audit needs after P4. A sweep compares each audited block with its previous sweep and applies one rule:

> **`MISSED_DIRTY_MARK`**: the block's hash changed since the previous sweep, `touch_serial` advanced (the block was opened), **no key was marked and no `ALL` was set** in that interval, and the block's key space has `quiet_writes_allowed == false`.

- A block that changed with the serial **not** advanced is a write outside a scope or a leaked mutable reference; that is the existing `SIM_WRITE_TRIPWIRE` and is not duplicated here.
- A key marked for an unchanged value is not a violation (marked, not diffed).
- **Cadence.** In unit, integration and CI soak tests with small worlds the sweep runs **every tick** (`AUDIT_EVERY_TICK`); under `STRICT_DEBUG` it runs at the tripwire's `TRIPWIRE_SWEEP_TICKS` (60) and within `TRIPWIRE_BUDGET_MS` (1.0), so there it is a sample, not a proof. A bool guards the whole path; release exports pay one branch per tick.
- **Quiet blocks** (`quiet_writes_allowed == true`) are not audited by the hash rule, because it cannot distinguish a forgotten bucket-crossing mark from a legal sub-quantum write. **The list is illustrative; each owner's `KeySpaceDescriptor` plus the per-owner key-space CI test is the normative source.** Illustratively: Graph, Emitter, Body, Suit, Controller, Sanity, `web.state` (Web Map marks zero keys per tick during a leg, WM-2; it is quiet unless the Web Map ADR guarantees that no write happens without a key during a leg, otherwise a leg-progress write would raise `MISSED_DIRTY_MARK` on every sweep), and any block with a continuous family. `attention.state` is flagged for ADR-0022 to decide (TR-attention-026: presentation polls it every frame, and it may write without keys). For quiet blocks the guarantee is the per-mutator test of Implementation Guidelines (a fixture runs each mutator in a fresh kernel and asserts that its discrete changes mark a declared kind), plus a per-block diagnostics counter `QUIET_WRITE_TICKS` (ticks on which the block was touched with zero keys), shown in the Debug Overlay so a block that never marks anything is visible.
- **Scope of coverage.** `block_hash_any` hashes every touched block, including `persist == false` derived blocks, so those are audited too (when they are not quiet).
- **What it cannot see.** A mutator that marks the wrong key of a declared kind. The mutator test asserts the specific keys.
- **Release.** Nothing detects a missed mark at run time in a release export; the consequence is a stale display until the next full reconcile (Lighting LO-3 reconciles every `FULL_RECONCILE_TICKS` 60). This is accepted and recorded.

New violation names (counters; each also calls `violation_handler` under `STRICT_DEBUG`): `SUBSCRIBE_REFUSED`, `DIRTY_KEY_INVALID`, `DIRTY_KEY_KIND_UNDECLARED`, `EVENT_OVERFLOW`, `MISSED_DIRTY_MARK`. They extend ADR-0003's closed violation set, which omitted `SUBSCRIBE_REFUSED` although the kernel GDD already requires it.

### Key Interfaces

**Where the types live (source layout, ADR-0010 § 1.1).**

| Type | Path | Who may name it |
|---|---|---|
| `ChangeBatch`, `NotifyEvent`, `NotifyContext`, `CompletionToken`, `SubscriptionHandle`, `SubscriberClass`, `DirtyKey`, `DirtyAccumulator`, `KeyBuf` | `src/kernel/facets/` (`KeyBuf` moves here so `DirtyAccumulator` has no kernel-core edge) | presentation handlers and accumulators (`ChangeBatch`, `NotifyContext`, `CompletionToken`, `DirtyAccumulator`), owners (`DirtyKey`), the kernel |
| `NotifyFacet` | `src/kernel/facets/notify_facet.gd` | `src/game/` (composition root) and `PresentationFacet` only; never held by a presenter |
| `KeySpaceDescriptor` | `src/kernel/` (kernel core) | block owners, through `BlockRegistrar.register_key_space`; ADR-0010 must let owners name this one type (Open Question 11) |
| subscription registry, per-block `KeyBuf` arrays, event pool, `touch_serial` | `src/kernel/` (kernel core, private) | the kernel only |

ADR-0010 narrows ARCH-08 (§ 2.2 rule 4) from "any `*Context`" to an explicit tick-phase deny list, because every presenter handler must name `NotifyContext` and `CompletionToken` (TD-01); see Cross-ADR amendments.

```gdscript
# Composition root (src/game/): the only holder of the kernel-side facet
#   var nf: NotifyFacet = kernel.issue_notify_facet(SubscriberClass.PRESENTATION, false)   # class and perception_access fixed here
#   presentation_facet = PresentationFacet.new(nf, ...)                                    # ADR-0009 class-bound wrapper

# Presentation side (src/presentation/..., any module that is not a block or hook owner) holds PresentationFacet:
#   presentation_facet.subscribe(&"graph.state", _on_graph, &"hud.graph")

class_name NotifyFacet extends RefCounted
var key_capacity: int
func subscribe(block_id: StringName, handler: Callable, name: StringName) -> SubscriptionHandle

class_name SubscriberClass extends RefCounted     # constants only
const PRESENTATION: int = 0
const INPUT_SURFACE: int = 1
const PROJECTOR: int = 2
const REPLICATION: int = 3
const DIAGNOSTIC: int = 4

class_name SubscriptionHandle extends RefCounted
func unsubscribe() -> void                         # idempotent
func is_active() -> bool

# Handler (a method Callable on a long-lived object; last statement return ctx.complete(); never await)
#   func _on_graph(batch: ChangeBatch, ctx: NotifyContext) -> CompletionToken

# Owner side (inside a mutator; see ADR-0003 for open_for)
#   scope.open_for(&"graph.state")
#   scope.mark_dirty(&"graph.state", DirtyKey.make(K_COMP, cid))
#   scope.emit_event(&"body.state", EV_BODY_TRANSFER_WELD, actor_id, seat, old_body, new_body, dx, dy)

# Registration (BOOTING only, with the block descriptor)
#   registrar.register_key_space(KeySpaceDescriptor)

# Driver (ADR-0001): TickDriver.frame() is two statements, _frame_body(...) then, unconditionally,
#   kernel.deliver_frame_end() -> void
# Kernel-internal: deliver_all() on LOADING -> RUNNING; touch_serial(block_index) -> int (ADR-0005 reads it);
#   block_hash_any(block_index) -> int (ADR-0005 § 5.3; only the tripwire and the § 7 audit call it)
```

### Implementation Guidelines
- **Must** deliver change notification only through the kernel registry (`subscribe`); **must never** declare an engine `signal` for kernel changes, `connect` to a kernel object, or use `Signal.emit` to notify a subscriber from `src/kernel/`.
- **Must** register a `KeySpaceDescriptor` for every block, with `quiet_writes_allowed` stated explicitly; **must never** mint a key of an undeclared kind.
- **Must** build keys with `(kind << 32) | id` from integers (`DirtyKey.make` or the same expression); **must never** build a key from a `String`, `StringName`, `float` or `Variant`, or let an `id` exceed 32 bits.
- **Must** mark dirty keys only with `scope.mark_dirty` inside the mutator that changed the data, after `scope.open_for`; **must never** mark from a property setter, from a hook that did not open the block, or outside a scope.
- **Must** mark a key on every discrete change a subscriber could need to wake for (create, remove, structural change, bucket crossing); **may** skip a key for a sub-quantum write only if the block's key space declares `quiet_writes_allowed`.
- **Must** emit P5 events only through `scope.emit_event` with integer arguments; **must never** put a state value, `String`, `Vector2i` or `Dictionary` in an event.
- **Must** give every mutator a unit test that runs it in a fresh kernel fixture and asserts the exact keys it marks (the fixture subscribes a `RecordingSubscriber` and compares the batch); **must** keep `AUDIT_EVERY_TICK` on in every integration and soak test.
- **Must** write subscriber handlers as `(batch, ctx) -> CompletionToken` method Callables on long-lived objects, ending `return ctx.complete()`; **must never** pass a lambda or a `.bind()` result, retain a `ChangeBatch` past the handler, hold or duplicate the kernel's buffer, or call a mutator, `submit` or a scope from a handler.
- **Must never `await`** in a subscriber handler or in anything it calls: the handler would return a coroutine state object (recorded `INCOMPLETE`) and its continuation would resume after P5 holding a poisoned `ChangeBatch` and a re-armed `NotifyContext`. ADR-0003 and ADR-0010 SIM-07 ban `await` only in shared, kernel, sim and data code and ban only `await ... process_frame` in `src/presentation/`, so ADR-0010 is asked to add a PRES rule covering every script registered through `PresenterDescriptor` or calling `.subscribe(` (Cross-ADR amendments).
- **Must** accumulate into a `DirtyAccumulator` and clear it in the render callback after use; **must** treat `is_all` as "re-read everything you use from this block".
- **Must** make a block or hook owner's own methods unsubscribed; if an owner needs presentation-like output it creates a separate object that subscribes (CR-10).
- **Must** keep `TickDriver.frame()` a thin wrapper: `_frame_body(...)` (which may `return` early) then an unconditional `_kernel.deliver_frame_end()`, with exactly one `return` in `frame()` (lint or test, ADR-0001); **must never** deliver a merged class from P5; **must** `close()` (sort) every frame buffer that received a union before delivering it.
- **Must** use `index_of.get(block_id, -1)`, never `index_of[block_id]`, and `make_read_only()` the map at the freeze; **must** bound `ChangeBatch.has` with `count`.
- **Must** treat `touch_serial` as the kernel's "this block may have changed" fact (set only by a granted `open_for`); **should** use it, not dirty keys, wherever a cache must be invalidated by "any write" (ADR-0005 uses it for the hash cache).
- **Must never** allocate in `mark_dirty`, `emit_event`, P4 close, a quiet-tick P5, the deferred subscriber-list rebuild or `deliver_frame_end`; the verification is the `OBJECT_COUNT` plus identity checks of Validation Criteria.
- **Must** keep the audit behind the one profile bool (it runs only where `SIM_WRITE_TRIPWIRE` runs); **must never** let it run a hash pass of its own beyond the tripwire's, and **must** read `block_hash_any`, never the public `block_hash`.

## Alternatives Considered

### Alternative 1: Engine signals for notification
- **Description**: The kernel declares a `changed(batch)` signal (or one per block); subscribers `connect` method Callables; P5 emits.
- **Pros**: The idiomatic Godot mechanism; the engine disconnects freed receivers; cost equals a `Callable.call` loop (probed 157 versus 155 ns per call at 10 subscribers).
- **Cons**: No return value, so the completion-token contract and per-subscriber `INCOMPLETE` cannot exist; no class and no cadence selection; no per-block filter, so every handler is called for every block; silent disconnect hides `DEAD_SUBSCRIBER`; the CR-10 snapshot behaviour and subscribe-time checks have nowhere to live; emission order is the engine's (training knowledge, not relied on).
- **Rejection Reason**: Semantic fit, not cost. The kernel GDD's lean is confirmed (OQ-4), gated on KRN-LAT-02.

### Alternative 2: A `Dictionary` per block as the dirty set
- **Description**: `dirty[block][key] = true`; deliver `dirty[block].keys()`.
- **Pros**: Deduplication for free; short code.
- **Cons**: Allocation on growth and `keys()` allocates a fresh `Array` every delivery (fails KRN-PERF-01's zero-container quiet tick and the active-tick budget); iteration order is not a contract the kernel wants to lean on (CR-13); every key is a boxed `Variant`.
- **Rejection Reason**: Allocation and order. The fixed `PackedInt64Array` with `has` gives the same semantics with neither.

### Alternative 3: String-keyed families (`"C/12"`)
- **Description**: Keys are strings built per mark, as the Graph GDD's "SD alternative" would have.
- **Pros**: Readable in logs; no encoding to learn.
- **Cons**: A `String` is built per key per run (the Graph's reason for choosing integers at VS-A, ADR-LSG-06); string compares in `has`; collisions between families need prefix discipline.
- **Rejection Reason**: Allocation at the Graph's volume. Names are recovered from the key space for diagnostics instead.

### Alternative 4: Automatic dirty tracking (setters, or a hash diff after each mutator)
- **Description**: Property setters mark a block dirty, or the kernel hashes a block before and after every mutator and marks it if different.
- **Pros**: Owners cannot forget a mark.
- **Cons**: 4.7 packed-array element writes call no setter (probed), and the Graph, Layout, Body and Emitter write element-wise; a diff needs a hash per mutator call (CR-12's tripwire is budgeted at two per batch, not one per call) and contradicts "marked, not diffed" for an unchanged write; neither produces a key, only a block-level change, which is the `ALL` cost the key scheme exists to avoid.
- **Rejection Reason**: Cannot work on 4.7 and cannot name what changed.

### Alternative 5: Deliver every tick to every class, no catch-up merge
- **Description**: PRESENTATION also gets each tick's batch in a catch-up frame.
- **Pros**: Simplest semantics.
- **Cons**: A four-tick frame pays P5 four times for a reader that only ever draws the last committed state; ratified against in R-12.
- **Rejection Reason**: Ruled out by the user (R-12). Per-tick delivery stays for INPUT_SURFACE and REPLICATION.

### Alternative 6: An open-addressed hash set instead of `has()` scanning
- **Description**: Per block, a power-of-two `PackedInt64Array` table with linear probing plus an insertion list; `mark` is O(1) expected.
- **Pros**: Constant-time marks regardless of capacity; matters if `MAX_DIRTY_KEYS` is raised toward 4096.
- **Cons**: More code, a hash that must avoid 64-bit overflow, a larger table to clear; at the default 256 the scan costs 160 to 230 ns per mark (probed), and a worst-case 256-distinct tick 52 us, well under the proposal.
- **Rejection Reason**: Not needed at 256. **Kept as the documented fallback** if KRN-LAT-02 shows the `has()` scan is a problem on a release export; the `KeyBuf` interface does not change.

### Alternative 7: Block-less `subscribe(class, handler)` receiving every block
- **Description**: The architecture sketch's two-argument form; each handler filters by `batch.block_id`.
- **Pros**: Fewer registrations.
- **Cons**: A subscriber that cares about two blocks is called for up to 128; the cost scales with subscribers times dirty blocks; Layout LY-32 already assumes per-block subscription.
- **Rejection Reason**: Wasted calls; the wildcard survives for `DIAGNOSTIC` only.

### Alternative 8: Copy the subscriber list at the start of every P5
- **Description**: The literal CR-10 snapshot.
- **Pros**: Obviously correct.
- **Cons**: One array allocation per tick for a list that almost never changes.
- **Rejection Reason**: Deferred application gives identical observable behaviour with no per-tick allocation (§ 2).

### Alternative 9: A `HOST_SERVICE` subscriber class for Save/Load
- **Description**: A sixth class for host services that need events (autosave triggers).
- **Pros**: Matches how Save/Load actually consumes `POD_ARRIVED` and `BERTHED`.
- **Cons**: CR-10's table is closed; the architecture already has Save/Load poll owner read views (`last_shape_change_T`, `last_salvage_T`). Polling a level flag once per frame would miss an arrival followed by a departure inside one catch-up frame, but a monotonic counter does not.
- **Rejection Reason**: Rejected outright: no sixth class, and no control-plane end-of-tick callback. Save/Load polls owner read views (ADR-0006 § 1.12), observing monotonic counters such as `web.view.pod_arrived_seq` and `berthed_seq` that cannot be lost to frame merging; the counters are a requirement on the Web Map ADR, owned and recorded by ADR-0006 (TD-09).

## Consequences

### Positive
- One notification path with the token contract, the two cadences, the owner rule and the lifetime rules in one place, testable with stub subscribers and no scene.
- Marking is a native scan of a fixed array: no allocation, exact collapse, canonical sorted batches.
- `ALL` is a flag, so no key value is reserved and the accumulator logic is one union function every presentation system shares.
- The touch serial offers ADR-0005 a sound invalidation signal even for quantised blocks, decoupled from presentation keys.
- Events share the block's batch, so audio and other presentation consumers need no second subscription and survive a catch-up frame; Save/Load's autosave triggers use monotonic counters in owner read views (ADR-0006 § 1.12), which also survive frame merging.
- A forgotten mark is caught in every integration test for non-quiet blocks, and every mutator has a test that names its keys.

### Negative
- Every presentation handler is `(batch, ctx) -> CompletionToken`, not the one-argument `void` of the architecture sketch.
- Manual marking remains a discipline; quiet blocks (the largest ones) get a test-level guarantee, not a runtime one, and a wrong key of a declared kind is invisible to the audit.
- The mark cost is O(capacity) per distinct key (native scan); raising `MAX_DIRTY_KEYS` toward 4096 makes the fallback of Alternative 6 necessary.
- `TickDriver.frame` becomes a wrapper and ADR-0003 gains `emit_event`, `register_key_space`, five violation names and the `touch_serial` registry member; ADR-0001, 0003, 0009 and 0010 must be amended in the same pass (Cross-ADR amendments) and acceptance depends on it.
- A block must declare its key space even if it never marks a key.
- Release builds detect no missed marks.
- Binding the subscriber class at issue needs the composition root to build one facet per class and ADR-0009 to wrap it (`PresentationFacet`); presenters cannot choose their class, which is the point.

## Risks
- **A mutator forgets `scope.mark_dirty` on a quiet block.** The audit cannot see it. Mitigation: the per-mutator test with exact keys, `QUIET_WRITE_TICKS` in the overlay, Lighting's `FULL_RECONCILE_TICKS`, and review against the key space table.
- **`has()` is slower than probed on a release template.** Mitigation: KRN-LAT-02 measures the 256-distinct worst case; Alternative 6 is the drop-in.
- **A subscriber retains a `ChangeBatch` or a key array.** Mitigation: the batch is poisoned under `STRICT_DEBUG`; no array is exposed; review against the guideline; ADR-0010 decides what a lint can catch.
- **A lambda subscriber slips in.** Mitigation: `is_standard()` check at subscribe time, counted and refused (probed: lambdas and `.bind()` are custom).
- **Merged delivery is skipped on an exit path.** Mitigation: `frame()` is a two-statement wrapper whose second statement is unconditional, so an early `return` in `_frame_body` cannot skip it; one `return` in `frame()` is linted; KRN-FRAME-END and KRN-NOT-01 cover pause, fault, restore and zero-tick exits.
- **A pending merged union survives a restore.** Mitigation: `deliver_all` clears every buffer before delivering.
- **The subscription list rebuild is slow with many pending requests.** Mitigation: rebuild only on a pending request, at most `MAX_SUBSCRIPTIONS` 96 entries; none in steady state.
- **Event overflow loses an event.** Mitigation: 64 events per tick against a handful designed per leg; counted, and a violation under `STRICT_DEBUG`. Save/Load no longer depends on events for autosave triggers (it polls monotonic counters, ADR-0006 § 1.12), so an overflow cannot lose one.
- **A release subscriber aborts on a script error.** Same open point as ADR-0003 Verification 1: the isolation is best-effort plus the token check.
- **The Graph's composite loop/anchor id overflows 32 bits.** Mitigation: the owner packs and range-checks (ADR-0011).
- **A handler `await`s.** The kernel records `INCOMPLETE` and the continuation later holds a poisoned batch. Mitigation: no `await` in subscriber handlers (Implementation Guidelines), the ADR-0010 PRES lint, the token identity check as runtime backstop.
- **A merged batch is not sorted.** The union appends after the swapped-in sorted keys. Mitigation: every frame buffer that received a union is closed (sorted) in `deliver_frame_end` (§ 6); the validation case keys 9, 3, 5 delivers `[3, 5, 9]` and `has(5)` is true.
- **A lowered `MAX_DIRTY_KEYS` has no effect.** Mitigation: the active cap is a field separate from the allocated capacity and is latched into every buffer at P0 (§ 3, § 4); the validation case lowers it and checks the collapse point.
- **A presenter reads dirty-key ids of a `PERCEPTION_OUT` block.** Mitigation: subscribe-time check 6 refuses it unless the facet carries `perception_access`.

## GDD Requirements Addressed

| GDD System | Requirement | How This ADR Addresses It |
|------------|-------------|--------------------------|
| command-simulation-layer.md | CR-10 / TR-cmd-023: notifications once per tick at P5 as (block, tick, change_keys), no state payload, collapse to `ALL` past `MAX_DIRTY_KEYS` 256; five subscriber classes; an object owning a block or hook never subscribes | §§ 1, 2, 4, 5, 6 |
| command-simulation-layer.md | CR-10 catch-up delivery and subscriber lifetime / TR-cmd-024: merged PRESENTATION, PROJECTOR, DIAGNOSTIC batches; INPUT_SURFACE and REPLICATION every tick; snapshot of the list, pruning of freed subscribers | § 2 (deferred application, `DEAD_SUBSCRIBER`), § 6 |
| command-simulation-layer.md | Open Question 8 and architecture OQ-4: transport (registry versus signals) | § 1; gated by KRN-LAT-02 |
| command-simulation-layer.md | D.9: `change_keys = dirty if <= MAX else ALL`; exactly 256 listed, 257 give `ALL`; knob change takes effect next tick | §§ 3, 4 (P0 latch of the active `cap`, exact capacity) |
| command-simulation-layer.md | KRN-NOT-01, KRN-NOT-02, KRN-NOT-03, KRN-PERF-01 (acceptance criteria without a TR id) | §§ 2, 6; Validation Criteria |
| life-support-graph.md | LS-20 / TR-lsg-020 and ADR-LSG-06: int keys `(KIND << 32) \| id`, no String per key, `MAX_DIRTY_KEYS` never below 128 | § 3 (layout, kinds 1 to 9, floor 128); the quantisation rules stay with the Graph (owned elsewhere table) |
| ship-section-and-layout-data-model.md | LY-27, LY-32 / TR-section-022: explicit marks inside each mutator, int keys SEC 1, BODY 2, SEAM 3, batched to the PROJECTOR, `ALL` past 256 | §§ 3, 4, 6 |
| ship-body-and-interior-physics.md | BDY-21 / TR-shipbody-022: Layout's encoding (BODY 2, ACT 4), explicit marks, per-block per-tick keys under 256 | §§ 3, 4 |
| ship-body-and-interior-physics.md | BDY-28 / TR-shipbody-027: `BODY_DOCK_CONTACT`, `BODY_HULL_SEPARATION`, `BODY_TRANSFER` published as P5 events that carry no simulation state | § 5 (`NotifyEvent`, `emit_event`) |
| suit-and-equipment.md | SE-34, SE-38 / TR-suit-013: explicit `S/<PlayerId>` keys under 256, manual because of the 4.7 rule, a restore yields one `ALL` | §§ 3, 4, 6 (`deliver_all`) |
| web-map-and-transit.md | WM-2 / TR-webmap-024: explicit `WEB/<derelict_id>`, `WEB/T`, `WEB/POD`; no write relies on a setter; zero keys per tick during a leg | §§ 3, 4 (singleton ids, explicit marks); the "zero keys" budget is the owner's |
| web-map-and-transit.md | WM-31 / TR-webmap-023: `POD_DEPARTED`, `POD_ARRIVED`, `DERELICT_LOADED`/`UNLOADED`, `BERTHED`/`UNBERTHED`, `FIRST_BOARDED` emitted in the P5 batch | § 5 (events ride the block's batch, for presentation consumers). Attention and Horror as consumers conflict with CR-10: Open Question 2. The Save/Load half (autosave on `POD_ARRIVED` and `BERTHED`) is ADR-0006's: it polls `web.state` counters (Alternative 9), and `/architecture-review` re-tags it to ADR-0006 |
| failure-and-trail-system.md | FT-15 / TR-ftrail-007 (union rule only): store and union key sets since the last callback; a union above 256 or an `ALL` is `ALL` | § 5 (`DirtyAccumulator`) |
| failure-and-trail-system.md | FT-40 / TR-ftrail-040: all dirty keys of one Graph step are applied in a single render callback, so no frame shows a half-applied step | § 6 (atomic batch delivery, single `present` drain) |

**Loaded or checked here; the rule itself is owned elsewhere.** These baseline rows cite ADR-0004, but the behaviour is decided by the named owner. This ADR supplies only the transport and the marking mechanism. `/architecture-review` should re-tag them to the owner so no story treats them as decided here.

| TR | Rule | Owner of the rule | What this ADR contributes |
|---|---|---|---|
| TR-emitter-026 | Which Emitter events fire a key (`E/<id>`: create, remove, switch, shape, output bucket `OUTPUT_DIRTY_QUANTUM_PM` 50, reach step); pose and subject lux never dirty | Emitter GDD (EM-26), ADR-0017 | key layout, explicit marking, `quiet_writes_allowed = true` for `emitter.state` |
| TR-light-007 | Lighting wakes on `E/<id>`, polls pose at render rate, three cadences, geometry from exact committed values | Lighting GDD (LO-3), ADR-0009, ADR-0028 | batch delivery to a PRESENTATION subscriber, `DirtyAccumulator` |
| TR-attention-026 | Attention marks `ATT/L` and `ATT/P` explicitly; presentation polls `attention.view` every frame rather than treating keys as events | Attention GDD (AT-1), ADR-0022, ADR-0009 | the marking mechanism (§ 4); the polling discipline is not decided here |
| TR-lsg-020 (quantisation part) | Quanta T 1000 mC, P 10 dkPa, p_o2 5 dkPa, hp 250 bp; one `C/<cid>` per compartment per step; worst cascade 38 keys | Graph GDD (LS-20, D.10), ADR-0011, ADR-0012 | the encoding and the 128 floor only |
| TR-ftrail-007 (handler discipline part) | A handler only stores and returns; reads and draws happen in the next render callback | Failure & Trail GDD (FT-15), ADR-0009 | the handler signature and the accumulator |
| TR-webmap-023 (Save/Load half) | Autosave on `POD_ARRIVED` and `BERTHED` | ADR-0006 § 1.12 (polls `web.state` counters), Web Map ADR | nothing beyond the events-in-batch half, which rides § 5 |

## Performance Implications
- **CPU.** All figures are **proposals pending KRN-LAT-02 on a release export (RM-DESKTOP, RM-DECK)**; the kernel GDD defines `t_P5` but no budget. Proposed: **P5 plus frame-end delivery <= 0.25 ms P99** on the stress workload (12 dirty blocks, 40 subscriptions, a 38-key Graph cascade, four events), inside the 4.0 ms one-tick simulation frame. Indicative editor-build costs (probed): 160 to 230 ns per mark including amortised sort and clear (38 distinct keys: 8.8 us per tick; the 256-distinct worst case: 52 us per tick); about 160 ns per `Callable.call` plus context re-arm per delivery; a quiet tick is one scan of at most 128 flag bytes and zero calls. A one-tick frame swaps buffers instead of unioning.
- **Full-capacity sort and scan (measurement target).** `buf.sort()` and `buf.has()` run over the full capacity (256 to 4096), not over `count`; for a 3-key block that is a 256-element sort per dirty block per tick, and 12 dirty blocks is plausibly tens of microseconds on a release template, noticeable against the 0.25 ms proposal (UNVERIFIED). KRN-LAT-02 records it. If it fails: bound the sort by `count` (an in-place insertion sort over the first `count` elements when `count <= 16`, the native sort above that; no allocation either way), or adopt Alternative 6.
- **Memory.** Two `KeyBuf`s per block at 256 keys of 8 bytes: 4 KiB per block, 128 KiB for 32 blocks (512 KiB for 128 blocks, the `MAX_BLOCKS` ceiling). One `ChangeBatch` per block per cadence; the global event pool (`MAX_TICKS_PER_FRAME` x 64 `NotifyEvent`s) and per-block fixed `PackedInt32Array` event index lists (only for blocks that declare event ids); at most 96 subscriptions. The registry and pools are allocated once at the freeze.
- **Load Time.** Buffer and pool allocation at the freeze; `deliver_all` once per load, one handler call per subscription.
- **Network.** None at MVP. REPLICATION receives per-tick batches in VS-B (ADR-0032); keys carry no state, so the replication shape does not change the encoding.

## Migration Plan
No code exists. Document changes (the sync is not applied by this ADR; see § GDD sync and Cross-ADR amendments):
- `docs/architecture/architecture.md`: the `subscribe` signature (L441), the subscriber example (L516-526: handler returns `CompletionToken`, accumulates into a `DirtyAccumulator`), the Data Flow 4 sentence "decision for ADR-0004" (L351), OQ-4, and the "Interface types still to define" row for `ChangeBatch` (L549: also `NotifyEvent`, `DirtyAccumulator`, `KeySpaceDescriptor`).
- `design/gdd/command-simulation-layer.md`: CR-10 (transport decided; per-block `subscribe`), Open Question 8, D.12 wording, the AC at ~L1014.
- `design/gdd/life-support-graph.md` LS-20: remove the bracketed "[UNV that the kernel's key type is Variant...]".
- `design/gdd/web-map-and-transit.md` WM-31: sim-side consumers read the view; the Tuning Knobs table of the kernel GDD gains the three new knobs.
- ADR-0001, ADR-0003: the amendments listed under Cross-ADR amendments.

## Validation Criteria
- **KRN-LAT-02 (gates acceptance for cost).** On a release export of the headless kernel skeleton, P5 plus frame-end delivery meets the proposal on the stress workload; the 256-distinct `has()` worst case, the full-capacity sort and scan cost per dirty block, and the registry-versus-signal baseline are recorded (the baseline is information only). The same spike settles `Callable.get_argument_count()` on a release template.
- Marks `a, b, b` in one tick give one batch `(B, T, {a, b})`; a block with no marks produces none; a write of an unchanged value still lists its key; the batch carries only `block_id`, ticks, keys and events.
- Exactly 256 distinct keys are listed and sibling block `C` with 3 keys is unaffected; 257 give `is_all` for `B` only; the same key marked 1,000 times is one key.
- **The knob is live.** A `MAX_DIRTY_KEYS` change to 128 during P2 of tick `T` leaves `T`'s P5 on the old value and `T+1` on the new: exactly 128 distinct keys are listed and the 129th gives `is_all` in a tick buffer and in the § 6 fold; a frame buffer holding 200 keys when the cap drops to 128 collapses to `is_all`; a `DirtyAccumulator` keeps the sized capacity; a raise above the sized capacity is `KNOB_REFUSED`.
- A four-tick catch-up frame in which `B` marks `a` in tick 1 and `b` in tick 3 delivers one batch `(B, T_last, {a, b})` with `first_tick` 1 to a PRESENTATION subscriber, four batches to an INPUT_SURFACE subscriber, and two blocks of 200 distinct keys each in different ticks merge to `is_all`. Events of all four ticks arrive, in order, in the merged batch.
- **Merged order.** A 4-tick frame with keys 9 (tick 1), 3 (tick 3) and 5 (tick 4) delivers `[3, 5, 9]` and `batch.has(5)` is true; `has(100)` and a `NO_KEY` probe are false (`bsearch` returns an insertion index, never -1, and `has` is bounded by `count`); a one-tick frame delivers by swap with no sort.
- KRN-NOT-01: a pause at tick 3 of 4 gives the union of ticks 1 to 3 with `tick == 3`; a fault at tick 2 gives tick 1's keys only; a frame ending in a restore delivers `ALL` for every block to every class after the kernel is `RUNNING`, and no pending merged union survives it.
- **KRN-FRAME-END** (ADR-0001 owns the driver): a pause at tick 3 of 4, a fault at tick 2, a restore request and a zero-tick frame each reach `deliver_frame_end` exactly once; `frame()` contains exactly one `return`.
- KRN-NOT-02: a freed subscriber is pruned, `DEAD_SUBSCRIBER` +1, no engine error, no `INCOMPLETE`; `unsubscribe()` stops delivery; a handler that unsubscribes a peer during P5 leaves the peer receiving this batch and none after; a handler that subscribes affects the next boundary only.
- KRN-NOT-03: after `dispose()` twice, registries are empty and the kernel's instance count (by `weakref`) is 0.
- Subscribe refusals each count `SUBSCRIBE_REFUSED` and fault nothing: a lambda, a `.bind()` result, an invalid Callable, a wrong arity, a block or hook owner's own method (while a separate input-adapter object of the same system is accepted), `DIAGNOSTIC` without the dev predicate, `REPLICATION` at MVP, an unknown block after the freeze, the 97th subscription; an unknown block before the freeze fails boot with `UNKNOWN_DEPENDENCY`; an owner's own method found at the freeze is a boot failure with code `SUBSCRIBE_REFUSED` under `STRICT_DEBUG` and a counted removal under `LENIENT_RELEASE`.
- **Check 6.** A `PRESENTATION` subscription to `sanity.perception` through a facet issued with `perception_access = false` is refused with `SUBSCRIBE_REFUSED`; through the Perception Filter's facet and through a `DIAGNOSTIC` facet it is accepted; a presenter cannot obtain a facet of another class (the class is fixed at issue).
- **INCOMPLETE tests use benign handlers.** The handler returns `null` or a stale token from an earlier delivery (recorded `INCOMPLETE` against its name while the other subscribers receive the batch and the kernel stays `RUNNING`). Tests must **not** use a handler that raises a real runtime script error: the project's gdUnit4 command runs with `-d`, where a script error drops to a `debug>` prompt and can hang CI. The real-error case is a manual or release-export check, as in ADR-0003 Verification 1. A handler that calls a mutator writes nothing and counts `WRITE_OUTSIDE_SCOPE` once.
- KRN-PERF-01: a quiet tick with 32 blocks and 10 subscriptions performs zero calls, zero kernel container allocations; **zero allocation is tested two ways** (as in ADR-0007, because `OBJECT_COUNT` misses arrays): `OBJECT_COUNT` is flat over 600 ticks that mark, deliver and clear, and the same `ChangeBatch` and `NotifyEvent` objects (`is_same`) are returned on consecutive ticks. A quiet tick that applies a `DEAD_SUBSCRIBER` prune (the deferred rebuild) also allocates nothing.
- Keys: `DirtyKey.make(kind, id)` round-trips through `kind_of` and `id_of` for kind 1 and `0x7FFF` and id 0 and `0xFFFFFFFF`; a key of 0, a negative key and `NO_KEY` are refused with `DIRTY_KEY_INVALID`; an undeclared kind under `STRICT_DEBUG` is `DIRTY_KEY_KIND_UNDECLARED`; `mark_dirty` for an unknown block id counts `WRITE_OUTSIDE_SCOPE` without a script error.
- `mark_dirty` and `emit_event` in `INIT` and `RESTORE` scopes record the touch serial and discard keys and events; a new game delivers exactly one `ALL` per block and nothing else on `LOADING -> RUNNING`.
- Events: 64 per tick (global across blocks) are delivered, the 65th is dropped with `EVENT_OVERFLOW`; an event with no key makes a batch with `count` 0 and `event_count` 1; the per-frame limit is `MAX_TICKS_PER_FRAME` x 64.
- Audit: a fixture mutator on a non-quiet block that changes data and marks nothing yields `MISSED_DIRTY_MARK` with `AUDIT_EVERY_TICK` in the test profile and under `STRICT_DEBUG`; the same mutator on a `quiet_writes_allowed` block does not; a write of an unchanged value does not; a fixture that changes a block without `open_for` is the tripwire's, not `MISSED_DIRTY_MARK`; a touched `persist == false` block is hashed through `block_hash_any`; the audit performs no hash pass beyond the tripwire's and never calls the public `block_hash`.
- `touch_serial` advances for every `open_for` grant, including a same-`T` paused cycle, and does not advance for a refused `open_for`.
- Each registered block's key space is consistent with the GDD tables of § 3 (a CI test per owner).

## GDD sync
Items below are returned to the orchestrator and not applied by this ADR.
- `docs/architecture/architecture.md` L441: `func subscribe(cls: SubscriberClass, handler: Callable) -> SubscriptionHandle` becomes `func subscribe(block_id: StringName, handler: Callable, name: StringName) -> SubscriptionHandle` on a class-bound facet (`NotifyFacet` held by the composition root; presentation holds ADR-0009's `PresentationFacet`).
- `docs/architecture/architecture.md` L516-526: the example handler `-> void` with `_dirty.merge(batch.change_keys)` becomes `(batch: ChangeBatch, ctx: NotifyContext) -> CompletionToken` with `_dirty.absorb(batch)` and `return ctx.complete()` (a `Dictionary.merge` of a packed array was never valid).
- `docs/architecture/architecture.md` L351 and OQ-4: "(decision for ADR-0004)" becomes "(ADR-0004, Proposed)"; OQ-4 resolution recorded, acceptance gated on KRN-LAT-02. L549: the `ChangeBatch` row lists the types defined here.
- `design/gdd/command-simulation-layer.md` CR-10: "to ADR for transport (signals vs registry)" becomes ADR-0004's decision; `subscribe(class, handler)` becomes per-block on a class-bound facet; Open Question 8 resolved.
- `design/gdd/command-simulation-layer.md` D.12 (**unconditional**: ADR-0005 § 5.2 adopts the touch serial): old text "cached per block and invalidated by the block's dirty bit** (any write in P2/P3 clears it; `restore` clears it)", new text "invalidated by the block's touch serial (every granted `open_for`; `restore` also advances it), not by a dirty key, because quantised blocks write without marking".
- `design/gdd/command-simulation-layer.md` Tuning Knobs: add `MAX_SUBSCRIPTIONS` (96, range 32 to 256, Boot), `MAX_EVENTS_PER_TICK` (64, range 16 to 256, Boot, global across blocks) and the build flag `AUDIT_EVERY_TICK` (test profile), each with its safe range and failure modes, as `MAX_SCHEDULE_CYCLE_TICKS` was added by ADR-0003; `MAX_DIRTY_KEYS`: runtime changes may only lower it (the active cap is latched at P0; raising above the sized capacity is `KNOB_REFUSED`).
- `design/gdd/command-simulation-layer.md` AC ~L1014 (subscribe returns a submit handle only to `INPUT_SURFACE`): conflicts with registry `presentation_submit_without_handle` (ADR-0002: subscribing confers nothing). The registry wins; the AC is reworded to "subscribing as `INPUT_SURFACE` confers no submit right; the handle is issued separately".
- `design/gdd/life-support-graph.md` LS-20: strike "[UNV that the kernel's key type is Variant; if it is String, ...]"; the kernel key is a 64-bit `int`.
- `design/gdd/web-map-and-transit.md` WM-31: Attention and Horror are listed as event consumers; as block and hook owners they cannot subscribe (CR-10), so they read `web.state`'s view in their hooks. (`horror-entities.md` ~L369 "load and unload events" was not read for this ADR; see Open Question 2.)
- `docs/architecture/tr-baseline.md` TR-webmap-023: re-tag the Save/Load half (autosave on `POD_ARRIVED` and `BERTHED`) to ADR-0006; ADR-0004 keeps only the events-in-batch half.

## Cross-ADR amendments
- **ADR-0001** (all kernel-driver amendments in one batch, TD-04): `TickDriver.frame()` becomes a two-statement wrapper, `_frame_body(...)` (which may return early on pause, restore request or fault) then an unconditional `_kernel.deliver_frame_end()`, with exactly one `return` in `frame()` (lint or test); test KRN-FRAME-END (a pause at tick 3 of 4, a fault at tick 2, a restore request and a zero-tick frame each reach `deliver_frame_end` exactly once). Key Interfaces: the composition root issues `NotifyFacet`, binding the subscriber class at issue (`Kernel.issue_notify_facet(cls, perception_access)`, § 2). `PresentationClock` gains read-only `state: int` and `load_seq: int` (incremented on every `LOADING -> RUNNING` transition immediately before `deliver_all`, never on `PAUSED`/`SAVING -> RUNNING`); `ProcessPriority` gains `HOST_SERVICE = -40` (used by SaveHost).
- **ADR-0003** (all registry, WriteScope, facet and view amendments in one batch, TD-07): `BlockRegistrar.register_key_space(desc: KeySpaceDescriptor) -> void` (BOOTING only, § 3); `WriteScope.emit_event(block_id, kind, a0..a5)`; `WriteScope.has_opened(block_index)` (ADR-0005 § 5.3); the violation table gains `SUBSCRIBE_REFUSED`, `DIRTY_KEY_INVALID`, `DIRTY_KEY_KIND_UNDECLARED`, `EVENT_OVERFLOW`, `MISSED_DIRTY_MARK`, and `UNKNOWN_DEPENDENCY` also covers an unknown subscription block; the H1 registries group adds `touch_serial` (this ADR) and the hash-cache arrays (ADR-0005 § 5.2); the § 4 "hash each block before and after a tick and compare" sentence becomes a pointer to this ADR's § 7 and the opaque-int comment at line 162 becomes "key layout: ADR-0004 § 3"; `NotifyContext` is placed in `src/kernel/facets/`; § 7 `PresentationViews` has two faces selected by a `perception_access` flag the composition root sets at issue (default face returns `empty_view` for `PERCEPTION_OUT` and counts `PERCEPTION_READ_BLOCKED`; the perception face returns the real facade), which resolves the contradiction with ADR-0009 § 1 (TD-08).
- **ADR-0005**: confirms `touch_serial` as the cache validity input (§ 5.2) and `block_hash_any` as the audit and tripwire path (§ 5.3, OQ-11); fixes its Depends On attribution (touch_serial to ADR-0004). No change otherwise.
- **ADR-0006**: § 1.12 adds the `web.state` read view with `pod_arrived_seq` and `berthed_seq` as conditional members (a requirement on the Web Map ADR), compared by `SaveService.poll_frame` against host-local last-seen copies re-read after a load; ADR-0006 closes its OQ-F and owns the Save/Load half of TR-webmap-023 (TD-09, TD-24).
- **ADR-0009**: adds ADR-0004 to its Depends On; § 1 declares `perception_access = true` only on the Perception Filter's `PresentationFacet` and on `DIAGNOSTIC` facets and `false` on every other (TD-08); its handler rule cites ADR-0010's `await` rule; it keeps `PresentationClock.load_seq` as the restore marker.
- **ADR-0010** (TD-01, TD-16, TD-27): § 2.2 rule 4 and ARCH-08 replace "any `*Context`" with the explicit deny list `Kernel, SimViews, WriteScope, Command, BlockDescriptor, CallContext, ValidateContext, ApplyContext, HookContext, SnapshotContext`; § 1.1 `src/kernel/facets/` lists `PresentationClock, SurfaceHandle, ControlHandle, ModuleFacet, BlockRegistrar, ChangeBatch, NotifyContext, CompletionToken, NotifyEvent, DirtyAccumulator, KeyBuf, SubscriptionHandle, SubscriberClass, DirtyKey, PresentationFacet, PresentationViews` (mirrored in edges.json); `NotifyFacet` may be named only by `src/game/` and by `PresentationFacet`; SIM-13 becomes an active rule citing § 1 of this ADR; a PRES rule bans `await` in any script registered through `PresenterDescriptor` or calling `.subscribe(` (with a bad fixture); it must let block owners name `KeySpaceDescriptor` (Open Question 11); ADR-0010 lists ADR-0004 under Enables only, and ADR-0004 lists ADR-0010 under Depends On as a soft dependency. *(2026-10-29: `NotifyFacet` removed from this list; it is named only by `src/game/` and inside `PresentationFacet`, ADR-0010 § 1.2.)*

## Registry Candidates
Reviewable candidates for `docs/registry/architecture.yaml`, copied from the run file and corrected by the 2026-10-28 rulings (TD-33). **Not applied by this ADR**; the orchestrator merges them serially once the revisions land. The registry holds no ADR-0004 entry yet, so all are additions.

### interfaces

`change_notification`

```yaml
  - contract: change_notification
    status: active           # ADR-0004 is Proposed
    pattern: direct_call     # kernel-owned observer registry invoking method Callables; not engine signals
    producer: command-simulation-layer (kernel P5 and frame-end delivery)
    consumers:
      - presentation-subscribers
      - input-surface-subscribers
      - projector-subscribers
      - diagnostic-subscribers
      - replication-subscriber (VS-B)
    adr: docs/architecture/adr-0004-change-notification-and-dirty-keys.md
    signal_signature: "NotifyFacet.subscribe(block_id: StringName, handler: Callable, name: StringName) -> SubscriptionHandle (subscriber class and perception_access fixed when the kernel issues the facet to the composition root via Kernel.issue_notify_facet(cls, perception_access); presentation holds only ADR-0009's class-bound PresentationFacet); handler(batch: ChangeBatch, ctx: NotifyContext) -> CompletionToken with return ctx.complete() last and no await; ChangeBatch {block_id, tick, first_tick, is_all, count, event_count, key(i), has(key), event(i)}, keys ascending and unique on every delivered batch; DirtyAccumulator.absorb(batch); types live in src/kernel/facets/; TickDriver.frame() = _frame_body() then unconditional Kernel.deliver_frame_end()"
    referenced_by:
      - docs/architecture/adr-0004-change-notification-and-dirty-keys.md
    added: 2026-10-28
    revised: ""
```

`dirty_key_marking`

```yaml
  - contract: dirty_key_marking
    status: active           # ADR-0004 is Proposed
    pattern: direct_call
    producer: block-owners (inside a mutator, after scope.open_for)
    consumers:
      - command-simulation-layer
    adr: docs/architecture/adr-0004-change-notification-and-dirty-keys.md
    signal_signature: "scope.mark_dirty(block_id: StringName, key: int) -> void; scope.emit_event(block_id: StringName, kind: int, a0..a5: int = 0) -> void; BlockRegistrar.register_key_space(desc: KeySpaceDescriptor) -> void (BOOTING only; kind_ids, kind_names, event_ids, event_names, quiet_writes_allowed); keys and events marked in INIT and RESTORE scopes are discarded"
    referenced_by:
      - docs/architecture/adr-0004-change-notification-and-dirty-keys.md
      - docs/architecture/adr-0003-state-block-registry-and-owner-contract.md
    added: 2026-10-28
    revised: ""
```

### api_decisions

`notification_transport`

```yaml
  - purpose: notification_transport
    status: active           # ADR-0004 is Proposed; KRN-LAT-02 notify cost gates acceptance
    api: "kernel-owned subscription registry; per-block subscriptions; Callable.call(batch, ctx) on method Callables guarded by is_valid(); subscribe and unsubscribe applied at the next delivery boundary into preallocated arrays; subscriber class bound when the facet is issued"
    not: "engine signal or Signal.emit for kernel to subscriber notification; connect() on a kernel object; lambda or .bind() handlers; await in a handler; a per-tick copy of the subscriber list; a subscriber choosing its own class"
    adr: docs/architecture/adr-0004-change-notification-and-dirty-keys.md
    reason: "The handler must return the completion token (INCOMPLETE per subscriber), delivery has two cadences by class, DEAD_SUBSCRIBER must be counted, and per-block filtering keeps a quiet tick at zero calls; cost does not separate the options (probed 2026-10-28, 155 vs 157 ns per call)"
    referenced_by:
      - docs/architecture/adr-0004-change-notification-and-dirty-keys.md
    added: 2026-10-28
    revised: ""
```

`dirty_key_encoding`

```yaml
  - purpose: dirty_key_encoding
    status: active           # ADR-0004 is Proposed
    api: "int key = (KIND << 32) | id, KIND 1..0x7FFF declared per block (bits 32..46 used, 47..62 reserved), id 0..0xFFFFFFFF; PackedInt64Array buffers filled with NO_KEY = 0x7FFFFFFFFFFFFFFF; ALL is the batch flag is_all, never a key value; buffers allocated at the MAX_DIRTY_KEYS value of the registry freeze, the active collapse cap (KeyBuf.cap) latched at P0 and lowered only; ChangeBatch.has = bsearch bounded by count"
    not: "String, StringName, float or Variant keys; PackedInt32Array key buffers; a Dictionary used as the dirty set; ALL as a reserved key value; raising MAX_DIRTY_KEYS above the sized capacity at run time; testing count against buf.size() instead of cap"
    adr: docs/architecture/adr-0004-change-notification-and-dirty-keys.md
    reason: "Every owner GDD already writes (KIND << 32) | id; 47 bits need 64-bit storage; a native has() scan on a preallocated array dedupes without allocating (probed 2026-10-28: 160-230 ns per mark, 52 us for 256 distinct keys, OBJECT_COUNT flat)"
    referenced_by:
      - docs/architecture/adr-0004-change-notification-and-dirty-keys.md
    added: 2026-10-28
    revised: ""
```

### state_ownership

`touch_serial`

```yaml
  - state: touch_serial
    status: active           # ADR-0004 is Proposed
    owner_system: command-simulation-layer (kernel registries group, H1 allow-list)
    adr: docs/architecture/adr-0004-change-notification-and-dirty-keys.md
    interface: "touch_serial[block] = scope serial, written only by a granted scope.open_for; read through Kernel.touch_serial(block_index) by ADR-0005's BlockHashCache for cache validity"
    write_access: kernel-only (written only by a granted open_for; never by a mutator directly)
    referenced_by:
      - docs/architecture/adr-0004-change-notification-and-dirty-keys.md
      - docs/architecture/adr-0005-determinism-keyed-rng-canonical-encoding-and-hashing.md
    added: 2026-10-28
    revised: ""
```

`notification_buffers`

```yaml
  - state: notification_buffers
    status: active           # ADR-0004 is Proposed
    owner_system: command-simulation-layer (kernel)
    adr: docs/architecture/adr-0004-change-notification-and-dirty-keys.md
    interface: "per-block tick and frame KeyBufs, the global NotifyEvent pool with per-block PackedInt32Array index lists, and the subscription table with per-block subscriber index arrays are kernel-internal (H1 allow-list: registries and diagnostics); subscribers see only ChangeBatch facades valid for the duration of one handler call"
    write_access: kernel-only (mutators reach the buffers only through scope.mark_dirty and scope.emit_event)
    referenced_by:
      - docs/architecture/adr-0004-change-notification-and-dirty-keys.md
    added: 2026-10-28
    revised: ""
```

### performance_budgets

`kernel_p5_notify`

```yaml
  - system: kernel_p5_notify
    status: active           # ADR-0004 is Proposed
    budget_ms: 0.25
    adr: docs/architecture/adr-0004-change-notification-and-dirty-keys.md
    notes: "PROPOSAL pending KRN-LAT-02, not a GDD number: P5 plus frame-end delivery P99 on a release export on RM-DESKTOP and RM-DECK for 12 dirty blocks, 40 subscriptions, a 38-key Graph cascade and 4 events; inside the 4.0 ms one-tick frame simulation budget; the full-capacity sort and has() scan per dirty block is a recorded measurement target"
    referenced_by:
      - docs/architecture/adr-0004-change-notification-and-dirty-keys.md
    added: 2026-10-28
    revised: ""
```

### forbidden_patterns

`kernel_signal_notification`

```yaml
  - pattern: kernel_signal_notification
    status: active           # ADR-0004 is Proposed
    description: "No engine signal carries kernel-to-subscriber change notification: src/kernel declares no signal for it, nothing connect()s to a kernel object, and subscribers register only through the class-bound NotifyFacet (presentation: PresentationFacet.subscribe). Engine signals remain legal inside presentation."
    why: "A signal returns nothing (no completion token, no per-subscriber INCOMPLETE), has no class cadence or block filter, and hides dead subscribers; OQ-4 decided on semantics."
    adr: docs/architecture/adr-0004-change-notification-and-dirty-keys.md
    added: 2026-10-28
```

`lambda_or_retained_batch_subscriber`

```yaml
  - pattern: lambda_or_retained_batch_subscriber
    status: active           # ADR-0004 is Proposed
    description: "A subscriber handler is a standard method Callable on a long-lived object with the signature (batch: ChangeBatch, ctx: NotifyContext) -> CompletionToken; never a lambda or .bind() result, and never retains a ChangeBatch, a kernel key buffer or an event object past the handler call (copy into a DirtyAccumulator)."
    why: "Probed 2026-10-28: lambdas and bind results are custom Callables (a non-capturing lambda reports the script as get_object(), a self-capturing one the creator, and it keeps self alive); the batch is a reused facade re-armed every delivery."
    adr: docs/architecture/adr-0004-change-notification-and-dirty-keys.md
    added: 2026-10-28
```

`await_in_subscriber_handler`

```yaml
  - pattern: await_in_subscriber_handler
    status: active           # ADR-0004 is Proposed; lint is ADR-0010's PRES rule
    description: "A subscriber handler, and anything it calls, never awaits; it returns the armed CompletionToken synchronously."
    why: "A handler containing await returns a coroutine state object, not the token (recorded INCOMPLETE); the continuation resumes after P5 holding a poisoned ChangeBatch and a re-armed NotifyContext."
    adr: docs/architecture/adr-0004-change-notification-and-dirty-keys.md
    added: 2026-10-28
```

`non_integer_or_undeclared_dirty_key`

```yaml
  - pattern: non_integer_or_undeclared_dirty_key
    status: active           # ADR-0004 is Proposed
    description: "A dirty key is an int (KIND << 32) | id built from integers, KIND declared in the block's KeySpaceDescriptor; never a String, StringName, float or Variant, never an id above 32 bits, never key 0 or NO_KEY; events carry only ints and never state."
    why: "No String is built per key per run (Graph LS-20, ADR-LSG-06); NO_KEY is the buffer sentinel; notifications carry no state (CR-10)."
    adr: docs/architecture/adr-0004-change-notification-and-dirty-keys.md
    added: 2026-10-28
```

`perception_out_subscribe_without_access`

```yaml
  - pattern: perception_out_subscribe_without_access
    status: active           # ADR-0004 is Proposed
    description: "A subscription to a block whose read class is PERCEPTION_OUT is refused (SUBSCRIBE_REFUSED) unless the issuing facet carries perception_access = true (only the Perception Filter's PresentationFacet and DIAGNOSTIC facets)."
    why: "Dirty-key ids leak the timing and identity of perception changes even when the read is blocked."
    adr: docs/architecture/adr-0004-change-notification-and-dirty-keys.md
    added: 2026-10-28
```

### referenced_by_update

`write_outside_writescope_or_setter_dirty`

```yaml
forbidden_patterns / write_outside_writescope_or_setter_dirty: add to referenced_by (the entry has no referenced_by field today): docs/architecture/adr-0004-change-notification-and-dirty-keys.md  # mark_dirty requires a prior open_for in the same scope use; INIT and RESTORE marks are discarded
```

`owner_callable_contract`

```yaml
interfaces / owner_callable_contract: add to referenced_by: docs/architecture/adr-0004-change-notification-and-dirty-keys.md  # subscriber handlers are (batch: ChangeBatch, ctx: NotifyContext) -> CompletionToken
```

`block_registration`

```yaml
interfaces / block_registration: add to referenced_by: docs/architecture/adr-0004-change-notification-and-dirty-keys.md  # BlockRegistrar gains register_key_space(KeySpaceDescriptor)
```

`presentation_submit_without_handle`

```yaml
forbidden_patterns / presentation_submit_without_handle: add to referenced_by: docs/architecture/adr-0004-change-notification-and-dirty-keys.md  # subscribing confers no submit right; this ADR reconciles kernel GDD AC ~L1014 to it
```

## Related
- Depends on: ADR-0001, ADR-0002, ADR-0003 (all Proposed); soft: ADR-0010 (source layout and lint enforcement, Proposed).
- Enables: ADR-0005, ADR-0006, ADR-0009, ADR-0031, ADR-0032 and every block-registering system ADR (see ADR Dependencies).
- Forward references: ADR-0005 (`block_hash_any`, H64, cache validity by touch serial), ADR-0009 (presentation subscriber contract; which keys fire when; `perception_access`), ADR-0010 (lints), ADR-0011 (Graph key families and composite ids), ADR-0017, ADR-0022, ADR-0028 (consumers of marks and events), ADR-0031 (counters, `DIAGNOSTIC` gate), ADR-0032 (REPLICATION cadence).
- GDDs: `command-simulation-layer.md` (CR-9, CR-10, CR-12, D.9, D.12, Open Question 8), `life-support-graph.md` (LS-20), `ship-section-and-layout-data-model.md` (LY-27, LY-32), `ship-body-and-interior-physics.md` (BDY-21, BDY-28), `suit-and-equipment.md` (SE-34, SE-38), `web-map-and-transit.md` (WM-2, WM-31), `failure-and-trail-system.md` (FT-15), `emitter-state-and-light-exposure-query.md` (EM-26), `2d-lighting-and-occlusion.md` (LO-3).
- `docs/architecture/architecture.md` § Data Flow 4, § API Boundaries, § Required ADRs, § Spikes that gate ADRs (KRN-LAT-02), OQ-4.
- `docs/registry/architecture.yaml`: `write_outside_writescope_or_setter_dirty`, `presentation_submit_without_handle`, `owner_callable_contract`, `block_registration`, `fault_domain_table`; this ADR's additions are in § Registry Candidates.

## Open Questions
Numbers are stable: Open Questions 3, 4, 5 and 7 were closed by the 2026-10-28 review (Save/Load trigger path: ADR-0006 § 1.12, no subscriber class; derived blocks: `block_hash_any`, ADR-0005 § 5.3; hash-cache invalidation: adopted by ADR-0005 § 5.2; PERCEPTION_OUT subscribers: subscribe-time check 6) and removed.
1. **P5 budget.** The kernel GDD has `t_P5` but no budget; 0.25 ms P99 is this ADR's proposal. KRN-LAT-02 on a release export sets the real figure, measures the full-capacity sort and scan, and confirms the registry shows no cost problem (OQ-4 gate).
2. **Sim-side event consumers.** WM-31 lists Attention and Horror as consumers of Web Map events; CR-10 forbids block and hook owners from subscribing. This ADR assumes they read `web.state`'s view (for example a loaded flag or `leg_epoch`) in their hooks. Web Map, Attention and Horror owners confirm (including `horror-entities.md` ~L369, not read for this ADR); if a sim-side event feed is wanted it is a new kernel mechanism, not a subscription.
6. **Kind numbers.** Emitter, Suit, Web Map, Attention and Controller families are unnumbered in their GDDs; only distinctness inside a block is required. Each owner ADR fixes its numbers; the key-space CI test (Validation Criteria) checks them.
8. **Composite ids.** Graph `L/<loop>/<anchor>` must pack into 32 bits; ADR-0011 fixes the packing and its range check.
9. **Release-export and profile behaviour** of `PackedInt64Array.has/sort/bsearch` cost, of `Callable.get_argument_count()` and of a subscriber that aborts on a script error is unmeasured (UNVERIFIED), settled by KRN-LAT-02 and ADR-0003's KRN-CTX-06. Whether playtest exports run the `STRICT_DEBUG` profile (and so the § 7 audit) is also UNVERIFIED.
10. **Event argument count.** Six integers fit every GDD payload today (`BODY_TRANSFER` is exactly six after splitting the sub-kind into the event kind); a longer payload would need a seventh slot or a second event.
11. **Naming `KeySpaceDescriptor`.** It sits in kernel core (`src/kernel/`) and block owners construct it; ADR-0010 must either allow owners to name this one type or move it to `src/kernel/facets/`.
12. **Quiet declarations to confirm.** The Web Map ADR states whether `web.state` guarantees no write without a key during a leg (otherwise it is declared quiet, § 7) and supplies `pod_arrived_seq` and `berthed_seq` (ADR-0006 § 1.12); ADR-0022 decides whether `attention.state` is quiet.

## Review Log
- **Joint review of the six Foundation ADRs, 2026-10-28.** The first pass returned "review unavailable" for both reviewers; the reviews were then run and returned the outcome below, applied in the same day's revision. Status stays Proposed.
- **Engine review (godot-specialist), verdict NOTES.**
  - **E1** (idiom and compile check OK): applied. One line added to Engine Compatibility ("Engine Review" row); no code change.
  - **E2** (sorted-order guarantee breaks on the merged path): applied. § 6 closes (sorts) every frame buffer that received a union before delivery (`KeyBuf.unioned`, `union_from`); Validation case keys 9, 3, 5 delivers `[3, 5, 9]` and `has(5)` is true. Merged with TD-31.
  - **E3** (`bsearch` never returns -1): applied. `ChangeBatch.has` contract written as `var i := buf.bsearch(key); return i < count and buf[i] == key`, bounded by `count`; validation case added.
  - **E4** (typed Dictionary miss): applied. `index_of.get(block_id, -1)` with -1 as `WRITE_OUTSIDE_SCOPE`; `make_read_only()` at the freeze (§ 4). The `d[k]` miss behaviour is recorded UNVERIFIED.
  - **E5** (probe claims mis-stated): applied. Probe (c) and (d) reworded in Engine Compatibility; § 1 point 3 and § 2 check 1 aligned; (e) and (f) remain as authored (UNVERIFIED on a release export).
  - **E6** (`await` in a handler): applied. Implementation Guidelines and § 2 handler contract; ADR-0010 asked to add the PRES rule (Cross-ADR amendments); merged with TD-16.
  - **E7** (no try/finally): applied. `frame()` is a thin wrapper over `_frame_body()` with an unconditional `deliver_frame_end()`; one-return lint/test and KRN-FRAME-END owned by ADR-0001 (TD-04).
  - **E8** (handler error under the test runner): applied. Validation Criteria require INCOMPLETE tests to use handlers returning null or a stale token; the real-error case is a manual or release-export check.
  - **E9** (performance and docs nits): applied. (1) full-capacity sort and scan recorded as a KRN-LAT-02 target with fallbacks; (2) bit table reads "bits 32 to 46 used, 47 to 62 reserved"; (3) wording "Dictionary typing (4.4, thinly covered per `VERSION.md`)"; (4) one declaration per line in `NotifyEvent`.
- **TD-ADR review (technical-director), verdict CONCERNS.**
  - **T1** (BLOCKING, `MAX_DIRTY_KEYS` knob had no effect): applied. `KeyBuf.cap` separate from `buf.size()`, `set_cap` at the P0 latch on every tick and frame buffer, `count == cap` in `mark` and in the fold union; `DirtyAccumulator` keeps the sized capacity with the reason stated; live-knob validation case.
  - **T2** (BLOCKING, acceptance depends on unmade amendments): applied. Ordering Note precondition now requires ADR-0001, 0002 and 0003 Accepted with this ADR's Cross-ADR amendments applied; Cross-ADR amendments expanded.
  - **T3** (BLOCKING, type placement and `NotifyFacet` vs ADR-0009): applied. Placement table in Key Interfaces; `KeyBuf` in `src/kernel/facets/`; `DirtyAccumulator` self-contained; `NotifyFacet` held only by the composition root with the class fixed at issue; presentation receives ADR-0009's `PresentationFacet`; ADR-0010 narrows ARCH-08 (merged with TD-01).
  - **T4**: applied. D.12 GDD sync made unconditional; OQ-5 closed as adopted by ADR-0005 § 5.2; registry candidate `touch_serial` added.
  - **T5**: applied. Audit tied to the KernelProfile (wherever `SIM_WRITE_TRIPWIRE` runs) and reads `block_hash_any`, not the public `block_hash` (merged with TD-32). Whether playtest exports run `STRICT_DEBUG` is UNVERIFIED.
  - **T6**: applied. Event limits stated global; per-block event lists are fixed-size `PackedInt32Array` index lists into one global pool preallocated at the freeze; subscriber lists are fixed-capacity packed index arrays rebuilt in place; pooled events poisoned under `STRICT_DEBUG`.
  - **T7**: applied. Quiet list marked illustrative with the descriptor and CI test normative; `web.state` added as quiet (conditional); `attention.state` flagged for ADR-0022.
  - **T8**: applied. Alternative 9 rejected outright; no control-plane callback; Save/Load polls monotonic counters (ADR-0006 § 1.12); OQ-3 closed; requirement recorded for the Web Map ADR (merged with TD-09).
  - **T9**: applied. Subscribe-time check 6 and its validation line; OQ-7 closed (merged with TD-08).
  - **T10**: applied. ADR-0010 moved from Enables to a soft Depends On; ADR-0006 added to Enables; ADR-0009 asked to add ADR-0004 to its Depends On; SIM-13 becomes an active rule citing § 1 (merged with TD-27).
  - **T11**: applied by probing instead of marking UNVERIFIED: `get_argument_count()` returns 2 for a two-parameter method Callable, 2 for a two-parameter lambda and 1 after `.bind(1)` on 4.7.2 (editor build, probe (g)); release-template behaviour stays with KRN-LAT-02.
  - **T12** (advisory): (b) applied (freeze-time owner and perception hits are a boot failure under `STRICT_DEBUG`, a counted removal under `LENIENT_RELEASE`); (c) applied (§ 6 states ADR-0004 supplies no load marker by design; see `PresentationClock.load_seq`); **(a) declined**: it is optional, the audit is already budgeted inside the tripwire's existing hash pass, and ruling TD-32 gates it on the tripwire's profile rather than on test builds only; `QUIET_WRITE_TICKS` is kept as the dev signal either way.
- **Cross-ADR rulings owned here.** **TD-08**: applied (check 6, OQ-7 closed, ADR-0003 § 7 two-face rule and ADR-0009 `perception_access` listed). **TD-31**: applied (frame-end close of any unioned frame buffer; a one-tick frame keeps the swap). **TD-33**: applied (Registry Candidates section added, corrected: handlers `(batch: ChangeBatch, ctx: NotifyContext) -> CompletionToken`, `register_key_space` under `block_registration`, `touch_serial` owned by the kernel registries and consumed by ADR-0005).
- **Cross-ADR rulings owned elsewhere, aligned.** **TD-01**: placement table, `KeyBuf` moved to `src/kernel/facets/`, `NotifyFacet` stays with the composition root. **TD-04**: frame wrapper, KRN-FRAME-END, `PresentationClock.state` and `load_seq`, `HOST_SERVICE` priority; § 6 states "no load marker in `ChangeBatch` by design". **TD-07**: ADR-0003 batch listed in Cross-ADR amendments. **TD-09**: Alternative 9 and OQ-3. **TD-16**: `await` ban and INCOMPLETE for any non-armed return including a coroutine state. **TD-24**: no ADR-0004 change; § 7 states a dirty key can never gate caching. **TD-27**: ADR-0010 moved to Depends On, ADR-0006 added to Enables. **TD-32**: § 7 cites `block_hash_any` and the profile gate.
- **Follow-up.** None required of the reviewers. Acceptance waits on the Cross-ADR amendments being applied and KRN-LAT-02 passing.
- **2026-10-28 /architecture-review fixes.** The soft ADR-0010 reference moved from Depends On to its own Soft references row; `adr-dep-graph.sh` reads Depends On as hard edges and reported a cycle through it. No change to the acceptance gate.
- **2026-10-29 /architecture-review follow-up.** Enables row: ADR-0006 re-labelled "related, not a dependency" because ADR-0006 does not list this ADR under Depends On (residual 10). No behaviour change.
- **2026-10-29 /architecture-review follow-up (Foundation gaps).** Added: TR-ftrail-040 (one step's keys delivered in one batch and drained in one `present`, § 6). Closes review gaps; no change to existing decisions.
