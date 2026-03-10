# Satellite Imagery Processing Pipeline: Methods

## Introduction

Satellite-derived shoreline mapping has become a standard tool for coastal monitoring, with toolkits such as CoastSat (Vos et al., 2019) enabling automated extraction of shoreline positions from publicly available Landsat and Sentinel-2 imagery. CoastSat established a practical workflow for continental sandy coastlines: imagery is retrieved via Google Earth Engine, cloud-covered images are discarded, a neural network classifier segments pixels into land/water classes, and the Modified Normalized Difference Water Index (MNDWI) with Otsu thresholding locates the shoreline at sub-pixel resolution along shore-normal transects. At validated open-coast sites, this approach achieves horizontal accuracies of 7–13 m RMSE.

However, several characteristics of small, dynamic islands expose limitations in this approach. First, CoastSat operates at the native sensor resolution (10 m for Sentinel-2, 15–30 m for Landsat). On islands whose total perimeters span only tens to hundreds of pixels, the shoreline position uncertainty approaches or exceeds the actual range of shoreline movement, making change detection unreliable. Second, CoastSat applies no geometric coregistration between images, relying instead on the provider's orthorectification. Inter-image misalignment of even one pixel (10 m) can introduce spurious shoreline shifts comparable to real erosion or accretion. Third, CoastSat discards all images exceeding a cloud cover threshold; it does not reconstruct partially cloudy images. In tropical and monsoon-affected regions where small islands are common, this can eliminate a large fraction of the available time series. Fourth, the transect-based shoreline representation assumes a roughly linear or gently curving coast; it does not naturally represent closed island perimeters. Finally, CoastSat uses tile-level cloud cover metadata for filtering, which is unreliable for small AOIs embedded within large Sentinel-2 tiles.

This pipeline addresses these limitations through a sequence of processing steps designed for small island monitoring. It introduces 4× super-resolution to increase the effective pixel size from 10 m to approximately 2.5 m before segmentation. It applies automated inter-image coregistration with temporal chaining and quality-controlled shift recovery, ensuring consistent geometric alignment across long time series. It reconstructs cloud-contaminated pixels rather than discarding entire images, preserving temporal coverage. It uses pixel-level cloud screening within the AOI rather than tile-level metadata. And it represents shorelines as closed spline curves with normal-vector-based tidal correction, rather than fixed shore-normal transects.

The pipeline is organized as a modular sequence of processing steps. Each step reads the output of the previous step and writes its results to a defined directory, allowing individual steps to be re-run or skipped.

> **[Figure 1: Pipeline overview.]** Flow diagram showing the full processing sequence from data acquisition through GeoJSON export. Each box represents a pipeline step, with arrows indicating data flow. Inputs (Sentinel-2, FES2022, site metadata) enter at the top; outputs (georeferenced shoreline time series, GeoJSON files) exit at the bottom. Intermediate data products (GeoTIFFs, PNGs, CSVs) are labeled on the connecting arrows.

### Data Acquisition

Sentinel-2 Level-2A (surface reflectance) imagery is acquired through the Planetary Computer STAC API. For a given site, the pipeline queries the catalog using a bounding box derived from the site's area of interest (AOI), a date range, and a cloud cover threshold.

**Search and filtering.** The STAC query uses a deliberately generous scene-level cloud cover limit (the greater of the user-specified threshold or 90%). This is because the Sentinel-2 scene classification applies to entire 110 × 110 km tiles. A tile flagged at 60% cloud cover may be entirely clear over a small island AOI that spans only a few hundred meters. Fine-grained cloud filtering is performed separately at the pixel level (see Cloud Imputation).

**Band streaming.** Rather than downloading full tiles, the pipeline streams only the pixels within the AOI from Cloud-Optimized GeoTIFFs (COGs). When multiple tiles from the same solar day cover the AOI, they are automatically mosaiced. The download produces three outputs per acquisition date:

