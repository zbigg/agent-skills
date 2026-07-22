---
name: carto-location-services
description: Call CARTO Location Data Services — geocoding, reverse geocoding, isolines (catchment areas), routing, and origin-destination matrices — synchronously and inline via the CARTO MCP server's LDS tools, for a handful of inputs needed right now in the conversation. Triggers on "geocode these few addresses", "show me where this address is", "locate this address on a map", "put this location / these few points on a map", "what's the address at these coordinates", "15-minute drive-time area around this point", "route / travel time from A to B", "small drive-time matrix between these sites". Provider is account-configured, never chosen by the agent. Distinct from carto-geocoding and carto-routing-od-analysis, which build CARTO Workflows for whole tables — use those, not this, for a column of addresses or a large batch.
license: MIT
---

# carto-location-services

Runs CARTO Location Data Services (LDS) — geocode, reverse-geocode, isolines, routing, OD matrix — **synchronously, inline in the chat**, via the CARTO MCP server's `lds_*` tools. Use it for a **handful of inputs the user needs answered now**: turn a few addresses into points, describe a coordinate, draw a catchment around a place, get a route/travel time, or compare a few sites with a small matrix.

**Tool contract.** This skill consumes the CARTO MCP server tools `lds_geocode`, `lds_reverse_geocode`, `lds_isolines`, `lds_routing`, `lds_od_matrix`, and `lds_capabilities`. Each tool's exact input shape lives in its own MCP description — read it via the host's tool-inspector or `tools/list`; this skill does not duplicate the spec. It focuses on *when* to use these tools vs. the Workflows path, and on the cross-cutting rules (provider, batching, size limits) that the per-tool descriptions don't spell out.

| Tool | Does | Key inputs (see the tool's own description for the full shape) |
|---|---|---|
| `lds_geocode` | Address(es) → coordinates | `addresses` (array — pass all in one call), `country?`, `limit?`, `options?` |
| `lds_reverse_geocode` | Coordinate → address | `lat`, `lon`, `lang?`, `options?` |
| `lds_isolines` | Catchment area around one origin | `origin` "lon,lat", `mode`, `range`, `range_type` (time\|distance) |
| `lds_routing` | Route + time/distance | `origin`/`destination` "lon,lat", `mode`, `waypoints?` |
| `lds_od_matrix` | Small origin→destination matrix | `origins`/`destinations` as `[lon,lat]` (TravelTime provider only) |
| `lds_capabilities` | Account's active provider per operation + remaining quota | none |

## Step 1 — detect what's available

| Check | How |
|---|---|
| LDS tools are callable | The `lds_*` tool names appear in your tool list. If not, the CARTO MCP server isn't attached — tell the user; don't fabricate coordinates. |
| The account can run the operation | Call `lds_capabilities` first when unsure. It returns the active provider per operation and remaining quota. Cheap, no LDS quota consumed. |

Unlike the map tools, these return JSON text (not a rendered widget), so there's no host-rendering requirement — any MCP host works.

## Core rules

- **Provider is account-configured — never selected by the agent.** Don't ask the user which provider or try to pass one; the account's configured provider (and its credentials) is used server-side. Advanced provider-specific parameters go through the tool's `options` passthrough.
- **Batch in ONE call.** `lds_geocode` takes an `addresses` array — pass all the addresses at once. Never loop the tool once per address; that's the exact anti-pattern this feature exists to remove.
- **Coordinate conventions (a real gotcha).** `origin`/`destination` for isolines and routing are `"lon,lat"` strings; matrix `origins`/`destinations` are `[lon, lat]` pairs; routing `waypoints` are the reverse — `"lat,lon:lat,lon"`. When in doubt, confirm against the tool's own description.
- **A few inputs → tool; a whole table → Workflow.** These tools are for inline, human-scale requests. To geocode a column of addresses, or run routing/isolines over a table, switch to the Workflows path (see below) — do not call the tool in a loop over many rows.
- **Oversize results.** An isoline or route can be large. If a result comes back with a `... (truncated)` marker, the geometry didn't fit the inline transport — tell the user and move the job to a Workflow rather than acting on the partial result.
- **OD matrix is provider-gated.** Synchronous matrices work only when the account's routing provider is TravelTime. Check `lds_capabilities` first; if it's another provider, tell the user and use the Workflows OD-matrix path instead of retrying.

## Putting a result on a map (optional)

`view_map` (from `carto-render-inline-map`) has no inline-GeoJSON layer — it renders CARTO sources only. **Counterintuitive but critical:** LDS hands you GeoJSON, yet feeding it to a generic deck.gl `GeoJsonLayer` does NOT work — `view_map` registers only CARTO tile layers, and an unregistered layer renders an **empty map with no error**. The GeoJSON's only path to the map is through SQL: **any** small LDS result — a geocoded point, a route line, an isoline polygon — is wrapped in a `vectorQuerySource` whose SQL constructs the geometry, against one of the user's connections, rendered by a `VectorTileLayer`.

**Honest tradeoff for a plain point.** For a single address lookup where a simple pin is enough and no CARTO connection is involved, a generic map display is lighter — no SQL, no warehouse round-trip. Use LDS + `view_map` when you specifically want the **account's configured geocoder**, or when the point will sit **alongside other CARTO layers** (an isoline, a warehouse-backed layer) on the same map.

- **Extract the coordinates/geometry first.**
  - **Geocode:** no GeoJSON in the response — plain numeric fields, keyed by address index: `data["0"].value` is the candidate list for the first address; take `value[0].longitude` / `value[0].latitude` for the top match.
  - **Routing:** `data.value.route` is a bare GeoJSON geometry — pass it straight through.
  - **Isolines:** a `FeatureCollection` with **one feature per requested range** — take `features[i].geometry`. Never pass a `Feature`/`FeatureCollection` to a SQL constructor — BigQuery and PostGIS reject them outright.
- **The geometry `type` is not fixed.** Routes may be `LineString` or `MultiLineString`; isolines `Polygon` or `MultiPolygon` — it changes with waypoints/fragmentation and with the account's provider. Every warehouse's GeoJSON constructor accepts all of these, so pass the geometry straight through; never assume the singular form or read `coordinates` as a single line/ring.
- **Use the connection's dialect constructor** (pick by the connection's provider from `list_connections`). Lines/polygons — the GeoJSON constructor: BigQuery `ST_GEOGFROMGEOJSON`, Snowflake `TO_GEOGRAPHY` (via `PARSE_JSON`), Databricks `st_geogfromgeojson` (needs DBR 17.1+ / SQL Pro or Serverless — not SQL Classic), Redshift `ST_GeomFromGeoJSON` (returns planar GEOMETRY), Oracle `SDO_UTIL.FROM_GEOJSON`, PostGIS `ST_GeomFromGeoJSON`. Geocoded **points** — the native point constructor is simplest: BigQuery `ST_GEOGPOINT(lon, lat)`, Snowflake `ST_MAKEPOINT(lon, lat)`, Databricks `st_point(lon, lat)`, Redshift `ST_Point(lon, lat)`, Oracle `SDO_GEOMETRY(2001, 4326, SDO_POINT_TYPE(lon, lat, NULL), NULL, NULL)`, PostGIS `ST_SetSRID(ST_MakePoint(lon, lat), 4326)` — or assemble a `{"type":"Point","coordinates":[lon,lat]}` literal for the same GeoJSON constructor as lines/polygons. **Watch the order:** geocode results name the fields `latitude`/`longitude`, but every constructor takes **lon first**. Full per-dialect caveats and verification queries: [`references/rendering-geometry.md`](references/rendering-geometry.md).

