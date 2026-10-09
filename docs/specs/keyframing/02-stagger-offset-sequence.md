# Spec 02: stagger, offset and sequence

Written without the repository open; every assumption about unseen code is marked `VERIFY:` and gathered in section 11. Depends on 00, 06 (selection, runs) and 07 (drag model, snapping); takes groups from 09.

## 1. Purpose and evidence

One gesture and one live popover replace Sequence Layers, Quick Offset, the `valueAtTime(time − d)` rigs and the stagger scripts (Rift, Dojo Shifter, Staircase, Stratify, Keystone, Mt. Mograph Stagger). Report 2 §3 shows that all of them expose the same five parameters and that the standing complaints are per-layer-only items, values that do not compound and clumpy random; report 2 §1 records Quick Offset in After Effects 25.4, its Total and Per Layer overlay and Adobe's admission that staggering keys inside one layer is open; report 2 §6 ranks the capability second. Report 1 §4 gives the evaluation model (a stagger is `f(t − delay(i))`) and report 1 §5 the caveat that stagger is a style choice, not an aid to comprehension.

## 2. Scope and non-goals

In scope: the Ctrl+Alt [Cmd+Option] drag on a selection; the Stagger popover (Items, Order, Spacing, Distribution, Anchor, Jitter, Mode); Sequence as Mode = Sequence with an opacity cross-fade; whole-frame rounding and collisions; the Time Offset driver; the layer offset and its inheritance; Convert to keys; live preview; the Total / Per item readout.

Not in scope: Follow, a delayed copy of another track (spec 05); grid ordering as in GSAP; ordering by label colour (spec 09 may add an Order); moving layer in and out points as such (`VERIFY:` whether layers have them; section 12).

## 3. Rulings

