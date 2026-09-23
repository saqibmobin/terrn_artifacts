# High-Level Design (HLD): Terrn Model Context Protocol (MCP) Server
## Standardized AI Tooling & Spatial Intelligence Gateway for Claude Desktop, Cursor, and Autonomous Agents

**Document Version:** 1.0.0  
**Date:** August 21, 2026  
**Target Environments:** Local Desktop (`stdio`) & Production Cloud (`mcp.terrn.ai` via SSE/HTTP)  
**Protocol Specification:** Model Context Protocol (MCP) v1.0 / JSON-RPC 2.0  

---

## 1. Executive Summary & Purpose

The **Terrn MCP Server** bridges Large Language Models (LLMs) and autonomous AI agents (such as Claude Desktop, Cursor, Windsurf, and custom agentic workflows) directly to Terrn's spatial analytics and cartographic visualization engine.

By implementing the open **Model Context Protocol (MCP)**, Terrn exposes its **DuckDB analytical engine, spatial algorithms, layer catalog, and styling controls** as standard, discoverable AI tools, resources, and prompt templates.

```
+----------------------------------------------------------------------------------------------------+
|                                      TERRN MCP ARCHITECTURE TOPOLOGY                               |
+----------------------------------------------------------------------------------------------------+
|                                                                                                    |
|  [AI CLIENTS]                                                                                      |
|  * Claude Desktop / Cursor / Windsurf / Custom AI Agents                                           |
|          |                                              |                                          |
|          | (Local: stdio Transport)                     | (Remote: SSE / HTTP Transport)           |
|          v                                              v                                          |
|  +----------------------------------------------------------------------------------------------+  |
|  | TERRN MCP SERVER (@modelcontextprotocol/sdk)                                                 |  |
|  |                                                                                              |  |
|  |  +--------------------------+  +--------------------------+  +----------------------------+  |  |
|  |  | 1. TOOLS (Execution)     |  | 2. RESOURCES (Data URIs) |  | 3. PROMPTS (Templates)     |  |  |
|  |  | * execute_spatial_sql    |  | * terrn://projects/{id}  |  | * analyze_distribution     |  |  |
|  |  | * apply_layer_style      |  | * terrn://layers/{id}/   |  | * find_optimal_location    |  |  |
|  |  | * spatial_buffer         |  |   schema                 |  | * generate_thematic_map   |  |  |
|  |  | * spatial_intersection   |  | * terrn://layers/{id}/   |  |                            |  |  |
|  |  | * export_dataset         |  |   summary                |  |                            |  |  |
|  |  +--------------------------+  +--------------------------+  +----------------------------+  |  |
|  +----------------------------------------------------------------------------------------------+  |
|          |                                              |                                          |
|          v                                              v                                          |
|  [LOCAL ENGINE (Local stdio mode)]              [TERRN CLOUD API (Remote SSE mode)]                |
|  * Local DuckDB WASM / Node-DuckDB              * api.terrn.ai (FastAPI Gateway)                   |
|  * In-Memory GeoJSON / FlatGeobuf               * PostgreSQL Metadata & Permissions                |
|  * Local Map Canvas WebSocket Bridge            * Cloudflare R2 / S3 Lakehouse Assets              |
+----------------------------------------------------------------------------------------------------+
```

---

## 2. Core Architectural Capabilities

1. **Dual Transport Support:**
   * **`stdio` (Local Mode):** Runs as a local Node.js/Python CLI binary (`npx @terrn/mcp-server`) configured inside Claude Desktop or Cursor settings.
   * **`SSE / HTTP` (Remote Cloud Mode):** Hosted at `https://mcp.terrn.ai/sse`, authenticated via Terrn API Key (`Bearer trn_live_...`) for cloud workspaces.
2. **Bidirectional Map Synchronization:**
   * When an AI client executes a tool (e.g. `apply_layer_style` or `execute_spatial_sql`), the MCP server dispatches live WebSocket events to the active Terrn map canvas, updating the user's map in real time.
3. **Zero-Geometry Prompt Privacy:**
   * To prevent LLM context window overflow and preserve privacy, the MCP server returns **compact summaries, column schemas, bounding boxes, and statistical distributions** rather than millions of raw coordinate pairs.

---

## 3. MCP Primitives: Tools, Resources, and Prompts

### 3.1 Exhaustive Tool Catalog (AI Function Calling)

The server exposes 8 high-leverage spatial tools categorized into Discovery, Analytics, Cartography, and Export:

