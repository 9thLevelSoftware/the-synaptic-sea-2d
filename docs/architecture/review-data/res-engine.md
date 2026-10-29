# Engine specialist consultation: /architecture-review Phase 5 (Godot 4.7.2)

Scope: ADR-0001..0010 (all Proposed). Reference docs read: VERSION.md, breaking-changes.md (4.6 to 4.7 table). The other reference files and module docs were not read in full; the checks below rely on the ADR text, targeted greps and four local probes.
Probes (Godot_v4.7.2-stable_mono_win64_console, headless, scripts under scratchpad/ar/probe/): P-A `get()` override, P-B `const OK`, P-C `Engine.time_scale` vs `_process` delta, P-D `driver.windows` default.

## Audit confirmation

1. **Engine Compatibility sections and version: CONFIRMED.** All 10 ADRs have the section and name 4.7.2. I saw no stale version text in the sections I read (0001, 0005, 0006, 0008 in full; 0003/0004/0007/0009/0010 rows via grep).
2. **Deprecated-API grep: CONFIRMED, no actual use.**
   - `OS.get_ticks_*` appears only in ban lists (ADR-0001 l.309 is an amendment note; ADR-0010 l.209/744 and ADR-0005 l.165 are lint bans).
   - `duplicate()` / `duplicate(true)` in ADR-0005 l.496/873 and ADR-0006 l.92/262/426/622 are worker-handoff ownership copies of `PackedByteArray`, Arrays and Dictionaries. They are valid on 4.7.2. ADR-0006 l.124 correctly names `duplicate(true)` (probe 15 says `duplicate_deep` also exists).
   - ADR-0010 l.321 HOT-01 lists `.duplicate(` / `.duplicate_deep(` as banned tokens, not as uses.
   - My regex for `yield`, `instance()`, `TileMap`, `YSort`, `get_world(`, string-form `connect("`, `event.device` and `GodotPhysics` found no code hits in the ADRs beyond the ban-list lines above.
   - Neither the audit nor my grep checked `Texture2D` default-texture params or the RichTextLabel percent flags. ADR-0008 uses neither.
3. **Post-cutoff API set and cross-ADR contradictions: CONFIRMED with two reservations.**
   - **@abstract.** ADR-0003 (l.26, Alt 1 l.392-395) avoids `@abstract` for the owner `Callable` contract. ADR-0010 §3 uses it only for ports, and states "ADR-0003 §1 stands. There is no `BlockOwner` port". These are consistent. The ADR-0010 P4 probe is real: parse-time error on direct construction, while reflective `load().new()` succeeds.
   - **Typed vs untyped Dictionary.** ADR-0002 requires an untyped `Dictionary` payload. ADR-0010 SIM-09 bans a bare `Dictionary` in `shared`, `kernel` and `sim`, so the two are reconciled only by an allow-list (`dictionary_variant.json`) whose contents are open (ADR-0010 OQ-H, l.682). See E5.
   - **Thread payload wording.** ADR-0010 IO-02 / l.391 say a `PackedByteArray` is "handed over by value". ADR-0006 probe 16 shows packed arrays are shared by reference, so only "ownership transfer or `.duplicate()`" is correct. See E4.
   - **driver.windows default.** The reference says D3D12 is the Windows default. ADR-0008 OQ-C notes that the probe disagrees, and I confirmed it (P-D, E6).

## Specialist findings

### E1 [HIGH] `PresentationViews.get()` overrides native `Object.get()` and will not load
- **Where:** ADR-0003 l.340 (`func get(block_id: StringName) -> Object` in `class_name PresentationViews extends RefCounted`). Also used at ADR-0009 l.570 ("its `PresentationViews.get` returns `empty_view`"). ADR-0009 l.119 and ADR-0010 OQ-K (l.685) depend on this class.
- **Claim:** the facet exposes `get(block_id)`.
- **Reality:** probed on 4.7.2 (P-A). `func get(block_id: StringName) -> Object` in a `RefCounted` script gives `Parse Error: The method "get()" overrides a method from native class "Object". This won't be called by the engine and may not work as expected. (Warning treated as error.)` The script then fails to load (`Failed to load script ... Parse error`). `NATIVE_METHOD_OVERRIDE` is an error by default, per the output of this run (the probe project set no warning options).
- **Contradiction inside the set:** ADR-0010 l.146 forbids accessors named `get`, `set`, `free`, `call` and the rest ("`NATIVE_METHOD_OVERRIDE`"), yet ADR-0003 and ADR-0009 define one.
- **Confidence:** HIGH, probed.
- **GDD:** no GDD names `PresentationViews.get`. My grep of `design/gdd/*.md` for `PresentationViews|views\.get\(|\.get\(&"` found no hits outside the ADRs. Fix: rename to `get_view` or `lookup`, and update ADR-0003 §7, ADR-0009 l.119/570 and ADR-0010 l.146.

