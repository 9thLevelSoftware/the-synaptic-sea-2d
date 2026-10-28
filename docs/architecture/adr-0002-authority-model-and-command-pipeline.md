# ADR-0002: Authority Model and Command Pipeline

## Status
Proposed

## Date
2026-10-28

## Last Verified
2026-10-28 (against `design/gdd/command-simulation-layer.md` CR-4 to CR-7, CR-15 to CR-17 and `docs/engine-reference/godot/modules/networking.md`). No code exists yet; nothing here has been run on 4.7.2.

## Decision Makers
User (delegated all remaining design calls, 2026-10-28); Claude (author); reviewed by godot-specialist (engine: 3 blocking and 16 minor items, all applied in this revision) and technical-director (TD-ADR: CONCERNS; findings 1-10 applied in this revision). Amended the same day for ADR-0003's TD review: result and control-outcome handlers are kernel-invoked callables with the `(record, ctx: NotifyContext) -> CompletionToken` shape, and `register_command` lives on the `BlockRegistrar`.

## Summary
Every change to the simulation must enter as an authority-checked, plain-data command that the kernel stamps and orders, even in single-player where the one local player is the host. This ADR fixes how: capability handles (not globals) are the only way to submit or to call the control plane, a single ingress function stamps every command, each command type declares a payload schema the kernel enforces, and ownership is judged by a kernel-held binding table keyed by a stable PlayerId rather than by network peer id.

## Engine Compatibility

| Field | Value |
|-------|-------|
| **Engine** | Godot 4.7.2 (stable), GDScript |
| **Domain** | Core (command ingress and authority), with a Networking boundary kept dormant until VS-B |
| **Knowledge Risk** | MEDIUM. Nothing used here is post-cutoff, but `var_to_bytes` size behaviour for payload checks, `Variant.Type` coverage and the 4.7 peer-id behaviour of `MultiplayerAPI` are not covered by the pinned reference. |
| **References Consulted** | `docs/engine-reference/godot/VERSION.md`, `breaking-changes.md` (4.6 → 4.7 GDScript and Core rows), `deprecated-apis.md`, `modules/networking.md`, `modules/input.md` |
| **Post-Cutoff APIs Used** | None in the MVP. Typed `Dictionary[K, V]` (4.4) is allowed inside the kernel but never at the payload boundary, where payloads are untyped `Dictionary` so the whitelist can inspect them. |
| **Verification Required** | (1) `var_to_bytes(payload).size()` is cheap enough at the admitted command rates and is a pure function of the payload's value (it is used only as a threshold; never as hash input, because the encoder's size choices for ints and floats are not a byte-stable cross-platform contract, UNVERIFIED). (2) A `Dictionary` or `Array` payload that contains itself is stopped by the depth cap (the walk recurses to depth 9 and rejects) **before** any serialisation is attempted; serialising a cyclic container is not assumed safe. (2a) A wide payload cannot burn time before the size check: the walk aborts as soon as a running upper-bound byte estimate exceeds `MAX_PAYLOAD`. (3) The peer id a `MultiplayerAPI` RPC reports for the host is `1` on 4.7.2 (VS-B only; recorded here because `HOST_PEER_ID` = 1 is baked into the binding table). (4) GDScript has no private constructors, so "only the kernel builds a `Command`" is enforced at run time (a `_sealed` flag and a ring-slot check) and also by a CI lint that greps for `Command.new()`. (5) `Callable.is_valid()` is false for a method Callable whose target object was freed, and `Callable.call()` on a freed target errors rather than skips (UNVERIFIED; both from memory). (6) Writing into a `PackedInt64Array` fetched from a `Dictionary` value mutates the stored array only through an explicit read-modify-write (UNVERIFIED across versions); the binding table avoids the question. |

## ADR Dependencies

| Field | Value |
|-------|-------|
| **Depends On** | ADR-0001 (the injected-kernel rule and the frame order this ADR relies on; **Proposed**, not yet Accepted) |
| **Enables** | ADR-0003 (block registry uses the command-type descriptor defined here), ADR-0016 (binding-table rebuild on restore), ADR-0019 (Input Adapter uses a surface handle), ADR-0020/0021/0022/0026/0027 (each registers command types against this contract), ADR-0030 (`PauseGate` uses a control handle), ADR-0032 (remote ingress for VS-B) |
| **Blocks** | Every story that registers or submits a command type: Player Controller, Interaction, Suit, Emitter toggles, Assembly, Web Map, Chart Table pause. |
| **Ordering Note** | Scope edge with ADR-0003: this ADR owns the **meaning** of the command fields, the ingress and the control-plane API; ADR-0003 owns the **registration descriptors**, the owner-callable contract and the kernel state machine. The `payload_schema` field defined here is added to ADR-0003's `CommandTypeDescriptor`. The kernel GDD's CR-6 field list does not name `payload_schema`; see § GDD sync. |

## Context

### Problem Statement
The kernel GDD (the owner) specifies command behaviour in detail: what a command is, who stamps what, the exactly-one-result guarantee, the authority check and the binding table. It leaves open how that becomes code in GDScript: how a caller obtains the right to submit, who may construct a command, how a payload is checked, how ownership is looked up, what the control plane looks like, and where the host/mirror split sits. Without this decision every system's author invents a submit path, and the one rule the project cannot lose later (co-op is an addition, not a rewrite) is decided by whichever author wrote first.

