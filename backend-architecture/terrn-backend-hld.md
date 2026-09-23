# High-Level Design (HLD): Terrn Core Backend & Cloud-Native Storage Architecture

**Document Version:** 3.0.0  
**Date:** August 21, 2026  
**Target Environment:** Production (`api.terrn.ai` / `app.terrn.org`)  
**Architecture Classification:** Lean Cloud Storage & Metadata Backend with Client-Side Dynamic Worker Tiling  

---

## 1. Executive Summary & Architectural Philosophy

Terrn adopts a **lean, cloud-native storage and metadata backend** paired with **client-side dynamic Web Worker tiling**.

Rather than maintaining heavy, expensive, and fragile server-side tiling infrastructure (GDAL/Tippecanoe container worker pools), Terrn stores vector layers in compressed, cloud-native binary formats (**GeoParquet** and **FlatGeobuf**) on edge object storage (S3 / Cloudflare R2). When a project is opened, the client fetches the compressed asset and slices it dynamically into vector tiles inside background Web Workers using the **`terrn-vt://` custom MVT protocol**.

```
+----------------------------------------------------------------------------------------------------+
|                                    TERRN END-TO-END SYSTEM TOPOLOGY                                |
+----------------------------------------------------------------------------------------------------+
|                                                                                                    |
|  +----------------------------------------------------------------------------------------------+  |
|  | CLIENT LAYER (Browser / Desktop Application)                                                 |  |
|  |  * MapLibre GL JS (Vector Tile / COG Shaders)                                                |  |
|  |  * terrn-vt:// Dynamic Tiling Engine in Web Worker (GeoJSONVT & Supercluster)                |  |
|  |  * DuckDB-WASM (Instant local SQL queries, calculated fields, and filters)                   |  |
|  |  * TanStack Table v9 + Virtualizer (60 FPS Data Grid)                                        |  |
|  +----------------------------------------------------------------------------------------------+  |
|          |                                              ^                        ^                 |
|          | (1. Direct Upload: S3 Presigned URL)         | (2. Metadata / Auth)   | (3. Fast Fetch) |
|          v                                              v                        |                 |
|  +-------------------------------------------------------------+                 |                 |
|  | API GATEWAY (FastAPI / Node.js Hono)                        |                 |                 |
|  |  * JWT Auth & Team Row-Level Security                       |                 |                 |
|  |  * Project, Layer & StyleConfig Metadata Management         |                 |                 |
|  |  * Managed AI Gateway Service (Server-Held Credentials)     |                 |                 |
|  +-------------------------------------------------------------+                 |                 |
|          |                                                                       |                 |
|          v                                                                       |                 |
|  +---------------------------+       +-------------------------------------------+                 |
|  | METADATA DB (PostgreSQL)  |       | EDGE STORAGE & CDN (S3 / Cloudflare R2)   |-----------------+
|  | - Projects & Workspaces   |       | - Vector: .parquet / .fgb / .geojson.gz   |
|  | - Layer Styles & Versions |       | - Raster: Cloud-Optimized GeoTIFF (.tif)  |
|  | - Org Token Quotas & RLS  |       | - Thumbnails & Export Snapshots           |
|  +---------------------------+       +-------------------------------------------+
+----------------------------------------------------------------------------------------------------+
```

---

## 2. Core Benefits of Client-Side Dynamic Tiling Over Server Tiling

