# DeepTrack2 — concepts behind the CW20 workshop

Reference notes for `CW20_Guigo_SPAOM2026.ipynb`. Everything here was checked against the
installed source (DeepTrack2 2.0.2 dev) or measured in that environment; numbers quoted are
measured values.

---

# Part 1 — How DeepTrack2 works

## 1.1 The one idea

> **A Feature is a recipe for data, not the data itself.**

`optics(particle)` is a `Microscope` object. `optics(particle).resolve()` is an `ndarray`.
Nothing is computed until you resolve.

Technically: `Feature` subclasses `DeepTrackNode` — **a feature *is* a node in a computation
graph**.

## 1.2 Three phases

| Phase | Operations | What happens |
|---|---|---|
| **Build** | `^`, `>>`, `&`, `+`, `optics(...)` | wires nodes; computes nothing; returns a Feature |
| **Resolve** | `()`, `.resolve()`, `.new()`, `.plot()` | walks the graph, evaluates, caches, returns a value |
| **Invalidate** | `.update()` | clears caches so the next resolve re-samples; computes nothing |

If you can say which phase a line belongs to, you can read any pipeline.

- `resolve` **is** `__call__` — literally `resolve = __call__` in the source. Identical.
- `new()` is exactly `update()` then call: `return self.update()(data_list, ...)`.
- `.plot()` **resolves**. It is not a passive display.

## 1.3 Caching

Each node stores its result. Calling twice returns the **identical object** (`f() is f()` is
`True`), not an equal copy.

- This is why re-running a cell gives the same image.
- This is why `image & TakeProperties(image, "position")` is consistent: the shared node is
  evaluated once and both branches read the same cached value.
- Two different optics resolving the same scatterers **without** `update()` see the *same*
  objects — which is what makes the §1.9 modality comparison valid.

## 1.4 `.update()` in detail

```python
for dependency in self.recurse_dependencies():        # walk UP
    for dependency_child in dependency.recurse_children():   # then DOWN from each
        dependency_child.data = DeepTrackDataDict()   # wipe
return self
```

Measured behaviour:

| Case | Refreshed? |
|---|---|
| Downstream (`a → b`, update `a`) | yes |
| **Upstream** (`a → b`, update `b`) | **yes** |
| **Siblings sharing an anchor** | **yes** |
| Disconnected graphs | no |

Consequences:

- **It computes nothing.** 0 `get()` calls after `update()`, 1 after the following call.
- **It clears every `_ID`.** With `particle ^ 5`, all five replicates re-draw.
- **Idempotent**, and returns `self` (so `.update()()` chains).
- **You cannot refresh one part of a connected pipeline while pinning another.** Calling
  `update()` on the optics also resamples the `Repeat` upstream. That is deliberate — otherwise
  you could refresh an image and keep stale labels.
- On a **static** feature, `update()` still re-runs `get()` and returns a *new array with
  identical values* (`is` is False, `allclose` is True). Wasted work, no change.

## 1.5 Properties

Properties are **rules, not values**:

```python
intensity=40                                 # fixed
intensity=lambda: np.random.uniform(20, 60)  # re-sampled per image
intensity=lambda base: base * 2              # reads another property BY NAME
```

**Arguments are injected by name.** DeepTrack inspects the function's parameter names
(`get_kwarg_names`) and passes only those. A rule written `def rule(**kwargs)` receives
**nothing**.

Where randomness is sampled depends on *where the lambda lives*:

| Lambda on… | Sampled |
|---|---|
| a scatterer property, with `^ N` | once **per particle** |
| a feature **after** the optics | once **per image** |

So `dt.Add(lambda: np.random.uniform(10, 15))` after the optics adds **one** value to the whole
image — verified: min = max = a single unique value across all pixels.

## 1.6 Operators