### Constraints
- **The host is the sole authority (CR-7).** Only a HOST kernel ticks and applies; a MIRROR never applies. Single-player is a host with one local player, with the same checks.
- **No network code at MVP.** No `MultiplayerAPI`, no RPC, no replication. The seam must exist; the transport must not.
- **Ids are kernel-stamped, never payload-borne** (`issuer_peer`, `origin`, `seq`, `correlation` echo, `submission_id`).
- **Fire-and-forget submit (CR-4).** `submit()` returns nothing and calls nothing, so it is safe from any handler.
- **Exactly one terminal result per command** (CR-5), delivered at the next flush in every kernel state.
- **Persistent per-player state is keyed by PlayerId, never by peer id** (Sanity, Suit, Controller, Interaction GDD rulings).
- **Injection, not singletons** (ADR-0001).
- **Queue and payload bounds are boot assertions** (CR-17: `MAX_CMDS_PER_TICK` 128, `QUEUE_CAPACITY` 1024, `PER_ISSUER_CAPACITY` 256, `MAX_PAYLOAD` 4 KiB, depth 8, 16 MiB joint memory).
- **Engine reference is silent** on `var_to_bytes` sizing and on 4.7.2 peer-id assignment.

### Requirements
- A caller can submit only if it was handed a submit handle at composition time; a presentation node that was not handed one cannot submit.
- Only the kernel builds a `Command`; no constructor takes an `issuer_peer` from a caller.
- Payload shape is enforced at admission per command type, not rediscovered by each owner.
- Ownership of an acting entity is decided by one table the kernel maintains, never by the payload and never by node ownership in the scene tree.
- Pause, save, restore, new game and shutdown reach the kernel through a host-local control plane that presentation without a control handle cannot call.
- The same kernel, with a different mode flag and a different ingress, must be able to run as a MIRROR in VS-B without changing its public API.

## Decision

### 1. Capability handles are the only way in

There is no global `submit`. The composition root (ADR-0001) holds the kernel in `BOOTING` and issues two kinds of handle, each by name, each once:

```gdscript
# Issued only in BOOTING by the composition root. Names appear in diagnostics.
var surface: SurfaceHandle = kernel.issue_surface_handle(&"input_adapter")   # may submit PLAYER commands
var control: ControlHandle = kernel.issue_control_handle(&"save_load")       # host-local control plane
```

