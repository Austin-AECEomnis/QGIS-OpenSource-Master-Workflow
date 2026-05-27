# CRS & Projection: Detailed Reference

Coordinate reference system issues are the single largest source of wasted time in this workflow. This file documents every CRS problem encountered across the Guadalupe and Llano River projects, the diagnostic process for identifying them, and the verified fixes.

[← Back to Main Guide](../QGIS_Master_Workflow_README.md)

---

## Contents

- [The Core Problem](#the-core-problem)
- [CRS Decision Framework for This Workflow](#crs-decision-framework-for-this-workflow)
- [Assigned CRS vs Embedded CRS](#assigned-crs-vs-embedded-crs)
- [The Zoom-to-Layer Diagnostic](#the-zoom-to-layer-diagnostic)
- [Reprojection Sequence for This Workflow](#reprojection-sequence-for-this-workflow)
- [CRS Issues Encountered in This Project](#crs-issues-encountered-in-this-project)
- [See Also](#see-also)

---

## The Core Problem

Every data source used in this workflow arrives in a different coordinate reference system. None of them match each other out of the box, and none of them are suitable for terrain analysis without reprojection.

| Dataset | Native CRS | Units | Suitable for terrain analysis |
|---------|-----------|-------|-------------------------------|
| USGS 3DEP DEM | EPSG:4269 NAD83 | Decimal degrees | No |
| FEMA NFHL flood zones | EPSG:4269 NAD83 | Decimal degrees | No |
| OSM building footprints | EPSG:4326 WGS84 | Decimal degrees | No |
| ArcGIS Online exports | EPSG:3857 Web Mercator | Meters (distorted) | No |
| Target for analysis | EPSG:26914 UTM Zone 14N | Meters | Yes |

The fundamental issue with geographic CRS units (decimal degrees) in terrain analysis is that horizontal distance and vertical elevation are in incompatible units. A slope calculation on a geographic raster compares elevation change in meters against horizontal distance in degrees. The result is a number that has no meaningful physical interpretation. QGIS and GRASS will run the tools without error, but the values produced are wrong.

**EPSG:26914 NAD83 / UTM Zone 14N** is the correct projected CRS for central Texas. It uses meters for all linear measurements, covers the study area without significant distortion, and is the target CRS for every reprojection step in this workflow.

---

## CRS Decision Framework for This Workflow

Before reprojecting anything, answer these two questions:

**Question 1: What is your analysis target CRS?**
For central Texas terrain analysis: EPSG:26914. Every dataset must be reprojected to this CRS before any terrain tool runs.

**Question 2: What is your publishing destination?**
For GitHub Pages via qgis2web: UTM works fine. Leaflet reprojects to display coordinates automatically.
For ArcGIS Online: the entire project must ultimately be in Web Mercator (EPSG:3857). See the ESRI Master Workflow repository for that path.

These two questions have different answers. Your analysis CRS and your publishing CRS do not have to match. Reproject for analysis first, then let the publishing tool handle display reprojection.

---

## Assigned CRS vs Embedded CRS

This is the most dangerous CRS problem in the workflow because it is silent. A layer can display the correct CRS in its Properties dialog while its coordinates are physically stored in an entirely different system.

**Assigned CRS:** The label QGIS applies to the layer. Visible in Layer Properties under the Source tab. Can be changed without touching the underlying data.

**Embedded CRS:** The coordinate system the data was actually written in. Determined by the actual coordinate values stored in the file.

When these two do not match, QGIS will display the layer in the wrong location. The layer Properties will show the correct CRS, the project CRS will be correct, and all reprojection dialogs will offer what appears to be the right transformation, but the layer will plot in the wrong place on the canvas.

**How this happened in this project:** During script debugging sessions, earlier intermediate building centroid files with incorrect coordinates accumulated in the Outputs folder. When the script ran again, it picked up these corrupted files rather than rebuilding from the raw OSM source. The resulting Buildings_Final_150 layer reported EPSG:26914 in Properties but plotted approximately 80 kilometers south of Llano city in the Johnson City and Blanco area.

**The key distinction:** A layer that reports the wrong location on the canvas despite showing the correct CRS in Properties has an embedded CRS problem. Reassigning the CRS label will not fix it. Reprojecting from the corrupted layer will not fix it. The only fix is to rebuild the layer from the raw source data.

---

## The Zoom-to-Layer Diagnostic

When layers that should overlap do not overlap, or when a layer appears in an unexpected location, use this diagnostic before attempting any fix:

**Step 1:** Right-click each layer in the Layers panel and select Zoom to Layer. Note where each layer zooms to on the canvas.

**Step 2:** Compare the zoom locations. Layers in the same CRS with correct coordinates will zoom to the same geographic area. A layer that zooms to a completely different location than expected has an embedded CRS problem.

**Step 3:** If a layer zooms to the right region but does not align precisely with other layers, it may have a legitimate CRS mismatch that reprojection can fix. Run Reproject Layer with the correct target CRS and test again.

**Step 4:** If a layer zooms to the correct location after reprojection but still does not align, check whether other layers in the project have been reprojected correctly. In the Guadalupe River QGIS project, layers exported from ArcGIS Online arrived in EPSG:3857 (Web Mercator) and required reprojection to EPSG:26914 before they would align with other layers.

**Step 5:** If a layer zooms to the wrong location entirely (wrong state, wrong country), the coordinates themselves are wrong. The only fix is to rebuild from the raw source.

---

## Reprojection Sequence for This Workflow

Reproject in this order. Each step depends on the previous one completing successfully.

**1. Reproject FEMA flood zone polygons**

Processing Toolbox: Vector general, Reproject layer
- Input: S_FLD_HAZ_AR.shp from FEMA package
- Target CRS: EPSG:26914
- Output: save to Outputs folder as a GeoPackage

Run this for each county in your study area. Confirm each output lands correctly by zooming to layer after loading.

**2. Reproject the DEM**

Processing Toolbox: GDAL, Raster projections, Warp (reproject)
- Input: raw USGS DEM
- Source CRS: EPSG:4269 (set explicitly, do not leave blank)
- Target CRS: EPSG:26914
- Resampling method: Bilinear
- Output: save to Processed folder

Bilinear resampling is appropriate for continuous elevation data. Do not use nearest neighbor for a DEM.

**3. Fix OSM geometries, convert to centroids, then reproject**

The order here matters. Fix geometries before converting to centroids, not after. Invalid polygon geometries produce incorrect centroid locations. Reprojection after centroid conversion is correct.

Processing Toolbox: Vector geometry, Fix geometries
- Input: OSM GeoJSON
- Output: TEMPORARY_OUTPUT or Outputs folder

Processing Toolbox: Vector geometry, Centroids
- Input: fixed geometries layer
- Output: Outputs folder

Processing Toolbox: Vector general, Reproject layer
- Input: centroids layer
- Target CRS: EPSG:26914
- Output: Outputs folder

**4. Confirm all layers are in the same CRS before running terrain tools**

Load all reprojected layers into a QGIS project set to EPSG:26914. Right-click each layer, check Properties, confirm Source CRS reads EPSG:26914 for all of them. Then zoom to each layer individually and confirm they all land in the correct geographic area before proceeding.

---

## CRS Issues Encountered in This Project

**Issue 1: All layers from ArcGIS Online export were in EPSG:3857**

When the Guadalupe River analysis layers were exported from ArcGIS Online for use in QGIS, they arrived in Web Mercator (EPSG:3857). This was expected behavior for ArcGIS Online exports but required reprojecting all layers to EPSG:26914 before any QGIS terrain analysis could begin.

**Issue 2: QGIS transformation dialog offered only Web Mercator pipelines**

When reprojecting layers from EPSG:3857 to EPSG:26914, the QGIS transformation selection dialog offered only options that routed through Web Mercator as an intermediate step. This appeared incorrect but was harmless for the analysis. The displayed transformation pipeline description was a QGIS internal routing choice, not an indication that the output would be in Web Mercator. The output was correctly projected to EPSG:26914 regardless of the pipeline description.

**Issue 3: Buildings layer reported correct CRS but plotted in wrong location**

Described in detail in the Assigned CRS vs Embedded CRS section above. Root cause was corrupted intermediate files in the Outputs folder from earlier debugging sessions. Fix was to delete all building-chain intermediate files and rebuild from the raw OSM GeoJSON source. The terrain rasters (FlowDir, FlowAcc, Slope) were preserved since they were correct and computationally expensive to regenerate.

**Issue 4: CRS selector in QGIS processing tools uses a globe icon, not a text field**

When setting the CRS in a QGIS processing tool dialog, the field is a button with a globe icon, not a text input. Click the globe icon to open the CRS selector dialog, then search for EPSG:26914. Setting the project CRS first (Project menu, Properties, CRS tab) makes it available as Project CRS in all tool dialogs, which is faster than searching each time.

---

## See Also

- [Data Sourcing: Detailed Reference](data-sourcing.md): native CRS of each data source
- [GRASS Tool Reference](grass-tool-reference.md): why UTM is required before running GRASS terrain tools
- [Script Behavior & Stale Data](script-behavior-and-stale-data.md): how corrupted intermediate files caused the embedded CRS problem

[← Back to Main Guide](../QGIS_Master_Workflow_README.md) | [↑ Back to top](#crs--projection-detailed-reference)