### E2 [MEDIUM] Time-scale risk text is stale: `time_scale` is now probed
- **Where:** ADR-0001 l.27 item (2), l.51, l.238 ("`Engine.time_scale` scales `delta` differently in 4.7.2" is listed as a risk and a spike item).
- **Reality (P-C, headless):** over 1.0 s of real time the sum of `_process` delta was 1.006 at scale 1.0, 0.502 at 0.5, and 2.010 at 2.0. So `_process(delta)` IS scaled by `Engine.time_scale`. The ADR's mitigation (pin 1.0 in release, flag `TIME_SCALE_NON_UNIT`) is correct and sufficient. Item (2) can be marked probed for headless. A release export and the windowed editor remain unchecked.
- **Confidence:** HIGH on headless, probed.
- **GDD:** `design/gdd/debug-overlay-true-values.md:499` says the sanity-drift fold is independent of `Engine.time_scale`. That holds only if the fold is driven by ticks T, not by `delta`, and this is consistent with a tick-driven kernel. Nothing contradicted.

### E3 [LOW] ADR-0006 rename-over-existing on Windows
- **Where:** ADR-0006 l.27 probe (1).
- **Claim:** the author's recollection (65%, UNVERIFIED) is that `DirAccessWindows::rename` is remove-then-rename, not an atomic `MoveFileEx`. The ADR therefore never claims atomicity.
- **Reality:** my own recollection of the engine source agrees (it removes an existing target first), MED confidence, not probed. The ADR's conservative design (generations plus verify) already covers this. No change needed. Keep it on the UNVERIFIED list until a kill-between-remove-and-rename test exists.

### E4 [MEDIUM] "by value" wording contradicts packed-array reference semantics
- **Where:** ADR-0010 IO-02 (l.309), l.391 and l.878 ("handed over by value and never mutated afterwards") versus ADR-0006 §1.6 and probes 16 and 7 (packed arrays alias by reference; ownership transfer or `.duplicate()` is required).
- **Reality:** the engine does not copy on pass to `Thread.start(callable.bind(...))` or on assignment. "By value" misleads a lint author or implementer. The rule should read "ownership transferred (no retained alias) or `.duplicate()`d".
- **Confidence:** HIGH (ADR-0006 probe 16 and ADR-0005 P7). Not re-probed by me.
- **GDD:** none found. I did not grep GDDs for this.

### E5 [MEDIUM] SIM-09 versus the ADR-0002 untyped payload boundary: reconciled only by an empty allow-list
- **Where:** ADR-0010 l.292 (SIM-09) and OQ-H l.682, versus ADR-0002 l.26.
- **Claim:** untyped `Dictionary` is required at the command payload boundary. SIM-09 bans a bare `Dictionary` in the sim layers except allow-listed files.
- **Reality:** no engine conflict. The risk is process: the allow-list `dictionary_variant.json` is not yet written (OQ-H), so a lint pass would fail every payload validator. Also, per ADR-0010 l.230, `Dictionary[K,V].keys()` and `.values()` return untyped `Array` (4.4), which interacts with the same SIM-09 bare-`Array` ban.
- **Confidence:** HIGH on reading, not probed.

### E6 [LOW] Engine reference is wrong about the Windows default renderer driver
- **Where:** VERSION.md timeline ("D3D12 default on Windows", 4.6 row) and the breaking-changes 4.5 to 4.6 section, versus ADR-0008 l.31 and l.43 (OQ-C).
- **Reality (P-D):** `ProjectSettings.get_setting("rendering/rendering_device/driver.windows")` returned `vulkan` on 4.7.2. ADR-0008 also saw `driver = vulkan` at runtime. The reference text is likely misleading: D3D12 may be the editor or template default only in some distributions. Treat as a doc defect in `docs/engine-reference`, not an ADR defect. ADR-0008 does the right thing by pinning and asserting the effective value.
- **Confidence:** HIGH that the built-in default setting is `vulkan`. MED that nothing else (for example a first-run editor project template) sets D3D12.