- A **`SurfaceHandle`** carries `submit(type, payload, actor, correlation)` and a result handler. It is stamped at issue time with the origin `PLAYER` and the `issuer_peer` it represents (`HOST_PEER_ID` = `MultiplayerPeer.TARGET_PEER_SERVER`, value 1, for every local surface; the constant is defined from the engine name, not the literal). At MVP exactly one surface is issued, to the Input Adapter (the Chart Table submits zero commands at MVP; a second surface is issued only when a system genuinely needs one, and is named at issue). **Holding a `SurfaceHandle` and subscribing as `INPUT_SURFACE` are separate rights:** subscribing in that class (CR-10) means receiving every tick's batch; it does not give submit rights, and the Interaction prompt object subscribes as INPUT_SURFACE without a handle. "Presentation never submits" is therefore enforced by a **state guard plus a lint**, not by construction: `Kernel` exposes no public submit or control method (both live only on handles); `issue_*_handle` refuses outside `BOOTING`; presentation subscribes only after registration freezes; and a CI check forbids calls to `_`-prefixed kernel members from outside `src/kernel/`, because GDScript has no private members.
- A **`ControlHandle`** carries the host-local control plane (§ 6). It is issued to Save/Load (`save_load`), the Chart Table's `PauseGate` (`pause_gate`) and the composition root (`game`); nobody else gets one.
- **SYSTEM-origin submission is not a handle.** `submit_system(type, payload, dedupe_key)` and `has_pending(type, dedupe_key)` are methods of the `HookContext` and `ApplyContext` objects the kernel passes into step hooks and `apply` (ADR-0003 defines the contexts). A SYSTEM command can therefore only be created inside a tick phase, by a registered owner, and never from a signal handler or `_process`. The test ingress (§ 8) is the one other caller.
- Handles are issued only while the kernel is `BOOTING` (like every registration, CR-6) and are not serialisable. Handle names are unique; a duplicate fails boot.
- Handles are plain `RefCounted` objects passed by `setup()` (ADR-0001). Holding a reference is the capability.
- **A forged handle fails closed.** GDScript cannot stop `SurfaceHandle.new()`, so a handle carries an integer slot id and a token that the kernel wrote into its issued-handle table at issue time. `submit` forwards both to `_admit`, which checks the pair against the table; an unregistered or mismatched handle logs `push_error`, counts `HANDLE_UNREGISTERED` in diagnostics and returns without admitting anything. Nothing relies on the lint alone.
- **No reference cycle.** The kernel stores handles only as slot ids and tokens (never as object references), and a handle holds the kernel, so the pair is a one-way reference. `Kernel.shutdown()` clears the table.
- **Result handlers are method Callables, cleared on teardown.** The handler passed to `set_result_handler` must be a method Callable on a long-lived object (a lambda that captures `self` creates a handle → Callable → owner → handle cycle that GDScript's reference counting leaks). The owner calls `clear_result_handler()` from its `_exit_tree`. At flush the kernel checks `handler.is_valid()` before every call and drops an invalid handler with a diagnostics count `RESULT_HANDLER_DEAD`.

### 2. One ingress function stamps every command

All paths (surface handle, `submit_system`, the future remote ingress, the test ingress) call one kernel-private function:

```gdscript
func _admit(origin: int, issuer_peer: int, type: StringName, payload: Dictionary,
		actor: int, correlation: int, supplied_seq: int) -> void   # supplied_seq only from the test or remote ingress
```

It performs, in order: stamp `submission_id` (a kernel-unique counter, so even a rejected submit has one) → stamp `seq` from the one kernel-held counter for `(issuer_peer, origin)` unless the test or remote ingress supplied one → resolve the type (`UNKNOWN_TYPE`) → check kernel state (`NOT_ACCEPTING`; `PAUSED` unless the type is `allowed_when_paused`) → origin allowed for the type (`ORIGIN_NOT_ALLOWED`) → duplicate `seq` (`DUPLICATE_SEQ`, test and remote only) → rate limit (`RATE_LIMITED`, remote only) → payload walk, size and schema (`MALFORMED`) → coalescing (§ 5) → queue capacity (`QUEUE_FULL`) → enqueue with `queued_tick` = current `T`. Any rejection appends one result record and returns; nothing is thrown. The counter advances on every submit including rejected ones, so locally stamped values are dense (CR-4). **This order is normative** and goes into the kernel GDD (§ GDD sync) so tests are written against one order. It differs from CR-5's prose in one observable way: the type is resolved before the kernel-state check, because the `allowed_when_paused` exemption (CR-16) needs the type, so an unknown type submitted while `PAUSED` is `UNKNOWN_TYPE`, not `PAUSED`. The capacity check comes after coalescing because a supersession can free room.

`Command` is a `RefCounted` record with typed fields (`type_idx`, `issuer_peer`, `origin`, `actor`, `seq`, `correlation`, `payload`, `queued_tick`, `submission_id`, `host_order`) plus two kernel-only fields, `_sealed` and `_slot` (the `submission_id` it was admitted under). **Only `_admit` fills it, and only the queue owns the instances.** GDScript cannot make a constructor private, so the rule is enforced twice: at run time, `_admit` sets `_sealed = true` after filling a `Command` and records its `submission_id` in `_slot`, and the dispatch path (P1, P2, apply) refuses any `Command` that is unsealed or whose `_slot` does not match the queue entry it came from; and a CI check greps for `Command.new()` outside `src/kernel/`. `Command` has no `_init` parameter for `issuer_peer` at all. This is the "no constructor without `issuer_peer` exists" rule (kernel H6) in a form GDScript can state.

**No object is reused.** The ring holds references to `Command` instances; each admitted submit allocates one (a few tens per second, negligible, and the same order as the `var_to_bytes` allocation the size check already accepts); a slot is released by assigning `null`. Nothing pools or recycles a `Command`, so no `validate`, `apply`, result record or log entry can see a reused object. Even so, **owners must not retain `cmd` past the callable it was passed to** (copy the scalar fields needed), and **result and log records copy the scalar fields they need rather than holding a `Command`**. The command log copies the payload bytes.

Command types are addressed by `StringName` in the call and resolved once to a small integer `type_idx` at registration; the hot path is one `Dictionary` lookup keyed by `StringName`.

### 3. Payload schema per command type, enforced at admission

The kernel GDD checks payloads against a recursive `Variant.Type` whitelist (finite floats, depth ≤ 8, uniform dictionary key type, size ≤ 4 KiB via `var_to_bytes(payload).size()`). That says what a payload may be made of, not what a given command carries. This ADR adds a **mandatory `payload_schema`** to the command-type descriptor:

```gdscript
# Keys are the payload dictionary keys; values describe the expected value.
payload_schema = {
	&"target": {&"type": TYPE_INT, &"required": true},
	&"face_octant": {&"type": TYPE_INT, &"required": false, &"min": 0, &"max": 8},
}
```

**Order of the payload checks.** (1) The whitelist walk: `typeof()` against a constant set of allowed `Variant.Type`s, every dictionary key a `StringName` for payload dictionaries (a `String` key is `MALFORMED`, so the schema lookup is exact), all keys of any nested dictionary of one type, depth cap `PAYLOAD_MAX_DEPTH` 8, `is_finite()` on every float and on every component of `Vector2` and each element of the float and `Vector2` packed arrays, and a running upper-bound byte estimate that aborts the walk the moment it exceeds `MAX_PAYLOAD`. (2) Only after the walk passes, `var_to_bytes(payload).size() <= MAX_PAYLOAD`. (3) The schema below. A cyclic container is rejected by the depth cap at step 1, never serialised. Packed arrays are whitelisted and counted in the byte estimate (a large `PackedByteArray` passes the type check but not the budget). The range checks in step 3 test `is_finite()` first, because `NAN < min` and `NAN > max` are both false and a naive range test would pass `NAN`.

Admission checks, after the whitelist walk passes: every required key present; no key outside the schema; each value of the declared `Variant.Type` (an `int` where `float` is declared is `MALFORMED`, never silently converted); integers and floats inside the declared inclusive range when one is given. Entity and actor ids are `int` (64-bit in GDScript) and are non-negative; `NO_ACTOR` is `-1`, which no real id can equal. Ids above 2^53 must never pass through JSON (a JSON number parses as a float and loses precision); saves carry them as packed integer arrays (ADR-0006). A mismatch is `MALFORMED`. The dictionary form above is authoring syntax. At registration the kernel **compiles each schema into parallel arrays** (keys, types, required flags, bounds) and validates it at boot: an unknown schema key (a typo such as `&"minn"`) fails boot, and a bound whose type does not match the field's declared type fails boot (an `int` field with a `float` bound would otherwise compare silently). Admission then walks the compiled arrays, not nested dictionaries. The schema is **flat**: it covers the top-level keys of the payload. Nested `ARRAY`, `DICTIONARY` and packed values are checked **by type only** (the whitelist walk already checked element types, depth and key-type uniformity), and what their elements mean stays in the owner's `validate`; a later `elements` sub-schema is possible but not needed by the MVP commands (SYSTEM payloads such as `weld {actor, target, cause}` are flat). Payload dictionaries use `StringName` keys; a caller that writes `{"target": x}` produces a `String` key and is `MALFORMED`, so no lookup depends on `String`/`StringName` equivalence. The schema describes **shape only**; whether the target exists, is in reach or is lit stays in the owner's `validate`, so a rejection reason is never produced by the schema layer except `MALFORMED`. A type with no payload registers an explicit empty schema `{}` (no silent default, matching CR-6's rule for the other mandatory fields); omitting the field fails boot.

The schema is data on the descriptor, so the same table drives the admission check, the STRICT_DEBUG serialisation round-trip test, the control-manifest's list of commands, and the debug overlay's command panel.

### 4. Ownership is a kernel binding table keyed by PlayerId

Exactly as CR-7: half (a) `peer_id → PlayerId` is session state set by a host-only API (a boot step in the MVP: `HOST_PEER_ID` 1 → PlayerId 1), never touched by a restore, new game or death restart, and never resetting any issuer's `seq`. Half (b) `PlayerId → {(source block_id, entity_id)}` is derived from saved state and is never saved; the kernel rebuilds it as the last step of every pass through `LOADING` from the registered binding sources in ascending `block_id` order. The data structure:

