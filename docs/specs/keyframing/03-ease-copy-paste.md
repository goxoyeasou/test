# Spec 03: copy and paste an ease independently of values

Written without the repository open; `VERIFY:` marks every assumption about unseen code (section 11). Report 1 is `reports/Smooth keyframing for WebGL motion graphics.md`, report 2 is `reports/AE keyframing hacks and plugins.md`, as in 00. Depends on spec 01 (the popover, `targetSegmentsV1`, `setSegmentEaseV1`, `parseCubicBezierV1`, `formatCubicBezierV1`).

## 1. Purpose and evidence

After Effects cannot copy an ease without its value, so EaseCopy exists, appears in every free-script roundup, and a request to bundle it was answered "Not going to happen" (report 2 §2, §6 rank 3). With the ease owned by the segment (00 Ruling 2) and stored as normalised handles (00 Ruling 1), an ease is four numbers or a few parameters that paste anywhere, and the same object round-trips as a CSS `cubic-bezier()` string with cubic-bezier.com, Figma, Flow and Scenery Curves (report 1 §2; report 2 §2). This spec ships EaseCopy's count and pass-through rules and the Easings Manager's three pastes (eases only, values only, both) as built-in commands.

## 2. Scope and non-goals

In scope: Copy Ease; Paste Ease (Ctrl+Alt+V) onto selected segments keeping values; Paste Values Only (Ctrl+Alt+Shift+V) onto selected keys keeping eases; the count rules with pass-through; the mirror-in-time option; pasting across property types and onto hold segments; the clipboard JSON payload and its text fallback; parsing of external `cubic-bezier()` strings; the right-click entries and the popover's Copy and Paste buttons; undo granularity.

Non-goals: plain paste of keys at the playhead and Paste Reversed (spec 10); the popover's controls and the library (spec 01); pasting eases onto drivers (spec 05); any scaling of handles (03-R2 says why none is needed).

## 3. Rulings

