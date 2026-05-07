# WRF-Hydro Coupling

WRF-Hydro is a hydrologic modeling extension that adds 1-D channel routing,
2-D overland flow routing, subsurface lateral flow, lake routing, and
optional groundwater bucket models on top of WRF's land surface scheme.
The same source code is used in **NOAA's National Water Model (NWM)**
operational system.

WRF-Hydro can run in two modes:
- **Coupled**: built into `wrf.exe`, exchanges water and energy with the
  atmosphere every land-surface timestep.
- **Offline (uncoupled)**: built as a standalone executable driven by
  external forcing files. This is the NWM operational mode.

This page covers the **coupled** path. For uncoupled details see the
NCAR/wrf_hydro_nwm_public repository.

---

## What is in `hydro/`

The WRF-Hydro source lives in WRF's `hydro/` subdirectory:

```
hydro/
|-- arc/                <- archive of older code
|-- CMakeLists.txt
|-- configure           <- configure script (separate from WRF's)
|-- CPL/                <- coupling: WRF_cpl/ joins to WRF, NoahMP_cpl/, etc
|-- Data_Rec/           <- record types (Fortran derived types) for hydro state
|-- Debug_Utilities/
|-- Doc/
|-- HYDRO_drv/          <- top-level hydro driver (called by WRF or by an offline driver)
|-- IO/                 <- NetCDF I/O for hydro variables
|-- Makefile
|-- MPP/                <- MPI parallelism for hydro grids (separate from WRF's RSL_LITE)
|-- nudging/            <- streamflow nudging (data assimilation)
|-- OrchestratorLayer/
|-- Routing/            <- THE physics: channel routing (Muskingum-Cunge), overland (diffusive wave), subsurface (Boussinesq)
|-- template/           <- example hydro.namelist and channel parameter files
|-- utils/
+-- wrf_hydro_config    <- config file fragment
```

The detailed documentation is in `doc/README.hydro` in the WRF source
tree, and in the WRF-Hydro Technical Description at
https://ral.ucar.edu/projects/wrf_hydro/technical-description-user-guide.

---

## Building WRF coupled with WRF-Hydro

```bash
# Set environment BEFORE configure
export WRF_HYDRO=1                     # 1 = activate WRF-Hydro coupling
export HYDRO_D=1                       # 1 = run-time diagnostic output (set 0 for production)
export NETCDF_INC=$NETCDF/include
export NETCDF_LIB=$NETCDF/lib

# Configure WRF as usual
./configure                            # the menus are unchanged

# Compile (em_real for coupled real-data runs)
./compile em_real >& compile.log
ls main/wrf.exe                        # only wrf.exe is built; no separate hydro executable
```

Two important details:

1. WRF-Hydro is invoked as a **function from inside `wrf.exe`** during the
   land-surface step. There is no separate `hydro.exe` for the coupled
   build. A successful coupled build produces only `wrf.exe`,
   `real.exe`, etc, but the executables now contain the hydro library.

2. `WRF_HYDRO=0` or unset reverts to standard WRF. There is no halfway
   build.

3. WRF-Hydro has its own NetCDF library settings (`NETCDF_INC`,
   `NETCDF_LIB`) that can differ from WRF's. If your site has a
   "combined" NetCDF library (one library, no `libnetcdff`), see the
   porting notes in `doc/README.hydro`.

4. WRF-Hydro does **not** support OpenMP. Use serial or MPI only.

---

## What WRF-Hydro adds: routing physics

The "Routing/" subdirectory contains the physics:

| Physics | Source | What it does |
|---------|--------|--------------|
| Surface overland flow | `Routing/module_overland_routing.F` | 2-D diffusive-wave runoff routing on a refined hydro grid |
| Subsurface lateral flow | `Routing/module_lateral_flow.F` | Boussinesq subsurface flow between adjacent cells |
| Channel routing (Muskingum) | `Routing/module_channel_routing.F` | 1-D channel network with Muskingum-Cunge |
| Channel routing (gridded) | (alternative formulation) | gridded reach-based routing |
| Lake routing | `Routing/module_lake.F` | level-pool reservoir routing |
| Groundwater bucket | `Routing/module_GW_baseflow.F` | conceptual exponential / linear bucket per catchment |

Channel topology is read from the `Route_Link.nc` file (vector
"reaches") for the Muskingum-Cunge routing. The gridded routing reads
`fulldom_hires.nc` plus channel grids.

---

## The hydro grid

WRF-Hydro typically runs at a finer horizontal resolution than the
parent atmosphere model. A 1 km WRF parent often pairs with a 250 m or
100 m hydro grid. The refinement ratio is set in `hydro.namelist`:

