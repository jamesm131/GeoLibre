## Summary

An ArcGIS Pro project remembers each layer's data folder as it was on the machine that saved it, often a Windows drive (`C:\GIS\Data`) or a network share. Open that project anywhere else and its shapefiles cannot be found. Today GeoLibre Desktop adds those layers and then drops them during the background reload, with only a toast to say so.

Now the import report lists the missing layers by the folder they were saved in. Each folder has a **Locate folder…** button. Pick the folder where the data now lives, and every layer from that folder whose file is there loads back with its original name, style, group, order and visibility. Layers whose files are not there stay in the report.

- It reads only the folder you pick. It never searches the disk, and it never changes the `.aprx`.
- Picking a folder only repairs the workspace you picked it for. If another workspace has the same folder name (say `D:\Other\Data` beside `C:\GIS\Data`), it is not matched, even if the picked folder holds a file with the right name.
- Windows, UNC and relative paths are resolved as ArcGIS Pro does. Folders compare without case on Windows and UNC paths, with case on POSIX paths. A file name that differs only in case still matches on a case-sensitive disk. `\Data` resolves on the project's drive or share. A relative path in a project opened from a network share resolves on that share. `..` stops at a drive or share root. Network shares are still never read directly, but their layers can now be located.
- Two layers that use the same file keep their own names and styles.
- Layers that loaded are left alone. Cancelling the picker changes nothing. Locating again never adds a layer twice. A network-share destination is refused, because a saved project cannot reopen layers from one.
- After a repair, Save stores the new paths, and reopening the project reads the layers from there.
- The browser build is unchanged: it keeps its existing "use GeoLibre Desktop" warning.

This covers folder-based sources (shapefiles, delimited text, other single files in a folder). Moved File Geodatabase workspaces will be handled in a follow-up that builds on the File Geodatabase reader once it lands.

Follow-ups (not in this PR):

- Locate for moved File Geodatabase workspaces, once the File Geodatabase reader lands.
- A folder name containing a dot (`2026.11`) is still read as a file extension, so those layers get a "format" warning and no Locate. This is unchanged from before this PR.
- GeoPackage and dataset-only connections are not checked before the project loads.
- A located layer is placed only among the vector and tiled layers. It is not ordered against rasters or services added after the project loads.

## Testing

Proved with new unit tests, the repo checks, and an end-to-end run in the real Tauri desktop app.

- New `tests/arcgis-workspace-repair.test.ts` covers grouping and paths (Windows, UNC, mixed separators, relative, drive and share roots), restore order, groups, visibility and styles, duplicate references, kept names, retries, network shares, and a save and reopen through `serializeProject`/`parseProject`.
- `npm run build`, `npm run lint` (0 errors), `npm run typecheck:tests` (0 errors) and oxfmt all pass. `npm run test:frontend` passes 14660 of 14663. The one failure is `tests/stac-render-tiles.test.ts`, which fails with a local-server `ECONNRESET` both with and without this change.
- Desktop app (`tauri dev`). A moved `.aprx` held a `C:\GIS\GISData` workspace with Roads, a second, hidden reference to Roads with a different colour, Parcels and Rivers. The moved folder held `roads.shp` and `Parcels.shp` (different case from the project) but no rivers, plus an unrelated `lakes.shp` that belongs to `D:\Other\GISData`.
  - One pick restored Roads, the hidden reference and Parcels into the Transport group, each with its own style and visibility.
  - Rivers stayed reported, and Lakes was not touched.
  - Cancel, a network-share pick and a second Locate each changed nothing.
  - Save (file references) wrote the new paths without embedded features. Reopening re-read all four layers from the new folder.
- Browser build: the same project still shows its existing "use GeoLibre Desktop" warning, with no Locate button.

## Related issue

Part of the ArcGIS Pro project import work (follows #3544).

## Council review

Rounds: 4

| Delegate | Round 1 | Round 2 | Round 3 | Round 4 |
| --- | --- | --- | --- | --- |
| GPT-6.2 Sol | 6/10 | 8/10 | 9/10 | 10/10 |
| GPT-6.2 Astra | 7/10 | 8/10 | 8/10 | 10/10 |
| GLM-5.3 (OpenCode) | 9/10 | 9/10 | 9/10 | approved in round 3, not re-run |

Created with Claude Opus 5.5 in Claude Code (T3 Code). Reviewed by GPT-6.2 Sol and GPT-6.2 Astra (Codex) and GLM-5.3 (OpenCode).
