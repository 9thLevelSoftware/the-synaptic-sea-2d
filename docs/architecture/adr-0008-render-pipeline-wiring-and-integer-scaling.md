# ADR-0008: Render Pipeline Wiring and Integer Scaling

## Status
Proposed

## Date
2026-10-28

## Last Verified
2026-10-28 (against `design/gdd/pixel-perfect-render-pipeline-camera.md` PR-1 to PR-4, PR-6 C2, C3, C6, C11, PR-7, PR-9, PR-10, PR-12, PR-15, D.14, D.16, States and Transitions and the Performance budget block; `design/art/art-bible.md` 8.4, 8.6, 8.12; `design/gdd/2d-lighting-and-occlusion.md` LO-16 and its spike table; `docs/architecture/tr-baseline.md` rows citing ADR-0008; ADR-0001 § 2 and § 4; ADR-0007; ADR-0009; ADR-0010; and `docs/engine-reference/godot/` `VERSION.md`, `breaking-changes.md`, `deprecated-apis.md`, `modules/rendering.md`). Engine members, enums and default values were probed on the installed 4.7.2 editor on 2026-10-28 in two tiers: a headless tier (members, defaults and arithmetic; the headless run reports `opengl3` / `gl_compatibility` and never draws) and a **windowed tier run by the joint review's engine seat on Forward+ (Vulkan, one NVIDIA GPU, dev build) with Compatibility cross-checks where stated**, which settled the hook order, the HDR colour path, resize behaviour of a disabled viewport, the layer tie-break, draw-call accounting and the snap-flag space (all listed under Engine Compatibility). Still UNVERIFIED: a release export, Direct3D 12, the Steam Deck, a minimised or occluded window, `low_processor_mode`, textured and checker content through the scaled container, and every lighting claim (`Light2D` under an override, glow, debanding), which stay bound to the lighting spike (art bible 8.12 steps 1 to 4, 7 to 9) and pipeline spike steps 11 and 12. No code exists.

## Decision Makers
Claude (author), running under modes.automation: autonomous (project.yaml); decisions logged for the user's review; the user has not yet reviewed this ADR.

## Summary
The Pixel-Perfect Render Pipeline must draw a 640x360 base-pixel world at integer scale `k` (and at integer zoom steps `k_z`) next to a HUD at an independent integer scale `H`, in one window, from MVP (the Steam Deck is world 2x and HUD 3x), and three GDDs gate their wiring, scaling, camera hook and budgets on this decision. This ADR fixes the viewport stack as root stretch disabled with every scale chosen by script (a world `SubViewport` in a `SubViewportContainer`, then `CanvasLayer`s for hold, HUD, menus and debug), derives all geometry in one pure `RenderLayout` function written only by a single pipeline root, implements zoom as a change of logical size, builds render pass A at MVP with pass B specified behind one `ViewportWiring` constructor argument, keeps the world viewport SDR at MVP (`use_hdr_2d = false`, so palette fills read back exact; the HDR path is a specified, gated contingency), pins Forward+ and the Vulkan driver on Windows provisionally and explicitly with silent driver fallback disabled (reversible at the lighting spike), makes `PipelineRoot` the single `PresentationHost` that runs ADR-0009's driver (the camera is solved in the `CAMERA_SOLVE` stage of `_process`; the `frame_pre_draw` handler only writes the already-solved `canvas_transform` and parallax offsets), and carries the rules for the single draw-call and timing budget table.

## Engine Compatibility

| Field | Value |
|-------|-------|
| **Engine** | Godot 4.7.2 (stable, mono build installed), GDScript |
| **Domain** | Rendering (2D), with Core (frame hook) and UI (layer scaling) edges |
| **Knowledge Risk** | HIGH. 4.6 made glow run before tonemapping and (per the reference) made D3D12 the Windows default; 4.7 removed the antialiasing feather from `CanvasItem` lines, added HDR output, and changed the stretch defaults for new projects. The pinned reference has no content on `SubViewport` 2D sizing, `size_2d_override`, `size_2d_override_stretch`, `use_hdr_2d`, `Light2D` under those, or `frame_pre_draw`. |
| **References Consulted** | `docs/engine-reference/godot/VERSION.md`, `breaking-changes.md` (4.6 to 4.7 Rendering and Project rows; 4.5 to 4.6 Rendering rows), `deprecated-apis.md`, `modules/rendering.md`. `modules/ui.md` and `current-best-practices.md` were not read for this ADR (UI scaling is decided here from probes; dual focus is ADR-0030's). |
| **Post-Cutoff APIs Used** | None required by the decision. Relevant post-cutoff facts it accounts for: D3D12 as the Windows default (4.6), glow before tonemapping (4.6), line drawing without a feather (4.7), HDR output flag (4.7; kept off), new-project stretch defaults `canvas_items` / `expand` (4.7; not relied on), Shader Baker (4.5; export option). |
| **Probed 2026-10-28 (installed 4.7.2: headless tier for members and arithmetic; windowed tier for drawing and hook order)** | See the list below the table. |
| **Verification Required** | (1) Pipeline spike step 11, narrowed to what is not yet probed: the wiring of § 2 draws the world at exactly `(k_z, k_z)` and the HUD at exactly `(H, H)` at every window in PR-1's table, read back by the chain in § 5.4, on a dev build and a release export, **with a textured sheet, a one-pixel checker, pass B (if built) and an odd window size** (a flat fill through a `stretch = false`, scale-3, NEAREST container already passes: exact colour, exactly 24 px wide, no bleed); the first frame after each resize shows no fractional transform; the letterbox residual of an awkward window (1365x768) is black; **pause, resize the window, confirm no black frame (the mandatory `UPDATE_ONCE`, § 2.2); pause with no resize, confirm the frame is held; HUD text sharpness at `H = 3`; a 2 s edge drag confirms the cadence of the modal resize loop**. (2) Pipeline spike step 12: the `frame_pre_draw` probe (all `_process`, then the handler, then the draw; the written transform is the one drawn) re-run on a **release export**, with a minimised window and under `low_processor_mode`; D3D12 and the Steam Deck stay UNVERIFIED until then. (3) Lighting spike steps 1, 2 and 9: printed driver and method, pass A vs pass B on the 16.6 ms criteria, Compatibility parity; these gate § 1 and § 4. **The shadow-origin probe pack SO-P1 to SO-P10 (ADR-0018 § 8.2, retained under `production/qa/evidence/`) is one of the correctness checks these steps consult** when the renderer or driver is chosen (including the tie-break "Vulkan fails a correctness check D3D12 passes"), so the same retained evidence serves both ADRs; steps 1, 2 and 9 also run its half-cell lattice scenes (SO-P6) and its light-position snap check under pass B (SO-P5, with the project snap settings and the `SubViewport` flags exactly as § 2.2 ships them, in the shipped `PipelineRoot` tree). (4) Lighting spike steps 7 and 8: whether the ambient floor and glow the Lighting GDD needs can be met with the world viewport SDR; **only if not**, the HDR contingency of § 2.4 (resolve material) is adopted by amending this ADR, and `use_debanding` stays off unless flat palette fills read back bit-exact with it on. (5) Whether the project `default_texture_filter` reaches the world `SubViewport` (probed: the `SubViewport` members default independently, so the pipeline sets them explicitly regardless); the snap flags are probed (base-pixel space, below). (6) The windowed palette read-back test of § 5.5(d), per renderer (Forward+ and Compatibility): a flat fill in, the same fill out. (7) Windows display scaling at 125 % and 150 % with `allow_hidpi` true (default, probed): the window pixel size the pipeline reads is physical pixels. (8) Release-export draw-call counts per viewport (`Viewport.get_render_info`; the root count includes the container composite) and the P99 table of § 7 on RM-DESKTOP and RM-DECK. (9) **`get_render_info` canvas draw calls are confirmed populated on a release export (RM-DESKTOP and RM-DECK)**, not only on the dev build where they were probed (§ 7); until confirmed, the `D_ysort` trigger and every per-viewport figure of § 7.1 are unmeasured (ADR-0034 SP-YSORT-1 depends on this). **Every draw-call harness waits at least 2 frames after any show, hide or reparent** before it reads a count, because a visibility change or reparent rebuilds the affected layers and mixes rebuild frames with steady state (ADR-0034 § 10); the one-frame lag of `DRAW_CALLS_IN_FRAME` (§ 7) is in addition. (10) **The lighting probe pack (ADR-0018 § 8.2) is rerun on any engine patch bump or any change to a snap setting** (`rendering/2d/snap/*` or the world viewport's snap members), because § 2.2 depends on a probed absence of `Light2D` snapping. |

