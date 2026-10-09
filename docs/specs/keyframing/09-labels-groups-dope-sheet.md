# Spec 09: keyframe labels, groups, the summary row and guides

Written without the repository open; assumptions about unseen code are marked `VERIFY:` and gathered in section 11.

## 1. Purpose and evidence

The June 2026 dope-sheet request asks to see "which layers hit on the same frame and where holds and overshoots fall" without "manually expanding and collapsing properties" (report 2, "Timing" section and ranked table, rank 9). Keyframe colour labels and grouping "like Nuke's backdrop node" have been on the After Effects wishlist since 2008 and 2012, and a 2025 utility still sells a "Keyframes Label Group" selector (report 2, "Adobe fixed timeline mechanics" and "Bulk editing" sections). Blender's typed keys change "only the colour" (report 1 §2; editor-UX research note, KQ5). Guides are a separate open request (timing-tools research note, KQ5).

## 2. Scope and non-goals

In scope: composition and per-layer summary rows; per-track span bars with overlap shading; eight colour labels; three key kinds plus the derived hold glyph; named, coloured groups across tracks and layers; the same-frame highlight with keyboard cycling; time guides. Out of scope: onion skinning and ghosting (canvas features for a later spec); markers (`VERIFY:` the existing model); layer colour labels; the graph editor's own drawing (spec 04); any evaluation change (09-R1).

## 3. Rulings

- **09-R1 (display data never evaluates):** labels, kinds, groups, guides, summary rows and span bars are read by views and selection commands only; `evaluateTrackV1` ignores them. Why: Blender's keyframe types "change only the colour", and animators trust them for it; Ruling 9. Cost if wrong: none.
- **09-R2 (a summary diamond is a derived set of keys):** summary rows store nothing; a diamond is computed per draw from the tracks beneath it, and dragging or selecting it runs the ordinary multi-key move or select. Why: a stored summary is a second source of truth that drifts. Cost if wrong: nothing.
- **09-R3 (groups reference keys by a stable id):** a key that joins a group receives an id (`readonly id?: string` on `KeyframeV1`, proposed in section 12). Moves, nudges, retimes, staggers and snaps keep the key and its id; Ruling 7 replacement removes the occupant with its id; copies (paste, Alt-drag) carry none. Why: the `${trackId}@${time}` reference of `KeySelectionV1` breaks on every drag; surviving "timing/value changes" is what users praised in Key Sets. Cost if wrong: an unread optional field.
- **09-R4 (guides are document data):** guides belong to the composition, save with it, never export, and each guide edit is one undo step. Why: a guide placed to line up a hit must be there next session; Ruling 11 exempts only selection, pan and zoom. Cost if wrong: one flag.
- **09-R5 (eight fixed labels, three kinds, hold derived):** eight named colours, no custom entries; kinds Key, Breakdown and Extreme; the hold glyph is drawn from the segment ease (Ruling 6), never stored. Why: labels sort passes (anticipation, hit, settle), not identities; eight fills one menu row and the digit keys with 0 left for "clear"; Blender's Jitter and Moving Hold mark baked keys, which spec 05 marks Breakdown; a stored hold kind could disagree with the ease. Cost if wrong: a palette constant and one enum value.

## 4. Model and interfaces

