# Guide: Bolted Flanged Joint Analysis in Abaqus

1. **Bolt pretension sequencing**: Use *PRE-TENSION SECTION to apply bolt load in Step 1. In Steps 2 and 3, switch the pretension node to FIXED AT CURRENT LENGTH to maintain the preload while allowing the bolt to respond to additional loading. This is the standard Abaqus bolt modeling approach.

2. **Cyclic symmetry constraints**: Apply MPC type or *EQUATION constraints on both cut faces of the 1/8 sector. The tangential degree of freedom must be coupled correctly — this is the most common source of errors in cyclic symmetry models.

3. **Gasket elements**: The gasket uses GK3D8 elements, not standard hex elements. The thickness direction must align with the gasket compression direction. Verify this orientation before defining gasket behavior.

4. **Gasket behavior data**: Import load/unload curves from the CSV file. Use *GASKET BEHAVIOR with MEMBRANE and THICKNESS DIRECTION keywords. The unloading curve defines the gasket's hysteretic behavior under bolt relaxation.

5. **End cap force**: Do not forget the end cap force. Internal pressure on a closed-end pipe creates an axial force equal to P times pi/4 times D-squared. For the 1/8 sector, divide by 8. This force acts on the pipe end faces.

6. **Contact formulation**: Use surface-to-surface contact with finite sliding and augmented Lagrange enforcement for the flange-gasket interfaces. Augmented Lagrange is more robust than pure penalty for gasket contact where precise pressure distribution matters.

7. **Contact initialization**: Adjust initial overclosure to zero. Imported meshes often have small gaps or penetrations between the flange and gasket surfaces that cause convergence issues in the first increment.

8. **Bolt cross-section definition**: Define the bolt pre-tension cut plane as a circular cross-section at the bolt mid-plane. The pre-tension node must be on this plane.

9. **Step 1 controls**: Use a static step with NLGEOM=YES. Initial increment 0.1, minimum 0.001. Bolt pretension typically converges easily but NLGEOM is needed for subsequent steps.

10. **Step 2 pressure ramping**: Use *AMPLITUDE with SMOOTH STEP to ramp the pressure gradually. Abrupt pressure application can cause convergence difficulties when combined with active contact and gasket nonlinearity.

11. **Step 3 bending application**: Apply the bending moment via a kinematic coupling on the pipe end face. The coupling reference point receives the moment load; the coupling distributes it to the face nodes.

12. **Monitor gasket contact status**: If the gasket lifts off entirely on one side during bending (Step 3), the boundary conditions or load magnitudes may be incorrect. Partial liftoff on the tension side is expected and physical.

13. **Output requests**: Request CPRESS for contact pressure, GKT for gasket closure and pressure, S for stress, and U for displacement. These are the standard outputs for bolted joint assessment.

14. **History output at bolt node**: Request history output (RF, U) at the pretension node to track how bolt force changes through all three steps. The bolt preload should drop slightly when pressure is applied — this is correct physical behavior.

15. **Verify bolt preload reduction**: When internal pressure is applied in Step 2, the bolt preload should decrease because the pressure tends to separate the flanges. If preload increases, the end cap force direction is likely wrong.

16. **Check reaction forces at symmetry planes**: The sum of reaction forces at the symmetry cut faces should equal zero in the tangential direction. A nonzero sum indicates incorrect cyclic symmetry constraint application.

17. **Gasket seating stress**: Verify that the gasket contact pressure after Step 1 meets the minimum seating stress for the gasket type (spiral wound gaskets typically require 68 MPa minimum per ASME PCC-1).

18. **Flange rotation**: Check flange rotation after each step. Excessive rotation (> 0.3 degrees per ASME PCC-1) indicates the flange is under-designed, which is a finding to report even if the analysis converges.
