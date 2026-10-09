# Spec 10: parity baseline: multi-layer copy and paste, Paste Reversed, scoped navigation and Alt-drag duplicate

Written without the repository open; assumptions about unseen code are marked `VERIFY:` and gathered in section 11. Depends on 00, 06 and 07; spec 03 consumes the clipboard payload; spec 09 supplies labels and the key filter.

## 1. Purpose and evidence

After Effects closed multi-layer copy, cut and paste and Paste Reversed in 24.4 and scoped key navigation in 23.0, so a new app ships them as parity rather than differentiation (report 2, "Adobe fixed timeline mechanics" and the ranked table, ranks 10 to 12). The paste rules are AE's own, which migrating users already know: the earliest key at the playhead, same-name properties many-to-many, type-compatible properties one-to-one, and copying from one layer at a time, the limit that Paste Multiple Keyframes and Key Cloner sold around before 2024 (bulk-editing research note, KQ1 and KQ2). Alt-drag duplicate is a 2022 request AE cannot grant because Alt is taken by time-stretch (report 2, "Timing" section; wishlist research note, KQ1). Landing on the exact key answers "the playhead does not fall on the exact keyframe" (wishlist research note, KQ1).

## 2. Scope and non-goals

In scope: Copy, Cut, Paste and Paste Reversed across any number of layers and tracks; the clipboard payload and its text fallback; paste into another composition; J/K, Shift+J/K and Go to key number; Alt-drag duplicate and its menu form, Duplicate to playhead. Out of scope: Paste Ease and Paste Values (spec 03, which reads the same payload); paste with links or instancing (section 12); reverse in place (spec 06); the stagger drag (spec 02); looping paste (spec 08's loops cover it); copying layers (`VERIFY:` existing); markers and work-area edges as navigation stops (section 12).

## 3. Rulings

- **10-R1 (the clipboard is a JSON fragment with relative exact times):** each key's time is stored as the exact rational difference from the earliest copied key, in seconds; paste adds the playhead's whole frame. Why: relative timing is then bit-exact, and a cross-rate paste keeps the motion's duration rather than its frame count. Cost if wrong: a "keep frame count" option on the paste.
- **10-R2 (target resolution order):** one track selected by name with one source track pastes one-to-one when the value type and dimension match; otherwise selected layers take every source track by property id, many-to-many; otherwise the source tracks take it. A refusal names what is missing and writes nothing. Why: these are AE's rules, and naming the rule in the refusal makes it teachable. Cost if wrong: one function.
- **10-R3 (Paste Reversed mirrors the whole payload):** times reflect within the payload's full span, not per track, and every bezier ease is mirrored by spec 03's involution; hold stays hold; spring, elastic and bounce keep their parameters. Why: a stagger copied from three layers and pasted reversed must run backwards across layers; reversing twice must give back the original. Cost if wrong: one flag for per-track spans.
- **10-R4 (navigation stops at the ends and lands exactly):** J and K set the playhead to the key's exact time and do nothing at the last key, with a status message. Why: AE and Blender stop, so nobody expects a wrap, and a wrap moves the playhead out of view at any useful zoom. Cost if wrong: one branch.
- **10-R5 (Alt-drag duplicates on whole frames, Escape cancels, copies are new keys):** the ghost copies follow spec 07's model without the Ctrl sub-frame bypass (Ctrl+Alt is Stagger); copies carry value, ease, tangent mode, label and kind but no key id or group membership; a release at zero offset writes nothing; the selection moves to the copies. Why: 00 §4 reserves the gesture, Illustrator and Premiere set the expectation, and a zero-offset write would replace every original with itself. Cost if wrong: one modifier.
- **10-R6 ("visible tracks" means not filtered):** the "visible tracks" of 00 §4's J/K row are tracks not hidden by a filter (spec 09's label filter, hidden or shy layers); a collapsed layer's tracks count. Why: spec 06 makes commands independent of what is expanded, and "only what your eyeballs can view" is the AE complaint. Cost if wrong: one predicate.

## 4. Model and interfaces

