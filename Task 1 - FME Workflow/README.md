# Task 1 - FME Tile Map Service (TMS) Generation

Here, I describe how to fill in each parameter in the FME Workbench Reader/Writer Variables panel for generating Tile Map Services from satellite imagery. I also provide an overview of the two main workflow blocks: Raster Prep and Tiling & Output (AOI Prep).

---

## 1. Reader/Writer Variables

### 1.1 Source GeoTIFF*

**What to enter:** The full file path to input satellite image (e.g., `C:\Data\satellite_scene.tif`).

**Assumptions and notes:**

- The file must be a georeferenced GeoTIFF with an embedded CRS. The Raster Prep block's "Convert to Web Mercator" reprojector depends on this.
- If the imagery is delivered as multiple files (e.g., one per spectral band), it would be ideal to composite or mosaic them beforehand, or point to the already-composited file.

---

### 1.2 Is the raster already RGB? (Recommended!)*

**What to enter:** Leave this checkbox **unchecked**.

**Rationale:** The Raster Prep workflow explicitly re-sorts bands (Red = band 2, Green = band 1, Blue = band 0), which indicates the source is not already in standard RGB order (it is likely in a non-standard band arrangement). Leaving this unchecked tells the workflow to run the band-reordering logic (visible in the Tester_3 branching path in the Raster Prep block).

**Trade-off:** If I had a standard 3-band RGB GeoTIFF that already matched the expected band order, I could check this box to skip the reordering step and save processing time. In this case, however, band reordering is required.

---

### 1.3 Team AOI Filter Type for clipping (Name or OID)*

**What to enter:** Either `Name` or `OID`, depending on how I want to identify the AOI polygon in the database.

**How this works:** The AOI Prep block in the Tiling & Output section has two branching paths — one for "Name / version" and one for "OID." The Tester and Tester_4 nodes route features down one path or the other based on this value.

**Trade-offs:**

- **Name** — Filter by a human-readable team/project name and version number. This is more intuitive but requires knowing the exact string stored in the database.
- **OID** — Filter by the database ObjectID. This is more precise and less error-prone if we know the specific record ID.

---

### 1.4 Team AOI Name (if Name Filter)

**What to enter:** The name string of AOI as stored in the database (e.g., `"ProjectABC"`).

**When to fill in:** Only when the filter type in Section 1.3 is set to `Name`. Leave blank if using OID.

---

### 1.5 Team AOI Version # (if Name Filter)

**What to enter:** The version number associated with the named AOI (e.g., `1`).

**When to fill in:** Only when the filter type in Section 1.3 is set to `Name`. Leave blank if using OID.

**Assumption:** The database may store multiple versions of the same named AOI. The version number disambiguates which geometry to use.

---

### 1.6 Team AOI ObjectID (if OID Filter)

**What to enter:** The database ObjectID of the desired AOI record (e.g., `42`).

**When to fill in:** Only when the filter type in Section 1.3 is set to `OID`. Leave blank if using Name.

---

### 1.7 Destination TMS Directory*

**What to enter:** The output folder path where the tile pyramid will be written (e.g., `C:\TMS_Output\project_alpha`).

**Assumptions and notes:**

- The workflow will create the standard `z/x/y.png` folder structure inside this directory.
- Ensure the path exists or that FME has permission to create it.
- **Disk space warning:** Tile pyramids from satellite imagery can be very large, especially at high maximum zoom levels. Each additional zoom level roughly quadruples the number of tiles.

---

### 1.8 TMS Minimum Zoom Level (normally 10)*

**What to enter:** An integer representing the most zoomed-out level of tiles to generate. The recommended default is **10**.

**Trade-offs:**

| Min Zoom | Behavior |
|----------|----------|
| Lower (e.g., 8) | Generates more zoomed-out overview tiles. Useful if the AOI is large or we want smooth zooming from a regional scale. Adds relatively few extra tiles. |
| Higher (e.g., 12) | Skips the overview levels, saving space and processing time, but the tile layer will not appear until the user zooms in fairly close. |

**Recommendation:** Use **10** as a safe default unless the use case requires otherwise.

---

### 1.9 TMS Maximum Zoom Level (normally 19, 20 if a small area)*

**What to enter:** An integer representing the most zoomed-in level of tiles to generate. This is the single most impactful parameter for both processing time and storage consumption.

**Trade-offs:**

| Max Zoom | Approximate Ground Resolution | Use Case |
|----------|-------------------------------|----------|
| 17 | ~1.2 m/pixel | Coarse overview; fast to generate |
| 18 | ~0.6 m/pixel | Good for medium-resolution imagery |
| 19 | ~0.3 m/pixel | Standard for high-resolution satellite data |
| 20 | ~0.15 m/pixel | Very high detail; practical only for small AOIs |

**Decision guidance:**

- Each additional zoom level roughly quadruples the number of tiles generated.
- If the AOI is small (e.g., a neighborhood or single facility), **20** is feasible.
- For larger regions, **19** is more practical.
- There is no benefit to setting the max zoom beyond what the source imagery's native resolution can actually resolve.

---

### 1.10 Bands labeled as "RGB" or "123"?

**What to enter:** Either `RGB` or `123`, depending on how the bands are named internally in the GeoTIFF.

**How to check:** Open the file in FME's Data Inspector or run `gdalinfo <filename>` in a terminal. Many satellite products label bands numerically (`1, 2, 3`), while some pre-processed composites label them by color (`R, G, B`).