```
┌────────────────────────────────────────────────────────────────────────────────────────────┐
│ TERRN MCP TOOL CATALOG                                                                     │
├─────────────────────────┬──────────────────────────────────────────────────────────────────┤
│ Category                │ Tool Identifier & Action                                         │
├─────────────────────────┼──────────────────────────────────────────────────────────────────┤
│ 1. Metadata & Discovery │ • terrn_list_projects — List available workspaces and maps       │
│                         │ • terrn_get_layer_schema — Get column types, sample values & bbox│
│                         │ • terrn_get_layer_statistics — Compute column stats & quantiles  │
├─────────────────────────┼──────────────────────────────────────────────────────────────────┤
│ 2. Spatial Analytics    │ • terrn_execute_spatial_sql — Run analytical DuckDB SQL queries  │
│                         │ • terrn_spatial_buffer — Generate proximity buffers around points│
│                         │ • terrn_spatial_intersection — Point-in-polygon & spatial joins  │
├─────────────────────────┼──────────────────────────────────────────────────────────────────┤
│ 3. Cartography & Style  │ • terrn_apply_thematic_style — Apply Categorized/Graduated styles│
│                         │ • terrn_set_layer_visibility — Toggle visibility and opacity     │
├─────────────────────────┼──────────────────────────────────────────────────────────────────┤
│ 4. Export & Snapshots   │ • terrn_export_dataset — Export query result as GeoJSON/Parquet  │
│                         │ • terrn_render_map_snapshot — Generate high-resolution map image │
└─────────────────────────┴──────────────────────────────────────────────────────────────────┘
```

#### Tool Schema Specifications:

#### `terrn_execute_spatial_sql`
Executes SQL queries against project layers using DuckDB’s spatial engine.
```json
{
  "name": "terrn_execute_spatial_sql",
  "description": "Execute spatial and tabular SQL queries against layer tables in DuckDB. Supports aggregations, window functions, and spatial predicates.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "projectId": { "type": "string", "description": "Target project UUID" },
      "sql": { "type": "string", "description": "DuckDB SQL statement (e.g. SELECT state, COUNT(*) FROM villages GROUP BY state)" },
      "createNewLayer": { "type": "boolean", "description": "If true, registers the query result as a new map layer", "default": false },
      "newLayerName": { "type": "string", "description": "Name for the newly created layer" }
    },
    "required": ["projectId", "sql"]
  }
}
```

#### `terrn_apply_thematic_style`
Applies data-driven visual symbology to an active map layer.
```json
{
  "name": "terrn_apply_thematic_style",
  "description": "Apply data-driven cartographic styling (Categorized colors, Graduated intervals, Bubble sizes, or Heatmaps) to a map layer.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "projectId": { "type": "string", "description": "Target project UUID" },
      "layerId": { "type": "string", "description": "Target layer UUID" },
      "mode": { "type": "string", "enum": ["Basic", "Categorized", "Graduated", "Bubble", "Heatmap"] },
      "field": { "type": "string", "description": "Attribute field to style by" },
      "colorRamp": { "type": "string", "enum": ["viridis", "plasma", "inferno", "spectral", "blues", "reds"], "default": "viridis" },
      "breaksCount": { "type": "integer", "description": "Number of classes for Graduated mode (e.g. 5)", "default": 5 },
      "opacity": { "type": "number", "minimum": 0, "maximum": 1, "default": 0.85 }
    },
    "required": ["projectId", "layerId", "mode"]
  }
}
```

#### `terrn_spatial_buffer`
Calculates proximity buffers around feature geometries.
```json
{
  "name": "terrn_spatial_buffer",
  "description": "Generate a geometric buffer zone around points, lines, or polygons at a specified distance in meters or kilometers.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "projectId": { "type": "string", "description": "Target project UUID" },
      "layerId": { "type": "string", "description": "Input layer UUID to buffer" },
      "distance": { "type": "number", "description": "Buffer distance value" },
      "unit": { "type": "string", "enum": ["meters", "kilometers", "miles"], "default": "meters" },
      "outputLayerName": { "type": "string", "description": "Name for the generated buffer polygon layer" }
    },
    "required": ["projectId", "layerId", "distance"]
  }
}
```

---

### 3.2 Resource URI Scheme (`terrn://`)

The MCP server exposes readable resources that AI agents can inspect to understand current project state without executing tool calls:

| Resource URI Pattern | Content Returned | MIME Type |
| :--- | :--- | :--- |
| `terrn://projects` | List of all accessible projects, names, and IDs | `application/json` |
| `terrn://projects/{id}` | Active project view state, basemap, and layer list | `application/json` |
| `terrn://projects/{id}/layers/{layerId}/schema` | Column names, data types, distinct categories, null counts | `application/json` |
| `terrn://projects/{id}/layers/{layerId}/stats` | Min, max, average, standard deviation, and quantile breaks | `application/json` |
| `terrn://projects/{id}/layers/{layerId}/preview.geojson` | Bounding-box sampled preview of up to 50 feature records | `application/geo+json` |

