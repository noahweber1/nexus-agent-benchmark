# Guide: Weldment Frame Design

## Member Sizing and Selection

1. **Size columns first — they carry vertical loads plus any overturning moment.** Horizontal operating loads applied at height create significant moments at the column bases. Verify column capacity for the worst-case moment plus dead loads (equipment weight, self-weight).

2. **SHS for columns, RHS for horizontals oriented to load.** Columns see loads from multiple directions, so square hollow section is appropriate. Horizontal members primarily resist bending in one plane, so orient the rectangular section's strong axis accordingly.

3. **Minimum wall thickness 3mm for all structural members.** Below 3mm, MIG welding becomes difficult without burn-through, and the thin wall limits thread engagement for bolted connections.

4. **Gusset thickness equals member wall thickness as a minimum.** Gussets should extend at least 1.5x the member width along each connected member.

## Equipment Mounting

5. **Adjustable mounts: slotted holes oriented in the adjustment direction.** Size slot length for the required adjustment range (typically 20-30mm for belt or chain tensioning) and slot width for the fastener size. Mounting plate thickness must resist vibration without fretting — 10mm minimum for motors and similar rotating equipment.

6. **Equipment platforms: continuous plate rather than rails.** A solid plate under a vibrating machine provides damping and distributes load more evenly than rails. Machine the top surface flat for foot contact.

## Fabrication Details

7. **Use coping (saddle cuts) for tube-to-tube joints, not end miter.** Coping provides better fit-up for welding and stronger joints than mitered ends. End miter is only appropriate for frame corners where both members terminate.

8. **Cut list: include saw kerf allowance in material ordering.** Add approximately 5mm per cut when calculating raw material lengths for purchasing. Kerf allowance belongs in ordering notes, not in CAD model dimensions.

9. **Fillet weld sizing rule of thumb: weld throat = 0.7 x thinner member wall thickness.** Compute the weld leg from the throat dimension (leg = throat / 0.707).

10. **Weld access: verify a welder can physically reach every joint.** Minimum 40mm clearance for a MIG torch nozzle. If a joint is inaccessible from outside, consider plug welds through the outer member or redesign the joint.

## Frame Stiffness and Stability

11. **Add diagonal bracing in the plane of the horizontal load.** Without bracing, the frame relies on joint rigidity alone to resist lateral loads. A single diagonal flat bar (e.g., 50x6mm) in the loaded plane significantly improves lateral stiffness.

12. **Check natural frequency against excitation sources.** Frame first-mode natural frequency should be at least 2x the dominant excitation frequency (motor RPM, operating cycle) to avoid resonance. Stiffer columns and diagonal bracing raise the first mode.

13. **Mounting feet sized for total weight plus a safety factor.** Estimate total weight (frame + mounted equipment) and size each foot's rating for at least 1.25-1.5x the per-foot static share to cover uneven loading and dynamic effects.

## Documentation and Finish

14. **End caps: use weld-on flat caps, not plastic inserts.** For outdoor or wash-down environments, plastic caps fall out; welded caps seal the tube permanently and can be painted over.

15. **Paint prep specified on the drawing.** For outdoor or wet service, specify surface preparation in the notes (e.g., hot-dip galvanizing, or SA 2.5 blast with 2-pack epoxy primer). Do not leave finish to the fabricator's discretion.

16. **Document weld procedure spec (WPS) reference in the weld map.** Each weld symbol should reference the applicable WPS number. Use a standard pre-qualified WPS for the material and process where possible (e.g., EN ISO 15614 for MIG on structural steel).

## SolidWorks API References (C#)

Weldment automation from `SolidWorks.Interop.sldworks`. Structural members live in the Weldment feature (`IWeldmentFeatureData`); cut-list data is exposed via `IBodyFolder` and `IWeldmentCutListAnnotation`.

- **Rule 1, 2, 3 (member sizing and profile selection):** structural members via `IFeatureManager.InsertStructuralMember(layoutSketch, path, profileStandard, profileType, profileSize, groups)`. Profiles live in the weldment-profile library (`.sldlfp`) and are browsed by `swStructuralMemberStandard_e` / type / size strings.
- **Rule 4 (gussets):** `IFeatureManager.InsertGusset(profile, thickness, direction, location, chamferType, ...)` → `IGussetFeatureData`.
- **Rule 5 (slotted holes for adjustable mounts):** sketch with slot entity via `ISketchManager.CreateSlot(slotType, arcType, x1, y1, z1, x2, y2, z2, x3, y3, z3, width)`; then `FeatureExtrusion3` with cut option.
- **Rule 6 (continuous plate):** `FeatureExtrusion3` on a sketched rectangle; assign as a separate body and add to the weldment via `IFeatureManager.InsertAddBody`.
- **Rule 7 (coping / trim-extend):** `IFeatureManager.InsertTrimExtend2(trimType, corner, keepBody, ...)` — `trimType` = `swTrimTypes_e.swTrimCoping` for saddle cuts.
- **Rule 8 (cut list + kerf allowance):** cut-list data via `IBodyFolder.GetBodies` and `IBody2.GetCutListCustomProperty` / `.SetCutListCustomProperty`; update lengths for ordering in the BOM, not the model.
- **Rule 9, 10 (weld beads and access):** `IFeatureManager.InsertFilletBead4(weldSize, beadType, tangentPropagate, ...)` → `IFilletBeadFeatureData.WeldSize`. Custom beads via `IFeatureManager.InsertWeldBead` with a weld path.
- **Rule 11 (diagonal bracing):** add another group to the structural member feature via `IStructuralMemberGroup` or create a flat-bar body with `FeatureExtrusion3`.
- **Rule 12 (natural frequency):** SolidWorks Simulation API: `ICWStudyManager.CreateNewStudy3(name, swsAnalysisStudyType_e.swsAnalysisStudyTypeFrequency, ...)` → `ICWSolidManager` to mesh and solve; results via `ICWResults.GetModesFrequency`.
- **Rule 13 (mounting feet):** modeled as assembly components; load sharing verified via mass properties (`IModelDocExtension.CreateMassProperty.Mass`).
- **Rule 14 (end caps):** `IFeatureManager.InsertEndCap(thickness, chamferType, offsetType, offset, ...)` → `IEndCapFeatureData`.
- **Rule 15 (paint prep on drawing):** general note on the drawing via `IModelDoc2.InsertNote`; link to title-block custom property through `ICustomPropertyManager.Add3`.
- **Rule 16 (WPS reference on weld map):** weld symbol via `IModelDoc2.InsertWeldSymbol3` → `IWeldSymbol.SetProperties2` (procedure reference field).
