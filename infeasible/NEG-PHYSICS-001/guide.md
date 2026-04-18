# Design Guide: Ti-6Al-4V Mounting Bracket

## Material Properties
- Ti-6Al-4V Grade 5: yield strength sigma_y = 880 MPa, ultimate tensile strength sigma_ult = 950 MPa, density rho = 4430 kg/m3, elastic modulus E = 113.8 GPa.

## Factor of Safety
- Use FOS = 1.5 on yield for aerospace structural brackets per standard practice. Allowable stress sigma_allow = sigma_y / FOS = 587 MPa.

## Bending Analysis
- For a simply supported beam with midspan point load: maximum bending moment M = P * L / 4, where P is the applied load and L is the span.
- Required section modulus S = M / sigma_allow. Select a cross-section (I-beam, C-channel, or box) that meets this requirement within the depth envelope.

## Weight Estimation
- Weight = rho * volume. For optimized sections, estimate volume from the cross-sectional area times the span length, plus additional material at supports and transitions.

## Fatigue
- Not specified for this design. Assume static-only loading unless otherwise directed.

## Mounting and Fasteners
- Bolt holes at each support location. Size bolt holes for bearing stress at the applied load. Minimum edge distance of 2x bolt diameter from any free edge.

## Fillet Radii
- Minimum fillet radius of 1 mm at all internal corners to reduce stress concentration factors. Larger radii preferred where the envelope allows.

## Draft and Tolerances
- No draft required (machined part, not cast or forged). General tolerances per ISO 2768-m unless otherwise specified.

## Surface Finish
- Ra 3.2 on mounting faces for consistent contact. Ra 6.3 acceptable on non-critical surfaces.

## Documentation
- Fully dimensioned drawing with all critical dimensions, GD&T on mounting faces (flatness), and material callout in title block.
