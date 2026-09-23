# Terrn Artifacts & Architecture Repository

A centralized repository containing architectural blueprints, High-Level Designs (HLDs), visual design system specifications, performance audits, comparative research, and engineering governance standards for the **Terrn** geospatial analytics platform.

---

## Repository Structure

```
terrn_artifacts/
├── backend-architecture/       # Backend HLDs, MCP server specifications & stack evaluations
├── design/                     # Visual design system, moodboards, icon specs & attribution UX
├── kepler-analysis/            # Kepler.gl architecture verification & adoption recommendations
├── performance-and-audits/     # RCAs, performance audits, defect catalogs & test optimizations
├── standards/                  # Engineering & ARB architectural compliance standards
└── voice-and-content/          # Product voice system, copy governance & PR compliance
```

---

## Directory Index & Documents

### 1. `backend-architecture/`
Architectural blueprints, cloud-native storage designs, and Model Context Protocol (MCP) server specifications for Terrn and related backend infrastructure.

| Document | Format | Description |
| :--- | :--- | :--- |
| [`terrn-backend-hld.md`](./backend-architecture/terrn-backend-hld.md) | Markdown | **HLD (v3.0.0):** Terrn Core Backend & Cloud-Native Storage Architecture (`api.terrn.ai` / `app.terrn.org`). |
| [`terrn-mcp-server-hld.md`](./backend-architecture/terrn-mcp-server-hld.md) | Markdown | **HLD (v1.0.0):** Terrn Model Context Protocol (MCP) Server for AI agent integration (Claude Desktop, Cursor). |
| [`terrn-backend-and-mcp-hld.md`](./backend-architecture/terrn-backend-and-mcp-hld.md) | Markdown | **HLD (`HLD-TERRN-BACKEND-MCP-001`):** Unified Core Backend and MCP Server architecture baseline. |
| [`terrn-backend-and-mcp-hld.html`](./backend-architecture/terrn-backend-and-mcp-hld.html) | HTML | Interactive, styled HTML edition of the Unified Backend & MCP HLD. |
| [`Terrn_Backend_HLD_Review_and_Critique.md`](./backend-architecture/Terrn_Backend_HLD_Review_and_Critique.md) | Markdown | Critical architectural review and remediation blueprint for `terrn-backend-hld.md`. |
| [`Terrn_Backend_Stack_Evaluation_Go_vs_Rust.md`](./backend-architecture/Terrn_Backend_Stack_Evaluation_Go_vs_Rust.md) | Markdown | Evaluation of Go vs. Rust for the Terrn Backend & MCP Server runtime. |
| [`dekart-backend-and-mcp-architecture.md`](./backend-architecture/dekart-backend-and-mcp-architecture.md) | Markdown | Architecture guide and codebase reference for Dekart backend services and MCP server. |
| [`dekart-backend-and-mcp-hld.md`](./backend-architecture/dekart-backend-and-mcp-hld.md) | Markdown | **HLD (`HLD-DEKART-BACKEND-MCP-001`):** Dekart Backend & MCP Server Architecture (ARB Level 3 approved). |

---

### 2. `design/`
Visual identity, component specifications, cartographic design principles, and interactive showcases.

| Document | Format | Description |
| :--- | :--- | :--- |
| [`DESIGN.md`](./design/DESIGN.md) | Markdown | **Master Specification (v2.4.0):** Complete visual design system, design tokens, color palettes, and component hierarchy. |
| [`visual-design-system.html`](./design/visual-design-system.html) | HTML | Interactive visual design system showcase and live token inspector. |
| [`MOODBOARD.md`](./design/MOODBOARD.md) | Markdown | Visual moodboard and design philosophy ("Industrial / Utilitarian Cartographic Workstation"). |
| [`moodboard.html`](./design/moodboard.html) | HTML | Interactive moodboard web showcase. |
| [`layer-type-icons-specification.md`](./design/layer-type-icons-specification.md) | Markdown | Design specification and SVG standards for geospatial layer panel cards (`.layer-type-icon`). |
| [`icon-showcase.html`](./design/icon-showcase.html) | HTML | Interactive showcase of custom layer type icons across all geometry types. |
| [`Terrn_Basemap_Attribution_Competitive_Benchmark_and_Design.md`](./design/Terrn_Basemap_Attribution_Competitive_Benchmark_and_Design.md) | Markdown | Competitive benchmark, legal compliance, and sleek UI implementation design for basemap attribution. |

---

