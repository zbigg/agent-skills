# Rendering LDS geometry on `view_map` — per-warehouse reference

> **This is a reference, not a standalone skill.** Read alongside the `carto-location-services` `SKILL.md` when embedding an LDS route/isoline geometry into a `vectorQuerySource`. The main skill carries the flow; this file carries the per-dialect constructor table, caveats, and verification queries.

## Extract the coordinates/geometry first

Every GeoJSON constructor below wants a **bare GeoJSON geometry object** (`Point`, `LineString`, `MultiLineString`, `Polygon`, `MultiPolygon`, …). BigQuery and PostGIS additionally **reject** `Feature`/`FeatureCollection` outright. The LDS operations hand you the location data in different shapes:

- **Geocode** — **no GeoJSON at all**: plain numeric fields, keyed by address index (verified against the live tool, Jul 2026). `data["0"].value` is the candidate list for the first address, `data["1"].value` for the second, and so on; each candidate carries `latitude`, `longitude`, address components, and `matchConfidence`. Take `value[0].longitude` / `value[0].latitude` for the top match and feed them to a point constructor (below) — remember the constructors take **lon first**.
- **Routing** — `data.value.route` **is already a bare geometry**. Pass it straight through.
- **Isolines** — the response is a `FeatureCollection` with **one feature per requested range**. Extract `features[i].geometry` for the range(s) you're rendering — never pass the collection or a feature.

(`lds_od_matrix` returns tabular travel-time/distance, not geometry — out of scope here.)

## The geometry `type` is not fixed

- Routing `route.type`: `LineString` **or** `MultiLineString`.
- Isoline geometry: `Polygon` **or** `MultiPolygon`.

It varies along two independent axes, so a single account hits both even with one provider:

1. **Waypoints / fragmentation.** On the same provider (observed on TomTom), A→B returns `LineString`; adding any waypoint returns `MultiLineString`. A contiguous catchment comes back `Polygon`, a fragmented one `MultiPolygon`.
2. **Provider.** The same single-leg A→B request returned `LineString` on TomTom/HERE but `MultiLineString` on TravelTime; the same 20-minute isoline was a fragmented `MultiPolygon` under one provider config and a single `Polygon` under another.

All constructors below ingest the multi- variants natively, so no type-branching is needed — as long as the SQL uses the GeoJSON geometry constructor (not a `LineString`/`Polygon`-specific one) and the code never reads `coordinates` assuming a single line/ring.

## Per-dialect GeoJSON → geometry constructor

Verified against official docs, Jul 2026. All six accept a bare GeoJSON geometry, support the Multi\* variants, and default the SRID to **4326** (matching LDS output — no reprojection needed).

| Warehouse | Function | Returns | Key caveats |
|---|---|---|---|
| BigQuery | `ST_GEOGFROMGEOJSON(json)` | GEOGRAPHY | Geometry fragments only — errors on `Feature`/`FeatureCollection`. RFC 7946; positions must be 2-element (no Z). |
| Snowflake | `TO_GEOGRAPHY(json)` / `TRY_TO_GEOGRAPHY(json)` | GEOGRAPHY | GeoJSON arg should be an `OBJECT` — wrap a string with `PARSE_JSON`. `TRY_` variant returns `NULL` instead of erroring. `TO_GEOMETRY`/`TRY_TO_GEOMETRY` exist for planar GEOMETRY. |
| Databricks | `st_geogfromgeojson(json)` (GEOGRAPHY) / `st_geomfromgeojson(json)` (GEOMETRY 4326) | GEOGRAPHY / GEOMETRY | **Requires Databricks Runtime 17.1+ / SQL Pro or Serverless — NOT available on SQL Classic warehouses.** Newest of the six; version-gate before relying on it. |
| Redshift | `ST_GeomFromGeoJSON(varchar_or_super)` | GEOMETRY (SRID 4326) | Produces planar `GEOMETRY`, not `GEOGRAPHY` — no direct GeoJSON→GEOGRAPHY constructor; cast/transform if a geography is needed. |
| Oracle | `SDO_UTIL.FROM_GEOJSON(json [, crs, srid])` | SDO_GEOMETRY | Overloads accept `VARCHAR2` or `CLOB`; default SRID 4326. Converts a geometry object, not a full GeoJSON doc. Idiosyncratic vs. the others — test the exact call. |
| PostGIS | `ST_GeomFromGeoJSON(json)` | geometry | Geometry fragments only (errors on full doc). SRID defaults to 4326 since PostGIS 3.0. |

All six GeoJSON constructors also accept a `Point` geometry (`{"type":"Point","coordinates":[lon,lat]}`), so one pattern covers every LDS shape. For geocoded points, though, the native point constructor is simpler — the geocode response gives you two numbers, not a GeoJSON object:

| Warehouse | Native point constructor (lon first) |
|---|---|
| BigQuery | `ST_GEOGPOINT(lon, lat)` |
| Snowflake | `ST_MAKEPOINT(lon, lat)` |
| Databricks | `st_point(lon, lat)` |
| Redshift | `ST_Point(lon, lat)` |
| Oracle | `SDO_GEOMETRY(2001, 4326, SDO_POINT_TYPE(lon, lat, NULL), NULL, NULL)` |
| PostGIS | `ST_SetSRID(ST_MakePoint(lon, lat), 4326)` |

