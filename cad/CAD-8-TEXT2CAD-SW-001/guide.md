# Guide: Text-to-CAD Design Sequence

## Design Sequence and Tribal Knowledge

1. **Start with the features that drive downstream geometry.** Identify the datum-bearing features in the spec (bore centers, primary axes, mounting-pattern references) and establish them first. All dependent geometry grows outward from these references.

2. **Local wall thickness must match local function.** A single "nominal wall" callout applies to general housing walls. Load-bearing regions (bearing seats, pressurized sections, threaded bosses) need a locally thicker wall sized for the mechanical duty — not the nominal value.

3. **Apply casting/molding draft consistently on vertical faces.** For cast or molded parts, apply the draft angle specified for the material and process on every vertical face. Apply draft before adding machined features so the machining cuts through the draft to final dimension.

4. **Fillet all internal corners above a minimum radius.** Sharp internal corners in cast/molded parts create stress risers and casting defects (hot tears). Follow the material's minimum internal fillet guidance (typically 3-5mm for iron castings). External corners can remain sharp or carry a small break-edge.

5. **Bore tolerances per the mating-component fit specification.** For press or transition fits (H7 or tighter), pull the tolerance directly from the mating component's OD and fit class. Never hand-pick tolerances without checking the fit chart for the specific diameter.

6. **Flange thickness: minimum 1.5x fastener diameter.** Bolted flanges must be thick enough for adequate thread engagement on the mating side and flange stiffness to maintain alignment of critical features across the joint.

7. **Locate dowel/alignment pins at the maximum practical spread.** Place alignment features as far apart as practical — ideally at opposite corners — to maximize angular alignment accuracy during reassembly.

8. **Fluid-level features centered at the correct operating level.** Sight glasses, level switches, and similar features must be centered at the specified operating level relative to the internal geometry, not at an arbitrary housing reference.

9. **Drain ports at the lowest point with a slope toward the drain.** Internal sump floors should slope gently (1-2 degrees) toward the drain port. Do not place a drain at a flat section or at a local high point.

10. **Vents and breathers away from splash/spray zones.** Mount breather features on a surface above the highest expected splash level and offset away from moving internal components, to prevent the vent element from being fouled.

11. **Mounting feet: add gusseting between the foot and the main body.** Each foot pad should have at least two triangular gusset ribs connecting it to the main wall. Size gussets at or above the nominal wall thickness.

12. **Seal bores per the standard for the shaft diameter.** Use standard seal dimensions (DIN 3760 or equivalent) for the given shaft size. The seal bore OD must be concentric to the bearing bore within the sealing tolerance.

13. **Internal fluid capacity: check against operating requirements.** For lubricant or coolant sumps, verify that the contained volume meets the minimum circulation or service-interval requirement. A rough check is often enough at the modeling stage, but it must be done.

14. **Lifting features on any component above manual-handling weight.** Any housing half or structural component above the manual-handling threshold (typically 15-20 kg) needs at least two lifting points (tapped holes, cast lugs, or eyes) for crane handling during assembly and maintenance.

15. **Design for serviceability.** Identify the maintenance operation (shaft removal, filter change, inspection) and ensure the relevant covers or split lines allow the operation without full disassembly of unrelated components.

16. **Machining stock allowance on machined faces of the cast model.** Bearing bores, flange faces, seal bores, and mounting pads all require machining. The as-cast model should include extra material (typically 2mm) on these faces that gets removed to final dimension.

## SolidWorks API References (C#)

Primary modeling calls from `SolidWorks.Interop.sldworks`. All sketch-based features start with `IModelDoc2.SketchManager` and a plane created via `IFeatureManager.InsertRefPlane`.

- **Rule 1 (foundation axes/datums):** `IFeatureManager.InsertAxis2` (from edge, two points, plane intersection, etc.); reference planes via `IFeatureManager.InsertRefPlane(constraints, d1, d2, d3, dimAngle, d1val)`.
- **Rule 2, 4 (local wall thickness & internal fillets):** shell with `IFeatureManager.InsertFeatureShell3(thickness, outward)` — call before the fillet pass. Fillets via `IFeatureManager.FeatureFillet3(options, r, conicRhoType, ...)`.
- **Rule 3 (casting draft):** `IFeatureManager.InsertDraft2(draftType, angle, reverseDir, propagate, neutralPlane, facesToDraft)`; apply before any machining cuts.
- **Rule 5 (bore tolerances / Hole Wizard):** `IFeatureManager.HoleWizard5(genericHoleType, standardIndex, fastenerType, size, fit, ...)` — `fit` parameter maps to H7/H8/etc.; add GD&T via `IModelDocExtension.InsertGtol` on the drawing.
- **Rule 6 (flange thickness):** extrude flange with `IFeatureManager.FeatureExtrusion3`; bolt pattern via `IFeatureManager.FeatureLinearPattern5` or `FeatureCircularPattern6`.
- **Rule 7 (dowel pin locations):** hole wizard with `swHoleWizardType_e.swHoleWizardPipeTapDrill` or sketch-based extruded cut; pattern via `FeatureCircularPattern6`.
- **Rule 8 (fluid-level features):** create reference plane at the operating level with `InsertRefPlane`, sketch sight-glass opening on that plane, cut with `FeatureExtrusion3` (cut option).
- **Rule 9 (drain port with slope):** sketch on a plane tilted via `InsertRefPlane(angle)`; alternatively `IFeatureManager.InsertDraft2` on the sump floor face.
- **Rule 10 (breather location):** hole on top face via `HoleWizard5`; breather offset enforced via sketch constraints.
- **Rule 11 (gussets):** `IFeatureManager.InsertRib2(flipToExtrudeSide, thickness, direction, sketchRef, linearPattern)` for rib-style gussets; for triangular gussets use `FeatureExtrusion3` on a sketched triangle.
- **Rule 12 (seal bore):** `HoleWizard5` with custom dimensions, or extruded cut to standard seal OD with bore tolerance via `IModelDoc2.AddDimension2` + `IDisplayDimension.Tolerance`.
- **Rule 13 (capacity check):** `IModelDocExtension.CreateMassProperty` → `IMassProperty` provides `Volume` of the negative space if the sump is modeled as a solid, or compute from the closed surface body.
- **Rule 14 (lifting features):** tapped holes via `HoleWizard5`; cast lugs as sketched extrusions with fillets at the body intersection.
- **Rule 15 (serviceability / configurations):** use design configurations via `IConfigurationManager.AddConfiguration2(name, comment, ...)` — one with covers in place, one with them suppressed for the service view.
- **Rule 16 (machining stock allowance):** model the casting with `+2mm` offsets via `InsertOffsetSurface` or separate configurations (as-cast vs machined) managed with `IFeature.SetSuppression2` per-configuration.