| | Class | Meaning | Returns |
|---|---|---|---|
| `a >> b` | `Chain` | **then** — output of `a` feeds `b` | Feature |
| `a & b` | `Stack` | **and also** — both evaluated, results concatenated | Feature → list |
| `a ^ n` | `Repeat` | n independent copies, each re-sampling | Feature → list |
| `a + b`, `a * b` | arithmetic | element-wise | Feature |

**Everything returns a Feature.** That closure is why pipelines nest without special cases.

`&` **splats**: `Stack.get` returns `[*inputs, *value]`, so the result is **flat**:

```python
image, *positions = (image_feature & TakeProperties(...))()   # [image, pos, pos, ...]
```

Hence the `*` (Python extended unpacking) — the number of positions varies, so plain
`image, positions = ...` raises `ValueError: too many values to unpack`.

`&` does not guarantee consistency by copying; it guarantees it because the branches **share
graph nodes**, and shared nodes resolve once.

## 1.7 Units

Each feature declares a `__conversion_table__`, e.g. for `Scatterer`:

```python
position=(u.pixel, u.pixel)     # position defaults to pixels
z=(u.zpixel, u.zpixel)
voxel_size=(u.meter, u.meter)   # sizes default to metres
```

- **Units are optional.** `radius=1e-6` and `radius=1*dt.units.um` give byte-identical images.
- **The payoff is stepping outside the default.** `position=(3,3)*dt.units.um` lands on pixel
  (28, 28) at 108 nm/px and pixel (9, 9) at 325 nm/px — the conversion is automatic.
- **One unit per property, not per element.** `radius=(0.8,0.4,0.4)*dt.units.um` works;
  mixing `(800*nm, 0.4*um, 400e-9)` raises `ValueError`.
- Angles work: `rotation=45*dt.units.deg` equals `rotation=0.785`.

## 1.8 `__distributed__`

Default `True`: `get()` is applied to **each item** of the input list.

A feature that **creates** data (a texture, a gradient) has an *empty* input, so `get()` runs
**zero times** and you get an array of shape `(0,)` with no error.

**Any generator feature you write needs `__distributed__ = False`.**

## 1.9 `_process_properties` — and why `Bacillus` overrides it

A **hook that runs on the sampled properties just before `get()`**:

```python
properties_copy = self.properties(_ID=_ID).copy()           # 1. sample
properties_copy = self._process_properties(properties_copy) # 2. ← the hook
results_list = self._process_and_get(inputs_list, **properties_copy)  # 3. get()
```

The base implementation is `return self._normalize(**property_dict)`, and `_normalize` walks the
MRO applying every `__conversion_table__` — **this is where units are converted**. So an override
must call `super()._process_properties(...)` first, or units never convert.

**Why `Bacillus` overrides it:** to normalise *shapes and types* that unit conversion does not
touch. Traced with deliberately awkward input:

```
user passes:       radius=(0.5, 0.3)*um,  length=2*um,  rotation=45*deg
after _normalize:  radius=array([5e-07, 3e-07]),  length=2e-06,  rotation=0.785
after the override: radius=5e-07,  length=2e-06,  rotation=0.785
```

`_normalize` did the units but left `radius` as a **two-element array** because the user passed a
tuple. The three lines in `Bacillus` force scalars:

```python
properties["radius"]   = float(np.ravel(properties["radius"])[0])
properties["length"]   = float(properties["length"])
properties["rotation"] = float(np.ravel(properties["rotation"])[0])
```

Without them, `get()` computes `extent` from an array and builds a broken grid.

**The design point:** `_process_properties` means *"make the inputs sane"*; `get()` means *"do the
geometry, assuming everything is already a scalar in metres and radians."* `Cylinder` in DTGS171
overrides it in the opposite direction — it *expands* a scalar radius to length 2, because a
cylinder has two.

## 1.10 Dynamics: `to_sequential` + `Sequence`

Three pieces, all required:

| Piece | Role |
|---|---|
| `position=lambda: ...` | the **initial** value |
| `.to_sequential(position=rule)` | how it **evolves** |
| `dt.Sequence(..., sequence_length=N)` | **drives** the clock, collects frames |

