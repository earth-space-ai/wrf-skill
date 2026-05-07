# WRF-Chem

WRF-Chem is the chemistry-coupled version of WRF. It adds gas-phase
chemistry, aerosol microphysics and optics, and emissions handling to the
standard ARW core. Unlike WRF-Hydro (an external library), WRF-Chem lives
in the WRF source tree under `chem/` and is enabled by an environment
variable at build time.

This doc covers what is in the chem/ subtree, the major mechanisms, and
the namelist options that drive a typical WRF-Chem run. The community
documentation is at https://www2.acom.ucar.edu/wrf-chem.

---

## Building WRF-Chem

```bash
export WRF_CHEM=1                    # turn on chem build
export WRF_KPP=1                     # turn on KPP-based mechanisms (recommended)
export YACC='/usr/bin/yacc -d'       # KPP needs yacc
export FLEX_LIB_DIR='/usr/lib/x86_64-linux-gnu'  # adjust to your install
./configure                          # the menus are unchanged
./compile em_real >& compile.log
```

Without `WRF_KPP=1` you only get the legacy hand-coded mechanisms
(RADM2, CBMZ, MOZART subset). With `WRF_KPP=1` the build runs the
**KPP** (Kinetic PreProcessor) over the `chem/KPP/` mechanism definition
files at compile time, generating Fortran solvers for the full
catalogue.

The chem build adds substantial compile time and produces a `wrf.exe`
that includes chemistry. There is no separate `wrfchem.exe`.

---

## What is in `chem/`

The `chem/` directory holds roughly 203 source files. The structural
breakdown:

| Group | Examples | Role |
|-------|----------|------|
| Drivers | `chem_driver.F`, `emissions_driver.F`, `aerosol_driver.F`, `mechanism_driver.F`, `dry_dep_driver.F`, `cloudchem_driver.F` | Top-level dispatch from the main timestep loop |
| Initialization | `chemics_init.F`, `convert_emiss.F` | Read emissions, set up mechanism tables |
| Gas-phase mechanisms (legacy hand-coded) | `module_radm.F`, `module_cbmz.F`, `module_mozart_*` | Pre-KPP solvers |
| KPP-generated mechanisms | `KPP/` subtree, plus generated `*_Integrator.F` etc | RADM2-KPP, RACM-KPP, CBMZ-KPP, MOZART-T1-KPP, MOZCART, SAPRC |
| Aerosols (sectional) | `module_mosaic_*`, `module_aerosols_sorgam.F`, `module_aerosols_sorgam_vbs.F` | MOSAIC, SORGAM, SORGAM-VBS |
| Aerosols (modal, CESM MAM) | `module_cam_mam_*` | CESM MAM3, MAM4, MAM7 |
| Biogenic emissions | `module_bioemi_simple.F`, `module_bioemi_beis314.F`, `module_bioemi_megan2.F` | BEIS, MEGAN2 |
| Dust | `module_uoc_dust.F`, `module_data_gocart_dust.F`, `module_gocart_*` | GOCART dust, UoC dust |
| Sea salt | `module_data_seasalt.F` | GOCART sea salt |
| Wildfires | `module_add_emiss_burn.F`, `module_plumerise1.F` | Plumerise |
| Anthropogenic emissions | `module_add_emis_cptec.F`, `module_aerosol_emis.F` | Anthropogenic preprocessing |
| Optics | `module_optical_averaging.F`, `module_aer_opt_out.F` | Aerosol radiative properties |
| Wet / dry deposition | `module_dep_simple.F`, `module_aer_drydep.F`, `module_wetscav_driver.F` | Removal processes |
| Heterogeneous chemistry | `module_aerosols_soa_vbs_het.F` | Aqueous chemistry on aerosols |

The full mechanism list is in the Registry file `registry.chem` plus
`registry.wrfchemvar`.

---

## Choosing a mechanism: `chem_opt`

