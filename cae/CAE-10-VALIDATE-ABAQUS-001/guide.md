# Guide: Modal Correlation and Model Updating

## Tribal Knowledge

1. **MAC formula**: MAC(i,j) = |{phi_A}^T {phi_X}|^2 / ({phi_A}^T{phi_A} * {phi_X}^T{phi_X}). Values range from 0 (no correlation) to 1 (perfect correlation). Compute for all 8x8 FEA-vs-experimental mode pairs.

2. **MAC interpretation thresholds**: MAC > 0.9 is good correlation, 0.7-0.9 is moderate (may indicate mode shape differences at specific locations), MAC < 0.7 is poor (likely wrong mode pairing or significant model error).

3. **Over-predicted frequencies indicate over-stiff joints** -- this is the most common issue in bolted structure models. Rigid connections (RBE2/KINEMATIC COUPLING) assume infinite joint stiffness, which is never true physically.

4. **CBUSH element stiffness starting values**: translational K = 1e6 N/mm in all 3 DOFs, rotational K = 1e4 N-mm/rad in all 3 DOFs. These are reasonable midpoints for M8-M12 bolted flange joints.

5. **Update translational stiffness first** -- rotational DOFs have significantly less influence on global bending and torsion modes of the cross-car beam. Only adjust rotational stiffness if translational updates alone cannot achieve the target correlation.

6. **Sensitivity study**: perturb each joint group's stiffness by +/-50% and record the frequency change for each mode. Joints with the largest influence on poorly-correlated modes are the best candidates for updating.

7. **Update one joint group at a time** (e.g., all A-pillar joints, then all tunnel joints) rather than all 12 simultaneously. This maintains physical understanding of which joints are over-constrained and keeps the inverse problem well-posed.

8. **Typical bolt joint stiffness ranges**: M6 bolts in sheet metal: 1e4-1e5 N/mm. M8-M10 bolts in cast brackets: 1e5-1e6 N/mm. M10-M12 bolts in thick flanges: 1e6-1e7 N/mm. Values outside 1e4-1e8 N/mm are physically unreasonable.

9. **Mode pairing: use MAC values, not frequency proximity.** Modes can cross (change order) between FEA and test, especially when multiple modes are closely spaced. The MAC matrix reveals the correct pairing even when frequencies don't match.

10. **Cross-MAC terms > 0.3** indicate coupled modes -- this may be physical (the structure has closely-spaced modes that couple) or may indicate a problem with the experimental data (poor sensor placement, spatial aliasing).

11. **Damping is not needed for frequency correlation** -- use undamped natural frequencies for both FEA and experimental comparison. However, report the experimental damping ratios for reference as they indicate joint behavior (higher damping = more joint friction).

12. **Extract FEA results at sensor DOFs only** -- use *NODE FILE output or create a node set matching the 24 sensor locations. Do not compare full-field FEA mode shapes against 24-point experimental shapes; reduce the FEA to the sensor DOFs first.

13. **Frequency error formula**: error = (f_FEA - f_test) / f_test x 100%. Positive error means FEA over-predicts (typical for over-stiff joints). Target: |error| < 5% for at least 6 of 8 modes.

14. **Plot mode shapes side-by-side** (FEA vs. experimental) for visual confirmation of MAC-based pairing. Numerical MAC can occasionally be misleading if sensor coverage is sparse.

15. **Orthogonality check on experimental data**: compute the experimental auto-MAC matrix (experimental modes against themselves). Off-diagonal terms should be < 0.1. Values > 0.2 indicate poor test data quality (incomplete separation of modes, noise, or insufficient sensor count).

16. **Report the condition number of the sensitivity matrix** when performing the joint stiffness update. An ill-conditioned sensitivity matrix means the inverse problem has non-unique solutions -- multiple stiffness combinations can produce similar frequency shifts. In this case, constrain the solution using physical reasoning.

17. **Do NOT over-fit**: if achieving < 5% error on all 8 modes requires joint stiffness values outside the physically reasonable range (1e4-1e8 N/mm), stop at the best achievable correlation with reasonable values. Over-fitting to 8/8 modes with unphysical stiffness produces a model that is numerically correlated but physically wrong and will not predict correctly under modified conditions.
