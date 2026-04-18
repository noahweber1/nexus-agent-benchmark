# Design Guide: External Aerodynamic CFD Analysis

## Domain and Boundary Conditions
- Define the fluid domain extents: inlet at least 5 chord lengths upstream, outlet at least 10 chord lengths downstream, far-field boundaries at least 5 chord lengths from the airfoil surface.
- Inlet: velocity inlet with freestream velocity magnitude and direction (angle of attack applied here or via geometry rotation).
- Outlet: pressure outlet at atmospheric pressure.
- Airfoil surface: no-slip wall boundary condition.
- Far-field / lateral boundaries: symmetry or free-slip wall.

## Freestream Conditions
- Set air properties for sea-level ISA: density 1.225 kg/m3, dynamic viscosity 1.789e-5 Pa*s. Compute Reynolds number Re = rho * V * c / mu to confirm the flow regime.

## Turbulence Model
- Select a turbulence model appropriate for the Reynolds number. k-epsilon with standard wall functions is suitable for moderate-Re external flows with y+ ~ 30-300 on walls.

## Mesh
- Generate a structured or hybrid mesh with boundary layer prism/inflation layers on the airfoil surface. Size the first cell height to achieve the target y+ for the selected wall treatment.
- Refine mesh in the wake region downstream of the trailing edge to capture the drag accurately.

## Convergence
- Monitor residuals (continuity, momentum, turbulence quantities) and integrated force coefficients (Cl, Cd) for convergence. Solution is converged when residuals drop below 1e-4 and force coefficients stabilize to within 0.1% over the last 100 iterations.

## Post-Processing
- Extract lift and drag from integrated surface pressure and wall shear stress. Compute Cl = L / (0.5 * rho * V^2 * A) and Cd = D / (0.5 * rho * V^2 * A).
- Plot pressure coefficient Cp = (p - p_inf) / (0.5 * rho * V^2) along the chord on upper and lower surfaces.
