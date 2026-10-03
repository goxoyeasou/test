# Effects Batch B: tasks B1, B7 and B9

Written without the repository open. Every assumption about unseen code is marked `VERIFY:` and gathered in section 5. Text a person sees in the app is written for a motion designer.

## 1. Rulings

- **Ruling 1 (runner author):** Batch B writes `native-webgl2-bloom.ts` in B8. B7 ships its interface as types plus a CPU plan function (`src/render-core/shading/bloom-runner.ts`), and the one line 3D-D's Task 11 must take. Why: the design's order is A, B, C, then 3D-D, and the layer Bloom cannot be proven without the runner. Cost if wrong: Task 11 calls an interface it did not choose (a `writeExcess` callback and a `destination`); no rewrite, one adapter at most.
- **Ruling 2 (smear under a shutter):** option (a). Each shutter sample measures the smear at its own sample time: its place then, less its place one frame before then. Why: the vector stays a pure function of time, so play = seek = export holds inside the shutter too, and the grain seed already follows this pattern. Named limit: Motion blur and Motion smear stack (a smear inside each sample, then the samples averaged). Cost: one extra off-grid replay per sample under motion blur, never cached (the sampler keeps off-grid times transient). Cost if wrong: a visible double trail on a layer with both effects, which the manual look list covers.
- **Ruling 3 (per-frame writer):** B9 renames `withGrainSeeds` to `withEffectClocks` and `hasGrainSeeds` to `hasEffectClocks`, as its own commit with a pinned test, then adds the smear vectors inside it. Batch C only adds its clock. Cost if wrong: Batch C's plan text names the old function; a one-word edit there.
- **Ruling 4 (soft clip):** the layer Bloom keeps `bloomGlowV1` as it is, soft clip included. Why: "the same functions in the same order"; the clip is the identity under 0.8 linear, so only a glow brighter than the knee changes, and it changes toward 1 instead of clipping. One shader, one reference, no second bloom. Cost if wrong: a layer's brightest glow is up to 9 % dimmer on screen than a hard clip would show; the pinned test (B7 T7) states the value, so a later change is deliberate.
- **Ruling 5 (composition place):** new exported `compositionMatrixV1(document, overlay, nodeId)` in `src/animation-core/composition-matrix.ts`, the parent chain root first, each node through `localTransformMatrixV1` with evaluated value then stored fallback. The layer's place is its anchor point through that matrix. Auto-layout is a named limit: a layer placed by an auto-layout parent smears by its parent's move only (its own layout place is not in the overlay). The Distribute closure and the Scene clip check are left alone. Cost if wrong: a child of a moving group would not smear; the test B9 T3 pins that it does.
- **Ruling 6 (Limit):** the Limit clamps the vector by its length, direction kept. Why: a per-component clamp bends a diagonal move off its path. Cost if wrong: one function, one test.
- **Ruling 7 (layer space):** Spin blur, Zoom blur and Progressive blur are layer-space effects (`layerSpace: true`, pass carries `transform` and `localBounds`). Their Centre, Direction, Start and End are measured on the layer; their lengths (Zoom's none, Spin's none, Progressive's Blur) are picture lengths. For bounds, `compileEffectStackV0` gains an optional `layerFrame: { transform, localBounds }`; with it the Centre is exact, without it the Centre is mapped onto `sourceBounds` as if the layer were upright. Cost if wrong: on a turned layer with a Centre far from the middle, a spin's lit pixels could be cut at the bounds; never a wrong pixel inside them.
- **Ruling 8 (reads outside the target):** every Batch B kernel, CPU and GLSL, treats a read outside the target's box as a clear pixel (0,0,0,0). Why: it removes any dependence on texture edge clamping and keeps "effects read nothing outside their scope's target" literal. Cost if wrong: a one-line guard in the prelude.
- **Ruling 9 (octave halving for sampled blurs):** the picture is halved until the gap between neighbouring samples is at most one texel: `workingScale = min(1, profile.resolutionScale, 1 / gapDevicePixels)`, `gapDevicePixels = pathDevicePixels / sampleCount`. This holds for Directional blur (path = Length) and Motion smear (path = the vector's length), where the gap is the same at every pixel. Spin and Zoom blur do not halve the picture (Ruling 24). Cost if wrong: streaks between samples on a long Directional blur; the proof catches it.
- **Ruling 10 (pyramid):** level 0 is the isolation picture itself (sharp, decoded to linear). Levels 1..n are made with `BLOOM_DOWNSAMPLE_TAPS_V1`, each half the one before, and read at the output with `BLOOM_UPSAMPLE_TAPS_V1` (the tent) at the level's own texel. n is as many as the Blur needs, at least 1, at most 8: five pictures at the default Blur 40 (σ 20 lies between levels 4 and 5). The two levels nearest the wanted σ blend by variance (section 3, B1 kernel 2). Below the first level's σ the sharp picture (a plain read, no tent) blends with level 1. Progressive blur is the same at every quality. Why: the bloom filters have no sigma knob, so the chain's sigmas are what they are; five levels reach σ ≈ 33 device px. Eight is derived, not chosen: the largest Blur after the contract's clamp is 256, so σ_max is 128 composition px, which is 256 device px at a display density of 2; level 8 (σ ≈ 266) covers it and level 7 (σ ≈ 133) would cap the slider at half. Levels exist only when the Blur needs them (`progressiveLevelsNeededV1`), so the default Blur 40 still makes five pictures. Cost of levels 6 to 8 when used: three small draw calls (about 0.03 ms by the project's figure); their pixels are negligible. Cost if wrong: the stop point (mean 2, max 12 at Blur 40) catches it.
- **Ruling 11 (sample counts):** Directional, Spin, Zoom and Motion smear take 16 samples at `draft` and `full`, 48 at `export`; `draft` keeps the profile's resolutionScale of 0.5. Cost if wrong: a number in one table.
- **Ruling 12 (smear nothing):** a smear vector whose length after Amount and Limit is under 1e-3 composition px compiles to nothing. Why: float noise from matrix products must not re-draw a resting layer every frame. Cost if wrong: a resting layer re-renders; the cache test catches it.
- **Ruling 13 (smear time per node):** a node's smear is measured at the time of the overlay the node carries. In a shutter sample a node with motion blur off carries the frame's overlay, so its smear is the frame's (same in every sample). Cost if wrong: a crisp node under a shutter shows a faint extra trail.
- **Ruling 14 (frame 0, in point, loop):** when the earlier time would be under zero, the vector is zero. A layer's in point is not special: its place a frame earlier is evaluated like any other. A loop's first frame is frame 0 and gets zero. Cost if wrong: one wrong first frame of a loop; stated on the limits list.
- **Ruling 15 (Softness units):** Outline's Softness (0 to 100 %) is a share of its Width: the soft band is `Width × Softness` composition px. Choke edge's Softness is already px. Cost if wrong: a scale factor on one line.
- **Ruling 16 (Long shadow steps):** distinct steps 1, 2, 4, … device px, the last shortened so the steps sum to the Length; passes = `ceil(log2(lengthDevice + 1))`. Exact for the design's form (B1 kernel 4).
- **Ruling 17 (K):** `BLOOM_REACH_TEXELS_V1[levels]` for 1 to 5 levels is the full support of one lit first-level texel through `bloomChainV1` at Spread 1 (farthest texel on the impulse row whose glow is above 0), fitted by a test. The design's "about 63 for five levels" is the support. Bounds grow by exactly Size. Cost if wrong: a glow a few faint pixels wider than Size, which the bounds test catches.
- **Ruling 18 (layer composite):** the layer's glow is drawn over the isolation target with the Scene's blend (`ONE, ONE_MINUS_SRC_COLOR` for colour, `ONE, ONE_MINUS_SRC_ALPHA` for alpha), so no glow leaves the pixel to the bit by the blend itself and one runner path serves both callers. Cost if wrong: nothing; the reference (B7 T8) pins the arithmetic either way.
- **Ruling 19 (draft):** Bloom, Progressive blur and Long shadow are the same at `draft` as at `full`. Cost: none measurable.
- **Ruling 20 (smear fields):** the vector rides the pass as `vectorXPixels` and `vectorYPixels` (ordinary fields, compared bit-exactly by the cache planner), never as `transform`.
- **Ruling 21 (eight-bit range):** the layer Bloom's levels hold `excess × tint`, never above 1, so `range` is 1 in eight bits and the runner's `range` is a plain parameter.
- **Ruling 22 (Progressive's blur):** Progressive blur's Blur is a picture length in `pictureLengths` (max 256); the ramp (`Direction`, `Start`, `End`) is layer space and never scaled.
- **Ruling 24 (Spin and Zoom blur read the pyramid):** Spin and Zoom blur never halve the whole picture, because their sample gap grows with the distance from the Centre and halving by the farthest corner would soften the Centre, which must stay sharp (a 10° spin on a 1000 px layer would otherwise render at one eighth resolution). Instead each sample reads the pyramid B4 builds (Ruling 10's levels, made with the bloom downsample) at the level whose texel matches that pixel's own gap: `level = clamp(log2(gapDevice), 0, levels)`, a plain bilinear read at `floor(level)` and `ceil(level)` blended by the fraction, the standard mip read, never the tent. `gapDevice = |p − c| × angleRadians / count × resolution` (Spin) or `|p − c| × amount / count × resolution` (Zoom). Levels built: `ceil(log2(gapDevice at the farthest corner))`, 0 at small settings, at most 8. B3 therefore runs after B4 and consumes its pyramid runner. Cost: at the default on a 1000 px layer about three pyramid levels, so 6 to 8 draw calls instead of 4. Cost if wrong: a soft Centre, which B1 T21 pins on the CPU.
- **Ruling 23 (Bloom's excess order):** the excess is taken at the isolation target's own resolution and only then averaged down to the chain's first level (halved with the bloom's thirteen-tap filter while a halving stays within two times the first level's size, then one bilinear resample). Why: the threshold is not linear, so averaging the colour first would drop a thin highlight under the threshold and give it no glow at all, while reading one point per texel makes it pulse as it moves when a texel covers more than two screen pixels. Cost: nothing at the default Size, about three small extra draw calls at Size 512. Cost if wrong: a thin bright line's glow flickers as it moves at large Sizes, which B7 T13 catches on the CPU and the Size 512 proof case catches on the GPU.

## 2. Shared interfaces

### 2.1 `src/render-core/effects/effect-kernels.ts` (B1 produces; B2–B6, B9 consume)

```ts
export const LINE_BLUR_SAMPLES_V1 = Object.freeze({ draft: 16, full: 16, export: 48 }) as const
/** The largest gap between neighbouring samples in texels of the working picture. */
export const MAX_SAMPLE_GAP_TEXELS_V1 = 1

export interface LineSampleV1 { readonly s: number; readonly weight: number }
/** Even samples: s = (k + 0.5) / count − 0.5 two-sided, (k + 0.5) / count one-sided; weight 1 / count. */
export const lineBlurSamplesV1 = (count: number, oneSided: boolean): readonly LineSampleV1[]
/** min(1, resolutionScale, MAX_SAMPLE_GAP_TEXELS_V1 / gapDevicePixels); gap ≤ 0 → min(1, resolutionScale). */
export const blurWorkingScaleV1 = (gapDevicePixels: number, resolutionScale: number): number

/** A raster of linear premultiplied light with pixel centres at +0.5; outside its box every read is clear. */
export interface RasterV1 { readonly width: number; readonly height: number; readonly at: (x: number, y: number) => readonly [number, number, number, number] }
export interface CoverageV1 { readonly width: number; readonly height: number; readonly at: (x: number, y: number) => number }
export const sampleBilinearV1 = (raster: RasterV1, x: number, y: number): readonly [number, number, number, number]
export const sampleCoverageBilinearV1 = (coverage: CoverageV1, x: number, y: number): number
export const rasterFromPixelsV1 = (width: number, height: number, rgba: Float32Array | Float64Array): RasterV1

export const directionalBlurReferenceV1 = (source: RasterV1, lengthPixels: number, angleDegrees: number, count: number): RasterV1
/** Spin and Zoom read a pyramid (Ruling 24): `pyramid[0]` is the source, higher levels its bloom-downsampled halves. */
export const pyramidReadV1 = (pyramid: readonly RasterV1[], x: number, y: number, gapPixels: number): readonly [number, number, number, number]
export const sampledBlurLevelsV1 = (gapAtFarthestCorner: number): number   // ceil(log2(gap)), 0..PROGRESSIVE_MAX_LEVELS_V1
export const spinBlurReferenceV1 = (pyramid: readonly RasterV1[], angleDegrees: number, centre: PointV1, count: number): RasterV1
export const zoomBlurReferenceV1 = (pyramid: readonly RasterV1[], amount: number, centre: PointV1, count: number): RasterV1
export const motionSmearReferenceV1 = (source: RasterV1, vectorX: number, vectorY: number, count: number): RasterV1
export const gaussianBlurReferenceV1 = (source: RasterV1, sigma: number): RasterV1
export const sharpenReferenceV1 = (source: RasterV1, amount: number, sigma: number): RasterV1

export const GROW_SIGMA_RATIO_V1 = 1.5
export const GROW_CUT_V1 = 0.0668
export const SHRINK_CUT_V1 = 0.9332
/** Slope of a blurred straight edge at the cut: the normal density at 1.5 sigma, 0.12952. */
export const EDGE_SLOPE_V1 = 0.12952
export const edgeCutV1 = (blurred: number, cut: number, sigmaDevice: number, softnessDevice: number): number
export const coverageBlurReferenceV1 = (coverage: CoverageV1, sigma: number): CoverageV1
export const coverageGrowReferenceV1 = (coverage: CoverageV1, radius: number, softness: number): CoverageV1
export const coverageShrinkReferenceV1 = (coverage: CoverageV1, radius: number, softness: number): CoverageV1

export const longShadowStepsV1 = (lengthDevice: number): readonly number[]
export const longShadowReferenceV1 = (coverage: CoverageV1, lengthPixels: number, angleDegrees: number, fade: number): CoverageV1

export const PROGRESSIVE_MAX_LEVELS_V1 = 8
/** Sigma, in level-0 pixels, of each pyramid level as read at the output; index 0 is the sharp picture (0). Fitted once (B1 T13). */
export const PROGRESSIVE_LEVEL_SIGMA_V1: readonly number[]
export const progressiveLevelsNeededV1 = (sigmaMax: number): number
export interface ProgressiveBlendV1 { readonly lower: number; readonly upper: number; readonly upperWeight: number }
export const progressiveBlendV1 = (sigma: number, levels: number): ProgressiveBlendV1
export const progressivePyramidReferenceV1 = (source: RasterV1, levels: number): readonly RasterV1[]
export const progressiveReadReferenceV1 = (pyramid: readonly RasterV1[], x: number, y: number, sigma: number): readonly [number, number, number, number]
export const progressiveRampV1 = (along: number, start: number, end: number): number

export interface PointV1 { readonly x: number; readonly y: number }
export interface LayerFrameV1 { readonly transform: RenderTransformV0; readonly localBounds: RenderBoundsV0 }
/** The Centre (fractions of the layer's box) in picture pixels: through the frame when given, else onto `sourceBounds`. */
export const effectCentrePixelsV1 = (centreX: number, centreY: number, sourceBounds: RenderBoundsV0, frame?: LayerFrameV1): PointV1
export const directionalBlurBoundsV1 = (bounds: RenderBoundsV0, lengthPixels: number, angleDegrees: number): RenderBoundsV0
export const spinBlurBoundsV1 = (bounds: RenderBoundsV0, centre: PointV1): RenderBoundsV0
export const zoomBlurBoundsV1 = (bounds: RenderBoundsV0, amount: number, centre: PointV1): RenderBoundsV0
export const motionSmearBoundsV1 = (bounds: RenderBoundsV0, vectorX: number, vectorY: number): RenderBoundsV0
export const outlineBoundsV1 = (bounds: RenderBoundsV0, widthPixels: number, softness: number, position: 'outside' | 'centre' | 'inside'): RenderBoundsV0
export const chokeBoundsV1 = (bounds: RenderBoundsV0, amountPixels: number, softnessPixels: number): RenderBoundsV0
export const longShadowBoundsV1 = (bounds: RenderBoundsV0, lengthPixels: number, angleDegrees: number): RenderBoundsV0
export const progressiveBlurBoundsV1 = (bounds: RenderBoundsV0, blurPixels: number): RenderBoundsV0
```

### 2.2 Contract (B1 adds to `effect-contract.ts`; B2–B6, B9 fill the compile cases)

Instance types (values in contract units: fractions, composition px, degrees, unit colours):

```ts
export interface DirectionalBlurEffectV0 { effectId; definitionId: directionalBlur; semanticVersion: 0; enabled; angleDegrees: number; lengthPixels: number }
export interface SpinBlurEffectV0 { …; angleDegrees: number; centreX: number; centreY: number }           // centre as fractions, −1..2
export interface ZoomBlurEffectV0 { …; amount: number; centreX: number; centreY: number }               // amount 0..1
export interface SharpenEffectV0 { …; amount: number; radiusPixels: number }                              // amount 0..5
export interface OutlineEffectV0 { …; widthPixels: number; color: EffectColorV0; position: 'outside' | 'centre' | 'inside'; softness: number }
export interface ChokeEdgeEffectV0 { …; amountPixels: number; softnessPixels: number }
export interface BloomEffectV0 { …; threshold: number; amount: number; sizePixels: number; spread: number; tint: EffectColorV0 }   // B7
export interface ProgressiveBlurEffectV0 { …; blurPixels: number; directionDegrees: number; start: number; end: number }
export interface LongShadowEffectV0 { …; angleDegrees: number; lengthPixels: number; fade: number; color: EffectColorV0 }
export interface MotionSmearEffectV0 { …; amount: number; limitPixels: number; vectorXPixels: number; vectorYPixels: number }       // B9; vector from the worker, default 0
```

Compiled operation types: every one carries `effectId`, `semanticVersion: 0`, `coordinateSpace: 'composition-pixel'`, `colorSpace: 'linear-srgb'`, and extends `EffectMixV0`. The sampled blurs share:

```ts
interface CompiledSampledBlurV0 extends EffectMixV0 { readonly sampleCount: 16 | 48; readonly resolutionScale: number; readonly alphaMode: 'premultiplied' }
export interface CompiledDirectionalBlurV0 extends CompiledSampledBlurV0 { type: 'directional-blur'; angleDegrees; lengthPixels }
export interface CompiledSpinBlurV0 extends CompiledSampledBlurV0 { type: 'spin-blur'; angleDegrees; centreX; centreY }            // + transform, localBounds from the graph
export interface CompiledZoomBlurV0 extends CompiledSampledBlurV0 { type: 'zoom-blur'; amount; centreX; centreY }
export interface CompiledMotionSmearV0 extends CompiledSampledBlurV0 { type: 'motion-smear'; vectorXPixels; vectorYPixels }
export interface CompiledSharpenV0 extends CompiledShadowBlurV0 { type: 'sharpen'; amount }                                        // sigmaPixels = radius / 2
export interface CompiledOutlineV0 extends CompiledShadowBlurV0 { type: 'outline'; widthPixels; softnessPixels; position; color }   // sigmaPixels = width / 1.5 (centre: width / 3)
export interface CompiledChokeEdgeV0 extends CompiledShadowBlurV0 { type: 'choke-edge'; amountPixels; softnessPixels }             // sigmaPixels = |amount| / 1.5
export interface CompiledBloomV0 extends EffectMixV0 { type: 'bloom'; threshold; amount; sizePixels; spread; tint; alphaMode: 'premultiplied' }
export interface CompiledProgressiveBlurV0 extends EffectMixV0 { type: 'progressive-blur'; blurPixels; sigmaMaxPixels; levels: number; directionDegrees; start; end; alphaMode: 'premultiplied' }
export interface CompiledLongShadowV0 extends EffectMixV0 { type: 'long-shadow'; angleDegrees; lengthPixels; fade; color; alphaMode: 'premultiplied' }
```

Added to `COMPILED_EFFECT_OPERATION_TYPES_V0`, in this order, after `'wipe'`: `'directional-blur', 'spin-blur', 'zoom-blur', 'sharpen', 'outline', 'choke-edge', 'bloom', 'progressive-blur', 'long-shadow', 'motion-smear'`. Compile input gains `layerFrame?: LayerFrameV1` (Ruling 7).

### 2.3 Effect table rows (B1)

| type | cost class | layerSpace | pictureLengths (field, max, signed) |
| --- | --- | --- | --- |
| directional-blur | neighbourhood | false | lengthPixels 1024 |
| spin-blur | neighbourhood | true | none |
| zoom-blur | neighbourhood | true | none |
| sharpen | neighbourhood | false | radiusPixels 64 |
| outline | neighbourhood | false | widthPixels 128 |
| choke-edge | neighbourhood | false | amountPixels 64 signed, softnessPixels 64 |
| bloom | neighbourhood | false | sizePixels 512 |
| progressive-blur | neighbourhood | true | blurPixels 256 |
| long-shadow | neighbourhood | false | lengthPixels 1024 |
| motion-smear | neighbourhood | false | vectorXPixels 8192 signed, vectorYPixels 8192 signed, limitPixels 1024 |

Each definition's `lengths` names the same picture lengths and no layer lengths (Batch B has none: Centre, Direction, Start and End are points or fractions).

### 2.4 `src/render-core/shading/bloom-runner.ts` (B7 produces; B8 and 3D-D Task 11 consume)

```ts
export const BLOOM_REACH_TEXELS_V1: readonly [number, number, number, number, number, number]   // index = levels 1..5; index 0 unused (0)
export interface BloomPlanV1 { readonly levels: number; readonly texelDevicePixels: number; readonly excess: BloomLevelSizeV1; readonly halvings: number; readonly firstLevel: BloomLevelSizeV1; readonly sizes: readonly BloomLevelSizeV1[]; readonly shares: readonly number[] }
/** `excess` is the target's own size; `halvings` is how many times it is halved before the resample into `firstLevel` (Ruling 23). */
/** The chain for a layer: as many levels as keep one first-level texel at or above one device pixel. */
export const layerBloomPlanV1 = (sizeDevicePixels: number, targetWidth: number, targetHeight: number, spread: number): BloomPlanV1
export const layerBloomExcessV1 = (premultipliedEncoded: readonly [number, number, number, number], threshold: number, tint: Vector3V1): Vector3V1
export const layerBloomCompositeV1 = (premultipliedEncoded: readonly [number, number, number, number], glowLinear: Vector3V1, amount: number): readonly [number, number, number, number]
export const layerBloomReferenceV1 = (raster: EncodedRasterV1, effect: CompiledBloomV0, resolution: number): EncodedRasterV1

/** What the GPU runner (`native-webgl2-bloom.ts`, B8) takes. The Scene (3D-D Task 11) and the layer Bloom both build one. */
export interface BloomRunV1 {
  /** Draws the excess into the bound target of size `excess` (viewport set by the runner); returns its draw calls. */
  readonly writeExcess: (gl: WebGL2RenderingContext, size: BloomLevelSizeV1) => number
  /** The size the excess is written at. Equal to `firstLevel` (the Scene): written straight into the first level. Larger (a layer): halved with the downsample filter while a halving stays within 2× of `firstLevel`, then resampled once into it. */
  readonly excess: BloomLevelSizeV1
  readonly firstLevel: BloomLevelSizeV1
  readonly levels: number                 // 1..5
  readonly shares: readonly number[]      // bloomSpreadV1(spread, levels)
  readonly range: number                  // 8 for a Scene in eight bits, 1 for a layer, 1 with half-float targets
  readonly amount: number
  readonly destination: { readonly framebuffer: WebGLFramebuffer | null; readonly viewport: BloomViewportV1; readonly scissor?: BloomViewportV1 }
}
export interface BloomViewportV1 { readonly x: number; readonly y: number; readonly width: number; readonly height: number }
export interface BloomRunResultV1 { readonly drawCalls: number }
```

B8's runner: `runBloomChainV1(gl, programs, targetPool, keyPrefix, run: BloomRunV1): BloomRunResultV1` (VERIFY the backend's program and target-pool types; the shape above is the contract, the carrier types are the backend's).

### 2.5 `src/animation-core/composition-matrix.ts` and `src/animation-core/motion-smear.ts` (B9 produces; Batch H consumes)

```ts
export const compositionMatrixV1 = (document: ProjectDocument, overlay: Overlay, nodeId: NodeId): LocalTransformMatrixV1 | null
export const compositionPlaceV1 = (document, overlay, nodeId): PointV1 | null       // the anchor point through the matrix

export interface SmearVectorV1 { readonly x: number; readonly y: number }           // composition px per frame
export const SMEAR_X_PARAMETER_V1 = 'core.effect.smear-x'
export const SMEAR_Y_PARAMETER_V1 = 'core.effect.smear-y'
/** Per node: its place at `timeOf(node)` less its place one frame earlier; zero when the earlier time is under zero. */
export const smearVectorsV1 = (document, overlay: Overlay, nodeIds: readonly NodeId[], timeOf: (nodeId: NodeId) => ExactTime, overlayAt: (time: ExactTime) => Overlay): ReadonlyMap<NodeId, SmearVectorV1>
export const withSmearVectorsV1 = (projection, snapshot, overlay, timeOf, overlayAt): RenderValueProjectionV0
export const hasSmearVectorsV1 = (projection): boolean
```

## 3. Tasks

### Task B1: the kernels

**Files.** Created: `src/render-core/effects/effect-kernels.ts`, `src/render-core/effects/effect-kernels.test.ts`, `src/render-core/effects/effect-contract-batch-b.test.ts`. Modified: `effect-contract.ts` (instance and compiled types, operation types list, `layerFrame` on the compile input, the bounds rule per type, compile cases that return a diagnostic `Not built yet` at every quality so B2–B6/B9 replace one case each; VERIFY the diagnostic code the table rule expects), `effect-table.ts` (ten rows), the effect registry (ten definitions with `lengths`; VERIFY file name, Batch A's definitions are the model), `render-graph.ts` (pass `layerFrame` to the compiler; VERIFY the call site in `appendNodePasses`/`mergeEffects`).

**Interfaces.** Produces section 2.1 to 2.3. Consumes `effectBlurTapsV0` (reference for the Gaussian; VERIFY it is exported from the native backend module or move a copy of the discrete weights into the kernels file), `bloomDownsampleV1`, `bloomUpsampleV1`, `bloomLevelSizesV1`, `localTransformMatrixV1`-style `RenderTransformV0` (VERIFY field names a, b, c, d, tx, ty).

**Kernels, exact.**

1. **Samples.** `lineBlurSamplesV1(N, oneSided)`: `s_k = (k + 0.5) / N − 0.5` (two-sided) or `(k + 0.5) / N` (one-sided), weight `1 / N`. Directional: `out(p) = Σ w_k · in(p + s_k · L · d)`, `d = (cos a, sin a)`, `a` in degrees, y down. Spin: `in(c + R(s_k · A)(p − c))`, `R` rotating by degrees, positive clockwise on screen (y down). Zoom: `in(c + (p − c) · (1 + s_k · amount))`. Smear: `in(p + s_k · v)` one-sided. All reads bilinear, pixel centres at +0.5, outside clear (Ruling 8). The backend evaluates the same sum in the shader with the samples as a uniform array of 48 and `uSampleCount`.
2. **Working scale.** `gap = pathDevice / N`; `scale = blurWorkingScaleV1(gap, profile.resolutionScale)`; the backend halves as `#blurToWorkingTarget` does until `ceil(w · scale)`. `pathDevice` = `lengthPixels · resolution` (Directional) or `|v| · resolution` (Smear). Spin and Zoom do not use it: they read the pyramid per pixel (Ruling 24), with `sampledBlurLevelsV1` of the gap at the farthest corner of `localBounds` through `transform` deciding how many levels the backend builds (`rFar` from `sourceBounds` in the compiler, for the operation's `levels` field).
3. **Pyramid** (Ruling 10). `progressivePyramidReferenceV1(source, levels)`: level 0 = source; sizes `bloomLevelSizesV1(w, h, levels + 1)`; level k = `bloomDownsampleV1(level k−1, u, v)` per texel centre. Read: level 0 plain bilinear; level k ≥ 1 by `bloomUpsampleV1(level_k, u, v)` with `u = x / w`. `PROGRESSIVE_LEVEL_SIGMA_V1` is fitted (T13): a unit impulse at the centre of a 1024 × 1 row (height padded to 8) through the pyramid, each level read back at every level-0 pixel, σ_k = sqrt of the second moment about the impulse. Expected near `[0, 1.94, 4.09, 8.29, 16.6, 33.3, 66.6, 133, 266]` (closed form `σ_k² = 1.75 · (4^k − 1) / 3 + 0.5 · 4^k`); the fit decides, the test pins each to 1 %. `progressiveLevelsNeededV1(σ)` = smallest k with `σ_k ≥ σ`, at least 1, at most 8. `progressiveBlendV1(σ, levels)`: lower = largest k ≤ levels with σ_k ≤ σ, upper = min(lower + 1, levels), `upperWeight = (σ² − σ_lower²) / (σ_upper² − σ_lower²)` clamped to [0, 1] (a mixture's variance is the weighted mean of variances, so the blend has the wanted σ exactly); at or above σ_levels the weight is 1. `progressiveRampV1(along, start, end)` = `end > start ? clamp((along − start) / (end − start), 0, 1) : (along >= start ? 1 : 0)`. `along` = projection of the layer point on `(cos dir, sin dir)` across the box, normalised to 0..1 over the box's extent along that direction.
4. **Sharpen.** `b = gaussianBlurReferenceV1(in, radius / 2)`; `out.rgb = clamp(in.rgb + amount · (in.rgb − b.rgb), 0, in.a)`, `out.a = in.a`.
5. **Grow and shrink.** `coverageBlurReferenceV1(coverage, r / 1.5)`; `edgeCutV1(b, cut, σ_dev, soft_dev)`: `h = EDGE_SLOPE_V1 / σ_dev · (0.5 + soft_dev) / 2`; `smoothstep(cut − h, cut + h, b)`; σ_dev under 1e-3 → a hard step at the cut. Grow: cut `GROW_CUT_V1`; shrink: `SHRINK_CUT_V1`. Why those numbers: Φ(−1.5) = 0.0668, and a straight edge blurred by σ = r / 1.5 reads 0.0668 at exactly r outside it.
6. **Long shadow.** `d = (cos a, sin a)`; `S_0 = coverage`; for each step `t` of `longShadowStepsV1(L_dev)`, `S_next(p) = max(S(p), S(p − t · d) − fade · t / L_dev)`, reads bilinear; `shadow = max(0, S_last)`. Steps: `[1, 2, 4, …, 2^(K−2), L − (2^(K−1) − 1)]` with `K = ceil(log2(L + 1))`, L ≥ 1; L < 1 → no steps. Exact because the penalty is linear in distance and every integer in 0..L is a sum of distinct steps.
7. **Bounds** (composition px, applied to the cumulative `outputBounds`, floored/ceiled outward to whole px): Directional `x ± ceil(L/2 · |cos a|)`, `y ± ceil(L/2 · |sin a|)`. Spin: `r = max over the 4 corners |corner − c|`, box `c ± ceil(r)`. Zoom: `c + (bounds − c) / (1 − amount / 2)`, ceiled outward. Smear: `vx > 0 → x −= ceil(vx)`, `vx < 0 → width += ceil(|vx|)`, same for y. Outline: outside `+ceil(W + W·soft + 1)`; centre `+ceil(W/2 + W·soft/2 + 1)`; inside `0`. Choke: `amount < 0 → +ceil(|amount| + 3·soft + 1)`, else 0. Bloom: `+ceil(Size)`. Progressive: `+ceil(3 · Blur / 2)` on every side. Long shadow: union with the box moved by `L · d`. The `+1` on coverage effects covers the half-device-pixel cut band at any resolution.
8. **Nothing when** (no operation, no scope): Directional `L = 0`; Spin `A = 0`; Zoom `amount = 0`; Sharpen `amount = 0`; Outline `W = 0` or `color.a = 0`; Choke `amount = 0 && soft = 0`; Bloom `amount = 0 || threshold ≥ 1`; Progressive `Blur = 0`; Long shadow `L = 0 || color.a = 0`; Smear `|v| < 1e-3` after Amount and Limit. Each `INVALID_PARAMETER` when a value is non-finite or outside its range after scaling (ranges of 3.2; Centre −1..2).
9. **Draw calls at defaults** (target, to be measured in B10): Directional 4, Spin 6 to 8, Zoom 6 to 8, Sharpen 7, Outline 6, Choke 7, Bloom 11, Progressive 7, Long shadow 9, Smear 4. All within 3 to 11.

**Rules (each a test).**
- R1 samples are even, sum to 1, symmetric two-sided and start at 0 one-sided.
- R2 the working scale is 1 when the gap is at most one texel and halves the gap to one texel otherwise, never above the profile's scale.
- R3 each sampled reference matches a hand case (below).
- R4 bounds match the design's table at angles 0, 45, 90 and a Centre outside the layer.
- R5 grow by r moves a straight edge by r ± 0.05 px; shrink likewise; a square's corner rounds (its diagonal grows by less than r√2).
- R6 the long-shadow recurrence equals the direct maximum over every integer distance.
- R7 the pyramid sigmas match the table to 1 %, and the blend has the wanted variance.
- R8 the far end of Progressive 40 vs Layer blur 40: mean ≤ 2, max ≤ 12 levels (stop point).
- R9 full and export compile the same operation except `sampleCount`, `passes`, `kernelSize`, `resolutionScale`.
- R10 every definition's `lengths` equals its row's `pictureLengths` fields.
- R11 the sixteen existing effects compile to the same stacks (deep equal to a pinned fixture) and the three differentials stay green.
- R12 bounds are the same at every quality.

**Tests by name** (`effect-kernels.test.ts` unless noted).
- T1 `lineBlurSamplesV1: 16 two-sided samples are symmetric and sum to 1`: asserts `s_0 = −15/32`, `s_15 = 15/32`, Σw = 1 ± 1e-12.
- T2 `blurWorkingScaleV1: gap 2.5 texels halves to 0.4; gap 0.5 stays 1; draft caps at 0.5`.
- T3 `directionalBlurReferenceV1: a 1-px white column at x=10 on 32×4, length 8 angle 0, 16 samples`: every pixel x in 6..13 reads 1/8 ± 1e-9 of white in premultiplied light (bilinear spreads each sample over two pixels; assert the row sums to 1 and values at x=6 and x=13 are 1/16, x=7..12 are 1/8).
- T4 `directionalBlurReferenceV1: angle 90 moves the streak to the y axis`: the column case transposed, equal to T3's numbers.
- T5 `spinBlurReferenceV1: a white pixel at (20,10) about centre (10,10), angle 180, 48 samples` (gap 0.65 px, so level 0 only; the pyramid is `progressivePyramidReferenceV1(source, 0)`): alpha summed over the raster equals 1 ± 0.02, and every lit pixel lies at distance 10 ± 0.75 from the centre.
- T6 `zoomBlurReferenceV1: a white pixel at (20,10) about (10,10), amount 1`: lit pixels lie on the row y=10 between x=15 and x=25 (±0.75), alpha sums to 1 ± 0.02.
- T7 `motionSmearReferenceV1: a white pixel at x=10, vector (−8, 0)`: lit pixels at x in 2..10 only (trail on the far side of the move), sums to 1 ± 1e-9.
- T8 `sharpenReferenceV1: amount 1, sigma 1 on a step edge`: the pixel on the bright side of the edge rises above the flat value and is clamped at alpha; a flat region is unchanged to 1e-12.
- T9 `coverageGrowReferenceV1: a 64×64 square in 160×160, r 8, softness 0`: edge crossing 0.5 at 8 ± 0.05 px outside (interpolate the row through the middle); T10 `shrink` likewise inside; T11 `corner rounds`: the diagonal extent grows by between 8 and 8√2 − 0.5.
- T21 `spinBlurReferenceV1 keeps the Centre sharp`: a 512 × 512 raster with a white pixel at the Centre (256, 256) and a white pixel at (500, 256), angle 20°, 16 samples: the Centre pixel stays 1 ± 1e-9; the far pixel's lit arc reads levels 2 and 3 (`pyramidReadV1` called with gap `244 × 0.349 / 16 = 5.3` px chooses `log2(5.3) = 2.41`); the arc's alpha sums to 1 ± 0.05.
- T22 `pyramidReadV1 picks the level by the gap`: gap 1 → level 0 exactly (equal to `sampleBilinearV1` on the source); gap 4 → level 2 exactly; gap 2.83 → half level 1, half level 2 (± 1e-9 against the two reads blended).
- T12 `longShadowReferenceV1 equals the direct maximum`: 48×48 random coverage, length 20, angle 30, fade 0.5: recurrence vs `max_s` over s = 0..20 integers with the same bilinear reads, equal to 1e-9. And `longShadowStepsV1(20) = [1,2,4,8,5]`, `(1) = [1]`, `(0.5) = []`.
- T13 `PROGRESSIVE_LEVEL_SIGMA_V1 is the measured pyramid sigma`: impulse fit as in kernel 3; each entry within 1 % of the measured value; prints the measured table once.
- T14 `progressiveBlendV1(20, 5) mixes levels 4 and 5 to variance 400 ± 1e-6`.
- T15 `Progressive 40 at its far end matches Layer blur 40` (the stop point): 320 × 180 fixture (a 120 × 80 white box plus a 50 % grey disc), sigma 20 everywhere (ramp t = 1), progressive reference vs `gaussianBlurReferenceV1(σ 20)`, compared in eight-bit encoded values: mean ≤ 2, max ≤ 12. If it fails: STOP.
- `effect-contract-batch-b.test.ts`: T16 `bounds per effect at 0, 45, 90 degrees and a Centre at (−50, 150) %` with the numbers of kernel 7 on a source box (100, 50, 200, 100): Directional L 40: 0° → x 80..320, y unchanged; 45° → ±15 both (ceil(14.14)); 90° → y ±20. Spin, Centre (−50 %, 150 %) → c = (0, 200): r = |(300, 50) − (0, 200)| = 335.4 → box (−336, −136, 672, 672). Zoom amount 1 about (200, 100): box scaled ×2 about c → (0, 0, 400, 200). Smear (12, −7) → (88, 50, 212, 107).
- T17 `full and export differ only in sampling and profile`; T18 `bounds are the same at draft, full and export`; T19 `lengths match pictureLengths` for all 26 definitions; T20 `existing stacks unchanged`: the sixteen Batch A and original effects compiled at full and export against a pinned JSON fixture written before this task (`Object.is` per leaf).

**Steps.**
1. Write T1–T15 and T16–T20 (T20's fixture is recorded from `main` first: a script writes the compiled stacks of the sixteen effects to `effect-contract-batch-b.fixture.json`; commit that alone: `test(effects): pin compiled stacks of the sixteen effects before Batch B`).
2. Run the new tests: expected FAIL (module missing).
3. Implement `effect-kernels.ts`; contract types; operation types list; table rows; definitions; `layerFrame`; graph call site; bounds rules.
4. Run the new tests and the three differential tests: expected PASS, no expectation changed.
5. Typecheck; Prettier on touched files.
6. Commit: `feat(effects): Batch B kernels, contract types and table rows (B1)`.

**Stop points.** T15 fails at the bar. T20 or a differential changes. A definition needs a schema step. `effectBlurTapsV0` is not reachable from render-core (then copy the discrete weights only and say so in the commit).

**Code the executor needs: the blend and the cut.**
```ts
export const progressiveBlendV1 = (sigma: number, levels: number): ProgressiveBlendV1 => {
  const top = Math.min(levels, PROGRESSIVE_MAX_LEVELS_V1)
  if (!(sigma > 0)) return { lower: 0, upper: 0, upperWeight: 0 }
  let lower = 0
  while (lower + 1 <= top && PROGRESSIVE_LEVEL_SIGMA_V1[lower + 1]! <= sigma) lower += 1
  const upper = Math.min(lower + 1, top)
  if (upper === lower) return { lower, upper, upperWeight: 1 }
  const a = PROGRESSIVE_LEVEL_SIGMA_V1[lower]! ** 2
  const b = PROGRESSIVE_LEVEL_SIGMA_V1[upper]! ** 2
  return { lower, upper, upperWeight: Math.min(1, Math.max(0, (sigma * sigma - a) / (b - a))) }
}
export const edgeCutV1 = (blurred: number, cut: number, sigmaDevice: number, softnessDevice: number): number => {
  if (!(sigmaDevice > 1e-3)) return blurred >= cut ? 1 : 0
  const half = (EDGE_SLOPE_V1 / sigmaDevice) * (0.5 + Math.max(0, softnessDevice)) * 0.5
  const t = Math.min(1, Math.max(0, (blurred - (cut - half)) / (2 * half)))
  return t * t * (3 - 2 * t)
}
```

### Task B7: the bloom core's level parameter, and the layer Bloom's contract

**Files.** Created: `src/render-core/shading/bloom-runner.ts`, `src/render-core/shading/bloom-runner.test.ts`, `src/render-core/shading/bloom-levels.test.ts`. Modified: `src/render-core/shading/bloom.ts` (the optional `levels` parameter only), `effect-contract.ts` (the `bloom` compile case replaces B1's placeholder), `effect-table.ts` row already from B1 (no edit), the Bloom definition from B1 (no edit). Not touched: 3D-D's bloom tests (VERIFY their file names; if any needs an edit: STOP).

**Interfaces.** Produces section 2.4. Consumes `bloomExcessV1`, `bloomChainV1`, `bloomSpreadV1`, `bloomLevelSizesV1`, `bloomGlowV1`, `bloomCompositeV1`, `linearToSrgbV1`/`srgbToLinearV1` (VERIFY the decode function's name in `pbr.ts` or the colour module).

**1. The additive change.**
```ts
export const bloomSpreadV1 = (spread: number, levels: number = BLOOM_LEVELS_V1): number[] => {
  const count = Math.min(BLOOM_LEVELS_V1, Math.max(1, Math.floor(levels)))
  // body unchanged except `level < count`
}
export const bloomChainV1 = (bright: BloomPictureV1, spread: number, levels: number = BLOOM_LEVELS_V1): BloomPictureV1 => {
  const count = Math.min(BLOOM_LEVELS_V1, Math.max(1, Math.floor(levels)))
  const sizes = bloomLevelSizesV1(bright.width, bright.height, count)
  const shares = bloomSpreadV1(spread, count)
  // body unchanged with `count` for BLOOM_LEVELS_V1 and `last = count − 1`
}
```
With `levels` left out the loop bounds are the same numbers, so every float operation happens in the same order: bit equality is by construction, and T1/T2 prove it.

**2. K.** `BLOOM_REACH_TEXELS_V1[n]`, n = 1..5: a 256 × 256 picture, one texel of light 1 at (128, 128), `bloomChainV1(picture, 1, n)`; the reach is the largest `|dx|` on row 128 with glow red strictly above 0 (and the same on column 128; assert equal). Expected about `[0, 1, 7, 19, 43, 91]`? No: fit, do not guess. The design says about 63 for five levels; the test prints the five values once, the executor writes them into the constant, and the test then asserts equality. The written values are the contract. VERIFY by running T3.

**3. The layer Bloom's contract.** Instance `BloomEffectV0` (2.2): `threshold` 0..1, `amount` 0..2, `sizePixels` 4..512 (picture length, scaled by the view, clamped to 512), `spread` 0..1, `tint` unit colour (alpha ignored). Nothing when `amount === 0 || threshold >= 1`. Diagnostic `INVALID_PARAMETER` when any value is non-finite or outside its range, or `sizePixels < 4` after scaling (a zoom-out can take it under 4: clamp up to 4 instead, since a smaller Size than one chain texel at four levels means nothing anyway; VERIFY `scaleRenderEffectInstancesV0` clamps only the maximum, then clamp the minimum in the compile case). Compiled `CompiledBloomV0` carries the five values plus `alphaMode: 'premultiplied'`. Bounds: `+ceil(sizePixels)` every side. Quality: identical at draft, full, export.

`layerBloomPlanV1(sizeDevice, targetW, targetH, spread)`:
```ts
let levels = BLOOM_LEVELS_V1
let texel = sizeDevice / BLOOM_REACH_TEXELS_V1[levels]
while (levels > 1 && texel < 1) { levels -= 1; texel = sizeDevice / BLOOM_REACH_TEXELS_V1[levels] }
texel = Math.max(1, texel)
const firstLevel = { width: Math.max(1, Math.ceil(targetW / texel)), height: Math.max(1, Math.ceil(targetH / texel)) }
let halvings = 0
let w = targetW, h = targetH
while (Math.ceil(w / 2) >= firstLevel.width && Math.ceil(h / 2) >= firstLevel.height && (w > 2 * firstLevel.width || h > 2 * firstLevel.height)) { w = Math.ceil(w / 2); h = Math.ceil(h / 2); halvings += 1 }
return { levels, texelDevicePixels: texel, excess: { width: targetW, height: targetH }, halvings, firstLevel, sizes: bloomLevelSizesV1(firstLevel.width, firstLevel.height, levels), shares: bloomSpreadV1(spread, levels) }
```
`sizeDevice = sizePixels × graph.resolution`, computed by the backend; the plan is pure and tested on the CPU.

**4. Excess and composite.**
```ts
export const layerBloomExcessV1 = (pixel, threshold, tint) => {
  const a = pixel[3]
  if (!(a > 0)) return [0, 0, 0]
  const straight = [decode(pixel[0] / a), decode(pixel[1] / a), decode(pixel[2] / a)]   // sRGB → linear, each held to 0..1
  const over = bloomExcessV1(straight, threshold)
  return [over[0] * tint[0] * a, over[1] * tint[1] * a, over[2] * tint[2] * a]           // alpha put back: premultiplied excess
}
export const layerBloomCompositeV1 = (pixel, glowLinear, amount) => {
  const glow = bloomGlowV1(glowLinear, amount)                                           // soft clip kept (Ruling 4)
  return [
    Math.min(1, glow[0] + pixel[0] * (1 - glow[0])),
    Math.min(1, glow[1] + pixel[1] * (1 - glow[1])),
    Math.min(1, glow[2] + pixel[2] * (1 - glow[2])),
    Math.min(1, glow[3] + pixel[3] * (1 - glow[3]))
  ]
}
```
The pixel is the isolation target's value as stored: sRGB-encoded, premultiplied (VERIFY: the native blur's decode mode 0 says the isolation target holds encoded values). The glow is a display value, so the screen is in display space, as the Scene's. Premultiplied validity holds: each colour channel of the result is at most the result's alpha (proof: `g_c ≤ g_a`, `p_c ≤ p_a`, so `g_c + p_c(1 − g_c) ≤ g_a + p_a(1 − g_a)`).

`layerBloomReferenceV1(raster, effect, resolution)` (Ruling 23): the excess picture at the target's own size, one `layerBloomExcessV1` per device pixel; halved `plan.halvings` times with `bloomDownsampleV1` (each level `bloomLevelSizesV1`'s next size); then resampled into the first level by `bloomSampleV1` at each first-level texel's centre (a bilinear read, never more than 2:1); `bloomChainV1(first, spread, levels)`; then for each device pixel the glow read by `bloomSampleV1` at the pixel's centre and `layerBloomCompositeV1`. The GPU does the same: a full-quad excess pass at the target's size, `halvings` downsample passes with the chain's own program, one transfer pass into the first level. At the default Size the first level is near the target's size, so `halvings` is 0 and the transfer pass is the only addition.

**5. The runner interface** (2.4) and what Task 11 assumes, kept: the levels' sizes are `bloomLevelSizesV1` of the first level; level k+1 is `bloomDownsampleV1` of level k with `texel = 1 / size_k`; the glow starts as the last level × its share and goes up by `own × share + bloomUpsampleV1(under)`; the last pass draws `bloomGlowV1(glow, amount)` with `blendFunc(ONE, ONE_MINUS_SRC_COLOR)` for colour and `(ONE, ONE_MINUS_SRC_ALPHA)` for alpha; in eight bits a level holds `excess / range` and the glow pass multiplies by `range` before `bloomGlowV1`. The runner takes `BloomRunV1` and does exactly that; `writeExcess` is the only part a caller owns. When `excess` is larger than `firstLevel` the runner halves it with the downsample program while a halving stays within 2× of `firstLevel`, then resamples once into the first level (Ruling 23); when equal, `writeExcess` draws straight into the first level and no pass is added. Target keys: `${keyPrefix}:bloom:${level}`. The runner releases every level in `finally` and restores the blend state and framebuffer it found.

**The line for 3D-D's Task 11:** "Scene bloom does not write level passes of its own. At `depth.clear` it builds a `BloomRunV1` (`src/render-core/shading/bloom-runner.ts`, Effects Batch B task B7) with `writeExcess` drawing the Scene's solids through the bloom-excess shading and other layers through the coverage program, `firstLevel = bloomLevelSizesV1(ceil(w/2), ceil(h/2))[0]`, `excess` equal to `firstLevel` (the Scene rasterises its excess at half resolution, so the runner halves nothing), `levels: 5`, `shares: bloomSpreadV1(spread)`, `range: 8` in eight bits or 1 with half-float targets, `destination` the canvas framebuffer with the Scene's clip as `scissor`, and calls `runBloomChainV1` from `native-webgl2-bloom.ts` (written in task B8). `scene.bloom` then has nothing left to draw."

**6. The CPU reference for B8's proof.** Fixture `bloom-layer` in `renderer-backend-fixtures.ts` (B8 adds it; B7 names it): 320 × 180, a 96 × 64 white box (over the threshold), a 40 × 40 patch at 50 % grey (0.214 linear, under the default threshold 0.7), a 40 × 40 patch at 90 % white with alpha 0.5 (over the threshold, translucent), and a 1-pixel-wide white line turned 30° (anti-aliased edge). Effect at Threshold 70 %, Amount 100 %, Size 64, Spread 50 %, Tint white; a second case Tint (1, 0.5, 0.2); a third case Size 512 (a first-level texel of about 8 screen pixels), which is where the thin line exercises Ruling 23. Tolerance: the neighbourhood class as the layer blur: mean ≤ 1, max ≤ 8, measured over the whole canvas; reported separately over the pixels the bypass frame leaves clear (the glow outside the shape), which must not be all zero (the fixture must light at least 500 such pixels). Fault to see caught before it counts: in the excess shader, take the excess after `encodeSrgb` instead of before (3D-D's required mutation, mirrored): the spec must fail. Second fault, seen caught by the Size 512 case: write the excess straight into the first level by one read per texel (the spec must fail on the thin line). Pixi: reported, never asserted.

**Rules (each a test).**
- R1 `bloomSpreadV1(s)` with `levels` left out: `Object.is` every share to the pre-change value for s in {0, 0.25, 0.5, 1}.
- R2 `bloomChainV1(picture, s)` with `levels` left out: byte-equal to the pre-change picture on a 64 × 36 random fixture (Float64Array compared with `Object.is` per element).
- R3 `bloomSpreadV1(s, n)` returns n shares summing to 1 ± 1e-12, ratio between neighbours `3^(2s−1)`.
- R4 `bloomChainV1(p, s, 1)` equals the picture itself times 1 (share of one level is 1).
- R5 `BLOOM_REACH_TEXELS_V1` equals the measured support for n = 1..5.
- R6 the plan drops finest levels until a texel is ≥ 1 device px, never below 1 level; at Size 64 × resolution 1 it keeps the most levels whose reach fits (print the levels for Size 4, 16, 64, 512).
- R7 no glow leaves the pixel to the bit; the result never passes 1; a glow of linear 0.8 at Amount 1 gives display `linearToSrgb(0.8)` and a glow of 2 gives `linearToSrgb(softClip(2)) < 1` (pins Ruling 4 with the numbers printed).
- R8 Threshold 1 or Amount 0 compiles to no operation; Threshold 0.999 compiles to one.
- R9 the excess of a clear pixel is 0; of a 50 % covered white pixel at threshold 0.7 it is `0.3 × 0.5` per channel; the Tint scales it.
- R10 bounds grow by `ceil(Size)` on every side at every quality; the operation is identical at draft, full and export.
- R11 `layerBloomReferenceV1` on the fixture lights at least 500 pixels outside the bypass shape and none beyond `Size` from it.
- R12 3D-D's tests pass unedited.
- R13 a one-pixel bright line's total glow is the same wherever the line sits within a first-level texel (Ruling 23).

**Tests by name.** `bloom-levels.test.ts`: T1 `bloomSpreadV1 without levels is bit-identical` (R1; the pre-change values are pinned as literals recorded from `main` before the edit: write the test first, with `toBe` on each number printed from `main`); T2 `bloomChainV1 without levels is byte-identical` (R2; the fixture's output hashed with a stable digest recorded from `main`, plus a 20-element spot check with `Object.is`); T3 `BLOOM_REACH_TEXELS_V1 is the chain's support` (R5); T4 `bloomSpreadV1 with 1..4 levels` (R3, R4). `bloom-runner.test.ts`: T5 `layerBloomPlanV1 keeps a texel at or above one device pixel` (R6, with Size 4 → levels 1 or 2 depending on K, asserted from the table, not a literal); T6 `layerBloomCompositeV1 leaves a pixel to the bit without glow` (R7, 256 greys × 4 alphas); T7 `the soft clip holds the glow under 1` (R7 numbers); T8 `premultiplied validity` (R7: 1000 random pixels and glows, every channel ≤ alpha + 1e-12); T9 `layerBloomExcessV1` (R9); T10 `layerBloomReferenceV1 on the bloom-layer fixture` (R11); T13 `a thin line's glow energy does not depend on its sub-texel place` (R13: 320 × 180, Size 512 at resolution 1, Threshold 70 %, a one-pixel-wide white vertical line at x = 100 + k for k = 0..7, the sum of the glow's alpha over the picture is above 0 and varies by under 1 % across the eight places; recorded once with the old one-read-per-texel reference it varies by over 50 %, so the test is seen failing first); T14 `layerBloomPlanV1 halvings` (Size 512 on 320 × 180 → `halvings` 2 or 3 as the 2× rule gives from the measured K, asserted from the rule, not a literal; Size 64 → 0). `effect-contract-batch-b.test.ts` gains T11 `bloom compiles to nothing at Threshold 100 % or Amount 0` and T12 `bloom bounds and quality` (R8, R10).

**Steps.**
1. Record the pre-change literals for T1 and the digest for T2 from `main` (a one-off script in the scratchpad; paste the numbers into the tests). Write T1–T14. Run: T1, T2 PASS (nothing changed yet), the rest FAIL.
2. Add `levels` to `bloomSpreadV1` and `bloomChainV1`. Run T1–T4 and 3D-D's bloom tests: PASS, unedited.
3. Run T3 once to print the reach; write the five numbers into `BLOOM_REACH_TEXELS_V1`; run again: PASS.
4. Implement `bloom-runner.ts` (plan, excess, composite, reference, the `BloomRunV1` types) and the `bloom` compile case. Run all: PASS.
5. Typecheck; Prettier; commit: `feat(effects): bloom core level parameter and the layer Bloom contract (B7)`.

**Stop points.** T1 or T2 differ after the edit. Any 3D-D test needs an edit. The measured reach for five levels is outside 48..80 (then the chain is not what 5.1 describes; report the number). The isolation target turns out not to hold encoded sRGB (then the excess and composite references change; report before writing them).

### Task B9: Motion smear

**Files.** Created: `src/animation-core/composition-matrix.ts` (+ `.test.ts`), `src/animation-core/motion-smear.ts` (+ `.test.ts`), `src/render-core/effects/motion-smear.test.ts` (contract, kernel, bounds, cache), a `motion-smear` fixture in `renderer-backend-fixtures.ts` and its proof case. Modified: `render.worker.ts` (rename, then the smear vectors at the four sites, `timeOf` at the shutter site), `effect-contract.ts` (the `motion-smear` compile case replaces B1's placeholder), the Motion smear definition (two hidden parameters, the grain seed's pattern; VERIFY how `EFFECT_GRAIN_SEED_PARAMETER_V1` is declared so it reaches the instance without the inspector showing it), `native-webgl2-effects.ts` SPECS entry (one-sided sampled blur; shares the sampled-blur prelude B2 writes, so B9 runs after B2).

**Interfaces.** Produces 2.5 and `CompiledMotionSmearV0`. Consumes `localTransformMatrixV1`, `HistoricalAnimationSamplerV1` through the worker's `historicalOverlayAt`, `frameDuration`, `subtractExactTime`, `exactSecondsV1`, `lineBlurSamplesV1`, `blurWorkingScaleV1`, `motionSmearReferenceV1`, `motionSmearBoundsV1`.

**1. Composition place.** `compositionMatrixV1(document, overlay, nodeId)`: collect the chain from the node to the root (VERIFY the parent link field, `node.parentId`?), then multiply root first: `M = M_root · … · M_node`, each `localTransformMatrixV1({ x, y, rotation, scaleX, scaleY, anchorX, anchorY, width, height })` read as the Scene clip check reads them (`overlay[`${id}/${propertyId}`]?.value` when a number, else the stored property, else the default; VERIFY the nine property ids there). Returns null when a node in the chain is missing. `compositionPlaceV1` = `M` applied to `(anchorX × width, anchorY × height)` of the node. Matrix product in the `{a, b, c, d, tx, ty}` form: `(P·Q)` = `{ a: P.a·Q.a + P.c·Q.b, b: P.b·Q.a + P.d·Q.b, c: P.a·Q.c + P.c·Q.d, d: P.b·Q.c + P.d·Q.d, tx: P.a·Q.tx + P.c·Q.ty + P.tx, ty: P.b·Q.tx + P.d·Q.ty + P.ty }` (column vectors, VERIFY against `localTransformMatrixV1`'s own convention by composing `T(10,0)` with `R(90)` and checking the point (1, 0) lands at (10, 1) with y down). Auto-layout: named limit (Ruling 5). The Scene clip check and the Distribute closure are not changed.

**2. The vectors.**
```ts
export const smearVectorsV1 = (document, overlay, nodeIds, timeOf, overlayAt) => {
  const step = frameDuration(document.settings.frameRate)
  const found = new Map<NodeId, SmearVectorV1>()
  const earlierOverlays = new Map<string, Overlay>()          // keyed by the serialized earlier time
  for (const nodeId of nodeIds) {
    const now = timeOf(nodeId)
    const earlier = subtractExactTime(now, step)
    if (earlier.value < 0n) { found.set(nodeId, { x: 0, y: 0 }); continue }      // Ruling 14
    const nowPlace = compositionPlaceV1(document, overlay, nodeId)
    const key = serializeExactTime(earlier)
    const past = earlierOverlays.get(key) ?? overlayAt(earlier); earlierOverlays.set(key, past)
    const then = compositionPlaceV1(document, past, nodeId)
    found.set(nodeId, nowPlace && then ? { x: nowPlace.x - then.x, y: nowPlace.y - then.y } : { x: 0, y: 0 })
  }
  return found
}
```
`withSmearVectorsV1(projection, snapshot, overlay, timeOf, overlayAt)`: for every node with an enabled Motion smear item (VERIFY `effectsOfNodeV1` and the definition id constant), take its vector; skip when both components are exactly 0; else write `[SMEAR_X_PARAMETER_V1]: x, [SMEAR_Y_PARAMETER_V1]: y` into `effects[node][item]` the way the grain seed is written; return the projection unchanged when nothing was written. `hasSmearVectorsV1`: any smear parameter present (they are only written when non-zero).

**3. The worker.** Step one (own commit): rename `withGrainSeeds` → `withEffectClocks`, `hasGrainSeeds` → `hasEffectClocks`, bodies untouched, the four call sites and the scheduling check renamed. Step two: `withEffectClocks(projection, snapshot, time, overlay, timeOf = () => time)` ends with `return withSmearVectorsV1(withGrain, snapshot, overlay, timeOf, (at) => historicalOverlayAt(snapshot, at))`; `hasEffectClocks = hasRippleTimesV1 || grain seed present || hasSmearVectorsV1`. The four sites pass their overlay (`overlay` at the normal, recovery and export sites; `compositionOverlay` at the shutter site; VERIFY the variable names) and the shutter site passes `timeOf = (id) => crisp.has(id) ? time : sampleTime` (Ruling 13; VERIFY `crisp` is a Set of node ids). The scheduling check is unchanged in form.

**4. The contract.** `v = (vectorXPixels, vectorYPixels) × amount`; `len = hypot(v)`; `if (len > limitPixels) v = v × limitPixels / len`; `if (len_after < 1e-3) → no operation`. `sampleCount = LINE_BLUR_SAMPLES_V1[quality]`, `resolutionScale = QUALITY_PROFILES[quality].resolutionScale`. Diagnostics: amount outside 0..4, limit outside 0..1024, a non-finite vector. The vector's two fields and the limit are picture lengths (scaled by the view, 2.3), so `v` is in view-scaled composition px when compiled, as a shadow's offset is. Bounds `motionSmearBoundsV1` (B1 kernel 7). The pass carries `vectorXPixels`, `vectorYPixels`, `sampleCount`, `resolutionScale` (Ruling 20). Backend: `gap = hypot(v) × resolution / sampleCount`, `blurWorkingScaleV1`, halve, one sampled pass with `lineBlurSamplesV1(sampleCount, true)` reading `in(p + s × v × resolution × workingScale)` in texels, encode back. 4 draw calls at the default.

**5. Shutter:** Ruling 2 and 13. Named limits (on the limits list): the two blurs stack; it follows the layer's place only, not its turning, scaling or the camera; a layer placed by auto-layout smears by its parent's move only; the first frame of a loop shows no smear; every Echo copy of a layer trails by the layer's own vector, not by the copy's own past move (one vector per effect item; giving each copy its own is Batch H's, at one more historical evaluation per copy).

**6. Cost.** One `historicalOverlayAt(time − step)` per frame, requested once by `smearVectorsV1` (the local map) and shared with an Echo of spacing 1 by the sampler's cache by exact time (VERIFY the cache key is the serialized exact time, so the Echo's `subtractExactTime(time, scaleExactTime(step, 1n))` and the smear's `subtractExactTime(time, step)` are the same key). Under a shutter: one transient replay per sample. Budget: the sampler's existing 256 entries and `sessionBudget`; nothing new.

**Rules (each a test).**
- R1 `compositionMatrixV1` of a child in a translated, turned, scaled group equals the hand product; a root node equals its local matrix; a missing node is null.
- R2 the vector of a layer whose own x/y stand still inside a group moving 5 px per frame is (5, 0).
- R3 the vector at frame 0 is zero; at frame 1 of a layer animated from (0,0) at 0 s to (60,0) at 1 s at 60 fps it is (1, 0) ± 1e-9.
- R4 the vector for frame N reached by playing 0..N (one sampler, warm) equals the vector by a seek to N (fresh sampler) equals export's (fresh sampler walking 0..N), `Object.is` on both components (stop point 6).
- R5 `withEffectClocks` leaves the grain seed and Ripple time byte for byte: the pinned projection of step one; and with a smeared moving layer it adds exactly two parameters on that item and nothing else.
- R6 `hasEffectClocks` is true while a smear vector is non-zero and false at rest (resting layer, grain speed 0, no Ripple).
- R7 the contract: zero or tiny vector → nothing; Amount scales; the Limit clamps by length keeping direction; full vs export differ only in `sampleCount`.
- R8 bounds grow on the far side of the move only.
- R9 the kernel reference (B1 T7) and the GPU proof: mean ≤ 1, max ≤ 8 against `motionSmearReferenceV1`.
- R10 cache: a moving layer's smear pass differs frame to frame (`samePass` false); a resting layer has no smear pass and keeps its token.
- R11 a shutter sample's vector equals the vector measured at the sample's own time; a crisp node's equals the frame's.

**Tests by name.** `composition-matrix.test.ts`: T1 `compositionMatrixV1 of a nested child` (R1, group at (100, 50) turned 90° scale 2, child at (10, 0): the child's anchor lands at the hand-computed point ± 1e-9); T2 `root and missing node` (R1). `motion-smear.test.ts` (animation-core): T3 `a child of a moving group smears` (R2); T4 `frame 0 is zero and frame 1 is one px` (R3); T5 `play equals seek equals export` (R4: a 2 s document, a layer with an eased position keyframe pair and a wiggle driver so the overlay is not trivially linear, N = 37, three `HistoricalAnimationSamplerV1` instances with the real registry, assert with `Object.is`); T6 `shutter samples measure at their own time` (R11, two sample times, the crisp set). `render.worker` tests (VERIFY the existing test file for `withEchoes`/`withGrainSeeds`; add there): T7 `withEffectClocks keeps the grain seed and Ripple time` (R5, pinned literals recorded before the rename); T8 `hasEffectClocks follows the smear` (R6). `render-core/effects/motion-smear.test.ts`: T9 `compiles to nothing at rest and at 1e-4 px` ; T10 `Amount and Limit` (vector (30, 40), Amount 2 → (60, 80) len 100; Limit 50 → (30, 40) exactly); T11 `bounds on the far side` (vector (12, −7) on (100, 50, 200, 100) → (88, 50, 212, 107)); T12 `cache planner` (R10, through the planner's exported comparison; VERIFY its name and that a two-frame graph pair can be built with the reference builder). Electron proof: fixture `motion-smear` (the bloom fixture's shapes, vector (−24, 10), 16 and 48 samples) in `renderer-backends.spec.ts` with the fault seen caught: flip the sample sign (a two-sided streak), the spec must fail. Command handed over: `npx playwright test tests/electron/renderer-backends.spec.ts` (VERIFY path).

**Steps.**
1. Record the pinned projection for T7 from `main`. Write T7 against `withGrainSeeds`; run: PASS. Rename; change T7's import only; run: PASS. Typecheck; Prettier; commit `refactor(worker): rename withGrainSeeds to withEffectClocks`.
2. Write T1–T6, T8–T12 and the fixture; run: FAIL.
3. Implement `composition-matrix.ts`, `motion-smear.ts`, the contract case, the definition's hidden parameters, the worker's four sites and `timeOf`, the SPECS entry; run: PASS; the three differentials green.
4. Typecheck; Prettier; commit `feat(effects): Motion smear, with the smear vectors written by the worker (B9)`.
5. Hand the user the proof command; record the measured mean and max in the report.

**Stop points.** T5 fails (stop point 6: the live overlay and the sampler disagree at the same time; report which properties differ). The hidden parameters cannot reach the instance without a schema step. `historicalOverlayAt` cannot be called from `withEffectClocks` at one of the four sites. A differential changes.

## 4. Stubs for the other tasks

Order: B1, B2, B4, B3, B5, B6, B7, B8, B9, B10 (B3 reads B4's pyramid, Ruling 24). Every proof fixture for B2 to B6 holds four kinds of pixel, Batch A's lesson: a translucent patch, an anti-aliased turned edge, a pixel outside the layer that the effect must light, and a pixel the effect must leave untouched. A proof without all four does not count.

- **B2 Directional blur.** Consumes `lineBlurSamplesV1`, `blurWorkingScaleV1`, `directionalBlurReferenceV1`, `directionalBlurBoundsV1`, `CompiledDirectionalBlurV0`; writes the sampled-blur SPECS prelude (48-sample uniform array, `uSampleCount`, outside-is-clear read) that B3 and B9 reuse. Proof: `directional-blur` fixture, mean ≤ 1, max ≤ 8; fault: drop the last sample.
- **B3 Spin and Zoom blur** (after B4). Consumes B2's sample prelude, B4's pyramid runner, `pyramidReadV1`, `sampledBlurLevelsV1`, `spinBlurReferenceV1`, `zoomBlurReferenceV1`, `effectCentrePixelsV1`, the two bounds, `layerFrame`; the Centre uniform from `transform` and `localBounds` in the setter; the shader picks the level per sample by the gap (Ruling 24). Proof fixtures with the Centre outside the layer and a sharp mark at the Centre that must stay within 1 level; fault: read level 0 everywhere (the far arc streaks) and, seen caught separately, swap sin and cos.
- **B4 Sharpen and Progressive blur.** Consumes `gaussianBlurReferenceV1`, `sharpenReferenceV1`, `#blurToWorkingTarget`, `progressivePyramidReferenceV1`, `progressiveBlendV1`, `progressiveRampV1`, `PROGRESSIVE_LEVEL_SIGMA_V1`; the pyramid on the GPU with `BLOOM_GLSL_V1`'s two filters. Proof: the far end against Layer blur (stop point 4) on the GPU too; fault: read the wrong level.
- **B5 Outline and Choke edge.** Consumes `coverageGrowReferenceV1`, `coverageShrinkReferenceV1`, `edgeCutV1`, the cut constants, the two bounds; the blur through `#blurToWorkingTarget` with `decode: false`. Proof on coverage; fault: swap the two cuts.
- **B6 Long shadow.** Consumes `longShadowStepsV1`, `longShadowReferenceV1`, `longShadowBoundsV1`; one pass per step plus the tint under the content. Fault: drop the fade. Named limit: at an Angle off the multiples of 45° the steps read between pixels, and the maximum over those reads grows a soft fringe of about one pixel on the shadow's edge; the default 45° and every multiple of it are exact.
- **B8 Bloom's runner and Tint.** Consumes `BloomRunV1`, `layerBloomPlanV1`, `layerBloomReferenceV1`, `BLOOM_GLSL_V1`; writes `native-webgl2-bloom.ts` (`runBloomChainV1`) and the `bloom-layer` fixture; proof mean ≤ 1, max ≤ 8; faults: excess after encode, and one read per texel at Size 512. Gives 3D-D the line in B7 §5. Named limit: without half-float targets the levels hold the excess in eight bits and the faint outer edge of a glow can show rings; B8's report says which path (half-float or eight-bit) the proof machine took.
- **B10 Fixtures, envelope, Looks, rules, report.** Consumes every row (the `effects-blur` scale kind cycles the ten), B1's draw-call targets, the limits named in Rulings 2, 5, 9, 10, 14 for `effect.md`; measures the GPU envelope again with `motion-blur:1000` at every window size (stop point 7).

## 5. VERIFY list

1. `compileEffectStackV0`'s input signature and every call site (the reference builder and the retained, incremental and scale-fixture builders): all of them pass `layerFrame`, or the differential tests fail on the new fixtures while the old effects stay green.
2. The diagnostic code used for a placeholder/unsupported case (`UNSUPPORTED_DEFINITION`?) and whether the table's derivation test tolerates a row without a backend SPECS entry mid-batch.
3. `effectBlurTapsV0` is importable by render-core, or copy the discrete weights.
4. `RenderTransformV0` field names (`a, b, c, d, tx, ty`) and the matrix convention of `localTransformMatrixV1` (column vectors, y down).
5. The registry file for definitions and how `lengths` is declared (Batch A's definitions).
6. 3D-D's bloom test file names; run them before and after B7 step 2.
7. The sRGB decode function's name and module; that the isolation target holds encoded sRGB premultiplied values (the blur's decode mode 0).
8. `scaleRenderEffectInstancesV0` clamps the maximum only (Bloom's Size minimum of 4 is clamped in the compile case).
9. The backend's program-cache and target-pool types for `runBloomChainV1`'s carrier parameters.
10. The nine transform property ids as the Scene clip check reads them; the parent link field on a node.
11. How `EFFECT_GRAIN_SEED_PARAMETER_V1` is declared on the grain definition so a hidden parameter reaches the instance (copy that for smear-x and smear-y).
12. `effectsOfNodeV1`, `EFFECT_DEFINITION_IDS_V1.motionSmear` (new), `serializeExactTime` names.
13. The sampler's cache key is the exact time; the Echo's and the smear's earlier time build the same key.
14. The overlay variable at each of the four `withGrainSeeds` sites; `crisp` is a `Set<NodeId>` at the shutter site.
15. The existing worker test file where `withEchoes`/`withGrainSeeds` are tested, and whether these functions are exported (if not, test `withSmearVectorsV1` in animation-core and the worker's wiring through the existing worker harness).
16. The scope-cache planner's exported comparison function and how a test builds two frames' graphs.
17. The Electron spec path and the fixtures module path.
18. Running B7 T3: the five reach numbers; five levels expected in 48..80.
19. Running B1 T13: the measured pyramid sigmas; five levels expected to reach σ ≥ 30 at level 5.
20. Whether auto-layout writes a layer's placed position into stored properties (if it does, Ruling 5's limit falls away for free; say so in the report).

## 6. Unresolved questions for the user

None open. Two were decided with the user on 2026-10-03: Progressive blur keeps Blur 0 to 256 with up to eight pyramid levels (Ruling 10), and Spin and Zoom blur read the pyramid per pixel instead of halving the picture (Ruling 24), which moves B3 after B4.
