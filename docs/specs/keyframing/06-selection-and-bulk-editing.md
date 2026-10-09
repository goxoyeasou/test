# Spec 06: selection and bulk editing

Written without the repository open; every assumption about unseen code is marked `VERIFY:` and gathered in section 11. Depends on 00 only. Specs 02, 04, 07, 09 and 10 build on the selection defined here.

## 1. Purpose and evidence

One selection of keys and segments, shared by the timeline, the graph, the inspector and the canvas, built by scope (track, layer, composition, work area, playhead, rule) and never by what happens to be scrolled into view; an inspector that says whether a typed or scrubbed number is Absolute, an Offset or a Proportional scale; a Reverse that honours the keys the user picked; all of it at 5,000 selected keys without slowing down. Evidence: report 2 §5 (visible-only Ctrl+Alt+A, typed versus dragged edits split invisibly between absolute and relative, Time-Reverse ignoring sparse selections and closed "won't fix", zl_Scriptlets' selection rules and Key Sets, KeyTweak), report 2 §1 (Proportional Scrubbing in 26.2 beta, excluded from the Properties panel; the August 2026 slowdown from about 100 marquee-selected keys), report 2 §6 rank 6, and report 1 §2 (selection lost on a view switch in AE, undo reverting pan and zoom in Rive). Details: research note `bulk_keyframe_editing_copy_paste_and_navigation.md`.

## 2. Scope and non-goals

In scope: the selection model and store; the Select menu; Key Sets; name-click and marquee rules; the inspector's multi-key value field (common value or Mixed, the Absolute / Offset toggle, `+ - * /` expressions, Proportional scrubbing); editing across tracks of one value kind; selection-faithful Reverse; the data structures and the 16 ms budget.

Non-goals: moving keys in time (spec 07), copy and paste (10), ease editing (01, 03), the graph's transform box (04), labels and groups beyond the "By label" filter (09), KeyTweak-style propagation to unselected keys (a later spec), selection of layers and nodes (existing behaviour, `VERIFY:`).

## 3. Rulings

- **06-R1 (selection by scope, never by view):** every Select command, name click and Key Set recall selects by document scope; collapsed, scrolled-off, shy and hidden tracks are included and the status bar reports the count. Only the marquee is a box on the screen, and it reaches a collapsed layer through its summary row (spec 09). Why: report 2 §5; AE's visible-only Ctrl+Alt+A silently excludes collapsed properties from bulk moves. Cost if wrong: nothing; the view-based variant is a subset the marquee provides.
- **06-R2 (one selection, keyed by time):** `KeySelectionV1` (00 §3) is the only selection; ids are `${trackId}@${serializedTime}`. Every edit that moves or removes keys returns the id moves it caused and the selection store applies them, so the selection follows its keys through edits, undo and redo without being in the undo stack (Ruling 11). Why: report 1 §2 (AE drops the selection when the graph opens; Rive undoes pan and zoom with a deletion). Cost if wrong: stale ids after a drag; the remap is one function.
- **06-R3 (bulk arithmetic in grid units):** Absolute, Offset and scale edits compute in integer units of the property's precision (`VERIFY:`; assumed hundredths) and write `units / 10^precision` back. Why: double addition does not preserve differences bit for bit; in integer units the difference between two selected values is unchanged by construction. Cost if wrong: an off-grid value from a bake or import moves by at most half a unit on its first bulk edit (section 8).
- **06-R4 (Proportional scales about zero):** Proportional scrubbing multiplies every selected value by one factor `k` about zero, not about the selection's minimum. Why: ratios are what proportional means; a keyed zero stays zero, so a fade in stays a fade in; the factor is one number the readout shows and the user can type as `*1.25`, so scrub and expression share one code path. Cost if wrong: one pivot parameter.
- **06-R5 (Reverse is an order reversal, not a time reversal):** selected keys keep their times; per track the selected values reverse in order, and the selected segments' eases reverse in order and are mirrored by spec 03's `mirrorEaseV1`; unselected keys and half-selected segments are untouched. Why: report 2 §5 (the "won't fix" bug); keeping times lets a sparse selection reverse without touching anything between. Cost if wrong: nothing; it is a pure function of the selection.
- **06-R6 (one field per value kind):** keys spanning several tracks get one inspector field per value kind and unit; Absolute is allowed only when the tracks are one property or share a range, otherwise Offset is forced on. Why: AE's "same layer property" rule is a data-model limit, not a UX choice (research note, KQ1 inferences). Cost if wrong: per-track rows added under the field.
- **06-R7 (Key Sets live in the document and go stale, never shrink):** a deleted member counts as missing until the user prunes. Why: zl_Scriptlets' Key Sets exist because selections do not survive timing changes; dropping members silently would lose exactly the keys the user wanted back. Cost if wrong: one prune command.
- **06-R8 (cost follows visible keys, not selected keys):** the selection is a `Set` of ids; the timeline draws only visible keys and asks `has` once per drawn key; an edit touches each selected track once. Why: report 2 §1, the August 2026 bug. Cost if wrong: T12 fails.

## 4. Model and interfaces

```ts
// src/animation-core/keyframes/key-selection.ts
export type KeyIdV1 = string          // `${trackId}@${serializedTime}`; VERIFY serializeExactTime
export const keyIdV1 = (trackId: string, time: ExactTime): KeyIdV1
// KeySelectionV1.segments holds bar-clicked segments only; a segment is named by the key that starts it
export const selectedSegmentsV1 = (doc: DocumentV1, s: KeySelectionV1): ReadonlySet<KeyIdV1>   // bar-clicked, plus segments whose two keys are selected

export type SelectModeV1 = 'replace' | 'add' | 'subtract'
export type SelectScopeV1 =
  | { kind: 'track'; trackId: string }
  | { kind: 'layer'; nodeIds: readonly string[] }
  | { kind: 'composition' }
  | { kind: 'workArea' | 'playhead'; nodeIds: readonly string[] | 'all' }
  | { kind: 'everyNth'; n: number; offset: number }        // thins the current selection per track
  | { kind: 'easeType'; ease: EaseV1['type'] | 'linear' | 'auto'; nodeIds: readonly string[] | 'all' }
  | { kind: 'label'; label: KeyLabelV1; nodeIds: readonly string[] | 'all' }
  | { kind: 'invert' }                                    // within tracks holding a selected key; all when none
  | { kind: 'marquee'; trackIds: readonly string[]; from: ExactTime; to: ExactTime }   // the rows the box covers
export const selectKeysV1 = (doc: DocumentV1, scope: SelectScopeV1, current: KeySelectionV1, mode: SelectModeV1): KeySelectionV1

export interface KeySetV1 { readonly id: string; readonly name: string; readonly members: readonly KeyIdV1[] }   // document data
export const keySetStatusV1 = (doc: DocumentV1, set: KeySetV1): { present: number; missing: number }
export const trackTimeIndexV1 = <V>(track: TrackV1<V>): ReadonlyMap<string, number>   // serialised time to index, memoised by keys identity

export interface KeyMoveV1 { readonly from: KeyIdV1; readonly to: KeyIdV1 }
export interface EditResultV1 { readonly doc: DocumentV1; readonly moves: readonly KeyMoveV1[]; readonly removed: readonly KeyIdV1[]; readonly report?: string }
export type BulkEditV1 =
  | { type: 'absolute'; value: number | readonly number[] }
  | { type: 'offset'; delta: number | readonly number[] }
  | { type: 'scale'; factor: number }                     // Proportional scrub, `*k` and `/k`
export const bulkEditV1 = (doc: DocumentV1, selection: KeySelectionV1, edit: BulkEditV1): EditResultV1
export const reverseSelectionV1 = (doc: DocumentV1, selection: KeySelectionV1): EditResultV1
export const toGridUnitsV1 = (v: number, precision: number): number   // Math.round(v * 10 ** precision)

export type ValueKindV1 = 'scalar' | 'vec2' | 'colour' | 'enum'
export interface FieldGroupV1 { readonly kind: ValueKindV1; readonly unit: string; readonly propertyIds: readonly string[]; readonly keyCount: number; readonly common: unknown | 'mixed'; readonly absoluteAllowed: boolean; readonly proportionalDefault: boolean }
export const fieldGroupsV1 = (doc: DocumentV1, selection: KeySelectionV1): readonly FieldGroupV1[]
```

The selection store (UI side) holds one `KeySelectionV1`, applies `moves` and `removed` from every `EditResultV1`, in reverse on undo and forward on redo (the moves ride on the undo entry as travel data, not document state; `VERIFY:` 00 §8.8), and persists across view switches and playback. A marquee binary-searches each covered track's sorted keys for its time range. Position tracks are `vec2`; Ruling 10's path is rebuilt from the keyed points after an edit.

## 5. UX

**Selecting.** One highlight in the timeline, the graph (spec 04), the inspector's key badges and the canvas (selected Position keys are points on the motion path). Click a key: replace. Shift-click: toggle. Click a segment bar: that segment alone. Click a track name: all its keys; Shift-click adds a track; Alt-click removes its keys (proposed, section 12). Marquee: Shift adds, Alt subtracts, layers stay selected (00 §4); over a collapsed layer's summary row (spec 09) it selects the keys of all that layer's tracks in the range. Click on empty timeline: clear keys, keep layers.

**Select menu** (timeline panel menu and right-click). All keys on track (the right-clicked or inspector-focused track); All on layer (selected layers); All in composition; All in work area and At playhead (selected layers; all when none); Every Nth… (dialog: N default 2, Offset default 0, the zl_Scriptlets alternating default; counted per track from its earliest selected key); By ease type ▸ (Linear, Ease, Hold, Spring, Elastic, Bounce, Auto tangent: the segments and their keys); By label ▸ (spec 09); Invert. None reads scroll, zoom, collapse or shy state. The status bar shows "412 keys, 380 eases selected" after each command.

**Key Sets.** "Save selection as Key Set…" in the Select menu (name field, default "Key Set 3"). A Key Sets list in the timeline side panel (`VERIFY:` placement) shows each name with a badge, "12", or "9 of 12" in amber with hover text "3 keys were deleted. Click to select the 9 that remain." Click: replace; Shift-click: add. Per-set menu: Update to current selection, Forget missing keys, Rename, Delete. Sets save with the file; each change is an undo step.

**Inspector field with several keys selected.** Badge: "3 keys" or "Opacity · 3 tracks · 9 keys". The number shows the common value, else "Mixed". Under the field a toggle, Absolute | Offset, with a third state, Proportional, for scalar and vec2 kinds. Defaults: Absolute when the values agree; Proportional when they differ and the property has a natural zero (Opacity, Scale, Rotation, numeric effect parameters); Offset for Position and Anchor Point (their zero is the composition's corner) and whenever Absolute is not allowed (06-R6; the disabled button's hover text names the properties). The choice is sticky per session per kind. A plain number follows the toggle; `+5` or `-5` applies an offset and `*2` or `/2` a scale whatever the toggle; `=-5` forces an absolute negative; unit suffixes parse (00 §4). Scrubbing in Offset mode adds 1 unit per px, Shift 10, Ctrl [Cmd] 0.1 (`VERIFY:` the existing gain, 00 §8.5); in Proportional mode it multiplies by 1 % per px, Shift 10 %, Ctrl 0.1 %, and while the pointer is down the field shows the factor ("×1.25") with a readout "20 → 25 … 80 → 100". Colour and enum groups allow Absolute only.

**Reverse.** Keys ▸ Reverse Selected Keys (menu, no default shortcut); status "Reversed 7 keys on 2 tracks"; disabled when no track has two selected keys.

**Keyboard and states.** J/K and Shift+J/K (spec 10); Delete removes the selected keys; Escape clears the key selection, not the layer selection; Tab moves focus into the inspector's first field of the selection (`VERIFY:` focus order); section 12 proposes Ctrl+Alt+A. Nothing selected: the inspector shows the value at the playhead, as today. Locked layers: keys select and show; edits skip them with the status "4 keys on locked layers skipped". A Key Set with no keys left shows "0 of 12" and a Delete hint.

## 6. User flows

**Flow 1 (primary): raise every key on an Opacity track by 10.** Start: Opacity keys 0, 60, 60, 0 at frames 0, 10, 40, 50; nothing selected. (1) Click "Opacity" in the track list: the four keys and three bars highlight in the timeline and the graph; the inspector's Opacity field shows "Mixed", badge "4 keys", toggle on Proportional. (2) Click Offset. (3) Scrub the field 10 px right: the field reads "+10"; the canvas at the playhead updates live. (4) Release: one undo step, "Offset Opacity on 4 keys". End: values 10, 70, 70, 10; differences unchanged; selection intact.

**Flow 2 (keyboard only): double every Scale key on the selected layers.** Start: three layers selected, each with Scale keys 50 and 100 on a collapsed track. (1) Ctrl+Alt+A (section 12): every key on the three layers selects, collapsed tracks included; status "18 keys selected". (2) Tab: focus lands on the first field group, "Scale · 3 tracks · 6 keys", showing "Mixed". (3) Type `*2`, Enter: every track reads 100 and 200; status "Scaled 6 keys ×2". End: one undo step; focus stays in the field.

**Flow 3 (multi-selection across tracks): set the first key of three fades to 0.** Start: three layers, each with Opacity 20 → 100. (1) Marquee frame 0 across the three Opacity rows: three keys select; the inspector shows "Opacity · 3 tracks · 3 keys", value 20, Absolute (values agree, one property). (2) Type 0, Enter: all three become 0. (3) Shift-marquee adds the three end keys: the field shows "Mixed"; Absolute still allowed. End: six keys selected, three at 0.

**Flow 4 (undo): reverse a sparse selection.** Start: an Opacity track with keys at frames 0, 10, 20, 30, 40 valued 0, 100, 0, 100, 0, all selected. (1) Select ▸ Every Nth…, N 2, Offset 1, OK: the keys at 10 and 30 stay selected. (2) Shift-click the key at frame 0: selected values 0, 100, 100 at frames 0, 10, 30. (3) Keys ▸ Reverse Selected Keys: values become 100, 100, 0 at the same frames; the keys at 20 and 40 are untouched; only the segment 0→10 (both ends selected and adjacent) has its ease mirrored. (4) Ctrl+Z: values return to 0, 100, 100; the same three keys are still selected (Ruling 11). End: the document as before step 3.

**Flow 5 (limit): a Key Set after deleting keys.** Start: 12 keys across four tracks saved as Key Set "Hit 1", badge "12". (1) Marquee three of them, Delete: the badge reads "9 of 12" in amber. (2) Click "Hit 1": the 9 remaining keys select; status "9 of 12 keys selected; 3 are missing". (3) Ctrl+Z: the three keys return and the badge reads "12" (the set never shrank). (4) Ctrl+Shift+Z, then the set's menu ▸ Forget missing keys: badge "9". End: 9 members; the prune is one undo step.

**Flow 6 (limit): 5,000 keys, then a proportional scrub.** Start: 60 layers, 6,000 keys. (1) Select ▸ All in composition: 6,000 keys select within one frame. (2) Alt-marquee a region: the update lands in the next frame. (3) Click a Scale track with keys 50, 100, 150: field "Mixed", toggle on Proportional. (4) Scrub 20 px right: the field reads "×1.20", readout "50 → 60 … 150 → 180"; release: 60, 120, 180. End: ratio 1 : 2 : 3 kept; no update exceeded 16 ms (T12).

## 7. Evaluation and determinism

Selection evaluates nothing. Every bulk edit and Reverse is a pure function `(document, selection) → document` producing ordinary keys, so `evaluate(document, time)` is unchanged in kind (Ruling 9). No driver, no bake. Reverse on a track with `auto` tangents re-derives slopes from the reversed neighbours at evaluation (Ruling 5); `free` segments carry their mirrored ease. Position tracks: Reverse reverses the keyed points, the spatial path is rebuilt through them with auto tangents, and a user-edited canvas tangent pair is swapped in for out at each key (Ruling 10).

## 8. Edge cases and named limits

- Ids name a time, so two selected keys never collide (Ruling 7). After a merge (spec 07) the survivor keeps its id; the removed id leaves the selection and counts as missing in Key Sets.
- An Offset or scale that leaves the property's range (Opacity above 100) clamps per key (`VERIFY:` range metadata); differences are then not preserved for the clamped keys and the status reads "3 keys clamped at 100".
- Proportional with every selected value 0 has no reference; the toggle falls back to Offset with hover text. Proportional on vec2 scales both channels about (0, 0), which is why Position defaults to Offset.
- Off-grid values (bake, import) round to the grid on their first bulk edit, by at most half a unit. Named limit.
- A hold between selected keys mirrors to a hold (spec 03). Mirrored spring, elastic and bounce eases: spec 03 decides; the involution test uses bezier and hold.
- Reverse skips a track with one selected key. Every Nth keeps a track's first selected key when it has fewer than N.
- Labels (spec 09) stay at their time under Reverse; values move. Spec 09 may overturn.

## 9. Interactions with other specs

- 01, 03: `selectedSegmentsV1` is what Ctrl+Shift+E and Paste Ease act on; a bar click selects one segment without its keys.
- 02, 08: a run (00 glossary) is derived from this selection per track; stagger and retime commands take the selection and return `moves`.
- 04: the graph shows and edits the same ids; its transform box returns `EditResultV1`.
- 05: baked tracks are ordinary keys; bulk edits never touch the retained driver parameters.
- 07: drags, nudges and Snap return `moves`; the edge handles exist when two or more keys are selected.
- 09: "By label"; the summary row's marquee rule; the same-frame highlight reads the selection.
- 10: copy, paste and J/K scope read this selection; Paste returns `moves` for the pasted keys and selects them.

## 10. Rules and tests

Each rule is a test in `key-selection.test.ts` or `bulk-edit.test.ts`.

- R1 scope beats view. T1 `select all on layer counts keys on collapsed tracks`: Opacity expanded (4 keys), Scale collapsed (6); `selectKeysV1({ kind: 'layer' })` gives 10 keys and 8 derived segments; All in composition on 60 layers, 6,000 keys, half shy, a scrolled view: 6,000.
- R2 segments derive, bar clicks add alone. T2: select keys 0 and 10: `selectedSegmentsV1` has 1; bar-click 20→30: 2 segments, still 2 keys.
- R3 typed absolute sets every key. T3 `typed absolute sets every selected key to the value`: 20, 40, 80, absolute 55: `[55, 55, 55]`, `Object.is` each.
- R4 scrubbed offset preserves differences bit-exactly. T4: 20.3, 40.7, 80.1, offset +0.1 ten times then −0.1 ten times: after every step `toGridUnitsV1(v_i) − toGridUnitsV1(v_j)` equals the original integer for every pair; the final values `Object.is` the originals.
- R5 proportional preserves ratios; scale and `*` share a path. T5 `proportional scrub preserves ratios`: 20, 40, 80 × 1.25 gives 25, 50, 100 (`Object.is`); 33.33, 66.66 × 1.1 gives each value within half a grid unit of the exact product; `bulkEditV1(scale 2)` deep-equals the parse of `*2`.
- R6 reverse is an involution. T6 `reverse of a sparse selection is an involution`: keys at 0, 10, 20, 30, 40 with distinct bezier and hold eases, selection {0, 10, 30}; reverse twice deep-equals the input; after one reverse every time is unchanged and the keys at 20 and 40 are untouched.
- R7 reverse mirrors only selected segments. T7: as T6; after one reverse `keys[0].ease` equals `mirrorEaseV1` of the original and `keys[1].ease` equals its original (the segment 10→20 was half-selected).
- R8 Key Sets survive unrelated undo. T8 `a Key Set survives undo of an unrelated edit`: save a set of 12; offset another track; undo: 12 present.
- R9 Key Sets go stale, not smaller. T9: delete 3 members: 9 present, 3 missing, 12 members; undo: 12 present.
- R10 selection follows keys. T10: move a selected key +5 frames through `EditResultV1.moves`: the selection holds the new id; undo: the old id.
- R11 one field group per kind. T11: Opacity and Scale keys: one scalar group, `absoluteAllowed` false; Opacity on three layers: true; Opacity and Position: two groups.
- R12 the 16 ms budget. T12 `5,000 selected keys stay under 16 ms`: a 6,000-key fixture; `selectKeysV1(composition)`, a marquee over half, `bulkEditV1(offset)` on 5,000 keys, each the median of 20 runs under 16 ms (one frame at 60 Hz; `VERIFY:` the CI runner's variance); `has` calls per timeline draw equal the visible key count, not the selected count.
- R13 locked layers select but do not edit. T13: 4 of 10 selected keys on a locked layer; offset: 6 change, the report names 4 skipped.
- D1 `play equals seek` on a fixture after `reverseSelectionV1` and after `bulkEditV1(offset)`: frames 0..N in order, then N alone from a fresh evaluator; `Object.is` per channel.
- D2 `random order`: a shuffled frame list equals D1.
- D3 `after an edit`: offset, undo, evaluate: equal to before the edit; the selection is unchanged.
- D4 `bake round-trip`: this spec bakes nothing; on spec 05's baked fixture, reverse twice then unbake restores the source parameters exactly, and a bulk offset on a baked track leaves its retained parameters untouched.

## 11. VERIFY list

1. `serializeExactTime`, `compareExactTime` and the track id format `${nodeId}/${propertyId}` (00 §8.1).
2. The document type: tracks, layers, the locked, shy and hidden flags, a `keySets` array, the version field (00 §8.6).
3. The undo entry type: whether an entry can carry `moves` as metadata (00 §8.8).
4. The inspector's number field: scrub gain, precision per property, range metadata, focus order (00 §8.5).
5. The selection colour and the existing layer selection store.
6. The CI runner's timing variance for T12.
7. Whether the timeline rows are virtualised (R12 assumes only visible keys are drawn).

## 12. Open questions for the user

1. Proportional defaults to Offset for Position and Anchor Point (06-R4). Confirm, or make Proportional the default wherever values differ.
2. Reverse keeps labels at their time (section 8). Confirm before spec 09.
3. Values round to the property's grid on a bulk edit (06-R3). Confirm the grid, hundredths, or set it per property.
4. **Proposed addition to 00 §4** (keyboard): "Ctrl+Alt+A: select every key on the selected layers, collapsed tracks included; all layers when none selected" (spec 06). The AE shortcut's visible-only meaning is deliberately replaced.
5. **Proposed addition to 00 §4** (gesture): "Alt [Option]-click a track name: remove the track's keys from the selection" (spec 06), the complement of Shift-click and consistent with Alt-subtract in the marquee.
6. **Proposed note for 00 §4 number fields:** with several keys selected a leading `-` is an offset, `=-5` forces an absolute negative, and `/2` joins `*2`.
