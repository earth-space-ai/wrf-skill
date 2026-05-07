---
name: wrf
description: >
  Self-contained guide to WRF, the Weather Research and Forecasting Model
  (wrf-model/WRF). Covers the Registry-driven build system, the Advanced
  Research WRF (ARW) dynamical core in dyn_em, the physics suite (microphysics,
  cumulus, PBL, radiation, land surface), WRF-Chem, the WPS to real.exe to
  wrf.exe production workflow, idealized cases, WRF-Hydro coupling, debugging,
  and contributing through GitHub pull requests. Progressive disclosure: start
  in this file for routing, drill into reference/ for depth.
version: 0.1.0
tags:
  - earth-science
  - atmospheric-science
  - numerical-weather-prediction
  - regional-climate-model
  - fortran
  - mpi
  - wrf
  - wrf-arw
  - wrf-chem
  - wrf-hydro
---

# WRF, the Weather Research and Forecasting Model

> **WRF** = Weather Research and Forecasting Model
> Maintainer: NCAR Mesoscale and Microscale Meteorology Laboratory (MMM)
> Source: https://github.com/wrf-model/WRF
> Users' site: https://www2.mmm.ucar.edu/wrf/users/
> Tech note (ARW v4): Skamarock et al. 2019, NCAR/TN-556+STR
> Current version in this skill: WRF 4.7.1 (April 2025 source tree)

**What WRF does:** Solves the compressible, non-hydrostatic Euler equations on
an Arakawa C-grid for limited-area or global atmospheric simulation. Two
dynamical cores ship in the same source tree: ARW (Advanced Research WRF,
`dyn_em/`) and the older NMM (`dyn_nmm/`, frozen). Couples to a wide physics
suite (microphysics, cumulus, PBL, radiation, surface layer, land surface,
urban canopy, ocean, gravity wave drag, sub-grid turbulence). Ships with
WRF-Chem (gas-phase chemistry, aerosols, emissions) and WRF-Hydro (channel and
overland routing, NWM coupling) as in-tree extensions.

**Who this skill is for:** Agents, students, and researchers who want to
download, configure, compile, run, modify, debug, and contribute to WRF.

---

## Quick Decision Tree

```
"What do I need?"
|
+-- First time. What is WRF, how do I install it?
|   -> reference/getting-started.md
|      (clone, prereqs NetCDF/HDF5/MPI/Jasper/zlib/libpng, ./configure, ./compile em_real)
|
+-- I want to understand the source layout and the Registry
|   -> reference/architecture.md
|      (arch/, dyn_em, frame, phys, chem, hydro, share, external, main, run, Registry)
|
+-- I want to know how the ARW dynamical core works
|   -> reference/dynamic-cores-and-em.md
|      (ARW vs NMM, RK3 time-split, hybrid sigma-pressure coordinate, &dynamics namelist)
|
+-- I want to pick physics options for a real-data run
|   -> reference/physics-options.md
|      (microphysics, cumulus, PBL, radiation, surface layer, land surface)
|
+-- I want to run WRF with chemistry / aerosols
|   -> reference/wrf-chem.md
|      (chem/ subdirectory, MOZART/CBMZ/RADM2, anthropogenic + biogenic emissions)
|
+-- I want to run a real-data forecast with WPS
|   -> reference/running-real-case.md
|      (geogrid -> ungrib -> metgrid -> real.exe -> wrf.exe, namelist.wps + namelist.input)
|
+-- I want to run an idealized case (b_wave, quarter_ss, LES, ...)
|   -> reference/running-idealized.md
|      (ideal.exe, the em_* test cases, no WPS)
|
+-- I want to couple with WRF-Hydro / NWM
|   -> reference/wrf-hydro-coupling.md
|      (WRF_HYDRO=1 build, hydro.namelist, channel + terrain routing)
|
+-- The build / run is failing
|   -> reference/debugging.md
|      (configure / compile failures, segfaults in physics, CFL violations, nesting, restart)
|
+-- I want to submit a pull request to wrf-model/WRF
    -> reference/contributing-pr.md
       (NCAR fork model, JIRA, regression tests, code review)
```

---

## What is in the WRF source tree