| Dimension | Legacy Server-Side Tiling (Tippecanoe on S3) | Terrn Client-Side Dynamic Tiling (terrn-vt://) |
| :--- | :--- | :--- |
| **Upload Speed** | ⚠️ User waits 30–90 seconds in a "Processing / Tiling..." queue | ✅ **Instant upload ($< 1\text{ second}$)**; ready to explore immediately |
| **Backend Cost** | 🔴 Expensive containerized worker compute (ECS Fargate/Cloud Run) | ✅ **Zero compute server cost**; only static storage (pennies/GB) |
| **Operational Reliability** | 🔴 High failure rate (worker OOMs, disk scratch exhaustion) | ✅ **100% stateless & crash-free**; runs in isolated client Web Workers |
| **Pipeline Complexity** | 🔴 Dual code paths (local layer rendering vs remote tile rendering) | ✅ **Unified single code path** for both local drops and cloud layers |
| **In-Memory Interactivity** | ⚠️ Re-querying or filtering requires re-tiling or server round-trips | ✅ **Instant client-side DuckDB SQL queries** ($0\text{ ms}$ network latency) |

---

## 3. Storage Tier: Cloud-Native Static Asset Store

### 3.1 Storage Format Matrix

| Layer Type | Storage Format on S3 / R2 | Compression | Client-Side Consumption Engine |
| :--- | :--- | :--- | :--- |
| **Standard Vector ($\le 100\text{k}$ features)** | **GeoParquet (`.parquet`)** or **FlatGeobuf (`.fgb`)** | ZSTD / Snappy | Fetched via CDN, sliced into vector tiles in Web Worker via `terrn-vt://` |
| **Raw Vector Drop** | **GeoJSON (`.geojson.gz`)** | Gzip | Parsed in Worker, indexed via `GeoJSONVT` |
| **Raster Imagery** | **Cloud-Optimized GeoTIFF (`.cog.tif`)** | Deflate / LERC | MapLibre WebGL raster shader requests tile overviews via HTTP byte ranges |

### 3.2 S3 Bucket Structure
```
s3://terrn-storage-prod/
├── orgs/{org_id}/
│   └── projects/{project_id}/
│       ├── layers/{layer_id}/
│       │   ├── data.parquet        <-- Compressed columnar vector data
│       │   ├── data.fgb            <-- Spatial indexed FlatGeobuf
│       │   └── metadata.json       <-- Pre-computed bbox, feature count, schema
│       └── thumbnail.webp
```

---

## 4. End-to-End Ingestion & Viewing Workflow

### 4.1 Ingestion Flow (Instant Save)
1. **Request Presigned Upload URL:** The client requests a direct S3/R2 upload URL (`POST /api/v1/projects/{id}/layers/upload-url`).
2. **Direct Binary Upload:** The client uploads the file directly to S3 via HTTP `PUT` with zero API server memory overhead.
3. **Register Metadata:** The client sends the pre-computed bounding box, feature count, column schema, and default style config to the API (`POST /api/v1/projects/{id}/layers`).
4. **Instant Confirmation:** The layer is immediately marked **Active** in the project ($< 1\text{ second}$ total time).

### 4.2 Layer Loading & Rendering Flow
1. **Fetch Layer Record:** On project load, the client retrieves the layer metadata and S3 asset URL from PostgreSQL.
2. **Stream Binary Buffer:** The client fetches the compressed `.parquet` or `.fgb` file via Edge CDN.
3. **Worker Dynamic Tiling:** The background Web Worker decompresses the data, loads it into `GeoJSONVT` (or `Supercluster` for point layers), and registers the `terrn-vt://` protocol.
4. **60 FPS MapLibre Rendering:** MapLibre requests viewport tiles (`terrn-vt://{layerId}/{z}/{x}/{y}`) and renders them with native GLSL style expressions.

---

## 5. Relational Database Schema (PostgreSQL 16+)

```sql
-- 1. Organizations & Workspaces
CREATE TABLE organizations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(100) UNIQUE NOT NULL,
    plan VARCHAR(50) DEFAULT 'pro',
    ai_monthly_token_quota BIGINT DEFAULT 1000000,
    ai_tokens_used_this_month BIGINT DEFAULT 0,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- 2. Projects (Maps)
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id UUID REFERENCES organizations(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    view_state JSONB NOT NULL DEFAULT '{"center": [78.9629, 20.5937], "zoom": 4}',
    basemap VARCHAR(50) DEFAULT 'dark',
    created_by UUID NOT NULL,
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- 3. Layer Registry
CREATE TABLE layers (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id UUID REFERENCES projects(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    geometry_type VARCHAR(50) NOT NULL,    -- 'Point', 'Polygon', 'LineString', 'Raster'
    feature_count INTEGER DEFAULT 0,
    bounds DOUBLE PRECISION[4],            -- [minX, minY, maxX, maxY]
    storage_url VARCHAR(512) NOT NULL,     -- S3 / R2 asset URL
    format VARCHAR(20) NOT NULL,           -- 'parquet', 'flatgeobuf', 'geojson', 'cog'
    schema_fields JSONB NOT NULL,          -- Column definitions & sample data
    style_config JSONB NOT NULL,           -- Terrn StyleConfig (color, mode, breaks)
    visible BOOLEAN DEFAULT TRUE,
    opacity REAL DEFAULT 0.85,
    layer_order INTEGER NOT NULL,
    version INTEGER NOT NULL DEFAULT 1,    -- Optimistic Concurrency Control
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

---

## 6. Server-Managed AI Intelligence Gateway

* **Server-Held Credentials:** LLM provider keys (Anthropic Claude, OpenAI, Google Gemini) are stored securely in backend vault infrastructure.
* **Organization Quota & Metering:** Token usage is tracked per organization and deducted from the workspace's monthly quota.
* **Zero-Geometry Privacy:** Raw coordinate arrays and sensitive geometries are **never transmitted to external LLMs**—only column schemas, bounding boxes, and metadata are sent.
* **Structured Spatial Tool Calling:** The AI Gateway streams structured geospatial operations (`setLayerStyle`, `filterLayer`, `generateSpatialSQL`, `createBuffer`) that execute safely inside the client's DuckDB-WASM and MapLibre engines.

---

## 7. Optional Extension: Opt-In Serverless Tiling for Mega-Datasets ($> 100\text{ MB}$)

While dynamic client-side tiling handles 95% of datasets, Terrn provides an **opt-in background tiling path** exclusively for mega-scale layers ($> 100\text{ MB}$ / $> 250\text{k}$ features) or high-traffic public embeds:

* **Trigger Condition:** Dataset $> 100\text{ MB}$ OR user explicitly toggles *"Optimize for Public Embed"*.
* **Mechanism:** An asynchronous containerized worker runs **Tippecanoe** to generate a single **PMTiles** archive.
* **Serving:** MapLibre seamlessly switches from `terrn-vt://` to `pmtiles://` using HTTP range requests.

---

## 8. Phased Implementation Roadmap

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│ PHASE 1: LEAN CLOUD STORAGE & AUTHENTICATION                                           │
│ 1. Set up PostgreSQL schema (Projects, Layers, Organizations) with Row-Level Security. │
│ 2. Build FastAPI Gateway with Presigned S3 Upload/Download endpoints.                 │
│ 3. Implement S3/Cloudflare R2 storage bucket with CDN caching headers.                 │
├────────────────────────────────────────────────────────────────────────────────────────┤
│ PHASE 2: CLIENT-SIDE DYNAMIC TILING INTEGRATION (terrn-vt://)                          │
│ 1. Wire client-side GeoParquet & FlatGeobuf streaming loaders in Web Workers.          │
│ 2. Connect terrn-vt:// dynamic MVT protocol to cloud-stored layers.                    │
│ 3. Implement Optimistic Concurrency Control (version / etag) on layer styles.          │
├────────────────────────────────────────────────────────────────────────────────────────┤
│ PHASE 3: SERVER-MANAGED AI GATEWAY & WORKSPACE COLLABORATION                           │
│ 1. Deploy server-managed AI Copilot streaming gateway with geospatial tool calling.    │
│ 2. Build project sharing, permission controls, and PNG/PDF export generation.          │
│ 3. Add opt-in background PMTiles generation for mega-datasets (> 100 MB).              │
└────────────────────────────────────────────────────────────────────────────────────────┘
```