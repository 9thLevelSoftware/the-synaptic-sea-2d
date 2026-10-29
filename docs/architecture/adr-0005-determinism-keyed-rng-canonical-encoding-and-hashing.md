# ADR-0005: Determinism, Keyed RNG, Canonical Encoding and Hashing

## Status
Proposed

## Date
2026-10-28

## Last Verified
2026-10-28 (against `design/gdd/command-simulation-layer.md` CR-8, D.10, D.11, D.12 and Acceptance Criteria "Determinism" / KRN-HASH-01..03 / KRN-EPO-01 / KRN-PERF-01..05; `life-support-graph.md` LS-9, LS-21; `save-load-and-versioned-format.md` SL-16, SL-23, F-10; `attention-system.md` AT-8, AT-28; `horror-entities.md` HE-42; `sanity-and-hallucination-rules.md` SN-24; `ship-section-and-layout-data-model.md` LY-33, LY-26 (e); `2d-lighting-and-occlusion.md` LO-19; `emitter-state-and-light-exposure-query.md` AC-2; `suit-and-equipment.md` conventions; `web-map-and-transit.md` WM-37; `docs/architecture/requirements-traceability.md` rows citing ADR-0005 (47 TR-IDs); ADR-0001, ADR-0002, ADR-0003 and ADR-0007 (all Proposed); and `docs/engine-reference/godot/` `VERSION.md`, `breaking-changes.md`, `deprecated-apis.md`, `current-best-practices.md`). No game code exists. Engine behaviour was probed on 2026-10-28 with the installed `Godot_v4.7.2-stable_mono_win64_console.exe` (a debug editor binary on the developer machine, not a release export), from a scratch directory outside the repository; those results are labelled "probed 2026-10-28". The 2026-10-28 engine and TD-ADR reviews are recorded in the Review Log; the reviewer's probes were reproduced by the author on the same binary with all 48 `debug/gdscript/warnings/*` settings except `return_value_discarded` set to 2 (the ADR-0010 § 5.1 configuration) and are labelled "re-probed" (P17 to P22 and the amended P2, P6, P12, P14). Every other engine claim is UNVERIFIED and names the spike that settles it. The pinned engine reference has no core or scripting module, so it is silent on integer semantics, `HashingContext`, packed-array aliasing and `PackedByteArray` encoding.

## Decision Makers
Claude (author), running under modes.automation: autonomous (project.yaml); decisions logged for the user's review; the user has not yet reviewed this ADR. Revised 2026-10-28 after the joint engine review (verdict BLOCKING, 13 findings), the TD-ADR review (verdict CONCERNS, 11 findings) and 15 cross-ADR rulings; every blocking item is applied and the outcome of each finding is in the Review Log. The ADR stays Proposed; the user has not yet reviewed it.

## Summary
Twenty-six GDDs promise that the same save bytes plus the same restore epoch and command stream give the same state hash, and each of them re-specified part of the machinery (integer rules, RNG keys, presentation hashes, byte layouts, cached hashes) with conflicting details. This ADR fixes one determinism contract in six parts. (1) An integer-only simulation with pinned GDScript integer semantics and a division convention, (2) a stateless `rng_derive` / `rng_derive_epoch` family on SplitMix64 with a facet that makes the threat-draw epoch impossible to get wrong, (3) one prefix-free canonical TLV encoder (dictionary keys sorted by value, packed arrays as raw little-endian bytes) that no owner re-implements, (4) `H64` as the first 8 bytes of SHA-256 over those bytes with a fully specified `state_hash` fold, (5) a kernel-owned per-block hash cache validated by ADR-0004's always-on touch serial (so a quantised write that marks no dirty key still invalidates it) and never used on a verification path, and (6) one shared pure-integer presentation hash. All pure determinism code lives in `src/shared/det/`; exactly two files there (the float RNG path and the canonical encoder) hold the only float and `Variant` handling, and every identifier and idiom is written to load under ADR-0010's warnings-as-errors gate.

## Engine Compatibility

