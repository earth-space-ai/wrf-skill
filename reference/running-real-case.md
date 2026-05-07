# Running a Real-Data Case End-to-End

This is the production workflow for any real-world simulation: regional
forecast, hindcast, regional climate downscaling, hurricane case study,
operational nowcast. The pipeline is:

```
external GRIB data (GFS, ERA5, NAM, ...)
       |
       v
   [WPS]
   geogrid.exe  -- gets terrain, land use from static datasets
   ungrib.exe   -- decodes GRIB to "intermediate" format
   metgrid.exe  -- horizontally interpolates to your grid
       |
       v  (writes met_em.d<dom>.<date>.nc)
       |
   [WRF]
   real.exe     -- vertical interpolation; writes wrfinput, wrfbdy
   wrf.exe      -- the actual integration; writes wrfout
       |
       v
   post-processing (ncview, NCL, UPP, RIP, wrf-python, MetPy, ...)
```

WPS is a separate repository (`wrf-model/WPS`). Build it after WRF.

---

## Step 1: Get and build WPS

```bash
git clone https://github.com/wrf-model/WPS.git
cd WPS
./configure
# pick the same compiler family as your WRF build, with GRIB2 support
# e.g. on Linux x86_64:  3 = serial gfortran with GRIB2
./compile >& compile.log
ls geogrid/geogrid.exe
ls ungrib/ungrib.exe
ls metgrid/metgrid.exe
```

WPS links against WRF's I/O library (`external/io_grib_share/`), so build
WRF first.

You also need to download the **WPS geographic data** (terrain, land use,
soil category, climatology). The full high-resolution package is large
(~50 GB); start with the lower-resolution datasets and expand as needed:

https://www2.mmm.ucar.edu/wrf/users/download/get_sources_wps_geog.html

Edit `WPS/geogrid/GEOGRID.TBL` only if you are adding new static data.

---

## Step 2: Get GRIB input

For a global forecast you typically use NCEP GFS analyses or forecasts:

```bash
# NCEP GFS 0.25 deg, every 3 hours
mkdir -p ~/wrf_grib_input
cd ~/wrf_grib_input
for hh in 000 003 006 009 012; do
  wget https://nomads.ncep.noaa.gov/pub/data/nccf/com/gfs/prod/gfs.YYYYMMDD/00/atmos/gfs.t00z.pgrb2.0p25.f${hh}
done
```

ERA5 (ECMWF reanalysis) is another common choice; download as GRIB1 or
GRIB2 from CDS. Use the matching `Vtable` (`Vtable.GFS`, `Vtable.ERA5`,
etc) under `WPS/ungrib/Variable_Tables/`.

---

## Step 3: Configure `namelist.wps`

Edit `WPS/namelist.wps`:

```
&share
 wrf_core              = 'ARW',
 max_dom               = 1,
 start_date            = '2024-08-26_00:00:00',
 end_date              = '2024-08-27_00:00:00',
 interval_seconds      = 10800,            ! 3-hour BC interval
 io_form_geogrid       = 2,
/
&geogrid
 parent_id             = 1,
 parent_grid_ratio     = 1,
 i_parent_start        = 1,
 j_parent_start        = 1,
 e_we                  = 100,
 e_sn                  = 100,
 geog_data_res         = 'default',
 dx                    = 12000,
 dy                    = 12000,
 map_proj              = 'lambert',
 ref_lat               = 38.0,
 ref_lon               = -95.0,
 truelat1              = 30.0,
 truelat2              = 60.0,
 stand_lon             = -95.0,
 geog_data_path        = '/path/to/WPS_GEOG',
/
&ungrib
 out_format            = 'WPS',
 prefix                = 'FILE',
/
&metgrid
 fg_name               = 'FILE',
 io_form_metgrid       = 2,
/
```

The `start_date` and `end_date` here MUST match the integration range of
the WRF run (or be longer to allow restart and forecast extensions).

---

## Step 4: Run the WPS chain

```bash
cd WPS
ln -sf ungrib/Variable_Tables/Vtable.GFS Vtable
./link_grib.csh ~/wrf_grib_input/gfs.t00z.pgrb2.0p25.f*

./geogrid.exe                            # produces geo_em.d01.nc
./ungrib.exe                             # produces FILE:YYYY-MM-DD_HH
./metgrid.exe                            # produces met_em.d01.YYYY-MM-DD_HH:00:00.nc
```

