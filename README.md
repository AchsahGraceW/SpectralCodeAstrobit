# Spectral Code — AstroBit Hackathon: Kepler Exoplanet Detection

Team: Spectral Code
Challenge: AI-Based Detection of Earth-Like Exoplanets in Kepler Data (AstroBit, IIT Tirupati)

## Approach

Pipeline: `raw SAP flux → clean → search_smart → threshold → confidence → submission`

1. **Cleaning** (`clean()`): Quality-mask bad cadences, per-quarter median normalization,
   detrend via centered rolling median (1-day window) to remove stellar variability
   and instrumental drift while preserving transit signals.

2. **Period search** (`search_smart()`): Coarse-to-fine Box Least Squares (BLS) search.
   A wide log-spaced coarse sweep (20,000 trial periods) locates candidate peaks;
   each peak is then refined with a fine local search. Unlike the baseline approach
   (picking the single highest-SDE peak), we score each refined candidate by a
   **plausibility metric** combining SDE, transit depth, and number of observed
   transits — favoring physically consistent signals over statistically marginal
   ones with a higher raw SDE. This was validated against a case (KIC 4935189)
   where the baseline approach picked an incorrect 3.6-day noise peak over the
   true 9.32-day planetary signal (near-tied SDE, but the true signal was ~4x
   deeper with far more supporting transits).

3. **Threshold & confidence**: SDE threshold (11.0) and a plausibility-based
   logistic confidence score, calibrated against a 30-star dev sample.

4. **Systematic artifact filtering**: Post-hoc inspection of initial private-set
   detections revealed 8 stars flagged with near-identical periods (~3.0–3.2 days)
   and suspiciously maxed-out transit durations (19.2h, the search grid's upper
   bound) — consistent with an instrumental/systematic period rather than genuine
   transits. These were reclassified as non-detections with reduced confidence
   (0.15) rather than removed outright, reflecting residual uncertainty.

## Known limitations

- Shallow signals (~100–250 ppm) near or below the per-cadence noise floor were
  not reliably recovered by either the baseline rolling-median detrending or a
  tested Savitzky-Golay alternative (see notebook history) — consistent with the
  problem statement's difficulty framing for the "Earth analog" tier.
- No dedicated odd/even transit or secondary-eclipse vetting was implemented due
  to time constraints; the systematic-period filter was the primary false-positive
  mitigation applied.

## How to run

Open `astrobit_pipeline.ipynb` in Google Colab. Requires `train/`, `dev/`, and
`private/` folders (unzipped from the provided data packs) in the working directory,
alongside `train_labels.csv`, `train_truth.csv`, `dev_labels.csv`, `dev_truth.csv`.
Run all cells top to bottom. Final output: `submission_spectralcode.csv`.

## Dependencies

astropy, astroquery, numpy, pandas, matplotlib, scipy

## Additional finding (post-submission)

After the deadline, further testing identified that long-timescale
(multi-day to multi-month) instrumental/stellar drift was not fully
removed by the single-window rolling-median detrend, causing long-period
systematics to dominate the BLS periodogram on some stars. A two-stage
detrending approach (`clean_v4`: wide-window pass to remove long-term
drift, followed by the original narrow-window pass) was tested and
confirmed to fix this on a verified case (KIC 4935189: recovered period
error reduced from 3394% to 0.00%, SDE improved from 25.5 to 57.0).
Validation on medium/shallow-depth stars was not completed due to time
and environment constraints.
