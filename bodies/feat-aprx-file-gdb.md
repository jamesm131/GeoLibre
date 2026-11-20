## Summary

Importing an ArcGIS Pro project whose data lives in a File Geodatabase dropped every one of those layers with a "not loaded from projects" warning, even on the desktop app, which can already read `.gdb` feature classes through Add data → File geodatabase. Most real `.aprx` projects keep their data in a geodatabase, so the import usually came through nearly empty.

On the desktop build, the importer now reads accessible local File Geodatabase feature classes through the same sidecar path Add data uses. Each layer keeps its table-of-contents position, group, visibility, opacity and CIM symbol. Its features are embedded like an Add-data GDB layer, so saving and reopening the project keeps them. A layer is dropped with a warning when:

- its `.gdb` or feature class is missing ("no readable local data source");
- its coordinate system can't be detected (new warning, rather than drawing it in the wrong place);
- the sidecar can't start or the read fails (new warning).

Groups left empty are pruned. Imported layers now also keep ArcGIS’s stacking order: CIM lists layers top first while GeoLibre stores them bottom first, so the whole import used to come in upside down. If another project is opened while the geodatabase reads run, the import no longer replaces it. Browser and Mac App Store builds keep the existing "File Geodatabase … not loaded" warning, and network-share and geodatabase-raster warnings are unchanged. `GdbSource` now shares the sidecar read helpers in `lib/file-gdb.ts`.

Proved with: unit tests for the importer and loader (order, groups, style, missing GDB/feature class, unknown CRS, failed read, sidecar down, browser default); `npm run lint`, `npm run typecheck`, `npm run typecheck:tests`, and the frontend suite; and a UI run in the dev app against a live sidecar (details below).

## Testing

- `tests/arcgis-project-import.test.ts` (23 pass): placeholders keep order, group, style and the `.gdb` path. The loader fills readable layers, lists each geodatabase once and records the name it read. It drops a missing geodatabase or feature class, an unknown CRS, a failed read and an unstartable sidecar, each with its warning, and prunes emptied groups. The browser default still warns.
- `npm run lint` (0 errors, 240 warnings, limit 241), `npm run typecheck`, and `npm run typecheck:tests` (0 errors) pass. `npm run test:frontend` gives 14,648 passes and 1 failure, `stac-render-tiles`, which fails identically on `main`.
- UI run: `npm run dev` with the sidecar's real conversion runtime, and only the Tauri IPC layer (file dialog, fs reads, sidecar start) stubbed in the page, because this machine cannot drive a native window. Project → Import → Import ArcGIS Pro project… restored a 200,000-polygon feature class with its group, style, CRS and extent. Save as wrote the features into the project, and reopening restored them without the sidecar. An edge-case project (moved `.gdb`, missing feature class, `.gdb` raster) showed the expected warnings and pruned the emptied group. The plain browser build still shows the old "not loaded from projects" warning.

### Large feature class measurement

| | |
| --- | --- |
| Dataset | Synthetic File GDB, 1 feature class, 200,000 polygons, 4 attributes, EPSG:2193; 27 MB on disk, 95.9 MB as WGS84 GeoJSON |
| Machine / build | Apple M2 Max, 96 GB, macOS 27.0; Vite dev build in Chromium (T3 preview); sidecar from `backend/geolibre_server` with its managed DuckDB 2.0 conversion runtime, warm |
| First usable map | 19.5 s from clicking the menu item to a MapLibre `idle` with the layer drawn (18.3 s until the features reached the store) |
| Peak memory, page | 579 MB JS heap (295 MB before import; `performance.memory`, JS heap only, not GPU or renderer RSS) |
| Peak memory, sidecar | about 225 MB RSS for uvicorn plus its conversion subprocess (45 MB idle), sampled every 50 ms |

A Node script running the same importer and loader against the same sidecar (3 runs) breaks the time down: listing the geodatabase takes 1.3–1.6 s, the sidecar conversion 12–20 s, and reading and parsing the GeoJSON 1.1–3.2 s. Node's peak RSS was 520–567 MB. Almost all of the wait is the sidecar's existing GDB-to-GeoJSON conversion, the same step Add data → File geodatabase runs. This is a proxy, not a measurement of the packaged Tauri app in WKWebView: the desktop build could not be run here, because another local service holds the sidecar's fixed port 8765 and screen capture is unavailable.

## Related issue

None. Follow-ups found in review (not in this PR):

- Standalone `.mapx` files whose `mapDefinition` references document-level `layerDefinitions` by URI import no layers, for every layer type (pre-existing on `main`).
- Rasters and services are still added after the project loads, so they land above the vector layers rather than in their ArcGIS position.
- The new `en.json` strings fall back to English in other locales until the catalogs are filled.

## Council review

Rounds: 2

| Delegate | Round 1 | Round 2 |
| --- | --- | --- |
| GPT-6.2 Sol (medium) | 6/10 | 9/10 |
| GPT-6.2 Astra (medium) | 6/10 | 9/10 |
| GLM-5.3 (high) | 9/10 | 9/10 |

Round 1 found the reversed layer order and the superseded-import race (both fixed). The MAPX and raster-placement findings are pre-existing and listed as follow-ups.

Reviewed with sol-loop: 2 rounds with GPT-6.2 Sol on medium. Sol verdict: approved

Created with Claude Opus 5.5 in T3 Code (Claude Code harness). Reviewed by GPT-6.2 Sol, GPT-6.2 Astra and GLM-5.3.
