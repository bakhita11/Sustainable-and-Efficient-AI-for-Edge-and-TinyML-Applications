# Ultra-Low-Power TinyML for Gas Classification on 8051-Class Microcontrollers

This repository contains the full implementation accompanying our work on deploying a quantized machine-learning classifier on a legacy 8-bit microcontroller. We train an 8-bit quantized decision tree on the UCI Gas Sensor Array Drift (GSAD) benchmark and deploy it on a Silicon Labs **EFM8BB1** (8051-class) MCU through a fully integer-only, non-recursive inference engine — no floating-point unit, no cloud, no external computation.

The model runs entirely on-device with a **~2 KB code footprint, ~124 bytes of RAM, 12.44 ms per inference, and ~205 µW average power**, while retaining useful multi-class gas discrimination.

> **Scope note.** All *quantitative* results are obtained on the public GSAD benchmark (recorded with Figaro TGS sensors). The physical prototype (MQ-series MOX sensor) is a **qualitative** end-to-end demonstration of the pipeline, not a validated sensing result.

---

## Key results

| Metric | Desktop (float) | On-device (INT8, EFM8BB1) |
|---|---|---|
| Classification accuracy | 94.6% | 79.0% |
| Macro F1-score | 0.943 | 0.779 |
| Avg. inference latency | — | 12.44 ms (80.4 inf/s) |
| Executable code size | — | 2,003 bytes |
| Constant Flash (model + test vectors) | — | 5,923 bytes |
| RAM footprint (internal + XRAM) | — | ~124 bytes |
| Avg. power / energy per inference | — | ~205 µW / ~201 µJ |
| Arithmetic | floating-point | integer-only, 8-bit |

8-bit quantization is **nearly lossless** versus full precision on the headline split (94.6% vs. 94.1%). Under the time-separated *drift* protocol (train batches 1–5, test 6–10), aggregate accuracy drops to 38.7%, and the per-gas breakdown shows **toluene, acetaldehyde, and ethanol drift earliest**, while ethylene and ammonia remain comparatively recoverable.

---

## Repository structure

```
.
├── data/                 # GSAD preprocessing outputs, selected feature indices,
│                         # per-feature quantization bounds (Qmin/Qmax)
├── training/             # Offline model training & evaluation (Python)
│   ├── preprocess.py     # merge GSAD batches, impute, feature selection
│   ├── train.py          # depth-limited decision tree, 8-bit quantization
│   └── export_c.py       # emit Flash-resident tables (thresholds, children, leaves)
├── firmware/             # EFM8BB1 integer-only inference engine (C / Keil C51)
│   ├── inference.c       # non-recursive tree traversal
│   ├── quantize.c        # 8-bit feature quantization
│   ├── aqi.c             # confidence-based GOOD/POOR AQI mapping
│   └── benchmark.c       # on-chip N=100 benchmark (accuracy, macro-F1, Timer0 latency)
├── figures/              # figure-generation scripts (architecture, pipeline, plots)
└── README.md
```

> Adjust the paths above to match your actual layout.

---

## Method overview

1. **Offline (desktop, Python).** Merge the ten GSAD batches, mean-impute missing values, and select `N_keep = 8` channels by a deterministic variance + correlation rule. Train a depth-10 decision tree (`min_samples_leaf = 10`, seed 42), then quantize each selected feature to an unsigned 8-bit integer using fixed per-feature `Qmin/Qmax` bounds.
2. **Export.** Emit the tree (thresholds, child pointers, leaf class/confidence) and quantization bounds as constant C lookup tables, stored in program Flash.
3. **On-device (EFM8BB1, C).** An iterative, non-recursive traversal runs over the 8-bit features using only integer comparisons. A confidence-aware stage maps the six-class prediction to a binary **GOOD/POOR** Air-Quality decision (configurable threshold γ and hazardous set H).
4. **Benchmark.** At startup the firmware runs all 100 held-out test vectors through the deployed engine and reports accuracy, macro-F1, and per-sample Timer0 latency over UART.

---

## Reproducing the results

### 1. Train and export (desktop)

```bash
# Python 3.10+ recommended
pip install -r requirements.txt

python training/preprocess.py     # writes selected features + quant bounds to data/
python training/train.py          # trains the depth-10 tree, prints desktop metrics
python training/export_c.py       # generates the C tables for firmware/
```

### 2. Build and flash the firmware

Open the `firmware/` project in **Keil µVision (C51)**, build, and flash to the EFM8BB1.
On reset, the device runs the on-chip benchmark and streams per-sample results over UART
(115200 baud, 8N1). It then enters the continuous real-time loop driven by the
`READ_READY` flag.

> Full code-size optimization in the Keil C51 compiler requires a paid license.

### 3. Read the on-device benchmark

Connect a serial terminal to the UART. The firmware prints, for each sample, the
ground-truth label, predicted class, leaf confidence, and measured latency, followed
by an aggregate summary (accuracy, macro-F1, throughput, memory footprint).

---

## Dataset

This work uses the public **UCI Gas Sensor Array Drift (GSAD)** dataset (six gas classes;
128 sensor-derived features; ten time-separated batches over ~three years). It is **not**
redistributed here — download it from the UCI Machine Learning Repository and place it where
`training/preprocess.py` expects it (see that script's header). The repository provides the
preprocessing scripts, the selected feature indices, and the per-feature quantization bounds
needed to reproduce the reported numbers.

---

## Hardware

- **MCU:** Silicon Labs EFM8BB1 (8051-class, 8-bit, integer-only)
- **Prototype sensor front-end:** MQ-series metal-oxide (MOX) gas sensor via a voltage
  divider scaling the 5 V output to the 2.4 V ADC range
- **I/O:** register-level UART for logging; GPIO-driven status LED (red = POOR, green = GOOD)

The prototype warm-up and humidity/temperature sensitivity of MOX sensors are **not**
compensated in this design; see the paper's Limitations section.

---

## Limitations

- All quantitative accuracy figures are on GSAD (Figaro TGS sensors); the MQ-series prototype
  is a qualitative demonstration only and is not validated against labelled ground truth.
- The headline split mixes acquisition batches and therefore does **not** reflect cross-batch
  drift, which is characterized separately under the drift protocol.
- The deployed model performs **no** drift correction.

---

## Citation

If you use this code or build on this work, please cite the paper:

```bibtex
@article{salman_tinyml_8051,
  title   = {Ultra-Low-Power TinyML for Gas Classification on 8051-Class Microcontrollers},
  author  = {Salman, Bakhita and Yassin, Muneeb and Solis, Juan},
  journal = {TODO: venue},
  year    = {2026},
  note    = {Code: https://github.com/bakhita11/Sustainable-and-Efficient-AI-for-Edge-and-TinyML-Applications}
}

## License

TODO — add a license file (e.g. MIT) so others can reuse the code. Without one, default
copyright applies and reuse is restricted.
