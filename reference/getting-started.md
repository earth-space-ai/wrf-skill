# Getting Started with WRF

This guide walks from zero to a built `wrf.exe` plus `real.exe` (real-data
ARW). After this, see `running-real-case.md` for the WPS workflow that
generates the boundary conditions, or `running-idealized.md` for the bundled
test cases that need no external input.

---

## Step 1: Decide what you are building

Before you compile anything, decide:

| Question | Choices |
|----------|---------|
| Which dynamical core? | **ARW** (Advanced Research WRF, `dyn_em/`, default). NMM is frozen. |
| Real-data or idealized? | `compile em_real` for real-data, `compile em_<case>` for idealized. |
| Serial, OpenMP (smpar), MPI (dmpar), or hybrid (dm+sm)? | Production runs are almost always **dmpar**. Use serial only for laptops, debugging, or small idealized tests. |
| Standard WRF or one of the extensions? | WRF-Chem (`WRF_CHEM=1`), WRF-Hydro (`WRF_HYDRO=1`), WRFDA (under `var/`), WRFPLUS (under `wrftladj/`). These are environment-variable switches set before `./configure`. |

This skill focuses on the standard ARW build. WRF-Chem and WRF-Hydro have
their own deep-dive docs (`wrf-chem.md`, `wrf-hydro-coupling.md`).

---

## Step 2: Clone the source

```bash
git clone https://github.com/wrf-model/WRF.git
cd WRF
ls
# arch  chem  cmake  CMakeLists.txt  compile  configure  doc  dyn_em
# external  frame  hydro  inc  LICENSE.txt  main  Makefile  phys
# README  README.md  Registry  run  share  test  tools  var  wrftladj
```

The repo is large (the full clone is ~1 GB and includes the sub-projects WRFDA
under `var/` and WRFPLUS under `wrftladj/`). Use a shallow clone if you only
want a single release:

```bash
git clone --depth 1 --branch release-v4.7.1 https://github.com/wrf-model/WRF.git
```

---

## Step 3: Install prerequisites

WRF needs a Fortran 2003-capable compiler, a C compiler, and several C-based
libraries. The required versus optional split:

### Required for any build

| Library | Purpose | Notes |
|---------|---------|-------|
| **NetCDF-C** | I/O backend | 4.x. Build with `--enable-netcdf-4` if you want compressed output. |
| **NetCDF-Fortran** | Fortran bindings | Built against NetCDF-C, **with the same compiler** as WRF. This is the most common install failure. |
| **Fortran compiler** | Compile WRF | gfortran 9+, ifort / ifx, NVHPC (PGI), Cray ftn, IBM xlf. |
| **C compiler** | Compile shims, externals | Match the Fortran compiler family (gcc with gfortran, icc/icx with ifort, ...). |
| **csh / tcsh** | `./configure` runs `arch/Config.pl` via tcsh on some systems | macOS / Ubuntu: `apt install tcsh`. |
| **Perl** | Used by `configure` and the registry | 5.10+. |
| **m4** | Used by the build | Standard package. |

### Strongly recommended

| Library | Purpose | When you need it |
|---------|---------|------------------|
| **MPI** | Distributed-memory parallel build | Anything beyond a laptop. OpenMPI 4.x, MPICH 3.4+, Intel MPI, Cray MPI. |
| **HDF5** | Backend for compressed NetCDF-4 | If your NetCDF-C is built `--enable-netcdf-4`. Build with `--enable-parallel` if you want parallel NetCDF I/O. |
| **zlib** | Compression | Comes with the OS on Linux. |
| **libpng** | GRIB2 in WPS | Strictly a WPS need, but install it now. |
| **Jasper** | jpeg2000 inside GRIB2 | Strictly a WPS need. macOS: `brew install jasper`. Ubuntu newer than 20.04 dropped libjasper-dev; build from source from https://github.com/jasper-software/jasper. |

### Optional

- **pnetCDF** for parallel NetCDF-3 output (`io_form_history=11`).
- **GRIB1 / GRIB2** support inside WRF (rarely used, most reads are netCDF).
- **PIO / ESMF** for advanced coupling.
- **CMake** 3.20+ if you want to use `cmake/` instead of `./configure`.

The canonical install tutorial is here:
https://www2.mmm.ucar.edu/wrf/OnLineTutorial/compilation_tutorial.php

---

## Step 4: Set the environment

WRF's `./configure` reads several environment variables:

```bash
# Library locations (point to your install prefixes)
export NETCDF=/usr/local/netcdf
export HDF5=/usr/local/hdf5
export PNETCDF=/usr/local/pnetcdf      # optional
export PHDF5=/usr/local/hdf5-parallel  # optional, for parallel HDF5

# Compiler-side help (rarely needed if libs are in standard paths)
export NETCDF_classic=1                # build older NetCDF-3 compatible only

# WRF feature switches (all default to 0 / unset)
export WRF_CHEM=0                      # 1 to build WRF-Chem
export WRF_KPP=0                       # 1 if WRF_CHEM=1 and you want KPP gas mechanisms
export WRF_HYDRO=0                     # 1 to build WRF-Hydro coupling
export WRF_DA_CORE=0                   # 1 to build WRFDA (3DVar / 4DVar)

# Large-file netCDF support (default since v3.9)
export WRFIO_NCD_LARGE_FILE_SUPPORT=1
```

