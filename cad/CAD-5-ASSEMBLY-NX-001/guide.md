# Guide: NX Assembly Restructuring

## Restructuring Strategy and Tribal Knowledge

1. **Use WAVE links cautiously.** Do not create new inter-part references during the restructure. WAVE geometry links between sub-assemblies create fragile dependencies that break when the assembly tree changes. If WAVE links already exist in the flat assembly, preserve them but do not add more.

2. **Group components by functional unit, not by proximity or part number.** A sub-assembly should contain components that move, function, or are handled together as a unit. Proximity alone is not a valid grouping criterion.

3. **Within each sub-assembly, identify and ground the primary component.** Anchor every sub-assembly to one component (the largest, the stationary one, or the kinematic reference) and constrain the remaining components to it. This makes the sub-assembly behavior predictable.

4. **Build sub-assemblies bottom-up.** Create lower-level sub-assemblies first, verify each is self-consistent, then place them into higher-level sub-assemblies. Do not try to restructure top-down by dragging components — this frequently breaks constraints.

5. **Preserve constraint types exactly.** Do not convert a touch constraint to a distance-zero constraint, or a concentric to an align + distance. The constraint type carries design intent information. Changing types may appear to work but causes problems during design updates.

6. **Split constraints across sub-assembly boundaries where appropriate.** For components that interface between two sub-assemblies (bearings between a shaft and a housing, fasteners between two plates), it is correct and expected for each side's constraint to live in its respective sub-assembly. Do not try to keep both sides of the interface constrained inside a single sub-assembly.

7. **Fastener grouping: one component pattern per logical fastener set.** Each circle, row, or array of fasteners should be a patterned set, not individual placements. This makes future pattern changes parametric.

8. **Exploded view sequence must follow real assembly order.** Sequence should match what a technician would actually do in the shop: base components first, then stacked components in the order they are installed, ending with cosmetic covers and mounting hardware.

9. **Check total component count before and after restructuring.** Query the assembly navigator for total leaf-level components. The count must be identical before and after. Any discrepancy means a component was lost or duplicated during the restructure.

10. **Use reference sets to control display complexity.** At the top level, set each sub-assembly's reference set to show only the outer envelope rather than all internal detail. This keeps the top-level display manageable without hiding components from the assembly structure.

11. **Name sub-assemblies by function, not by part number range.** Use descriptive names tied to the functional role (e.g., `INPUT_SHAFT_ASSY`, `LOWER_HOUSING_ASSY`, `FASTENER_PACK_SPLIT`). Avoid names like `SUB_001` that carry no meaning.

12. **Run a top-level interference check after restructuring.** The same interferences (or lack thereof) that existed in the flat assembly must exist in the restructured assembly. New interferences indicate a constraint was altered during the restructure. Missing interferences indicate a component shifted.

13. **Document the hierarchy.** Add a note (assembly attribute or description field) describing the sub-assembly breakdown and the rationale. This helps future users understand why the hierarchy was chosen.

14. **Do not break existing interpart references.** If the flat assembly contains interpart expressions, linked dimensions, or positional references between components, those references must survive the restructure. Test by editing a driving dimension and confirming the linked dimension updates.

15. **Verify each sub-assembly opens independently.** After restructuring, open each sub-assembly file on its own (not in the context of the top-level assembly). It must load without missing-component errors or broken constraints. This confirms the sub-assembly is self-contained.

16. **Assembly mass must be unchanged.** Query mass properties on the top-level assembly before and after. The values must match within a tight tolerance (typically 0.01 kg or the smallest significant digit). A mass discrepancy means a component was lost, duplicated, or had its material altered during restructuring.

## SolidWorks API References (C#)

SolidWorks equivalents for the same assembly-restructuring principles, from `SolidWorks.Interop.sldworks`. This eval targets NX, so these are cross-references for when the same workflow is automated in SolidWorks. The SW counterpart of NX WAVE links is in-context (external) references.