The single most important namelist option in `&chem`. Each value selects
a coupled gas-phase + aerosol package. A non-exhaustive but representative
list:

| `chem_opt` | Gas mechanism | Aerosol package | Notes |
|------------|---------------|-----------------|-------|
| 0 | none | tracers only | for transport-only experiments |
| 1 | RADM2 | none | gas only, classic |
| 2 | RADM2 + SORGAM | yes | OG photolysis |
| 7 | CBMZ | MOSAIC 4-bin | popular for North America |
| 8 | CBMZ | MOSAIC 8-bin | finer aerosol resolution |
| 10 | RADM2 + SORGAM-VBS | yes | volatility basis set |
| 11 | RACM-KPP | none | regional acid chemistry |
| 12 | RACM-MIM | MADE/SORGAM | |
| 31 | MOZART | none | global tropospheric |
| 32 | MOZCART | GOCART aerosols | MOZART + GOCART |
| 41 | CB05 | MOSAIC | CB05 carbon-bond |
| 100+ | various MOZART-MOSAIC, SAPRC, MAM combinations | | research |
| 198 | MOZART-T1 | MOSAIC 4-bin | troposphere-stratosphere |
| 200 | MOZART-T1 | MOSAIC 4-bin VBS | |

The full list is documented in the WRF-Chem User's Guide at
https://www2.acom.ucar.edu/sites/default/files/wrf-chem/. Treat the
namelist tables in that PDF as authoritative; the source `Registry/registry.chem`
is the actual ground truth.

---

## Aerosol packages

| Package | Source files | Treatment | Notes |
|---------|-------------|-----------|-------|
| **GOCART** | `module_gocart_*` | bulk + sectional dust + sea salt | NASA, simple, fast |
| **MADE/SORGAM** | `module_aerosols_sorgam.F` | modal | original WRF-Chem aerosol |
| **MADE/SORGAM-VBS** | `module_aerosols_sorgam_vbs.F` | modal + volatility basis set for SOA | newer SOA chemistry |
| **MOSAIC** (4 or 8 bin) | `module_mosaic_*` | sectional | most-used for regional research |
| **CESM MAM3 / MAM4 / MAM7** | `module_cam_mam_*` | modal | for global / climate-style runs coupled with CAM physics |

Pick MOSAIC for chemistry-heavy regional runs where you want PM2.5
size-resolved. Pick MAM for runs that need CAM-compatible aerosol-cloud
interactions. Pick GOCART for fast dust + sea salt + sulfate runs.

---

## Emissions

Two kinds of emissions feed WRF-Chem:

### Anthropogenic (offline preprocessed)

You preprocess external emission inventories (NEI, EDGAR, EPA, MEIC for
China, ...) into NetCDF files in WRF's grid, then read them via the
auxinput stream. The standard preprocessor is `prep_chem_sources` (a
separate utility). Output files are named `wrfchemi_d<dom>_<datetime>`
and read via:

```
&chem
 emiss_inpt_opt          = 1,        ! emission input format (1 = NETCDF)
 emiss_opt               = 3,        ! emission scheme: 3 = RADM2 / SORGAM, 4 = CBMZ, ...
 io_style_emissions      = 2,        ! one file per day (1) or one per run (2)
 emiss_inpt_opt_aer      = 0,
 kemit                   = 8,        ! number of vertical emission levels
/
```

### Biogenic (online)

| `bio_emiss_opt` | Scheme | Source |
|-----------------|--------|--------|
| 0 | none | |
| 1 | Simple | `module_bioemi_simple.F` |
| 2 | BEIS3.14 | `module_bioemi_beis314.F` |
| 3 | MEGAN2 | `module_bioemi_megan2.F` |

MEGAN2 is the standard choice. It needs the `wrfbiochemi_d<dom>` input
file (preprocessed offline using land cover and PFT data) read via
`auxinput6`.

### Dust (online)