Each step writes a log to the current directory. Inspect with
`tail geogrid.log`. On success metgrid produces one `met_em.d01.*` file
per BC time.

---

## Step 5: Configure `namelist.input`

The WRF `namelist.input` is much larger than namelist.wps. The
canonical reference is `run/README.namelist` in the WRF source tree
(documents every variable). A working real-data starter:

```
&time_control
 run_days                = 0,
 run_hours               = 24,
 run_minutes             = 0,
 run_seconds             = 0,
 start_year              = 2024,
 start_month             = 8,
 start_day               = 26,
 start_hour              = 0,
 end_year                = 2024,
 end_month               = 8,
 end_day                 = 27,
 end_hour                = 0,
 interval_seconds        = 10800,
 input_from_file         = .true.,
 history_interval        = 60,            ! min
 frames_per_outfile      = 24,
 restart                 = .false.,
 restart_interval        = 720,           ! min (every 12 h)
 io_form_history         = 2,
 io_form_restart         = 2,
 io_form_input           = 2,
 io_form_boundary        = 2,
 debug_level             = 0,
/

&domains
 time_step               = 60,            ! ~6 * dx_km, here dx=12 km -> 72 s safe; 60 fine
 max_dom                 = 1,
 e_we                    = 100,
 e_sn                    = 100,
 e_vert                  = 35,
 dx                      = 12000,
 dy                      = 12000,
 grid_id                 = 1,
 parent_id               = 0,
 i_parent_start          = 1,
 j_parent_start          = 1,
 parent_grid_ratio       = 1,
 parent_time_step_ratio  = 1,
 num_metgrid_levels      = 34,            ! must match metgrid output
 num_metgrid_soil_levels = 4,
 p_top_requested         = 5000,
/

&physics
 mp_physics              = 8,             ! Thompson
 cu_physics              = 1,             ! Kain-Fritsch
 ra_lw_physics           = 4,             ! RRTMG
 ra_sw_physics           = 4,
 bl_pbl_physics          = 1,             ! YSU
 sf_sfclay_physics       = 1,             ! MM5
 sf_surface_physics      = 2,             ! Noah
 sf_urban_physics        = 0,
 radt                    = 12,
 cudt                    = 0,
 num_soil_layers         = 4,
 num_land_cat            = 21,            ! MODIS-modified IGBP
/

&dynamics
 rk_ord                  = 3,
 time_step_sound         = 4,
 h_mom_adv_order         = 5,
 v_mom_adv_order         = 3,
 h_sca_adv_order         = 5,
 v_sca_adv_order         = 3,
 moist_adv_opt           = 1,             ! positive-definite (mandatory)
 scalar_adv_opt          = 1,
 diff_opt                = 1,
 km_opt                  = 4,
 damp_opt                = 3,
 zdamp                   = 5000,
 dampcoef                = 0.2,
 hybrid_opt              = 2,             ! HVC default
 etac                    = 0.2,
/

&bdy_control
 spec_bdy_width          = 5,
 spec_zone               = 1,
 relax_zone              = 4,
 specified               = .true.,
/

&namelist_quilt
 nio_tasks_per_group     = 0,
 nio_groups              = 1,
/
```

---

## Step 6: Run real.exe and wrf.exe

```bash
cd WRF/test/em_real

# Link WPS output into the run dir (or copy)
ln -sf /path/to/WPS/met_em.d01.* .

# Link namelist (or copy)
cp /path/to/your/namelist.input .

# Real-data initialization
./real.exe                                # serial; or:
mpirun -np 4 ./real.exe                   # parallel
ls wrfinput_d01 wrfbdy_d01

# Inspect rsl.out.0000 / rsl.error.0000 for the SUCCESS message:
tail rsl.out.0000
# wrf: SUCCESS COMPLETE REAL_EM INIT

# The model
mpirun -np 16 ./wrf.exe                   # production
ls wrfout_d01_*
```

`real.exe` reads `met_em.d01.*`, vertically interpolates to model levels,
applies the surface match, and writes:
- `wrfinput_d01` (initial state)
- `wrfbdy_d01` (lateral boundary conditions, contains all of the integration range plus tendencies)

`wrf.exe` then reads these and integrates. Output goes to `wrfout_d01_*`
files at `history_interval` cadence.

