# Guide: ASME VIII-2 Stress Linearization in Ansys Mechanical

1. **SCL placement**: Place stress classification lines through the thickest section at the structural discontinuity, oriented perpendicular to the mid-surface. An incorrectly oriented SCL will give wrong membrane/bending decomposition.

2. **WRC 429 recommended locations**: For a radial nozzle in a cylindrical vessel, place SCLs at 0, 90, 180, and 270 degrees around the nozzle-shell junction. The 0 and 180 degree positions (longitudinal plane) typically show the highest stresses.

3. **Reference SCL placement**: Place the reference SCL at least 2.5 times sqrt(R*t) away from the nozzle centerline. This ensures the stress field is not influenced by the nozzle discontinuity. For this vessel: 2.5 * sqrt(750*30) = 375mm from the nozzle center.

4. **Use stress intensity (Tresca), not von Mises**. ASME VIII Division 2 explicitly requires stress intensity for all stress category comparisons. Using von Mises will give non-conservative results by approximately 15%.

5. **Pm (primary membrane)**: The constant (average) stress through the wall thickness. Compare against Sm. This is the most critical check — exceeding Sm for Pm indicates insufficient wall thickness.

6. **Pm+Pb (primary membrane plus bending)**: The linear stress distribution through the wall thickness. Compare against 1.5 times Sm. Bending stresses are self-limiting to some extent, hence the higher allowable.

7. **Pm+Pb+Q (primary plus secondary)**: The total linearized stress. Compare against 3 times Sm. Secondary stresses (Q) are displacement-controlled and self-limiting. The 3Sm limit ensures shakedown to elastic behavior under cyclic loading.

8. **F (peak stress)**: Computed as the total stress minus the linearized stress (Pm+Pb+Q). Peak stress is used only for fatigue assessment, not for static strength checks.

9. **Sm for SA-516 Gr 70 at 250 degrees C**: Look up in ASME II-D Table 1A. The value is approximately 136 MPa. This is the allowable stress intensity for the vessel shell.

10. **Sm for SA-106 Gr B at 250 degrees C**: Look up in ASME II-D Table 1A. The value is approximately 118 MPa. This is the allowable stress intensity for the nozzle.

11. **At the nozzle-to-shell junction**: Both vessel and nozzle materials are present. Use the lower Sm value (118 MPa for SA-106 Gr B) for conservatism, unless the SCL lies entirely within one material.

12. **Ke factor**: Computed per ASME VIII-2 Table 5.13 for simplified elastic-plastic fatigue screening. Ke adjusts the elastic peak stress to account for local plasticity. It is only needed if the Pm+Pb+Q exceeds 3Sm.

13. **Linearization method**: Use through-thickness integration for stress decomposition, not nodal averaging. Ansys Mechanical's built-in stress linearization tool performs this correctly when the SCL endpoints are defined properly.

14. **Sanity check on the reference SCL**: The membrane stress at the reference SCL should be close to the hoop stress P*R/t. For this vessel: 3.5 * 750 / 30 = 87.5 MPa. If the FEA result deviates by more than 10%, investigate the mesh or boundary conditions.

15. **Stress classification at nozzle junctions**: Only primary membrane Pm has a hard failure limit. Exceeding 1.5Sm for Pm+Pb is a warning that typically requires design review but is not always an automatic failure if it can be shown that the bending is secondary.

16. **Report format**: Create one detailed table per SCL showing the through-thickness stress distribution, and a summary table comparing all 5 SCLs side by side. Include a figure showing the SCL locations on the model for traceability.

17. **Always include a figure showing SCL locations on the model**. Reviewers (and future engineers) need to see where the SCLs are placed to assess whether the locations are appropriate.

18. **Mesh adequacy at SCLs**: Verify at least 6 elements through the wall thickness at each SCL location. Fewer elements degrade the accuracy of the stress linearization, particularly the bending component.

19. **If peak stress is high**: When the peak stress F is significantly higher than 3Sm, recommend a detailed fatigue analysis (full elastic-plastic or strain-life approach) rather than relying on the simplified Ke factor method. The Ke approach is a screening tool, not a definitive fatigue assessment.