### 3. `kepler-analysis/`
Deep-dive codebase reviews, data ingestion models, and rendering pipeline analyses of Kepler.gl (`/tern/kepler.gl`) for architectural adoption in Terrn.

| Document | Format | Description |
| :--- | :--- | :--- |
| [`kepler_adoption_recommendations_for_terrn.md`](./kepler-analysis/kepler_adoption_recommendations_for_terrn.md) | Markdown | Analysis of patterns, algorithms, and defensive guardrails from Kepler.gl for Terrn's performance architecture. |
| [`kepler_architecture_review_independent.md`](./kepler-analysis/kepler_architecture_review_independent.md) | Markdown | Independent verification of Kepler.gl v3.3.0-alpha.12 and Terrn adoption strategy. |
| [`kepler_data_ingestion_and_rendering_architecture.md`](./kepler-analysis/kepler_data_ingestion_and_rendering_architecture.md) | Markdown | Comprehensive analysis of Kepler.gl's WebGL-powered data ingestion and Deck.gl rendering pipeline. |

---

### 4. `performance-and-audits/`
Root cause analyses, performance benchmarks, defect catalogs, and test suite optimization plans.

| Document | Format | Description |
| :--- | :--- | :--- |
| [`RCA_and_Competitive_Analysis.md`](./performance-and-audits/RCA_and_Competitive_Analysis.md) | Markdown | Root Cause Analysis & Architectural Competitive Analysis: Terrn (`tern_poc`) vs. GeoLibre (`geolibre`). |
| [`Terrn_Performance_Optimization_Guide.md`](./performance-and-audits/Terrn_Performance_Optimization_Guide.md) | Markdown | Full-stack client-side geospatial performance audit and optimization guide. |
| [`Terrn_Detailed_Issues_Catalog.md`](./performance-and-audits/Terrn_Detailed_Issues_Catalog.md) | Markdown | Exhaustive technical audit of rendering, state, ingestion, and interaction bottlenecks. |
| [`Terrn_Defects_to_Solutions_Mapping.md`](./performance-and-audits/Terrn_Defects_to_Solutions_Mapping.md) | Markdown | Defect-to-solution mapping and architectural remediations cross-referenced with GeoLibre. |
| [`Terrn_Architectural_Proposals_and_Risk_Analysis.md`](./performance-and-audits/Terrn_Architectural_Proposals_and_Risk_Analysis.md) | Markdown | Critical review and risk analysis of enhanced remediations for spatial clustering and raster ingestion. |
| [`Terrn_Sprint11_R1_Benchmark_Implementation_and_PR_Review.md`](./performance-and-audits/Terrn_Sprint11_R1_Benchmark_Implementation_and_PR_Review.md) | Markdown | Sprint 11 R1 benchmark harness flow, methodology, and review of Pull Requests #42 through #56. |
| [`Terrn_Unit_Test_Suite_Audit_and_Optimization.md`](./performance-and-audits/Terrn_Unit_Test_Suite_Audit_and_Optimization.md) | Markdown | Unit test suite audit, runtime profiling, and parallelization strategy. |
| [`Terrn_E2E_Test_Suite_Audit_and_Optimization.md`](./performance-and-audits/Terrn_E2E_Test_Suite_Audit_and_Optimization.md) | Markdown | Playwright E2E test suite audit and consolidation guide to reduce CI runtimes. |

---

### 5. `voice-and-content/`
Standards, glossaries, and automated test compliance audits governing product copy and terminology across Terrn.

| Document | Format | Description |
| :--- | :--- | :--- |
| [`Terrn_Voice_System_Review_and_PR_Compliance_Audit.md`](./voice-and-content/Terrn_Voice_System_Review_and_PR_Compliance_Audit.md) | Markdown | Comprehensive review of `VOICE.md`, enforcement test harnesses, and compliance audit of PRs 31–35. |
| [`Terrn_Voice_Changes_Reference_Table.md`](./voice-and-content/Terrn_Voice_Changes_Reference_Table.md) | Markdown | Canonical reference matrix mapping user-facing copy refactors (`REF-xxx`) across PRs 31–35. |

---

### 6. `standards/`
Engineering governance, compliance requirements, and standards for architecture documentation.

| Document | Format | Description |
| :--- | :--- | :--- |
| [`LLM_Agent_HLD_Creation_and_Compliance_Guide.md`](./standards/LLM_Agent_HLD_Creation_and_Compliance_Guide.md) | Markdown | **Standard (`STD-ARCH-LLM-HLD-002` v3.0.0):** Progressive HLD authoring and ARB compliance guide for LLM agents. |
