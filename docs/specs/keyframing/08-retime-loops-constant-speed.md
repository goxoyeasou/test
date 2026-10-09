# Spec 08: retime, loops and constant speed

Written without the repository open; every assumption about unseen code is marked `VERIFY:` and gathered in section 11. Depends on 07 (drag model, snapping) and 01 (eases); uses 03 (ease flip), 05 (bake service) and 06 (selection, runs).

## 1. Purpose and evidence

After Effects retimes with Alt-drag, which leaves sub-frame keys and cannot be typed, loops with `loopOut`, which Lottie cannot carry, and smooths speed along a path only through an undocumented roving-key solver (report 2 §3). Its Time-Reverse ignores a sparse selection (report 2 §5); report 2 §6 ranks whole-frame snapping seventh and retime, loops and Constant Speed eighth. Report 1 §2 supplies ripple (Motion Canvas), retime markers (Maya) and Absolute Time Snap (Blender). Report 1 §1 gives the extrapolation formulas, Blender's cyclic tangent solve, the arc-length path model and the roving-keys idea.

## 2. Scope and non-goals

In scope: proportional retime by the selection's edge handles with ripple (mechanics in 07-R5); the Retime dialog; retime markers on a run; six extrapolation modes with exact formulas, the seam rule, ghost keys and bake; Constant Speed on Position tracks; Time-Reverse of a sparse selection in place.

Not in scope: Time Remap as a keyed time curve; speed ramps on footage; stretching a layer bar as such (`VERIFY:` in and out points); key reduction (spec 05); looping only the last N keys (section 12).

## 3. Rulings

