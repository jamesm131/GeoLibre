## Summary

In the desktop build, **Project → Import → Import ArcGIS Pro Project…** now loads feature classes stored in a local File Geodatabase (`.gdb`) instead of listing them as unsupported. Each one keeps its group, its simple symbol and its place in the drawing order (subject to the nested-group limit below). Its features are embedded in the GeoLibre project, as they are for layers added through **Add Data → File Geodatabase (GDB)**, so the project reopens without the geodatabase.

When the project holds several 2D maps, the map picker from #3544 still comes first, and only the chosen map's geodatabase layers are read.

A feature class that cannot be loaded is skipped, with a warning in the import summary:

- the geodatabase opens but no longer holds the feature class: `missing-source`
- the geodatabase cannot be listed (moved, deleted, sidecar busy or timed out), the feature class cannot be read, or the sidecar cannot start: `file-geodatabase-read`. Its text names both a moved geodatabase and the processing server failing to start or being busy; the exact cause is logged to the console. A failed listing does not say why it failed, so it is not reported as "missing".
- its coordinate system cannot be detected: `file-geodatabase-crs` (the warning points to Add Data → File Geodatabase, where a source CRS can be entered)

Groups emptied by skipped feature classes are removed before the project loads. The browser and Mac App Store builds behave as before: they cannot run the Python sidecar, so geodatabase feature classes are still listed as `file-geodatabase`.

## How it works

