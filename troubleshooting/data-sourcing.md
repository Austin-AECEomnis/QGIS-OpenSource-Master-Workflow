# Data Sourcing: Detailed Reference

This file covers the complete data sourcing workflow for a QGIS flood terrain analysis built from raw federal and open data sources. All three datasets used in this project are freely available with no license or account required.

[← Back to Main Guide](../QGIS_Master_Workflow_README.md)

---

## Contents

- [USGS 3DEP DEM](#usgs-3dep-dem)
- [FEMA National Flood Hazard Layer](#fema-national-flood-hazard-layer)
- [OSM Building Footprints via Overpass Turbo](#osm-building-footprints-via-overpass-turbo)
- [Data Organization After Download](#data-organization-after-download)
- [See Also](#see-also)

---

## USGS 3DEP DEM

**Source:** [USGS National Map Downloader](https://apps.nationalmap.gov/downloader)

**Product:** 3DEP 1/3 arc-second (approximately 10 meter resolution) elevation data as GeoTIFF

### Download steps

1. Go to apps.nationalmap.gov/downloader
2. Navigate to your study area using the map interface
3. In the Data section select Elevation Products (3DEP)
4. Select 1/3 arc-second DEM as the subcategory
5. Draw a bounding box over your study area or use a predefined boundary
6. Download the GeoTIFF tile covering your area

### Known issues

**Double file extension:** USGS downloads sometimes produce a file named with a double `.tif.tif` extension. This is correct behavior from the USGS download system and should not be corrected. Windows Explorer hides known file extensions by default, so the file may appear to have no extension at all in Explorer. To verify the actual filename, open a Command Prompt, navigate to the folder, and run `dir` to see the true filename including all extensions. Script file paths must reference the double extension exactly as the file is stored on disk.

Example: the Llano River project DEM was stored as `LlanoDEM_raw.tif.tif` and all script paths referenced it as such.

**Native CRS is geographic NAD83 (EPSG:4269):** The raw DEM arrives in geographic coordinates, meaning units are decimal degrees, not meters. Running any terrain tool on this file without first reprojecting to a projected metric CRS will produce unreliable slope and flow accumulation values. Always reproject to EPSG:26914 (UTM Zone 14N, NAD83) before running GRASS terrain tools. Save the reprojected DEM to your Processed folder.

**File size reference:** The two-county Llano and Burnet DEM reprojected to EPSG:26914 produced an output of 456,031 KB. If a reprojection appears to hang but a file of approximately this size exists in the Processed folder, the reprojection completed successfully. See [Script Behavior & Stale Data](script-behavior-and-stale-data.md) for the GDAL Warp hang issue.

---

## FEMA National Flood Hazard Layer

**Source:** [FEMA Map Service Center](https://msc.fema.gov)

**Product:** National Flood Hazard Layer county shapefile packages

### Download steps

1. Go to msc.fema.gov
2. Search by county name or navigate the map to your study area
3. Select the NFHL Data download option for your county
4. Download the ZIP package for each county in your study area

### Package structure

Each county ZIP package contains multiple shapefiles covering different flood-related features. The flood zone polygon layer is always named `S_FLD_HAZ_AR.shp`. This is the layer used for all flood zone analysis and filtering in this workflow.

Other files in the package (roads, cross sections, base flood elevations) are not required for terrain analysis and can be ignored.

### Known issues

**Windows folder naming with retained .zip suffix:** When Windows extracts a FEMA NFHL ZIP package, it sometimes creates the extraction folder with the `.zip` suffix retained in the folder name. For example, extracting `NFHL_LlanoCounty.zip` may produce a folder literally named `NFHL_LlanoCounty.zip` rather than `NFHL_LlanoCounty`. This is a Windows behavior, not a FEMA issue.

The consequence is that all script file paths must include the `.zip` suffix in the folder name. In this project the paths were written as:

```
RawData/NFHL_LlanoCounty.zip/S_FLD_HAZ_AR.shp
RawData/NFHL_BurnetCounty.zip/S_FLD_HAZ_AR.shp
```

Before finalizing any script that references FEMA data, confirm the actual folder name in Windows File Explorer and match it exactly in the path.

**Native CRS is geographic NAD83 (EPSG:4269):** Same as the USGS DEM. Reproject to EPSG:26914 before using flood zone polygons as clip boundaries or analysis inputs.

**Flood zone categories relevant to this analysis:**

| Zone | Designation | Notes |
|------|------------|-------|
| AE | 1% annual chance, base flood elevations determined | Primary high-risk zone |
| A | 1% annual chance, no base flood elevations | High risk, less precise boundary |
| AO | 1% annual chance, sheet flow | Shallow flooding, common in flat terrain |
| X | 0.2% annual chance or outside flood boundary | Moderate to minimal risk per FEMA |

The X zone classification gap is a key finding in this project. Structures identified by terrain analysis as high-risk that fall within FEMA X zones represent locations where the official flood designation does not reflect actual terrain-based exposure. This gap was confirmed in both the Guadalupe River corridor (Hunt VFD, Repository 4) and the Llano River corridor (Llano city, Repository 5).

---

## OSM Building Footprints via Overpass Turbo

**Source:** [Overpass Turbo](https://overpass-turbo.eu)

**Product:** OpenStreetMap building polygon GeoJSON

### Why OSM for building footprints

No single free federal dataset provides complete building footprint coverage for rural Texas. OSM is the most accessible option and provides adequate coverage in incorporated areas. Coverage in rural sections between towns is sparse and will undercount structures. The final structure count from any OSM-based analysis reflects the OSM dataset, not the true number of flood-vulnerable structures on the ground.

### Overpass Turbo query

Use this query structure, replacing the bounding box coordinates with values appropriate for your study area:

```
[out:json][timeout:25];
(
  node["building"](MIN_LAT,MIN_LON,MAX_LAT,MAX_LON);
  way["building"](MIN_LAT,MIN_LON,MAX_LAT,MAX_LON);
  relation["building"](MIN_LAT,MIN_LON,MAX_LAT,MAX_LON);
);
out body;
>;
out skel qt;
```

**Validated bounding box for Llano and Burnet Counties:**

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

### Known issues

**Bounding box verification is mandatory before export:** The first bounding box attempted for this project was geographically incorrect and returned over 64,000 features located entirely in the Johnson City and Blanco area, entirely missing Llano. The coordinates appeared reasonable but were off by several degrees. Always run the query first and visually confirm on the map that results appear in the correct location before exporting. Do not export based on feature count alone.

**Large export behavior:** If the query returns more than approximately 60,000 features, Overpass Turbo displays a warning before attempting to render them. Click Continue to proceed, then immediately click the Export button before the browser attempts to render the full dataset on the map. Rendering 60MB or more of features will crash the browser tab. Export as GeoJSON and save to your RawData folder.

**Duplicate file from repeated exports:** If you export more than once, Overpass Turbo may save a second file with a ` (2)` suffix in the name alongside the original. The script references the original filename. The duplicate can be ignored but should not replace the original if the original is the validated export.

**OSM geometry errors:** OSM building polygons frequently contain invalid geometries including self-intersections. The PyQGIS script includes a `native:fixgeometries` pass before the centroid conversion step specifically to handle this. If you load the raw OSM GeoJSON directly into QGIS without running fixgeometries first, some centroid calculations will fail silently on the invalid polygons.

**Native CRS is geographic WGS84 (EPSG:4326):** Reproject to EPSG:26914 after converting polygons to centroids.

---

## Data Organization After Download

Once all three datasets are downloaded, organize them in the RawData folder before opening QGIS or running any scripts:

```
YourProject/
└── RawData/
    ├── LlanoDEM_raw.tif.tif          (USGS DEM, double extension intentional)
    ├── NFHL_LlanoCounty.zip/         (folder named with .zip suffix)
    │   └── S_FLD_HAZ_AR.shp          (plus associated .dbf, .shx, .prj files)
    ├── NFHL_BurnetCounty.zip/        (second county if multi-county study area)
    │   └── S_FLD_HAZ_AR.shp
    └── OSM_Buildings_Llano.geojson   (Overpass Turbo export)
```

Confirm this structure matches your actual files before writing or running any scripts. Every path in the PyQGIS script references this structure exactly. A mismatch between the actual folder names and the script paths is the most common cause of script failure on the first run.

---

## See Also

- [CRS & Projection: Detailed Reference](crs-and-projection.md): reprojection steps for all three data sources
- [Script Behavior & Stale Data](script-behavior-and-stale-data.md): what happens when old intermediate files remain in the output folder
- [GRASS Tool Reference](grass-tool-reference.md): terrain analysis tools that consume these datasets

[← Back to Main Guide](../QGIS_Master_Workflow_README.md) | [↑ Back to top](#data-sourcing-detailed-reference)