```ts
// src/animation-core/keyframes/key-labels.ts
export type KeyLabelColourV1 = 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8
export type KeyKindV1 = 'key' | 'breakdown' | 'extreme'
export interface KeyLabelV1 { readonly colour?: KeyLabelColourV1; readonly kind?: KeyKindV1 }   // 00 §3's label slot
export const KEY_LABEL_NAMES_V1 = ['Red', 'Orange', 'Yellow', 'Green', 'Cyan', 'Blue', 'Purple', 'Pink'] as const
export type KeyGlyphV1 = 'key' | 'breakdown' | 'extreme' | 'hold'
export const keyGlyphV1 = (key: KeyframeV1<unknown>, isLast: boolean): KeyGlyphV1   // 'hold' when !isLast && ease.type === 'hold', else kind ?? 'key'
export interface KeyFilterV1 { readonly colours: ReadonlySet<KeyLabelColourV1>; readonly unlabelled: boolean; readonly glyphs: ReadonlySet<KeyGlyphV1> }

// src/animation-core/keyframes/key-groups.ts
export interface KeyGroupV1 { readonly id: string; readonly name: string; readonly colour: KeyLabelColourV1; readonly members: readonly string[] }   // key ids
export const liveMembersV1 = (group: KeyGroupV1, doc: DocumentV1): { trackId: string; time: ExactTime }[]   // [] means stale
export const groupSpanV1 = (group: KeyGroupV1, doc: DocumentV1): { start: ExactTime; end: ExactTime } | null
export const groupsOfSelectionV1 = (sel: KeySelectionV1, doc: DocumentV1): KeyGroupV1[]   // one item each for specs 02 and 07

// src/timeline/summary-row.ts   (VERIFY: timeline module path)
export interface SummaryDiamondV1 { readonly time: ExactTime; readonly trackIds: readonly string[]; readonly colour: KeyLabelColourV1 | null; readonly selected: 'all' | 'some' | 'none'; readonly lit: boolean }
export const summaryRowV1 = (tracks: ReadonlyMap<string, TrackV1<unknown>>, sel: KeySelectionV1, filter: KeyFilterV1 | null, playhead: ExactTime): SummaryDiamondV1[]
export interface SpanBarV1 { readonly trackId: string; readonly start: ExactTime; readonly end: ExactTime }
export const spanBarsV1 = (tracks: ReadonlyMap<string, TrackV1<unknown>>): SpanBarV1[]
export const spanCoverageV1 = (bars: readonly SpanBarV1[]): { start: ExactTime; end: ExactTime; count: number }[]
export const tracksKeyedAtV1 = (doc: DocumentV1, compId: string, time: ExactTime): string[]

// src/animation-core/document/time-guides.ts
export interface TimeGuideV1 { readonly id: string; readonly time: ExactTime; readonly name?: string; readonly locked: boolean }
// guides and groups live per composition (VERIFY: where editor data sits in the document)
```

Commands, one undo step each (Ruling 11): `labelKeysV1` "Label 6 keys Red"; `setKeyKindV1` "Set 6 keys to Breakdown"; `createGroupV1` "Group 10 keys as Hit"; `addToGroupV1`, `removeFromGroupV1`, `renameGroupV1`, `deleteGroupV1`; `moveSummaryDiamondV1(diamond, delta)`, which calls spec 07's move command (`VERIFY:` name) and reports "Move 3 keys (replaced 1)"; `addGuideV1(time, name?)` "Add guide at 24f"; `moveGuideV1`, `lockGuideV1`, `renameGuideV1`, `deleteGuideV1`. Guide times pass through `snapToFrameV1` unless spec 07's Ctrl modifier is held.

## 5. UX

**Summary rows.** The composition row "All keys" is the first row and cannot be collapsed. A layer draws its summary on its own row when collapsed and as a "Summary" row above its tracks when expanded (timeline menu "Show summary rows when expanded", default on, view state). One diamond per distinct key time, filled with the label colour when every key there shares one label, else the default key colour; a count badge top right when more than one track keys there; outline only when some of its keys are selected, filled when all; a lit ring when the playhead sits there. Gestures from 00 §4: click selects every key at that time on the layer (Shift adds, Alt subtracts, a marquee on the row selects by time); drag moves them with spec 07's model (whole frames; Shift adds keys, markers, guides and the playhead as snap targets; Ctrl free), the ghost red on any track where the drop would replace a key (Ruling 7).

