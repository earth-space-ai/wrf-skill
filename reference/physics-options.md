# WRF Physics Options

WRF's physics suite is large. As of v4.7, `phys/` contains roughly 235
files implementing dozens of schemes across microphysics, cumulus
parameterization, planetary boundary layer, surface layer, land surface,
radiation, sub-grid turbulence, gravity wave drag, urban canopy, ocean
mixed layer, and miscellaneous diagnostics.

This page is a tour of the options, what their namelist switches are, and
which combinations are commonly used. It is not a substitute for the
scheme-specific papers, but it tells you where in the source tree to look
and which `&physics` namelist option drives each choice.

---

## How physics dispatch works

Each physics group has a single **driver** module. The driver reads an
integer namelist option (e.g. `mp_physics`) per nest from
`grid%mp_physics` and dispatches to the chosen scheme via a `select case`.

| Group | Driver file | Namelist option |
|-------|-------------|-----------------|
| Microphysics | `phys/module_microphysics_driver.F` | `mp_physics` |
| Cumulus | `phys/module_cumulus_driver.F` | `cu_physics` |
| Shallow cumulus (separate) | `phys/module_shcu_driver.F` | `shcu_physics` |
| Planetary boundary layer | `phys/module_pbl_driver.F` | `bl_pbl_physics` |
| Surface layer | `phys/module_surface_driver.F` | `sf_sfclay_physics` |
| Land surface | `phys/module_surface_driver.F` (same driver) | `sf_surface_physics` |
| Urban canopy | inside surface driver | `sf_urban_physics` |
| Lake | `phys/module_sf_lake.F` | `sf_lake_physics` |
| Longwave radiation | `phys/module_radiation_driver.F` | `ra_lw_physics` |
| Shortwave radiation | same driver | `ra_sw_physics` |
| Gravity wave drag | `phys/module_gwd.F` (and gsl variants) | `gwd_opt` |
| Sub-grid turbulence (LES) | `phys/module_sf_sfclayrev.F`, `dyn_em/module_diffusion_em.F` | `km_opt`, `sfs_opt` |
| Ocean mixed layer | `phys/module_sf_oml.F` etc | `sf_ocean_physics` |
| Stochastic perturbations | `phys/module_stoch.F` (with `dyn_em/module_stoch.F`) | `stoch_force_opt` |

To add a new scheme: write `phys/module_<group>_<name>.F`, register a
package value in the appropriate Registry file (e.g. `Registry.EM_COMMON`),
add a `case` to the relevant driver. Then `./clean -a && ./compile em_real`.

The full per-namelist physics combinatorics is documented in
`run/README.namelist`. The doc README at `doc/README.physics_files`
inventories which lookup-table files each physics option needs.

---

## Microphysics (`mp_physics`)

Source files all match `phys/module_mp_*.F`. Schemes in WRF 4.7:

| `mp_physics` | Scheme | Source | Notes |
|--------------|--------|--------|-------|
| 0 | none | | dry dynamics only |
| 1 | Kessler (warm rain) | `module_mp_kessler.F` | warm rain only; idealized |
| 2 | Lin et al. | `module_mp_lin.F` | classical 6-class single-moment |
| 3 | WSM3 | `module_mp_wsm3.F` | 3-class single-moment |
| 4 | WSM5 | `module_mp_wsm5.F` | 5-class single-moment |
| 5 | Eta Ferrier | `module_mp_etanew.F` (`ETAMPNEW_DATA`) | NCEP operational legacy |
| 6 | **WSM6** | `module_mp_wsm6.F` | 6-class single-moment, common production choice |
| 7 | Goddard | `module_mp_gsfcgce.F` (`BROADBAND_CLOUD_GODDARD.bin`) | NASA |
| 8 | **Thompson** | `module_mp_thompson.F` | 6-class with predicted Nc, default for many real-data runs |
| 9 | Milbrandt-Yau 2-moment | `module_mp_milbrandt2mom.F` | |
| 10 | **Morrison 2-moment** | `module_mp_morr_two_moment.F` | research / cloud-resolving |
| 11 | CAM 5.1 2-moment | `module_mp_cammgmp_driver.F` | bundled with CESM aerosols |
| 13 | SBU-Lin | `module_mp_sbu_ylin.F` | |
| 14 | WDM5 | `module_mp_wdm5.F` | WRF Double Moment 5-class |
| 16 | WDM6 | `module_mp_wdm6.F` | WRF Double Moment 6-class |
| 17, 18 | NSSL 2-moment | `module_mp_nssl_2mom.F` | with predicted hail |
| 19 | NSSL 1-moment | (same file) | |
| 21 | WSM7 | `module_mp_wsm7.F` | with hail/graupel split |
| 22 | WDM7 | `module_mp_wdm7.F` | |
| 28 | Thompson aerosol-aware | `module_mp_thompson.F` (with `MP_AER` flag) | requires QNIFA, QNWFA inputs |
| 30 | HUJI fast SBM | `module_mp_fast_sbm.F` | spectral bin |
| 32 | HUJI full SBM | `module_mp_full_sbm.F` | spectral bin, very expensive |
| 50 | P3 1-cat | `module_mp_p3.F` (`p3_lookupTable_*.dat-*`) | predicted ice properties |
| 51 | P3 2-cat | (same) | |
| 52, 53 | P3 3-mom | (same) | research |

