# Phase 4 cross-ADR conflict detection (read-only; checked against working copy 2026-10-28)

## A. Residuals

Tally: 2 FIXED, 11 STILL_OPEN (all STILL_OPEN items are LOW/MEDIUM wording or allow-list gaps; none blocks acceptance on its own).

1. **FIXED** (ADR-0001/0002/0003/0004/0009, docs/registry/architecture.yaml). The registry was merged after the run-2 check.
   - control_plane (yaml:293) reads `request_restore(payload: Dictionary, restore_epoch: int, expected_state_hash: int = 0, has_expected_hash: bool = false)`, matching ADR-0002:203.
   - frame_order (yaml:728) reads "INPUT_ADAPTER -100, TICK_DRIVER -50, HOST_SERVICE -40 (SaveHost only), PRESENTATION 0".
   - presentation_clock (yaml:308) has `state: int` and `load_seq: int`.
   - block_registration (yaml:344) has `register_key_space(KeySpaceDescriptor)` and "a host service that owns no block receives HostRegistrarFacet with register_fault_domain only".
   - owner_callable_contract (yaml:327) reads "subscriber and result handlers (record_or_batch, NotifyContext)". That is the subscriber form in generic wording rather than literally `(batch: ChangeBatch, ctx) -> CompletionToken`. It is acceptable.
2. **STILL_OPEN** (ADR-0005 vs ADR-0010). ADR-0010 section 5.3 (line 233) was amended to `sorted_int_keys` / `sorted_string_keys` in `canonical_encoder.gd`. Three stale spots contradict it:
   - ADR-0010:292 SIM-09 row: "allow-list entries `dictionary_variant.json` ... and `sorted_keys.gd` (bare `Array`, `.keys()`)".
   - ADR-0010:311 DET row: "`sorted_int_keys`/`sorted_text_keys` of section 5.3".
   - ADR-0010:679 OQ-N(2) still asks ADR-0005 to "Name `sorted_int_keys` and `sorted_text_keys`".
   - ADR-0010 is therefore internally inconsistent on the file and the name. Severity MEDIUM, because the lint config will be built from the SIM-09 row.
3. **FIXED** (ADR-0005 vs ADR-0010). ADR-0010:227 now allow-lists "ADR-0005's named Variant pass-through signatures `Hash64.of_value(value: Variant)` in `src/shared/det/hash64.gd` and `canonical_encode(value: Variant)`, ... by file and function name" in `dictionary_variant.json`. Cosmetic leftovers: the SIM-09 row (ADR-0010:292) does not mention them, and ADR-0005:700 OQ-13 still shows as open.
4. **STILL_OPEN** (ADR-0006 vs ADR-0003). ADR-0006:212 `SaveService.setup(control, clock, wall_clock, block_versions, knobs)` has no registrar facet. ADR-0006:422 still says "Registering fault domain 1 from a module that registers no block is outside ADR-0003's stated examples". ADR-0003:331 provides the BOOTING-only `register_fault_domain` host facet. Nothing wires it to SaveHost or SaveService.
5. **STILL_OPEN** (ADR-0008 vs ADR-0009). ADR-0009:430 (Alternative 4) says "the snap flags snap in base-pixel space only under ADR-0008's pass B variant". ADR-0008:132 and :854 (E8) say "Snap flags snap in base-pixel space in both passes".
6. **STILL_OPEN** (ADR-0009 vs ADR-0004). ADR-0009:124 comment says "forwards to NotifyFacet.subscribe(cls, block_id, handler, name)". ADR-0004:133 has `subscribe(block_id, handler, name)` with the class fixed at issue.
7. **STILL_OPEN** (ADR-0010 vs ADR-0009). PRES-03 (ADR-0010:314) lacks `set_transform(`, `set_global_transform(`, `set("position")`, `global_transform =` and `Sprite2D.offset`. ADR-0009:305 requires the lint to "cover the whole engine API surface" including those names.
8. **STILL_OPEN** (ADR-0010, LOW). ADR-0010:137 lists `NotifyFacet` among classes "These and only these kernel classes may be named from `src/presentation/`...". The next sentences restrict it to `src/game/` and `PresentationFacet`. The list is unchanged, so `edges.json` could be generated wrongly from it.
9. **STILL_OPEN** (informational, no conflict). `IntMath` stays at `src/shared/int_math.gd` (ADR-0010:93, :138). TD-12's literal wording puts all determinism code in `src/shared/det/`. Both ADRs agree, so only DET-01's path scope matters.
10. **STILL_OPEN** (Depends On / Enables symmetry, LOW).
    - ADR-0001:34 Enables omits ADR-0007 and ADR-0010.
    - ADR-0002:34 omits ADR-0004, ADR-0009 and ADR-0010.
    - ADR-0003:34 omits ADR-0007 and ADR-0010.
    - ADR-0004:35 Enables lists ADR-0006, while ADR-0006:37 says "Related, not a dependency".
    - Partly fixed: ADR-0007:33 now lists ADR-0005 in Depends On and ADR-0007:34 lists ADR-0006.
