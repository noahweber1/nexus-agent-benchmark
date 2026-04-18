# Design Guide: Stepped Shaft Drawing

## Datum Selection
- Datum A: the largest diameter section (35 mm) establishes the primary datum axis. This is the main bearing surface and functional reference for all rotational controls.

## Cylindricity
- Cylindricity is a form control that limits the combined effect of roundness (circularity) and straightness deviations. It has no datum reference; it controls the feature in isolation.

## Total Runout
- Total runout controls the composite effect of form, orientation, and position of a surface relative to a datum axis. It is measured with a full-indicator sweep across the entire surface while rotating about the datum axis.

## Concentricity
- Concentricity controls the symmetry of the median points of a feature relative to a datum axis. It is a derived-median-point control per ASME Y14.5-2018 and is expensive to inspect. Consider position or runout as alternatives where appropriate.

## Surface Finish
- Ra 0.4 on the 25 mm journal: achieved by cylindrical grinding. This is consistent with precision bearing journal surfaces.

## Section Views
- Provide a cross-section through the 30 mm section showing keyway geometry, including width, depth, and corner radii per the applicable key standard.

## Dimension Tolerances
- h6 tolerance class on bearing journals (25 mm and 35 mm diameters) for interference or transition fits with bearings.
- js6 tolerance class on the keyway section for a close sliding fit with the mating hub.

## Material Callout
- Material specification must appear in the title block. Include hardness requirement if applicable (e.g., HRC 28-32 for case-hardened journals).