- **6-band GeoTIFF** containing Blue (B02, 490 nm), Green (B03, 560 nm), Red (B04, 665 nm), NIR (B08, 842 nm), Narrow NIR (B8A, 865 nm), and SWIR (B11, 1610 nm), stored at 10 m resolution.
- **RGB preview** (PNG) composed from Red, Green, and Blue bands, normalized to 8-bit.
- **NIR composite** (PNG) derived from a blend of the NIR band and the Normalized Difference Water Index (NDWI). NDWI is computed as −(Green − NIR) / (Green + NIR), which renders land as bright and water as dark. The NIR band and NDWI values are averaged and normalized to 8-bit. This blended composite accentuates the land-water boundary for downstream segmentation.

**Pixel-level cloud screening.** Before downloading the full-resolution bands for a given date, the pipeline loads only the Scene Classification Layer (SCL) at its native 20 m resolution. Pixels classified as medium-probability cloud (class 8), high-probability cloud (class 9), or thin cirrus (class 10) are counted. If the proportion of cloudy pixels within valid (non-nodata) AOI pixels exceeds the user-defined threshold, that date is skipped. Unlike tile-level cloud metadata used by CoastSat, this pixel-level approach evaluates cloud conditions within the AOI itself, preventing the unnecessary rejection of scenes that are locally clear over a small island.

**Coverage check.** After cloud screening, the pipeline verifies that the downloaded bands contain at least 1% valid (non-zero) pixels within the AOI. Dates with insufficient spatial coverage are discarded.

**Site metadata.** Site parameters—AOI coordinates, date ranges, and processing history—are stored in a BigQuery table. The pipeline can also read from a local CSV file as a fallback.

> **[Figure 2: Data acquisition outputs for a single date.]** Three-panel image showing the products generated for one acquisition date: (a) RGB preview PNG, (b) NIR/NDWI blended composite PNG, and (c) false-color rendering of the 6-band GeoTIFF. The AOI boundary is overlaid on each panel. An inset map shows the AOI footprint relative to the full Sentinel-2 tile to illustrate the scale difference relevant to cloud filtering.

### Coregistration

Satellite images of the same location acquired on different dates exhibit small spatial misalignments (sub-pixel to several-pixel shifts) due to variations in sensor orientation, orbital geometry, and terrain projection. If left uncorrected, these shifts introduce positional errors in the extracted shorelines that can exceed the actual shoreline movement. This is particularly consequential for small islands, where typical shoreline changes may be on the order of a few meters—comparable to or smaller than uncorrected misalignment. CoastSat applies no coregistration, relying on provider orthorectification alone. This pipeline corrects inter-image alignment explicitly.

**Shift estimation.** Each image pair is aligned using global coregistration, which computes a single translational offset (x, y) per image using FFT-based phase correlation in the frequency domain. The algorithm identifies the displacement that maximizes cross-correlation between a reference image and a target image within a matching window (default: 256 × 256 pixels, maximum shift: 100 pixels). The result includes the shift magnitude in pixels and meters, a reliability score, and structural similarity (SSIM) values before and after correction.

**Chunked temporal processing.** Long time series (spanning years) are divided into overlapping temporal chunks to avoid error accumulation. By default, each chunk spans 12 months with a 3-month overlap between consecutive chunks (producing a 9-month step). Within each chunk:

1. The first chunk selects its temporally central image as the global reference (assigned a shift of zero).
2. All other images in that chunk are coregistered to this reference.
3. For subsequent chunks, the reference image is selected from the overlap period with the previous chunk—specifically the overlap image closest to the temporal center. This image was already coregistered in the prior chunk, so its accumulated shift relative to the global reference is known.
4. Each image's total shift equals its local shift (relative to the chunk reference) plus the accumulated shift of that chunk reference back to the global reference.

Shifts are applied by modifying the GeoTIFF affine geotransform (translation components only), without resampling pixel data.

