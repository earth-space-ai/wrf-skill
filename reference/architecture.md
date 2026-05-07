# WRF Architecture

WRF is built around a clean separation between a **driver layer** that
manages parallelism, I/O, time, and nesting, and a **model layer** that
implements the physics and dynamics. Glueing the two together is the
**Registry**, a declarative table-based system that auto-generates Fortran
code for state-variable definitions, I/O streams, namelist handling, and
nest interpolation.

Read this page before editing any WRF source file. The Registry trips up
every new contributor at least once.

---

## Top-level source tree

```
WRF/
|-- arch/              <- per-platform configure stanzas, postamble, preamble
|-- chem/              <- WRF-Chem: gas chemistry, aerosols, emissions
|-- cmake/             <- CMake build system (alternative to ./configure)
|-- compile            <- top-level shell wrapper around make
|-- configure          <- top-level configure script (writes configure.wrf)
|-- doc/               <- README files for sub-features (README.namelist, ...)
|-- dyn_em/            <- ARW dynamical core (~40 files)
|-- external/          <- I/O bindings, communication libraries
|-- frame/             <- driver layer: domain manager, decomposition, nesting
|-- hydro/             <- WRF-Hydro coupling library
|-- inc/               <- Registry-generated headers (gitignored content)
|-- main/              <- top-level main programs (wrf.F, real_em.F, ...)
|-- phys/              <- physics suite (~235 files)
|-- Registry/          <- the declarative registry tables
|-- run/               <- packaged namelists, parameter tables, lookup files
|-- share/             <- I/O mediation, output streams, common helpers
|-- test/              <- per-case test directories (em_real, em_b_wave, ...)
|-- tools/             <- the Registry compiler and assorted code-gen tools
|-- var/               <- WRFDA: 3DVar, 4DVar, hybrid
+-- wrftladj/          <- WRFPLUS: tangent linear and adjoint
```

File counts (WRF 4.7.1):

| Subdirectory | Approx file count |
|--------------|-------------------|
| `phys/`      | 235 |
| `chem/`      | 203 |
| `share/`     | 64  |
| `frame/`     | 64  |
| `Registry/`  | 40  |
| `dyn_em/`    | 41  |
| `arch/`      | 17  |
| `main/`      | 14  |

---

## The Registry

The Registry is the most important concept in WRF. Files under `Registry/`
are not Fortran. They are declarative tables read by `tools/registry`
(a small C program built before the main compile), which writes Fortran
header files into `inc/`. Those headers are then `#include`d by hand-written
Fortran in `frame/`, `share/`, `dyn_em/`, and `phys/`.

### What the Registry controls

| Concern | Registry file (typical) |
|---------|-------------------------|
| Dimension specifications (i, j, k, soil layers, ...) | `registry.dimspec` |
| State variables (3D / 2D / 4D fields) and their I/O | `Registry.EM_COMMON`, `Registry.EM`, `Registry.EM_CHEM` |
| Hybrid sigma-pressure coordinate fields | `registry.hyb_coord` |
| Namelist options and their default values | `Registry.EM_COMMON` (`rconfig` lines) |
| Two-way nest interpolation rules | `Registry.EM_COMMON` (state-variable `IO` flags) |
| Chemistry packages (RADM2, CBMZ, MOZART, ...) | `registry.chem`, `registry.wrfchemvar` |
| Stochastic perturbations | `registry.stoch` |
| WRF-Hydro coupling fields | (from `hydro/` Registry tables) |
| WRFDA variables | `registry.var`, `Registry.wrfvar` |
| WRFPLUS (tangent linear) | `Registry.tladj`, `registry.wrfplus` |

There are roughly 40 Registry files total. The master file selected for
each build is named in `arch/postamble`. For ARW it is `Registry.EM`,
which `include`s `Registry.EM_COMMON` plus a series of subsystem registries.

### Registry record types

A Registry table is a series of whitespace-delimited records. The four most
common record types:

| Record  | Meaning |
|---------|---------|
| `dimspec` | Define a dimension (name, axis, source) used by state variables |
| `state`   | Define a model state variable (rank, dimensions, I/O streams, units, description) |
| `i1`      | Define a per-tile temporary (intermediate) variable, not output |
| `rconfig` | Define a namelist option (group, type, default value, max-domain replication) |
| `package` | Define a "package" of state and rconfig items grouped under a single switch (e.g. `mp_physics==6` enables WSM6) |
| `halo`, `period` | Define communication patterns for distributed-memory parallelism |