These must be set in your shell **before** `./configure`. They get baked
into `configure.wrf`.

---

## Step 5: Configure

```bash
./configure
```

You are prompted twice:

1. **Compiler + parallelism combo.** A typical menu (numbering varies):

   ```
    1.  Linux x86_64, gfortran (serial)
    2.  Linux x86_64, gfortran (smpar)
    3.  Linux x86_64, gfortran (dmpar)
    4.  Linux x86_64, gfortran (dm+sm)
    ...
   33.  Linux x86_64, INTEL (ifort/icc) (dmpar)
    ...
   ```

   Pick the line matching your compiler and intended parallelism. For
   production: `dmpar`. For laptop development: `serial` or `smpar`.

2. **Nesting type.** Almost always **basic** (1). Use `vortex-following` (3)
   only for moving-nest hurricane runs. `pre-set moves` (2) is rarely used.

The script writes `configure.wrf` in the source root. Inspect it:

```bash
grep -E "DM_FC|SFC|NETCDFPATH|HDF5_PATH|FCOPTIM" configure.wrf
```

`DM_FC` should be your MPI Fortran wrapper (`mpif90`, `mpiifort`, `ftn`).
`NETCDFPATH` should point to your NetCDF install. If anything is wrong, edit
`configure.wrf` directly or fix the environment and rerun `./configure`.

---

## Step 6: Compile

```bash
./compile em_real >& compile.log
```

Compile time ranges from 15 minutes (modern desktop, parallel make) to
60+ minutes (laptop, serial). Use `J="-j 4"` to parallelize (default since
v3.2 is `-j 2`):

```bash
export J="-j 4"
./compile em_real >& compile.log
```

If you have a heavily loaded system or a flaky compile, force serial:

```bash
export J="-j 1"
```

Verify success:

```bash
grep -E "Error|error" compile.log | grep -v "ignore" | head
ls -la main/wrf.exe main/real.exe
ls -la test/em_real/wrf.exe test/em_real/real.exe   # symlinks into main/
```

You should have `wrf.exe` plus `real.exe` (em_real) or `ideal.exe` (idealized
cases) in `main/`, with symlinks in `test/<case>/` and `run/`.

---

## Step 7: Smoke test on a packaged idealized case

The fastest way to confirm the executable actually runs:

```bash
./compile em_b_wave >& compile_bwave.log
cd test/em_b_wave
./ideal.exe              # writes wrfinput_d01
./wrf.exe                # serial run; ~1 minute on a modern laptop
ls -la wrfout_d01_*
```

If `wrf.exe` produces a `wrfout_d01_*` file, your build is functional. Move
on to `running-real-case.md` for a real-data run with WPS or
`running-idealized.md` for the rest of the idealized cases.

---

## Step 8: Switch cases or recompile after edits

WRF does not auto-clean between cases. To switch from `em_real` to
`em_b_wave`, or after editing the Registry, or after changing
`configure.wrf`:

```bash
./clean -a              # full clean: removes all object files AND configure.wrf
./configure             # regenerate configure.wrf
./compile em_b_wave >& compile.log
```

Without `./clean -a`:
- Registry edits leave stale auto-generated headers in `inc/` and you get
  cryptic linker errors.
- Compiler/flag changes leave incompatible `.o` files and you get hard-to-diagnose
  segfaults.
- Switching from `em_real` to `em_b_wave` (or vice versa) without cleaning
  produces an executable that is partly the wrong case.

A plain `./clean` (no `-a`) keeps `configure.wrf` and removes only object
files. Use that after a small physics edit when nothing in the Registry or
build options has changed.

---

## Common install pitfalls

| Symptom | Cause | Fix |
|---------|-------|-----|
| `configure` fails with `tcsh: command not found` | tcsh not installed | `apt install tcsh` (Ubuntu) / `brew install tcsh` (macOS) |
| `configure` says `NETCDF environment variable not set` | `$NETCDF` unset or pointing to a path without `lib/libnetcdff.*` | Set `NETCDF` to the prefix that contains `lib/libnetcdf.so` AND `lib/libnetcdff.so` (Fortran bindings, separate package) |
| Link error `undefined reference to nf90_*` | NetCDF-Fortran built with a different compiler than WRF | Rebuild NetCDF-Fortran with the same compiler family used in `configure.wrf` |
| Link error involving HDF5 | NetCDF-4 was enabled but `$HDF5` is unset or pointing nowhere | `export HDF5=/path/to/hdf5` and `./configure` again |
| Compile dies in `dyn_em/module_first_rk_step_part1.F` with hundreds of warnings | Out-of-memory during compile | Reduce parallelism: `export J="-j 2"` |
| Compile finishes but `wrf.exe` missing | Silent error earlier | `grep -i error compile.log` and fix the first error, not the last |
| `wrf.exe` segfaults immediately at runtime | Library mismatch between configure and run environment | `ldd main/wrf.exe`; ensure the LD_LIBRARY_PATH at run time matches the compile-time paths |

For an FAQ maintained by the WRF community, see
https://forum.mmm.ucar.edu/ and https://github.com/wrf-model/WRF/issues.

---

## Where to next

- Real-data forecast end-to-end: `running-real-case.md`
- Idealized supercell, baroclinic wave, LES: `running-idealized.md`
- Source-tree tour and the Registry: `architecture.md`
- A run is failing: `debugging.md`