```
WRF/
|-- arch/              <- per-platform configure stanzas + postamble
|-- chem/              <- WRF-Chem: gas chemistry, aerosols, emissions (200+ files)
|-- cmake/             <- CMake build system (alternative to ./configure)
|-- compile            <- top-level build script (wraps make)
|-- configure          <- top-level configure script (wraps arch/Config.pl)
|-- doc/               <- README files (README.namelist, README.io_config, ...)
|-- dyn_em/            <- ARW Eulerian Mass core (~40 files: solve_em.F, RK3, advection)
|-- external/          <- I/O bindings: netCDF, pnetCDF, HDF5, GRIB, ESMF, RSL_LITE, ...
|-- frame/             <- driver layer: domain manager, nesting, decomposition (~64 files)
|-- hydro/             <- WRF-Hydro coupling library
|-- inc/               <- generated header files (filled by Registry at build time)
|-- main/              <- top-level mains: wrf.F, real_em.F, ideal_em.F, ndown_em.F, tc_em.F
|-- phys/              <- physics suite (~235 files: module_mp_*, module_cu_*, module_bl_*, ...)
|-- Registry/          <- declarative registry (~40 files) that drives code generation
|-- run/               <- run directory: namelist.input, parameter tables, lookup tables
|-- share/             <- I/O helpers, output streams, mediation (~64 files)
|-- test/              <- per-case test directories (em_real, em_b_wave, em_les, ...)
|-- tools/             <- registry compiler, code-gen utilities
|-- var/               <- WRFDA (variational data assimilation)
|-- wrftladj/          <- WRFPLUS (tangent linear / adjoint)
|-- LICENSE.txt        <- WRF public-domain notice
|-- README             <- top-level README (release notes per version)
+-- README.md          <- short pointer to users' site
```

For most users:
- Edit `run/namelist.input` to configure a run.
- Edit physics in `phys/module_*.F`.
- Edit dynamics in `dyn_em/`.
- Add or modify state variables and I/O streams in `Registry/`.

---

## Critical Rules

1. **WRF is built in two stages: `./configure` then `./compile`.** `./configure`
   writes `configure.wrf` (compiler, MPI, optimization, library paths). `./compile
   <case>` recursively builds the chosen case. You must rerun `./configure` if
   you change compilers, MPI, or library paths. You must `./clean -a` before
   switching cases or after editing the Registry.

2. **The Registry is code generation, not runtime config.** Files under
   `Registry/` define state variables, dimensions, I/O streams, and namelist
   options. The `tools/registry` program reads them at build time and emits
   Fortran headers into `inc/`. After editing any Registry file you MUST
   `./clean -a` and recompile from scratch. Without a clean, the headers and
   the user code disagree and you get cryptic linker errors or runtime garbage.

