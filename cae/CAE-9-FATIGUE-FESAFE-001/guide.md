# Guide: Weld Fatigue Assessment per BS 7608 in fe-safe

## Tribal Knowledge

1. **Unit load approach**: the FEA was run at 1kN. fe-safe scales the stress field by the load history. The scaling factor for peak load is 50x (50kN / 1kN). Verify that the scaled peak stress at the most loaded weld toe gives physically reasonable MPa values (expected range: 50-200 MPa stress range at critical locations).

2. **Weld classification zones**: create element groups in fe-safe that match the 6 weld locations from the drawing. Each group gets its own BS 7608 SN curve assignment. Do not apply a single classification to the entire model.

3. **BS 7608 classification definitions**:
   - Class D: butt welds dressed flush, transverse to stress direction. SN curve: log(N) = 12.182 - 3*log(S) above CAFL.
   - Class F: fillet welds, toe cracking expected. SN curve: log(N) = 11.801 - 3*log(S) above CAFL.
   - Class F2: fillet welds with high local stress concentration factor. SN curve: log(N) = 11.492 - 3*log(S) above CAFL.

4. **Apply classification to elements within 1 element width of the weld toe** -- not the whole weld, not the whole model. The weld toe is where fatigue cracking initiates, and only elements at the toe experience the relevant stress field.

5. **Rainflow cycle counting**: use the ASTM E1049 standard method (4-point algorithm with residual handling). fe-safe implements this natively. Verify the rainflow matrix output shows a reasonable distribution of cycle counts vs. stress range.

6. **Mean stress correction: use R-ratio method per BS 7608**, not Goodman or Smith-Watson-Topper. BS 7608 SN curves already account for high residual stresses typical in welded joints. Applying Goodman on top of BS 7608 curves is double-counting the mean stress effect.

7. **CAFL (constant amplitude fatigue limit)**: for BS 7608 Class F, CAFL is approximately 56 MPa stress range. Class D CAFL is approximately 74 MPa. Class F2 CAFL is approximately 50 MPa.

8. **Cycles below CAFL still contribute damage** under variable amplitude loading. BS 7608 specifies a modified slope below CAFL: the SN curve continues with slope m'=m+2 (i.e., slope changes from 3 to 5). Do NOT truncate the SN curve at the CAFL for variable amplitude assessment.

9. **Thickness correction**: if plate thickness exceeds 25mm, apply the BS 7608 clause 14.2 thickness correction factor: (t/t_ref)^0.25 where t_ref=25mm. Check the bracket plate thicknesses in the model.

10. **Miner's sum threshold**: D < 1.0 is the standard pass criterion, but for safety-critical crane applications use D < 0.5 (factor of 2 on life). Report both thresholds and which applies.

11. **Report safety factor on life**: N_allowable / N_applied, where N_allowable = N_from_SN_curve and N_applied = actual cycles from the load history extrapolated to design life.

12. **fe-safe element groups**: create separate groups for each weld classification. This allows assigning different SN curves to each group in a single analysis run rather than running multiple analyses.

13. **Hot spot stress method is NOT needed** for BS 7608 nominal stress approach. Use nodal stress directly from the FEA at the weld toe. Hot spot extrapolation is a separate methodology (used in DNV-GL or IIW approaches) and should not be mixed with BS 7608.

14. **Verify peak stress from unit load case**: before running fatigue, check that the 1kN unit load produces stress values that, when scaled by 50x, give reasonable stress ranges. If the scaled peak stress exceeds yield (355 MPa for S355J2), the linear elastic assumption may be invalid.

15. **Check for compressive cycles**: BS 7608 treats full tension-tension cycles differently from tension-compression. For welded joints with high residual stress, the full stress range is used regardless of R-ratio, but verify fe-safe is applying this correctly.

16. **Sensitivity analysis**: report how cumulative damage changes if the load scaling factor is varied by +/-10% (i.e., 45x and 55x instead of 50x). This indicates the robustness of the design and sensitivity to load uncertainty.