**Quality filtering.** After coregistration, each result is evaluated against multiple criteria: shift reliability score (minimum 40 out of 100), FFT window size (minimum 50 pixels per dimension), maximum shift magnitude (250 m), and statistical outlier detection using a combined z-score on the shift vector components (threshold: 2.0 standard deviations). Images that fail any criterion are separated into a failed set.

**Recovery of failed images.** Failed images are re-coregistered against their nearest temporal neighbor that passed quality filtering. The shift from this second pass is chained with the neighbor's known shift to produce an estimated total shift. If re-coregistration also fails, the neighbor's shift is assigned as a best-guess fallback. This ensures every image in the time series has a shift estimate, preventing temporal gaps in downstream processing. The complete shift table is saved as a separate file used by later pipeline steps.

> **[Figure 3: Chunked coregistration with temporal chaining.]** Timeline diagram showing how a multi-year image series is divided into overlapping 12-month chunks with 3-month overlap. Within each chunk, arrows connect target images to the chunk reference. Between chunks, the overlap region is highlighted, showing how the reference for chunk N+1 inherits its accumulated shift from chunk N. A second panel shows a scatter plot of measured (x, y) shifts across all images, with passed and failed quality-filter results distinguished by color.

### Cloud Imputation

Despite the pixel-level cloud screening during download, some images retain partial cloud cover. Rather than discarding these images—as is standard in CoastSat and similar tools—the pipeline reconstructs cloud-contaminated pixels to produce a fully usable image. This is important for island sites in persistently cloudy regions, where discarding all partially cloudy scenes would leave large temporal gaps.

**Cloud detection.** A semantic segmentation model classifies each pixel as clear, thin cloud, thick cloud, or cloud shadow. The model operates on the multi-band Sentinel-2 data and produces a per-pixel mask. If the image dimensions are below the model's minimum input size, the image is upsampled before inference and the mask is mapped back to the original resolution.

**Pixel reconstruction.** Masked (cloudy) pixels are reconstructed using variational probabilistic interpolation. The algorithm selects the least-cloudy image in the time series as a reference (features image), then fills each masked pixel by learning the spatial relationship between the reference image and the target image in clear regions and applying that relationship to the cloudy regions. A similarity threshold (default: 0.2) controls the interpolation fidelity. The output is a cloud-free multi-band GeoTIFF for each image.

> **[Figure 4: Cloud imputation example.]** Three-panel image for a single date: (a) the original partially cloudy image with the cloud mask overlay, (b) the reference (least-cloudy) image used for interpolation, and (c) the reconstructed cloud-free output. Difference insets highlight the imputed regions.

### RGB and NIR Composite Creation