**Span bars.** Each track row draws a passive bar from its first to its last key under the diamonds at the theme's faint step (`VERIFY:` token). A summary row draws `spanCoverageV1` of the tracks beneath it: faint where one track spans, one step darker where two or more overlap.

**Labels.** Context menu "Label ▸" with eight named swatches and "None"; Ctrl+Alt+1 to 8 assign, Ctrl+Alt+0 clears (section 12). The colour fills the diamond in every view. A "Keys" chip in the timeline header toggles colours, "Unlabelled" and the four glyphs; keys failing the filter draw at 30 % opacity and are skipped by marquee, summary rows, Select All and J/K (spec 10). Select menu: "Select by Label ▸ Red … Pink, Any label, None" and "Select by Kind ▸ Key, Breakdown, Extreme, Hold", scoped as spec 06 defines (`VERIFY:`).

**Kinds.** Context menu "Kind ▸ Key, Breakdown, Extreme"; no shortcut. Glyphs: Key a diamond; Breakdown at 60 % size; Extreme at 120 % with a heavier outline; Hold a diamond with its right half squared, drawn whenever the segment starting at the key is a hold.

**Groups.** Context menu "Group ▸ New group…" asks a name and swatch. A "Groups" row under the composition row shows each group as a tag over `groupSpanV1` in its colour; click selects its live keys in every view (Shift adds, Alt subtracts). Right-click: Rename, Colour ▸, Add or Remove selected keys, Label keys with group colour, Delete group (keys stay). A group with no live members draws in italic as "Settle (0 keys)", hover "All of this group's keys were deleted. Undo to restore them". Spec 02's Stagger popover and spec 07's edge-handle retime offer "Items: Groups" when `groupsOfSelectionV1` is non-empty.

**Same-frame highlight.** When the playhead equals a key time exactly, every diamond there draws a lit ring, the layer list shows a dot after each keyed layer, and the status bar reads "Frame 12: 4 tracks keyed". Alt+Down / Alt+Up (section 12) select the next / previous keyed track's key at the playhead, expanding and scrolling to it: "Frame 12: Scale (2 of 4)", or "No key at the playhead".

**Guides.** Double-click the ruler to add a guide at that whole frame (section 12); Timeline menu "Add Guide…" takes a time (`12f`, `0.5s`) and a name. A guide is a vertical line through every row with a named ruler handle; drag the handle to move it (whole frames; Ctrl free; Shift snaps). Right-click: Rename, Lock, Delete, Go to guide. A locked guide shows a padlock and refuses drags and Delete with hover "Locked. Right-click to unlock". Shift+drag of keys, summary diamonds and the playhead snaps to guides. Adding at an occupied time selects that guide.

Empty and error states: no keyed track, no diamonds and no bar; a filter matching nothing shows "No keys match" on the chip; an empty group name is refused with "Give the group a name".

## 6. User flows