- **08-R1 (scale first, then snap each key):** `t' = snapToFrameV1(a + s × (t − a))`, `a` the anchor, `s` the scale. Why: Ruling 8; snapping the scale instead would allow only spans that divide evenly. Cost if wrong: one line.
- **08-R2 (scaling down stops at the minimum span; a retime never merges selected keys):** as 07-R5, the span stops at the smallest value at or above the requested one where every selected key keeps its own frame; the handle resists. Why: a merge mid-drag deletes a pose when the user cannot see it; Ruling 7 still governs a selected key landing on an unselected one. Cost if wrong: one function.
- **08-R3 (ripple moves later keys by the last key's displacement):** on each track with a selected key, unselected later keys shift by the last selected key's whole-frame displacement (07-R5); nothing ripples when the last key is the anchor; Shift isolates. Why: report 1 §2 (Motion Canvas). Cost if wrong: one filter.
- **08-R4 (linear is the secant, continue is the tangent):** `linear` extrapolates along the line through the last two keys; `continue` along the ease's end tangent; both are flat after a hold. Why: they differ exactly when the end ease is eased out: continue holds, linear keeps moving. Cost if wrong: two slopes swapped.
- **08-R5 (cyclic auto tangents):** in cycle, offset and ping-pong an end key with tangent mode `auto` takes its missing neighbour from the wrapped copy, so the seam is C1 by construction; a seam marker is drawn when the loop is not continuous. Why: report 1 §1 (Blender solves cyclically). Cost if wrong: one neighbour lookup.
- **08-R6 (Constant Speed is one ease across the run):** intermediate key times are derived from arc length and only the first segment's start handle and the last segment's end handle apply, as one bezier across the run; other eases are kept but unused. Why: report 1 §1 (roving keys: one speed curve across several segments). Cost if wrong: the toggle hides eases it should show.
- **08-R7 (reverse keeps times):** Time-Reverse is spec 06's `reverseSelectionV1` under 06-R5: times kept, values and the eases between consecutive selected keys reversed, mirrored by `mirrorEaseInTimeV1` (spec 03). Why: report 2 §5 ("ignoring my carefully selected subset"). Cost if wrong: nothing.

## 4. Model and interfaces

```ts
// src/animation-core/keyframes/retime.ts                  VERIFY: folder, DocumentV1 (00 §3); Vec2 is { x: number; y: number }
export interface RetimeRequestV1 {
  readonly keys: readonly string[]; readonly anchor: 'first' | 'last' | 'playhead'; readonly playhead: ExactTime
  readonly newSpan: number; readonly ripple: boolean; readonly subFrame: boolean     // frames; Shift clears ripple; Ctrl sets subFrame
}
export interface RetimeResultV1 extends EditResultV1 {   // spec 06: doc, moves, removed, report
  readonly span: number; readonly scale: number; readonly rippled: number; readonly stopped: boolean
}
export const retimeV1 = (doc: DocumentV1, req: RetimeRequestV1, fps: number): RetimeResultV1
export const retimeMarkerDragV1 = (run: readonly number[], bead: number, delta: number): readonly number[]

// src/animation-core/keyframes/extrapolation.ts
export interface RemappedTimeV1 { readonly time: ExactTime; readonly cycles: bigint }
export const remapTimeV1 = (track: TrackV1<unknown>, time: ExactTime): RemappedTimeV1          // identity inside [t0, t1]
export const extrapolatedValueV1 = <V>(track: TrackV1<V>, time: ExactTime, ctx: EvalContextV1): V
export const bakeExtrapolationV1 = <V>(track: TrackV1<V>, range: readonly [ExactTime, ExactTime], fps: number): TrackV1<V>   // calls bakeTrackV1 in exact mode (05-R8)

// Position tracks (Ruling 10)                               VERIFY: the Position track type and the 150-sample table
export interface PositionTrackV1 extends TrackV1<Vec2> { readonly constantSpeed: boolean }
export const constantSpeedFramesV1 = (first: number, last: number, segmentLengths: readonly number[], runEase: EaseV1): readonly number[]
```

**Retime.** `F`, `L` are the selection's first and last frames, `a` the anchor, `S = L − F`, `s = S' / S` for the requested span `S'`; `t'_k = snap(a + s × (t_k − a))`, unsnapped with `subFrame`. If two selected keys of one track share a frame and `S' < S`, `S'` rises to spec 07's `minSpanFramesV1`, the smallest integer in `[S', S]` with all frames distinct, and `stopped` is set. Ripple: unselected keys later than `L` on those tracks move by `t'_L − L`. Isolated: they stay, and a selected key landing on one replaces it (Ruling 7), counted. No ease changes: every ease, springs included, is normalised to its segment (Ruling 1, 01-R4).

**Markers.** A run `r_0..r_{m−1}`, `m ≥ 3`, has a bead at each segment's midpoint. Dragging bead `k` by `Δ` sets `r'_{k+1} = r_{k+1} + Δ` and scales the keys after it about the last key: `r'_j = snap(r_{m−1} − (r_{m−1} − r_j) × (r_{m−1} − r'_{k+1}) / (r_{m−1} − r_{k+1}))`; keys up to `k` and the last stay. Limits: `r'_{k+1} ≥ r_k + 1` and 08-R2 on the trailing keys.

**Extrapolation.** `t0`, `t1`, `T = t1 − t0 > 0`, `v0`, `v1`, `keyed(t)` on `[t0, t1]`; `n = floor((t − t0) / T)` as an exact integer, `p = t − t0 − n × T`:
- constant: `v0` before `t0`, `v1` after `t1`.
- linear and continue: `v0 + m × (t − t0)` or `v1 + m × (t − t1)`, `m` per second: the end segment's secant (linear) or the ease's end tangent (continue): `y1 / x1 × Δv / Δt` at a start, `(1 − y2) / (1 − x2) × Δv / Δt` at an end, the secant when `x` is 0 or 1, 0 for non-bezier eases.
- cycle: `keyed(t0 + p)`.
- ping-pong: `q = t − t0 − 2T × floor((t − t0) / 2T)`; `keyed(t0 + q)` when `q ≤ T`, else `keyed(t0 + 2T − q)`.
- offset: `keyed(t0 + p) + n × (v1 − v0)` per channel; `n < 0` before `t0`.

Each end has its own mode; one key means constant; hold-only properties offer constant, cycle and ping-pong. Under 08-R5 the first key's missing neighbour is `(t_{n−2} − T, v_{n−2} − d)` and the last key's `(t_1 + T, v_1 + d)`, `d = v1 − v0` for offset and 0 for cycle; ping-pong reflects, `(t0 − (t_1 − t0), v_1)` and `(t1 + (t1 − t_{n−2}), v_{n−2})`, which Ruling 5's formula turns into a flat tangent. `seamV1(track)` reports `{ before, after }`, true at an end when the value steps (cycle, `v0 ≠ v1`), a free end slope differs from its wrapped neighbour's, or a ping-pong end slope is not 0.

**Bake.** `bakeTrackV1` (spec 05, exact mode) copies keys per cycle inside the range: cycle `(t_i + kT, v_i, ease_i)`; offset adds `k × (v1 − v0)`; ping-pong writes odd cycles reversed with flipped eases (spec 03); linear and continue add one key at the range end on the line, linear ease. Auto end tangents are written as `free` with their cyclic handles; the record keeps `{ before, after }` for Unbake. `ghostKeysV1(track, range, cap)` lists the ghost times.

**Constant Speed.** Segment lengths `L_j` come from the 150-sample tables (Ruling 10); `F_i = Σ_{j<i} L_j / Σ L_j`. The run ease `E` is the bezier `(x1, y1)` of segment 0 with `(x2, y2)` of the last segment when both are bezier with `y1, y2 ∈ [0, 1]`; otherwise linear, with a notice. `s_i = solveBezierXV1(y1, y2, F_i)` (Ruling 4's solver on the y cubic, valid because `y` is monotone), `u_i = X(s_i)`, `t_i = snap(t0 + u_i × T)`; equal derived frames are pushed to the next frame in order. The position is `pathAtFraction(E((t − t0) / T))` over the concatenated tables.

**Reverse.** Per track, the selected keys sorted `k_0..k_{m−1}`: `k_j` takes the value of `k_{m−1−j}`; a segment `(k_j, k_{j+1})` whose keys are adjacent in the track takes the mirror of segment `(k_{m−2−j}, k_{m−1−j})`'s ease when that one is adjacent too, else keeps its ease (06-R5's half-selected rule). Position keys swap path points (`VERIFY:` path storage).

## 5. UX

**Edge handles.** A bracket spans the selection's first and last frames across the involved rows (`VERIFY:` spec 07 draws it). Dragging a handle retimes about the far edge (00 §4); Shift isolates; Ctrl frees sub-frame placement. Readout (spec 07's format plus a per-key line): `Span 30 f → 45 f (150 %)`, `0, 15, 30, 45`, `1 later key moves`; at the stop `Minimum span 3 f`, the handle resisting.

**Retime dialog.** New span (frames) and Scale (%), linked; Anchor: First key (default) | Playhead | Last key; Ripple (default on); Apply. It previews live, commits on Apply or Enter as one undo step and discards on Escape; a span below the minimum clamps and reads `Minimum span 3 f`. Opened by Ctrl+Shift+R [Cmd+Shift+R] (proposed, section 12) or `Animate > Retime…` (`VERIFY:` menu names).

**Retime markers.** On a run of three or more keys the context menu offers `Show retime markers`, a view toggle outside undo (Ruling 11). Dragging a bead previews live, lists the run's frames and commits one undo step per release.

**Loops.** Each track's inspector shows Before and After: Constant (default) | Linear | Continue | Cycle | Ping-pong | Offset; hold-only properties list Constant, Cycle, Ping-pong. Ghost keys are drawn dimmed in the extrapolated part of the visible range, not selectable, hover `Looped: cycle. Bake to edit.`, at most 500 per track per view (five times the 100 keys at which report 2 §1 records the timeline slowing). A seam marker is a small triangle at the end key. The context menu offers `Bake loop to keys` over the composition (`VERIFY:` work-area option) and `Unbake`.

**Constant speed.** A checkbox on a Position track. On: intermediate keys are drawn as circles at derived times and cannot be dragged in time (hover `Constant speed sets this key's time. Turn it off to drag.`); the first and last segments' Ease popovers edit the run's start and end handles and say `Edits the ease of the whole run`; an intermediate segment's popover reads `Constant speed is on: this ease is not used` with a Turn off button. A non-bezier or overshooting end ease shows `Constant speed uses a linear ease here`.

**Time-Reverse.** `Keys > Reverse Selected Keys` (spec 06), Ctrl+Alt+R [Cmd+Option+R] proposed; non-bezier eases are kept and the status says so.

**Empty and error states.** One selected key: no bracket. Fewer than three keys: `Retime markers need three keys`. One key on a track: Before and After disabled, `Add a second key to loop`. Sub-frame keys that would snap together: the handle resists with `Snap keys to frames first (Ctrl+Shift+F)`.

## 6. User flows

**Flow 1, primary (edge handle with ripple).** Start: a Scale track with keys at 0, 10, 20, 30 f selected and an unselected key at 40 f. (1) Press the right handle and drag to 45 f: keys preview at 0, 15, 30, 45 f, the key at 40 f at 55 f; readout `Span 30 f → 45 f (150 %)`, `0, 15, 30, 45`, `1 later key moves`. (2) Release. End: keys at 0, 15, 30, 45, 55 f; one undo step; selection and bracket kept.

**Flow 2, keyboard only (Retime dialog).** Start: as flow 1 before step 1. (1) Ctrl+Shift+R: the dialog opens at New span 30, Scale 100 %, focus in New span. (2) Type `45`, Tab: Scale reads 150 %; preview 0, 15, 30, 45 and 55 f. (3) Type `50%` into Scale, Tab: New span reads 15; preview 0, 5, 10, 15 f and the key at 40 f at 25 f. (4) Enter. End: keys at 0, 5, 10, 15, 25 f; one undo step.

**Flow 3, multi-selection, isolated.** Start: Position keys at 0 and 30 f, Scale keys at 10 and 20 f, Opacity keys at 0, 15, 30 f, all selected; an unselected Opacity key at 40 f. (1) Shift-drag the right handle to 60 f: Position 0, 60; Scale 20, 40; Opacity 0, 30, 60; the key at 40 f stays; readout `Span 30 f → 60 f (200 %)` with no ripple line. (2) Release. End: as previewed; one undo step.

**Flow 4, undo.** Start: the end of flow 1. (1) Ctrl+Z: keys at 0, 10, 20, 30 and 40 f; selection and bracket stay (Ruling 11). (2) Ctrl+Shift+Z: 0, 15, 30, 45, 55 f again. End: as flow 1.

**Flow 5, limit (minimum span).** Start: keys at 0, 1, 2, 3 f selected. (1) Drag the right handle left to 1 f: nothing moves, the handle resists at 3 f, readout `Minimum span 3 f`. (2) Release: no edit, no undo step. (3) With keys at 0, 1, 10 f, dragging to 2 f stops at 0, 1, 5 f (`Minimum span 5 f`); release. End: keys at 0, 1, 5 f; one undo step.

**Flow 6, loop and bake.** Start: a Rotation track keyed 0° at 0 f and 360° at 24 f, linear ease, auto tangents; composition 120 f. (1) After = Cycle: ghosts at 48, 72, 96, 120 f; frame 30 shows 90°; a seam marker at 24 f, hover `The loop jumps at 24 f: first and last values differ. Use Offset to keep turning.` (2) After = Offset: the marker goes; frame 30 shows 450°, frame 54 shows 810°. (3) `Bake loop to keys`: keys every 24 f from 0 to 120 f, values 0 to 1800°; After reads Constant, badge `Baked`; `Unbake` offered. End: three undo steps.

**Flow 7, Constant Speed.** Start: a Position path through (0, 0), (100, 0), (400, 0) keyed at 0, 20, 40 f, linear eases. (1) Tick Constant speed: the middle key becomes a circle at 10 f. (2) Double-click the first segment bar and pick ease-in-out (0.42, 0, 0.58, 1): the middle key moves to 14 f. (3) Drag the middle key: refused, with the hover text. (4) Untick: the key stays at 14 f; the stored eases return. End: three undo steps.

**Flow 8, retime markers, then reverse.** Start: an Opacity run at 0, 10, 20, 40 f with values 0, 25, 50, 100, all selected. (1) `Show retime markers`: beads at 5, 15, 30 f. (2) Drag the bead at 5 f right by 10 f: keys 0, 20, 27, 40 f (the key at 20 scales about 40: 40 − 20 × 20 / 30 = 26.7, snapped 27). (3) Release: one undo step. (4) Ctrl+Alt+R: values 100, 50, 25, 0 at the same frames, eases swapped and flipped. (5) Ctrl+Alt+R: the original values and eases. End: three undo steps.

## 7. Evaluation and determinism

`evaluateTrackV1` becomes: `r = remapTimeV1(track, t)`; `v = keyed(r.time)` inside `[t0, t1]`, else the formulas of section 4; `v += r.cycles × (v1 − v0)` for offset; then the driver stack. Constant Speed evaluates `pathAtFraction(E((t − t0) / T))`, ignoring intermediate key times. Ruling 9 statement: every value is `evaluate(document, t)`; the remap is closed-form in `t`, the floor is exact integer arithmetic, no state crosses frames, so play, seek and export agree bit for bit. Loops bake only for Lottie export (report 1 §1) and on request, through `bakeTrackV1`, whose `BakeRecordV1` keeps the source modes for Unbake. Cycle bakes are bit-exact provided the segment fraction `u` is formed from rational times before the one conversion to double (`VERIFY:`); offset and ping-pong bakes reassociate floating-point sums, so they are held to 1e-9 relative (far above double rounding, far below a pixel).

## 8. Edge cases and named limits

- Offset on a colour adds per channel and gamut-maps; large cycle counts clip. Named limit.
- Under Constant Speed the layer passes an intermediate key's point within its frame, not exactly on it: the time is snapped, the motion is not.

## 9. Interactions with other specs

01: eases survive a retime exactly (normalised handles); the run ease is an ordinary `EaseV1`. 02: offset, then extrapolate. 03: `mirrorEaseInTimeV1` for ping-pong bakes and reverse. 04: the graph draws extrapolation at 35 % opacity. 05: the bake service and its record. 06: `reverseSelectionV1` and `EditResultV1`. 07: the bracket, snapping, Ctrl for sub-frame, Shift for isolate (00 §4). 10: Paste Reversed reflects times within the payload (10-R3), unlike reverse in place.

## 10. Rules and tests

Each rule is one test (`retime.test.ts`, `extrapolation.test.ts`); numbers are exact unless a tolerance is stated.

- R1 `scaling 0..30 f by 1.5 about the first key`: keys 0, 10, 20, 30 → 0, 15, 30, 45.
- R2 `ripple moves a key at 40 f to 55 f`; `Shift keeps it at 40 f`.
- R3 `scale first, then snap`: span 38 → 0, 13, 25, 38 (ideal 12.67, 25.33).
- R4 `scaling 0..3 f down by 0.5 stops at the minimum span`: keys 0, 1, 2, 3 stay, `stopped` true, span 3; keys 0, 1, 10 requested span 2 → 0, 1, 5.
- R5 `anchor at the playhead`: keys 0, 10, 20, 30, playhead 10, span 60 → −10, 10, 30, 50; a later key at 40 → 60.
- R6 `a bead moves only the keys on one side`: run 0, 10, 20, 40, bead 0, +10 → 0, 20, 27, 40.
- R7 `cycle of a 0..24 f track at frame 30 equals frame 6`, `Object.is`.
- R8 `offset adds exactly one delta per cycle`: frames 30, 54 and −18 equal `keyed(6)` plus 360, 720 and −360, `Object.is` against the same expression.
- R9 `ping-pong at frame 30 equals frame 18`, `Object.is`.
- R10 `linear and continue differ after an ease-out`: 0 → 100 over 24 f with `DEFAULT_EASE_V1`: continue holds 100 at frame 30, linear gives 125; with the linear ease both give 125.
- R11 `cyclic auto tangents are C1`: keys 0, 8, 16, 24 f with values 0, −4, 4, 0 in cycle: both end slopes are −0.5 per frame, `Object.is`.
- R12 `seam marker`: cycle 0 → 360 shows it; offset of the same does not; ping-pong with a non-flat free end shows it, with auto ends it does not.
- R13 `ghost keys`: 0..24 f cycle in a 120 f view → 4 ghosts; a 10,000-frame view returns 500.
- R14 `constant speed gives the middle key its arc-length share`: flow 7's path → 10 f; with (0.42, 0, 0.58, 1) → 14 f (`u` 0.346 by the same solver); the position at 10 f is (100, 0) within 1e-9.
- R15 `reversal is an involution`: four keys, values and times `Object.is` after two reversals, ease handles within 2⁻⁵² (two subtractions from 1); a sparse selection keeps the unselected key and the non-adjacent segment's ease.
- D1 `play equals seek`, D2 `random order`, D3 `after an edit` on a fixture with an offset loop, a Constant Speed path and a Time Offset of 3 f; D4 `bake round-trip`: cycle `Object.is` at every frame of 0..120, offset and ping-pong within 1e-9, Unbake restores `before` and `after`.

## 11. VERIFY list

1. Folder, `DocumentV1`, `ExactTime` floor and modulo helpers.
2. The segment fraction `u` is formed from rational times before one conversion to double.
3. `BakeRecordV1`'s shape (spec 05).
4. The Position track type, path storage and the 150-sample table function.
5. Spec 07 draws the selection bracket.
6. The keymap for Ctrl+Shift+R and Ctrl+Alt+R; menu names.
7. The Lottie exporter's bake hook; the work area as a bake range.
8. The timeline's draw cost per ghost, for the cap of 500.

## 12. Open questions for the user

1. Proposed additions to 00 §4: `Ctrl+Shift+R [Cmd+Shift+R]: open the Retime dialog for the selected keys (spec 08)` and `Ctrl+Alt+R [Cmd+Option+R]: Time-Reverse the selected keys in place (spec 08)`.
2. Is "loop the last N keys" (After Effects' second `loopOut` argument) wanted?
3. Spec 05 lists `continue` under fit bake; one key on the line is exact. Agree on exact?
4. Should Anchor = Playhead scale about a playhead outside the selection, or fall back to the nearer edge?
