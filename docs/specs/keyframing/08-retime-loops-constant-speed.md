# Spec 08: retime, loops and constant speed

Written without the repository open; every assumption about unseen code is marked `VERIFY:` and gathered in section 11. Depends on 07 (drag model, snapping) and 01 (eases); uses 03 (ease flip), 05 (bake service) and 06 (selection, runs).

## 1. Purpose and evidence

After Effects retimes a key group with Alt-drag, which leaves sub-frame keys and cannot be typed; loops with `loopOut`, which Lottie cannot carry; smooths speed along a path only through an undocumented roving-key solver; and its expert workaround for easing many keys at once is "precompose, then time remap" (report 2 §3). Its Time-Reverse ignores a sparse selection and a rate change leaves keys on half frames (report 2 §5); report 2 §6 ranks whole-frame snapping seventh and retime, loops and Constant Speed eighth. Report 1 §2 names the gestures to borrow: ripple from Motion Canvas, retime markers from Maya, Absolute Time Snap from Blender. Report 1 §1 gives the extrapolation formulas, Blender's cyclic tangent solve, the arc-length path model and the roving-keys idea, and notes that Lottie has no extrapolation, so loops bake on export.

## 2. Scope and non-goals

In scope: proportional retime by the selection's edge handles with ripple; the Retime dialog; retime markers on a run; six extrapolation modes with exact formulas, the seam rule, ghost keys and bake; Constant Speed on Position tracks; Time-Reverse of a sparse selection in place.

Not in scope: Time Remap as a keyed time curve on a precomp; speed ramps on footage; stretching a layer bar as such (`VERIFY:` in and out points; a layer retime is the retime of all its keys); smoothing and key reduction (spec 05); looping only the last N keys (section 12).

## 3. Rulings

- **08-R1 (scale first, then snap each key):** `t' = snapToFrameV1(a + s × (t − a))`, `a` the anchor, `s` the scale. Why: Ruling 8; snapping the scale instead would allow only spans that divide evenly. Cost if wrong: one line.
- **08-R2 (scaling down stops at the minimum span; a retime never merges selected keys):** the span stops at the smallest value at or above the requested one where every selected key keeps its own frame; the handle resists and the readout names the minimum. Why: the selection is the thing being shaped, and a merge mid-drag deletes a pose at the moment the user cannot see it; Ruling 7 still applies to a selected key landing on an unselected one (Shift-isolated scale-up), the ordinary drop rule. Cost if wrong: replace the stop with Ruling 7's replacement in one function.
- **08-R3 (ripple moves the keys on the moving edge's side):** on each track with a selected key, unselected keys beyond the edge that moved shift by that edge's whole-frame displacement; Shift isolates. Why: report 1 §2 (Motion Canvas: fixing one pause fixes everything after it). Cost if wrong: one filter.
- **08-R4 (linear is the secant, continue is the tangent):** `linear` extrapolates along the line through the last two keys; `continue` along the ease's end tangent (After Effects' "velocity at the last keyframe"); both are flat after a hold. Why: both exist in the sources and differ exactly when the end ease is eased out, where continue holds and linear keeps moving. Cost if wrong: two slopes swapped.
- **08-R5 (cyclic auto tangents):** in cycle, offset and ping-pong an end key with tangent mode `auto` takes its missing neighbour from the wrapped copy, so the seam is C1 by construction; a seam marker is drawn when the loop is not continuous. Why: report 1 §1 (Blender solves cyclically; Maya and After Effects show a bump). Cost if wrong: one neighbour lookup.
- **08-R6 (Constant Speed is one ease across the run):** with the toggle on, intermediate key times are derived from arc length and only the first segment's start handle and the last segment's end handle apply, as one bezier across the run; intermediate eases are kept but unused. Why: report 1 §1 (roving keys: one speed curve across several path segments); a per-segment ease and a constant speed contradict. Cost if wrong: the toggle hides eases it should show.
- **08-R7 (reverse keeps times):** Time-Reverse keeps the selected keys' times and reverses their values and the eases between consecutive selected keys, mirrored through spec 03's flip. Why: report 2 §5 ("ignoring my carefully selected subset"). Cost if wrong: nothing; spec 06 asks for the same.

## 4. Model and interfaces

