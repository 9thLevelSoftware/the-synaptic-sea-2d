# ADR-0003: State-Block Registry, Write Discipline and Owner-Callable Contract

## Status
Proposed

## Date
2026-10-28

## Last Verified
2026-10-28 (against `design/gdd/command-simulation-layer.md` CR-2, CR-3, CR-6, CR-8, CR-9, CR-12, CR-14, CR-18, CR-19 and States and Transitions, and `docs/engine-reference/godot/breaking-changes.md` and `current-best-practices.md` 4.5–4.7). No code exists; nothing here has been run on 4.7.2.

## Decision Makers
User (delegated all remaining design calls, 2026-10-28); Claude (author). Reviewed by godot-specialist (engine: design idiomatic and implementable; three verification and wording items and twelve minor notes, all applied) and technical-director (TD-ADR: CONCERNS; finding A1 and amendments B1–B11, C and E applied in this revision). Amended 2026-10-28 by the Foundation-ADR adjudication (TD-07, TD-01, TD-08): see § Amendments.

## Summary
About fifteen systems register state blocks, step hooks and command handlers with the kernel, and the kernel must call them, check that they finished, and let them write only inside a kernel-issued scope. This ADR fixes that contract in GDScript: owners are plain objects that register typed `Callable`s (no base class), every kernel call receives a reused context and ends in `return ctx.complete()`, all writes and dirty marks go through a reused `WriteScope` (including during new-game construction), faults are integers with static per-system domains, hooks run from a precomputed schedule, and the kernel state machine is a data table that generates its own test.

## Engine Compatibility

| Field | Value |
|-------|-------|
| **Engine** | Godot 4.7.2 (stable), GDScript |
| **Domain** | Core (scripting contracts, callable invocation) |
| **Knowledge Risk** | HIGH for the owner-callable contract. 4.7 made an override of a typed-return method inherit the return type (it needs an explicit `return`) and stopped packed-array element writes from calling the property setter; both bear on this contract (confirmed in `breaking-changes.md`). What `Callable.call()` returns after the callee aborts on a runtime error is documented only in the kernel GDD (CR-19), not in the pinned engine reference. |
| **References Consulted** | `docs/engine-reference/godot/VERSION.md`, `breaking-changes.md` (4.6 → 4.7 GDScript rows), `deprecated-apis.md`, `current-best-practices.md` (GDScript 4.5+ and 4.7: `@abstract`, typed-return override rule, packed-array element writes, lambda capture by value, release-build script backtracing) |
| **Post-Cutoff APIs Used** | None required. `@abstract` (4.5) is deliberately not used. The 4.7 packed-array element-write change is avoided by marking dirty keys explicitly in mutators. `Callable.get_argument_count()` is used for a boot-time signature check (present on 4.7.2, probed 2026-10-28). |
| **Verification Required** | (1) **Gates acceptance.** On a **release export**, a registered owner `Callable` typed `-> CompletionToken` that aborts on a script error returns something that is not the armed token, so `INCOMPLETE` is detected; and a typed function with a path that never reaches `return` is rejected at parse time ("not all code paths return a value"). Both are UNVERIFIED against the reference and are proved by KRN-CTX-06. Release templates compile out many VM checks, so a callee may instead log the error, carry on and still return `ctx.complete()`; KRN-CTX-06 records which of the two outcomes the release export shows, and fail-closed must hold under both (§ 5). (2) `assert()` is stripped in release; a lint plus test (1) prove no control flow depends on it. (3) **Settled, probed 2026-10-28:** a `Callable` bound to a `RefCounted` method holds only the object's id and does **not** keep it alive; when the last strong reference drops the owner is freed and `is_valid()` turns false. The composition root therefore holds every owner in an explicit strong-reference list (Guidelines). (4) Packed-array copy semantics for the schedule and dirty set (S0-a). (5) `Callable.call()` cost with a context argument at the worst-case calls per tick (15 hooks plus the command batch) fits the P3 budget, and `PackedInt32Array` element reads allocate nothing (S1). |

## ADR Dependencies

| Field | Value |
|-------|-------|
| **Depends On** | ADR-0001 (injected facets, frame order; Proposed), ADR-0002 (command-type descriptor meaning, handles; Proposed) |
| **Enables** | ADR-0004 (notification consumes the dirty keys collected here), ADR-0005 (canonical encode and hash through `snapshot`), ADR-0006 (snapshot and restore callables), ADR-0009 (the two-face `PresentationViews`, `NotifyContext`), ADR-0016 (restore ordering; consumes `restore_after`), ADR-0031 (diagnostics channel descriptor), and every system ADR that registers a block or hook (ADR-0011, 0013, 0015, 0017, 0019, 0020 to 0027) |
| **Blocks** | Every story that registers a block, hook or command type. |
| **Ordering Note** | Scope edge with ADR-0002: ADR-0002 owns command-field meaning, ingress and the control plane; **this ADR states the complete `CommandTypeDescriptor`** and moves `register_command` onto the `BlockRegistrar` (ADR-0002 is amended to match, together with its result-handler signature, finding B5). Scope edge with ADR-0004: the dirty key is an opaque `int` here; ADR-0004 fixes its layout. The kernel GDD needs amendments listed in § GDD sync (epoch token; `FAULTED` re-entry qualifier; long-lived read views; fault-code table). **Registry amendments requested by ADR-0004, ADR-0005, ADR-0006 and ADR-0009 are applied here as one batch (§ Amendments, TD-07):** `register_key_space`, `emit_event`, `has_opened`, five violation names, the `touch_serial` and hash-cache registries, the `kernel.clock` view, `ModuleFacet.rng`, the kernel-internal hash `SnapshotContext`, the two-face `PresentationViews` and the host-service registrar facet. **ADR-0003 owns `kernel.clock`** (it is on the H1 allow-list; ADR-0001 does not define it); ADR-0005's attribution of the clock view to ADR-0001 is corrected in ADR-0005. `NotifyContext` is placed in `src/kernel/facets/` (ADR-0010, TD-01). |

## Context

### Problem Statement
The kernel GDD defines the owner contract in prose: eleven kernel-invoked callables, a completion sentinel, a fault channel, a write scope, a hook ordering rule and a seven-state machine. Every owner GDD (Graph, Layout, Body, Emitter, Controller, Interaction, Suit, Assembly, Web Map, Attention, Horror, Sanity) already assumes it. Missing is the code-level shape: whether owners subclass something or register callables; what the contexts, the token and the scope are and how they are reused; how writes happen while a new world is being built; how a fault code is represented and who owns the table; how the hook order becomes a per-tick schedule without a scan; and how the state table is expressed so the generated state test (KRN-STATE-01) is mechanical. Without one answer, fifteen systems would build fifteen variants of a contract whose value is uniformity.

### Constraints
- **GDScript has no exceptions** and `assert()` is compiled out of release (CR-19). Control flow never depends on either.
- **Typed-return override rule (4.7).** An override of a typed-return method inherits the type; a `void` function cannot return the token.
- **Block base type is `RefCounted`**, never `Resource` (a `.tres` can embed scripts; Save/Load SL-2) or `Node` (kernel H10).
- **Write discipline (CR-9, H3).** One owner per block; writes only inside a `WriteScope`; a write outside is refused in every build, counted and reported.
- **New-game construction writes (CR-18).** Each `init_world` step runs "under one `WriteScope`, before `post_restore()`", and Web Map's `init_world` calls Layout, Emitter and Interaction mutators.
- **No rollback (CR-9).** Validate-then-apply; a fault inside `apply` is a defect that moves the kernel to `FAULTED`.
- **Packed-array element writes call no setter (4.7).** Dirty tracking cannot ride a setter.
- **Determinism (CR-13).** Hook order and registration order are pure functions of registration data; iteration is over sorted arrays.
- **Performance (R12, TR-cmd-032).** P3 ≤ 2.0 ms P99, `STEP_BUDGET_MS` 4 ms alarm; a quiet tick performs zero kernel container allocations.
- **Kernel H1.** The kernel's members are an allow-listed set (`kernel.clock`, the queue, the registries, the binding tables, the command log, kernel state, diagnostics); an acceptance criterion lints it.
- **Registration only in `BOOTING`** (CR-6), frozen before the first `LOADING`.

