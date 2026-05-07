# Dynamical Cores and the ARW Eulerian Mass Solver

WRF ships with two dynamical cores in the same source tree. Only one is
actively developed.

| Core | Directory | Status | Equations |
|------|-----------|--------|-----------|
| **ARW** (Advanced Research WRF, Eulerian Mass) | `dyn_em/` | Active. Default for all new work. | Compressible, non-hydrostatic, conservative for dry mass. |
| **NMM** (Non-hydrostatic Mesoscale Model) | `dyn_nmm/` | Frozen as of WRF v4. Kept for legacy. Not bundled in newer 4.x source releases. | Same Euler equations on a different staggering and integration scheme. |

This page covers ARW. NMM is documented in `doc/README.NMM` for the
historically interested.

---

## What ARW solves

ARW integrates the fully compressible, non-hydrostatic Euler equations on
an Arakawa C-grid in flux form. The prognostic variables are:

| Symbol | Field | Notes |
|--------|-------|-------|
| u, v | horizontal winds | Staggered on C-grid (u at east-west cell faces, v at north-south faces) |
| w | vertical wind | Staggered on full eta levels |
| theta_m | moist potential temperature | Replaces dry theta in v4. `theta_m = theta * (1 + R_v/R_d * q_v)` |
| mu | dry hydrostatic surface pressure | Mass coordinate; perturbation `mu` and base `mub` |
| q_v, q_c, q_r, q_i, ... | moist scalars | Number depends on microphysics scheme |
| Plus passive scalars, TKE, chemistry tracers, ... |

The vertical coordinate is **eta**, defined either as terrain-following
(TF) or as the new hybrid sigma-pressure (HVC). See below.

---

## Time integration: time-split RK3

ARW uses a 3rd-order Runge-Kutta (RK3) outer loop with time-split small
steps for acoustic and gravity-wave modes.

```
RK3 outer loop (big timestep dt):
  for rk_step in 1..3:
    Evaluate slow tendencies (advection, Coriolis, pressure gradient,
                              physics tendencies stored from start of step)
    Inner small-step loop (acoustic timestep dts < dt):
      Update u, v, w, mu, theta_m using forward-backward acoustic scheme
        - Horizontally explicit
        - Vertically implicit
      Apply divergence damping
    End inner loop
  End outer loop
```

The split is what makes ARW efficient. The outer (big) timestep is
constrained by horizontal advection. The acoustic mode would need a much
smaller timestep if integrated explicitly; instead it is integrated with
its own short timestep `dts = dt / time_step_sound` inside the inner
loop. `time_step_sound` defaults to `4` (sometimes `6`) for typical
real-data runs.

The outer big timestep should satisfy roughly `dt <= 6 * dx_km` for stable
real-data runs, with `dx_km` the horizontal grid spacing in km. Larger
timesteps trigger **CFL violations** and the integration aborts with
`cfl violation` messages in `rsl.error.*`.

### Namelist controls

