# Spec 05: Settle, Follow and Wiggle behaviours, and the bake service

Written without the repository open; every assumption about unseen code is marked `VERIFY:` and gathered in section 11. The user sees Behaviour, Settle, Follow, Wiggle and Bake; "driver" is the code name for a behaviour (00 Ruling 12). The rulings, interfaces and modifier map of `00-overview.md` bind this spec.

## 1. Purpose and evidence

Settle, Follow and Wiggle replace the three expression families every After Effects designer wraps in a plug-in: the Ebberts inertial bounce and overshoot, the `valueAtTime(time − delay)` chain and `wiggle()`. Report 1 §4 shows all three are closed-form functions of time, baked for cost and for Lottie rather than from any need of history; its ranked table puts the set at rank 5. Report 2 §4 supplies the closed-form oscillator in Apple's duration and bounce form with Motion's overdamped branch, the rule that noise must hash `(t, seed)`, and Schneider's fitter with the shipping tolerances; report 2 §3 supplies the out-of-order frame test. The bake service is shared with specs 01, 02 and 08.

## 2. Scope and non-goals

In scope: three behaviour types with presets; the Behaviours section of the inspector; stacking semantics; `bakeTrackV1` in exact and fit modes with Unbake; the Lottie export and import rules; the pure-versus-bake classification. Non-goals: Newton-class physics, soft body and a spring chasing a keyed target (baked keys only); anticipation; a windowed `smooth()`; `temporalWiggle`; a Wiggle loop length (section 12); drawing the post-behaviour curve (spec 04); the Time Offset behaviour itself (spec 02; this spec places it in the menu and the stack).

## 3. Rulings

