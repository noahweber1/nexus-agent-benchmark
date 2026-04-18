# Guide: STEP Import Repair and Healing

## Tribal Knowledge and Constraints

1. **Run Import Diagnostics immediately after import, before any manual editing.** SolidWorks Import Diagnostics can auto-fix many minor issues (small gaps, faulty faces) and gives you a baseline count of problems. Always start here.

2. **Fix gaps before sliver faces.** Gap healing often reshapes adjacent face boundaries, which can eliminate nearby sliver faces as a side effect. Fixing slivers first may result in wasted work if gap repair changes the surrounding topology.

3. **Surface gap repair strategy: auto-heal for small gaps, manual for larger.** Use Import Diagnostics "Heal" for gaps under a threshold (typically 0.1mm). For larger gaps, manually extend or rebuild the adjacent surfaces and knit them together.

4. **Sliver face removal: use Delete Face with Patch option, not Knit Surface.** Knitting preserves the sliver geometry. Delete Face with Patch removes the sliver and lets SolidWorks reconstruct the region with a properly proportioned face.

5. **Missing face reconstruction: use Fill Surface with tangent constraints.** When rebuilding a missing face, constrain the fill surface to be tangent to all neighboring faces. This preserves curvature continuity and ensures the reconstructed region blends smoothly.

6. **Flipped normal identification: use Check Entity on each body.** Check Entity reports face orientation issues. After identifying flipped normals, flip individual faces rather than reversing the entire body — some faces may already be correct.

7. **Do not merge bodies during repair.** A multi-body import typically reflects manufacturing reality (separate fabrication components, distinct materials). Merging bodies loses that distinction and can make downstream analysis impossible.

8. **Verify each body individually after repair.** Run Check Entity on every single body after all repairs are complete. A body can appear visually correct but still have topological errors that only Check Entity detects.

9. **Material assignment by component function.** Review the drawing/spec or component function to assign the correct material to each body. Do not rely on imported colors to infer material.

10. **Body naming convention: TYPE_LOCATION_SEQUENCE.** Use consistent, descriptive names (e.g., `OUTER_CASING_UPPER`, `INNER_LINER_FWD`, `MOUNTING_LUG_03`). Keep location terminology consistent (FWD/AFT, UPPER/LOWER, PORT/STBD).

11. **Compare total volume before and after repair.** Sum the volumes of all bodies before starting repairs and after completing them. The difference should be less than ~0.5%. A larger change indicates over-aggressive surface modifications.

12. **Watch for micro-faces that Import Diagnostics may miss.** Faces with area less than 0.01mm squared can cause meshing failures in downstream FEA but may not trigger Import Diagnostics warnings. Use Check Entity with tight tolerance settings to catch these.

13. **Color-code bodies by material for visual verification.** After assigning materials, set display colors to match a consistent scheme (e.g., silver for stainless, gold for high-nickel alloys, dark gray for tool steels). This makes it immediately obvious if a component has the wrong material.

14. **Repair log format: one entry per defect.** Each entry should include: body name, defect type (gap/sliver/missing face/flipped normal/other), location description, repair method used, and before/after status. Group entries by body for readability.

15. **Save intermediate states during repair.** Save the file after Import Diagnostics auto-heal, after gap repairs, after sliver fixes, and after missing face reconstruction. This provides rollback points if a later repair introduces new problems.

16. **Do not rely on imported colors for material identification.** Color mapping between source CAD systems (CATIA, NX, Creo) and SolidWorks is unreliable — colors may shift or be lost entirely. Identify components by geometry and position, not by imported display color.

## SolidWorks API References (C#)

Import/repair calls from `SolidWorks.Interop.sldworks`.

- **Rule 1 (Import Diagnostics):** `IPartDoc.ImportDiagnosis()` runs the diagnostic and applies auto-heal; `IPartDoc.ImportDiagnosisEntities2(swImportDiagnosisEntities_e)` returns arrays of problem entities (faces, gaps) to iterate.
- **Rule 2, 3 (gap repair):** for auto-heal gaps under threshold, use `IPartDoc.HealEdges(tolerance)`. For manual rebuilds: `IFeatureManager.InsertExtendSurface` and `IFeatureManager.InsertSewSurface(toleranceValue, mergeBodies, tryFormSolid)`.
- **Rule 4 (delete face + patch):** `IFeatureManager.InsertDeleteFace2(options)` where options include `swDeleteFaceOptions_e.swDeleteFaceOptions_DeleteAndPatch`.
- **Rule 5 (fill surface, tangent):** `IFeatureManager.InsertFillSurface2(bndMerge, trimEdge, tryFormSolid, curvatureControl, optimizeSurface)` — `curvatureControl` set to `swEndCondTangency_e.swEndCondTangency_Tangency` for G1, `_Curvature` for G2.
- **Rule 6 (flipped normals):** iterate faces via `IPartDoc.GetBodies2` → `IBody2.GetFaces` → detect via `IFace2.Normal` vs expected outward direction; flip with `IFace2.Flip` (works only on standalone surface bodies; for solid, use `IFeatureManager.InsertDeleteFace2` and rebuild).
- **Rule 7, 8 (verify per body):** iterate via `IPartDoc.GetBodies2(swBodyType_e.swSolidBody, false)` and call `IBody2.Check2(options)` with `swBodyFaultOptions_e` flags; `IModelDocExtension.CheckGeometry(checkOptions)` for the whole document.
- **Rule 9 (material assignment):** `IBody2.SetMaterialPropertyName(config, database, material)`; enumerate existing material via `IBody2.GetMaterialPropertyName2`.
- **Rule 10 (body naming):** `IBody2.Name` (set).
- **Rule 11 (volume comparison):** `IBody2.GetMassProperties2(accuracy, useSystemUnits)` returns volume at index 3; sum across bodies.
- **Rule 12 (micro-face detection):** iterate faces via `IFace2.GetArea` and filter below threshold; flag via custom `IEntity.Select4`.
- **Rule 13 (color by material):** `IBody2.MaterialIdName` and `IModelDocExtension.SetUserPreferenceColor`, or per-body via `IBody2.SetMaterialPropertyValues2(values, swInConfigurationOpts_e, configNames)` where values include display color RGB.
- **Rule 14, 15 (repair log + save):** log to file via standard .NET file I/O; save document with `IModelDoc2.Save3(saveOptions, errors, warnings)` or `SaveAs3`.
- **Rule 16 (ignore imported colors):** on `ISldWorks.OpenDoc6`, set import options via `ISldWorks.SetUserPreferenceIntegerValue(swUserPreferenceIntegerValue_e.swImportIGESandSTEPEntityColors, 0)`.
