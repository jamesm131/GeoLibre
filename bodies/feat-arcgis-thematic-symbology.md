Importing an ArcGIS Pro project (.aprx/.mapx) flattened every thematic layer to a single colour. A zoning map came in as one flat fill, and graduated colour maps lost their classes. Nothing told the user.

This PR adds a shared CIM renderer translator to `@geolibre/core` (`translateCimRenderer`) and uses it in the ArcGIS Pro project importer. The `.lyrx` style import (#3545) can call the same function once both land.

**What is reproduced**

- **Unique values (single field):** one rule per class, with the class's values, label, colour, order and outline. `<Null>` matches null and missing values only. Matching is exact and string-based, as ArcGIS stores it, so the number `3` matches a stored `"3"`. An unchecked class stays hidden.
- **Class breaks (graduated colour):** each class covers `(previous upper bound, upper bound]`, and the first covers `[minimumBreak, upper bound]`. That follows Pro's `≤24` labels, the JS API's max-inclusive breaks and arcpy's `lowerBound`. Only numeric values classify, so null is never read as 0.
- **"All other values" / out of range:** drawn with the default symbol only when ArcGIS draws it (`useDefaultSymbol`, `isDefaultSymbolVisible`), and hidden otherwise.

These become GeoLibre **rule-based** styles, so the map, legend and saved project all carry the same rules. The native categorised/graduated modes aren't used, and their behaviour is unchanged. Their lower-bound `step` would put every value that sits on a break in the wrong class, and it would draw nulls and out-of-range values in the first colour.

**What is reported instead of flattened**

- Renderers that can't be reproduced fall back to a single symbol and get a note in the import dialog. These are multi-field unique values, Arcade expressions, normalised or size-graduated breaks, exclusion clauses, unclassed colours, symbols without a solid colour, and other renderer types.
- Hatching, marker decorations, casings and visual variables beside a translated renderer get a "partly shown" note.
- The dialog no longer counts these notes, or the map-extent note, as layers that failed to load.

CIM details handled: properties left out at their default (an absent `upperBound`/`minimumBreak` is 0, and an absent `visible` beside written ones means hidden), and `CIMLABColor`.

**How I proved it**

- `tests/arcgis-cim-renderer.test.ts` runs the generated paint and visibility filter through MapLibre's own expression engine. It checks colours at each boundary, ±0.000001 either side, null, missing and out of range, plus outline widths, and that colours and legend rows survive `serializeProject` → `parseProject`.
- I ran the translator over 100 real Pro `.lyrx` files from public GitHub repositories. 38 of 39 class-breaks renderers translate (the remaining one, an unclassed-colours renderer, is reported). 40 unique-value renderers translate, 14 of them with a partial note. The rest are reported, mostly multi-field, Arcade, or classes with no solid colour.
- In the web app, I opened a project made by this importer, with zoning and density data embedded: the map and legend match the rules. The Project → Import ArcGIS Pro project dialog shows the new notes and the corrected count.
- `npm run typecheck`, `typecheck:tests`, lint and oxfmt pass, and the frontend suite passes apart from `tests/stac-render-tiles.test.ts`, which fails locally on a socket reset unrelated to this change.

**Not included (follow-ups):** legend swatches for outline-targeted colour renderers (none in the corpus), mixed partial outline opacities between classes, and group headings in the legend.

## Council review

Rounds: 3

| Delegate | Round 1 | Round 2 | Round 3 |
| --- | --- | --- | --- |
| GPT Sol 6.2 (medium) | 6/10 | 8/10 | 10/10 |
| GPT Astra 6.2 (medium) | 7/10 | 8/10 | 10/10 |
| GLM 5.3 (high) | 9/10 | 9/10 | not run (approved in round 2) |

Created with Claude Opus 5.5 in Claude Code (T3 Code). Reviewed by GPT Sol 6.2, GPT Astra 6.2 and GLM 5.3.