### E7 [LOW] `NATIVE_METHOD_OVERRIDE` hazard on other class names: no further clashes found
- I grepped ADR code blocks for `var|func|const|signal` named `name`, `owner`, `set`, `get`, `call`, `free`, `hash`, `tr`, `position`, `scale`, `state`, `id` and more. Only E1 is a clash.
- `var name: StringName` at ADR-0005 l.208/401 and ADR-0009 l.229 sits in `RefCounted` classes (`RngStream`, `PresenterDescriptor`), where `name` is not a native property. It would clash if any of them were changed to extend `Node`. They are all `RefCounted`-based as written.
- `const OK: int = 0` in `MutatorStatus` (ADR-0003 l.346) shadows the global `OK` (Error enum). Probed with `shadowed_global_identifier=2`: it loaded and returned 0 (P-B). The 2 here is my probe's setting; the ADR-0010 warning set was not run. Confidence on "no error in the real gate" is MED, because the real gate uses the full ADR-0010 §5.1 configuration. Worth a LINT-SPIKE-1 fixture line.

### E8 [LOW] ADR-0010 bans `Callable(` and `.bind(` in `## @hot` functions, while ADR-0006 hands work to a Thread with `run.bind(...)`
- ADR-0006 l.92: `WriteJob.run() via WriteWorker.run.bind(...)`. This is on the save path, not the sim hot path, so HOT-01 does not apply. Just confirm the `## @hot` scope never covers `SaveService`. Not an engine issue.

## Open probes (UNVERIFIED)

Not re-reported as new findings. These remain open, or only partly closed, from the ADR text:
- **ADR-0001:** KRN-LAT-02 `_process` order adapter, driver and presentation on a release export; `_input` / `_unhandled_input` delivered before `_process` (item 3b); `physics_jitter_fix` / `delta_smoothing` effect (3a); first-frame `delta` after a long block (3). Item (2) `time_scale` is closed for headless only (E2).
- **ADR-0001/0008/0009:** `RenderingServer.frame_pre_draw` ordering on a release export, under D3D12, on the Steam Deck, with a minimised window and under `low_processor_mode` (pipeline spike step 12). Verified only on the windowed 4.7.2 editor, Forward+/Vulkan.
- **ADR-0003:** `Callable.get_argument_count` verified present only.
- **ADR-0005:** S0-a to S0-d on `template_release` and on Linux, arm64 and Deck; `PackedByteArray.resize` reallocation behaviour; `slice` need before `HashingContext.update`; the cross-OS vector job.
- **ADR-0006:** `DirAccessWindows::rename` semantics (E3); `Thread.start` error codes; invalid-UTF-8 handling; a `Thread` freed while alive; a worker blocked in a syscall at exit; kill between remove and rename.
- **ADR-0008:** pipeline spike steps 11 and 12; HUD sharpness at H=3; the display-scaling cases at 125% and 150%; Compatibility parity; `use_hdr_2d` is Forward+/Mobile only (probed but not on a release export).
- **ADR-0009:** `speed_scale` freezing particles and animation; driver overhead on RM-DESKTOP and RM-DECK.
- **ADR-0010:** LINT-SPIKE-1(a) `@abstract` surviving a release export and `Script.is_abstract()`; `mock()` over an `@abstract` class; (c) Performance monitors on a release export; `directory_rules` behaviour in an export or a later patch; whether a `template_release` export is a warning gate.
- **ADR-0007:** CONTENT-EXP-1 passed (not re-checked).

## GDD assumptions contradicted

- E1: none. The `PresentationViews.get` name appears only in ADR-0003, ADR-0009 and ADR-0010.
- E2: `design/gdd/debug-overlay-true-values.md:499` mentions `Engine.time_scale`; it is compatible with the probed behaviour (time_scale does scale `_process` delta).
- No GDD found that is designed around a wrong engine assumption in this pass. I grepped GDDs only for `time_scale` and the `PresentationViews` names, not exhaustively for every API in the ADRs. The `frame_pre_draw` / `PR-8 item 3` text in the Pipeline GDD is a known sync item (ADR-0008 l.251, returned to GDD sync), not a new finding.