```ts
// src/animation-core/keyframes/retime.ts                  VERIFY: folder, DocumentV1 (00 §3); Vec2 is { x: number; y: number }
export interface RetimeRequestV1 {
  readonly keys: readonly string[]; readonly anchor: 'first' | 'last' | 'playhead'; readonly playhead: ExactTime
  readonly newSpan: number; readonly ripple: boolean; readonly subFrame: boolean     // frames; Shift clears ripple; Ctrl sets subFrame
}
export interface RetimeResultV1 {
  readonly document: DocumentV1; readonly span: number; readonly scale: number
  readonly rippled: number; readonly replaced: number; readonly stoppedAtMinimum: boolean; readonly unscaledEases: number
}
export const retimeV1 = (doc: DocumentV1, req: RetimeRequestV1, fps: number): RetimeResultV1
export const minimumRetimeSpanV1 = (framesByTrack: readonly (readonly number[])[], anchor: number, requested: number, current: number): number
export const retimeMarkerDragV1 = (run: readonly number[], bead: number, delta: number): readonly number[]

// src/animation-core/keyframes/extrapolation.ts
export interface RemappedTimeV1 { readonly time: ExactTime; readonly cycles: bigint; readonly reversed: boolean }
export const remapTimeV1 = (track: TrackV1<unknown>, time: ExactTime): RemappedTimeV1          // identity inside [t0, t1]
export const extrapolatedValueV1 = <V>(track: TrackV1<V>, time: ExactTime, ctx: EvalContextV1): V
export const endSlopeV1 = <V>(track: TrackV1<V>, end: 'before' | 'after', rule: 'secant' | 'tangent'): V   // per second
export const seamV1 = (track: TrackV1<unknown>): { readonly before: boolean; readonly after: boolean }
export const ghostKeysV1 = (track: TrackV1<unknown>, range: readonly [ExactTime, ExactTime], cap: number): readonly ExactTime[]
export const bakeExtrapolationV1 = <V>(track: TrackV1<V>, range: readonly [ExactTime, ExactTime], fps: number): TrackV1<V>   // spec 05's bake service; VERIFY its name

// Position tracks (Ruling 10)                               VERIFY: the Position track type and the 150-sample table
export interface PositionTrackV1 extends TrackV1<Vec2> { readonly constantSpeed: boolean }
export const constantSpeedFramesV1 = (first: number, last: number, segmentLengths: readonly number[], runEase: EaseV1): readonly number[]
export const positionAtConstantSpeedV1 = (track: PositionTrackV1, time: ExactTime): Vec2
export const reverseSelectionV1 = (doc: DocumentV1, sel: KeySelectionV1): DocumentV1        // spec 06 and 10 call it
```

**Retime.** `F`, `L` are the selection's first and last frames, `a` the anchor, `S = L − F`, `s = S' / S` for the requested span `S'`; `t'_k = snap(a + s × (t_k − a))`, unsnapped with `subFrame`. If two selected keys of one track share a frame and `S' < S`, `S'` rises to `minimumRetimeSpanV1`, the smallest integer in `[S', S]` with all frames distinct (`n − 1` is a floor; `0, 1, 10` needs 5). Ripple: on each track with a selected key, unselected keys later than `L` move by `t'_L − L`, earlier than `F` by `t'_F − F`. Isolated: they stay, and a selected key landing on one replaces it (Ruling 7), counted. Eases are unchanged because normalised handles are time-free (Ruling 1); auto tangents are recomputed; spring, elastic and bounce keep their absolute seconds, counted in `unscaledEases` (named limit).

**Markers.** A run `r_0..r_{m−1}`, `m ≥ 3`, has a bead at each segment's midpoint. Dragging bead `k` by `Δ` sets `r'_{k+1} = r_{k+1} + Δ` and scales the keys after it about the last key: `r'_j = snap(r_{m−1} − (r_{m−1} − r_j) × (r_{m−1} − r'_{k+1}) / (r_{m−1} − r_{k+1}))`; keys up to `k` and the last stay. Limits: `r'_{k+1} ≥ r_k + 1` and 08-R2 on the trailing keys.

