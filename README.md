# IIR Band-Pass Filter Coefficients (MATLAB → C)

Coefficients for a 6th-order elliptic IIR band-pass filter, designed in MATLAB and exported as C arrays. The filter is split into three second-order sections (biquads), so it runs well in single-precision `float` on microcontrollers.

## Filter specification

| Parameter | Value |
|---|---|
| Type | Elliptic (Cauer) band-pass, IIR |
| Order | 6 (3 second-order sections) |
| Passband | 0.35 – 0.40 × Nyquist (0.175 – 0.200 × fs) |
| Passband ripple | 1 dB |
| Stopband attenuation | ≥ 40 dB |
| Coefficient format | Single-precision `float`, `a0` normalised to 1 |
| Stability | All poles inside the unit circle (max \|p\| ≈ 0.982) |

Frequencies are normalised, so the actual passband depends on your sample rate. For example:

| Sample rate (fs) | Passband |
|---|---|
| 1 kHz | 175 – 200 Hz |
| 10 kHz | 1.75 – 2.0 kHz |
| 48 kHz | 8.4 – 9.6 kHz |

## Coefficients

```c
#define NUM_SECTIONS 3

float b[NUM_SECTIONS][3] = {
{1.83835751E-01, -2.64252596E-02, 1.83835751E-01},
{1.83835751E-01, -2.30690662E-01, 1.83835751E-01},
{7.56777553E-02, 0.00000000E+00, -7.56777553E-02} };

float a[NUM_SECTIONS][3] = {
{1.00000000E+00, -8.88104606E-01, 9.64603833E-01},
{1.00000000E+00, -6.10718201E-01, 9.62332122E-01},
{1.00000000E+00, -7.38234946E-01, 9.23154105E-01} };
```

Each row is one section: `b[i] = {b0, b1, b2}` and `a[i] = {1, a1, a2}`. The overall gain is already folded into the `b` coefficients, so no separate gain stage is needed (peak passband gain is 0 dB).

## Usage (C)

Run the three sections in series. Direct Form II Transposed is recommended for `float`, because it keeps only two state variables per section and behaves well numerically:

```c
static float z[NUM_SECTIONS][2] = {0};   /* filter state, zero at start */

float bandpass_filter(float x)
{
    for (int i = 0; i < NUM_SECTIONS; i++) {
        float y  = b[i][0] * x + z[i][0];
        z[i][0]  = b[i][1] * x - a[i][1] * y + z[i][1];
        z[i][1]  = b[i][2] * x - a[i][2] * y;
        x = y;                            /* output feeds the next section */
    }
    return x;
}
```

Call `bandpass_filter()` once per sample, at the sample rate the passband is scaled to.

**CMSIS-DSP:** to use `arm_biquad_cascade_df1_f32` or `arm_biquad_cascade_df2T_f32`, pack each section as `{b0, b1, b2, -a1, -a2}`. CMSIS expects the `a` coefficients with their signs flipped.

## Regenerating in MATLAB

The response matches an elliptic design of this form:

```matlab
[z, p, k] = ellip(3, 1, 40, [0.35 0.40], 'bandpass');
sos = zp2sos(z, p, k);
fvtool(sos)          % check the response
```

Change the order, ripple, attenuation or band edges, then export the rows of `sos` (columns 1–3 → `b`, columns 4–6 → `a`)