### A `state` record example

From `Registry.EM_COMMON`:

```
state    real  XLAT             ij      misc        1   -   i0123rh0156d   "XLAT"  "LATITUDE, SOUTH IS NEGATIVE"  "degree_north"
```

Reading the columns: type `real`, name `XLAT`, dimensions `ij` (2D, mapped to
the i,j horizontal indices via `registry.dimspec`), used by `misc`,
`NumTLev=1` (no time levels), no staggering, **I/O flags**
`i0123rh0156d` (input on streams 0, 1, 2, 3; restart; history on streams
0, 1, 5, 6; written by netCDF-4 `d` flag). Then the netCDF variable name,
description, units.

After `./compile`, that one line generates:
- An entry in the `domain` derived type so `grid%xlat(i,j)` is a real
  Fortran allocation.
- A read on each input stream that has `0123` set.
- A write on each history / restart stream where the I/O flag is set.
- Boundary copies and (if interpolation rules are present) two-way-nest
  interpolation calls.

This is why **editing the Registry without `./clean -a` corrupts the build**:
the headers in `inc/` are stale, so the auto-generated I/O code references
a different `domain` type than the hand-written code that uses it.

### An `rconfig` namelist record example

```
rconfig   integer mp_physics              namelist,physics    max_domains    0       irh   "mp_physics"   "microphysics scheme"   ""
```

This declares a namelist variable `mp_physics` in the `&physics` group, of
type `integer`, with `max_domains` replicates (one per nest), default 0,
written to input + restart + history headers as `mp_physics`.

The full README on Registry semantics is the inline comments at the top of
`Registry.EM_COMMON`. There is no formal Registry manual; the source is the
spec.

---

## The build system: `./configure` and `./compile`

WRF has two build systems in the same tree.

### Classic: `./configure` + `./compile`

This is the production path used by 99% of WRF runs.

1. **`./configure`** is a tcsh wrapper. It sources `arch/Config.pl` (a Perl
   script), which:
   - Reads `arch/configure.defaults` (long list of stanzas, one per OS +
     compiler + parallelism combination, see `arch/README.canonical_stanza`).
   - Prompts the user for choice of stanza and nesting type.
   - Reads `arch/preamble` and `arch/postamble`.
   - Substitutes paths from `$NETCDF`, `$HDF5`, `$JASPER`, etc.
   - Writes `configure.wrf` in the source root.

2. **`./compile <case>`** is a tcsh wrapper around `make`. It:
   - Builds `tools/registry`.
   - Runs `tools/registry Registry/Registry.<core>` to generate headers in `inc/`.
   - Recursively builds `external/`, `frame/`, `share/`, `phys/`, `dyn_em/` (or
     `dyn_nmm/`), `chem/` (if `WRF_CHEM=1`), then the case-specific main in
     `main/`.
   - Links the executables (`wrf.exe`, plus `real.exe` or `ideal.exe`,
     plus `ndown.exe` and `tc.exe` for em_real) and symlinks them into
     `test/<case>/`.

Available cases (run `./compile` with no arguments to list):

```
compile em_b_wave              compile em_real
compile em_grav2d_x            compile em_scm_xy
compile em_heldsuarez          compile em_seabreeze2d_x
compile em_hill2d_x            compile em_squall2d_x
compile em_les                 compile em_squall2d_y
compile em_quarter_ss          compile em_tropical_cyclone
compile em_convrad             compile nmm_real
```

### Modern: CMake (alternative)

There is also a top-level `CMakeLists.txt` plus a `cmake/` directory. The
CMake build is documented in `doc/README.cmake_build`. The classic
`./configure` path is the supported default; the CMake path exists for
groups that want a more standard build system but is still maturing. For
this skill, treat `./configure` as canonical.

### `./clean` levels

| Command | Removes |
|---------|---------|
| `./clean` | Object files only. Keeps `configure.wrf` and Registry-generated headers. |
| `./clean -a` | Everything: objects, executables, `configure.wrf`, generated headers in `inc/`. Forces a from-scratch rebuild. |

After Registry edits or `configure.wrf` edits: `./clean -a`. After a small
physics edit: `./clean` is enough, but `./clean -a` is safer.

---

## The driver layer: `frame/` and `share/`

`frame/` and `share/` together form WRF's **driver layer**, isolated from
the science modules in `phys/` and `dyn_em/`.

