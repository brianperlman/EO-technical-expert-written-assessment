<img width="400" height="170" alt="image" src="https://github.com/user-attachments/assets/c493f6bd-8bf0-42ac-ab94-250d2bc1d60f" />


# Building Extraction Feature Class Schema

## 1. Assignment Overview

MSF relies on EO data and geospatial analysis to support operational planning, population estimation, camp management, and disaster response. A critical component of this work is building extraction, which is the delineation of building footprints from satellite or aerial imagery using manual digitization, semi-automated, or fully automated (e.g., ML/DL) methods.

This document defines a **standardized feature class schema** for building extraction results intended for **organization-wide adoption** across all MSF OCs. The goal is to ensure that every building footprint dataset produced or commissioned by MSF — regardless of the extraction method, imagery source, geographic context, or project — conforms to a single, interoperable schema. This enables:

- Consistent data sharing across missions, OCs, and partner organizations (e.g., HOT, MapAction, UNOSAT).
- Reliable downstream analysis such as population estimation, damage assessment, and change detection.
- Long-term data stewardship and the ability to integrate historical datasets with new extractions.
- Reproducibility and transparency in analytical workflows.

The schema is designed to be compatible with OGC standards, GeoPackage and Esri Geodatabase formats, and common humanitarian data exchange platforms.

---

## 2. Schema Definition

### 2.1 Geometry

| Property         | Specification                                |
|------------------|----------------------------------------------|
| **Geometry Type** | Polygon (MultiPolygon permitted where necessary) |
| **CRS**          | EPSG:4326 (WGS 84) for storage and exchange  |
| **Topology**     | No overlaps, no self-intersections, no slivers. Minimum vertex spacing ≥ 0.1 m in projected units. |

> **Note:** While EPSG:4326 is the canonical storage CRS for interoperability, area and distance calculations must be performed in an appropriate local projected CRS (e.g., UTM zone). The `area_m2` field stores the projected-CRS-derived value.

---

### 2.2 Attribute Fields

The schema is organized into thematic groups for clarity. All field names use `snake_case` to ensure compatibility across platforms (GeoPackage, Shapefile, PostGIS, Esri FGDB).

#### A. Identification

| Field Name     | Type         | Nullable | Description |
|----------------|--------------|----------|-------------|
| `uid`          | Text (36)    | No       | Universally unique identifier (UUID v4). Primary key. Immutable once assigned. |
| `bld_id`       | Text (20)    | Yes      | Optional human-readable building ID, scoped to a project or site (e.g., `MAL-BAM-00421`). |

#### B. Source Imagery

| Field Name         | Type         | Nullable | Description |
|--------------------|--------------|----------|-------------|
| `img_source`       | Text (100)   | No       | Imagery provider or platform (e.g., `Maxar`, `Airbus Pléiades`, `Planet SkySat`, `drone`). |
| `img_date`         | Date         | No       | Acquisition date of the source image (ISO 8601: `YYYY-MM-DD`). |
| `img_res_m`        | Double       | No       | Ground sample distance (GSD) of the source imagery in metres. |
| `img_catalog_id`   | Text (50)    | Yes      | Catalog or scene ID of the source image tile, for traceability. |

#### C. Extraction Method

| Field Name          | Type         | Nullable | Description |
|---------------------|--------------|----------|-------------|
| `extr_method`       | Text (20)    | No       | Extraction methodology. **Coded domain:** `manual`, `semi_auto`, `automated`. |
| `extr_model`        | Text (100)   | Yes      | Name and version of the model or tool used (e.g., `ramp-v1.2`, `SpaceNet-6-baseline`, `QGIS-manual`). Null for manual digitization without tool assistance. |
| `extr_date`         | Date         | No       | Date the extraction was performed (ISO 8601). |
| `extr_analyst`      | Text (100)   | Yes      | Analyst name or team identifier responsible for the extraction. |
| `extr_confidence`   | Double       | Yes      | Model confidence score, normalized 0.0–1.0. Null for manual extractions. |

#### D. Building Attributes

