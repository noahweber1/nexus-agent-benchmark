# Guide: Thermal Defeaturing

## Quality Checklist

1. **Thermal interface regions are sacred.** Never merge, simplify, or modify faces that define thermal interfaces (pads, contact zones, mating surfaces between heat-conducting components). Their thickness drives thermal resistance (R = t/kA). Even small thickness changes corrupt downstream results.
2. **Preserve contact face topology.** Where a fastener was removed, the mating faces between components must remain as-is. Do not fill bolt holes on thermal contact faces — the hole pattern affects real contact area and the analyst may want to model it.
3. **Fill bolt holes on non-contact faces.** Through-holes on external faces (not at thermal interfaces) should be filled to avoid meshing artifacts. Use the Delete Face → Fill option, not manual patching.
4. **Suppress before deleting.** Always suppress features first and rebuild to check for cascading failures before permanently deleting. Feature trees often have parent-child dependencies that are not obvious.
5. **Small fillet threshold is context-dependent.** Any blanket small-fillet cutoff applies to non-thermal faces only. At thermal contact interfaces, retain all fillets regardless of size — they affect contact area and local mesh refinement zones.
6. **Check for internal voids after merging.** When combining bodies, verify no trapped internal volumes were created. Use Intersect → check for zero-volume regions.
7. **Material continuity at merge boundaries.** If merging two bodies with different materials, do not merge them. The thermal solver needs the material boundary.
8. **Verify watertightness body-by-body.** Run Check Entity (Import Diagnostics) on each remaining body individually. A clean assembly-level check can mask per-body issues.
9. **Chamfer removal order matters.** Remove chamfers before fillets. Chamfers sometimes depend on fillet edges, and removing fillets first can leave dangling chamfer features.
10. **Do not over-simplify mounting interfaces.** Faces where thermal boundary conditions will be applied (mounting feet, attach points) must retain their geometry and flatness. Identify these before any defeaturing pass.
11. **Cosmetic thread removal.** Remove all cosmetic thread annotations — they cause meshing failures in downstream tools but have no geometric effect.
12. **Naming convention.** Rename remaining bodies to indicate function (e.g., `MAIN_STRUCTURE_01`, `THERMAL_PAD_03`, `CONNECTOR_PLATE_LEFT`). The analyst needs to identify bodies in the solver.
13. **Mass sanity check.** Compare the defeatured assembly mass against the original. A moderate reduction is expected (primarily from fastener and small-feature removal). An unexpectedly large drop suggests structural geometry was accidentally removed.
14. **Export settings for STEP.** Use AP214 with solid bodies only. Disable surface body export. Tighten angular and chord tolerance for curved thermal contact surfaces.
15. **Document what was removed.** Leave a text note (or comment in the FeatureManager) listing the categories of removed features and counts. The analyst needs to know what simplifications were made.

## SolidWorks API References (C#)

The following SolidWorks API calls are the ones an automation or macro would invoke for the topics above. References use the standard SW namespace (`SolidWorks.Interop.sldworks`).

- **Rule 3 (fill holes), 5 (small fillets), 9 (chamfer removal):** `IFeatureManager.InsertDeleteFace2` (options: Delete, Delete-and-Patch, Delete-and-Fill). For blanket small-fillet suppression, iterate features via `IModelDoc2.FirstFeature` / `IFeature.GetNextFeature` and filter by `TypeName2 == "Fillet"`.
- **Rule 4 (suppress before delete):** `IFeature.SetSuppression2(swSuppressFeature, swConfigOption)` and `IModelDoc2.EditRebuild3()` to test for cascades.
- **Rule 6 (internal voids check):** `IBody2.GetMassProperties2` (check for negative/zero volume regions), and `IModelDocExtension.IntersectFeature` for explicit intersect operations.
- **Rule 7 (do not merge different-material bodies):** `IPartDoc.GetBodies2(swBodyType_e.swSolidBody, false)` to iterate, `IBody2.MaterialPropertyName` / `GetMaterialPropertyName2` to compare.
- **Rule 8 (watertightness per body):** `IModelDocExtension.CheckGeometry` (type flags include face-topology, manifold, etc.) and `IBody2.Check2`.
- **Rule 11 (cosmetic threads):** iterate and remove features with `TypeName2 == "CosmeticThread"` via `IFeature.Select2` + `IModelDoc2.EditDelete`.
- **Rule 12 (body naming):** `IBody2.Name` (set).
- **Rule 13 (mass sanity):** `IModelDocExtension.CreateMassProperty()` returning `IMassProperty` → `.Mass`.
- **Rule 14 (STEP export):** `IModelDocExtension.SaveAs3` with `swSaveAsVersion_e.swSaveAsCurrentVersion`, filename `*.step`, and set STEP export options via `ISldWorks.SetUserPreferenceIntegerValue(swUserPreferenceIntegerValue_e.swStepAP)`, chord/angular tolerance via `swStepExportFacetedSurfacesChordTol` / `swStepExportFacetedSurfacesAngularTol`.
- **Rule 15 (feature-tree comment):** `IFeature.Comment` → `IComment.Text`.