- The original property becomes the **initial value**; the new rule produces steps 1…N-1.
- Rule arguments by name: `previous_value`, `sequence_index`, `sequence_length`,
  `previous_values`, plus any other property of the same feature.
- Without `Sequence`, a sequential property **never advances** — no error, just a static result.
- `Sequence` returns a **list of frames**. If the wrapped feature returns several outputs, the
  result is **transposed** into a tuple of lists.
- `sequence_length` can itself be random.
- **Non-sequential properties are sampled once for the whole movie**, not per frame.
- `.update()` regenerates the **entire trajectory**; there is no "advance one frame".

## 1.11 Feature reference

| Feature | What it does |
|---|---|
| `dt.Value(x)` | holds a value and ignores any input; lifts plain data (scalars, arrays) into the graph. Used in §4.1 as `make_bg` / `make_fil` to inject generated textures, and to carry a label alongside an image |
| `dt.Add(b)` / `dt.Background(offset)` | **element-wise** addition; identical to each other (verified). `b` can be scalar, array, lambda or Feature. `image + 0.5`, `0.5 + image`, `dt.Add(0.5)(image)` are all the same |
| `dt.Poisson(snr, background)` | redraws each pixel from `Poisson(mean = pixel)`. Signal-dependent |
| `dt.Gaussian(mu, sigma)` | `mu + image + noise*sigma`. Signal-independent |
| `dt.ElasticTransformation(alpha, sigma, order)` | random displacement field (noise smoothed by a Gaussian) that locally warps the object. `alpha` = strength, `sigma` = smoothness |
| `dt.Pad(px=..., keep_size=False)` | adds a margin of zeros, giving the deformation room to push into. `px` is a flat sequence `(before_axis0, after_axis0, before_axis1, ...)`, so **four values pad x and y only** and six pad z as well. A **single integer pads all three axes**, z included |
| `dt.CropTight(eps)` | trims the empty border, restoring a tight bounding box. `eps` (default `1e-10`) is the threshold below which a value counts as empty, so interpolation leftovers do not keep the box large |
| `dt.SampleToMasks(fn, output_region, merge_method)` | builds a mask from the **same scatterers** that produced the image. `merge_method`: `"or"`, `"add"`, `"overwrite"`, `"mul"`. It does **not** go through the optics — see below |
| `dt.TakeProperties(feature, "name")` | reads a property across every instance. For sequences returns `(N_particles, N_frames, 2)` |
| `dt.NonOverlapping(feature, min_distance)` | resamples positions until the **3D** volumes do not overlap — separation along z counts, see below |
| `dt.NormalizeQuantile(quantiles)` | rescales by intensity percentiles |
| `dt.pytorch.ToTensor` / `dt.pytorch.Dataset` | converts to tensors; exposes a pipeline as a PyTorch dataset (`replace` controls cache refresh) |
| `dt.SphericalAberration(coefficient)` etc. | Zernike pupil aberrations, passed as `pupil=` to the optics |

**The `Pad → deform → CropTight` idiom.** A scatterer volume is a *tight* bounding box, so
warping it directly clips material at the edges. Measured on one nucleus:

| Stage | Volume shape |
|---|---|
| raw ellipsoid | (39, 43, 15) |
| `+ Pad(10)` | (59, 63, 15) |
| `+ ElasticTransformation` | (59, 63, 15) — size unchanged, contents warped |
| `+ CropTight` | (43, 46, 15) |

The final crop matters: DeepTrack places volumes **by their bounding box**, so a loose box wastes
memory and would confuse `NonOverlapping`.

