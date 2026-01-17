# CEEMDAN Extended

A simple monkey-patch for the emd-signal library that enables extraction of more than 8 IMFs from the CEEMDAN algorithm.

## Problem

The original emd-signal library internally limits the number of IMFs that can be extracted from white noise during CEEMDAN decomposition, causing an `IndexError` when requesting more than 8 IMFs.

## Solution

This patch modifies the internal behavior by:
1. Passing `max_imfs` parameter to internal sift calls
2. Disabling energy threshold and rilling criteria for complete extraction

## Usage

```python
from sift_patch import apply_patch
apply_patch()

import emd
imfs = emd.sift.complete_ensemble_sift(data, max_imfs=15)
```

## Files

- `sift_patch.py` - The monkey-patch module
- `LICENSE` - GNU General Public License v2
