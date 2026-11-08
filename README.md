# Elliptic IIR Filters for Cochlear Implant Processing (MATLAB → C)

Coefficients for two 6th-order elliptic IIR filters, a **band-pass** and a **band-stop**, designed in MATLAB and exported as C arrays. Each filter is split into three second-order sections (biquads), so it runs well in single-precision `float` on microcontrollers.

## Background

These filters were designed for a university project on cochlear implant signal processing. Cochlear implant speech processors split incoming audio into a bank of band-pass channels, and each channel drives an electrode along the cochlea. The band-pass filter here is one such channel. The band-stop filter rejects a band in the mid-frequency speech range.

The original sample rate wasn't recorded. All frequencies below are therefore given as a fraction of the sample rate, with values in Hz for an assumed rate of 16 kHz, which is typical for speech processing.

## Filter specifications

| Parameter | Band-pass | Band-stop |
|---|---|---|
| Type | Elliptic (Cauer), IIR | Elliptic (Cauer), IIR |
| Order | 6 (3 second-order sections) | 6 (3 second-order sections) |
| Band edges (× Nyquist) | Passband 0.35 – 0.40 | Passbands below 0.15 and above 0.60 |
| Band edges (× fs) | 0.175 – 0.200 | below 0.075 and above 0.300 |
| At fs = 16 kHz | Passes 2.8 – 3.2 kHz | Passes < 1.2 kHz and > 4.8 kHz |
| Stopband (≥ 50 dB rejection) | Outside the passband and transition bands | 0.26 – 0.41 × Nyquist (≈ 2.1 – 3.3 kHz at 16 kHz) |
| Passband ripple | 1 dB | 1 dB |
| Stopband attenuation | 50 dB | 50 dB |
| Passband gain | 0 dB | 0 dB |
| Stability | All poles inside the unit circle (max \|p\| ≈ 0.982) | All poles inside the unit circle (max \|p\| ≈ 0.926) |

To use another sample rate, multiply the "× fs" values by your rate. At 10 kHz, for example, the band-pass passes 1.75 – 2.0 kHz.

## Coefficients

Each row is one section: `b[i] = {b0, b1, b2}` and `a[i] = {1, a1, a2}`. The overall gain is already folded into the `b` coefficients, so no separate gain stage is needed.

### Band-pass

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

### Band-stop

```c
#define NUM_SECTIONS 3

float b[NUM_SECTIONS][3] = {
{4.98654396E-01, -2.97345317E-01, 4.98654396E-01},
{4.98654396E-01, -6.62163081E-01, 4.98654396E-01},
{7.34323010E-01, -7.39113743E-01, 7.34323010E-01} };

float a[NUM_SECTIONS][3] = {
{1.00000000E+00, -1.65043829E+00, 8.56736127E-01},
{1.00000000E+00, 5.16582396E-01, 7.23128309E-01},
{1.00000000E+00, -3.75261241E-01, -2.54342199E-01} };
```

Both sets use the names `b`, `a` and `NUM_SECTIONS`. To use both filters in the same C file, rename them first, for example to `bp_b` / `bp_a` and `bs_b` / `bs_a`.

## Usage (C)

Run the three sections in series. Direct Form II Transposed is recommended for `float`, because it keeps only two state variables per section and behaves well numerically:

```c
static float z[NUM_SECTIONS][2] = {0};   /* filter state, zero at start */

float iir_filter(float x)
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

Call the function once per sample, at the sample rate the band edges are scaled to. Each filter needs its own state array `z`.

**CMSIS-DSP:** to use `arm_biquad_cascade_df1_f32` or `arm_biquad_cascade_df2T_f32`, pack each section as `{b0, b1, b2, -a1, -a2}`. CMSIS expects the `a` coefficients with their signs flipped.

## Regenerating in MATLAB

Both filters match these designs:

```matlab
% Band-pass: order-3 prototype, 1 dB ripple, 50 dB stopband, 0.35-0.40 x Nyquist
[z, p, k] = ellip(3, 1, 50, [0.35 0.40], 'bandpass');
sos_bp = zp2sos(z, p, k);

% Band-stop: order-3 prototype, 1 dB ripple, 50 dB stopband, edges 0.15 / 0.60 x Nyquist
[z, p, k] = ellip(3, 1, 50, [0.15 0.60], 'stop');
sos_bs = zp2sos(z, p, k);

fvtool(sos_bp, sos_bs)   % check the responses
```

Change the order, ripple, attenuation or band edges, then export the rows of `sos` (columns 1–3 