**What `SampleToMasks` actually returns.** The **geometric footprint of the scatterer volumes**,
placed on the image grid — *not* the object as the microscope would see it. It bypasses the optics
entirely: no PSF, no blur, no defocus. Measured on one ellipsoid, the mask is **identical** at
NA 0.4 and NA 1.4 (323 px both times) while the imaged area changes (533 px vs 423 px above 10% of
peak), and it is **unchanged by defocus** (323 px at both `z = 0` and `z = 15`, while the image
peak falls from 5.4 to 2.0). That is what you want from a label: it describes the object, not the
instrument. It also explains why a mask always looks tighter than the blurred image beside it.

**What `NonOverlapping` actually enforces.** Separation of the **3D bounding cubes**
`[x1, y1, z1, x2, y2, z2]`, and the source requires a gap along **at least one axis**. Since z is
one of them, two objects at different depths count as non-overlapping even when their footprints
land on the same pixels — so they can still appear superimposed in the final image. It prevents
objects from occupying the same *space*, not from projecting onto the same place.

Note these act on the **scatterer geometry**, before the optics — you are deforming the object,
not the photograph.

---

# Part 2 — The physics

## 2.1 Scatterers: two families

Every object is a **scatterer** — it defines **where** it is (`position`, `z`) and **what shape**
it has. The optics decide what physics applies.

| | `VolumeScatterer` | `FieldScatterer` |
|---|---|---|
| Hands over | a **3D voxel volume** (pure geometry) | the **complex scattered field** |
| Who does the diffraction | the **optics**, slice by slice | already done, by **Mie theory** |
| Members | `PointParticle`, `Sphere`, `Ellipsoid`, custom | `MieSphere`, `MieStratifiedSphere` |
| Accuracy | approximate (weak-phase, forward only) | **exact**, but spheres only |
| Fluorescence | yes | **no** — `TypeError: Fluorescence microscope cannot operate on ScatteredField` |

A `PointParticle` is literally a **1×1×1 voxel containing 1.0**. Everything you see in its image
is the microscope, not the object.

**A scatterer resolved alone gives a `(1,1,1)` volume** — the voxel grid comes from the active
optics, and there is none. Resolve through the optics first.

## 2.2 How the field is solved — three levels

All three solve the same Helmholtz equation `∇²E + k₀²n²(r)E = 0`; the geometry forces the method.

**Level 1 — Fluorescence: no scattering solved at all.**
Emitters are incoherent, so **intensities add**: `I = (emitter density) ⊛ |PSF|²`. The only
electromagnetics is the PSF itself, from the pupil:

```
pupil = (ρ < 1) · exp(i · defocus · k_z),   k_z = (2π n/λ)·√(1 − (NA/n)²ρ²)
PSF   = |FT(pupil)|²
```

**Level 2 — coherent + VolumeScatterer: split-step beam propagation (BPM).**

```python
light_in  = fft2(illumination)
for each z-slice:
    light_in  = light_in * pupil_step             # diffraction over dz, Fourier space
    light     = ifft2(light_in)
    light_out = light * exp(1j * Δn * dz * K)     # refractive phase, real space
    light_in  = fft2(light_out)
```

**Operator splitting**: diffraction is diagonal in Fourier space, the index term is diagonal in
real space; alternating them over small `dz` approximates the full solution.

- Captures: diffraction, refraction, focusing, absorption (complex `n`), **forward** multiple
  scattering.
- Misses: backscattering, polarization (it is scalar), wide-angle accuracy, strong contrast.
- Hence the warning *"assumes a weak-phase / projection approximation"*.

**Level 3 — Mie theory: exact, for spheres.**
Expand incident, internal and scattered fields in vector spherical harmonics; impose continuity of
tangential **E** and **H** at `r = a`. Orthogonality **decouples the problem order by order**, so
an infinite boundary-value problem becomes independent 2×2 solves. DeepTrack computes the standard
Bohren–Huffman coefficients with Riccati–Bessel functions:

```python
A = (m*Smx*dSx - Sx*dSmx) / (m*Smx*dxix - xix*dSmx)   # aₙ, electric multipoles
B = (Smx*dSx - m*Sx*dSmx) / (Smx*dxix - m*xix*dSmx)   # bₙ, magnetic multipoles
```

