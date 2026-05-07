# Contributing to WRF via Pull Request

WRF is a community model developed primarily at NCAR/MMM. Contributions
flow through the GitHub repository at https://github.com/wrf-model/WRF
via the standard fork + pull-request model, with NCAR-specific
expectations around testing and release timing. This page walks through
the contribution path for a typical bug fix or new physics option.

The community contribution policy is documented at
https://www2.mmm.ucar.edu/wrf/users/wrf_users_guide/build/html/contribution.html
(WRF v4 User's Guide, "Contribution" chapter). Treat that page as the
authoritative source; this doc summarizes and adds tooling tips.

---

## The release cadence

WRF has a roughly annual major release (v4.0 in 2018, v4.1 in 2019, ...,
v4.7 in 2025) plus point releases for bug fixes (v4.7.1, v4.7.2). PRs
land on the `develop` branch; periodically a release branch
(`release-v4.7.1`) is cut from `develop` and tagged.

For a contribution to land in the next release you typically need to
have it merged into `develop` at least 2 months before the release
date. NCAR's WRF release coordinator publishes the schedule on the
users' web site.

---

## What kinds of changes are accepted

| Change type | Expected? | Notes |
|-------------|-----------|-------|
| Bug fix in existing physics, dynamics, I/O | Yes | Need a clear repro and a regression test |
| New physics option (microphysics, PBL, etc) | Yes | Need a publication or technical note describing the scheme; need regression test results |
| New idealized test case | Yes | Need a `test/em_<name>/` directory with namelist, sounding, and a brief README |
| New compiler / platform support (`arch/configure.defaults` stanza) | Yes | Need to demonstrate end-to-end compile + run on the new platform |
| Code refactoring with no behavior change | Yes, but reviewed cautiously | Need to show bit-for-bit reproducibility on a regression suite |
| Style-only changes (whitespace, capitalization) | Discouraged | Adds review burden without benefit |
| Removal of an existing scheme | Almost never | WRF prizes backward compatibility for production users |

---

## Step 1: Open an issue first (for non-trivial changes)

Before writing code for anything beyond a one-line bug fix, open a
GitHub issue describing:

1. The problem or proposed feature.
2. The intended approach (which files, which Registry changes, which
   namelist option).
3. The test plan (which idealized or real-data case will demonstrate the
   change).

The WRF developers will typically comment on whether the approach is
acceptable, and may flag conflicts with parallel work. This avoids
multi-month rework after submission.

---

## Step 2: Fork and clone

```bash
# Fork on the GitHub web UI: https://github.com/wrf-model/WRF -> Fork
git clone https://github.com/<you>/WRF.git
cd WRF
git remote add upstream https://github.com/wrf-model/WRF.git
git fetch upstream
git checkout -b fix/my-thing upstream/develop
```

WRF uses **`develop`** as the integration branch. Always branch from
`upstream/develop`, not `master`. `master` typically tracks the latest
released version.

Branch naming is informal but conventionally:
- `fix/<short-desc>` for bug fixes
- `feature/<short-desc>` for new features
- `refactor/<short-desc>` for code refactoring
- `arch/<platform>` for new compiler stanzas

---

## Step 3: Make the change

Concrete tactics:

1. **Edit Registry first if your change adds state, namelist options,
   or I/O streams.** This drives the rest of the design.
2. **Edit the physics or dynamics module.** Keep changes isolated; do
   not refactor neighboring code.
3. **Update `run/README.namelist`** if you added or changed a namelist
   option.
4. **Update `doc/README.<feature>`** if your change touches a
   documented feature, or add a new README if it is a new feature.
5. **Build the fresh code from a clean tree** (`./clean -a`) and run
   the relevant test case end-to-end.

For physics additions, also:
- Add the new option value to the appropriate `package` lines in
  `Registry.EM_COMMON` and the corresponding `case` block in the
  driver (e.g. `phys/module_microphysics_driver.F`).
- If the new scheme has tunable parameters, put them in a `.TBL` file
  under `run/` rather than hard-coding.

---

## Step 4: Commit hygiene

WRF maintainers prefer small, atomic commits that build cleanly at
each step. Avoid one giant commit at the end. Suggested commit message
form:

```
<area>: <short summary>

<body explaining the why and the what, in plain prose, wrapped at 72 chars>

Closes #<issue-number>
```

Where `<area>` is one of `dyn_em`, `phys`, `chem`, `hydro`, `frame`,
`share`, `Registry`, `arch`, `doc`, `tools`, etc.

Example:

```
phys: fix YSU PBL height calculation for stable boundary layer

YSU previously used an unstable-profile formula for h when the bulk
Richardson number indicated stability. This caused unphysically deep
stable PBLs at night over snow surfaces. The fix uses the standard
stable-side closure from Hong (2010) when Rib > 0.

Closes #1234
```

---

## Step 5: Run the regression test

The WRF Test suite under `test/` is not a formal unit-test framework;
it is a set of test cases that produce reference output. The minimum
acceptable regression test for a contribution:

1. Pick the most relevant idealized case (`em_b_wave` for dynamics,
   `em_quarter_ss` for microphysics, `em_les` for PBL/turbulence,
   `em_seabreeze2d_x` for surface).
2. Compile both the unmodified and the modified WRF.
3. Run both with the same `namelist.input` and `wrfinput`.
4. Diff the `wrfout`. For a pure bug fix the diff should be small
   and explainable. For a refactor the diff should be exactly zero
   (`cdo diffn` or `nccmp -d`).

For new physics options that are switched off by default (i.e. existing
users keep the old behavior), demonstrate bit-for-bit reproducibility
when the new option is OFF, and show a sensible result when it is ON.

If your change touches the dynamics, the WRF developers will also run
their internal regression tests on multiple platforms before merge.
That can take a week or two of calendar time.

---

## Step 6: Open the pull request

```bash
git push -u origin fix/my-thing
# Then on GitHub, open a PR from <you>:fix/my-thing into wrf-model/WRF:develop
```

PR description should include:
- **Issue link**: `Fixes #1234` or `Refs #1234`.
- **Summary**: 1-2 paragraphs in plain English explaining what
  changed and why.
- **Test results**: which case you ran, expected vs observed
  outcome, screenshot or numerical diff if relevant.
- **Backward compatibility**: state explicitly whether existing
  namelists continue to work unchanged.
- **Documentation updates**: list the README files / User's Guide
  sections that need updating (you should already have updated them
  in the PR).

---

## Step 7: Code review

A WRF maintainer will review. Expect:
- Style comments (capitalization, indentation; WRF mostly uses fixed
  uppercase Fortran).
- Requests for additional test cases.
- Questions about edge cases (HVC vs TF, two-way nest, restart
  reproducibility).
- Requests to split unrelated changes into separate PRs.

Iterate by pushing more commits to the same branch (do not force-push
once review has started, as that loses comment threads).

When the maintainer marks the PR ready, it will be merged into
`develop`. It will appear in the next release.

---

## Special workflows

### JIRA

Historically NCAR's internal WRF tracker was on JIRA. New external
contributions go through GitHub Issues + PRs only. Internal NCAR
developers may still use JIRA for project planning, but you do not need
to touch it.

### WRFDA (`var/`) and WRFPLUS (`wrftladj/`)

These sub-projects have their own review process and reviewers.
Contributions touching them should be flagged in the PR description so
the relevant reviewer is tagged.

### WRF-Chem (`chem/`)

WRF-Chem has a separate active developer community at NOAA/CSL and
NCAR/ACOM. PRs touching `chem/` may be routed to those reviewers.
Major chemistry additions are usually coordinated via the WRF-Chem
mailing list before a PR is opened.

### WRF-Hydro (`hydro/`)

The `hydro/` directory is mirrored from the
NCAR/wrf_hydro_nwm_public repository. Contributions to the hydro
physics should usually go to that upstream repo, not directly to WRF.
The WRF maintainers periodically refresh the in-tree mirror.

---

## Where to next

- The Registry and build system you will need to edit: `architecture.md`
- The physics or dynamics where you are adding code: `physics-options.md`, `dynamic-cores-and-em.md`
- Test cases you will use for regression testing: `running-idealized.md`
- Real-data case if you need a more realistic regression check: `running-real-case.md`
- Community: https://forum.mmm.ucar.edu/ and https://github.com/wrf-model/WRF/discussions
