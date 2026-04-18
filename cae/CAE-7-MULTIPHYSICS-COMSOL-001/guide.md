# Guide: Thermo-Mechanical Analysis of Power Electronics Heatsink

## Tribal Knowledge

1. **Always model the TIM layer explicitly** -- it dominates thermal resistance at the IGBT-to-heatsink interface. Omitting it or using a thermal contact resistance approximation will significantly under-predict junction temperatures.

2. **TIM thickness: use actual compressed thickness (0.1mm)**, not free-standing thickness. Under bolt preload, TIM compresses and its effective conductivity may differ from datasheet values. The specified k=5 W/(m-K) is for the compressed state.

3. **Convection coefficient h=45 W/(m2-K) is for moderate forced air** (typical of a 80-120 CFM fan at 2-3 m/s air velocity over the fins). In practice, validate this against the fan curve and heatsink pressure drop. For this eval, use the given value.

4. **Fin efficiency**: COMSOL calculates this automatically if fins are properly meshed, but verify by checking that fin tip temperatures are lower than fin base temperatures. If fin tips are at ambient, the fins may be too long for the given h.

5. **Reference temperature for thermal stress**: the stress-free state is at 25 degrees C (assembly/room temperature), not at the operating ambient of 40 degrees C. Thermal strain = CTE * (T - T_ref) where T_ref = 25 degrees C.

6. **Bolt constraints**: use fixed displacement on the bolt hole cylindrical surfaces, not point constraints. Point constraints create artificial stress singularities. Cylindrical surface constraints distribute the reaction over a realistic area.

7. **Symmetry**: if the IGBT layout and fin geometry are symmetric about a plane, use a half-model for computational efficiency. However, report results for the full model (double any half-model integrated quantities).

8. **Mesh: at least 3 elements through TIM layer thickness** (0.1mm thickness means ~0.033mm element size in the TIM). This is critical for capturing the thermal gradient across the TIM.

9. **Mesh: at least 3 elements through each fin thickness** (1.5mm thickness means ~0.5mm element size in the fins). Under-meshing fins will miss the temperature gradient across the fin and affect both thermal and structural results.

10. **Check energy balance**: the total heat leaving the model via convection (surface integral of h*(T-T_amb)) should equal the total heat input (360W). Imbalance > 1% indicates a modeling error (missing BC, internal heat generation error, etc.).

11. **Al 6063-T5 properties**: E=69 GPa, nu=0.33, CTE=23.4e-6/C, k=209 W/(m-K), density=2700 kg/m3. Use these consistently for both thermal and structural physics.

12. **Deflection limit of 0.05mm**: this is to maintain TIM contact pressure. Excessive base plate bowing (cupping) can create an air gap at the TIM edges, dramatically increasing thermal resistance. Report the deflection pattern (cupping vs. doming).

13. **Thermal-structural coupling direction**: one-way coupling (thermal to structural) is usually sufficient because the deformation is small and does not significantly change the thermal conduction paths. Two-way coupling adds solve time with negligible accuracy improvement for this class of problem.

14. **Plot temperature along a line through each IGBT center** perpendicular to the base plate to verify the thermal drop across the TIM layer. The TIM should account for 40-60% of the total thermal resistance from IGBT pad to fin tip.

15. **Report IGBT pad average temperature, not just peak** -- the junction temperature is: T_junction = T_pad_avg + R_jc * P, where R_jc is the junction-to-case thermal resistance of the IGBT module (typically 0.2-0.5 K/W). If R_jc is not provided, report pad temperature as a lower bound on junction temperature.
