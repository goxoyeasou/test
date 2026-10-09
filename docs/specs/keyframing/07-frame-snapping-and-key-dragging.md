# Spec 07: frame snapping and key dragging

Written without the repository open; every assumption about unseen code is marked `VERIFY:` and gathered in section 11. Depends on 00 and 06. Specs 02, 08 and 10 drag through the model defined here.

## 1. Purpose and evidence

One drag model for keys: a pointer-down on a key or an edge handle, a ghost that follows the pointer, a readout of the frame and the delta, and a commit on release that lands on whole frames unless the user asks otherwise; with a Snap to Frames command that repairs sub-frame keys without touching eases. Evidence: report 2 §3 and §5 (Alt-drag "almost always leads to subframe positioning", the standing quantise request, a user moving "around 50 keyframes manually" after a frame-rate change because the only script "does not preserve keyframe easing"), report 2 §6 rank 7, and report 1 §2 (Blender's Absolute Time Snap and Maya 2025.1's Time Snap exist for this reason; Motion Canvas ripples later events and Shift isolates). Details: research notes `timing_offset_stagger_and_retime_tools.md` and `keyframe_editor_ux_in_shipping_tools.md` (KQ5, KQ6).

## 2. Scope and non-goals

In scope: hit targets and the drag threshold; dragging one key or a selection across tracks and layers; the ghost and readout; frame snapping, Shift magnets, Ctrl free placement, the "Allow sub-frame keys" setting and its glyph; nudges; the edge handles (scale about the far edge, ripple, Shift isolate, minimum-span stop); Snap Keys to Frames, selected and all, and its automatic run after a frame-rate change; pass-through and replace on drop; Escape; J/K landing on sub-frame keys.

Non-goals: Alt-drag duplicate (spec 10, same model), Ctrl+Alt stagger (02), the numeric Retime dialog and retime markers (08), value drags in the graph (04), dragging layers or markers (existing, `VERIFY:`), touch and pen gestures.

## 3. Rulings

- **07-R1 (preview, then commit):** every drag computes a `DragPreviewV1` on each pointer move from the exact times recorded at pointer-down and commits once on release; nothing is written before release; Escape discards the preview. Why: restoring exact times is then trivial, and the undo stack gets one step per drag (Ruling 11). Cost if wrong: nothing; a live-commit variant needs the same preview.
- **07-R2 (the anchor snaps, the rest follow by delta):** the key under the pointer is the anchor; the delta is `snap(anchorTime + pointerDelta) − anchorTime`; every dragged key moves by exactly that delta. Why: relative times across tracks and layers are kept exactly, and keys on whole frames stay there because the delta is a whole number of frames. Cost if wrong: a selection that drifts apart as it is dragged.
- **07-R3 (frames always, magnets on Shift at 6 px, sub-frames on a hundredth):** whole-frame snapping is on except under Ctrl [Cmd] or the document setting, when placement lands on `n / (100 · fps)`; Shift adds magnets to other keys, markers, guides and the playhead within 6 px of the pointer, nearest wins. Why: the frame counter shows two decimals, so a finer grid would display rounded; 6 px sits above the 3 px drag threshold and below the 8 px hit target, so a magnet pull is never mistaken for drag slop and never reaches into the next key's target. Cost if wrong: three constants.
- **07-R4 (a nudge joins the grid):** Alt+Left/Right moves to the next whole frame in the pressed direction (from 7.4, right lands at 8 and left at 7); Alt+Shift moves to that frame and nine more. Why: a nudge is a walk on the frame grid and its first step joins it, so a sub-frame key is repaired by a gesture the user already makes. Cost if wrong: one function.
- **07-R5 (edge handles scale about the far edge, snap per key, stop at the minimum span):** `t' = far + (t − far) · k`, each result snapped; later keys on the same tracks move by the last selected key's displacement unless Shift is held; the handle stops at the smallest whole-frame span at which all scaled keys stay on distinct frames. Why: Blender and Maya snap after scaling (report 1 §2); the stop applies Ruling 7 before the fact instead of merging keys the user did not mean to lose. Cost if wrong: one search function.
- **07-R6 (Snap to Frames keeps eases and reports merges):** each key moves to its nearest whole frame, ties toward later; eases are untouched because normalised handles (Ruling 1) are invariant under time scaling; when several keys land on one frame the latest original wins (Ruling 7) and the report names the track and count. Why: the forum case in report 2 §5. Cost if wrong: T9 fails.
- **07-R7 (no vertical drag in v1):** a key never moves to another track by dragging; vertical pointer movement is ignored. Named limit. Why: values do not transfer between properties, and the same-property case is what spec 10's paste does. Cost if wrong: one gesture added later.
- **07-R8 (pass-through, replace on drop):** a dragged key may pass unselected keys on its track (the track re-sorts; each key keeps its own outgoing ease); a drop on an occupied frame replaces the occupant (Ruling 7) and the ghost turns red before release. Why: a drag that stopped at the neighbour would make reordering a two-step job. Cost if wrong: nothing; one undo step restores the replaced key.

