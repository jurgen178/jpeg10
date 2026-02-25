# jpeg-10 — Lossless Exposure & Contrast Adjustment

Fork of the [Independent JPEG Group's](https://ijg.org/) JPEG-10 library that extends
`jpegtran` with two new tonal-adjustment options:

| Switch | Purpose |
|---|---|
| `-exposure-comp EV` | Adjust exposure by EV stops — DC-only shift, sRGB-linearised |
| `-contrast CV` | Adjust contrast by CV stops — scales all DCT coefficients (DC + AC) |

Both operate entirely in the **DCT coefficient domain**.  No decode, no re-encode, no
generation loss.  They compose cleanly with all existing `jpegtran` options
(`-rot`, `-flip`, `-crop`, …).

---

## Usage

```
jpegtran [standard options] [-exposure-comp EV] [-contrast CV] input.jpg output.jpg
```

**Examples**

```sh
# Brighten by 1 stop
jpegtran -copy all -exposure-comp 1 input.jpg output.jpg

# Darken by 0.5 stops
jpegtran -copy all -exposure-comp -0.5 input.jpg output.jpg

# Increase contrast by 1 stop
jpegtran -copy all -contrast 1 input.jpg output.jpg

# Combine: rotate 90°, brighten 0.5 EV, increase contrast 0.5 CV
jpegtran -copy all -rot 90 -exposure-comp 0.5 -contrast 0.5 input.jpg output.jpg
```

Both switches accept fractional values.  Practical ranges:

| Option | Practical range | Neutral |
|---|---|---|
| `-exposure-comp EV` | −3 … +3 | 0 |
| `-contrast CV` | −1.5 … +1.5 | 0 |

---

## How It Works

### DCT Coefficient Basics

A JPEG image is stored as a grid of 8×8 (or scaled N×N) DCT blocks.
Each block has one **DC coefficient** and 63 **AC coefficients**:

- **DC[0]** — represents the (level-shifted) average sample value of the block.
  The relationship to pixel mean is:

  $$\text{block\_mean} = \frac{DC_\text{unquant}}{N} + \text{center}$$

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

$$u_\text{ref,lin} = \text{sRGB\_to\_linear}(u_\text{ref})$$

$$u_\text{new,lin} = \min(u_\text{ref,lin} \cdot \text{gain},\; 1.0)$$

$$u_\text{new} = \text{linear\_to\_sRGB}(u_\text{new,lin})$$

The sRGB transfer functions used:

$$\text{sRGB\_to\_linear}(u) = \begin{cases}
  u / 12.92 & u \le 0.04045 \\
  \left(\dfrac{u + 0.055}{1.055}\right)^{2.4} & u > 0.04045
\end{cases}$$

$$\text{linear\_to\_sRGB}(u) = \begin{cases}
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

### `-contrast CV` — Contrast Adjustment

#### Concept

Contrast is scaled by a factor:

$$c = 2^{CV}$$

All DCT coefficients — DC **and** AC — are multiplied by $c$:

$$\text{coef}_\text{new}[k] = \text{round}(c \cdot \text{coef}[k])$$

clamped to −32768 … 32767.

- **Scaling DC[0]** shifts each block's mean level towards (CV > 0) or away
  from (CV < 0) zero (mid-grey in level-shifted representation) — bright
  blocks get brighter, dark blocks get darker.
- **Scaling AC[1..N²−1]** scales all spatial frequencies proportionally —
  edges and textures become sharper (CV > 0) or softer (CV < 0).

Together this gives a **natural contrast adjustment around mid-grey** without
any decode/re-encode.

#### Formula Summary

$$c = 2^{CV}, \quad CV \in \mathbb{R}$$

$$\text{coef}_\text{new}[k] = \text{clamp}\!\left(\text{round}(c \cdot \text{coef}[k]),\; -32768,\; 32767\right)$$

Applied to all $k = 0 \ldots N^2 - 1$ (for non-square scaled blocks:
$k = 0 \ldots N_h \cdot N_v - 1$).

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
double  contrast_cv;      /* Contrast in stops: c = 2^CV */
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
