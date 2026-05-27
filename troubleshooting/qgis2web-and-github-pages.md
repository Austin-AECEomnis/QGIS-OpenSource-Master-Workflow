# qgis2web & GitHub Pages: Detailed Reference

Complete configuration guide for exporting a QGIS project to a Leaflet web map using qgis2web and publishing it to GitHub Pages. Covers plugin setup, export settings, file size management, GitHub upload sequencing, and the index.html title bar edit.

[← Back to Main Guide](../QGIS_Master_Workflow_README.md)

---

## Contents

- [Installing qgis2web](#installing-qgis2web)
- [Preparing Your Project Before Export](#preparing-your-project-before-export)
- [Configuring the Export Dialog](#configuring-the-export-dialog)
- [File Size Management](#file-size-management)
- [Uploading to GitHub](#uploading-to-github)
- [Enabling GitHub Pages](#enabling-github-pages)
- [Adding a Title Bar to index.html](#adding-a-title-bar-to-indexhtml)
- [Troubleshooting](#troubleshooting)
- [See Also](#see-also)

---

## Installing qgis2web

qgis2web is a plugin, not a built-in QGIS feature. It must be installed before the Web menu appears in QGIS.

1. Go to **Plugins, Manage and Install Plugins**
2. In the search bar type qgis2web
3. Select it in the results list and click Install Plugin
4. Once installed, a **Web** menu appears in the QGIS top menu bar
5. Access the export dialog via **Web, qgis2web, Create web map**

---

## Preparing Your Project Before Export

qgis2web exports whatever is currently loaded, visible, and styled in the QGIS project. Prepare the project completely before opening the export dialog. Changes made after export require a full re-export.

### Layer naming

The layer name displayed in the web map legend comes from the layer name in the QGIS Layers panel, not from the file name on disk. Rename layers in the Layers panel before exporting. In this project, `Buildings_Final_150` was renamed to `Flood Risk Structures` before export.

To rename: slow double-click on the layer name in the Layers panel and type the new name.

### Flood zone symbology

For the flood zone polygon layers, use Categorized symbology on the FLD_ZONE field. The categories and suggested styling used in this project:

| Zone | Fill color | Opacity | Notes |
|------|-----------|---------|-------|
| AE | Blue | 50% | Primary high-risk zone |
| A | Light blue | 50% | High risk, less precise |
| AO | Teal | 50% | Sheet flow zone |
| X | Yellow/orange | 30% | Moderate risk, FEMA gap zone |

### Basemap

Load an OpenStreetMap basemap from the Browser panel under XYZ Tiles before exporting. This provides geographic context in the web map without requiring any additional configuration.

### Layer and popup field preparation

In the qgis2web Layers and Groups tab, configure popup fields for each layer before exporting. Popup fields set to Hidden will not appear when a user clicks a feature. Recommended configuration:

**Flood Risk Structures layer:**
- elev_m: Inline visible with data
- slope_deg: Inline visible with data
- flow_acc: Inline visible with data
- All other OSM attribute fields: Hidden

**Flood zone layers:**
- FLD_ZONE: Inline visible with data
- All other fields: Hidden

**OpenStreetMap basemap:** No popup configuration needed.

---

## Configuring the Export Dialog

The qgis2web dialog has four tabs that matter: Layers and Groups, Appearance, Export, and the export format selector.

### Layers and Groups tab

Set popup field visibility as described above. Confirm all layers intended for the web map are checked.

### Appearance tab

| Setting | Value | Notes |
|---------|-------|-------|
| Layers list | Expanded | Shows layer toggle panel by default on load |
| Title | Set a descriptive title | Displays as a header on the map if the template supports it |
| Add layers list | Checked | Enables the layer visibility toggle panel |

### Export tab

| Setting | Value | Notes |
|---------|-------|-------|
| Export to folder | Point to a webmap folder inside your project | Create the folder first if it does not exist |
| Minify GeoJSON | Checked | Reduces file size |
| Precision | 3 | See File Size Management below |

### Export format

Select **Leaflet** from the mapping library dropdown. Leaflet is simpler and more broadly supported than OpenLayers for a GitHub Pages deployment. The output is a self-contained folder of HTML, CSS, JavaScript, and GeoJSON files that requires no server-side processing.

---

## File Size Management

GeoJSON file size is the primary constraint for GitHub Pages deployment. GitHub enforces a 25MB file size limit per file on upload, and a per-commit size limit that applies to the total upload batch.

### Precision reduction

At the default full precision setting, the GeoJSON files for a two-county analysis can total 180MB or more. This is far over GitHub's limits. Reducing the Precision setting in the Export tab dramatically reduces file size with no visible effect on web map display.

| Precision setting | Approximate data folder size | Coordinate accuracy |
|------------------|------------------------------|---------------------|
| Full (maintain) | 180MB+ | Millimeter scale |
| 6 decimal places | ~90MB | Sub-centimeter |
| 5 decimal places | ~45MB | Centimeter scale |
| 3 decimal places | ~22MB | Meter scale, sufficient for web display |

**Use 3 decimal places.** One meter of positional accuracy is more than sufficient for a web map viewed at county scale. The visual difference between full precision and 3 decimal places is not detectable at normal map zoom levels.

If file size is still too large after reducing to 3 decimal places, consider splitting the export into separate uploads by data type (flood zones and structures in separate commits).

---

## Uploading to GitHub

### What to upload

Upload the contents of the webmap export folder, not the folder itself. The export folder contains:

```
css/
data/
images/
js/
legend/
markers/
webfonts/
index.html
```

Navigate into the export folder in File Explorer so you can see these items directly, then select all of them and drag them into the GitHub repository upload area.

### Upload in batches

GitHub has a per-commit size limit. If the total size of all files exceeds this limit, upload in batches:

1. Upload index.html alone and commit
2. Upload the css folder contents and commit
3. Upload the js folder contents and commit
4. Upload the data folder contents and commit (this is usually the largest batch)
5. Upload images, legend, markers, webfonts together and commit

Each commit adds to the repository without overwriting previous commits. All batches stack together correctly. The web map will not render correctly until all files are uploaded.

### Overwriting files after re-export

If you re-export with different settings (for example, after reducing precision), upload the new files using the same process. GitHub automatically overwrites files with the same name and path. No manual deletion is required.

---

## Enabling GitHub Pages

1. In your repository click the **Settings** tab
2. In the left sidebar click **Pages**
3. Under Source select **Deploy from a branch**
4. Set Branch to **main** and folder to **/ (root)**
5. Click **Save**

GitHub takes 1 to 2 minutes to build and deploy the site after saving. Refresh the Settings, Pages page after waiting and the live URL will appear at the top in a green box. The URL format is:

```
https://[username].github.io/[repository-name]/
```

### First load behavior

A blank page on first load after enabling GitHub Pages is normal. GitHub Pages needs 2 to 3 minutes to fully propagate after the first deploy. Hard refresh the page with Ctrl+Shift+R after waiting. If the page still does not load after 5 minutes, check the Actions tab in the repository for a workflow named "pages build and deployment" and confirm it shows a green checkmark.

---

## Adding a Title Bar to index.html

The default qgis2web export does not include a title bar. Adding one requires a direct edit to index.html. This can be done in the GitHub web editor without any local text editor.

### Edit in GitHub

1. In your repository navigate to index.html
2. Click the pencil icon (Edit this file) in the top right of the file view
3. Use Ctrl+F to find the `<body>` tag
4. Click at the end of the `<body>` line, press Enter to create a new line below it
5. Paste the title div block (see below) on the new line
6. The `<body>` tag itself remains unchanged on its own line

### Title bar HTML

```html
<div style="position:fixed;top:0;left:0;width:100%;background:#1a1a2e;color:white;padding:10px 20px;z-index:9999;font-family:Arial,sans-serif;display:flex;align-items:center;justify-content:space-between;">
  <div>
    <strong style="font-size:16px;">🌊 Llano River Flood Analysis</strong>
    <span style="font-size:12px;margin-left:15px;opacity:0.85;">149 high-risk structures identified via terrain analysis, Llano and Burnet Counties, TX, October 2018 flood event</span>
  </div>
  <a href="https://github.com/Austin-AECEomnis/QGIS-LlanoRiver-FloodAnalysis" target="_blank" style="color:#7ec8e3;font-size:12px;text-decoration:none;">GitHub Repo →</a>
</div>
```

Modify the title text, description, and GitHub URL to match your project. Commit directly to main. GitHub Pages will rebuild and the title bar will appear within 2 to 3 minutes.

### If the map is hidden behind the title bar

The title bar is approximately 45px tall and fixed to the top of the viewport. If the map content starts at the top of the page, the title bar will overlap it. Fix by finding the `#map` CSS rule in index.html and adding `top: 45px;` or `margin-top: 45px;` to push the map down. This edit can also be made directly in the GitHub editor.

---

## Troubleshooting

**SSL error when opening qgis2web dialog**

qgis2web sometimes throws an SSL certificate warning when the dialog first opens. The full error reads something like: Unable to Get Local Issuer Certificate. Click ignore or dismiss the dialog and reopen via Web, qgis2web, Create web map. The error is harmless and does not affect the export. It occurs because qgis2web attempts to check for plugin updates when it opens.

**Map loads but layers are not visible**

Confirm that all layers were visible (checked) in the QGIS Layers panel at the time of export. qgis2web exports the visibility state of layers as they were at export time. Re-export with all desired layers checked.

**Layer names in the web map do not match QGIS**

Layer names in the web map come from the QGIS Layers panel at export time. Rename layers in the panel before exporting. Re-export after renaming.

**GitHub Pages URL shows a blank page after 5 minutes**

Check the Actions tab in the repository. If the pages build and deployment workflow shows a red X, there is a file in the repository that GitHub Pages cannot process. The most common cause is a file named incorrectly or a corrupted upload. Check the workflow log for the specific error.

**index.html has "html" text at the top of the page**

This is a paste corruption that occurs when copying and pasting HTML into the GitHub editor. The word "html" fuses to the first line of the file, appearing as visible text on the page. Fix by editing index.html in GitHub, finding the corrupted first line, deleting the "html" prefix, and committing. Allow 2 to 3 minutes for Pages to rebuild.

---

## See Also

- [Script Behavior & Stale Data](script-behavior-and-stale-data.md): ensuring the layers exported are the correct final outputs
- [CRS & Projection: Detailed Reference](crs-and-projection.md): Leaflet handles display reprojection, so UTM layers export correctly

[← Back to Main Guide](../QGIS_Master_Workflow_README.md) | [↑ Back to top](#qgis2web--github-pages-detailed-reference)
