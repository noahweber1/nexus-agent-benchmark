# CAD-3-DFM-SW-001 Evaluation Guide

## Tribal Knowledge for Injection Molding DFM

1. **Uniform wall thickness is the single most important DFM rule.** Maintain nominal wall thickness within +/-10% across the part. Variations beyond this cause differential cooling, leading to warpage, sink marks, and internal voids. Thick-to-thin transitions must be gradual (tapered over approximately 3x the wall thickness difference).

2. **Draft angle minimum is 1 degree per side on untextured surfaces.** For textured surfaces, add 1 degree per 0.025mm (0.001 inch) of texture depth. The draft direction must follow the mold pull -- verify that each face's normal has a component along the pull axis.

3. **Rib thickness must not exceed 60% of the adjacent wall.** Thicker ribs cause sink marks visible on the opposite (cosmetic) face. Rib height should not exceed 3x the rib thickness without adding draft or taper to aid ejection.

4. **Rib base radius is 25-50% of wall thickness.** Add a fillet at the rib root. This reduces stress concentration and improves mold filling without causing excessive sink.

5. **Boss OD = approximately 2x screw hole ID.** Boss wall thickness should equal roughly 60% of the nominal wall. Add 2-4 gussets connecting the boss to adjacent walls for lateral support.

6. **Bosses must not be directly attached to sidewalls.** If a boss is near a wall, connect it with a thin rib (60% wall thickness) rather than merging it flush. A boss merged into a thick wall section creates a mass concentration and guaranteed sink.

7. **Snap-fit cantilever design: check permissible strain for the chosen material.** Deflection formula: `deflection = (permissible_strain * L^2) / (1.5 * t)`, where L is cantilever length and t is thickness at the root. Longer, thinner beams are preferred. The hook must release in the pull direction for a two-plate mold.

8. **Undercuts eliminated by "passing shutoff" or redesign.** If a feature creates an undercut, check whether it can be formed by the meeting of core and cavity (a shutoff). If not, redesign: replace through-holes on sidewalls with open-ended slots, convert snap hooks to deflecting cantilevers that release along the pull axis, or use bump-offs for shallow undercuts (< ~5% of diameter for round features).

9. **Gate location on a non-cosmetic face.** Gate on a surface that is not visible in the final product, near the geometric center where possible to minimize flow length. Edge gates on the parting line are acceptable if the vestige falls on a non-visible edge. Gate vestige must not interfere with assembly or cosmetics.

10. **Ejector pin placement on non-cosmetic faces only.** Pins leave a circular witness mark. Place them on internal ribs, boss tops, or hidden flanges. Ensure pins are on the ejector (core) side of the mold. Space pins to give balanced ejection force and prevent part distortion.

11. **Parting line follows the maximum cross-section perimeter.** On a cosmetic part, the parting line should fall on a designed step, chamfer, or edge so flash and mismatch are not visible. Avoid placing the parting line across a smooth, visible face.

12. **Weld lines must not land on structural or cosmetic-critical areas.** Weld lines form where two flow fronts meet and have reduced strength (typically ~80% of full material strength). Predict their location from gate position and flow paths around cores. Reposition the gate if a weld line would cross a snap-fit root or a visible panel.

13. **Apply material shrinkage from the datasheet.** Shrinkage is typically expressed as a percentage (often 0.4-0.7% for unfilled amorphous thermoplastics) and must be applied when scaling the mold cavity. The designer does not need to pre-shrink the CAD model, but non-uniform wall thickness causes differential shrinkage and warpage -- another reason uniform walls matter.

14. **Text and logos should be raised (standing proud) on the part.** Raised text is recessed in the mold steel (easier to machine via pocket milling). Recessed text on the part means raised steel in the mold, which traps gas and wears faster. If recessed text is required, add vent channels behind each character.

15. **No trapped steel conditions.** Verify that no thin steel projections exist in the mold that would overheat or break. Minimum steel thickness between features is generally 3mm. Core pins for holes must have a length-to-diameter ratio below about 3:1, or support them with interlocking features.

