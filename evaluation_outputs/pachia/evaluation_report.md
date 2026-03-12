# Pipeline Evaluation Report: pachia
**Generated**: 2026-03-12 18:28
## Data Summary
- **Site**: pachia
- **CRS**: EPSG:32635
- **Observation period**: 2017-04-23 to 2025-07-15
- **Total shorelines evaluated**: 10
- **Points per shoreline**: 200
- **Rolling-window reference**: ±365 days (min 5 obs)

## Accuracy Metrics
| Metric | Value |
|---|---|
| RMSE (local ref) | 38.10 m |
| RMSE (global ref) | 38.10 m |
| MAE (local ref) | 33.37 m |
| Bias (local ref) | 3.88 m |
| 90th percentile | 55.58 m |
| Effective resolution | 2.5 m (4× SR) |

## CoastSat Comparison
CoastSat reported RMSE of 7.0–13.0 m at open-coast sites using native 10 m Sentinel-2 imagery. This pipeline achieves an extraction-noise RMSE of 38.10 m with 2.5 m effective resolution (4× Real-ESRGAN super-resolution).

## Output Files
All outputs are saved in `evaluation_outputs/pachia/`:
- `evaluation_metrics.csv` — summary accuracy metrics
- `per_shoreline_statistics.csv` — per-observation statistics
- `azimuthal_sector_analysis.csv` — directional variability
- `cross_shore_distances.csv` — full distance matrix
- `fig*.png` — publication-quality figures (300 dpi)
- `evaluation_report.md` — this narrative summary