### Requirements
- One registration surface and one calling convention for blocks, hooks, command handlers, binding sources, fault domains and diagnostics channels.
- The kernel can prove after every call, in release builds, that the owner reached `return ctx.complete()`.
- A mutator cannot write or mark a dirty key outside a kernel scope, including during new-game construction.
- Faults identify their origin stably across builds and print as names.
- The hook schedule is data, printable, and costs no scan per tick.
- The state machine is a table that generates its own test.
- Quiet ticks allocate nothing in the kernel.

## Decision

### 1. Owners register `Callable`s; owners override nothing

An owner is any object (normally a `RefCounted`) that, during `BOOTING`, passes the kernel descriptors containing `Callable`s bound to its methods. There is no `BlockOwner` base class and nothing is overridden. Each owner method the kernel calls is declared `-> CompletionToken` and its last statement is `return ctx.complete()`.

The 4.7 typed-return override rule applies only to overrides, so it does not touch owner methods. The context classes of § 3 add methods to their base and **override none**; a Guideline forbids overriding a context method, because an override of `complete() -> CompletionToken` would fall under the rule. This also avoids `@abstract` (documented since 4.5, not needed) and lets tests register a plain stub. **Every override of a typed-return owner callable returns on all paths** *(Amended 2026-10-28, TD-07)*: a stub or subclass that overrides an owner callable inherits `-> CompletionToken` (4.7 rule) and must end every branch in `return ctx.complete()`; an early `return` without it is an `INCOMPLETE`, and a path that falls off the end is a parse error or an `INCOMPLETE` by Verification 1. That GDScript rejects at parse time a typed function with a path that has no `return` is UNVERIFIED (Verification 1); whatever it shows, the token check (§ 3) is what catches a function that aborts at run time. At registration the kernel checks every `Callable` with `is_valid()` and, where available, `get_argument_count()` against the expected arity, so a wrong signature fails boot naming the registration rather than appearing later as `INCOMPLETE`.

### 2. A block is a `RefCounted` held by its owner; the kernel holds descriptors

The kernel never stores or inspects block data. It stores descriptors and calls the owner's callables. The owner's block object is a `RefCounted` with typed fields (packed arrays for struct-of-arrays state), never a `Resource` or `Node`. **`restore` copies into the existing block object and never replaces it**, because read views are long-lived facades over that object (§ 7); a test restores a block and checks its facade sees the restored values.

**Snapshot payload contract** *(Amended 2026-10-28, TD-07; ADR-0005 § 5.3, ADR-0006)*: every packed array an owner puts in a `snapshot` payload is a **duplicate** (`PackedXArray.duplicate()` or a freshly built array), never a live reference to the block's own array. The kernel reads the payload after `snapshot` returns, hashes it outside SAVING (ADR-0005) and may hold it while the owner keeps writing; a live reference would let a later mutator change a payload already encoded or queued for the writer. `restore` likewise copies out of `data` and keeps no reference into it. A test mutates the block after `snapshot` and checks the payload unchanged.

Every field is mandatory unless marked optional; a missing field fails boot naming the descriptor (CR-6: no silent defaults). GDScript reserves `class` as a keyword, so the GDD's `class` field is `block_class` in code.

**`BlockDescriptor`**

| Field | Type | Meaning |
|---|---|---|
| `block_id` | `StringName` | Unique, namespaced (e.g. `&"graph.state"`) |
| `owner_name` | `StringName` | Registering system, for diagnostics |
| `schema_version` | `int` | Bumped on any serialised-shape change |
| `supported_versions` | `PackedInt32Array` | Versions `restore` accepts |
| `block_class` | `int` enum `SIM` / `PERCEPTION_OUT` | CR-12 |
| `persist` | `bool` | false = derived, rebuilt after load |
| `persist_scope` | `int` enum `WORLD` / `PROFILE` | Required when `persist` |
| `scope` | `int` enum `SHARED` / `PER_PLAYER` | PER_PLAYER keyed by PlayerId |
| `gameplay` | `bool` | The paused-write test (CR-16) |
| `interpolated` | `bool` | Kernel runs `commit_baseline` at P4 |
| `repl_hint` | `int` enum `HOST_ONLY` / `SHARED_SUMMARY` / `OWNER_PRIVATE` | Unused until VS-B, mandatory now |
| `restore_after` | `Array[StringName]` | DAG for ADR-0016 |
| `snapshot`, `restore`, `default_state`, `post_restore`, `read_view`, `empty_view` | `Callable` | Mandatory |
| `commit_baseline`, `snap_baseline` | `Callable` | Mandatory iff `interpolated` |
| `init_world`, `binding_source` | `Callable` | Optional (the binding source is this field only, never a separate registration) |

**`CommandTypeDescriptor`** (complete; ADR-0002 owns the meaning of each field)

| Field | Type |
|---|---|
| `type` | `StringName` |
| `owner_name` | `StringName` |
| `allowed_origins` | `int` bitmask (`ORIGIN_PLAYER = 1`, `ORIGIN_SYSTEM = 2`) |
| `actor_binding` | `int` enum `NONE` / `OWNED_BY_ISSUER` |
| `actor_source` | `StringName` (block id; empty iff `NONE`) |
| `coalesce` | `int` enum `NONE` / `LATEST_PER_ACTOR_PER_TICK` |
| `max_age_ticks` | `int` |
| `allowed_when_paused` | `bool` |
| `writes_blocks` | `Array[StringName]` |
| `payload_schema` | `Dictionary` (authoring form; compiled at registration, ADR-0002 § 3) |
| `validate`, `apply` | `Callable` |

**`HookDescriptor`**: `hook_id: StringName`, `owner_name: StringName`, `divisor: int` (≥ 1), `offset: int` (0 ≤ offset < divisor), `after: Array[Dictionary]` each `{&"id": StringName, &"tolerates_stale": bool}`, `callable: Callable`.

