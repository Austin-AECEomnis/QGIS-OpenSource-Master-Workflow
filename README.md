# 🗺️ QGIS Open-Source Master Workflow

A field manual for building a complete flood terrain analysis from raw federal data through automated PyQGIS scripting to a published GitHub Pages web map. Built and documented through hands-on work across two Texas Hill Country watersheds: the Guadalupe River corridor (Kerr County) and the Llano River corridor (Llano and Burnet Counties).

This repository documents not just what to do, but what goes wrong and why, capturing every CRS mismatch, every tool failure, every stale data trap, and every file organization mistake encountered across the full workflow so you don't have to.

**Platform:** QGIS 3.44.10-Solothurn | GDAL 3.12.4 | GRASS GIS | Python 3.12.13 | PROJ 9.8.1

---

## 📋 Quick Reference Index

Use this index to jump directly to what you need. If you are building this workflow for the first time, start at [Before You Start](#before-you-start) and read straight through.

**Setup & Planning**
- [Before You Start](#before-you-start): folder structure, naming conventions, end-destination planning
- [CRS & Projection Decisions](#crs--projection-decisions): the most common source of wasted time in this entire workflow

**Data**
- [Data Sourcing from Scratch](#data-sourcing-from-scratch): USGS, FEMA NFHL, OSM via Overpass Turbo
- [Data Sourcing: Detailed Reference](troubleshooting/data-sourcing.md)

**Analysis**
- [DEM & Terrain Analysis](#dem--terrain-analysis): reprojection, clipping, slope, flow accumulation
- [GRASS Tool Reference](troubleshooting/grass-tool-reference.md): r.watershed, r.slope.aspect, known failures and fixes

**Automation**
- [PyQGIS Script Workflow](#pyqgis-script-workflow): script structure, validated parameters, execution
- [Script Behavior & Stale Data](troubleshooting/script-behavior-and-stale-data.md): the invisible problem that costs the most time

**Publishing**
- [qgis2web Export & GitHub Pages](#qgis2web-export--github-pages): Leaflet export, file size, publishing
- [qgis2web & GitHub Pages: Detailed Reference](troubleshooting/qgis2web-and-github-pages.md)

**CRS Deep Dive**
- [CRS & Projection: Detailed Reference](troubleshooting/crs-and-projection.md)

**Order of Operations**
- [Sequence That Matters](#sequence-that-matters): what to build when, and why the order is not optional

---

## Before You Start

The single most expensive mistakes in this workflow happen before any tool is opened. Thirty minutes of planning here saves hours of debugging later.

### Know your end destination first

Before downloading a single file, decide where the finished product will live. This workflow ends at GitHub Pages, a Leaflet web map served from a public repository. That destination is compatible with UTM projected coordinates (EPSG:26914 for central Texas) throughout the analysis. Every tool, every output, every script parameter in this repository is built around that target.

If your end destination is ArcGIS Online instead, your CRS strategy changes entirely, see the [ESRI GIS Master Workflow](https://github.com/Austin-AECEomnis/ESRI-GIS-Master-Workflow) for that path. The two destinations require different projection decisions from the start, and trying to retrofit one into the other mid-project is where the worst time loss happens.

### Build your folder structure before touching any data

Create this structure before downloading anything:

```
YourProject_FloodAnalysis/
├── RawData/          ← source files only, never modified
├── Processed/        ← intermediate outputs (reprojected DEM, buffers)
└── Outputs/          ← final analytical outputs and script products
```

The separation between these three directories is not organizational preference, it is functional protection. A PyQGIS script that writes to Outputs cannot accidentally grab a corrupted intermediate from a previous run if intermediates live in Processed. This exact mistake happened in this project and cost a full debugging session. See [Script Behavior & Stale Data](troubleshooting/script-behavior-and-stale-data.md) for the full account.

### Name your folders correctly the first time

Rename before you create any file paths. A typo in the project folder name, even one missing letter, will propagate through every file path in every script. In this project the folder was created as `LlanoRiver_FloodAnalyis` (missing the second 's' in Analysis). Fixing it after layers were loaded required relinking every layer manually after renaming the folder in Windows File Explorer. Close QGIS first, rename, then reopen and relink when prompted.

### Understand what CRS your source data arrives in

None of the three data sources used in this workflow arrive in the same coordinate system:

| Source | Native CRS | Notes |
|--------|-----------|-------|
| USGS 3DEP DEM | EPSG:4269 NAD83 geographic | Must reproject to UTM before terrain tools |
| FEMA NFHL flood zones | EPSG:4269 NAD83 geographic | Must reproject to UTM before analysis |
| OSM building footprints via Overpass Turbo | EPSG:4326 WGS84 geographic | Must reproject to UTM before sampling |

Running any terrain analysis tool (slope, flow accumulation) on geographic coordinates (decimal degrees) produces unreliable results. Distance and elevation are in different units. Always reproject to a projected CRS before any terrain work. The target for central Texas is **EPSG:26914 NAD83 / UTM Zone 14N**.

---

## CRS & Projection Decisions

> For the full technical reference including the assigned-vs-embedded CRS trap and the zoom-to-layer diagnostic, see [CRS & Projection: Detailed Reference](troubleshooting/crs-and-projection.md).

The CRS decision tree for this workflow is straightforward once you understand the destination:

**For local terrain analysis only:** Use EPSG:26914 (UTM Zone 14N, NAD83) throughout. This is a projected metric CRS appropriate for central Texas. All terrain tools in QGIS and GRASS produce reliable results in this system.

**For publishing to GitHub Pages via qgis2web:** UTM works fine. Leaflet handles the reprojection to display coordinates automatically. No additional CRS conversion is needed before export.

**The most important CRS rule in this workflow:** A layer that *reports* the correct CRS in its Properties does not necessarily *contain* coordinates in that system. The "assigned CRS" is a label. The "embedded CRS" is what the coordinates actually are. When layers that should overlap do not overlap on the canvas, or when a layer zooms to an unexpected location, the coordinates themselves are wrong, not just mislabeled. The fix is to reproject from the actual source data, not to reassign the CRS label. See [CRS & Projection: Detailed Reference](troubleshooting/crs-and-projection.md) for the diagnostic process.

[↑ Back to top](#-quick-reference-index)

---

## Data Sourcing from Scratch

> For complete sourcing instructions including Overpass Turbo query syntax and FEMA package structure, see [Data Sourcing: Detailed Reference](troubleshooting/data-sourcing.md).

All three input datasets are freely available from federal and open sources. No licensed data is required.

### USGS 3DEP DEM

Download from the [USGS National Map Downloader](https://apps.nationalmap.gov/downloader). Select the 1/3 arc-second (approximately 10 meter) elevation product. Download the tile covering your study area as a GeoTIFF.

**Known issue: double file extension:** USGS downloads sometimes produce files named with a double `.tif.tif` extension. This is correct as downloaded. Windows Explorer hides known file extensions, so the file may appear to have no extension at all. Verify via the Type column in File Explorer. Script paths must reference the double extension exactly as stored. Do not rename the file.

### FEMA National Flood Hazard Layer

Download county flood zone shapefiles from [FEMA Map Service Center](https://msc.fema.gov). Select your county, download the NFHL package as a ZIP file.

**Known issue: Windows folder naming:** When Windows extracts a ZIP file, it sometimes creates a folder that retains the `.zip` suffix in its name (e.g. `NFHL_LlanoCounty.zip` becomes a folder literally named `NFHL_LlanoCounty.zip`). Script file paths must include that suffix. The flood zone polygon layer inside the package is always named `S_FLD_HAZ_AR.shp`.

### OSM Building Footprints via Overpass Turbo

Go to [overpass-turbo.eu](https://overpass-turbo.eu) and use this query structure, replacing the bounding box coordinates with your study area:

```
[out:json][timeout:25];
(
  node["building"](30.5,-99.0,31.0,-98.2);
  way["building"](30.5,-99.0,31.0,-98.2);
  relation["building"](30.5,-99.0,31.0,-98.2);
);
out body;
>;
out skel qt;
```

**Critical: verify the bounding box against actual geography before exporting.** In this project the first bounding box pulled Johnson City instead of Llano entirely because the coordinates were wrong. Always run the query first and confirm the results land where expected visually before exporting.

**Large export behavior:** If the query returns more than approximately 60,000 features, Overpass Turbo will warn you before rendering. Click Continue, then immediately click Export before the browser attempts to render the full dataset. Rendering 60MB+ of features will crash the browser tab. Export as GeoJSON.

**OSM coverage limitation:** OSM building coverage in rural Texas is sparse. Structure counts from this dataset reflect what OSM volunteers have mapped, not actual ground truth. Rural structures between incorporated areas will be significantly underrepresented.

[↑ Back to top](#-quick-reference-index)

---

## DEM & Terrain Analysis

### Reproject the DEM before running any terrain tools

Open the Processing Toolbox and run **GDAL → Raster projections → Warp (reproject)**:

- Input: your raw USGS DEM
- Target CRS: EPSG:26914
- Resampling method: Bilinear
- Output: save to your Processed folder

**Known issue: GDAL Warp hang:** On QGIS 3.44, the Warp tool occasionally completes the reprojection successfully but freezes on reporting back to the QGIS interface. If the tool appears to hang for more than 5 minutes, check whether the output file exists in your Processed folder and has a reasonable file size (the two-county Llano DEM produced a 456,031 KB output). If the file exists with size, the reprojection completed. Cancel the tool and use the existing file. Subsequent script runs can skip the reprojection step by checking for the file's existence first.

### Clip the DEM to your study area before running slope or flow accumulation

Running GRASS terrain tools on a full county-wide DEM is extremely slow and in some cases will not complete in a reasonable time. Always clip the DEM to a buffered version of your flood zone extent before running any terrain analysis.

In the Processing Toolbox run **GDAL → Raster extraction → Clip raster by mask layer** using a buffered version of your flood zone polygon. A 914-meter buffer (approximately 3,000 feet) is sufficient to capture all structure points within the raster extent. A tighter clip risks producing null terrain values on structure points near the flood zone boundary, and this happened in early runs of this project and produced incorrect structure counts.

### Generate slope

Use **GRASS → r.slope.aspect**. Set format to degrees (0). All other optional outputs (aspect, curvature) can be set to Skip Output or TEMPORARY_OUTPUT.

**Known warning: ERROR 6 SetColorTable:** This warning appears after slope generation on every run. It is a known harmless compatibility note between GRASS raster output and GeoTIFF color table handling. It does not affect the slope values or any downstream steps.

### Generate flow direction and flow accumulation

Use **GRASS → r.watershed** with a threshold of 10,000. This is the most computationally intensive step, expect 3 to 4 minutes on a two-county DEM. Do not cancel the run.

**GRASS r.watershed uses MFD (Multiple Flow Direction),** not the D8 single-direction algorithm used by ArcGIS. This is a fundamental difference. MFD distributes flow across multiple downslope neighbors. Flow accumulation values from this tool are not directly comparable to ArcGIS flow accumulation values. Thresholds calibrated in one platform cannot be applied to the other without recalibration.

**Negative flow accumulation values are expected MFD behavior.** GRASS flags cells with uncertain flow direction as negative values. These are not errors. Do not filter them out.

**Required parameter: threshold:** The r.watershed optional outputs (basin, stream, half_basin, etc.) require the threshold parameter to be set before they will process. Leaving optional outputs as TEMPORARY_OUTPUT without setting threshold causes the entire tool to fail. Set all unused optional outputs to Skip Output.

[↑ Back to top](#-quick-reference-index)

---

## PyQGIS Script Workflow

> For the full account of stale data issues, file lock errors, and the pre-run cleanup strategy, see [Script Behavior & Stale Data](troubleshooting/script-behavior-and-stale-data.md).

### Script structure overview

The validated PyQGIS script for this workflow executes seven sequential steps:

1. Reproject flood zone vectors to EPSG:26914
2. Fix OSM geometries, convert to centroids, reproject to EPSG:26914
3. Reproject DEM to EPSG:26914 (skip if output already exists)
4. Generate slope via GRASS r.slope.aspect
5. Generate flow direction and accumulation via GRASS r.watershed
6. Sample terrain values to structure centroids: three sequential passes
7. Apply terrain filter expression to produce final structure output

### Validated parameters: Llano River corridor

| Parameter | Value | Notes |
|-----------|-------|-------|
| Target CRS | EPSG:26914 | NAD83 / UTM Zone 14N |
| r.watershed threshold | 10,000 | Adjust for study area size |
| Elevation ceiling | elev_m < 420 | Valley floor extent at Llano and Kingsland |
| Slope ceiling | slope_deg < 20 | Excludes upland and hillside structures |
| Flow accumulation floor | flow_acc > 90 | Calibrated against recalibrated raster run |
| Original validated threshold | flow_acc > 15.88 | From first validated raster run |
| Final structure count | 149 | Current validated output |

**Flow accumulation thresholds vary between script runs.** Because GRASS r.watershed MFD values depend on the specific raster generation run, the threshold that produces a target structure count will differ if the flow accumulation raster is regenerated. Always run a diagnostic query on the Buildings_Final layer to find the correct threshold for the current raster set before applying the filter. See [GRASS Tool Reference](troubleshooting/grass-tool-reference.md) for the diagnostic approach.

### Column naming

The three-pass Sample Raster Values chain uses specific column prefixes to avoid collisions with existing OSM fields:

| Pass | Prefix | Generated column | Renamed to |
|------|--------|-----------------|------------|
| Elevation | elev | elev1 | elev_m |
| Slope | slp | slp1 | slope_deg |
| Flow accumulation | acc | acc1 | flow_acc |

Using a generic prefix like `1` collides with the OSM field `building_1`, causing QGIS to auto-increment column names unpredictably. The prefixes above are validated to avoid this.

### Running the script

Open the QGIS Python Console via **Plugins → Python Console**, click the Editor button (notepad icon), open or paste the script, confirm the BASE_DIR path matches your local folder structure, and click Run Script.

**Pre-run cleanup:** The script begins by calling `QgsProject.instance().removeAllMapLayers()` to release file locks from any previously loaded layers. This prevents WinError 32 permission errors on repeat runs. If a permission error still occurs after this call, close and reopen QGIS entirely before rerunning, as some locks are not released fast enough by the programmatic call alone.

**Extract by Expression will not overwrite existing output files.** Either delete the target file before running or use an incremented filename. The script's `delete_if_exists()` helper handles this automatically for all outputs.

[↑ Back to top](#-quick-reference-index)

---

## qgis2web Export & GitHub Pages

> For complete configuration settings and GitHub upload sequencing, see [qgis2web & GitHub Pages: Detailed Reference](troubleshooting/qgis2web-and-github-pages.md).

### Install qgis2web

qgis2web is a plugin, not a built-in QGIS feature. Install via **Plugins → Manage and Install Plugins**, search qgis2web, and install. Once installed a **Web** menu appears in the top menu bar.

### Prepare your layers before opening the export dialog

qgis2web exports whatever is currently loaded and styled in the project. Before opening the dialog:

- Rename layers in the Layers panel to the names you want displayed in the web map. The display name in QGIS is what qgis2web uses.
- Apply final symbology to all layers: flood zone categories, structure point styling
- Load a basemap from XYZ Tiles (OpenStreetMap works well)

### Key export settings

| Setting | Value | Reason |
|---------|-------|--------|
| Export format | Leaflet | Simpler, more broadly supported than OpenLayers |
| Precision | 3 decimal places | Reduces GeoJSON file size to under GitHub's 25MB limit |
| Layers list | Expanded | Shows layer toggle panel by default on load |

**SSL error on open:** qgis2web sometimes throws an SSL certificate error when the dialog first opens. Click ignore and reopen. It is harmless. qgis2web is checking for plugin updates and the error does not affect export.

**GeoJSON file size:** At full precision, the data folder in a two-county export can reach 180MB+, far over GitHub's 25MB per-file upload limit. Reducing precision to 3 decimal places is sufficient for web map display and brings the data folder to approximately 22MB.

### Upload to GitHub

Upload the contents of the webmap folder, not the folder itself, to your repository. Drag css, data, js, images, legend, markers, webfonts, and index.html directly into the GitHub upload area. GitHub has a total upload size limit per commit, so upload in batches if needed. Each batch commits successfully and they stack together.

### Enable GitHub Pages

In your repository go to **Settings → Pages**, set source to Deploy from a branch, select main branch and root folder, and click Save. The live URL appears after 1 to 2 minutes. Refresh the Pages settings page to see it. A blank page on first load is normal, so hard refresh with Ctrl+Shift+R after allowing 2 to 3 minutes for propagation.

### Add a title bar to index.html

Edit index.html directly in GitHub by clicking the pencil icon. Immediately after the `<body>` tag, add a new line and paste your title div. This is the safest edit to make. You are only adding content after an existing tag, not replacing anything. The `<body>` tag itself stays unchanged on its own line.

[↑ Back to top](#-quick-reference-index)

---

## Sequence That Matters

The order of operations in this workflow is not arbitrary. Several steps will fail silently or produce wrong results if run out of order.

**Always reproject before analyzing.** All terrain tools must receive data in a projected metric CRS. Running slope or flow accumulation on geographic coordinates produces results that appear to succeed but are numerically wrong.

**Always clip the DEM before running GRASS tools.** r.watershed on a full county-wide DEM may run for over an hour or not complete at all. The clip step is not optional for large study areas.

**Always delete or archive iterative intermediates before rerunning scripts.** Intermediate files from earlier debugging runs sitting in the same Outputs folder as your final layers are invisible to you but visible to the script. This is the root cause of the displaced buildings issue in this project. Earlier OSM centroid files with incorrect coordinates were picked up by a rerun of the script instead of being rebuilt from the raw source. See [Script Behavior & Stale Data](troubleshooting/script-behavior-and-stale-data.md).

**Always verify DEM coverage before running terrain analysis.** If structure points fall outside the raster extent, they return null values silently, with no error, just missing data. Confirm the DEM tile covers the full study area extent before running.

**Build scripts save output files across sessions but lose calibrated parameters.** The script file committed to GitHub is permanent, but the specific threshold values that produce a validated structure count (flow_acc > 90 for the current raster run) must be verified against the actual raster output each time the terrain rasters are regenerated. Document threshold values explicitly. Do not rely on memory.

[↑ Back to top](#-quick-reference-index)

---

## Related Repositories

This repository is part of a multi-platform flood vulnerability analysis series:

| Repository | Platform | Study Area | Key Finding |
|-----------|---------|------------|-------------|
| [ArcPy-Guadalupe-Terrain-Analysis](https://github.com/Austin-AECEomnis/ArcPy-Guadalupe-Terrain-Analysis) | ArcGIS Pro | Guadalupe River, Kerr County | 107 structures, Hunt VFD FEMA zone gap |
| [QGIS-LlanoRiver-FloodAnalysis](https://github.com/Austin-AECEomnis/QGIS-LlanoRiver-FloodAnalysis) | QGIS | Llano River, Llano + Burnet Counties | 149 structures, Llano city X zone gap |
| [USGS-Guadalupe-LiveStage](https://github.com/Austin-AECEomnis/USGS-Guadalupe-LiveStage) | GitHub Actions | Guadalupe River gauge at Hunt | Live stage pipeline |
| [ESRI-GIS-Master-Workflow](https://github.com/Austin-AECEomnis/ESRI-GIS-Master-Workflow) | ArcGIS ecosystem | Both watersheds | Full ESRI platform documentation |

---

## Supporting Reference Files

| File | Contents |
|------|----------|
| [troubleshooting/data-sourcing.md](troubleshooting/data-sourcing.md) | USGS, FEMA, OSM sourcing with full query syntax and known issues |
| [troubleshooting/crs-and-projection.md](troubleshooting/crs-and-projection.md) | CRS decision framework, diagnostic process, assigned vs embedded CRS |
| [troubleshooting/grass-tool-reference.md](troubleshooting/grass-tool-reference.md) | r.watershed and r.slope.aspect parameters, MFD behavior, threshold calibration |
| [troubleshooting/script-behavior-and-stale-data.md](troubleshooting/script-behavior-and-stale-data.md) | File lock errors, stale intermediate data, pre-run cleanup strategy |
| [troubleshooting/qgis2web-and-github-pages.md](troubleshooting/qgis2web-and-github-pages.md) | Full export configuration, file size management, GitHub Pages setup |

---

## Author

Austin Addington Berlin
Founder, AECE Omnis LLC | AI-GIS Convergence Research
[LinkedIn](https://linkedin.com/in/austinberlin) | [GitHub](https://github.com/Austin-AECEomnis) | [QGIS Web Map (qgis2web)](https://austin-aeceomnis.github.io/QGIS-LlanoRiver-FloodAnalysis/) | ArcGIS Online portfolio: [aeceomnis.maps.arcgis.com](https://aeceomnis.maps.arcgis.com)
