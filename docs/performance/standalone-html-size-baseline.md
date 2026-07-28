# Standalone HTML Size Measurement

## Scope

`MyTaskAssistant.html` is a self-contained offline application. This measurement records the impact of removing JavaScript declarations that were embedded inside the document's stylesheet while retaining the active equivalents in the primary script block.

## Method

- Raw: UTF-8 file bytes.
- Gzip: .NET `GZipStream` with `CompressionLevel.Optimal`.
- Brotli: Node.js `zlib.brotliCompressSync` with `BROTLI_PARAM_QUALITY=11`.

## Results

| Variant | SHA-256 | Raw bytes | Gzip bytes | Brotli bytes |
| --- | --- | ---: | ---: | ---: |
| Before | `A67AB398CCC6214A9EB1A0FF48A226487EA1581BA0D6FF85B4E16AD0F5E15F16` | 145,703 | 31,452 | 25,193 |
| After | `D8C905C45CA4B52A150A1349E6FD4411C79FF9C8C14B34E819A97D7C03238B1D` | 141,366 | 30,147 | 24,870 |
| Savings | — | 4,337 (2.98%) | 1,305 (4.15%) | 323 (1.28%) |

## Change Evidence

- Removed invalid stylesheet declarations for `firstNoteLink`, `taskHierarchyMarker`, and `renderDependencyIndicator`.
- Retained the single active declaration of each helper in the primary application `<script>` block.
- The removal changes no callable JavaScript implementation and prevents CSS parsing from encountering JavaScript tokens.

## Offline Regression Validation

Validated without network access or external application dependencies:

- The document has one primary head `<style>` block and one main application `<script>` block.
- The primary stylesheet contains no JavaScript function declarations.
- The main application script passed `new Function(...)` syntax parsing.
- All 95 named JavaScript function declarations in the main application script are unique.
- Final artifact sizes match the **After** row above.

This validation confirms the consolidation did not remove a callable helper from the offline application's executable script. Interactive browser workflows were not changed by this edit and should remain available when opening the HTML file locally.