| Directory | What it does |
|-----------|--------------|
| `frame/module_domain.F` | Defines the `domain` derived type. The single biggest type in WRF. |
| `frame/module_integrate.F` | Top-level time integration loop. Per timestep, calls `solve_em` and handles I/O alarms, restart, nesting. |
| `frame/module_io_quilt.F` | Asynchronous I/O quilt servers (a subset of MPI ranks dedicated to writing output). |
| `frame/module_dm.F` | Distributed-memory abstraction (RSL_LITE wrapper). |
| `frame/module_configure.F` | Namelist read; auto-generated from Registry `rconfig` lines. |
| `share/mediation_integrate.F` | Mediates between driver and model layers per timestep. |
| `share/output_wrf.F`, `share/input_wrf.F` | Auto-generated I/O glue from Registry I/O flags. |
| `share/wrf_fddaobs_in.F`, `share/wrf_tsout.F` | Specialized I/O streams (FDDA, time series). |

Most user code edits do not touch `frame/` or `share/`. Edits there are
needed for new auxiliary streams, new I/O formats, or changes to nesting
behavior.

---

## The I/O layer: `external/`

| Subdirectory | Purpose |
|--------------|---------|
| `external/io_netcdf/` | NetCDF-3 / NetCDF-4 classic I/O |
| `external/io_netcdfpar/` | Parallel NetCDF-4 (HDF5 backend) |
| `external/io_pnetcdf/` | Parallel NetCDF-3 (Argonne pnetCDF) |
| `external/io_grib1/` and `external/io_grib2/` | GRIB output |
| `external/io_int/` | "Internal" raw-binary format (used between WPS and real) |
| `external/io_phdf5/` | Parallel HDF5 |
| `external/RSL_LITE/` | Distributed-memory communication layer (halo exchange, nest comms) |
| `external/esmf_time_f90/` | ESMF time management (drift-free fractional timesteps) |

`io_form_input`, `io_form_history`, `io_form_restart`, `io_form_boundary`,
and `io_form_auxinput<N>` / `io_form_auxhist<N>` in `&time_control` map
integers to these backends:

| Value | Backend |
|-------|---------|
| 1 | binary (internal) |
| 2 | netCDF (default) |
| 4 | parallel HDF5 (PHDF5) |
| 5 | GRIB1 |
| 10 | GRIB2 |
| 11 | parallel NetCDF (pnetCDF) |
| 102 | netCDF, split into one file per task (used for very large grids) |

---

## The model layer: `dyn_em/` and `phys/`

The two halves of the science.

### `dyn_em/`: ARW Eulerian Mass core

Forty-one files. Highlights:

| File | Role |
|------|------|
| `solve_em.F` | The heart of the ARW core. Per timestep: RK3 outer loop, advection, small-step acoustic, physics calls. |
| `module_first_rk_step_part1.F` and `module_first_rk_step_part2.F` | Physics calls inside the first RK step. |
| `module_after_all_rk_steps.F` | Bookkeeping after the time integration. |
| `module_advect_em.F` | Scalar advection (2nd to 6th order, WENO, monotonic, positive-definite). |
| `module_small_step_em.F` | Time-split small step for acoustic and gravity-wave modes. |
| `module_big_step_utilities_em.F` | Big-step coefficients, pressure gradient, Coriolis. |
| `module_diffusion_em.F` | Explicit diffusion options. |
| `module_damping_em.F` | Rayleigh damping at the model top. |
| `module_initialize_real.F` | Initialization for real-data cases. |
| `module_initialize_ideal.F`, `module_initialize_<case>.F` | Per-idealized-case initialization. |
| `start_em.F` | Per-domain startup. |
| `module_bc_em.F` | Lateral boundary application. |

Read more in `dynamic-cores-and-em.md`.

### `phys/`: physics suite

By prefix convention, every physics module starts with `module_<group>_<scheme>.F`:

| Prefix | Domain |
|--------|--------|
| `module_mp_*` | microphysics (Kessler, WSM, Lin, Thompson, Morrison, NSSL, P3, ...) |
| `module_cu_*` | cumulus parameterization (KF, BMJ, Grell-Devenyi, Grell 3D, GF, Tiedtke, NTiedtke, SAS, ZM, ...) |
| `module_bl_*` | planetary boundary layer (YSU, MYJ, MYNN, ACM2, BouLac, QNSE, TEMF, GBM, Shin-Hong) |
| `module_ra_*` | radiation (RRTM, RRTMG, CAM, Goddard, FLG, Dudhia) |
| `module_sf_*` | surface layer + LSM (MM5, Eta, MYNN, Pleim-Xu, Noah, Noah-MP, RUC, CLM4, SSiB, urban) |
| `module_fr_*` | fire spread (SFIRE) |
| `module_ltng_*` | lightning |
| `module_diag_*` | runtime diagnostics |
| `module_check_*` | runtime sanity checks |
| `module_*_driver.F` | per-group driver that dispatches to the chosen scheme based on the namelist option |

