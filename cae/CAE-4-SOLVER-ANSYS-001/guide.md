# Guide: Nonlinear Rubber Convergence in Ansys Mechanical

1. **Enable auto time stepping with bisection** — this is the single most impactful fix for rubber convergence issues. Fixed time stepping with too few substeps forces the solver to take displacement increments that are too large for the contact nonlinearity.

2. **Substep settings**: Start with initial substeps = 20, minimum substeps = 100, maximum substeps = 1000. The solver will automatically refine when needed and coast through easy regions.

3. **Mesh refinement in the contact zone**: The O-ring contact zone needs at least 6 elements across the contact width. Coarse meshes cause stress oscillation and contact chattering.

4. **Element type**: Use PLANE183 (8-node axisymmetric quadrilateral) with mixed u-P formulation. The mixed formulation is essential for nearly incompressible rubber (Poisson's ratio effectively > 0.49). Without it, volumetric locking produces artificially stiff response and convergence failure.

5. **Contact algorithm**: Switch from Penalty to Augmented Lagrange for rubber models. Augmented Lagrange is less sensitive to the contact stiffness factor and provides more stable contact pressure distribution.

6. **Contact stiffness**: Reduce the initial contact stiffness factor to 0.1 times the default. Rubber is very compliant, and the default stiffness (calibrated for steel-on-steel contact) is far too high. Let Ansys auto-update the stiffness during the solution.

7. **Pinball radius**: Increase to 1.5 times the expected contact width. This prevents premature contact open/close cycling (chattering) by keeping contact elements in the "near contact" state longer.

8. **Contact stabilization**: Use contact stabilization damping to smooth the transition between open and closed contact states. Do not use global displacement stabilization — it adds artificial stiffness to the entire model and corrupts the rubber deformation.

9. **Line search**: Enable line search for Mooney-Rivlin models. Line search helps the Newton-Raphson solver find the correct step length when the stiffness matrix changes significantly between iterations, which is common with hyperelastic materials.

10. **NLGEOM must be ON**: Large deformation (NLGEOM) is mandatory for rubber analysis. Verify this is active. Without it, the Mooney-Rivlin model gives incorrect results even if the analysis converges.

11. **Hourglassing check**: If using reduced integration elements with large deformation, check for hourglass modes. Hourglassing manifests as a zigzag displacement pattern and can cause divergence. Full integration or mixed u-P formulation mitigates this.

12. **Convergence criteria**: Both force AND displacement convergence criteria must be satisfied. Check that Ansys is not using displacement-only convergence, which can declare convergence prematurely for rubber.

13. **Normal contact stiffness update frequency**: Set to "Each Iteration" rather than "Each Substep." Per-iteration updates track the changing contact state more accurately during the Newton-Raphson process.

14. **Mesh bias on the O-ring**: Use a finer mesh on the O-ring surface that contacts first. For gland contact, this is typically the inner diameter surface of the O-ring.

15. **D1 compressibility check**: D1=0.01 per MPa gives a bulk-to-shear modulus ratio (K/G) of approximately 100, which is reasonable for engineering analysis. If D1 is too small (ratio > 1000), volumetric locking occurs even with mixed formulation. If D1 is too large, the rubber becomes artificially compressible. Do not change D1 — it is correct as given.