- `readArcgisProject` / `importArcgisProject` take a new `readFileGeodatabase` option. With it set, a feature layer whose workspace is a `.gdb` becomes an empty GeoJSON placeholder in `project.layers` (grouped, styled, and in the import's `layerOrder`) and an entry in a new `fileGeodatabaseLayers` list. Without it (the default), the importer reports `file-geodatabase` exactly as before. A geodatabase raster is still reported as `file-geodatabase-raster` either way.
- `useProjectFileActions` sets the option only when `isTauri() && !IS_MAS_BUILD`, the same guard the Add Data GDB source uses.
- After the map is chosen and before the project loads, `loadArcgisFileGeodatabaseLayers` (new, `lib/arcgis-gdb-import.ts`) starts the sidecar, lists each geodatabase once (feature classes in the same `.gdb` share one listing) and reads each feature class to WGS84 GeoJSON. Readable layers get their features, `sourceCrs`, `featureCount` and the listing's spelling of the feature-class name. The rest are removed with `dropArcgisImportLayers` (from #3553), which also takes them out of `layerOrder` and prunes groups they leave empty, and a warning says why. A progress toast ("Reading N File Geodatabase layers…") shows while the reads run. It has its own dedupe key, so an abandoned import that finishes late cannot dismiss a newer import's identical notice.
- Drawing order and supersession come from the shared import base on `main` (#3553). The placeholders are ordinary project layers, so they enter the hook's import-id → store-id map with the rest of `project.layers`, and the import's single `applyLayerOrder` call places them among rasters, vector-tile services and hosted feature services. The reads take the import session's `signal`, checked before the sidecar starts, between polls of each sidecar job, after each listing and after each read: if the user opens another project, no further geodatabase is listed or read, the loader rejects (even when the last read finished after the switch), and the import stops without loading.
- The sidecar calls (list layers, convert one layer, poll the job, clean up the temporary GeoJSON) moved from `GdbSource.tsx` into `lib/file-gdb.ts` (first commit), so the Add Data source and the importer share one implementation. The reader is injected into `loadArcgisFileGeodatabaseLayers`, so unit tests stub the sidecar.

### Add Data → File Geodatabase is unchanged

The `GdbSource.tsx` change is a move, not a rewrite: `listFileGeodatabaseLayers` and `readFileGeodatabaseLayer` contain the same job calls, the same filter (attribute-only tables are dropped), the same 1 s poll and 10 min timeout, the same `source_srs` rule (sent only when the layer declares no CRS), and the same temporary-file clean-up gated on the job having finished. Error text uses the same `addData.gdb.*` keys, now through `i18next.t` because the helpers live outside a component.

## Design decisions and known limits

- **Reads happen before the project loads,** not after. A layer that cannot be read is dropped rather than left as an empty layer, and its now-empty groups are pruned before anything reaches the store. The cost is that the import waits for the reads; the progress toast says so.
- **Reads are sequential.** Each layer is a separate sidecar job, and the sidecar refuses new jobs with HTTP 429 once eight are in flight (`MAX_IN_FLIGHT_JOBS` in `backend/geolibre_server/geolibre_server/app/conversion.py`). Reading one at a time means this import never causes a burst of its own. Other tools running at the same time share the limit and can still cause a 429; that layer is then reported as `file-geodatabase-read` and the rest of the import continues.
- **A sidecar job already running is not cancelled** when another project opens. The import stops waiting for it at once (the poll loop observes the signal) and reads nothing further, but the job in flight finishes on the sidecar. For an abandoned read, polling carries on in the background and removes the temporary file once the job finishes. Clean-up needs the job to be seen finishing: if polling times out or the request fails first, the temporary file is left for the operating system's temporary-file clean-up, as in the Add Data source. The sidecar has no cancel endpoint for conversion jobs.
- **Nested groups.** The group-contiguity limit from #3553 applies here too: for `Outer[Top, Inner[Middle], Bottom]` the import computes `[Bottom, Middle, Top]`, but the store keeps `Outer`'s own layers in one block, giving `[Bottom, Top, Middle]`. Changing that store rule is out of scope.
- **Feature-class name matching.** GDAL's OpenFileGDB driver keeps the feature class's case, while ArcGIS stores the dataset name as typed in the connection, so the name is matched exactly first and then case-insensitively. The listing's spelling is what gets read and saved.
- **No CRS, no layer.** A feature class with no detectable CRS is refused with a warning, the same rule the Add Data GDB source applies, rather than drawing raw projected coordinates as longitude and latitude.
- **Saved like an Add Data GDB layer.** The placeholder carries `metadata.sourceKind: "gdb"` and `gdbLayer`, so saving embeds the features rather than recording a `.gdb` path that the reopen code cannot read.
- **Network-share geodatabases** stay refused as `network-path`, the importer's existing rule for UNC paths.
- **Only local `.gdb` folders are read.** The path comes from the project file, and the desktop sidecar does not confine reads (`GEOLIBRE_CONVERSION_ROOTS` is unset there). A workspace that is not an absolute local folder named `*.gdb`, such as a URL, a GDAL `/vsi*` path or a plain folder, is listed as `file-geodatabase` and never sent to the sidecar. Any local geodatabase the map names is read, wherever it is on disk, as with the importer's other local sources; the user guide says so and suggests checking the Layers panel before saving a project imported from someone else's `.aprx`.
- **Unsaved attribute drafts are asked about first.** When the chosen map has geodatabase layers, the discard question comes before the reads rather than after them, and is asked again only if the drafts changed while the reads ran.
- **Existing `file-geodatabase` warning text is unchanged.** It still suggests Add Data → File Geodatabase in GeoLibre Desktop. The Mac App Store build hides that source, so the hint does not apply there. The string comes from `main` and is out of scope here.

## Changes from the earlier draft of this branch

This branch was rebuilt on the shared ArcGIS import base from #3553 and is now rebased onto `main`. The base provides what this branch used to carry itself: `ImportContext`, `pruneArcgisImportGroups`, bottom-first layer storage, the `layerOrder` + `applyLayerOrder` reorder (replacing `arcgisImportStoreOrder` and the per-layer `moveLayer` loop) and the import session (replacing the `projectGeneration` guards and `removeLayerFromReplacedImport`). Because loaders now check the session's signal before inserting anything, a raster or service that finishes after a project switch never reaches the replacement, and the import never removes a layer by id after a switch. That closes three issues found in review of the earlier draft:

- A late service left the replacement dirty with an extra undo step. #3553's hook test opens a project while a service loads and checks layers, dirty flag and history.
- A collaboration refresh could get a retained raster deleted. The session watches `projectLoadCount`, which a collaboration snapshot does not change, and nothing is removed by id after a switch. #3553's hook test covers a collaboration refresh with services; there is no raster-specific test.
- A superseded non-tiled raster could be left in the replacement. `addRasterToMap` checks the signal before inserting, and its failure clean-up is skipped once cancelled (`tests/component-raster-add-signal.test.ts`, `tests/component-cog-raster-handler.test.ts`).

The earlier draft's race test polled without a bound; it is gone, and the hook tests use the harness's bounded `waitFor`.

## Testing

- `node --import tsx --test tests/arcgis-project-import.test.ts`: 40 passed. New cases:
  - feature classes kept as ordered, grouped, styled placeholders when the option is set, and still reported as `file-geodatabase` by default
  - readable layers filled, a missing geodatabase dropped with its empty group, one listing per geodatabase, case-insensitive name match
  - a layer with no CRS and a failed read each warned and dropped; a sidecar that cannot start reports every layer as unreadable without listing or reading
  - a two-map `.aprx` where each map uses a different geodatabase: choosing the second map queues and lists only its geodatabase, and the option also reaches `importArcgisProject`
  - a map with geodatabase layers, a hosted FeatureServer layer, a grouped raster and vector-tile service, and a file layer: one `layerOrder` across all of them; an unreadable feature class leaves the order with its layer; then, as the hook does, the project is loaded into the store, the raster and services are added on top under new ids in a different order, and one `applyLayerOrder` call gives ArcGIS Pro's order
  - abandoning the import before the reads, during the only read (whether it then succeeds or fails), or during an earlier read rejects without filling any layer
  - a failed geodatabase listing is reported as `file-geodatabase-read`, not `missing-source`
  - an abandoned listing is not logged as a failed read
  - only an absolute local folder whose name ends in `.gdb` is read; a plain folder, a URL, a GDAL `/vsicurl/` or `/vsizip/` path and a path with a query stay listed as `file-geodatabase` and never reach the sidecar
- `node --import tsx --test tests/component-arcgis-project-import.test.ts`: 12 passed. Six new cases run the real hook as the desktop build does, with Tauri's IPC (file dialog, file reads, sidecar start) and the sidecar's HTTP API stubbed:
  - a project with a geodatabase layer, a geodatabase layer whose feature class is gone (in its own group) and a vector-tile service: the readable layer loads with its features and ends up above the service, as in ArcGIS Pro, although the service was added after it; the missing one is reported and its group removed, and the sidecar sees one listing and one read
  - another project opened while the geodatabase is being listed: the replacement keeps no imported layer, is not marked changed, gains no undo step and shows no import report, and the feature class is never read. This test fails if the signal check after the listing is removed.
  - another project opened while the listing job is still running on the sidecar: the import settles straight away rather than waiting out the job (without the signal in the poll loop, this test hangs)
  - another project opened while a read job is running: the import settles, the temporary file is left alone while the job writes it, and is removed once the job finishes
  - with unsaved attribute table drafts open, the discard question comes before any sidecar call; cancelling it leaves the project as it was with no listing or read, and agreeing asks only once
- The import base's tests (`tests/apply-layer-order.test.ts`, `tests/project-import-session.test.ts`, `tests/layer-groups.test.ts`, `tests/arcgis-project-feature-service-desktop.test.ts`, `tests/component-raster-add-signal.test.ts`, `tests/component-cog-raster-handler.test.ts`), `tests/component-shared-save-back.test.ts` and `tests/i18n*.test.ts` all pass.
- `npx eslint` on the changed files, `npm run typecheck:fast`, `npm run typecheck`, `npm run typecheck:tests` (0 errors), `git diff --check`: all clean. The full `npm run test:frontend` suite passes locally after the rebase except one unrelated case in `tests/stac-render-tiles.test.ts`, which fails on macOS with a localhost `ECONNRESET` from its test server; this branch does not touch STAC code. No sidecar code changed, so the backend suite is unaffected. `oxfmt@0.59.0 --check` is clean on the changed code and tests; `docs/features.md` and `docs/user-guide/projects.md` were already unformatted on `main` and are left as they were.

The CIM fixtures are hand-written. The `workspaceFactory: "FileGDB"`, `DATABASE=...` connection string and `dataset` keys match the File Geodatabase data connections already handled on `main`.

## Related PRs

- #3553 (merged) added the shared ArcGIS import base this branch uses: one layer-order step, the import session and group pruning. It also moved the hosted feature layer import from #3548 (merged) onto that base, so this branch's geodatabase layers share its single layer-order step.
- #3544 (merged) added the map picker that runs before the geodatabase reads.
- Two follow-ups, the thematic renderer import (`feat/arcgis-thematic-symbology`) and locating moved folders (`feat/aprx-repair-workspaces`), also edit `arcgis-project-import.ts` and `useProjectFileActions.ts`. Whichever of these three merges later will need a small rebase.
