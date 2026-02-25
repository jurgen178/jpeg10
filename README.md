# jpeg-10 — Lossless Exposure & Contrast Adjustment

Fork of the [Independent JPEG Group's](https://ijg.org/) JPEG-10 library that extends
`jpegtran` with two new tonal-adjustment options:

| Switch | Purpose |
|---|---|
| `-exposure-comp EV` | Adjust exposure by EV stops — DC-only shift, sRGB-linearised |
| `-contrast DC LOW MID HIGH` | Adjust contrast in the DCT domain using four band controls (DC / low / mid / high) |

Both operate entirely in the **DCT coefficient domain**.  No decode, no re-encode, no
generation loss.  They compose cleanly with all existing `jpegtran` options
(`-rot`, `-flip`, `-crop`, …).

---

## Usage

```
jpegtran [standard options] [-exposure-comp EV] [-contrast DC LOW MID HIGH] input.jpg output.jpg
```

**Examples**

```sh
# Brighten by 1 stop
jpegtran -copy all -exposure-comp 1 input.jpg output.jpg

# Darken by 0.5 stops
jpegtran -copy all -exposure-comp -0.5 input.jpg output.jpg

# Contrast (uniform: DC=LOW=MID=HIGH)
jpegtran -copy all -contrast -1   -1   -1   -1   input.jpg out-contrast-u-1.jpg
jpegtran -copy all -contrast -0.5 -0.5 -0.5 -0.5 input.jpg out-contrast-u-0.5.jpg
jpegtran -copy all -contrast  0.5  0.5  0.5  0.5 input.jpg out-contrast-u+0.5.jpg
jpegtran -copy all -contrast  1    1    1    1   input.jpg out-contrast-u+1.jpg

# Contrast (band-specific examples)
jpegtran -copy all -contrast 0 0 0.6 0   input.jpg out-contrast-mid+0.6.jpg
jpegtran -copy all -contrast 0 0 0 0.4   input.jpg out-contrast-high+0.4.jpg
jpegtran -copy all -contrast 0 0.4 0 0   input.jpg out-contrast-low+0.4.jpg

# Combine: rotate 90°, brighten 0.5 EV, and add uniform contrast +0.5
jpegtran -copy all -rot 90 -exposure-comp 0.5 -contrast 0.5 0.5 0.5 0.5 input.jpg output.jpg
```

Both switches accept fractional values.  Practical ranges:

| Option | Practical range | Neutral |
|---|---|---|
| `-exposure-comp EV` | −3 … +3 | 0 |
| `-contrast DC LOW MID HIGH` | −1.5 … +1.5 (each) | 0 |

---

## How It Works

### DCT Coefficient Basics

A JPEG image is stored as a grid of 8×8 (or scaled N×N) DCT blocks.
Each block has one **DC coefficient** and 63 **AC coefficients**:

- **DC[0]** — represents the (level-shifted) average sample value of the block.
  The relationship to pixel mean is:

  $$\mu = \frac{DC_\text{unquant}}{N} + \text{center}$$

  where $N$ is the DCT block size (typically 8) and
  $\text{center} = 2^{\text{precision}-1}$ (e.g. 128 for 8-bit).

- **AC[1..N²−1]** — represent spatial frequency components (texture, edges, contrast).

Both DC and AC are stored **quantized**: the actual stored integer is
$\text{round}(\text{value} / Q_k)$, where $Q_k$ is the quantization step for
coefficient $k$.  The stored range is −32768 … 32767 (`JCOEF`).

---

### `-exposure-comp EV` — Exposure Compensation

#### Concept

A photographic EV step corresponds to doubling (or halving) the amount of light.
Applied in **linear light**:

$$\text{gain} = 2^{EV}$$

Because JPEG samples are gamma-coded (sRGB), we cannot multiply pixel values
directly.  Instead we:

1. Estimate the image's representative level from the DC blocks.
2. Compute the equivalent additive pixel-domain offset by applying the gain in
   linear light at that reference level.
3. Translate the offset into a quantized DC delta.
4. Add the delta to every DC coefficient.

Only DC is modified.  **AC coefficients are untouched**, so local contrast and
texture are perfectly preserved.

#### Reference Level — Log-Average

A geometric mean (log-average) of all block mean levels is used as the
exposure reference, because EV is inherently multiplicative:

$$\bar{L} = \exp\!\left(\frac{1}{B}\sum_{i=1}^{B} \ln(L_i + 1)\right) - 1$$

where $L_i$ is the intensity mean of block $i$ (clamped to $[0, \text{MAX}]$)
and $B$ is the total number of blocks.  The $+1$ shift keeps $\ln$ defined at
zero.

#### sRGB Linearisation

The gain is applied in **linear light**:

$$u_\text{ref} = \frac{\bar{L}}{\text{MAX}}$$

$$u_\text{ref,lin} = f_\text{lin}(u_\text{ref})$$

$$u_\text{new,lin} = \min(u_\text{ref,lin} \cdot \text{gain},\; 1.0)$$

$$u_\text{new} = f_\text{sRGB}(u_\text{new,lin})$$

The sRGB transfer functions used:

$$f_\text{lin}(u) = \begin{cases}
  u / 12.92 & u \le 0.04045 \\
  \left(\dfrac{u + 0.055}{1.055}\right)^{2.4} & u > 0.04045
\end{cases}$$

$$f_\text{sRGB}(u) = \begin{cases}
  12.92\,u & u \le 0.0031308 \\
  1.055\,u^{1/2.4} - 0.055 & u > 0.0031308
\end{cases}$$

#### Pixel-Domain Offset → Quantized DC Delta

$$\Delta_\text{samples} = (u_\text{new} - u_\text{ref}) \cdot \text{MAX}$$

Clamped to available headroom/shadow room to limit clipping, then converted to
a quantized DC delta:

$$\Delta_{DC} = \text{round}\!\left(\frac{\Delta_\text{samples} \cdot N}{Q_0}\right)$$

where $N$ is the DCT block size and $Q_0$ is the DC quantization step of that
component.

This delta is added to every DC coefficient:

$$DC_\text{new}[i] = \text{clamp}(DC[i] + \Delta_{DC},\; -32768,\; 32767)$$

#### Component Policy

| Color space | Components adjusted |
|---|---|
| YCbCr, BG_YCC, YCCK | Luma only (component 0) |
| RGB, BG_RGB + subtract-green transform | Green channel only (component 1) |
| CMYK, all others | All components |

For CMYK and YCCK the delta is computed in an **inverted intensity domain**
($I = \text{MAX} - \text{sample}$) so that +EV always means "brighter" and
−EV always means "darker".

---
### `-contrast DC LOW MID HIGH` — Contrast Adjustment

#### Concept

This option provides **four separate controls** (all in stops):

- `DC` controls the DC coefficient (block mean)
- `LOW`, `MID`, `HIGH` control the AC coefficients in zigzag frequency order

All controls are interpreted as log2 gains (stops). For a value $x$, the gain is:

$$g(x) = 2^{x}$$

#### DC

DC is scaled by:

$$g_\mathrm{DC} = 2^{DC}$$

and applied as:

$$DC' = \mathrm{clamp}(\mathrm{round}(g_\mathrm{DC} \cdot DC),\; -32768,\; 32767)$$

#### AC (zigzag, low/mid/high weighting)

AC coefficients are processed in **zigzag order** (the JPEG natural order).
Let $z$ be the AC zigzag position with $z = 1 \ldots A$, where $A$ is the
number of AC coefficients in the block.

Define a normalized position:

$$t = \begin{cases}
  \dfrac{z-1}{A-1} & A > 1 \\
  0 & A = 1
\end{cases}$$

Triangular weights:

$$w_\mathrm{low}  = \max(0, 1 - 2t)$$

$$w_\mathrm{mid}  = 1 - |2t - 1|$$

$$w_\mathrm{high} = \max(0, 2t - 1)$$

The per-coefficient gain exponent is:

$$v(z) = LOW\cdot w_\mathrm{low} + MID\cdot w_\mathrm{mid} + HIGH\cdot w_\mathrm{high}$$

and the per-coefficient gain is:

$$g(z) = 2^{v(z)}$$

Applied to each AC coefficient (in zigzag order):

$$AC'[z] = \mathrm{clamp}(\mathrm{round}(g(z)\cdot AC[z]),\; -32768,\; 32767)$$

This yields a smooth transition from low → mid → high frequencies.

If you set `DC = LOW = MID = HIGH = X`, then all coefficients are scaled by
the same gain $2^X$ (uniform contrast adjustment).

#### Component Policy

Same as `-exposure-comp`: YCbCr/BG_YCC/YCCK → luma only;
RGB/BG_RGB with subtract-green → green only; otherwise all components.

---

## Interaction with Other Transforms

Both `-exposure-comp` and `-contrast` are applied as a **post-step** after
any geometric transform (`-rot`, `-flip`, `-crop`, `-wipe`, `-drop`, …).
The tonal operations work on the final output coefficient arrays, so the
order of switches on the command line does not matter.

---

## Implementation

All new code is in `transupp.c` (functions `do_exposure_comp` and
`do_contrast`) and `transupp.h` (two new fields in `jpeg_transform_info`):

```c
boolean exposure_comp;    /* if TRUE, adjust exposure via DC shift */
double  exposure_comp_ev; /* EV input; DC-only exposure shift */
boolean contrast_adj;     /* if TRUE, scale all DCT coefficients (DC+AC) */
double  contrast_dc;      /* DC control (stops): gain = 2^contrast_dc */
double  contrast_low;     /* Low-frequency AC control (stops) */
double  contrast_mid;     /* Mid-frequency AC control (stops) */
double  contrast_high;    /* High-frequency AC control (stops) */
```

The CLI parsing is in `jpegtran.c`.

---

## Building

Use the existing Visual Studio solution or nmake makefiles included in the
repository (unchanged from upstream IJG JPEG-10).

```
# Visual Studio (x64 Debug)
msbuild jpegtran.vcxproj /p:Configuration=Debug /p:Platform=x64
```

---

## License

Same as upstream IJG JPEG: see `README` and `usage.txt` in the source tree.
This fork adds no additional restrictions.