**Probed 2026-10-28 on the installed 4.7.2 editor (headless; transforms, sizes, members and defaults only):**
- **Built-in defaults.** `display/window/stretch/mode = disabled`, `aspect = keep`, `scale_mode = fractional` (so a project that writes none of them runs at 1152x648 unscaled, not at the `canvas_items` / `expand` the 4.7 migration row describes for new projects, which is an editor project-template behaviour); `display/window/size/viewport_width/height = 1152 / 648`; `window_width_override` and `window_height_override` = 0; `rendering/rendering_device/driver.windows = vulkan` (the engine built-in; this conflicts with the reference's "D3D12 is the Windows default", see Open Question OQ-C); `rendering/renderer/rendering_method = forward_plus`; `rendering/textures/canvas_textures/default_texture_filter = 1` (linear); `rendering/2d/snap/snap_2d_transforms_to_pixel` and `snap_2d_vertices_to_pixel = false`; `rendering/environment/defaults/default_clear_color = (0.3, 0.3, 0.3, 1)`; `rendering/viewport/hdr_2d = false`; `rendering/anti_aliasing/quality/use_debanding = false`; `physics/common/physics_interpolation = false`; `display/window/vsync/vsync_mode = 1` (enabled); `application/run/low_processor_mode = false`; `display/window/dpi/allow_hidpi = true`; `gui/common/snap_controls_to_pixels = true`.
- **Key names the GDD called NOT SOURCEABLE.** The HDR-output flag is `display/window/hdr/request_hdr_output` (default `false`); per-pixel window transparency is `display/window/per_pixel_transparency/allowed` (default `false`); debanding is `rendering/anti_aliasing/quality/use_debanding`.
- **`SubViewport` defaults.** `size` 512x512; `size_2d_override` (0,0); `size_2d_override_stretch` false; `use_hdr_2d` false; `use_debanding` false; `canvas_item_default_texture_filter` 1 (`Viewport.DEFAULT_CANVAS_ITEM_TEXTURE_FILTER_NEAREST` is 0); `snap_2d_transforms_to_pixel` and `snap_2d_vertices_to_pixel` false; `transparent_bg` false; `render_target_update_mode` 2 (`UPDATE_WHEN_VISIBLE`; `UPDATE_DISABLED` 0, `UPDATE_ONCE` 1, `UPDATE_ALWAYS` 4); `handle_input_locally` true.
- **`SubViewportContainer` defaults.** `stretch` false; `stretch_shrink` 1; `mouse_target` false; `texture_filter` 0 (`PARENT_NODE`; `CanvasItem.TEXTURE_FILTER_NEAREST` is 1); `material` null. `CanvasLayer.layer` 1, `scale` (1,1), `transform` identity. `Camera2D.zoom` (1,1), `position_smoothing_enabled` false, `process_callback` 1 (IDLE), `ignore_rotation` true. `Window.min_size` can be set and read back (1280x720).
- **Sizing and transform arithmetic.** A container with `stretch = true`, `stretch_shrink = 3` and size 1920x1080 sizes its viewport 640x360; `stretch_shrink = 7` on 3840x2160 gives 548x308 (floor), so `stretch_shrink` cannot reach `picture / k_z` at every ladder step. A container with `stretch = false` takes its viewport's size (640x360) and a `scale` of (3,3) gives a `get_global_transform` scale of exactly 3. A `CanvasLayer.scale` of (3,3) gives a child `Control` a `get_global_transform_with_canvas` scale of exactly 3 with the root `get_final_transform` identity (H not equal to k works from script). Under pass B, a `SubViewport` of 1920x1080 with `size_2d_override` 640x360 and `size_2d_override_stretch = true` has a stretch and final transform scale of exactly (3,3); a viewport of 1366x768 with an override of 682x384 has (2.002933, 2.0): **the override must divide the viewport size exactly or the scale is not an integer**. With `size_2d_override_stretch = false` the stretch transform stays identity and only `get_visible_rect()` changes. Root `content_scale_mode = canvas_items`, `aspect = keep`, `stretch = integer`, `content_scale_size = (640,360)` gave `get_final_transform` scale (2,2) with offset (43,24) at 1366x768, (2,2) with (160,90) at 1600x900, (3,3) with (0,60) at 1920x1200, (3,3) with (320,0) at 2560x1080, (4,4) with (440,0) at 3440x1440, and `get_visible_rect` 640x360 in every case: root integer stretch is correct at the default step with H equal to k and nowhere else.
- **Process order.** `Node._process` ran in ascending `process_priority` (-100, -50, 0, 100, 1000) in one frame. This is the headless fact only; the release-export confirmation and the cross-node case with presentation are ADR-0001's spike KRN-LAT-02.
- **Not probed headless, and why.** `RenderingServer.frame_pre_draw` and `frame_post_draw` never fired in a four-frame headless run (no draw happens headless), and `get_current_rendering_driver_name()` / `get_current_rendering_method()` read `opengl3` and `gl_compatibility` headless, so **every headless probe above ran under the Compatibility renderer**: CPU-side arithmetic and member existence carry over, drawing behaviour does not. Only project-setting defaults are renderer-independent. The windowed tier below supersedes the headless tier for every drawing claim.

**Probed 2026-10-28 windowed (engine-seat review; 4.7.2 editor dev build; Forward+ on Vulkan, one NVIDIA GPU; Compatibility cross-checks where stated).** These are the evidence for § 2, § 5 and § 6; each cites its section.
- **HDR colour path (§ 2.2, § 2.4, § 5.1).** A `SubViewport` with `use_hdr_2d = true`, drawn through a `SubViewportContainer` onto the SDR root, outputs **unconverted linear colour**: a flat `Color8(128, 64, 200)` fill reads back as `(55, 13, 147)` from the HDR viewport and as `(128, 64, 200)` from an SDR viewport; clear-colour pixels read `(19, 19, 19)` instead of `76`. The engine converts linear to sRGB only when a viewport is attached to the screen. A container `ShaderMaterial` doing the exact transfer function (`mix(pow(c, 1/2.4) * 1.055 - 0.055, c * 12.92, c <= 0.0031308)`) restores `(128, 64, 200)` exactly under Forward+ and is renderer-specific: under `--rendering-method gl_compatibility` `use_hdr_2d` is silently ignored (confirming the "Forward+ and Mobile only" claim), the plain viewport is exact, and the same shader double-converts to `(188, 137, 229)`. Values above 1.0 clamp at the container draw unless a resolve shader tonemaps them.
- **Hook order (§ 6.2).** `RenderingServer.frame_pre_draw` fires on the main thread after **every** node `_process` of the frame (including `process_priority` 1000) and before the draw, under `thread_model` 1 and 2; a `canvas_transform` written in the handler is the one drawn that same frame (the marker's x equalled the written value); a write from `_process` is also drawn that same frame. It fires only when a frame is actually drawn. Consequence: anything reading the camera pose during `_process` sees the previous frame's pose if the solve lives in the handler.
- **Resize of a disabled viewport (§ 2.2, § 3.2).** Resizing a `SubViewport` with `render_target_update_mode = UPDATE_DISABLED` **blanks it** (Forward+: `(0, 0, 0)` in the container; Compatibility: the clear colour). Setting `UPDATE_ONCE` redraws it and it is composited the same frame. After the one-shot the node's property still reads `UPDATE_ONCE` (the engine does not write it back); re-assigning `UPDATE_ONCE` on a later resize worked. Whether a disabled viewport holds its texture without a resize is not tested.
- **Driver and fallback keys (§ 1, § 5.1).** A project with no driver key reports `driver = vulkan` and `method = forward_plus` on Windows 4.7.2; the `ProjectSettings` default of `rendering/rendering_device/driver.windows` is `vulkan`, `driver.linuxbsd` `vulkan`, `driver.macos` `metal`. `rendering/rendering_device/fallback_to_opengl3` (default true), `fallback_to_d3d12` and `fallback_to_vulkan` (both default true) exist. The silent fallback sequence itself is not probed. All defaults listed under the headless tier were re-confirmed present with the stated values.
- **Layer tie-break (§ 2.5).** Two `CanvasLayer`s at the same `layer` index: the later sibling draws over the earlier (consistent with the sibling index as the sublayer key). Not tested across different parents. This ADR does not depend on it (§ 2).
- **Snap flags and units under pass B (§ 2.2, § 4).** On a 600x12 viewport with override 200x4, stretch true and `snap_2d_transforms_to_pixel = true`: `canvas_transform` is in override (base-pixel) units (an x offset of -5 moved content 15 screen px); snapping happens in base-pixel space (a node at 10.4 with offset -5 gives 5.4, snapped to 5, drawn at screen x = 15); rounding is to nearest, not floor. Single probe, Forward+.
- **Container and draw calls (§ 4, § 7).** A `SubViewportContainer` with `stretch = false`, `scale = (3, 3)`, `texture_filter = NEAREST` drew an 8x4 SDR viewport as exactly 24x12 screen px (x = 700..723), flat fill, exact colour, no bleed. `Viewport.get_render_info(RENDER_INFO_TYPE_CANVAS, RENDER_INFO_DRAW_CALLS_IN_FRAME)` works per viewport in Forward+ and Compatibility: a root with three `SubViewportContainer`s reported 3 canvas draw calls, so **the container composite is counted on the root, one call per container**; the sub-viewport's own count is separate. The signature and enum names in § 7 are correct.
- **Compile check.** The Key Interfaces of this ADR (as drafted before this revision) were compiled headless in a scratch project: no failure on 4.7.2; all 24 `class_name`s are free of `ClassDB` collisions; `layout`, `version` and `present_time` are not `Node` or `Object` members.

## ADR Dependencies

| Field | Value |
|-------|-------|
| **Depends On** | **ADR-0001** (the `ProcessPriority` constants `INPUT_ADAPTER`, `TICK_DRIVER`, `HOST_SERVICE`, `PRESENTATION`; `PresentationClock` with `tick_alpha`, `T`, `running`, and the read-only `state: int` and `load_seq: int` added by the ADR-0001 amendment batch; `snap_baseline` on entry to `RUNNING`; `present_time` ownership handed to the pipeline root; Proposed). **ADR-0007** (the tuning loader: `RenderTuning` is a `ScalarTable` with `SCALED` fields for `tau_follow`, `DT_MAX` and `ZOOM_STEP_MIN_INTERVAL`, plus a typed `ZOOM_MIN_K` row table, § 6.1; Proposed). **ADR-0009** (this ADR's `PipelineRoot` is the single `PresentationHost` and hosts the `PresentationDriver`, `PresentFrame`, `PresentationStage` and `Placement` of ADR-0009 § 3 and § 4; Proposed; ADR-0009 lists ADR-0008 under Enables and the two are accepted together). **ADR-0010** (paths under `src/presentation/pipeline/`, the lint packs and rule ids `PRJ-02` and `PRES-nn`, the `RenderConfigReader` cast pattern; Proposed). |
| **Enables** | ADR-0028 (lighting budget: the renderer, driver, SDR-or-HDR decision and the zoom-independent light rule), ADR-0030 (UI framework: HUD scale, the two menu layers, reading-face rasterisation; also the Chart Table store ADR that persists `DisplaySettings`), ADR-0034 (PROJECTOR: the camera limits rectangle is consumed here; Y-sort, z-bands and PROJECTOR shape are its own), ADR-0019 (Input Adapter: zoom actions call the pipeline, cursor mapping), every Pipeline, Lighting, HUD, Chart Table and Instruments story. |
| **Blocks** | Epics: Pixel-Perfect Render Pipeline & Camera (the GDD names this ADR the first implementation-story gate: `world_to_hud`, `RESCALING`, the `get_final_transform` criteria), 2D Lighting & Occlusion (renderer and driver), HUD & Suit Layer (layer 2 and H). |
| **Ordering Note** | **Do not Accept** until (a) the lighting spike has run steps 1, 2 and 9 (renderer, driver, pass, Compatibility parity) and pipeline spike steps 11 **and 12** have passed (step 12 includes the release-export re-run of the hook probe), because § 1, § 4 and § 6.2 are provisional until then; (b) ADR-0001 is Accepted **including its amendment batch** (`PresentationClock.state` and `load_seq`, `HOST_SERVICE`; until they exist `setup()` cannot read the kernel state and the CUT trigger has no source); and (c) ADR-0007, ADR-0009 and ADR-0010 are Accepted or co-accepted. The sibling ADRs are drafted, so the former forward references are real references: ADR-0009 owns what presentation may read and write and **placement only** (`Placement`, `PresentFrame`, the stage list, the `Callable` registration); **ADR-0034 owns the Y-sort, z-band and PROJECTOR-shape contract**; ADR-0010 owns the `src/` tree, the lint tool and which of § 5.5's rules are regex, boot-walk or runtime counters, and hosts `PRJ-02` as this ADR's § 5.5(a); ADR-0004 owns the dirty-key and notification transport (this ADR relies only on "a notification marks dirty, drawing happens later"); ADR-0007 owns the tuning-table loader that carries `RenderTuning` and `ZOOM_MIN_K`. **Display-settings persistence is not a dependency**: `DisplaySettings` is host-local Chart Table state (CT-19), persisted by the Chart Table's `SettingsStore` (ADR-0030 or whichever ADR takes CT ADR candidate (c)); it is not kernel state, not WORLD or PROFILE scope, and ADR-0006 is not involved. |

## Context

### Problem Statement
The game's whole visual promise is that base pixels are square, unblurred and never at a fractional transform, and that "when the ship is still, the screen is still". The Pipeline GDD states the rules and defers the wiring to this ADR in six places: how a world viewport reaches the window size (pass B), how it reaches `picture / k_z` (every zoom step), how the HUD draws at `H` different from `k` under one root (mandatory on the Steam Deck from MVP), where the container's material slot and the camera write live, what hook the camera writes in, and what the renderer and Windows driver are. The Lighting GDD and the architecture add two more gates: the renderer method (Forward+ versus Compatibility) and the Windows driver (Direct3D 12 versus Vulkan), and one table for the draw-call budget (architecture IC-6), which today exists only as per-system figures. Until this ADR lands no Pipeline implementation story can start, and Lighting, HUD and Chart Table each build against different assumptions about the viewport stack.

### Constraints
- **Integer scales only, pixel integrity first** (PR-1, PR-4): no scale anywhere in the world or the HUD is non-integer; the only non-integer text is the chart table's vector reading face (7.1).
- **Two integer scales in one window from MVP** (PR-12): Deck 1280x800 is world 2x, HUD 3x.
- **The simulation is untouched** (ADR-0001, kernel H9): nothing here reads or writes a state block; `k`, `H` and `k_z` are presentation state, never saved, never hashed.
- **One interpolator, no engine physics interpolation** (PR-4 rule 8): `physics_interpolation` stays off; the camera is a direct write, not a `Camera2D` scroll.
- **No autoload, no global** (registry `kernel_autoload_or_global`, ADR-0001): geometry reaches consumers through an injected object, not a singleton.
- **Headless CI draws nothing** (`commands.test` runs `--headless`): the geometry is a pure function testable without a window; pixel-level checks are windowed and manual with retained evidence until a windowed runner exists (Pipeline "Gaps", Open Question 22).
- **Reference machines and build** (`project.yaml` `qa.min_spec`, ruling R12): RM-DESKTOP is a GTX 1050 Ti / RX 570 class machine at 1920x1080, "Vulkan 1.2+", P99 CPU frame ceiling 16.6 ms; RM-DECK is a Steam Deck LCD at 1280x800 with a 30 fps target and a 33.3 ms ceiling; every wall-clock figure is a Godot 4.7.2 release export, renderer `forward_plus`, vsync on, 120-frame warm-up, P99 at most 90 % of the ceiling.
- **Engine reference silent** on `SubViewport` 2D sizing, override stretch, `use_hdr_2d`, `Light2D` under them and `frame_pre_draw`; the headless probes settle sizes and arithmetic, and the windowed probes under Engine Compatibility settle the hook order and several drawing behaviours (colour path, resize, tie-break, draw-call accounting).

### Requirements
- A viewport stack in which the world is the only lit layer and the HUD, menus, hold plate and debug overlay are structurally outside world light, glow, the Filter and the camera jolt.
- A single writer for every viewport size, scale, offset and the camera transform, and a single caller of `HudLayout`.
- One re-derivation path for every cause of change (window resize, display mode, zoom step, setting change), completed inside one frame.
- Zoom by logical size, with `Camera2D.zoom` and `Control.scale` pinned at (1,1) everywhere they could be tempting.
- A render-mode flag with identical logical output in both modes, and a coordinate, sort, camera and HUD contract that does not mention which mode is live.
- A renderer-and-driver decision that names its gate and its reversal cost.
- One draw-call and timing table that nests the per-system figures and says who owns each row.
- A startup check that fails loudly in a dev build on any wrong effective value, and that can run in CI without a window.

## Decision

Five numbered sub-decisions, each reviewable alone: **§ 1 renderer method and Windows driver**, **§ 2 the viewport stack and scale wiring**, **§ 3 geometry, `k`, `H`, aspect and zoom**, **§ 4 render mode A and B**, **§ 5 project pins and startup guard rails**. Two shorter ones follow because other ADRs cite them: **§ 6 the presentation host, the camera solve and write, and `present_time`**, and **§ 7 the draw-call and timing budget rules**.

### 1. Renderer method and Windows driver

**Decision (provisional, pinned, spike-reversible).**
- **Method: Forward+** (`rendering/renderer/rendering_method = "forward_plus"`, already in `project.godot`). Compatibility and Mobile are assumed nowhere.
- **Windows driver: Vulkan** (`rendering/rendering_device/driver.windows = "vulkan"`), and the Linux / Steam Deck driver is Vulkan (`driver.linuxbsd = "vulkan"`). Direct3D 12 is the recorded alternative.
- **The driver is never left to the engine default.** Windowed probe on 4.7.2: a project with no driver key reports `vulkan` / `forward_plus`, and the `ProjectSettings` default of `driver.windows` is `vulkan` (`driver.linuxbsd` `vulkan`, `driver.macos` `metal`); the reference's "D3D12 is the Windows default since 4.6" is therefore not what a key-absent project gets and is most plausibly a new-project-template value (UNVERIFIED, OQ-C). Either could be the value a later 4.7.x patch or a template re-save writes, and the editor drops a key whose value equals its default, so the key may be absent from `project.godot`. The pin is therefore an **effective-value assertion** (§ 5.3): CI asserts the effective `ProjectSettings` values (headless-safe); a dev build halts at boot when the live printed driver is not the expected one. A later choice of D3D12 is written explicitly and changes the expected value in one constant.
- **Silent fallback is disabled.** The engine has `rendering/rendering_device/fallback_to_opengl3` (default true), `fallback_to_d3d12` and `fallback_to_vulkan` (both default true). With Vulkan the pin, `fallback_to_opengl3 = false` and `fallback_to_d3d12 = false` are **required keys in `project.godot`** (their defaults are `true`, so they cannot be absent); `fallback_to_vulkan = false` is added if the D3D12 reversal is taken. A failed driver init then errors visibly instead of degrading to Compatibility, where `use_hdr_2d` is ignored, `Light2D` and glow behave differently and the colour path changes (§ 2.2). **A wrong effective driver or method in a release build is a visible fatal "unsupported GPU" screen** (text only: expected and actual method and driver, then exit on any key), written to the diagnostics header as well; it never logs and continues, because a release build that silently ships a different look is the failure this pin exists to prevent.

**Why Forward+.** The Lighting GDD's ambient floor, glow and `hdr_2d` model (steps 7 and 8 of the spike) assume a float 2D framebuffer and the 4.6 glow-before-tonemapping behaviour, and the art bible sets `hdr_2d` for exactly that reason. The claim that `use_hdr_2d` is a Forward+ and Mobile feature is now **CONFIRMED by probe** (Compatibility silently ignores it). At MVP the world viewport is nevertheless SDR (§ 2.2), because an HDR viewport drawn through a container on the SDR root outputs unconverted linear colour; Forward+ is kept for `Light2D` and glow parity with the reference machines and so that the gated HDR contingency of § 2.4 stays open. The reference cost of Forward+ is the Vulkan-class requirement, which both reference machines state (RM-DESKTOP "Vulkan 1.2+", the Deck). Compatibility remains the **reversible fallback** the Lighting GDD keeps open (step 9); a reversal re-opens the glow and ambient design, shader support for the five-material cap, the colour path (the HDR resolve is renderer-specific, § 2.4) and the pass-B figures of § 7, and re-runs the spike, so it is a decision for the technical director, not a settings flip.

**Why Vulkan on Windows.** (a) RM-DESKTOP is specified in Vulkan terms and RM-DECK is Vulkan, so one driver family produces the numbers the budget table is judged on and any driver-specific rendering difference (shader variants, glow, float framebuffers) is found once. (b) The reference gives D3D12's reason as "better driver compatibility", which concerns old and integrated GPUs below the minimum specification; it does not apply to a GTX 1050 Ti or RX 570 class card. (c) The installed binary's built-in default is Vulkan, and a key-absent project reports Vulkan (windowed probe). (d) The D3D12 backend is the newer one (UNVERIFIED stability on 4.7.2); the cost of choosing the older backend is nothing at this tier. **Tie-break rule recorded for the spike:** switch to D3D12 only if lighting spike steps 1 and 2 on RM-DESKTOP show Vulkan failing a correctness check (palette-exact fills, hard cone edge, glow) that D3D12 passes, or D3D12 beating Vulkan's P99 CPU frame time by 10 % or more on the same scene; record both numbers either way. HDR output is off (§ 5), so the one Windows feature that might favour D3D12 is not used at MVP.

**What gates it.** Lighting spike step 1 (print the driver name and rendering method and whether `hdr_2d` is honoured), step 2 (pass A/B), step 9 (Compatibility parity), pipeline spike step 11 and (for the release export) step 12. This section's status is therefore "provisional until those have run", independent of the rest of the ADR.

### 2. The viewport stack and scale wiring (wiring candidate (i), refined)

**Decision.** The root `Window` runs with **stretch disabled**. Every scale in the window is chosen by script and is an integer by construction. One node, the **`PipelineRoot`**, is the only writer of the world viewport's size, override, scale, offset (the world `canvas_transform` is written by the `CameraHost` handler it owns, § 6.2), of every `CanvasLayer` transform, and the only caller of `HudLayout`.

```
root Window  (content_scale_mode = DISABLED, content_scale_factor = 1.0, min_size = 1280x720, clear colour black)
└─ PipelineRoot (Node; process_priority = ProcessPriority.PRESENTATION; the single PresentationHost (ADR-0009 § 3);
   │              owns the PresentationDriver; sole writer of everything below)
   ├─ WorldContainer: SubViewportContainer          layer 0   top-level Control on the root canvas
   │    stretch = false · mouse_filter = IGNORE · mouse_target = false · texture_filter = TEXTURE_FILTER_NEAREST (1)
   │    position = world_rect.position (integer valued) · scale = (k_z, k_z) under pass A, (1, 1) under pass B
   │    material = null at MVP (the single Filter hook, § 2.4)
   │    └─ WorldViewport: SubViewport                     size, override, SDR, snap, filter: § 2.2
   │         └─ the body frame: floors, Y-sorted layer, caps, lights, occluders, fringe, dust, horrors, web, instruments
   │             (CameraHost's frame_pre_draw handler writes this viewport's canvas_transform, § 6.2)
   ├─ CameraHost: Node                                no geometry; holds the solved pose; its frame_pre_draw handler is the only camera write
   ├─ HoldLayer        CanvasLayer  layer = 1   one opaque #000000 quad, only in TRANSIT_HOLD and BOOTING / LOADING
   ├─ HudLayer         CanvasLayer  layer = 2   group "hud"; scale = (H, H); offset = hud_rect.position
   │    └─ HudRoot: Control, size = hud_rect.size / H (fractional allowed), Control.scale = (1, 1)
   ├─ MenuPixelLayer   CanvasLayer  layer = 3   scale = (H, H)   bitmap-face chart table and menus
   ├─ MenuReadingLayer CanvasLayer  layer = 4   scale = (1, 1)   reading-face text, sizes in screen px (§ 2.5)
   └─ DebugLayer       CanvasLayer  layer = 5   scale = (H, H)   dev builds only; absent from release exports
```

**2.1 Why this wiring (what the probes show).** Root integer stretch gave exactly `(k, k)` with the correct centring offset at every window tried, but only for the **default step with `H` equal to `k`**: its single scale applies to everything under the root, so a HUD at 3x over a 2x world, or a world at `picture / k_z`, would need a second, non-integer multiplier (the double-scale hazard of PR-7, candidate (ii)). With stretch disabled, a `CanvasLayer.scale` of (3,3) gave a child `Control` an exact scale of 3 with the root transform identity, and a `scale` of (3,3) on a `stretch = false` container gave an exact scale of 3; so the Deck's world 2x and HUD 3x are two independent integers chosen from script. `stretch_shrink` floors (3840/7 gives 548) and is an integer divisor of the container size, so candidate (iii) cannot reach `picture / k_z` at every step and is rejected as the primary; it remains the recorded fallback for the default step only (Alternative 2).

**2.2 The world viewport members the root sets explicitly (every value is the probed built-in default's opposite or must not be inherited).**
- `size`: under pass A `field_px` (the logical size in base px, § 3); under pass B `world_rect.size` (`field_px * k_z`). **Never the 512x512 default and never the window size.** A scene that finds `size == (512, 512)` fails the scene-tree lint.
- `size_2d_override` and `size_2d_override_stretch`: unset (0,0) and `false` under pass A; `field_px` and `true` under pass B. **The override must divide the size exactly** (probed: 1366x768 over 682x384 gives a scale of 2.002933), which § 3 guarantees by sizing the viewport as `field_px * k_z`.
- `use_hdr_2d = false` **at MVP** (the world viewport is SDR like the root). Windowed probe: an HDR viewport drawn through a container on the SDR root outputs unconverted linear colour (a flat `(128, 64, 200)` fill reads back `(55, 13, 147)`; the clear colour `(19, 19, 19)` instead of 76), which contradicts the palette-exact-fills goal, the container-material-null rule (§ 2.4) and `filter_route = PER_ELEMENT, slot = NOT_USED`. The flat-palette guarantee therefore holds only with the world SDR. The HDR path is a specified but unbuilt contingency (§ 2.4), adopted only if lighting spike steps 7 and 8 prove the ambient floor and glow cannot be met without it. Never set on the root viewport.
- `canvas_item_default_texture_filter = Viewport.DEFAULT_CANVAS_ITEM_TEXTURE_FILTER_NEAREST` (0); `use_debanding = false`; `transparent_bg = false`; `handle_input_locally = false` (the world has no `Control`; the effect of this member under a container that forwards no mouse events is UNVERIFIED, step 11, and the member may be left at its default if it proves irrelevant).
- `snap_2d_transforms_to_pixel = true`; `snap_2d_vertices_to_pixel = true` (belt and braces). **Probed (pass B, Forward+): the flags snap in base-pixel space, not screen-pixel space**, because `canvas_transform` is in override (base-pixel) units; they round to nearest rather than floor. Under pass A the viewport is the base-pixel field, so the same holds. Nothing relies on them: placement is by integer write (ADR-0009), and `pose_q` and the parallax offsets are still written as integer-valued `Vector2` so rounding and flooring never differ. This answers ADR-0009's question on the flags. **The snap settings and flags apply to `CanvasItem` draw transforms; they were probed not to snap `Light2D` positions on 4.7.2** (ADR-0018 P-5 and P-11; SO-P5 re-checks with the project settings in the shipped tree, including pass B). ADR-0018's fractional light position depends on that absence, so the lighting probe pack (ADR-0018 § 8.2) is rerun on any patch bump or snap-setting change (Verification Required (10)).
- `render_target_update_mode`: `UPDATE_ALWAYS` in `PRESENTING`, `TRANSIT_HOLD` with something to show and `RESCALING`; `UPDATE_DISABLED` in `PAUSED_FROZEN` and `HIDDEN`. **Probed: any size change while the viewport is `UPDATE_DISABLED` clears its target (Forward+ shows `(0, 0, 0)`, Compatibility the clear colour), so `UPDATE_ONCE` set on the frame of any `RESCALING` that happens while updates are disabled is mandatory, not a guard**; it redraws the target and the result is composited the same frame, so it is also sufficient. The engine does not write the mode back after the one-shot (the property still reads `UPDATE_ONCE`), so `PipelineRoot` sets `UPDATE_DISABLED` itself on the following frame and **`ViewportWiring` tracks the desired mode in its own field and never reads the property back**. Whether a disabled viewport holds its last texture when no resize occurs (the `PAUSED_FROZEN` hold) is UNVERIFIED but likely; step 11 checks both (pause + resize: no black frame; pause alone: frame held).
- **Nearest filtering is set in three places, with two differently numbered enums**: the project default (`default_texture_filter = 0`), the world viewport member above (`Viewport.DEFAULT_CANVAS_ITEM_TEXTURE_FILTER_NEAREST`, value 0), and `SubViewportContainer.texture_filter = CanvasItem.TEXTURE_FILTER_NEAREST` (value 1; the probed default is 0 = `PARENT_NODE`, which would inherit and blur the pass-A upscale). Mipmaps are off on every drawn sheet (art bible 8.4). `TileSet.tile_size` is 48x48 and set explicitly on every tile set (checked by ADR-0010's asset and scene lint, § 5.5(b), not at startup).

**2.3 Layers and what is structurally excluded.** The hold plate (1), HUD (2), menus (3 and 4) and debug overlay (5) are `CanvasLayer`s on the **root** viewport; world lights, glow, `CanvasModulate`, the Filter material on the container and the camera transform live in the world viewport or on the container below them and cannot reach them (PR-3, HR-1, HR-6, HR-13). Nothing that must mix with world light may live above layer 0 (it goes in the world); a `Control` found under the world `SubViewport` fails the scene-tree lint, which is a build gate (HUD HS-2). Layers 2 to 5 are **never dimmed, blurred or desaturated** by pause.

**2.4 The single full-screen hook, and the HDR contingency.** The `SubViewportContainer.material` slot is the pipeline's only full-screen hook (PR-10): assigned and nulled **only** by `PipelineRoot.set_filter_material(material: ShaderMaterial)`, colour-writing only (a policy the Filter owns; the pipeline cannot inspect a shader), drawn below every `CanvasLayer`, receiving the world at screen resolution under both passes. **At MVP the slot is null** (the world viewport is SDR, § 2.2) and the pipeline publishes `filter_route = PER_ELEMENT`, `slot = NOT_USED` as a normal state, not an error (TR-pxl-023, TR-percfilter-030). The truth-channel mask route is a VS-A option that this ADR does not pre-build.

**HDR contingency (specified, not built at MVP).** If lighting spike steps 7 and 8 show the ambient floor and glow need a float framebuffer, `use_hdr_2d = true` on the world viewport is adopted by amending this ADR, and the slot then stops being optional: a **mandatory `PipelineRoot`-owned resolve material** (exact linear-to-sRGB per the probed formula in Engine Compatibility, plus a stated clamp or tonemap policy for values above 1.0) is always in the slot. It composes with the Filter **by inclusion**: the Filter's shader includes the same resolve function, so there is one material and one full-screen pass, no second stage; `set_filter_material` refuses a material that lacks the resolve marker while HDR is on. The resolve is **gated on the effective rendering method**: none under Compatibility, which ignores `use_hdr_2d` (the same shader would double-convert, probed `(188, 137, 229)`). Adoption changes `D_filter` to 1 (not 0), makes `filter_route` / `slot` no longer `NOT_USED`, and extends `StartupCheck` with a resolve-present row; the palette read-back test (§ 5.5(d)) then runs against the resolved output on each renderer. Until adoption, `StartupCheck` asserts `use_hdr_2d == false` and `material == null`.

**2.5 Two integer scales and text.** The HUD canvas is `HudRoot`, a `Control` of size `hud_rect.size / H` at the HUD layer's offset; every HUD element anchors to it, `HudLayout` outputs integer HUD-px rectangles relative to it, and `world_to_hud` returns coordinates in its space (PR-12, HUD HS-9). `Control.scale` stays (1,1) on every HUD node and no node animates scale or rotation. **Reading-face text is never drawn under an integer-scaled layer**: the chart table's reading face lives on `MenuReadingLayer` (layer 4, scale (1,1)), whose labels set their font size to `round(base_size * scale_pct * window_scale)` and whose sizes are in screen px, while its pixel-grid face lives on the sibling `MenuPixelLayer` (layer 3, scale `H`). **The two menu layers have distinct indices, so no draw order rests on a tie-break.** (Probed: two same-index siblings draw later-over-earlier, consistent with the sibling index as the sublayer key, which is fragile to reparenting; giving each layer its own index removes the dependency, and the debug layer moves to 5.) The Chart Table's layer budget (`D_L3`) counts both menu layers.

**2.6 One writer, enforced.** Only `PipelineRoot` (and its `ViewportWiring` helper, `src/presentation/pipeline/viewport_wiring.gd`) may write: `SubViewport.size`, `size_2d_override*`, the container's `position`, `scale`, `stretch_shrink` and `material`, any `CanvasLayer.scale` / `offset` / `transform`, `HudRoot.size`; the world viewport's `canvas_transform` and the parallax layer offsets are written only by the **`CameraHost`'s `frame_pre_draw` handler** (§ 6.2). These two are the geometry writers named on ADR-0009's lint allow-list; they are legitimate and do not go through `Placement` (parallax offsets still use `Placement`'s integer helpers). Presentation subscribers (the Filter, Lighting, HUD, Chart Table) call neither; they receive a read-only `RenderLayout` (§ 3) and `world_to_hud`.

### 3. Geometry: `k`, `H`, aspect, zoom, and one `RESCALING` path

**Decision.** All derived geometry is one **pure function**, `RenderLayout.derive(...)`, a `RefCounted` value with no scene-tree dependency, injected where read. It implements the GDD's Formulas D.1 to D.5 and D.14 as written (the GDD owns the formulas; this ADR owns where their outputs go). `PipelineRoot` is the only caller that applies a result.

- **Inputs:** the window pixel size `W x H_w` (physical pixels, never DPI-scaled; `content_scale_factor` stays 1.0), a `DisplaySettings` value (aspect `KEEP | EXPAND`, manual world scale `0` for Auto, HUD scale choice), the current zoom step `k_z`, the static `ZOOM_MIN_K` row table (carried by `RenderTuning`, § 6.1), and the `offered(P_w, P_h)` subset of HUD scales from `HudLayout` (HUD owns the predicate `bottom_row_fit`). `offered_h` is an `Array[int]` that `derive` duplicates on entry, so a caller that mutates it afterwards, or a layout that keeps it, cannot change a result.
- **Outputs (all integers or integer-valued):** `k_auto = floor(min(W/640, H_w/360))` (a window below `min_size` 1280x720 cannot occur on a compliant window manager; if one does, `k_auto` is clamped to the Section G invariant `k_min == 2` and a `WINDOW_BELOW_MIN` diagnostics entry is written, never a `k` of 1 or 0), `k_base` (manual `k` when below Auto, else `k_auto`; a manual `k` below Auto forces `KEEP`), `H` (`H_default = max(k_auto, 3)`, never below 3, in `[H_default, 2 * H_default]` and in `offered`), `field_px` (D.1/D.3 at `k_base`, D.5 at a ladder step), `picture_rect` (the window region that shows the game, centred, integer), `world_rect = field_px * k_z` centred in the picture (a ladder field that does not divide the picture leaves a few black pixels; the HUD does not move), `hud_rect` (**pinned to the `k_base` picture rect**, so a zoom step never moves the HUD), `hud_canvas = hud_rect.size / H`.
- **Letterbox bars** are the root viewport's **clear colour** (`default_clear_color = (0,0,0,1)`, § 5.1), outside every node, so the Filter, the jolt and any `CanvasLayer` cannot touch them. A 1-HUD-px hairline at the picture edge, if the art director adds one, is a layer-1 element excluded from the HUD count (PR-12 resolution).

**3.1 Zoom is a change of logical size, never `Camera2D.zoom`.** The ladder is `[effective_min, k_base + 2]` with `effective_min = ZOOM_MIN_K(k_base, target class)` (default 1 until the spike fills it); `k_z` is persistent, edge-triggered, never held and never saved. `PipelineRoot.request_zoom_step(direction: int)` is the only entry point (the Input Adapter calls it from its `_process`; ADR-0019). A step is accepted when it is inside the ladder, no other step was accepted this frame, and at least `ZOOM_STEP_MIN_INTERVAL` (0.15 s) of **engine process time** has passed since the last accepted step; the interval is a countdown decremented by the `_process` delta divided by `Engine.time_scale` (guarded when the scale is 0) in `PipelineRoot`, so the cooldown is wall-clock even if `time_scale` is not 1 (whether `time_scale` scales `delta` is UNVERIFIED in ADR-0001; the injected monotonic `Clock` of ADR-0010 is an equivalent source and either satisfies the rule), **not** `present_time` (frozen under pause, where zoom must still work) and not any `Time.*` call (forbidden by ADR-0001's clock lint); surplus presses are discarded, not queued, and counted in `diag.zoom_step_rate_limited`. A refused press (beyond either end, or below `ZOOM_MIN_K`, or while the camera is `HELD`) increments `diag.zoom_step_refused` with a reason. Accepting a step sets the layout dirty, so the step is **one `RESCALING` frame and a camera cut**: under pass A `SubViewport.size = field_px(k_z)` and `container.scale = k_z`; under pass B `size = field_px * k_z`, `override = field_px`. Lighting has no zoom handler: every `Light2D` length is in base px and independent of `k_z` under both passes (LO-16), which this wiring honours because the world canvas is never scaled by `Camera2D`.

**3.2 One `RESCALING` path for every cause.** `Window.size_changed`, a zoom step, a display-mode change, a world-scale or HUD-scale or aspect setting change, and `dpi_changed` each **only set a dirty flag** (the same dirty-mark principle as every notification). `PipelineRoot._process` is the single presentation entry point; it does the following **in this fixed order inside one function** (no priority tie, no tree-order dependency):
1. advance `present_time` (§ 6.3);
2. decrement the zoom cooldown (§ 3.1);
3. if the layout is dirty, run `RESCALING` (below);
4. call `driver.frame(delta, present_time)`, which runs ADR-0009's stages in order, `CAMERA_SOLVE` first (§ 6.2).

The re-derivation therefore happens at most once per rendered frame and completes before any presenter of that frame draws (the main loop processes events, then `_process`, then draws), so a continuous drag of the window edge is a sequence of at most one re-derivation per frame converging on the final size (coalescing by frame, not by time). `RESCALING`:
1. computes the new `RenderLayout`;
2. **reallocates the viewport only if `field_px`, `k_z` or the render mode's size changed**; a pure translation (the picture re-centring by a pixel during a drag) moves only the container position and the HUD rect;
3. applies it through `ViewportWiring` (container position and scale, viewport size and override, layer offsets and scales, `HudRoot.size`), calls `HudLayout` once, re-derives `rect_eff` from the raw `CameraLimits` (never writes `CameraLimits`; § 6.1a), increments the layout version counter that `PipelineRoot` holds (consumers poll it), and republishes `world_to_hud`'s inputs;
4. sets `UPDATE_ONCE` on the frame of any size change while updates are disabled (**mandatory**, not a guard: a resize of a disabled viewport clears it, § 2.2), then `PipelineRoot` sets `UPDATE_DISABLED` itself on the next frame;
5. enters the `RESCALING` state for that frame and `PRESENTING` (or the held state) on the next.

The re-derivation completes inside the frame and costs at most 4 ms CPU (§ 7). No frame shows a fractional transform because there is none to show: every transform written is integer-valued. Code that runs right after a resize must not read `WorldContainer.size` or mouse rects: the container is a `Control` under a non-`Control` parent, its `size` refreshes from the viewport's minimum size at the next deferred update and is stale until then (drawing is unaffected with `stretch = false`); only `scale` and `position`, which the pipeline itself wrote, are read (§ Implementation Guidelines, Gotchas).

**3.3 The coordinate conversions are `RenderLayout` methods, not re-derived anywhere (TR-pxl-019, TR-pxl-014).**
- `world_to_hud(body_pos: Vector2i, pose_q: Vector2i) -> Vector2i` (the two-argument form is the only form) is the single projection from a body-frame base-px position to HUD-root coordinates, built from `k_z`, `H`, the container's integer origin, the unjolted `pose_q` and the HUD origin (D.14), floored to the HUD grid. No system re-derives it. `pose_q` is the pose the `CAMERA_SOLVE` stage published earlier in the same `_process`, and the HUD stage runs after it (ADR-0009 stage 5 after stage 1), so the HUD never projects frame N's prompts with frame N-1's pose.
- `mouse_to_world_aim(cursor_px: Vector2i, p: Vector2i) -> Vector2` is the open-loop aim mapping (D.14b): `f_q + (cursor_px - picture_centre) / k_z`, with the published look-ahead-free follow pose `f_q`, no look-ahead, no inner-box clamp, no jolt. Because the root has **no stretch**, `Viewport.get_mouse_position()` on the root already returns window pixels and **no inverse `get_final_transform` conversion is needed** (the GDD's leading candidate (i) says so; candidate (ii) would have needed it). `mouse_to_world_view(cursor_px: Vector2i) -> Vector2` is the full unjolted-pose inverse for picking and pings and **must never feed beam aim** (a lint, ADR-0010). The pipeline root's input mapping tests event types, never `device == 0` (4.7 device IDs, ADR-0019).

### 4. Render mode A or B: one constructor argument, A built at MVP

**Decision.** The render mode is `ViewportWiring.Mode { A, B }`, a **constructor argument** of `ViewportWiring` (the composition root passes a default read from one build constant or feature tag), read only inside `ViewportWiring`. It is an argument rather than a `const` read inside the class because a GDScript `const` cannot be varied in a test, and the equivalence test must construct both wirings. **A is the default and the recorded fallback; B is adopted only if the criteria of art bible 8.12 step 2 are met on the named machines. MVP builds pass A only** behind the `ViewportWiring` seam: B is specified here (table below) but its wiring branch and the A/B equivalence test are built only if lighting spike step 2 adopts B. Everything outside `ViewportWiring` sees an identical `RenderLayout` and identical coordinate, sort, camera and HUD contracts in both modes (PR-7); once B exists, a test constructs both wirings from the same input trace and compares the logical outputs. The lint "only `ViewportWiring` branches on `Mode`" is kept.

| | Pass A (default) | Pass B |
|---|---|---|
| `SubViewport.size` | `field_px` (640x360 at the default step) | `field_px * k_z` (the **picture**, never the window) |
| `size_2d_override` / `_stretch` | unset / false | `field_px` / true |
| `WorldContainer` | `scale = (k_z, k_z)`, position = `world_rect.position` | `scale = (1, 1)`, same position |
| Lights, shadows, glow resolve at | base resolution (cone and shadow edges stair-stepped by `k_z` screen px) | window resolution (1 to 2 screen px edges) |
| Lit pixels, 1080p default / `k_z = 1` | 230,400 / 2,073,600 | 2,073,600 / 2,073,600 |
| World render targets, 1080p | at most 3 MB | at most 22 MB |

**Why the viewport is the picture, not the window, under B.** The PR-7 text puts the pass-B viewport "at window size". The probe shows that an override that does not divide the viewport size gives a non-integer scale (1366x768 over 682x384: 2.002933), so under B the viewport must be exactly `field_px * k_z`, with the letterbox and the ladder remainder outside the container where the clear colour is black. This is a refinement of PR-7, returned as a GDD sync item. Under B the rotated horror and web segments rasterise at window resolution off the grid (tolerated for horrors and web only, PR-4 rule 5); the Q = 1/k camera quantum (Open Question 11, not MVP) is `[UNV]` under B. The cost of B at the bottom ladder step equals A's, so any decision that rests on A's saving is made at the default field and zoom-in steps only (D.16). Under pass B `canvas_transform` is in base-pixel (override) units and the `snap_2d_*` flags snap in base-pixel space (probed, § 2.2); the flags snap to base pixels in both passes, which ADR-0009 may cite.

### 5. Project pins and startup guard rails

**5.1 The pinned configuration (effective values, asserted).** The table lists every value the pipeline requires, the probed built-in default, and whether the key must therefore be present in `project.godot`. A key whose required value equals the default may be absent; it is still asserted by its **effective** value.

| Setting | Required | Built-in default (probed) | In `project.godot`? |
|---|---|---|---|
| `display/window/stretch/mode` | `disabled` (wiring (i); the GDD's `canvas_items` flips, § 2.1) | `disabled` | absent is correct |
| `display/window/stretch/aspect` | `keep` | `keep` | absent is correct |
| `display/window/stretch/scale_mode` | `integer` (defence: should anyone re-enable stretch, it cannot go fractional) | `fractional` | **required** |
| `display/window/size/viewport_width` / `viewport_height` | 640 / 360 | 1152 / 648 | **required** |
| `display/window/size/window_width_override` / `window_height_override` | 1280 / 720 (an editor launch must not open below `MIN_WINDOW`) | 0 / 0 | **required** |
| `Window.min_size` (set in code by the composition root) | 1280x720 | (0,0) | not a project key |
| `rendering/textures/canvas_textures/default_texture_filter` | 0 (NEAREST) | 1 (LINEAR) | **required** |
| `rendering/2d/snap/snap_2d_transforms_to_pixel` / `snap_2d_vertices_to_pixel` | true / true | false / false | **required** |
| `rendering/environment/defaults/default_clear_color` | `Color(0, 0, 0, 1)` | `(0.3, 0.3, 0.3, 1)` | **required** |
| `rendering/viewport/hdr_2d` (root viewport) | false (root stays SDR-exact) | false | absent is correct |
| `use_hdr_2d` on the **world** `SubViewport` | **false at MVP** (SDR; an HDR viewport through the container outputs unconverted linear colour, probed). The HDR contingency of § 2.4 requires an amendment of this ADR, the resolve material and a changed expected value | false | set in code (§ 2.2); asserted |
| `rendering/anti_aliasing/quality/use_debanding` and the world viewport's | false unless spike item 1 shows both: flat palette fills read back bit-exact with it on, and the 0 to 1 % ambient gradient bands visibly with it off | false | absent is correct |
| `display/window/hdr/request_hdr_output` | false (the 4.6 palette is SDR-exact) | false | absent is correct |
| `display/window/per_pixel_transparency/allowed` and world `transparent_bg` | false / false | false / false | absent is correct |
| `physics/common/physics_interpolation` | false | false | absent is correct |
| `application/run/low_processor_mode` | false (ADR-0001) | false | absent is correct |
| `display/window/vsync/vsync_mode` | 1 (enabled); Mailbox offered as "uncapped, no tearing"; Adaptive never the default | 1 | absent is correct |
| `rendering/renderer/rendering_method` | `forward_plus` (§ 1) | `forward_plus` | present today; asserted by effective value |
| `rendering/rendering_device/driver.windows` and `driver.linuxbsd` | `vulkan` / `vulkan` (§ 1) | `vulkan` / `vulkan` (also `macos` `metal`) | absent is correct; asserted by effective `ProjectSettings` value in CI and by the live driver name at boot |
| `rendering/rendering_device/fallback_to_opengl3` | `false` (no silent degrade to Compatibility, § 1) | `true` | **required** |
| `rendering/rendering_device/fallback_to_d3d12` | `false` while Vulkan is the pin | `true` | **required** |
| `rendering/rendering_device/fallback_to_vulkan` | left `true`; becomes `false` only if the D3D12 reversal of § 1 is taken | `true` | absent is correct |
| `rendering/driver/threads/thread_model` | `1` (default; value `2` is unsupported: the probed `frame_pre_draw` ordering was observed under 1 and 2, but only 1 is pinned) | `1` | absent is correct; asserted by effective value |
| `gui/common/snap_controls_to_pixels` | left at default; **never relied on** (HUD HS-9 snaps manually) | true | absent |
| Shader Baker | on in the release export preset (4.5 export option; first-frame hitch measured by lighting spike step 10); the export preset must **enable the Vulkan driver** (the baker bakes only for drivers enabled in the preset; not supported under Compatibility, recalled) | n/a | export preset |

The HDR project-level `hdr_2d` is **false on the root viewport**, and the world viewport's own `use_hdr_2d` is **also false at MVP** (§ 2.2), so the whole window is SDR-exact and palette fills read back bit-exact. The art bible 8.4 text writes `rendering/viewport/hdr_2d = true` at project level; the Pipeline PR-3 text says world-viewport members and "never on the root". This ADR follows PR-3 for the root (a root-level HDR framebuffer would put the HUD and bars on the float path) and defers the world viewport's HDR to the gated contingency of § 2.4; both texts are flagged as sync items. Whether the project setting reaches the `SubViewport` at all is NOT SOURCEABLE, which is why the world viewport's members are set in code and read back.

**5.2 The `RenderConfigSnapshot` and `StartupCheck` (pure).** The startup assertion reads the effective values into one value object, `RenderConfigSnapshot`, built by `RenderConfigReader` (the only code that calls `ProjectSettings`, `RenderingServer` and the viewport getters for this purpose), and `StartupCheck.run(snapshot, expected) -> Array[StartupFinding]` is a pure function over it. **The reader uses explicit typed assignments**, never inference and never `as`: `var v: int = ProjectSettings.get_setting(&"...")` (the cast pattern ADR-0010 `PRJ-02` states once), because ADR-0010 makes inferred and unsafe declarations errors. A `StartupFinding` is a value `{code: StringName, key: String, expected: String, actual: String}`; there is no `assert`, `push_error` or `quit` in the check. The pipeline root (dev build) halts before any window is presented when the array is non-empty and prints the table. A release build treats a **method or driver finding as fatal** (the visible "unsupported GPU" screen of § 1, plus the diagnostics header) and writes every other finding to the diagnostics header and continues. Doctored snapshots exercise every finding in gdUnit4 with no window. **Startup findings are pipeline-local values written to the diagnostics header; they are not kernel faults, there is no fault domain for them at MVP (the pipeline root builds in BOOTING step 2, before the kernel, architecture § 8), and the registry's 15 to 31 presentation and tooling reservation is unaffected.**

**5.3 What the check reads back (dev build; PR-15 (2), TR-pxl-028).** `content_scale_mode == DISABLED`, `content_scale_factor == 1.0`, `Window.min_size == (1280, 720)`; the world viewport's `canvas_item_default_texture_filter == 0`, both snap members, `use_hdr_2d == false` (MVP), `use_debanding`, `transparent_bg`, `size != (512, 512)`; the container's `texture_filter == 1`, `stretch == false`, `material == null` at MVP (with the HDR contingency adopted: the resolve material present and the effective method not Compatibility); `physics_interpolation == false`, every `Camera2D` (if the variant has one) `position_smoothing_enabled == false` and `zoom == (1, 1)`; the clear colour; the window overrides; the HDR-output flag; `fallback_to_opengl3 == false`, `fallback_to_d3d12 == false`, `thread_model == 1`; and the **rendering method and driver name**. When `DisplayServer.get_name() == "headless"` the **live** `RenderingServer` method and driver rows are reported **NOT ASSESSED** (probed: a headless run reports `gl_compatibility`), never as a mismatch and never as a pass. **CI does not need the live rows:** `ProjectSettings.get_setting` of `rendering_method`, `driver.windows` and `driver.linuxbsd` returns the effective configured value headless, independent of the live headless `opengl3` driver, and CI asserts those (the driver keys are default-equal and therefore absent from `project.godot`, so a `ConfigFile` read would find nothing). The `TileSet.tile_size == (48, 48)` check is **not** here: tile sets are Layout content that loads after the pipeline builds, so it lives in ADR-0010's asset and scene lint (§ 5.5(b)).

**5.4 The scale readback chain (the acceptance criterion is pinned here because the GDD wording measures different things in A and B).** `get_final_transform()` of the world `SubViewport` is identity under pass A (the multiplication lives in the container) and the stretch scale under pass B. The criterion `get_final_transform().get_scale() == (k, k)` is therefore replaced by one chain that is mode-independent:
- **World effective scale** = `root.get_final_transform().get_scale()` (identity, stretch disabled) x `WorldContainer.get_global_transform().get_scale()` x `WorldViewport.get_stretch_transform().get_scale()`, asserted exactly `(k_z, k_z)` (integer comparison on `Vector2` components that are integer-valued by construction; no epsilon).
- **HUD effective scale** = `root.get_final_transform().get_scale()` x `HudLayer.transform.get_scale()` (and `Control.get_global_transform_with_canvas().get_scale()` of a probe `Control` under it), asserted exactly `(H, H)`.
`ViewportWiring.read_back() -> WiringReadback` returns these, and the same readback table is logged at boot.

**5.5 CI, lints and the tree walk (TR-pxl-028).** (a) **One headless gdUnit4 test with two read tiers** (it is ADR-0010 `PRJ-02`; this section's table is its normative key list, including the `fallback_to_*` keys and `thread_model`). Rows marked **required** are checked for presence and value with `ConfigFile`; rows marked **absent is correct**, including `physics/common/physics_interpolation` (absent or false) and the renderer and driver keys, are checked with `ProjectSettings.get_setting` effective values, because a `ConfigFile` read cannot see an absent key's effective default. `ConfigFile` structural checks (an empty `[autoload]`) stay at ADR-0010's E2 tier. The test fails naming exactly the offending key when a fixture copy has one key removed or set wrong, one key per run. (b) The scene-tree and asset lint (ADR-0010 chooses the tool and which rules are regex or boot walk): no `Camera2D` descends from a `PhysicsBody2D`; no `Control` and no node in group `hud` under the world viewport; `rotation == 0` on every node under the environment, pipe, prop, actor and HUD groups; `Control.scale == (1,1)` on every HUD node; **every loaded `TileSet.tile_size == (48, 48)` and none `(16, 16)`**; no `SubViewport.size == (512,512)`; a scan for assignments to `Camera2D.zoom`; the layer indices of § 2 are distinct. (c) A debug-only per-frame tree walk behind a console toggle flags fractional positions in the world viewport, rotation under human-built groups, `Camera2D.zoom != (1,1)`, `Control.scale != (1,1)` on the HUD, Y-sort enabled on the floor layer, a `physics_interpolation_mode` other than OFF on a pipeline-owned node, and a `CanvasItem` whose rect covers the letterbox (`diag.bar_overdraw`); it is dev-only (at most 2.0 ms), absent from release exports, and **a later, optional story on top of (b), not an MVP gate**. (d) **The windowed palette read-back test** (manual with retained evidence until a windowed runner exists): a flat `Color8(128, 64, 200)` fill, a one-pixel checker and a clear-colour pixel drawn in the world are read back from the root's screen texture and must equal the input exactly; run per renderer (Forward+ and Compatibility) on a dev build and a release export. It is the acceptance test for "palette-exact fills" and the guard for the HDR contingency of § 2.4.

The player-facing render options are an enumerated set only: world scale Auto or an integer of at least 2 from the offered list, HUD scale an integer in `[H_default, 2 * H_default]` (3 to 6 at 1080p), frame cap and V-sync mode (enabled, mailbox, off; no adaptive), and the reading-face scale (80 to 200 percent, the one continuous control), persisted by the same host-local `SettingsStore` (ADR-0030). No gamma, brightness, contrast, sharpening or filter option exists, and `apply_display_settings` rejects any `aspect`, `manual_k` or `hud_scale` value outside the offered enumeration (Pipeline GDD PR-14, TR-pxl-033).

### 6. The presentation host, the camera solve and write, and `present_time`

**6.1 The camera is a pure solver run in the `CAMERA_SOLVE` stage (TR-pxl-010, 011, 012, 013, 015; the formulas D.6 to D.10 are the GDD's).** `CameraSolver` is a `RefCounted` with `solve(input: SolverInput, state: SolverState, params: SolverParams) -> Pose`, no scene-tree dependency, float64 per-axis state held as scalars (not `Vector2`), `dt` clamped to `DT_MAX` (0.05 s). The **type of `SolverInput` is the enforcement of Camera C1**: it carries only the follow target's previous and current body-local positions, `tick_alpha`, the raw local beam aim, the zoom step, `dt`, cut / rebase flags, and `limits: CameraLimits` (read-only; the raw hull rectangle of § 6.1a, a body-frame geometry value and not world state); it has no field through which Attention, breach, horrors, sound, the graph or sanity could enter, so the same input trace gives the same camera trace in calm, tell and breach by construction. The Perception Filter has no write path to a camera or viewport property (§ 2.6). The camera is a `PresenterDescriptor` registered at `PresentationStage.CAMERA_SOLVE` (stage 1, ADR-0009 § 3): it calls `solve` once per frame from `driver.frame` and **publishes `pose_q`, the follow pose `f_q`, the jolt offset, the parallax offsets and the `world_to_hud` inputs** to `CameraHost` and `RenderLayout` before the Avatar, Filter, HUD and Lighting stages run. Solver parameters (`tau_follow`, `DT_MAX`) and `ZOOM_MIN_K` are the `RenderTuning` table loaded by ADR-0007's pipeline and injected (schema in Key Interfaces). Camera state is never persisted.

**6.1a The limits split and the clamp rules (CT-56; this ADR owns the clamp, ADR-0034 owns the raw rectangle).** `CameraLimits` (ADR-0034 § 9: `valid`, `min_x`, `min_y`, `max_x`, `max_y` in integer base px, half-open on the max side, `body_id`, `epoch`) is the **raw hull rectangle**, written to **only by `body.projector`** in stage `PROJECT`, in the avatar's body frame (the canvas frame, C2). It reaches the solver as the read-only `SolverInput.limits` field, read in `CAMERA_SOLVE`; stage 0 runs before stage 1, so a `BODY_TRANSFER` rebase and the new body's rectangle are consumed in the same frame. Nothing in the pipeline writes `CameraLimits`; the `RESCALING` step 3 "re-derive the limits" (§ 3.2) re-derives `rect_eff` from the then-current `CameraLimits` and the new `RenderLayout`, it does not write the raw rectangle. **`rect_eff` is pipeline-derived solver state** (the eased copy of the rectangle with the viewport-size handling, never published by the projector): its four edges approach the raw rectangle with the D.6 factor and snap to it on a `CUT`, on a `RESCALING` frame (a zoom step is a cut) and on load, so a changed rectangle never pops the pose; where `field` is larger than the rectangle on an axis, `lo = hi` is the centre of `rect_eff` on that axis (GDD C8). **The clamp rules are this ADR's, as the GDD's D.8b states them** (the TR-pxl-017 row in `tr-baseline.md` had attributed them to ADR-0034, and no ADR wrote them): (1) the inner-box clamp (D.8) on the offset, then (2) the limits clamp on the pose with `[lo, hi] = [rect_eff.min + field/2, rect_eff.max - field/2]` per axis, and **the limits win**; the clamp acts on the pose, not on the follow target, so `f_q`, the cut predicates (evaluated on the unclamped follow) and `mouse_to_world_aim` never see it; (3) `CLAMPED` is the camera state in which the pose is clamped on an axis (it suspends the 60 % guarantee on that axis, GDD D.8b), published with `diag.limits_engaged`; (4) **`world_to_hud` is never offset while `CLAMPED`** (the HUD projects the clamped, unjolted `pose_q`); (5) the **JOLT** offset (at most `JOLT_PX`, default 0, cap 2, P1 events only, zero under reduced motion) is added after the clamp, stays inside the viewport and is never a container offset, `world_to_hud` and `mouse_to_world_aim` ignoring it. `Camera2D.limit_*` is never written in either variant; the solver is the only clamp. The figures and the formulas stay the GDD's (D.8, D.8b, D.11).

*Restore and discontinuity.* The kernel calls `snap_baseline` on every interpolated block on each transition into `RUNNING` (ADR-0001 § 4, ADR-0003), so the first frame after a restore interpolates nothing; that is the **only** restore-continuity mechanism and this ADR adds no owner-side rule. The camera stage additionally **CUTs** the solver (starting with `L = 0` easing in) in its `on_discontinuity(DISCONTINUITY_RESTORE)`, which ADR-0009's driver delivers when `PresentationClock.load_seq` changes (the kernel increments it on every `LOADING → RUNNING` transition, restore and new game, never on `PAUSED` or `SAVING → RUNNING`; ADR-0001 amendment batch). A pod or transit body transfer is a CUT; a weld or detach `BODY_TRANSFER` carrying `frame_delta_px` calls `rebase(frame_delta_px)`, which shifts the follow and history and keeps lag and `L`.

**6.2 The frame model and the hook.** `PipelineRoot` **is** ADR-0009's single `PresentationHost` (closing ADR-0009 OQ-E, OQ-J and OQ-L): no other node hosts the driver, and because every step below runs inside one function, no priority tie and no tree-order dependency exists. The composition root constructs the `PresentationDriver` and injects it; `PipelineRoot` holds it. Its single `_process`, at `ProcessPriority.PRESENTATION`, runs in this fixed order: (1) advance `present_time` (§ 6.3); (2) the zoom cooldown (§ 3.1); (3) the `RESCALING` re-derive if dirty (§ 3.2); (4) `driver.frame(delta, present_time)`, which runs the stages with `CAMERA_SOLVE` first. Presenters have no `_process`; **there are no subscribers drawing from their own `_process`** (ADR-0009 Alternatives 1 and 2 reject that shape).

The `CameraHost` (a top-level presentation node under `PipelineRoot`, never a child of a body) connects one handler to **`RenderingServer.frame_pre_draw`**. The handler does exactly two writes from already-solved values and nothing else: the world viewport's `canvas_transform` (from `pose_q` plus the jolt offset) and the parallax layer offsets (integer offsets the camera stage computed with `Placement`'s integer helpers, `floori`, not a bare `-floor(...)`). It **does not solve**, it is **idempotent and branch-free**, and it is **safe if the handler is skipped for a frame** (a minimised or occluded window): the next call writes the then-current solved values and nothing accumulates. **Primary variant: no `Camera2D` in the world tree** (a lint); the transform has scale (1,1), no rotation and no skew every frame and only the host writes it. **Alternative variant**, if the direct write is rejected at step 12: a `Camera2D` with `anchor_mode = ANCHOR_MODE_FIXED_TOP_LEFT` (probed: `anchor_mode` default 1 is `DRAG_CENTER`), drag margins off, and `force_update_scroll()` called after each write; every camera assertion is read as holding on the `Camera2D` if the variant has one. `Camera2D.limit_*` stays at the engine defaults in both variants; **the solver is the only clamp** against the `rect_eff` it derives from the raw `CameraLimits` that `body.projector` publishes (ADR-0034; § 6.1a), so `world_to_hud` is never offset while `CLAMPED`. A `Camera2D` never descends from a physics body.

*Why the solve is not in the handler.* Probed ordering (windowed 4.7.2 editor, Forward+, main thread, `thread_model` 1 and 2): all node `_process`, then `frame_pre_draw`, then `frame_post_draw`; a `canvas_transform` written in the handler is the one drawn that frame, and a write from `_process` is also drawn that frame. The handler therefore runs after the HUD stage, so a solve inside it would leave `world_to_hud`, the Avatar presenter's beam pose and `mouse_to_world_aim` one frame behind the world: a permanent HUD-versus-world disagreement on exactly the "screen is still when the ship is still" surface (PR-6 C11, TR-pxl-019). Solving in the `_process` pass also keeps the solver unit-testable without a signal that never fires headless.

*Verification status (TD-20).* Windowed 4.7.2 editor: all node `_process`, then `frame_pre_draw`, then `frame_post_draw` (probed); it fires only when a frame is drawn (never headless). Release export, Direct3D 12, the Steam Deck, a minimised or occluded window and `low_processor_mode` are UNVERIFIED (pipeline spike step 12 re-runs the same probe on a release export). `rendering/driver/threads/thread_model` is pinned at 1 (§ 5.1; value 2 unsupported) so the probed ordering holds. **Fallback, used only if `frame_pre_draw` fails the release-export spike:** the canvas write moves to the last statement of `PipelineRoot._process`, after `driver.frame(...)`. That needs no new `ProcessPriority` constant: `PRESENTATION_LATE` is withdrawn (ADR-0001 keeps three constants plus `HOST_SERVICE`).

*Canvas coordinate exactness (TD-23, E14; ADR-0005 TR-shipbody-038 and ADR-0010 cite this paragraph).* **Presentation coordinates are integer-valued `float32` and exact for `|v| <= 2^24` (16,777,216) base px.** `Transform2D`, `Node2D.position` and `Control.position` are `float32` in the standard build (`Engine.is_double_precision()` was not checked and is assumed false). `pose_q` and every placement are `Vector2i`; a `Vector2i` is converted to `Vector2` **only** at `Placement.put` and at the `canvas_transform` write, and the solver's float64 scalars are quantised to `Vector2i` before that conversion, so that conversion is the one point where exactness is kept. World and body-frame extents stay below 2^24 base px; if body-frame extents could exceed that, ADR-0015 owns the bound (the rebase mechanism of § 6.1 exists to keep body-frame coordinates inside it). A unit test constructs the write at `2^24 - 1`, `2^24` and `2^24 + 1` and asserts the first two round-trip and the third is rejected or rebased, never silently rounded.

*Scope edge with ADR-0009.* The Pipeline GDD PR-8 item 3 calls the pipeline root's `frame_pre_draw` handler the "render callback" in which every subscriber reads and draws. That is **not** adopted: subscribers draw from `driver.frame` in the `_process` pass (ADR-0009 § 3), and `frame_pre_draw` is reserved for the `CameraHost`'s two writes. This is returned as a GDD sync item.

**6.3 `present_time`, the pipeline-owned clock (ADR-0001 handed the owner to this ADR).** `PipelineRoot` owns `present_time: float` (seconds, a 64-bit float). Step (1) of `_process` advances it by the `_process` delta while `PresentationClock.state` is `RUNNING`, `SAVING`, or `LOADING` with the hold shown, and leaves it frozen while `PAUSED`; it is **never scaled by reduced motion** (reduced motion scales individual effects). Layer 0 animation uses only `present_time` (delivered to presenters as `PresentFrame.present_time`); the menu layers (3 and 4) and the debug layer (5) run on engine process time so the chart table responds while paused; the HUD (layer 2) freezes with the world. The cosmetic-phase rule ("the tick counter is never used for cosmetic phase") is ADR-0001's and is not restated. The kernel state comes from the read-only `PresentationClock.state: int` (the kernel state enum: `RUNNING`, `PAUSED`, `SAVING`, `LOADING`, `FAULTED`), and `setup()` takes `PresentationClock` only; no `KernelStateView` type exists. This depends on the ADR-0001 amendment batch (Depends On, Ordering Note). `SceneTree.paused` is not the kernel's pause and is not used by the pipeline.

**6.4 Pipeline states.** The pipeline state machine (`BOOTING`, `PRESENTING`, `RESCALING`, `TRANSIT_HOLD`, `PAUSED_FROZEN`, `HIDDEN`, with the `DEBUG_OVERLAY` modifier) is implemented as the GDD's States and Transitions table, with the camera state as three independent fields (follow, zoom, hold). The wiring-specific rules recorded here: `PAUSED_FROZEN` sets `UPDATE_DISABLED` and holds an undimmed last frame (a resize while paused forces the one-shot `UPDATE_ONCE` of § 2.2, or the held frame would be blank); `TRANSIT_HOLD` shows the layer-1 black quad (`#000000`, no text) with a 120 ms fade in and a cut out, and `PipelineRoot` reads `ProjectorReadiness.is_ready()` (ADR-0034 § 6: `pending_units == 0` and every `ProjectionSet` section complete and mounted) for the latch that releases it (WM-14: the cut to the avatar needs the pipeline's own projector latch as well); **the `HoldLayer` shown while the kernel is `LOADING` (and in `BOOTING`) is released on `RUNNING` and `ProjectorReadiness.is_ready()` together, not on `RUNNING` alone** (CT-33; it covers a cold load from the title screen, where no old roots exist, so the first `RUNNING` frame never shows an empty interior); `present_time` advances during `LOADING` with the hold shown (§ 6.3) whichever condition is still pending, and a readiness that never arrives is the projector's diagnostic (`late_count`), never a pipeline fault; `FAULTED` is not a pipeline state (the pipeline keeps presenting the last committed state; a dev build halts with the overlay); any `(state, trigger)` pair not listed is ignored and reported to diagnostics.

### 7. The draw-call and timing budget rules (architecture IC-6)

The Pipeline GDD owns the frame budget (D.16 and the Performance block) and therefore the **figures**; this ADR owns the **rules**: one table, draw calls read **per viewport**, an owner column, and withholding a zoom step rather than cutting content. The table below mirrors the GDD's D.16 as of 2026-10-28 so the nesting is visible in one place. **A system raises its row by amending the GDD's D.16 block (then this mirror is refreshed); this Foundation ADR is amended only if a rule changes, not after each spike measurement or system budget change.** Every figure is a proposal pending release-export measurement on RM-DESKTOP and RM-DECK (recorded with the printed renderer and driver); until measured, a performance criterion is neither a pass nor a fail. Draw calls are read per viewport with `Viewport.get_render_info(Viewport.RENDER_INFO_TYPE_CANVAS, Viewport.RENDER_INFO_DRAW_CALLS_IN_FRAME)` (enums probed: type 2, info 2; 0 headless; works per viewport in Forward+ and Compatibility, probed) so the HUD's number is never confused with the sorted layer's. `DRAW_CALLS_IN_FRAME` is read after the frame, so a value read in `_process` **lags one frame**. **Probed: the container composite is counted on the root, one call per `SubViewportContainer`** (a root with only three containers reported 3), so the table carries it as its own row.

**7.1 Draw calls (default field and every zoom-in step).**

| Row | Viewport | Term | Cap | Owner (GDD) | Notes |
|---|---|---|---|---|---|
| Floor `TileMapLayer` | world | `D_floor` | about 4 | Layout / art bible 8.6 | `rendering_quadrant_size = 16` |
| Floor trails and stains | world | `D_marks` | 4 to 20 | Failure & Trail | |
| Y-sorted layer (walls, props, actors, horror rigs, pipes, emissive overlay) | world | `D_ysort` | 50 to 300 | ADR-0034 (the layer and this row's ownership, confirmed), Pipeline D.13 | **about 300 is the trigger for the per-row z-band fallback** (TR-pxl-009); measured, not sourceable; readable on a release export only once Verification Required (9) confirms it. The world viewport content is the single `ProjectorRoot/World` tree (ADR-0034, scene-tree diagram); the z band names (`ZBand`) are owned by `src/presentation/projector/z_band.gd` and no number is restated here |
| Wall caps `TileMapLayer` | world | `D_caps` | about 4 | Layout | |
| Effects (fringe up to 10, up to 12 live particle emitters) | world | `D_fx` | at most 22 | Perception Filter / Failure & Trail | |
| **World total** | world | `D_world` | **at most 350** | Pipeline D.16 | `4 + 20 + 300 + 4 + 22 = 350` |
| Failure & Trail share | world | `D_FT_MAX` | 38 (20 to 60 under the silhouette-mask route) | Failure & Trail D.9 | **nested inside `D_marks + D_ysort + D_fx`**, not additive |
| Lighting shadow passes | world | n/a | not separately capped | Lighting / ADR-0028 | whether `RENDER_INFO_DRAW_CALLS_IN_FRAME` counts them is UNVERIFIED (lighting spike step 3 records it) |
| HUD | root | `D_hud` | 20 to 60 | HUD & Suit Layer (spike S2) | `HUD_PRIMITIVE_MAX` 90 is its primitive cap, a different unit |
| **Container composite** | root | `D_composite` | **1** (one per `SubViewportContainer`; there is one) | Pipeline D.16 | probed: counted on the root |
| Filter pass | root | `D_filter` | 0 at MVP (`PER_ELEMENT`, slot null), 1 with a material (the Filter, or the HDR resolve of § 2.4) | Pipeline PR-10 | |
| Transit hold quad | root | `D_hold` | 0 or 1 | Pipeline PR-3 | |
| **Root total, chart closed** | root | `D_root` | **at most 63** | Pipeline D.16 | `60 + 1 (hold) + 1 (filter) + 1 (composite)`; with the slot null at MVP the expected figure is 62 and the cap stays 63 |
| Chart table (layers 3 and 4, counted only while open) | root | `D_L3` | at most 40 (`LAYER3_DRAW_CALL_BUDGET`, range 30 to 60); about 20 to 33 estimated | Chart Table CT / ASK-CT-12 | counts both menu layers |
| **Root total, chart open** | root | `D_root` | **at most 93** | Pipeline D.16 | holds only because **the HUD is hidden while a full-bleed plate is open** (`D_L3 <= 40` plus hold, filter and composite is at most 43), and the panel case (HUD up to 60 plus panel draws well under 30, plus 3) is at most 93; a plate with the HUD drawn (`63 + 40 = 103`) is excluded by that rule |
| **Frame total** | both | `D_world + D_root` | **at most 1000** | art bible 8.6 / D.16 | |
| Debug overlay (layer 5) | root | n/a | excluded | Debug Overlay | dev builds only; not counted against any release figure |

**7.2 Zoom-out steps.** The single `D_world <= 350` cap cannot hold when 4x to 9x the content is in view (D.16 arithmetic). At every zoom-out step (the Deck's 1280x800 field, 1080p at 1x, the bottom steps of 1440p and 2160p) only `D_world + D_root <= 1000` applies, **plus** the P99 total-frame ceilings below. A step that cannot meet both on RM-DESKTOP or RM-DECK is **withheld** (static table `ZOOM_MIN_K`, set from the measured table and never changed by a live measurement), never "fixed" by cutting content (default D5, confirmed by ruling R20). The GDD's predicted-withheld table is the starting hypothesis for the spike.

**7.3 Time and memory.**

| Quantity | Budget | Source |
|---|---|---|
| `PipelineRoot._process` including `world_to_hud` for 32 prompts, the state machine and `driver.frame` dispatch (excluding each stage's own budget) | at most 0.5 ms P99 | Pipeline Performance block |
| `CameraSolver.solve` (the `CAMERA_SOLVE` stage, inside `driver.frame`) | at most 0.05 ms P99 | same |
| `CameraHost` `frame_pre_draw` handler (two assignments) | negligible; folded into the frame total | this ADR |
| Container composite (the xk upscale, Filter null), GPU | at most 0.3 ms at 1920x1080, at most 1.0 ms at 3840x2160 | same |
| `RESCALING` re-derivation (CPU) | at most 4 ms, in one frame | same |
| Fractional-transform tree walk | at most 2.0 ms, dev builds only | same |
| World render targets, 1080p | at most 3 MB (A), at most 22 MB (B) | art bible 8.6 |
| Total frame, CPU P99, warm-up discarded | at most 14.9 ms RM-DESKTOP, at most 30.0 ms RM-DECK (90 % of 16.6 / 33.3 ms) at the default field and every ladder field; excludes frames in which the kernel is in `SAVING` (the autosave capture frame is bounded separately by ADR-0006 `SNAPSHOT_BUDGET_MS` and SAVE-WRK-1 (f)) | ruling R12 |

The table's own arithmetic (component caps sum to the viewport caps; the open-chart rows hold) is checked by review of the GDD's D.16 block, or by a doc-lint if ADR-0010's documentation checks adopt one; there is **no runtime `DrawBudget` function**. The measured comparison is the windowed, manual run with retained CSV and the printed renderer and driver, re-run on the release export.

GPU milliseconds and VRAM ceilings (the world and root render targets, including the `hdr_2d` target at 8 bytes per pixel if the § 2.4 contingency is taken, the shadow atlas and the glow buffers) are recorded per target and per pass from RM-DESKTOP and RM-DECK release-export measurement into `project.yaml` (a `performance` block beside `qa.min_spec`, to be created with the first recorded figure) before epics are created; until recorded they are proposals and a performance criterion on them is neither pass nor fail (Lighting GDD, TR-light-039).

### Architecture Diagram

```
Process / draw order, one rendered frame (ADR-0001 owns the priority constants; ADR-0009 owns the stage list; this ADR owns the PRESENTATION row)
  ProcessPriority.INPUT_ADAPTER (-100)  Input Adapter -> request_zoom_step(dir) (a host-local call, never a command), cursor read in window px
  ProcessPriority.TICK_DRIVER    (-50)  KernelHost -> TickDriver.frame()   (tick loop; tick_alpha; P5 marks dirty)
  ProcessPriority.HOST_SERVICE   (-40)  SaveHost (ADR-0006; not a presentation node)
  ProcessPriority.PRESENTATION     (0)  PipelineRoot._process == the single PresentationHost, one function, fixed order:
        (1) present_time += delta   (RUNNING / SAVING / LOADING-with-hold; frozen in PAUSED)
        (2) zoom cooldown -= delta / Engine.time_scale
        (3) if layout dirty: RESCALING  (RenderLayout.derive -> ViewportWiring.apply -> HudLayout; UPDATE_ONCE if disabled and resized)
        (4) driver.frame(delta, present_time)            [load_seq changed -> every presenter.on_discontinuity(RESTORE)]
              stage 1 CAMERA_SOLVE  CameraSolver.solve -> pose_q, f_q, jolt, parallax offsets, world_to_hud inputs   (published, not yet written)
              stages 2.. Avatar, Filter, Instruments, HUD (5; world_to_hud with THIS frame's pose_q), Lighting (6), Audio (8)   (ADR-0009)
  --- RenderingServer.frame_pre_draw (probed 4.7.2 windowed: after every _process, main thread, before the draw; release export UNVERIFIED, step 12):
        CameraHost: WorldViewport.canvas_transform = solved pose ; parallax layer offsets      two writes, no solve, idempotent
  --- draw

Window (stretch DISABLED)  --- pixels are screen pixels everywhere ---
┌───────────────────────────────── root Window, black clear colour (letterbox bars) ───────────────────┐
│        picture_rect (integer, centred; HUD pinned to the k_base picture)                                │
│   ┌──────────────────────────────────────────────────────────────────────────────────────────────┐   │
│   │ layer 0  WorldContainer (xk_z, NEAREST, material null)  ->  WorldViewport (field_px base px, SDR)│   │
│   │            lights · glow · ambient · occlusion · Filter (per-element) · camera jolt live HERE only │   │
│   │ layer 1  HoldLayer      opaque #000000 quad (TRANSIT_HOLD, BOOTING, LOADING)                      │   │
│   │ layer 2  HudLayer  (xH) -> HudRoot Control (hud_rect / H)      no light, glow, Filter, jolt       │   │
│   │ layer 3  MenuPixelLayer (xH)     layer 4  MenuReadingLayer (x1, screen-px sizes)                  │   │
│   │ layer 5  DebugLayer (xH)  dev builds only                                                         │   │
│   └──────────────────────────────────────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────────────────────────────────────┘
  world scale k_z ∈ [ZOOM_MIN_K, k_base+2]   HUD scale H ∈ [max(k_auto,3), 2*max(k_auto,3)]   independent integers
```

### Key Interfaces

```gdscript
class_name DisplaySettings extends RefCounted   # value; persisted host-locally by the Chart Table SettingsStore (CT-19, ADR-0030);
                                                # not a kernel block, not WORLD or PROFILE scope; ADR-0006 is not involved
var aspect: int            # KEEP | EXPAND; a manual k below Auto forces KEEP
var manual_k: int          # 0 = Auto
var hud_scale: int         # 0 = default; else an integer inside [H_default, 2 * H_default] that HUD offers

# RenderTuning: a typed facade (RefCounted) over an ADR-0007 ScalarTable plus one typed row table.
# Schema: src/data/render/render_tuning_schema.gd (ADR-0007 closed-schema convention); values are data, never constants in code.
#   Flat fields (INT / SCALED, converted once by the loader): tau_follow_ms (SCALED, 180), dt_max_ms (50), zoom_step_min_interval_ms (150).
#   ZOOM_MIN_K is NOT a flat scalar (two keys): a nested-row table of ZoomMinKRow {k_base: int, target_class: int, min_k: int}, default min_k 1.
class_name ZoomMinKRow extends RefCounted
class_name RenderTuning extends RefCounted
static func from_tables(scalars: ScalarTable, zoom_rows: Array[ZoomMinKRow]) -> RenderTuning   # built by the composition root
func tau_follow() -> float                  # seconds, converted at use from the SCALED field
func dt_max() -> float
func zoom_step_min_interval() -> float
func zoom_min_k(k_base: int, target_class: int) -> int

class_name RenderLayout extends RefCounted      # value; pure; no scene tree; fields are assigned in derive and never mutated afterwards
var window_px: Vector2i
var k_auto: int
var k_base: int
var k_z: int
var h: int
var field_px: Vector2i     # logical size in base px at k_z
var picture_rect: Rect2i   # screen px
var world_rect: Rect2i     # screen px, == field_px * k_z, centred in the picture
var hud_rect: Rect2i       # screen px, pinned to the k_base picture rect
var hud_canvas: Vector2    # hud_rect.size / h (may be fractional)
# No version field: PipelineRoot holds the layout version counter (a change counter is not part of a value).
static func derive(window_px: Vector2i, settings: DisplaySettings, zoom_k: int,
		tuning: RenderTuning, offered_h: Array[int]) -> RenderLayout   # offered_h is duplicated on entry
func world_to_hud(body_pos: Vector2i, pose_q: Vector2i) -> Vector2i       # the only form
func mouse_to_world_aim(cursor_px: Vector2i, f_q: Vector2i) -> Vector2    # open loop, D.14b
func mouse_to_world_view(cursor_px: Vector2i, pose_q: Vector2i) -> Vector2   # never for beam aim

class_name ViewportWiring extends RefCounted    # the ONLY reader of Mode; the only toucher of container / viewport / layer geometry
enum Mode { A, B }                              # a constructor argument, so the equivalence test can construct both (B built only if adopted)
func _init(mode: Mode) -> void                  # the composition root passes the default from one build constant or feature tag
func apply(layout: RenderLayout) -> void        # tracks the desired render_target_update_mode itself; never reads the property back
func read_back() -> WiringReadback              # effective world and HUD scales, sizes, filters (§ 5.4)

class_name PipelineRoot extends Node            # process_priority = ProcessPriority.PRESENTATION; the single PresentationHost (ADR-0009 § 3)
func setup(clock: PresentationClock, driver: PresentationDriver, tuning: RenderTuning,
		settings: DisplaySettings, hud_layout: HudLayout) -> void   # PresentationClock only: no kernel-state view type exists
func request_zoom_step(direction: int) -> void        # +1 zoom in, -1 zoom out; edge-triggered; rate-limited
func apply_display_settings(settings: DisplaySettings) -> void   # marks dirty only; the only entry point for a settings change
func set_filter_material(material: ShaderMaterial) -> void      # null at MVP; sole writer of the slot
var layout: RenderLayout:                              # read-only
	get: return _layout
var layout_version: int:                               # read-only; incremented on every applied layout; consumers poll it
	get: return _layout_version
var present_time: float:                               # read-only; advances per § 6.3
	get: return _present_time
var follow_pose_q: Vector2i:                           # read-only; the look-ahead-free follow pose f_q (same type as mouse_to_world_aim's f_q parameter);
	get: return _follow_pose_q                         # set only by the CAMERA_SOLVE stage inside _process, so a read from ProcessPriority.INPUT_ADAPTER
	                                                   # (-100, before this node's _process) holds the PREVIOUS frame's CAMERA_SOLVE value (ADR-0019 § 3.1, § 6.1)
# Published state: filter_route (PER_ELEMENT | NOT_USED), pipeline state, camera state, diag.* (names frozen under the Debug Overlay channel convention)

class_name CameraHost extends Node              # top-level presentation node under PipelineRoot; never a child of a body
func publish(pose_q: Vector2i, jolt: Vector2i, parallax: Array[Vector2i]) -> void   # called by the CAMERA_SOLVE stage (§ 6.1)
func _on_frame_pre_draw() -> void               # connected to RenderingServer.frame_pre_draw; two writes, idempotent, no solve (§ 6.2)

class_name SolverInput extends RefCounted       # the Camera C1 enforcement (§ 6.1): player-owned inputs and geometry only, no world-state field
# Fields: prev_pos / curr_pos (Vector2i, body-local), tick_alpha, raw_aim (Vector2), zoom_step, dt, cut and rebase flags, and
var limits: CameraLimits                        # read-only here. CameraLimits (ADR-0034 § 9, src/presentation/projector/) is the raw hull rect; its ONLY writer is
                                                # body.projector in stage PROJECT. rect_eff, the clamp order, CLAMPED and JOLT are derived and owned here (§ 6.1a)

class_name CameraSolver extends RefCounted
func solve(input: SolverInput, state: SolverState, params: SolverParams) -> Pose
func rebase(state: SolverState, frame_delta_px: Vector2i) -> void
func cut(state: SolverState, keep_zoom: bool) -> void

class_name RenderConfigSnapshot extends RefCounted   # value built by RenderConfigReader (explicit typed casts, § 5.2)
class_name StartupCheck extends RefCounted
static func run(snapshot: RenderConfigSnapshot, expected: RenderConfigExpected) -> Array[StartupFinding]

class_name WiringReadback extends RefCounted
var world_scale: Vector2        # root final x container global x viewport stretch; must equal (k_z, k_z)
var hud_scale: Vector2          # root final x HudLayer.transform; must equal (h, h)
```

(Each `class_name` is its own file under `src/presentation/pipeline/` except the `RenderTuning` schema under `src/data/render/`; the directory rules are ADR-0010's. The 24 class names of the first draft were checked free of `ClassDB` collisions on 4.7.2; the three names added by this revision (`ZoomMinKRow`, `RenderTuning`, `CameraHost`) are unchecked and are re-checked at the first story. `DisplaySettings`, `RenderLayout` and `RenderTuning` are values; `PipelineRoot` is the only `Node` that hosts the driver.)

### Implementation Guidelines
- **Must** keep the root `Window` at `content_scale_mode = DISABLED`, `content_scale_factor = 1.0`, `min_size = (1280, 720)`; **must never** enable root stretch or `canvas_items` scaling to obtain `k`, `H` or `k_z`.
- **Must** keep `PipelineRoot` the only `PresentationHost`: one `_process`, in the fixed order of § 3.2 and § 6.2; **must never** let a presenter define a `_process`, add a second host, or rely on tree order or a literal priority; use `ProcessPriority.PRESENTATION` by name.
- **Must** route every write of `SubViewport.size`, `size_2d_override*`, the container's `position`, `scale`, `stretch_shrink` and `material`, any `CanvasLayer` `scale` / `offset` / `transform` and `HudRoot.size` through `PipelineRoot` and `ViewportWiring`, and the world `canvas_transform` and parallax offsets through the `CameraHost`'s `frame_pre_draw` handler; **must never** write them from a presentation subscriber, the Filter, Lighting, the HUD or the Chart Table.
- **Must** compute all geometry in `RenderLayout.derive` from the window's **pixel** size and apply a result only in `PipelineRoot._process`; **must never** derive `k`, `H`, `world_to_hud` or the cursor mapping anywhere else, never read `DisplayServer` DPI for `k`, and never test `device == 0` for the cursor.
- **Must** size the world viewport as `field_px` (pass A) or `field_px * k_z` (pass B) with `size_2d_override` equal to `field_px`; **must never** leave the 512x512 default, size it to the window, or set an override that does not divide the size exactly.
- **Must** keep the world viewport SDR (`use_hdr_2d = false`) at MVP; **must never** set `use_hdr_2d = true` without the amendment, the resolve material and the renderer gate of § 2.4.
- **Must** set `UPDATE_ONCE` on any size change while the world viewport is disabled and track the desired update mode in `ViewportWiring`; **must never** read `render_target_update_mode` back as state.
- **Must** implement zoom by changing logical size and the container scale; **must never** set `Camera2D.zoom`, tween a scale, or write `Control.scale` on any HUD node, and **must never** apply a non-integer scale to the world container or a HUD layer.
- **Must** set NEAREST in all three places (project default, world viewport member, container `texture_filter = TEXTURE_FILTER_NEAREST`), keep mipmaps off on drawn sheets, set `SubViewport.size` and `TileSet.tile_size` (48x48) explicitly, and set the world viewport's snap and SDR members in code.
- **Must** keep the HUD, menus, hold plate and debug overlay in `CanvasLayer`s on the root viewport with the distinct indices of § 2; **must never** place a `Control` or a HUD node under the world `SubViewport`, and **must never** put reading-face text under an integer-scaled layer.
- **Must** treat `Window.size_changed`, zoom presses and setting changes as dirty marks and re-derive at most once per frame; **must never** re-derive inside the signal handler, and never queue surplus zoom presses.
- **Must** use the `_process` delta divided by `Engine.time_scale` (not `present_time`, not `Time.*`) for the zoom cooldown; **must** use `present_time` for all layer-0 animation.
- **Must** run `CameraSolver.solve` only in the `CAMERA_SOLVE` stage and write the camera only by `canvas_transform` in the `CameraHost`'s `frame_pre_draw` handler, from already-solved values (primary variant); **must never** solve in the handler, add a `Camera2D` to the world tree in the primary variant, set `Camera2D.limit_*`, parent a camera to a body, or let any module other than the pipeline root, `ViewportWiring` or the camera host write a camera or viewport property.
- **Must** convert `Vector2i` to `Vector2` only at `Placement.put` and at the `canvas_transform` write, and keep body-frame coordinates below 2^24 base px (§ 6.2); **must never** write a non-integer-valued presentation coordinate.
- **Must** keep `SolverInput` free of any world-state field (its one geometry field is the read-only `limits: CameraLimits`, written only by `body.projector`); **must never** pass Attention, breach, horror, sound, graph or sanity values to the solver, write `CameraLimits` from the pipeline, or apply the limits clamp anywhere but the solver (inner-box clamp, then limits clamp on the pose, `world_to_hud` never offset while `CLAMPED`; § 6.1a).
- **Must** release the `HoldLayer` shown during `LOADING` only when the kernel is `RUNNING` **and** `ProjectorReadiness.is_ready()`; **must never** release it on `RUNNING` alone (§ 6.4).
- **Must** assert the pinned configuration at startup through `StartupCheck` over a `RenderConfigSnapshot` (a pure function that returns findings) built by a reader that uses explicit typed assignments; **must never** `assert`, `push_error` or `quit` inside the check, **must** report the live renderer and driver rows NOT ASSESSED under a headless display server, and **must** treat a method or driver mismatch in a release build as the fatal "unsupported GPU" screen.
- **Must** pass the render mode only as the `ViewportWiring` constructor argument; **must never** branch on it in any other class.
- **Must** change a draw-call or timing figure only in the GDD's D.16 block (and refresh § 7's mirror), and a rule only by amending § 7; **must** read draw calls per viewport with `get_render_info`; **must never** infer the HUD's count from the frame total.
- **Must** pin the renderer and driver as in § 1, keep `fallback_to_opengl3` and `fallback_to_d3d12` false, and record the printed driver name with every performance result; **must never** rely on the engine default driver or on a silent fallback.
- **Should** unit-test `RenderLayout.derive` against every row of PR-1's table (including 1365x768 and 3440x1440), the zoom ladder and `ZOOM_MIN_K` clamping, `world_to_hud` / `mouse_to_world_aim` as pure functions with no window, and the 2^24 write bound.

**Gotchas (engine behaviours the code must respect).**
1. **Zoom cooldown and `time_scale`.** `_process` delta is scaled by `Engine.time_scale`; the cooldown divides it back out (or the project states `time_scale` is always 1). ADR-0001 leaves the scaling of `delta` UNVERIFIED.
2. **`WorldContainer.size` is stale after a resize.** The container is a `Control` under a non-`Control` parent: it lays out against the window and its `size` refreshes from the viewport's minimum size at the next deferred update. Drawing is unaffected with `stretch = false` (probed), but code that reads `WorldContainer.size` or mouse rects right after a resize sees stale values; read only `scale` and `position`, which the pipeline wrote.
3. **HUD text sharpness at `H = 3`.** Dynamic-font `Label` and `RichTextLabel` under `CanvasLayer.scale = H` are rasterised at font size and then upscaled (blurry, not pixel-snapped) unless a bitmap face or font oversampling is configured. § 2.5 covers the reading face at scale 1; HUD text faces must be bitmap pixel faces or have oversampling set. Not probed; checked at step 11.
4. **Edge-drag cadence.** On Windows the main loop is driven from a timer during the modal edge-drag resize loop (recalled, not verified); the step-11 2 s drag covers it.
5. **Export preset drivers.** Shader Baker bakes only for the drivers enabled in the export preset, so the preset must include Vulkan; Shader Baker is not supported under Compatibility (recalled).
6. **`DRAW_CALLS_IN_FRAME` lags a frame** when read in `_process` (§ 7).
7. **`UPDATE_ONCE` is not written back.** After the one-shot the property still reads `UPDATE_ONCE`; set `UPDATE_DISABLED` yourself (§ 2.2).
8. **Reader casts.** With ADR-0010's warnings-as-errors, `RenderConfigReader` casts every `ProjectSettings.get_setting()` `Variant` with a typed assignment (`var v: int = ...`), never inference or `as`.

## Alternatives Considered

### Alternative 1: Root integer stretch plus script-set container size (candidate (ii))
- **Description**: Keep `canvas_items` / `integer` stretch on the root so the engine computes `k` and the centring offset, and drive the world `SubViewport.size` from script.
- **Pros**: the engine does the default-step arithmetic (probed: exact `(k, k)` and correct offsets at every window); fewer values for the pipeline to compute.
- **Cons**: the single root scale multiplies everything under it, so a HUD at `H` different from `k` and a world at `picture / k_z` need a second multiplier on top of the stretch: the double-scale hazard, with effective scales that become products and, off the default step, floats. The cursor needs the inverse final transform. The Deck's world 2x and HUD 3x, an MVP requirement, cannot be integers.
- **Rejection Reason**: it is correct only at `H = k` and the default zoom (probed), which the Deck violates on day one.

### Alternative 2: `SubViewportContainer.stretch = true` with `stretch_shrink`
- **Description**: Let the container size the viewport (`stretch_shrink = k` on a screen-pixel-sized container, or on a base-unit container under root stretch, candidate (iii)).
- **Pros**: no script-set scale on the container; the integer lives in an int property; probed to size the viewport correctly (shrink 3 on 1920x1080 gives 640x360).
- **Cons**: `stretch_shrink` floors (3840/7 gives 548x308), so a ladder step is not always `picture / k_z`; the container, not the pipeline root, then writes `SubViewport.size`, which weakens the single-writer rule (TR-pxl-003); under root stretch it does not reach `H` different from `k`.
- **Rejection Reason**: not adopted as the primary. It is the recorded fallback if step 11 shows that a scaled `stretch = false` container mis-samples or bleeds, with the single-writer rule restated for the container's `stretch_shrink` (`PipelineRoot` still writes it and asserts the resulting viewport size).

### Alternative 3: Fractional stretch with a sharpening filter, or `Camera2D.zoom`
- **Description**: Scale the whole picture by `min(W/640, H/360)` and sharpen; or zoom with `Camera2D.zoom`.
- **Pros**: fills every window; no letterbox; the simplest wiring.
- **Cons**: breaks the 1 px contour, 2 px band and 2 px fringe widths (art bible 5.6); a non-integer `Camera2D.zoom` is exactly the fractional transform the pipeline exists to forbid; `Camera2D.zoom` shares the unresolved wiring question and has no pixel-snap member.
- **Rejection Reason**: rejected by PR-1 and PR-4 rule 4 before this ADR; recorded so a later change of mind meets the reasons.

### Alternative 4: Compatibility renderer (or Mobile) instead of Forward+
- **Description**: Use OpenGL 3.3 for the broadest hardware support and lower driver risk.
- **Pros**: runs on GPUs below Vulkan 1.2; the headless probe runs on it; cheap on integrated GPUs; no D3D12 / Vulkan driver choice at all.
- **Cons**: `use_hdr_2d` (and therefore the glow and the 0 to 1 % ambient floor the Lighting GDD designs around) is ignored there (probed: Compatibility silently ignores `use_hdr_2d`); Light2D, shadow and shader parity is the open lighting spike step 9; it is not the renderer the reference machines' build string names; it re-opens pass-B figures and the five-shader cap.
- **Rejection Reason**: not adopted; kept as the reversible fallback (§ 1) gated by lighting spike step 9. A reversal is a technical-director decision that re-runs the lighting spike.

### Alternative 5: Direct3D 12 as the Windows driver
- **Description**: Follow the reference's stated 4.6 default.
- **Pros**: the engine's own default for Windows per the reference; better compatibility on old and integrated GPUs; HDR output may be easier on Windows (not used at MVP).
- **Cons**: the reference and the installed binary disagree about the default (OQ-C); the backend is newer; the Steam Deck cannot use it, so Windows and Deck numbers would differ in driver; RM-DESKTOP is specified in Vulkan terms; the compatibility benefit is for hardware below the minimum specification.
- **Rejection Reason**: not adopted as the default; the tie-break rule in § 1 switches to it on measured evidence.

### Alternative 6: Pass B as the default, or the viewport sized to the window
- **Description**: Render lights, shadows and glow at window resolution (the art director's IDEAL), with the viewport at window size and an override.
- **Pros**: smooth interior falloff of the beam; edges 1 to 2 screen px wide.
- **Cons**: up to 9x the lit pixels at 1080p (2.07 M vs 0.23 M) and 22 MB of render target against 3 MB; at the bottom ladder step A already costs the same; an override that does not divide the window gives a non-integer scale (probed); rotated segments are off the grid.
- **Rejection Reason**: A is the default and the art-director's accepted fallback; B stays behind the flag and is adopted only if it fits 16.6 ms. The window-sized viewport is rejected outright for the picture-sized one (§ 4).

### Alternative 7: HUD inside the world viewport with unshaded materials, or a global `RenderLayout` autoload
- **Description**: Draw the HUD in the world viewport unshaded; or expose `k`, `H` and the layout through an autoload.
- **Pros**: one viewport; trivial access.
- **Cons**: unshaded items still receive glow and the full-screen pass, making HR-1 a per-node discipline one forgotten material breaks (the GDD rejected it in PR-3); an autoload breaks registry `kernel_autoload_or_global` and headless tests that need several layouts at once.
- **Rejection Reason**: structural exclusion by viewport is the requirement; layout is injected as a value.

## Consequences

### Positive
- Every scale in the window is an integer chosen by one script, so the world and the HUD can be two different integers (the Deck) and the fractional-transform failure has nowhere to hide.
- Zoom, resize, mode change and setting change are one `RESCALING` path; the geometry is a pure function testable headless against every window in PR-1's table.
- The HUD, menus, hold plate and debug overlay cannot receive world light, glow, the Filter or the jolt, by viewport, not by discipline.
- A and B differ in one class; once B is built the identical-output test is a unit test (MVP builds A only).
- The renderer and driver are pinned, asserted by effective value, and reversible by a named spike result with a named cost.
- The draw-call and timing figures are one nested table with owners, the open-chart arithmetic is made explicit, and zoom-out steps are withheld rather than content cut.
- The cursor needs no inverse final transform, and the HUD no root stretch. The camera, the HUD's `world_to_hud` and the Avatar presenter read one solved pose per frame, so there is no HUD-to-world lag by construction.

### Negative
- The pipeline computes centring, the HUD rect and the letterbox itself instead of getting them from the engine's stretch (a few lines of pure arithmetic, tested against the engine's probed offsets).
- A scaled `stretch = false` container is a wiring not exercised by the engine's own templates; step 11 must prove its sampling and edge behaviour.
- A ladder field that does not divide the picture leaves a few black pixels inside it (D.5).
- Pass B costs memory and lit pixels and its viewport must be sized to the picture, not the window.
- The renderer and driver are provisional, so Lighting and the performance budgets are conditional until the spike steps (lighting 1, 2, 9; pipeline 11, 12) have run.
- The world viewport is SDR at MVP, so any ambient floor or glow that needs a float framebuffer waits on lighting spike steps 7 and 8 and on the HDR contingency (a renderer-specific resolve material, § 2.4).
- Headless CI cannot assert the renderer, the driver, draw calls or pixels; those are windowed and manual with retained evidence until a windowed runner exists.
- The camera write depends on `frame_pre_draw`, whose ordering is probed only on a dev build; its fallback (the write moves to the end of `PipelineRoot._process`) needs no new constant.
- `PipelineRoot` is now also the presentation host, so its `_process` is the one function every presentation stage runs inside: a larger blast radius than a pure geometry writer, kept testable by the pure driver.
- Pass B is specified but not built at MVP; its figures stay unmeasured until lighting spike step 2.

## Risks
- **`frame_pre_draw` behaves differently on a release export, under D3D12, on the Deck, with a minimised window or under `low_processor_mode`.** Mitigation: the handler is idempotent, branch-free and does no solving, so a skipped or late call costs at most a one-frame canvas lag and nothing accumulates; step 12 re-runs the probe on a release export; fallback: the write moves to the last statement of `PipelineRoot._process`.
- **A scaled container blurs, bleeds an edge pixel or ignores its `texture_filter`.** Mitigation: three-place NEAREST; step 11 screenshots the first frame after every resize and every ladder step with a textured sheet, a one-pixel checker and an odd window size (a flat fill already passes); fallback Alternative 2.
- **The root default stretch or driver changes in a later 4.7.x patch or by a project-template re-save.** Mitigation: effective-value assertion plus the headless `project.godot` test; a key whose required value differs from the default is required in the file.
- **Forward+ or Vulkan turns out wrong on a reference machine.** Mitigation: the gate and tie-break in § 1; the reversal cost is stated; budgets are recorded with the printed driver.
- **Edge-drag resize allocates a viewport every frame.** Mitigation: reallocate only when `field_px` or `k_z` changes; translate otherwise; budget 4 ms; step 11 drags for 2 s.
- **Windows display scaling or `allow_hidpi` gives a window size that is not physical pixels.** Mitigation: `k` derived from the window pixel size only; step 11 logs it at 125 % and 150 %.
- **A subscriber writes a viewport property, or a `Control` slips under the world viewport.** Mitigation: the scene-tree lint is a build gate; the debug tree walk; the single-writer guideline (lint rules ADR-0010).
- **The draw-call table's open-chart row (93) is silently exceeded by a chart plate with the HUD drawn (103).** Mitigation: the row states the hidden-HUD rule; review of the GDD's D.16 block, which must state the rule where the 93 is stated (sync item).
- **A misconfigured `project.godot` ships.** Mitigation: the dev halt, the release diagnostics header and the two-tier CI test with one-key-at-a-time fixtures.
- **The world viewport is switched to HDR without the resolve, shipping unconverted linear colour (probed `(55, 13, 147)` for `(128, 64, 200)`).** Mitigation: `StartupCheck` asserts `use_hdr_2d == false`; the windowed palette read-back test of § 5.5(d); the contingency of § 2.4 is the only sanctioned path.
- **The engine silently falls back to Compatibility on a user's machine.** Mitigation: `fallback_to_opengl3 = false` and `fallback_to_d3d12 = false` are required keys, and a release build treats a method or driver mismatch as a fatal "unsupported GPU" screen (§ 1).
- **Someone moves the camera solve back into the `frame_pre_draw` handler.** Mitigation: the lint "no `CameraSolver.solve` outside the camera stage" (ADR-0010 `PRES` pack) and the step-12 zero-disagreement test across 1,000 frames.

## GDD Requirements Addressed

**Requirements owned by this ADR (26; every row below cites ADR-0008 in its last column in `tr-baseline.md`).**

| GDD System | Requirement | How This ADR Addresses It |
|------------|-------------|--------------------------|
| pixel-perfect-render-pipeline-camera.md | TR-pxl-001: 640x360 logical canvas at integer `k = floor(min(W/640, H/360))` from window pixel size, minimum 1280x720, never a non-integer world scale | § 3 `RenderLayout.derive`, § 5.1 pins (`min_size`, no DPI), § 2 integer-only wiring |
| pixel-perfect-render-pipeline-camera.md | TR-pxl-002: world in its own `SubViewport` in a container; hold (1), HUD (2), menus (3 and 4), debug (5) are `CanvasLayer`s receiving no world light, glow, ambient, occlusion, Filter or jolt | § 2 stack and § 2.3 structural exclusion |
| pixel-perfect-render-pipeline-camera.md | TR-pxl-003: one pipeline root is the only writer of viewport size, override and camera limits and the only `HudLayout` caller; any change fires one `RESCALING` frame | § 2.6 single writer, § 3.2 one `RESCALING` path. Reworded 2026-10-29: the pipeline root is the only writer of viewport size, override and layer geometry and the only `HudLayout` caller; the raw camera-limits rectangle (`CameraLimits`) has one writer, `body.projector` (ADR-0034 § 9), and the pipeline derives `rect_eff` from it and owns the clamp (§ 6.1a) |
| pixel-perfect-render-pipeline-camera.md | TR-pxl-004: nearest in three places; mipmaps off; `SubViewport.size` and `TileSet.tile_size` explicit | § 2.2 (three places, two enums, probed defaults), § 5.1 |
| pixel-perfect-render-pipeline-camera.md | TR-pxl-010: pure `CameraSolver` (RefCounted, float64 scalar state, headless-testable), hosted by a top-level presentation node that writes the world viewport `canvas_transform` from `pose_q` in a `frame_pre_draw` handler (the solve itself runs in the `CAMERA_SOLVE` stage; fallback: the write moves to the end of `PipelineRoot._process`), no `Camera2D.limit_*`, solver the only clamp, no `Camera2D` under a physics body | § 6.1 (solver in stage 1), § 6.2 (handler does two writes; ordering probed on a dev build, release export UNVERIFIED, step 12; `PRESENTATION_LATE` withdrawn) |
| pixel-perfect-render-pipeline-camera.md | TR-pxl-011: camera responds only to player-owned inputs; identical trace gives identical camera trace in calm, tell, breach; the Filter cannot write a camera or viewport property | § 6.1 `SolverInput` type excludes world state; § 2.6 |
| pixel-perfect-render-pipeline-camera.md | TR-pxl-012: first-order exponential follow, `tau_follow` 0.18 s, `DT_MAX`, `SETTLE_EPS`, quantised after smoothing, agreement across 30/60/144 fps within 1 base px, D.6b fallback | § 6.1 fixes the solver interface, float64 scalar state, `dt` clamp; the formula and the D.6b fallback stay the GDD's (parameters loaded by ADR-0007) |
| pixel-perfect-render-pipeline-camera.md | TR-pxl-013: look-ahead reads local raw aim, bounded, run-weighted by committed loco state, zero under reduced motion, build invariants | § 6.1 `SolverInput` carries the raw aim and zoom step; invariants checked by `StartupCheck` over `SolverParams` (the Section G build-invariant check) |
| pixel-perfect-render-pipeline-camera.md | TR-pxl-014: aim-pose loop is open: `mouse_to_world_aim` uses the follow pose `f_q`; cursor in physical px; `mouse_to_world_view` never feeds aim (shared with ADR-0019) | § 3.3: both are `RenderLayout` methods; no stretch so no inverse final transform; lint for the aim rule is ADR-0010's |
| pixel-perfect-render-pipeline-camera.md | TR-pxl-015: camera state never persisted; `CUT` on `LOADING → RUNNING` and body transfer; rebase on weld/detach; restore continuity (the TR text's `post_restore` wording is superseded) | § 6.1: the kernel's `snap_baseline` on entry to `RUNNING` (ADR-0001 § 4, ADR-0003) is the only continuity mechanism; the camera CUTs on `DISCONTINUITY_RESTORE` from `PresentationClock.load_seq`; `cut`, `rebase` |
| pixel-perfect-render-pipeline-camera.md | TR-pxl-016: zoom is a persistent edge-triggered integer ladder implemented by logical size, never `Camera2D.zoom`, one cut per step, at most one per frame and 0.15 s apart, static `ZOOM_MIN_K` withholding | § 3.1 (logical size, rate limit on engine delta, refusal reasons), § 7.2 (static table) |
| pixel-perfect-render-pipeline-camera.md | TR-pxl-019: `world_to_hud` is the single body-space-to-HUD projection; HUD elements anchor to a HUD-root `Control` sized `hud_rect / H` pinned to the `k_base` picture | § 2.5, § 3 (`hud_rect` pinned), § 3.3 |
| pixel-perfect-render-pipeline-camera.md | TR-pxl-023: exactly one full-screen hook: a `ShaderMaterial` slot on the container, assigned only by the pipeline root, null (`PER_ELEMENT`, `NOT_USED`) at MVP | § 2.4 `set_filter_material`, `filter_route` published |
| pixel-perfect-render-pipeline-camera.md | TR-pxl-024: HUD scale `H` independent of `k` and zoom, `H_default = max(k, 3)`, up to 2x where `bottom_row_fit` holds; two integer scales in one window from MVP; reading-face text at final screen-px size | § 2 (candidate (i) refined; stretch disabled, `CanvasLayer.scale` from script, probed), § 2.5, § 3 |
| pixel-perfect-render-pipeline-camera.md | TR-pxl-025: `RENDER_MODE` a single build flag, A default; `get_final_transform().get_scale()` equals `(k,k)` world and `(H,H)` HUD at every window; Forward+; A and B identical in coordinate space, sort, camera and HUD | § 4 (mode is a `ViewportWiring` constructor argument, A built at MVP, picture-sized B), § 5.4 (the readback chain replaces the mode-dependent wording), § 1 |
| pixel-perfect-render-pipeline-camera.md | TR-pxl-026: aspect `keep` default, `expand` post-spike with logical size `2*floor(W/2k)` capped 960x540; manual `k` below Auto forces `keep`; bars true black outside the viewport | § 3 (`RenderLayout.derive`, bars are the clear colour), § 5.1 |
| pixel-perfect-render-pipeline-camera.md | TR-pxl-028: `project.godot` pins stretch, filter, snap, `hdr_2d`, debanding, physics interpolation, clear colour, window override, HDR output; debug assertion reads back effective values; headless test parses `project.godot`; debug tree walk and lints | § 5 in full (pins table with probed defaults, `StartupCheck`, headless parse test, lint list, tree walk); stretch keys flip to `disabled`; adds `fallback_to_opengl3` / `fallback_to_d3d12` false and `thread_model` 1; one two-tier headless test (ADR-0010 `PRJ-02`); `hdr_2d` false on the root and the world at MVP |
| pixel-perfect-render-pipeline-camera.md | TR-pxl-029: pipeline states BOOTING / PRESENTING / RESCALING / TRANSIT_HOLD / PAUSED_FROZEN / HIDDEN (+DEBUG_OVERLAY), camera as three independent fields; PAUSED_FROZEN holds an undimmed frame; RESCALING while updates are disabled forces UPDATE_ONCE; hold plate `#000000` | § 6.4, § 2.2 (`UPDATE_*` modes probed), § 3.2 step 4 |
| pixel-perfect-render-pipeline-camera.md | TR-pxl-030: pipeline `_process` 0.5 ms (solver 0.05), `D_world` 350, `D_root` 63 (93 with the chart open; includes the container composite), sum 1000, total frame 14.9 / 30.0 ms, `RESCALING` 4 ms, targets 3 / 22 MB | § 7 (the single table; pending release-export spike) |
| 2d-lighting-and-occlusion.md | TR-light-019: zoom by world logical size, never `Camera2D.zoom`; all `Light2D` lengths in base px independent of `k_z` under both passes; Lighting has no zoom handler | § 3.1 (zoom changes only the viewport size and container scale); the lengths rule is Lighting's LO-16 and holds because the world canvas is never scaled by a camera |
| perception-filter.md | TR-percfilter-030: container route `PER_ELEMENT` with the container `material` null at MVP, assigned only by the pipeline root, published as `filter_route` / `NOT_USED`; decoy list for Lighting (VS-A) | § 2.4 (the decoy list is the Filter's and Lighting's, not this ADR's) |
| hud-and-suit-layer.md | TR-hud-002: HUD is a `CanvasLayer` (layer 2, group `hud`) in the root viewport, never under the world `SubViewport`; a `Control` under it fails the lint | § 2 (HudLayer), § 5.5 (scene-tree lint, build gate) |
| hud-and-suit-layer.md | TR-hud-003: `H` in `{H_default..2*H_default}`, independent of `k`; HUD code never reads `k`; an `H` change alters no simulation value | § 3 (`H` derivation, `offered` from HUD), § 2.5 |
| hud-and-suit-layer.md | TR-hud-011: `HudLayout` outputs integer HUD-px rectangles by manual snapping at every `H`; `Control.scale` (1,1); `gui_snap_controls_to_pixels` never relied on | § 2.5 (HudRoot, sole `HudLayout` caller), § 5.1 (`snap_controls_to_pixels` left at default and never relied on); the snapping itself is HUD's |
| chart-table-web-map-and-menus.md | TR-charttable-017: routes, seams and rule lines are 1-art-px quads on whole art px (no width-1 line draws on fractional transforms); filament drift in whole-art-px steps | § 2 (no fractional transform exists in the world or on `MenuPixelLayer`); the quad authoring rule is Chart Table's; the 4.7 line-feather change is the reason lines are never used (art bible 8.4) |
| chart-table-web-map-and-menus.md | TR-charttable-030: Display & Comfort changes apply immediately, read back the applied mode, recompute `k`, `H_default`, canvas and panel width live while retaining focus | § 3.2: `apply_display_settings` marks dirty and the next frame's `RESCALING` re-derives; `WiringReadback` provides the applied-mode readback; focus is ADR-0030's |
| pixel-perfect-render-pipeline-camera.md | PR-14: render options are an enumerated set; no gamma, brightness, sharpening or filter option; `apply_display_settings` rejects out-of-set values (TR-pxl-033) | § 5 (closing paragraph) |
| 2d-lighting-and-occlusion.md | GPU ms and VRAM ceilings recorded per target and per pass from release-export measurement before epics (TR-light-039) | § 7.3 |
| chart-table-web-map-and-menus.md | Fractional HUD canvas: every HUD edge on a whole screen pixel, 1-art-px stroke exactly `H` px, integer panel left edge (TR-charttable-037) | Validation Criteria |

**Requirements this ADR loads or checks but does not own (the owner's ADR is in the last column of `tr-baseline.md`).**

| GDD System | Requirement | Owned by | What this ADR supplies |
|------------|-------------|----------|------------------------|
| pixel-perfect-render-pipeline-camera.md | TR-pxl-005: integer base-px placement `floor(prev + (curr-prev)*tick_alpha)`, snap flags a safety net | ADR-0009 | the snap members set in code (§ 2.2) and the tree-walk check for fractional positions (§ 5.5) |
| pixel-perfect-render-pipeline-camera.md | TR-pxl-006, 007, 008: Y-sort key and class ranks, door sort edge, z-band order | ADR-0034 (Y-sort and z-bands; ADR-0009 owns placement only) | the world canvas is the body frame (C2) and the draw-call row `D_ysort` (§ 7.1); the key and bands are not decided here |
| pixel-perfect-render-pipeline-camera.md | TR-pxl-009: per-row z-band fallback if the Y-sorted layer exceeds about 300 draws | ADR-0034 | the trigger figure in § 7.1 |
| pixel-perfect-render-pipeline-camera.md | TR-pxl-017, 018: camera limits rectangle set by a PROJECTOR; world canvas is the current body's frame, parallax driven by the camera host | ADR-0034 (PROJECTOR shape and limits; ADR-0009 owns placement only) | the solver reads the raw `CameraLimits` as `SolverInput.limits`, derives `rect_eff` and owns the clamp order, `CLAMPED` and JOLT (§ 6.1a; the traceability row had attributed these to ADR-0034), the host drives parallax layers with whole-base-px offsets (§ 6.2); the raw rectangle and its single writer are ADR-0034 § 9 |
| pixel-perfect-render-pipeline-camera.md | TR-pxl-020: every visual system is a kernel PRESENTATION subscriber; writes only `CanvasItem` / `Material` / `Light2D` / particle / audio properties, never a viewport or camera property | ADR-0009 | § 2.6 reserves viewport and camera properties to the pipeline root |
| pixel-perfect-render-pipeline-camera.md | TR-pxl-021: `present_time` semantics, the menu and debug layers (3 to 5) on engine time, `SceneTree.paused` is not kernel pause | ADR-0001 | the owner node and advance rule (§ 6.3) |
| pixel-perfect-render-pipeline-camera.md | TR-pxl-022: render rate never changes the tick; identical `state_hash` at 30/60/144/0 fps | ADR-0001 | nothing beyond not reading render state in the simulation |
| pixel-perfect-render-pipeline-camera.md | TR-pxl-027: flash safety clamps computed analytically from modifier state, no GPU readback in the clamp path | ADR-0028 | the table row for the dev-only flash scan cost is outside § 7's release figures |

## Performance Implications
- **CPU**: `PipelineRoot._process` at most 0.5 ms P99 including the state machine and `world_to_hud` for 32 prompts; `CameraSolver.solve` (the `CAMERA_SOLVE` stage) at most 0.05 ms; the `RESCALING` re-derivation at most 4 ms, once per resize frame, with viewport reallocation only when `field_px` or `k_z` changes. All unmeasured until the release-export spike.
- **Memory**: world render targets at most 3 MB (A) or 22 MB (B) at 1080p, scaled by lit pixels at other targets (D.16). `RenderLayout` is a handful of integers per rederivation.
- **Load Time**: negligible; the pipeline root builds the stack in BOOTING step 2 before content loads. Shader Baker reduces first-frame hitching (measured by lighting spike step 10).
- **GPU**: the container composite at most 0.3 ms (1080p) and 1.0 ms (2160p). Pass A shades 230,400 lit pixels at the default field; at the bottom ladder step A costs the same as B (2,073,600 at 1080p).
- **Network**: none. Nothing here is replicated; each client has its own window and camera.

## Migration Plan
No code exists. The current `project.godot` has only `renderer/rendering_method="forward_plus"` and a feature list; the pinned block of § 5.1 (every **required** row, now including `fallback_to_opengl3 = false` and `fallback_to_d3d12 = false`) is added by the first Pipeline story. **GDD, TR and reference sync (items returned to the orchestrator, not applied by this ADR):**
- `pixel-perfect-render-pipeline-camera.md`:
  - PR-7: under pass B the viewport is the picture (`field_px * k_z`), not the window; the override must divide the size exactly.
  - PR-15 (1)(2) and the Startup assertions criteria: stretch `mode = disabled`, `scale_mode = integer` required in the file, the assertion reads `DISABLED` plus the wiring readback; the HDR-output key name `display/window/hdr/request_hdr_output` and the debanding key are now sourced; add the fallback keys, `thread_model`, and `use_hdr_2d == false` (MVP).
  - PR-7 and the scale criteria: replace `get_final_transform().get_scale()` by the readback chain of § 5.4.
  - PR-8 item 3 and PR-6 C2: subscribers draw from `driver.frame` stages in the `_process` pass (ADR-0009); `frame_pre_draw` is the camera host's two writes; the camera solve is the `CAMERA_SOLVE` stage.
  - PR-6 C11 / D.14b: the cursor is physical px with no inverse `get_final_transform` under disabled stretch (TR-pxl-014 wording).
  - D.16: name the container composite draw call (probed: counted on the root, one per container); `D_root` is 63 (62 expected at MVP with a null slot) and 93 with the chart open; state the HUD-hidden-under-a-plate rule where the 93 is stated.
  - PR-3 versus art bible 8.4 on where `hdr_2d` is set; PR-9: a resize while `PAUSED_FROZEN` forces the one-shot `UPDATE_ONCE`.
  - Layer table: layers are 1 hold, 2 HUD, 3 menu pixel face, 4 menu reading face, 5 debug (distinct indices).
- `docs/architecture/tr-baseline.md`: TR-pxl-010 (fallback wording), TR-pxl-014 (no inverse final transform), TR-pxl-015 (snap_baseline wording replaces `post_restore`), TR-pxl-025 ("B: picture size" replaces "B: window size"), TR-pxl-028 (pins, fallback keys, SDR), TR-pxl-030 (`D_root` 63); TR-pxl-025 and TR-pxl-028 are synced alongside PR-7 and PR-15; the owner column of TR-pxl-006 to 009, 017 and 018 re-points from ADR-0009 to ADR-0034 (ADR-0009 owns placement only).
- `chart-table-web-map-and-menus.md`: the menu layers are two sibling `CanvasLayer`s with distinct indices (pixel face at 3 and scale `H`, reading face at 4 and scale 1); `D_L3` counts both; `DisplaySettings` persistence is the Chart Table's host-local `SettingsStore` (CT-19, ADR-0030), not PROFILE.
- `design/art/art-bible.md` 8.4 and `2d-lighting-and-occlusion.md`: `use_hdr_2d` is false on the world viewport at MVP; HDR is the gated contingency of § 2.4 decided by lighting spike steps 7 and 8.
- `docs/architecture/architecture.md`: layer diagram platform row and § 8 step 1 mention "Forward+" and "stretch" as settled; they become provisional and `disabled`; OQ-2 points to this ADR; **Data Flow (around line 279) is redrawn to ADR-0009's stage list with `PipelineRoot` as the single presentation host**.
- `docs/engine-reference/godot/modules/rendering.md` and `breaking-changes.md` (reference corrections, suggested): a project with no driver key reports `vulkan` on 4.7.2 (windowed probe) and the `ProjectSettings` default of `driver.windows` is `vulkan`; the "D3D12 is the Windows default since 4.6" line is probably a new-project-template value (UNVERIFIED).
- Other ADRs (applied by their owners): ADR-0001 (`PresentationClock.state` and `load_seq`, `HOST_SERVICE`, one presentation host in its diagram, the probe wording of § 6.2, no `PRESENTATION_LATE`); ADR-0009 (closes OQ-E, OQ-J and OQ-L citing § 6.2; allow-list entries for `viewport_wiring.gd` and the `CameraHost` handler; the snap-flag wording cites § 2.2); ADR-0005 TR-shipbody-038 and ADR-0010 replace "open" with a citation of § 6.2's coordinate paragraph; ADR-0010 `PRJ-02` names § 5.1 as its key list and § 5.5(a) as its host test; ADR-0006's scope note excludes settings persistence.
- `tests/unit/render/` (accepted by the GDD for the auto tests) holds the `RenderLayout`, `StartupCheck`, the 2^24 write bound and the two-tier `project.godot` tests; the windowed runner is a `/test-setup` item (Pipeline Open Question 22).

## Validation Criteria
- **Spike step 11** passes on the combinations not yet probed: at every window in PR-1's table the readback chain gives exactly `(k_z, k_z)` and `(H, H)`, under pass A (and pass B if built), on a dev build and a release export, **with a textured sheet, a one-pixel checker and an odd window size** (the flat-fill case already passes); the first frame after each resize, zoom step and mode change shows no fractional transform (retained screenshots); a 2 s edge drag produces no fractional frame, shows the expected cadence and stays within the 4 ms `RESCALING` budget; 1365x768 shows true-black residual; **pause then resize shows no black frame; pause with no resize holds the frame; HUD text is sharp at `H = 3`**.
- **Unit tests (headless, no window):** `RenderLayout.derive` matches PR-1's table for every row including 1365x768, 1366x768, 2560x1080 and 3440x1440, centring offsets agree with the probed engine offsets (43,24), (160,90), (0,60), (320,0), (440,0); `derive` does not change when the caller mutates `offered_h` afterwards; the ladder and `ZOOM_MIN_K` clamping; manual `k` below Auto forces `keep`; `H` choice is clamped to the offered subset; `world_to_hud` and `mouse_to_world_aim` examples from D.14 and D.14b; the `canvas_transform` write at `2^24 - 1`, `2^24` and `2^24 + 1`; `StartupCheck` returns exactly one finding naming the key for a snapshot with one wrong value, per key (including `use_hdr_2d`).
- **CI:** the two-tier `project.godot` test (§ 5.5(a)) passes the shipped file and fails, naming the key, for each fixture with one required key removed or one effective value set wrong; the scene-tree and asset lint finds zero violations (distinct layer indices, 48x48 tile sets); no `Camera2D` in the world tree (primary variant); no assignment to `Camera2D.zoom` outside the reset path; no `CameraSolver.solve` outside the camera stage; the dev-only files (Debug Overlay scene, tree-walk, flash-scan) are absent from the release export list.
- **Equivalence (only if pass B is built):** the same input trace under `Mode.A` and `Mode.B` gives identical `k`, `H`, `k_z`, `pose_q` sequence, cut list, Y-sort order and `world_to_hud` outputs; only viewport size, memory and frame time differ.
- **Hook (pipeline spike step 12, gates acceptance):** zero render-pixel disagreement between camera and sprite, and between `world_to_hud` and the world, across 1,000 frames on a **release export** with `frame_pre_draw` as the write point, including a minimised-and-restored window; or the fallback (write at the end of `PipelineRoot._process`) is adopted.
- **Palette read-back (§ 5.5(d)):** a flat fill, a one-pixel checker and a clear-colour pixel read back exactly, per renderer (Forward+ and Compatibility), dev build and release export.
- **Renderer and driver (gates acceptance):** lighting spike steps 1, 2 and 9 run on RM-DESKTOP and RM-DECK with the printed method and driver recorded; the tie-break rule of § 1 is applied to the measured numbers; the recorded choice replaces "provisional" in § 1; the `fallback_to_*` keys are confirmed to turn a failed driver init into a visible error.
- **Budgets:** the P99 table of § 7.3 and the per-viewport draw-call table of § 7.1 are measured on the release export on both machines with the driver name recorded; each withheld zoom step is written into `ZOOM_MIN_K`. Until then performance criteria are neither pass nor fail.
- **Fractional HUD canvas (Chart Table fractional-canvas rule, TR-charttable-037):** on a fractional HUD canvas (for example 426.67 x 266.67 base px at `H` 3) the windowed read-back on the real display path asserts that every HUD glyph, bezel, seam and route edge lands on a whole screen-pixel boundary, that a 1-art-px stroke is exactly `H` screen px on every side, and that the panel left edge is an integer screen px. A failure withholds that HUD scale step rather than adding a filter.

## Related
- Depends on: ADR-0001 (Proposed; with its amendment batch: `PresentationClock.state` and `load_seq`, `HOST_SERVICE`), ADR-0007 (Proposed; `RenderTuning`), ADR-0009 (Proposed; the host model, driver, stages and `Placement`), ADR-0010 (Proposed; paths, lint packs, `PRJ-02`).
- Referenced and enabled: ADR-0004 (change notification: dirty marks only), ADR-0015 (owns the body-frame coordinate bound if extents could exceed 2^24 base px, § 6.2), ADR-0019 (Input Adapter: zoom actions, cursor), ADR-0028 (lighting budget: renderer consumer, the SDR-or-HDR decision, flash-clamp ownership), ADR-0030 (UI framework: the two menu layers, reading face; the Chart Table `SettingsStore` that persists `DisplaySettings`, CT-19), ADR-0034 (PROJECTOR: camera limits rectangle; Y-sort, z-bands and PROJECTOR shape). ADR-0006 is **not** a dependency (display settings are host-local, not PROFILE); ADR-0016's `post_restore` order is not used (the kernel's `snap_baseline` is the only restore-continuity mechanism).
- GDDs: `pixel-perfect-render-pipeline-camera.md` (PR-1 to PR-15, D.1 to D.16, States and Transitions, Performance budget); `2d-lighting-and-occlusion.md` (LO-16, spike table steps 1 to 10); `hud-and-suit-layer.md` (HS-2, HS-3, HS-9); `perception-filter.md` (PF-25, PF-26); `chart-table-web-map-and-menus.md` (CT-11, CT-13, CT-18, CT-19, ASK-CT-12, `LAYER3_DRAW_CALL_BUDGET`); `failure-and-trail-system.md` (`D_FT_MAX`); `design/art/art-bible.md` 8.4, 8.6, 8.10, 8.12.
- `docs/architecture/architecture.md` § Engine Knowledge Gap Summary, § Required ADRs, § Spikes that gate ADRs, § Data Flow, OQ-2, IC-6; `docs/registry/architecture.yaml` (entries proposed by this ADR: see Registry Candidates).

## Registry Candidates
Reviewable entries for `docs/registry/architecture.yaml`, corrected by the 2026-10-28 rulings. **This ADR does not write the registry**: the orchestrator merges these serially once the revisions land. None of these names exists in the registry yet, so all are additions; the three `referenced_by_update` items append this ADR to existing entries (`presentation_clock`, `tick_alpha`, `frame_order`) whose fields are revised by ADR-0001's amendment batch.

**`state_ownership`**

```yaml
  - state: render_geometry_and_viewport_stack
    status: active           # ADR-0008 is Proposed; entries become binding when it is Accepted
    owner_system: pixel-perfect-render-pipeline (PipelineRoot, ViewportWiring, CameraHost)
    adr: docs/architecture/adr-0008-render-pipeline-wiring-and-integer-scaling.md
    interface: "RenderLayout (read-only RefCounted value: k_auto, k_base, k_z, h, field_px, picture_rect, world_rect, hud_rect, hud_canvas; no version field) via PipelineRoot.layout, with PipelineRoot.layout_version as the change counter; world SubViewport size and override, WorldContainer position and scale, every CanvasLayer scale and offset and HudRoot size are written only by PipelineRoot and ViewportWiring; the world canvas_transform and parallax offsets only by the CameraHost frame_pre_draw handler; presentation, the Filter, Lighting, HUD and Chart Table read the layout and call world_to_hud; never a state block, never saved, never hashed"
    write_access: pipeline-root-only
    referenced_by:
      - docs/architecture/adr-0008-render-pipeline-wiring-and-integer-scaling.md
    added: 2026-10-28
    revised: ""
  - state: present_time
    status: active           # ADR-0008 is Proposed
    owner_system: pixel-perfect-render-pipeline (PipelineRoot)
    adr: docs/architecture/adr-0008-render-pipeline-wiring-and-integer-scaling.md
    interface: "PipelineRoot.present_time (read-only float seconds): advanced in step 1 of PipelineRoot._process by the delta while PresentationClock.state is RUNNING, SAVING or LOADING-with-hold, frozen while PAUSED, never scaled by reduced motion; delivered to presenters as PresentFrame.present_time; the clock for all layer-0 cosmetic animation; presentation-only, never hashed or saved"
    write_access: pipeline-root-only
    referenced_by:
      - docs/architecture/adr-0008-render-pipeline-wiring-and-integer-scaling.md
      - docs/architecture/adr-0001-kernel-tick-driver-and-frame-order.md   # handed ownership to the pipeline root
    added: 2026-10-28
    revised: ""
```

**`interfaces`**

```yaml
  - contract: render_layout
    status: active           # ADR-0008 is Proposed
    pattern: direct_call
    producer: pixel-perfect-render-pipeline (PipelineRoot)
    consumers:
      - hud-and-suit-layer
      - in-world-instruments
      - chart-table-and-menus
      - input-adapter
      - 2d-lighting-and-occlusion
    adr: docs/architecture/adr-0008-render-pipeline-wiring-and-integer-scaling.md
    signal_signature: "RenderLayout.derive(window_px: Vector2i, settings: DisplaySettings, zoom_k: int, tuning: RenderTuning, offered_h: Array[int]) -> RenderLayout (static, pure; offered_h duplicated on entry); world_to_hud(body_pos: Vector2i, pose_q: Vector2i) -> Vector2i (the only form); mouse_to_world_aim(cursor_px: Vector2i, f_q: Vector2i) -> Vector2 (open loop, D.14b); mouse_to_world_view(cursor_px: Vector2i, pose_q: Vector2i) -> Vector2 (never for beam aim); PipelineRoot.layout (read-only), PipelineRoot.layout_version (read-only)"
    referenced_by:
      - docs/architecture/adr-0008-render-pipeline-wiring-and-integer-scaling.md
    added: 2026-10-28
    revised: ""
  - contract: filter_hook_slot
    status: active           # ADR-0008 is Proposed
    pattern: direct_call
    producer: perception-filter
    consumers:
      - pixel-perfect-render-pipeline
    adr: docs/architecture/adr-0008-render-pipeline-wiring-and-integer-scaling.md
    signal_signature: "PipelineRoot.set_filter_material(material: ShaderMaterial) -> void (sole writer of SubViewportContainer.material; null at MVP because the world viewport is SDR; with the HDR contingency adopted the slot always holds the resolve material and the Filter shader includes the resolve); published filter_route: PER_ELEMENT | NOT_USED"
    referenced_by:
      - docs/architecture/adr-0008-render-pipeline-wiring-and-integer-scaling.md
    added: 2026-10-28
    revised: ""
  - contract: zoom_request
    status: active           # ADR-0008 is Proposed
    pattern: direct_call     # host-local presentation call, not a command; never saved
    producer: input-adapter
    consumers:
      - pixel-perfect-render-pipeline
    adr: docs/architecture/adr-0008-render-pipeline-wiring-and-integer-scaling.md
    signal_signature: "PipelineRoot.request_zoom_step(direction: int) -> void (+1 in, -1 out; edge-triggered; one accepted step per frame and >= 0.15 s apart on the _process delta divided by Engine.time_scale; surplus discarded); PipelineRoot.apply_display_settings(settings: DisplaySettings) -> void (marks dirty only; DisplaySettings is persisted host-locally by the Chart Table SettingsStore, CT-19, ADR-0030)"
    referenced_by:
      - docs/architecture/adr-0008-render-pipeline-wiring-and-integer-scaling.md
    added: 2026-10-28
    revised: ""
```

**`api_decisions`**

```yaml
  - purpose: renderer_and_windows_driver
    status: active           # ADR-0008 is Proposed; PROVISIONAL until lighting spike steps 1, 2, 9 and pipeline spike steps 11, 12
    api: "rendering/renderer/rendering_method = forward_plus; rendering/rendering_device/driver.windows = vulkan and driver.linuxbsd = vulkan, asserted by effective ProjectSettings value in CI and by the live driver name at boot; required keys fallback_to_opengl3 = false and fallback_to_d3d12 = false (fallback_to_vulkan = false if the D3D12 reversal is taken); rendering/driver/threads/thread_model held at default 1 and asserted; a release build treats a method or driver mismatch as a fatal unsupported-GPU screen"
    not: "Compatibility or Mobile renderer; Direct3D 12 on Windows unless the spike tie-break fires (Vulkan fails a correctness check D3D12 passes, or D3D12 beats Vulkan P99 CPU frame time by >= 10 % on RM-DESKTOP); relying on the engine default driver; silent driver fallback; thread_model 2; a release build that logs a wrong method and continues"
    adr: docs/architecture/adr-0008-render-pipeline-wiring-and-integer-scaling.md
    reason: "Forward+ for Light2D and glow parity and the gated HDR contingency (Compatibility ignores use_hdr_2d, probed); RM-DESKTOP and RM-DECK are specified in Vulkan terms; a key-absent project reports vulkan on 4.7.2 (windowed probe) while the reference says D3D12 since 4.6, so only an effective-value assertion is safe; the fallback keys default true and would silently ship a Compatibility look"
    referenced_by:
      - docs/architecture/adr-0008-render-pipeline-wiring-and-integer-scaling.md
    added: 2026-10-28
    revised: ""
  - purpose: world_viewport_wiring
    status: active           # ADR-0008 is Proposed
    api: "Root Window content_scale_mode DISABLED; world SubViewport (size field_px under pass A, field_px * k_z with size_2d_override = field_px and size_2d_override_stretch = true under pass B; use_hdr_2d = false at MVP; UPDATE_ONCE mandatory on any resize while UPDATE_DISABLED, desired mode tracked in ViewportWiring) inside a SubViewportContainer (stretch false, scale (k_z, k_z) under A, texture_filter NEAREST, material null at MVP); CanvasLayers hold 1, HUD 2 (scale (H, H)), MenuPixel 3, MenuReading 4, Debug 5, distinct indices; ViewportWiring.Mode {A, B} is a constructor argument, pass A built at MVP; all written only by PipelineRoot / ViewportWiring"
    not: "root canvas_items / integer stretch to obtain k, H or k_z; SubViewportContainer.stretch_shrink as the primary (floors, 3840/7 gives 548); a pass-B viewport sized to the window (an override that does not divide the size gives a non-integer scale); a Control under the world viewport; two CanvasLayers sharing an index; reading render_target_update_mode back as state; Mode as a const read inside the class"
    adr: docs/architecture/adr-0008-render-pipeline-wiring-and-integer-scaling.md
    reason: "Probed: root integer stretch is exact only at H = k and the default zoom; CanvasLayer.scale and a stretch=false container scale give exact independent integers; an override that does not divide the size gives 2.0029 not 2; a resize of a disabled viewport blanks it; same-index layers rely on a sibling-order tie-break"
    referenced_by:
      - docs/architecture/adr-0008-render-pipeline-wiring-and-integer-scaling.md
    added: 2026-10-28
    revised: ""
  - purpose: world_viewport_sdr_at_mvp
    status: active           # ADR-0008 is Proposed; HDR decided by lighting spike steps 7 and 8
    api: "World SubViewport use_hdr_2d = false and root rendering/viewport/hdr_2d = false at MVP, so palette fills read back exact; the HDR path (use_hdr_2d = true) is adopted only by amending ADR-0008 together with a mandatory PipelineRoot-owned resolve material in the container slot (exact linear-to-sRGB plus a clamp or tonemap policy, included by the Filter shader, gated on the effective rendering method, absent under Compatibility); windowed palette read-back test per renderer"
    not: "use_hdr_2d = true on a viewport drawn through a SubViewportContainer without a resolve (outputs unconverted linear colour: (128,64,200) reads (55,13,147)); a resolve under Compatibility (double-converts to (188,137,229)); use_debanding unless flat fills read back bit-exact with it on"
    adr: docs/architecture/adr-0008-render-pipeline-wiring-and-integer-scaling.md
    reason: "Windowed probe 4.7.2 Forward+: an HDR viewport on the SDR root is not sRGB-converted; Compatibility silently ignores use_hdr_2d, so the fix is renderer-specific"
    referenced_by:
      - docs/architecture/adr-0008-render-pipeline-wiring-and-integer-scaling.md
    added: 2026-10-28
    revised: ""
  - purpose: zoom_via_logical_size
    status: active           # ADR-0008 is Proposed
    api: "Zoom step k_z in [ZOOM_MIN_K, k_base + 2] changes the world viewport logical size and the container scale in one RESCALING frame (a camera cut); ZOOM_MIN_K is a typed row table (k_base, target class, min_k) in RenderTuning loaded by ADR-0007's pipeline"
    not: "Camera2D.zoom, any tweened or non-integer scale, a runtime-measured ladder, queued zoom presses"
    adr: docs/architecture/adr-0008-render-pipeline-wiring-and-integer-scaling.md
    reason: "Camera2D.zoom has no pixel-snap member; Lighting lengths stay in base px independent of k_z (LO-16); a mid-session ladder change would be a player-visible surprise"
    referenced_by:
      - docs/architecture/adr-0008-render-pipeline-wiring-and-integer-scaling.md
    added: 2026-10-28
    revised: ""
  - purpose: camera_canvas_transform_write
    status: active           # ADR-0008 is Proposed; ordering probed on a 4.7.2 dev build, release export UNVERIFIED (spike step 12)
    api: "CameraSolver.solve runs in the CAMERA_SOLVE stage (ADR-0009 stage 1) of PipelineRoot._process and publishes pose_q, f_q, jolt, parallax offsets and the world_to_hud inputs; the CameraHost RenderingServer.frame_pre_draw handler only writes the world SubViewport canvas_transform and the parallax offsets from the solved values (idempotent, branch-free, safe if skipped); primary variant has no Camera2D in the world tree; fallback if step 12 rejects the hook: the write moves to the last statement of PipelineRoot._process after driver.frame (no new ProcessPriority constant)"
    not: "solving inside the frame_pre_draw handler (HUD, Avatar presenter and mouse_to_world_aim would read the previous frame's pose); a Camera2D scroll on its own clock; Camera2D.limit_*; a camera parented to a body; any writer other than the pipeline root, ViewportWiring or the camera host; subscribers drawing from their own _process; ProcessPriority.PRESENTATION_LATE"
    adr: docs/architecture/adr-0008-render-pipeline-wiring-and-integer-scaling.md
    reason: "Probed: frame_pre_draw fires after every _process and before the draw, so a solve in it is invisible to the HUD stage; a write in the handler or in _process is drawn the same frame"
    referenced_by:
      - docs/architecture/adr-0008-render-pipeline-wiring-and-integer-scaling.md
    added: 2026-10-28
    revised: ""
  - purpose: nearest_filter_three_places
    status: active           # ADR-0008 is Proposed
    api: "NEAREST set in three places: project default_texture_filter = 0, world SubViewport.canvas_item_default_texture_filter = Viewport.DEFAULT_CANVAS_ITEM_TEXTURE_FILTER_NEAREST (0), SubViewportContainer.texture_filter = CanvasItem.TEXTURE_FILTER_NEAREST (1); mipmaps off on drawn sheets; SubViewport.size and TileSet.tile_size (48x48) set explicitly; snap_2d members set on the world viewport in code (they snap in base-pixel space in both passes; nothing relies on them)"
    not: "inheriting the container filter (probed default 0 = PARENT_NODE), relying on project settings reaching the SubViewport, the 512x512 SubViewport default, a 16x16 tile_size"
    adr: docs/architecture/adr-0008-render-pipeline-wiring-and-integer-scaling.md
    reason: "Probed: project default filter is linear (1), SubViewport members default false / 1 independently, two differently numbered enums; any miss blurs the pass-A upscale"
    referenced_by:
      - docs/architecture/adr-0008-render-pipeline-wiring-and-integer-scaling.md
    added: 2026-10-28
    revised: ""
  - purpose: canvas_coordinate_exactness
    status: active           # ADR-0008 is Proposed
    api: "Presentation coordinates are integer-valued float32, exact for |v| <= 2^24 base px; Vector2i is converted to Vector2 only at Placement.put and at the canvas_transform write; pose_q is Vector2i quantised from the solver's float64 scalars before that conversion; world and body-frame extents stay below 2^24 base px (ADR-0015 owns the bound if body-frame extents could exceed it; the rebase mechanism keeps coordinates inside it); unit test at 2^24 - 1, 2^24 and 2^24 + 1"
    not: "a non-integer-valued presentation coordinate; silent float32 rounding of a canvas origin"
    adr: docs/architecture/adr-0008-render-pipeline-wiring-and-integer-scaling.md
    reason: "Transform2D, Node2D.position and Control.position are float32 in the standard build; closes the float32 canvas exactness reference of ADR-0005 TR-shipbody-038 and ADR-0010"
    referenced_by:
      - docs/architecture/adr-0008-render-pipeline-wiring-and-integer-scaling.md
    added: 2026-10-28
    revised: ""
```

**`forbidden_patterns`**

```yaml
  - pattern: viewport_geometry_written_outside_pipeline_root
    status: active           # ADR-0008 is Proposed
    description: "Only PipelineRoot and ViewportWiring (src/presentation/pipeline/viewport_wiring.gd) write SubViewport.size and size_2d_override*, the world container's position, scale, stretch_shrink and material, any CanvasLayer scale, offset or transform, and HudRoot.size; only the CameraHost frame_pre_draw handler writes the world canvas_transform and the parallax offsets; no subscriber, the Filter, Lighting, HUD or Chart Table writes a viewport or camera property; no code but RenderLayout.derive computes k, H, world_to_hud or the cursor mapping."
    why: "Two writers or a re-derived projection produce fractional transforms and disagreeing HUD anchors (TR-pxl-003, TR-pxl-019); the pixel-integrity promise needs one source."
    adr: docs/architecture/adr-0008-render-pipeline-wiring-and-integer-scaling.md
    added: 2026-10-28
  - pattern: root_stretch_or_camera2d_zoom_for_scale
    status: active           # ADR-0008 is Proposed
    description: "No root canvas_items or integer stretch to obtain k, H or k_z; no Camera2D.zoom, no tweened or non-integer scale, no Control.scale on HUD nodes, no scale animation anywhere in the world or HUD."
    why: "Root stretch is exact only when H equals k at the default zoom (probed), so it cannot host the Deck's world 2x and HUD 3x; non-integer scale is the failure the pipeline exists to prevent."
    adr: docs/architecture/adr-0008-render-pipeline-wiring-and-integer-scaling.md
    added: 2026-10-28
  - pattern: control_or_reading_text_under_world_or_scaled_layer
    status: active           # ADR-0008 is Proposed
    description: "No Control and no node in group hud under the world SubViewport; no reading-face (vector) text under an integer-scaled CanvasLayer or the world container; reading-face text lives on a scale (1,1) layer (MenuReadingLayer, layer 4) with sizes in screen px; HUD text uses bitmap faces or configured oversampling."
    why: "A Control under the world receives world light and the Filter (HR-1); text under an integer scale is bitmap-stretched and blurs."
    adr: docs/architecture/adr-0008-render-pipeline-wiring-and-integer-scaling.md
    added: 2026-10-28
  - pattern: render_mode_branch_outside_viewport_wiring
    status: active           # ADR-0008 is Proposed
    description: "ViewportWiring.Mode (pass A or B) is a constructor argument read only inside ViewportWiring; no other class branches on it, and no rule, formula or dependent may assume which pass is live."
    why: "A and B must be identical in coordinate space, sort contract, camera contract and HUD (PR-7); a leak of the mode makes the spike result a rewrite."
    adr: docs/architecture/adr-0008-render-pipeline-wiring-and-integer-scaling.md
    added: 2026-10-28
  - pattern: camera_solve_inside_frame_pre_draw
    status: active           # ADR-0008 is Proposed
    description: "CameraSolver.solve is called only from the CAMERA_SOLVE stage inside PipelineRoot._process; the frame_pre_draw handler performs the two canvas writes from already-solved values and never solves; no presenter defines its own _process; PipelineRoot is the only PresentationHost."
    why: "frame_pre_draw fires after every _process (probed), so a solve in it leaves world_to_hud, the beam pose and the aim mapping one frame behind the world (PR-6 C11, TR-pxl-019)."
    adr: docs/architecture/adr-0008-render-pipeline-wiring-and-integer-scaling.md
    added: 2026-10-28
  - pattern: silent_renderer_or_driver_fallback
    status: active           # ADR-0008 is Proposed
    description: "project.godot must set rendering_device/fallback_to_opengl3 = false and fallback_to_d3d12 = false while Vulkan is the pin; a release build that finds a wrong effective rendering method or driver shows a fatal unsupported-GPU screen and never logs and continues."
    why: "A silent degrade to Compatibility ships a different look (use_hdr_2d ignored, Light2D and glow differ, colour path changes) with only a diagnostics-header line."
    adr: docs/architecture/adr-0008-render-pipeline-wiring-and-integer-scaling.md
    added: 2026-10-28
```

**`performance_budgets`**

```yaml
  - target_fps: 60
    total_frame_budget_ms: 14.9   # 90 % of the 16.6 ms ceiling (ruling R12); P99 CPU frame time, release export, warm-up discarded
    platform: PC                  # RM-DESKTOP
    adr: docs/architecture/adr-0008-render-pipeline-wiring-and-integer-scaling.md   # ADR-0008 is Proposed; figure pending spike
    added: 2026-10-28
    revised: ""
  - target_fps: 30
    total_frame_budget_ms: 30.0   # 90 % of the 33.3 ms ceiling (ruling R12); RM-DECK handheld, 30 fps target
    platform: Console             # nearest registry platform value; this is the Steam Deck
    adr: docs/architecture/adr-0008-render-pipeline-wiring-and-integer-scaling.md   # ADR-0008 is Proposed; figure pending spike
    added: 2026-10-28
    revised: ""
  - system: render_pipeline_root
    status: active           # ADR-0008 is Proposed; all figures pending release-export spike on RM-DESKTOP and RM-DECK; figures owned by the Pipeline GDD D.16, mirrored in ADR-0008 s7
    budget_ms: 0.5
    adr: docs/architecture/adr-0008-render-pipeline-wiring-and-integer-scaling.md
    notes: "PipelineRoot._process P99 <= 0.5 ms including world_to_hud for 32 prompts and driver.frame dispatch (CameraSolver.solve <= 0.05 ms in the CAMERA_SOLVE stage; the frame_pre_draw handler is two assignments); RESCALING re-derivation <= 4 ms in one frame; container composite GPU <= 0.3 ms at 1080p, <= 1.0 ms at 2160p. DRAW-CALL TABLE (unit is draw calls, not ms; read per viewport with Viewport.get_render_info, which lags one frame in _process): D_world <= 350 (floor 4 + marks 20 + ysort 300 + caps 4 + fx 22; Failure and Trail D_FT_MAX 38 nested inside), D_root <= 63 chart closed (HUD 60 + hold 1 + filter 1 + container composite 1; probed: the composite is counted on the root; 62 expected with a null slot at MVP), <= 93 chart open (holds only because the HUD is hidden under a full-bleed plate; Chart Table D_L3 <= 40), D_world + D_root <= 1000; zoom-out steps: sum <= 1000 only plus the P99 frame ceilings, failing steps withheld via ZOOM_MIN_K. Render targets <= 3 MB (A) / 22 MB (B) at 1080p."
    referenced_by:
      - docs/architecture/adr-0008-render-pipeline-wiring-and-integer-scaling.md
    added: 2026-10-28
    revised: ""
```

**`referenced_by_update`**

```yaml
      - docs/architecture/adr-0008-render-pipeline-wiring-and-integer-scaling.md   # under interfaces: contract presentation_clock; PipelineRoot reads tick_alpha, T, running, and the ADR-0001-amendment fields state and load_seq (state drives present_time; load_seq drives the camera CUT through the driver); no KernelStateView type
      - docs/architecture/adr-0008-render-pipeline-wiring-and-integer-scaling.md   # under state_ownership: state tick_alpha; the CAMERA_SOLVE stage reads it only through PresentFrame.alpha (ADR-0009)
      - docs/architecture/adr-0008-render-pipeline-wiring-and-integer-scaling.md   # under api_decisions: purpose frame_order; PipelineRoot._process at ProcessPriority.PRESENTATION is the single PresentationHost; no PRESENTATION_LATE; the frame_pre_draw handler runs after all _process (probed)
```

## Open Questions
(OQ-D, OQ-E and OQ-F are closed or withdrawn; their letters are not reused. See the Review Log.)
- **OQ-A.** Renderer method and Windows driver stay provisional until lighting spike steps 1, 2 and 9 and pipeline spike steps 11 and 12 have run. Owner: the spike; the technical director rules on a reversal.
- **OQ-B.** Whether a scaled `stretch = false` `SubViewportContainer` samples with NEAREST and without an edge bleed for **textured and non-flat content**, a one-pixel checker, pass B (if built), an odd window size, a fractional container position, Compatibility and a release export, and whether `stretch_shrink` (Alternative 2) is the better primary. The flat-fill case already passes (probed: exact colour, exactly 24 px for an 8x4 viewport at scale 3, no bleed). Spike step 11.
- **OQ-C.** Reference versus binary: windowed probe on 4.7.2 shows a project with **no** `driver.windows` key reports `driver = vulkan` / `forward_plus`, and the `ProjectSettings` default is `vulkan` (`linuxbsd` `vulkan`, `macos` `metal`). The reference's "D3D12 is the Windows default since 4.6" is therefore not what a key-absent project gets; it is most plausibly a value written by the new-project template (UNVERIFIED; the project-creation flow was not opened to confirm). Suggested corrections: `docs/engine-reference/godot/modules/rendering.md` and `breaking-changes.md`. This ADR's effective-value assertion handles both cases. Nothing further is claimed.
- **OQ-G.** Whether a disabled viewport holds its last texture **when no resize occurs** (the `PAUSED_FROZEN` hold). The resize case is resolved by probe (it blanks; `UPDATE_ONCE` is mandatory) and the same-index layer question is resolved by distinct indices. Spike step 11 checks pause-without-resize.
- **OQ-H.** Whether the ambient floor and glow the Lighting GDD needs can be met with the world viewport SDR (the MVP default), or require `use_hdr_2d = true` with the resolve material of § 2.4: lighting spike steps 7 and 8 decide, and the art bible 8.4 text needs the same correction as the GDD.

## Review Log
- **2026-10-28 joint review (godot-specialist engine review and technical-director TD-ADR review), revised the same day (autonomous, action "Revise flagged items"; status stays Proposed).** The earlier "review unavailable" entry is replaced by the real outcome. Engine verdict BLOCKING; TD-ADR verdict CONCERNS. Every finding below was applied unless noted; the engine-behaviour claims that were probed are marked as such in the text, and the rest remain UNVERIFIED.
  - **E1 (BLOCKING, applied).** Chose option (a): `use_hdr_2d = false` on the world viewport at MVP (§ 2.2, § 5.1, § 5.3), container material null, flat-palette guarantee kept. Option (b) is written as a gated contingency (§ 2.4: mandatory PipelineRoot-owned resolve material, composed with the Filter by inclusion, gated on the effective method, `D_filter` becomes 1). Added a `StartupCheck` row, the windowed palette read-back test (§ 5.5(d)), and the probe numbers in Engine Compatibility. OQ-H updated.
  - **E2 (BLOCKING, applied) and T1 (BLOCKING, applied) and TD-03 (applied).** Rewrote § 6.1, § 6.2, § 6.3, § 7.3, the diagram, the stack tree and the Key Interfaces: `PipelineRoot` is the single `PresentationHost` and owns the `PresentationDriver`; its one `_process` at `ProcessPriority.PRESENTATION` does present_time, zoom cooldown, `RESCALING`, `driver.frame`; `CameraSolver.solve` runs in `CAMERA_SOLVE` (stage 1) and publishes; the `frame_pre_draw` handler only writes `canvas_transform` and the parallax offsets (idempotent, branch-free, safe if skipped). Deleted every "subscribers draw from their own `_process`" sentence; the literal priority 0 is replaced by `ProcessPriority.PRESENTATION`. PRESENTATION_LATE withdrawn; fallback is the write at the end of `PipelineRoot._process`. PR-6 C2 and the TR-pxl-010 row updated.
  - **E3 (applied) and TD-20 (applied).** The UNVERIFIED frame_pre_draw text replaced by the probe result and its limits; OQ-D withdrawn; release-export re-run kept in spike step 12; `thread_model` pinned at 1 in § 5.1; snap-flag space recorded in § 2.2 and § 4.
  - **E4 (applied).** `UPDATE_ONCE` on a resize while disabled is mandatory (§ 2.2, § 3.2 step 4, § 6.4); `ViewportWiring` tracks the desired mode; step-11 checks added for pause + resize and pause alone; OQ-G narrowed.
  - **E5 (applied).** `fallback_to_opengl3 = false` and `fallback_to_d3d12 = false` required in § 5.1; a release-build method or driver mismatch is a fatal "unsupported GPU" screen (§ 1, § 5.2).
  - **E6 (applied).** OQ-C rewritten with the key-absent probe result and the template-write hypothesis; corrections suggested for `modules/rendering.md` and `breaking-changes.md` (Migration Plan).
  - **E7 (applied) and T8 (applied).** Distinct layer indices: hold 1, HUD 2, MenuPixel 3, MenuReading 4, Debug 5; the shared-index dependency and the UNVERIFIED tree-order claim are gone; Chart Table sync item updated.
  - **E8 (applied).** Snap flags snap in base-pixel space in both passes; `canvas_transform` in base-pixel units; integer-valued writes still required (§ 2.2, § 4); ADR-0009 told via the sync list.
  - **E9 (applied).** Step 11 narrowed to textured sheet, one-pixel checker, pass B, odd window size; the flat-fill probe recorded as passing (Engine Compatibility, OQ-B).
  - **E10 (applied).** Container composite row added to § 7.1; `D_root` 63 and 93; one-frame lag note; OQ-F closed from the probe; release-export re-run kept in Verification (8).
  - **E11 (applied).** `offered_h` is `Array[int]` duplicated on entry; `world_to_hud` is the two-argument form everywhere; `version` moved out of `RenderLayout` to `PipelineRoot.layout_version`; reader-cast note added (§ 5.2, Gotchas 8).
  - **E12 (applied).** `ViewportWiring.Mode { A, B }` is a constructor argument (§ 4, Key Interfaces); the "only `ViewportWiring` branches on it" lint is kept.
  - **E13 (applied).** Gotchas list added to Implementation Guidelines; zoom cooldown divides by `Engine.time_scale`; step-11 lines for HUD text sharpness at `H = 3` and edge-drag cadence; export preset must include Vulkan.
  - **E14 (applied), T11 (applied) and TD-23 (applied).** Canvas coordinate exactness paragraph in § 6.2 (integer-valued `float32`, exact to 2^24 base px; `Vector2i` converted only at `Placement.put` and the `canvas_transform` write; ADR-0015 owns the bound if extents could exceed it); unit test at the bound; ADR-0005 TR-shipbody-038 and ADR-0010 are asked to cite it.
  - **T2 (BLOCKING, applied via TD-04).** `KernelStateView` and "Cross-ADR Assumption A2" struck. The kernel state comes from `PresentationClock.state: int` and the CUT trigger from `PresentationClock.load_seq` (ADR-0001 amendment batch, TD-04); `setup()` takes `PresentationClock` only; the batch is an Accept gate. The OQ that T2 proposed is not added because TD-04 already answers it.
  - **T3 (BLOCKING, applied) and TD-26 (applied).** `DisplaySettings` is persisted host-locally by the Chart Table `SettingsStore` (CT-19, ADR-0030), not PROFILE and not ADR-0006; ADR-0006 removed from Ordering Note and Related.
  - **T4 (BLOCKING, applied) and TD-22 (applied).** The `post_restore` / ADR-0016 rule is dropped; § 6.1 cites `snap_baseline` on entry to `RUNNING` plus the camera CUT on `DISCONTINUITY_RESTORE`; TR-pxl-015 wording in the sync list.
  - **T5 (BLOCKING, applied) and TD-27 (applied).** Depends On is ADR-0001, ADR-0007, ADR-0009, ADR-0010; ADR-0009 removed from Enables (co-acceptance stated); Ordering Note refreshed; step 12 added to the Accept gate; `RenderTuning` schema defined in Key Interfaces (a `ScalarTable` facade with `SCALED` fields plus a typed `ZOOM_MIN_K` row table).
  - **T6 (applied).** Migration Plan sync list extended: TR-pxl-014, 010, 015, 025, 028, 030 and the owner column re-point; PR-6 C11 / D.14b alongside PR-7 and PR-15.
  - **T7 (applied in part).** Pass A only is built at MVP behind the seam and B and its equivalence test are conditional (§ 4); `DrawBudget.validate` dropped for review or a doc-lint; the debug tree walk is an optional later story; § 7 now states that the GDD's D.16 owns the figures and this ADR owns the rules, so a spike measurement amends the GDD, not this ADR. **Not done:** a new `frame-budget.md` was not created (this pass edits only this file), so the mirrored table stays in § 7.
  - **T9 (applied).** § 5.3 and § 5.5(a): CI asserts the effective `ProjectSettings` values of `rendering_method` and both driver keys; live rows stay NOT ASSESSED headless.
  - **T10 (applied).** OQ-E closed: startup findings are pipeline-local diagnostics values, no fault domain at MVP, the 15 to 31 reservation unaffected (§ 5.2). The `TileSet.tile_size` check moved to ADR-0010's asset and scene lint (§ 5.5(b)) and removed from `StartupCheck`.
  - **TD-19 (applied).** The Ordering Note and the TR rows for TR-pxl-006 to 009, 017 and 018 re-pointed to ADR-0034 (Y-sort, z-bands, PROJECTOR shape) with ADR-0009 as placement only; the `D_ysort` owner column too.
  - **Alignment rulings (no decision taken here).** TD-04 (PresentationClock fields, `HOST_SERVICE`, no PRESENTATION_LATE, no ChangeBatch load flag), TD-18 (allow-list entries for `viewport_wiring.gd` and the camera host handler; parallax offsets use `Placement`'s integer helpers), TD-21 (one two-tier test hosted as § 5.5(a); `fallback_to_*` and `thread_model` in the key list; typed-assignment cast pattern), TD-33 (this ADR carries the Registry Candidates section; OQ-E closed; `present_time` owned here).
  - **Consequence.** The ADR remains Proposed. Its drawing, hook and colour-path claims are now probed on a Windows dev build with Forward+ on Vulkan; release export, D3D12, the Steam Deck, Compatibility parity for lighting, and all performance figures remain UNVERIFIED until the named spike steps run.

- **2026-10-29 /architecture-review follow-up.** Frame budget row ("Total frame, CPU P99") now states that it excludes frames in which the kernel is in `SAVING`; the autosave capture frame is bounded by ADR-0006 `SNAPSHOT_BUDGET_MS` and SAVE-WRK-1 (f). Depends On / Enables checked: unchanged (no ADR lists ADR-0008 under Depends On). No behaviour change.
- **2026-10-29 /architecture-review follow-up (Foundation gaps).** Added: TR-pxl-033 (closed render-option set, § 5), TR-light-039 (GPU ms and VRAM recording rule, § 7.3), TR-charttable-037 (fractional HUD canvas read-back, Validation Criteria). Closes review gaps; no change to existing decisions.
- **2026-10-29 Core-tier amendments** (requested by ADR-0019, ADR-0018 and ADR-0034, and the binding Core-tier technical-director adjudication; status stays Proposed). Applied: (a) `PipelineRoot.follow_pose_q: Vector2i` read-only accessor beside `layout`, `layout_version` and `present_time`, holding the previous frame's `CAMERA_SOLVE` value when read from `INPUT_ADAPTER` priority (ADR-0019); (b) `limits: CameraLimits` (read-only) added to `SolverInput` with a `SolverInput` stub in Key Interfaces, and the `SolverInput` guideline amended to allow that one geometry field (ADR-0034, adjudicated); (c) new § 6.1a: `CameraLimits` is the raw hull rectangle with `body.projector` as its only writer, `rect_eff` is pipeline-derived solver state, and this ADR explicitly owns the clamp order (inner box, then limits, limits win), `CLAMPED`, `rect_eff`, "`world_to_hud` never offset while clamped" and JOLT; the TR-pxl-003 and TR-pxl-017 rows reworded, § 3.2 step 3 and § 6.2 wording aligned (adjudicated; ADR-0034); (d) `HoldLayer` shown during `LOADING` is released on `RUNNING` and `ProjectorReadiness.is_ready()`, and `PipelineRoot` reads `is_ready()` for the `TRANSIT_HOLD` latch (§ 6.4, guideline; adjudicated; ADR-0034); (e) § 2.2 states the snap flags were probed not to snap `Light2D` positions on 4.7.2 (ADR-0018 P-5/P-11) and Verification Required (10) reruns the lighting probe pack on any patch bump or snap-setting change (adjudicated; ADR-0018); (f) Verification Required (3) names the shadow-origin probe pack SO-P1 to SO-P10 as a correctness check for lighting spike steps 1, 2 and 9, with the half-cell lattice (SO-P6) and light-position snap under pass B (SO-P5) in the step list (ADR-0018 F4); (g) Verification Required (9): `get_render_info` canvas draw calls confirmed populated on a release export (RM-DESKTOP, RM-DECK) and draw-call harnesses wait at least 2 frames after any show, hide or reparent (adjudicated); (h) `D_ysort` row ownership by ADR-0034 confirmed, `ZBand` names owned by `src/presentation/projector/z_band.gd`, world viewport content is the single `ProjectorRoot/World` tree (§ 7.1; ADR-0034). No behaviour, budget or ownership decision of this ADR other than the adjudicated clamp ownership changed.