Two inputs fully determine it:
- **size parameter** `x = 2π·n_medium·a / λ`
- **relative index** `m = n_particle / n_medium` (complex if absorbing)

Then `S₁(θ)`, `S₂(θ)` (perpendicular and parallel polarization) follow, which is why the code
carries `S1_coef` / `S2_coef` and ISCAT exposes `input_polarization`. Series truncated near
`n_max ≈ x + 4x^(1/3) + 2`.

**Mie is not "the regime where particles ≈ λ".** It is exact for *any* size. Rayleigh is its
small-particle limit (`x ≪ 1`, only `a₁` survives, `I ∝ a⁶/λ⁴`); geometric optics is the large
limit.

**Why a FieldScatterer bypasses propagation:** Mie already gives the field everywhere, including
the pupil plane — nothing is left for the optics to propagate.

## 2.3 The five modalities

| Modality | Illumination | Reads | Contrast |
|---|---|---|---|
| `Fluorescence` | incoherent | `intensity` | the object **emits** |
| `Brightfield` | coherent | `refractive_index` | transmitted light delayed by the object |
| `Holography` | coherent | `refractive_index` | **alias of Brightfield** (`class Holography(Brightfield): pass`) |
| `ISCAT` | coherent | `refractive_index` | scattered field interfering with a reference beam |
| `Darkfield` | coherent | `refractive_index` | **only** the scattered light, `|E − 1|²` |

**`intensity` is used by `Fluorescence` only.** The coherent modalities ignore it and warn;
fluorescence ignores `refractive_index` and warns. Physically correct: fluorophores emit, so
brightness belongs to the object; in coherent imaging nothing is emitted.

`intensity` means **emitted light per unit volume** — the scatterer array is a geometry mask and
fluorescence multiplies it by `intensity × voxel volume`. So total brightness scales with
`intensity × volume`: a big nucleus at `intensity=1` is far brighter than a point particle at
`intensity=1`.

If `intensity` is absent, fluorescence falls back to the generic `value` property and warns.

## 2.4 Which modalities need Mie

**None require it** — all accept volume scatterers. But quality degrades very differently. Same
0.3 µm sphere:

| Modality | with `MieSphere` | with `Sphere` | verdict |
|---|---|---|---|
| Brightfield / Holography | [0.91, 1.94] | [0.77, 1.11] | both plausible, 1 warning |
| **ISCAT** | [0.9955, 1.0053] | **[0.77, 1.11] — identical to Brightfield** | silently **not ISCAT** |
| **Darkfield** | [0.0000, 0.0005] | **[0.177, 0.286] — bright background** | qualitatively wrong, 2 warnings |

- **Darkfield** images only the scattered light at **wide angles** — exactly where the forward BPM
  is worst — and falls back to a `(Δn)²` heuristic. With Mie the background is 0.0000, as
  darkfield demands; with a volume scatterer it is 0.177.
- **ISCAT** depends on backscattering and a reference beam. Its defining parameters
  (`illumination_angle=π`, `amp_factor`) are read **only** by the Mie code, and BPM propagates
  strictly forward. With a plain `Sphere` it returns byte-identical output to Brightfield.
- **Holography/Brightfield** is dominated by *forward* scattering interfering with the unscattered
  beam — precisely what BPM computes well. Most forgiving; use volume scatterers for cells.

## 2.5 Coherence

**Decisive test:** if light is coherent, fields add then square, so `I(a+b) ≠ I(a) + I(b)`.

| Modality | `|I(a+b) − I(a) − I(b)|` |
|---|---|
| Fluorescence | **5.6 × 10⁻¹⁷** — machine zero |
| ISCAT | 4.9 × 10⁻⁵ |
| Brightfield / Holography | 3.0 × 10⁻² |
| Darkfield | 1.0 |

