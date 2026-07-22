---
name: carto-render-inline-map
description: Render an ad-hoc interactive map inline in the chat from a deck.gl declarative spec via the CARTO MCP server's view_map tool. Use whenever the user asks to map, visualize, or show the geographic distribution of points, polygons, hexagons, quadbins, clusters, density (heatmaps), or raster — and the map is exploratory or throwaway, not meant to be saved as a permanent CARTO Builder map. Triggers on "show me X on a map", "visualize Y", "make a heatmap of Z", "render the points/clusters/raster of W", "put this GeoJSON / geocoded point / route / isoline on a map". Also read this BEFORE rendering any raw GeoJSON or LDS result — view_map has no GeoJSON layer; there is a specific recipe. Distinct from carto-create-builder-maps (CLI authoring of permanent maps), carto-preview-builder-map (loading an existing saved Builder map), and carto-develop-app (writing a from-scratch deck.gl app in TypeScript / JavaScript).
license: MIT
---

# carto-render-inline-map

Renders an ad-hoc interactive map inline in the chat via the CARTO MCP server's `view_map` tool. The agent emits a `@deck.gl/json` declarative spec; the renderer handles credentials, basemap, and tooltips. The user sees the map without leaving the chat.

**Closed layer universe — the #1 cause of an empty map.** `view_map` accepts ONLY CARTO tile layers fed by CARTO sources. Generic deck.gl layers (`GeoJsonLayer`, `ScatterplotLayer`, `HexagonLayer`, …) and inline/raw GeoJSON data are NOT accepted, and the failure is **silent** — the map renders empty with no error. Holding raw GeoJSON (an LDS geocode/route/isoline result, a computed geometry) is NOT a reason to reach for `GeoJsonLayer`: the GeoJSON goes into SQL instead — see "Rendering raw GeoJSON / LDS results" below. An empty map means a wrong layer/source pairing, never that a different rendering technology is needed.

**Legend required after every render.** The `view_map` renderer does NOT show an auto-legend. After invoking `view_map`, render a legend through the host's widget surface (e.g., `show_widget` in Claude.ai / Claude Desktop) — that is the ONLY visual transport that works. Chat-message HTML is escaped by every major host's renderer and appears as raw text — DO NOT emit HTML in your chat reply. If the host has no widget tool, fall back to a plain-text legend (markdown bullets with emoji color squares per bucket, hex codes in parentheses). The full HTML template, style rules, and per-helper variants (`colorBins` swatches, `colorContinuous` gradient bar, `colorCategories` per-category, raster ternary-bucket + nodata) live in the `view_map` tool description's LEGEND section. Applies to `view_map` ONLY; `load_builder_map` has its own Builder-native legend.

**Tool contract.** This skill consumes the `view_map` tool exposed by the CARTO MCP server. The tool's input shape (`deckglProps`), layer-source compatibility, `aggregationExp` requirements, and `@@=` expression-eval restrictions are documented in the tool's own MCP description — read it via the MCP host's tool-inspector or by calling `tools/list`. This skill stays focused on routing, cartography, and the agent's reply; it does NOT duplicate the tool's spec.

This skill assumes the **CARTO MCP server is attached** (the `view_map` tool is in your tool list) AND the **host supports MCP Apps** (interactive widgets — Claude.ai, Claude Desktop, ChatGPT). If either is missing, see "Step 1 — detect what's available" below.

## Step 1 — detect what's available

| Check | How |
|---|---|
| `view_map` is callable | Tool name `view_map` is in your tool list. |
| Host renders MCP Apps | Hosts that DO: Claude.ai, Claude Desktop, ChatGPT. Hosts that DON'T (Gemini CLI, Codex CLI, plain MCP Inspector, current MCPJam) execute the tool but only show a text confirmation — no map widget. |

| Setup | What to do |
|---|---|
| Tool present + host renders | Proceed normally. |
| Tool present + host doesn't render | Tell the user the host can't render maps inline; suggest switching hosts or using `carto-create-builder-maps` (CLI) for a screenshot-based alternative. |
| Tool not present | The MCP server isn't attached. Tell the user; don't fall back to a generic visualization widget. |

## When to pick a different skill

- **Permanent / shareable map** → `carto-create-builder-maps` (CLI). `view_map` specs aren't saved or shareable as URLs; they live in the chat.
- **Open an existing saved map by name/URL/ID** → `carto-preview-builder-map`. That skill uses `load_builder_map` to render a saved Builder map inline.
- **Writing a TypeScript/JavaScript app from scratch** → `carto-develop-app`. Different runtime (full deck.gl surface in JS), different cartography rules.

## Discovery flow before composing the spec