In MPI runs each rank writes its own `rsl.out.*` and `rsl.error.*`. Check
`rsl.error.0000` for the first error. With `&namelist_quilt` and
`nio_tasks_per_group > 0`, asynchronous I/O quilt servers handle output;
this scales much better above 256 ranks.

---

## Step 7: Restart

```
&time_control
 restart           = .true.,
 restart_interval  = 720,
/
```

After a previous run with `restart_interval=720` (12 h), you have files
named `wrfrst_d01_2024-08-26_12:00:00`. To resume:

1. Set `restart = .true.`
2. Set `start_year/month/day/hour` to the restart time (here 12 UTC).
3. Re-run `wrf.exe` with the same namelist (do NOT rerun real.exe).

WRF reads `wrfrst_d01_<datetime>` automatically. Restart preserves
**bit-for-bit reproducibility** within a configuration if `nproc_x` and
`nproc_y` are unchanged.

---

## Step 8: Two-way nesting

For a parent domain plus one child:

```
&domains
 max_dom                 = 2,
 e_we                    = 100,    150,
 e_sn                    = 100,    150,
 dx                      = 12000,  4000,
 dy                      = 12000,  4000,
 grid_id                 = 1,      2,
 parent_id               = 0,      1,
 i_parent_start          = 1,      30,
 j_parent_start          = 1,      30,
 parent_grid_ratio       = 1,      3,
 parent_time_step_ratio  = 1,      3,
 feedback                = 1,
 smooth_option           = 2,
/
```

Then in `&time_control` and `&physics` every `(max_dom)` variable has
two values. Run `real.exe` and `wrf.exe` exactly as before. A two-way
nest writes `wrfout_d01_*` and `wrfout_d02_*` and feeds the child's
solution back into the parent every parent timestep.

`parent_grid_ratio` must be **odd** for real-data cases (3, 5, 7).
Idealized cases allow even ratios with `feedback=0`.

For one-way (offline) nesting, run the parent to completion, then
`ndown.exe` reads parent `wrfout` and writes `wrfinput_d02` and
`wrfbdy_d02` for the child.

---

## Step 9: Common production tweaks

- **Adaptive timestep**: `&domains use_adaptive_time_step = .true., step_to_output_time = .true., target_cfl = 1.2`.
  Lets WRF shrink dt automatically when CFL grows. Particularly useful for
  hurricanes and convection-allowing runs.
- **Auxiliary history streams**: define an extra output stream with
  `auxhist3_outname = 'wrfdiag_d<domain>_<date>'`,
  `auxhist3_interval = 60`, `io_form_auxhist3 = 2`, `frames_per_auxhist3 = 24`.
  Combined with `iofields_filename` that lists which variables go to which
  stream, you can write specialty subsets at different cadences.
- **Run-time output customization**: see `doc/README.io_config`. A line
  like `+:h:3:U,V,W` adds U, V, W to history stream 3 without
  recompiling.
- **Asynchronous I/O quilting**: `&namelist_quilt nio_tasks_per_group=4, nio_groups=1`
  dedicates 4 ranks per quilt group to writing output. Essential at
  >256 ranks.
- **Time-series output at point locations**: `tslist` file lists
  lat/lon/labels for which to write per-step time series in
  `<label>.d01.{TS,UU,VV,...}` files. Doc: `run/README.tslist`.

---

## Step 10: Post-processing

| Tool | Use |
|------|-----|
| `ncview` | Quick variable browse |
| `ncdump -h wrfout_d01_*` | Inspect metadata, dimensions |
| **wrf-python** | Python interface: extract slabs, compute derived diagnostics, regrid for plotting |
| **NCL** (legacy) | NCAR Command Language; many production scripts still use it |
| **UPP** (Unified Post Processor) | Produces operational GRIB2 fields from `wrfout` |
| **RIP4** | Older NCAR plotting; superseded by wrf-python |
| **MetPy** | Pure Python alternative for derived fields |

For a 24 h forecast at 12 km on continental US with the namelist above,
expect roughly 2-4 GB of `wrfout` output and similar restart files.

---

## Where to next

- The dynamical core that drives this: `dynamic-cores-and-em.md`
- The physics namelist tuning: `physics-options.md`
- Add chemistry: `wrf-chem.md`
- A run is failing: `debugging.md`
- Building before running: `getting-started.md`