Fluorescence is exactly linear — emitters never interfere. All scattering modalities interfere.

ISCAT's small number is **not** incoherence: the reference beam dominates, so the signal is nearly
**linear in the scattered field** (which is why ISCAT signal scales with mass, not mass²). Compare
4.9 × 10⁻⁵ with fluorescence's 5.6 × 10⁻¹⁷ — twelve orders of magnitude apart.

**No partial coherence.** Scattering is always fully coherent, fluorescence always fully
incoherent; there is no middle ground, no Köhler illumination, no finite bandwidth.
`dt.Incoherent` averages intensities over **polarization** states — not over phases.

## 2.6 PSF, resolution, sampling

A **point particle** is diffraction-limited: far smaller than the resolution, so it probes the
optics. Its image is the **PSF** — an Airy pattern, bright core plus faint rings.

Two width conventions, which differ:

| | Formula | At λ=680 nm, NA=1.4 |
|---|---|---|
| FWHM | ≈ 0.51 λ/NA | **248 nm** ≈ 2.3 px |
| Airy radius = Rayleigh criterion | 0.61 λ/NA | **296 nm** ≈ 2.7 px |

**FWHM** = full width at half maximum: the width of the profile at half the peak. It exists
because a PSF has no edge, so "the width" needs a convention. Half-max is robust and easy to fit
(Gaussian: FWHM = 2.355 σ). Measuring it by *counting pixels* is crude — counting gave 325 nm
against 248 nm theory, purely a sampling artefact.

**NA and angle.** `NA = n·sin θ` (θ = half-angle), so **NA ≤ n**. NA > 1 means *immersion*, not an
angle beyond 90°.

| medium | n | NA 0.95 | NA 1.2 | NA 1.4 |
|---|---|---|---|---|
| air | 1.000 | 71.8° | — | — |
| water | 1.333 | 45.5° | 64.2° | — |
| oil | 1.518 | 38.7° | 52.2° | **67.3°** |

Counterintuitively a NA 1.4 oil lens collects a *narrower* cone (67.3°) than a NA 0.95 air lens
(71.8°); it wins because the denser medium shortens the wavelength.

**`resolution` in DeepTrack is the CAMERA PIXEL SIZE**, not resolving power. Sample pixel =
`resolution / magnification`. The two are independent and routinely differ by 10×.

DeepTrack warns when `NA/λ × sample_pixel > 0.5`. Rule of thumb: **2–3 pixels across the FWHM**.

**Refractive index of the medium** only affects **defocus** in fluorescence. In focus, the image is
identical for any `n`. If `NA > n_medium`, part of the pupil is evanescent and DeepTrack zeroes
that defocus term — giving unphysically sharp defocused spots.

## 2.7 Noise

**Gaussian is additive; Poisson is not** — because they model different physics.

```python
# Gaussian: the image is a term in a sum
return mu + image + noise * sigma

# Poisson: the image IS the distribution's mean
noisy = np.random.poisson(image * rescale) / rescale
```

Measured on one image with four brightness patches:

| patch signal | Gaussian std | Poisson std | Poisson ÷ √signal |
|---|---|---|---|
| 0.05 | 0.0501 | 0.0112 | 0.0501 |
| 0.20 | 0.0499 | 0.0226 | 0.0504 |
| 0.50 | 0.0499 | 0.0353 | 0.0499 |
| 1.00 | 0.0495 | 0.0499 | 0.0499 |

Gaussian is **flat**; Poisson grows as **√signal** (last column constant).

- **Gaussian = read noise.** Amplifier and ADC electronics, which behave identically whether 0 or
  10⁶ photons arrived. Signal-independent → additive. Takes an **absolute** `sigma`.
- **Poisson = shot noise.** It *is* the measurement, not something added to it. Photon arrival is
  random; for a Poisson distribution variance = mean, so σ = √N. Takes a **relative** `snr`,
  because the noise is set by the signal.

