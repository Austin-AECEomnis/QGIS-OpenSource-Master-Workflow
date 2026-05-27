# Script Behavior & Stale Data

The most time-consuming problems in this workflow were not tool failures or CRS errors. They were cases where the script ran successfully, produced output, and the output was wrong because of stale intermediate files the script could not distinguish from fresh ones. This file documents every instance of that problem, why it happened, and the organizational strategy that prevents it.

[← Back to Main Guide](../QGIS_Master_Workflow_README.md)

---

## Contents

- [The Core Problem](#the-core-problem)
- [Folder Separation as Protection](#folder-separation-as-protection)
- [The Displaced Buildings Incident](#the-displaced-buildings-incident)
- [Pre-Run Cleanup Strategy](#pre-run-cleanup-strategy)
- [File Lock Errors: WinError 32](#file-lock-errors-winerror-32)
- [GDAL Warp Hang](#gdal-warp-hang)
- [Windows File Extension Visibility](#windows-file-extension-visibility)
- [Session Boundaries and Saved Parameters](#session-boundaries-and-saved-parameters)
- [See Also](#see-also)

---

## The Core Problem

A PyQGIS script has no concept of which files in a folder are final outputs and which are corrupted intermediates from a previous debugging run. When the script is told to read `Buildings_Centroids_UTM14N.gpkg`, it reads whatever file has that name in that location. If that file contains coordinates from an earlier incorrect run, the script processes incorrect data and produces incorrect output, with no error or warning.

This is invisible. The script reports success. The output file is created. The structure count may even be close to expected. The only way to detect the problem is to load the output layer and inspect its geographic location, which is easy to skip when a run appears to have succeeded.

The solution is not vigilance. Vigilance fails. The solution is a folder structure that makes it physically impossible for the script to reach old intermediate files.

---

## Folder Separation as Protection

The three-folder structure documented in the main guide is not a tidiness preference. It is the mechanism by which the script is prevented from consuming its own stale outputs.

```
YourProject/
├── RawData/      ← read-only source data, never written to by scripts
├── Processed/    ← expensive intermediates (reprojected DEM)
└── Outputs/      ← all script-generated outputs
```

**RawData is never written to.** The script reads from RawData but never writes there. If a RawData file is corrupt or wrong, the script will consistently fail in the same way every run, making the problem immediately visible.

**Processed holds only the computationally expensive DEM reprojection output.** The reprojected DEM takes significant time to produce and does not need to be regenerated on every run. It lives in Processed rather than Outputs so that clearing Outputs before a fresh run does not force a DEM reprojection.

**Outputs holds all building-chain intermediates and final outputs.** When debugging requires a clean rebuild of the building analysis chain, delete everything in Outputs except the terrain rasters (Llano_Slope.tif, Llano_FlowDir.tif, Llano_FlowAcc.tif). The script will rebuild every building-chain file from the raw OSM source.

**What to delete before a clean run of the building chain:**

```
Outputs/Buildings_Fixed.gpkg
Outputs/Buildings_Centroids.gpkg
Outputs/Buildings_Centroids_UTM14N.gpkg
Outputs/Buildings_Sampled_1.gpkg
Outputs/Buildings_Sampled_2.gpkg
Outputs/Buildings_Sampled.gpkg
Outputs/Buildings_Renamed_1.gpkg
Outputs/Buildings_Renamed_2.gpkg
Outputs/Buildings_Final.gpkg
Outputs/Buildings_Final_150.gpkg
```

**What to preserve:**

```
Outputs/Llano_Slope.tif       (expensive GRASS output, reuse if valid)
Outputs/Llano_FlowDir.tif     (expensive GRASS output, reuse if valid)
Outputs/Llano_FlowAcc.tif     (expensive GRASS output, reuse if valid)
```

---

## The Displaced Buildings Incident

This is the specific incident that cost the most time in the Llano River project and directly motivated the organizational guidance above.

**What happened:** During iterative script debugging, multiple runs of the building centroid and reprojection steps produced intermediate GeoPackage files in the Outputs folder. Some of these intermediate files contained building centroids with incorrect geographic coordinates, positioned approximately 80 kilometers south of the study area in the Johnson City and Blanco area.

When a later script run ran successfully and produced Buildings_Final_150.gpkg, the zoom-to-layer test showed the structures landing far south of Llano city rather than in the correct location. All three layers reported EPSG:26914 in Properties. The project CRS was correct. The coordinates were wrong.

**Root cause:** The script was reading Buildings_Centroids_UTM14N.gpkg from a previous incorrect run rather than rebuilding the centroid layer from the raw OSM GeoJSON. The file existed from an earlier session, its filename matched what the script expected, and the script used it without question.

**The fix:** Delete all building-chain .gpkg files from the Outputs folder, keeping only the three terrain rasters. Rerun the script. The building centroid chain rebuilt from the raw OSM source, produced correct coordinates, and the structures landed correctly over Llano city and the Kingsland corridor.

**The lesson:** A script run that completes without errors is not a script run that produced correct output. Always verify the geographic location of the final output layer after every run that rebuilds the building chain, not just after runs that report errors.

---

## Pre-Run Cleanup Strategy

The PyQGIS script begins with this call:

```python
QgsProject.instance().removeAllMapLayers()
```

This removes all layers currently loaded in the QGIS project before the script begins writing any output files. Its purpose is to release file locks that QGIS holds on any previously loaded GeoPackage files. Without this call, attempting to delete or overwrite a loaded layer file produces a WinError 32 permission error even if `delete_if_exists()` is called first.

The script's `delete_if_exists()` helper then removes any existing output files before writing new ones:

```python
def delete_if_exists(path):
    if os.path.exists(path):
        os.remove(path)
        print(f"  Removed existing file: {os.path.basename(path)}")
```

This combination handles most repeat-run scenarios. The pre-run layer removal releases locks, and the delete helper removes stale files before the script writes fresh ones.

---

## File Lock Errors: WinError 32

**Symptom:**

```
PermissionError: [WinError 32] The process cannot access the file because
it is being used by another process:
'C:\GIS_Projects\LlanoRiver_FloodAnalysis\Outputs\Buildings_Final_150.gpkg'
```

**Cause:** A GeoPackage file that is currently loaded as a layer in QGIS is locked by the QGIS process. The script cannot delete or overwrite a locked file even if it calls `delete_if_exists()` first.

**Fix 1:** The `removeAllMapLayers()` call at the top of the script should release these locks. If the error occurs despite this call, the locks were not released fast enough before the script reached the delete step.

**Fix 2:** Close QGIS entirely, then reopen and rerun the script without loading any layers manually before running. This is the most reliable fix when Fix 1 does not work.

**Fix 3:** If closing QGIS is not practical, wait approximately 30 seconds after the `removeAllMapLayers()` call before the script attempts any file operations. A brief sleep can be inserted into the script if this is a recurring issue.

---

## GDAL Warp Hang

**Symptom:** The script reaches Step 3 (DEM reprojection via GDAL Warp) and appears to hang indefinitely. The QGIS Python Console shows no progress and the spinning cursor continues for 15 minutes or longer.

**Cause:** On QGIS 3.44, the GDAL Warp tool occasionally completes the actual reprojection computation successfully but freezes when attempting to report back to the QGIS processing framework. The output file is written to disk correctly, but the script receives no completion signal and does not advance.

**Diagnosis:** Without cancelling the script, open Windows File Explorer and navigate to the Processed folder. If `LlanoDEM_UTM14N.tif` exists and has a file size of approximately 456,031 KB, the reprojection completed. The QGIS interface is simply frozen on the reporting step.

**Fix:** Cancel the script run. Modify the Step 3 block to skip reprojection if the output file already exists:

```python
print("\n--- Step 3 of 7: Reprojecting DEM to EPSG:26914 ---")
if os.path.exists(DEM_UTM):
    print(f"  DEM already exists, skipping reproject: {DEM_UTM}")
else:
    processing.run("gdal:warpreproject", { ... })
    print(f"  DEM reprojected: {DEM_UTM}")
```

This skip pattern is safe because the DEM reprojection is deterministic. The same input DEM with the same target CRS will always produce the same output. Once the file exists and has been verified, there is no reason to regenerate it.

---

## Windows File Extension Visibility

Windows hides known file extensions by default. This causes two specific problems in this workflow:

**Problem 1: The USGS DEM double extension is invisible.** The file is stored as `LlanoDEM_raw.tif.tif` on disk. In File Explorer it appears as `LlanoDEM_raw.tif` because Windows hides the last `.tif`. Script paths must reference the actual filename including both extensions. To see the true filename, open Command Prompt, navigate to the folder, and run `dir`.

**Problem 2: GRASS output rasters appear to have no extension.** After r.slope.aspect and r.watershed run, the output files appear in File Explorer without any extension because Windows hides the `.tif`. This led to a debugging session where the GRASS tools appeared to have failed to write GeoTIFF files. They had written the files correctly. Verify by checking the Type column in File Explorer, which reads TIF File regardless of whether the filename displays the extension.

**To permanently enable file extension display in Windows File Explorer:** Open any File Explorer window, click the View menu, click Show, and click File name extensions.

---

## Session Boundaries and Saved Parameters

**Script files survive session boundaries. Calibrated parameters do not.**

The PyQGIS script committed to GitHub is permanent and will run correctly on a fresh QGIS session. The flow accumulation threshold value that produces a specific structure count is not permanent. It depends on the specific raster generation run and must be recalibrated if the terrain rasters are regenerated.

In this project, the flow accumulation threshold changed from 15.88 (first validated run) to 90 (after raster regeneration) for the same study area. Both values were correct for their respective raster generation runs. The change was caused by differences in the GRASS r.watershed computation across runs, which is expected MFD behavior.

**Always document the threshold value used in a given run alongside the structure count.** The script README and the session summary documents both contain this reference. Before reusing a threshold value from a previous session, verify that the terrain rasters have not been regenerated since that threshold was validated.

---

## See Also

- [GRASS Tool Reference](grass-tool-reference.md): flow accumulation threshold calibration procedure
- [CRS & Projection: Detailed Reference](crs-and-projection.md): how stale files caused the embedded CRS problem
- [Data Sourcing: Detailed Reference](data-sourcing.md): RawData folder organization

[← Back to Main Guide](../QGIS_Master_Workflow_README.md) | [↑ Back to top](#script-behavior--stale-data)