**`DiagnosticsChannelDescriptor`** (CR-6 via the Debug Overlay's E-DO-38): the field list is ADR-0031's; registration is here.

### 3. Contexts: reused, re-armed per call, non-re-entrant

Every kernel-invoked callable receives a context as its last argument and reports only through it. The kernel allocates **one context object per kind at boot and re-arms it before every call**; nothing is allocated per call. *(Amended 2026-10-28, TD-07; ADR-0005 § 5.3.)* **One further kernel-internal `SnapshotContext`** serves hash reads (`block_hash`, `state_hash`) made **outside `SAVING`**, so a hash read never re-arms the context a save capture or a restore is using. It carries its own re-entrancy flag: a `block_hash` call made from inside a `snapshot` call (which would arm the same context twice) is a `CONTEXT_REENTRY` violation and returns a not-ok result, never a hash. It is a kernel member under H1 `kernel_state` and is never exposed to owners.

```gdscript
class_name CallContext extends RefCounted          # base; subclasses add methods and override none
var completed: bool
var epoch: int
var faulted: bool:                                  # true if fault() was called here OR on the linked WriteScope
	get: return _faulted or (_scope != null and _scope.faulted)
func put(payload: Variant) -> void                  # snapshot / default_state / read_view / empty_view / binding_source
func fault(code: int, detail: Dictionary = EMPTY_DETAIL) -> void   # packed FAULT_CODE (§ 6); EMPTY_DETAIL is shared and never mutated
func complete() -> CompletionToken                  # sets completed, stamps the armed token's epoch, returns it

class_name ValidateContext extends CallContext      # validate: read-only
func ok() -> void                                   # first outcome stands; a second is CTX_DOUBLE_OUTCOME
func reject(reason: StringName) -> void             # -> Rejected(VALIDATION_FAILED(reason))

class_name TickContext extends CallContext          # shared base of the two contexts that may submit
var tick: int
func submit_system(type: StringName, payload: Dictionary, dedupe_key: Variant = null) -> void
func has_pending(type: StringName, dedupe_key: Variant = null) -> bool
class_name ApplyContext extends TickContext         # apply
class_name HookContext extends TickContext          # step hooks; adds hook_id
class_name SnapshotContext extends CallContext      # snapshot, restore, post_restore, default_state, init_world,
                                                    # commit_baseline, snap_baseline, binding_source, read_view, empty_view
class_name NotifyContext extends CallContext        # subscriber, result and control-outcome handlers (a fault is counted, never fatal)
                                                    # file: src/kernel/facets/notify_context.gd (ADR-0010 § 1.1, TD-01): a presenter handler must name it
```

`submit_system` exists only on `TickContext`, so `validate` cannot submit (it receives a `ValidateContext`) and no "refused call" path is needed.

**Placement of `NotifyContext`** *(Amended 2026-10-28, TD-01)*: the subscriber, result and control-outcome handler context lives in `src/kernel/facets/` with `CompletionToken`, `ChangeBatch`, `NotifyEvent` and the other types a presenter must name (ADR-0010 § 1.1, ADR-0004 placement table), because a presenter's handler signature `(batch: ChangeBatch, ctx: NotifyContext) -> CompletionToken` has to compile in `src/presentation/`. The tick-phase contexts (`CallContext`, `ValidateContext`, `ApplyContext`, `HookContext`, `SnapshotContext`) stay in kernel core and are on ADR-0010's presentation deny list; there is no `*Context` glob.

**Re-arm list.** Before each call the kernel, in one step: increments `epoch`; sets `completed = false`, `_faulted = false`, the outcome (`ok`/`reject`) to none, the `put` payload to none and the fault detail to `EMPTY_DETAIL`; links the context to the call's scope (or to none); and sets a re-entrancy flag. After the call it clears the flag. A call made while the flag is set is a `CONTEXT_REENTRY` violation; the MVP needs one context per kind, because a hook's `submit_system` queues and does not call another kernel callable synchronously.

**The token check.** The kernel keeps one `CompletionToken` per context object. `complete()` sets `completed = true`, writes the current epoch into the token and returns it. After the call the kernel accepts the result only if the returned value **is** that token (identity), `token.epoch == ctx.epoch` and `ctx.completed` is true. The kernel holds the call result as a `Variant` and tests identity with `is_same()`, so a coroutine state returned by an owner that hit `await` fails the check like any other non-token value. Anything else, including `null`, a type default, a coroutine state or a token kept from an earlier call, is `fault(INCOMPLETE)` (or, for a handler or a view callable, a recorded `INCOMPLETE` diagnostic without a fault). The check does not depend on what an aborted call returns (Verification 1). One hole is closed by rule: an owner that aborted *after* `complete()` ran would read as success, so `return ctx.complete()` is the **last statement** and nothing follows `complete()` in the same expression (a CI check). A `validate` that calls neither `ok()` nor `reject()` is `Faulted(NO_OUTCOME)`. This satisfies the GDD's "fresh `ctx.token`" guarantee without allocating; the GDD wording is amended (§ GDD sync).

### 4. `WriteScope`: reused, the only way to write and to mark dirty

```gdscript
class_name WriteScope extends RefCounted
enum Phase { CLOSED, APPLY, HOOK, PAUSED_APPLY, INIT, RESTORE }
var phase: Phase
var faulted: bool                                   # set by fault(); closes the scope (fail-closed)
func open_for(block_id: StringName) -> bool         # see the table below
func mark_dirty(block_id: StringName, key: int) -> void   # key layout: ADR-0004 § 3
func emit_event(block_id: StringName, kind: int, a0: int = 0, a1: int = 0, a2: int = 0, a3: int = 0, a4: int = 0, a5: int = 0) -> void   # P5 event record; ADR-0004 § 5
func has_opened(block_index: int) -> bool           # true from the first granted open_for of that block until the scope is re-armed (ADR-0005 § 5.3)
func fault(code: int, detail: Dictionary = EMPTY_DETAIL) -> void
```

The kernel allocates the scope objects once and re-arms them (phase, writable set, `faulted = false`, dirty buffer) per use, like the contexts.

| Phase | When | `open_for(block)` true for |
|---|---|---|
| `APPLY` | P2 `apply` of one command (`RUNNING`) | any block (cross-block applies call other owners' published mutators) |
| `PAUSED_APPLY` | the paused service cycle | only blocks in the command type's `writes_blocks` (GDD CR-16) |
| `HOOK` | one P3 hook run | any block |
| `INIT` | new-game construction in `LOADING`: all `init_world` steps share one `INIT` scope, before `post_restore` (CR-18) | any block |
| `RESTORE` | an owner's `restore` / `post_restore` that uses its own mutators | the restoring block only |
| `CLOSED` | everything else, and after the first `fault` | nothing |

`init_world` therefore receives the scope: `init_world(world_seed: int, scope: WriteScope, ctx: SnapshotContext)`, and Web Map's `init_world` passes it to Layout `build_section`, Emitter `instantiate_section_emitters` and Interaction `sync_section` (architecture § Initialisation order step 6). **Dirty keys marked in `INIT` and `RESTORE` scopes are discarded**: on the `LOADING → RUNNING` transition the kernel delivers one `ALL` batch per block instead (the GDD's "one ALL then silence"), which is what the PROJECTOR needs after a new game or a load; ADR-0004 defines that delivery.

`open_for` returning false writes nothing, counts `WRITE_OUTSIDE_SCOPE` and reports a violation (CR-9) — except when the scope is closed because of an earlier fault, which is not counted again. Dirty keys are marked **only** through `scope.mark_dirty` inside the mutator that changed the data, never from a setter. The debug-build check that catches a missed mark is ADR-0004 § 7 (it compares a block's hash with the kernel's `touch_serial` and with whether any key was marked, for blocks that do not declare `quiet_writes_allowed`); this ADR no longer states the rule. `emit_event` follows the same scope rules as `mark_dirty`: it needs an open block and its records are discarded in `INIT`, `RESTORE` and `CLOSED` (ADR-0004 § 5). `has_opened` is a pure read used by the kernel's hash path (ADR-0005 § 5.3): it never opens anything and never counts a violation. `writes_blocks` is enforced in `PAUSED_APPLY` as the GDD requires; under `STRICT_DEBUG` it is also checked during `RUNNING`, where a write to a block outside `writes_blocks` is a `WRITES_BLOCKS_UNDECLARED` violation, to catch undeclared cross-block writes early.

### 5. Mutators: status returns, fail-closed, no rollback

**Status values**: `OK = 0`; `REFUSED = 1` (the scope was not open for this block: a counted violation, **not** a fault, and never escalated by the caller); a **negative** value is `-fault_code` (a defect the mutator detected; it has also called `scope.fault`).

- **Class A** (validate-then-apply): a pure `check_*(views, args) -> StringName` (`&""` = OK) takes read views only; the matching mutator `name(scope, args, out) -> int` is infallible once its check passed. A mutator that produces data (a new id, a `{seam_id, opening_ids}` record) writes it into a **caller-owned `out` object** passed in, so the return value is always the status. Example: `weld_join(scope: WriteScope, plan: JoinPlan, out: WeldResult) -> int`.
- **Class B** (clamp, never reject): `name(scope, args) -> void`; it clamps out-of-range input and no-ops on a stale id; it still begins with `scope.open_for`.

**Fail-closed.** The first `scope.fault` sets `scope.faulted`, which closes the scope: every later `open_for` on it is false (not counted again), so later mutators in the same apply write nothing and the dev-build diagnostics show the blocks exactly as they were at the fault. `ctx.faulted` includes the scope's flag (§ 3). **Fail-closed never depends on an engine error aborting the callee** *(Amended 2026-10-28 — architecture review)*: a release build may log a script error and continue. It is carried by the explicit paths only — mutator status checks, `scope.fault` and the closed scope — with the `STRICT_DEBUG` checks and ADR-0004's debug-build missed-mark hash as backstops. An engine error that a release build swallows and that no mutator detects is outside what the kernel can see; KRN-CTX-06 documents it rather than claims it.

**Caller procedure.** After every Class A call the caller checks the status. On `REFUSED` or a negative status it returns `ctx.complete()` at once: no further mutator calls and no second `ctx.fault` (the scope fault already counts). The kernel reads `ctx.faulted` after the callable returns.

**No rollback.** A fault leaves the blocks touched so far half-applied. This is acceptable because `FAULTED` is terminal until an explicit reload or new game (both rebuild every `WORLD` block) and the kernel never writes or autosaves over an existing save in `FAULTED` (GDD). Nothing here rules out OQ-15's alternative for the weld: ADR-0021 may snapshot the touched blocks before its apply and restore them on a fault, using the `snapshot` / `restore` callables this ADR defines.

### 6. Fault codes: static domains, pre-packed constants

A fault code is a positive `int`: `(DOMAIN_ID << 16) | local`, `local` in `1..65535`. **Domain ids are static**, so a code means the same thing in every build, log and fixture. Each system's `*_faults.gd` (a `class_name XFaults extends RefCounted` holding only constants, never instantiated, one class per file) declares `const DOMAIN_ID` and its codes **already packed** (`const NOT_IN_SCOPE: int = (DOMAIN_ID << 16) | 1`), plus `const NAMES: Array[StringName]` indexed by local code. It registers with `register_fault_domain(XFaults.DOMAIN_ID, owner_name, XFaults.NAMES)`; a duplicate or out-of-range id fails boot. A unit test checks each file's constants and `NAMES` agree. Codes are append-only and never reused. The kernel prints a fault as `domain.NAME`.

**Domain table** (static; new systems take the next free id; ids 32 and up are reserved for VS and later)

| Id | Domain | Id | Domain |
|---|---|---|---|
| 0 | kernel | 8 | interaction |
| 1 | save_load | 9 | suit |
| 2 | content data (item DB, templates, tuning) | 10 | assembly |
| 3 | graph | 11 | web_transit |
| 4 | layout | 12 | attention |
| 5 | body | 13 | horror |
| 6 | emitter | 14 | sanity |
| 7 | controller | 15–31 | presentation and tooling (read-view faults are diagnostics only) |

**Kernel fault codes (domain 0, closed)**

| Local | Name | Raised when |
|---|---|---|
| 1 | `INCOMPLETE` | a kernel-invoked callable did not return the armed token |
| 2 | `NO_OUTCOME` | `validate` called neither `ok()` nor `reject()` |
| 3 | `BINDING_REBUILD_CONFLICT` | the `LOADING` binding rebuild found a conflict or a binding-source fault |
| 4 | `RESTORE_UNKNOWN_BLOCK` | a restore payload names an unregistered block |
| 5 | `TICK_RATE_MISMATCH` | a payload was saved at another `TICK_RATE` |
| 6 | `RESTORE_EPOCH_RANGE` | the supplied `restore_epoch` is out of range |
| 7 | `CAPTURE_ERROR` | a `SAVING` capture failed |
| 8 | `DUPLICATE_ID` | boot: duplicate block, command, hook, domain or channel id |
| 9 | `HOOK_CYCLE` | boot: a cycle in hook `after` edges |
| 10 | `UNKNOWN_DEPENDENCY` | boot: unknown `after` hook, `restore_after` block or `writes_blocks` block, or a subscription to an unknown block (ADR-0004 § 2) |
| 11 | `ACTOR_SOURCE_INVALID` | boot: a command type's `actor_source` rule fails (CR-6) |
| 12 | `COALESCE_DUAL_ORIGIN` | boot: a coalescing type allows both origins |
| 13 | `DESCRIPTOR_FIELD_MISSING` | boot: a mandatory descriptor field is absent |
| 14 | `SCHEDULE_CYCLE_TOO_LONG` | boot: `lcm` of divisors exceeds `MAX_SCHEDULE_CYCLE_TICKS` |
| 15 | `HOOK_OFFSET_INVALID` | boot: `divisor < 1` or `offset` not in `0..divisor-1` |
| 16 | `CALLABLE_SIGNATURE` | boot: a `Callable` is invalid or has the wrong arity |
| 17 | `MEMORY_BUDGET` | boot: the CR-17 joint memory assertion fails |

A P3 above `STEP_BUDGET_MS` is a **diagnostic** (it logs the slowest hooks), not a fault.

**Violations** are counted diagnostics, not fault codes, and never change kernel state unless the GDD says so. Closed set: `WRITE_OUTSIDE_SCOPE`, `PERCEPTION_READ_BLOCKED`, `SIM_WRITE_TRIPWIRE`, `CTX_DOUBLE_OUTCOME`, `CONTEXT_REENTRY`, `SUBMIT_DURING_SAVING`, `HANDLE_UNREGISTERED`, `RESULT_HANDLER_DEAD`, `DEAD_SUBSCRIBER`, `TIME_SCALE_NON_UNIT`, `KNOB_REFUSED`, `BINDING_DRIFT`, `WRITES_BLOCKS_UNDECLARED`, `STALE_EDGE_UNDECLARED` (a boot warning), and, added 2026-10-28 and classified as ADR-0004 §§ 2, 3, 5 and 7 define them, `SUBSCRIBE_REFUSED` (a subscribe-time refusal, including the `PERCEPTION_OUT` rule of check 6), `DIRTY_KEY_INVALID`, `DIRTY_KEY_KIND_UNDECLARED`, `EVENT_OVERFLOW` and `MISSED_DIRTY_MARK` (dev, playtest and test builds only). Under `STRICT_DEBUG` each also calls `violation_handler(code, detail)`.

Each owner's local code list is written with its system ADR (ADR-0011 onward); its domain id is fixed now.

### 7. Read views: long-lived facades, issued by facet

The kernel never hands out a live container. **Each owner's `read_view(ctx)` is called once, at the registry freeze**, and delivers a long-lived getter-only facade over its block object (typed scalar getters and caller-owned-buffer queries, so hot paths allocate nothing). Each owner also supplies **`empty_view(ctx)`**: a facade of the same class whose getters return neutral values (zero, false, empty) and whose `is_empty()` is true. This replaces the GDD's per-request `read_view` (§ GDD sync); it is valid only because `restore` copies into the existing block object (§ 2).

`ReadViews` has two faces, issued as facets by the composition root (ADR-0001): **`SimViews`** (to `validate`, `apply` and hooks) and **`PresentationViews`** (to subscribers). Both have `get(block_id: StringName) -> Object`, returning the owner's facade, which the caller casts to the owner's view class. For a `PERCEPTION_OUT` block, `SimViews.get` returns that block's typed `empty_view`, increments `PERCEPTION_READ_BLOCKED` and reports a violation; `PresentationViews` returns the real facade (superseded for `PERCEPTION_OUT` by the two-face rule below). **Two faces of `PresentationViews`** *(Amended 2026-10-28, TD-07 item 10 and TD-08; ADR-0004 § 2 check 6, ADR-0009 § 1)*: `PresentationViews` carries a `perception_access: bool` that the composition root sets **once, at issue**, and presentation code cannot change. The **default face** (`perception_access == false`, every facet except the two below) answers `get(block_id)` for a `PERCEPTION_OUT` block with that block's typed `empty_view`, increments `PERCEPTION_READ_BLOCKED` and reports a violation, exactly as `SimViews` does. The **perception face** (`perception_access == true`) returns the real facade. ADR-0009 § 1 fixes which facets carry the flag: the Perception Filter's `PresentationFacet` and the `DIAGNOSTIC` facets only. The same flag gates the subscription: ADR-0004 § 2 check 6 refuses, at subscribe time and in the kernel, a subscription to a `PERCEPTION_OUT` block with `SUBSCRIBE_REFUSED` unless the issuing facet carries `perception_access = true`. A `SIM`-class block is readable through either face. ADR-0024 owns only the Filter's query surface, not this rule.

A fault in `read_view` or `empty_view` at the freeze is recorded and leaves the empty view in place; it never faults the kernel (CR-19(a)). Under `STRICT_DEBUG` the `SIM_WRITE_TRIPWIRE` hashes SIM blocks per dispatch batch and around P5 (`TRIPWIRE_BUDGET_MS` 1.0 ms, full sweep every `TRIPWIRE_SWEEP_TICKS` 60), which catches a facade that leaks a mutable reference.

### 8. Hooks: validated, sorted once, run from a precomputed schedule

At the registry validation pass (an internal step of the boundary that serves the first `request_new_game` / `request_restore`, CR-6) the kernel:

1. Fails boot on a duplicate id, an unknown `after` id, a cycle, `divisor < 1` or an `offset` outside `0..divisor-1` (an offset at or above its divisor would never fire).
2. Runs Kahn's algorithm with the ready list ordered by **registration index** (unique, so a total order; `hook_id` is the printed name), always taking the smallest ready index and never iterating a Dictionary.
3. Records a `STALE_EDGE_UNDECLARED` boot warning for any `after` edge between hooks that do not always run on the same tick unless it declares `tolerates_stale` (ruling R-8). The warning list must be empty at the end of every CI run (today it is not, because of `transit.step`, IC-11, until the Web Map GDD declares that edge).
4. **Compiles a schedule.** `C = lcm(all divisors)`, computed with a small integer `gcd`/`lcm` at boot (no builtin), guarding overflow; a `C` above `MAX_SCHEDULE_CYCLE_TICKS` (3,600) fails boot naming the divisor. For each residue `r` in `0..C-1` the schedule holds a `PackedInt32Array` of hook indices, in the total order, of the hooks with `r mod divisor == offset`. Each entry is built as a local array, filled completely, then assigned once and never mutated in place. With the MVP divisors (1, 5, 6, 12) `C` = 60. A tick does `schedule[T mod C]` and calls each index: no per-tick scan, and the schedule is data the Debug Overlay prints.

**P3 budget semantics (IC-4).** Settled by the owner and recorded here: **P3 ≤ 2.0 ms P99 is the target; `STEP_BUDGET_MS` = 4 ms is an alarm threshold, not a target.** A P3 above 4 ms logs the slowest hooks; the tick completes; nothing aborts a hook mid-run.

### 9. The kernel state machine is a data table

States: `BOOTING`, `LOADING`, `RUNNING`, `PAUSED`, `SAVING`, `FAULTED`, `SHUTTING_DOWN`. The GDD's transition table is encoded as one constant table `(state, trigger) → target`, where a target may be the sentinel **`PRIOR`** (`SAVING` returns to the state it came from).

- **Triggers** (closed): `request_pause`, `request_resume`, `request_snapshot`, `request_restore`, `request_new_game`, `request_shutdown`, `fault`, `capture_done`, `capture_error`, `load_done`, `load_failed`, `boot_validation_failed`, `teardown_done`.
- **Boot validation is not a trigger.** It is an internal step of the boundary that serves the first `request_new_game` / `request_restore` in `BOOTING`: on success the request proceeds (`BOOTING → LOADING` via the request's row); on failure the same boundary fires `boot_validation_failed` (`BOOTING → FAULTED`).
- **Deferral comes before the lookup.** A request made while an atomic step (`SAVING`, `LOADING`) is active is `DEFERRED` and evaluated, in arrival order, at the first boundary after the step, against the state that boundary finds. Only then is the table consulted; a pair not in the table is `IGNORED` with one invalid-transition diagnostics entry and no state change.
- **Seat binding is outside the table.** `request_bind_seat` / `request_unbind_seat` (ADR-0002) are non-transition requests: served at the boundary in `BOOTING`, `RUNNING` and `PAUSED` (`SERVED`), deferred during an atomic step, and `IGNORED` in `FAULTED` and `SHUTTING_DOWN`. They never change state.
- **Re-entry from `FAULTED` (closes OQ-5).** The kernel keeps `registry_valid: bool`, set true only by a passing validation pass. The `FAULTED` rows for `request_restore` and `request_new_game` apply only when `registry_valid`; a boot-failure `FAULTED` exits only through `request_shutdown`. A runtime fault keeps all three exits.

The control-plane test KRN-STATE-01 is **generated from this table**: every row, every invalid pair, both kinds of `FAULTED`, and the non-transition requests, so a row added without a fixture row fails.

### 10. Kernel members (the H1 allow-list)

Everything this ADR adds to the kernel lives under the GDD's allow-listed groups, so the H1 lint stays a fixed list: **registries** (descriptors, the compiled schedule, the fault-domain table, `registry_valid`, the transition table); **kernel_state** (current state, the `PRIOR` return state, the reused contexts, tokens and scope objects); **diagnostics** (violation counters, boot warnings, the fault log).

*(Amended 2026-10-28, TD-07.)* The H1 **registries** group also holds, by name: `touch_serial` (a `PackedInt64Array` of `MAX_BLOCKS`, written by every granted `open_for`; ADR-0004 § 7), the **hash-cache arrays** (`{hash, enc_len, serial}` as three parallel `PackedInt64Array`s sized once at the freeze; ADR-0005 §§ 5.1, 5.2), the per-block key-space table (ADR-0004 § 3) and the subscription tables (ADR-0004 § 2). **kernel_state** also holds the kernel-internal hash `SnapshotContext` of § 3 and the `kernel.clock` below. The H1 lint's fixed list is extended by exactly these names.

**The `kernel.clock` view.** `kernel.clock` is an H1 member owned by this ADR. It carries `(T, world_seed, restore_epoch)` and is exposed to modules and the composition root only as a read-only view (getters `tick()`, `world_seed()`, `restore_epoch()`; no setters). `T` advances only in the tick driver's boundary. **`world_seed`** is set before `INIT` (by `request_new_game`, or from the payload by `request_restore`) and does not change until the next new game or restore. **`restore_epoch`** is a `u32`, **read-only to owners, non-persisted and non-hashed**: it is in no state block, no canonical byte string, no `state_hash` and no snapshot header (ADR-0005 § 2.4). `request_restore` and `request_new_game` set it **before** the first `restore()` or `init_world`; Save/Load supplies the value and persists it in `slot_epoch.bin` (ADR-0006), but the kernel's copy is the only one the simulation can see. It is read only inside `RngFacet.draw_epoch` (ADR-0005 § 2.3); a lint forbids any other read in `src/sim/`.

### Architecture Diagram

```
BOOTING (ModuleFacet.registrar, one-way; nulled at the freeze)
  owner.setup(facet) ──► registrar.register_block(BlockDescriptor{... callables})
                         registrar.register_command(CommandTypeDescriptor)
                         registrar.register_hook(HookDescriptor)
                         registrar.register_key_space(KeySpaceDescriptor)       (ADR-0004 § 3)
                         registrar.register_fault_domain(XFaults.DOMAIN_ID, owner, XFaults.NAMES)
                         registrar.register_diagnostics_channel(DiagnosticsChannelDescriptor)
  (host service, e.g. Save/Load: HostRegistrarFacet.register_fault_domain only)
  first request_new_game / request_restore boundary:
     validate · check Callables · Kahn sort · stale-edge warnings · compile schedule[C] · read_view + empty_view once
     → registry_valid = true → LOADING     (failure: boot_validation_failed → FAULTED, registry_valid = false)

LOADING (new game): one INIT scope → each owner's init_world(seed, scope, ctx) in restore_after order → post_restore
                    → binding rebuild → one ALL batch per block → RUNNING

Per tick (main thread; contexts and scopes reused, re-armed per call)
  P2: ValidateContext → owner.validate(cmd, sim_views, vctx) → token? outcome?
      APPLY scope + ApplyContext → owner.apply(cmd, scope, actx) → mutators: open_for · write · mark_dirty · status
  P3: for i in schedule[T mod C]: HOOK scope + HookContext → hook(sim_views, scope, hctx) → token? faulted?
  any fault: scope closes (fail-closed) → kernel FAULTED: Faulted(code) · Applied(uncommitted) · Dropped(FAULTED)
```

### Key Interfaces

```gdscript
class_name ModuleFacet extends RefCounted            # what a simulation module gets in setup() (ADR-0001)
var registrar: BlockRegistrar                         # BOOTING only; set to null at the freeze
var views: SimViews                                   # kept after the freeze
var fault_domain: int                                 # the module's static DOMAIN_ID
var rng: RngFacet                                     # null when the module declares no RNG streams (ADR-0005 § 2.3)
var clock: ClockView                                  # read-only (T, world_seed, restore_epoch); see § 10
# Other owners' published mutator objects are wired by the composition root into the module's
# own setup() arguments (for example Assembly receives Graph, Layout, Body and Interaction).

class_name BlockRegistrar extends RefCounted          # forwards to the kernel; BOOTING only
func register_block(desc: BlockDescriptor) -> void
func register_command(desc: CommandTypeDescriptor) -> void
func register_hook(desc: HookDescriptor) -> void
func register_fault_domain(domain_id: int, owner_name: StringName, names: Array[StringName]) -> void
func register_diagnostics_channel(desc: DiagnosticsChannelDescriptor) -> void
func register_key_space(desc: KeySpaceDescriptor) -> void   # BOOTING only; descriptor and checks are ADR-0004 § 3

class_name HostRegistrarFacet extends RefCounted      # issued to a host service that owns no block (Save/Load, ADR-0006)
func register_fault_domain(domain_id: int, owner_name: StringName, names: Array[StringName]) -> void   # BOOTING only; nothing else

class_name ClockView extends RefCounted               # read-only view of kernel.clock (§ 10)
func tick() -> int
func world_seed() -> int
func restore_epoch() -> int                           # u32; read only inside RngFacet.draw_epoch

class_name PresentationViews extends RefCounted       # § 7; src/kernel/facets/presentation_views.gd
var perception_access: bool                           # set once by the composition root at issue
func get(block_id: StringName) -> Object              # PERCEPTION_OUT: empty_view + PERCEPTION_READ_BLOCKED unless perception_access

class_name CompletionToken extends RefCounted
var epoch: int

class_name MutatorStatus extends RefCounted           # constants only
const OK: int = 0
const REFUSED: int = 1                                 # scope not open: violation, never escalated

# Owner callable shapes (Callables; the last statement of each is `return ctx.complete()`):
#   validate(cmd: Command, views: SimViews, ctx: ValidateContext) -> CompletionToken
#   apply(cmd: Command, scope: WriteScope, ctx: ApplyContext) -> CompletionToken
#   hook(views: SimViews, scope: WriteScope, ctx: HookContext) -> CompletionToken
#   init_world(world_seed: int, scope: WriteScope, ctx: SnapshotContext) -> CompletionToken
#   snapshot(ctx) / restore(data: Dictionary, stored_version: int, ctx) / post_restore(ctx) /
#   default_state(ctx) / read_view(ctx) / empty_view(ctx) / commit_baseline(ctx) /
#   snap_baseline(key: int, ctx) / binding_source(ctx)                            -> CompletionToken
#   result and control-outcome handlers (ADR-0002): (record, ctx: NotifyContext) -> CompletionToken
#   subscriber handlers (ADR-0004 § 2): (batch: ChangeBatch, ctx: NotifyContext) -> CompletionToken
#   NotifyContext is declared in src/kernel/facets/notify_context.gd (ADR-0010 § 1.1)

class_name GraphFaults extends RefCounted              # one per domain, own file; never instantiated
const DOMAIN_ID: int = 3
const NOT_IN_SCOPE: int = (DOMAIN_ID << 16) | 1        # pre-packed; mutators return -NOT_IN_SCOPE on a defect
const NAMES: Array[StringName] = [&"", &"NOT_IN_SCOPE"]
```

Each `class_name` above is its own file; the block shows them together for reading.

### Implementation Guidelines
- **Must** register blocks, commands, hooks, fault domains and diagnostics channels only through `ModuleFacet.registrar` in `BOOTING`; **must never** register after the freeze.
- **Must** declare every kernel-invoked owner callable `-> CompletionToken` with `return ctx.complete()` as its **last** statement and nothing after `complete()` in that expression; **must never** declare one `-> void`, return a domain value, or use `assert()` or an engine error for control flow.
- **Must** report outcomes through `ctx` (`ok`/`reject`/`put`/`fault`); **must never** signal success or failure through the return value.
- **Must never** override a `CallContext` method in a subclass; subclasses only add methods.
- **Must** begin every mutator with `scope.open_for(<its block>)` and mark dirty keys with `scope.mark_dirty` inside the mutator; **must never** rely on a property setter for dirty tracking, and **must never** write a block outside a scope.
- **Must** return a status from every Class A mutator (`OK`, `REFUSED`, or `-code`) and deliver data through a caller-owned `out` object; **must** stop and `return ctx.complete()` at the first non-`OK` status, with no second `ctx.fault`.
- **Must** make `restore` copy into the existing block object; **must never** replace the block object a facade wraps.
- **Must** keep block objects `RefCounted` and plain-data; **must never** make a block a `Resource` or `Node`, or hand out a live container from a view.
- **Must** declare fault codes pre-packed in the owner's `*_faults.gd` with its static `DOMAIN_ID`; **must never** reuse or renumber a code.
- **Must** declare `tolerates_stale` on every `after` edge between hooks that do not always coincide.
- **Must** keep every hook allocation-free in steady state; **must** call kernel callables with `Callable.call()` and a fixed argument list, **never** `callv` on the hot path.
- **Must** keep owners alive explicitly: a registered `Callable` does **not** own its object (Verification 3), so the composition root holds every owner in one strong-reference list from `BOOTING` to `SHUTTING_DOWN`; boot checks every registered callable `is_valid()` after the freeze.
- **Must** drop references at teardown: in `SHUTTING_DOWN` the kernel clears every descriptor and `Callable`, owners drop their facets and the composition root clears its owner list last; a teardown test checks `Performance.OBJECT_COUNT` returns to its pre-boot value. The test starts from "callables do not own": an owner freed early shows up as an invalid callable at boot, not as a leak.
- **Must never** use `await` in a kernel-invoked owner callable or anything it calls (a lint over `src/sim/` and `src/kernel/`): a suspended call returns a coroutine state (caught as `INCOMPLETE`) but could later resume against a re-armed context and scope and write through them. *(Amended 2026-10-28 — architecture review.)*
- **Must never** let an owner callable subscribe to notifications (CR-10).
- **Must** put only duplicated packed arrays in a `snapshot` payload; **must never** return a live reference to a block's own array *(2026-10-28, TD-07)*.
- **Must** end every branch of an override of a typed-return owner callable in `return ctx.complete()` *(2026-10-28, TD-07)*.
- **Must** register a `KeySpaceDescriptor` with every block (ADR-0004 § 3) and emit P5 events only through `scope.emit_event`; **must never** read `restore_epoch` outside `RngFacet.draw_epoch` *(2026-10-28, TD-07)*.
- **Must** give a host service that owns no block only the `HostRegistrarFacet`; **must never** hand it a full `BlockRegistrar` *(2026-10-28, TD-07)*.

## Alternatives Considered

### Alternative 1: An abstract `BlockOwner` base class with `@abstract` methods
- **Description**: Each owner subclasses `BlockOwner` and overrides `validate`, `apply`, `snapshot`, `restore` and the rest; `@abstract` (documented since 4.5) forces implementation.
- **Pros**: The contract is a visible type; editor completion lists the methods.
- **Cons**: `@abstract` is a post-cutoff feature this project has not exercised. Overrides inherit the base's typed return in 4.7 and need an explicit `return` on every path, which is the trap the token check guards against. Hooks, handlers and binding sources are many callables per owner and do not fit one class's method set. Inheritance couples every owner to the base.
- **Rejection Reason**: Registered `Callable`s give the same contract with no 4.5/4.7 language dependency and let tests register stubs.

### Alternative 2: Blocks as `Resource` subclasses
- **Description**: Each block is a `Resource`, saved with `ResourceSaver` and editable in the Inspector.
- **Pros**: Free serialisation and editor tooling.
- **Cons**: A `.tres` can embed scripts (a save-file trust problem); serialisation is not canonical; the interchange is plain data by ruling (SL-2).
- **Rejection Reason**: Determinism and trust.

### Alternative 3: A fresh `CompletionToken` (and context and scope) per call
- **Description**: Exactly the GDD's wording: new objects before each call.
- **Pros**: Matches the GDD literally.
- **Cons**: Several allocations per hook per tick and per command, against a zero-allocation quiet tick.
- **Rejection Reason**: Reuse with an epoch and a re-arm list gives the same guarantee without allocating.

### Alternative 4: Exceptions or `assert` for faults
- **Description**: Owners `assert(...)` invariants and rely on the engine's error output.
- **Pros**: Simplest to write.
- **Cons**: No `try`/`catch`; an aborted function returns a value the caller cannot distinguish; `assert` is stripped in release.
- **Rejection Reason**: Debug and release would behave differently (CR-19).

### Alternative 5: Domain ids assigned in registration order
- **Description**: `register_fault_domain` hands out the next id at boot.
- **Pros**: No central table.
- **Cons**: Inserting a system renumbers every later domain, breaking log comparison and fixtures that assert `Faulted(code)`; constants cannot carry a runtime-assigned domain.
- **Rejection Reason**: Codes must be stable across builds; a fixed table is cheap.

### Alternative 6: Scan all hooks each tick
- **Description**: Each tick loops the sorted hook list and tests `T mod divisor == offset`.
- **Pros**: No schedule to build.
- **Cons**: A modulo and branch per hook per tick; the executed set is not a printable artifact.
- **Rejection Reason**: The schedule is compiled once (`C` = 60), removes per-tick work, and is what the Debug Overlay shows.

### Alternative 7: Node-based owners
- **Description**: Each owner is a `Node` and registers by being added to the tree.
- **Pros**: The usual Godot idiom.
- **Cons**: Violates H10; registration would depend on tree order rather than an explicit declaration (CR-6).
- **Rejection Reason**: Registration is an explicit declaration by the composition root.

## Consequences

### Positive
- One contract for fifteen owners, testable with stub objects and no scene.
- A forgotten `return`, an aborted function and a missing outcome are detected in release.
- "Write only in a scope" and "no dirty key without a scope" hold by construction, including during new-game construction; the first fault freezes the evidence.
- Fault codes are stable across builds and print as names.
- Hook order is a printable table; per-tick selection is a lookup.
- The state machine and its test come from one table.
- Quiet ticks allocate nothing in the kernel.

### Negative
- Every owner method is `-> CompletionToken` with a context parameter; mutators that produce data need an `out` object.
- Dirty marking is manual; a missed mark is silent without the ADR-0004 check.
- `Callable.call()` overhead is paid per hook per due tick; spike S1 must show it fits.
- Long-lived facades require `restore` to copy in place, a rule every owner must follow.
- The epoch token, long-lived views and the `FAULTED` qualifier need kernel GDD amendments.

## Risks
- **Callable overhead exceeds the P3 budget.** Mitigation: spike S1; the schedule minimises calls; GDExtension for the Graph is the documented escape (OQ-10).
- **An owner forgets `scope.mark_dirty`.** Mitigation: the ADR-0004 debug check plus a per-mutator unit test.
- **A stale token or a re-armed field leaks between calls.** Mitigation: identity and epoch check; the full re-arm list; a test returns a previous call's token and expects `INCOMPLETE`, and a two-command batch test checks no spurious `CTX_DOUBLE_OUTCOME`; proved on a release export (KRN-CTX-06).
- **A context is used re-entrantly.** Mitigation: the re-entrancy flag and `CONTEXT_REENTRY`.
- **An owner replaces its block object in `restore` and its facade goes stale.** Mitigation: the in-place rule and the restore-then-read facade test per block.
- **A reference cycle leaks at shutdown or between tests.** Mitigation: the `SHUTTING_DOWN` clear and the `OBJECT_COUNT` teardown test.
- **An owner is freed while its callables are still registered** (callables hold only an id). Mitigation: the composition root's strong-reference list and the boot `is_valid()` check.
- **`registry_valid` is set wrongly.** Mitigation: set in one place; KRN-STATE-01 covers both kinds of `FAULTED`.
- **A facet leak gives presentation `SimViews` or a scope.** Mitigation: facets are issued by the composition root; a lint forbids `SimViews` and `WriteScope` in `src/presentation/`.
- **`C` exceeds the cap when a new divisor is added.** Mitigation: boot fails naming the divisor.

## GDD Requirements Addressed

| GDD System | Requirement | How This ADR Addresses It |
|------------|-------------|--------------------------|
| command-simulation-layer.md | CR-3: hooks topologically sorted; cycle or unknown id fails boot; stale-edge warning (TR-cmd-007) | § 8 |
| command-simulation-layer.md | CR-6: registration only in `BOOTING`, frozen after; no silent defaults (TR-cmd-011) | §§ 2, 8; Guidelines |
| command-simulation-layer.md | CR-8: single-owner blocks with the full descriptor; PlayerId-keyed persistent state (TR-cmd-014) | § 2 |
| command-simulation-layer.md | CR-8: ids block-local; cross-block references keyed `(block_id, entity_id)` (TR-cmd-015) | § 2, with ADR-0002 for the binding table |
| command-simulation-layer.md | CR-9: writes only by owner mutators inside a `WriteScope`; refused elsewhere; validate-then-apply, fault on defect (TR-cmd-021) | §§ 4, 5 |
| command-simulation-layer.md | CR-18: `init_world` steps run under one `WriteScope` before `post_restore` | § 4 (`INIT` phase) |
| command-simulation-layer.md | CR-19(a)(b): ctx, `return ctx.complete()`, completion sentinel (TR-cmd-022) | § 3 |
| command-simulation-layer.md | CR-12: read views are getter-only facades; `SIM_WRITE_TRIPWIRE` (TR-cmd-026) | § 7 |
| command-simulation-layer.md | States and Transitions: seven states (TR-cmd-029) | § 9 |
| emitter, player-controller-input, interaction, suit, ship-section-and-layout, ship-body, web-map, attention, horror, sanity GDDs | Each registers its block with `persist`/`persist_scope`/`scope`/`repl_hint`/`restore_after` and its hooks with divisor, offset and `after` (TR-emitter-001, TR-pc-002, TR-pc-005, TR-interact-014, TR-suit-001, TR-section-001, TR-shipbody-005, TR-shipbody-011, TR-webmap-002, TR-attention-001, TR-horror-001, TR-sanity-001) | §§ 2, 8 |
| player-controller-input.md, modular-ship-assembly.md | Every kernel-invoked callable receives `ctx` and ends in `return ctx.complete()` (TR-pc-019, TR-assembly-008) | § 3 |
| modular-ship-assembly.md | `assembly.state` SIM block with a four-block `restore_after` (TR-assembly-011) | § 2 |
| chart-table-web-map-and-menus.md | Settings and menu state registers as a non-`gameplay` block (TR-charttable-023) | § 2 (`gameplay`) |

## Performance Implications
- **CPU**: One `Callable.call()` per due hook and per dispatched command, selected by one schedule lookup. The MVP's 15 hooks at divisors 1, 5, 6 and 12 average well under 15 due per tick. Re-arming a context or scope is a handful of field writes. P3 ≤ 2.0 ms P99 with a 4 ms alarm, measured on a release export on RM-DESKTOP and RM-DECK (spike S1).
- **Memory**: The schedule is `C` (60) small `PackedInt32Array`s built once; one context per kind, one token per context and the scope objects are allocated at boot. No per-tick kernel allocation in a quiet tick.
- **Load Time**: Registration, validation and the hook sort run once; facades are built once at the freeze.
- **Network**: None. `repl_hint` is stored now and consumed in VS-B (ADR-0032).

## Migration Plan
No code exists. Document changes: amend the kernel GDD (§ GDD sync); amend ADR-0002 so its result and control-outcome handlers take `(record, ctx: NotifyContext) -> CompletionToken` and `register_command` is listed on the `BlockRegistrar` (done in the same pass as this ADR); amend `docs/architecture/architecture.md` § API Boundaries so `weld_join` returns a status and fills a caller-owned `WeldResult`, Class B mutators return `void`, and the `ReadViews`/`WriteScope` placeholders point here; change Derelict Authoring and Web Map wording that treats `STEP_BUDGET_MS` as a target (IC-4).

## Validation Criteria
- On a **release export**: a callable that aborts yields `fault(INCOMPLETE)`; one returning a previous call's token yields `INCOMPLETE`; one returning the armed token with `completed` unset yields `INCOMPLETE`; one that hits `await` yields `INCOMPLETE` and never resumes into a later call's scope; the outcome for a callee that logs a script error and continues is recorded; a two-command batch on the reused context gives no spurious `CTX_DOUBLE_OUTCOME`.
- A mutator on a closed scope writes nothing, marks nothing, returns `REFUSED` and counts `WRITE_OUTSIDE_SCOPE`; after a `scope.fault` later mutators in the same apply return `REFUSED` and nothing is counted twice; in `PAUSED_APPLY` `open_for` is false for blocks not in `writes_blocks`.
- A new game builds the MVP world through one `INIT` scope (Web Map `init_world` → Layout, Emitter, Interaction mutators) and delivers exactly one `ALL` batch per block on `LOADING → RUNNING`.
- A `validate` with no outcome is `Faulted(NO_OUTCOME)`; a second outcome call is `CTX_DOUBLE_OUTCOME`; a re-entrant call is `CONTEXT_REENTRY`.
- Registration after the freeze, a duplicate id, an unknown dependency, a cycle, a missing descriptor field, an invalid `Callable`, `divisor < 1`, an offset outside `0..divisor-1` and `C > 3,600` each fail boot naming the registration, in debug and release, with the kernel fault code of § 6.
- The compiled schedule for the MVP hook set equals the table in `docs/architecture/architecture.md` § Data Flow 1, `C` = 60, and a 600-tick run calls each hook exactly when `T mod divisor == offset`, in total order.
- KRN-STATE-01, generated from the table, reaches every tabled target (including `PRIOR`), ignores every invalid pair with one diagnostics entry, covers boot-failure `FAULTED` (shutdown only) and runtime `FAULTED` (three exits), and treats bind/unbind as non-transition requests.
- `SimViews.get` of a `PERCEPTION_OUT` block returns its typed empty view and counts `PERCEPTION_READ_BLOCKED`; the default face of `PresentationViews` does the same, the perception face returns the real facade, and the flag cannot be changed after issue; restoring a block and reading its facade shows the restored values.
- `emit_event` and `has_opened` obey the scope rules (discard in `INIT`/`RESTORE`/`CLOSED`; `has_opened` true from the first granted `open_for` until re-arm); the five new violation names are counted and call `violation_handler` under `STRICT_DEBUG`; `register_key_space` after the freeze fails boot; the clock view exposes `(T, world_seed, restore_epoch)` with no setter and `restore_epoch` is set before the first `restore()`/`init_world` and absent from every hash; a `block_hash` call from inside `snapshot` is `CONTEXT_REENTRY`; a `snapshot` payload is unchanged by a later mutation.
- Each `*_faults.gd` constant matches its `NAMES` entry and domain id; two builds with an extra system inserted produce identical codes for every existing fault.
- A quiet tick performs zero kernel container allocations (`Performance.OBJECT_COUNT` flat over 600 ticks plus a static scan of hot files, OQ-16); after teardown `OBJECT_COUNT` returns to its pre-boot value.

## GDD sync
**Status 2026-10-28: all six items below are applied** to `design/gdd/command-simulation-layer.md` (CR-8, CR-18, CR-19, the States and Transitions rows, Tuning Knobs), `derelict-authoring-handmade-kits.md` and `web-map-and-transit.md`. The new `MAX_SCHEDULE_CYCLE_TICKS` knob is not yet in `docs/registry/architecture.yaml`. Acceptance still waits on the review noted in the Ordering Note. What was needed:
- `design/gdd/command-simulation-layer.md` CR-19(b): "a fresh `ctx.token`" → "a token armed per call (identity and epoch checked; contexts reused and re-armed)".
- CR-8 and CR-19(a): `read_view` is called once at the registry freeze and returns a long-lived facade; owners also supply `empty_view`; `restore` copies into the existing block object. The CR-19(a) clause about a `read_view` fault inside P2/P3 no longer applies.
- CR-18: `init_world(world_seed, scope, ctx)` receives the shared `INIT` scope; dirty keys from `INIT`/`RESTORE` scopes are replaced by one `ALL` batch per block on `LOADING → RUNNING`.
- States and Transitions, the `RUNNING / PAUSED / FAULTED request_new_game` row and the `FAULTED request_restore / request_new_game` row: "legal only when the registries validated (`registry_valid`); a boot-failure `FAULTED` exits only to `SHUTTING_DOWN`" (closes OQ-5). Add bind/unbind as non-transition requests.
- Add the static fault-domain table, the kernel fault codes, the violation set and `MAX_SCHEDULE_CYCLE_TICKS` (Tuning Knobs).
- *(Added 2026-10-28, TD-07)* `design/gdd/command-simulation-layer.md`: `kernel.clock` carries `(T, world_seed, restore_epoch)` with the `restore_epoch` rules of § 10; the violation set and the registries gain the names of § 6 and § 10; the `PERCEPTION_OUT` row of CR-10 matches the two-face rule of § 7.
- `design/gdd/derelict-authoring-handmade-kits.md` (TR-derelict-021), `design/gdd/web-map-and-transit.md` (TR-webmap-025): `STEP_BUDGET_MS` is an alarm threshold; the P3 target is 2.0 ms P99 (IC-4).

## Related
- `design/gdd/command-simulation-layer.md` (CR-2, CR-3, CR-6, CR-8, CR-9, CR-12, CR-14, CR-18, CR-19; States and Transitions; KRN-CTX-06, KRN-STATE-01)
- `docs/architecture/architecture.md` (§ Data Flow 1 hook table, § Initialisation order, § API Boundaries, OQ-5, OQ-15, OQ-16)
- ADR-0001, ADR-0002, ADR-0004, ADR-0005, ADR-0006, ADR-0009, ADR-0010 (Proposed); ADR-0016, ADR-0021, ADR-0024, ADR-0031 (to be written)
- Review rulings: delegated ruling R3 (completion sentinel), R-8 (stale-edge warning), G1 (owner wins)

## Amendments

### 2026-10-28 - Foundation-ADR adjudication (status stays Proposed)
Review Log. Earlier outcome: the architecture-review amendments of 2026-10-28 (finding A1, B1-B11, C, E; marked inline) stand unchanged. This pass applies only the cross-ADR rulings the adjudicator assigned to this ADR at the request of ADR-0004, ADR-0005, ADR-0006 and ADR-0009; nothing else in the decision changed. No earlier text said "review unavailable"; the Decision Makers line now points here.

| Ruling / source id | What was done |
|---|---|
| **TD-07** [R14(ADR-0003), ADR-0004/R14, ADR-0005/R14, ADR-0005/X4, ADR-0006/R14, ADR-0009/R14] | One batch, items 1-13 below. `kernel.clock` is owned here; ADR-0005's attribution of the clock view to ADR-0001 is corrected in ADR-0005. |
| TD-07 (1) | `BlockRegistrar.register_key_space(desc: KeySpaceDescriptor) -> void`, BOOTING only (Key Interfaces, diagram; ADR-0004 § 3). |
| TD-07 (2) | `WriteScope.emit_event(block_id, kind, a0..a5 = 0)` (§ 4). |
| TD-07 (3) | `WriteScope.has_opened(block_index) -> bool` (§ 4; ADR-0005 § 5.3). |
| TD-07 (4) | Violation set adds `SUBSCRIBE_REFUSED`, `DIRTY_KEY_INVALID`, `DIRTY_KEY_KIND_UNDECLARED`, `EVENT_OVERFLOW`, `MISSED_DIRTY_MARK`; `UNKNOWN_DEPENDENCY` also covers a subscription to an unknown block (§ 6). |
| TD-07 (5) | H1 registries group adds `touch_serial` (ADR-0004) and the hash-cache arrays (ADR-0005 § 5.2) (§ 10). |
| TD-07 (6) | `kernel.clock` view `(T, world_seed, restore_epoch)`: `world_seed` set before `INIT`; `restore_epoch` a `u32`, read-only, non-persisted, non-hashed, set by `request_restore` / `request_new_game` before the first `restore()` / `init_world`, read only in `RngFacet.draw_epoch` (§ 10). |
| TD-07 (7) | `ModuleFacet.rng: RngFacet`, null when the module declares no streams (Key Interfaces). |
| TD-07 (8) | One kernel-internal `SnapshotContext` for hash reads outside `SAVING`, with a re-entrancy flag (§ 3). |
| TD-07 (9) | The "hash before and after and compare" sentence is now a pointer to ADR-0004 § 7; the `mark_dirty` comment reads "key layout: ADR-0004 § 3" (§ 4). |
| TD-07 (10) / **TD-08** | `PresentationViews` has two faces selected by `perception_access`, set by the composition root at issue; the default face returns `empty_view` for `PERCEPTION_OUT` and counts `PERCEPTION_READ_BLOCKED`; the perception face returns the real facade (§ 7). Subscribe-time refusal (check 6) is ADR-0004's; the flag's holders are ADR-0009 § 1's; ADR-0024 owns only the Filter's query surface. |
| TD-07 (11) | Snapshot payload packed arrays are duplicates, never live references (§ 2); every override of a typed-return owner callable returns on all paths (§ 1). |
| TD-07 (12) | A host service (Save/Load) receives a BOOTING-only `HostRegistrarFacet` limited to `register_fault_domain` (Key Interfaces). |
| TD-07 (13) / **TD-01** | `NotifyContext` is placed in `src/kernel/facets/` (§ 3). The presentation deny list is ADR-0010's explicit list, not a `*Context` glob. ADR-0005 fixes its Depends On attribution: `kernel.clock`, facets and `has_opened` belong to ADR-0003; `touch_serial` to ADR-0004. |

Cross-ADR note for ADR-0010 (not decided here): `NotifyContext extends CallContext`, and `CallContext` stays in kernel core, so `src/kernel/facets/` needs an allowed edge to that one base class (or ADR-0010 moves `CallContext`). ADR-0010 owns the answer; this ADR only requires that a presenter can name `NotifyContext` and `CompletionToken`.

Open questions: this ADR carried none; none added.