**`snr` depends on image scale.** `rescale = (snr/peak)²`, so `snr` only means a true SNR when the
image peak is ≈ 1. Scale intensities accordingly.

**Background is not cosmetic**, and **order matters**:

| | empty-corner mean | empty-corner std |
|---|---|---|
| no background | 0.0000 | **0.00000** |
| background **before** Poisson | 0.0985 | **0.02172** |
| background **after** Poisson | 0.1000 | **0.00000** |

Background photons carry their own shot noise, so `Add` must come **before** `Poisson`. Put it
after and you get a flat, silent grey floor that no real camera produces. Quadrupling the
background doubles the noise (√N, measured).

The two `0.1`s do different jobs: `dt.Add(0.1)` actually puts the photons there;
`dt.Poisson(background=0.1)` only tells Poisson where the floor is so `snr` refers to signal above
it. They must agree.

---

# Part 3 — Common mistakes

Each of these cost real debugging time while building the notebook.

| Trap | Symptom | Fix |
|---|---|---|
| `dt.Poisson(SNR=...)` | **silently accepted, does nothing**; noise stays at the default `snr=100` | lowercase `snr=` |
| `position_units=` | silently accepted as a junk property | `position_unit=` (singular) |
| `dt.TakeProperties(particle, ...)` on the *single* scatterer | `AssertionError: () is not a valid index` | pass the **repeated** feature (`particles`) |
| Resolving a `Repeat` **standalone** before the optics | same `AssertionError`, `keylength=1` vs `()` | resolve **through the optics first**, then read volumes |
| Custom generator feature without `__distributed__ = False` | returns shape `(0,)`, no error | set it |
| Scatterer resolved with no active optics | `(1,1,1)` volume | resolve through optics first |
| `dt.NonOverlapping` in a training pipeline | ~10 s/image at 256 px — 512 images ≈ 85 min | drop it, or cut `length` / `IMAGE_SIZE` |
| ISCAT + plain `Sphere` | byte-identical to Brightfield, no specific warning | use `MieSphere` |
| `.plot()` used to "just look" | it **resolves**, consuming RNG and caching | use it knowingly |
| `update()` to get "new noise, same particles" | resamples upstream too — the particles change | resolve once into an array, then transform |
| Seeding once, then `update()` repeatedly | still different each time (the RNG advances) | re-seed before each resolve, or re-run the whole cell |
| Rebuilding scatterers to make masks | misaligned masks (measured IoU 0.73) | run `SampleToMasks` on the **same** scatterers |

**Per-instance masks.** `SampleToMasks` merges everything. To label each scatterer separately, use
a counter inside the transformation function and `merge_method="overwrite"`:

```python
def label_each_scatterer():
    counter = itertools.count(1)
    def inner(volume):
        return np.any(volume > 0, axis=-1, keepdims=True) * next(counter)
    return inner
```

DeepTrack calls `inner` once per scatterer and does the placement itself, so the result is aligned
by construction. (A per-scatterer `value` property does **not** work — the volume array stays 0/1.)

---

# Part 4 — The argument for simulation

- Supervised deep learning needs labels; in microscopy labels are expensive.
- **More importantly, you cannot annotate what you cannot see**: sub-pixel position, depth,
  refractive index, dry mass, diffusion coefficient, which punctum pairs with which. For these,
  annotation is not expensive — it is *impossible*.
- Microscopy is one of the few fields where the **forward model is known**, so the data-generating
  process can be simulated.
- **The label is the input, not the output.** You do not measure the ground truth; you chose it.

**When it works:** the imaging physics is understood, objects are describable by a few parameters,
annotation is slow/unreliable/impossible.

**When it struggles:** appearance too complex or variable to model. Even then, simulation is often
good for pre-training before fine-tuning on a few real annotations.

**The rule of thumb:** the simulation does not need to be *beautiful*, it needs to **cover** the
variability of the real images. When in doubt, randomise a parameter over a wider range than you
expect to encounter.