## 4. Model and interfaces

```ts
// src/animation-core/keyframes/key-drag.ts
export const DRAG_THRESHOLD_PX_V1 = 3
export const HIT_TARGET_PX_V1 = 8
export const MAGNET_PX_V1 = 6
export const SUB_FRAME_DIVISIONS_V1 = 100

export interface ModifiersV1 { readonly shift: boolean; readonly ctrl: boolean; readonly alt: boolean }   // Cmd reads as ctrl on Mac
export interface TimeViewV1 { readonly pxPerFrame: number; readonly originPx: number; readonly fps: number }  // VERIFY the timeline's scale object

export interface SnapTargetV1 { readonly kind: 'frame' | 'key' | 'marker' | 'guide' | 'playhead'; readonly time: ExactTime; readonly name?: string }
export const snapTargetsV1 = (doc: DocumentV1, excluding: ReadonlySet<KeyIdV1>): readonly SnapTargetV1[]

export interface DragStartV1 {
  readonly kind: 'move' | 'edgeFirst' | 'edgeLast'
  readonly origin: ReadonlyMap<KeyIdV1, ExactTime>   // exact times at pointer-down
  readonly anchor: KeyIdV1                            // the key under the pointer, or the handle's own edge
  readonly pointerDownPx: number
}
export interface DragPreviewV1 {
  readonly times: ReadonlyMap<KeyIdV1, ExactTime>
  readonly deltaFrames: number                        // of the anchor; fractional under Ctrl
  readonly snappedTo: SnapTargetV1 | null
  readonly collisions: ReadonlySet<KeyIdV1>           // occupied drop frames; ghost red
  readonly ripple: ReadonlyMap<KeyIdV1, ExactTime>    // edge drags only
  readonly stopped: boolean                           // edge drag at its minimum span
}
export const proposeKeyDragV1 = (doc: DocumentV1, start: DragStartV1, pointerPx: number, view: TimeViewV1, mods: ModifiersV1, allowSubFrame: boolean): DragPreviewV1
export const commitKeyDragV1 = (doc: DocumentV1, preview: DragPreviewV1): EditResultV1   // spec 06

export const nudgeTimeV1 = (time: ExactTime, fps: number, frames: -10 | -1 | 1 | 10): ExactTime
export const scaleTimesV1 = (times: ReadonlyMap<KeyIdV1, ExactTime>, far: ExactTime, k: number, fps: number, snap: boolean): ReadonlyMap<KeyIdV1, ExactTime>
export const minSpanFramesV1 = (times: ReadonlyMap<KeyIdV1, ExactTime>, far: ExactTime, fps: number): number

export interface SnapReportV1 { readonly moved: number; readonly merged: readonly { readonly trackId: string; readonly count: number }[] }
export const snapKeysToFramesV1 = (doc: DocumentV1, scope: KeySelectionV1 | 'all'): EditResultV1 & { report: SnapReportV1 }
export const changeFrameRateV1 = (doc: DocumentV1, fps: number): EditResultV1 & { report: SnapReportV1 }   // VERIFY the existing rate-change command
// DocumentSettingsV1.allowSubFrameKeys: boolean, default false; VERIFY the settings object
```

`snapToFrameV1` and `frameIndexV1` are 00 §3's. `proposeKeyDragV1` is pure: the same inputs give the same preview, which is what the tests drive. The timeline calls it on every pointer move and draws `times` as the ghost; collisions are found through spec 06's `trackTimeIndexV1` in constant time per key.

## 5. UX