```ts
// src/animation-core/keyframes/key-clipboard.ts
export type ExactTimeJsonV1 = { readonly n: string; readonly d: string }   // bigints as decimal strings; VERIFY serializeExactTime
export interface ClipboardKeyV1 { readonly dt: ExactTimeJsonV1; readonly value: unknown; readonly tangent: TangentModeV1; readonly ease: EaseV1; readonly label?: KeyLabelV1 }   // dt = time − earliest copied key
export interface ClipboardTrackV1 { readonly propertyId: string; readonly valueType: string; readonly dimension: 1 | 2 | 3 | 4; readonly sourceLayerId: string; readonly sourceTrackId: string; readonly keys: readonly ClipboardKeyV1[]; readonly easeText: string }   // easeText: spec 03's text form of the eases
export interface KeyClipboardV1 { readonly format: 'motion-keys/1'; readonly sourceCompositionId: string; readonly sourceFps: number; readonly span: ExactTimeJsonV1; readonly tracks: readonly ClipboardTrackV1[] }
export const copyKeysV1 = (doc: DocumentV1, sel: KeySelectionV1): KeyClipboardV1 | null
export const clipboardTextV1 = (clip: KeyClipboardV1): string                     // JSON; a one-track copy also offers spec 03's ease text flavour
export const parseClipboardV1 = (text: string): KeyClipboardV1 | null
export const reverseClipboardV1 = (clip: KeyClipboardV1): KeyClipboardV1          // dt' = span − dt; eases through spec 03's mirrorEaseV1; an involution

export type PasteTargetV1 = { readonly kind: 'track'; readonly trackId: string } | { readonly kind: 'layers'; readonly layerIds: readonly string[] } | { readonly kind: 'source' }
export const resolvePasteTargetV1 = (doc: DocumentV1, sel: KeySelectionV1, compId: string): PasteTargetV1   // 10-R2 order; VERIFY spec 06's track-by-name selection
export interface PastePlanV1 { readonly writes: readonly { trackId: string; keys: readonly KeyframeV1<unknown>[] }[]; readonly replaced: number; readonly skipped: readonly string[]; readonly subFrame: number; readonly refusal?: string }
export const planPasteV1 = (doc: DocumentV1, clip: KeyClipboardV1, target: PasteTargetV1, playhead: ExactTime, fps: number): PastePlanV1   // pure; a plan with `refusal` writes nothing
export const compatibleV1 = (src: ClipboardTrackV1, dst: { valueType: string; dimension: number }): boolean

// src/animation-core/keyframes/key-navigation.ts
export interface NavScopeV1 { readonly layerIds: readonly string[] | 'all'; readonly filter: KeyFilterV1 | null }
export const keyTimesInScopeV1 = (doc: DocumentV1, compId: string, scope: NavScopeV1): ExactTime[]   // distinct, sorted, exact
export const nextKeyTimeV1 = (times: readonly ExactTime[], playhead: ExactTime): ExactTime | null       // strictly later; null at the end
export const previousKeyTimeV1 = (times: readonly ExactTime[], playhead: ExactTime): ExactTime | null
export const keyNumberAtV1 = (times: readonly ExactTime[], playhead: ExactTime): number | null         // 1-based, null between keys

// src/animation-core/keyframes/key-duplicate.ts
export interface DuplicatePlanV1 { readonly copies: readonly { trackId: string; key: KeyframeV1<unknown> }[]; readonly collisions: readonly { trackId: string; time: ExactTime }[] }
export const planDuplicateV1 = (doc: DocumentV1, sel: KeySelectionV1, delta: ExactTime): DuplicatePlanV1   // delta already snapped by spec 07; no copy carries an id
```

Commands, one undo step each (Ruling 11): Copy (no step); Cut "Cut 12 keys"; Paste "Paste 12 keys" or "Paste 12 keys (replaced 3)"; Paste Reversed "Paste reversed 12 keys"; Duplicate "Duplicate 4 keys (replaced 1)"; Duplicate to playhead, the same plan with the earliest copy on the playhead's frame. Go to key number and J/K move the playhead only.

## 5. UX

**Copy and Cut.** Ctrl+C copies every selected key, across any number of layers and tracks, as one payload; the status bar reads "Copied 12 keys from 4 tracks on 2 layers". Ctrl+X (section 12) copies then deletes, one undo step. The system clipboard receives the JSON as text; a one-track copy also offers spec 03's ease text so Paste Ease (Ctrl+Alt+V) works from the same copy. The in-app payload is preferred when the system text still matches it (`VERIFY:` clipboard access in Electron and the browser).

**Paste (Ctrl+V).** Target by 10-R2:
- One track selected by name (spec 06) and one source track: one-to-one when `compatibleV1`, else refuse "Can't paste Position (2 values) onto Opacity (1 value)". Two or more source tracks: refuse "The clipboard holds 3 tracks. Select a layer to paste them by property, or copy one track to paste it here".
- Layers selected: each source track pastes to the same property id on every selected layer. A layer lacking a property is skipped and named: "Pasted 6 tracks on 3 layers. Skipped: Logo has no Stroke Width". No match at all: refuse "None of the selected layers has Position or Scale".
- Nothing selected: the source tracks, same composition, at the playhead. In another composition: refuse "Select a layer to paste into".