Worked example — "show me where 350 Fifth Ave is":

1. `lds_geocode` with `addresses: ["350 Fifth Ave, New York"]` → `data["0"].value[0]` → `longitude: -73.985, latitude: 40.7482`.
2. Render (BigQuery connection shown — swap the constructor per dialect):

```
view_map({ deckglProps: { layers: [{
  "@@type": "VectorTileLayer",
  "data": { "@@function": "vectorQuerySource",
            "connectionName": "<a CARTO connection>",
            "sqlQuery": "SELECT ST_GEOGPOINT(-73.985, 40.7482) AS geom" } }] } })
```

For a route/isoline, same shape with the GeoJSON constructor: `"sqlQuery": "SELECT <dialect GeoJSON constructor>('<bare geometry JSON>') AS geom"`.

This needs a CARTO connection and only works for geometry small enough to embed. Confirm with the user before rendering. For large geometry, go through the Workflows path (write results to a table, then map the table).

## When to pick a different skill

- **A whole column/table of addresses, or a large/scheduled job** → `carto-geocoding` (builds a Workflow with `native.geocode`).
- **Routing / isolines / OD matrix over a table, or OD-flow aggregation** → `carto-routing-od-analysis` (Workflows).
- **Render the geometry on a map** → `carto-render-inline-map` (`view_map`), per the pattern above.

## Anti-patterns to avoid

- **Calling `lds_geocode` once per address.** Pass the whole `addresses` array in a single call.
- **Selecting or asking for a provider.** It's account-configured; the agent can't change it.
- **Using these tools for a whole warehouse table.** That's the Workflows path (`carto-geocoding` / `carto-routing-od-analysis`).
- **Acting on a truncated result, or trying to map a huge isoline/route inline.** If it's truncated or large, move to Workflows.
- **Retrying `lds_od_matrix` on a non-TravelTime account.** Check `lds_capabilities`; if unsupported, use the Workflows OD-matrix path.
- **Passing a `Feature`/`FeatureCollection` (or the whole isolines response) into a SQL geometry constructor.** Extract the bare geometry first: `data.value.route` for routes, `features[i].geometry` for isolines.
- **Rendering LDS GeoJSON with a generic deck.gl layer (`GeoJsonLayer` etc.) in `view_map`.** Not registered — the map renders empty with no error. The geometry goes into `vectorQuerySource` SQL (see "Putting a result on a map").
- **Switching rendering technology when a `view_map` render comes back empty.** An empty map is almost always a layer↔source mismatch (a generic `GeoJsonLayer` instead of `VectorTileLayer` + `vectorQuerySource`) or a missing dialect constructor — NOT a reason to fall back to SVG/HTML or any non-CARTO renderer. Re-read "Putting a result on a map" and fix the layer/source, don't change the transport.