**Extrapolation.** `t0`, `t1`, `T = t1 − t0 > 0`, `v0`, `v1`, `keyed(t)` on `[t0, t1]`; `n = floor((t − t0) / T)` as an exact integer, `p = t − t0 − n × T`:
- constant: `v0` before `t0`, `v1` after `t1`.
- linear: `v0 + m × (t − t0)` or `v1 + m × (t − t1)`, `m` the secant of the end segment in value per second.
- continue: as linear with `m` the ease's end tangent: bezier `y1 / x1 × Δv / Δt` at a start, `(1 − y2) / (1 − x2) × Δv / Δt` at an end (a degenerate `x` falls back to the secant); hold, spring, elastic, bounce give 0.
- cycle: `keyed(t0 + p)`.
- ping-pong: `q = t − t0 − 2T × floor((t − t0) / 2T)`; `keyed(t0 + q)` when `q ≤ T`, else `keyed(t0 + 2T − q)`.
- offset: `keyed(t0 + p) + n × (v1 − v0)` per channel; `n < 0` before `t0`.

Each end has its own mode; one key means constant; hold-only properties offer constant, cycle and ping-pong. Under 08-R5 the first key's missing neighbour is `(t_{n−2} − T, v_{n−2} − d)` and the last key's `(t_1 + T, v_1 + d)`, `d = v1 − v0` for offset and 0 for cycle; ping-pong reflects, `(t0 − (t_1 − t0), v_1)` and `(t1 + (t1 − t_{n−2}), v_{n−2})`, which Ruling 5's formula turns into a flat tangent. `seamV1` is true at an end when the value steps (cycle, `v0 ≠ v1`), a free end slope differs from its wrapped neighbour's, or a ping-pong end slope is not 0.

**Bake.** Copies keys per cycle inside the range: cycle `(t_i + kT, v_i, ease_i)`; offset adds `k × (v1 − v0)`; ping-pong writes odd cycles reversed with flipped eases (spec 03); linear and continue add one key at the range end on the line with the linear ease. Auto end tangents are written as `free` with their cyclic handles. The record keeps `{ before, after }` for Unbake.