- Half (a): a `Dictionary` `int → int`.
- Half (b): per PlayerId, one small `PlayerBindings` object holding two parallel `PackedInt64Array`s (source block index, entity id) behind one `add` / `remove` / `owns(source, entity)` interface, so the arrays cannot drift out of step. A player owns a handful of entities (an avatar and a beam), so `owns` is a linear scan of a few elements with no composite-key hashing and no allocation. Mutation is an explicit read-modify-write on the object's own arrays (never `dict[k].append(x)` on a packed array stored in a dictionary, whose behaviour has varied across versions), covered by a unit test.
- **Determinism of iteration.** The `LOADING` rebuild and every registry walk iterate **sorted** arrays (block ids ascending, command types in registration order recorded in an array), never `keys()` of a Dictionary filled from an unordered source. Dictionaries preserve insertion order in Godot 4, but erasing and re-inserting a key moves it to the end, so no hash or rebuild may depend on it.

A command type with `actor_binding = OWNED_BY_ISSUER` passes P2 iff `(actor_source(type), actor)` is in half (b) of `PlayerId(issuer_peer)`; otherwise it is `Rejected(ACTOR_NOT_OWNED)` with no state change. Persistent state is keyed by PlayerId; the peer id appears only in the ingress stamp and the binding table. Ownership is **never** derived from scene-tree node ownership, `set_multiplayer_authority()` or any engine multiplayer primitive.

### 5. Ordering, coalescing and the queue

- Apply order is ascending `(queued_tick, host_order)`, where `host_order` is the arrival counter at `_admit`; in single-player it is submission order (CR-13 G1).
- The queue is one preallocated ring of `Command` slots sized to `QUEUE_CAPACITY` with per-issuer counters for `PER_ISSUER_CAPACITY` and a `SYSTEM_RESERVE` so SYSTEM commands cannot be starved by a flood of PLAYER ones. Overflow is `QUEUE_FULL` at admission, never a silent drop. **Accounting rule:** the capacity counters (`QUEUE_CAPACITY`, `PER_ISSUER_CAPACITY`) decrement the moment a command is superseded, P1 skips tombstones and counts only live commands against `MAX_CMDS_PER_TICK`, and the tombstone's ring position is reclaimed when the head passes it, so an aim stream under backlog cannot cause a spurious `QUEUE_FULL`. On entry to `LOADING`, `SHUTTING_DOWN` or `FAULTED` the queue is flushed and every queued command receives `Dropped(RELOAD|SHUTDOWN|FAULTED)` (the exactly-one-result guarantee, CR-5 and G5).
- **Coalescing** (`coalesce = LATEST_PER_ACTOR_PER_TICK`) is done by scanning the unfrozen portion of the queue for a command with the same `(type_idx, actor, issuer_peer)` and marking the older one `Superseded`. The scan runs only when a coalescing type is submitted, and intent types are sent a few times per second (aim at most every 3 ticks), so the scan over at most 1,024 slots is cheap and avoids a second index structure that would have to be kept in step with the ring. A type with `coalesce ≠ NONE` that allows both origins fails boot (`COALESCE_DUAL_ORIGIN`).
- Each tick, P1 moves at most `MAX_CMDS_PER_TICK` commands into the frozen batch for P2; the rest wait FIFO.
- A command older than its type's `max_age_ticks` is `STALE`. At local defaults this is unreachable (CR-17) and is guarded by an invariant test.

### 6. The control plane is a host-local API on a `ControlHandle`

`request_pause()`, `request_resume()`, `request_snapshot()`, `request_restore(payload, restore_epoch, expected_state_hash)`, `request_new_game(world_seed, restore_epoch)`, `request_shutdown()`, `request_bind_seat(peer_id, player_id)` and `request_unbind_seat(peer_id)` are methods of `ControlHandle`, not commands. They are not stamped, queued, authority-checked or logged as commands. Each call is appended to a small host-local request list and served **at the next boundary service, in arrival order** (ADR-0001 § 1), and each request ends in exactly one outcome: `SERVED`, `DEFERRED` (made while an atomic step was active; evaluated at the first boundary after it) or `IGNORED` (a (state, trigger) pair the transition table does not list; one invalid-transition entry, no state change). **Outcome delivery.** Outcomes are always recorded in diagnostics. Callers that must act on one set `ControlHandle.set_outcome_handler(Callable)` and receive a typed `ControlOutcome` (`request_kind`, `outcome`, `tick`, and for `request_restore` the K-4 verdict `{T, computed_hash, match}`; for `request_snapshot` the captured payload, a deep copy) at the same per-frame flush point as command results, **in a separate list that is never part of the CR-5 command result buffer**. This amends one sentence of the kernel GDD (States and Transitions, "control-plane requests have none"; § GDD sync): the outcome list is not a command result record, but it exists, because Save/Load needs the snapshot payload and the K-4 verdict (CR-15) and PauseGate needs to converge. Two consequences are intentional: a `request_pause` and a `request_resume` in the same boundary pair up in order, and `PauseGate` (Chart Table) re-issues the latest desired pause state until the kernel matches it, so a same-boundary pair leaves no wedged half-open state (TR-charttable-004).

**Seat binding (binding table half (a)).** `request_bind_seat(peer_id, player_id)` and `request_unbind_seat(peer_id)` are the host-only writers of `peer_id → PlayerId` (CR-7). The composition root, holding the `game` control handle, calls `request_bind_seat(HOST_PEER_ID, 1)` as its boot step; the Co-op Session calls the same two methods in VS-B, so the kernel's public API does not widen then. A restore, new game or death restart never touches half (a). Runtime seat reassignment (kernel open question 4) is deferred to ADR-0032 under this signature. When `issuer_peer` has no half (a) entry, a type with `actor_binding = OWNED_BY_ISSUER` is `Rejected(ACTOR_NOT_OWNED)`; a `NONE` type proceeds.