| `dust_opt` | Scheme |
|-----------|--------|
| 0 | none |
| 1 | GOCART dust |
| 3 | UoC dust (Shao 2004 / 2011) |
| 13 | GOCART AFWA (default for AFWA configs) |

### Sea salt, fires, dimethyl sulfide

```
&chem
 dmsemis_opt    = 1,    ! GEIA DMS oceanic emissions
 seas_opt       = 2,    ! 1 = GOCART sea salt, 2 = MOSAIC sea salt
 biomass_burn_opt = 0,  ! 1 = MOZBC fire emissions, 2 = WRF-Fire coupling, ...
 plumerisefire_frq = 30,
/
```

---

## Boundary conditions

Lateral boundary chemistry comes from a global model, typically MOZART or
WACCM. The `mozbc` utility (separate from WRF-Chem) reads MOZART output
and writes to `wrfbdy_d01` and `wrfinput_d01` so chemistry has plausible
inflow.

```
&chem
 chem_in_opt    = 1,    ! 0 = idealized profiles, 1 = read from wrfinput
 have_bcs_chem  = .true.,
 chem_bc_inname = "wrfbdy_d01",
/
```

---

## Photolysis

The Fast-J photolysis scheme is the default. The Madronich TUV is
optional. Photolysis is selected by `phot_opt`:

| `phot_opt` | Scheme | Source |
|-----------|--------|--------|
| 0 | none | |
| 1 | Madronich F-TUV | `module_ftuv_*` |
| 2 | **Fast-J** | `module_phot_fastj.F` (most common) |
| 3 | Fast-J for tropospheric | |

---

## Aerosol-radiation and aerosol-cloud interactions

Aerosol direct effect (radiation):

```
&chem
 aer_ra_feedback = 1,    ! 0 off, 1 on
 aer_op_opt      = 1,    ! aerosol optical property scheme: 1 = volume mixing, 2 = Maxwell-Garnett shell, 3 = exact Mie
/
```

Aerosol indirect effect (clouds): only when paired with the right
microphysics (Lin double-moment, Morrison 2-moment with Abdul-Razzak
activation, or the CAM5.1 mp). Set via `progn = 1` when supported.

---

## Coupling with WRF-Hydro

WRF-Chem coexists with WRF-Hydro (you can have `WRF_CHEM=1` and
`WRF_HYDRO=1`). Aerosol deposition fluxes can feed into the runoff
quality, but that path is a research feature, not a production path.

---

## Common namelist additions for WRF-Chem

Beyond the standard `&time_control`, `&domains`, `&physics`, you add
`&chem`:

```
&chem
 kemit                = 8,
 chem_opt             = 7,        ! CBMZ + MOSAIC 4-bin
 bioemdt              = 30,       ! biogenic emissions update interval (min)
 photdt               = 30,
 chemdt               = 5,        ! gas-phase chemistry timestep (min)
 io_style_emissions   = 2,
 emiss_inpt_opt       = 1,
 emiss_opt            = 4,        ! CBMZ
 chem_in_opt          = 1,
 phot_opt             = 2,        ! Fast-J
 gas_drydep_opt       = 1,
 aer_drydep_opt       = 1,
 bio_emiss_opt        = 3,        ! MEGAN2
 dust_opt             = 1,        ! GOCART dust
 dmsemis_opt          = 1,
 seas_opt             = 2,
 depo_fact            = 0.25,
 aer_ra_feedback      = 1,
 have_bcs_chem        = .true.,
/
```

You also set `auxinput5_inname = "wrfchemi_d<domain>_<date>"` and
`auxinput5_interval` in `&time_control` to load anthropogenic emissions
on schedule.

---

## Where to next

- The build system: `architecture.md`
- Reading emission files / boundary conditions: `running-real-case.md`
- Compile failures specific to KPP: `debugging.md`
- Underlying microphysics that aerosol-cloud interactions need: `physics-options.md`
- WRF-Chem User's Guide: https://www2.acom.ucar.edu/wrf-chem