| Field Name       | Type         | Nullable | Description |
|------------------|--------------|----------|-------------|
| `area_m2`        | Double       | No       | Footprint area in square metres, computed in a local projected CRS. |
| `perimeter_m`    | Double       | Yes      | Footprint perimeter in metres. |
| `bld_type`       | Text (30)    | Yes      | Building type or land use. **Coded domain:** `residential`, `commercial`, `industrial`, `institutional`, `health_facility`, `school`, `religious`, `warehouse`, `tent_temporary`, `unknown`. |
| `bld_material`   | Text (30)    | Yes      | Primary roof material observed. **Coded domain:** `metal_sheet`, `thatch`, `concrete`, `tile`, `tarpaulin`, `plastic_sheeting`, `unknown`. |
| `n_floors`       | Short Int    | Yes      | Estimated number of floors, where discernible (e.g., from shadow analysis or ancillary data). |

#### E. Damage & Condition Assessment

| Field Name        | Type         | Nullable | Description |
|-------------------|--------------|----------|-------------|
| `dmg_level`       | Text (20)    | Yes      | Damage classification. **Coded domain** aligned with UNOSAT/Copernicus EMS grading: `no_damage`, `minor`, `moderate`, `severe`, `destroyed`. Null if no damage assessment was performed. |
| `dmg_date`        | Date         | Yes      | Date of the event or assessment to which the damage classification refers. |
| `dmg_source`      | Text (100)   | Yes      | Source of the damage classification (e.g., `UNOSAT-rapid`, `MSF-field`, `pre_post_comparison`). |

#### F. Location Context

| Field Name    | Type         | Nullable | Description |
|---------------|--------------|----------|-------------|
| `country_iso` | Text (3)     | No       | ISO 3166-1 alpha-3 country code (e.g., `MLI`, `SDN`, `BGD`). |
| `adm1_name`   | Text (100)   | Yes      | Admin level 1 name (province/region), from COD-AB or OCHA boundaries. |
| `adm2_name`   | Text (100)   | Yes      | Admin level 2 name (district/department). |
| `site_name`   | Text (100)   | Yes      | Site or locality name (e.g., camp name, settlement name). |

#### G. Data Management & Lineage

| Field Name       | Type         | Nullable | Description |
|------------------|--------------|----------|-------------|
| `project_code`   | Text (30)    | No       | MSF project or emergency code to which this extraction belongs. |
| `oc_code`        | Text (10)    | Yes      | Operational Centre code (`OCA`, `OCB`, `OCBA`, `OCG`, `OCP`). |
| `val_status`     | Text (15)    | No       | Validation status. **Coded domain:** `unvalidated`, `auto_qa`, `manual_qa`, `field_verified`. Default: `unvalidated`. |
| `val_date`       | Date         | Yes      | Date validation was last performed. |
| `val_analyst`    | Text (100)   | Yes      | Analyst or team who performed the validation. |
| `is_active`      | Short Int    | No       | Whether this feature represents the current state of the building. `1` = active/current, `0` = superseded or historical. Default: `1`. |
| `superseded_by`  | Text (36)    | Yes      | `uid` of the feature that replaced this one (used for temporal versioning). Null if `is_active = 1`. |
| `created_date`   | DateTime     | No       | Timestamp when the record was created in the database. |
| `modified_date`  | DateTime     | No       | Timestamp of the last modification to the record. |
| `remarks`        | Text (500)   | Yes      | Free-text field for analyst notes, caveats, or context. |

---

## 3. Coded Domains Summary

To enforce data quality and consistency, the following fields use controlled vocabularies (coded domains). These should be enforced at the database level via domain constraints or lookup tables.

| Field              | Permitted Values |
|--------------------|-----------------|
| `extr_method`      | `manual`, `semi_auto`, `automated` |
| `bld_type`         | `residential`, `commercial`, `industrial`, `institutional`, `health_facility`, `school`, `religious`, `warehouse`, `tent_temporary`, `unknown` |
| `bld_material`     | `metal_sheet`, `thatch`, `concrete`, `tile`, `tarpaulin`, `plastic_sheeting`, `unknown` |
| `dmg_level`        | `no_damage`, `minor`, `moderate`, `severe`, `destroyed` |
| `val_status`       | `unvalidated`, `auto_qa`, `manual_qa`, `field_verified` |