For real-data convective forecasts at 1-15 km grid: **WSM6 (6)** or
**Thompson (8)** are the most-used choices. For convection-allowing runs
at 1-3 km that resolve clouds: Thompson, Morrison, NSSL 2-moment.

Pair with `mp_zero_out` and `mp_zero_out_thresh` to clean up small
negative moisture from advection.

---

## Cumulus parameterization (`cu_physics`)

For grid spacings coarser than ~10 km, cumulus convection cannot be
resolved and must be parameterized. For grid spacings finer than ~4 km,
cumulus is usually turned off and microphysics handles convection
explicitly. The "grey zone" (4-10 km) is where scale-aware schemes shine.

| `cu_physics` | Scheme | Source | Notes |
|--------------|--------|--------|-------|
| 0 | none | | use for grid spacing < 4 km |
| 1 | **Kain-Fritsch** (with shallow conv) | `module_cu_kf.F`, `module_cu_kfeta.F` | classic, mass-flux |
| 2 | **Betts-Miller-Janjic (BMJ)** | `module_cu_bmj.F` | adjustment scheme |
| 3 | **Grell-Freitas (GF)** ensemble | `module_cu_gf_*.F` | scale-aware |
| 4 | old SAS | `module_cu_sas.F` | |
| 5 | Grell 3D | `module_cu_g3.F` | scale-aware |
| 6 | Tiedtke | `module_cu_tiedtke.F` | with shallow conv and momentum transport |
| 7 | Zhang-McFarlane | `module_cu_camzm.F` | with momentum transport |
| 10 | Multi-scale Kain-Fritsch | `module_cu_mskf.F` | scale-aware KF |
| 11 | Kain-Fritsch CuP | `module_cu_kfcup.F` | cumulus potential |
| 14 | NSAS | `module_cu_nsas.F` | new SAS, with shallow conv |
| 16 | New Tiedtke | `module_cu_ntiedtke.F` | improved Tiedtke |
| 84 | KSAS | `module_cu_ksas.F` | Korean SAS |
| 93 | Grell-Devenyi ensemble | `module_cu_gd.F` | older GF predecessor |
| 99 | old Kain-Fritsch (Eta) | `module_cu_kfeta.F` | not for new work |

Common production choices: **Kain-Fritsch (1)** for North America 12 km,
**BMJ (2)** for tropics, **Grell-Freitas (3)** if you need scale-aware
behavior in the grey zone, **Tiedtke (6)** for global / European runs.

Shallow convection is sometimes a separate option:
`shcu_physics = 2` selects UW shallow, `3` selects GRIMS, `4` selects Deng.

---

## Planetary boundary layer (`bl_pbl_physics`)

| `bl_pbl_physics` | Scheme | Source | Pair with `sf_sfclay_physics` |
|------------------|--------|--------|-------------------------------|
| 0 | none | | for LES with explicit sub-grid turbulence |
| 1 | **YSU** (Yonsei University) | `module_bl_ysu.F` | 1 (MM5) or 91 (MYNN sf revised) |
| 2 | **MYJ** (Mellor-Yamada-Janjic) | `module_bl_myjpbl.F` | 2 (Eta) |
| 3 | GFS | `module_bl_gfs.F` | 3 (GFS) |
| 4 | QNSE-EDMF | `module_bl_qnsepbl.F` | 4 (QNSE) |
| 5 | **MYNN level 2.5** | `module_bl_mynn.F` | 5 (MYNN) |
| 6 | MYNN level 3 | `module_bl_mynn.F` | 5 (MYNN) |
| 7 | ACM2 (Pleim) | `module_bl_acm.F` | 7 (Pleim-Xu) |
| 8 | BouLac | `module_bl_boulac.F` | 1 or 2 (often paired with multilayer urban) |
| 9 | UW (Bretherton) | `module_bl_camuwpbl_driver.F` | 1 or 5 |
| 10 | TEMF | `module_bl_temf.F` | 10 (TEMF) |
| 11 | Shin-Hong scale-aware | `module_bl_shinhong.F` | 1 |
| 12 | GBM (Grenier-Bretherton-McCaa) | `module_bl_gbmpbl.F` | |
| 16 | EEPS-Ling | `module_bl_eepsilon.F` | |
| 99 | MRF (legacy) | `module_bl_mrf.F` | not for new work |