1. **Primary: line up a hit across three layers.** Start: three layers with Position runs; "All keys" shows diamonds at 0, 10, 12 and 14 with badges. (a) Click the diamond at 12: the keys at 12 on all three layers select. (b) Drag it: readout "14f (+2)"; on the layer already keyed at 14 the ghost is red. (c) Release: keys land on 14, the occupant replaced, undo label "Move 3 keys (replaced 1)". End: one diamond at 14 with badge 4; nothing at 12.
2. **Keyboard only: label the settle keys.** Start: playhead at 30 where Scale, Rotation and Opacity key; layer collapsed. (a) Alt+Down: the layer expands, Scale's key at 30 selects, status "Frame 30: Scale (1 of 3)". (b) Ctrl+Alt+3: the key turns Yellow, undo "Label 1 key Yellow". (c) Alt+Down, Ctrl+Alt+3, twice more. (d) Alt+Down wraps to Scale. End: three Yellow keys; the "All keys" diamond at 30 fills Yellow.
3. **Multi-selection: group an anticipation and stagger it.** Start: 10 keys at frames 4 to 8 selected on five tracks across two layers. (a) Group ▸ New group…, "Anticipation", Red: a Red tag 4 to 8 appears; undo "Group 10 keys as Anticipation". (b) Click empty space, then the tag: all 10 keys select in every view. (c) Ctrl+Alt+drag (spec 02): the Stagger popover opens with Items "Tracks", 2 frames per track. End: keys spread 4 to 16; the tag spans 4 to 16; membership unchanged.
4. **Undo: a group survives an unrelated edit.** Start: group "Hit", 6 members. (a) Change an Opacity value elsewhere. (b) Ctrl+Z: Opacity restored; "Hit" still lists 6 keys. (c) Delete 2 members: the tag shortens to 4 keys. (d) Ctrl+Z: the keys return with their ids; 6 keys. End: document as at start; selection as the user left it (Ruling 11).
5. **Limit: label shortcuts under AltGr.** Start: a German layout (`VERIFY:` detection), keys selected. (a) Ctrl+Alt+2: the browser delivers "²"; no label applies. (b) The app, seeing a printable character with Ctrl+Alt held, shows once: "Your keyboard layout uses Ctrl+Alt+1 to 8 for characters. Rebind the label shortcuts in Settings ▸ Keyboard". End: no change; the context menu still works.
6. **Guides: place, name, snap, lock, reload.** Start: an empty 24 fps composition. (a) Double-click the ruler near 24: a guide at 24, undo "Add guide at 24f". (b) Rename to "Beat 1". (c) Shift+drag a key: it snaps, readout "24f (guide Beat 1)". (d) Lock; drag the handle: nothing moves. (e) Save, reopen: the guide is at 24, named, locked.

## 7. Evaluation and determinism

This spec evaluates nothing: every value stays `evaluate(document, time)` (Ruling 9). The view functions of section 4 are pure in the document, selection, filter and playhead and compare times as exact rationals. Bakes: none here; spec 05's bake writes `kind: 'breakdown'` and no colour on baked keys, and unbake restores the source keys' labels and kinds exactly (test D4).

## 8. Edge cases and named limits

- Two sub-frame keys 1/1000 frame apart are two diamonds; the row never merges times. Named limit.
- A summary drag skips tracks on locked layers (`VERIFY:` lock model).
- A group's colour never recolours its keys unless "Label keys with group colour" is run.
- Paste and Alt-drag (spec 10) carry labels and kinds, never ids or membership. Ruling 7 replacement drops the occupant's id, so it leaves its groups.
- Guides never export, two cannot share a time, and they stay put under spec 08's ripple retime.
- "Kind ▸ Hold" does not exist; set the ease to Hold (spec 01).
- The filter hides keys from J/K, marquee and Select All, never from the evaluator or from drags of keys already selected.

## 9. Interactions with other specs

01: a hold ease gives the hold glyph. 02 and 07: `groupsOfSelectionV1` supplies "Items: Groups"; summary drags call 07's move command; guides join 07's snap targets. 04: the graph editor draws `keyGlyphV1`, label colours and guides. 05: bake and unbake as in section 7. 06: Select menu entries and the filter chip. 10: the payload carries `label`; J/K honour the filter; duplicates carry no id.

## 10. Rules and tests

Rules, each a test: R1 a summary diamond at 12 exists iff a track keys at 12. R2 a summary drag moves every key at that time, eases unchanged. R3 a click selects exactly those keys. R4 a label filter selects only labelled keys. R5 a group survives undo of an unrelated edit, a move, and a member's deletion and undo. R6 the overlap region equals the intersection of the two span bars. R7 guides are stored and reloaded. R8 kinds change the glyph only. R9 the hold glyph follows the ease. R10 same-frame uses exact equality. R11 the filter hides keys from J/K, marquee and Select All.

