# Guide: Injection Mold Core/Cavity Design

## Material and Process Parameters

1. **Apply material shrinkage as a uniform scale factor on the cavity geometry.** Shrinkage values come from the material datasheet (typically 0.4-0.7% for unfilled amorphous thermoplastics, higher for semi-crystallines). For materials that shrink isotropically at the part's wall thickness, no directional compensation is required.

2. **Texture compensation: add draft proportional to texture depth.** Add approximately 1 degree of additional draft per 0.025mm of texture depth on top of the functional minimum draft. Always confirm feasibility with the texture vendor.

3. **Core steel selection by expected shot count and material aggressiveness.** Pre-hardened mold steel (~30-34 HRC) is adequate for moderate-volume unfilled thermoplastics. Higher volumes, filled resins, or corrosive plastics require hardened tool steel or specialty grades.

4. **Cavity steel selection by surface finish requirement.** Standard pre-hardened tool steel works for textured or non-cosmetic surfaces. Mirror-polished cosmetic surfaces benefit from steels chosen for polishability (e.g., NAK80).

## Feature Redesign for Straight Pull

5. **Convert side-facing openings to through-features where possible.** An opening perpendicular to the pull direction forces a side action. Redesign as a through-slot in the pull direction and install any windowed element (lens, bezel) as a secondary press-fit part.

6. **Convert side-action latches to straight-pull mechanisms.** Replace side-action snap latches with living hinges, bayonet twists, or deflecting cantilevers that release along the pull axis. Confirm the chosen material supports the required flex cycles.

## Gate and Fill

7. **Gate on a non-cosmetic surface, at an internal thick section.** A sub-gate on a rib intersection or internal boss hides the vestige and provides a thick section for clean fill. Gate diameter should scale with wall thickness (typically ~60-80% of wall).

8. **Weld lines: predict and verify they do not cross cosmetic A-surfaces or structural features.** Flow fronts split around cutouts and cores. Locate the gate so weld lines land on non-cosmetic, low-load regions.

9. **Runner balance for multi-cavity tools.** For single-cavity tools, verify the gate diameter and cold runner drop match the part volume and material. For multi-cavity tools, size the runners so that all cavities fill simultaneously (naturally balanced or runner-sized to compensate).

10. **Venting at last-fill locations.** Provide shallow vent channels (typically 0.02mm deep with a land length of ~3mm, then relief to ~0.5mm) opposite the gate and at the perimeter of any feature that traps the flow front.

## Cooling and Ejection

11. **Cooling: bubblers for deep core features, baffles for shallow.** Deep cores (bosses, posts) trap heat and require bubblers (fountain tubes). Shallow regions use standard waterline circuits. Avoid leaving any core feature without cooling within reasonable distance.

12. **Cooling channel spacing.** Maintain approximately 2-3x channel diameter between channels and 1.5-2x channel diameter from the molding surface. Closer spacing risks hot spots and interference with ejector pins.

13. **Ejector pins on ribs and bosses, never on cosmetic surfaces.** Pin marks are unavoidable. Use the natural rib and boss layout to provide ejector locations. For cored posts, use sleeve ejectors around the post core pins.

14. **Ejector pin minimum diameter sized to wall thickness and material.** Pins that are too small risk punching through the part or leaving excessive marks. Size pins per the tool-builder's standard for the chosen material (typically >=3mm for amorphous thermoplastics at sub-2mm walls).

## Mold Construction

15. **Shut-off angles: minimum 3 degrees on all parting-line shut-offs.** Where the parting surface transitions from flat to angled (around cutouts, edge contours), maintain at least 3 degrees to prevent steel-on-steel wear and flash.

16. **Through-core features need adequate steel between openings.** Maintain a minimum steel web (typically 2mm or more) between adjacent openings on the core side. Thinner webs deflect under injection pressure and cause flash.

17. **Interlock: guide pins and heel blocks for alignment under pressure.** The cavity and core must align within a tight tolerance (approximately 0.02mm) to prevent wall thickness variation. Use guide pins at the insert corners plus heel blocks on long sides.

18. **Parting line: follow the maximum periphery of the part to minimize witness line visibility.** The parting line witness mark should fall on a designed edge or step, not on a flat cosmetic surface.

19. **Cycle time estimate from material and wall thickness.** Cycle consists of injection (~1-3s), cooling (dominant, scales with wall thickness squared), and mold open/close/eject (~3-5s). Cooling circuit design directly determines whether cycle time hits the low or high end.

## SolidWorks API References (C#)

Mold-tools automation from `SolidWorks.Interop.sldworks`. Mold features are accessed via `IFeatureManager` on the part document that contains the combined part/core/cavity bodies.

- **Rule 1 (shrinkage scale):** `IFeatureManager.InsertScale(scaleAboutOption, uniform, factorX, factorY, factorZ)` — `uniform=true` with equal factors for isotropic shrinkage (e.g., 1.005 for 0.5%).
- **Rule 2 (draft for texture):** `IFeatureManager.InsertDraft2(draftType, angle, ...)` applied after functional draft; verify with `IModelDocExtension.RunDraftAnalysis`.
- **Rule 3, 4 (core/cavity block material):** model block with `FeatureExtrusion3`; assign steel via `IBody2.SetMaterialPropertyName`.
- **Rule 5, 6 (straight-pull feature redesign):** modeled via extrusion cuts / revolves; no dedicated mold API. Verify all new features pass with `IModelDocExtension.RunUndercutAnalysis`.
- **Rule 7 (gate as feature):** modeled as an extruded cut or revolved cut on the runner body; position referenced via reference points.
- **Rule 8 (weld-line prediction, Plastics add-in):** `IPlasticsStudyManager.CreateStudy("FlowStudy")` → set injection location, run `.Analyze()`, read `.GetWeldLineResults()`.
- **Rule 9 (runner):** sketch runner path; create with `IFeatureManager.InsertMoldRunner(sketch, sectionType, sectionParams)` — section types enumerated in `swRunnerSectionType_e`.
- **Rule 10 (venting):** modeled as shallow extruded cuts on the parting surface using `FeatureExtrusion3` with small depth.
- **Rule 11, 12 (cooling channels):** `IFeatureManager.InsertMoldCoolingLine` (or sketched path + swept cut); positions governed by user reference geometry.
- **Rule 13, 14 (ejector pins):** modeled as extruded cuts in the core block; use `HoleWizard5` for standard ejector pin bores.
- **Rule 15 (shut-off angles):** `IFeatureManager.InsertShutOffSurface(faces, knit, ...)` — set drafted angle on the shut-off via `IShutOffSurfaceFeatureData.DraftAngle`.
- **Rule 16, 17 (core/cavity, tooling split, interlock):** parting line: `IFeatureManager.InsertMoldPartingLine4(...)`; parting surface: `IFeatureManager.InsertMoldPartingSurface(...)`; tooling split: `IFeatureManager.InsertToolingSplit(blockSketch, depthUp, depthDown)`; guide pins/heel blocks added as derived part inserts.
- **Rule 18 (parting line):** `IFeatureManager.InsertMoldPartingLine4(drftAngle, faceAngle, useForCore, splitFaces, propagate, ...)`.
- **Rule 19 (cycle time estimation, Plastics):** `IPlasticsStudyManager` results include estimated cycle-time; read via the Plastics COM interface after analysis completes.
