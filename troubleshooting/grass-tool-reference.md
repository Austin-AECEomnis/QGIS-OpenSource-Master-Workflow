# GRASS Tool Reference

Detailed reference for the two GRASS tools used in this workflow: r.slope.aspect for slope generation and r.watershed for flow direction and flow accumulation. Includes validated parameters, known warnings, algorithm behavior, and threshold calibration.

[← Back to Main Guide](../QGIS_Master_Workflow_README.md)

---

## Contents

- [Before Running Any GRASS Tool](#before-running-any-grass-tool)
- [r.slope.aspect: Slope Generation](#rslopeaspect-slope-generation)
- [r.watershed: Flow Direction and Flow Accumulation](#rwatershed-flow-direction-and-flow-accumulation)
- [MFD vs D8: Why Thresholds Cannot Be Transferred from ArcGIS](#mfd-vs-d8-why-thresholds-cannot-be-transferred-from-arcgis)
- [Flow Accumulation Threshold Calibration](#flow-accumulation-threshold-calibration)
- [Validated Parameters: Llano River Corridor](#validated-parameters-llano-river-corridor)
- [See Also](#see-also)

---

## Before Running Any GRASS Tool

**The DEM must be in a projected metric CRS before running any GRASS terrain tool.** Both r.slope.aspect and r.watershed require the input raster to be in a coordinate system where horizontal distance and elevation are in compatible units. UTM Zone 14N (EPSG:26914) is the correct CRS for central Texas. Running these tools on a geographic raster (decimal degrees) produces output that appears to succeed but contains numerically incorrect values.

**Clip the DEM to your study area before running r.watershed.** Running r.watershed on a full county-wide DEM is extremely slow and may not complete in a reasonable time. In this project, running on the full county DEM was cancelled after one hour without completion. After clipping to a flood zone buffer, r.watershed completed in 3 to 4 minutes on the same machine. See the DEM & Terrain Analysis section of the main guide for clip parameters.

**All optional outputs must be explicitly handled.** GRASS tools in QGIS require that every optional output field be set to either a file path or Skip Output. Leaving optional outputs as TEMPORARY_OUTPUT without setting the required threshold parameter causes the entire tool to fail with no useful error message. This is one of the most common first-run failures.

---

## r.slope.aspect: Slope Generation

### Purpose

Derives a slope raster from the input DEM. Slope values represent the maximum rate of elevation change at each cell, expressed in degrees.

### Validated parameters

| Parameter | Value | Notes |
|-----------|-------|-------|
| Elevation input | reprojected DEM in EPSG:26914 | Must be projected, not geographic |
| Format | 0 (degrees) | Use degrees, not percent |
| Model | 0 (default) | Horn's formula, standard for terrain analysis |
| Z scale factor | 1.0 | No vertical exaggeration |
| Minimum slope | 0.0 | Include flat terrain |
| Slope output | path in Outputs folder | The only required output |
| All other outputs | Skip Output | Aspect, curvature, etc. not needed for this workflow |
| GRASS_RASTER_FORMAT_OPT | GTiff | Explicitly force GeoTIFF output |

### Setting optional outputs

In the r.slope.aspect dialog, every optional output field (aspect, profile curvature, tangential curvature, dx, dy, dxx, dyy, dxy) must be set explicitly. Click each field and select Skip Output. Do not leave them as TEMPORARY_OUTPUT if the threshold issue described below has been encountered.

### Known warning: ERROR 6 SetColorTable

This warning appears after every successful r.slope.aspect run on QGIS 3.44:

```
ERROR 6: SetColorTable() not supported for multi-sample TIFF files.
```

This is a known harmless compatibility note between the GRASS raster output format and the GeoTIFF color table specification. The slope values in the output file are not affected. The file loads and samples correctly in all subsequent steps. Do not attempt to fix this warning.

### Known issue: Windows hides .tif extension

On Windows with file extension display disabled (the default), the output slope file may appear in File Explorer with no extension. The file is a valid GeoTIFF. Verify by checking the Type column in File Explorer, which will read TIF File or similar regardless of whether the extension is displayed. Script paths must reference the full filename including the .tif extension.

### Value range reference: Llano River corridor

Slope values in the Llano and Burnet County study area ranged from 0 to approximately 26.68 degrees. Structures flagged in the final output fell below 20 degrees, consistent with valley floor and low-gradient floodplain positions.

---

## r.watershed: Flow Direction and Flow Accumulation

### Purpose

Derives flow direction and flow accumulation surfaces from the input DEM using the Multiple Flow Direction (MFD) algorithm. Flow accumulation represents the number of upstream cells draining to each cell, used to identify drainage convergence zones where water concentrates.

### Validated parameters

| Parameter | Value | Notes |
|-----------|-------|-------|
| Elevation input | reprojected DEM in EPSG:26914 | Must be projected |
| Threshold | 10,000 | Minimum cell count for stream channel definition |
| Convergence | 5 | MFD convergence exponent, default |
| Memory | 300 | MB allocated to GRASS process |
| -a flag | True | Use positive flow accumulation |
| Accumulation output | path in Outputs folder | Flow accumulation raster |
| Drainage output | path in Outputs folder | Flow direction raster |
| All other outputs | Skip Output | Basin, stream, half_basin, etc. not needed |
| GRASS_RASTER_FORMAT_OPT | GTiff | Explicitly force GeoTIFF output |

### Required parameter: threshold

The threshold parameter (minimum number of cells required to define a stream channel) must be set for r.watershed to process correctly. A threshold of 10,000 worked well for the two-county Llano study area. Without the threshold, optional outputs set to TEMPORARY_OUTPUT will cause the tool to fail.

### Expected runtime

r.watershed on a clipped two-county DEM at 1/3 arc-second resolution runs approximately 3 to 4 minutes. On a full county-wide DEM without clipping, the tool exceeded one hour before being cancelled. Do not cancel the run once it has started on a properly clipped DEM. There is no progress indicator in the Python console during execution.

### Negative flow accumulation values

GRASS r.watershed MFD produces negative flow accumulation values for cells where the algorithm cannot determine a single dominant flow direction. This is expected behavior specific to the MFD algorithm and is not an error. These negative values appear throughout the output raster. Do not filter them out of the final buildings layer. Structures with negative flow accumulation values that otherwise pass the elevation and slope filters should be retained in the output.

---

## MFD vs D8: Why Thresholds Cannot Be Transferred from ArcGIS

This distinction caused significant confusion and debugging time in this project and is worth understanding clearly.

**ArcGIS Flow Direction and Flow Accumulation tools use the D8 algorithm.** D8 assigns the entire flow from each cell to a single downslope neighbor, the one with the steepest descent. Flow accumulation values represent the number of cells that drain through each cell via these single-direction paths. Values are always positive integers.

**GRASS r.watershed uses MFD (Multiple Flow Direction).** MFD distributes flow across all downslope neighbors proportionally based on slope gradient. This produces more hydrologically realistic accumulation patterns in low-gradient terrain where flow does not always concentrate into a single path. Values can be fractional and can be negative for uncertain-direction cells.

**The consequence:** A flow accumulation threshold of 100 in ArcGIS and a flow accumulation threshold of 100 in QGIS with GRASS do not represent the same physical condition. The numeric scales are different because the algorithms are different. Thresholds validated in one platform cannot be applied to the other.

In this project, the original ArcGIS-validated threshold for the Guadalupe corridor was approximately 100 (D8 units). When the same analysis was replicated in QGIS using GRASS MFD, the equivalent threshold was approximately 15.88 on the first validated raster run. On a subsequent raster regeneration run, the threshold shifted to approximately 90 for the same study area due to differences in the raster generation process.

Always calibrate the flow accumulation threshold against the actual raster output from the current run. Never assume a threshold from a previous run or a different platform will produce the correct structure count.

---

## Flow Accumulation Threshold Calibration

When the terrain analysis produces an unexpected structure count, use this diagnostic approach to find the correct flow accumulation threshold.

### Step 1: Load the Buildings_Final layer

Load the Buildings_Final.gpkg layer (the full set of structures with terrain values sampled, before the flow accumulation filter is applied) into QGIS.

### Step 2: Run the diagnostic script in the Python Console

Open the Python Console (Plugins, Python Console), click the Editor icon, and paste the following:

```python
from qgis.core import QgsVectorLayer

FINAL_OUT = r"C:\GIS_Projects\LlanoRiver_FloodAnalysis\Outputs\Buildings_Final.gpkg"

layer = QgsVectorLayer(FINAL_OUT, "diag", "ogr")
values = sorted([f['flow_acc'] for f in layer.getFeatures() if f['flow_acc'] is not None])

total = len(values)
print(f"Total structures with flow_acc values: {total}")
print(f"Min flow_acc: {values[0]:.4f}")
print(f"Max flow_acc: {values[-1]:.4f}")

for threshold in [10, 15, 20, 25, 30, 50, 75, 100, 150, 200, 500]:
    count = sum(1 for v in values if v > threshold)
    print(f"  flow_acc > {threshold:>6}  -->  {count} structures")
```

### Step 3: Narrow the range

The output will show structure counts at each threshold. Identify the range where your target count falls and run a second diagnostic with finer increments:

```python
for threshold in range(85, 100):
    count = sum(1 for v in values if v > threshold)
    print(f"  flow_acc > {threshold}  -->  {count} structures")
```

### Step 4: Accept the nearest achievable count

Flow accumulation values are continuous, so a specific target count (such as exactly 150) may not be achievable with an integer threshold. Multiple structures may share values between consecutive integer thresholds. Accept the nearest achievable count and document the threshold value used. In this project, 149 structures at flow_acc > 90 was accepted as the validated output after the raster regeneration run.

### Threshold reference: both runs of this project

| Run | Platform | Algorithm | Threshold | Count |
|-----|---------|-----------|-----------|-------|
| Original validated (Guadalupe, QGIS) | QGIS GRASS | MFD | SLOPE_1 <= 3.3563, FLOWACC_1 <= 49 | 107 |
| First Llano validated run | QGIS GRASS | MFD | flow_acc > 15.88 | 150 |
| Llano after raster regeneration | QGIS GRASS | MFD | flow_acc > 90 | 149 |
| Guadalupe ArcPy validated | ArcGIS Pro | D8 | SLOPE_VAL <= 0.179256, FLOWACC_VAL threshold | 107 |

---

## Validated Parameters: Llano River Corridor

Complete parameter set for the Llano River flood analysis, validated against the 2018 flood event study area.

```python
# r.slope.aspect
'format': 0,          # degrees
'model': 0,
'zscale': 1.0,
'min_slope': 0.0,
'GRASS_RASTER_FORMAT_OPT': 'GTiff'

# r.watershed
'threshold': 10000,
'convergence': 5,
'memory': 300,
'-s': False,
'-m': False,
'-4': False,
'-a': True,           # positive accumulation
'-b': False,
'GRASS_RASTER_FORMAT_OPT': 'GTiff'

# Terrain filter expression
'"elev_m" < 420 AND "slope_deg" < 20 AND "flow_acc" > 90'

# Value ranges in study area
# Elevation: 217.82 to 492.67 meters
# Slope: 0.00 to 29.58 degrees
# Flow accumulation: 1.0 to 22484.1
```

---

## See Also

- [CRS & Projection: Detailed Reference](crs-and-projection.md): why UTM is required before running GRASS tools
- [Script Behavior & Stale Data](script-behavior-and-stale-data.md): how stale raster files affect threshold calibration
- [Data Sourcing: Detailed Reference](data-sourcing.md): DEM download and preparation

[← Back to Main Guide](../QGIS_Master_Workflow_README.md) | [↑ Back to top](#grass-tool-reference)