11. **STILL_OPEN** (ADR-0001, LOW). ADR-0001:27 Verification Required 1 says "adapter -> driver -> presentation". Line 84 and line 270 include SaveHost at HOST_SERVICE.
12. **STILL_OPEN** (ADR-0008 vs ADR-0009, LOW). ADR-0009:188 says PipelineRoot "owns a `PresentationDriver`". ADR-0008:241 says "The composition root constructs the `PresentationDriver` and injects it; `PipelineRoot` holds it".
13. **STILL_OPEN** (ADR-0010 PRES-01, LOW). The `_process` exemption set (PipelineRoot, INPUT_SURFACE, layer-3/4 UI) does not mention debug layer 5. ADR-0008:567 puts menu and debug layers (3 to 5) on engine time. Nothing in ADR-0009 says a DIAGNOSTIC presenter is exempt. Still UNVERIFIED as to intent.

## B. New conflicts

Most integration contracts agree across the ten ADRs. Compared and consistent:
- `ProcessPriority` values: -100, -50, -40 and 0 in ADRs 0001, 0004, 0006, 0008, 0009 and 0010.
- Fault domain ids: 0 kernel, 1 save_load, 2 content, 3 graph.
- `request_restore` / `request_new_game` signatures: ADR-0002 :203-205 vs ADR-0006.
- `WriteScope` / `BlockRegistrar` signatures: ADR-0003 vs ADR-0004 vs the registry.
- `restore_epoch` ownership: Save/Load owns it, with a read-only copy in `kernel.clock` owned by ADR-0003 (ADR-0003:283, ADR-0005:231, ADR-0006:193). `RESTORE_KEY_STRIDE` is 2^20 (ADR-0005:190), and 2^32 times 2^20 stays inside int64.
- `STEP_BUDGET_MS` semantics: ADR-0003:263.
- `HashingContext` file list: ADR-0005:173 vs ADR-0010 IO-01.

### Conflict: ADR-0006 vs ADR-0001
- **Type:** dependency / integration contract (phantom owner).
- **ADR-0006 claims:** line 36 Depends On lists "ADR-0001 (frame order, `ProcessPriority.HOST_SERVICE`, boundary service, quit transition ...)". Line 153 says "ADR-0001 owns the quit transition".
- **ADR-0001 claims:** no mention of quit, shutdown or window-close anywhere (grep for `quit|shutdown|close_request` is empty). It owns the boundary service and `HOST_SERVICE`. `request_shutdown` is an ADR-0002 control call (:205) and a trigger in ADR-0003's state machine (:269, `SHUTTING_DOWN`).
- **Impact:** ADR-0006's `SaveHost` window-close handling (:153) leans on a quit transition that ADR-0001 does not define. An implementer cannot find the kernel-side contract for "write a QUIT generation then quit" in the cited owner.
- **Severity:** MEDIUM.
- **Resolution 1:** Re-attribute the quit transition in ADR-0006 to ADR-0003 (state machine, `request_shutdown`) and ADR-0002 (control call), and drop "quit transition" from the ADR-0001 dependency text.
- **Resolution 2:** Add a short "Quit / shutdown" paragraph to ADR-0001 (SaveHost owns `WM_CLOSE_REQUEST`, kernel reaches SHUTTING_DOWN only through `request_shutdown`) and keep ADR-0006's text.

