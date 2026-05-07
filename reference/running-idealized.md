# Running Idealized Cases

WRF ships with a dozen idealized test cases that need no external boundary
data, no WPS, and no GRIB. They are the right starting point for testing a
build, validating a code change, prototyping a new physics scheme, and
running classroom demonstrations. Each case has its own initialization
module under `dyn_em/module_initialize_<case>.F` and its own test directory
under `test/<case>/`.

---

## How idealized cases differ from real

| Aspect | `em_real` | `em_<idealized>` |
|--------|-----------|------------------|
| Initialization | `real.exe` reads `met_em.*` from WPS | `ideal.exe` calls a per-case `module_initialize_<case>.F` to construct the initial state analytically or from a sounding |
| Boundary conditions | Specified, from `wrfbdy_d01` | Periodic, symmetric, or open. No BC file needed |
| Static data | terrain, soil from WPS geogrid | analytical (flat surface, idealized hill, ...) |
| Vertical coordinate | usually HVC (`hybrid_opt=2`) | usually TF (`hybrid_opt=0`); some cases support HVC since v4.0 |
| Physics | full suite | usually a minimal subset |
| Compile target | `./compile em_real` | `./compile em_<case>` |

You cannot have both `real.exe` and `ideal.exe` from a single
`./compile`. Switching between cases requires `./clean -a && ./compile
em_<other_case>`.

---

## Available idealized cases

The full list (from `doc/README.test_cases` and the WRF top-level README,
WRF v4.7.1):

| Compile target | Test directory | Dimensions | Physics | What it is |
|----------------|----------------|------------|---------|------------|
| `em_b_wave` | `test/em_b_wave/` | 3D | dry | Baroclinically unstable jet on f-plane. 100 km grid, 16 km top, 41 x 81 x 64. Symmetric N/S, periodic E/W |
| `em_quarter_ss` | `test/em_quarter_ss/` | 3D | with mp | Classic Weisman-Klemp quarter-circle shear supercell. Produces left + right movers |
| `em_les` | `test/em_les/` | 3D | LES | Free convective boundary layer LES. No PBL scheme; sub-grid turbulence handled in dynamics |
| `em_grav2d_x` | `test/em_grav2d_x/` | 2D (x,z) | dry | Straka et al 1993 dense-bubble gravity current |
| `em_hill2d_x` | `test/em_hill2d_x/` | 2D (x,z) | dry | Linear hydrostatic flow over a 100 m bell-shaped hill. Tests open BC and damping layer |
| `em_squall2d_x` | `test/em_squall2d_x/` | 2D (x,z) | Kessler mp | Squall line in x. Periodic in y so 3D solver runs as 2D |
| `em_squall2d_y` | `test/em_squall2d_y/` | 2D (y,z) | Kessler mp | Same but in y |
| `em_seabreeze2d_x` | `test/em_seabreeze2d_x/` | 2D (x,z) | full physics | Sea-breeze; demo for full physics in 2D |
| `em_heldsuarez` | `test/em_heldsuarez/` | 3D global | dry | Held-Suarez 1994 forcing on a global lat-lon grid. Tests the global solver |
| `em_tropical_cyclone` | `test/em_tropical_cyclone/` | 3D | full | Idealized TC on f-plane, constant SST, Rotunno-Emanuel vortex. Useful for testing physics changes for TCs |
| `em_scm_xy` | `test/em_scm_xy/` | column | full | Single-column model. Forced by external sounding, runs only column physics |
| `em_convrad` | `test/em_convrad/` | 3D | full at cloud-resolving | Convective-radiative equilibrium, periodic, tropical |
| `em_fire` | `test/em_fire/` | 3D | dry + SFIRE | Coupled atmosphere + wildfire (level-set based fire spread) |

---

## The build + run cycle

```bash
# Pick a case, e.g. quarter-circle supercell
./clean -a
./configure                       # interactive; pick a parallel mode
./compile em_quarter_ss >& compile.log
ls main/ideal.exe main/wrf.exe
ls test/em_quarter_ss/ideal.exe   # symlink

cd test/em_quarter_ss
ls
# input_sounding namelist.input ideal.exe wrf.exe ...

./ideal.exe                       # writes wrfinput_d01
ls wrfinput_d01

./wrf.exe                         # serial; or mpirun -np N ./wrf.exe
ls wrfout_d01_*
```