Pause is single-player only. While `PAUSED` the kernel accepts only command types flagged `allowed_when_paused` whose `writes_blocks` name no `gameplay = true` block; everything else is rejected `PAUSED` at admission without queuing (CR-16). The control plane is host-local by construction: a remote ingress has no `ControlHandle`.

This ADR owns how requests are accepted, ordered and answered. The states they move between (`RUNNING`, `PAUSED`, `SAVING`, `LOADING`, `FAULTED`, `SHUTTING_DOWN`) and the transitions' internals belong to ADR-0003.

### 7. Result delivery

Results are plain records (`result_id`, `issuer_peer`, `origin`, `seq`, `correlation`, `ok`, `reason`, `subject`, `detail`, `tick`) appended to the kernel's result buffer and flushed at P5 and once per rendered frame in every kernel state. A flush swaps the buffer out before delivering, so a handler that resubmits cannot recurse; it iterates that local array and re-checks each handler's validity per item, so a handler that frees its owner or its surface mid-flush invalidates only itself. `issue_surface_handle` and `issue_control_handle` are legal only in `BOOTING`, so a handler calling them mid-flush is a violation (CR-19), not a new handle. PLAYER results go to the issuing `SurfaceHandle`'s result handler and to the diagnostics log. SYSTEM results go to the command log and diagnostics counters only; an owner that needs an outcome reads state. Reason codes are computed from true state and never filtered by the kernel; whether and how cheaply a reason reaches the player is the presenter's rule (Pillar 4 cost rule), not this pipeline's.

### 8. HOST and MIRROR share one API; ingress is the only difference

On a MIRROR kernel `SurfaceHandle.submit` routes to an egress adapter (VS-B, ADR-0032), not to `_admit`; the handle's API is unchanged. `Kernel` is constructed with `Mode.HOST` or `Mode.MIRROR` and a `KernelProfile` (`STRICT_DEBUG` or `LENIENT_RELEASE`, injected, never read from `OS.is_debug_build()`). The public API is identical. A MIRROR does not tick and does not apply; it exists in the MVP only as a mode flag with a unit test proving `step()` refuses to apply in that mode. The ingress kinds are:

| Ingress | Origin it stamps | `issuer_peer` | May supply `seq` | Exists in |
|---|---|---|---|---|
| Local surface handle | PLAYER | the handle's peer (`HOST_PEER_ID` = 1) | no | MVP |
| `submit_system` on a context | SYSTEM | `HOST_PEER_ID` | no | MVP |
| Test ingress | either | caller's choice | yes | tests only: lives under `tests/`, is injected by the harness, and is never named by a `class_name` that shipped code references (an export filter removes files, and a referenced missing class fails to load) |
| Remote ingress adapter | PLAYER, always (it has no origin parameter) | the network sender id, read only here | yes | VS-B (ADR-0032) |

The remote ingress adapter is the only place in the project that reads a network sender id, and `SYSTEM` commands never arrive from a network by construction: the adapter cannot stamp that origin.

### 9. Command log and STRICT_DEBUG

A bounded ring of `COMMAND_LOG_CAPACITY` records (command metadata, a copy of the payload bytes and the result) supports diagnostics and replay tests only; the log is not saved and is not the source of truth. Under `STRICT_DEBUG` every command is round-tripped through `var_to_bytes` / `bytes_to_var` before dispatch; one that cannot survive is a violation (CR-19). Under `LENIENT_RELEASE` the round trip is skipped and a violation is counted rather than faulting where the GDD allows it.

### Architecture Diagram

```
Composition root (BOOTING)
  issue_surface_handle("input_adapter") ─► Input Adapter ─┐  submit(type, payload, actor, correlation)
  issue_control_handle("save_load")     ─► Save/Load       │
  issue_control_handle("pause_gate")    ─► PauseGate       │
                                                           ▼
 hook / apply ctx.submit_system(...)  ─────────►  Kernel._admit(origin, issuer_peer, ...)
 test ingress (tests only)            ─────────►    stamp submission_id, seq · type · state · origin ·
 remote ingress adapter (VS-B)        ─────────►    dup seq · rate · payload schema + whitelist ·
                                                     coalesce · capacity · enqueue
                                                           │ one result record per command
                                                           ▼
 tick P1 intake (≤ MAX_CMDS_PER_TICK) → P2: origin re-check · binding table (issuer's PlayerId owns
 (actor_source, actor)?) · age · owner validate(cmd, views, ctx) · owner apply(cmd, write_scope, ctx)
                                                           │
 result buffer ── flush at P5 and once per frame ──► surface handlers · command log · diagnostics

 ControlHandle.request_* ─► host request list ─► served at the next boundary service, in order
```

### Key Interfaces

```gdscript
class_name SurfaceHandle extends RefCounted
func submit(type: StringName, payload: Dictionary, actor: int = NO_ACTOR, correlation: int = 0) -> void
func set_result_handler(handler: Callable) -> void      # handler(result: CommandResult, ctx: NotifyContext) -> CompletionToken (ADR-0003)
func clear_result_handler() -> void

class_name ControlHandle extends RefCounted
func request_pause() -> void
func request_resume() -> void
func request_snapshot() -> void
func request_restore(payload: Dictionary, restore_epoch: int, expected_state_hash: int = 0) -> void
func request_new_game(world_seed: int, restore_epoch: int = 0) -> void
func request_shutdown() -> void
func request_bind_seat(peer_id: int, player_id: int) -> void
func request_unbind_seat(peer_id: int) -> void
func set_outcome_handler(handler: Callable) -> void   # handler(outcome: ControlOutcome, ctx: NotifyContext) -> CompletionToken; method Callable, cleared on teardown

# On HookContext and ApplyContext only (ADR-0003 defines the contexts):
func submit_system(type: StringName, payload: Dictionary, dedupe_key: Variant = null) -> void
func has_pending(type: StringName, dedupe_key: Variant = null) -> bool

# Kernel, BOOTING only:
func issue_surface_handle(name: StringName) -> SurfaceHandle
func issue_control_handle(name: StringName) -> ControlHandle
# Command types are registered through ModuleFacet.registrar.register_command(desc) (ADR-0003), which forwards to the kernel.
# CommandTypeDescriptor: this ADR defines the meaning of its fields; ADR-0003 § 2 states the complete typed field list:
#   type, owner_name, allowed_origins, actor_binding, actor_source, coalesce, max_age_ticks,
#   allowed_when_paused, writes_blocks, payload_schema, validate, apply
```