The earliest key lands on `snapToFrameV1(playhead)`; every other key keeps its exact offset. Keys landing on occupied times replace the occupant (Ruling 7) and the count goes in the undo label. The pasted keys become the selection; the playhead stays. A refusal is a status-bar message with a "Why?" link to these rules (`VERIFY:` messaging surface), never a dialog.

**Across compositions and rates.** The payload is global and property ids match by name. When the target rate differs, offsets stay exact in seconds; keys that fall between frames are counted: "Pasted 12 keys, 4 between frames (24 to 30 fps). Snap them with Ctrl+Shift+F" (spec 07).

**Paste Reversed (Ctrl+Shift+V).** `reverseClipboardV1`, then the paste above: the last copied key lands on the playhead's frame and the earliest at playhead plus span; bezier eases are mirrored; hold stays hold; spring, elastic and bounce keep their parameters and the message adds "2 springs kept as they are". Undo label "Paste reversed 12 keys".

**Navigation.** J / K: previous / next distinct key time among the selected layers' tracks (collapsed or not; filtered keys skipped), or all layers when no layer is selected; Shift+J / Shift+K: all layers always. The playhead lands on the exact time; a sub-frame landing reads "12.5f" in the time display (`VERIFY:` spec 07's display). At an end nothing moves and the status bar reads "No later key in the selected layers" or "No earlier key". After each jump the status bar reads "Key 7 of 31". Timeline menu "Go to Key Number…" (section 12 proposes a shortcut) takes an integer; out of range reads "There are 31 keys in this scope".

**Alt-drag duplicate.** Alt+mousedown on a selected key duplicates the whole selection; on an unselected key, that key alone. The originals stay; ghost copies follow the pointer with spec 07's model (whole frames; Shift adds snap targets; Ctrl unavailable, 10-R5); the readout shows "+6f". A ghost over an occupied frame on its track, including an original of the same selection when the offset is shorter than the selection's extent, draws red and the readout adds "(replaces 2)". Release writes the copies, replaces by Ruling 7, moves the selection to the copies and labels the step "Duplicate 4 keys (replaced 2)". Release at zero offset: nothing happens. Escape: the ghosts vanish, nothing is written, the selection is unchanged. Works in the dope sheet, spec 09's summary rows and the graph editor (spec 04, `VERIFY:`). Hover text on a key: "Alt+drag to duplicate". The context menu offers "Duplicate to playhead".

Empty states: Ctrl+V with an empty clipboard reads "Nothing to paste"; J/K with no key in scope reads "No keys in the selected layers".

## 6. User flows

1. **Primary: copy from two layers, paste to three.** Start: on layers Title and Icon, Position (3 keys) and Opacity (2 keys) selected, 10 keys; playhead at 48. (a) Ctrl+C: "Copied 10 keys from 4 tracks on 2 layers". (b) Select layers A, B and C. (c) Ctrl+V: Position and Opacity on each of A, B and C receive keys with the earliest at 48; "Pasted 6 tracks on 3 layers"; undo label "Paste 30 keys". End: 30 new keys, selected; the sources untouched.
2. **Keyboard only: copy, navigate, paste, paste reversed.** Start: layer Ball selected, Position keyed at 0, 6, 12, 24; playhead at 0; another layer keyed at 30. (a) Ctrl+A (spec 06, `VERIFY:`) selects Ball's keys; Ctrl+C. (b) K three times: playhead 6, 12, 24, status "Key 4 of 4". (c) K: "No later key in the selected layers". (d) Shift+K: playhead 30. (e) Ctrl+V: Ball's Position receives the run at 30 to 54; "Paste 4 keys". (f) K to 54, then Ctrl+Shift+V: the run pasted reversed from 54 to 78 with mirrored eases; "Paste reversed 4 keys". End: original, forward copy, reversed copy; status "Key 10 of 10".
3. **Multi-selection: Alt-drag a cross-layer selection into a collision.** Start: 6 keys selected on Position and Opacity of two layers, frames 10 to 20; Opacity on layer B also keyed at 25. (a) Alt+mousedown on a selected key, drag right: at +0 every ghost is red (over its original). (b) At +5 only B's Opacity ghost at 25 is red; readout "+5f (replaces 1)". (c) Release: copies at 15 to 25, the key at 25 replaced, undo "Duplicate 6 keys (replaced 1)", selection on the copies. End: originals unchanged.
4. **Undo: a paste that replaced keys.** Start: Opacity keyed 0 at 48 and 100 at 60; clipboard holds Opacity 100 at dt 0 and 0 at dt 12 frames; playhead 48. (a) Ctrl+V: both keys replaced; undo label "Paste 2 keys (replaced 2)"; the canvas at 48 shows 100. (b) Ctrl+Z: the original keys return with values and eases; the canvas at 48 shows 0; selection unchanged. End: as start.
5. **Limit: an incompatible one-to-one paste.** Start: clipboard holds one Position track; the user clicks the Opacity track name. (a) Ctrl+V: refused, "Can't paste Position (2 values) onto Opacity (1 value)"; nothing changes, no undo step. (b) Click the Anchor track name; Ctrl+V: "Pasted 3 keys onto Anchor". End: Anchor keyed, Opacity untouched.
6. **Limit: a cross-rate paste leaves sub-frame keys.** Start: keys at frames 0, 1, 2 copied from a 24 fps composition; a 30 fps composition open, playhead at 0, layer selected. (a) Ctrl+V: keys at 0, 1/24 s and 2/24 s (1.25 and 2.5 frames); "Pasted 3 keys, 2 between frames (24 to 30 fps). Snap them with Ctrl+Shift+F". (b) Ctrl+Shift+F: keys at 0, 1 and 3 (2.5 ties later, 00 §3). End: whole frames.
7. **Escape cancels a duplicate.** Start: 4 keys selected. (a) Alt+drag to +8f: ghosts shown. (b) Escape: ghosts vanish. End: document identical, no undo step, selection unchanged.

