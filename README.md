#### Lossless Tonal Adjustments in JPEG's DCT Domain: Exposure Compensation and Multi-Band Contrast

Most JPEG workflows treat exposure (brightness) and contrast as inherently "lossy": decode pixels, apply curves, then re-encode. That approach works, but it always introduces an additional step of quantization error.

In this fork of the IJG JPEG-10 code, I added two options to `jpegtran` that operate directly on quantized DCT coefficients:

- `-exposure-comp EV`
- `-contrast DC LOW MID HIGH`

Both are applied during transcoding, so they combine naturally with existing `jpegtran` operations such as rotation, flipping, cropping, marker copying, and progressive conversion.

<br />

---

<br />

##### Quick Usage

```sh
jpegtran [standard options] [-exposure-comp EV] [-contrast DC LOW MID HIGH] input.jpg output.jpg
```

Examples:

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

Both switches accept fractional values. Practical ranges:

| Option | &nbsp;&nbsp;&nbsp;Practical range | &nbsp;&nbsp;&nbsp;&nbsp;Neutral |
|-:-|:-:|:-:|
| `-exposure-comp EV` | -3 … +3 | 0 |
| `-contrast DC LOW MID HIGH` | -1.5 … +1.5 | 0 |

<br />

##### Integrated into [cPicture](https://bitfabrik.io/cPicture/) with live preview:

<img width="864" height="873" alt="dlg" src="https://github.com/user-attachments/assets/027dd237-3369-4dc4-b816-b80f84406d51" />

<br />

---

<br />

##### Background: DCT Coefficient Basics

A JPEG image is encoded as a grid of DCT blocks (with 8×8 Elements in size). Each block has one **DC coefficient** and 63 **AC coefficients**.  Each MCU might have more than one block depending on the color subsampling.

- **DC[0]** represents the (level-shifted) average sample value of the block. The relationship to pixel mean is:

  $$\mu = \frac{DC_\text{unquant}}{N} + \text{center}$$

  where $N$ is the DCT block size (typically 8) and $\text{center} = 2^{\text{precision}-1}$ (e.g. 128 for 8‑bit).

- **AC[1..N²−1]** represent spatial frequency components (texture, edges, contrast).

Both DC and AC are stored **quantized**: the actual stored integer is $\text{round}(\text{value} / Q_k)$, where $Q_k$ is the quantization step for coefficient $k$.

<br />

---

<br />

##### `-exposure-comp EV` — Exposure Compensation

<br />

