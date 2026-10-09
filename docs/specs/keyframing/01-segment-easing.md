# Spec 01: segment easing, presets and the ease library

Written without the repository open; `VERIFY:` marks every assumption about unseen code (gathered in section 11). Report 1 is `reports/Smooth keyframing for WebGL motion graphics.md`, report 2 is `reports/AE keyframing hacks and plugins.md`, as in 00.

## 1. Purpose and evidence

One popover on the bar between two keys replaces F9, the Keyframe Velocity dialog and the five plug-in families that sell easing for After Effects (report 2 §2 "Easing: one segment popover", §6 rank 1). It edits the one ease object a segment owns (00 Ruling 2), so presets, the library and paste all write the same four numbers or the same procedural parameters. The maths is settled: report 1 §1 (the x-clamped cubic, the bezier-easing 2.1.0 solver, Easy Ease equals smoothstep), §2 (segment ownership, the asymmetric default, the spring hand-off problem) and §4 (closed-form springs, Rive's elastic, the Penner split between single cubics and procedural shapes).

## 2. Scope and non-goals

In scope: the Ease popover and its controls; the influence-to-handle mapping; Bezier, Hold, Spring, Elastic and Bounce with their closed forms; the preset grid; Apply to all selected; the library with slots 1 to 9 and JSON import and export; Reset to Auto; live preview; the timeline bar glyph; the right-click entries of 00 §4.

Non-goals: the graph editor (spec 04, which draws the same object); Copy and Paste Ease (spec 03); Settle, Follow, Wiggle and the bake service (spec 05); selection (spec 06); multi-segment eases between one key pair (report 1 §4 recommends them; `EaseV1` has no such variant, section 12); per-channel eases (00 Ruling 3); colour gamut mapping after an overshoot.

## 3. Rulings

- **01-R1 (sliders set influence only):** the Out slider at p % sets `x1 = p/100`; the In slider at q % sets `x2 = 1 − q/100`; neither touches `y1` or `y2`. A fresh bezier has `y1 = 0, y2 = 1` (speed 0 at both keys, AE's Easy Ease), so the sliders behave as AE's influence. A `y` outside `[0, 1]` (overshoot, anticipation) is not reachable from the sliders: it needs the handle graph or the numeric field, after which an "Overshoot" badge shows and the sliders keep that `y`. Why: report 2 §2 (influence is the handle's horizontal pull; Easy Ease is speed 0 at 33.33 %); a slider that reset `y` would destroy a designed overshoot on every nudge. Cost if wrong: two lines.
- **01-R2 (asymmetric default, user-settable):** new segments get `DEFAULT_EASE_V1`, Material standard `(0.2, 0, 0, 1)`; "Set as default" writes the current ease into the document setting `defaultEase`; Easy Ease `(1/3, 0, 2/3, 1)` sits in the grid for migrating users. Why: report 1 §2 and report 2 §2 (Easy Ease is "a weak starting point"; AE has no changeable default). Cost if wrong: one constant and one setting.
- **01-R3 (one fraction per segment, extrapolated for Position):** `easeFractionV1` runs once per segment per time; every channel of a vector uses the same `f` as `v0 + (v1 − v0)·f` (00 Ruling 3); Position uses `f` as the arc-length fraction through the 150-sample table (00 Ruling 10), and when `f` leaves `[0, 1]` the point continues along the path's end tangent by `(f − 1)·L`, or the start tangent by `f·L` when `f < 0`, the spatial form of CSS's extrapolation rule (report 1 §1). Why: an overshoot preset must read the same on Position as on Scale. Cost if wrong: one function.
- **01-R4 (procedural eases are shares of the segment and end at exactly 1):** spring, elastic and bounce parameters are fractions of the segment, not seconds; `easeFractionV1` returns exactly 0 at `u ≤ 0` and exactly 1 at `u ≥ 1` for every type (bezier-easing 2.1.0 does the same); spring and elastic subtract their residual linearly, `f(u) = s(u) + u·(1 − s(1))`. Why: a normalised ease is invariant under retime and paste (spec 03 needs no scaling); the linear residual keeps Apple's duration and bounce semantics in one line, where Motion's `findSpring` (envelope 0.001 at the end) gives up the duration control; at the defaults the residual is 1.5 % of the move. Cost if wrong: one formula per type.
- **01-R5 (Penner polynomial presets are beziers):** sine, quad, cubic, quart, quint, expo, circ and back presets are easings.net's cubic-bezier approximations; only elastic, bounce and spring are procedural. Why: report 1 §4 (each of those families is a single cubic); a bezier preset keeps its handles and exports to Lottie without a bake. Cost if wrong: a table swap.
- **01-R6 (editing frees the segment; Reset to Auto reflows it):** every popover edit, preset, slot, paste or handle drag calls `setSegmentEaseV1`, which stores the ease and sets both keys' `tangent` to `free` (00 Ruling 5). The ease of a segment whose keys are `auto` is derived by `autoEaseV1` and written back by every command that edits the track, so it is never stale. Reset to Auto sets both keys to `auto`, which also reflows the neighbouring segments' handles at those keys. Why: 00 Ruling 5. Cost if wrong: one refresh function.

## 4. Model and interfaces

```ts
// src/animation-core/keyframes/ease.ts   VERIFY: folder; EaseV1, KeyframeV1, TrackV1, ExactTime, KeySelectionV1 from 00 §3
export type BezierEaseV1 = Extract<EaseV1, { type: 'bezier' }>
export const easeFractionV1 = (ease: EaseV1, u: number): number
export const bezierFractionV1 = (e: BezierEaseV1, u: number): number
export const springFractionV1 = (duration: number, bounce: number, u: number): number
export const elasticFractionV1 = (amplitude: number, period: number, u: number): number
export const bounceFractionV1 = (bounces: number, restitution: number, u: number): number
export const isProceduralEaseV1 = (ease: EaseV1): boolean
export const influenceToHandlesV1 = (e: BezierEaseV1, outPercent: number, inPercent: number): BezierEaseV1
export const handlesToInfluenceV1 = (e: BezierEaseV1): { out: number; in: number; overshoot: boolean }
export const parseCubicBezierV1 = (text: string): { ok: true; ease: BezierEaseV1 } | { ok: false; reason: string }
export const formatCubicBezierV1 = (e: BezierEaseV1, precision: 'display' | 'exact'): string
export const autoEaseV1 = (prev: [number, number] | null, a: [number, number], b: [number, number], next: [number, number] | null): BezierEaseV1
export const refreshAutoEasesV1 = <V>(track: TrackV1<V>): TrackV1<V>
export interface SegmentRefV1 { readonly trackId: string; readonly time: ExactTime }
export const targetSegmentsV1 = (selection: KeySelectionV1, doc: DocumentV1): readonly SegmentRefV1[]
export const setSegmentEaseV1 = (doc: DocumentV1, segments: readonly SegmentRefV1[], ease: EaseV1): DocumentV1
export const resetSegmentsToAutoV1 = (doc: DocumentV1, segments: readonly SegmentRefV1[]): DocumentV1
export interface EasePresetV1 { readonly id: string; readonly name: string; readonly group: 'css' | 'material' | 'ae' | 'penner'; readonly ease: EaseV1 }
export const EASE_PRESETS_V1: readonly EasePresetV1[]
export interface EaseLibraryV1 { readonly format: 'ease-library'; readonly version: 1; readonly entries: readonly { readonly id: string; readonly name: string; readonly ease: EaseV1 }[] }
```

Ranges (`EASE_RANGES_V1`, same module): influence 0 to 100, step 0.01 (0 is a hard corner; AE's scripting floor is 0.1); spring duration 0.1 to 1 of the segment, default 1 (under 0.1 the spring runs over ten cycles and reads as a vibration: Wiggle, spec 05); bounce −1 to 0.9, default 0.3 (1 is undamped and never settles; 0.3 is Motion's default); elastic amplitude 1 to 3, default 1 (under 1 Penner's form degenerates), period 0.1 to 1, default 0.3 (Penner's and Blender's); bounces 1 to 6, default 3, restitution 0.1 to 0.9, default 0.5 (Penner's; at 6 the last bounce is 0.5^12 of the move).

`targetSegmentsV1`: the selected segments plus the segment starting at each selected key that has a later key, ordered by track order then time, without duplicates (`VERIFY:` the track order API; spec 06 owns `KeySelectionV1`); its key in `selection.segments` is the earlier key's `${trackId}@${serializedTime}`. `formatCubicBezierV1`: display is three decimals trimmed; exact is the JS shortest round-trip string.

`autoEaseV1`: slopes `m0 = autoTangentSlopeV1(prev, a, b)`, `m1 = autoTangentSlopeV1(a, b, next)` in value per second, `Δt = b.t − a.t`, `Δv = b.v − a.v`: `x1 = 1/3`, `y1 = m0·Δt/(3Δv)`, `x2 = 2/3`, `y2 = 1 − m1·Δt/(3Δv)`; when `Δv = 0`, `y1 = 0, y2 = 1` (Hermite handles at one third of the interval, report 1 §1). A `free` key keeps its stored side; a hold on either side leaves the segment untouched; with `track.c2` the slopes come from spec 04's C2 solve.

**Closed forms** (`u ∈ [0, 1]`, fixed arithmetic in `u`; every type returns the literal 0 at `u ≤ 0` and 1 at `u ≥ 1`):
- Bezier: `y(solveBezierXV1(x1, x2, u))`, with the linear shortcut when `x1 = y1` and `x2 = y2`, as bezier-easing.
- Spring (Apple WWDC23; Motion `spring.ts` and plan 031): `ω0 = 2π/duration`, `ζ = 1 − bounce`, zero initial velocity, raw `s(t)`: for `ζ < 1`, `ωd = ω0·√(1−ζ²)` and `s = 1 − e^(−ζω0t)·(cos(ωd·t) + (ζω0/ωd)·sin(ωd·t))`; for `ζ = 1`, `s = 1 − e^(−ω0t)·(1 + ω0t)`; for `ζ > 1`, `r = √(ζ²−1)`, `k = ζ + r`, `λslow = −ω0/k`, `λfast = −ω0·k`, `cslow = k/(2r)`, `cfast = 1 − cslow`, `s = 1 − (cslow·e^(λslow·t) + cfast·e^(λfast·t))`, the two-exponential form that avoids cancellation. Output `f(u) = s(u) + u·(1 − s(1))`.
- Elastic (Penner ease-out, easings.net and Rive `elastic_ease.cpp`): `a ≥ 1`, `p = period`, `sh = (p/2π)·asin(1/a)`, `e(t) = a·2^(−10t)·sin((t − sh)·2π/p) + 1`; output `e(u) + u·(1 − e(1))`.
- Bounce (Penner ease-out generalised; Penner's 7.5625 and 2.75 are `N = 3, r = 0.5`): fall width `t0 = 1/(1 + 2·Σ_{k=1..N} r^k)`; for `u < t0`, `(u/t0)²`; bounce `k` starts at `t0·(1 + 2·Σ_{j<k} r^j)`, has half-width `h = r^k·t0` and centre `c = start + h`; inside it, `1 − r^(2k)·(1 − ((u − c)/h)²)`, the last bounce taking `u ≤ 1`.

`s(1)` and `e(1)` are recomputed from the parameters on every call; a memo keyed by the parameters changes no bit.

**Presets** (`EASE_PRESETS_V1`): CSS `linear (0,0,1,1)`, `ease (0.25,0.1,0.25,1)`, `ease-in (0.42,0,1,1)`, `ease-out (0,0,0.58,1)`, `ease-in-out (0.42,0,0.58,1)` (CSS Easing 2); Material `standard (0.2,0,0,1)`, `standard accelerate (0.3,0,1,1)`, `standard decelerate (0,0,0,1)`, `emphasised accelerate (0.3,0,0.8,0.15)`, `emphasised decelerate (0.05,0.7,0.1,1)` (material-web tokens v0.192; the full emphasised curve is a multi-segment path the token file aliases to standard, a named limit); AE `Easy Ease (1/3,0,2/3,1)`, `Easy Ease Out (1/3,0,1,1)`, `Easy Ease In (0,0,2/3,1)` (exact thirds, shown as 33.33 %); Penner in, out and in-out for sine, quad, cubic, quart, quint, expo, circ and back from easings.net's `easings.yml` (`VERIFY:` the thirteen values not quoted in the research note; for example out back is `(0.34,1.56,0.64,1)`); elastic out `{amplitude 1, period 0.3}` and bounce out `{bounces 3, restitution 0.5}` (in and in-out need a direction field, section 12); spring `{duration 1, bounce 0.3}`.

**Library**: user-level (`VERIFY:` the settings store), ordered; slots 1 to 9 are the first nine entries; the file is `EaseLibraryV1` JSON named `*.eases.json`; import appends entries whose `id` is new, skips the rest and any invalid entry, and reports both counts.

## 5. UX

**Opening.** Double-click a segment bar, right-click it and choose a preset or "Edit ease", or press Ctrl+Shift+E with segments or keys selected (00 §4). The popover anchors to the bar; with several targets it edits all of them, its header reads "3 eases" and differing fields show "mixed"; double-clicking one bar while others are selected edits that one and offers "Apply to all selected (4)". It follows 00 §4's popover pattern (live preview, one undo step per release, Escape closes without reverting, no selection change). The header names property and range, "Scale, 12f to 30f", with badges "Auto", "Hold" or "Overshoot".

**Type selector.** Bezier, Hold, Spring, Elastic, Bounce. Booleans, enums and visibility show Hold only, with "This property can only hold" (00 Ruling 6).

**Bezier.** A handle graph (unit square, 160 px wide, `VERIFY:` the popover width) with P1 and P2 draggable anywhere in `x ∈ [0, 1]`; the vertical range grows to fit `y`, with dashed lines at 0 and 1; no modifiers until section 12's proposals are accepted. Below it the In and Out influence sliders, 0 to 100 % with two decimals, and a centre handle that drags both mirrored (Mt. Mograph's Ease Sliders and Easy Keyframes, report 2 §2); scrubbing follows 00 §4's number-field steps. The numeric field shows `cubic-bezier(0.2, 0, 0, 1)` at display precision and accepts, on Enter, `cubic-bezier(a, b, c, d)`, four numbers separated by commas or spaces, or a CSS keyword; an `x` outside `[0, 1]` or a non-finite number keeps the field red with "x values must be between 0 and 1" and changes nothing.

**Procedural controls.** Spring: Duration 10 to 100 % of the segment, Bounce −100 to 90 %, hover text "Settles by 0.42 s on this segment". Elastic: Amplitude 1 to 3, Period 10 to 100 %. Bounce: Bounces 1 to 6, Restitution 10 to 90 %. The graph shows the curve read-only, sampled at 64 points (a display constant).

**Preset grid.** Groups CSS, Material, AE and Penner; the Penner group is a table of families by In, Out and In-out, where elastic and bounce have the Out cell only (Blender's "Automatic" easing also picks ease-out for them, report 1 §4). Hovering a cell previews on the graph and the canvas without committing (the Flow request, report 2 §2); clicking commits one step. A search field filters by name. Buttons: "Set as default" (01-R2), "Reset to Auto" (enabled when either key is `free`), "Save to library" (asks for a name), "Apply to all selected", Copy and Paste (spec 03).

**Library strip.** The first nine entries as chips numbered 1 to 9; the keys 1 to 9 apply them to the target segments whenever the timeline or the popover has focus and no text field is active (00 §4); the strip's menu has Import, Export and Manage (rename, reorder, delete).

**Timeline bar glyph.** A 12 px glyph at the bar's centre (`VERIFY:` the bar height): a miniature of the curve for bezier, a step for hold, a damped wave for spring, a wave with a tall first peak for elastic, three shrinking arcs for bounce; hover text names it, "Spring, bounce 30 %, settles by 100 %". The graph editor (spec 04) draws a procedural ease as a sampled curve without handles; double-click opens this popover.

**Live preview and errors.** Every change renders the canvas at the playhead; outside the segment the ghost of 00 §4 is drawn. The last key of a track has no segment: the double-click is ignored and hover text reads "No ease after the last key". With nothing selected, Ctrl+Shift+E shows "Select a segment or two keys first".

## 6. User flows

1. **Primary: ease a scale pop.** Start: Scale keys at 0f (100 %) and 12f (140 %), both `auto`. Double-click the bar: the popover opens on Bezier with the derived auto handles and the badge "Auto". Drag the Out slider to 60 %: P1 moves to `x = 0.6`, the canvas re-renders at the playhead, both keys turn `free`, one undo step on release. Hover "out back": the graph and canvas preview the overshoot; click it: badge "Overshoot". End: the segment is out back, keys `free`, two undo steps.
2. **Keyboard only.** Start: an Opacity segment selected, timeline focused, "Snappy" in library slot 3. Press 3: Snappy applies and the bar glyph changes. Press Ctrl+Shift+E: the popover opens with focus on the Out slider showing Snappy's values. Press Shift+Up twice: Out rises by 20 and the preview updates. Tab to the numeric field, type `cubic-bezier(0.2, 0, 0, 1)`, Enter: the handles jump to Material standard. Escape closes. End: Material standard on the segment, three undo steps.
3. **Multi-selection: one ease across a stagger.** Start: twelve layers' Position segments selected (spec 02 made the stagger). Press Ctrl+Shift+E: header "12 eases", fields "mixed". Click "ease-out": every segment becomes `(0, 0, 0.58, 1)` in one undo step and every layer follows its own path with the same arc-length ease. Choose Spring, Bounce 40 %: twelve springs, twelve glyphs. End: twelve identical normalised springs; durations unchanged.
4. **Undo.** Start: flow 1's end, popover open. Ctrl+Z: the ease returns to the 60 % slider state; the popover stays open and shows it; selection unchanged. Ctrl+Z: back to the derived auto ease, badge "Auto". Redo (`VERIFY:` the keymap): forward one. End: the slider state; the view never moved.
5. **Limit: overshoot between equal values.** Start: Rotation keys at 0f (0°) and 10f (0°). Open the popover, choose Elastic: the canvas does not move and the header reads "Both keys have the same value, so this ease moves nothing. Add a middle key or a Settle behaviour". End: the ease is stored, the layer rests (section 8).
6. **Library round trip.** Start: popover open on a tuned bezier. "Save to library", name "Lift": chip 4 appears. Library menu, Export: `Lift.eases.json` is written (`VERIFY:` the file picker). On another machine, Import: "1 added, 0 skipped". End: slot 4 is "Lift" there too.

## 7. Evaluation and determinism

For a time `t` inside the segment from key `a` to key `b`: `u = (t − a.time)/(b.time − a.time)` is formed from exact rationals and converted to a double once (00 Ruling 9; report 1 §3); `f = easeFractionV1(a.ease, u)`; each scalar channel is `a.v + (b.v − a.v)·f`; Position maps `f` through the arc-length table with 01-R3's extrapolation; hold returns `a.v` for `u < 1`. The solve is `solveBezierXV1` (00 Ruling 4) on every path: play, seek, export, popover preview and graph. Nothing here carries state between frames; a procedural ease is a closed form in `u` with fixed arithmetic, so random-order evaluation is exact. This spec bakes nothing itself; on Lottie export the bake service of spec 05 replaces each procedural ease with keys and bezier eases fitted to `easeFractionV1` samples and keeps the parameters for unbake. Ruling 9 statement: every value this spec shows is `evaluate(document, time)`.

## 8. Edge cases and named limits

- Equal key values: any ease, including elastic and spring, moves nothing, because the fraction scales `Δv = 0`; unlike Rive's Cubic Value, handles are not values. Named limit: a bump between equal values needs a middle key or a Settle driver (spec 05).
- A preset on a hold segment converts it to bezier (R9); the keys stay `free`. A hold on one side of a key: no auto slope is computed there; the free side keeps its handle.
- Spring at bounce −100 %: ζ = 2, a slow two-exponential approach whose residual at `u = 1` is about 0.2 of the move at duration 100 %; hover text says "Shorten the duration for a sharper settle". Overshoot on colour may leave gamut; mapping is the colour spec's job. Keys 1 to 9 type digits while a text field has focus.
- Lottie export: hold and overshoot `y` are representable (Lottie's `y` is unbounded); procedural eases are baked (spec 05).

## 9. Interactions with other specs

- 03: Copy and Paste buttons; `targetSegmentsV1`, `parseCubicBezierV1`, `formatCubicBezierV1` and `setSegmentEaseV1` are shared.
- 04: the graph draws the same `EaseV1`; a handle drag calls `setSegmentEaseV1`; procedural eases are sampled curves with no handles; the C2 opt-in feeds `autoEaseV1`.
- 05: `isProceduralEaseV1` and `easeFractionV1` are what the bake service samples on export; Settle answers the equal-values limit.
- 06: selection semantics and the number-field steps; the popover never changes selection.
- 07 and 08: retime, stagger and loops never touch the ease; normalised handles make "maintain proportional easing" automatic.
- 10: plain paste carries eases with keys; Paste Reversed mirrors each bezier in time with spec 03's `mirrorEaseInTimeV1`.

## 10. Rules and tests

Each rule is one test in `ease.test.ts` (D1 to D4 in `ease.determinism.test.ts`); the test name follows the rule number.
- R1, T1 `Easy Ease is smoothstep at ten points`: `bezierFractionV1((1/3, 0, 2/3, 1), u) = 3u² − 2u³` within 1e-9 at `u = 0.1, 0.2, …, 1.0`.
- R2, T2 `bezierFractionV1 matches bezier-easing 2.1.0 on a 21 by 21 grid`: `x1, x2 ∈ {0, 0.05, …, 1}`, `u ∈ {0, 0.01, …, 1}`, `y1 = 0, y2 = 1`; equal to the vendored 2.1.0 reference within 1e-6.
- R3, T3 `every ease type returns exactly 0 and 1 at its ends`: 1,000 random parameter sets per type; `easeFractionV1(e, 0)` is `0` and `easeFractionV1(e, 1)` is `1` by `Object.is`; the spring at `u = 1 − 1e-9` is within 1e-6 of 1.
- R4, T4 `bounce with 3 bounces and restitution 0.5 is Penner easeOutBounce`: within 1e-12 at ten points including `u = 1/2.75` and `2.5/2.75`.
- R5, T5 `influence sliders set x only`: out 33.33 gives `x1 = 0.3333`, in 33.33 gives `x2 = 0.6667`; `y1 = 0.5` and `y2 = 1.5` are untouched; `handlesToInfluenceV1` inverts it.
- R6, T6 `cubic-bezier text parses, refuses and round-trips`: the three syntaxes and five keywords parse; `x = 1.2`, `NaN` and three numbers refuse with the stated reasons; `parse(format(e, 'exact'))` is `e` by `Object.is` per field on the 21 × 21 grid.
- R7, T7 `a preset on Scale changes both channels`: Scale `(100, 100) → (200, 50)` with ease-out evaluates at `u = 0.5` to `100 + 100·f` and `100 − 50·f`, `f = bezierFractionV1(ease-out, 0.5)`, by `Object.is`.
- R8, T8 `a preset on Position runs through arc length and extrapolates`: the point sits at arc-length fraction `f`; `f = 1.2` lands `0.2·L` beyond the end along the end tangent.
- R9, T9 `a preset converts a hold segment`: type `bezier`, both keys `free`; the right-click "Hold" entry converts back.
- R10, T10 `edit frees, Reset to Auto reflows`: after an edit both keys are `free`; after Reset to Auto both are `auto`, the ease equals `autoEaseV1`, and the neighbouring segment's handle at the shared key reflows.
- R11, T11 `library slots, import and export`: key 3 applies entry 3 to every target segment in one undo step; import appends and reports counts; export then import reproduces the entries by deep equality.
- R12, T12 `elastic starts at exactly 0`: amplitude 1 and 2.5 (the `sh` phase cancels the sine).
- D1, T13 `play equals seek`: a fixture with an overshoot bezier, a spring, an elastic, a bounce and a hold on scalar and Position tracks; frames 0..120 in order, then frame 120 from a fresh evaluator; `Object.is` per channel.
- D2, T14 `random order`: a shuffled frame list equals D1.
- D3, T15 `after an edit`: apply a preset, undo, evaluate; equal to before by `Object.is`.
- D4, T16 `bake round-trip`: spec 05's `bakeTrackV1` on the spring and bounce tracks evaluates within spec 05's tolerance at every frame (`VERIFY:` the figure); unbake restores `{duration, bounce}` and `{bounces, restitution}` by `Object.is`.

## 11. VERIFY list

1. `src/animation-core/keyframes/` and the names in 00 §3 (`EaseV1`, `KeyframeV1`, `TrackV1`, `ExactTime`, `solveBezierXV1`, `autoTangentSlopeV1`).
2. How eases are stored today; 00 §7's adapter must produce `EaseV1` before this popover reads it.
3. The popover component and its anchoring; the number field component (steps, unit suffixes) of 00 §4.
4. The settings store for a user-level library; the file picker for import and export (browser `showSaveFilePicker` versus Electron).
5. The keymap: Ctrl+Shift+E, redo, and the digits 1 to 9 when the timeline has focus.
6. The worker's draft-quality render path for per-edit preview.
7. The undo stack's granularity (one step per release).
8. The timeline bar's height for the glyph; whether bars are DOM or canvas.
9. The track order API for `targetSegmentsV1`.
10. Spec 05's bake tolerance for D4 and the Lottie exporter's hook.
11. The thirteen easings.net values not quoted in the research note.

## 12. Open questions for the user

1. The asymmetric default (00 open question 2): this spec keeps Material standard and adds "Set as default". Confirm.
2. Spring, elastic and bounce parameters are shares of the segment (01-R4), so a spring keeps its shape when pasted or retimed; the alternative is seconds, which keeps the absolute feel and lets a short segment cut a spring off. Confirm shares.
3. Proposed addition to 00 §3: an optional `direction: 'in' | 'out' | 'in-out'` on the elastic and bounce variants of `EaseV1`, default `'out'`, so their In and In-out presets can exist and spec 03's mirror can apply to them.
4. Multi-segment eases between one key pair (report 1 §4) are out of scope. Should a `path` variant of `EaseV1` be planned?
5. Proposed addition to 00 §4: Shift + drag a popover handle constrains it horizontally; Ctrl [Cmd] + drag a handle moves both handles mirrored; Up / Down on a focused influence slider nudge by 1 % (Shift 10, Ctrl 0.1); Alt [Option] + drag a slider moves both sliders mirrored.
6. Should the library also live in the document, so a team file carries its eases, or stay user-level with file sharing only?