## 7. Evaluation and determinism

Copy, paste, reverse, duplicate and navigation are transforms on key data and the playhead; `evaluateTrackV1` is untouched and every value stays `evaluate(document, time)` (Ruling 9). `planPasteV1`, `reverseClipboardV1` and `planDuplicateV1` are pure in their inputs, so the same clipboard, target and playhead always produce the same keys. Nothing here bakes. Named limit: a copied baked track (spec 05) pastes as plain keys without its retained source parameters.

## 8. Edge cases and named limits

- A one-key payload has span 0, so Paste Reversed equals Paste.
- Auto-tangent keys stay `auto` after reversal and are recomputed on the reversed data; the mirrored shape follows from Ruling 5's symmetric formula. The involution test compares stored data, not recomputed slopes.
- The last copied key's ease travels with it and becomes the ease of the segment to whatever key follows after the paste.
- A sub-frame playhead (spec 07) pastes at `snapToFrameV1(playhead)`.
- Position keys carry their path tangents (Ruling 10, `VERIFY:` value shape), so Position pastes onto Anchor one-to-one with its path.
- A boolean or enum track takes nothing from a numeric one: `valueType` differs.
- A payload with another `format` is refused: "This clipboard is from a newer version".
- Ctrl held at mousedown with Alt is Stagger (spec 02), never duplicate.
- Keys may be duplicated or pasted past the composition end (`VERIFY:` legal today).
- Copies and pasted keys never join spec 09's groups.

## 9. Interactions with other specs

03: Ctrl+Alt+V and Ctrl+Alt+Shift+V read the eases and the values of the same payload; `mirrorEaseV1` and the ease text are spec 03's (`VERIFY:` names). 06: selection model, track-by-name selection, Ctrl+A, the selection after a paste. 07: `snapToFrameV1`, the drag model and ghost drawing, the sub-frame display, Snap All Keys. 09: `label` travels; the key filter narrows J/K; summary diamonds Alt-drag; no ids travel. 02: Ctrl+Alt+drag reserved. 04: the graph editor pastes and duplicates with the same commands. 05: bake parameters do not travel. 00 §9 question 1 must be answered before the paste rules are built.

## 10. Rules and tests

Rules, each a test: R1 copying from two layers and pasting to three selected layers produces six tracks of keys at the playhead. R2 relative times are preserved bit-exactly. R3 a one-to-one paste to a compatible track works and to an incompatible track is refused with the message and writes nothing. R4 reversal is an involution, so Paste Reversed twice equals Paste. R5 J/K with two layers selected skip the third layer's keys; Shift+J/K visit them. R6 navigation lands on the exact time, including sub-frame keys, and stops at the ends. R7 Alt-drag duplicate leaves the originals untouched and places the copies at the dropped frames with eases and labels equal. R8 undo of a paste that replaced keys restores them. R9 nothing selected pastes to the source tracks. R10 key numbers match J/K steps. R11 Escape and a zero offset write nothing.