- **02-R1 (an item is a layer, a run or a group):** Items picks what moves as one unit: every selected key of a layer (After Effects' unit), each run of contiguous selected keys on one track, or each group of spec 09. Default: Layers when the selection spans several layers, Runs otherwise. Why: report 2 §3 (Rift "shifts keyframes as a whole on a per layer basis"). Cost if wrong: the default flips.
- **02-R2 (moves are computed from base times and never compound):** the gesture captures every selected key's time once; each change in the popover recomputes from that base, so Per item 5 then 10 gives 10, not 15. The base is dropped when the popover closes. Why: report 2 §3 (Keystone: "you need to enter a new value, e.g. 10 if you want to double it"). Cost if wrong: one captured array.
- **02-R3 (every move is a whole frame, rounded per item):** `move_i = snapToFrameV1(ideal_i)`, ties toward later, each item on its own, so gaps may differ by one frame (10 f over five items gives 0, 3, 5, 8, 10). Why: Ruling 8; rounding the step instead would make the last item miss the dragged total by up to (n − 1) / 2 frames. Cost if wrong: one function.
- **02-R4 (three modes, one set of moves):** Interval and Overlap type the same delays two ways (interval = item length − overlap), with the conversion shown live; Sequence places items end to end from the first item instead of adding to base times. Why: report 2 §3 (Sequence Layers' Overlap is bar overlap: 10 f on 30 f layers is a 20 f stagger). Cost if wrong: one conversion line.
- **02-R5 (Time Offset is first in the driver stack, one per track):** it evaluates the keyed curve, extrapolation included, at `t − delay`; Settle, Follow and Wiggle then read the shifted curve. Why: a delay applies to the animation itself; two offsets add, so one field suffices. Cost if wrong: a reorder in a list.
- **02-R6 (a layer offset is inherited; offsets add along the parent chain):** Why: a group is delayed as one thing, which is why it was grouped, and a child's own offset then compounds, the daisy chain users build by hand (report 2 §3). Cost if wrong: an "Apply to children" toggle, default on.
- **02-R7 (collisions on one track):** when a stagger lands a key on an occupied frame of its track, the key of the later item in Order wins and an unselected occupant is replaced (Ruling 7); the overlay and popover show the count; one undo restores. Why: Ruling 7 already governs drops and pastes. Cost if wrong: nothing beyond the count.

## 4. Model and interfaces

```ts
// src/animation-core/keyframes/stagger.ts            VERIFY: folder, DocumentV1 (00 §3)
export type StaggerItemsV1 = 'layers' | 'runs' | 'groups'
export type StaggerOrderV1 =
  | { readonly kind: 'selection' }                                                   // VERIFY spec 06 keeps insertion order
  | { readonly kind: 'layer-order'; readonly direction: 'top-down' | 'bottom-up' }
  | { readonly kind: 'spatial'; readonly axis: 'left-to-right' | 'right-to-left' | 'top-to-bottom' | 'bottom-to-top' }
  | { readonly kind: 'distance'; readonly x: number; readonly y: number }            // composition px
  | { readonly kind: 'random'; readonly seed: number }
export type StaggerSpacingV1 = { readonly kind: 'per-item' | 'total' | 'overlap'; readonly frames: number }
export type StaggerDistributionV1 =
  | { readonly kind: 'linear' }
  | { readonly kind: 'in' | 'out' | 'in-out'; readonly strength: number }           // power p in [1, 5]
  | { readonly kind: 'ease'; readonly ease: EaseV1 }
export interface StaggerParamsV1 {
  readonly items: StaggerItemsV1; readonly order: StaggerOrderV1; readonly spacing: StaggerSpacingV1
  readonly distribution: StaggerDistributionV1; readonly anchor: 'first' | 'last' | 'centre'
  readonly jitter: { readonly frames: number; readonly seed: number }
  readonly mode: 'interval' | 'overlap' | 'sequence'; readonly crossFade: boolean
}
export interface StaggerItemV1 { readonly id: string; readonly keys: readonly string[]; readonly start: ExactTime; readonly end: ExactTime; readonly place: { readonly x: number; readonly y: number } | null }
export const staggerItemsV1 = (doc: DocumentV1, sel: KeySelectionV1, items: StaggerItemsV1, order: StaggerOrderV1, playhead: ExactTime): readonly StaggerItemV1[]
export const staggerMovesV1 = (p: StaggerParamsV1, ordered: readonly StaggerItemV1[], fps: number): readonly number[]
export const applyStaggerV1 = (doc: DocumentV1, ordered: readonly StaggerItemV1[], moves: readonly number[], p: StaggerParamsV1): EditResultV1 & { readonly written: number }   // spec 06; replaced keys are its `removed`
export const staggerRandomV1 = (seed: number, salt: number, index: number): number            // [0, 1)

// Time Offset (Ruling 12)
export interface TimeOffsetDriverV1 extends DriverV1 { readonly type: 'time-offset'; readonly params: { readonly delay: number } }   // whole frames, negative allowed
export const timeOffsetValueV1 = <V>(keyed: (t: ExactTime) => V, delay: number, t: ExactTime, fps: number): V   // keyed(t − delay / fps)
export const convertTimeOffsetToKeysV1 = <V>(track: TrackV1<V>, fps: number): TrackV1<V>   // exact shift; drops the driver
export const effectiveLayerOffsetV1 = (doc: DocumentV1, nodeId: string): number          // sum up the parent chain
```

`timeOffset` is a new optional integer on the node (`VERIFY:` the node type and its parent link), default 0; `place` is the item's composition place at the playhead (`VERIFY:` `compositionPlaceV1`). `staggerRandomV1` is mulberry32 seeded with `(seed ^ salt × 0x9e3779b9 ^ index) >>> 0`: five lines, public domain (`VERIFY:` share it with spec 05's Wiggle). Salt 1 orders, salt 2 jitters, so one never changes the other.

**The moves.** n ordered items; n = 1 is refused. Interval: `T` = per item × (n − 1) or the typed total; `ideal_i = D(i / (n − 1)) × T`. Overlap: `c_i = Σ_{j<i} (len_j − overlap)`, `len_j` the item's first-to-last key span in frames; `ideal_i = D(c_i / c_{n−1}) × c_{n−1}` (Interval with per item = len − overlap when lengths are equal). Sequence: `start'_0 = start_0`, `start'_i = start'_{i−1} + len_{i−1} − overlap`, `ideal_i = start'_i − start_i`; a negative overlap is a gap. `D(u)` is `u`, `u^p`, `1 − (1 − u)^p`, those two joined at u = 0.5, or `easeFractionV1(ease, u)`, unclamped: an overshooting ease places a middle item after the last. Jitter: `m_i = snapToFrameV1(ideal_i) + snapToFrameV1((2 × staggerRandomV1(seed, 2, i) − 1) × J)`. Anchor subtracts `a = m_0` (first), `m_{n−1}` (last) or `snapToFrameV1((m_0 + m_{n−1}) / 2)` (centre) from every `m_i`. Every key of item i moves by exactly `move_i / fps`; a sub-frame base key keeps its fraction. Random order is a Fisher-Yates shuffle on `staggerRandomV1(seed, 1, k)`; ties break by layer order. Cross-fade (Sequence, Layers, overlap > 0) writes Opacity keys with the linear ease (`VERIFY:` the opacity property id and range): item i > 0 fades 0 to full over `[start'_i, start'_i + overlap]`, item i < n − 1 fades full to 0 over `[end'_i − overlap, end'_i]`; occupied frames follow Ruling 7.

## 5. UX

**Gesture** (00 §4, Ctrl+Alt [Cmd+Option] + drag a selection). With two or more items selected, press any selected key and drag. The pointer's travel, snapped to whole frames by spec 07, is the Total: the first item in Order stays, the last moves by the Total, the rest spread linearly; dragging left gives a negative Total. Adding Shift takes its meaning from the map (snap to keys, markers and the playhead) and applies it to the item under the pointer, any item. Ctrl is already held, so sub-frame placement is unavailable during a stagger drag (named limit).

**Overlay while dragging.** Near the pointer: `Total 20 f · Per item 5 f`, or `Per item 2.5 f, rounded` when not whole, plus `2 keys replaced` in the warning colour when 02-R7 applies. The canvas renders the playhead's frame with the preview timing (`VERIFY:` 00 §8.4).

**Stagger popover.** Opens on release at the last item's first key, prefilled from the drag: Items per 02-R1, Order = Selection, Spacing = Total (the travel), the rest at defaults. Rows:
- Items: Layers | Runs | Groups.
- Order: Selection | Layer order, top-down or bottom-up | Left to right and the other three directions | Distance from point… | Random (Seed, Reseed).
- Spacing: Per item | Total | Overlap, one frames field (00 §4 number fields); Overlap shows `= Interval 20 f` live; hovering lists the rounded frames.
- Distribution: Linear (default) | Ease in | Ease out | Ease in-out, Strength 1 to 5, default 2 (Dojo Shifter's quad to quint); Library…, the spec 01 picker.
- Anchor: First stays (default) | Last stays | Centre stays.
- Jitter: ± frames 0 to 100, default 0 (above any stagger in report 2's tutorials; larger typed values accepted); Seed; Reseed.
- Mode: Interval (default) | Overlap | Sequence. Cross-fade: only in Sequence with Layers, otherwise `Cross-fade needs layers`.

Each change previews live and commits as one undo step on release or Enter; Escape or a click outside closes and keeps the edits (00 §4). Undo restores keys and fields together (`VERIFY:` 00 §8.8).

**Readout.** Read-only `Total 12 f · Per item 3 f` in the timeline header for two or more items; `(uneven)` when gaps differ.

**Time Offset.** The inspector shows an Offset field (frames) on every track and layer. A track with a non-zero effective offset carries a badge, `+6 f` or `+4 f (+6 f from Group)`; the timeline draws its keys at their stored times plus dimmed ghosts at the shifted times, hover `Offset +6 f. Convert to keys to edit them.` Track and layer context menus have `Convert offset to keys`. The field edits the same driver as spec 05's Time Offset behaviour entry.

**Keyboard.** Ctrl+Shift+O [Cmd+Shift+O] opens the popover for the selection with Spacing = Per item 0 f and focus in the field (proposed, section 12); `Animate > Stagger…` and `Animate > Sequence…` (`VERIFY:` menu names) do the same.

**Empty and error states.** One item: `Select two or more layers or runs to stagger`, nothing moves. Groups with none selected: `No groups in the selection`, Items falls back to Layers.

## 6. User flows

**Flow 1, primary.** Start: five text layers L1 to L5, each with Position keys at 0 and 12 f, all ten keys selected in that order; playhead at 8 f. (1) Hold Ctrl+Alt, press a key of L5, drag right 20 f: L1 unmoved, L5 at +20 f, L2 to L4 at +5, +10, +15 f; overlay `Total 20 f · Per item 5 f`; the canvas at frame 8 updates. (2) Release: one undo step; the popover opens, prefilled. (3) Type `3` into Per item: moves 0, 3, 6, 9, 12 from the base (not 5 + 3); readout `Total 12 f · Per item 3 f`. (4) Ease out, Strength 2: moves 0, 5, 9, 11, 12; readout adds `(uneven)`. (5) Escape. End: keys at base + 0, 5, 9, 11, 12 f; selection unchanged; three undo steps.

**Flow 2, keyboard only.** Start: as flow 1 before step 1. (1) Ctrl+Shift+O: the popover opens, focus in Per item. (2) Type `4f`, Enter: moves 0, 4, 8, 12, 16 f. (3) Tab to Order, Down twice to `Layer order, bottom-up`, Enter: the bottom layer stays, the top moves 16 f. (4) Tab to Jitter, type `2`, Tab, `7` into Seed, Enter: items move by up to ±2 f, repeatable with seed 7. (5) Escape. End: four undo steps.

**Flow 3, multi-selection within one track.** Start: one layer whose Opacity keys at 10, 14, 18 f and 30, 34, 38 f are selected (two runs). (1) Ctrl+Shift+O: Items = Runs, two items; readout `Total 20 f · Per item 20 f`. (2) `3` into Per item: the second run moves to 33, 37, 41 f, its 4 f spacing kept; the first stays. (3) Anchor = Last stays: the second run returns to 30, 34, 38 f, the first moves to 7, 11, 15 f. End: two undo steps.

**Flow 4, undo.** Start: the end of flow 1, popover open. (1) Ctrl+Z: moves back to 0, 3, 6, 9, 12 f and Distribution reads Linear; selection and popover stay (Ruling 11). (2) Ctrl+Z: 0, 5, 10, 15, 20 f. (3) Ctrl+Z: base times. (4) Ctrl+Shift+Z: the drag result returns. End: the popover still open.

**Flow 5, limit (collisions).** Start: one Opacity track with selected keys at 0 and 2 f (run A) and 10 and 12 f (run B), an unselected key at 14 f. (1) Ctrl+Alt drag run B left 10 f: overlay `Total −10 f · Per item −10 f · 2 keys replaced`; B lands on 0 and 2 f and wins (02-R7). (2) Release: keys at 0, 2 (B's values) and 14 f; the popover opens. (3) `−8` into Total: B lands on 2 and 4 f, A's key at 2 f is replaced, `1 key replaced`. (4) Ctrl+Z twice. End: all five keys as at the start.

**Flow 6, Time Offset and layer offset.** Start: group G with children C1, C2, each with a Position track keyed 0 to 24 f; playhead at 10 f. (1) Select G, type `6` into Offset: both children start 6 f later; the canvas shows the pose of frame 4; their rows read `+6 f (from G)`. (2) Select C2, type `4` into Offset: badge `+4 f (+6 f from G)`. (3) Right-click C2's Position track, `Convert offset to keys`: its keys sit at 4 to 28 f, its Offset reads 0, the badge reads `+6 f (from G)`, every frame renders as before. End: three undo steps.

## 7. Evaluation and determinism

Stagger and Sequence are document edits; the evaluator is untouched. The driver: for a track with keys `K`, extrapolation `before`/`after` (spec 08) and a Time Offset of `delay` frames on a node whose ancestors' offsets sum to `L`, `value(t) = rest(keyedV1(K, t − (delay + L) / fps), t)`, where `keyedV1` is spec 05's `baseValueAt` (keys, eases, extrapolation) and `rest` the rest of the stack at `t`. `L` is summed at evaluation time; nothing is written into child tracks. Before the first key the `before` rule shows for `delay` frames longer; a negative delay makes the track lead. The subtraction is exact rational arithmetic.

Ruling 9 statement: every value is `evaluate(document, t)`; the driver reads its own keyed curve at one other exact time and carries no state, so play, seek and export agree bit for bit; reading only its own keys, it cannot form a cycle. Nothing bakes for playback. Lottie export, which has no time offset (report 1 §1), converts each offset by the exact shift through `bakeTrackV1` (spec 05, exact mode; `VERIFY:` the exporter hook).

## 8. Edge cases and named limits

- A move may place keys before frame 0 (`VERIFY:` negative key times; if refused, it stops at frame 0 and says so).
- Jitter is uniform, so clumps occur (report 2 §3). Named limit.
- `Convert offset to keys` on a layer shifts every track of the layer and of every descendant by the layer's own offset, then sets it to 0; descendants keep their own offsets, so nothing changes on screen.

## 9. Interactions with other specs

01: Distribution may use any library ease; a stagger never changes an ease, because handles are normalised (Ruling 1). 05: Follow is the two-track form of Time Offset; 05 folds it as a behaviour entry, and first in the stack the two readings agree; `bakeTrackV1` (exact mode, 05-R8) does the export conversion. 06: selection order, runs and Absolute / Offset entry are defined there. 07: the drag controller supplies the snapped travel and Shift snapping. 08: extrapolation applies at the shifted time, so a looping track loops later by the delay. 09: `groupsOfSelectionV1` supplies group items; the same-frame highlight shows a collision before release. Motion smear (effects plan B9) reads places through the same evaluator, so offset tracks smear correctly.

## 10. Rules and tests

Each rule is one test in `stagger.test.ts`; numbers are exact.

- R1 `per item 3 f over five items gives 0, 3, 6, 9, 12`.
- R2 `total 10 f over five items rounds per item, ties later: 0, 3, 5, 8, 10` (ideal 0, 2.5, 5, 7.5, 10).
- R3 `ease out, strength 2, total 12 f front-loads: 0, 5, 9, 11, 12`, gaps 5, 4, 2, 1 strictly decreasing; `ease in mirrors it: 0, 1, 3, 7, 12`.
- R4 `random order with seed 7 is reproducible`: two calls on five items return the same permutation, pinned; seed 8's differs.
- R5 `a run keeps its internal spacing`: runs at 10, 14, 18 and 30, 34, 38, per item 3 → 33, 37, 41.
- R6 `overlap converts to interval`: three 30 f items, overlap 10 → moves 0, 20, 40, label `Interval 20 f`.
- R7 `sequence lays items end to end`: lengths 30, 20, 10, overlap 0 → starts 0, 30, 50; overlap −5 → 0, 35, 60.
- R8 `cross-fade writes 8 opacity keys for three 30 f layers at overlap 10`: A full at 20, 0 at 30; B 0 at 20, full at 30, full at 40, 0 at 50; C 0 at 40, full at 50.
- R9 `anchor last and centre`: 0, 3, 6, 9, 12 → −12, −9, −6, −3, 0 and −6, −3, 0, 3, 6.
- R10 `jitter is seeded and salted`: ±2 f, seed 7, pinned; changing Jitter keeps the random order; Anchor first leaves item 0 at 0.
- R11 `moves never compound`: per item 5 then 10 from one base → 0, 10, 20, 30, 40.
- R12 `collision: the later item wins`: flow 5 → `removed` has 2; undo restores all keys.
- R13 `Time Offset 4 f at frame 10 equals the unoffset track at frame 6`, `Object.is` per channel; `at frame 2 it equals the before rule at frame −2`.
- R14 `convert to keys is exact`: frames 0..60 `Object.is` before and after; the driver is gone.
- R15 `layer offset inherits and adds`: child effective 6 + 4 = 10; Convert on the layer shifts descendants by 6, zeroes the layer, keeps the child's 4, equal at every frame.
- R16 `readout`: total 12, per item 3, even true; after R3's ease, even false.
- D1 `play equals seek`, D2 `random order`, D3 `after an edit` on a fixture with a layer offset of 6 f, a track offset of −3 f and a Wiggle below it; D4 `bake round-trip`: the export conversion equals the source `Object.is` at every frame and undo restores the driver with `delay` intact.

## 11. VERIFY list

1. Folder, `DocumentV1`, `ExactTime` helpers, `document.settings.frameRate`.
2. `KeySelectionV1` keeps insertion order; the run definition (spec 06).
3. The node type and parent link for `timeOffset`; `compositionPlaceV1` (effects plan B9).
4. The opacity property id and range; negative key times.
5. The undo stack can carry the popover's parameters (00 §8.8); the draft render path (00 §8.4).
6. The keymap for Ctrl+Shift+O; menu names.
7. The existing wiggle driver's place in the stack (00 §8.2), so Time Offset sits first.
8. The Lottie exporter hook.
9. Whether layers have in and out points.

## 12. Open questions for the user

1. Proposed addition to 00 §4: `Ctrl+Shift+O [Cmd+Shift+O]: open the Stagger popover for the selected keys (spec 02)`; and a note on the stagger row that Shift snaps the item under the pointer.
2. If layers have in and out points, should Items = Layers move the bar with the keys, or is that a checkbox?
3. Is After Effects' "Dissolve front layer" variant of the cross-fade wanted?
4. May a move place keys before frame 0, or should it stop there?