```
&domains
 time_step                  = 60      ! seconds; ~6 * dx_km for stability
 time_step_fract_num        = 0       ! integer numerator if you need a fractional dt
 time_step_fract_den        = 1
 max_dom                    = 1
 dx                         = 10000   ! grid spacing, m
 dy                         = 10000
 e_we                       = 91      ! east-west grid points (staggered)
 e_sn                       = 82
 e_vert                     = 35      ! vertical full levels (W levels)
/

&dynamics
 rk_ord                     = 3       ! 3 = RK3 (default), 2 = RK2 (rarely used)
 time_step_sound            = 4       ! acoustic substeps per RK step
 h_mom_adv_order            = 5       ! horizontal momentum advection order (3, 5)
 v_mom_adv_order            = 3       ! vertical momentum advection order
 h_sca_adv_order            = 5       ! horizontal scalar advection order
 v_sca_adv_order            = 3       ! vertical scalar advection order
 moist_adv_opt              = 1       ! 0 standard, 1 PD, 2 monotonic, 3-4 WENO
 scalar_adv_opt             = 1       ! same scheme menu for scalars
 chem_adv_opt               = 1       ! same for chemistry
 tracer_adv_opt             = 1       ! same for tracers
 tke_adv_opt                = 1
 diff_opt                   = 1       ! explicit diffusion: 0 off, 1 simple, 2 full
 km_opt                     = 4       ! eddy coefficient: 1 const, 4 horizontal Smag, 2-3 3D Smag/TKE
 damp_opt                   = 3       ! upper damping: 0 off, 2 Rayleigh, 3 implicit gravity-wave
 zdamp                      = 5000    ! damping layer depth, m
 dampcoef                   = 0.2
 epssm                      = 0.1     ! time off-centering for vertical sound waves
 non_hydrostatic            = .true.  ! .false. selects hydrostatic mode
 hybrid_opt                 = 2       ! 2 = hybrid sigma-pressure (default since v4.0); 0 = terrain-following
 etac                       = 0.2     ! transition eta to fully isobaric (HVC only)
/
```

A common safe baseline for real-data continental runs at 9-15 km grid:
`rk_ord=3`, `time_step_sound=4`, `h_*_adv_order=5`, `v_*_adv_order=3`,
`moist_adv_opt=1`, `diff_opt=1`, `km_opt=4`, `damp_opt=3`,
`hybrid_opt=2`.

---

## Vertical coordinate: hybrid sigma-pressure

Through v3.x WRF used a pure terrain-following (TF) eta coordinate. This
draws topography signatures all the way to the model lid, which is
unphysical and degrades upper-level forecasts.

WRF v3.9 added the **hybrid sigma-pressure** (HVC) option as a compile-
and run-time switch. WRF v4.0 (Summer 2018) made HVC the default and the
TF coordinate a run-time option only.

```
&dynamics
 hybrid_opt = 2     ! HVC (default)
 hybrid_opt = 0     ! pure TF
 etac       = 0.2   ! eta level above which surfaces become fully isobaric
                    ! 0.2 is globally safe; 0.3 ok for east US; 0.4 for pure ocean.
                    ! values > 0.22 cause failures over the Himalayan plateau with a 10 hPa lid.
/
```

The pivotal subtlety: **`real.exe` and `wrf.exe` must use the same
`hybrid_opt`**. The HVC code redefines `d(p_dry)/d(eta)` from a 2D quantity
(`mu(i,j)`) to a 3D quantity:

```
mu_new_3d(i,k,j) = c1(k) * mu(i,j) + c2(k)         ! base / total mu
mu_new_3d(i,k,j) = c1(k) * mu(i,j)                 ! perturbation mu
```

Almost thirty mu-related variables are involved. Mixing HVC and TF
between `real.exe` and `wrf.exe` aborts the model. If you want to know
what `wrfinput_d01` was generated with:

```
ncdump -h wrfinput_d01 | grep HYBRID
        :HYBRID_OPT = 2 ;
```

Users are also warned that the original definitions of base-state and
dry pressure are no longer generally valid in HVC. Most users find either
`P + PB` (perturbation plus base) or `p_hyd` (hydrostatic pressure) to be
satisfactory pressure substitutes.

The Registry file controlling HVC is `Registry/registry.hyb_coord`. The
canonical doc is `doc/README.hybrid_vert_coord` in the WRF source tree.

### Vertical refinement

HVC supports almost everything: nesting, moving nests, FDDA, DFI, global
domains, ndown, WRF-DA 3DVar, WRF-Chem. The one capability that does NOT
work with HVC is **vertical refinement** (different `e_vert` per nest).
Use `vert_refine_method=0` if you have HVC on.

---

## Advection schemes

ARW supports advection orders from 2nd to 6th in horizontal and vertical
directions, plus monotonic / positive-definite limiters and WENO variants.