**Impact:** This value affects how the band-reordering logic (Red = band 2, Green = band 1, Blue = band 0) references bands internally within the workflow.

---

### 1.11 Apply Gamma/Brightness Correction?*

**What to enter:** **Check this box (enabled).**

**Rationale:** The workflow requirements call for both gamma correction and brightness adjustment. The Raster Prep block contains dedicated "Brightness" and "Corrections" stages that depend on this flag being enabled. Leaving it unchecked would skip those transformers entirely.

**Additional note:** This flag also ensures the full Raster Prep pipeline runs, including the "Remove Black Edges" stage at the end of the block.

---

### 1.12 Red (or all) band(s) gamma*

**What to enter:** A numeric gamma correction value. A value of **1.0** means no change.

**Trade-offs:**

| Gamma Value | Effect |
|-------------|--------|
| < 1.0 (e.g., 0.7–0.9) | Darkens midtones. Useful if the image looks washed out or overexposed. |
| 1.0 | No correction. |
| > 1.0 (e.g., 1.2–1.5) | Brightens midtones. Useful for dark satellite scenes (e.g., winter imagery, shadowed areas). |

**Per-band vs. all-bands:**

- The label says "Red (or all)," suggesting either applying one gamma value uniformly to all bands or tune each band independently (if additional per-band fields are available).
- Applying the same value to all bands preserves color balance.
- Adjusting per-band allows compensation for atmospheric effects (e.g., boosting the blue band to reduce haze).

**Recommendation:** Start with **1.0** on a test run, visually inspect the output tiles, and then adjust incrementally. Satellite imagery varies significantly by sensor, time of day, and atmospheric conditions, so there is no universal "correct" value.

---

## 2. Raster Prep Block

The Raster Prep block prepares the raw satellite imagery for tiling. The workflow proceeds through the following stages from left to right:

### 2.1 Read Original Imagery

The source GeoTIFF (from Section 1.1) is read into the workflow.

### 2.2 Convert to Web Mercator

The raster is reprojected from its native coordinate system to **EPSG:3857 (Web Mercator)**, which is the standard projection for web-based tile map services.

### 2.3 Band Sorting / RGB Check (Tester_3)

A conditional test (Tester_3) checks whether band reordering is needed (controlled by the "Is the raster already RGB?" flag from Section 1.2). If the raster is not already RGB, bands are re-sorted to: Red = band 2, Green = band 1, Blue = band 0.

### 2.4 Corrections and Brightness (Tester_5)

A second conditional test (Tester_5) routes the raster through gamma correction and brightness adjustment stages if the "Apply Gamma/Brightness Correction?" flag (Section 1.11) is enabled. Gamma and brightness values are applied according to the parameter from Section 1.12.

### 2.5 Remove Black Edges

After reprojection, satellite imagery often contains black (0,0,0) pixel borders where the rotated or warped image was padded to fill a rectangular grid. This stage identifies and removes those borders — typically by setting them to transparent or clipping them away — so they don't appear as black tiles in the final TMS output. This step runs automatically as part of the pipeline when gamma/brightness correction is enabled.

### 2.6 PNG Compatible Format (RasterInterpolationCoercer)

The raster is coerced into a PNG-compatible format (8-bit RGBA) in preparation for tile output. This is the final stage before the data is handed off to the Tiling & Output block.

---

## 3. Tiling & Output Block (AOI Prep)

The Tiling & Output block clips the prepared raster to the AOI and generates the final tile pyramid.

### 3.1 AOI Selection

The AOI polygon is read from the database (`eo_owner` table). Based on the filter type (Section 1.3), the workflow routes through either:

- **Name / version path** — Tester filters by team name and version number.
- **OID path** — Tester_4 and Tester_2 filter by ObjectID.

### 3.2 Reproject AOI to Web Mercator (Reprojector_2)

The selected AOI polygon is reprojected to EPSG:3857 to match the raster's coordinate system.

### 3.3 Tiling Process

The prepared raster is intersected with the AOI and sliced into a pyramid of 256×256 pixel PNG tiles, organized by zoom level (`z`), column (`x`), and row (`y`). Tiles are generated for every zoom level between the minimum (Section 1.8) and maximum (Section 1.9) values.

### 3.4 Output via StringConcatenator and Fanout

The StringConcatenator builds the output file path for each tile, and the writer uses a **Fanout Expression** to save individual PNG tiles into the standard TMS directory structure (`z/x/y.png`) under the destination directory (Section 1.7).

---

## 4. Quick-Reference Summary

| Parameter | Suggested Value | Notes |
|---|---|---|
| Source GeoTIFF | `C:\Data\scene.tif` | Must be georeferenced |
| Is raster already RGB? | ☐ Unchecked | Bands need reordering |
| AOI Filter Type | `Name` or `OID` | Choose based on identification method |
| AOI Name / Version / OID | *(depends on filter type)* | Fill only the relevant fields |
| Destination TMS Directory | `C:\TMS_Output\tiles` | Ensure adequate disk space |
| TMS Min Zoom | `10` | Standard default |
| TMS Max Zoom | `19` (or `20` for small AOI) | Biggest impact on output size and processing time |
| Bands labeled as | `123` or `RGB` | Verify with `gdalinfo` or FME Data Inspector |
| Apply Gamma/Brightness | ☑ Checked | Required per workflow spec |
| Gamma | `1.0` (adjust after test) | Start neutral; tune based on visual inspection |

---

Transparency note: Please note that I used Claude Opus 4.6 to help guide this workflow.