### Implementation Guidelines
- **Must** obtain submit and control rights only through handles issued by the composition root in `BOOTING`; **must never** add an autoload, static function or group lookup that submits or calls the control plane.
- **Must** construct commands only inside `src/kernel/` (`_admit`); **must never** call `Command.new()` elsewhere, and **must never** read or write `issuer_peer`, `origin`, `seq` or `submission_id` from payload data.
- **Must** register every command type with `allowed_origins`, `actor_binding`, `coalesce`, `max_age_ticks` and `payload_schema`; there are no defaults, and a missing field fails boot naming the type.
- **Must** keep the `payload_schema` to shape and range only; **must never** put semantic checks, existence checks or light/reach checks in it. Those are the owner's `validate`.
- **Must** treat subscribing as `INPUT_SURFACE` and holding a `SurfaceHandle` as separate rights; **must never** infer submit rights from the subscriber class.
- **Must** key persistent state by PlayerId; **must never** key it by peer id, and **must never** decide ownership from node ownership or engine multiplayer authority.
- **Must** edge-trigger actions with `coalesce = NONE`; intents may use `LATEST_PER_ACTOR_PER_TICK`. A coalescing type allows one origin only.
- **Must** make a hook that submits SYSTEM commands restart-safe and rejection-tolerant (decision a function of persisted state, `has_pending` guard at divisor 1); **must never** wait on a SYSTEM command's result.
- **Must** call the control plane only through a `ControlHandle` held by the named scripts (`save_load`, `pause_gate`, `game.gd`) and treat the call as fire-and-forget (outcomes are diagnostics only); **must never** read `kernel.state` to decide whether a request took effect, except `PauseGate`, which re-issues the latest desired pause state until the kernel matches.
- **Must never** read a network sender id outside the remote ingress adapter, and **must never** stamp `SYSTEM` from remote data.
- **Must** use payload keys of type `StringName`; a `String` key is `MALFORMED`.
- **Must** hand result handlers as method Callables on long-lived objects and clear them in `_exit_tree`; **must never** pass a lambda that captures `self`.
- **Must** treat a `Command` as owned by the ring: **must never** keep a reference to one past the callable it was passed to; copy the scalar fields needed.
- **Must** sort before iterating anything that feeds a hash, a rebuild or a registration order.
- **Where the greps run:** the "no `Command.new()`", "no `submit(` in `src/presentation/`" and "no handle outside the registered surfaces" rules run as a CI script (and a gdUnit4 test that scans `src/`), because GDScript has no analyser hook; the `_sealed`/slot checks and the issued-handle table are the run-time backstop.

## Alternatives Considered

### Alternative 1: Dictionary commands with no typed record
- **Description**: Commands are plain `Dictionary` objects with the stamped fields as keys, passed through the queue.
- **Pros**: Fewest classes; trivially serialisable.
- **Cons**: Any code can add or overwrite `issuer_peer`; no static type for the stamped fields; typos in key names are silent.
- **Rejection Reason**: The stamped fields are the security boundary. A typed record that only `_admit` fills keeps the rule checkable, and the payload remains a `Dictionary`.

### Alternative 2: Signals or an event bus for submission
- **Description**: Systems `emit_signal("command", type, payload)` and the kernel connects to it.
- **Pros**: Decoupled; no handle plumbing.
- **Cons**: Anyone can emit, so there is no capability; the emitter's identity is not known to the kernel, so `issuer_peer` cannot be stamped; signal delivery order is not the kernel's `host_order`; and a signal can be emitted from presentation.
- **Rejection Reason**: It removes exactly the properties this ADR exists to guarantee.

### Alternative 3: Ownership by node authority (`set_multiplayer_authority`)
- **Description**: Use Godot's per-node multiplayer authority to decide who may act on an entity.
- **Pros**: Free, and it is what high-level multiplayer is built around.
- **Cons**: It ties simulation ownership to scene-tree nodes, which the simulation deliberately does not have; the avatar has no node in the sim; it does not exist in the headless kernel, replay or tests; and it would key identity by peer id, which the persistent-state rulings forbid.
- **Rejection Reason**: Ownership must be a pure-data kernel table that saves, tests and replays can rebuild.

### Alternative 4: Key persistent state and ownership by peer id
- **Description**: Use the transport's peer id everywhere as the player identity.
- **Pros**: No second identifier; no binding table.
- **Cons**: Peer ids change on reconnect and on a different host; a save would bind a character to a session; Sanity, Suit and Controller state would not survive a re-join or a copied save.
- **Rejection Reason**: Settled in the kernel GDD (CR-7, CR-8): PlayerId is the stable seat, the peer id only an ingress stamp.

### Alternative 5: A free-form payload checked only by each owner's `validate`
- **Description**: The kernel applies only the type whitelist; each owner validates the whole payload shape inside `validate`.
- **Pros**: Matches the GDD literally; no new descriptor field.
- **Cons**: Every owner re-implements required-key, type and range checks, and several will do it differently or forget; an unexpected key or an `int`/`float` mix reaches `apply`; and the control manifest has no machine-readable list of what a command carries.
- **Rejection Reason**: The lead-programmer review asked for a per-command payload schema explicitly. The cost is one descriptor field and one admission pass.