**Hit targets.** A key's target is at least 8 px wide and the row's full height, centred on the glyph (`VERIFY:` the glyph is 7 px). Where targets overlap at low zoom the nearest centre wins, ties to the later key. Edge handles are brackets 4 px outside the first and last selected keys, each with an 8 px target; under 16 px of span (two targets) they are hidden and the selection's hover text says "Zoom in to retime".

**Starting a drag.** Pointer-down on a selected key moves the whole selection; on an unselected key, that key becomes the selection and moves alone. The drag begins after 3 px of movement (the common desktop drag slop; Windows' default is 4 px). Modifiers are read on every move, so pressing Shift mid-drag starts snapping; Alt is read at pointer-down because it changes what the drag is (spec 10).

**The ghost and readout.** The dragged keys draw at their proposed times at 50 % opacity in the selection colour, originals dimmed. A readout follows the pointer: "Frame 12 (+5 f)"; with a magnet, "Frame 12 (+5 f) ▸ marker Hit"; sub-frame, "Frame 7.40 (+0.40 f)"; a collision turns the ghost red and the readout says "Replaces key at frame 12". Edge drags: "Span 20 f → 10 f (50 %)", a second line "6 later keys move" when rippling, "Minimum span 2 f" when stopped.

**Snapping.** Default: whole frames (Ruling 8). Shift: also keys on every track, markers, guides (spec 09) and the playhead within 6 px; the readout names the target. Ctrl [Cmd]: free placement on the hundredth-frame grid for this drag. Document setting "Allow sub-frame keys" (Composition settings, default off): free placement becomes the default and Ctrl does nothing; sub-frame keys draw as a half-filled diamond, colour-independent, with hover text "Between frames: 7.40 f. Snap with Ctrl+Shift+F".

**Nudging.** Alt+Left/Right one frame, Alt+Shift ten (00 §4), 07-R4. Collisions follow Ruling 7 with the status "1 key replaced on Opacity".

**Edge handles.** With two or more keys selected, brackets at the earliest and latest selected time across all selected tracks. Dragging one scales about the other. Later keys on the same tracks ripple by default; Shift isolates (00 §4; for this gesture Shift does not add magnets). The handle stops at the minimum span (07-R5).

**Snap to Frames.** Keys ▸ Snap Keys to Frames (Ctrl+Shift+F, 00 §4: the selection; the whole composition when nothing is selected) and Keys ▸ Snap All Keys to Frames. The status bar reports; when keys merged a notice says "12 keys moved to frames. 3 keys merged on Opacity. Undo to restore." After a composition frame-rate change the command runs on all keys in the same undo step and the notice shows; with "Allow sub-frame keys" on, the change does not snap and the notice offers a Snap All button instead.

**J/K, Escape and errors.** J/K land the playhead on a sub-frame key's exact time; the frame counter shows "7.40", its hover text the rational "37/5 f", and the time field accepts `7.4f`. Escape discards the preview; keys and selection are as before pointer-down. Ctrl+Shift+F with every key already on a frame: "All keys are on frames." A drag that would put a key before frame 0 clamps at 0 (section 8).

## 6. User flows

**Flow 1 (primary): move a key 7.4 frames.** Start: 30 fps, an Opacity key at frame 10 selected, 10 px per frame. (1) Pointer-down on the key and move 74 px right: at 3 px the ghost appears; at 74 px the readout says "Frame 17 (+7 f)". (2) Release: the key is at 17/30 s exactly; one undo step, "Move 1 key". End: value and ease unchanged.

**Flow 2 (keyboard only): repair a sub-frame key with nudges.** Start: "Allow sub-frame keys" was on and a key sits at frame 7.4 (37/5 f); the setting is now off; nothing selected. (1) K: the playhead lands on the key; the counter shows "7.40". (2) Select ▸ At playhead (spec 06, by its menu accelerator; `VERIFY:`): the key selects. (3) Alt+Right: the key lands on frame 8; status "Moved 1 key to frame 8". (4) Alt+Shift+Right: frame 18. (5) Ctrl+Shift+F: "All keys are on frames." End: the key at 18/30 s.

**Flow 3 (multi-selection): retime a hit across three tracks with ripple.** Start: Position, Scale and Opacity keys on one layer at frames 0, 10, 20, plus later keys at 30 and 40 on each track; the nine keys 0..20 selected; brackets at 0 and 20; 10 px per frame. (1) Drag the right bracket 100 px left: readout "Span 20 f → 10 f (50 %)", "6 later keys move"; the ghost shows keys at 0, 5, 10 and the later keys at 20, 30. (2) Hold Shift: the later keys return to 30 and 40 in the ghost; the second line disappears. (3) Release Shift, then the pointer: keys at 0, 5, 10, 20, 30 on all three tracks; one undo step, "Retime 9 keys". End: every ease unchanged.

**Flow 4 (undo): a frame-rate change.** Start: a 30 fps composition with a key at 37/30 s (frame 37) and eases on both sides. (1) Composition settings: 24 fps, OK. The key would sit at frame 29.6; Snap All runs: it moves to frame 30, 5/4 s; notice "1 key moved to frames." (2) Ctrl+Z: the rate returns to 30 fps and the key to 37/30 s in one step; eases identical. End: as before step 1.

**Flow 5 (limit): scaling down stops at the minimum span.** Start: keys at frames 0, 1, 2 on one track, all selected, 10 px per frame. (1) Drag the right bracket 15 px left: the ghost keeps 0, 1, 2; readout "Span 2 f → 2 f (100 %)", "Minimum span 2 f"; the bracket draws at its stop. (2) Drag further left: nothing changes. (3) Drag 20 px right of the start: keys at 0, 2, 4; readout "Span 2 f → 4 f (200 %)". (4) Release: 0, 2, 4. End: three keys, one undo step.

## 7. Evaluation and determinism

Dragging, nudging, scaling and snapping are pure functions `(document, …) → document` on key times; nothing evaluates differently in kind (Ruling 9). A sub-frame key is an ordinary key with a rational time; `evaluateTrackV1` solves the segment on either side of it at any time, frame or not, so play, seek and export agree. No bake.

## 8. Edge cases and named limits

- A drag that would place the earliest dragged key before frame 0 clamps the delta so it lands at 0; keys past the composition's end are allowed (`VERIFY:` negative key times in old documents).
- Sub-frame keys in a dragged selection keep their fraction (07-R2 keeps relative times); Snap Keys to Frames repairs them on demand. Named limit against Ruling 8's letter, in favour of its intent.
- Ripple moves keys on the same tracks only; other tracks, markers and the work area stay. Named limit; spec 08's ripple retime covers the layer. When the far edge is the last key, nothing ripples.
- Edge drag with keys on several tracks: the minimum span is the smallest span valid on every track. When the proposed span collides, the preview uses the nearest valid span in the drag's direction and sets `stopped`.
- Snap All with several keys landing on one frame: the latest original wins; the removed ids leave the selection and count as missing in Key Sets (spec 06). 

## 9. Interactions with other specs

- 06: the selection and `EditResultV1`; brackets need two or more selected keys; Snap's merged keys update Key Sets; `trackTimeIndexV1` answers collision checks.
- 08: owns the numeric Retime dialog and retime markers; must match `scaleTimesV1` for a uniform scale and use `minSpanFramesV1` for its stop; owns ripple across a whole layer.
- 09: guides are magnets; labels stay with their key through drags; the summary row's keys drag like any key.
- 10: Alt-drag duplicate uses the same preview and commits copies; J/K landing on sub-frame keys as in section 5.
- 04: the graph's horizontal drags call `proposeKeyDragV1` with its own `TimeViewV1`.

## 10. Rules and tests

Tests in `key-drag.test.ts` and `snap-keys.test.ts`; 30 fps unless stated; `view = { pxPerFrame: 10, originPx: 0, fps: 30 }`.

- R1 whole frames by default. T1 `a drag of 7.4 frames lands at 7`: anchor at frame 10, pointer +74 px: `times` holds 17/30 s exactly (`VERIFY:` reduced form), `deltaFrames` 7, `snappedTo.kind` `'frame'`.
- R2 Shift magnets within 6 px. T2 `Shift-drag within 6 px of a marker lands on the marker`: `pxPerFrame` 4, key at 10, marker "Hit" at 17, pointer +22 px: without Shift 16; with Shift 17 and `snappedTo.kind` `'marker'`; a marker at 18 (10 px away) gives 16.
- R3 Ctrl and the setting free the drag. T3 `Ctrl-drag keeps 7.4`: as T1 with ctrl: 17.4 frames, 29/50 s; with `allowSubFrame` true and no modifier: the same.
- R4 nudges join the grid. T4 `nudging from 7.4 lands at 8`: `nudgeTimeV1(37/5 f, +1)` is 8; `−1` is 7; `+10` is 17; from 8, `+1` is 9.
- R5 relative times are kept across tracks. T5: keys at 10 (A, Opacity), 12 (B, Scale) and 12.4 (B, Position); drag the anchor 10 by +7.4: 17, 19, 19.4; frame differences unchanged.
- R6 edge scaling snaps per key. T6 `scaling 0..20 to 0..10 keeps keys at 0, 5, 10`: `scaleTimesV1` with far 0, k 0.5: 0, 5, 10; k 0.37: 0, 4 (3.7), 7 (7.4).
- R7 the minimum span. T7 `scaling stops at span 2 f for keys at 0, 1, 2`: `minSpanFramesV1` is 2; the right handle dragged to span 1 gives `stopped` true and times 0, 1, 2; keys at 0, 10, 20 also give 2; keys at 0, 1, 20 give 10 (the first span at which 1 rounds clear of 0).
- R8 ripple by the last key's displacement unless Shift. T8: keys 0, 10, 20, 30, 40; select 0..20; scale to 0..10: `ripple` has 30 → 20 and 40 → 30; with shift it is empty.
- R9 Snap keeps eases and reports. T9 `Snap All after a 30→24 fps change moves a key at 37/30 s to the nearest 24 fps frame and reports it`: `changeFrameRateV1(doc, 24)`: the key is at 5/4 s; `report.moved` 1, `merged` empty; every key's `ease` is the same object as before (`toBe`); undo restores 37/30 s with `Object.is` on `n` and `d`.
- R10 merges keep the later key, ties go later. T10: keys at 7.1 f and 7.4 f on Opacity: one key at 7 with the 7.4 key's value; `merged` is `[{ trackId, count: 1 }]`; the status reads "1 key merged on Opacity"; a key at 7.5 f snaps to 8.
- R11 pass-through and replace. T11: keys at 10 and 20; drag 10 by +15: `collisions` empty, commit gives 20, 25 with each key's ease kept; drag 10 by +10: `collisions` holds the key at 20, commit gives one key at 20 with the dragged value, undo gives both.
- R12 Escape restores exact times. T12 `Escape restores the exact times`: start a drag, propose at +74 px, discard: the document is the same reference, every time `Object.is` on `n` and `d`, the selection unchanged.
- R13 threshold, targets and budget. T13: a pointer move of 2 px yields no preview and 3 px yields one; two keys 5 px apart: the hit test picks the nearer, ties later; a 5,000-key selection: `proposeKeyDragV1` median under 16 ms over 20 runs (one frame at 60 Hz; `VERIFY:` the runner).
- D1 `play equals seek` on a track with a sub-frame key at 37/5 f after a Ctrl-drag: frames 0..N in order, then N alone from a fresh evaluator; `Object.is` per channel.
- D2 `random order`: a shuffled frame list equals D1.
- D3 `after an edit`: drag, undo, evaluate: equal to before the drag.
- D4 `bake round-trip`: no bake here; on spec 05's baked fixture (whole-frame keys) `snapKeysToFramesV1('all')` returns a document deep-equal to its input with `moved` 0, and unbake still restores the parameters exactly.

## 11. VERIFY list

1. `ExactTime` helpers: construction from `n / (divisions · fps)`, reduction, comparison (00 §8.1).
2. The timeline's scale object and pointer handling: px per frame, CSS versus device px, pointer capture.
3. The platform modifier mapping (Cmd as ctrl) and keymap collisions for Alt+arrows and Ctrl+Shift+F (00 §8.3).
4. The key glyph's size; the existing marker, guide and playhead objects for `snapTargetsV1`.
5. The composition settings object and the existing frame-rate change command; whether old documents hold negative key times.
6. The undo stack's grouping: a rate change plus its snap must be one step (00 §8.8).
7. Menu accelerators for keyboard-only use of Select ▸ At playhead (flow 2).

## 12. Open questions for the user

1. 07-R7: no vertical drag in v1. Confirm, or ask for same-property drags between layers, which spec 10's paste covers.
2. Keys before frame 0 clamp (section 8). Confirm.
3. With "Allow sub-frame keys" on, a frame-rate change does not snap (section 5). Confirm.
4. 00 §4 lists Ctrl+Shift+F as "snap all selected keys"; this spec reads it as the selection, or every key when nothing is selected. Confirm the reading.
5. Proposed addition to 00 §4: none; every gesture here is taken from the map.