- **Rule 1 (avoid new inter-part references):** enumerate existing external references with `IAssemblyDoc.ListExternalFileReferences2` and `IFeature.GetParents`; break unwanted references with `IModelDoc2.BreakExternalReference3()`. Lock components with `IAssemblyDoc.LockComponent(comp)` while restructuring.
- **Rule 2, 3 (functional grouping, ground primary component):** promote components into a sub-assembly via `IAssemblyDoc.FormNewSubAssembly6(subAsmTemplate, name, options, ...)`; fix the primary component with `IAssemblyDoc.FixComponent` or set `IComponent2.Solving = swComponentSolving_e.swComponentRigidSolving`.
- **Rule 4 (bottom-up):** create each sub-assembly as its own file with `ISldWorks.NewDocument(template, papersize, width, height)`, then drop into parent via `IAssemblyDoc.AddComponent5(compName, configOption, newConfigName, useConfigForPartRefs, configName, x, y, z)`.
- **Rule 5 (preserve constraint types):** iterate mates via `IComponent2.GetMates()` → `IMate2.Type` (`swMateType_e` — `swMateCONCENTRIC`, `swMateCOINCIDENT`, `swMateDISTANCE`, etc.). Do not delete and re-create as a different type.
- **Rule 6 (split constraints across boundaries):** mates in the outer assembly add via the outer `IAssemblyDoc.AddMate5(mateTypeFromEnum, alignFromEnum, flip, distance, ...)`; mates inside the sub-assembly add from its own `IAssemblyDoc`.
- **Rule 7 (fastener pattern):** `IFeatureManager.InsertComponentPattern2(...)` for assembly patterns (`swComponentPatternTypes_e.swComponentPatternType_Circular`, `_Linear`, `_DerivedLinear`, etc.).
- **Rule 8 (exploded view sequence):** `IAssemblyDoc.NewExplodedView`; add steps with `IAssemblyDoc.InsertExplodeStep2(componentNames, direction, distance, rotation)` → `IExplodeStep`.
- **Rule 9 (component count):** `IAssemblyDoc.GetComponents(topOnly=false)` returns the full flat array of leaf components; compare array length pre- and post-restructure.
- **Rule 10 (reference sets / display complexity):** SW equivalent is display states — `IConfiguration.CreateDisplayState(name)`, `IComponent2.SetSuppression2(swComponentSuppressionState_e.swComponentLightweight)`. For envelope-only top-level display, use component envelope references via `IComponent2.SetEnvelope`.
- **Rule 11 (name sub-assemblies by function):** `IComponent2.Name2` controls the instance name; the file name is set by `ISldWorks.NewDocument` or by `IModelDoc2.SaveAs3(path, ...)` at sub-assembly save time.
- **Rule 12 (interference check):** `IAssemblyDoc.InterferenceDetectionMgr` → `IInterferenceDetectionMgr.GetInterferences()`; compare results count and affected components pre- and post-restructure.
- **Rule 13 (hierarchy documentation):** `ICustomPropertyManager.Add3("HierarchyNotes", swCustomInfoType_e.swCustomInfoText, note, swCustomPropertyAddOption_e.swCustomPropertyReplaceValue)` on the top-level assembly.
- **Rule 14 (interpart expressions):** SW linked dimensions via `IEquationMgr.Add3("\"D1@Part1\" = \"D1@Part2\"")`; verify by editing one side and calling `IModelDoc2.ForceRebuild3(false)`.
- **Rule 15 (independent open):** `ISldWorks.OpenDoc6(path, swDocumentTypes_e.swDocASSEMBLY, swOpenDocOptions_e.swOpenDocOptions_Silent, "", errors, warnings)` — success means no missing-reference or broken-mate errors are flagged.
- **Rule 16 (mass unchanged):** `IModelDocExtension.CreateMassProperty()` → `IMassProperty.Mass` before and after restructuring.
