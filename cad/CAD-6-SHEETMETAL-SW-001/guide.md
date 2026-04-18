# Guide: Solid-to-Sheet-Metal Conversion

## Tribal Knowledge and Constraints

1. **Non-developable transitions require segmented approximation.** Smooth transitions between cross-sections of different shape (e.g., rectangular to circular) cannot be unrolled into a flat blank. Use a segmented (gore/facet) approximation that breaks curved sections into planar facets that can be developed.

2. **Minimum number of segments per curved section.** Fewer segments degrade flow performance by creating sharp transitions. Use enough segments that each facet closely approximates the curve — typically 8 minimum, 12 preferred.

3. **Seam location matters.** Place longitudinal weld seams on the low-visibility, low-stress side, never where they would obstruct function or create a flow disturbance. For cooling ducts, the bottom face is typical; for structural parts, the lowest-stress side.

4. **K-factor depends on material, thickness, and bend radius.** Use the bend table for the specified material and gauge if available. A single nominal K-factor value is only valid for a narrow radius range; re-check when bend radius deviates.

5. **Relief cuts: use obround (elongated slot) rather than rectangular.** Obround reliefs produce cleaner bends with less tearing. Minimum relief width must be at least the material thickness.

6. **Minimum bend radius depends on material and coating.** For coated sheet (galvanized, painted), bend radii below the material minimum risk cracking the coating on the outside of the bend. Always bend above the published minimum for the material and thickness.

7. **Features that break developability should be modeled as separate parts.** Internal dividers, baffles, or branches that would prevent the main body from unrolling should be modeled as separate pieces and joined at assembly (welded, riveted, spot-welded).

8. **Check grain direction on the flat pattern.** Bends perpendicular to the rolling grain are preferred for most sheet materials. Orient the flat pattern so the majority of bends cross the grain. Annotate the assumed grain direction on the DXF.

9. **Flat pattern nesting orientation.** Orient the flat pattern to minimize the bounding rectangle of the blank. This reduces material waste during laser cutting or punching.

10. **Number bends in forming sequence order.** Inner bends must be formed before outer bends that would obstruct tooling access. Mark the sequence clearly on the flat pattern (BEND-1, BEND-2, etc.).

11. **Use tab-and-slot features at closure seams.** Small tabs and matching slots (sized for the material thickness) provide self-locating alignment during assembly before welding. This is standard practice for fabricated sheet metal.

12. **DXF layer conventions.** Export with at least three distinct layers: CUT (outline and holes), BEND (bend lines with direction arrows), and NOTES (forming sequence, grain direction, part number). Use different colors per layer for clarity.

13. **Tolerance on developed (flat) length.** Establish a tolerance (typically +/-0.5mm for sheet metal) and verify by comparing total developed length of each face strip against the source solid model's surface lengths.

14. **Mark bend direction (up/down) on the flat pattern.** Convention: mountain folds (away from viewer) marked with solid lines, valley folds (toward viewer) marked with dashed lines. Include a legend on the drawing.

15. **Corner relief must not create stress risers.** Use a radius relief (minimum 0.5mm) rather than sharp-corner rectangular reliefs at corner intersections, to prevent fatigue cracking during service loading or thermal cycling.

## SolidWorks API References (C#)

Sheet-metal automation from `SolidWorks.Interop.sldworks`. The base feature type is `ISheetMetalFeatureData`; flat pattern is `IFlatPatternFeatureData`.

- **Rule 1, 2 (segmented gore for non-developable transitions):** `IFeatureManager.InsertLoftedBend2(thickness, bendAllowanceType, kFactor, direction, facets, autoProp, mergeFaceEdges, facetPercent)` — facet count / facet-percent controls the segmentation.
- **Rule 3 (seam on specific face):** `ISheetMetalFeatureData.SeamFaceID` / sketch-driven seam via `IFeatureManager.InsertRip`.
- **Rule 4 (K-factor, bend table):** `ISheetMetalFeatureData.KFactor` or `.BendTableFilePath`; set bend-allowance type via `ISheetMetalFeatureData.BendAllowanceType` (`swSheetMetalBendAllowance_e`).
- **Rule 5 (relief cuts):** `ISheetMetalFeatureData.AutoReliefType` (`swSheetMetalReliefType_e.swSheetMetalReliefObround`), `.AutoReliefRatio` (relief width/material thickness).
- **Rule 6 (min bend radius):** `ISheetMetalFeatureData.DefaultBendRadius`.
- **Rule 7 (non-developable features as separate bodies):** create second body via `IFeatureManager.FeatureExtrusion3` then use `IFeatureManager.InsertCombineFeature` only if needed (usually kept separate).
- **Rule 8 (grain direction on flat pattern):** `IFlatPatternFeatureData.GrainDirection` — a sketch vector; set via `ISketchManager` before flatten.
- **Rule 9 (nesting):** `IFlatPatternFeatureData.FlatPatternOrientation` — rotate the flat pattern to minimize bounding box.
- **Rule 10 (bend order numbering):** iterate bends via `ISheetMetalFeatureData.GetBendSequence` / `SetBendSequence`.
- **Rule 11 (tab-and-slot):** `IFeatureManager.InsertTabAndSlot` (or extrusion cut for custom geometry).
- **Rule 12 (DXF layer export):** `IModelDocExtension.SaveAs3(path, swSaveAsVersion_e.swSaveAsCurrentVersion, swSaveAsOptions_e.swSaveAsOptions_Silent, ...)` with DXF options configured via `ISldWorks.SetUserPreferenceIntegerValue(swUserPreferenceIntegerValue_e.swDxfMappingFileType)`; or `IModelDoc2.ExportToDWG2(path, srcFile, exportType, sheets, ...)`.
- **Rule 13 (developed length verification):** query flat-pattern body via `IFlatPatternFeature.FlatPatternBody` → `IBody2.GetBox` bounding box.
- **Rule 14 (bend direction up/down marks):** `IFlatPatternFeatureData.ShowBendLines`, `.ShowBendNotes`; programmatic access to each bend line via `IBendLineFeature`.
- **Rule 15 (corner relief):** `IFeatureManager.InsertCornerTrim` / `ICornerTrimFeatureData`.
