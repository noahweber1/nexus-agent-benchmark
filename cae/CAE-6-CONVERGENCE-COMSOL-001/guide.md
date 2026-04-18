# Guide: Mesh Convergence Study at Shoulder Fillet

## Tribal Knowledge

1. **Local refinement only at the fillet** -- global refinement wastes DOFs and solve time. Use COMSOL's "Size" node scoped to the fillet boundaries, with a coarser mesh elsewhere.

2. **Element size progression**: use a geometric ratio for the 5 levels (e.g., 2.0mm, 1.0mm, 0.5mm, 0.25mm, 0.125mm at the fillet root). Each level should halve the previous element size.

3. **Use quadratic elements (second order)** for stress accuracy at stress concentrations. Linear elements significantly under-predict peak stress at geometric discontinuities.

4. **Extract stress at the same node/point for all levels** -- use a point probe at a fixed coordinate on the fillet root, not "maximum in body." The max-in-body location can shift between mesh levels, corrupting the convergence curve.

5. **Nominal stress for bending**: sigma_nom = 32*M / (pi*d^3) for the smaller diameter (d=40mm). This is the standard formula for a solid circular cross-section. Verify: sigma_nom = 32*500 / (pi*0.04^3) should give approximately 79.6 MPa.

6. **Kt = sigma_max / sigma_nom** -- this is the stress concentration factor to compare against Peterson's value of 1.72 for D/d=1.5 and r/d=0.1.

7. **Richardson extrapolation**: use the finest 3 data points. For quadratic elements, assume convergence order p=2. The extrapolated value is: f_exact = f1 + (f1 - f2) / (r^p - 1), where r is the refinement ratio and f1 is the finest mesh result.

8. **Convergence from below is expected** for stress -- peak stress typically increases monotonically toward the true value as the mesh is refined. If stress decreases at a refinement level, suspect mesh quality issues.

9. **If convergence is non-monotonic**, check element aspect ratios at the fillet. Poorly shaped elements at the curved fillet surface will produce erratic stress values. Aspect ratios should stay below 3:1.

10. **Report asymptotic convergence rate** using a log-log plot of error vs. element size. The slope should approach the theoretical value of p=2 for quadratic elements.

11. **Mesh the fillet root with at least 8 elements around the fillet arc** at the finest level. Insufficient circumferential resolution at the fillet will under-predict the peak stress regardless of element size.

12. **Boundary effects**: ensure the fixed BC is far enough from the fillet (L > 3D, i.e., > 180mm) to not influence the stress field at the fillet. The 100mm length on each side satisfies this for the large diameter but is borderline -- verify by checking that stress at the fixed end is purely from the applied BC, not from fillet proximity.

13. **Compare element count vs. solve time** to show the cost-benefit of each refinement level. The finest mesh may take 10x longer but only improve accuracy by 0.5% -- this demonstrates diminishing returns and justifies stopping.