Production picks: **YSU (1)** with MM5 surface layer for general-purpose
North America, **MYNN 2.5 (5)** with MYNN surface layer for boundary-
layer-sensitive runs (offshore wind, fog), **MYJ (2)** with Eta surface
layer for HRW-style operational runs.

Note PBL schemes have implicit assumptions about vertical resolution.
YSU works fine with 30-50 vertical levels and ~50 m lowest level
thickness; MYNN tolerates higher resolution.

---

## Surface layer (`sf_sfclay_physics`)

The surface layer module computes friction velocity, exchange coefficients
for heat and moisture, and the 2 m / 10 m diagnostics from PBL similarity
theory.

| `sf_sfclay_physics` | Scheme | Pair with PBL |
|---------------------|--------|---------------|
| 0 | none | only when `bl_pbl_physics = 0` |
| 1 | MM5 (Monin-Obukhov, original) | YSU |
| 2 | Eta similarity (Janjic) | MYJ |
| 3 | GFS | GFS |
| 4 | QNSE | QNSE |
| 5 | MYNN | MYNN 2.5 / 3 |
| 7 | Pleim-Xu | ACM2 |
| 10 | TEMF | TEMF |
| 11 | MYNN revised | MYNN |
| 91 | old MM5 / Pleim hybrid (revised) | YSU |

Pairings are not arbitrary. The PBL scheme expects specific exchange
coefficients computed by the matching surface layer. Mismatches usually
do not crash the model but produce silently wrong fluxes.

---

## Land surface model (`sf_surface_physics`)

| `sf_surface_physics` | Scheme | Source | Levels |
|----------------------|--------|--------|--------|
| 0 | none | | for ocean-only runs |
| 1 | 5-layer thermal diffusion (slab) | `module_sf_noahdrv.F` (legacy) | 5 |
| 2 | **Noah** | `module_sf_noahdrv.F`, `module_sf_noahlsm.F` | 4 soil layers |
| 3 | RUC | `module_sf_ruclsm.F` | 6-9 soil levels |
| 4 | **Noah-MP** | `module_sf_noahmpdrv.F`, `module_sf_noahmplsm.F` | 4 |
| 5 | CLM4 | `module_sf_clm.F` | 10 |
| 7 | Pleim-Xu | `module_sf_pxlsm.F` | 2 |
| 8 | SSiB | `module_sf_ssib.F` | 3 |

Production picks: **Noah (2)** for portability and coupling with WRF-Hydro;
**Noah-MP (4)** for richer physics (multi-parameterization for canopy
radiation, runoff, snow, etc, see the companion noahmp-skill); **CLM4 (5)**
for long climate runs with PFTs and biogeochemistry; **RUC (3)** for
ice/snow-dominated regimes.

Reads `LANDUSE.TBL` (land use category to surface property), `VEGPARM.TBL`
(vegetation parameters), `SOILPARM.TBL` (soil parameters), `GENPARM.TBL`
(general). Noah-MP additionally reads `MPTABLE.TBL`.

The Noah-MP coupling lives in `phys/module_sf_noahmpdrv.F`. The Noah-MP
core source is now maintained at `NCAR/noahmp` (v5+) and is kept in sync
with WRF via the `Registry/registry.noahmp` table.

### Urban canopy (`sf_urban_physics`)

Layered on top of Noah / Noah-MP for urban land use categories:

| `sf_urban_physics` | Scheme |
|--------------------|--------|
| 0 | bulk scheme (urban treated as a high-roughness vegetation type) |
| 1 | Single-Layer Urban Canopy Model (SLUCM) |
| 2 | BEP (multi-layer urban canopy), requires MYJ or BouLac PBL |
| 3 | BEM (BEP + building energy model), requires MYJ or BouLac PBL |

Reads `URBPARM.TBL`. With v4.5+ the LCZ (Local Climate Zones)
classification is supported via `URBPARM_LCZ.TBL`.

### Lake (`sf_lake_physics`)

`sf_lake_physics = 1` enables the CLM4-derived lake model on freshwater
lakes from MODIS land use.

---

## Radiation

Two parallel options: longwave and shortwave.

### Longwave (`ra_lw_physics`)