16. **Minimum internal corner radius is 50% of wall thickness.** Sharp internal corners cause 2-3x stress concentration, impede flow, and create cooling hot spots. External corners should have a radius equal to the internal radius plus the wall thickness to maintain uniform thickness through the turn.

17. **Living hinge design (if applicable): short land length, radius on both sides.** Some materials support living hinges well (polypropylene), others are marginal (ABS). For marginal materials, use a separate hinge pin or snap assembly instead.

18. **Core-out thick sections rather than leaving solid.** Any section significantly thicker than the nominal wall should be cored from the non-cosmetic side, leaving a uniform wall. Coring also reduces part weight, cycle time, and material cost.

19. **Verify the mold halves separate cleanly.** Mentally (or in CAD) split the part at the parting line. Each half must pull away without interference. No feature should lock the core and cavity together. Use the CAD undercut analysis or draft analysis tool to verify.

## SolidWorks API References (C#)

DFM-automation calls from `SolidWorks.Interop.sldworks`. Most mold-related analyses live under `IModelDocExtension` and the Mold Tools features under `IFeatureManager`.

- **Rule 1, 16, 18 (wall thickness / corners / coring):** `IModelDocExtension.GetThicknessAnalysisData` / `RunThicknessAnalysis` returns thin and thick regions. Shelling with `IFeatureManager.InsertFeatureShell3` enforces uniform walls.
- **Rule 2 (draft):** apply with `IFeatureManager.InsertDraft2(draftType, angle, reverseDir, propagate, neutralPlane, facesToDraft)`; verify with `IModelDocExtension.RunDraftAnalysis` (returns face categories: positive/negative/straddle/requires-draft).
- **Rule 3, 4 (ribs):** `IFeatureManager.InsertRib2(flipToExtrudeSide, thickness, direction, sketchRef, linearPattern)`; rib root fillets via `IFeatureManager.FeatureFillet3`.
- **Rule 5, 6 (bosses and gussets):** modeled as extruded bosses — `IFeatureManager.FeatureExtrusion3` with `startCondition`/`endCondition` per `swStartConditions_e`/`swEndConditions_e`.
- **Rule 7 (snap-fit cantilevers):** no direct API; modeled via extrusion + fillets. Analyze deflection with SolidWorks Simulation → `ICWStudyManager.CreateNewStudy3` (static), `ICWLoadsAndRestraintsManager` to apply loads/restraints.
- **Rule 8, 19 (undercut detection):** `IModelDocExtension.RunUndercutAnalysis` returns faces in undercut; configure pull direction via `IMoldToolsData` / `UndercutAnalysisData` parameters.
- **Rule 9, 11 (gate location / parting line):** `IFeatureManager.InsertMoldPartingLine4(drftAng, faceAngle, useForCore, splitFaces, ...)` generates the parting line entity from the part faces.
- **Rule 10 (ejector pin witness placement):** modeled as reference points or sketches on non-cosmetic faces; no dedicated SW API — annotation only.
- **Rule 12 (weld-line prediction):** SolidWorks Plastics API: `IPlasticsStudyManager.CreateStudy` → `IPlasticsMeshManager`, `IPlasticsAnalysisManager.Run`, inspect `IPlasticsWeldLinesResult`.
- **Rule 13 (shrinkage):** `IFeatureManager.InsertScale(scaleAboutOption, uniform, factorX, factorY, factorZ)` — apply after the CAD model is complete, before generating the mold cavity.
- **Rule 14 (raised text):** `IFeatureManager.FeatureExtrusion3` of a sketched text via `ISketchManager.InsertSketchText`.
- **Rule 15 (trapped steel / min steel web):** `IModelDocExtension.RunThicknessAnalysis` on the cavity block after tooling split; also `IMoldCoreCavityFeature` inspection.
- **Rule 17 (living hinge):** no direct API — modeled as thin extrusion + fillets.
