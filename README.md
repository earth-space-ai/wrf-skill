# WRF Skill

A progressive-disclosure skill for the
[WRF](https://github.com/wrf-model/WRF) (Weather Research and Forecasting)
model.

> **Maintainer of WRF:** NCAR Mesoscale and Microscale Meteorology Laboratory (MMM)
> **Skill author:** Koutian Wu (ktwu01@gmail.com)
> **Skill version:** 0.1.0
> **WRF source version covered:** 4.7.1

> ⚠️ **Disclaimer — please read before using this skill.**
> This skill is **not a gold-standard reference**. It is a helper that lowers
> the barrier for new users to **get their hands dirty** with the model. AI
> agents (and the humans drafting this material) make mistakes; commands, file
> paths, namelist options, and physics explanations here can be wrong,
> incomplete, or out of date. **Always cross-check with the official model
> documentation, the source code, and a human expert before trusting any
> output for research, publication, or operational use.**

## What This Is

A self-contained knowledge package that teaches AI agents (and humans) how to
**install, configure, compile, run, modify, debug, and contribute to** WRF,
covering the Advanced Research WRF (ARW) dynamical core, the physics suite,
WRF-Chem, the WPS production workflow, idealized cases, and WRF-Hydro
coupling.

The skill captures the **procedural knowledge** that is normally only
transmitted by working alongside an experienced WRF developer: why
`./clean -a` is mandatory after a Registry edit, how `./configure` and
`./compile <case>` interact, why WPS lives in a separate repo, the
`real.exe` then `wrf.exe` order, and the gotchas around hybrid
sigma-pressure vs terrain-following vertical coordinates.

**Progressive disclosure:**
- `SKILL.md` is the routing hub: decision tree, repo layout, quick start, critical rules.
- `reference/*.md` are deep-dive docs loaded on demand.

## Contents

| Document | What is inside |
|----------|----------------|
| `SKILL.md` | Entry point: decision tree, repo layout, quick start, critical rules |
| `reference/getting-started.md` | Clone, prereqs, configure, compile, smoke test |
| `reference/architecture.md` | Source-tree map, Registry-driven build, frame / share / external layers |
| `reference/dynamic-cores-and-em.md` | ARW vs NMM, RK3 time-split, hybrid vertical coordinate |
| `reference/physics-options.md` | Microphysics, cumulus, PBL, radiation, surface layer, land surface |
| `reference/wrf-chem.md` | Chemistry mechanisms, aerosols, emissions |
| `reference/running-real-case.md` | WPS workflow, real.exe, wrf.exe, restart, nesting |
| `reference/running-idealized.md` | em_* test cases and ideal.exe |
| `reference/wrf-hydro-coupling.md` | WRF-Hydro build, namelist, routing, NWM |
| `reference/debugging.md` | Configure / compile / runtime failure modes |
| `reference/contributing-pr.md` | Fork model, JIRA, regression tests, code review |

## Sources

This skill is grounded in:

1. The **wrf-model/WRF** repository (release 4.7.1, April 2025): `README`,
   `LICENSE.txt`, `arch/`, `Registry/`, `dyn_em/`, `phys/`, `chem/`, `hydro/`,
   `run/`, `doc/`.
2. The **WRF Users' Guide v4** at https://www2.mmm.ucar.edu/wrf/users/docs/user_guide_v4/contents.html
3. The **WRF Users' web site** at https://www2.mmm.ucar.edu/wrf/users/
4. The **wrf-model/WRF GitHub wiki** at https://github.com/wrf-model/WRF/wiki
5. The **ARW v4 technical note**, Skamarock et al. 2019, NCAR/TN-556+STR.

Where a primary source could not be fetched at authoring time the doc links
out so the reader can retrieve it directly.

## Install

This skill follows the same layout as
[laps-skill](https://github.com/huangzesen/laps-skill) and the
[noahmp-skill](https://github.com/ktwu01/noahmp-skill):

```
wrf-skill/
|-- SKILL.md              <- routing hub (read first)
|-- README.md             <- this file
|-- LICENSE
|-- .gitignore
+-- reference/            <- deep-dive docs
```

To use with a Claude Code or LingTai agent, drop the directory into your
skills library and refresh.

## License

MIT for this skill package. WRF itself is in the public domain. See
https://www2.mmm.ucar.edu/wrf/users/public.html for the WRF public-domain
notice. WRF is a registered trademark of UCAR.
