# CEEMDAN Long-Time-Series Extension

**Extended Complete Ensemble Empirical Mode Decomposition with Adaptive NoiseAN) for long time (CEEMD-series analysis.**

[![License: GPL v2+](https://img.shields.io/badge/License-GPL%20v2+-blue.svg)](https://www.gnu.org/licenses/old-licenses/gpl-2.0.html)
[![Python 3.9+](https://img.shields.io/badge/python-3.9+-blue.svg)](https://www.python.org/downloads/)

## 📋 Overview

This project provides an **extended version of CEEMDAN** that resolves a limitation in the original `emd-signal` library, enabling decomposition of very long time-series (27+ years of hourly data, 236,000+ data points) into more than 8 Intrinsic Mode Functions (IMFs).

## 🔧 Problem & Solution

### Problem
The original `emd-signal` library (v0.8.1) contains a hidden limitation that restricts CEEMDAN decomposition to **maximum 8 IMFs** for practical purposes, caused by internal white noise IMF extraction stopping early due to default stopping criteria.

**Error encountered:**
```
IndexError: index 8 is out of bounds for axis 1 with size 8
```

### Solution
Modified the `complete_ensemble_sift` function in `emd/sift.py` to pass the `max_imfs` parameter to internal `sift()` calls, enabling extraction of the requested number of IMFs.

**Modified code (sift.py, line 883-887):**
```python
# Before:
modes_white_noise = [sift(white_noise[ii, :],
                          imf_opts=imf_opts,
                          envelope_opts=envelope_opts,
                          extrema_opts=extrema_opts) for ii in range(nensembles)]

# After:
modes_white_noise = [sift(white_noise[ii, :],
                          imf_opts=imf_opts,
                          envelope_opts=envelope_opts,
                          extrema_opts=extrema_opts,
                          max_imfs=max_imfs) for ii in range(nensembles)]
```

## 📊 Results on ASU GHI Data (1998-2024)

Successfully decomposed **236,520 hourly data points (27 years)** into **13 IMFs**.

| IMF | Energy % | Dominant Period | Physical Meaning |
|-----|----------|-----------------|------------------|
| 1 | 0.15% | 6.0 hours | High-frequency |
| 2 | 13.85% | 12.0 hours | Semi-diurnal |
| 3 | **34.84%** | **24.0 hours** | **Daily cycle** |
| 4 | 0.22% | 2.7 days | Sub-daily |
| 5 | 0.07% | 7.0 days | Weekly |
| 6 | 0.03% | 2.2 weeks | Bi-weekly |
| 7 | 0.02% | 1.1 months | Monthly |
| 8 | **3.52%** | **1.0 years** | **Annual cycle** |
| 9 | 0.50% | 1.6 years | Inter-annual |
| 10 | 0.03% | 3.4 years | Multi-year |
| 11 | 0.01% | 5.4 years | Multi-year |
| 12 | 0.00% | 13.5 years | Long-term |
| 13 | **46.77%** | **27.0 years** | **Ultra-long baseline** |

### Key Findings
- **Daily cycle (IMF 3)**: Dominates short-term variability (34.84% energy)
- **Annual cycle (IMF 8)**: Captures seasonal variation (3.52% energy)
- **Ultra-long baseline (IMF 13)**: Represents stable long-term mean (46.77% energy)
- **27-year trend**: Nearly stable with slight decrease (-0.10 W/m²/year)

### Verification Results
| Test | Result | Status |
|------|--------|--------|
| Reconstruction completeness (RMSE) | 5.46e-14 | ✅ Passed |
| Baseline correlation (365-day MA) | r = 0.68 | ✅ Valid |
| Physical validity | Mean ≈ 269 W/m² | ✅ Valid |

## 🚀 Installation

### Prerequisites
- Python 3.9+
- numpy
- scipy
- tqdm
- pathos

### Install Modified Version

1. **Option A: Direct patch (recommended)**
   ```bash
   # Install original emd-signal
   pip install emd-signal==0.8.1
   
   # Apply the patch manually (see sift.py modification above)
   ```

2. **Option B: Using monkey-patching in your code**
   ```python
   import emd.sift as sift_module
   original_sift = sift_module.sift
   
   def aggressive_sift(X, sift_thresh=1e-8, energy_thresh=50, rilling_thresh=None,
                       max_imfs=None, verbose=None, return_residual=True,
                       imf_opts=None, envelope_opts=None, extrema_opts=None):
       return original_sift(X, sift_thresh=sift_thresh, 
                           energy_thresh=1e10,  # Disable energy threshold
                           rilling_thresh=None,  # Disable rilling
                           max_imfs=max_imfs, verbose=verbose, 
                           return_residual=return_residual,
                           imf_opts=imf_opts, envelope_opts=envelope_opts, 
                           extrema_opts=extrema_opts)
   
   sift_module.sift = aggressive_sift
   ```

### Usage Example
```python
import numpy as np
import pandas as pd
import emd

# Load your long time-series data
df = pd.read_csv('your_data.csv')
data = df['your_column'].values

# Decompose with extended IMF count
imfs = emd.sift.complete_ensemble_sift(
    data, 
    nensembles=10,     # Number of ensemble members
    max_imfs=15,       # Extract up to 15 IMFs (was limited to 8)
    energy_thresh=50,
    sift_thresh=1e-8
)

print(f"Extracted {imfs.shape[1]} IMFs")
print(f"IMF energy distribution: {[np.sum(imfs[:, i]**2)/np.sum(imfs**2)*100:.2f}% for i in range(imfs.shape[1])]}")

# Extract long-term baseline
baseline = imfs[:, -1]  # Last IMF is the ultra-long trend
```

## 📁 Project Structure

```
ceemdan-long-series/
├── LICENSE              # GPL v2+ License
├── README.md            # This file
├── requirements.txt     # Python dependencies
├── sift_patch.py        # Monkey-patch script (one-line fix)
├── sift_modification_notice.py  # Detailed modification documentation
└── SUMMARY.md           # Complete analysis report
```

## 📖 Documentation

See [SUMMARY.md](./SUMMARY.md) for:
- Detailed explanation of the modification
- Complete IMF statistics and analysis
- Verification methodology and results
- Physical interpretation of each IMF

## 🔬 Background

### Original Library
- **Name**: emd (EMD-signal)
- **Author**: Andrew Quinn et al.
- **Repository**: https://gitlab.com/emd-dev/emd
- **License**: GNU General Public License v2+
- **Version**: 0.8.1 (modified)

### Why This Modification?

The original library's internal white noise sift uses conservative stopping criteria (`energy_thresh=50`, `rilling_thresh=(0.05, 0.5, 0.05)`) that cause it to naturally stop after ~8 IMFs. For long time-series analysis where we need to extract ultra-long trends (spanning the entire data length), this limitation is problematic.

Our modification:
1. Passes `max_imfs` to internal sift calls
2. Optionally uses aggressive parameters (`energy_thresh=1e10`, `rilling_thresh=None`) for complete extraction
3. Successfully extracts coherent long-term components

## ⚠️ License Notice

This project is a **modified version** of the [emd](https://gitlab.com/emd-dev/emd) library, licensed under **GNU General Public License v2+**.

**Your obligations under GPL:**
1. ✅ Preserve this LICENSE file unchanged
2. ✅ Include this notice in your derivative works
3. ✅ Make your source code available when distributing
4. ✅ Apply the same GPL license to your modifications

## 📝 Citation

If you use this code in your research, please cite:

```bibtex
@software{ceemdan_long_series,
  author = {author9527},
  title = {CEEMDAN Long-Time-Series Extension},
  url = {https://github.com/author9527/ceemdan-long-series},
  version = {1.0.0},
  date = {2026-01},
  license = {GPL-2.0+}
}
```

Also cite the original emd library:
```bibtex
@software{emd_lib,
  author = {Andrew Quinn et al.},
  title = {EMD: Empirical Mode Decomposition},
  url = {https://gitlab.com/emd-dev/emd},
  version = {0.8.1},
  license = {GPL-2.0+}
}
```

## 🤝 Contributing

Contributions are welcome! Please note that by contributing, you agree to license your contributions under GPL v2+.

---

**Disclaimer**: This project is provided "as is" without warranty. Use at your own risk.