- **05-R1 (Settle after every key that arrives with speed):** a Settle entry adds a term after every key of its track whose arrival velocity is non-zero, with a per-key opt-out glyph. Why: it is the Ebberts expression's rule ("use this on any property with two keyframes", report 1 §4); a key arrived at with zero speed gets no term, so the default is harmless on eased keys; the opt-out is Kleaner's "True stop". Cost if wrong: the same flag, default inverted.
- **05-R2 (terms are additive and continue past the next key):** `value(t) = base(t) + Σ term_k(t − t_k)` over the keys at or before `t` whose term has not rested; a hold key cuts every term at its time and launches none; no term exists before the first key. Why: using only the newest key's term jumps when a key arrives mid-wobble; a sum is continuous and still closed-form; a hold means stop dead (00 Ruling 6). Cost if wrong: one loop bound.
- **05-R3 (arrival velocity from the undriven track):** `v = (base(t_k) − base(t_k − δ)) / δ`, `δ = 1 / (10 · fps)` s, read through a keys-only sampler of the own track, so the sample excludes every behaviour and cannot recurse. Why: the expression reads `velocityAtTime(key.time − frameDuration / 10)`. Cost if wrong: one constant.
- **05-R4 (rest threshold and cap):** a term is exactly zero once its closed-form envelope is under the property's rest threshold (position 1e-3 px, the effects plan's Ruling 12 figure for a resting smear) and in any case 10 s after its key (Motion's `maxDuration`); the rest time comes from the parameters, never from watching the value. Why: a resting layer must evaluate bit-identical to its base so the render cache keeps its token; `Math.exp(−decay · τ)` with τ ≤ 10 cannot overflow, so Ebberts' Infinity bug cannot recur. Cost if wrong: a table of five numbers.
- **05-R5 (Follow addresses a track by node id):** a Follow stores `nodeId` and `propertyId`, never a layer index; a cycle is refused at edit time with a message naming the chain. Why: AE's `index` chains break on reorder (report 1 §4); 00 Ruling 12. Cost if wrong: nothing.
- **05-R6 (Wiggle is a hash):** noise is `hash4V1(documentSeed, seed, lane, lattice)` interpolated with a smooth step; `Math.random` is forbidden in the animation core and a test proves it is never called. Why: report 2 §4 and Remotion's contract; Duik's stored seed survives reorder and copy. Cost if wrong: not accepted (00 Ruling 9).
- **05-R7 (stack order is evaluation order):** behaviours apply top to bottom as listed, each receiving the running value from the one above. Why: Settle after Follow and Follow after Settle differ (section 7) and the user must see which they have. Cost if wrong: one reversed loop.
- **05-R8 (two bake modes):** exact mode shifts or copies keys and eases with no sampling; fit mode samples, fits time-parametrised Béziers to a per-property tolerance, snaps every key to a whole frame (00 Ruling 8) and keeps the sources disabled so Unbake restores them exactly. Why: an offset or loop must stay exact; Duik's smart mode, Easy Bake's pruning and AE's "expression preserved" rule (report 1 §4) are what users expect of a fit. Cost if wrong: one branch.
- **05-R9 (Lottie bakes everything procedural):** export bakes every non-bezier ease, behaviour and loop into a copy, the document untouched; recorded or generated motion imports as baked keys only. Why: native Lottie players run no expressions and lottie-web lacks `wiggle` and `velocityAtTime` (report 1 §4). Cost if wrong: files that differ on iOS and Android.

## 4. Model and interfaces

```ts
// src/animation-core/keyframes/behaviours.ts        VERIFY: folder; 00 §3 types
export type SettleTypeV1 = 'overshoot' | 'bounce' | 'spring'
export interface SettleParamsV1 {                      // DriverV1 type 'settle'; meanings in §5
  readonly settle: SettleTypeV1; readonly amplitude: number           // %
  readonly frequency: number; readonly decay: number                  // Hz, 1/s (overshoot, bounce)
  readonly duration: number; readonly bounce: number                  // s, −1..0.95 (spring)
  readonly preset: string                                             // '' when edited by hand
}
export interface FollowParamsV1 {                      // type 'follow'; 05-R5
  readonly nodeId: string; readonly propertyId: string
  readonly delayFrames: number; readonly strength: number             // frames; 0..100 %
  readonly lagDuration: number; readonly lagBounce: number            // lagDuration 0 = no Lag
}
export interface WiggleParamsV1 { readonly amplitude: number; readonly frequency: number; readonly octaves: number; readonly seed: number }
export interface DriverEvalContextV1 extends EvalContextV1 {
  readonly baseValueAt: (trackId: string, time: ExactTime) => unknown   // keys and eases only, no behaviours
  readonly keysOf: (trackId: string) => readonly KeyframeV1<unknown>[]
  readonly restThreshold: number
}
export const settleTermV1 = (p: SettleParamsV1, v: number, tau: number, rest: number): number
export const settleRestTimeV1 = (p: SettleParamsV1, v: number, rest: number): number     // s, ≤ 10
export const findFollowCycleV1 = (doc: DocumentV1, from: string, to: string): readonly string[] | null
export const REST_THRESHOLD_V1 = { position: 1e-3, rotation: 1e-3, scale: 1e-3, opacity: 1e-5, other: 1e-3 } as const
export const hash4V1 = (a: number, b: number, c: number, d: number): number => {   // uint32 mix → [0, 1)
  let h = (a | 0) >>> 0
  for (const x of [b, c, d]) { h = Math.imul(h ^ ((x | 0) >>> 0), 0x9e3779b1) >>> 0; h ^= h >>> 15 }
  h = Math.imul(h ^ (h >>> 16), 0x85ebca6b) >>> 0; h = Math.imul(h ^ (h >>> 13), 0xc2b2ae35) >>> 0; h ^= h >>> 16
  return (h >>> 0) / 4294967296
}

// src/animation-core/keyframes/bake.ts
export interface BakeOptionsV1 {
  readonly mode: 'exact' | 'fit'; readonly range: { readonly start: ExactTime; readonly end: ExactTime }
  readonly tolerance: number; readonly targets: ReadonlySet<'behaviours' | 'eases' | 'loops'>
}
export interface BakeRecordV1 {                        // on the track (section 12)
  readonly options: BakeOptionsV1; readonly writtenTimes: readonly string[]
  readonly restore: string                                            // JSON of the keys, eases, extrapolation replaced
  readonly disabledEntries: readonly number[]
}
export const bakeTrackV1 = <V>(track: TrackV1<V>, o: BakeOptionsV1, ctx: DriverEvalContextV1):
  { track: TrackV1<V>; keyCount: number; maxError: number; split: boolean }
export const unbakeTrackV1 = <V>(track: TrackV1<V>, r: BakeRecordV1): { track: TrackV1<V>; editedKeysDiscarded: number }
export const BAKE_TOLERANCE_V1 = { position: 0.25, opacity: 0.002, rotation: 0.1, scale: 0.1 } as const   // px, 0..1, degrees, %
```

`DriverV1.params` holds scalars only (00 §3), so the Lag is two numbers and the bake record is a field. The hash is written out so two builds agree bit for bit.

## 5. UX

**Behaviours section.** Under each animatable property in the inspector: the header "Behaviours" and an "Add" menu listing Settle, Follow, Wiggle and Time Offset (spec 02). Each entry is a numbered row: enable checkbox, drag handle (by keyboard, Space picks the row up, Up and Down move it, Space drops it), type, preset menu, three number fields, "Bake", "Remove". Hover text on the handle: "Behaviours apply from top to bottom". Fields follow 00 §4's number-field rules. With several layers selected the section edits the matching entry on every selected layer as one undo step. Empty state: header and Add only.

**Settle.** Types: Overshoot (passes the key's value and wobbles about it), Bounce (rebounds on one side, like a dropped ball), Spring (a damped spring in Duration and Bounce).

| Control | Overshoot | Bounce | Spring |
| --- | --- | --- | --- |
| Amplitude | 0..200 %, default 100: the wobble launches at this share of the arrival speed | 0..50 %, default 5 (Animoplex `amp = 5.0`): the first rebound is arrival speed × Amplitude/100 s | as Overshoot |
| Frequency or Duration | 0.1..10 Hz, default 3 (overshoot variant) | 0.1..10 Hz, default 2 (Animoplex) | Duration 0.1..3 s, default 0.5 (Apple's example) |
| Decay or Bounce | 0.5..20, default 5 (overshoot variant) | 0.5..20, default 4 (Animoplex) | Bounce −1..0.95, default 0.3 (Apple, Motion); ζ ≥ 0.05 because damping 0 runs forever (report 2 §4) |

The note's `amp` 0.025..0.12 at 2..3 Hz is 30..190 % in launch form, inside the Overshoot range. Presets (type; Amplitude; Frequency or Duration; Decay or Bounce): **Overshoot** (Overshoot; 100; 3; 5), **Bounce** (Bounce; 5; 2; 4), **Friction** (Overshoot; 100; 2; 10, the "friction" variant's decay), **Alive** (Spring; 100; 0.5 s; 0.3), **Inanimate** (Spring; 100; 0.4 s; 0, critically damped), **Spring** (Spring; 100; 0.8 s; 0.3, Motion's defaults). Names from Mt. Mograph Excite and Duik Kleaner (report 1 §4). Editing a number sets the preset to "Custom".

**Per-key glyph.** A key with an active term shows a small wave glyph above it in the timeline; clicking it toggles "Settle after this key", also in the key's context menu (`VERIFY:` spec 10). A key arriving with zero speed shows no glyph; when none qualifies the entry reads "No key on this track arrives with speed. Settle needs a Linear or an ease-in on the segment before the key."

**Follow.** Target picker: "Parent: <same property>" first when the parent has it, then a searchable list of layers and properties of the same kind (scalar to scalar, vector to vector of the same length; others greyed, "Different kind of value"); self excluded. Delay 0..600 f, default 4 f (the low end of the research note's "4 to 8 frames" rule of thumb). Strength 0..100 %, default 100. Lag: None, or Spring with Duration 0.1..3 s (default 0.5) and Bounce −1..0.95 (default 0.3), iExpressions' "elastic connection". A deleted target reads "Missing: <layer name>" in the warning colour and the entry is the identity; a cycle refuses the change with "Layer A already follows this layer (A → B → C). Choose another layer."

**Wiggle.** Amplitude in the property's unit, 0..1000, default 10 (visible at 1080p, inside a layer's footprint). Frequency 0.1..30 Hz, default 2 (drift, not shake). Octaves 1..4, default 1 (AE's default). Seed 0..9999, set at creation from `hash4V1` of the track id, shown, editable, with "Reseed"; a copied entry keeps its seed (Duik's rule).

**Graph and bake dialog.** Spec 04 draws the keys' curve solid and the post-behaviour curve dashed in the track's colour, read only, as AE's post-expression graph. The Bake dialog (an entry's Bake, or "Bake behaviours" in the property menu) has Range (Work area, Whole layer, Custom in frames), Tolerance per property defaulting to `BAKE_TOLERANCE_V1`, a live "n keys" count, Bake. A baked entry shows greyed, "Baked, 23 keys", with Unbake; with edited baked keys Unbake asks "Unbake discards 3 edited keys". No new key bindings; Tab order is Add, then each row left to right.

## 6. User flows

**Flow 1, primary: a scale pop.** Start: Scale keys 0 % at frame 0 and 100 % at frame 10, Linear, playhead at frame 10. (1) Under Scale, Add, Settle: row 1 "Settle, Overshoot" with 100, 3, 5; the canvas at frame 10 is unchanged (τ = 0 is the base); a glyph appears above the frame-10 key; the graph's dashed curve wobbles past 100. (2) Scrub to frame 14: the layer shows above 100 %. (3) Choose Friction: one overshoot, canvas updated. (4) Scrub Decay to 8: the curve tightens live, one undo step on release. End: keys plus one decaying term after frame 10, at rest by about frame 60.

**Flow 2, keyboard only: a Wiggle.** Start: Position focused in the inspector, no keys. (1) Tab to Add, Enter: the menu opens. (2) Down twice, Enter: row 1 "Wiggle" with 10, 2, 1 and a seed; the canvas drifts. (3) Tab to Amplitude, type `25`, Enter: the drift grows. (4) Tab to Seed, type `4242`, Enter: a different drift; the old seed restores the old one. End: Wiggle 25, 2 Hz, 1 octave, seed 4242.

**Flow 3, multi-selection: a chain of five.** Start: Leaf 1..5 selected, Leaf 1 has Rotation keys. (1) Under Rotation, Add, Follow: the picker offers "Each follows the layer above" because several layers are selected. (2) Choose it, Delay 4 f: Leaf 2..5 each get a Follow targeting the layer above by node id, one undo step; the canvas cascades. (3) Set Lag to Spring: every selected entry gets 0.5 s and 0.3; each leaf overshoots at the leader's arrivals. (4) Reorder the layers: the canvas does not change (05-R5). End: a four-link chain with Lag.

**Flow 4, undo: reorder the stack.** Start: Position with row 1 Follow, row 2 Settle. (1) Drag Settle above Follow: rows renumber; the canvas changes because the Follow now discards the Settle at Strength 100 % (section 7); the timeline selection stays. (2) Ctrl+Z: the order returns; the canvas matches the start; selection and inspector scroll unchanged (00 Ruling 11). End: as the start.

**Flow 5, limit: a cycle.** Start: A's Position follows B, B follows C. (1) On C, Add, Follow, pick A: refused with "Layer A already follows this layer (A → B → C). Choose another layer."; the picker stays open; nothing is added; the undo stack is unchanged. (2) Pick the Null: the entry is added. End: C follows the Null.

**Flow 6, bake for Lottie, then edit.** Start: Scale with the Bounce preset after a frame-10 key, 30 fps. (1) Bake on the entry: the dialog shows Work area, Tolerance 0.1 %, "n keys: 9". (2) Bake: 9 keys on whole frames between frames 10 and 50 with bezier eases; the entry greys to "Baked, 9 keys"; the dashed and solid curves coincide; the glyph goes. (3) In the graph, drag the frame-14 key higher: the canvas follows; it is an ordinary key. (4) Export, Lottie: the dialog lists "Scale: 9 baked keys", nothing else to bake. End: the file carries the edited keys; the entry remains disabled with 5, 2, 4.

**Flow 7, unbake.** Start: the end of flow 6. (1) Unbake: "Unbake discards 1 edited key". (2) Confirm: the 9 keys go, the frame-10 key regains Linear, the entry re-enables with 5, 2, 4, the glyph returns, the dashed curve equals flow 6's start at every frame. End: the live behaviour as before the bake, one undo step.

## 7. Evaluation and determinism

`evaluateTrackV1` computes `base(t)` from keys, eases and extrapolation; `applyDriversV1` folds the enabled entries top to bottom, each `(running, t, ctx) → value`, reading other tracks only through `trackValueAt` and `baseValueAt` at exact times. `t` is the exact rational converted once to seconds.

**Settle.** Per channel, over keys `k` with `t_k ≤ t` that are not hold, not opted out and within 10 s: `v` per 05-R3, `τ = t − t_k`, `ω = 2π · frequency`, `u = v · amplitude / 100`.
- Overshoot: `term = u · sin(ω τ) · exp(−decay · τ) / ω` (slope `u` at τ = 0, the overshoot variant).
- Bounce: `term = u · |sin(ω τ)| · exp(−decay · τ)`.
- Spring: `ω₀ = 2π / duration`, `ζ = 1 − bounce` (reproduces Apple's example exactly, report 2 §4). ζ < 1: `ω_d = ω₀ √(1 − ζ²)`, `term = (u / ω_d) · exp(−ζ ω₀ τ) · sin(ω_d τ)`. ζ = 1: `term = u · τ · exp(−ω₀ τ)`. ζ > 1, Motion's two-exponential form: `λ_s = −ω₀ / (ζ + √(ζ² − 1))`, `λ_f = −ω₀ (ζ + √(ζ² − 1))`, `term = u · (exp(λ_s τ) − exp(λ_f τ)) / (λ_s − λ_f)`. Every branch has `term(0) = 0` and slope `u`, so they meet at ζ = 1.
- Rest: envelope `E₀ · exp(−r τ)` with `(E₀, r)` = `(|u| / ω, decay)`, `(|u|, decay)`, `(|u| / ω_d, ζ ω₀)`, `(2|u| / (e ω₀), ω₀ / 2)` and `(|u| / (λ_s − λ_f), −λ_s)` for the five cases in order. `settleRestTimeV1 = min(10, ln(E₀ / ε) / r)`, or 0 when `E₀ ≤ ε`, `ε` the rest threshold. For `τ ≥ rest` the term is the number 0, so `running + 0` is bit-identical to `running`. A term is also zero from the first hold key after `t_k`.

**Follow.** `θ = t − delayFrames / fps` as an exact rational; `L = trackValueAt(target, θ)`; with Lag, `L += Σ term_j(θ − t_j)` over the target's keys using the Spring branch, `v_j` from `baseValueAt` on the target; a keyless target that itself follows passes the Lag on with its own delay added, so a chain lags at the leader's keys. `value = running + (strength / 100) · (L − running)`. Cost: a chain of N followers is N evaluations per frame at one exact time each; `t − d` for frame n is frame n − d, already evaluated, and the sampler memoises by exact time, so there is no quadratic cost (`VERIFY:` `HistoricalAnimationSamplerV1` keys its cache by the serialised exact time, effects plan B9 §6; a sub-frame delay is off-grid and pays per frame).

**Wiggle.** Per channel `c` and octave `o`: `lane = 4c + o`, `x = seconds(t) · frequency · 2^o`, `i = floor(x)`, `s = (x − i)² (3 − 2(x − i))`, `n_o = h(i) + s (h(i + 1) − h(i))`, `h(i) = 2 · hash4V1(ctx.seed, seed, lane, i) − 1`; `value = running + amplitude · Σ 0.5^o n_o / Σ 0.5^o` (AE's `amp_mult` 0.5). Any frame evaluates alone; the output is bounded by Amplitude. `VERIFY:` the existing wiggle driver: if its noise already hashes `(seed, t)` it becomes this function with its parameters renamed by the 00 §7 adapter; if not, the adapter maps them onto `WiggleParamsV1` and the effects plan's T5 fixtures are re-recorded in a commit that says so.

**What bakes and when.** Nothing bakes for play, seek or video export. Lottie export runs `bakeTrackV1` on a copy: exact mode for Time Offset (shift every key) and spec 08's loops (copy the keys per cycle; ping-pong reverses the copy with spec 03's flip); fit mode for every behaviour, every spring, elastic and bounce ease, and `continue` extrapolation. Fit mode: (1) sample `base + behaviours` at every frame of the range, and at the half-frames when any term's frequency exceeds `fps / 4`; (2) force keys at value extrema (sign change of the first difference) and velocity extrema (sign change of the second); (3) fit each span with Schneider's algorithm on `(t, value)` points: chord-length parametrisation as the initial `u`, the 2 × 2 least-squares solve for the two handle lengths along the end tangents, acceptance at max error ≤ tolerance, up to four Newton reparametrisation rounds while the error is under 4 × tolerance, else a split at the maximum-error sample; handle times are clamped to the span so `x(u)` is monotone, and the error is measured in value at each sample's time through 00 Ruling 4's solver; (4) snap every key to a whole frame (ties later) and re-fit the adjacent spans; if the tolerance still fails, split them at the maximum-error frame and repeat, down to a key per frame at worst. Vectors share the union of forced times and one normalised ease (00 Ruling 3); when the channels' handles disagree beyond the tolerance the bake splits the property into per-channel tracks through spec 04's reversible command and says so. Position is fitted as a spatial path at the pixel tolerance plus a timing curve per path segment (00 Ruling 10). The sources stay as disabled entries and a `BakeRecordV1` holds what was replaced; Unbake removes the written keys, restores the record and re-enables the entries.

**Classification** (report 2 §4):

| Motion | Class | Live | Lottie |
| --- | --- | --- | --- |
| Bezier and hold eases | pure | yes | as is |
| Spring, elastic, bounce eases (spec 01); `continue` extrapolation | pure | yes | fit bake |
| Time Offset (spec 02); cycle, ping-pong, offset loops (spec 08) | pure | yes | exact bake |
| Settle, Follow with Lag, Wiggle | pure | yes | fit bake |
| Spring chasing a keyed target, Newton-class physics, soft body | history | never | keys only (import) |
| Recorded pointer motion, generated or AI motion | keys | as keys | as is |

**Ruling 9 statement.** Every value above is `evaluate(document, time)`: the active term set depends on key times and parameters alone, the rest time is closed-form, the followed time is an exact rational, the noise is a hash. No evaluator instance holds state that changes a value; the cache changes only cost.

## 8. Edge cases and named limits

- A track whose keys all arrive with zero speed (the default ease ends flat) gets no settle; Settle does not synthesise speed.
- Keys closer together than a term's rest time stack terms: at most 10 s of keys are active, 300 terms at a key per frame at 30 fps; the bake dialog's key count warns first.
- A Follow at `θ < 0` reads the target's `before` extrapolation; a target whose property changes kind becomes "Missing".
- Wiggle's lattice index is an int32: `|t · frequency · 2^o| < 2^31`, 2.5 days at 30 Hz and four octaves.
- A fit needing a key per frame is reported; the user raises the tolerance or shortens the range. A vector bake that splits the property is reversed only by Unbake or undo.
- Settle after Follow on a keyless follower adds nothing; the Lag is the way to wobble at the leader's arrivals, and the §5 hint appears.

## 9. Interactions with other specs

- Spec 01: its spring ease and this Spring settle share `ζ = 1 − bounce` and `ω₀ = 2π / duration` (`VERIFY:` spec 01's settle-based duration rule; the settle term has no end constraint because it is additive, not a segment).
- Spec 02: Time Offset joins the stack and bakes exactly; `VERIFY:` that it replaces the running value with `baseValueAt(own, t − offset)`, so a Settle above it is discarded and one below applies to the offset motion.
- Spec 03: the ease flip for ping-pong. Spec 04: the dashed curve and the per-channel split. Spec 06: multi-selection editing. Spec 07: `snapToFrameV1`. Spec 08: the loop definitions. Spec 09: labels survive Unbake. Spec 10: the key context menu hosting "Settle after this key".

## 10. Rules and tests

Each rule is a test in `behaviours.test.ts` or `bake.test.ts`; the fixture is a 2 s, 30 fps document with Scale keys 0 at frame 0 and 100 at frame 10 (Linear) unless stated.

- R1 Amplitude 0 is the identity: T1 `settle: amplitude 0 is the identity`, `Object.is` on frames 0..60.
- R2 No term before the first key: T2 `settle: zero before the first key`, frames −10..0 equal the base exactly.
- R3 Overshoot at τ = 0 is the base: T3 `overshoot: value at the key is the key value`, frame 10 `=== 100`.
- R4 The spring branches meet: T4 `spring: ζ 0.999, 1 and 1.001 agree`, max difference under 1e-6 over 2 s.
- R5 Rest is exact: T5 `settle: after the rest time the value is bit-identical to the base`, `Object.is` at every later frame; the Overshoot preset with v = 300 %/s rests at `ln(15.9 / 1e-3) / 5 = 1.94 s` within 1e-3.
- R6 A hold cuts: T6 `settle: a hold key at frame 20 cuts the term`, frames ≥ 20 equal the hold value exactly.
- R7 Opt-out: T7 `settle: opted-out key`, the frame-10 term is absent while a second key's term at frame 30 remains.
- R8 The chain: T8 `follow: three layers at 2 f, layer 3 at frame 10 equals layer 1 at frame 6`, `Object.is` per channel.
- R9 Cycle: T9 `follow: C → A is refused when A → B → C`, `findFollowCycleV1` returns `['A', 'B', 'C']`, the document deep-equal to before.
- R10 Lag through a keyless follower: T10 `follow: a chain lags at the leader's keys`, layer 3's term times equal layer 1's key times plus two delays.
- R11 Wiggle is bit-identical and seeded: T11 `wiggle: two evaluator instances agree at every frame and seeds 1 and 2 differ on at least 90 % of 60 frames`, `Object.is`, with `Math.random` replaced by a throwing stub for the file.
- R12 Order: T12 `stack: Settle then Follow differs from Follow then Settle`, not equal, each within 1e-9 of its hand computation.
- R13 Exact bake: T13 `bake: exact mode shifts Time Offset 3 f`, every key time `(n + 3) / fps` by bigint equality, eases deep-equal.
- R14 Fit bake of a spring ease: T14 `bake: spring ease (0.5 s, 0.3) over 30 f fits within 0.1 % and unbakes`, every frame within tolerance, every key on a whole frame, at most 12 keys (a fifth of the frames; a higher count is reported and the bound revisited), unbake restores `{ type: 'spring', duration: 0.5, bounce: 0.3 }` deep-equal with the two original keys only.
- R15 Snap then re-fit: T15 `bake: a mid-frame split snaps and still fits`, a 7 Hz Bounce at 30 fps within tolerance after the snap.
- R16 Lottie and import: T16 `export: no behaviour, non-bezier ease or loop survives`, eases bezier or hold only, keys equal to the fit bake's; T17 `import: recorded samples arrive as keys only`, no behaviours, within tolerance.
- D1 `play equals seek`: frames 0..60 in order, then 60 from a fresh evaluator, `Object.is` per channel, on a fixture with all three behaviours.
- D2 `random order`: a shuffled frame list equals D1.
- D3 `after an edit`: change Decay, undo, evaluate; equal to before.
- D4 `bake round-trip`: the baked fixture is within tolerance at every frame and unbake restores every parameter deep-equal.

## 11. VERIFY list

1. The animation core folder, `ExactTime` helpers, `KeyframeV1` and `TrackV1` as the 00 §7 adapter produces them.
2. The existing wiggle driver: parameter names, noise function, whether it hashes `(seed, t)`.
3. `HistoricalAnimationSamplerV1` and `historicalOverlayAt`: that they serve `trackValueAt` and key the cache by the serialised exact time.
4. Whether the evaluator can expose a keys-only read (`baseValueAt`) without a second code path.
5. `DocumentV1`, the parent link for the picker, the property ids, and the opacity scale (0..1 assumed; at 0..100 the tolerance is 0.2).
6. The key context menu (spec 10) for "Settle after this key".
7. The Lottie exporter's entry point for the bake-on-export copy, and the importers for recorded and generated motion.
8. Spec 01's spring duration rule, spec 02's Time Offset semantics, spec 03's flip, spec 04's split, spec 08's loops.
9. The undo stack's granularity, so a bake and an unbake are single steps.

## 12. Open questions for the user

1. Proposed addition to 00 §3: `KeyframeV1.settle?: 'skip'` for the per-key opt-out (05-R1); without it the opt-out is a serialised time list in the entry's params, which a key drag would orphan.
2. Proposed addition to 00 §3: `TrackV1.bakes?: readonly BakeRecordV1[]`; without it the record is a JSON string param on a disabled `bake` entry.
3. Proposed addition to 00 §4: `Ctrl+Shift+B` opens the Add behaviour menu for the selected property; `Alt+Up` / `Alt+Down` move the focused row. The keyboard flow works without them.
4. Wiggle: expose the per-octave multiplier, a per-channel amplitude and a loop length so a loop closes seamlessly (iExpressions and Duik each added these)?
5. Should the Lottie export dialog allow a tolerance per track, or is the per-property default enough for a first release?
6. Keep the raw samples of an imported recording for a re-fit at another tolerance (report 2 §4)? It costs storage per import and one more entry type.