---

### 3.3 Prompt Templates (Workflows)

The MCP server provides standard, structured prompts for common GIS analysis patterns:

1. **`analyze_spatial_distribution`:** Guides the LLM to inspect numeric distributions, compute quantile intervals, apply a graduated color ramp, and explain spatial clustering patterns.
2. **`find_optimal_location`:** Automates multi-criteria site selection by running buffer intersections, filtering constraints, and scoring suitability across layers.
3. **`generate_thematic_map`:** Examines attribute columns, suggests appropriate cartographic color ramps (e.g. sequential vs. diverging), and applies the optimal styling.

---

## 4. Implementation Architecture & Tech Stack

### 4.1 Technology Stack
* **Core SDK:** `@modelcontextprotocol/sdk` (TypeScript / Node.js 20+)
* **Query Engine:** `@duckdb/node-api` (local execution) or `node-fetch` (cloud REST bridge)
* **Spatial Algorithms:** `@turf/turf` and `polylabel` for geometric operations
* **Transport Drivers:**
  * Stdio Server: `StdioServerTransport`
  * Cloud Server: `SSEServerTransport` with Express / Hono

### 4.2 Project Structure
```
packages/mcp-server/
├── bin/
│   └── terrn-mcp.ts              <-- CLI entrypoint for npx execution
├── src/
│   ├── index.ts                  <-- Server initialization & transport routing
│   ├── tools/                    <-- Tool handlers
│   │   ├── discovery.ts          <-- list_projects, get_layer_schema
│   │   ├── analytics.ts          <-- execute_spatial_sql, buffer, intersect
│   │   ├── styling.ts            <-- apply_thematic_style, visibility
│   │   └── export.ts             <-- export_dataset, map_snapshot
│   ├── resources/                <-- Resource URI resolvers (terrn://)
│   ├── prompts/                  <-- Prompt template definitions
│   └── client/                   <-- Backend API / WebSocket canvas bridge
├── package.json
└── tsconfig.json
```

---

## 5. Setup & Integration Guides

### 5.1 Connecting to Claude Desktop
Users add the Terrn MCP server to their `claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "terrn": {
      "command": "npx",
      "args": ["-y", "@terrn/mcp-server@latest"],
      "env": {
        "TERRN_API_KEY": "trn_live_your_api_key_here",
        "TERRN_DEFAULT_PROJECT": "project-uuid-here"
      }
    }
  }
}
```

### 5.2 Connecting to Cursor / Windsurf IDE
In `.cursor/mcp.json`:
```json
{
  "mcpServers": {
    "terrn-spatial": {
      "url": "https://mcp.terrn.ai/sse",
      "headers": {
        "Authorization": "Bearer trn_live_your_api_key_here"
      }
    }
  }
}
```

---

## 6. Security, Authentication & Governance

* **API Key Scoping:** Cloud MCP connections enforce token permissions (`read:projects`, `write:styles`, `execute:sql`).
* **SQL Injection Prevention:** All SQL queries are executed in read-only DuckDB transactions with disabled filesystem execution (`SET enable_external_access = false`).
* **Rate Limiting:** MCP tool executions are metered against workspace API quotas to prevent runaway LLM execution loops.

---

## 7. Phased Implementation Roadmap

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│ PHASE 1: LOCAL STDIO MCP SERVER (Core Tools)                                           │
│ 1. Implement TypeScript MCP server using @modelcontextprotocol/sdk.                    │
│ 2. Build metadata tools (terrn_list_projects, terrn_get_layer_schema).                 │
│ 3. Build styling tools (terrn_apply_thematic_style, terrn_set_layer_visibility).       │
│ 4. Publish as npm package: @terrn/mcp-server.                                          │
├────────────────────────────────────────────────────────────────────────────────────────┤
│ PHASE 2: SPATIAL SQL & ANALYTICS TOOLS                                                 │
│ 1. Wire terrn_execute_spatial_sql via DuckDB spatial engine.                           │
│ 2. Implement terrn_spatial_buffer and terrn_spatial_intersection with Turf.js.         │
│ 3. Add terrn:// resource URI handlers for layer schemas and statistical summaries.     │
├────────────────────────────────────────────────────────────────────────────────────────┤
│ PHASE 3: CLOUD SSE GATEWAY & LIVE CANVAS SYNC                                          │
│ 1. Deploy hosted SSE server at mcp.terrn.ai with API Key authentication.               │
│ 2. Connect WebSocket bridge for real-time visual updates on active browser maps.       │
│ 3. Release prompt templates and Claude Desktop / Cursor extension integrations.       │
└────────────────────────────────────────────────────────────────────────────────────────┘
```
