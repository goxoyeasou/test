# Spec 04: the graph editor

Written without the repository open; every assumption about unseen code is marked `VERIFY:` and gathered in section 11. Depends on 00, 01 and 06. Report 1 is `reports/Smooth keyframing for WebGL motion graphics.md`; report 2 is `reports/AE keyframing hacks and plugins.md`.

## 1. Purpose and evidence

One value graph with Bézier handles on every channel of every property, merged Position included, so AE's two largest complaint clusters (value graph versus speed graph, and the Separate Dimensions trap that removes motion-path handles) cannot arise (report 1 §2; report 2 §2). The graph owns no state the timeline does not: selection, pan and zoom survive the switch and undo (report 1 §2: AE deselects keys when the graph opens, Rive undoes a pan; 00 Ruling 11). The rest is small: a numeric transform box, Fit with shortcuts, select-all in the graph, Blender's normalised view, a marquee fast past 100 keys (report 2 §1, §2, §5; report 1 §2). Demand rank 4 of 12 (report 2 §6).

## 2. Scope and non-goals

In scope: the pane, drawing and hit testing, handle editing, Split and Merge channels, normalised view, framing, the transform box, overlays, the strip, performance. Out of scope: the Ease popover and library (01), paste (03), drivers (05), selection commands and Absolute/Offset (06), the bar drag model (07), retime and loops (08), labels and the summary row (09). No speed-graph editing mode is planned; roving keys are an open question.

## 3. Rulings

- **04-R1 (one graph; speed is an overlay):** the graph edits values only; speed is a toggled, read-only secondary curve per channel from the same samples. Why: AE's two modes are its largest confusion (report 2 §2); a derivative needs no handles. Cost if wrong: an unused toggle.
- **04-R2 (handle to ease):** an outgoing handle at absolute `(t, v)` on segment `[t0, t1]` of channel `k` sets `x1 = clamp((t − t0) / (t1 − t0), 0, 1)` and `y1 = (v − v0k) / (v1k − v0k)`; the incoming handle sets `x2, y2` likewise; every other channel redraws from the shared ease (00 Ruling 3). When `v1k = v0k` the handle sits on a flat guide at the key's value and vertical drag is ignored for that channel. Why: the inverse of the draw. Cost if wrong: one function.
- **04-R3 (channels drawn in the blend space):** colour draws L, a, b and alpha in Oklab; Position draws X and Y of the evaluated path; other vectors their stored components. Why: the ease applies to the Oklab blend fraction (report 1 §1), so only there is a colour curve the cubic its handle describes. Cost if wrong: one conversion at draw time.
- **04-R4 (split is a command, never a mode):** Split channels makes per-channel scalar tracks that own their eases; Merge requires equal key times or offers to add the missing keys; a split record makes split then merge the identity. Why: 00 Ruling 3; the Separate Dimensions trap (report 1 §2). Cost if wrong: one record type.
- **04-R5 (joined by default; Alt breaks; the target resolves Alt):** the two handles meeting at a free key keep one slope unless the key is marked broken. Alt+drag on a handle breaks and moves it alone; on a key duplicates (spec 10); on empty space the marquee subtracts (00 §4). Priority: handle, key, empty, nearest within 8 px; the evaluator never reads the join flag. Why: report 1 §1 asks for an explicit continuous/broken flag; 00 Ruling 13 forbids a private modifier. Cost if wrong: one flag, one priority order.
- **04-R6 (a 2-D canvas sampling the one evaluator):** two stacked `<canvas>` 2-D contexts (curves; glyphs and selection); every curve is `evaluateTrackV1` sampled twice per CSS pixel column at exact rational times. Why: the preview owns the WebGL context in the worker (report 1 §3); sampling the evaluator keeps 00 Ruling 4 literal; two per column so a hold step inside a column is never skipped. Cost if wrong: a renderer swap behind `graphSamplesV1`.
- **04-R7 (view state is not document state):** keys and segments live in the shared `KeySelectionV1` (00 Ruling 11); handle selection, pan, zoom and the toggles live in `GraphViewStateV1`; open, close, undo, redo and playback touch none of it. Why: report 1 §2, report 2 §5. Cost if wrong: nothing.
- **04-R8 (performance budget):** 2,000 keys visible, 200 selected, marquee at 60 Hz: the marquee repaints only the glyph layer and hit-tests through a per-track time-sorted index. Why: AE's timeline slows past about 100 marquee-selected keys (report 2 §1); 2,000 is twenty times that onset. Cost if wrong: T11 fails.