| `ra_lw_physics` | Scheme | Source | Lookup file |
|-----------------|--------|--------|-------------|
| 0 | none | | |
| 1 | RRTM | `module_ra_rrtm.F` | `RRTM_DATA` |
| 3 | CAM | `module_ra_cam.F` | `CAM_ABS_DATA`, `ozone.formatted`, `aerosol.formatted` |
| 4 | **RRTMG** | `module_ra_rrtmg_lw.F` | `RRTMG_LW_DATA` |
| 5 | new Goddard | `module_ra_goddard.F` | |
| 7 | FLG | `module_ra_flg.F` | |
| 14 | RRTMG-K | RRTMG with reduced-cost option | |
| 24 | RRTMG (fast) | a faster RRTMG variant | |
| 99 | GFDL (legacy) | | |

### Shortwave (`ra_sw_physics`)

| `ra_sw_physics` | Scheme | Source |
|-----------------|--------|--------|
| 0 | none | |
| 1 | Dudhia | `module_ra_sw.F` |
| 2 | old Goddard | `module_ra_gsfcsw.F` |
| 3 | CAM | `module_ra_cam.F` |
| 4 | **RRTMG** | `module_ra_rrtmg_sw.F` |
| 5 | new Goddard | `module_ra_goddard.F` |
| 7 | FLG | |
| 24 | RRTMG (fast) | |

Production picks: **RRTMG longwave (4) + RRTMG shortwave (4)** is the
default for most real-data runs. CAM (3 + 3) is heavier but couples
naturally with WRF-Chem aerosol direct effects. Goddard (5 + 5) supports
direct + diffuse aerosol coupling.

`radt` (in minutes, in `&physics`) is the radiation call interval. Set
to roughly the grid spacing in km divided by 1 minute (so `radt=10`
for a 10 km run). Smaller is more accurate but more expensive.

Terrain slope and shading: `slope_rad = 1`, `topo_shading = 1`.

---

## Sub-grid turbulence (LES)

For LES (grid spacing of order 100 m or less), turn off the PBL
scheme (`bl_pbl_physics = 0`) and use 3D sub-grid turbulence in the
dynamics:

```
&dynamics
 km_opt      = 2     ! 2 = TKE prognostic; 3 = 3D Smagorinsky
 diff_opt    = 2     ! full diffusion in physical space
 mix_isotropic = 1
 sfs_opt     = 1     ! NBA (nonlinear backscatter, anisotropic) sub-grid model
/
```

Source: `dyn_em/module_diffusion_em.F`, `dyn_em/module_sfs_driver.F`,
`dyn_em/module_sfs_nba.F`. Test case: `compile em_les`.

---

## Gravity wave drag (`gwd_opt`)

Parameterized orographic gravity wave drag and flow blocking, important
for runs with heavy mountain terrain at coarser grid spacings.

| `gwd_opt` | Source |
|-----------|--------|
| 0 | off |
| 1 | original WRF GWD | `phys/module_bl_gwdo.F` |
| 3 | NCAR GSL GWD | `phys/module_bl_gwdo_gsl.F` |

---

## Stochastic perturbations

For ensemble forecasting WRF supports several stochastic schemes
(SKEBS, SPPT, SPP, RAND_PERTURB). Switches:

```
&stoch
 skebs                = 0  ! 1 to enable SKEBS
 sppt                 = 0  ! Stochastically Perturbed Parameterization Tendencies
 spp_conv             = 0  ! Stochastically Perturbed Parameters
 rand_perturb         = 0  ! random perturbation of initial conditions
/
```

Source: `phys/module_stoch.F` and the SKEBS code in `dyn_em/module_stoch.F`.
Read `STOCHPERT.TBL` for SKEBS parameters.

---

## A common combo: continental US 12 km real-data forecast

```
&physics
 mp_physics              = 8,    ! Thompson
 cu_physics              = 1,    ! Kain-Fritsch
 ra_lw_physics           = 4,    ! RRTMG
 ra_sw_physics           = 4,    ! RRTMG
 bl_pbl_physics          = 1,    ! YSU
 sf_sfclay_physics       = 1,    ! MM5
 sf_surface_physics      = 2,    ! Noah
 sf_urban_physics        = 0,
 radt                    = 12,
 bldt                    = 0,
 cudt                    = 0,
 num_soil_layers         = 4,
 num_land_cat            = 21,   ! MODIS-modified IGBP (default since v3.8)
/
```

---

## Where to next

- ARW dynamics that drive these physics: `dynamic-cores-and-em.md`
- Adding a chemistry mechanism: `wrf-chem.md`
- Land surface details: `wrf-hydro-coupling.md` and the companion noahmp-skill
- Failure modes ("the model crashed in microphysics"): `debugging.md`
- Where the namelist values come from: `architecture.md` (Registry section)
