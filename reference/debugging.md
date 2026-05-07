# Debugging WRF

A field guide to the most common WRF failures, what they look like, and
how to triage. Covers configure failures, compile errors, runtime
segfaults, CFL violations, nesting issues, restart reproducibility, and
output sanity checks.

---

## 1. `./configure` failures

### `tcsh: command not found` or `Perl not found`

`./configure` is a tcsh wrapper. Install tcsh: `apt install tcsh` (Ubuntu),
`brew install tcsh` (macOS). Perl 5.10+ is required.

### `Will use NETCDF in dir: /usr/local/netcdf` but no menu appears

`./configure` failed silently before showing the menu. Check by running:

```bash
./configure 2>&1 | tee config_attempt.log
```

Common causes:
- `NETCDF` env var unset.
- `NETCDF` set to a directory that does not contain `lib/libnetcdf*` AND
  `lib/libnetcdff*`. WRF needs both the C and the Fortran NetCDF.
- The `arch/configure.defaults` file does not contain a stanza for your
  OS + compiler combo. On exotic systems you can hand-edit
  `configure.wrf` after the fact.

### Picked the wrong number from the menu

Just rerun `./configure`. It overwrites `configure.wrf`. No need to
clean.

---

## 2. Compile failures

### Link error: `undefined reference to nf90_*`

NetCDF-Fortran missing or built with a different compiler than WRF.

```bash
nf-config --flibs        # should include -lnetcdff -lnetcdf
nf-config --fc           # should match SFC / DM_FC in configure.wrf
```

If the compiler differs, rebuild NetCDF-Fortran with the same compiler
family used by WRF, then `./clean -a && ./configure && ./compile em_real`.

### Link error: `undefined reference to H5*`

NetCDF-4 enabled but HDF5 not findable. `export HDF5=/path/to/hdf5` and
`./clean -a && ./configure`. If your WRF is intentionally NetCDF-3 only,
rebuild NetCDF-C with `--disable-netcdf-4`.

### Link error: `undefined reference to jpc_*` (jpeg2000 / Jasper)

Comes up in WPS, not WRF. Install Jasper from
https://github.com/jasper-software/jasper and set `JASPER=/path` plus
`JASPERLIB=$JASPER/lib JASPERINC=$JASPER/include`, then reconfigure WPS.

### `Error: Out of memory` mid-compile

`module_first_rk_step_part1.F` and several physics modules are huge. Reduce
parallelism: `export J="-j 2"` (or `-j 1`) and recompile. Each
`gfortran` instance can need 2-4 GB for the larger files.

### Compile finishes but `wrf.exe` missing

Search the log for the FIRST error, not the last:

```bash
grep -i "Error\|fatal\|undefined" compile.log | head -20
```

The build has a long "ignore the following error" preamble in `tools/`
that you should skip; the real error is later.

### Cryptic Fortran type-mismatch errors after a Registry edit

You did not `./clean -a`. The auto-generated headers in `inc/` are stale
and the hand-written code does not agree with the generated `domain`
type. Always:

```bash
./clean -a
./configure
./compile em_real >& compile.log
```

after editing anything under `Registry/`.

### `mpif90: command not found`

You picked a `dmpar` configure option but MPI is not in `PATH`. Either
load the MPI module (`module load openmpi`) or pick a serial
configuration.

---

## 3. real.exe failures

### `real.exe` aborts: `mismatch in num_metgrid_levels`

Your `namelist.input` `num_metgrid_levels` does not match the actual
number of levels in `met_em.d01.*`. Inspect:

```bash
ncdump -h met_em.d01.2024-08-26_00:00:00.nc | grep num_metgrid_levels
```

Set `num_metgrid_levels` in `&domains` to that value.

### `real.exe` aborts: `Could not find trimmed namelist`

The namelist file has a syntax error: missing comma, missing `/`
terminator, stray character. Fortran namelist parsers give terrible
error messages. Strategy: temporarily replace `&time_control` with a
known-good copy, then re-add sections one at a time until you find the
broken one.

### `real.exe` aborts: `module_initialize_real: vert_init: ZNW field is wrong size`

Vertical-coordinate mismatch between `wrfinput` and the expected number
of levels. Check `e_vert` matches `e_vert` used to build the previous
input.