### Conflict: ADR-0006 vs ADR-0008
- **Type:** performance budget (frame ceiling).
- **ADR-0006 claims:** line 554 Performance. Capture is bounded by `SNAPSHOT_BUDGET_MS` 16.7 ms ceiling, plus `SAVE_ENCODE_BUDGET_MS` 4. "One over-budget frame per autosave is accepted pending SAVE-WRK-1 f".
- **ADR-0008 claims:** line 296 "Total frame, CPU P99 ... at most 14.9 ms RM-DESKTOP / 30.0 ms RM-DECK ... at every ladder field". It carries no autosave exclusion.
- **Impact:** a capture frame can reach 16.7 ms plus the other work (sim up to 4 ms, pipeline 0.5 ms, lights 1.0 ms), above 14.9 ms. P99 is probably safe because autosaves are rare, but the table is unqualified, so a Pipeline perf test that includes an autosave could fail.
- **Severity:** LOW.
- **Resolution 1:** Add "excludes SAVING-state frames" to ADR-0008's 14.9 / 30.0 row.
- **Resolution 2:** Cap capture-frame cost (SAVE-WRK-1 f) so capture plus the normal frame fits 14.9 ms, or amortise capture over frames.

No HIGH conflicts found. Ten ADRs were scanned; the cross-ADR code blocks and constants above were compared. Open items are the residuals in A plus the two entries above. Not checked: the full text of every 100 KB+ ADR (grep sweeps and anchor sections only).

## C. Dependency cycle judgement

**Verdict: no real acceptance-order deadlock.** The cycles the script reports exist only through soft or post-acceptance edges. The hard acceptance prerequisites form a DAG.

Hard acceptance prerequisites quoted from each ADR:
- ADR-0001: Ordering Note (:36) "Do not Accept until ... stale tick-source text is fixed ... spike KRN-LAT-02 has run ... TD-ADR amendments re-reviewed". No ADR prerequisite. It calls its references to ADR-0010 "forward references ... not a dependency".
- ADR-0002: Depends On ADR-0001.
- ADR-0003: Depends On ADR-0001 and ADR-0002.
- ADR-0004: (:37) "Acceptance requires ADR-0001, 0002 and 0003 Accepted with the Cross-ADR amendments applied ... and KRN-LAT-02 passed". ADR-0010 is labelled "Soft dependency" (:34).
- ADR-0005: (:60) "ADR-0003, ADR-0004 and ADR-0001 must be Accepted before this ADR is Accepted". ADR-0010 is "a soft dependency: Proposed is enough". The gate is spikes S0-a to S0-e plus the LINT-SPIKE-1 load-all check, not ADR-0010's Accepted status.
- ADR-0006: (:39) "Acceptance order: ADR-0001, 0002, 0003, 0005, 0007 and 0010 Accepted, then SAVE-WRK-1-ACCEPT passed".
- ADR-0007: (:36) "Acceptance requires ADR-0001 and ADR-0003 Accepted and CONTENT-EXP-1 passed". ADR-0005 appears in Depends On (:33) as "golden content-hash vectors only, not an authoring dependency". Line 444 puts it under Validation Criteria as a post-acceptance step: "After ADR-0005 is Accepted (it depends on ADR-0005): golden `catalogue_hash` ... vectors are committed". Its wording "(it depends on ADR-0005)" is looser than the table.
- ADR-0008: (:56) "ADR-0001 Accepted including its amendment batch ... and (c) ADR-0007, ADR-0009 and ADR-0010 Accepted or co-accepted".
- ADR-0009: (:36) "Acceptance requires ADR-0001, 0002, 0003, 0004 and 0010 Accepted, and the order half of spike KRN-LAT-02 passed".
- ADR-0010: (:54) "Acceptance requires ADR-0001, ADR-0002, ADR-0003 and ADR-0007 Accepted and LINT-SPIKE-1 passed".

Reported cycles, resolved:
- ADR-0004 -> 0010 and ADR-0005 -> 0010 are labelled soft and are not acceptance prerequisites.
- ADR-0007 -> 0005 is a post-acceptance validation task and is not required by ADR-0007's own acceptance gate.
- With those three edges ignored, the hard order is: 0001 -> 0002 -> 0003 -> {0004, 0007} -> {0005, 0010} -> {0006, 0009} -> 0008 (0008 co-accepts with 0009). No ADR requires an ADR that requires it back.

Remaining risk is only mechanical. `adr-dep-graph.sh` parses the Depends On table and will keep reporting cycles. To clear it, either move ADR-0004's and ADR-0005's ADR-0010 reference and ADR-0007's ADR-0005 reference out of Depends On into a "Related / Validation follow-up" row, or teach the script to skip rows marked "soft". Also align ADR-0007:444 wording with its Depends On row.
