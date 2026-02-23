<img width="1076" height="776" alt="EO data architecture" src="https://github.com/user-attachments/assets/b53c905f-1bbd-4602-b057-0a7772af6915" />


# Data Architecture — Storage & Image Server Strategy

## a. Storage Constraint Scenario

**Scenario:** A processing script raises a low-disk-space warning on the **E: eo_storage** drive (2 TB network drive, daily incremental backups).

**Immediate response:** Stop or pause the running script before the drive fills completely and begins producing corrupt or incomplete outputs. Verify the current state of disk usage to determine exactly how much space remains and which directories are consuming the most. Check whether the script is writing intermediate files (scratch rasters, temporary geodatabases) that are inflating usage beyond the expected final output size.

**Short-term mitigations:** Clear the **Temporary Archive** folder on E: — by definition this content is transient and should be safe to purge. Move any completed "Draft" mosaic datasets and finalized ArcGIS Pro project packages that no longer need active editing to the **I: eo_image** drive (15 TB, with 90-day full backups) or directly to the appropriate Production (PROD) Amazon S3 bucket (e.g., **eo-archive** for original/processed raster zips, **eo-enterprise** for published Cloud Raster Formats (CRFs)). Redirect the script's scratch workspace and temporary raster outputs to a dedicated `temp` directory on I: where space is less constrained. Once enough headroom is recovered, resume the processing job.

**Longer-term recommendations:** Implement automated housekeeping — a scheduled task that enforces a retention policy on E:, flagging or archiving files that haven't been modified in *n* days and purging temp directories on a weekly cycle. Establish a formal **tiered storage lifecycle**: raw imagery lands on E: for active processing, completed outputs are promoted to I: (COG archive) and then to S3, and E: is regularly swept. If 2 TB proves structurally insufficient given growing imagery volumes, make the case to expand E: or provision a dedicated scratch volume sized to the largest expected processing job (e.g., a national mosaic). Finally, add disk-usage monitoring with alerting thresholds (e.g., 75% and 90%) so warnings arrive well before a script fails mid-run.

---

## b. Image Server Strategy

**Scenario:** Propose a strategy for implementing an image server within the existing MSF-Austria EO architecture.

**Steps and approach:** The architecture already has foundational pieces in place — the **image-rasterstore** S3 bucket is federated with ArcGIS Image Server via Portal, the PostgreSQL master databases (PROD and UAT, or User Acceptance Testing) host mosaic datasets that can back image services, and imagery is archived in COG format on I: eo_image. The strategy builds outward from these components rather than introducing parallel infrastructure.

**Short-term (0–3 months):** Stand up image services from the existing mosaic datasets in the PROD PostgreSQL database, publishing through ArcGIS Server federated with Portal. Prioritize the highest-demand layers — typically the most recent very high resolution (VHR) imagery for active emergency sites. Use the COGs already stored on I: as the raster source so that no format conversion is needed. Publish as dynamic image services with default on-the-fly processing templates (e.g., stretch, band combination) and configure the **image-rasterstore** bucket as the cache store so that Portal-served tiles don't consume local disk. Validate performance and access via **eo-share** credentials with a small group of MSF GIS analysts and field users.

**Medium-term (3–9 months):** Expand coverage to the full processed raster archive by building mosaic datasets that reference S3-hosted COGs in **eo-archive** and **eo-enterprise** (published CRFs), reducing dependency on the on-premises I: drive for serving. Introduce server-side raster analytics — NDVI, change detection, on-the-fly clipping — so that field teams can consume analysis-ready products directly from Portal without downloading raw imagery. Develop standardized processing templates and configure role-based access: internal analysts via **eo-enterprise**, external partners and contractors via **eo-exchange** with scoped read/write permissions. Use the **User Acceptance Testing (UAT)** environment (S3 mirror + UAT PostgreSQL) as a staging ground to test new mosaic definitions and processing templates before promoting to PROD.

**Long-term (9–18 months):** Migrate the primary serving architecture to be **cloud-native and S3-backed**, with image services reading directly from COGs in S3 rather than routing through the on-premises network drives. This positions the system for elastic scaling during surge operations (e.g., a major natural disaster requiring rapid imagery dissemination to multiple OCs simultaneously). Integrate a **SpatioTemporal Asset Catalog (STAC)** API layer in front of the S3 archive so that imagery discovery, filtering by date/location/resolution, and service consumption follow open standards — reducing lock-in and enabling interoperability with partners like HOT, MapAction, and UNOSAT. Decommission or repurpose the on-premises I: drive as a local cache/staging tier rather than the primary archive, and consolidate documentation and access workflows on the existing **SharePoint pages** so that the full image service catalog is discoverable organization-wide.

---

**Transparency note:** Claude Opus 4.6 helped me with solutions to these scenarios, but I am responsible for all content.