## 4. Model and interfaces

```ts
// src/animation-core/keyframes/graph-editor-model.ts   VERIFY: folder and the 00 §3 names. Fields readonly.
export type HandleJoinV1 = 'joined' | 'broken'   // proposed on KeyframeV1 (section 12); editor-only
export interface GraphViewStateV1 {               // 04-R7: never in the undo stack
  timeStart: number; timeEnd: number; valueMin: number; valueMax: number   // seconds; units, or −1..1 when normalised
  normalised: boolean; speedOverlay: boolean; postDriver: boolean; autoZoomHeight: boolean; dopeStrip: boolean
  handles: ReadonlySet<string>                   // `${trackId}@${time}:${channel}:out|in`
}

type Span = [t0: number, t1: number, v0: number, v1: number]
export const absoluteHandlesV1 = (ease: EaseV1, span: Span): { p1: [number, number]; p2: [number, number] } | null   // null for hold and procedural
export const easeFromHandleDragV1 = (ease: EaseV1, end: 'out' | 'in', t: number, v: number, span: Span): EaseV1   // 04-R2
export const joinedNeighbourEaseV1 = (moved: EaseV1, movedSpan: Span, neighbour: EaseV1, neighbourSpan: Span, atKey: 'in' | 'out'): EaseV1
export const autoHandleEaseV1 = (prev: [number, number] | null, here: [number, number], next: [number, number] | null, span: Span): EaseV1   // Ruling 5 slope, x = 1/3; VERIFY spec 01

export interface SplitRecordV1 { sourceTrackId: string; channelTrackIds: readonly string[]; pathTangents?: readonly unknown[]; c2: boolean }   // VERIFY: path tangent type (00 Ruling 10)
export type MergeResultV1<V> = { kind: 'merged'; track: TrackV1<V>; droppedEases: readonly number[] } | { kind: 'mismatch'; missing: readonly { trackId: string; time: ExactTime }[] }
export const splitChannelsV1 = <V>(track: TrackV1<V>, trackId: string): { tracks: readonly TrackV1<number>[]; record: SplitRecordV1 }
export const mergeChannelsV1 = <V>(tracks: readonly TrackV1<number>[], record: SplitRecordV1 | null): MergeResultV1<V>
export const addMissingKeysV1 = (tracks: readonly TrackV1<number>[], missing: readonly { trackId: string; time: ExactTime }[], ctx: EvalContextV1): readonly TrackV1<number>[]   // value = evaluateTrackV1 there, tangent 'auto'

export const normalisedValueV1 = (v: number, range: { min: number; max: number }): number   // 2·(v − min)/(max − min) − 1; 0 when max === min
export const graphSamplesV1 = <V>(track: TrackV1<V>, channel: number, view: GraphViewStateV1, widthPx: number, ctx: EvalContextV1, withDrivers: boolean): Float64Array   // 2·widthPx samples
export const speedSamplesV1 = (samples: Float64Array, secondsPerSample: number): Float64Array   // central difference

export interface TransformBoxV1 { timeScale: number; valueScale: number; leftEdge: ExactTime; bottomEdge: number; ripple: boolean }
export const applyTransformBoxV1 = (doc: Document, selection: KeySelectionV1, box: TransformBoxV1, fps: number, subFrame: boolean): Document   // Rulings 7 and 8
export const marqueeHitV1 = (index: KeyIndexV1, rect: { t0: number; t1: number; v0: number; v1: number }, mode: 'replace' | 'add' | 'subtract', current: KeySelectionV1): KeySelectionV1   // KeyIndexV1: per-track arrays sorted by time
```

## 5. UX

**Surface.** Ctrl+Shift+G (00 §4) toggles the graph for the selected tracks, else the keyed tracks of the selected layers. Bars become curves; the track list gains a chip per channel (click hides, Alt+click solos). A 24 px dope-sheet strip (spec 09) at the bottom shows every visible track's keys on the same selection. Toolbar: Fit selection, Fit all, Auto zoom height, Normalise, Show speed, Show behaviours, Split or Merge channels (one button).

