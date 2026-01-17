# CEEMDAN Extended - Modified emd-signal Library

A modified version of the emd-signal library that fixes the internal IMF count limitation.

## Problem

The original emd-signal library has an internal constraint that limits CEEMDAN decomposition to a maximum of 8 IMFs. This limit is embedded in the internal white noise sift process and causes an `IndexError` when attempting to extract more IMFs.

## Solution

The fix is straightforward: pass the `max_imfs` parameter through to internal sift calls.

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

## Important Note on max_imfs

The `max_imfs` parameter is an **upper limit**, not a target. The actual number of IMFs extracted depends on:
1. The `max_imfs` value you specify
2. The intrinsic characteristics of your data
3. The library's stopping criteria (energy threshold, rilling test, etc.)

The decomposition will naturally stop when the signal can no longer be meaningfully decomposed, regardless of `max_imfs`.

## Usage

1. Replace the original `sift.py` with this modified version
2. Use CEEMDAN normally with your desired `max_imfs` value:

```python
import emd

# Extract up to 15 IMFs (actual count may be less)
imfs = emd.sift.complete_ensemble_sift(data, max_imfs=15)
```

## Files

- `sift.py` - Modified library file (replace original)
- `LICENSE` - GNU General Public License v2

## License

This project is based on [emd-signal](https://github.com/laszukdawid/emd) and licensed under **GNU General Public License v2**.