```
&NOAHLSM_OFFLINE
 ...
/
&HYDRO_nlist
 sys_cpl                = 2,           ! 2 = WRF-coupled (vs 1 = HRLDAS, 3 = NUOPC, 4 = WRF-Hydro standalone)
 GEO_STATIC_FLNM        = "geo_em.d01.nc",        ! WRF static fields
 GEO_FINEGRID_FLNM      = "geo_em_finegrid.d01.nc", ! refined hydro grid
 LAND_SPATIAL_META_FLNM = "geo_em.d01.nc",
 RESTART_FILE           = "HYDRO_RST.YYYY-MM-DD_HH:00_DOMAIN1",
 IGRID                  = 1,
 rst_dt                 = 720,         ! restart interval (min)
 out_dt                 = 60,          ! output interval (min)
 SUBRTSWCRT             = 1,           ! subsurface routing on
 OVRTSWCRT              = 1,           ! overland routing on
 channel_option         = 2,           ! 1 = Muskingum, 2 = Muskingum-Cunge, 3 = diffusive
 route_topo_f           = "Fulldom_hires.nc",
 route_chan_f           = "Route_Link.nc",
 route_link_f           = "Route_Link.nc",
 route_lake_f           = "LAKEPARM.nc",
 GWBASESWCRT            = 1,           ! groundwater bucket on
 gwbasmskfil            = "GWBUCKPARM.nc",
 nudgingFlag            = 0,
/
```

The parent WRF `geo_em.d01.nc` is preprocessed by the WRF-Hydro
GIS Preprocessor (`whitebox`-based, separate Python tool) to produce the
refined `Fulldom_hires.nc`, the channel network `Route_Link.nc`, and the
groundwater catchment parameters `GWBUCKPARM.nc`.

---

## Running coupled WRF + WRF-Hydro

The execution command is unchanged from standard WRF:

```bash
mpirun -np 64 ./wrf.exe
```

But WRF-Hydro requires:
1. `hydro.namelist` in the run directory.
2. Hydro parameter files (`HYDRO.TBL`, `CHANPARM.TBL`, plus the
   geometric files mentioned above).
3. A hydro initial condition or `RESTART_FILE` from a prior spin-up.

WRF-Hydro produces additional output files alongside the standard WRF
output:
- `HYDRO_RST.<datetime>_DOMAIN1`: hydro restart
- `<datetime>.CHRTOUT_DOMAIN1`: channel reach outputs (streamflow,
  velocity, head)
- `<datetime>.LSMOUT_DOMAIN1`: land surface diagnostics on the LSM grid
- `<datetime>.RTOUT_DOMAIN1`: routing variables on the routing grid
- `<datetime>.LDASOUT_DOMAIN1`: LDAS-style hourly land state
- `<datetime>.GWOUT_DOMAIN1`: groundwater bucket output

The cadence of each is controlled by per-stream interval variables in
`hydro.namelist`.

---

## Streamflow nudging

WRF-Hydro supports gauge-based streamflow nudging (a simple but
effective form of channel-state DA). Enable via:

```
&NUDGING_nlist
 nudgingParamFile         = "nudgingParams.nc",
 nLastObs                 = 960,
 persistBias              = .false.,
 biasWindowBeforeT0       = .true.,
 invDistTimeWeightBias    = .true.,
 ...
/
```

Used operationally in NWM to keep simulated streamflows close to USGS
observations.

---

## Coupling pattern: how WRF calls hydro

Per WRF land-surface timestep, the hydro library:

1. Receives runoff (`SFCRUNOFF`, surface), drainage (`UDRUNOFF`,
   subsurface), and infiltration excess from the LSM in each parent
   grid cell.
2. Disaggregates them onto the refined hydro grid using a
   nearest-neighbor / area-weighted scheme.
3. Routes overland flow (diffusive wave) and subsurface flow
   (Boussinesq) on the hydro grid.
4. Pours the runoff that reaches a channel cell into the channel
   network.
5. Routes the channel network with Muskingum-Cunge.
6. (Optional) Solves the groundwater bucket for each catchment and
   feeds baseflow back to channels.
7. Aggregates moisture exchanges back to the LSM grid for next-step
   feedback (specifically, soil moisture in the bottom layer can be
   modified by the subsurface routing if `SUBRTSWCRT=1`).

The atmosphere does NOT see the hydro state directly. The coupling is
one-way for sensible/latent fluxes (LSM -> atm), but the LSM soil
moisture state is updated by the routing each timestep, which then
indirectly modifies surface fluxes via the standard Noah / Noah-MP path.

---

## Coupling with the National Water Model (NWM)

The NWM uses an offline (uncoupled) WRF-Hydro driver, forced by
post-processed atmospheric output (HRRR or GFS) regridded to a 1 km
CONUS hydro grid. The model is the same `hydro/` source tree compiled
with `sys_cpl=4` (standalone). For an end-to-end NWM-style setup see
https://github.com/NCAR/wrf_hydro_nwm_public.

For coupled WRF + WRF-Hydro retrospective runs (atmosphere driving
hydrology bidirectionally), see https://ral.ucar.edu/projects/wrf_hydro.

---

## Where to next

- The land surface that feeds hydro: `physics-options.md` (Noah, Noah-MP)
- The companion noahmp-skill at https://github.com/ktwu01/noahmp-skill for the LSM internals
- WRF base documentation: `doc/README.hydro` in the WRF source tree
- WRF-Hydro Technical Description: https://ral.ucar.edu/projects/wrf_hydro/technical-description-user-guide
- Build / runtime issues: `debugging.md`
