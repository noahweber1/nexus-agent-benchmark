# Guide: Parametric Modeling in Creo

## Modeling Sequence and Strategy

1. **Build foundation geometry first** as datum curves and datum surfaces. Any profile or reference that downstream features depend on (flowpaths, section profiles, axis locations) must be established and verified before derived features are added.

2. **Use Variable Section Sweep (VSS) for compound-curve surfaces**, not Blend or Boundary Blend. VSS allows section shape, orientation, and size to vary continuously along a trajectory and across spanwise sections — essential for compound curves that must match analytical data.

3. **Confirm angle and reference conventions before modeling.** Misinterpreting convention (e.g., whether an angle is measured from the meridional direction or the tangential direction) produces features that curve the wrong way. Verify against the spec figure before proceeding.

4. **Interpolate across enough sections to capture compound curvature.** For compound-curve features, use at least 4 sections across the span (e.g., 0%, 33%, 66%, 100%). Fewer sections produce features that are too linear and will not match source data.

5. **Derive related features from a base pattern**, then trim. Do not model related features (splitters, derivative instances, shortened variants) independently — they must stay linked to the base definition so parameter changes propagate.

6. **Pattern features using a parametric pattern with count as a parameter.** Never copy-paste features; copied features break when the count parameter changes.

7. **G1 (tangent) continuity at filleted transitions is mandatory for downstream analysis.** CFD and FEA meshing fail on tangency breaks. Use tangent constraints at fillet boundaries and verify with surface analysis (Gaussian curvature, zebra stripes).

8. **Apply thickness as an offset from a base surface**, not by creating independent opposing solids. Offset ensures thickness stays consistent when upstream parameters change.

9. **Leading edges: use a manufacturable, mesh-friendly profile.** Do not use sharp edges (unmeshable, unmanufacturable) or overly blunt profiles (poor flow behavior, separation). An elliptical profile aligned with the governing curve is the common compromise.

10. **Trailing/terminal edges: use a blunt cutoff sized to the manufacturing process.** Avoid infinitely thin edges that cause regeneration failures and cannot be machined.

11. **Drive all key dimensions from a relations table** with top-level parameters. Expose the parameters a downstream designer or optimizer needs to change (size parameters, counts, angle scale factors, control-point positions). Avoid hard-coded dimensions buried in sketches.

12. **Test regeneration at parameter extremes before declaring done.** Set driving parameters to +/-15% of nominal, regenerate, and verify no features fail. Fix failures by loosening over-constrained sketches or adding relations guards.

13. **Standard features must be parametric on their governing dimension.** Use a relation to size keyways, bearing seats, and thread callouts per the applicable standard (DIN, ISO, etc.) based on the governing diameter. Do not hard-code standard-feature dimensions.

14. **Mass properties sanity check.** Assign the correct material density and compare computed mass against an expected range from prior designs or hand calculations. A large deviation suggests missing material or a runaway offset.

15. **Do not use Warp or Deform features.** These features break parametric regeneration when dimensions change and produce unpredictable surface quality. Build all geometry with native datum curves, sweeps, offsets, and patterns.

## SolidWorks API References (C#)

SolidWorks equivalents for the same parametric-modeling principles, from `SolidWorks.Interop.sldworks`. This eval targets Creo, so these are cross-references for when the same workflow is automated in SolidWorks.

- **Rule 1 (foundation datum curves / surfaces):** reference geometry via `IFeatureManager.InsertRefPlane(constraints, d1, d2, d3, dimAngle, d1val)`, `IFeatureManager.InsertAxis2`, and 3D sketches via `ISketchManager.Insert3DSketch2(addToDb)`. Parametric curves with `ISketchManager.CreateSplinesByEqnParams`.
- **Rule 2 (variable section sweep):** closest SW equivalents are `IFeatureManager.InsertProtrusionSwept4(...)` (swept boss with guide curves and twist control via `swSweepTwistControlType_e`) and `IFeatureManager.InsertBoundaryBaseSurface(...)` for full compound-curve control across multiple directional curves.
- **Rule 3 (angle convention):** no direct API — validate with `IModelDocExtension.CreateMeasure` → `IMeasure.Angle` between selected edges/faces.
- **Rule 4 (interpolate across multiple sections):** `IFeatureManager.InsertBoundaryBaseSurface` or `IFeatureManager.InsertProtrusionBlend` (loft) with multiple profiles and optional guide curves.
- **Rule 5 (derive from base, then trim):** `IFeatureManager.DerivedComponentPattern(...)` (derived patterns); trim with `IFeatureManager.FeatureCut3(...)` or `InsertCutExtrude4`.
- **Rule 6 (parametric pattern with count):** `FeatureLinearPattern5`, `FeatureCircularPattern6`, `FeatureCurveDrivenPattern2`. Expose count as a global via `IEquationMgr.Add3("\"blade_count\" = 12")` and link the pattern's instance-count dimension to it.
- **Rule 7 (G1 tangent continuity):** verify with `IEdge.GetConvexity` and `IFace2.GetTangencyType`; enable zebra stripes via `IModelView.EnableRender` + display-settings toggle (or `IModelViewManager` view setting `ZebraStripes`).
- **Rule 8 (offset from base surface):** `IFeatureManager.InsertOffsetSurface(thickness, flipDir)` and `IFeatureManager.InsertFeatureThicken2(...)`.
- **Rule 9, 10 (LE/TE profile treatment):** sweep profiles sketched and driven by relations; fillets with variable radius via `IFeatureManager.FeatureFillet3(options, r1, conicRhoType, r2, ...)` — use `swFeatureFilletType_e.swFeatureFilletType_VariableRadius`.
- **Rule 11 (relations and top-level parameters):** `IEquationMgr.Add3(equation, index, solveOrderType)` and `IEquationMgr.set_Equation(index, value)`; expose as global variables in the format `"\"name\" = value"`.
- **Rule 12 (regeneration at extremes):** drive a parameter via `IEquationMgr`, then `IModelDoc2.ForceRebuild3(topOnly=false)`; capture failures via `IFeature.GetErrorCode2`.
- **Rule 13 (parametric standard features):** `IFeatureManager.HoleWizard5(...)` for standard holes with `fit` class; keyways typically modeled as extruded cuts with dimensions driven by equations referencing the shaft diameter.
- **Rule 14 (mass properties):** `IModelDocExtension.CreateMassProperty()` → `IMassProperty.Mass`, `.Volume`, `.Density`.
- **Rule 15 (avoid Warp/Deform):** do not call `IFeatureManager.InsertDeformFreeForm`, `InsertDeformSurfaceToSurface`, `InsertDeformCurveToCurve`, or `InsertWarp`. Use native sweep/boundary/offset features instead.
