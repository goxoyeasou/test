# Keyframing specs: overview and shared model

Written without the repository open. Every assumption about unseen code is marked `VERIFY:` and gathered in section 8. Text a person sees in the app is written for a motion designer. The evidence behind each feature is in `reports/Smooth keyframing for WebGL motion graphics.md` (methods, UX patterns, architecture) and `reports/AE keyframing hacks and plugins.md` (the After Effects workarounds and plug-ins that prove demand, ranked). The specs do not repeat the evidence; they cite the report section that carries it.

## 0. The ten specs

| Spec | Capability | Rank by demand | Depends on |
| --- | --- | --- | --- |
| `01-segment-easing.md` | Per-segment ease with popover, presets, library, overshoot/elastic/bounce/spring types | 1 | 00 |
| `02-stagger-offset-sequence.md` | Stagger, offset and sequence across layers and within one track; Time Offset driver | 2 | 00, 06, 07 |
| `03-ease-copy-paste.md` | Copy and paste an ease independently of values, scaled and flipped | 3 | 01 |
| `04-graph-editor.md` | Value graph with handles on every channel, no Separate Dimensions trap, selection persistence | 4 | 01, 06 |
| `05-settle-follow-wiggle-drivers.md` | Settle (overshoot, bounce, spring), Follow and Wiggle drivers; the bake service | 5 | 00, 01 |
| `06-selection-and-bulk-editing.md` | Scope-independent selection, absolute versus offset entry, proportional scrubbing, selection-faithful reverse | 6 | 00 |
| `07-frame-snapping-and-key-dragging.md` | Whole-frame snapping, the drag and modifier model, Snap All Keys | 7 | 00, 06 |
| `08-retime-loops-constant-speed.md` | Ripple retime, retime markers, loop modes, Constant Speed | 8 | 07, 01 |
| `09-labels-groups-dope-sheet.md` | Keyframe colours and groups, per-layer summary row, same-frame highlight, guides | 9 | 06 |
| `10-parity-baseline.md` | Multi-layer copy, cut, paste and Paste Reversed; scoped key navigation; Alt-drag duplicate | 10 to 12 | 06, 07 |

Build order: 06, 07, 10 (foundations: selection, dragging, paste); then 01, 03, 04 (ease); then 02, 08 (timing); then 05 (drivers and bake); then 09 (views). Each spec stands alone for its implementer, but the rulings here bind all of them.

## 1. Rulings

