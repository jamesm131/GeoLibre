Builds on #THEMATIC (thematic renderers) — review the commits after `dda90cf4`.

Follow-up to #3545, which added `.lyrx` style import with its own CIM reader: colours, symbol layers, markers and dashes, a second copy of what the ArcGIS Pro project importer does. #THEMATIC moves CIM symbology reading into one module, `packages/map/src/arcgis-cim-renderer.ts`. This PR rebuilds `packages/map/src/lyrx-import.ts` on it, so `.lyrx` and `.aprx` files give a layer the same look. `lyrx-import.ts` becomes an adapter (456 → 437 lines, most of it warning sentences and label reading) with no colour, symbol or marker reading of its own.

## Summary

What changes for users importing a `.lyrx` through **Layer actions → Styles → Import style from file…**, **Import style from text…** or the plugin API's `importLayerStyle`:

- **Unique values and graduated colours now import**, as rule-based styles with one rule per class, exactly as an `.aprx` import does (#THEMATIC explains the class semantics). Before, every renderer except single symbol was refused. Across 113 public ArcGIS Pro `.lyrx` files, **89 now import, up from 13** (all 13 still import with the same fill and outline colours).
- **Label size, colour and halo now import.** A label that is a single field (`[NAME]`, `$feature.NAME` or `$feature["NAME"]`) takes the text size, the text colour and the halo colour and width from the file's text symbol. The "label font, size, colour and halo were not imported" warning is now only about the font, which GeoLibre labels have no setting for.
- **The symbol replaces the layer's whole symbol.** A CIM symbol describes the symbol in full, so the import now resets the layer's dash, blur, line decorations, fill pattern, inverted fill, icon marker, heat map or cluster renderer, proportional sizing, outline transparency, widths, width unit and circle radius to GeoLibre's defaults before applying the file's values. Before, a layer's own dash, for example, could show through an imported solid line.
- **The imported label replaces the layer's data-defined label overrides.** Size and colour expressions are cleared where the file sets a size or colour, and opacity and visibility expressions are always cleared, since they would otherwise override or hide what the file shows.
- **Hatched fills and dashed lines follow the shared reader.** A polygon whose only fill is a hatch now imports as its outline (hollow), and a dashed line imports solid, each with a warning that says exactly that. So does a polygon with a readable outline and a fill colour that cannot be read (an HSL colour, a null channel). #3545 kept the layer's own fill or dash in those cases. One rule for `.aprx` and `.lyrx` is worth more than the difference, and the warnings now say what actually happens.
- **A symbol layer without an `enable` flag counts as drawn**, as the project importer reads it. #3545 treated an absent flag as off. ArcGIS Pro writes the flag on every symbol layer (all 3,753 symbol layers in the 113 files carry it), so for Pro's own files the two readings agree; only hand-written CIM is affected.

Unchanged: renderers GeoLibre cannot draw are refused by name and the layer keeps its style; everything left out is named in a warning; null colour channels are refused rather than read as black; non-circle markers are named; detection works from the content (`"type": "CIMLayerDocument"`).

## Review threads on #3545 this resolves

- **Duplicated `cimColor` / `hsvToRgb`.** Several threads on `lyrx-import.ts` asked for the CIM colour code copied from the APRX importer to be shared, and pointed out that the copies had already drifted (the APRX copy read `null` channels as black). The reply then was that it would move once the shared CIM translator landed. #THEMATIC is that translator and fixes the APRX side; this PR deletes the `.lyrx` copy.
- **`enable` handling contradicting the APRX reader.** One thread noted that `.lyrx` skipped symbol layers without `enable: true` while the APRX reader ignored the flag. Both now read it in one place, `cimSymbolLayerEnabled`, with absent meaning drawn (see above for why).
- **No real ArcGIS Pro fixtures.** The real-file run below checks the reader against 113 Pro exports rather than only hand-written CIM.

## How it works

`packages/map/src/lyrx-import.ts` is an adapter around `translateCimRenderer` / `translateCimSymbol` from `@geolibre/map/style-import`:

1. `isLyrxDocument` recognises the document; `parseLyrx` takes the first `CIMFeatureLayer*` in `layerDefinitions`.
2. `translateCimRenderer(layer.renderer)` returns `{ translated, style, issues }`. A refused renderer becomes a no-match with one sentence naming why; a translated one becomes the style, plus one sentence per issue code.
3. Labels are read from the first visible label class. The text and halo colours come from the text symbol's polygon symbols through `translateCimSymbol`, so labels use the same colour reader as everything else.
4. `applyLyrxImport` resets the whole-symbol fields listed above to `DEFAULT_LAYER_STYLE`, which is where the APRX importer starts too, then merges the file's style over the layer.

The first commit moves renderer and symbol reading onto the shared reader (with the reset, and unique values and class breaks); the second adds label size, colour and halo; the third keeps two warnings #3545 had (naming a picture or hatch fill in a refused symbol, and an unreadable label text symbol). This PR changes nothing in `arcgis-cim-renderer.ts`.

## Design decisions and known limits

- **An outline-only marker is refused.** #3545 imported a vector marker whose only graphic layer is an outline as a hollow circle. The shared reader refuses it (the layer keeps its style, and the warning says the point symbol has nothing GeoLibre can import), because the shared reader draws a marker as a filled circle and needs a fill colour for it rather than drawing an empty ring. The `.aprx` importer behaves the same way, and changing that belongs in the shared reader for both.
- **A symbol with nothing drawable names what it has.** When a symbol is refused because nothing in it can be drawn, a second warning names the layers it does use, such as a picture or hatch fill, as #3545 did.
- **A refused renderer is refused whole.** The shared reader still returns a refused renderer's default symbol (the APRX importer uses it to give a new layer some colour), but on an existing layer that would replace the user's style with a fraction of the file's, so the `.lyrx` import keeps the layer as it is.
- **The whole-symbol reset covers every geometry a field applies to.** On a layer that mixes geometries, a line symbol also resets the points' circle radius and the polygons' fill opacity and pattern, as replacing the symbol does in ArcGIS Pro. Diagrams and geometry generators are left alone: they draw beside the symbol rather than changing it. So is extrusion, which ArcGIS Pro also keeps on the layer rather than in the symbol. The plugin API docs say `.lyrx` is the exception to "fields it does not describe keep their values".
- **Label transparency is not imported.** `LabelStyle` colours are written to SLD and QML as hex, so the import keeps them hex and warns when the text or halo is partly transparent. A fully transparent halo imports as no halo, which is what ArcGIS Pro draws. Anything else the text or halo symbol draws that a GeoLibre label cannot (an outlined halo, outlined text, a hatch) is named in one warning; of the 113 real files, one has outlined label text.
- **Only the first feature layer** of a multi-layer file is read; the warning names it and says how many there were.
- **Warnings are English strings from the parser**, like the QML, SLD and Mapbox readers'. No `en.json` strings change.
- **`simpleStyleEnabled` is left alone**, as in the other three readers and as agreed on #3545.
- **Unique values on zero or several fields, Arcade expressions, normalised breaks, graduated symbols, exclusion clauses and renderers whose class symbols have no solid colour** are refused, as in #THEMATIC, because a partial translation would draw some classes wrongly.

## Testing

- `node --import tsx --test tests/lyrx-import.test.ts tests/style-import.test.ts tests/arcgis-cim-renderer.test.ts`: 155 passed, 0 failed. `tests/lyrx-import.test.ts` is table-driven: one row per refusal reason and per partial-import issue, checking the sentence each becomes, plus the whole-symbol reset, unique values and class breaks, the absent `enable` flag, and labels (field forms, off in the file, computed expressions, text size, colour, halo, font and filter). How symbols, colours and renderers are read is tested once, in `tests/arcgis-cim-renderer.test.ts`.
- `npx playwright test e2e/lyrx-style-import.spec.ts --project=features` (builds and serves the app): 1 passed. It picks a `.lyrx` in the file chooser, checks the rendered circle takes `#008000` with radius 8, and pastes a heat map `.lyrx` to confirm the dialog refuses it by name.
- `npm run typecheck:fast` passes; `npm run typecheck:tests` reports 0 errors. `npx eslint` on the changed source, test and e2e files: no errors. `git diff --check`: clean.
- **Real files.** `importStyleText` was run (outside the repo, files not committed) over 113 distinct ArcGIS Pro `.lyrx` files from public GitHub repositories, including Esri's `arcgis-pro-sdk-community-samples`, `USDA-NRCS/NRCS-Engineering-Tools---ArcGIS-Pro`, `ncss-tech/SSURGO-QA-ArcGIS-Pro` and `arc2qgis/lyrxtoqml`. None threw. 89 import (13 with #3545's reader). The 24 refused are: unique values on several fields (8), Arcade-driven symbology (4), unclassed colours (1), unique value renderers with a class drawn only by character or picture markers, which the shared reader refuses as a whole (10), and one empty file. The most common warnings are the label font (10 files), a second outline on a cased line (9) and dash or offset effects (9).

## Related PRs

- #3545 (merged) added the `.lyrx` import this PR rebuilds.
- #THEMATIC adds the shared CIM reader this PR is built on, and uses it for `.aprx` / `.mapx` imports.
- #3553 is the ArcGIS import foundation #THEMATIC builds on; this PR does not touch the project importer.