- **03-R1 (count rules, EaseCopy's with pass-through defined):** with `S` source eases and `T` target segments in order: `S = 1` pastes to every target; `T = k·S` repeats the sources `k` times in order; `S ≥ 3` and `T > S` (not a multiple) is pass-through: the first target gets the first ease, the last target the last ease, and interior target `j` (0-based, `1 ≤ j ≤ T − 2`) gets source `1 + floor((j − 1)·(S − 2)/(T − 2))`; anything else is refused with a message naming both counts. Why: report 2 §2 (the same count or a multiple; a pass-through option for strips of three or more); keeping the departure and arrival eases and spreading the interior ones is what a strip's "pass-through velocity" means in the segment model. Cost if wrong: one mapping function.
- **03-R2 (no scaling):** handles are relative to the target's own `Δv` and `Δt`, so a pasted ease is copied bit-exactly; the handle graph looks the same on the target and only the absolute slope in the graph editor differs; spring, elastic and bounce parameters are shares of the segment (spec 01 01-R4) and copy unchanged. Why: 00 Ruling 1; EaseCopy scales because AE stores speeds in value per second (report 1 §1). Cost if wrong: nothing to remove.
- **03-R3 (direction needs no flip; mirror in time is a separate option):** an ease-out pasted from a rising segment onto a falling one is still an ease-out, because `y` is a fraction of the target's own `Δv`; EaseCopy's flip exists only to re-sign AE's speeds. The brief's point reflection `(x1, y1, x2, y2) → (1 − x2, 1 − y2, 1 − x1, 1 − y1)` is a time reversal: it turns an ease-out into an ease-in. It is therefore offered as "Mirror in time", off by default and independent of direction, for pasting an entrance's ease onto an exit that should mirror it; it applies to bezier eases only until the direction field of spec 01 §12 lands. Why: a direction-triggered reversal would change the shape class exactly when the user expects it kept (a stagger of bars rising and falling with one ease). Cost if wrong: one flag's default; section 12 asks for confirmation.
- **03-R4 (one clipboard, three pastes):** Copy Ease writes an `EaseClipboardV1` JSON payload plus a text fallback of one line per ease; Paste Ease also reads spec 10's key payload (the eases of the segments starting at the copied keys), and Paste Values Only reads that payload's values; so Ctrl+C then Ctrl+Alt+V works without a separate copy, as in the Easings Manager (report 2 §2). Why: one mental model; the text fallback makes cubic-bezier.com and Figma a two-way street. Cost if wrong: one reader function.
- **03-R5 (undo):** one paste is one undo step however many segments or keys it touches; a refused paste and a paste that changes nothing add no step; Copy Ease touches no document and is not undoable. Why: 00 Ruling 11 and the popover pattern. Cost if wrong: a stack-grouping call.

## 4. Model and interfaces

```ts
// src/animation-core/keyframes/ease-clipboard.ts   VERIFY: folder; EaseV1, KeyframeV1 from 00 §3; SegmentRefV1 from spec 01
export interface EaseClipboardV1 { readonly format: 'eases'; readonly version: 1; readonly eases: readonly EaseV1[] }
export const EASE_CLIPBOARD_TYPE_V1 = 'web application/x-motion-eases+json'   // VERIFY: Chrome needs the 'web ' prefix; Electron takes any name
export const collectSourceEasesV1 = (doc: DocumentV1, selection: KeySelectionV1): readonly EaseV1[]   // targetSegmentsV1 (spec 01) mapped to eases
export const easesFromKeyPayloadV1 = (payload: KeyClipboardV1): readonly EaseV1[]                      // VERIFY: spec 10's payload type
export const serialiseEaseClipboardV1 = (eases: readonly EaseV1[]): { readonly json: string; readonly text: string }
export const formatEaseTextV1 = (ease: EaseV1): string
export const parseEaseTextV1 = (text: string): { ok: true; eases: readonly EaseV1[] } | { ok: false; line: number; input: string; reason: string }
export type EaseAssignmentV1 =
  | { ok: true; mode: 'one' | 'repeat' | 'pass-through'; pairs: readonly { target: SegmentRefV1; ease: EaseV1 }[] }
  | { ok: false; reason: string }
export const assignEasesV1 = (sources: readonly EaseV1[], targets: readonly SegmentRefV1[], forcePassThrough?: boolean): EaseAssignmentV1   // 03-R1
export const mirrorEaseInTimeV1 = (ease: EaseV1): EaseV1   // bezier: (1 − x2, 1 − y2, 1 − x1, 1 − y1); other types unchanged
export const pasteEasesV1 = (doc: DocumentV1, assignment: EaseAssignmentV1 & { ok: true }, mirror: boolean): DocumentV1   // setSegmentEaseV1 per pair
export const pasteValuesOnlyV1 = <V>(doc: DocumentV1, targets: readonly KeyRefV1[], values: readonly V[]): DocumentV1 | { refused: string }
```

**Text fallback grammar.** One ease per non-empty line; keywords are case-insensitive; `n` is a JavaScript number, optionally with `%` (divided by 100):

```
line       := bezier | keyword | numbers | procedural | url
bezier     := 'cubic-bezier(' n ',' n ',' n ',' n ')'        anywhere in the line, so "transition: all 0.3s cubic-bezier(…);" parses
keyword    := 'linear' | 'ease' | 'ease-in' | 'ease-out' | 'ease-in-out' | 'hold' | 'step-end'   (step-end is hold)
numbers    := n (',' | ws) n (',' | ws) n (',' | ws) n
procedural := 'spring(' n ',' n ')' | 'elastic(' n ',' n ')' | 'bounce(' int ',' n ')'
url        := 'https://cubic-bezier.com/#' n ',' n ',' n ',' n
```

`formatEaseTextV1` writes `cubic-bezier(x1, y1, x2, y2)` through `formatCubicBezierV1(e, 'exact')` (JS shortest round-trip, so `parse(format(e))` is bit-equal), `hold`, `spring(1, 0.3)`, `elastic(1, 0.3)`, `bounce(3, 0.5)`. Parsing validates every number as finite, bezier `x` in `[0, 1]`, and procedural parameters inside spec 01's `EASE_RANGES_V1`; the first bad line refuses the whole paste with its line number, the offending text and the reason. A JSON payload with `version > 1` is refused with "Copied from a newer version".

**Order of sources and targets.** Both come from `targetSegmentsV1` (spec 01): selected segments plus the segment starting at each selected key that has a later key, ordered by track order then time. Pasting onto keys therefore maps the `n`th copied key's outgoing ease to the `n`th selected key's outgoing ease, the per-key model EaseCopy users expect, and a single selected key copies or receives the ease of the segment starting at it.

## 5. UX

**Commands.** Copy Ease, Paste Ease and Paste Values Only appear in the right-click menu of a segment bar and of a key (00 §4 lists Paste Ease on the bar; the other entries are proposed in section 12), in the Edit menu, and as the Copy and Paste buttons of the Ease popover (spec 01). Paste Ease carries a submenu: Paste Ease, Paste Ease mirrored in time, Paste Ease pass-through. Keyboard: Ctrl+Alt+V [Cmd+Option+V] pastes eases with the defaults (no mirror, automatic mode); Ctrl+Alt+Shift+V pastes values only; Ctrl+C copies keys (spec 10) and feeds both (03-R4). The entries are disabled with hover text "Copy an ease first" when the clipboard holds neither a payload nor parseable text; the shortcut then shows "Nothing to paste: copy an ease, or a cubic-bezier string".

**Feedback.** A status line after each paste: "Pasted 1 ease to 3 segments", "Pasted 2 eases to 4 segments, repeated", "Pasted 3 eases to 5 segments, pass-through", "Pasted 1 ease, mirrored in time". A refusal is an inline message at the selection, not a modal: "Copied 2 eases; 3 segments selected. Select 2, or a multiple of 2." or "Line 2 'cubic-bezier(1.2, 0, 0, 1)': x values must be between 0 and 1". Target bars flash once and their glyphs change.

**Mirror in time.** In the submenu and as a checkbox in the popover's Paste menu, remembered for the session, off at launch. When it is on and a source is procedural the status adds "spring eases are not mirrored".

**Paste Values Only.** Targets are the selected keys in `targetSegmentsV1` order (keys, so the last key counts); sources are the copied keys' values; count rules as 03-R1; a value type mismatch (2-D onto scalar) refuses with "Copied Position values; Opacity needs a single number". Times, eases and tangent modes are untouched; auto segments reflow through `refreshAutoEasesV1`.

**Selection.** Never changed by any of the three commands (00 Ruling 11).

## 6. User flows

1. **Primary: one ease onto three segments.** Start: a tuned Scale segment on layer A; three Position segments on layers B, C and D selected. Right-click A's bar, Copy Ease: nothing visible changes. Right-click one selected bar, Paste Ease: the three bars flash, their glyphs change, status "Pasted 1 ease to 3 segments"; B, C and D still start and end where they did and move along their own paths with A's timing. End: four segments share one `EaseV1` by value; one undo step.
2. **Keyboard only, from a website.** Start: `cubic-bezier(0.17, 0.67, 0.83, 0.67)` copied from cubic-bezier.com into the system clipboard; four Opacity segments selected, timeline focused. Press Ctrl+Alt+V: the text parses, the four bars flash, status "Pasted 1 ease to 4 segments". Press Ctrl+Shift+E: the popover's field shows `cubic-bezier(0.17, 0.67, 0.83, 0.67)`. Escape. End: four identical beziers, one undo step.
3. **Multi-selection, mixed types and a hold.** Start: two Opacity segments copied (ease-out, then a spring). Select four segments: Scale (2-D), Rotation, Position and a hold on Opacity. Press Ctrl+Alt+V: status "Pasted 2 eases to 4 segments, repeated"; Scale gets ease-out on both channels, Rotation the spring, Position ease-out through arc length, the hold becomes the spring. End: four segments eased, every key value unchanged, both keys of each `free`.
4. **Undo.** Start: flow 3's end. Press Ctrl+Z: the four eases return, the Opacity segment is a hold again, tangent modes return, selection and scroll unchanged. Redo (`VERIFY:` the keymap) re-applies all four. End: flow 3's end; selection never moved.
5. **Limit: counts.** Start: two eases copied; three segments selected. Press Ctrl+Alt+V: refused, inline "Copied 2 eases; 3 segments selected. Select 2, or a multiple of 2.", no undo step, nothing flashes. Select a fourth segment, Ctrl+Alt+V: "Pasted 2 eases to 4 segments, repeated". Then copy a strip of four keys on one track (eases A, B, C) and paste onto five segments: "pass-through", the targets get A, B, B, B, C. End: as stated.
6. **Values only.** Start: three Scale keys copied from layer A (Ctrl+C); three Scale keys on layer B selected, with their own eases. Press Ctrl+Alt+Shift+V: B's keys take A's values in order and keep their times and eases; status "Pasted 3 values to 3 keys". End: B's timing and feel unchanged, poses from A; one undo step.
7. **Out to Figma.** Start: a segment with `(0.2, 0, 0, 1)` selected. Copy Ease; paste into Figma's easing field: the text fallback `cubic-bezier(0.2, 0, 0, 1)` lands. End: the same curve in Figma.

## 7. Evaluation and determinism

This spec evaluates nothing new. A paste is a pure document transform: `pasteEasesV1` returns a new document in which the target keys carry the pasted `EaseV1` by value and `tangent: 'free'`; `pasteValuesOnlyV1` returns one with new values and the same eases, then `refreshAutoEasesV1`. The evaluator reads the result through spec 01's `easeFractionV1`, so play, seek and export agree by construction (00 Ruling 9). Nothing bakes; a pasted spring is baked on Lottie export by spec 05 like any other. Ruling 9 statement: the pasted document is a pure function of time exactly as the source document was.

## 8. Edge cases and named limits

- A selected key that is the last on its track has no outgoing segment: it is skipped as a source and as a target; if that leaves no targets the message is "The selected keys have no segment after them".
- Targets on hold-only properties (booleans, enums, visibility) are refused by name, "Visibility can only hold" (00 Ruling 6); a pasted `hold` onto any property is allowed.
- Pasting onto the segment the ease came from, or an ease bit-equal to the existing one: no change, no undo step (03-R5).
- Equal key values on a target: the ease is stored and moves nothing (spec 01 §8).
- Mirror of a symmetric ease (`ease-in-out`, Easy Ease) is the identity; the status still says "mirrored".
- Text with several `cubic-bezier()` on one line: the first is taken; blank lines and `;` are ignored; a bare fragment `#.17,.67,.83,.67` parses as numbers.
- `T` a multiple of `S` with `S ≥ 3`: repeat wins; the submenu's pass-through entry forces it (`forcePassThrough`); with `T < S` the paste is refused.
- Named limit: pass-through spreads the interior eases by index, not by segment duration; a strip whose middle segments differ widely in length may want a manual pass.
- Named limit: Mirror in time does not apply to spring, elastic or bounce (03-R3).
- A clipboard item with the JSON type but invalid content is refused as "Not an ease"; the text fallback is then tried.

## 9. Interactions with other specs

- 01: the popover's Copy and Paste buttons; `targetSegmentsV1`, `setSegmentEaseV1`, `parseCubicBezierV1`, `formatCubicBezierV1`, `EASE_RANGES_V1`.
- 10: Ctrl+C's key payload is read by Paste Ease and Paste Values Only; plain Ctrl+V pastes both at the playhead and is unchanged; Paste Reversed uses `mirrorEaseInTimeV1`.
- 06: selection order defines source and target order; no command changes selection.
- 04: the graph editor's context menu carries the same three entries.
- 05: procedural eases paste as parameters and bake only on export.
- 02: the usual sequence is stagger, then select the run's segments, then paste one ease.
- 07 and 08: pasted eases survive retime and loops unchanged (normalised form).

## 10. Rules and tests

Each rule is one test in `ease-clipboard.test.ts` (D1 to D4 in `ease-clipboard.determinism.test.ts`); the test name follows the rule number.
- R1, T1 `paste keeps every target value bit-exactly`: a scalar track with 20 random double values and a 2-D track; after `pasteEasesV1` every key's value and time is `Object.is`-equal per channel to before; only `ease` and `tangent` changed.
- R2, T2 `mirror in time is an involution and swaps ease-out for ease-in`: `mirrorEaseInTimeV1(mirrorEaseInTimeV1(e))` equals `e` by `Object.is` per field on the 21 × 21 handle grid with `y1 ∈ {−0.6, 0, 1.56}`; ease-out maps to `(0.42, 0, 1, 1)`.
- R3, T3 `an ease-out stays an ease-out across a direction change`: ease-out pasted from `0 → 100` onto `100 → 0` stores bit-equal handles, and at `u = 0.5` the target has covered more than half its delta (`f ≈ 0.68`; an ease-in would give 0.32).
- R4, T4 `count rules 1→3, 2→4, 2→3 refused, 3→5 pass-through`: `1 → 3` gives three copies; `2 → 4` gives `[a, b, a, b]`, mode `repeat`; `2 → 3` is refused with "Copied 2 eases; 3 segments selected. Select 2, or a multiple of 2."; `3 → 5` gives `[a, b, b, b, c]`, mode `pass-through`; `4 → 7` gives `[a, b, b, b, c, c, d]`; `3 → 6` is `repeat`; `3 → 2` is refused.
- R5, T5 `CSS text round-trips`: `parseEaseTextV1(formatEaseTextV1(e))` is bit-equal for every preset of spec 01, 1,000 random beziers (`y` in `[−2, 3]`) and the three procedural defaults; multi-line text yields the list in order.
- R6, T6 `text parsing accepts the six syntaxes and refuses with reasons`: `cubic-bezier(.17,.67,.83,.67)`, `0.17 0.67 0.83 0.67`, `ease-in-out`, `step-end`, `transition: all 0.3s cubic-bezier(0.2, 0, 0, 1);`, `https://cubic-bezier.com/#.17,.67,.83,.67` and `spring(100%, 30%)` parse; `cubic-bezier(1.2, 0, 0, 1)` refuses with line 1 and the x reason; `bounce(0, 0.5)` refuses with the range reason.
- R7, T7 `one ease across Scale, Rotation and Position`: one Opacity ease pasted onto Scale (2-D) and Rotation evaluates at `u = 0.5` to `v0 + (v1 − v0)·f` on each channel; onto Position it moves the point to arc-length fraction `f`.
- R8, T8 `hold and spring conversions`: a bezier onto a hold gives type `bezier` with both keys `free`; a `hold` onto a bezier converts the other way; a spring pastes onto Opacity with `Object.is` parameters.
- R9, T9 `hold-only properties and last keys`: paste onto Visibility is refused by name; the last key is skipped as a target.
- R10, T10 `one undo step per paste`: a `2 → 4` paste is one step; undo restores eases and tangent modes by deep equality; a refused paste and a no-change paste leave the stack length unchanged.
- R11, T11 `eases from the key clipboard`: Ctrl+C of three consecutive keys then Paste Ease yields three eases (two between them, then the third key's outgoing ease) in order; one copied key yields the ease of its segment.
- R12, T12 `Paste Values Only`: times, eases and tangents kept by `Object.is`; a 2-D onto scalar is refused with the message.
- D1, T13 `play equals seek`: a fixture after flow 3's paste, frames 0..120 in order, then 120 fresh; `Object.is` per channel.
- D2, T14 `random order`: a shuffled frame list equals D1.
- D3, T15 `after an edit`: paste, undo, evaluate; equal to before the paste.
- D4, T16 `bake round-trip`: a pasted spring and bounce through spec 05's `bakeTrackV1` evaluate within spec 05's tolerance at every frame (`VERIFY:` the figure); unbake restores the pasted parameters by `Object.is`.

## 11. VERIFY list

1. The clipboard API: Electron's `clipboard.writeBuffer` for the custom type; the browser's `navigator.clipboard.write` with the `web ` prefix (Chrome 104 and later); the fallback to `text/plain` only where custom types are unavailable.
2. Spec 10's key payload type and clipboard type name (`KeyClipboardV1` above is a placeholder).
3. The timeline's track order API used by `targetSegmentsV1`.
4. The keymap: Ctrl+Alt+V and Ctrl+Alt+Shift+V are free on both platforms; redo; the Mac modifier names.
5. The inline message and status line components.
6. The undo stack's grouping call for one step per paste.
7. The Edit menu structure, to place the three commands.

## 12. Open questions for the user

1. The brief asked for "Mirror for direction", on by default, applying the point reflection when the target's delta sign differs. In the normalised model that reflection is a time reversal and would turn an ease-out into an ease-in exactly when the user expects it kept (03-R3). This spec ships "Mirror in time", off by default and direction-independent. Confirm, or ask for the direction-triggered version with the shape change accepted.
2. When `T` is a multiple of `S` and `S ≥ 3`, repeat wins and pass-through is a submenu choice. Confirm.
3. Proposed addition to 00 §4: the right-click menu of a segment bar gains "Copy Ease" and "Paste Values Only"; the right-click menu of a key gains "Copy Ease", "Paste Ease" and "Paste Values Only" (acting on the segment starting at the key); Paste Ease carries the submenu "mirrored in time" and "pass-through".
4. Should the text fallback for a procedural ease be the app's own `spring(…)` syntax (this spec) or a bezier approximation for cubic-bezier.com's benefit?