- **Ruling 1 (one curve model):** every animatable scalar is a piecewise cubic Bézier in (time, value) whose handle times are clamped inside the segment. Normalised handles `(x1, y1, x2, y2)` with `x ∈ [0, 1]` and `y` unbounded are the stored form; the absolute form the graph editor draws is derived. Why: report 1 §1 (every surveyed tool has landed here; the x-clamp makes `x(t)` monotone so the solve has one root). Cost if wrong: one conversion function.
- **Ruling 2 (ease belongs to the segment):** the ease is a property of the pair of keys it joins, stored on the earlier key as the ease of the segment that starts there. A key does not own two half-tangents. Why: report 1 §2 (every AE complaint cluster and ease plug-in traces to half-tangents on keys); presets, copy and paste and vector properties all need one object per segment. Cost if wrong: the whole ease UX; this ruling is not revisited.
- **Ruling 3 (one ease per segment for a vector):** Position, Scale, Anchor and every other vector property have one ease per segment shared by all channels. The graph editor shows the channels as separate curves that share timing. A user who needs independent timing per channel splits the property into per-channel tracks with an explicit, reversible command (spec 04), never by a hidden mode. Why: report 1 §2 (the Separate Dimensions trap). Cost if wrong: a split command that is used more than expected.
- **Ruling 4 (one solver):** `x → t` is solved by the fixed-iteration solver of bezier-easing 2.1.0 (11-sample table, up to 4 Newton steps when the slope is at least 0.001, else bisection to 1e-7 within 10 steps), the same code path for play, seek, export, the popover preview and the graph. Why: report 1 §1 (the closed form needed a double-root fix in October 2026; two solvers break play = seek = export). Cost if wrong: a solver swap behind one function.
- **Ruling 5 (monotone auto tangents by default):** a new key's tangent mode is `auto`: the slope at the key is the time-weighted three-point formula `ẋᵢ = (Δᵢ·vᵢ₋₁ + Δᵢ₋₁·vᵢ) / (Δᵢ₋₁ + Δᵢ)`, clamped by Steffen's monotone rule so a key that is a local extremum gets a flat tangent and the curve never overshoots a keyed value. A segment whose ease the user edits (popover, preset, paste or handle drag) becomes `free` at both ends. C2 "continuous acceleration" is a per-track opt-in, never the default. Why: report 1 §1 (AE's documented "boomerang", Blender's Auto Clamped, Maya's Auto). Cost if wrong: one tangent function and one flag.
- **Ruling 6 (hold is a time rule):** a hold ease keeps the earlier key's value until the next key, bypasses the solve, and terminates any auto-tangent run at both ends. Booleans, enums and visibility have hold only. Why: report 1 §1. Cost if wrong: nothing; this is universal.
- **Ruling 7 (one key per frame per track):** two keys of one track never share a time. Dropping, pasting or retiming a key onto an occupied time replaces the occupant (the moved key wins; one undo step restores it). An instant cut is a hold segment followed by the next key. Importers that meet two keys at one time (Lottie permits this as a jump) place the second at the next frame and report it. Named limit: an ease that arrives at a value and cuts away in the same frame is not representable; the cut lands one frame later. Why: AE and Blender have the same rule, so no migrating user expects otherwise; a "jump key" with an arrive and a leave value would touch every surface. Cost if wrong: a `leaveValue` field added later, with the same frame-later import rule kept for old documents.
- **Ruling 8 (whole frames by default):** every key time produced by a drag, nudge, retime, stagger, paste or bake is a whole frame of the composition rate, as the exact rational `n / fps`. Sub-frame placement exists only through an explicit modifier or a document setting (spec 07). Why: report 2 §5 and §6 (AE's sub-frame keys after Alt-drag are a standing complaint; Blender and Maya snap). Cost if wrong: one snapping function with one bypass.
- **Ruling 9 (pure function of time):** every value the app shows is `evaluate(document, time)` with no state carried between frames. Eases, holds, loops, offsets, Settle, Follow and Wiggle are closed-form in `t` or read the track at another exact time through the same evaluator. Anything that cannot be written that way (a simulation, a spring chasing a moving keyed target, recorded or generated motion) exists in the document only as baked keys. Why: the existing play = seek = export invariant (`docs/plans/effects-batch-b-hard-tasks.md`, Ruling 2 there) and report 1 §3 and §4. Cost if wrong: not accepted; a feature that cannot meet this ruling is cut or baked.
- **Ruling 10 (position is a path plus a timing curve):** a Position track stores a spatial Bézier path through the keyed points (auto tangents through the keys, editable on the canvas) and one scalar timing curve per segment (Ruling 2). The timing curve's output is an arc-length fraction mapped through a fixed 150-sample table per path segment. Retiming never reshapes the path; reshaping never retimes. Why: report 1 §1 (the AE and Lottie model). Cost if wrong: one table size constant.
- **Ruling 11 (selection, pan and zoom are not undoable):** undo and redo move the document only. Selection, scroll, zoom, the open popover and the graph's framing survive undo, view switches and playback. Why: report 1 §2 (the state-loss complaints in AE and Rive). Cost if wrong: nothing; this is a rule about what the undo stack records.
- **Ruling 12 (drivers are a stack on a track):** a track's evaluated value passes through an ordered list of drivers, each a pure function `(t, trackValueAt, context) → value`. Drivers may read their own track or another track at any exact time through `trackValueAt`; the evaluator rejects a cycle at edit time. Spec 02 adds Time Offset; spec 05 adds Settle, Follow and Wiggle and the bake service. Why: report 2 §4 (every AE expression wrapper is one of these); Ruling 9. Cost if wrong: one interface. `VERIFY:` the existing "wiggle driver" named in the effects plan is already shaped like this and becomes the first entry in the list.
- **Ruling 13 (one modifier map):** the drag and keyboard modifiers in section 4 are reserved across all ten specs. A spec that needs a gesture takes it from the map or adds a row here; no spec defines a modifier privately. Why: ten features designed apart will collide on Alt and Shift. Cost if wrong: an afternoon of rebinding.

## 2. Glossary

- **Track**: one animatable property of one node, keyed as `${nodeId}/${propertyId}` (`VERIFY:` the overlay's key format in `docs/plans/effects-batch-b-hard-tasks.md` B9 §1). A track holds an ordered list of keys, a tangent-mode flag per key, an ease per segment, an extrapolation rule for each end, and a driver stack.
- **Key**: a time (exact rational) and a value. Never two at one time on a track (Ruling 7).
- **Segment**: the interval between two adjacent keys of one track. Owns exactly one ease.
- **Ease**: how the segment moves from its earlier value to its later value: `bezier`, `hold`, `spring`, `elastic`, `bounce` (spec 01). `linear` is the bezier `(0, 0, 1, 1)` and is shown by name.
- **Run**: a contiguous group of selected keys on one track. Stagger and retime treat a run as one item when asked to (spec 02, 07).
- **Driver**: a pure function layered on a track's evaluated value (Ruling 12).
- **Bake**: replacing a driver, a procedural ease or a loop with keys and bezier eases fitted to its samples, keeping the source parameters so it can be unbaked (spec 05 §bake).
- **Ease library**: the user's named eases, shareable as a file (spec 01).
- **Playhead**: the current time; **work area**: the preview range; both exact rationals.

## 3. Shared interfaces

Names are new unless marked `VERIFY:`. Module paths follow the effects plan's layout (`src/animation-core/…`); `VERIFY:` the real folder and the existing key, track and overlay types, and adapt the names below to them rather than duplicating.

```ts
// src/animation-core/keyframes/keyframe-model.ts
export type ExactTime = { readonly n: bigint; readonly d: bigint }      // VERIFY: the existing ExactTime shape and helpers (subtractExactTime, scaleExactTime, serializeExactTime, exactSecondsV1, frameDuration)

export type TangentModeV1 = 'auto' | 'free'
export interface KeyframeV1<V> {
  readonly time: ExactTime
  readonly value: V
  readonly tangent: TangentModeV1                 // Ruling 5; a hold on either side forces 'free' at that side
  readonly ease: EaseV1                           // the ease of the segment starting at this key; ignored on the last key
  readonly label?: KeyLabelV1                     // spec 09
  readonly kind?: 'key' | 'breakdown' | 'extreme' // spec 09; glyph and filters only, never evaluation
  readonly id?: string                            // spec 09; stable id assigned lazily for group membership
  readonly join?: 'joined' | 'broken'             // spec 04; editor-only: handles either side of the key stay mirrored; default 'joined'
  readonly settle?: 'skip'                        // spec 05; per-key opt-out of Settle behaviours
}

export type EaseV1 =
  | { readonly type: 'bezier'; readonly x1: number; readonly y1: number; readonly x2: number; readonly y2: number }   // x in [0,1], y free
  | { readonly type: 'hold' }
  | { readonly type: 'spring'; readonly duration: number; readonly bounce: number }       // Apple form; spec 01 §springs
  | { readonly type: 'elastic'; readonly amplitude: number; readonly period: number; readonly direction?: EaseDirectionV1 }     // Rive/Penner form
  | { readonly type: 'bounce'; readonly bounces: number; readonly restitution: number; readonly direction?: EaseDirectionV1 }
export type EaseDirectionV1 = 'in' | 'out' | 'in-out'      // default 'out'; spec 01

export const LINEAR_EASE_V1: EaseV1 = { type: 'bezier', x1: 0, y1: 0, x2: 1, y2: 1 }
export const DEFAULT_EASE_V1: EaseV1 = { type: 'bezier', x1: 0.2, y1: 0, x2: 0, y2: 1 }   // Material "standard"; spec 01 Ruling

export type ExtrapolationV1 = 'constant' | 'linear' | 'cycle' | 'ping-pong' | 'offset' | 'continue'   // spec 08

export interface TrackV1<V> {
  readonly keys: readonly KeyframeV1<V>[]        // sorted by time, unique times (Ruling 7)
  readonly before: ExtrapolationV1
  readonly after: ExtrapolationV1
  readonly drivers: readonly DriverV1[]          // Ruling 12
  readonly c2: boolean                            // Ruling 5 opt-in
  readonly bakes?: readonly BakeRecordV1[]        // spec 05; retained sources so Unbake restores them exactly
}

// Evaluation (pure; Ruling 9)
export const evaluateTrackV1 = <V>(track: TrackV1<V>, time: ExactTime, ctx: EvalContextV1): V
export const easeFractionV1 = (ease: EaseV1, u: number): number       // u in [0,1] → fraction, may leave [0,1] for overshoot
export const solveBezierXV1 = (x1: number, x2: number, x: number): number   // Ruling 4; bezier-easing 2.1.0 constants
export const autoTangentSlopeV1 = (prev: [number, number] | null, here: [number, number], next: [number, number] | null): number   // Ruling 5

// Drivers (Ruling 12)
export interface EvalContextV1 {
  readonly trackValueAt: (trackId: string, time: ExactTime) => unknown   // the same evaluator; cycles rejected at edit time
  readonly fps: number
  readonly seed: number                                                    // document seed for Wiggle; spec 05
}
export interface DriverV1 { readonly type: string; readonly enabled: boolean; readonly params: Readonly<Record<string, number | string | boolean>> }
export const applyDriversV1 = <V>(base: V, track: TrackV1<V>, time: ExactTime, ctx: EvalContextV1): V

// Time helpers (Ruling 8)
export const snapToFrameV1 = (time: ExactTime, fps: number): ExactTime          // nearest n / fps, ties toward later
export const frameIndexV1 = (time: ExactTime, fps: number): number

// Selection (Ruling 11; spec 06)
export interface KeySelectionV1 { readonly keys: ReadonlySet<string /* `${trackId}@${serializedTime}` */>; readonly segments: ReadonlySet<string> }
```

## 4. Shared UX conventions

**Modifier map** (Ruling 13). Mac names in brackets.

| Gesture | Meaning | Spec |
| --- | --- | --- |
| Drag a key | Move in time, snapped to whole frames; value unchanged | 07 |
| Shift + drag a key | Also snap to other keys, markers, guides and the playhead; in the graph, also to other keys' values | 07, 09, 04 |
| Ctrl [Cmd] + drag a key | Free sub-frame placement for this drag | 07 |
| Alt [Option] + drag a key | Duplicate the selection and drag the copy | 10 |
| Drag the edge handle of a multi-key selection | Proportional retime about the far edge; ripple later keys unless Shift is held | 07, 08 |
| Ctrl+Alt [Cmd+Option] + drag a selection | Stagger: the first item stays, the last moves the full drag, the rest spread evenly; Shift while dragging snaps the item under the pointer; opens the Stagger popover on release | 02 |
| Double-click a segment bar | Open the Ease popover for that segment | 01 |
| Right-click a segment bar | Context menu: presets, Copy Ease, Paste Ease (submenu: mirrored in time, pass-through), Paste Values Only, Hold, Linear, Reset to Auto | 01, 03 |
| Right-click a key | Context menu for the segment starting at the key: Copy Ease, Paste Ease, Paste Values Only; label; kind | 03, 09 |
| Double-click the ruler | Add a time guide at that frame | 09 |
| Marquee | Select keys in the box; Shift adds; Alt subtracts; layers are not deselected | 06 |
| Click a track name | Select every key on the track, in every view; Shift adds a track; Alt [Option] removes the track's keys from the selection | 06 |
| Alt [Option] + drag a handle in the graph | Break the join at that key; handles are joined by default. Alt is resolved by hit target: handle, then key (duplicate), then empty space (marquee subtract) | 04 |
| Shift + drag a popover handle | Constrain the handle horizontally | 01 |
| Ctrl [Cmd] + drag a popover handle | Move both handles of the segment mirrored | 01 |
| Alt [Option] + drag an influence slider | Move both sliders mirrored | 01 |
| Scrub a number field | Drag changes the value; Shift steps by 10, Ctrl [Cmd] by 0.1; with several keys selected the change is an offset | 06 |
| Type into a number field with several keys selected | Absolute by default; the field shows an Absolute / Offset toggle | 06 |

**Keyboard** (defaults; `VERIFY:` the app's keymap and rebind on collision).

| Keys | Meaning | Spec |
| --- | --- | --- |
| J / K | Previous / next key among the selected layers' visible tracks; with nothing selected, all layers | 10 |
| Shift+J / Shift+K | Previous / next key on all layers | 10 |
| Alt+Left / Alt+Right | Nudge selected keys one frame; Alt+Shift nudges ten | 07 |
| Ctrl+C / Ctrl+X / Ctrl+V | Copy or cut keys; paste at the playhead to the selected layers | 10 |
| Ctrl+Shift+V | Paste reversed | 10 |
| Ctrl+Alt+V | Paste ease only (values kept); Ctrl+Alt+Shift+V pastes values only | 03 |
| Ctrl+Shift+E | Open the Ease popover for the selected segments | 01 |
| Ctrl+Shift+G | Toggle the graph editor for the selected tracks | 04 |
| Ctrl+Shift+F | Snap the selected keys to whole frames; every key when nothing is selected | 07 |
| Ctrl+Alt+A | Select every key on the selected layers, collapsed tracks included; all layers when none selected | 06 |
| Ctrl+Shift+O | Open the Stagger popover for the selected keys | 02 |
| Ctrl+Shift+R | Open the Retime dialog for the selected keys | 08 |
| Ctrl+Alt+R | Time-Reverse the selected keys in place | 08 |
| Ctrl+Shift+B | Open the Add behaviour menu for the selected property | 05 |
| Alt+Up / Alt+Down | With a behaviour row focused, move it up or down; otherwise previous or next track keyed at the playhead | 05, 09 |
| Ctrl+Alt+1 to 8 / Ctrl+Alt+0 | Label the selected keys with palette colour 1 to 8; 0 clears. AltGr layouts use the context menu | 09 |
| Ctrl+Alt+J | Go to key number | 10 |
| Up / Down on a focused influence slider | Nudge 1 %; Shift 10 %; Ctrl [Cmd] 0.1 % | 01 |
| F / Shift+F / N (graph focused) | Fit selection / Fit all / toggle Normalise | 04 |
| Ctrl+A (graph focused) | Select every key shown in the graph | 04 |
| 1 to 9 | Apply ease library slots 1 to 9 to the selected segments | 01 |

**Popover pattern.** A popover is anchored to the thing it edits, opens in one gesture, previews live on the canvas and timeline as the user drags, commits on every change as one undo step per release, and closes on Escape (reverting nothing: the committed edits stay) or on a click outside. A popover never steals the selection.

**Number fields.** Unit suffixes are typed and parsed (`12f`, `0.4s`, `40%`); a field shows the composition's unit by default (frames for time). Expressions of the form `+5`, `-5`, `*2`, `/2` apply an offset or scale to every selected key; with several keys selected a leading `-` is an offset and `=-5` forces an absolute negative (spec 06).

**Live preview.** Every edit in a popover or the graph renders the canvas at the playhead and, when the playhead is outside the edited segment, also draws a ghost of the segment's end pose on the canvas. `VERIFY:` the worker's draft-quality path for scrubbing is reusable for popover previews.

**Text in the app.** Verbs on buttons (Apply, Bake, Snap), not nouns. Hover text names the gesture: "Double-click to edit the ease". No internal names (segment, driver) in user text: the user sees Ease, Behaviour, Offset.

## 5. Spec template

Every feature spec has these sections in this order, numbered as here, so an implementer can find the same thing in the same place.

1. **Purpose and evidence**: two to four sentences; the report sections that carry the demand.
2. **Scope and non-goals**.
3. **Rulings**: numbered, each with Why and Cost if wrong, continuing from the shared rulings above (spec-local numbering: 01-R1, 01-R2 …).
4. **Model and interfaces**: TypeScript for new types and functions; `VERIFY:` on anything that must match existing code.
5. **UX**: surfaces, controls with their ranges and defaults, visual states, modifiers taken from the shared map, keyboard, empty and error states.
6. **User flows**: numbered; each flow names the starting state, the steps the user takes, what the app shows after each step, and the end state; at least one primary flow, one keyboard-only flow, one multi-selection flow, one undo flow, one flow that hits a limit.
7. **Evaluation and determinism**: the pure function the feature evaluates; what it bakes and when; the Ruling 9 statement.
8. **Edge cases and named limits**.
9. **Interactions with other specs**.
10. **Rules and tests**: each rule a test; tests by name with the number they assert.
11. **VERIFY list**.
12. **Open questions for the user**.

## 6. Determinism tests shared by every spec

Every spec's test file includes these four, parametrised by a fixture that uses the feature:

- D1 `play equals seek`: evaluate frames 0..N in order, then frame N alone from a fresh evaluator; `Object.is` on every channel.
- D2 `random order`: evaluate a shuffled frame list; equal to D1's values.
- D3 `after an edit`: edit, undo, evaluate; equal to the values before the edit.
- D4 `bake round-trip`: where the feature bakes, the baked track evaluates within the stated tolerance of the source at every frame of the range, and unbake restores the source parameters exactly.

## 7. Data migration

Existing documents (`VERIFY:` the current key and ease storage) are read through an adapter that produces `TrackV1`: a per-key in/out handle pair becomes the ease of the segment between the keys; a key whose handles were never edited becomes `tangent: 'auto'`; a key marked hold becomes a `hold` ease on its segment. The adapter is pure and tested against pinned fixtures; no document is rewritten until the user saves.

## 8. VERIFY list

1. The folder, type and helper names of the animation core: `ExactTime` and its helpers, the track and key types, the overlay key format `${nodeId}/${propertyId}`, `HistoricalAnimationSamplerV1` and `historicalOverlayAt` (the evaluator a driver's `trackValueAt` should call).
2. The existing wiggle driver's interface, to make it the first `DriverV1`.
3. The current keymap, to resolve collisions with section 4.
4. The worker's draft-quality render path and whether a popover preview can request it per edit.
5. How the inspector edits a property today (number field component, scrub behaviour) to host the Absolute / Offset toggle of spec 06.
6. The document save format and version field, for the adapter in section 7.
7. The Lottie importer and exporter, for Ruling 7's frame-later rule and the bake-on-export rule of spec 05.
8. The undo stack's granularity, so one popover release is one step (Ruling 11).

## 9. Unresolved questions for the user

Decisions the feature specs made that depart from, or go beyond, this overview and the briefs, listed first because they bind until overturned:

1. **Ruling 7** chooses one key per frame with the one-frame-later cut as a named limit. The alternative is a jump key with an arrive and a leave value. Spec 10's paste rules depend on it; confirm or overturn before spec 10 is built.
2. **Default ease** is Material standard, `cubic-bezier(0.2, 0, 0, 1)`, asymmetric; spec 01 adds a "Set as default" document setting. A symmetric default (AE's Easy Ease, `(0.33, 0, 0.67, 1)`) is what migrating users expect. Confirm the asymmetric default.
3. **Spec 03 drops the direction flip.** With normalised handles an ease-out stays an ease-out whether the value rises or falls, so EaseCopy's flip (which exists because AE stores signed speeds) is unnecessary. Spec 03 ships "Mirror in time" as an explicit option, off by default. Confirm.
4. **Spec 02** makes a layer-level Time Offset inherit and add down the parent chain (a delayed group delays its children further). It also asks whether a move may place keys before frame 0 and whether Items = Layers moves layer bars with the keys.
5. **Spec 05** asks whether Wiggle exposes a per-octave multiplier, per-channel amplitude and a loop length; whether the Lottie export dialog allows a tolerance per track; and whether raw samples of an imported recording are kept for a re-fit.
6. **Spec 08** asks whether "loop the last N keys" is wanted; proposes that `continue` extrapolation bakes exactly (one key on the line) rather than by fit; and asks what Anchor = Playhead does when the playhead is outside the selection.
7. **Spec 06** defaults Proportional scrubbing to Offset for Position and Anchor Point, keeps labels at their time under Reverse, uses a hundredths grid for sub-frame values, and clamps keys at frame 0. Confirm each.
8. **Spec 09** defaults the per-layer summary row to on when a layer is expanded and keeps guides in place under a ripple retime. Confirm.
9. **Spec 10** makes J/K visit keys only (not markers or work-area edges) and stop at the ends; paste with property links is deferred. Confirm.
10. **Spec 01** stores spring, elastic and bounce parameters as shares of the segment, not seconds; **spec 03** writes a procedural ease's text fallback in the app's own `spring(…)` syntax rather than a bezier approximation. Confirm both.
11. **Spec 04** defers roving keys (Adobe's solver is unpublished), keeps handle selection out of `KeySelectionV1`, and resolves a merge of split channels with unequal eases by "larger delta wins". Confirm.