| Field | Value |
|-------|-------|
| **Engine** | Godot 4.7.2 (stable), GDScript |
| **Domain** | Core (integer semantics, hashing, byte encoding, deterministic ordering) |
| **Knowledge Risk** | MEDIUM. The APIs used (`HashingContext`, `PackedByteArray.encode_s64` / `decode_s64` / `encode_double` / `hex_encode`, `String.to_utf8_buffer`, `Array.sort`, `PackedInt64Array.to_byte_array`, `@warning_ignore*`) all predate 4.4 and none appears in `breaking-changes.md` for 4.4 to 4.7. The reference has no core or scripting module, so integer wrap, shift and division behaviour, packed-array aliasing and hash throughput are not sourced by it; they were probed on the 4.7.2 editor binary (below) and stay unverified on a release export, on Linux / Steam Deck and on macOS until spikes S0-a to S0-d and the cross-OS vector job pass. The 4.7 rows that matter are sourced: "Packed array element writes no longer call the property setter" (this ADR's cache invalidation therefore never rides a setter) and "Overrides inherit a typed return". That second row does **not** touch `IntMath`, `DetRng`, `Hash64`, `Hex64` or `CanonicalEncoder` (statics on `RefCounted`; nothing overrides them); it binds the code this ADR constrains but does not write, namely owner block classes overriding the ADR-0003 callables (`snapshot`, `restore`, `default_state`, `init_world`) and any `_to_string` override on `BlockHashResult` or `RngStream`, where a missing `return` is the 4.7 break (Implementation Guidelines). The ADR-0010 warning gate (`SHADOWED_*` and `UNSAFE_CAST` at error level) constrains every identifier and the `Variant`-unpacking idiom of § 3.8 (P20), and has two holes the lexical rules close (P18, P19). |
| **References Consulted** | `docs/engine-reference/godot/VERSION.md`; `breaking-changes.md` (4.6 to 4.7 GDScript rows; 4.4 `FileAccess.store_*` row, irrelevant here); `deprecated-apis.md` (untyped Array/Dictionary pattern row; `duplicate()` for nested data replaced by `duplicate_deep()` since 4.5); `current-best-practices.md` (packed-array element-write rule, lines 48 to 52). No module file exists for this domain; none of `modules/*.md` applies. |
| **Post-Cutoff APIs Used** | `Dictionary.duplicate_deep()` / `Array.duplicate_deep()` (4.5, already required by CR-8 for `snapshot` payloads). `@warning_ignore_start` / `@warning_ignore_restore` are 4.3 features, before the model cutoff, and are listed only because they were probed to compile under 4.7.2 with `integer_division` at error level. No other post-cutoff API is introduced. |
| **Verification Required** | **S0-a (packed-array aliasing and element-write semantics), S0-b (integer semantics), S0-c (literal and shift edge cases) and S0-d (hash throughput)** are the Graph GDD's spikes; the dev-machine results are recorded below and the spikes remain open. **S0-b must run on the exact export templates that will ship** (standard or mono, whichever the Steam build uses), as a `template_release` export, on Windows, Linux x86-64 and Linux arm64 / Steam Deck. The engine implements `int` multiplication in C++, where signed overflow and shifts by 64 or more are formally undefined, so P1, P3 and P4 are recorded as **defined only for runtime shift counts 0..63 and non-negative-literal folds**; no shift count outside 1..63 and no negative-literal shift exists in `src/`. P2 and P14 are re-run on the release template. **Acceptance gate** (a gate that needs Graph code would deadlock, because stories citing a Proposed ADR are auto-blocked): S0-a to S0-d as scratch scripts on a `template_release` export (Windows plus Linux / Deck); S0-e with the synthetic stand-ins named below; the § 2.4, § 2.6, § 3.7 and § 4.5 vectors reproducing on Windows and Linux CI runners; and the LINT-SPIKE-1 load-all check of every Key Interfaces snippet under the ADR-0010 § 5.1 warning configuration (Implementation Guidelines). macOS is an advisory leg until a target decision is made (open question 5). The committed golden `state_hash` files for the Graph fixtures (TR-lsg-031) move to the Validation Criteria of the Graph epic. **Spike-list sync:** S0-e and the cross-OS vector job are added to `architecture.md` § Spikes, with the user's approval, before this ADR is Accepted (`architecture.md` carries neither today). **S0-e (new): canonical encoder cost** for a 32-block snapshot with a 2,000-node Graph stand-in, a 250-field Dictionary block and a 12-instance holder block, on RM-DESKTOP and RM-DECK release exports, plus the allocation behaviour of the `PackedByteArray` scratch buffer: record whether `resize(n)` after a larger `resize` reallocates (UNVERIFIED) and whether `slice(0, n)` is needed before each `HashingContext.update`, which takes no length argument. **Ship Body fixture:** the 2-bodies / 3-actors fixture whose length the Body GDD computes as 518 bytes must be encoded by the reference encoder and the length confirmed or corrected (D.12 left it open). **Independent reference:** the vectors below were reproduced by a Python implementation (`hashlib`, explicit masking) in addition to the GDScript probe; the frozen goldens are those the independent implementation produces. |

**Probe record (probed 2026-10-28, `Godot_v4.7.2-stable_mono_win64_console.exe --headless --script`, developer machine, debug editor binary; indicative only, not a release-export measurement). P17 to P22 were added in the 2026-10-28 revision from the engine review's probes, reproduced by the author on the same binary.**

| # | Question | Result |
|---|---|---|
| P1 | `int` width and overflow | Signed 64-bit. `9223372036854775807 + 1` is `-9223372036854775808`; `9223372036854775807 * 2` is `-2`. Wrap is silent. |
| P2 | Hex literal above `0x7FFFFFFFFFFFFFFF` | The engine logs `Cannot represent 0x9E3779B97F4A7C15 as a 64-bit signed integer, since the value is too large` (from `hex_to_int`) while the script compiles, **but the script still loads and runs, and the literal then holds `9223372036854775807` (INT64_MAX), saturated** (re-probed for `0x8000000000000000`, `0xFFFFFFFFFFFFFFFF` and the golden-ratio constant: all read `9223372036854775807`). A load-all gate that checks only that `load()` succeeds therefore does not catch the hex form; the lexical rule of § 1.7 does. The signed decimal constants (`-7046029254386353131` and the others in § 2.1) are mandatory and were used for every vector below. Re-run on the release template (S0-c). |
| P3 | `>>` | Arithmetic at run time: `-8 >> 1` through variables is `-4`; `(-1) >> 63` is `-1`; the same holds through an untyped `Variant` (reviewer probe, `-8 >> 1` is `-4`), so the result does not depend on the typed path. **A constant-folded negative shift is a parse error** (`Invalid operands for bit shifting. Only positive operands are supported`, for the literal expression `-8 >> 1`). |
| P4 | Shift counts | Counts wrap modulo 64: `1 << 63` is `-9223372036854775808` and `1 << 64` is `1` (typed or `Variant`). That is a hardware accident of the C++ shift (formally undefined for counts of 64 or more), not a language rule, so P4 is recorded as **defined only for runtime counts 0..63**. So `lsr(x, 0)` would compute `(x >> 0) & ((1 << 64) - 1)` = `x & 0` = 0 and is invalid; D.10's `1 <= n <= 63` is a real bound. |
| P5 | Division and modulus on negatives | Truncate toward zero; the remainder takes the dividend's sign: `-7 / 2 = -3`, `7 / -2 = -3`, `-7 % 2 = -1`, `7 % -2 = 1`. `posmod(-7, 2) = 1`, `floori(-3.5) = -4`, `roundi(2.5) = 3`, `roundi(-2.5) = -3` (half away from zero). |
| P6 | `integer_division` warning | With `debug/gdscript/warnings/integer_division=2` a bare `a / b` on two `int`s fails to load (`Integer division. Decimal part will be discarded. (Warning treated as error.)`). `@warning_ignore("integer_division")` on the statement and `@warning_ignore_start("integer_division")` / `@warning_ignore_restore("integer_division")` around a function both compile. `debug/gdscript/warnings/directory_rules` exists in 4.7.2 (default `{ "res://addons": 0 }`); its values are settled by ADR-0010 P2 and P3: 0 = excluded, 1 = included, and subtrees inherit. **The engine gate covers the binary `a / b` form only** (P19). |
| P7 | Packed-array aliasing | A `PackedInt32Array` is shared by assignment (`var b := a`), by function argument, by a getter that returns the field, and by a Dictionary value read into a local: a write through any alias is visible through the others. `PackedInt32Array.duplicate()` isolates; `Dictionary.duplicate()` (shallow) does **not** isolate a packed array inside it; `Dictionary.duplicate_deep()` does. |
| P8 | Element write and setter | Writing one element of a packed-array property does not call the property setter (the whole-array assignment did, once). This confirms the 4.7 row in `breaking-changes.md`. |
| P9 | `PackedInt32Array` range | A stored value wraps to 32 bits: `3000000000` reads back as `-1294967296`; `& 0xFFFFFFFF` recovers `3000000000`. |
| P10 | Byte layout | `encode_s64`, `encode_u32`, `PackedInt32Array.to_byte_array()`, `PackedInt64Array.to_byte_array()` and `PackedFloat32Array.to_byte_array()` are little-endian. |
| P11 | Float bit patterns | `encode_double(-0.0)` writes the sign bit (`0000000000000080`), and `-0.0 == 0.0` is true. `encode_double` **preserves NaN payloads**: a signalling-NaN pattern `0x7FF0000000000001` and the all-ones pattern both re-encode unchanged, and a `PackedFloat32Array` NaN payload survives `to_byte_array()`. The `NAN` constant encodes as `0x7FF8000000000000`. So normalisation needs explicit branches (`f != f`, `f == 0.0`) and cannot be left to the engine. |
| P12 | String ordering | `Array.sort()` on `String` elements orders by code point, including characters above the BMP (probed `a` 97, `ÿ` 255, `U+FFFF`, `U+1F600` in that order), which equals UTF-8 byte order. `Array.sort()` on `StringName` elements is **not lexical** (`[&"b", &"a", &"c", &"aa", &"B"]` sorted to `[&"c", &"a", &"b", &"B", &"aa"]`). `StringName == String` for equal text is true, and their UTF-8 bytes are equal. Re-probed: the `StringName` order above reproduces on 4.7.2; it looks intern-order or pointer based, so **`StringName` ordering is not a function of text and may change between runs and builds**, not merely "not lexical". |
| P13 | `HashingContext` | `SHA-256("abc")` is `ba7816bf8f01cfea414140de5dae2223b00361a396177a9cb410ff61f20015ad`; `finish()` returns 32 bytes; `decode_s64(0)` of them is `0xEACF018FBF1678BA` (the D.12 value for `H64("abc")`); one `HashingContext` object is reusable by calling `start` again after `finish`. |
| P14 | Throughput (not a release export) | SHA-256 over 4 KB: 0.009 ms; 32 KB: 0.071 ms; 150 KB: 0.332 ms; 300 KB: 0.664 ms. `PackedInt64Array` of 12,000 elements `to_byte_array()`: 0.002 ms. Writing 500 tagged ints with `encode_s64` into a preallocated buffer: 0.020 ms. `DetRng.derive` (three `lsr` calls and two `mix64` calls): 0.48 microseconds per call. Re-measured in the revision (1,000,000 calls, all 48 warnings at 2): the ADR-0010 § 5.4 body `floor_div(a, b) = trunc_div(a - posmod(a, b), b)` costs 0.20 microseconds per call, the first draft's quotient-plus-sign-fix body 0.21, a `trunc_div` call 0.10, the `posmod` builtin 0.018 and a bare `%` 0.018 (the first draft measured 0.37 against 0.016 for a bare `/` in a different loop: the ratio to a bare operator is about 12, not 23). Both bodies agreed with a float oracle for `a` in -50..50 and `b` in 1..11. Windows editor binary only; re-run on the release template (S0-d). |
| P15 | Vectors | Every D.10 golden vector (`fnv64("horror.tell")`, `mix64(1)`, `mix64(0x8000000000000000)`, the five `rng_derive` values, epoch 1 and epoch 2, `unit` 0.69791535127599 and `rng_int(600, 1800)` = 1438) reproduced exactly on 4.7.2 and in the Python reference. |
| P16 | Hex bytes | `PackedByteArray.hex_encode()` gives lowercase hex of the bytes in order; `String.hex_decode()` **accepts uppercase** and reports an error (empty array) on odd length or a non-hex character, so exact-width and lowercase validation (SL F-10) must be done before decoding. |
| P17 | `Vector2i` width (re-probed) | Components are 32-bit: the x component of `Vector2i(2147483647, 0)` plus 1 stores `-2147483648`; `Vector2i(2147483647 + 1, 0)` reads back `-2147483648`; `Vector2i(65536, 0) * 65536` reads x as `0`. No warning fires with all 48 warnings at 2 (`NARROWING_CONVERSION` does not cover it). |
| P18 | Decimal literal above INT64_MAX (re-probed) | `9223372036854775808` loads silently as `-9223372036854775808` and `9223372036854775809` as `-9223372036854775807` (wrap, no message). `10000000000000000000` and larger log `Cannot represent ... too large` and read `9223372036854775807` (the script still loads). `-9223372036854775807 - 1` and `1 << 63` both give `-9223372036854775808`. |
| P19 | Compound division (reviewer probe, re-probed) | With `integer_division` at 2, `c /= b` and `e %= b` on two ints both load and run (`7 /= 2` gives 3, `-7 %= 2` gives -1). The engine gate covers the binary `a / b` only, and no engine warning covers `%` or `%=`. |
| P20 | Warning gate on the first draft's Key Interfaces (re-probed) | With `SHADOWED_VARIABLE`, `SHADOWED_VARIABLE_BASE_CLASS`, `SHADOWED_GLOBAL_IDENTIFIER` and `UNSAFE_CAST` at 2: `static func h(seed: int, ...)` fails ("same name as a built-in function"); `var hash: int` fails (same); `derive(world_seed, stream_key, counter)` beside `static func stream_key` fails ("shadowing an already-declared function"); `var e := v as Dictionary` on a `Variant` fails ("Casting "Variant" to "Dictionary" is unsafe"). These load cleanly: `h(variation_seed, ...)`, `derive(world_seed, key, counter)`, `var d: Dictionary = v` and `var i: int = v` (typed assignment from a `Variant`), `Array[int].assign(d.keys())`, `Array[String].assign(d.keys())` for a dictionary with `StringName` keys (the elements become `String`), and `(raw >> 11) / 9007199254740992.0` with `lo + floori(u * (hi - lo + 1))` (int-to-float promotion in a binary operator, no `float(` call): no warning among the 48. |
| P21 | JSON number type (re-probed) | `JSON.parse_string("{\"n\": 5}")["n"]` is `TYPE_FLOAT`, and `str()` of it prints `5.0`; every parsed JSON number is a `float`. |
| P22 | `Dictionary` key identity of `String` and `StringName` (re-probed) | `d["a"] = 1; d[&"a"] = 2` leaves one key (size 1; the stored key is a `String`, the first form inserted). |

## ADR Dependencies

| Field | Value |
|-------|-------|
| **Depends On** | ADR-0003 (state blocks, the `snapshot` callable, `WriteScope.open_for` / `has_opened`, `ModuleFacet.rng`, the `kernel.clock` view carrying `(T, world_seed, restore_epoch)` with `restore_epoch` read-only, non-persisted and non-hashed, a second kernel-internal `SnapshotContext` for hash reads outside `SAVING`, fault domains; Proposed), ADR-0004 (the touch serial that validates the hash cache, `Kernel.touch_serial(block_index)`; Proposed), ADR-0001 (the composition root injects narrow facets; no autoload or global; cited for injection only, because ADR-0001 does not mention `restore_epoch` or `kernel.clock`; Proposed), ADR-0003, ADR-0004 and ADR-0001 must be Accepted before this ADR is Accepted, and the ADR-0003 amendment batch under Related must have landed. |
| **Soft references (not acceptance prerequisites)** | ADR-0010 (source layout, allow-lists and the lint mechanism this ADR's § 1.7 rules run on; Proposed is enough to author against, but the allow-list entries listed under Related must be in its text; Proposed). |
| **Enables** | ADR-0006 (the save container stores `hash_algo`, `state_hash`, `block_state_hash`, `hex64`), ADR-0007 (golden `catalogue_hash` and template hash vectors, § 3.4 / § 3.5 inputs), ADR-0011 and ADR-0012 (Graph data model and propagation: integer headroom, the packed-array payload shape, the `(step_no << 20) \| subject_id` key), ADR-0013 (§ 4.6 geometry and visual hash, `low32`), ADR-0015 (Body integer kinematics), ADR-0016 (restore verification), ADR-0017 (Emitter `floor_div`, TR-emitter-038), ADR-0019 (TR-pc-027 timers), ADR-0021 (the weld property test uses the hash), ADR-0022 and ADR-0023 (threat draws through `rng_derive_epoch`), ADR-0024 (migrates PF D.14 to `Ihash.h32`), ADR-0025 (Sanity streams), ADR-0028 (presentation hash), ADR-0029 (Audio variation via `Ihash`), ADR-0031 (the Overlay's hash display). ADR-0010 is a dependency, not an enabled ADR; its DET rule pack is a forward reference to § 1.7. ADR-0004's audit reads through the internal `block_hash_any` (§ 5.3), a read-only use that is not an enable edge. |
| **Blocks** | Epics: Command & Simulation Layer (hashing, RNG, determinism tests), Life-Support Graph (canonical payload, goldens), Save/Load (hash fields), and every story that registers a state block, draws a random number, divides two possibly negative integers or computes a hash. |
| **Ordering Note** | Written in parallel with its siblings and revised after the 2026-10-28 reviews and rulings; every assumption about a sibling is listed under Related as a cross-ADR amendment required before Accept. **Scope edges.** ADR-0003 owns the `snapshot` callable and the descriptor; this ADR owns what the kernel does with the payload (encode, hash, cache). ADR-0004 owns the dirty-key layout and the debug check that every changed block marked a key; this ADR owns the per-block hash cache, which adopts ADR-0004's touch serial as its validity input, and the hash-side stale-cache detector. ADR-0006 owns the container and what is stored; this ADR owns what is hashed and `hash_algo`. ADR-0007 owns the hash *input* of content (post-cast typed rows); this ADR owns the bytes and `H64`. ADR-0010 owns the lint tool, the forbidden-token lists and the allow-list files; this ADR owns the rules they enforce and the list of `IntMath` functions. The Acceptance gate is stated under Engine Compatibility, Verification Required. **Fallback if ADR-0004's touch serial does not survive the joint review:** § 5 reverts to invalidation by `mark_dirty` plus the explicit reset on `restore`, `init_world` and `default_state`, the kernel GDD D.12 wording stays as written, KRN-HASH-02 stays as originally written, and the ADR-0004 dependency is dropped; nothing else in this ADR changes, because the cache is diagnostics-only and every verification path is uncached. |

## Context

### Problem Statement
Determinism is the property that makes replay tests, save verification (K-4), the weld property test, MIRROR comparison and later co-op possible, and it fails silently: one `randf()`, one iteration over an unsorted `Dictionary`, one `-7 / 2` that a designer expected to be `-4`, or one owner writing its own byte layout breaks it with no error. The kernel GDD defines the machinery (D.10 RNG, D.12 hash) and twenty-five other GDDs depend on it, but the GDDs disagree or leave gaps on the details that matter at the byte level:

1. The Graph's `canonical_bytes()` is `[u32 tag][u32 len][bytes]` per field in an owner-chosen order (LS-21), while D.12 says the TLV form is "the only canonical-bytes definition in the project" and no owner ships a byte writer.
2. D.12 sorts dictionary keys "bytewise by the encoded key bytes", which orders integer keys by little-endian two's-complement bytes (3, 10, then -5) and strings length-first. That is deterministic but useless for the diff the G4 test must print (OQ-8).
3. The widths of the `state_hash` fold fields are not stated; `T ‖ world_seed ‖ …` could be hashed with several framings (OQ-8).
4. Attention AT-8 computes `counter_eff` itself and calls `rng_derive`, although the restore-epoch re-keying is "the kernel's one helper" (OQ-6).
5. At least five GDDs define their own pure-integer presentation hash (`he_hash`, Attention `hash32`, Lighting `H(id, k)`, Instruments tremor, Perception Filter D.14) and each marks it "ADR candidate".
6. Integer semantics (division of negatives, shift, wrap, literal range) are marked `[UNV]` in a dozen GDDs and each invents its own guard (Emitter `floor_div`, Suit "non-negative operands only", Web Map "every division takes non-negative operands").
7. `restore_epoch` is "owned by Save/Load" and "exposed read-only in `kernel.clock`" (IC-5), and its exclusion from hashes is stated in several places in different words.

### Constraints
- **Set-once, ratified decisions stand.** `mix64` is the SplitMix64 finaliser (R-17: changeable only before the first save format ships); `H64` is the first 8 bytes, little-endian, of SHA-256 through `HashingContext` (R-17, user ruling X13); `hash_algo` 2 names it (SL-16). D.10's constants and golden vectors are acceptance values.
- **Engine**: Godot 4.7.2, GDScript. `int` is signed 64-bit and wraps; `float` is IEEE-754 binary64; `Dictionary` preserves insertion order; `Array.sort()` is not stable; packed arrays alias by reference. Registry stances in force: no kernel autoload or global (`kernel_autoload_or_global`); blocks are `RefCounted` plain data (`block_as_resource_or_node`); a block is written only inside a `WriteScope` and dirty keys are marked only through `scope.mark_dirty` (`write_outside_writescope_or_setter_dirty`); `var_to_bytes` bytes are never hash input (`payload_size_check`).
- **Budgets** (measured on a release export on RM-DESKTOP and RM-DECK, pending spike): a quiet tick performs 0 canonical encodes and 0 kernel container allocations; a read of the hash after a tick that dirtied `k` blocks performs `k` encodes (KRN-PERF-01, -02); a 32-block snapshot captures in at most 16.7 ms including encode and hash (D.11); the P3 budget is 2.0 ms and is unaffected because no hash runs in P3.
- **Hot paths allocate nothing per tick**: nothing in this ADR may allocate in a tick that dirties no block and has no hash reader.
- **No wall clock, no global generator, no scene tree** in anything that feeds a hash (kernel H9, H10).
- Co-op (VS-B) mirrors the host's blocks; the hash definition must be platform-independent in design even though cross-OS identity is verified later.

### Requirements
- One definition of every byte that feeds a hash, shared by all 31 blocks, with no owner-written layout.
- `rng_derive` stays a pure function of `(world_seed, stream_name, counter)`; threat draws re-key per restore epoch through one helper that owners cannot misuse; Graph and every persisted counter never read the epoch.
- Cached per-block hashes for the diagnostics cadence and `layout_hash()` memoisation, and uncached hashes on every verification path, with a debug detector for the stale-cache failure.
- Integer semantics settled once, with a division convention that lints can enforce and a headroom rule that boot assertions can check.
- A replay guarantee stated precisely: what is promised, for which inputs, on which builds and platforms, and what is not.
- Every hash and RNG value that crosses a JSON boundary survives as `hex64` (S-1).

## Decision

### Architecture Diagram

```
 owners (blocks, hooks)            kernel                                      consumers
 ----------------------            ------                                      ---------
 ModuleFacet.rng (RngFacet) --->  RngRegistry  (stream name -> RngStream{key, epoch_keyed}, BOOTING)
   draw(stream, counter)            DetRng.derive(clock.world_seed, key, counter)       pure statics
   draw_epoch(stream, rng_counter)  + clock.restore_epoch (read inside the facet only)
   int_in(raw, lo, hi)              DetRngFloat.int_in  (the one sanctioned float expression)

 owner.snapshot(ctx) --payload-->  CanonicalEncoder.encode(payload)  -->  PackedByteArray (TLV, § 3)
                                            |                                   |
                                            |                      Hash64.of_bytes (HashingContext, SHA-256, first 8 bytes LE)
                                            v                                   v
                                   BlockHashCache {hash, enc_len, valid}   block_hash(id, uncached) -> BlockHashResult
                                      ^  valid iff cached serial == touch_serial[block]   (ADR-0004 touch serial, advanced by every granted open_for)
                                      |
                       state_hash(scope, T, uncached) = H64( T || world_seed || fold over blocks sorted by id )  (§ 4)

 pure, stateless, shared:  IntMath (lsr, floor_div, ceil_div, posmod_i, low32)   Ihash (63-bit, presentation and non-RNG variation)   Hex64
 verification:  G3 two-kernel, G4 snapshot/restore/continue, K-4 reproduction compare   all call uncached
```

### 1. The determinism contract: integer-only simulation

**1.1 What is promised.** All promises are about the pair `(save bytes, restore_epoch)`, not the bytes alone, because the epoch re-keys threat draws (D.10).

| Level | Promise | Holds for |
|---|---|---|
| **R1** | Same build, same platform, same `(save bytes, restore_epoch)` and the same ordered command stream plus control-plane request log yield the same `state_hash(scope, N, uncached = true)` at every tick `N`, at any render rate, pause pattern, camera zoom, seat count or MIRROR role. | G3 (two kernels, ALL_SIM), G4 (snapshot, restore, continue, PERSIST), the cross-process test, K-4. |
| **R2** | Same build commit on Windows and Linux x86-64 (macOS advisory until a target decision): the hashes of the golden vectors (§ 2.4, § 2.6, § 3.7, § 4.5) are equal, and the Graph fixtures' golden files too once the Graph epic commits them. Expected by construction (integer arithmetic, explicit little-endian bytes, SHA-256, no platform call), **UNVERIFIED** until the cross-OS vector job runs (TR-lsg-031). | The committed `tests/golden/` files. |
| **R3** | Across commits, hashes are comparable only while `hash_algo`, every compared block's `schema_version` and the block payload shapes are unchanged. A deliberate behavioural change moves hashes; goldens are regenerated by an explicit, reviewed job that records the commit. | Golden maintenance. |
| **R4** | A restore with a different `restore_epoch` differs in exactly the threat draws (Attention `attention.tell`, Horror `horror.tell`), nothing else (KRN-EPO-02). | D.10 re-keying. |

**Not promised**: identity across Godot versions (4.7.2 is pinned; a patch upgrade re-runs the goldens), identity of physics-coupled blocks (there is no engine physics in the simulation, so none exist), network timing in VS-B, and anything in `PERCEPTION_OUT` or `PROFILE`-scoped presentation state.

**1.2 Types.** Simulation state is `int`, `bool`, `String` / `StringName` (identifiers, `def_id`), and containers of those, including packed integer and byte arrays. A `float` is not permitted in a persisted SIM block or in any value a SIM rule reads from another block. The sanctioned float expressions are exactly these: (a) `DetRngFloat.unit` and `DetRngFloat.int_in` (§ 2.1, § 2.5), the kernel's one pinned float arithmetic path, and `CanonicalEncoder`'s private `_encode_float` (§ 3.8), which only normalises and copies the bits of a float it is handed and does no float arithmetic; (b) boot-time content casts (`roundi(x * scale)`, ADR-0007), whose results are integers; (c) the driver's `delta_us` conversion (ADR-0001), outside the simulation; (d) test oracles and analysis. Presentation uses floats freely but never writes a sim field and never reads one back through a path a save can see (ADR-0009). A block that must hold a float (none is known at MVP) is added to a committed allow-list, with a one-line reason, and the encoder's strict mode (§ 3.5) then permits it; the allow-list is empty at MVP.

**1.3 Pinned integer semantics** (P1 to P5, P9; recorded in Engine Compatibility). The simulation relies on: 64-bit two's-complement wrap for multiplication in `mix64` and `fnv64` only (intended, and the one place wrap is a feature); arithmetic `>>`; truncating `/` and `%`. It never relies on wrap anywhere else (§ 1.5). Consequences stated as rules:
- Every right shift of a value that can be negative is a masked logical shift: `IntMath.lsr(x, n)` with `1 <= n <= 63`. A bare `>>` is allowed only on an operand proven non-negative (a `& mask` just before, or an unsigned-by-construction value), because `>>` on a negative is arithmetic (P3).
- No shift of a negative **literal** appears in source (P3: a parse error). Constants are written as signed decimal `int` literals within range. A hexadecimal literal above `0x7FFFFFFFFFFFFFFF` never appears (P2: the engine logs an error and the literal saturates to INT64_MAX while the script still loads, so a load check does not catch it). **A decimal literal greater than 9223372036854775807 in magnitude is banned** (P18: values just above it wrap silently and larger ones saturate, so a mistyped constant, such as a digit added to a signed decimal, silently becomes a wrong `int`). INT64_MIN is written `-9223372036854775807 - 1` or `1 << 63`, never as a literal of magnitude 2^63. Both checks are lexical (a digits-only token compare against 9223372036854775807, the ADR-0010 SIM-03 sibling, with a bad fixture each); the six § 2.1 constants are all below 2^63 in magnitude. Hex constants in comments are for reference only.
- A shift count is a constant in `1..63` or a variable proven in that range; a count of 64 or more silently wraps (P4, a hardware accident that is defined only for runtime counts 0..63).
- A value stored in a `PackedInt32Array` is read back masked when it is meant as unsigned: `IntMath.low32(x)` = `x & 0xFFFFFFFF` (P9). Layout's `geometry_hash` and `visual_hash` (`low32(H64(...))`, LY-26 e) are compared only after this mask (TR-section-032).

**1.4 Division and modulus.** GDScript `/` and `%` truncate toward zero (P5), which is wrong for any quotient a designer meant as a floor. Three rules:
1. **Default: the helpers.** In `src/sim/**` and `src/kernel/**`, division and modulus go through `IntMath.floor_div(a, b)`, `IntMath.ceil_div(a, b)` and `IntMath.posmod_i(a, b)`, all with `b > 0` and correct for either sign of `a` (`floor_div(-7, 2)` is `-4`; `posmod_i(-7, 2)` is `1`; `ceil_div(-7, 2)` is `-3`). `b <= 0` is a caller defect: the helper returns 0 (never aborts: an aborted GDScript function returns a default the caller cannot tell from a result), and the debug-build `assert(b > 0)` fires in tests. ADR-0005 is the sole owner of the `IntMath` function list (`trunc_div`, `floor_div`, `ceil_div`, `posmod_i`, `lsr`, `low32`); ADR-0010 § 5.4 cites it and keeps only the file path and the single-annotation rule. Bodies: `trunc_div(a, b)` is `a / b` (the one annotated site); `floor_div(a, b)` is `trunc_div(a - posmod(a, b), b)` (exact, because `a - posmod(a, b)` is a multiple of `b`); `posmod_i(a, b)` is the global `posmod(a, b)` builtin, which returns an `int` for two ints (P5); `ceil_div(a, b)` is `floor_div(a + b - 1, b)`; each sits behind the `b <= 0` guard.
2. **There is no bare `/` or `%` anywhere else.** **The engine gate covers the binary `a / b` only** (P6, P19). `integer_division` at error level rejects `a / b` on two ints, but `c /= b` loads and runs, and no engine warning covers `%` or `%=`. The lexical rules close the holes: ADR-0010 `SIM-04` bans `%`, `/=` and `%=` outside `IntMath` (bad fixtures `x /= y` and `x %= y`; any `//`-style helper joins the ban), and `SIM-05` keeps exactly one `@warning_ignore("integer_division")` in `src/`, on the `return a / b` of `IntMath.trunc_div`. This ADR adopts that design instead of a marked bare form and supplies the semantics: `IntMath` exposes `trunc_div` (the engine's truncation, named when truncation is meant), `floor_div`, `ceil_div`, `posmod_i`, plus `lsr` and `low32`; none of the new helpers divides. The cost is measured (P14): the static `floor_div` costs about 12 times a bare operator (0.20 against 0.017 microseconds in a tight loop on the editor binary, re-measured). If S1 (SP-ENG-1) shows that cost matters in the Graph propagation or the Emitter exposure loop on a release export, the remedy (inlining a proven-non-negative form in named hot files) is an ADR-0010 amendment decided on that measurement, not an exception taken here.
3. **Fractional designer values are scaled once.** A quantity like 1.5 tiles is stored as an integer in a named unit (ADR-0007 `SCALED` fields); a rule that needs a ratio multiplies first and divides once, on a non-negative numerator (the Emitter wane formula in the Emitter GDD is the model).

The Suit "convention for negative operands" (ADR candidate 4, TR-suit-030) is therefore: floor for every quotient, `floor_div` / `posmod_i`; a thermal integrator that needs truncation toward zero calls `IntMath.trunc_div(a, b)` explicitly, which names the intent. Emitter `floor_div` (non-negative, checked) is a special case of the helper here and Emitter's lint fixtures stay valid.

**1.5 Overflow and headroom.** GDScript `int` wraps silently (P1) and nothing in the simulation detects it at run time. Overflow is designed out, not caught:
- Every sim intermediate stays below `2^62` in magnitude, so a sum of two such values cannot wrap (the Graph and Attention already state this: TR-attention-036, Graph D.13).
- Each owner declares the maxima of its factors as constants; its boot check multiplies declared maxima and fails boot (its own domain's boot code) when a product reaches `2^62` (the Graph's `GRAPH_BOOT_PRODUCT_OVERFLOW` is the model). Tests verify a sample of worst-case inputs against an arbitrary-precision oracle (an offline generator or a test-only big-integer helper; ADR-0010 chooses the tooling).
- The two wrap-intended computations are `mix64` / `fnv64` / `Ihash` (§ 2) and the fold arithmetic in § 4, which use only multiplication and XOR on the full 64 bits and are covered by golden vectors.
- **32-bit engine types.** `Vector2i` components and `PackedInt32Array` elements are 32-bit (P9, P17): a sum or product stored back into one truncates silently, with no warning (`NARROWING_CONVERSION` does not cover it), and the encoder's widening of `Vector2i` to 8-byte ints hides the truncation from the hash (both sides hold the same wrong value). Owners that use these types declare their maxima against `2^31`, do the arithmetic in plain `int` and range-check before the store, and carry a boot check or unit test for it; the `2^62` headroom rule above is for plain `int` only.
- Every JSON-sourced number is a `float` on parse (P21). Values that travel through JSON are below `2^53` and are cast only through the § 4.8 reader, or are `hex64` strings.

**1.6 Ordering.** Anything that feeds a hash, a rebuild, a registration or a result order iterates in an order that does not depend on insertion history, the allocator or the engine version:
- **Dense arrays in ascending id order** is the default container for entities (Layout, Graph, Body). Tombstones stay in place and are part of the array.
- A `Dictionary` is never iterated for order in simulation code unless its keys are first extracted and sorted (through `CanonicalEncoder.sorted_int_keys` / `sorted_string_keys`, which extract the keys into a typed array and `sort()` it on all-`int` or all-`String` keys; a mix is refused). Insertion order is preserved by the engine but erase-and-reinsert changes it, and a restore rebuilds in file order.
- **`StringName` is never sorted.** `Array.sort()` on `StringName`s is not lexical and not a function of the text, and may change between runs (P12); convert with `String(k)` first. This is a must-never for canonical ordering and for any sorted iteration.
- `Array.sort_custom` and any sort over keys that can tie are banned in simulation code, because the sort is not stable. A sort that is needed packs the sort key and the stable id into one `int` (`(primary << 20) | id`, with headroom checked) and sorts the `int`s with `Array.sort()`.
- `hash()`, `get_instance_id()`, `ResourceUID`, `shuffle()` and `pick_random()` are banned (engine hash values are not a stable contract; instance ids differ between runs; the other two draw from the global generator).

**1.7 What the lints enforce.** The lint tool, the forbidden-token lists and the allow-list files are ADR-0010's: this ADR keeps no list of its own for time, RNG, float or division tokens and cites ADR-0010 `SIM-02` to `SIM-05`, `SIM-09` and the DET rules, which take the union of this ADR's and ADR-0001's entries. Token rules use word boundaries (`\bsin\b`, `\bcos\b`, `\bfloat\b`, `\bseed\(`), so `using`, `cost`, `set_world_seed(` and `float_allowlist` do not match. Where a rule needs more than a regex, the third column says so.

| Rule | Where | Enforced by |
|---|---|---|
| Time, frame-counter and global-RNG tokens (`Time.` members, `OS.get_ticks_*`, `OS.get_unix_time*`, `Engine.get_frames_*`, `get_process_delta_time`, `randi`, `randf`, `seed(`, `RandomNumberGenerator`, `shuffle`, `pick_random` and the rest of the ADR-0010 list) | `src/**` outside the ADR-0010 `SIM-02` allow-list | ADR-0010 `SIM-02` (regex, allow-list with a reason per entry). Render-time timers in Adapter / Presenter (TR-pc-027) use `delta` or `present_time`, which need no `Time.*` token, so this ADR keeps no separate allowance. Wall time has one sanctioned source, the Save/Load-only `WallClock` port (ADR-0010), and never enters a block, a hash or an ordering decision |
| No `float`, `Vector2(`, `Vector3(`, `sqrt`, `pow(`, `sin`, `cos`, `tan`, `atan`, `atan2`, `asin`, `acos`; no `float(` / `floor` conversions | `src/shared/**`, `src/sim/**` | regex (ADR-0010 `SIM-03` / `SIM-04`), except the two allow-listed files `src/shared/det/det_rng_float.gd` and `src/shared/det/canonical_encoder.gd` on `float_allowlist.json`; `det_rng_float.gd` is also exempt from the `SIM-04` `float(` / `floor` bans (§ 2.1 needs `floori` there) |
| No `hash(`, `get_instance_id`, `.sort_custom(`, `ResourceUID` | `src/sim/**`, `src/kernel/**` | regex (ADR-0010 DET rules) |
| No `/` on two ints and no `%` outside `IntMath`; no `/=` or `%=`; exactly one `integer_division` ignore, in `IntMath.trunc_div` | `src/shared/**`, `src/kernel/**`, `src/sim/**` | engine warning at error level for the binary `/` only, plus ADR-0010 `SIM-04` for `%`, `/=`, `%=` (bad fixtures `x /= y`, `x %= y`) and `SIM-05` |
| No decimal integer literal above 9223372036854775807 and no hexadecimal literal above `0x7FFFFFFFFFFFFFFF` | `src/**` | lexical, digits-only token compare (ADR-0010 `SIM-03` sibling); bad fixtures `9223372036854775808` and `0x8000000000000000`. A load check cannot catch either (P2, P18) |
| Iterating a `Dictionary` (`for k in d`, `.keys()`, `.values()`) without a sort | `src/sim/**`, `src/kernel/**` | AST or runtime counter (not a regex); fallback: `CanonicalEncoder.sorted_int_keys(d)` / `sorted_string_keys(d)` is the only allowed form and the regex bans the raw forms |
| `StringName` passed to `sort()` | same | AST (type-aware); fallback: a test that sorts the canonical key set through `CanonicalEncoder` only |
| Canonical byte writers (`encode_s64`, `encode_u32`, `encode_double`, `to_byte_array`) and hashing of a `var_to_bytes` result are forbidden in `src/kernel/`, `src/sim/` and block owners. They are allowed in `src/shared/det/` and tests. `src/save/` may use them for container byte layout, `slot_epoch.bin` and integrity hashes (`payload_sha256`, the trailer), whose output never feeds `H64` or `state_hash` | `src/**` | regex |
| `HashingContext` only in `src/save/`, `src/data/`, `src/shared/det/hash64.gd` and `src/kernel/state_hasher.gd` (the list is ADR-0010's § 4.1 / IO-01; mirrored, not restated, here) | `src/**` | regex |
| No `Variant`, bare `Array`, or `as` cast of a `Variant`, except `canonical_encoder.gd` (the unpacking idiom of § 3.8) and the named pass-through signatures | `src/shared/**`, `src/kernel/**`, `src/sim/**` | ADR-0010 `SIM-09` with `canonical_encoder.gd` on `dictionary_variant.json`; `UNSAFE_CAST` at error level makes `as` on a `Variant` a load error |
| No member, parameter or local named after a `@GlobalScope` or GDScript utility function or after a sibling member | `src/**` | engine `SHADOWED_*` warnings at error level (ADR-0010 § 5.1) and the load-all check; no lint needed |
| No re-derivation of the epoch key (`RESTORE_KEY_STRIDE` appears only in `DetRng` and `RngFacet`) | `src/**` | regex |
| No block declares a float field unless on the allow-list | block payloads | strict-mode encoder in tests (§ 3.5) |

### 2. Keyed RNG

**2.1 The pure family** (D.10, set-once; this ADR restates it so the constants have one home in the ADR set). All functions are stateless statics in `DetRng`, except the two float-bearing ones (`unit`, `int_in`), which live alone in `DetRngFloat` (`det_rng_float.gd`) so that the sanctioned float arithmetic path is one file. All of these files are in `src/shared/det/`, and `float_allowlist.json` for `src/shared` names exactly two files: `det_rng_float.gd` and `canonical_encoder.gd` (§ 3.8). Constants are signed decimal `int` literals (P2); the hexadecimal values are reference only.

| Constant | Signed decimal | Reference hex |
|---|---|---|
| `GOLDEN` | `-7046029254386353131` | `0x9E3779B97F4A7C15` |
| `C1` | `-4658895280553007687` | `0xBF58476D1CE4E5B9` |
| `C2` | `-7723592293110705685` | `0x94D049BB133111EB` |
| `FNV_OFFSET` | `-3750763034362895579` | `0xCBF29CE484222325` |
| `FNV_PRIME` | `1099511628211` | `0x100000001B3` |
| `RESTORE_KEY_STRIDE` | `1048576` | `2^20` |

```
lsr(x, n)      = (x >> n) & ((1 << (64 - n)) - 1)                  1 <= n <= 63
mix64(z)       : z = (z ^ lsr(z, 30)) * C1 ; z = (z ^ lsr(z, 27)) * C2 ; return z ^ lsr(z, 31)
fnv64(name)    : h = FNV_OFFSET ; for each UTF-8 byte b: h = (h ^ b) * FNV_PRIME
stream_key(n)  = mix64(fnv64(n))
derive(world_seed, key, counter) = mix64(world_seed ^ key ^ (counter * GOLDEN))      == rng_derive(world_seed, name, counter) for key = stream_key(name)
unit(raw)      = lsr(raw, 11) / 9007199254740992.0                      in [0, 1); the int promotes to float in the binary operator, no float( call
int_in(raw, lo, hi) = lo + floori(unit(raw) * (hi - lo + 1))             lo <= hi, hi - lo + 1 <= 2^32; the int span promotes likewise
```

`mix64` is bijective with the single fixed point `mix64(0) = 0`; this is harmless for `derive` (the stream key is never 0 for a real name) and is why `Ihash` (§ 2.6) demands a non-zero seed or salt.

**2.2 Streams are registered; the key is computed once.** `fnv64` and `mix64` of a name are computed at registration and cached in an `RngStream` object, never per draw (D.10).

```gdscript
class_name RngStream extends RefCounted    # read-only after registration
var name: StringName
var key: int                               # stream_key(name), computed once
var epoch_keyed: bool                      # true for threat streams only (§ 2.4)
```

Owners register streams in `BOOTING` through the facet (§ 2.3). A duplicate name, or two names whose `key` is equal, fails boot with the kernel's `DUPLICATE_ID` fault (ADR-0003 domain 0, code 8) with the stream name in the detail. **Stream names** are lower-case dotted ASCII, `<domain>.<name>` (`horror.tell`, `attention.tell`, `graph.fail`), optionally followed by `/`-separated decimal integers formatted with `str(int)` and no padding (`sanity.lie/3/17`). Names that depend on a runtime id (Sanity's per-seat, per-subject streams, SN-24) are obtained through `facet.dynamic_stream(name, epoch_keyed)`, which memoises the `RngStream` in a bounded cache (`MAX_DYNAMIC_STREAMS` 4096; past it the stream is built without caching and `RNG_STREAM_CACHE_FULL` is counted). The cache is a performance structure only: it is rebuilt on demand, never persisted, never hashed, and its contents are a pure function of the names asked for. The `fnv64` cost per distinct dynamic name is one-time; the string allocation per first use is an accepted cost on a player-driven path (open question 1).

**2.3 The facet owners use.** A module draws only through an injected `RngFacet`, issued by the composition root as `ModuleFacet.rng` (the field is ADR-0003's; null for a module that declares no streams, which structurally enforces "Web Map registers no `rng_derive` stream", TR-webmap-031, and "Layout draws no RNG", TR-section-026). The facet reads `world_seed` and `restore_epoch` from the `kernel.clock` view (ADR-0003) inside its methods; an owner never passes either, so the epoch cannot be mixed in wrongly.

```gdscript
class_name RngFacet extends RefCounted
func register_stream(name: StringName, epoch_keyed: bool = false) -> RngStream      # BOOTING only
func dynamic_stream(name: String, epoch_keyed: bool = false) -> RngStream            # memoised, bounded
func draw(stream: RngStream, counter: int) -> int          # rng_derive(world_seed, stream, counter); plain streams only
func draw_epoch(stream: RngStream, rng_counter: int) -> int   # rng_derive_epoch; epoch-keyed streams only
func int_in(raw: int, lo: int, hi: int) -> int             # rng_int; counts RNG_RANGE_VIOLATION
```

`draw_epoch` computes `counter_eff = rng_counter + RESTORE_KEY_STRIDE * restore_epoch` and calls `derive`; it is the only code that reads `restore_epoch` for a draw. Violations are counters, not faults (CR-19): `draw` on an epoch-keyed stream and `draw_epoch` on a plain stream count `RNG_STREAM_KIND_VIOLATION`; `rng_counter` outside `0..RESTORE_KEY_STRIDE - 1` or `restore_epoch` outside `0..2^32 - 1` counts `RNG_COUNTER_RANGE_VIOLATION` and the value is still computed as written; `int_in` with `lo > hi` or a span above `2^32` returns `lo` and counts `RNG_RANGE_VIOLATION`; a negative `counter` on a plain stream counts `RNG_COUNTER_RANGE_VIOLATION` and is still computed. Each violation also reports through the CR-19 channel. The `request_restore` pre-check for an out-of-range epoch is ADR-0003's existing `RESTORE_EPOCH_RANGE` fault (code 6). The pure statics in `DetRng` take explicit arguments and are what unit tests and offline tools call.

**2.4 Epoch re-keying: one place, threat draws only** (resolves OQ-6). `rng_derive_epoch(world_seed, stream, rng_counter) = rng_derive(world_seed, stream, rng_counter + RESTORE_KEY_STRIDE * restore_epoch)`.
- **Epoch-keyed streams at MVP**: `attention.tell` (AT-8) and `horror.tell` (the REACH length, HE-42). Nothing else. A stream is registered `epoch_keyed = true` only with a technical-director ruling; the facet makes the registration visible in one list.
- **Everything else is plain**: the Graph's thaw-burst and spark draws (`graph.fail` keyed `(step_no << 20) | subject_id`, a 63-bit owner key, valid per the D.10 maintenance note), Sanity's `sanity.*` streams, and every persisted counter. They never read the epoch and replay exactly after any restore.
- **`restore_epoch` is stated here once** (IC-5). Save/Load owns it, persists it in `slot_epoch.bin` and is its single registry owner (ADR-0006). The kernel holds a copy in the ADR-0003 `kernel.clock` view as a `u32`: read-only, **non-persisted and non-hashed**, set by `request_restore(payload, restore_epoch, …)` and `request_new_game(world_seed, restore_epoch)` before the first `restore()` / `init_world`. It is in no state block, in no canonical byte string, in no `state_hash` input and in no kernel snapshot payload; ADR-0006's recovery copy in the container header is file metadata covered only by the container trailer, never an `H64` input. The only simulation-visible use is inside `RngFacet.draw_epoch`. This ADR references the registry entry and does not duplicate it.
- **Golden vectors** (reproduced on 4.7.2 and in the Python reference; `world_seed = 0x5EED000000002026`): `rng_derive(seed, "horror.tell", 17)` = `0xB2AA94991B52A02E` (`unit` 0.6979153513, `rng_int(600, 1800)` = 1438); counter 0 = `0x4988DBEFD7F20FF5`; `rng_derive(seed, "graph.failure", 0)` = `0x0CEEC43F97221B55`; with `world_seed = -1` (all ones), `("horror.tell", 1)` = `0x883EA97FED4A4613`; epoch 1 (`counter_eff` 1,048,593) = `0x6EFE2D9E722654C6`; epoch 2 (2,097,169) = `0xA46E009E591F2390`. Also `fnv64("horror.tell")` = `0xE500B5B58A7689E2`, `mix64(1)` = `0x5692161D100B05E5`, `mix64(0x8000000000000000)` = `0x25C26EA579CEA98A`. All values here and below print as numeric hex of the signed `int`'s bit pattern, most significant digit first.

**2.5 The one float path.** `unit` and `int_in` are the only floating-point expressions in the simulation. `lsr(raw, 11)` is below `2^53`, so the integer promotes to `float` exactly and the division by `2^53` is exact; `unit * span` (the `int` span promoted by the binary operator, exact because the span is at most `2^32`) is a single IEEE-754 binary64 multiplication, correctly rounded on every platform Godot supports, and `floori` is exact. There is no transcendental, no fused multiply-add opportunity in a single product, and no accumulated float. Both promoted forms need no `float(` conversion call and load cleanly with all 48 warnings at 2 (P20). The only possible disagreement with an exact-integer evaluation is a product within `2^-53` relative of an integer boundary, a probability of about `span * 2^-53` per draw (below `2^-42` for the spans in use); that is the bias D.10 already states. The kernel GDD owns this formula and its goldens; this ADR does not replace it with an integer split (a 64-bit split overflows at spans near `2^32`, and a different rounding would silently change drawn values). The test oracle is an arbitrary-precision evaluation of `floor(top53 * span / 2^53)`; a mismatch there is reported to the technical-director as a candidate for an exact integer path, never fixed ad hoc.

**2.6 The shared pure-integer hash `Ihash`** (closes the "ADR candidate" in HE-42, AT-28, LO-19 and Instruments; PF D.14 keeps its own masked hash until ADR-0024 migrates it). It is the form `he_hash` already defines, parameterised by a seed:

```
ihash(variation_seed, k1, k2, k3, salt) = lsr( mix64( mix64(k1 ^ variation_seed) ^ mix64(k2 ^ salt) ^ (k3 * GOLDEN) ), 1 )
```

The result is a **non-negative 63-bit `int`**, so every `mod` on it acts on a non-negative number. `Ihash.h32(...)` is `ihash(...) & 0xFFFFFFFF`. It is not a random stream: it has no counter and no state, it is never persisted, and it is used for variation that must be reproducible without being a persisted draw (Horror mark seeds, LURK waypoint rank and L3 timing with `HE_HASH_SEED` 0x4845; Attention beat jitter `h32(HASH_SEED, tell_epoch, k, 0, SALT_BEAT)`; Lighting gutter onset `H(emitter_id, k)`; Instruments tremor from `ref_key`). Rules: callers pick a fixed non-zero seed or salt (`mix64(0) = 0` makes `ihash(0, 0, 0, 0, 0)` equal 0); inputs are stable ids and indices, never `tick_alpha`, `present_time` or a frame count; each system keeps its own seed constant so changing one re-rolls only that system. Presentation may call it (it is pure); it must never call `DetRng.derive`. **A truth-channel visual that must be reproducible (the Failure & Trail's `trail.vis`) uses `Ihash` keyed by `world_seed` and a fixed per-system seed**: `Ihash.h(TRAIL_VIS_SEED, subject_id, stamp_index, world_seed, SALT_TRAIL_VIS)`, indexed by `IntMath.posmod_i(h, n)` for variant, phase and flip. The simulation never draws a `trail.vis` stream, so the hash reproduces the variant without opening the sim RNG API to presentation, and a different `world_seed` still changes at least one variant. `TRAIL_VIS_SEED` is a fixed non-zero constant the Trail GDD defines; its FT-20, D.6 and AC-134 stub vectors regenerate (sync item), and ADR-0009 line 137 changes to "reads `Ihash` keyed by `world_seed` (ADR-0005 § 2.6)". The alternative of letting presentation call the pure `DetRng.derive` for names ending `.vis` under a lint row was rejected. **Overflow waiver:** the Perception Filter's PF-37 "no dependence on integer overflow" wording is waived for the `Ihash` family, because 64-bit wrap is the pinned, golden-tested behaviour (deterministic on 4.7.2, P1; formally C++ undefined behaviour, guarded by the golden vectors and S0-b). This ADR chose the waiver plus ADR-0024's migration of PF D.14, with no separate masked variant. Presentation hashes use `Ihash` or `Ihash.h32` with a fixed non-zero per-system seed. Reference vectors (Python and GDScript agree): `ihash(0x4845, 1, 0, 0, 1)` = `0x2B1792AE18418A3E`; `ihash(0x4845, 17, 3, 5, 2)` = `0x31500F22473CCE18`; `ihash(0x4845, -1, -1, -1, 5)` = `0x0CA990E4614F7103`.

**2.7 Who uses what.**

| System | Draws | Mechanism |
|---|---|---|
| Graph | thaw-burst (and spark ignition, VS-A) | plain stream `graph.fail`, counter `(step_no << 20) \| subject_id`, `subject_id < 2^20` asserted |
| Attention | tell length | epoch-keyed `attention.tell`, `draw_epoch`, then `int_in`; counter persisted; beat jitter by `Ihash` |
| Horror | REACH length | epoch-keyed `horror.tell`; marks, waypoints, L3 by `Ihash` |
| Sanity | lie, lure, phantom, hide | plain `sanity.<stream>/<PlayerId>[/<stable_id>]` via `dynamic_stream`; counters persisted; draw count fixed per outcome |
| Layout, Body, Assembly, Suit, Web Map, Boarding, Emitter, Interaction, Controller | none | `ModuleFacet.rng` is null |
| Lighting, Instruments, Perception Filter, Audio | presentation variation | `Ihash` (pure), never `DetRng.derive` |
| Failure & Trail | stamp variant, drip phase, fan flip (truth-channel visual) | `Ihash.h(TRAIL_VIS_SEED, subject_id, stamp_index, world_seed, SALT_TRAIL_VIS)`; no sim stream, never `DetRng.derive` (§ 2.6) |

### 3. Canonical encoding

**3.1 One encoder, no owner byte writers** (resolves the LS-21 / D.12 conflict and the framing half of OQ-8). Canonical bytes are produced only by `CanonicalEncoder.encode(value)` over the payload an owner's `snapshot(ctx)` delivers through `ctx.put(payload)`, or over a content value (ADR-0007 inputs). An owner's GDD describes the *shape* of its payload (keys, value types, row order) and never a byte layout. Consequently:
- The Graph publishes no `[u32 tag][u32 len][bytes]` writer. Its `snapshot` payload is a `Dictionary` whose values are named `PackedInt64Array` / `PackedInt32Array` / `PackedByteArray` fields (and the `def_off` / `def_utf8` pair), and the kernel encodes it. The "fixed documented order" of LS-21 is replaced by **sorted key order** (§ 3.3): renaming a payload key changes the hash and therefore needs a `schema_version` bump. The raw-bytes cost the Graph wanted is kept: a packed array reaches the buffer by one `to_byte_array()` copy (P14: 0.002 ms for 96 KB), with no per-element loop.
- `canonical_bytes()` and `block_hash()` on an owner (TR-lsg-021, TR-emitter-038, TR-debug-033) become kernel-side reads: `canonical_encode(snapshot payload)` and the kernel's `block_hash(block_id)` (§ 5). An owner that wants to expose "its hash" forwards to the kernel read; it computes nothing.
- The Item DB's `ItemInstance` "canonical positional encoding with canonical zeros" (TR-itemdb-012) is a **payload-shape rule**: an instance is an `Array` of `int`s in a fixed positional order, every non-applicable field present as the canonical zero, so a holder block has one encoding per state. The encoder then writes each as `INT`. The byte bound in TR-itemdb-029 (96 bytes per instance) was derived from `var_to_bytes(inst.to_array(), false)`, which is ADR-0002's size check and ADR-0006's storage form, not hash input; the TLV size of the same array is 9 bytes per int plus 9 for the array header (open question 6).
- Packed arrays in a snapshot payload are duplicates, never live references to block fields (P7; the ADR-0003 snapshot contract), so neither the encoder nor a worker that later holds the payload can race a write to the block.

**3.2 The type set is closed.** One tag byte, then a self-delimiting body, so no encoding is a prefix of another and the map from normalised values to bytes is injective (D.12; KRN-HASH-01). All counts and lengths are **u64 little-endian**. Integers are 8-byte little-endian two's complement.

| Tag | Type | Body |
|---|---|---|
| `0x00` | NIL | none |
| `0x01` | BOOL | 1 byte, 0 or 1 |
| `0x02` | INT | 8 bytes, little-endian two's complement |
| `0x03` | FLOAT | 8 bytes, IEEE-754 binary64 bits, little-endian, after normalisation (§ 3.4) |
| `0x04` | STRING | u64 UTF-8 byte length, then the bytes; **a `StringName` encodes as the `STRING` of its text** |
| `0x05` | VECTOR2 | two normalised FLOAT bodies (16 bytes, no inner tags); components widen from `float32` to binary64 |
| `0x06` | VECTOR2I | two 8-byte little-endian ints (16 bytes, no inner tags) |
| `0x07` | ARRAY | u64 element count, then that many tagged values in order. Typed and untyped arrays encode identically. |
| `0x08` | DICT | u64 entry count, then `key, value` tagged pairs sorted by key (§ 3.3). Typed and untyped dictionaries encode identically. |
| `0x09` | PACKED | 1 element-type byte, u64 element count, then the elements as raw little-endian bytes |

Packed element-type bytes: `1` `PackedByteArray` (1 byte per element), `2` `PackedInt32Array` (4), `3` `PackedInt64Array` (8), `4` `PackedFloat32Array` (4), `5` `PackedFloat64Array` (8), `6` `PackedStringArray` (each element a u64 length plus UTF-8), `7` `PackedVector2Array` (two 4-byte floats per element). Any other `Variant` type (an `Object`, `Callable`, `Signal`, `RID`, `Color`, `Rect2i`, `Vector3`, a packed array not listed) is `ENC_UNSUPPORTED_TYPE`. Adding a tag or a packed type changes the hash definition and needs a new `hash_algo` (§ 4.4). `bool` is never confused with `int`: they have separate tags.

**3.3 Dictionary key order** (resolves the key-order half of OQ-8; this deliberately replaces D.12's "sorted bytewise by the encoded key bytes"). A dictionary's keys are all `int` or all of the string family (`String` / `StringName`); a mixture is `ENC_MIXED_KEYS`, and any other key type is `ENC_KEY_TYPE` (CR-4 already requires one key type). The encoder sorts:
- **`int` keys** ascending by signed numeric value (`-5, 3, 10`).
- **String-family keys** ascending by code point of `String(key)`, which equals UTF-8 byte order (P12). `StringName` keys are converted to `String` before the sort; the `StringName` comparison is never used.
- After sorting, two keys that are equal as text would be `ENC_DUP_KEY`. The check is kept for defence in depth (a hand-built key list in a test) but **cannot trigger from a `Dictionary`**: the engine treats a `String` and a `StringName` key of equal text as one key (P22), so the "never both" shape rule of § 3.6 is vacuous for dictionaries and no `ENC_DUP_KEY` fixture can be built from one. § 6 tests instead that a payload built with `String` keys and the same payload built with `StringName` keys encode identically.

The reason for the change: the bytewise order of the *encoded* integer key puts `3, 10` before `-5` and orders strings by length first, because the encoded key begins with a little-endian length. That order is deterministic but meaningless, and the diff the G4 test prints ("the first differing `block_id` and key") reads naturally only in value order. Value order is also what `Array.sort()` gives with no custom comparator, so the encoder needs no byte-comparison helper (GDScript cannot order two `PackedByteArray`s). Both orders are equally injective, so the change costs nothing in collision safety, and no hash exists yet. `gdd_sync` item for the kernel GDD D.12 and KRN-HASH-01.

**3.4 Normalisation.** `FLOAT` and the components of `VECTOR2`: `f != f` (any NaN, whatever its payload) encodes as the bits `0x7FF8000000000000`; `f == 0.0` (both zeros) encodes as `0x0000000000000000`; everything else as its binary64 bits. These need explicit branches because `encode_double` preserves NaN payloads and the sign of zero (P11). `PackedFloat32Array`, `PackedFloat64Array` and `PackedVector2Array` are **not** normalised by the encoder: they reach the buffer as raw bytes, and a NaN payload or a negative zero inside one survives (P11). Therefore owners keep NaN and `-0.0` out of packed float state (their mutators normalise), and in strict mode (§ 3.5) the encoder scans every packed float array for a NaN or a negative-zero bit pattern and fails with `ENC_FLOAT_NONCANONICAL`. Floats in SIM blocks are in any case outside the contract (§ 1.2); this rule only makes the exception safe.

**3.5 Errors and strict mode.** `encode` returns a `PackedByteArray` and sets `error` (an `ENC_*` code) and `error_path` (a short text path such as `bodies/2/pos`, built only when an error occurs, never on the success path). On error the returned array is empty and the caller treats the block as unhashable: `block_hash` returns `ok = false` with `BLOCK_HASH_ENCODE_ERROR`; inside a capture it is the existing `CAPTURE_ERROR` fault (ADR-0003 domain 0, code 7) with `error` and `error_path` in the detail. No new domain-0 fault code is added by this ADR. Codes: `ENC_OK` 0, `ENC_UNSUPPORTED_TYPE` 1, `ENC_MIXED_KEYS` 2, `ENC_KEY_TYPE` 3, `ENC_DUP_KEY` 4 (defence only, § 3.3), `ENC_DEPTH` 5 (nesting deeper than `ENC_MAX_DEPTH` 16, which also stops a cyclic container), `ENC_SIZE` 6 (output above `ENC_MAX_BYTES`, default 16 MiB, never below Save/Load's `MAX_BLOCK_BYTES`), `ENC_FLOAT_IN_BLOCK` 7 (strict only), `ENC_FLOAT_NONCANONICAL` 8 (strict only). **Strict mode** is on in tests and `STRICT_DEBUG` builds and for content hashes (`Hash64.of_value`); it additionally rejects any `FLOAT` / `VECTOR2` / packed float in a block not on the float allow-list (§ 1.2). Release builds run the non-strict encoder on the capture and K-4 paths: correctness there rests on the owners' mutators and the tests, and a payload float would still encode canonically (normalised) rather than silently.

**3.6 Shape rules for payloads** (so the encoder stays cheap). A block of more than about 250 scalar fields snapshots as packed arrays, not nested dictionaries (D.11's per-block budget). Payload dictionaries use `StringName` or `String` keys (or `int` keys for per-id tables), never both (the engine merges the two string forms into one key anyway, P22). Derived caches, diagnostics counters (`graph_*`, `hash_invocations`, `horror.view.debug`) and anything marked non-persisted are not in the payload and never in a hash (TR-lsg-039, TR-horror-020). Entity references are stable ids, never array positions of a different block. The encoder walks the payload once; it allocates one scratch `PackedByteArray` that it reuses across calls, one `keys()` array per dictionary node, and the final exactly-sized result; that allocation belongs to the dirty path and is bounded by the number of dirty blocks (S0-e measures it). **Scratch and aliasing rules (P7).** `encode` returns a fresh, exactly-sized array (`slice` or `duplicate`); the reused scratch buffer is never returned, stored or handed to another thread. `PackedByteArray.encode_s64`, `encode_u32` and `encode_double` raise an error rather than growing the buffer when the offset is out of range, so the scratch is preallocated (or `resize`d) to the bound before each walk; `HashingContext.update` takes no length argument, so the scratch is `resize`d or `slice`d to its true length before every `update` (S0-e records whether shrinking keeps capacity: UNVERIFIED). Any array handed to the Save/Load worker (§ 5.6 ladder step 2) is a `duplicate()` owned by the job.

**3.7 Worked vectors** (the first four rows follow D.12's hand-derived examples; every row was produced by the GDScript probe, and the independent Python implementation reproduced the `INT 1`, `FLOAT 1.0` and `[[1],[2,3]]` rows, so the rest are still to be cross-checked in S0; `u64(n)` is 8 little-endian bytes and `s64(n)` is the same for a signed value; the `H64` column is the numeric value of the first 8 digest bytes read little-endian).

| Value | Canonical bytes | `H64` |
|---|---|---|
| `INT 1` | `02 u64(1)` | `0xA8514B868B9B6F67` |
| `FLOAT 1.0` | `03 00 00 00 00 00 00 F0 3F` | `0xF9257072CFA72232` |
| `[[1],[2,3]]` | `07 u64(2) 07 u64(1) 02 u64(1) 07 u64(2) 02 u64(2) 02 u64(3)` | `0xE5C1840AAE2E4EC0` |
| `[[1,2],[3]]` | `07 u64(2) 07 u64(2) 02 u64(1) 02 u64(2) 07 u64(1) 02 u64(3)` | `0xA09749F81F3D6976` |
| `&"a"` and `"a"` | `04 u64(1) 61` (equal) | `0x33A10EBD15E04FB8` |
| `{"b": 1, "a": 2}` and `{"a": 2, "b": 1}` | `08 u64(2) 04 u64(1) 61 02 u64(2) 04 u64(1) 62 02 u64(1)` (equal) | `0xE39AE16B84B9F022` |
| `{10: 1, -5: 2, 3: 3}` | `08 u64(3) 02 s64(-5) 02 u64(2) 02 u64(3) 02 u64(3) 02 u64(10) 02 u64(1)` | `0x3994B5446692B06D` |
| `PackedInt32Array([1, -2])` | `09 02 u64(2) 01 00 00 00 FE FF FF FF` | `0xA02103D5B9B9D438` |
| `Vector2(-0.0, 1.0)` | `05` then 8 zero bytes then `00 00 00 00 00 00 F0 3F` | `0x3B288246B7BCE33A` |
| `Vector2i(-1, 2)` | `06 FF FF FF FF FF FF FF FF u64(2)` | `0xBA1BD293E871B0D2` |
| `null` / `true` | `00` / `01 01` | `0x987AB3FF9C0B346E` / `0x2326F384A197CF9D` |

The two nested arrays differ at the second count, so they cannot collide; `INT 1` and `FLOAT 1.0` differ at the tag. A property test over 10,000 seeded random values asserts `encode(a) == encode(b)` only when `a` and `b` are equal after normalisation (KRN-HASH-01).

**3.8 Writing the encoder inside the ADR-0010 gates.** ADR-0010 § 5 sets every warning except `RETURN_VALUE_DISCARDED` to error and lints `float`, `Variant`, bare `Array`, `float(` / `int(` conversions and `Vector2(` in `src/shared` and `src/kernel`. The encoder unpacks `Variant`s by nature, so the rules below are stated here and mirrored by ADR-0010 (cross-ADR requirements under Related).
- **(a) Allow-lists.** `src/shared/det/canonical_encoder.gd` is the second file on `float_allowlist.json` for `src/shared` (the first is `det_rng_float.gd`) and an entry on `dictionary_variant.json`, with the reason "the single site that unpacks `Variant` payloads and normalises floats for hashing". Float handling is confined to one private function `_encode_float` (NaN and zero normalisation, `VECTOR2` widening, `encode_double`) plus the strict-mode packed-float scan; the file holds no other float. The `Variant`-typed parameters of `Hash64.of_value` (a pass-through) and of the kernel's `canonical_encode` forwarder need the same treatment; ADR-0010 decides whether that is the same entry or adjacent ones (open question 13).
- **(b) Unpacking idiom.** `match typeof(value)`, then a typed assignment (`var d: Dictionary = value`, `var i: int = value`), **never `as`**: `UNSAFE_CAST` at error rejects `value as Dictionary` on a `Variant`, while the typed assignment loads cleanly (P20). Dictionary keys go into typed arrays: `var ks: Array[int] = []; ks.assign(d.keys())` for an all-int dictionary, and `var ss: Array[String] = []; ss.assign(d.keys())` for the string family (a `StringName` key becomes its `String`, P20), then `ks.sort()` or `ss.sort()`. The dictionary is classified first (`typeof` over its keys), so `assign` never sees a mix; the bare `Array` that `Dictionary.keys()` returns is consumed only by `assign`, so no `Array` token appears in source. The public helpers are `CanonicalEncoder.sorted_int_keys(d)` and `CanonicalEncoder.sorted_string_keys(d)`.
- **(c) No conversion calls in `unit` and `int_in`.** § 2.1 uses int-to-float promotion in binary operators, so the `SIM-04` `float(` / `int(` ban needs no exemption beyond the file allow-list; P20 shows no warning fires.
- **(d) Fallback.** If any of this cannot be made to load in LINT-SPIKE-1, the fallback is `@warning_ignore("unsafe_cast")` on the single site, counted like `SIM-05`.

### 4. `H64`, `block_hash` and `state_hash`

**4.1 `H64`.** `H64(bytes)` is the first 8 bytes of `SHA-256(bytes)`, read as a little-endian two's-complement `int` (`digest.decode_s64(0)`, P13). It is computed by `HashingContext` (native; P14: 0.33 ms for 150 KB on the dev machine), through one reusable context held by the kernel's `StateHasher` (`start` / `update` / `finish`, P13). `hash_algo` = **2** names it (SL-16); ids 0 (xxHash64) and 1 (FNV-1a 64) are reserved and not written. **A scripted per-byte 64-bit hash is banned** for state hashing (D.11, TR-section-032): SL estimates 15 to 150 ms over 90 KiB (UNVERIFIED) against 0.33 ms for the native path, and the built-in `hash()` is not a stable cross-version contract. Reference: `H64("abc")` is `0xEACF018FBF1678BA`. `HashingContext` is permitted only in `src/shared/det/hash64.gd` and `src/kernel/state_hasher.gd` for state and content hashing (and in `src/save/` and `src/data/` for container integrity and content, per ADR-0010 § 4.1; the list is ADR-0010's and mirrored here). **No `H64` value is reserved: 0 is a legal hash.** An absent or no-verdict state is a separate flag, never a sentinel value (ADR-0002's restore signature carries `has_expected_hash: bool`, and the K-4 compare is skipped when it is false).

**4.2 Two representations, never mixed.** The numeric `int` of a hash prints (for logs and the Overlay) as `0x` plus 16 hex digits **most significant first**. The `hex64` string used in headers and JSON (S-1, SL F-10) is the **8 little-endian bytes** of the same `int` as 16 lowercase hex characters, which for an `H64` is exactly the first 16 hex characters of the SHA-256 digest. So `H64("abc")` is numeric `0xEACF018FBF1678BA` and `hex64` `"ba7816bf8f01cfea"`; `world_seed = -1` is `"ffffffffffffffff"`; `2^53 + 1` is `"0100000000002000"`; `2^63` (the `int` `-9223372036854775808`) is `"0000000000000080"`. `Hex64.encode(x)` builds the string from `encode_s64` and `hex_encode()`; `Hex64.decode(s)` returns a `Hex64Result` (`ok`, `value`) and first validates exactly 16 characters of `[0-9a-f]` (P16: `hex_decode` alone accepts uppercase and reports errors by logging) and then `decode_s64`; neither uses `String.hex_to_int` or a general integer parse. ADR-0006 calls these two functions instead of keeping its own codec; its 128-bit `campaign_id` keeps a local helper with the same exact-width, lowercase pre-check.

**4.3 `block_hash`.** `block_hash(b) = H64(canonical_encode(snapshot payload of b))` with no framing, so it equals the value Save/Load stores as `block_state_hash` (SL-16) and the value Layout reads as `layout_hash()`.

**4.4 `state_hash`** (resolves the fold half of OQ-8). Every length, count and scalar in the fold is 8 bytes little-endian (`u64` / `s64`), and `block_hash` is the 8 digest bytes as `H64` produced them (equivalently `encode_s64` of the `int`). The fold is hashed as a stream, never built as one big array:

```
state_hash(scope, T, uncached) = H64( s64(T) || s64(world_seed) || record(b1) || record(b2) || ... )     blocks sorted ascending by block_id
record(b) = u64(len(block_id)) || utf8(block_id) || u64(schema_version) || u64(len(canonical_bytes(b))) || s64(block_hash(b))
```

`block_id` order is ascending code point of `String(block_id)` (P12); `len(block_id)` is its UTF-8 byte length. An empty scope hashes `s64(T) || s64(world_seed)`. Because every variable-length part is length-prefixed and every other part is fixed width, the concatenation is injective. `restore_epoch` is not an input (§ 2.4), and neither is `correlation`, a diagnostics counter or the command log. **Scopes** (D.12): `PERSIST` = blocks with `persist = true` **and** `persist_scope = WORLD`; `ALL_SIM` = every `SIM` block (WORLD, PROFILE and non-persisted derived); both exclude `kernel.clock` (its `T` and `world_seed` are the prefix, counted once) and never include `PERCEPTION_OUT`. `hash_algo` 2 names **the whole definition in this section and § 3**: the tag set, the key order, the normalisation, the fold framing and the digest truncation. Any change to any of them is a new `hash_algo` value, not a silent edit; a save with an unsupported `hash_algo` loads REBASED (SL-12) rather than comparing.

**4.5 Worked fold** (probed and reproduced in Python): `T = 5000`, `world_seed = 0x5EED000000002026`, block `a.state` (`schema_version` 1, payload `{"x": 1}`, bytes `08 u64(1) 04 u64(1) 78 02 u64(1)`, 26 bytes, `block_hash` `0x85A68464E47356C1`) and block `b.state` (`schema_version` 2, payload `[1, 2]`, bytes `07 u64(2) 02 u64(1) 02 u64(2)`, 27 bytes, `block_hash` `0xF1B6635FE3BC2290`). The fold input is 16 + (8 + 7 + 8 + 8 + 8) + (8 + 7 + 8 + 8 + 8) = 94 bytes, and `state_hash(PERSIST, 5000)` = **`0x81AB3842EF1E8DEF`**. These, plus the `"abc"` vector and the § 2.4 vectors, are the frozen goldens of KRN-HASH-01..03; the Ship Body 518-byte fixture is added when the reference encoder confirms or corrects its length.

**4.6 Content hashes** (inputs fixed by ADR-0007 § 3.4 and § 3.5). `Hash64.of_value(value)` = `H64(CanonicalEncoder.encode(value, strict = true))`. `catalogue_hash` = `of_value(rows)` where `rows` is an `Array` of per-def `Array`s of post-cast integers and `def_id` strings, sorted ascending by `def_id` in code-point order (equal to the byte order ADR-0007 requires). `template_hash`, `geometry_hash` and `visual_hash` = `of_value` of the post-cast template payload (or its geometry / visual sub-payload, whose shape is Layout's and ADR-0013's), computed once at load. Layout stores the `u32` form `Hash64.low32(h)` in a `PackedInt32Array` and compares only after `& 0xFFFFFFFF` (P9, TR-section-032). Content hashes are diagnostic and never gate a load (ADR-0007 § 3.4). Their golden vectors are committed after this ADR is Accepted (ADR-0007 Validation).

**4.7 What the kernel hashes and when.** No hash runs in P2 or P3, and none runs on a tick where no consumer asks (§ 5.4). The consumers are: the Overlay cadence (`DIAG_HASH_CADENCE_TICKS` 60, outside P3, cached profile); `layout_hash()` and Horror's path cache (`block_hash` read from P3 onward or a boundary, memoised by the caller on its own mutation counter); the snapshot capture (`SAVING`, uncached, one `snapshot` per block whose payload feeds both the canonical encode and Save/Load's `var_to_bytes`); the K-4 reproduction compare after `post_restore` (uncached); the `STRICT_DEBUG` tripwire (D.12: at most two whole-set uncached hashes per dispatch batch plus one around P5); and the tests. **Capture hand-off.** `request_snapshot()` returns `void` and the outcome arrives through the outcome handler (ADR-0002); the snapshot `ControlOutcome` carries `T`, `world_seed`, `state_hash`, `hash_algo`, `block_ids`, `schema_versions`, `block_state_hashes` and `enc_lens` for every persist block (WORLD and PROFILE; PROFILE blocks are hashed per block though excluded from PERSIST), all computed uncached in `SAVING`. ADR-0006 § 1.8 triggers PROFILE writes by comparing those uncached `block_state_hashes`.

**4.8 64-bit interchange** (TR-cmd-038, shared with ADR-0006). A value that can exceed `2^53` (`T`, `world_seed`, `state_hash`, every block hash, `campaign_id`) crosses a JSON boundary only as `hex64`; a `Marshalls.raw_to_base64` string is allowed for opaque byte blobs. A snapshot's in-memory round trip through `var_to_bytes` / `bytes_to_var` is lossless for `int64` (the binary form carries 8 bytes), and that is Save/Load's storage form; it is never hash input (registry `payload_size_check`). The Web Map's 8-byte seeds are `PackedByteArray` bytes in their block and so need no integer treatment (WM-37). **Every JSON-sourced number is a `float` on parse (P21)**, so even small ints arrive as floats and `str()` can print `5.0`; above `2^53` the parse loses bits silently. Header and content readers therefore cast through the ADR-0007 `IntField.read` / the `Hex64.decode` path only, reject non-integral or `> 2^53` values, and never pass a parsed `Variant` into a block; a boundary that skipped the cast would hand the encoder a `FLOAT` (`ENC_FLOAT_IN_BLOCK` in strict mode). ADR-0006's header decode is the first consumer.

### 5. Cached block hashes (K-1)

**5.1 What is cached.** The kernel holds, per registered block, `{hash: int, enc_len: int, serial: int}` in three parallel `PackedInt64Array`s sized once at the registry freeze (serial `-1` = never hashed). A block is addressed by its **`block_index`**: the dense index assigned at the registry freeze in ascending `block_id` code-point order, never persisted and never hashed. `open_for` takes the `block_id` and the kernel resolves it through the freeze-time id-to-index map; `has_opened`, `touch_serial` and the cache arrays take the `block_index`. The arrays live in the kernel's registries group (ADR-0003 § 10, H1, which gains the hash-cache arrays and `touch_serial`) and the counters in its diagnostics group; `BlockHashCache` (Key Interfaces) wraps them. The cache stores the length too, so `state_hash` can fold a cached block without re-encoding it. An entry is **valid** iff its `serial` equals `touch_serial[block]`, the always-on counter ADR-0004 § 7 keeps (`Kernel.touch_serial(block_index)`).

**5.2 Invalidation by the touch serial, not by a dirty key.** ADR-0004 advances `touch_serial[block]` on every granted `scope.open_for(block)` (in `APPLY`, `HOOK`, `PAUSED_APPLY`, `INIT` and `RESTORE` scopes alike). Belt and braces: `restore`, `init_world` and `default_state` do not all open a `WriteScope` the same way (ADR-0003 § 3: `init_world` gets a `WriteScope` (INIT) and a `SnapshotContext`, and `restore` and `post_restore` may write through a RESTORE scope), so the kernel resets the cache entry itself rather than relying on the owner's scope usage; it resets it (`serial = -1`) for every block it replaces on those three calls, independent of the serial. That is a strictly sound "this block may have changed" signal, because every mutator begins with `open_for` (ADR-0003), and it covers the case a dirty key cannot: a quantised write (the Graph's sub-bucket changes, Web Map's marks-nothing leg ticks) changes the hash and marks no key (ADR-0004 § 7, `quiet_writes_allowed`). This ADR therefore adopts the serial in place of the kernel GDD D.12's "invalidated by the block's dirty bit" (and TR-cmd-040's "only by `mark_dirty`"); the cost is that a scope that opens a block and changes nothing also invalidates it, which costs one extra encode at the next cached read and nothing on a tick with no reader. A write outside any scope (no `open_for`, so no serial change) leaves the cache stale; that is the existing `SIM_WRITE_TRIPWIRE` case, is caught by every uncached verification path (KRN-HASH-02, re-targeted to a fixture that writes without a scope) and by the detector in § 5.5. A property setter never invalidates anything (4.7: a packed-array element write skips the setter, P8).

**5.3 Reads.** `block_hash(block_id, uncached = true) -> BlockHashResult` (D.12) is available in every build. It is a pure read: the kernel calls the owner's `snapshot` with the kernel-internal `SnapshotContext` reserved for hash reads outside `SAVING` (a second instance beside the one capture uses, with a re-entrancy flag; ADR-0003 adds it), encodes, hashes, and mutates no block. Order of checks: (1) unknown id or a non-`persist` block: `ok = false`, `BLOCK_HASH_UNKNOWN`, never a hash of 0; (2) the active `WriteScope` (one whose phase is not CLOSED) has opened the block (`scope.has_opened(block_index)`, true from the first `open_for` of that block until the scope is re-armed): `ok = false`, `BLOCK_HASH_IN_WRITE`; (3) `uncached = false` and `valid`: return the cache; (4) otherwise `snapshot`, encode, `H64`, set `ok`; an encoder error gives `BLOCK_HASH_ENCODE_ERROR`. Only a **cached** read stores into the cache (it records the current `touch_serial` with the hash, and counts one encode if it had to compute); an **uncached** read ignores the cache, writes nothing, and counts exactly one `canonical_bytes` encode. A hook that has written Layout and then reads `layout_hash()` inside the same hook therefore gets `BLOCK_HASH_IN_WRITE`: read it from another hook or a boundary, or after the writing hook returns. `snapshot` is not re-entrant (ADR-0003), so `block_hash` called from inside a `snapshot` is a defect that the context's re-entrancy flag reports. **Internal read `block_hash_any`.** In profiles that run `SIM_WRITE_TRIPWIRE` (tests and `STRICT_DEBUG`), the kernel exposes an internal `block_hash_any(block_index: int) -> int`: uncached, on the same path `ALL_SIM` uses, skipping check (2) and the `persist == false` `BLOCK_HASH_UNKNOWN` rule. Only the tripwire and ADR-0004's `MISSED_DIRTY_MARK` audit call it, so they can hash every touched block, derived ones included, after P4; the published `block_hash` keeps KRN-HASH-03 unchanged. ADR-0004 § 7 cites it and gates the audit on the profile that runs `SIM_WRITE_TRIPWIRE`, not on build names; whether playtest builds run that profile is UNVERIFIED.

**5.4 Which path uses the cache.** Only the **diagnostics cadence** (the Overlay's `state_hash` display and its HASH_DRIFT block naming) and a caller memoising its own reads may use `uncached = false`. **Every determinism check uses `uncached = true`**: G3, G4, the K-4 window, the cross-process test, the `STRICT_DEBUG` tripwire, the snapshot capture, the weld property test and the golden comparisons. The reason is the same as D.12's: a cached comparison would pass falsely on the missed-dirty-mark bug. Capture is always uncached, and a dirty key can never gate caching (ADR-0004 § 7). ADR-0006's rung 1 (the kernel skipping `snapshot()` for a block whose cache entry is valid by touch serial since the last capture) stays a deferred optimisation, **off by default**: enabling it needs a technical-director ruling on the SAVE-WRK-1 measurement and a matching amendment to this section, and `BlockHashCache.entry_valid` exists for it.

**5.5 Stale-cache detector (hash side of ADR-0004's audit).** In `STRICT_DEBUG` and test builds, each diagnostics-cadence refresh recomputes `uncached` for every block whose cache is valid and, if the two differ, counts `HASH_CACHE_STALE`, reports a violation naming the block, and invalidates the entry. With the serial rule this fires only for a write outside a `WriteScope` or a leaked mutable reference into a block, which is the failure `SIM_WRITE_TRIPWIRE` hunts; it is cheap corroboration, not a replacement for ADR-0004's `MISSED_DIRTY_MARK` audit (which concerns presentation keys). It costs one extra encode per valid block per cadence in debug only. Release builds do not run it.

**5.6 Cost accounting.** Counters (diagnostics, never hashed, reset at BOOTING and at restore): `enc_blocks` (encodes performed), `enc_bytes`, `enc_elem_loops` (element-loop iterations over non-packed scalars), `hash_calls`, `cache_hits`, `HASH_CACHE_STALE`. KRN-PERF-01: a quiet tick with no dirty block and no reader shows `enc_blocks` = 0. KRN-PERF-02: a cached-profile read after a tick that touched `k` blocks (opened them for writing) shows `enc_blocks` = `k`, the blocks whose touch serial moved since the previous cached read; a block that was written has been opened, so `k` is the count of written blocks plus any opened and left unchanged. KRN-PERF-05: packed arrays reach the buffer by `to_byte_array()` and contribute 0 to `enc_elem_loops`. `HashingContext` over 150 KB must stay at or below 1 ms on RM-DECK (TR-lsg-037, S0-d); if a release-export measurement exceeds it, the ladder is: keep the cache (already), then hash on the Save/Load worker from the encoded bytes (the worker may call `HashingContext`, on a `duplicate()` of the bytes owned by the job; the encoder's scratch buffer is never handed to a worker, because the next main-thread encode would overwrite bytes the worker is hashing, a data race with silent hash drift, P7), then split large packed arrays into per-section sub-payloads whose hashes fold, under a new `hash_algo`.

### 6. Verification and replay tests

The determinism claims are tested, not asserted:
- **G3:** two independent kernels, the fixture blocks of the Determinism acceptance criteria, `world_seed` fixed, a recorded 1,000-tick stream: `state_hash(ALL_SIM, 1000, uncached = true)` equal; plus the cross-process variant through `OS.execute` with `(save bytes, restore_epoch)` passed on the command line.
- **G3 worker variant:** the same G3, with Save/Load worker hashing of a captured payload and the main-thread encode of the next tick running in the same tick, asserting equal hashes (the scratch buffer is never shared with the worker, § 3.6).
- **G4:** snapshot at `T = 500` at a boundary with no `PLAYER` command pending, restore with the same pinned epoch, continue to 1,100: `state_hash(PERSIST, 1100)` equal to the control; on a mismatch the diff names the first differing `block_id` and key over the uncached TLV bytes.
- **Render-rate independence:** the same 10-second scripted replay at `max_fps` 30, 60, 144 and 0 gives the same hash (TR-pxl-022, ADR-0001).
- **Encoder:** the KRN-HASH-01 property test (injectivity under normalisation), the § 3.7 vectors, the § 4.5 fold, KRN-HASH-03 (`block_hash` read semantics and counters), and a strict-mode rejection test for each reachable `ENC_*` code (`ENC_DUP_KEY` is unreachable from a `Dictionary`, P22: in its place, a test that a payload built with `String` keys and the same payload built with `StringName` keys encode identically).
- **RNG:** the § 2.4 vectors, the epoch triple (KRN-EPO-01: the three epoch values differ), a facet misuse test (`draw` on an epoch-keyed stream), a stream-registration collision test, and a 1,000,000-draw comparison of `int_in` against the arbitrary-precision oracle.
- **Cross-OS vector job** (CI, Windows and Linux; macOS advisory until a target decision): the § 2.4, § 2.6, § 3.7 and § 4.5 vectors byte-for-byte equal, plus the S0-a to S0-d scratch scripts on a `template_release` export. The Graph golden files (`tests/golden/graph_state_hash_f_mvp_healthy.txt`, `graph_state_hash_weld_cut.txt` and the thaw-roll draw sequence, TR-lsg-031) belong to the Graph epic's Validation Criteria, because they need Graph code that this ADR's Acceptance must not wait for.
- **Lints:** the § 1.7 table, each rule with a committed bad and good fixture file, including `x /= y`, `x %= y`, `9223372036854775808` and `0x8000000000000000` as bad fixtures. **LINT-SPIKE-1 (first story):** a load-all check of every Key Interfaces snippet under the ADR-0010 § 5.1 warning configuration, which also settles the § 3.8 idioms.

### Key Interfaces

```gdscript
# Paths follow ADR-0010 (Proposed). All pure determinism code is in src/shared/det/ (DetRng, DetRngFloat, Ihash, Hex64,
# Hex64Result, Hash64, CanonicalEncoder), except IntMath, which stays at src/shared/int_math.gd; this ADR is the sole
# owner of the IntMath function list. The instance classes (RngFacet, RngRegistry, StateHasher, BlockHashCache) are in
# src/kernel/. All are scene-tree free. Every identifier below avoids the GDScript utility-function names (seed, hash,
# min, max, ...) and sibling members, because SHADOWED_* is an error under ADR-0010 5.1 (P20).

class_name IntMath extends RefCounted                 # src/shared/int_math.gd; constants and statics only; no state
static func trunc_div(a: int, b: int) -> int          # a / b: the engine's truncation; the one annotated `/` in src/ (ADR-0010 SIM-05)
static func lsr(x: int, n: int) -> int                # logical shift right, 1 <= n <= 63
static func floor_div(a: int, b: int) -> int          # trunc_div(a - posmod(a, b), b); b > 0, any sign of a; b <= 0 returns 0 (defect)
static func ceil_div(a: int, b: int) -> int           # floor_div(a + b - 1, b); b > 0, any sign of a
static func posmod_i(a: int, b: int) -> int           # the posmod builtin; b > 0, result in [0, b - 1]
static func low32(x: int) -> int                      # x & 0xFFFFFFFF

class_name DetRng extends RefCounted                  # § 2.1 constants and statics; no state
static func mix64(z: int) -> int
static func fnv64(name: String) -> int
static func stream_key(name: String) -> int           # mix64(fnv64(name))
static func derive(world_seed: int, key: int, counter: int) -> int   # 'key' is a stream_key; never named after the sibling function

class_name DetRngFloat extends RefCounted             # det_rng_float.gd: one of the two float-allow-listed files in src/shared (§ 2.5)
static func unit(raw: int) -> float                   # lsr(raw, 11) / 9007199254740992.0 (promotion, no float( call)
static func int_in(raw: int, lo: int, hi: int) -> int # lo + floori(unit(raw) * (hi - lo + 1)); caller guarantees lo <= hi and span <= 2^32

class_name Ihash extends RefCounted                   # § 2.6
static func h(variation_seed: int, k1: int, k2: int, k3: int, salt: int) -> int     # non-negative 63-bit
static func h32(variation_seed: int, k1: int, k2: int, k3: int, salt: int) -> int  # h(...) & 0xFFFFFFFF

class_name RngStream extends RefCounted               # § 2.2; read-only after registration
var name: StringName
var key: int
var epoch_keyed: bool

class_name RngRegistry extends RefCounted             # src/kernel/rng_registry.gd; kernel-private
func register(name: StringName, epoch_keyed: bool) -> RngStream   # BOOTING only; DUPLICATE_ID on a duplicate name or key
func dynamic(name: String, epoch_keyed: bool) -> RngStream        # memoised, bounded by MAX_DYNAMIC_STREAMS

class_name RngFacet extends RefCounted                # § 2.3; issued as ModuleFacet.rng, null for a module with no streams
func register_stream(name: StringName, epoch_keyed: bool = false) -> RngStream   # BOOTING only
func dynamic_stream(name: String, epoch_keyed: bool = false) -> RngStream
func draw(stream: RngStream, counter: int) -> int
func draw_epoch(stream: RngStream, rng_counter: int) -> int
func int_in(raw: int, lo: int, hi: int) -> int       # calls DetRngFloat.int_in; counts RNG_RANGE_VIOLATION

class_name CanonicalEncoder extends RefCounted        # § 3; canonical_encoder.gd (second float-allow-listed and Variant-allow-listed file); one instance per thread, reuses its scratch buffer
var error: int                                        # ENC_* of the last encode
var error_path: String                                # only set on error
func encode(value: Variant, strict: bool = false) -> PackedByteArray   # fresh exactly-sized array; empty on error
static func sorted_int_keys(d: Dictionary) -> Array[int]               # caller has checked all-int keys; ascending
static func sorted_string_keys(d: Dictionary) -> Array[String]         # String and StringName keys as String; ascending by code point
# private: _encode_float(...) is the only float handling (NaN and zero normalisation, VECTOR2 widening)

class_name Hash64 extends RefCounted                  # hash64.gd; with state_hasher.gd, the only det files that use HashingContext
const HASH_ALGO: int = 2
static func of_bytes(bytes: PackedByteArray) -> int   # allocates a HashingContext; not for hot paths
static func of_value(value: Variant) -> int           # strict encode + of_bytes; content hashes, tests (Variant pass-through, § 3.8 a)
static func low32(h: int) -> int

class_name Hex64Result extends RefCounted
var ok: bool
var value: int

class_name Hex64 extends RefCounted                   # § 4.2
static func encode(x: int) -> String                  # 16 lowercase hex chars, little-endian bytes
static func decode(s: String) -> Hex64Result          # ok false on wrong width, uppercase or non-hex

class_name BlockHashResult extends RefCounted         # one reused instance per kernel; read immediately
const NONE: int = 0
const BLOCK_HASH_IN_WRITE: int = 1
const BLOCK_HASH_UNKNOWN: int = 2
const BLOCK_HASH_ENCODE_ERROR: int = 3
var ok: bool
var h64: int                                          # not 'hash': SHADOWED_VARIABLE
var reason: int

class_name BlockHashCache extends RefCounted          # src/kernel/block_hash_cache.gd; three PackedInt64Arrays sized once at the registry freeze (§ 5.1)
func setup(block_count: int) -> void
func entry_valid(block_index: int, current_serial: int) -> bool    # serial == Kernel.touch_serial(block_index); also the hook for ADR-0006 rung 1
func cached_hash(block_index: int) -> int
func cached_len(block_index: int) -> int
func store(block_index: int, h64: int, enc_len: int, serial: int) -> void   # cached reads only
func reset(block_index: int) -> void                  # serial = -1: restore, init_world, default_state

class_name StateHasher extends RefCounted             # src/kernel/state_hasher.gd; holds the one reusable HashingContext
func block_hash(block_id: StringName, uncached: bool = true) -> BlockHashResult
func block_hash_any(block_index: int) -> int          # internal; SIM_WRITE_TRIPWIRE profiles only; uncached; tripwire and ADR-0004 audit only
func state_hash(scope: int, T: int, uncached: bool = true) -> int      # scope: PERSIST (G4) or ALL_SIM (G3)
func canonical_encode(value: Variant) -> PackedByteArray                # pure; the same function the hashes use (Variant pass-through, § 3.8 a)

# Published by the kernel (D.12), forwarding to StateHasher; available in every build:
#   block_hash(block_id: StringName, uncached: bool = true) -> BlockHashResult
#   state_hash(scope: int, T: int, uncached: bool = true) -> int
#   canonical_encode(value: Variant) -> PackedByteArray
```

Cross-ADR amendments this interface set assumes (also under Related): `ModuleFacet.rng: RngFacet`, `WriteScope.has_opened(block_index: int) -> bool`, the `kernel.clock` view `(T, world_seed, restore_epoch)`, the hash-cache arrays and a second kernel-internal `SnapshotContext` (ADR-0003); `Kernel.touch_serial(block_index: int) -> int` (ADR-0004 § 7, already specified there); the snapshot `ControlOutcome` fields and `has_expected_hash` (ADR-0002); `IntMath.ceil_div`, `lsr`, `low32` and the `b <= 0` guard, which ADR-0010 § 5.4 now cites rather than restates; `det_rng_float.gd` and `canonical_encoder.gd` on `float_allowlist.json` and `canonical_encoder.gd` on `dictionary_variant.json` (ADR-0010 § 5.3 and § 5.5).

### Implementation Guidelines

- **Must** write every simulation quantity as an `int`; **must never** declare a `float` in `src/sim/**` or store one in a persisted SIM block, except on the committed allow-list.
- **Must** write every 64-bit constant as a signed decimal `int` literal; **must never** write a hexadecimal literal above `0x7FFFFFFFFFFFFFFF` or a decimal literal greater than 9223372036854775807 in magnitude (write INT64_MIN as `-9223372036854775807 - 1` or `1 << 63`), and **must never** shift a negative literal.
- **Must** use `IntMath.lsr` for any right shift of a possibly negative value; **must never** call it with `n` outside `1..63`.
- **Must** divide and take modulus only through `IntMath` (`trunc_div`, `floor_div`, `ceil_div`, `posmod_i`) with `b > 0` for the floor family, using `trunc_div` only where truncation toward zero is meant and says so; **must never** write `/` on two ints, `%`, `/=` or `%=` outside `IntMath`, or rely on truncation for a quotient meant as a floor.
- **Must** keep every sim intermediate below `2^62` in magnitude and check declared factor maxima at boot; **must never** rely on integer wrap outside `mix64`, `fnv64`, `Ihash` and the fold.
- **Must** iterate dense arrays in ascending id order, and extract and sort `Dictionary` keys before iterating one; **must never** iterate an unsorted `Dictionary`, sort `StringName`s, use `sort_custom` or any sort over tie-able keys, or call `hash()`, `get_instance_id()`, `shuffle()` or `pick_random()`.
- **Must** draw randomness only through `ModuleFacet.rng`, with one registered stream per independent sequence and its counter inside the owner's persisted block; **must never** call `randi`, `randf`, `RandomNumberGenerator` or any global generator, and **must never** register a stream on a module whose GDD says it draws nothing.
- **Must** call `draw_epoch` for `attention.tell` and `horror.tell` and `draw` for every other stream; **must never** compute `rng_counter + RESTORE_KEY_STRIDE * restore_epoch` outside `RngFacet`, store `restore_epoch` in a block, or hash it.
- **Must** compute a stream's key once at registration; **must never** call `fnv64` or `stream_key` on a draw path.
- **Must** use `Ihash` for presentation variation and non-RNG sim variation, with a fixed non-zero seed per system; **must never** feed it `tick_alpha`, `present_time` or a frame count, and **must never** call `DetRng.derive` from presentation. A truth-channel visual that must be reproducible (the Trail) uses `Ihash` keyed by `world_seed` and a fixed per-system seed.
- **Must** deliver a state block's canonical state through `snapshot(ctx)` as a plain `Dictionary` / `Array` / packed-array payload and let the kernel encode it; **must never** write an owner-side byte layout, call `encode_s64` / `encode_double` / `to_byte_array` for hashing in `src/kernel/`, `src/sim/` or a block owner (they are allowed in `src/shared/det/`, in tests and, for container layout and integrity hashes only, in `src/save/`), or feed `var_to_bytes` output to a state hash.
- **Must** keep NaN and `-0.0` out of packed float arrays; **must never** put an `Object`, `Callable`, `RID` or `Color` in a snapshot payload.
- **Must** compute every hash through `HashingContext`, used only in `hash64.gd`, `state_hasher.gd`, `src/save/` and `src/data/`; **must never** write a scripted per-byte hash or use the built-in `hash()` for state.
- **Must** open a block with `scope.open_for` before any write to it (this advances the touch serial the hash cache depends on) and mark dirty keys through `scope.mark_dirty` where the block's key space requires it (ADR-0004); **must never** write a block outside a scope, or rely on a property setter to invalidate anything.
- **Must** call `block_hash(id)` and `state_hash` with `uncached = true` on every determinism check, capture, K-4 compare and test; **must never** use `uncached = false` for anything but diagnostics display and a caller's own memoised read.
- **Must** read `block_hash` from P3 onward (from a hook that did not write that block) or from a boundary; **must never** read it inside an `apply` or from a hook after that hook has opened the block.
- **Must** print a hash for people as `0x` plus 16 hex digits most significant first and serialise it as `hex64`; **must never** write a 64-bit value as a JSON number or parse hex with `String.hex_to_int`.
- **Must** treat a change to the tag set, key order, normalisation, fold framing or digest truncation as a new `hash_algo`; **must** bump a block's `schema_version` when its payload shape changes; **must never** edit the definition in place.
- **Must** regenerate golden files only through the reviewed golden-update job, recording the commit; **must never** hand-edit a golden value.
- **Must** name members, parameters and locals so they do not match a `@GlobalScope` or GDScript utility function (`seed`, `hash`, `min`, `max`, `floor`, `round`, `sign`, `range`, `len`, `str`, `type`) or a sibling member or function of the same class; **must never** rely on a shadowing name loading, because `SHADOWED_VARIABLE`, `SHADOWED_VARIABLE_BASE_CLASS` and `SHADOWED_GLOBAL_IDENTIFIER` are errors under ADR-0010 § 5.1 (P20).
- **Must** run a load-all check of every Key Interfaces snippet under the ADR-0010 § 5.1 warning configuration as part of the first story (LINT-SPIKE-1); **must never** merge a snippet that has not loaded.
- **Must** unpack a `Variant` with a `typeof` check and a typed assignment (`var d: Dictionary = value`), extract dictionary keys with `Array[T].assign(d.keys())`, and do so only in the allow-listed `canonical_encoder.gd`; **must never** use `as` on a `Variant` (`UNSAFE_CAST` is an error).
- **Must** make every override of a typed-return kernel callable (`snapshot`, `restore`, `default_state`, `init_world`, `_to_string`) return on all paths and re-declare the return type explicitly (4.7 typed-return inheritance); **must never** leave a path without a `return`.
- **Must** declare Vector2i and `PackedInt32Array` maxima against `2^31` and range-check in plain `int` before the store (P17); **must never** store a sum or product into one unchecked.
- **Must** treat every JSON-sourced number as a `float`, cast it through `IntField.read` / `Hex64.decode`, and reject non-integral or `> 2^53` values; **must never** pass a parsed `Variant` into a block.
- **Must** return a fresh exactly-sized array from `encode` and hand any array given to the Save/Load worker over as a `duplicate()`; **must never** return, store or share the encoder's scratch buffer.
- **Must** treat `0` as a legal `H64` and carry "no verdict" as a separate bool; **must never** use a hash value as a sentinel.
- **Must** read `block_hash_any` only from the `SIM_WRITE_TRIPWIRE` tripwire and the ADR-0004 audit; **must never** call it from gameplay, the Overlay or Save/Load.

## Alternatives Considered

### Alternative 1: Owner-written `canonical_bytes()` per block (the Graph's `[u32 tag][u32 len][bytes]`)
- **Description**: Each owner serialises its persisted fields in a documented order and publishes `canonical_bytes()`; the kernel hashes whatever it gets (LS-21, TR-lsg-021).
- **Pros**: The Graph pays no dictionary walk; the layout is fixed by the owner and visible in its GDD.
- **Cons**: 31 byte writers, each with its own normalisation of floats, strings, empty arrays and tombstones, and no shared injectivity argument; D.12 forbids it; every owner needs its own golden bytes; a cross-owner diff tool cannot name the first differing key.
- **Rejection Reason**: The cost the Graph wanted to avoid (a per-element loop) is already avoided by the `PACKED` tag, which memcpy-copies the arrays (P14). One encoder, one set of vectors, one injectivity test.

### Alternative 2: `var_to_bytes` output as the hash input
- **Description**: Hash the same bytes Save/Load stores, saving the second encoding pass at capture.
- **Pros**: Native speed; one serialisation per block.
- **Cons**: The encoder's size choices for ints and floats and its handling of NaN, `-0.0`, `StringName` versus `String`, typed containers and Dictionary insertion order are not a documented cross-platform byte contract. ADR-0006 probed 2026-10-28 that `var_to_bytes` carries `int` at full 64-bit width exactly (2^53 + 1, `PackedInt64Array`), which settles the integer size choice for storage; the float, NaN, `-0.0`, `StringName` and Dictionary-order choices are unprobed, and `-0.0` / NaN payloads would differ between equal states. Registry `payload_size_check` already records "never hash input".
- **Rejection Reason**: Equal states must have equal bytes (injectivity after normalisation); `var_to_bytes` does not promise it. The capture shares one `snapshot` payload between the two encoders, so the extra cost is one encode pass, budgeted by D.11 `t_encode`.

### Alternative 3: A scripted 64-bit hash (xxHash64, FNV-1a 64) over the canonical bytes
- **Description**: Keep the full 64 bits without truncating a longer digest.
- **Pros**: No dependence on `HashingContext`; `hash_algo` ids 0 and 1 already reserved.
- **Cons**: A per-byte GDScript loop; SL estimates 15 to 150 ms over 90 KiB (UNVERIFIED) against 0.33 ms for native SHA-256 on 150 KB (P14), and the cost lands on every capture and every K-4 compare in release builds.
- **Rejection Reason**: R-17 ratified the native truncated SHA-256; the probe supports it. The reserved ids stay unused.

### Alternative 4: The engine `hash()` or a `RandomNumberGenerator` per stream
- **Description**: Use `hash(Variant)` for state fingerprints and a seeded `RandomNumberGenerator` (one per system) for draws, saving its `state` in the block.
- **Pros**: Built in; less code.
- **Cons**: `hash()` is not a stable cross-version or cross-platform contract; the generator's algorithm is an engine detail that can change in a patch; its `state` is a second 64-bit value per stream to persist; draws would depend on call order, so a weld that adds candidates shifts unrelated outcomes (the Graph rejected a sequential counter for exactly that reason, TD R10); no replay by key.
- **Rejection Reason**: D.10's pure keyed function has no hidden state, replays from `(seed, name, counter)`, and is pinned by golden vectors.

### Alternative 5: D.12's "bytewise by encoded key bytes" dictionary order
- **Description**: Keep the kernel GDD's original rule.
- **Pros**: Needs no type-specific comparison; the kernel GDD already says it.
- **Cons**: Needs a byte-lexicographic comparison of two `PackedByteArray`s, which GDScript lacks as an operator (a hand loop per comparison); orders integer keys `3, 10, -5` and strings length-first, which makes the G4 diff output unreadable.
- **Rejection Reason**: Equally injective, strictly worse to implement and to read. § 3.3 sorts by value; the GDD sync item records it.

### Alternative 6: An exact-integer `rng_int` in place of the float path
- **Description**: Replace `floor(unit * span)` with a 64-bit split multiplication.
- **Pros**: No floating point anywhere in the simulation.
- **Cons**: The split overflows `int64` for spans near `2^32` (probed in the scratch analysis) and needs a 128-bit emulation; any change of rounding silently changes drawn values and invalidates D.10's goldens before the first save ships.
- **Rejection Reason**: One IEEE-754 multiply of exactly representable operands is deterministic on every supported platform. The path is isolated, named, and tested against an arbitrary-precision oracle; if the oracle ever disagrees, that is a technical-director decision, not a silent patch.

### Alternative 7: Always-fresh hashes (no cache), or an eager dirty-time recompute
- **Description**: Either re-encode every block every time a hash is asked for, or recompute a block's hash the moment it is opened for writing.
- **Pros**: No stale-cache failure mode at all (no cache), or always-current hashes (eager).
- **Cons**: No cache makes the Overlay cadence and Layout's `layout_hash()` re-encode every block each time (O(blocks) instead of O(dirty)); eager recompute re-encodes a block on every mutator call in P2 or P3, which breaks the P3 budget and the "quiet tick = 0 encodes" proxy.
- **Rejection Reason**: A lazy cache validated by the touch serial, used only on the diagnostics path, with every verification path uncached, gives O(dirty) cost and keeps the stale-cache bug detectable (KRN-HASH-02), plus a debug detector (§ 5.5).

### Alternative 8: Per-system presentation hashes (status quo)
- **Description**: Leave `he_hash`, `hash32`, Lighting's `H` and the Instruments hash as each GDD defined them.
- **Pros**: No change to any GDD.
- **Cons**: Four hash functions with four sets of golden vectors and four ways to mis-handle sign and overflow (PF D.14 already needs explicit 32-bit masking to avoid wrap).
- **Rejection Reason**: `he_hash` is already defined on the pinned `mix64` with a non-negative result and is the only one with a stated sign rule; generalising it with a seed costs one parameter.

## Consequences

### Positive
- One byte definition, one hash definition, one RNG derivation, one place the epoch is read. The determinism guarantee becomes a property of three small modules (`DetRng`, `CanonicalEncoder`, `StateHasher`) and a lint list, not of thirty GDDs.
- The two most likely silent determinism bugs become structural errors: a threat draw without the epoch (the facet will not let `draw` run on an epoch-keyed stream) and an owner byte layout (no owner has the means or the permission).
- Quiet ticks cost nothing; a diagnostics refresh is O(dirty blocks); the stale-cache failure is detected in debug within one cadence and on every verification path in all builds.
- Every integer-semantics guess in twelve GDDs is replaced by a probed fact with a named spike for the release export.
- `hash_algo` gives hash evolution without a save-format bump.

### Negative
- The kernel GDD's D.12 wording changes (dictionary key order), the Graph's LS-21 layout is withdrawn, Attention AT-8 and several GDD hash definitions change; the GDD sync list below must be applied by their owners.
- Every capture and K-4 compare pays a GDScript canonical encode of every persisted block on the main thread, plus a deep-copying `snapshot` per block read for hashing; the encoder cost for Dictionary-heavy blocks is unmeasured (S0-e) and may force more blocks onto packed arrays.
- `block_hash` is unavailable for a block while the active scope has opened it, so a hook that writes Layout cannot read `layout_hash()` until it returns.
- Sorting dictionary keys allocates an `Array` per dictionary node on the encode path.
- A float in a SIM block needs an allow-list entry; designers who want a float field in a state block are told no.
- Sanity's dynamic per-subject stream names allocate a string on first use of each name (accepted, bounded by player-driven events).
- Cross-platform identity is a design property until the CI matrix proves it (R2); a Steam Deck divergence would surface late.
- Two files in `src/shared/det/` (`det_rng_float.gd`, `canonical_encoder.gd`) sit on ADR-0010's float allow-list and one on its `Variant` allow-list, so the lint rules need three small exceptions that must stay minimal.
- The ADR-0010 warning gate forbids several natural names (`seed`, `hash`, a parameter named like a sibling function) and the `as` cast on a `Variant`; every snippet needs the load-all check (LINT-SPIKE-1).
- `ENC_DUP_KEY` cannot be produced from a `Dictionary`, so one error code is defence only.

## Risks
- **Integer behaviour differs on a release export, on Linux/Deck or on macOS.** Mitigation: the § 6 vectors run in the cross-OS vector job (Windows and Linux; macOS advisory) and on a `template_release` export on the exact shipping templates (S0-b); S0-a to S0-d gate acceptance; the sign-split fallback (a form that never divides a negative) is documented for any helper that fails.
- **The GDScript canonical encoder exceeds the capture budget** (D.11: about 0.5 ms per block, 250 to 500 scalar fields). Mitigation: packed arrays for large blocks (rule 3.6); S0-e measures on RM-DECK; ladder in § 5.6 (worker hashing, sub-payload folding under a new `hash_algo`); an optional owner `hash_view` callable that returns a non-copied read-only payload, so hashing skips the deep copy, is a candidate amendment to ADR-0003 if the copy dominates (open question 4).
- **A block is written outside a `WriteScope`** (or through a leaked mutable reference), so the touch serial does not advance and the cache goes stale. Mitigation: every verification path is uncached; `SIM_WRITE_TRIPWIRE`; the debug detector (§ 5.5) names the block; the weld property test. A forgotten `mark_dirty` no longer stales the hash cache (the serial rides `open_for`); it is ADR-0004's `MISSED_DIRTY_MARK` concern.
- **A designer or contributor reintroduces `randf()`, `hash()` or an unsorted dictionary walk.** Mitigation: the § 1.7 lint rules with bad/good fixtures; regex where a regex is sound, runtime or AST where it is not (ADR-0010); the `StringName`-sort rule has a runtime fallback through the encoder.
- **`Ihash` weakness at structured inputs** (`mix64(0) = 0`; the three-key combine is not a full avalanche). Mitigation: it is cosmetic and non-cryptographic; a non-zero seed per system; golden vectors pin its output; if a system needs better statistics it asks for a stream.
- **The encoding or fold is wrong in a way the vectors do not exercise** (for example astral-plane strings, very long names, or a packed `PackedVector2Array`). Mitigation: the property test, an astral-string vector added in S0-a, and the Ship Body fixture; the Python reference is the arbiter.
- **`restore_epoch` leaks into a hash through a new code path.** Mitigation: the lint on `RESTORE_KEY_STRIDE` and `restore_epoch`, and the KRN-EPO-02 test that a different epoch changes exactly the threat draws.
- **An interface snippet or idiom does not load under the ADR-0010 warning gate** (shadowed names, `as` on a `Variant`, a `float(` call, an over-range literal that loads silently, `/=`). Mitigation: P18 to P20 record the real behaviour; the names and idioms in Key Interfaces and § 3.8 were re-probed; LINT-SPIKE-1 loads every snippet in the first story; the lexical rules cover what the engine does not.
- **Cross-ADR drift** (ADR-0004's mark path, ADR-0003's `has_opened`, ADR-0006's use of `hash_algo` and `block_state_hash`). Mitigation: each assumption is listed under Related and must be confirmed at the joint review.

## GDD Requirements Addressed

The planned-ADR column of `docs/architecture/requirements-traceability.md` assigns 47 TR-IDs to ADR-0005 (28 as sole ADR, 19 shared). `tr-baseline.md` carries 21 of them; the other 26 come from the Acceptance Criteria completeness pass recorded in the traceability matrix. The first table covers the 33 requirements this ADR addresses in substance (the 28 sole-owner rows and five shared rows where this ADR holds the primary decision); the sub-table below it lists the 14 shared rows another ADR owns and where this ADR supplies only the determinism or hashing half. The traceability index row for ADR-0005 lists 42 IDs; the five rows beyond it (TR-cmd-027, TR-cmd-043, TR-debug-031, TR-debug-033, TR-emitter-038) are secondary rows owned elsewhere and should be added to the index row by `/architecture-review`.

| GDD System | Requirement | How This ADR Addresses It |
|------------|-------------|--------------------------|
| command-simulation-layer.md | TR-cmd-019: RNG never global; named streams with persisted counters; pure `rng_derive` and `rng_derive_epoch` on SplitMix64 `mix64`, pinned by golden vectors | § 2.1 to § 2.4: constants, stream registry, `RngFacet`, the D.10 vectors reproduced on 4.7.2 and in Python (P15) |
| command-simulation-layer.md | TR-cmd-020: `state_hash(PERSIST, T)` = H64 over `canonical_encode` bytes, packed arrays raw, published `block_hash`, prefix `T \|\| world_seed` | § 4.1 to § 4.5: H64 definition, record framing, worked fold `0x81AB3842EF1E8DEF` |
| command-simulation-layer.md | TR-cmd-032: a quiet tick performs 0 encodes; a tick dirtying `k` blocks performs `k`; capture and P3 budgets | § 5.1 to § 5.6 (lazy cache, counters, which paths hash); the wall-clock budgets stay the kernel's and are gated on RM-DESKTOP / RM-DECK (S0-d, S0-e) |
| command-simulation-layer.md | TR-cmd-039: TLV injective after normalisation; key order irrelevant, `-0.0 == +0.0`, NaNs equal, `StringName == String`, INT and FLOAT differ; `PERSIST` covers WORLD persist blocks only, `ALL_SIM` every SIM block | § 3.2 to § 3.4, § 3.7 vectors, § 4.4 scopes |
| command-simulation-layer.md | TR-cmd-040: block hashes cached, invalidated only by `mark_dirty`; every verification path uncached; `BLOCK_HASH_IN_WRITE` / `BLOCK_HASH_UNKNOWN` | § 5.2 to § 5.4. The invalidation input changes from `mark_dirty` to ADR-0004's touch serial (a quantised write marks no key); the TR wording and D.12 are synced (Migration Plan 1). |
| life-support-graph.md | TR-lsg-021: `canonical_bytes()` as tagged raw packed arrays; cached H64 `block_hash()` recomputed only after a write | § 3.1 (the kernel encodes the Graph's packed-array payload; LS-21's owner layout withdrawn), § 5 (cache), GDD sync item |
| life-support-graph.md | TR-lsg-031: `graph.state` hash and `rng_derive` draws for fixed scripts bit-identical on Windows, Linux, macOS via committed golden files | § 1.1 R2, § 6 cross-OS vector job; the Graph fixtures' golden files are the Graph epic's Validation Criteria; UNVERIFIED until it runs |
| life-support-graph.md | TR-lsg-037: SHA-256 over about 150 KB at most 1 ms (S0-d) | P14: 0.33 ms indicative on the dev machine; RM-DECK release-export measurement is the gate (§ 5.6 ladder if exceeded) |
| life-support-graph.md | TR-lsg-039: diagnostics counters never in canonical bytes or any hash | § 3.6 (payload excludes counters), § 5.6 (kernel counters are likewise unhashed) |
| item-and-module-database.md | TR-itemdb-012: `ItemInstance` positional encoding with canonical zeros so holder blocks have one encoding per state, no float, no `res://` path | § 3.1: a payload-shape rule over `Array` of `int`, encoded as `INT` |
| emitter-state-and-light-exposure-query.md | TR-emitter-034: no bare integer `/` or `%` in `src/sim/emitter/`; possibly negative operands through checked helpers; grep lint with bad/good fixtures | § 1.4 (helper semantics: `floor_div`, `posmod_i`, the single annotated `/`) and § 1.7 (the rule); the lint tool and fixtures are ADR-0010's (`SIM-04`, `SIM-05`) |
| 2d-lighting-and-occlusion.md | TR-light-032: cosmetic randomness is a pure hash of (emitter id, epoch), no global generator, no `rng_derive` stream | § 2.6 `Ihash`, § 2.7 table; ADR-0028 uses it for the gutter recipe |
| player-controller-input.md | TR-pc-027: controller and body sources contain no forbidden time or RNG calls except allow-listed render-time timers | § 1.7 row 1: the forbidden-token list is ADR-0010 `SIM-02`; render-time timers use `delta` or `present_time` and need no `Time.*` token, so no separate allowance exists here (wall time is the Save/Load-only `WallClock` port) |
| failure-and-trail-system.md | TR-ftrail-044: mapping a 64-bit hash result to variant, phase and flip indices is non-negative for every bit pattern including all-ones and the sign bit alone | § 2.6 and § 2.7: the Trail's variant, phase and flip come from `Ihash.h(TRAIL_VIS_SEED, subject_id, stamp_index, world_seed, SALT_TRAIL_VIS)`, a non-negative 63-bit `int`, indexed with `IntMath.posmod_i(h, n)` (or `>>` on the non-negative `h`), never `raw % n`; presentation never calls `DetRng.derive`. The all-ones inputs are covered by the `Ihash` `-1, -1, -1` vector; FT-20, D.6 and AC-134 regenerate (sync item). The matrix row stays a Gap until `/architecture-review` moves it |
| suit-and-equipment.md | TR-suit-005: no RNG, no wall clock, no `delta`; all block fields int32-safe integers or plain-data dictionaries | § 1.2, § 1.7; the null `ModuleFacet.rng` (§ 2.3) makes "no RNG" structural |
| suit-and-equipment.md | TR-suit-030: defined integer-division convention for negative operands in the thermal integrator and rung dwell | § 1.4: floor for every quotient; explicit `IntMath.trunc_div` when truncation is meant (closes ADR candidate 4) |
| ship-section-and-layout-data-model.md | TR-section-026: `layout_hash` is the kernel `block_hash` of `layout.state`; ascending-id iteration; no RNG; `welded_at_T` the only time value | § 3.1, § 4.3, § 1.6, § 2.7 |
| ship-section-and-layout-data-model.md | TR-section-031: `layout_hash()` memoised, recomputed only after a mutation or restore; `geometry_hash` / `visual_hash` computed once per template load | § 4.6, § 5.3: the caller memoises on its own mutation counter; the kernel read is uncached by default and never per tick |
| ship-section-and-layout-data-model.md | TR-section-032: template hashes through `PackedInt32Array` compared after `& 0xFFFFFFFF`; `mutation_count` never persisted; only hash is kernel H64 via `HashingContext`; lint bans `hash()` and byte-accumulator loops | § 1.3 (`low32`, P9), § 4.1 (scripted hashes and `hash()` banned), § 4.6, § 1.7 |
| ship-body-and-interior-physics.md | TR-shipbody-031: no RNG, wall clock, float state or unordered-container iteration; correct for 1..4 `body.integrate` runs per frame | § 1.2, § 1.6, § 1.7, § 2.7; the render-rate independence test in § 6 |
| modular-ship-assembly.md | TR-assembly-016: weld apply is integer arithmetic only; no RNG, no wall clock inside any Assembly apply or hook | § 1.2, § 1.5, § 1.7; null `ModuleFacet.rng` |
| boarding-and-salvage.md | TR-boarding-027: all values integer tick/AU arithmetic over owners' constants; no wall-clock performance criterion of its own | § 1.2, § 1.4, § 1.5 |
| web-map-and-transit.md | TR-webmap-026: restore-equivalence determinism: restored runs have identical `state_hash` sequences, bounded accruals excepted | § 1.1 R1, § 4.4, § 6 G4 variant; the three accruals are Web Map's to name |
| web-map-and-transit.md | TR-webmap-031: integer-only, scene-tree-free; every division and modulo on non-negative operands; 8-byte seed never a GDScript int; no `rng_derive` stream | § 1.4 (`IntMath` only), § 2.3 (null facet), § 4.8 (seeds as `PackedByteArray` bytes) |
| attention-system.md | TR-attention-013: the tell draw is the only `rng_derive` use; beat jitter and presentation randomness a pure integer hash of `(tell_epoch, k, HASH_SEED)`; diagnostics outside `state_hash` | § 2.4, § 2.6 (`Ihash.h32`), § 3.6 |
| attention-system.md | TR-attention-036: F-1, F-3, F-4, F-5 intermediates below `2^62` under worst-case inputs, verified against an arbitrary-precision oracle | § 1.5: the headroom rule, boot product checks, oracle tests |
| horror-entities.md | TR-horror-016: all simulation randomness is the REACH length via `rng_derive_epoch` (counter below `2^20`); everything else the pure-integer hash with fixed salts; golden vectors | § 2.3, § 2.4 (epoch-keyed `horror.tell`, `RESTORE_KEY_STRIDE`), § 2.6 (`he_hash` becomes `Ihash` with `HE_HASH_SEED` 0x4845; its vectors in § 2.6) |
| horror-entities.md | TR-horror-020: the same (save bytes, `restore_epoch`) pair replays identically; mark ring, counters and epoch ledgers hashed; `horror.view.debug` non-hashed | § 1.1 R1, § 3.6, § 2.4 |
| sanity-and-hallucination-rules.md | TR-sanity-016: draws only through `rng_derive` / `rng_int` with persisted counters; fixed draw count per outcome | § 2.2 (dynamic stream names `sanity.lie/<PlayerId>/<stable_id>`), § 2.7 |
| sanity-and-hallucination-rules.md | TR-sanity-017: bit-exact replay; no content depends on restore reason or epoch; sorted-key snapshots and hashes; `get_instance_id`, `hash()`, `shuffle`, `pick_random`, unstable `sort_custom`, unsorted dictionary iteration banned | § 1.6, § 1.7, § 2.4 (epoch read only in `draw_epoch`) |
| in-world-instruments-and-inspect-view.md | TR-instr-034: tremor frequencies and phases from a hash of the stable `ref_key`, never `rng_derive` or a wall-clock source | § 2.6 (`Ihash`), § 2.7 |
| debug-overlay-true-values.md | TR-debug-019: the Overlay never hashes; shows the kernel's `state_hash`, `hash_T`, `hash_algo` and age; kernel hash every `DIAG_HASH_CADENCE_TICKS` 60, outside P3 | § 4.4 (`hash_algo` 2), § 4.7, § 5.4 (cadence is a cached-profile read) |
| debug-overlay-true-values.md | TR-debug-033: every owner optionally publishes `block_hash()`; Overlay shows n/a where none; HASH_DRIFT names a block only from an owner-published hash | § 3.1, § 5.3: the kernel publishes `block_hash` for every `persist` block, so no owner publishes one and the Overlay shows `BLOCK_HASH_UNKNOWN` as n/a; GDD sync item |

**Owned elsewhere (this ADR supplies the determinism or hashing half only).**

| GDD System | Requirement | Owner ADR and this ADR's part |
|------------|-------------|-------------------------------|
| command-simulation-layer.md | TR-cmd-018: `restore_epoch` owned and persisted by Save/Load (`slot_epoch.bin`), held read-only in the clock view, excluded from the snapshot header and all hashes | ADR-0006 (file, advance and durability rules; ADR-0006's container-header recovery copy is file metadata outside every hash). This ADR states, once, that the epoch is a non-persisted, non-hashed `kernel.clock` value read only inside `draw_epoch` (§ 2.4, IC-5). |
| command-simulation-layer.md | TR-cmd-027: commands apply in ascending `(queued_tick, host_order)`; the same stream and `(save bytes, restore_epoch)` yield an identical canonical hash at tick N | ADR-0002 (ordering). This ADR defines the hash that is compared and the replay level R1 (§ 1.1, § 4.4, § 6). |
| command-simulation-layer.md | TR-cmd-038: snapshots round-trip losslessly through `var_to_bytes`; JSON interchange writes every 64-bit value as `hex64` | ADR-0006 (container). This ADR defines `Hex64` and the 64-bit rule (§ 4.2, § 4.8). |
| command-simulation-layer.md | TR-cmd-043: `host_order` and `T` are 64-bit and never wrap; `correlation` excluded from canonical bytes and hashes | ADR-0002 (stamping, boot assertion). This ADR fixes `T` as `s64` in the fold and excludes `correlation` and the command log from every hash (§ 4.4). |
| item-and-module-database.md | TR-itemdb-029: positional `ItemInstance` encoding at most 96 bytes, 1,024-byte holder block, each integer at most `2^53` | ADR-0006 and ADR-0002 (the size is a `var_to_bytes` size). This ADR notes the wording issue (§ 3.1, open question 6). |
| emitter-state-and-light-exposure-query.md | TR-emitter-038: mutators do nothing while RESTORING or FAULTED; unsupported `schema_version` is a reported failure; the block exposes byte-identical `canonical_bytes()` and `block_hash()` | ADR-0003 (restore contract). This ADR supplies the kernel `block_hash` / `canonical_encode` reads that replace the owner-published ones (§ 3.1, § 5.3). |
| interaction-system.md | TR-interact-033: every packed-array write marks its dirty key; snapshot-restore-snapshot is byte-identical; replaying a command log twice gives the same `state_hash` every tick | ADR-0004 (dirty marks). This ADR supplies the uncached hash comparison and the stale-cache detector (§ 5.4, § 5.5, § 6). |
| ship-section-and-layout-data-model.md | TR-section-043: engine behaviours (JSON numbers, packed-array element-write setter semantics, `var_to_bytes` determinism, negative `/` and `%`, SHA-256 and `>>`, `PackedInt32Array` at or above `2^31`) probed on 4.7.2 before dependent ADRs are accepted | ADR-0010 (spike). Probed 2026-10-28 on the editor binary: P3, P5, P8, P9, P13; JSON numbers are ADR-0007's CONTENT-EXP-1 (and P21 here: every parsed JSON number is a `float`); `var_to_bytes` determinism is deliberately not relied upon; release-export confirmation remains the S0 gate. |
| ship-body-and-interior-physics.md | TR-shipbody-038: integer-semantics assumptions (`posmod`, truncation, float32 canvas exactness, `OBJECT_COUNT`, directory-scoped warning levels) unverified and gated by spike S2 | ADR-0010 and ADR-0008. This ADR probed `posmod`, truncation and the warning gate (P5, P6, P19, P20). Float32 canvas exactness is ADR-0008's coordinate rule (presentation coordinates are integer-valued float32, exact for \|v\| <= 2^24 base px; `Vector2i` converts to `Vector2` only at `Placement.put` and the `canvas_transform` write; ADR-0015 owns the bound if body-frame extents could exceed it). `OBJECT_COUNT` is ADR-0010's, and the `directory_rules` values are settled by ADR-0010 P2 and P3. |
| modular-ship-assembly.md | TR-assembly-040: weld and cut conserve integer positions and keep the canonical `state_hash` identical at every tick regardless of zoom, seat count or MIRROR role | ADR-0015 (kinematics). This ADR supplies the replay level R1 and the uncached comparison used by the property test. |
| boarding-and-salvage.md | TR-boarding-030: the simulation is drivable headlessly; golden schedules come from the owners' real code, never typed | ADR-0010 (test harness). This ADR requires that golden hash and schedule files be generated by the reviewed golden-update job (§ 6, Implementation Guidelines). |
| save-load-and-versioned-format.md | TR-saveload-032: death-stub restore advances `restore_epoch` and makes it durable; a bad `slot_epoch.bin` is rebuilt; saturates at `2^32 - 1` | ADR-0006. This ADR fixes the consumer side: the epoch range is `0..2^32 - 1` (`RESTORE_EPOCH_RANGE` already exists in ADR-0003) and `counter_eff` stays below `2^52` (§ 2.3). |
| save-load-and-versioned-format.md | TR-saveload-035: post-restore verification in EXACT or REBASED mode against the K-4 `state_hash` or a recorded baseline | ADR-0006. This ADR defines the compared `state_hash(PERSIST, T, uncached = true)`, `hash_algo` 2, and that an unsupported `hash_algo` forces REBASED (§ 4.4); "no expected hash" is `has_expected_hash = false`, never a hash value (§ 4.1). |
| debug-overlay-true-values.md | TR-debug-031: the kernel `diagnostics` accessor publishes allocation-free per tick (including a forced hash at restore) | ADR-0003 and ADR-0031 (accessor shape). This ADR supplies the hash fields and counters (`state_hash`, `hash_T`, `hash_algo`, § 5.6) and requires the cadence hash to allocate only on the dirty path. |

## Performance Implications
- **CPU**: A tick that dirties no block and has no hash reader costs nothing from this ADR. A cached diagnostics refresh costs one encode plus one SHA-256 per block dirtied since the last refresh. A capture or K-4 compare encodes every persisted block once (uncached): the packed-array blocks cost a `to_byte_array()` copy and a SHA-256 pass (P14: 0.002 ms and 0.33 ms for 96 KB and 150 KB respectively on the dev machine, editor binary); Dictionary blocks cost a GDScript walk whose rate is unmeasured (S0-e). `rng_derive` is about 0.48 microseconds per call in the editor binary, and draws are rare (tells, thaw rolls, sanity events). The static helper `floor_div` costs about 12 times a bare operator, 0.20 against 0.017 microseconds in a tight loop (P14, re-measured), which S1 re-measures on a release export before any hot-path exception is considered (§ 1.4). All figures are indicative; the gate is a release export on RM-DESKTOP and RM-DECK.
- **Memory**: The hash cache is `3 * 8 * 128` bytes at the registry cap (128 blocks), about 3 KB. The encoder holds one scratch buffer sized to the largest block seen (the Graph, about 100 to 300 KiB). A dirty-path encode allocates the result array, one `keys()` array per dictionary node and strings for sorted string keys; none of it persists.
- **Load Time**: A restore recomputes `state_hash(PERSIST, T)` once after `post_restore` (K-4): one uncached pass over the WORLD blocks (about 160 KiB at MVP, D.11). Boot registers a few dozen streams and computes their keys once.
- **Network**: None at MVP. For VS-B the hash definition is the MIRROR comparison primitive; it is platform-independent by design (R2, unverified).

## Migration Plan
Nothing is built yet, so this ADR changes documents, not code. The GDD edits below are listed as sync items in the author's return value for the owners to apply (this ADR does not edit them).
1. Kernel GDD D.12: dictionary keys sorted by value (§ 3.3) in place of "bytewise by the encoded key bytes"; KRN-HASH-01's worked dictionary example regenerated; the `state_hash` fold widths stated (§ 4.4); `hash_algo` defined as the whole definition; `block_hash` for every `persist` block published by the kernel; the cache is "invalidated by the block's touch serial (every granted `open_for`; restore also advances it), not by a dirty key, because quantised blocks write without marking" (the text ADR-0004 drafted, conditional on this ADR); KRN-HASH-02 re-targeted to a write made outside any scope; TR-cmd-040's "only by `mark_dirty`" reworded.
2. Life-Support Graph LS-21: withdraw the owner-side `[u32 tag][u32 length][field.to_byte_array()]` layout; the Graph's `snapshot` payload is a `Dictionary` of named packed arrays encoded by the kernel; `canonical_bytes()` / `block_hash()` forward to the kernel reads.
3. Attention AT-8: call `rng_derive_epoch` through the facet and derive nothing (OQ-6); AT-28 / `hash32` becomes `Ihash.h32(HASH_SEED, tell_epoch, k, 0, salt)`.
4. Horror HE-42: `he_hash` is `Ihash.h(HE_HASH_SEED, ...)`; Lighting LO-19 `H(id, k)` and Instruments tremor become `Ihash` callers with their own seeds.
5. Debug Overlay TR-debug-033 wording: the kernel, not the owners, publishes `block_hash`.
6. Save/Load: `hash_algo` names the whole hash definition; TR-itemdb-029's size bound is stated over `var_to_bytes`.
7. Emitter, Suit and Web Map: reference § 1.4 for the division convention instead of per-GDD wording.
8. Open spikes: run S0-a to S0-d on a `template_release` export (Windows, Linux / Deck), run S0-e with the synthetic stand-ins, and add S0-e and the cross-OS vector job to `architecture.md` § Spikes with the user's approval before Accept (`architecture.md` has neither today). Remove the Graph row's `HashingContext` use from `architecture.md` (the Graph publishes no byte writer, § 3.1).
9. Failure & Trail FT-20, D.6 and AC-134 (TR-ftrail-044): replace `rng_derive(world_seed, "trail.vis", ...)` with the `Ihash` form of § 2.6; the stub vectors regenerate. ADR-0009 line 137 changes to "reads `Ihash` keyed by `world_seed` (ADR-0005 § 2.6)".
10. Sibling ADR amendments, applied by their owners (listed under Related): the ADR-0003 batch, the ADR-0002 snapshot outcome and restore signature, ADR-0006 (Hex64 codec, `canonical_bytes()` sentence, rung 1, `block_state_hash` naming), ADR-0010 (allow-lists, token list, rule additions, WallClock), ADR-0008 (coordinate rule), ADR-0009 (OQ-F), ADR-0004 (`block_hash_any`, Depends On), and the Enables / Depends On symmetry edits.
11. Traceability: add the five secondary rows to the ADR-0005 index row and move TR-ftrail-044 off Gap once item 9 lands (`/architecture-review`).

## Validation Criteria
- All D.10 vectors in § 2.4 reproduce on Windows and Linux runners (macOS advisory) and on a release export; `rng_derive_epoch` at epochs 0, 1 and 2 gives three different values (KRN-EPO-01).
- The § 3.7 vectors and the § 4.5 fold reproduce exactly from the GDScript encoder; the Python reference agrees on every row; the Ship Body fixture length is confirmed or corrected.
- KRN-HASH-01 (injectivity property test, 10,000 seeded values), KRN-HASH-02, re-targeted (a write made outside any `WriteScope` leaves a stale cache that the uncached paths catch and `HASH_CACHE_STALE` flags; a write inside a scope that forgets `mark_dirty` does not stale the hash cache), KRN-HASH-03 (`block_hash` read semantics, counters, IN_WRITE and UNKNOWN) pass.
- KRN-PERF-01, -02 and -05: 0 encodes on a quiet tick; `k` encodes after `k` dirty blocks on a cached read; 0 element-loop iterations over packed arrays.
- G3 (two kernels, ALL_SIM), G4 (snapshot, restore, continue, PERSIST), the cross-process test and the render-rate independence test pass, with the first differing `block_id` and key named on a mismatch.
- `H64` over 150 KB of canonical bytes is at most 1 ms on RM-DECK (TR-lsg-037). The golden `state_hash` files for the Graph fixtures (TR-lsg-031) are a Validation Criterion of the Graph epic, not of this ADR's Acceptance.
- Each reachable `ENC_*` code is produced by a strict-mode fixture; a payload built with `String` keys and the same payload built with `StringName` keys encode identically (P22: `ENC_DUP_KEY` is unreachable from a `Dictionary`); mixed key types are `ENC_MIXED_KEYS`.
- The § 1.7 lints each have a committed bad fixture that fails and a good fixture that passes; a bare `/` without the annotation fails to compile with `integer_division` at error level, and the repository holds exactly one such annotation (ADR-0010 `SIM-05`); `x /= y`, `x %= y`, an over-range decimal literal and an over-range hex literal each fail their lexical fixture.
- The facet refuses misuse: `draw` on an epoch-keyed stream and `draw_epoch` on a plain stream each count `RNG_STREAM_KIND_VIOLATION`; a duplicate stream name or key collision fails boot with `DUPLICATE_ID`.
- LINT-SPIKE-1 loads every Key Interfaces snippet under the ADR-0010 § 5.1 warning configuration with no error; a snippet naming a member `seed`, `hash` or after a sibling function fails it (negative fixture).
- Each owner that uses `Vector2i` or `PackedInt32Array` has a boot check or unit test against `2^31` (P17 fixture: `Vector2i(2147483647 + 1, 0)` reads back negative); the JSON readers reject non-integral and `> 2^53` numbers (P21 fixture).
- `encode` returns an array distinct from its scratch buffer (mutating the next encode does not change an earlier result), and the G3 worker variant passes.
- After this ADR is Accepted: ADR-0007's golden `catalogue_hash` and template hash vectors are committed.

## Related
- **Depends on**: ADR-0003, ADR-0004, ADR-0001 (injection only), ADR-0010 (soft); all Proposed.
- **Enables**: ADR-0006, ADR-0007 (golden vectors), ADR-0011, ADR-0012, ADR-0013, ADR-0015, ADR-0016, ADR-0017, ADR-0019, ADR-0021, ADR-0022, ADR-0023, ADR-0024, ADR-0025, ADR-0028, ADR-0029, ADR-0031.
- **Cross-ADR amendments required before Accept** (sibling-owned; this ADR does not take over their decisions; also returned to the orchestrator):
  - ADR-0003 (one batch): `ModuleFacet.rng: RngFacet`, null when the module declares no streams; `WriteScope.has_opened(block_index: int) -> bool`; the `kernel.clock` view carrying `(T, world_seed, restore_epoch)`, `world_seed` set before `INIT`, `restore_epoch` a `u32`, read-only, non-persisted and non-hashed, set by `request_restore` / `request_new_game` before the first `restore()` / `init_world` and read only inside `RngFacet.draw_epoch`; the hash-cache arrays and `touch_serial` in the H1 registries group; a second kernel-internal `SnapshotContext` for hash reads outside `SAVING`, with a re-entrancy flag; packed arrays in a snapshot payload are duplicates; every override of a typed-return owner callable returns on all paths; the `block_id` to `block_index` map built at the registry freeze.
  - ADR-0004: `Kernel.touch_serial(block_index)` exists in every build and advances on every granted `open_for` in every scope kind, including `INIT` and `RESTORE`; § 7 cites `block_hash_any` and gates the `MISSED_DIRTY_MARK` audit on the profile that runs `SIM_WRITE_TRIPWIRE`; ADR-0010 moves from Enables to Depends On; the dirty-key layout does not affect any hash.
  - ADR-0002: the snapshot `ControlOutcome` adds `T`, `world_seed`, `state_hash`, `hash_algo`, `block_ids`, `schema_versions`, `block_state_hashes` and `enc_lens`, per persist block (WORLD and PROFILE), computed uncached in `SAVING`; `request_restore(payload, restore_epoch, expected_state_hash = 0, has_expected_hash = false)` with the K-4 compare skipped when the flag is false; the registry `control_plane` signature follows.
  - ADR-0006: calls `Hex64.encode` / `Hex64.decode` (its duplicate codec is removed; `campaign_id` keeps a local exact-width lowercase helper); drops its `canonical_bytes()` embedding sentence (the Graph payload is a Dictionary of named packed arrays, stored by `var_to_bytes`, hashed as ADR-0005's TLV); rung 1 is deferred and off by default; the per-block hash field is `block_state_hash`; `created_unix` comes from `WallClock`; ADR-0005 and ADR-0010 move to its Depends On.
  - ADR-0010: § 1.1 gains a `src/shared/det/` row; `DET-01` path is `src/shared/det/` plus the `src/save/` exemption of § 1.7; the § 5.4 `IntMath` snippet becomes a citation of this ADR; OQ-E closed; `float_allowlist.json` for `src/shared` is exactly `det_rng_float.gd` and `canonical_encoder.gd`; `dictionary_variant.json` adds `canonical_encoder.gd` (and the pass-through signatures, open question 13) with a reason; `SIM-04` exempts `det_rng_float.gd` for `float(` / `floor`, bans `%`, `/=`, `%=`; `SIM-02` is the unified token list; `SIM-03` / `SIM-04` add the trig names and a new `SIM-03` sibling bans over-range decimal literals, each with a bad fixture; § 4.1 and `IO-01` list `HashingContext` as allowed only in `src/save/`, `src/data/`, `hash64.gd` and `state_hasher.gd`; `src/shared` is "scene-tree-free, I/O-free, no global state"; the `WallClock` port; the sanctioned `Variant` unpacking idiom; § 5.4 corrected (`INTEGER_DIVISION` covers `a / b` only).
  - ADR-0008: the canvas coordinate rule (integer-valued float32, exact for \|v\| <= 2^24 base px) that TR-shipbody-038 cites. ADR-0009: line 137 changes to "reads `Ihash` keyed by `world_seed` (ADR-0005 § 2.6)"; OQ-F closes with the PF-37 waiver for the `Ihash` family; Depends On gains ADR-0004 and ADR-0010. ADR-0001: Enables gains ADR-0005, and its token-list pointer cites ADR-0010. ADR-0007: Depends On gains ADR-0005 (golden content-hash vectors only).
  - `architecture.md`: remove the Graph row's `HashingContext` use; add S0-e and the cross-OS vector job to § Spikes (user approval required).
- **GDDs**: `command-simulation-layer.md` (CR-8, D.10, D.11, D.12, KRN-HASH-01..03, KRN-EPO-01/02, KRN-PERF-01..05, OQ 16), `life-support-graph.md` (LS-9, LS-21, D.13), `save-load-and-versioned-format.md` (SL-16, SL-23, F-10), `attention-system.md` (AT-8, AT-28), `horror-entities.md` (HE-42), `sanity-and-hallucination-rules.md` (SN-24), `ship-section-and-layout-data-model.md` (LY-26 e, LY-33), `2d-lighting-and-occlusion.md` (LO-19), `in-world-instruments-and-inspect-view.md`, `perception-filter.md` (D.14), `suit-and-equipment.md`, `emitter-state-and-light-exposure-query.md`, `web-map-and-transit.md` (WM-37), `item-and-module-database.md` (IDB-15).
- `docs/architecture/architecture.md` § Required ADRs, § Spikes that gate ADRs, OQ-6, OQ-8, IC-5; `docs/architecture/requirements-traceability.md`; `docs/registry/architecture.yaml` (entries proposed by this ADR are in Registry Candidates below and not yet written).

## Open Questions
Numbering is kept stable: questions 2 (PF D.14, answered by the waiver and ADR-0024's migration), 10 (rung 1, deferred and off by default), 11 (derived blocks, `block_hash_any`) and 12 (float allow-list, two files) were closed by the 2026-10-28 rulings and removed.
1. **Dynamic stream names allocate.** Sanity's `sanity.lie/<PlayerId>/<stable_id>` streams build a string on first use of each name. Accepted on a player-driven path, but ADR-0025 should confirm the bound (`MAX_DYNAMIC_STREAMS` 4096) is enough or replace the name with a two-integer key `stream_key_child(parent_key, a, b)` (a different byte-level value than the GDD's text-hash form, so Sanity's draws would change; decide before Sanity ships goldens).
3. **Encoder cost for Dictionary-heavy blocks** is unmeasured (S0-e). If a block exceeds the per-block share of `SNAPSHOT_BUDGET_MS`, the options are packed arrays (§ 3.6), worker-side hashing, or sub-payload folding under a new `hash_algo`.
4. **Hash without the deep copy.** The kernel hashes a block by calling `snapshot`, which deep-copies. If the copy dominates (a 96 KB Graph), an optional `hash_view` callable returning a read-only, non-copied payload would avoid it; that is an amendment to ADR-0003's descriptor and is held until S0-e shows it matters.
5. **macOS in the vector job.** TR-lsg-031 names Windows, Linux and macOS runners, but macOS may not be a shipping target; the leg is advisory until a target decision is made, and the CI cost and its value are ADR-0010's call.
6. **TR-itemdb-029 wording.** Its 96-byte bound is over `var_to_bytes`; the TLV form of the same instance is larger (9 bytes per integer). The Item DB GDD should state which form the bound applies to.
7. **Ship Body's 518-byte fixture** is a candidate length (D.12); the reference encoder confirms or corrects it at S0.
8. **Strict-mode float allow-list (the block allow-list of § 1.2, not the file allow-list)** is empty at MVP. If a future block needs a float, the owner and the technical-director decide, and the encoder's NaN / `-0.0` normalisation already makes it hashable.
9. **A kernel GDD wording question:** `canonical_encode` is published as a pure static helper and also as a kernel method here (§ Key Interfaces); the kernel owner picks one surface (this ADR needs only that the function is one).
13. **Granularity of the `Variant` allow-list.** `Hash64.of_value` and the kernel's `canonical_encode` forwarder take a `Variant` by nature; ADR-0010 decides whether `dictionary_variant.json` lists them as separate entries or covers them with the `canonical_encoder.gd` entry (§ 3.8 a).
14. **Do playtest builds run the `SIM_WRITE_TRIPWIRE` profile?** UNVERIFIED. It decides whether the ADR-0004 `MISSED_DIRTY_MARK` audit and `block_hash_any` exist in playtest builds (§ 5.3).
15. **`TRAIL_VIS_SEED` and `SALT_TRAIL_VIS` values and the regenerated FT-20 / AC-134 vectors** are the Failure & Trail GDD owner's to fix (sync item); they must be fixed non-zero constants.

## Registry Candidates
Reviewable candidates for `docs/registry/architecture.yaml`, corrected by the 2026-10-28 rulings (TD-33). The orchestrator merges them serially once the revisions land; this ADR does not write the registry. `restore_epoch` has one owner, Save/Load (ADR-0006), so this ADR adds only a `referenced_by` update for it; no existing registry entry needs a changed signature or fields from this ADR (no `revise_existing` item).

### interfaces
```yaml
  - contract: canonical_encode_and_hash
    status: active           # ADR-0005 is Proposed
    pattern: direct_call
    producer: command-simulation-layer (kernel: StateHasher, BlockHashCache)
    consumers:
      - save-load
      - debug-overlay
      - layout
      - horror
      - tests
    adr: docs/architecture/adr-0005-determinism-keyed-rng-canonical-encoding-and-hashing.md
    signal_signature: "block_hash(block_id: StringName, uncached: bool = true) -> BlockHashResult {ok, h64, reason: BLOCK_HASH_IN_WRITE | BLOCK_HASH_UNKNOWN | BLOCK_HASH_ENCODE_ERROR}; state_hash(scope: PERSIST | ALL_SIM, T: int, uncached: bool = true) -> int; canonical_encode(value: Variant) -> PackedByteArray; Hash64.of_value(value) -> int; Hex64.encode(x) -> String / Hex64.decode(s) -> Hex64Result {ok, value}; internal block_hash_any(block_index: int) -> int (SIM_WRITE_TRIPWIRE profiles; tripwire and ADR-0004 audit only); no H64 value is reserved as a sentinel (0 is legal); hash_algo 2 names the whole definition (TLV tags, key order, normalisation, fold framing, SHA-256 first 8 bytes LE)"
    referenced_by:
      - docs/architecture/adr-0005-determinism-keyed-rng-canonical-encoding-and-hashing.md
    added: 2026-10-28
    revised: ""
  - contract: keyed_rng
    status: active           # ADR-0005 is Proposed
    pattern: direct_call     # RngFacet injected as ModuleFacet.rng (ADR-0003 field); null for modules that draw nothing
    producer: command-simulation-layer (kernel: RngRegistry, RngFacet)
    consumers:
      - life-support-graph
      - attention-system
      - horror-entities
      - sanity-and-hallucination-rules
    adr: docs/architecture/adr-0005-determinism-keyed-rng-canonical-encoding-and-hashing.md
    signal_signature: "RngFacet.register_stream(name: StringName, epoch_keyed: bool = false) -> RngStream (BOOTING only); dynamic_stream(name: String, epoch_keyed) -> RngStream (memoised, bounded); draw(stream, counter) -> int (plain streams); draw_epoch(stream, rng_counter) -> int (epoch-keyed streams: attention.tell, horror.tell); int_in(raw, lo, hi) -> int; pure statics DetRng.mix64/fnv64/stream_key/derive(world_seed, key, counter) and DetRngFloat.unit/int_in; Ihash.h(variation_seed, k1, k2, k3, salt) -> non-negative 63-bit int for presentation and non-RNG variation (the Trail's trail.vis included: Ihash keyed by world_seed and a fixed per-system seed)"
    referenced_by:
      - docs/architecture/adr-0005-determinism-keyed-rng-canonical-encoding-and-hashing.md
    added: 2026-10-28
    revised: ""
```

### state_ownership
```yaml
  - state: block_hash_cache
    status: active           # ADR-0005 is Proposed
    owner_system: command-simulation-layer (kernel)
    adr: docs/architecture/adr-0005-determinism-keyed-rng-canonical-encoding-and-hashing.md
    interface: "Per-block {hash, enc_len, serial} in three kernel-private PackedInt64Arrays in the kernel registries group, indexed by block_index (dense, assigned at the registry freeze in ascending block_id order, never persisted or hashed); valid iff serial == Kernel.touch_serial(block_index) (ADR-0004); read only through block_hash(id, uncached) and the diagnostics cadence; never persisted, never hashed; uncached reads never write it; entry_valid(block_index) is the hook for the deferred ADR-0006 rung 1"
    write_access: kernel-only
    referenced_by:
      - docs/architecture/adr-0005-determinism-keyed-rng-canonical-encoding-and-hashing.md
      - docs/architecture/adr-0004-change-notification-and-dirty-keys.md   # supplies the touch serial
    added: 2026-10-28
    revised: ""
```

### api_decisions
```yaml
  - purpose: state_hash_function
    status: active           # ADR-0005 is Proposed; S0-d confirms release-export cost
    api: "H64 = digest.decode_s64(0) of SHA-256 computed by one reusable HashingContext (allowed only in src/shared/det/hash64.gd and src/kernel/state_hasher.gd for state hashing) over canonical_encode bytes; state_hash folds s64(T) || s64(world_seed) || records(u64 len(id), id, u64 schema_version, u64 len(bytes), s64 block_hash) sorted by block_id; hash_algo 2; no H64 value is reserved (0 is a legal hash); 'no verdict' is a separate bool"
    not: "var_to_bytes bytes as hash input; a scripted per-byte 64-bit hash (xxHash, FNV); the built-in hash(); an owner-written byte layout; a hash value used as a sentinel"
    adr: docs/architecture/adr-0005-determinism-keyed-rng-canonical-encoding-and-hashing.md
    reason: "Native SHA-256 is 0.33 ms for 150 KB (probed 2026-10-28, editor binary); the encoder is injective after normalisation, var_to_bytes is not a documented byte contract, hash() is not stable across versions"
    referenced_by:
      - docs/architecture/adr-0005-determinism-keyed-rng-canonical-encoding-and-hashing.md
    added: 2026-10-28
    revised: ""
  - purpose: canonical_encoding
    status: active           # ADR-0005 is Proposed
    api: "CanonicalEncoder TLV (src/shared/det/canonical_encoder.gd, the float- and Variant-allow-listed file): one tag byte (NIL 0, BOOL 1, INT 2, FLOAT 3, STRING 4, VECTOR2 5, VECTOR2I 6, ARRAY 7, DICT 8, PACKED 9) then a self-delimiting body, u64 LE counts, DICT keys sorted by value (ints signed ascending, String(key) by code point), StringName encoded as STRING, floats normalised (NaN to 0x7FF8000000000000, -0.0 to +0.0), packed arrays as raw LE bytes; Variants unpacked by typeof plus typed assignment, never 'as'; encode returns a fresh exactly-sized array and never exposes its scratch buffer"
    not: "dictionary key order by encoded key bytes; sorting StringName directly; owner-side canonical_bytes writers; Object/Callable/RID/Color in a payload; 'as' cast on a Variant"
    adr: docs/architecture/adr-0005-determinism-keyed-rng-canonical-encoding-and-hashing.md
    reason: "Equal states must give equal bytes and unequal states unequal bytes; value order is cheap in GDScript and readable in the G4 diff; StringName.sort() is not lexical and not a function of text (probed); a Dictionary merges String and StringName keys of equal text"
    referenced_by:
      - docs/architecture/adr-0005-determinism-keyed-rng-canonical-encoding-and-hashing.md
    added: 2026-10-28
    revised: ""
  - purpose: integer_division_convention
    status: active           # ADR-0005 is Proposed; ADR-0010 owns the lint rules and the single annotated division
    api: "IntMath (this ADR owns the function list): trunc_div = a / b (the one annotated site), floor_div = trunc_div(a - posmod(a, b), b), ceil_div = floor_div(a + b - 1, b), posmod_i = the posmod builtin, lsr(x, n) for shifts of possibly negative values (1 <= n <= 63), low32; b <= 0 returns 0 with a debug assert; floor family for every quotient meant as a floor, trunc_div only where truncation toward zero is meant"
    not: "bare / or % or /= or %= on ints outside IntMath; arithmetic >> on a possibly negative value; hexadecimal literals above 0x7FFFFFFFFFFFFFFF; decimal literals above 9223372036854775807 (INT64_MIN is -9223372036854775807 - 1 or 1 << 63); shifting a negative literal; shift counts outside 1..63; reliance on integer wrap outside mix64/fnv64/Ihash/the fold"
    adr: docs/architecture/adr-0005-determinism-keyed-rng-canonical-encoding-and-hashing.md
    reason: "GDScript / and % truncate toward zero (probed -7/2 = -3); >> is arithmetic; the integer_division warning covers the binary / only (c /= b loads, probed) and no warning covers % or %=; a hex literal above int64 logs an error and saturates while the script still loads; a decimal literal just above int64 wraps silently; int wraps silently"
    referenced_by:
      - docs/architecture/adr-0005-determinism-keyed-rng-canonical-encoding-and-hashing.md
    added: 2026-10-28
    revised: ""
  - purpose: keyed_rng_derivation
    status: active           # ADR-0005 is Proposed; D.10 set-once (R-17)
    api: "rng_derive(world_seed, stream, counter) = mix64(world_seed ^ mix64(fnv64(name)) ^ (counter * GOLDEN)) with signed decimal constants; stream key computed once at registration; rng_derive_epoch adds RESTORE_KEY_STRIDE (2^20) * restore_epoch inside RngFacet.draw_epoch only; Ihash.h(variation_seed, k1, k2, k3, salt) for non-RNG variation and for truth-channel visuals that must be reproducible (Trail: world_seed as k3, fixed per-system seed)"
    not: "randi/randf/RandomNumberGenerator/shuffle/pick_random/hash() in sim; a sequential counter shared across unrelated subjects; owner code computing counter_eff; presentation calling rng_derive or DetRng.derive"
    adr: docs/architecture/adr-0005-determinism-keyed-rng-canonical-encoding-and-hashing.md
    reason: "Pure keyed draws replay from (seed, name, counter) with no hidden state; the D.10 goldens reproduce on 4.7.2 and in an independent Python reference; the sim never draws the trail.vis stream, so a world_seed-keyed Ihash reproduces it without opening the sim RNG API to presentation"
    referenced_by:
      - docs/architecture/adr-0005-determinism-keyed-rng-canonical-encoding-and-hashing.md
    added: 2026-10-28
    revised: ""
```

### forbidden_patterns
```yaml
  - pattern: owner_canonical_byte_writer
    status: active           # ADR-0005 is Proposed
    description: "Canonical byte writers (encode_s64, encode_u32, encode_double, to_byte_array) and hashing of a var_to_bytes result are forbidden in src/kernel/, src/sim/ and block owners; owners deliver a plain Dictionary/Array/packed-array payload through snapshot(ctx) and the CanonicalEncoder encodes it. They are allowed in src/shared/det/ and tests; src/save/ may use them for container byte layout, slot_epoch.bin and integrity hashes (payload_sha256, trailer) whose output never feeds H64 or state_hash."
    why: "One injectivity argument, one set of golden vectors, one diff tool; per-owner layouts diverge on float, string and tombstone handling. Container layout and integrity hashing are not state hashing."
    adr: docs/architecture/adr-0005-determinism-keyed-rng-canonical-encoding-and-hashing.md
    added: 2026-10-28
  - pattern: stringname_sort_for_canonical_order
    status: active           # ADR-0005 is Proposed
    description: "StringName is never passed to Array.sort() or compared for order in simulation, kernel or hashing code; convert with String(k) first. sort_custom, and any sort over keys that can tie, is banned in sim: pack the key and a stable id into one int and sort the ints."
    why: "StringName.sort() is not lexical and not a function of text (probed 2026-10-28: [&b,&a,&c,&aa,&B] sorted to [&c,&a,&b,&B,&aa]; intern-order based, may change between runs); the sort is not stable."
    adr: docs/architecture/adr-0005-determinism-keyed-rng-canonical-encoding-and-hashing.md
    added: 2026-10-28
  - pattern: global_rng_or_engine_hash_in_sim
    status: active           # ADR-0005 is Proposed; the token list itself is ADR-0010 SIM-02 and the DET rules
    description: "No global RNG token (randi, randf, randf_range, randi_range, randomize, seed(, RandomNumberGenerator, shuffle, pick_random), hash(), get_instance_id() or ResourceUID in src/sim, src/kernel or any presentation file whose output a save or test can observe; randomness is RngFacet streams, variation is Ihash. Iterating a Dictionary without sorted keys is banned."
    why: "Global generators and engine hashes are not a stable contract and depend on call order; unsorted iteration depends on insertion history."
    adr: docs/architecture/adr-0005-determinism-keyed-rng-canonical-encoding-and-hashing.md
    added: 2026-10-28
  - pattern: restore_epoch_outside_draw_epoch
    status: active           # ADR-0005 is Proposed
    description: "restore_epoch (owned by Save/Load, ADR-0006; a read-only non-persisted non-hashed copy sits in kernel.clock) is read only inside RngFacet.draw_epoch; no block stores it, no canonical byte string or hash includes it, and no owner computes rng_counter + RESTORE_KEY_STRIDE * restore_epoch. The Graph's draws and every persisted counter use plain draw."
    why: "Keeps G4 defined on (save bytes, restore_epoch), makes the threat re-keying impossible to misapply (OQ-6), and keeps quit and death from rerolling physical outcomes."
    adr: docs/architecture/adr-0005-determinism-keyed-rng-canonical-encoding-and-hashing.md
    added: 2026-10-28
  - pattern: presentation_calls_det_rng_derive
    status: active           # ADR-0005 is Proposed; amends ADR-0009 line 137 (TD-28)
    description: "Presentation never calls DetRng.derive or any rng_derive form. A truth-channel visual that must be reproducible (the Failure & Trail trail.vis variant, phase and flip) uses Ihash keyed by world_seed and a fixed per-system seed; other presentation variation uses Ihash or Ihash.h32 with a fixed non-zero seed."
    why: "The simulation never draws the trail.vis stream, so Ihash reproduces it exactly without opening the sim RNG API to presentation; a lint row allowing .vis names was rejected."
    adr: docs/architecture/adr-0005-determinism-keyed-rng-canonical-encoding-and-hashing.md
    added: 2026-10-28
  - pattern: utility_function_names_and_variant_cast
    status: active           # ADR-0005 is Proposed; enforced by engine SHADOWED_* and UNSAFE_CAST at error (ADR-0010 5.1)
    description: "No member, parameter or local is named after a GlobalScope or GDScript utility function (seed, hash, min, max, floor, round, sign, range, len, str, type) or after a sibling member or function; no 'as' cast of a Variant (unpack with typeof plus a typed assignment, Array[T].assign(d.keys()), only in the allow-listed canonical_encoder.gd)."
    why: "SHADOWED_VARIABLE, SHADOWED_VARIABLE_BASE_CLASS, SHADOWED_GLOBAL_IDENTIFIER and UNSAFE_CAST make these load errors under the project warning gate (probed 2026-10-28)."
    adr: docs/architecture/adr-0005-determinism-keyed-rng-canonical-encoding-and-hashing.md
    added: 2026-10-28
```

### performance_budgets
```yaml
  - system: block_hash_sha256_150kb
    status: active           # ADR-0005 is Proposed; probed 0.33 ms on the dev machine editor binary, release-export RM-DECK measurement pending (S0-d)
    budget_ms: 1.0
    adr: docs/architecture/adr-0005-determinism-keyed-rng-canonical-encoding-and-hashing.md
    notes: "HashingContext SHA-256 over about 150 KB of canonical bytes (TR-lsg-037); a quiet tick performs 0 canonical encodes; a cached read after a tick that touched k blocks performs k encodes; the encoder cost for Dictionary-heavy blocks is unmeasured (S0-e)"
    referenced_by:
      - docs/architecture/adr-0005-determinism-keyed-rng-canonical-encoding-and-hashing.md
    added: 2026-10-28
    revised: ""
```

### referenced_by_update (existing entries that gain this ADR in referenced_by)
- payload_size_check (api_decisions, ADR-0002): add docs/architecture/adr-0005-determinism-keyed-rng-canonical-encoding-and-hashing.md to referenced_by (ADR-0005 repeats 'var_to_bytes bytes are never hash input' as canonical_encoding / state_hash_function not-lists)
- write_outside_writescope_or_setter_dirty (forbidden_patterns, ADR-0003): add docs/architecture/adr-0005-determinism-keyed-rng-canonical-encoding-and-hashing.md to referenced_by (the hash cache is validated by the touch serial advanced by open_for, never by a setter; a write outside a scope is the stale-cache case)
- fault_domain_table (state_ownership, ADR-0003): add docs/architecture/adr-0005-determinism-keyed-rng-canonical-encoding-and-hashing.md to referenced_by (duplicate RNG stream name or key collision uses domain-0 DUPLICATE_ID code 8; encoder errors in a capture use CAPTURE_ERROR code 7; no new domain-0 codes)
- restore_epoch (state_ownership, ADR-0006 candidate; owner Save/Load, persisted in slot_epoch.bin): add docs/architecture/adr-0005-determinism-keyed-rng-canonical-encoding-and-hashing.md to referenced_by (kernel.clock holds a read-only, non-persisted, non-hashed copy read only inside RngFacet.draw_epoch; ADR-0005 references the entry and does not duplicate it)

## Review Log
- **2026-10-28 godot-specialist (engine): verdict BLOCKING, 13 findings.** All applied except E13, which is informational. The author re-probed the load-bearing claims on the same 4.7.2 binary (P2, P6, P12, P14 amended; P17 to P22 added).
  - **E1 (high, BLOCKING) applied.** `Ihash.h` / `h32` parameter `seed` renamed `variation_seed`; `BlockHashResult.hash` renamed `h64`; `DetRng.derive` parameter `stream_key` renamed `key`; every prose and formula mention updated; the utility-name rule and the LINT-SPIKE-1 load-all check added to Implementation Guidelines; the failures reproduced as P20.
  - **E2 (high, BLOCKING) applied.** New § 3.8: `canonical_encoder.gd` is the second `float_allowlist.json` file and an entry on `dictionary_variant.json`; the unpacking idiom (`typeof` then typed assignment, never `as`; `Array[T].assign(d.keys())`); `unit` and `int_in` rewritten without `float(` (promotion). The finding's UNVERIFIED point (no warning for the promoted forms) is now verified by P20 with all 48 warnings at 2. The `@warning_ignore("unsafe_cast")` fallback is kept as § 3.8 (d).
  - **E3 (medium) applied.** `/=` and `%=` added to the lexical bans (§ 1.4, § 1.7) with bad fixtures; P6 and P19 state that the engine gate covers the binary `/` only.
  - **E4 (medium) applied.** § 1.3 and the guidelines ban decimal literals above 9223372036854775807 and give the INT64_MIN spellings; P18 recorded; P2 re-recorded with the value actually held. The author found more than the finding stated: a hex literal above INT64_MAX saturates to INT64_MAX and the script still loads (so a load check cannot catch it), and decimals above about 2^64 also saturate with a logged error.
  - **E5 (low) applied.** P12 and § 1.6 say `StringName` ordering is not a function of text and may change between runs; § 3.3 keeps `ENC_DUP_KEY` as defence only and states it is unreachable from a `Dictionary` (P22); § 6 and Validation replace its fixture with the `String`-keys versus `StringName`-keys identity test.
  - **E6 (low) applied.** § 4.8 and § 1.5: every JSON-sourced number is a `float` (P21); readers cast through `IntField.read` / `Hex64.decode`, reject non-integral or `> 2^53` values, and never pass a parsed `Variant` into a block; ADR-0006 header decode cross-referenced.
  - **E7 (medium) applied.** § 1.5 adds the `2^31` limit for `Vector2i` and `PackedInt32Array`, per-owner boot check or unit test, P17 probe (reproduced) and a Validation Criterion.
  - **E8 (low) applied.** Verification Required: S0-b on the exact shipping templates, on Windows, Linux x86-64 and arm64 / Deck; P1, P3, P4 recorded as defined only for runtime counts 0..63 and non-negative-literal folds.
  - **E9 (low) applied.** Post-Cutoff row corrected (`@warning_ignore_start` / `_restore` are 4.3); P6 points to ADR-0010 P2 and P3 for `directory_rules`; Alternative 2 cites ADR-0006's `var_to_bytes` int64 probe.
  - **E10 (low) applied.** Engine Compatibility reworded to owner overrides of the ADR-0003 callables and `_to_string`; the claim about `DetRng` / `IntMath` dropped; guideline added.
  - **E11 (low) applied.** `Hex64Result` declared; `floor_div`, `ceil_div` and `posmod_i` aligned to the ADR-0010 § 5.4 bodies (the `posmod` builtin) and P14 re-measured (0.20 microseconds, about 12 times a bare operator, not 23); S0-e records the `resize` / `slice` questions (still UNVERIFIED).
  - **E12 (low) applied.** § 3.6 and § 5.6: `encode` returns a fresh array, the scratch buffer is never exposed, worker arrays are `duplicate()`s owned by the job; the G3 worker variant added to § 6.
  - **E13 (info) no change.** Plausibility confirmed; P2 and P14 re-runs on the release template are listed under Verification Required.
- **2026-10-28 technical-director (TD-ADR): verdict CONCERNS, 11 findings.**
  - **T1 (high, BLOCKING) applied.** One location (`src/shared/det/`); byte writers allowed there (§ 1.7); the encoder's float handling confined to `_encode_float` and both float files named; `HashingContext` permitted in `hash64.gd` and `state_hasher.gd`; word boundaries required on the lexical tokens; the matching ADR-0010 amendments listed under Related.
  - **T2 (high, BLOCKING) applied** by ruling TD-28: the ban on presentation `DetRng.derive` stands; the Trail uses `Ihash` keyed by `world_seed` and a fixed per-system seed; Failure & Trail row in § 2.7; TR-ftrail-044 reworded; FT-20 / D.6 / AC-134 and ADR-0009 line 137 sync items. Alternative (b), a lint row for `.vis` names, was rejected.
  - **T3 (medium, BLOCKING) applied.** Depends On rewritten (ADR-0003 owns `ModuleFacet.rng`, `has_opened`, the `kernel.clock` view and the second `SnapshotContext`; ADR-0001 cited for injection only); `block_index` defined once in § 5.1 with the id-to-index map; cross-ADR amendments listed under Related.
  - **T4 (medium, BLOCKING) applied.** Acceptance gate restated without Graph code; Graph golden files moved to the Graph epic; macOS advisory; S0-e and the vector job to be added to `architecture.md` § Spikes with the user's approval.
  - **T5 (medium) applied.** § 4.1 states that no `H64` value is reserved; § 4.7 and Related name the snapshot outcome fields.
  - **T6 (low) applied.** § 5.2 reason corrected; § 2.4 now says the epoch is in no kernel snapshot payload and no `H64` input, and names ADR-0006's header recovery copy as file metadata.
  - **T7 (low) applied.** Key Interfaces declares `Hex64Result`, `CanonicalEncoder.sorted_int_keys` / `sorted_string_keys` (typed, because bare `Array` is banned in `src/shared`), `StateHasher`, `BlockHashCache` with `entry_valid`, and `RngRegistry`.
  - **T8 (low) applied** through ruling TD-11 (which supersedes the finding's `Clock.now_us()` wording): render timers use `delta` or `present_time`, with no `Time.*` token and no separate allowance.
  - **T9 (low) applied.** ADR-0010 moved to Depends On; ADR-0013, 0017, 0019, 0024 and 0029 added to Enables; ADR-0004 is not added to Enables because it is a dependency; the sibling-owned edits are listed under Related.
  - **T10 (low) applied.** The five secondary rows are noted in GDD Requirements Addressed for `/architecture-review`; moving TR-ftrail-044 off Gap is left to it.
  - **T11 (low) partly applied.** Registry candidates returned (Registry Candidates section; the `restore_epoch` entry is ADR-0006's per TD-33 and is only referenced). **Declined:** moving the probe record and vector tables to a companion file, because this revision may edit only this file and the ADR is the single record the reviewers and the control-manifest extraction read.
- **Cross-ADR rulings.** Owned and applied: TD-12 (`src/shared/det/`, `IntMath` list owned here, byte-writer ban scoped to state hashing; § 1.7), TD-28 (Trail via `Ihash`; § 2.6, guidelines), TD-32 (active scope is one not CLOSED; `block_hash_any` and resolution of former open question 11; § 5.3). Aligned with rulings owned elsewhere: TD-05 (ADR-0002 snapshot outcome and `has_expected_hash`; § 4.1, § 4.7), TD-07 (ADR-0003 batch, `kernel.clock` attribution; Depends On, Related), TD-11 (WallClock; § 1.7 row 1), TD-13 (two float files, `Variant` allow-list, `HashingContext` file list, strict-mode wording; § 1.7, § 3.8), TD-14 (unified token list, `/=` / `%=` and over-range literals closed lexically, `INTEGER_DIVISION` correction; § 1.4, § 1.7), TD-23 (ADR-0008 coordinate rule cited in TR-shipbody-038), TD-24 (capture always uncached, rung 1 deferred and off by default, `block_state_hash` naming; § 5.4), TD-25 (ADR-0006 uses `Hex64`; § 4.2), TD-27 (Depends On and Enables edges), TD-29 (`Ihash` overflow waiver, PF D.14 left to ADR-0024; § 2.6), TD-31 (frame-end sort: no change here), TD-33 (Registry Candidates section; `restore_epoch` single owner ADR-0006).
- **Open questions removed:** 2, 10, 11, 12 (answered by TD-29, TD-24, TD-32 and TD-13). Added: 13, 14, 15.
- **Status stays Proposed.** The ADR was revised in autonomous mode and the user has not reviewed it. The cross-ADR amendments under Related, the LINT-SPIKE-1 load-all check, S0-a to S0-e on a release export and the vector job must land before it can be Accepted. This log should be extended, not replaced, when further reviews run.
- **2026-10-28 /architecture-review fixes.** The soft ADR-0010 reference moved from Depends On to its own Soft references row (dependency-graph cycle through a soft edge). No change to the acceptance gate.