Exposure compensation from -2EV to +2EV:
![index_exp](https://github.com/user-attachments/assets/d78c9d63-415c-4718-b653-16abc329a83d)

<br />

###### Concept

A photographic EV step corresponds to doubling (or halving) the amount of light. Applied in **linear light**:

$$\text{gain} = 2^{EV}$$

Because JPEG samples are gamma-coded (sRGB), pixel values cannot be multiplied directly. Instead:

1. Estimate a representative level from the DC blocks.
2. Compute the equivalent additive pixel-domain offset by applying the gain in linear light at that reference level.
3. Translate the offset into a quantized DC delta.
4. Add the delta to every DC coefficient.

Only DC is modified. AC coefficients are not modified, so local contrast and texture are preserved.

###### Reference Level — Log-Average

A geometric mean (log-average) of all block mean levels is used as the exposure reference:

$$\bar{L} = \exp\!\left(\frac{1}{B}\sum_{i=1}^{B} \ln(L_i + 1)\right) - 1$$

where $L_i$ is the intensity mean of block $i$ (clamped to $[0, \text{MAX}]$) and $B$ is the total number of blocks.

###### sRGB Linearisation

The gain is applied in linear light:

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

###### Pixel-Domain Offset → Quantized DC Delta

$$\Delta_\text{samples} = (u_\text{new} - u_\text{ref}) \cdot \text{MAX}$$

Clamped to available headroom/shadow room to limit clipping, then converted to a quantized DC delta:

$$\Delta_{DC} = \text{round}\!\left(\frac{\Delta_\text{samples} \cdot N}{Q_0}\right)$$

where $N$ is the DCT block size and $Q_0$ is the DC quantization step.

###### Component Policy

| Color space | Components adjusted |
|---|---|
| YCbCr, BG_YCC, YCCK | Luma only (component 0) |
| RGB/BG_RGB + subtract-green transform | Green/base only (component 1) |
| CMYK, all others | All components |

For CMYK and YCCK the delta is computed in an inverted intensity domain ($I = \text{MAX} - \text{sample}$) so that +EV brightens and −EV darkens.

<br />

---

<br />

##### `-contrast DC LOW MID HIGH` — Contrast Adjustment

<br />

Contrast from -1CV to +1CV:
![index_contrast](https://github.com/user-attachments/assets/3663545c-3870-4b0a-933b-105ebb4d2bb1)

<br />

###### Concept

This option provides four separate controls (all in stops):

- `DC` controls the DC coefficient (block mean)
- `LOW`, `MID`, `HIGH` control the AC coefficients in frequency order

All controls are interpreted as log2 gains (stops). For a value $x$, the gain is:

$$g(x) = 2^{x}$$

So +1 doubles, -1 halves.

###### DC

DC is scaled by:

$$g_\mathrm{DC} = 2^{DC}$$

and applied as:

$$DC' = \mathrm{clamp}(\mathrm{round}(g_\mathrm{DC} \cdot DC))$$

###### AC (low/mid/high weighting)

AC coefficients are processed in zigzag order (the JPEG natural order). Let $z$ be the AC position with $z = 1 \ldots A$, where $A$ is the number of AC coefficients.

Define a normalized position:

$$t = \begin{cases}
  \dfrac{z-1}{A-1} & A > 1 \\
  0 & A = 1
\end{cases}$$

Triangular weights:

  - low weight fades out from low frequencies

$$w_\mathrm{low}  = \max(0, 1 - 2t)$$

  - mid weight peaks in the middle

$$w_\mathrm{mid}  = 1 - |2t - 1|$$

  - high weight fades in toward high frequencies

$$w_\mathrm{high} = \max(0, 2t - 1)$$

Per-coefficient exponent and gain:

$$v(z) = LOW\cdot w_\mathrm{low} + MID\cdot w_\mathrm{mid} + HIGH\cdot w_\mathrm{high}$$

$$g(z) = 2^{v(z)}$$

Applied to each AC coefficient:

$$AC'[z] = \mathrm{clamp}(\mathrm{round}(g(z)\cdot AC[z]))$$

If `DC = LOW = MID = HIGH = X`, then all coefficients are scaled by the same gain $2^X$ (uniform contrast adjustment).

###### Component Policy

Same as `-exposure-comp`:

- YCbCr/BG_YCC/YCCK: luma only
- RGB subtract-green: base/green only
- otherwise: all components

<br />

---

##### Ordering and Composition

Both `-exposure-comp` and `-contrast` are applied as a post step after any geometric transform (`-rot`, `-flip`, `-crop`, …). The tonal operations work on the final output coefficient arrays, so the order of switches on the command line does not matter.

<br />

---

##### Implementation notes

- Core implementation:
  - `transupp.c`: `do_exposure_comp()` and `do_contrast()`
  - `transupp.h`: adds new fields to `jpeg_transform_info`
- CLI parsing:
  - `jpegtran.c`
- Feature flags and parameters are stored in `jpeg_transform_info` in `transupp.h`

<br />

---

##### Summary

- `-exposure-comp EV` shifts brightness by changing only DC coefficients, with EV evaluated in linear light (sRGB transfer) at a log-average reference.
- `-contrast DC LOW MID HIGH` scales DC and AC coefficients, with AC gains varying smoothly over frequency order using low/mid/high controls.
- Both run in the DCT domain and integrate naturally into the lossless-transformation workflow of `jpegtran`.