### Alternative 6: Stamp-only dictionary index for coalescing
- **Description**: Keep a `Dictionary` keyed by a composite key to find the older command to supersede.
- **Pros**: O(1) lookup.
- **Cons**: A second structure to keep consistent with the ring on enqueue, supersede, dequeue, restore and fault; composite keys of a type index, a 64-bit actor id and a peer do not pack losslessly into one `int`.
- **Rejection Reason**: At a few intent submissions per second a scan of at most 1,024 slots is cheaper than the bookkeeping. Revisit only if profiling shows it.

## Consequences

### Positive
- "Presentation never submits" and "SYSTEM never comes from a network" become properties of what objects exist, not review rules.
- Co-op is an ingress adapter and a binding-table writer added later; nothing inside the kernel or any owner changes.
- Every system gets one admission path with one set of reason codes, and `MALFORMED` is the same everywhere.
- Command shape is data, so the manifest, the debug overlay and the STRICT_DEBUG round trip are generated from the registry.
- A copied save or a re-join keeps its characters because ownership is by PlayerId.

### Negative
- Handles must be threaded through `setup()` for every submitter and control caller; forgetting one fails at first use, so a diagnostics entry at shutdown (or after a fixed number of ticks) lists named handles that were issued and never used; this cannot run at boot, when no handle has had a chance to be used.
- Every command type needs a `payload_schema`, which is more registration text and one more thing to keep in step with the owner's `validate`.
- GDScript cannot make a constructor private, so the "only the kernel constructs commands" rule rests on a run-time seal plus a CI lint rather than the language.
- `payload_schema` is an addition to the kernel GDD's registration list (CR-6) and needs that GDD amended.
- The `var_to_bytes(...).size()` check allocates once per submit; this is fine at command rates (a few tens per second) but would not be at per-tick rates, so no system may submit per tick.

## Risks
- **A system submits per tick (for example a held-button repeat) and the size check or queue becomes hot.** Mitigation: intents are change-based, aim is rate-limited to every 3 ticks, and the Adapter's rule is in ADR-0019; a per-issuer queue cap bounds the damage and `QUEUE_FULL` is visible.
- **`payload_schema` drifts from an owner's `validate`.** Mitigation: schema covers shape only; STRICT_DEBUG round-trip and a registry-generated fuzz test submit each type's schema extremes.
- **The binding table drifts from its sources during `RUNNING` (a weld or pod hop changes an owned entity) and a command is wrongly rejected `ACTOR_NOT_OWNED`.** Today no pair changes during `RUNNING`: a weld does not renumber actors, a pod hop is Body's `transfer_actor` and keeps the actor id, and beams are created only in `LOADING`. Mitigation: half (b) is rebuilt on every `LOADING`; under `STRICT_DEBUG` the binding sources are re-run after each body-changing `apply` and compared with half (b), a mismatch being a `BINDING_DRIFT` violation; `block_id → source index` is resolved once at registration; a test submits a command immediately after each body-changing operation.
- **A coalescing scan is slower than expected.** Mitigation: bounded by `QUEUE_CAPACITY`; an index is the documented fallback (Alternative 6).
- **VS-B reveals that the peer id for the host is not 1 on 4.7.2.** Mitigation: `HOST_PEER_ID` is a constant in one place and the remote ingress adapter is the only reader; verification item (3).
- **A handle leaks to a presentation node, or is forged.** Mitigation: handle issue is named and logged, a CI check forbids passing a `SurfaceHandle` into any script under `src/presentation/` other than the surfaces named at issue, and a forged handle fails closed at `_admit`.
- **A freed result-handler target produces error spam at flush.** Mitigation: validity check per call, `clear_result_handler()` from `_exit_tree`, diagnostics count `RESULT_HANDLER_DEAD`.
- **A `Command` is retained past its callable, or forged.** Mitigation: nothing is pooled, so a retained reference stays valid but stale; records copy scalars; the sealed/`submission_id` check refuses a `Command` the queue did not admit.

## GDD Requirements Addressed

| GDD System | Requirement | How This ADR Addresses It |
|------------|-------------|--------------------------|
| command-simulation-layer.md | CR-4: commands are plain data stamped by the kernel (`issuer_peer`, `origin`, `seq`, `submission_id`), never payload-borne; no constructor without `issuer_peer` (TR-cmd-008) | § 2: only `_admit` builds and fills a `Command`; lint forbids `Command.new()` elsewhere |
| command-simulation-layer.md | CR-5: `submit()` is fire-and-forget; exactly one terminal result per command, flushed at P5 and once per frame (TR-cmd-009) | § 2 and § 7 |
| command-simulation-layer.md | CR-4 payload whitelist and `MAX_PAYLOAD` (TR-cmd-010) | § 2 order of checks; § 3 adds per-type shape and range on top |
| command-simulation-layer.md | CR-7: origin and actor-binding authority check at P2; single-player is host peer 1 → PlayerId 1 (TR-cmd-012) | § 4 |
| command-simulation-layer.md | CR-7, CR-18: the kernel rebuilds binding half (b) on every `LOADING` pass (TR-cmd-013) | § 4 states the data structure; the rebuild order is ADR-0016 |
| command-simulation-layer.md | CR-13: apply order ascending `(queued_tick, host_order)`; identical stream gives identical hash (TR-cmd-027) | § 5; hashing is ADR-0005 |
| command-simulation-layer.md | CR-15: control plane is host-local, served at the next boundary in arrival order (TR-cmd-028) | § 6 |
| command-simulation-layer.md | CR-16: pause is single-player only; only `allowed_when_paused` types, non-gameplay blocks (TR-cmd-030) | § 6 |
| command-simulation-layer.md | CR-17: queue, per-issuer and payload bounds are boot assertions; `QUEUE_FULL`, never a silent drop (TR-cmd-031) | § 5 |
| command-simulation-layer.md | HOST/MIRROR share an identical public API (TR-cmd-034, API identity only; the block-descriptor `repl_hint` field is ADR-0003) | § 8; replication shape is ADR-0032 |
| player-controller-input.md | PC command authority: a command whose issuer does not own the actor is `Rejected(ACTOR_NOT_OWNED)`; opaque `correlation` echoed to the caller (TR-pc-024) | § 4 and § 7 |
| interaction-system.md | IA-40: while paused no diegetic verb is admitted (`PAUSED`, never queued) (TR-interact-029) | § 6 |
| chart-table-web-map-and-menus.md | `PauseGate` is the single adapter to the control plane and re-issues the desired pause state until the kernel matches (TR-charttable-003, TR-charttable-004) | § 1 (control handle) and § 6 |
| sanity-and-hallucination-rules.md, suit-and-equipment.md, interaction-system.md | Persistent per-player state keyed by PlayerId, never peer id | § 4; Implementation Guidelines |