3. **WPS is a separate repository.** The WRF source tree does not include
   WPS (WRF Pre-processing System: geogrid, ungrib, metgrid). Clone
   `wrf-model/WPS` separately, and build it after WRF (it links against
   WRF's I/O libraries built in `external/io_*`).

4. **`real.exe` and `wrf.exe` must use the same vertical coordinate.**
   Setting `hybrid_opt=2` in `&dynamics` for `real.exe` and `hybrid_opt=0`
   for `wrf.exe` (or the reverse) crashes immediately. Use the same
   `namelist.input` for both.

5. **Library order matters.** WRF needs (Fortran-callable) NetCDF, plus
   typically HDF5 and zlib for compressed NetCDF-4, plus Jasper, libpng,
   and zlib for GRIB2 in WPS. Build NetCDF C first, then HDF5, then
   NetCDF-Fortran against the same compiler used for WRF. Mixed compilers
   are the single most common install failure.

6. **NMM is frozen; ARW is the active core.** Treat `dyn_nmm/` as
   historical. New physics, new test cases, and active development land in
   `dyn_em/`. The `phys/` suite serves both, but most physics options
   only document or test the ARW combinations.

7. **Idealized cases use `ideal.exe`, real-data cases use `real.exe`.**
   `compile em_b_wave` produces `ideal.exe` and `wrf.exe` in `main/`. `compile
   em_real` produces `real.exe` and `wrf.exe`. Each `em_*` case has its own
   initialization module under `dyn_em/module_initialize_*.F`.

8. **Run experiments outside the source tree.** `wrf.exe` writes
   `wrfout_d01_*`, `wrfrst_d01_*`, and `rsl.out.*` / `rsl.error.*` to the
   current directory. Always run from a scratch directory with the linked
   namelist, parameter tables, and executables, never from the source
   `run/` directory itself.

---

## Quick Start (real-data ARW, single domain, serial gfortran)

```bash
# 1. Get the code
git clone https://github.com/wrf-model/WRF.git
cd WRF

# 2. Set library environment (adjust to your install)
export NETCDF=/usr/local/netcdf
export HDF5=/usr/local/hdf5
export PATH=$NETCDF/bin:$PATH

# 3. Configure (interactive). For local laptop pick:
#      gfortran serial    -> option 32 on most platforms
#    For HPC pick:
#      INTEL dmpar        (distributed-memory MPI)
./configure
# inspect configure.wrf, especially DM_FC, NETCDFPATH, HDF5_PATH

# 4. Compile a real-data case (15-45 min single-threaded)
./compile em_real >& compile.log
grep -i "Error" compile.log    # should be empty
ls main/wrf.exe main/real.exe  # both must exist

# 5. Get WPS, build it, generate met_em files
git clone https://github.com/wrf-model/WPS.git
cd WPS
./configure   # pick GRIB2 (Jasper) option matching your WRF compiler
./compile >& compile.log
# (set namelist.wps, link Vtable, run geogrid.exe / ungrib.exe / metgrid.exe)

# 6. Run real.exe then wrf.exe
cd ../WRF/test/em_real
ln -sf ../../WPS/met_em.d01.* .
./real.exe              # produces wrfinput_d01 + wrfbdy_d01
./wrf.exe               # produces wrfout_d01_<date>

# 7. Inspect output
ncdump -h wrfout_d01_2024-01-01_00:00:00 | less
ncview wrfout_d01_2024-01-01_00:00:00
```

Full walkthrough is in `reference/getting-started.md` and
`reference/running-real-case.md`.

---

## Reference Documents

| Document | What is inside |
|----------|----------------|
| `reference/getting-started.md` | Clone, prereqs (NetCDF, HDF5, MPI, Jasper, zlib, libpng), `./configure` walkthrough, `./compile em_real` and `em_b_wave`, what success looks like |
| `reference/architecture.md` | Source-tree map, the Registry-driven build system, `arch/` configure infrastructure, `frame/` driver layer, `share/` mediation, `external/` I/O, `tools/registry` codegen |
| `reference/dynamic-cores-and-em.md` | ARW vs NMM, time-split RK3 integration, Arakawa C-grid, hybrid sigma-pressure vertical coordinate, `&dynamics` namelist options |
| `reference/physics-options.md` | Microphysics (WSM6, Thompson, Morrison, P3, NSSL), cumulus (KF, BMJ, GF, Tiedtke), PBL (YSU, MYJ, MYNN, QNSE), radiation (RRTMG, Goddard), surface layer, land surface (Noah, Noah-MP, CLM, RUC, Pleim-Xu) |
| `reference/wrf-chem.md` | `chem/` layout, gas-phase mechanisms (RADM2, CBMZ, MOZART, SAPRC), aerosol options (GOCART, MADE/SORGAM, MOSAIC, MAM), emissions (anthropogenic, biogenic MEGAN/BEIS, dust, sea salt, fire) |
| `reference/running-real-case.md` | Full WPS workflow: `geogrid.exe` -> `ungrib.exe` -> `metgrid.exe` -> `real.exe` -> `wrf.exe`. `namelist.wps` and `namelist.input` pairing. Restart, two-way nesting, ndown, `tc.exe` |
| `reference/running-idealized.md` | `em_b_wave`, `em_quarter_ss`, `em_les`, `em_grav2d_x`, `em_squall2d_*`, `em_heldsuarez`, `em_tropical_cyclone`, `em_scm_xy`, `em_seabreeze2d_x`. `ideal.exe` vs `real.exe` and per-case initialization modules |
| `reference/wrf-hydro-coupling.md` | `hydro/` subdirectory, `WRF_HYDRO=1` build switch, `hydro.namelist`, channel and overland routing, NWM (National Water Model) coupling |
| `reference/debugging.md` | `./configure` failures, link errors, physics segfaults, timestep CFL violations, nesting issues, restart bit-for-bit reproducibility, `rsl.error.*` triage |
| `reference/contributing-pr.md` | wrf-model/WRF fork model, JIRA workflow, branch naming, regression tests, the WRF code review process |