**Curves and glyphs.** A channel is a 1.5 px line in its colour (`VERIFY:` the app's X, Y, Z colours). Keys are 8 px diamonds, handles 8 px circles on a stem, both 8 px hit targets. An auto key (00 Ruling 5) draws its handles with a lock glyph; hover: "Auto ease. Click or drag a handle to edit it. Reset to Auto in the right-click menu". A broken key draws a square. A hold segment is a step: flat to the next key, then vertical. A spring, elastic or bounce segment (spec 01) draws its evaluated curve with no handles and a midpoint badge ("Spring 0.5 s · 0.3") that opens the Ease popover. Show behaviours adds the post-driver curve (spec 05) dashed, not editable; keys and handles stay on the base curve. Outside the first and last key the extrapolation (spec 08) draws at 35 % opacity (below the 50 % that reads as a curve), not selectable. Show speed draws `speedSamplesV1` per channel, muted, on a right-hand axis in units per second (px/s along the path for Position), never hit-testable.

**Handle drag.** A drag starts after 3 px of travel, so a selecting click never nudges a handle. The handle maps by 04-R2 and the segment becomes `free` at both ends (00 Ruling 5). At a joined key the handle across the key keeps its x and takes the y that gives the same slope on the dragged channel (for Position, in path fraction per second, 00 Ruling 10). Named limit: another channel is continuous only when its deltas across both segments share a ratio. Zero-delta channel: flat guide, vertical drag ignored, hint "This channel doesn't move across this ease, so its handle only sets the timing. Drag sideways, or drag a handle on a channel that moves". x clamps to the segment (00 Ruling 1).

**Modifiers (00 §4).** Drag a key: time snapped to whole frames, value free. Shift+drag: also snaps time to other keys, markers and the playhead, and value to other keys' values on the track within 8 px, with a guide. Ctrl+drag: sub-frame time. Alt+drag: by target (04-R5). Marquee replaces, Shift adds, Alt subtracts; it takes keys and handles, never layers. Right-click a key: Reset to Auto, Join handles, Break handles, then spec 01's menu.

**Multi-handle drag.** With several handles selected the dragged handle's ratios apply to all (x 0.3 to 0.6 doubles every selected x, clamped to 1; y likewise). Named limit: a component at 0 stays at 0.

**Transform box.** Two or more selected keys show a box with eight grips and fields: W (time scale %, 100), H (value scale %, 100), X (left edge, frames), Y (bottom edge, units; disabled across mixed units unless Normalise is on). Time scales about the far edge and ripples later keys unless Shift is held (00 §4); values scale about the opposite edge, per track. Keys land on whole frames (00 Ruling 8; Ctrl for sub-frame); collisions follow 00 Ruling 7.

**Framing.** The wheel zooms about the pointer; Space+drag pans. Fit selection frames the selected keys and handles with 10 % padding (an edge glyph is unreachable), or everything when nothing is selected. Auto zoom height refits the value axis on every pan. Normalise maps each channel through `normalisedValueV1` over its own range (keyed values and absolute handles, whole track), recomputed on edit, never on pan, so 0..1 opacity and 0..1920 X share one extent. Time zoom: whole composition to 400 px per frame (a handle at x = 0.01 of a one-frame segment is then 4 px, the smallest drawable). Keys beyond 00 §4 are proposed in section 12.

**Empty and error states.** No tracks: "Select a layer or a track to see its curves". Locked layer: curves at 50 %, "Layer is locked" on any edit. Merge mismatch: "Keys don't line up on these channels", the missing frames, Add keys, Cancel.

## 6. User flows

**Flow 1 (primary): shape X and watch Y follow.** Start: Position keyed (0, 0) at 0 s and (300, 100) at 1 s, 30 fps, default ease, bar selected. 1. Ctrl+Shift+G: X (0..300) and Y (0..100) appear with locked handles. 2. Drag X's outgoing handle to (0.5 s, 30): at 3 px the lock drops; X bends; Y's handle redraws at (0.5 s, 10) and Y bends the same; the canvas shows the playhead pose and an end-pose ghost. 3. Release: one undo step. End: ease `(0.5, 0.1, 0, 1)`, both keys `free`.

**Flow 2 (keyboard-only).** Start: Opacity keyed at frames 0, 12, 24, layer selected, graph closed. 1. Ctrl+Shift+G: the curve appears framed as last left. 2. Ctrl+A (proposed): three keys and two segments select. 3. F (proposed): Fit selection. 4. 3: library slot 3 applies to both segments; the curves reshape. 5. K, K: the playhead steps to 12, then 24. 6. Ctrl+Shift+E: the Ease popover opens for the segments; Escape closes it. End: both segments carry slot 3; selection and framing unchanged.

**Flow 3 (multi-selection): retime 200 keys by typing.** Start: 2,000 keys visible on 40 tracks. 1. Marquee 200 keys across 6 tracks: the glyph layer repaints under 4 ms per move (T11); the box appears. 2. Type W 80, Enter: selected times scale to 80 % about the far edge, each on a whole frame; later keys ripple. 3. Shift+drag the right grip: the same without ripple. End: whole-frame keys (00 Ruling 7 on collisions), one undo step per commit.

**Flow 4 (undo): zoom survives.** Start: Flow 1's end. 1. Wheel-zoom four steps around the first key. 2. Ctrl+Z: the ease and locks return to auto; the view stays zoomed; selection unchanged. 3. Ctrl+Shift+Z: the ease returns, still zoomed. End: document as after Flow 1; `GraphViewStateV1` untouched.

**Flow 5 (limit): a channel that does not move.** Start: Position keyed (0, 100) to (400, 100). 1. Open the graph: Y is flat, its handles on flat guides. 2. Drag Y's outgoing handle up 40 px: nothing moves; the hint appears. 3. Drag right: x1 changes and X bends; Y stays flat. End: new x1, original y1; the hint fades on release.

**Flow 6 (split, edit, merge with a mismatch).** Start: Position keyed at frames 0, 20, 40. 1. Split channels: "Position X" and "Position Y" replace the row, eases copied; the canvas path becomes a sampled trail (section 8). 2. Drag Y's middle key to frame 24 and shape its handle. 3. Select both tracks, Merge channels: the dialog lists "Y: frame 20 missing; X: frame 24 missing". 4. Add keys: X gains 24 and Y gains 20 at their evaluated values with auto tangents; nothing on screen moves. 5. Merge: one Position track with keys 0, 20, 24, 40; toast "Merged. Y's ease was kept on 1 segment" (larger delta wins, ties to X); the path tangents return. End: one vector track; Ctrl+Z reverses merge, keys and split in three steps.

## 7. Evaluation and determinism

The graph evaluates nothing of its own. `graphSamplesV1` calls `evaluateTrackV1(track, columnTime, ctx)` at `2 · widthPx` exact rational column times (`timeStart + i · (timeEnd − timeStart) / (2 · widthPx)`), with the driver stack when `withDrivers` (`VERIFY:` an evaluator option to skip `applyDriversV1`). Nothing bakes. 00 Ruling 9 holds by construction: every drawn pixel is `evaluate(document, time)`, and every edit writes `EaseV1`, key times, values or `join` through the popover's undo stack.

## 8. Edge cases and named limits

- A split Position loses its editable path; the canvas shows a trail sampled from X(t), Y(t) until Merge restores the tangents from the record (auto tangents without one).
- On a curved path X and Y are coordinates along the eased fraction, not cubics; the handle still describes the ease exactly.
- An overshooting colour ease leaves the gamut; the swatch shows the gamut-mapped colour.
- Normalise draws a constant channel at 0. One selected key: no box. A marquee over a ghosted region selects nothing.
- The ratio drag cannot move a component at 0; hint "Drag this handle alone to move it off zero".

## 9. Interactions with other specs

01: popover and graph edit the same `EaseV1`; badges open the popover. 05: the dashed curve; drivers never editable here. 06: `KeySelectionV1` is the only selection; selecting both keys of a segment selects it (`VERIFY:`); Absolute/Offset applies to the box fields. 07: frame snapping, Shift and Ctrl, the edge grip. 08: ghosted extrapolation, ripple. 09: the strip, label colours. 10: Alt+drag duplicate, J/K scope.

## 10. Rules and tests

Each rule is one test in `graph-editor-model.test.ts` (T4 and T11 in `graph-editor.test.ts`).

- R1, T1 `handle drag on X redraws Y`: a handle drag on channel X of Position changes channel Y's curve through the shared ease. Flow 1's document; `easeFromHandleDragV1(ease, 'out', 0.5, 30, [0, 1, 0, 300])` returns `(0.5, 0.1, 0, 1)`; `absoluteHandlesV1` on Y gives `p1 = [0.5, 10] ± 1e-12`.
- R2, T2 `split then merge is the identity`: a Position track with 5 keys and mixed eases; `mergeChannelsV1(splitChannelsV1(t).tracks, record)` deep-equals `t`, `Object.is` per leaf, path tangents included.
- R3, T3 `normalised extents agree`: a 0..1 opacity and a 0..1920 position map onto the same vertical extent: `normalisedValueV1` gives −1 for 0 in both ranges and 1 for 1 and for 1920; a constant range returns 0.
- R4, T4 `selection survives open and close`: select 7 keys and 3 segments, toggle Ctrl+Shift+G twice; the `KeySelectionV1` sets are equal.
- R5, T5 `undo of a handle drag keeps the zoom`: drag a handle, zoom twice, undo; the ease equals the pre-drag ease and `GraphViewStateV1` the post-zoom state.
- R6, T6 `a hold segment draws a step`: Opacity 1 at 0 s, hold, 0 at 1 s, 2 at 2 s; `graphSamplesV1` over 0..2 s at 400 px returns 800 samples, all before 1 s equal to 1, changing value at exactly one index.
- R7, T7 `zero delta ignores vertical`: `easeFromHandleDragV1(ease, 'out', 0.5, 40, [0, 1, 100, 100])` returns `x1 = 0.5` and the input `y1`.
- R8, T8 `joined slope`: keys (0 s, 0), (1 s, 100), (2 s, 50); drag the second key's outgoing handle to `(1.4, 90)`; the incoming slope equals the outgoing within 1e-9; with `join = 'broken'` the neighbour is unchanged.
- R9, T9 `Alt by target`: Alt pointer-downs on a handle, a key and empty space, 10 px each; the first changes `join` and one ease, the second adds one key (spec 10), the third deselects one key.
- R10, T10 `typed width scales and snaps`: keys at frames 0, 7, 20, 31; W = 50 about the far edge gives 16, 19, 26, 31 (`round(31 − 0.5 · (31 − f))`, ties later); no sub-frame times.
- R11, T11 `marquee budget`: 2,000 keys on 40 tracks, 100 marquees of about 200 keys; `marqueeHitV1` visits at most `2 · (keys in the time range)` entries per call; the browser bench repaints the glyph layer under 4 ms median (16.7 ms less 8 ms for a curve redraw and 4 ms margin). If the bench fails: STOP.
- R12, T12 `Add keys is invisible`: Flow 6's tracks; evaluate frames 0..40 before and after `addMissingKeysV1`; `Object.is` on both channels.
- R13, T13 `procedural badge`: `absoluteHandlesV1` returns null for spring, elastic and bounce; the pane model lists one badge per such segment.
- R14, T14 `auto key click and reset`: a click on a locked handle sets `tangent` to `free` with the ease equal to `autoHandleEaseV1` to the bit; Reset to Auto restores `'auto'` and the same ease.
- D1 `play equals seek`: Flow 1's document after the drag plus a hold on Opacity; frames 0..60 in order, then frame 60 alone from a fresh evaluator; `Object.is` on every channel.
- D2 `random order`: a shuffled frame list evaluates equal to D1.
- D3 `after an edit`: drag a handle, undo, evaluate; equal to the values before the edit.
- D4 `bake round-trip`: the graph bakes nothing; a spring-eased Scale track baked by spec 05, sampled by `graphSamplesV1`, lies within spec 05's tolerance of the live spring at every frame, and unbake restores the parameters exactly.

## 11. VERIFY list

1. The folder and names of 00 §3; whether the evaluator can skip the driver stack.
2. Spec 01's rule for auto ends (x = 1/3 assumed from Blender, report 1 §1) and how an auto end overrides stored ease numbers.
3. The spatial path tangent type of 00 Ruling 10, for `SplitRecordV1`.
4. The evaluator blends colour in Oklab (report 1 §1); if not, 04-R3 draws the stored channels instead.
5. The app's keymap for the proposed keys (section 12); spec 06's rule that selecting both keys of a segment selects it.
6. The undo stack records one step per release (00 §8 item 8); the timeline's canvas sizing.

## 12. Open questions for the user

1. Roving keys (report 2 §2) are not here; Adobe does not publish the solver (report 1 §1). Confirm they wait, or that Constant Speed (spec 08) suffices.
2. Should handle selection join `KeySelectionV1`, or stay graph-local as 04-R7 says?
3. Merge keeps the ease of the channel with the larger delta rather than asking per segment. Confirm.
4. Proposed addition to 00 §3: `readonly join?: HandleJoinV1` on `KeyframeV1`, default `'joined'`, editor-only.
5. Proposed addition to 00 §4 (keyboard): `F` Fit selection, `Shift+F` Fit all, `N` toggle Normalise, `Ctrl+A` select all keys in the graph (the December 2025 AE request, report 2 §1).
6. Proposed wording change to 00 §4, row "Shift + drag a key": add "and, in the graph, to other keys' values".