Each driver (e.g. `module_microphysics_driver.F`) reads `mp_physics` from
`grid%mp_physics` and calls the right scheme. Adding a new microphysics
scheme: write `phys/module_mp_yourscheme.F`, register a new package value
in the Registry, add a `case` to the driver.

Read more in `physics-options.md`.

### `chem/`: WRF-Chem

Parallel structure to `phys/`. Drivers in `chem_driver.F`,
`emissions_driver.F`, `aerosol_driver.F`, then dozens of mechanism modules
(`module_kpp_*` for KPP-generated, `module_mosaic_*` for MOSAIC aerosols,
`module_cam_mam_*` for CESM MAM, `module_aerosols_sorgam_*` for SORGAM).

Read more in `wrf-chem.md`.

---

## The main programs in `main/`

WRF builds several executables, all linked from the same library tree:

| Executable | Source | Purpose |
|------------|--------|---------|
| `wrf.exe` | `wrf.F` + `module_wrf_top.F` | The model integrator |
| `real.exe` | `real_em.F` | Real-data initialization (reads `met_em.*`, writes `wrfinput`, `wrfbdy`) |
| `ideal.exe` | `ideal_em.F` | Idealized-case initialization (one of the `module_initialize_*.F` modules) |
| `ndown.exe` | `ndown_em.F` | Offline one-way nesting from a coarser run |
| `tc.exe` | `tc_em.F` | Tropical cyclone bogussing |
| `convert_em.exe` | `convert_em.F` | Convert `wrfinput` between coordinate options |

The NMM equivalents are `real_nmm.F`, `ideal_nmm.F`. They are kept for
historical reasons.

---

## The `run/` directory

After build, `wrf.exe`, `real.exe`, `ideal.exe`, `ndown.exe`, `tc.exe` are
symlinked into `run/` alongside the parameter tables and lookup files. A
non-exhaustive inventory:

| File | Purpose |
|------|---------|
| `namelist.input` | Default real-data namelist (very large) |
| `RRTMG_LW_DATA`, `RRTMG_SW_DATA` | RRTMG radiation absorption coefficients |
| `RRTM_DATA` | RRTM (older) absorption coefficients |
| `CAM_ABS_DATA`, `CAM_AEROPT_DATA`, `CAMtr_volume_mixing_ratio*` | CAM radiation tables and CO2/CH4/N2O scenarios |
| `LANDUSE.TBL`, `VEGPARM.TBL`, `SOILPARM.TBL`, `GENPARM.TBL` | Surface property tables (USGS / MODIS land use, vegetation parameters) |
| `URBPARM.TBL`, `URBPARM_LCZ.TBL`, `URBPARM_UZE.TBL` | Urban canopy parameters |
| `MPTABLE.TBL` | Noah-MP parameters (legacy; v5 uses `NoahmpTable.TBL`) |
| `ETAMPNEW_DATA`, `ETAMPNEW_DATA_DBL` | Eta microphysics lookup tables |
| `p3_lookupTable_*.dat-*` | P3 microphysics lookup tables |
| `tr49t67`, `tr67t85`, `tr49t85` | Goddard radiation transmission tables |
| `ozone.formatted`, `aerosol.formatted` | climatological ozone, aerosol |
| `wind-turbine-1.tbl` | Wind farm drag |

Many of these are read only when the matching physics option is enabled. A
run that uses RRTMG longwave reads `RRTMG_LW_DATA` and ignores `RRTM_DATA`.

To run, you copy or link these files into your scratch run directory. Do
not run `wrf.exe` inside the source `run/` directory: output overwrites
the area and pollutes the git checkout.

---

## Where to next

- Compile and run a case: `getting-started.md`
- ARW dynamical core internals: `dynamic-cores-and-em.md`
- Physics options: `physics-options.md`
- WRF-Chem internals: `wrf-chem.md`
- WPS to wrf.exe production workflow: `running-real-case.md`
