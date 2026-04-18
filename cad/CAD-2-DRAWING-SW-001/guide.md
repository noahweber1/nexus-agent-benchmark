# CAD-2-DRAWING-SW-001 Evaluation Guide

## Tribal Knowledge for Production Drawing Packages

1. **Hidden lines removed in all standard views.** Dense internal geometry makes views unreadable when hidden lines are shown. Only section views should reveal internal features.

2. **Section views are mandatory for every internal feature intersection.** Each intersecting internal feature (crosshole, pocket, passage) must be shown in at least one section view so the machinist can verify intersection geometry, dimensions, and deburr requirements.

3. **Standard feature dimensions reference the applicable standard.** O-ring grooves, dowel bores, keyways, and other standard features must call out the specific standard and dash/size number (e.g., AS568, DIN, ISO). Do not dimension standard features with generic values — reference the specific standard and its tolerances.

4. **Thread callouts must include class of fit and specification reference.** Include thread size, pitch, and tolerance class (e.g., "7/16-20 UNF-2B", "M14x1.5-6H"). Add the governing spec (SAE J1926/J514, ISO metric port standards) where applicable.

5. **Surface finish tiers applied per function.** Sealing and mating faces (O-ring lands, gasket seats, dynamic seal bores) get the tightest Ra callout. General machined mating surfaces get an intermediate tier. Rough-machined or non-functional faces get the loosest tier. Use consistent values across the drawing set.

6. **Datum scheme reflects the fixturing order.** Datum A is the primary seating face, B constrains rotation, C constrains translation. Establish the scheme so it matches how the part is actually held in the CNC vise or fixture. All feature positions reference this scheme.

7. **True position for all critical feature locations referenced to the datum scheme.** Use cylindrical (diameter) tolerance zones, not rectangular. Apply MMC modifiers where gauging is the intended inspection method.

8. **Internal feature intersections flagged with a deburr note.** Internal intersections create burrs that can break free and contaminate downstream systems. Use a specific note such as "DEBURR ALL INTERNAL INTERSECTIONS -- NO LOOSE PARTICLES", not just a general deburr callout.

9. **Edge break note as a general note.** All external sharp edges require a break. Typical note: "BREAK ALL SHARP EDGES 0.2-0.5mm UNLESS OTHERWISE SPECIFIED."

10. **Test and inspection notes in general notes.** Any required proof, pressure, leak, or acceptance test must be referenced in the general notes with the applicable test specification and conditions (e.g., proof pressure = 1.5x operating pressure).

11. **Material specification callout uses the governing material standard, not just the alloy.** The title block should reference the specification number and condition/temper (e.g., AMS/EN/ISO spec and temper), not just the alloy name.

12. **Part marking location and method specified.** Permanent marking (vibro-etch, laser, cast-in) should be called out in a specific zone with content requirements (part number, serial number, manufacturer code).

13. **Scale discipline: standard views at 1:1 where possible, detail views at enlarged scale.** Small features (grooves, thread reliefs, tight fillets) must be shown at a scale where dimensions are legible. Never scale section views below 1:1.

14. **Depth dimensions measured from the functional reference face.** Threaded hole and port depths are measured from the sealing/seating face to the bottom of usable thread — not from the outer surface. This matches how the machinist programs the operation.

15. **Each sheet should have a consistent viewport arrangement.** Do not crowd views. Maintain reasonable spacing (typically 25mm minimum) between views. Dimension leaders should not cross other dimension lines.

16. **Feature patterns use composite position tolerance where appropriate.** A larger tolerance controls pattern location relative to datums; a tighter tolerance controls feature-to-feature spacing within the pattern (feature-relating).

17. **Spot face and counterbore callouts include depth and diameter.** Every sealed or seated feature that requires a counterbore/spot face must be dimensioned with diameter and depth. Apply the correct surface finish to the spot-faced area.

18. **General tolerance block per the applicable standard.** Include default linear, angular, and interpretation-convention callouts (e.g., "INTERPRET PER ASME Y14.5-2018"). Do not rely on default CAD template values without review.

19. **Revision block follows company convention.** Preliminary/first-release revision codes should match the governing drawing-control procedure (e.g., Rev "-" preliminary, Rev A first release). Include a revision history table on Sheet 1 or the last sheet.

## SolidWorks API References (C#)

Drawing-automation calls from `SolidWorks.Interop.sldworks`. A drawing document is `IDrawingDoc` (inherits `IModelDoc2`); each sheet is `ISheet`; each view is `IView`.

- **Rule 1 (hidden lines off):** `IView.DisplayMode = (int)swDisplayMode_e.swHIDDEN_GREYED` (HLR) or `swHIDDEN` (HLV); `IView.RemoveHiddenLines = true` for performance.
- **Rule 2 (section views):** `IDrawingDoc.CreateSectionViewAt5(x, y, z, sectionViewName, options, excludedComponents, arrowHeight)`; options come from `swCreateSectionViewAtOptions_e`.
- **Rule 3 (standard feature callouts) and Rule 4 (thread callouts):** `IModelDoc2.AddHoleCallout2`, `IHoleCallout` / `IHoleCalloutsCollection`. For thread class of fit, edit `IDisplayDimension.GetText` / `SetText` with `swDimensionTextParts_e.swDimensionTextCalloutAbove`.
- **Rule 5 (surface finish):** `IModelDoc2.InsertSurfaceFinishSymbol3` → `ISurfaceFinishSymbol.Ra`, `.RoughnessMax`, `.RoughnessMin`.
- **Rule 6 (datum scheme):** `IModelDoc2.InsertDatumTagSymbol` / `IDatumTag2`.
- **Rule 7 (true position / GD&T):** `IModelDoc2.InsertGtol` → `IGtol` (control frame parsed via `IGtol.SetFrameValues2`).
- **Rule 8, 9, 10, 18 (notes and general notes):** `IModelDoc2.InsertNote(text)` → `INote`; multi-line via `.SetTextAtIndex`.
- **Rule 11 (material in title block):** `IModelDoc2.SetMaterialPropertyName2(config, database, materialName)`; expose via custom properties with `ICustomPropertyManager.Add3`.
- **Rule 12 (part marking zone):** combine `IModelDoc2.InsertNote` with a `IBlockDefinition` for the marking box.
- **Rule 13 (view scale):** `IView.ScaleDecimal` or `IView.ScaleRatio`; for detail views `IDrawingDoc.CreateDetailViewAt4`.
- **Rule 14 (depth dim from functional face):** `IModelDoc2.AddDimension2(x, y, z)` — the reference face is selected before the call; use `IDisplayDimension.GetDimension2(0).ReferenceType`.
- **Rule 15 (view layout):** `IView.Position` (double[2]); sheet extents via `ISheet.GetProperties2`.
- **Rule 16 (composite position tolerance):** same `IGtol` API; set composite via `IGtol.CompoundCompositeFrame` and `CompoundFrameCount`.
- **Rule 17 (spot face / counterbore):** `IModelDoc2.AddHoleCallout2` with hole type `swHoleType_e.swHoleCounterbore`.
- **Rule 19 (revision table):** `IModelDocExtension.InsertRevisionTable3(configOpt, x, y, anchorType, templateName)` → `IRevisionTableAnnotation.AddRevision`.