| Namelist value | Meaning |
|----------------|---------|
| `h_mom_adv_order = 3` | upwind, default for momentum hor. (rarely used) |
| `h_mom_adv_order = 5` | 5th-order upwind, common |
| `v_mom_adv_order = 3` | 3rd-order upwind vertical |
| `h_sca_adv_order = 5` | scalars |
| `moist_adv_opt = 0` | standard advection (can produce negative q) |
| `moist_adv_opt = 1` | positive-definite (default for production) |
| `moist_adv_opt = 2` | monotonic (more diffusive but no overshoot) |
| `moist_adv_opt = 3,4` | WENO 5th order (research) |

For real-data forecasts always use `moist_adv_opt=1` at minimum. Negative
moisture from non-PD advection produces non-physical microphysics and
crashes some schemes (notably the more sensitive 2-moment ones).

---

## Horizontal grid: Arakawa C-staggering

```
       v(i,j+1)
         |
 u(i,j) -+- T(i,j) -+- u(i+1,j)
         |
       v(i,j)
```

Mass-related fields (T, theta, theta_m, mu, p, q_*, scalars, chemistry)
live at cell centers (i, j). U is staggered to east-west cell faces (i+1
unstaggered = `e_we - 1`). V is staggered to north-south faces. W is
staggered to full eta levels (`e_vert` full levels means `e_vert - 1`
half levels for mass).

When you write to a NetCDF state variable, the dimensions in `wrfout`
include `west_east`, `west_east_stag`, `south_north`, `south_north_stag`,
`bottom_top`, `bottom_top_stag` to reflect the staggering.

---

## Lateral boundary conditions

Real-data runs use **specified BCs** with a relaxation zone read from
`wrfbdy_d01` (produced by `real.exe`). Idealized runs use periodic,
symmetric, open, or radiative BCs depending on the case.

```
&bdy_control
 spec_bdy_width             = 5      ! BC zone width in cells (4 + 1 specified row)
 spec_zone                  = 1
 relax_zone                 = 4
 specified                  = .true. ! real-data (default for em_real)
 periodic_x                 = .false.
 symmetric_xs               = .false. ! .true. for em_b_wave x
 open_xs                    = .false. ! .true. for em_squall2d_x
 nested                     = .false. ! .true. on inner nests in two-way nesting
/
```

---

## Nesting

ARW supports:
- **One-way nesting** via offline `ndown.exe` (read coarse `wrfout`, write fine `wrfinput` + `wrfbdy`).
- **Two-way nesting** with feedback, multiple nest levels, integer grid ratios
  (`parent_grid_ratio = 3` or `5` typical), and per-nest time step ratios.
- **Moving nests** with either pre-set moves (`vortex_interval`, namelist
  schedule) or vortex-following (mid-level vortex tracker), used for
  hurricane runs.

Two-way nesting feedback is controlled by:

```
&domains
 max_dom                = 2
 grid_id                = 1, 2
 parent_id              = 0, 1
 i_parent_start         = 1, 30
 j_parent_start         = 1, 25
 parent_grid_ratio      = 1, 3
 parent_time_step_ratio = 1, 3
 feedback               = 1                   ! 1 = on, 0 = off
 smooth_option          = 2                   ! parent smoothing during feedback
/
```

Forcing and feedback fields per state variable are configured by the
**Registry I/O flags**. To change which variable is exchanged across nest
boundaries, edit the IO-flag column in `Registry.EM_COMMON` and
`./clean -a`-rebuild.

---

## Global modeling

ARW can be run as a global model on a latitude-longitude grid with
polar Fourier filtering. Set `polar=.true.` and configure the grid to
cover the full sphere. The global Held-Suarez test case
(`compile em_heldsuarez`, `test/em_heldsuarez/`) is the canonical example.

---

## Where to next

- Build and run a real-data case: `running-real-case.md`
- Idealized cases: `running-idealized.md`
- Physics options that plug into the dynamics: `physics-options.md`
- The Registry that owns advection options and dynamics namelist: `architecture.md`
- A run is going unstable / hitting CFL: `debugging.md`