After cloud imputation, the pipeline regenerates RGB and NIR composites from the cloud-free GeoTIFFs. This step handles two possible band layouts: a 6-band format (produced by this pipeline's download step) and a 12-band format (produced by some cloud imputation outputs). The module identifies the correct band positions by inspecting band descriptions in the file metadata, falling back to positional indexing based on the total band count.

The NIR composite is computed identically to the download step: the NIR band and NDWI are blended and normalized. RGB composites are formed from Red, Green, and Blue bands, each normalized by the joint maximum to preserve color balance. The output images are center-cropped to match the dimensions of the originals, accounting for any slight size differences introduced by reprojection during cloud imputation.

### Resolution Enhancement

Sentinel-2 imagery at 10 m resolution is too coarse for precise shoreline delineation on small islands, where total shoreline perimeters may span only tens of pixels. CoastSat operates at the native sensor resolution, applying only bilinear downsampling of the SWIR band from 20 m to 10 m and, for Landsat, pansharpening from 30 m to 15 m. Neither approach changes the effective spatial resolution of the shoreline detection. This pipeline instead upsamples each cloud-free composite by a factor of 4 using a generative adversarial network trained for real-world image super-resolution. The model synthesizes plausible high-frequency details (textures, edges) consistent with the low-resolution input, increasing the effective resolution to approximately 2.5 m. The upsampled images are written as PNGs for input to the segmentation step.

> **[Figure 5: Resolution enhancement comparison.]** Side-by-side comparison of the same island AOI at (a) the original 10 m Sentinel-2 resolution and (b) the 4× upsampled ~2.5 m resolution. Zoomed insets of the land-water boundary show the difference in edge definition between the two resolutions.

### Normalization

The upsampled NIR images are screened for defects and normalized. Each image is checked for the presence of pixel values within a configurable grey-level range (default: 25–230). Images lacking values in this range—indicating saturation, severe noise, or blank data—are discarded. Remaining images are linearly rescaled so that pixel intensities span the full 0–255 range, ensuring consistent input contrast for segmentation regardless of variations in illumination or atmospheric conditions across the time series.

### Segmentation

The normalized NIR images are processed by an instance segmentation model that produces a binary land-water mask for each image. The model accepts RGB or single-channel images, applies contrast enhancement, and pads the image boundary (default: 256 pixels) to prevent edge artifacts in the detection bounding box. After inference, the predicted mask is cropped back to the original dimensions. If multiple disconnected land regions are detected, only the largest connected component is retained, producing a single-island mask. The output is a binary PNG per image.

This approach differs from CoastSat's shoreline detection in two ways. CoastSat classifies pixels into four categories (sand, water, white-water, other land features), then applies Otsu thresholding on the MNDWI to locate the shoreline boundary between sand and water classes. That method was trained on and validated against open-coast sandy beaches. For small islands—which lack white-water zones and fixed sandy transects, and where the entire landmass may occupy a small portion of the image—a direct segmentation of the land-water mask followed by contour extraction is more appropriate. The instance segmentation model produces a closed boundary around the full island perimeter, rather than detecting a shoreline segment along a linear coast.

> **[Figure 6: Segmentation outputs.]** Three-panel image: (a) the normalized NIR composite input, (b) the raw segmentation mask with padding boundary visible, and (c) the cleaned binary mask after largest-connected-component extraction. The extracted contour is overlaid on the input image in panel (a).

### Boundary Extraction

The binary segmentation mask is converted into a shoreline polyline. Contours are detected at the 0.5 intensity level using the marching squares algorithm, and the longest contour is selected as the primary shoreline. The raw contour is simplified using the Visvalingam-Whyatt algorithm (tolerance: 1 unit) to reduce point count, then smoothed with a moving average filter (window size: 3 points). The result is saved as a CSV of (x, y) pixel coordinates.

Optionally, an implicit signed distance function (SDF) can be fit to the mask using a neural network (4-layer MLP with 128-dimensional hidden layers, trained for 1000 iterations). The SDF produces evenly spaced boundary points (default: 200 points), which can be useful for consistent temporal comparison.

### Boundary Refinement

The extracted shoreline is refined to sub-pixel accuracy by sampling image intensity along transects perpendicular to the shoreline.

**Curve fitting.** A NURBS (Non-Uniform Rational B-Spline) curve of degree 3 is fit to the extracted shoreline points. Control points are adaptively spaced and the curve is evaluated at fine intervals (default parameter spacing: 0.005) to generate densely sampled points along the shoreline and corresponding normal vectors.

**Transect analysis.** At each evaluation point, a transect is cast perpendicular to the curve. The pixel intensities of the underlying image are sampled along this transect (half-length determined by image dimensions, minimum 16 pixels). Self-intersection detection prevents sampling beyond points where the transect would re-cross the shoreline in concave regions. This self-intersection handling is necessary for closed island perimeters with concavities—a geometry that CoastSat's fixed shore-normal transects do not accommodate.

**Boundary detection.** Along each transect, the land-water boundary is located using gradient analysis: finite differences identify the locations of steepest intensity change, and a rolling average across neighboring transects selects the most consistent boundary position. An alternative method using k-means clustering (k=2) on transect intensities is available as a fallback.

**Validation.** Each refined shoreline is checked for geometric plausibility. Shorelines whose enclosed area falls below 5% of the image area, or whose perimeter exceeds 3 times the image perimeter, are rejected. The refined coordinates are saved as a CSV.

> **[Figure 7: Boundary refinement process.]** (a) The extracted shoreline (from marching squares) overlaid on the NIR image, with the fitted NURBS curve shown as a smooth line. (b) Detail view showing several normal transects cast perpendicular to the curve, with sampled pixel intensities plotted along each transect. The detected land-water boundary point on each transect is marked. (c) The initial extracted shoreline compared to the refined shoreline, showing the sub-pixel adjustment at representative points.

### Geospatial Transformation

The refined shoreline coordinates (in upsampled pixel space) are converted to geographic (projected) coordinates using the affine geotransform embedded in the original GeoTIFF.

Three corrections are applied during transformation:

1. **Super-resolution scaling.** Pixel coordinates are divided by the upsampling factor (4) to map back to the original 10 m pixel grid.
2. **Crop offset.** The pipeline accounts for any dimensional differences between the GeoTIFF and the PNG composites (caused by center-cropping during composite creation) by computing a pixel offset: (tiff_dimension − png_dimension) / 2.
3. **Coregistration shift.** The per-image translational shift from the coregistration step is added to the affine transform's translation components, ensuring the georeferenced shoreline reflects the corrected image alignment.

The resulting coordinates are in the projected coordinate reference system (typically UTM) of the source imagery. Rows containing invalid values (NaN or infinity) cause that file to be skipped.

> **[Figure 8: Geospatial transformation corrections.]** Diagram showing the three coordinate corrections applied during transformation. Left panel: the upsampled pixel grid (4× resolution) mapped back to the original 10 m grid by scaling. Center panel: the crop offset between the GeoTIFF extent and the PNG extent, with the offset vector illustrated. Right panel: the coregistration shift vector applied to the affine geotransform, with before and after positions of a sample shoreline point.

### Shoreline Filtering

The georeferenced shoreline set is filtered to remove anomalous or defective results using a multi-stage process.

**Spatial clustering.** The centroid position and total polyline length of each shoreline are normalized to [0, 1]. DBSCAN clustering identifies a main spatial cluster; shorelines outside this cluster (indicating mislocated detections) are removed. This filter activates only when the dataset contains 50 or more shorelines.

**Temporal consistency filtering.** For datasets exceeding a configurable size threshold (default: 50 shorelines), each shoreline is compared to its temporal neighbors. A B-spline is fit to each shoreline, and the closest-point distance between consecutive shorelines is computed at sampled positions. A shoreline is flagged as defective if more than a threshold fraction of its sampled points (default: configurable ratio) have distances exceeding a multiple (default: configurable) of the global mean inter-shoreline distance, and the displacement directions from the preceding and following shorelines are aligned (within 90°, indicating a consistent outward or inward shift inconsistent with typical evolution). This directional check prevents filtering of genuine rapid changes.

Filtered shorelines are copied to a separate directory for use in subsequent steps.

> **[Figure 9: Shoreline filtering.]** (a) 3D scatter plot of all georeferenced shorelines in normalized centroid-x, centroid-y, and perimeter-length space, with the DBSCAN main cluster and outliers colored differently. (b) Time-series plot of consecutive inter-shoreline distances, with the global mean distance shown as a horizontal line and defective shorelines highlighted. (c) Map view showing retained shorelines (solid lines) and rejected shorelines (dashed lines) overlaid on an example image.

### Tidal Modeling

The tidal elevation at each image acquisition time is computed using harmonic tidal prediction. The pipeline extracts the center coordinates of the site's AOI and the UTC timestamps of all images in the processing file.

Tidal elevations are modeled using the FES2022 (Finite Element Solution) global ocean tide model, which provides harmonic constituents on an unstructured grid. Constituent amplitudes and phases are interpolated to the site location (default method: bilinear), and tidal heights are predicted for each timestamp by summing constituent oscillations and inferring minor corrections. Extrapolation beyond the model domain is enabled by default, with a cutoff distance of 10 km.

The predicted tidal height at each timestamp is converted to a horizontal shoreline displacement using:

$$\text{correction} = \frac{\text{tide} - \text{reference elevation}}{\text{beach slope}}$$

where reference elevation is the vertical datum and beach slope is the tangent of the foreshore slope angle (default: 0.08). The corrections are saved as a CSV for application in the next step.

CoastSat recommends tidal correction using the same slope-based formula but does not implement it; correction is left to the user. This pipeline integrates tidal modeling and correction as automated steps, using a global tide model rather than requiring access to local tide gauge data.

### Tidal Correction

Each filtered shoreline is adjusted for the tidal state at the time of image acquisition. A B-spline curve is fit to the georeferenced shoreline coordinates, producing 200 evenly spaced evaluation points. At each point, a normal vector is computed from the curve tangent (rotated 90°). The tidal horizontal correction (from the Tidal Modeling step) is applied by translating each point along its normal vector by the correction distance. This shifts the entire shoreline inward or outward uniformly, approximating the horizontal position the shoreline would occupy at the reference tidal elevation.

This spline-and-normal approach differs from CoastSat's transect-based correction. CoastSat applies tidal shifts along fixed shore-normal transects, which assumes a roughly linear coastline. For closed island perimeters, the normal-vector approach computes correction directions from the actual shoreline geometry at each point, correctly handling convex, concave, and irregularly shaped boundaries.

The corrected shoreline is matched to the closest tidal prediction by date. The tidally corrected coordinates are saved as a new CSV.

> **[Figure 10: Tidal correction on a closed island perimeter.]** (a) The predicted tidal elevation time series at the site, with the reference elevation marked as a horizontal line and the image acquisition times marked as points. (b) Plan view of a single shoreline showing the original georeferenced position (solid line) and the tidally corrected position (dashed line), with normal vectors drawn at representative points showing the direction and magnitude of the correction. The contrast with a transect-based correction along a linear coast is illustrated in an inset.

### GeoJSON Conversion and Data Export

The final step converts processed shoreline CSVs (both geotransformed and tidally corrected variants) to GeoJSON format for visualization and distribution.

**Coordinate transformation.** Shoreline coordinates are reprojected from the source UTM projection to WGS84 (EPSG:4326) for web mapping compatibility.

**GeoJSON structure.** Each shoreline is represented as a GeoJSON LineString feature within a FeatureCollection. Properties include the site name, image date, processing timestamp, coordinate system, correction status, and point count.

**Cloud storage and metadata.** GeoJSON files are uploaded to cloud storage with public read access for direct web retrieval. Metadata for each shoreline—including a unique identifier, acquisition date, processing date, public URL, bounding box geometry, total length, feature count, and a quality score—is recorded in a data warehouse for programmatic query access.

## Pipeline Execution Modes

**Full mode.** All steps execute sequentially for the complete date range of available imagery.

**Update mode.** The pipeline checks for images processed in previous runs, downloads only new acquisitions, processes them through all steps, and merges the results with existing data. Backups of prior results are created before merging.

**Step selection.** Individual steps can be skipped or run in isolation. This supports re-processing specific stages without repeating the full pipeline.

## Data Sources

| Source | Purpose |
|--------|---------|
| Sentinel-2 Level-2A | Primary optical imagery (10 m visible/NIR, 20 m SWIR/SCL) |
| Scene Classification Layer (SCL) | Pixel-level cloud, shadow, and surface classification |
| FES2022 | Global ocean tide harmonic constituents |
| BigQuery | Site metadata storage, processing history, and shoreline data catalog |