### `real.exe` succeeds but produces all-zero `wrfinput`

Almost always a missing variable in `met_em.*`. Check `metgrid.log` for
warnings about fields the Vtable expected but did not find. The classic
cause is using a Vtable for one model (GFS) with GRIB from another
(ERA5). Match the Vtable to the source.

---

## 4. wrf.exe runtime crashes

WRF runtime errors land in `rsl.error.0000` (or `rsl.out.0000` for
informational), one file per MPI rank. Always start by inspecting
`rsl.error.0000`:

```bash
tail -100 rsl.error.0000
```

### CFL violation

Message:

```
cfl,w_cfl,h_cfl(1,2,3),i,k,j,vert_cfl,horiz_cfl ...
d01 ... cfl violation at point ...
```

The model timestep is too large for the local wind speed. The general
guideline is `time_step <= 6 * dx_km`. Common fixes:

| Severity | Fix |
|----------|-----|
| Occasional, isolated | Enable adaptive time stepping: `&domains use_adaptive_time_step=.true., target_cfl=1.2, target_hcfl=0.84`. WRF will shrink dt automatically when CFL grows. |
| Persistent | Reduce `time_step` (maybe halve it). |
| Persistent over a specific topographic feature | Increase `epssm` to 0.3-0.5 (more vertical implicit damping). |
| Persistent in convective parameterization | Try a different `cu_physics`; KF and BMJ can sometimes produce CFL near very deep updrafts. |
| Persistent in moist physics | Set `moist_adv_opt=2` (monotonic) instead of 1 (PD). |
| Run on a moving nest | Set `vortex_interval` and `max_vortex_speed` reasonably. |

### Segfault inside `module_mp_*` (microphysics)

Usually a NaN or negative value reaches the microphysics. Common causes:
- `moist_adv_opt = 0` (no positive-definite limiter): negative q_v from
  advection blows up the saturation calculation. Switch to `moist_adv_opt=1`.
- Numerically aggressive scheme (Morrison 2-moment, P3) hit by an extreme
  initial condition. Try a more forgiving scheme as a sanity check
  (WSM6, Thompson).
- Bad input: `wrfinput` with unrealistic upper-air q_v or T. Inspect
  `wrfinput_d01` with `ncview`.

Compile WRF with `-fcheck=all -g -fbacktrace -ffpe-trap=invalid,zero`
(gfortran) or `-check all -traceback -fpe0` (ifort) and re-run to get a
backtrace. Add to `configure.wrf`'s `FCDEBUG` line. Strip these flags
for production: bound-checking is 3-5x slower.

### Segfault inside `module_radiation_driver` or `module_ra_rrtmg_*`

Usually a missing or empty data file. Confirm:

```bash
ls -la RRTMG_LW_DATA RRTMG_SW_DATA
```

These files MUST be present in the run directory (or symlinked from
`run/`). Empty / 0-byte means a broken download.

### Segfault inside `module_sf_noahmplsm`

For Noah-MP, missing `MPTABLE.TBL` or namelist option that points to a
parameterization not implemented in the linked Noah-MP. Confirm with:

```bash
ls MPTABLE.TBL
```

For deep Noah-MP debugging, see the companion noahmp-skill.

### MPI process death without a clean error

```bash
grep -i "ERROR\|FATAL\|aborted" rsl.error.*
```

Often only one rank has the real error message. `for i in rsl.error.000*; do echo "=== $i ==="; tail $i; done`
shows them quickly.

### `wrf: SUCCESS COMPLETE WRF` but no `wrfout`

The output went to the wrong directory, or `frames_per_outfile` is so
large that the first frame is still buffered. Check `history_interval`
and `frames_per_outfile` consistency.

---

## 5. Nesting failures

### `boundary fields not found for nest`

The child nest expects boundary forcing from the parent, but the parent
domain has not been integrated up to that time. Ensure:
- `parent_id`, `parent_grid_ratio`, `i_parent_start`, `j_parent_start`
  match between `namelist.input` and what `geogrid` produced.
- `start_year/month/day/hour` for the child equals (or is later than)
  the parent's `start_*`.

### Two-way nesting with `parent_grid_ratio = 4` aborts