Domain lists should be maintained centrally and extended through a governed change-request process — not modified ad hoc in the field.

---

## 4. Best Practice Recommendations

### 4.1 Coordinate Reference System

Store all data in **EPSG:4326** for exchange and interoperability. Compute all metric attributes (`area_m2`, `perimeter_m`) in the appropriate **UTM zone** or national projected CRS, and store the resulting values in the attribute table. Document the projected CRS used in the dataset-level metadata.

### 4.2 Unique Identifiers

Use **UUID v4** for the `uid` field. Never reuse or recycle identifiers. UUIDs ensure global uniqueness across missions and time, and prevent collisions when merging datasets from different sources.

### 4.3 Minimum Mapping Unit (MMU)

Define and document a minimum mapping unit per project. A recommended default is **5 m²** for urban settings and **7 m²** for rural/camp settings. Structures smaller than the MMU should be excluded or flagged.

### 4.4 Topology & Quality Assurance

- Run topology checks before any dataset is marked as `manual_qa` or `field_verified`.
- No polygon overlaps between features within the same extraction epoch.
- No self-intersecting geometries.
- Gaps less than 0.5 m between adjacent buildings should be reviewed for potential merge.
- Automated extractions should include a post-processing step to regularize geometry (orthogonalize corners, remove sliver vertices).

### 4.5 Metadata

Every delivered dataset must include an ISO 19115-compliant metadata record (or a simplified humanitarian profile) documenting: the imagery source, extraction method, analyst, date range, spatial extent, CRS, known limitations, and accuracy assessment results.

### 4.6 File Format

**GeoPackage (`.gpkg`)** is the recommended exchange format. It is an open OGC standard, supports long field names, date/datetime types, and does not impose the field-name or row-count limitations of Shapefile. Esri File Geodatabase (`.gdb`) is acceptable for Esri-centric workflows but should not be the sole distribution format.

### 4.7 Accuracy Assessment

For any dataset used in population estimation or operational planning, a quantitative accuracy assessment should be performed and documented. At a minimum, report:

- **Precision** — what proportion of extracted features are true buildings.
- **Recall** — what proportion of actual buildings were captured.
- **F1 Score** — the harmonic mean of precision and recall.
- **IoU (Intersection over Union)** — for geometry quality evaluation.

Accuracy results should be recorded in the dataset metadata and, where applicable, as a companion point feature class of sampled validation sites.

### 4.8 Naming Convention

Dataset filenames should follow a structured pattern:

```
msf_bld_{country_iso}_{site}_{img_date}_{version}.gpkg
```

Example: `msf_bld_sdn_elphasher_20240315_v01.gpkg`

---

## 5. Integrating Historical Building Extraction Results

A primary design consideration for this schema is its ability to **accommodate multi-temporal data** — that is, building extractions derived from imagery captured at different dates for the same geographic area. This is essential for MSF, where operational areas are surveyed repeatedly over months or years for population monitoring, camp growth tracking, and post-disaster damage assessment.

### 5.1 Temporal Versioning Model

When a building changes over time — it gets damaged, expanded, rebuilt, or demolished — the database needs a strategy for handling the existing record. In data warehousing this problem is known as a **slowly changing dimension (SCD)**, and there are three common approaches:

- **Type 1 — Overwrite the record.** The existing row is updated in place. This is simple but destroys all history. You would never know what a building looked like before an airstrike, or that a camp had 200 fewer shelters six months ago. For MSF operational analysis, this is usually unacceptable.
- **Type 2 — Keep both versions.** Instead of overwriting, the old record is marked as historical and a new record is inserted representing the current state. Both rows live in the same table, linked together. History is fully preserved.
- **Type 3 — Add columns for previous values.** Fields like `previous_dmg_level` or `previous_area_m2` are added to the same row. This becomes unwieldy quickly and only supports a single generation of history — insufficient for areas surveyed repeatedly over months or years.

This schema adopts the **Type 2** approach because it preserves the complete lifecycle of every structure without imposing a limit on the number of temporal states, and because it keeps all data in a single feature class rather than requiring separate layers per epoch. It relies on three key fields:

| Field            | Role |
|------------------|------|
| `is_active`      | Flags whether a feature represents the **current known state** (`1`) or a **historical record** (`0`). |
| `superseded_by`  | Links a historical record to the `uid` of the feature that replaced it, creating a traceable lineage chain. |
| `img_date`       | Anchors each feature to a specific moment in time — the date the source imagery was captured. |

When a new extraction is performed for an area that already has existing data:

1. **Unchanged buildings:** The existing feature's `is_active` remains `1`. No new record is created unless attributes have been updated.
2. **Modified buildings** (e.g., expanded, partially destroyed): The existing feature is set to `is_active = 0`, and its `superseded_by` is populated with the `uid` of the new feature. A new feature is inserted with `is_active = 1`.
3. **Demolished or disappeared buildings:** The existing feature is set to `is_active = 0`. No new feature is created. The `remarks` field may note the reason.
4. **New buildings:** A new feature is inserted with `is_active = 1` and no `superseded_by` reference.

#### Worked Example

Consider a building in El Fasher, Sudan, extracted from March 2024 imagery (`uid = A`, `is_active = 1`, `dmg_level = 'no_damage'`). In September 2024, new imagery shows the structure has been severely damaged. The update process is:

1. The **original record** (`uid = A`) is updated: `is_active` → `0`, `superseded_by` → `B`.
2. A **new record** (`uid = B`) is inserted with `is_active = 1`, `img_date = '2024-09-15'`, and `dmg_level = 'severe'`.

Both records occupy the same geographic footprint but represent different moments in time. The `superseded_by` field creates a forward-linked chain — an analyst can walk through the full lifecycle of any structure. Meanwhile, a field coordinator running `WHERE is_active = 1` sees only the current operational picture, without needing to manage multiple files or vintage layers.

### 5.2 Querying Current vs. Historical State

This design allows simple filtering for operational use while preserving the full temporal record:

```sql
-- Current building stock only
SELECT * FROM buildings WHERE is_active = 1;

-- Full history for a specific building
SELECT * FROM buildings
WHERE uid = 'target-uuid'
   OR superseded_by = 'target-uuid'
ORDER BY img_date;

-- Snapshot of buildings as they existed on a specific image date
SELECT * FROM buildings
WHERE img_date <= '2023-06-15'
  AND (superseded_by IS NULL OR modified_date > '2023-06-15');
```

### 5.3 Ingesting Legacy Datasets

Many MSF missions hold historical building footprints that predate this schema — often delivered as Shapefiles with inconsistent or incomplete attributes. To integrate these:

1. **Field mapping:** Create a crosswalk table that maps legacy field names to the standard schema fields. Populate unmatched fields with `NULL` or domain default values (e.g., `val_status = 'unvalidated'`).
2. **UUID assignment:** Generate a new `uid` (UUID v4) for every legacy feature. If the legacy dataset has its own ID field, preserve it in `bld_id` for backward compatibility.
3. **Temporal anchoring:** Populate `img_date` from whatever metadata is available (image acquisition date, project date, file timestamp). If no date can be determined, use a best-estimate date with a note in `remarks`.
4. **Active-state determination:** If a more recent extraction exists for the same area, set ingested legacy features to `is_active = 0` and link them via `superseded_by` where spatial correspondence can be established (e.g., through centroid-within or IoU-based matching).
5. **Quality flagging:** Set `val_status` to `unvalidated` for all ingested legacy data unless it has been previously quality-checked.

### 5.4 Change Detection Support

The temporal fields in this schema directly support change-detection workflows. By joining features across time epochs on spatial correspondence (e.g., intersect or nearest-centroid matching), analysts can derive:

- **Settlement growth rates** — count of new buildings per epoch.
- **Displacement indicators** — appearance/disappearance of tent or temporary structures.
- **Damage progression** — tracking `dmg_level` changes across multiple post-event image dates.
- **Reconstruction monitoring** — buildings transitioning from `destroyed` back to `no_damage`.

These derived products are critical for MSF operational reporting and should be generated as separate analytical layers, not by modifying the source feature class.

---

**Transparency note:** Claude Opus 4.6 assisted me with the schema, but I am responsible for all content.

---

*This schema is intended as a living document. Proposed amendments should be submitted through the MSF GIS Technical Working Group for review and consensus before adoption.*