Tests by name (`summary-row.test.ts`, `key-labels.test.ts`, `key-groups.test.ts`, `time-guides.test.ts`):
- T1 `summaryRowV1: a diamond at 12 exists iff a track keys at 12`: tracks keyed {0, 12}, {12, 24}, {5} at 24 fps give entries 0, 5, 12, 24 with two trackIds at 12; delete both keys at 12: no entry. T2 `badge and colour`: two Red keys at 12 → colour 1; one Red, one unlabelled → null.
- T3 `moveSummaryDiamondV1 moves every key at 12 by 2 frames, eases unchanged`: Position (bezier, free) and Opacity (hold) land at exactly 14/24, eases and tangents deep-equal, the Opacity occupant at 14 gone, label "Move 2 keys (replaced 1)". T4 `undo of T3 restores the replaced key`.
- T5 `select by label selects only labelled keys`: 3 Red, 2 Blue, 5 unlabelled; Red → 3; Red and Blue → 5; None → 5.
- T6 `a group survives undo of an unrelated edit`: 6 members; set a value on another track; undo; members 6, ids equal. T7 `membership survives a nudge and a stagger`: +3 frames; `liveMembersV1` lists 6 at the shifted exact times. T8 `a deleted key leaves the group; undo returns it`: delete 2 → 4 live; delete all → `[]`; undo → 6.
- T9 `spanCoverageV1: the overlap equals the intersection`: Position 10..40 and Scale 30..60 → count 2 exactly on [30, 40]; disjoint bars → no count 2; identical bars → the whole span.
- T11 `guides are stored and reloaded`: guides at 24/24 "Beat 1" and 37/24 "Beat 2" locked; serialise, parse → deep-equal including `n`, `d`, `locked`; an old document loads `[]`. T12 `guides are unique whole frames`: add at 24.4 frames → 24/24; add at 24 again → the existing id.
- T13 `keyGlyphV1`: hold ease on a non-last key → 'hold' for every kind; last key → its kind. T14 `setKeyKindV1 leaves evaluation unchanged`: frames 0..48, `Object.is` per channel. T15 `tracksKeyedAtV1 is exact`: a key at 1/3 s is listed at playhead 8/24, not at 333333/1000000.
- T16 `the filter hides keys from navigation and marquee`: filter Red; `nextKeyTimeV1` (spec 10) and a marquee skip unlabelled keys.
- D1 `play equals seek`, D2 `random order`, D3 `after an edit` (move a summary diamond, undo), D4 `bake round-trip` (spec 05's bake of a labelled track: baked keys are Breakdown with no colour; unbake restores labels and kinds exactly), on a fixture with labels, kinds, two groups and three guides.

## 11. VERIFY list

1. The timeline module path and whether its row renderer can draw a summary row.
2. Theme tokens: faint and overlap steps, eight label colours, the lit ring.
3. The marker model and whether markers export.
4. Where per-composition editor data lives; the save-format version (00 §8 item 6).
5. Whether `KeyframeV1` can take an optional `id`.
6. The keymap for Ctrl+Alt+digits, Alt+Up/Down and ruler double-click; AltGr detection.
7. Spec 06's Select menu scope; spec 07's move command and snap-target list; the layer lock model; spec 05's bake hook for `kind`.

## 12. Open questions for the user

1. Proposed addition to 00 §4, keyboard: "Ctrl+Alt+1 to 8: label the selected keys; Ctrl+Alt+0 clears (09)". AltGr layouts type characters on these keys; the context menu is the fallback.
2. Proposed addition to 00 §4, keyboard: "Alt+Up / Alt+Down: previous / next track keyed at the playhead (09)".
3. Proposed addition to 00 §4, gestures: "Double-click the ruler: add a time guide (09)"; add guides to the Shift+drag row's snap targets (07, 09).
4. Proposed addition to 00 §3: `readonly id?: string` on `KeyframeV1` (09-R3); the alternative is rewriting group references in every move command.
5. Confirm: per-layer summary rows when expanded default on; no kind shortcut; guides stay put under ripple retime.