**Constant Speed.** Segment lengths `L_j` come from the 150-sample tables (Ruling 10); `F_i = Σ_{j<i} L_j / Σ L_j`. The run ease `E` is the bezier `(x1, y1)` of segment 0 with `(x2, y2)` of the last segment when both are bezier with `y1, y2 ∈ [0, 1]`; otherwise linear, with a notice. `s_i = solveBezierXV1(y1, y2, F_i)` (the y cubic through Ruling 4's solver with axes swapped, valid because `y` is then monotone), `u_i = X(s_i)`, `t_i = snap(t0 + u_i × T)`; equal derived frames are pushed to the next frame in order. The position is `pathAtFraction(E((t − t0) / T))` over the concatenated tables.

**Reverse.** Per track, the selected keys sorted `k_0..k_{m−1}`: `k_j` takes the value of `k_{m−1−j}`. A segment `(k_j, k_{j+1})` whose keys are adjacent in the track takes `flip` of the mirror segment `(k_{m−2−j}, k_{m−1−j})` when that one is adjacent too; otherwise it keeps its ease. Position keys swap path points and spatial handles (`VERIFY:` path storage). Spring, elastic and bounce eases are kept and counted.

## 5. UX

**Edge handles.** A bracket spans the selection's first and last frames across the involved rows (`VERIFY:` spec 07 draws it). Dragging a handle retimes about the far edge (00 §4); Shift isolates; Ctrl frees sub-frame placement. Readout beside the handle: `Span 30 f → 45 f · ×1.50 · 0, 15, 30, 45 · ripple +15 f (1 key)`; `no ripple (Shift)` when isolated; `1 key replaced` when Ruling 7 fired; at the stop `Minimum span 3 f: every key keeps its frame`, the handle in the resist colour, unmoved. The canvas renders the playhead's frame live (`VERIFY:` 00 §8.4).

**Retime dialog.** New span (frames) and Scale (%), linked, Scale shown to one decimal; Anchor: First key (default) | Playhead | Last key; Ripple (default on); Apply. It previews live, commits on Apply or Enter as one undo step, discards the preview on Escape; a span below the minimum clamps on Apply and reads `Minimum span 3 f`. Opened by Ctrl+Shift+R [Cmd+Shift+R] (proposed, section 12) or `Animate > Retime…` (`VERIFY:` menu names).

**Retime markers.** On a run of three or more keys the context menu offers `Show retime markers`, a view toggle outside undo (Ruling 11). Beads sit at segment midpoints; dragging one previews live, lists the run's frames in the readout and commits one undo step per release.

**Loops.** Each track's inspector shows Before and After: Constant (default) | Linear | Continue | Cycle | Ping-pong | Offset; hold-only properties list Constant, Cycle, Ping-pong. Ghost keys are drawn dimmed in the extrapolated part of the visible range, not selectable, hover `Looped: cycle. Bake to edit.`, at most 500 per track per view (report 2 §1 records the timeline slowing past 100 selected keys; ghosts are cheaper, so five times that is the budget; `VERIFY:` the draw cost). The graph (spec 04) draws the extrapolated curve dashed. A seam marker, a small triangle at the end key, carries hover text such as `The loop jumps at 24 f: first and last values differ. Use Offset to keep turning.` The context menu offers `Bake loop to keys` over the composition (`VERIFY:` the work area as an option) and `Unbake`.

**Constant speed.** A checkbox on a Position track. On: intermediate keys are drawn as circles (After Effects' roving glyph) at derived times and cannot be dragged in time (hover `Constant speed sets this key's time. Turn it off to drag.`); the end keys drag normally; the first and last segments' Ease popovers edit the run's start and end handles and say `Edits the ease of the whole run`; an intermediate segment's popover reads `Constant speed is on: this ease is not used`, with a Turn off button. A non-bezier or overshooting end ease shows `Constant speed uses a linear ease here`.

**Time-Reverse.** `Animate > Time-Reverse keys`, Ctrl+Alt+R [Cmd+Option+R] (proposed), acts on the selection and reports `2 eases kept their direction` when springs, elastics or bounces are present.

**Empty and error states.** One selected key: no bracket. Fewer than three keys: `Retime markers need three keys`. One key on a track: Before and After disabled, `Add a second key to loop`. Sub-frame keys that would snap together: the handle resists with `Snap keys to frames first (Ctrl+Shift+F)`.

## 6. User flows

**Flow 1, primary (edge handle with ripple).** Start: a Scale track with keys at 0, 10, 20, 30 f selected and an unselected key at 40 f; playhead at 12 f. (1) Press the right handle and drag to 45 f: keys preview at 0, 15, 30, 45 f, the key at 40 f at 55 f; readout `Span 30 f → 45 f · ×1.50 · 0, 15, 30, 45 · ripple +15 f (1 key)`; the canvas shows frame 12 with the new timing. (2) Release. End: keys at 0, 15, 30, 45, 55 f; one undo step; selection and bracket kept.

**Flow 2, keyboard only (Retime dialog).** Start: as flow 1 before step 1. (1) Ctrl+Shift+R: the dialog opens with New span 30, Scale 100 %, Anchor First key, Ripple on, focus in New span. (2) Type `45`, Tab: Scale reads 150 %; preview 0, 15, 30, 45 and 55 f. (3) Type `50%` into Scale, Tab: New span reads 15; preview 0, 5, 10, 15 f and the key at 40 f at 25 f. (4) Enter. End: keys at 0, 5, 10, 15, 25 f; one undo step.

**Flow 3, multi-selection, isolated.** Start: Position keys at 0 and 30 f, Scale keys at 10 and 20 f, Opacity keys at 0, 15, 30 f, all selected; an unselected Opacity key at 40 f. (1) Shift-drag the right handle to 60 f: Position 0, 60; Scale 20, 40; Opacity 0, 30, 60; the key at 40 f stays; readout `Span 30 f → 60 f · ×2.00 · no ripple (Shift)`. (2) Release. End: as previewed; one undo step.

**Flow 4, undo.** Start: the end of flow 1. (1) Ctrl+Z: keys at 0, 10, 20, 30 and 40 f; the selection and bracket stay (Ruling 11). (2) Ctrl+Shift+Z: 0, 15, 30, 45, 55 f again. End: as flow 1.

**Flow 5, limit (minimum span).** Start: keys at 0, 1, 2, 3 f selected. (1) Drag the right handle left to 1 f: nothing moves, the handle stays at 3 f in the resist colour, readout `Minimum span 3 f: every key keeps its frame`. (2) Release: no edit, no undo step. (3) Select keys at 0, 1, 10 f and drag the right handle to 2 f: the preview stops at 0, 1, 5 f, readout `Minimum span 5 f`. (4) Release. End: keys at 0, 1, 5 f; one undo step.

**Flow 6, loop and bake.** Start: a Rotation track keyed 0° at 0 f and 360° at 24 f, linear ease, auto tangents; composition 120 f. (1) After = Cycle: ghosts at 48, 72, 96, 120 f; frame 30 shows 90°; a seam marker at 24 f with the hover text of section 5. (2) After = Offset: the marker goes; frame 30 shows 450°, frame 54 shows 810°. (3) `Bake loop to keys`: keys at 0, 24, 48, 72, 96, 120 f with values 0 to 1800°; After reads Constant with a `Baked` badge; `Unbake` is offered. End: three undo steps.

**Flow 7, Constant Speed.** Start: a Position path through (0, 0), (100, 0), (400, 0) keyed at 0, 20, 40 f, linear eases. (1) Tick Constant speed: the middle key becomes a circle at 10 f; readout `Keys retimed by path length`. (2) Double-click the first segment bar and pick ease-in-out (0.42, 0, 0.58, 1): the popover says it edits the whole run; the middle key moves to 14 f. (3) Drag the middle key: refused, with the hover text. (4) Untick: the key stays at 14 f; the stored eases return. End: three undo steps.

**Flow 8, retime markers, then reverse.** Start: an Opacity run at 0, 10, 20, 40 f with values 0, 25, 50, 100, all selected. (1) `Show retime markers`: beads at 5, 15, 30 f. (2) Drag the bead at 5 f right by 10 f: keys 0, 20, 27, 40 f (the key at 20 scales about 40: 40 − 20 × 20 / 30 = 26.7, snapped 27); the readout lists them. (3) Release: one undo step. (4) Ctrl+Alt+R: values 100, 50, 25, 0 at the same frames, eases swapped and flipped. (5) Ctrl+Alt+R: the original values and eases. End: three undo steps.

## 7. Evaluation and determinism

`evaluateTrackV1` becomes: `r = remapTimeV1(track, t)`; `v = keyed(r.time)` inside `[t0, t1]`, else the formulas of section 4; `v += r.cycles × (v1 − v0)` for offset; then the driver stack. With Constant Speed on, a Position track evaluates `pathAtFraction(E((t − t0) / T))` and ignores intermediate key times. Ruling 9 statement: every value is `evaluate(document, t)`; the remap is closed-form in `t`, the floor is exact integer arithmetic, no state crosses frames, so play, seek and export agree bit for bit. Loops never bake for playback; they bake for Lottie export (report 1 §1) and on request, through spec 05's bake service, which keeps the source modes for Unbake. Cycle bakes are bit-exact provided the segment fraction `u` is formed from rational times before the one conversion to double (`VERIFY:`); offset and ping-pong bakes differ by floating-point reassociation, bounded at 1e-9 relative, far above double rounding and far below a pixel.

## 8. Edge cases and named limits

- Sub-frame keys that would snap together cannot be retimed with snapping; the handle resists (section 5).
- A track whose selected keys all share one frame moves with the retime but does not scale.
- Springs, elastics and bounces keep their absolute duration under a retime; named limit until spec 01 decides whether to scale them.
- Offset on a colour adds per channel in the interpolation space and gamut-maps; large cycle counts clip. Named limit.
- Offset on Position adds the start-to-end displacement per cycle, which gives a walk cycle for free.
- The ghost cap drops the farthest ghosts first; ghosts stop at the composition end.
- With Constant Speed on, the layer passes through an intermediate key's point within that key's frame, not exactly on it, because the time is snapped and the motion is not.
- A Time Offset (spec 02) applies before the remap, so a looping track loops later by its delay.

## 9. Interactions with other specs

01: eases survive a retime exactly (normalised handles); the run ease under Constant Speed is an ordinary `EaseV1`. 02: offset, then extrapolate. 03: the flip used by ping-pong bakes and by reverse (`VERIFY:` its name). 04: the graph draws extrapolation dashed and shows the run ease's two handles under Constant Speed. 05: the bake service and its record; Settle reads the extrapolated curve. 06: `reverseSelectionV1` is the selection-faithful reverse that spec asks for. 07: the bracket, snapping, Ctrl for sub-frame; Shift's meaning on the handle is isolate (00 §4). 09: ghost keys appear in the summary row; the roving glyph is a key style. 10: Paste Reversed calls `reverseSelectionV1` on the pasted keys.

## 10. Rules and tests

Each rule is one test in `retime.test.ts` or `extrapolation.test.ts`; numbers are exact unless a tolerance is stated.

- R1 `scaling 0..30 f by 1.5 about the first key`: keys 0, 10, 20, 30 → 0, 15, 30, 45.
- R2 `ripple moves a key at 40 f to 55 f`; `Shift keeps it at 40 f`.
- R3 `scale first, then snap`: span 38 → 0, 13, 25, 38 (ideal 12.67, 25.33).
- R4 `scaling 0..3 f down by 0.5 stops at the minimum span`: keys 0, 1, 2, 3 stay, `stoppedAtMinimum` true, span 3; keys 0, 1, 10 requested span 2 → 0, 1, 5.
- R5 `anchor at the playhead`: keys 0, 10, 20, 30, playhead 10, span 60 → −10, 10, 30, 50; a later key at 40 → 60.
- R6 `the dialog links span and scale`: 45 ↔ 150 %.
- R7 `a bead moves only the keys on one side`: run 0, 10, 20, 40, bead 0, +10 → 0, 20, 27, 40.
- R8 `cycle of a 0..24 f track at frame 30 equals frame 6`, `Object.is`.
- R9 `offset adds exactly one delta per cycle`: frame 30 equals `keyed(6) + 360`, frame 54 equals `keyed(6) + 720`, frame −18 equals `keyed(6) − 360`, each `Object.is` against the same expression.
- R10 `ping-pong at frame 30 equals frame 18`, `Object.is`.
- R11 `linear and continue differ after an ease-out`: 0 → 100 over 24 f with `DEFAULT_EASE_V1`: continue holds 100 at frame 30, linear gives 125; with the linear ease both give 125.
- R12 `cyclic auto tangents are C1`: keys 0, 8, 16, 24 f with values 0, −4, 4, 0 in cycle: both end slopes are −0.5 per frame, `Object.is`.
- R13 `seam marker`: cycle 0 → 360 shows it; offset of the same does not; ping-pong with a non-flat free end shows it, with auto ends it does not.
- R14 `ghost keys`: 0..24 f cycle in a 120 f view → 4 ghosts; a 10,000-frame view returns 500.
- R15 `constant speed gives the middle key its arc-length share`: flow 7's path → 10 f; with (0.42, 0, 0.58, 1) → 14 f (`u` 0.346 by the same solver); the position at 10 f is (100, 0) within 1e-9.
- R16 `reversal is an involution`: four keys, values and times `Object.is` after two reversals, ease handles within 2⁻⁵² (two subtractions from 1); the sparse case of section 4 keeps the unselected key and the non-adjacent segment's ease.
- R17 `hold-only tracks offer three modes` and evaluate cycle and ping-pong as holds.
- D1 `play equals seek`, D2 `random order`, D3 `after an edit` on a fixture with an offset loop, a Constant Speed path and a Time Offset of 3 f; D4 `bake round-trip`: cycle `Object.is` at every frame of 0..120, offset and ping-pong within 1e-9, Unbake restores `before` and `after`.

## 11. VERIFY list

1. Folder, `DocumentV1`, `ExactTime` floor and modulo helpers.
2. The segment fraction `u` is formed from rational times before one conversion to double (bit-exact cycle bakes).
3. Spec 05's bake service and record; spec 03's flip name.
4. The Position track type, path storage and the 150-sample table function.
5. Spec 07 draws the selection bracket and exposes Shift and Ctrl to the handle.
6. The keymap for Ctrl+Shift+R and Ctrl+Alt+R; menu names.
7. The Lottie exporter's bake hook and whether the work area is a bake range.
8. The timeline's draw cost per ghost, for the cap of 500.
9. Whether layers have in and out points.

## 12. Open questions for the user

1. Proposed additions to 00 §4: `Ctrl+Shift+R [Cmd+Shift+R]: open the Retime dialog for the selected keys (spec 08)` and `Ctrl+Alt+R [Cmd+Option+R]: Time-Reverse the selected keys in place (spec 08)`.
2. Is "loop the last N keys", After Effects' second `loopOut` argument, wanted, or is a separate track enough?
3. Should springs, elastics and bounces scale their duration with a retime (spec 01's call)?
4. When the playhead is outside the selection, should Anchor = Playhead still scale about it, or fall back to the nearer edge?
