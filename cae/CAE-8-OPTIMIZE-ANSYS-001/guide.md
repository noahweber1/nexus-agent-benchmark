# Guide: Topology Optimization of Suspension Upright

## Tribal Knowledge

1. **Multi-load-case optimization: use weighted compliance objective** with equal weight for all 3 load cases unless one is safety-critical. If bump is the safety case, consider weighting it 1.5x. Equal weighting is the default starting point.

2. **Bump load case typically drives the most material** -- expect the thickest structural sections to align with the vertical load path between the bearing bore and the lower A-arm pickup. The cornering case usually creates the widest lateral ribs.

3. **Manufacturing constraint (single draw direction)** eliminates enclosed voids and undercuts. This is essential for forging or simple casting. Align the draw direction with the wheel/hub axis (the axis through the bearing bore center).

4. **Exclusion zones must be properly defined** -- the topology optimizer WILL remove material from pickup points and bearing bore surfaces if they are not explicitly excluded from the design space. Verify exclusion by checking that these regions remain fully dense (density=1.0) after optimization.

5. **Minimum member size: set to 6mm** as the manufacturing floor for forging/machining. Thinner members are not reliably producible and create stress risers in the as-forged condition.

6. **Maximum member size: set to 40mm** to avoid overly thick sections that risk internal porosity in castings or poor through-hardening in forgings. This forces the optimizer to distribute material rather than create monolithic blocks.

7. **Checkerboard filtering: ensure it is enabled** (default in Ansys) to avoid mesh-dependent topology artifacts where alternating elements are full/empty in a checkerboard pattern. This is a numerical artifact, not a physical optimum.

8. **Symmetry constraint: the upright is NOT symmetric** -- do not apply a symmetry plane. The brake caliper mount and steering tie rod pickup break any potential symmetry. Applying symmetry will produce an incorrect result.

9. **SIMP penalty factor**: the default p=3 is appropriate for aluminum. Higher penalty values (p=4-5) push the result toward a cleaner 0/1 density distribution but may miss intermediate-density regions that indicate secondary load paths.

10. **Convergence: run minimum 100 iterations** and check that the objective function (compliance) has stabilized over the last 10 iterations. If still changing by > 1%, continue to 200 iterations.

11. **Post-optimization: the raw result is a density field, not usable geometry.** Elements have density values between 0 and 1. You must interpret the result by extracting an iso-surface and smoothing it into a manufacturable shape.

12. **Extract the iso-surface at density threshold 0.3-0.5** for the concept interpretation. Lower thresholds (0.3) include more material and show secondary load paths; higher thresholds (0.5) show only the primary structure.

13. **Stress in low-density elements is meaningless** -- the SIMP method assigns reduced stiffness E*rho^p to low-density elements, making their stress values artificially low. Evaluate stress only in regions with density > 0.8.

14. **Verify that all pickup points have continuous load paths to the bearing bore.** If any pickup appears disconnected or connected only by a thin bridge, the optimization may need more iterations or adjusted constraints.

15. **Concept sketch: round off topology artifacts** into fillets and smooth curves that are feasible for casting or forging. Sharp notches in the topology result are numerical, not structural -- replace them with generous radii (R > 3mm).

16. **Weight estimate**: multiply the topology volume (sum of element volumes times their density) by Al 6082-T6 density (2.71 g/cm3). Expected range for FSAE upright: 0.8-1.5 kg depending on load severity and volume constraint.