Tests by name (`key-clipboard.test.ts`, `key-navigation.test.ts`, `key-duplicate.test.ts`):
- T1 `two layers to three selected layers gives six tracks at the playhead`: sources A and B with Position {0, 6, 12} and Opacity {0, 12} frames at 24 fps; targets C, D, E; playhead 48/24: six writes, 30 keys, the earliest on each track exactly 48/24.
- T2 `relative times are bit-exact`: keys at 7/24, 1/3 s and 101/24; pasted at 50/24 the reduced (n, d) differences equal the originals'; into a 30 fps composition the differences in seconds are equal too.
- T3 `one-to-one compatible and incompatible`: Position (2) onto Anchor (2) writes; onto Opacity (1) the plan has `refusal` equal to "Can't paste Position (2 values) onto Opacity (1 value)" and no writes. T4 `two source tracks onto one track refuses`.
- T5 `reverseClipboardV1 is an involution`: a payload with bezier (0.2, 0, 0, 1), hold and spring; twice deep-equals; once, dt' = span − dt, the bezier is (1, 0, 0.8, 1), hold stays, spring unchanged.
- T6 `paste reversed twice equals paste`: paste reversed at 48, cut, paste reversed at 48 → deep-equal to a plain paste at 48.
- T7 `J/K scope`: layers A and B selected, keys A {6}, B {12}, C {9}; from 0, K gives 6, 12, then null, never 9; Shift+K gives 6, 9, 12. T8 `lands on a sub-frame key exactly`: key at 1/3 s; from 7/24 the next is {n: 1, d: 3}. T9 `stops at the ends`: previous from the first key is null.
- T10 `key numbers`: times [6, 9, 12]; `keyNumberAtV1(9)` is 2; at 10 null.
- T11 `duplicate leaves the originals untouched and places copies at the dropped frames`: 6 keys, delta 5/24; originals deep-equal before and after; copies at t + 5/24 with eases and labels deep-equal and no id.
- T12 `duplicate collision replaces by Ruling 7`: one collision; after commit the occupant is gone; label "Duplicate 6 keys (replaced 1)". T13 `zero offset and Escape write nothing`: document identity unchanged, undo stack length unchanged.
- T14 `undo of a replacing paste restores the replaced keys`: values `Object.is`, eases deep-equal, times unique and sorted.
- T15 `nothing selected pastes to the source tracks`. T16 `skipped layers are named and zero matches refuse`: a target without Stroke Width → `skipped` ["Logo has no Stroke Width"], other writes proceed; no match → `refusal`.
- T17 `cross-rate count`: 24 to 30 fps keys at 0, 1, 2 frames → `subFrame` 2. T18 `clipboard text round-trip`: `parseClipboardV1(clipboardTextV1(c))` deep-equals c; the ease flavour equals spec 03's text for one track.
- D1 `play equals seek`, D2 `random order`, D3 `after an edit` (paste reversed, undo), D4 `bake round-trip` (nothing bakes: paste, reverse and duplicate never call the bake service; a pasted copy of a baked track evaluates equal to its keys at every frame), on a fixture built by a multi-layer paste, a paste reversed and an Alt-drag duplicate.

## 11. VERIFY list

1. Clipboard access in Electron and the browser: flavours, permissions, an in-app store.
2. `serializeExactTime`'s JSON form for `ExactTimeJsonV1`.
3. Spec 06: track-by-name selection, Ctrl+A, how the selection is set after a paste.
4. Spec 07: the drag model's ghost and collision drawing hook; the sub-frame time display.
5. Spec 03: `mirrorEaseV1` and the ease text function.
6. Property ids are stable names across layer types and compositions; the Position key value shape.
7. The keymap for Ctrl+X and Ctrl+Alt+J; whether the browser build can intercept Ctrl+Shift+V.
8. Whether keys past the composition end are legal; the status-bar messaging surface.

## 12. Open questions for the user

1. Proposed addition to 00 §4, keyboard: "Ctrl+X: cut keys (10)".
2. Proposed addition to 00 §4, keyboard: "Ctrl+Alt+J: Go to key number… (10)".
3. Markers and work-area edges as J/K stops, as AE does: this spec visits keys only; confirm.
4. Stop at the ends (10-R4) rather than wrap; confirm.
5. 10-R6's reading of "visible tracks" in 00 §4; confirm or amend the row.
6. Cross-rate paste keeps seconds (10-R1) rather than frame counts; confirm.
7. Paste with links (instancing) is deferred; say if it belongs in this spec.
8. 00 §9 question 1 (Ruling 7 versus a jump key) must be settled before this spec builds.
