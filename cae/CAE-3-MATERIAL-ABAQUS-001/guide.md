# Guide: Composite Material Modeling in Abaqus

1. **Ply orientation reference**: The 0-degree direction must correspond to the tube axial direction. Define this explicitly with an *ORIENTATION keyword. Incorrect orientation is the single most common error in composite tube analysis.

2. **Plus/minus 45 plies provide torsional stiffness** — they belong on the outer plies for maximum torsional rigidity. The given layup places them correctly on the outside of the tube.

3. **0-degree plies carry bending loads** — they are placed near the midplane in this layup. For a cantilever tube under shear, the 0-degree plies dominate the bending response.

4. **Symmetric layup is mandatory for tubes**. An asymmetric layup introduces bend-twist coupling through the B-matrix in Classical Lamination Theory (CLT). The [plus/minus 45/0/90/0/plus/minus 45]s layup is symmetric by definition.

5. **Hashin damage requires 4 strength values plus shear**: Xt (tensile fiber), Xc (compressive fiber), Yt (tensile matrix), Yc (compressive matrix), and S (in-plane shear). All five must be defined for the damage initiation criterion to function.

6. **Fracture energies for damage evolution**: Typical values for T700/M21 are fiber tension ~100 kJ/m-squared, fiber compression ~60 kJ/m-squared, matrix tension ~0.5 kJ/m-squared, matrix compression ~1.6 kJ/m-squared. Matrix fracture energies are orders of magnitude lower than fiber energies.

7. **Viscous regularization**: Start with a coefficient of 1e-5. Increase to 1e-4 only if convergence is severely problematic. Higher values introduce artificial energy dissipation that can mask real damage behavior.

8. **CTE in the fiber direction is slightly negative for carbon fiber**. During cure cool-down, the composite contracts transversely but can slightly expand in the fiber direction. This creates residual tension in the matrix and compression in the fiber direction.

9. **Cure residual stress setup**: Use *INITIAL CONDITIONS, TYPE=TEMPERATURE to set the stress-free temperature to 180 degrees C. Then apply a temperature field of 20 degrees C in the thermal step. The delta-T of -160 degrees C drives the residual stress.

10. **Element type consideration**: S4R (reduced integration) is acceptable for global response but may miss localized damage initiation due to the single integration point per layer. S4 (full integration) is more accurate for damage predictions. Note this limitation in the results.

11. **Mesh refinement**: The 2mm element size is adequate for global structural response but may not capture localized damage initiation at stress concentrations. Document this as a known limitation.

12. **Output TSAIW (Tsai-Wu index) as a complementary check** even when using Hashin as the primary failure criterion. Tsai-Wu provides a single scalar failure index that is useful for quick screening.

13. **Verify ply orientations visually in CAE** before submitting the analysis. Use the "Ply Stack Plot" or element orientation display to confirm that each ply angle is correct.

14. **Stacking sequence convention**: Define plies from bottom to top in the *SHELL SECTION, COMPOSITE definition. For a tube, this corresponds to inside surface to outside surface.

15. **CLT sanity check**: Compute the ABD matrix and predict the undamaged tube stiffness using Classical Lamination Theory. Compare the FEA deflection under a known load to the CLT prediction. They should agree within 5% for the undamaged case. Significant deviation indicates a setup error.