1. `list_connections` → identify the right connection (often `carto_dw`).
2. `search_resources` (by name) or `list_resources` (by FQN) to find the table.
3. **Always call `get_column_stats` for any unfamiliar numeric column you'll bin on** — quantiles, min, max, categories. Skipping this and hardcoding `colorBins` thresholds is the #1 styling failure mode.
4. Compose the `view_map` spec.

## Composition essentials

For the full deck.gl declarative spec — layer-source compatibility, `aggregationExp` rules, `mapStyle` URLs, `@@function` shapes, expression-eval restrictions — read the `view_map` tool description directly. This skill stays focused on routing and cartographic decisions.

For cartographic decisions on the spec (palette, scale, basemap, stroke, drawing order, hierarchy, picking, anti-patterns, worked recipes), read [`references/cartography.md`](references/cartography.md). Mandatory before composing any styled spec.

## Rendering raw GeoJSON / LDS results (geocoded point, route, isoline)

`view_map` has no GeoJSON layer. The sanctioned path for a small in-hand geometry is a `vectorQuerySource` whose SQL constructs it, rendered by a regular `VectorTileLayer`:

```
{ "@@type": "VectorTileLayer",
  "data": { "@@function": "vectorQuerySource",
            "connectionName": "<a CARTO connection>",
            "sqlQuery": "SELECT ST_GEOGFROMGEOJSON('<bare geometry JSON>') AS geom" } }
```

(BigQuery shown — pick the constructor by the connection's provider from `list_connections`.)

| Provider | GeoJSON geometry (line/polygon) | Point from lon/lat |
|---|---|---|
| BigQuery | `ST_GEOGFROMGEOJSON('…')` | `ST_GEOGPOINT(lon, lat)` |
| Snowflake | `TO_GEOGRAPHY(PARSE_JSON('…'))` | `ST_MAKEPOINT(lon, lat)` |
| Databricks | `st_geogfromgeojson('…')` (DBR 17.1+, not SQL Classic) | `st_point(lon, lat)` |
| Redshift | `ST_GeomFromGeoJSON('…')` | `ST_Point(lon, lat)` |
| Oracle | `SDO_UTIL.FROM_GEOJSON('…')` | `SDO_GEOMETRY(2001, 4326, SDO_POINT_TYPE(lon, lat, NULL), NULL, NULL)` |
| PostGIS | `ST_GeomFromGeoJSON('…')` | `ST_SetSRID(ST_MakePoint(lon, lat), 4326)` |

Rules that prevent the classic failures:

- Pass a **bare geometry** (`Point`/`LineString`/`MultiLineString`/`Polygon`/`MultiPolygon`), never a `Feature`/`FeatureCollection` — BigQuery and PostGIS reject those. LDS extraction: geocode → `data["0"].value[0].longitude`/`.latitude` (no GeoJSON in that response — use the point constructor); routing → `data.value.route` as-is; isolines → `features[i].geometry` (one feature per requested range).
- The geometry `type` varies (`LineString` vs `MultiLineString`, `Polygon` vs `MultiPolygon`) — the constructors accept all of them; don't type-branch or read `coordinates` as a single line/ring.
- Small geometry only (it's embedded in the SQL). A large one goes through a Workflow to a table, then `vectorTableSource`.

LDS usage rules (batching, provider, size limits) live in `carto-location-services`; full per-dialect caveats and verification queries in that skill's `references/rendering-geometry.md`.

## Anti-patterns to surface or self-correct

- **Falling back to a generic visualization widget when `view_map` is available.** If the tool is in your list, use it.
- **`view_map` for a saved map referenced by name.** Switch to `carto-preview-builder-map` and call `list_maps` first.
- **Hardcoded `colorBins` domain values without `get_column_stats`.** Always fetch real percentiles for unfamiliar columns.
- **Mixing tile schemes** (e.g., `vectorTableSource` → `HeatmapTileLayer`). Silent empty render. The `view_map` tool description has the full compatibility matrix.
- **Generic deck.gl layers** (`ScatterplotLayer`, `HexagonLayer`, `GeoJsonLayer`, etc.). The MCP JSON converter only registers CARTO layers — anything else silently produces nothing. For raw GeoJSON in hand, use the recipe above.
- **Switching rendering technology when the map comes back empty.** An empty render is almost always a layer↔source mismatch — a generic layer class, mixed tile schemes, or a missing dialect constructor. Fix the layer/source per this skill; do NOT fall back to SVG/HTML, a charting widget, or any non-CARTO renderer.
- **Treating an inline preview as a saved/shareable map.** It isn't. If the user wants to keep it, route to `carto-create-builder-maps`.

## Post-CLI-creation preview pattern

When the user creates a permanent map via the CLI (`carto maps create` from `carto-create-builder-maps`), the response is a `mapId` + Builder URL. The fastest way to verify the result inline is `load_builder_map` (in `carto-preview-builder-map`) — NOT a re-rendered `view_map`. Hand off to that skill rather than reconstructing the spec from scratch.