## Performance Implications
- **CPU**: Admission is O(1) per submit apart from the payload walk (bounded by 4 KiB and depth 8) and, for coalescing types only, a scan of at most 1,024 slots. At the admitted rates (a few tens of commands per second) this is far inside the kernel's budgets (frame simulation ≤ 4.0 ms P99; P3 ≤ 2.0 ms), measured on release export on RM-DESKTOP and RM-DECK. The P2 ownership check is a scan of a few `int64`s.
- **Memory**: The queue ring is preallocated (`QUEUE_CAPACITY` slots); the joint queue and command-log bound is a boot assertion against `COMMAND_MEMORY_BUDGET` (16 MiB). One `Command` object per admitted submit (allocated, not pooled); `PackedInt64Array`s for the binding table.
- **Load Time**: Rebuilding binding half (b) on `LOADING` is O(players × entities) and runs inside the existing atomic boundary service.
- **Network**: None at MVP. The remote ingress adapter and rate limiting (`REMOTE_RATE_LIMIT`, `RATE_WINDOW_TICKS`) are VS-B.

## Migration Plan
No code exists. The document changes are: amend the kernel GDD (CR-4, CR-6, CR-10, CR-15, States and Transitions; § GDD sync); in `docs/architecture/architecture.md` replace the `Kernel.submit` example in § API Boundaries with a surface-handle call and correct § Data Flow 2, which says P1 stamps `issuer_peer`, `seq` and `submission_id` (the GDD and this ADR stamp them at `_admit`, before admission); and in ADR-0001 replace the `kernel.submit(...)` line in its diagram with `surface.submit(...)`.

## Validation Criteria
- A command submitted by a surface handle carries `origin = PLAYER` and `issuer_peer = 1`; a payload containing the keys `issuer_peer`, `origin` or `seq` is `MALFORMED` (not in the schema) and the stamped fields are unchanged.
- A grep finds no `Command.new()` outside `src/kernel/`, no `submit(` call in any `src/presentation/` script other than the surfaces named at issue, no call to a `_`-prefixed kernel member from outside `src/kernel/`, and no `request_*` call outside the named scripts (`save_load`, `pause_gate`, `game.gd`).
- A command type registered without `payload_schema` (or with any other mandatory field missing) fails boot naming the type, in debug and release builds.
- A PLAYER command acting on an entity not in the issuer's binding half (b) is `Rejected(ACTOR_NOT_OWNED)` with a state hash identical before and after.
- Two surfaces submitting in the same frame get distinct dense `seq` values; only the test ingress can produce `DUPLICATE_SEQ`.
- Filling the queue yields `QUEUE_FULL` for the next submit and never drops an already admitted command; a SYSTEM command is still admitted at `PER_ISSUER_CAPACITY` for PLAYER.
- `request_pause` and `request_resume` issued in the same boundary leave the kernel `RUNNING`; `PauseGate` converges in one more boundary.
- A MIRROR kernel refuses to step or apply (unit test), and the same registry loads in both modes.

## GDD sync
**Status 2026-10-28: the five kernel-GDD items below are applied** to `design/gdd/command-simulation-layer.md` (CR-4, CR-5, CR-6, CR-7, CR-15 and the States and Transitions sentence). The per-command-owner item stays open by design: each owner lists its `payload_schema` when its stories are written. Acceptance still waits on the review noted in the Ordering Note. What was needed:
- `design/gdd/command-simulation-layer.md` CR-6 (registration field list) and CR-4: add the mandatory `payload_schema` field (shape and range only). Owner GDD, so this ADR's addition becomes the spec.
- `design/gdd/command-simulation-layer.md` CR-4 / CR-10: record that `submit_system` is exposed on `HookContext`/`ApplyContext` and the test ingress, not as a free function, and that submit rights are capability handles issued in `BOOTING`.
- `design/gdd/command-simulation-layer.md` CR-5: record the **normative admission order** in § Admission above (type resolved before the kernel-state check; capacity after coalescing).
- `design/gdd/command-simulation-layer.md` CR-7 and CR-15: name `request_bind_seat` / `request_unbind_seat` as the host-only writers of binding half (a); and amend the States and Transitions sentence "control-plane requests have none" to say outcomes are delivered to the requester's outcome handler in a list separate from the CR-5 command result buffer.
- `design/gdd/command-simulation-layer.md` CR-5/CR-16: the queue-flush `Dropped(...)` results on entry to `LOADING`, `SHUTTING_DOWN` and `FAULTED` are already there; state that superseded tombstones free capacity.
- Per-command owners (Controller, Interaction, Suit, Emitter, Assembly, Web Map, Sanity) list each command's `payload_schema` in their GDD when their stories are written (ADR-0020, -0021, -0026, -0027, -0019).

## Related
- `design/gdd/command-simulation-layer.md` (CR-4 to CR-10, CR-13, CR-15 to CR-17, CR-19; D.8)
- `docs/architecture/architecture.md` (§ Data Flow 2, § API Boundaries, OQ-13)
- ADR-0001 (injected kernel; frame order), ADR-0003, ADR-0005, ADR-0016, ADR-0019, ADR-0030, ADR-0032 (to be written)
- Review rulings: delegated ruling R2 (`seq` and `correlation`), G1 (owner wins)