For real-data, `parent_grid_ratio` MUST be odd (1, 3, 5, 7). For
idealized cases with `feedback=0`, even ratios are allowed.

### `i_parent_start` / `j_parent_start` out of bounds

The child nest extends beyond the parent. Recompute using the parent
grid spacing. The corner (`i_parent_start`, `j_parent_start`) plus
`(e_we_child - 1) / parent_grid_ratio` must be less than `e_we_parent`.

---

## 6. Restart not bit-for-bit

### Symptoms

You restart from `wrfrst_d01_*` and the resulting forecast diverges from
a non-restarted forecast within a few timesteps.

### Causes

- **Different number of MPI ranks.** WRF restart is bit-for-bit only when
  `nproc_x` and `nproc_y` are unchanged. Different decomposition
  produces different round-off accumulation order.
- **Different compile flags.** A restart from a `-O3` run resumed by a
  `-O0` build is not bit-for-bit.
- **Different namelist physics.** Changed `mp_physics` between the
  original run and the restart.
- **Timing alarms.** If `restart_interval` is not a multiple of
  `time_step`, the restart time can be off by a few seconds, causing
  `wrf.exe` to start at a slightly different state. Always set
  `restart_interval` as integer minutes that divide cleanly into the
  timestep.

---

## 7. Output sanity checks

After every run, before publishing or post-processing, check:

```bash
ncdump -h wrfout_d01_<datetime> | grep -i "Time"            # number of frames
ncdump -h wrfout_d01_<datetime> | grep "TITLE\|HYBRID"      # version + HVC option
ncks -m wrfout_d01_<datetime>                               # all variables
ncview wrfout_d01_<datetime>                                # eyeball T2, U10, RAINNC
```

Common silent failures:
- **All-zero precipitation everywhere**: probably `mp_physics=0`
  (none) was set inadvertently. Or the run is too short for accumulated
  rain to register.
- **U10/V10 zero**: surface layer scheme not matched to PBL scheme.
- **2 m temperature is unphysically uniform**: LSM not initialized
  (missing `LANDUSE.TBL`).
- **Pressure at top suspicious**: HVC mismatch between real and wrf.

For systematic regression testing of physics changes use the WRF Test
suite under `test/em_real/` which has reference output for a small
case (Hurricane Katrina default).

---

## 8. Performance issues (not crashes, just slow)

| Symptom | Likely fix |
|---------|-----------|
| Compile is fine but `wrf.exe` runs at 1/10 expected speed | Compile with `-O3` not `-O0`. Check `FCOPTIM` in `configure.wrf`. |
| MPI run hangs at 100% CPU on every rank | Decomposition is bad. WRF prefers `nproc_y > nproc_x` for typical aspect ratios. Set `&domains nproc_x=4, nproc_y=16` instead of letting WRF pick. |
| I/O dominates wall time | Use `nio_tasks_per_group` quilt servers; switch to `io_form_history=11` (pnetCDF) for parallel writes; reduce `frames_per_outfile`. |
| OpenMP build no faster than serial | OpenMP scaling in WRF is limited; use MPI. |

---

## 9. Useful debug levers

| Lever | Effect |
|-------|--------|
| `&time_control debug_level = 50` | mild verbosity |
| `&time_control debug_level = 200` | per-step physics call printout |
| `&time_control debug_level = 1000` | huge log volume; use only in extremis |
| `&time_control diag_print = 1` | per-step domain-averaged surface pressure tendencies |
| `&time_control diag_print = 2` | adds rain, evap, sensible/latent flux |
| `&dynamics check_surface_diff = 1` | sanity-check surface diffusion magnitudes |
| Compile with `-DRSL_LITE_DEBUG` | extra debug from the parallel layer |
| `&namelist_quilt nio_tasks_per_group = 0` | turn off async I/O so I/O errors crash where they happen |

---

## Where to next

- Compiling: `getting-started.md`
- Per-physics scheme details: `physics-options.md`
- Real-data run pipeline: `running-real-case.md`
- Idealized cases (use these for clean-room debugging): `running-idealized.md`
- Submitting a fix back to NCAR: `contributing-pr.md`
- Community forum: https://forum.mmm.ucar.edu/
- Issues tracker: https://github.com/wrf-model/WRF/issues
