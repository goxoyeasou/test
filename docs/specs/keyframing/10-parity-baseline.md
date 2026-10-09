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