`ideal.exe` in serial finishes in seconds. `wrf.exe` for `em_quarter_ss`
on a single core takes 5-10 minutes for the default 2 hour simulation.

`em_b_wave` is the canonical "smoke test" for a fresh build. It is
small (41 x 81 x 64), dry (no microphysics), serial-friendly, and
finishes in about a minute on a modern laptop.

---

## Per-case configuration files

Most idealized cases have:
- `namelist.input`: case-specific defaults (smaller than the em_real namelist).
- `input_sounding`: the initial vertical profile (T, q_v, U, V vs height).
  Simple plain-text ASCII. Edit this to start the model from a different
  sounding.
- `LANDUSE.TBL`, `SOILPARM.TBL`, etc: standard parameter tables, only
  read when the matching physics is enabled.

Some cases have additional files. `em_les` has an `input_sounding` plus
namelist controls for the surface heat flux that drives the convection.
`em_seabreeze2d_x` has a fairly elaborate namelist that turns on full
physics in a 2D vertical plane.

---

## Reading a typical idealized namelist

`em_b_wave/namelist.input`:

```
&time_control
 run_hours       = 12,
 history_interval = 60,           ! min
 frames_per_outfile = 1000,
 io_form_history = 2,
/
&domains
 time_step       = 600,           ! big timestep, low CFL because dx=100 km
 max_dom         = 1,
 e_we            = 41,
 e_sn            = 81,
 e_vert          = 64,
 dx              = 100000,        ! 100 km
 dy              = 100000,
 ztop            = 16000,
/
&physics
 mp_physics              = 0,     ! dry
 cu_physics              = 0,
 ra_lw_physics           = 0,
 ra_sw_physics           = 0,
 bl_pbl_physics          = 0,
 sf_sfclay_physics       = 0,
 sf_surface_physics      = 0,
/
&dynamics
 rk_ord          = 3,
 diff_opt        = 1,
 km_opt          = 4,
 damp_opt        = 0,
 zdamp           = 5000,
 dampcoef        = 0.2,
/
&bdy_control
 periodic_x      = .true.,        ! periodic E/W
 symmetric_ys    = .true.,        ! symmetric southern boundary
 symmetric_yt    = .true.,        ! symmetric northern boundary (for f-plane channel)
/
```

Note `ztop` is used for some idealized cases that build the vertical grid
from a top elevation rather than from `p_top_requested`.

---

## When to use which case

| If you want to ... | Run |
|--------------------|-----|
| Smoke test a fresh WRF build | `em_b_wave` (60 s) |
| Validate a microphysics change | `em_squall2d_x` or `em_quarter_ss` |
| Validate a PBL change | `em_les` (no PBL scheme; sub-grid turbulence direct) |
| Validate a radiation change | `em_convrad` (periodic 3D RCE) |
| Validate a surface layer change | `em_seabreeze2d_x` |
| Test the global solver | `em_heldsuarez` |
| Test moving nests | `em_tropical_cyclone` |
| Test wildfire coupling | `em_fire` |

---

## Custom idealized initial conditions

To build a new idealized case:

1. Copy an existing `dyn_em/module_initialize_<case>.F` to a new file,
   say `dyn_em/module_initialize_mycase.F`.
2. Edit the `init_domain_1` (or `init_domain`) subroutine to construct
   the initial state you want.
3. Add a corresponding `compile em_mycase` target. The cleanest approach
   is to copy a similar case stanza in `Makefile` and `arch/postamble`.
4. Create `test/em_mycase/` with `namelist.input`, `input_sounding`,
   and any tables you need.
5. `./clean -a && ./configure && ./compile em_mycase`.
6. `cd test/em_mycase && ./ideal.exe && ./wrf.exe`.

Most users instead modify an existing case in place
(e.g., a different sounding for `em_squall2d_x`) rather than building a
new compile target.

---

## Where to next

- ARW dynamics that drive these cases: `dynamic-cores-and-em.md`
- Physics that you can turn on: `physics-options.md`
- Real-data alternative: `running-real-case.md`
- A case is unstable / produces zero output: `debugging.md`