The SRID-4326 default and bare-geometry-only caveats above apply to points identically.

## Verification queries

One Multi\* round-trip plus one native-point round-trip per dialect — the singular line/polygon forms are trivially covered if the Multi\* checks pass. The two most worth actually running are **Databricks** (version gate) and **Oracle** (idiosyncratic call/return handling and polygon ring-orientation expectations).

```sql
-- BigQuery
SELECT ST_GEOGFROMGEOJSON('{"type":"MultiLineString","coordinates":[[[0,0],[1,1]],[[2,2],[3,3]]]}');
SELECT ST_GEOGFROMGEOJSON('{"type":"MultiPolygon","coordinates":[[[[0,0],[0,1],[1,1],[0,0]]],[[[2,2],[2,3],[3,3],[2,2]]]]}');
SELECT ST_GEOGPOINT(0, 0);

-- Snowflake
SELECT TO_GEOGRAPHY(PARSE_JSON('{"type":"MultiLineString","coordinates":[[[0,0],[1,1]],[[2,2],[3,3]]]}'));
SELECT TO_GEOGRAPHY(PARSE_JSON('{"type":"MultiPolygon","coordinates":[[[[0,0],[0,1],[1,1],[0,0]]],[[[2,2],[2,3],[3,3],[2,2]]]]}'));
SELECT ST_MAKEPOINT(0, 0);

-- Databricks (DBR 17.1+ / SQL Pro or Serverless)
SELECT st_asewkt(st_geogfromgeojson('{"type":"MultiLineString","coordinates":[[[0,0],[1,1]],[[2,2],[3,3]]]}'));
SELECT st_asewkt(st_geogfromgeojson('{"type":"MultiPolygon","coordinates":[[[[0,0],[0,1],[1,1],[0,0]]],[[[2,2],[2,3],[3,3],[2,2]]]]}'));
SELECT st_asewkt(st_point(0, 0));

-- Redshift
SELECT ST_AsText(ST_GeomFromGeoJSON('{"type":"MultiLineString","coordinates":[[[0,0],[1,1]],[[2,2],[3,3]]]}'));
SELECT ST_AsText(ST_GeomFromGeoJSON('{"type":"MultiPolygon","coordinates":[[[[0,0],[0,1],[1,1],[0,0]]],[[[2,2],[2,3],[3,3],[2,2]]]]}'));
SELECT ST_AsText(ST_Point(0, 0));

-- Oracle
SELECT SDO_UTIL.TO_WKTGEOMETRY(SDO_UTIL.FROM_GEOJSON('{"type":"MultiLineString","coordinates":[[[0,0],[1,1]],[[2,2],[3,3]]]}')) FROM dual;
SELECT SDO_UTIL.TO_WKTGEOMETRY(SDO_UTIL.FROM_GEOJSON('{"type":"MultiPolygon","coordinates":[[[[0,0],[0,1],[1,1],[0,0]]],[[[2,2],[2,3],[3,3],[2,2]]]]}')) FROM dual;
SELECT SDO_UTIL.TO_WKTGEOMETRY(SDO_GEOMETRY(2001, 4326, SDO_POINT_TYPE(0, 0, NULL), NULL, NULL)) FROM dual;

-- PostGIS
SELECT ST_AsText(ST_GeomFromGeoJSON('{"type":"MultiLineString","coordinates":[[[0,0],[1,1]],[[2,2],[3,3]]]}'));
SELECT ST_AsText(ST_GeomFromGeoJSON('{"type":"MultiPolygon","coordinates":[[[[0,0],[0,1],[1,1],[0,0]]],[[[2,2],[2,3],[3,3],[2,2]]]]}'));
SELECT ST_AsText(ST_SetSRID(ST_MakePoint(0, 0), 4326));
```

Point via the GeoJSON constructor is covered by the same functions — e.g. `SELECT ST_GEOGFROMGEOJSON('{"type":"Point","coordinates":[0,0]}');` on BigQuery.

Expected: a valid 2-part MULTILINESTRING / 2-part MULTIPOLYGON / POINT in every case. BigQuery is verified end-to-end (routing `LineString`/`MultiLineString` up to a 2,862 km multi-leg route, isoline `Polygon`/`MultiPolygon`).

## Sources (official docs, checked Jul 2026)

- BigQuery `ST_GEOGFROMGEOJSON`: https://docs.cloud.google.com/bigquery/docs/reference/standard-sql/geography_functions
- Snowflake `TO_GEOGRAPHY` / `TRY_TO_GEOGRAPHY`: https://docs.snowflake.com/en/sql-reference/functions/to_geography
- Databricks `st_geomfromgeojson` / `st_geogfromgeojson`: https://docs.databricks.com/aws/en/sql/language-manual/functions/st_geomfromgeojson
- Redshift `ST_GeomFromGeoJSON`: https://docs.aws.amazon.com/redshift/latest/dg/ST_GeomFromGeoJSON-function.html
- Oracle `SDO_UTIL.FROM_GEOJSON`: https://docs.oracle.com/en/database/oracle/oracle-database/19/spatl/SDO_UTIL-reference.html
- PostGIS `ST_GeomFromGeoJSON`: https://postgis.net/docs/ST_GeomFromGeoJSON.html
