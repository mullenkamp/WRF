# WVT Upstream Contribution Guide

## What this repo is

A fork of `wrf-model/WRF` (`github.com/mullenkamp/WRF`) at tag `v4.7.1`, with the Water Vapour
Tracer (WVT) modifications applied as a series of commits on `feature/water-vapor-tracers`. It is the integrated source
tree — the form the work would take if offered upstream. It is not the form it is developed or
deployed in.

For the current commit series and its area partitioning:

```bash
git log --oneline v4.7.1..HEAD
```

Tags:

- `wvt-4.7.1-single-region` — the single-region scheme exactly as compiled by the single-region
  production images.

## Where development actually happens

Development happens in the **Docker source overlays** in the `wrf-docker-builds` repository, not
here. An overlay is a partial mirror of the WRF tree; the Dockerfile untars a stock WRF release
and copies the overlay over it.

| Overlay | Status | Built by |
|---|---|---|
| `debian/wvt-multi/` | live — the current production source | the multi-region images |
| `debian/wvt-single/` | frozen — kept as an independent reference | the single-region images |
| `debian/wvt-ref/` | provenance — the original authors' WRF 4.3.3 modules, unmodified | the reference image |

**This repo is synced *from* the overlay, never the other way round.** The two are meant to be
byte-identical on every file the overlay contains; `debian/check_fork_sync.sh` in
`wrf-docker-builds` is what says whether they still are. Run it before trusting this tree.

## Syncing overlay changes into this repo

1. Make and validate the change in `debian/wvt-multi/`.
2. Copy the changed files here at the same relative path (the exceptions are the overlay's
   `test/` → `test/wvt/` here, and `MULTI_REGION_WIP.md` → the repo root).
3. Commit **by area**, following the existing convention: `Registry:`, `WSM6:`, `New Tiedtke:`,
   `Drivers:`, `Dynamics:`, `Diagnostics:`, `Validation:`, `Docs:`, `Tests:`. Never amend —
   fixes go on top. This branch is **published**, so rewriting it would break anyone who has
   fetched it; that includes tidying a bad comment or a stray path, which must now be a new commit.
4. Run `check_fork_sync.sh` and confirm it is clean.

**On the area partition.** The commits are *porting units*, not bisection points: the split
exists so each area can be re-applied and conflict-resolved independently against a future WRF
release. Several commits therefore do not compile alone. When a change has a partner in another
area — a routine's signature and its call site, a field's declaration and its use — **say so in
both commit messages**. Registry and Dynamics are the areas most likely to conflict when WRF
restructures.

## Scope limits of the multi-region path

Enforced in `share/module_check_a_mundo.F`, which is the authoritative statement:

- `num_wvt_regions` is 1..8. Raising the cap means regenerating the Registry with the
  `Registry/gen_wvt_*.py` generators, raising `MAX_WVT_REGIONS`, and rebuilding.
- `num_wvt_regions > 1` requires `tracer_opt = 4`, `bl_pbl_physics = 0`, and
  `tracer3dsource = tracer3dsink = 0`.
- YSU multi-region mixing and 3D source/sink are deliberately not wired. Multi-region cumulus
  transport is implemented for New Tiedtke only.
- `num_wvt_regions = 1` reproduces the single-region scheme bit-for-bit.

## Before attempting an upstream contribution

1. Open an issue on `wrf-model/WRF` describing WVT and the intent to contribute; get maintainer
   feedback on approach, style and testing before writing anything.
2. **Get permission from the owners of each modified scheme.** WSM6, YSU, KF, Tiedtke, MSKF and
   SMS-3DTKE have owners, and changes to them are not the contributor's to make unilaterally.
   Expect possible WRF Physics Review Panel (`wprp@ucar.edu`) involvement.
3. Target the **development** branch, never a release tag.
4. **Expect to rebase, and make sure you can.** This branch is built on `v4.7.1` and upstream moves
   continuously, so by the time a PR is opened the base will have shifted — port the commits area by
   area onto the current development branch rather than trying to merge. Two things to check before
   starting: that `upstream` points at `wrf-model/WRF` (`origin` should be your own fork), and that
   the clone has **full history** — a shallow clone cannot rebase onto a new release:

   ```bash
   git rev-parse --is-shallow-repository     # must be false
   git fetch --unshallow upstream            # if it is not
   ```

5. Re-check upstream's contribution policy at the time — including whether one now exists on
   AI-assisted contributions. As of 2026-07-18 there was none, and no CLA or DCO. Disclose the
   AI assistance in the PR description regardless.
6. Attribute the original scheme (see References).

### The `phys/physics_mmm/` problem — read this before planning a PR

Three of the most heavily modified files in this tree are **not part of the `wrf-model/WRF`
repository**:

```
phys/physics_mmm/mp_wsm6.F90
phys/physics_mmm/bl_ysu.F90
phys/physics_mmm/cu_ntiedtke.F90
```

`phys/physics_mmm` is not a submodule and has no files at the `v4.7.1` tag; it is pulled in by
`tools/manage_externals` from NCAR's MMM-physics repository. WRF 4.7.1 routes WSM6, YSU and New
Tiedtke through it, so the WVT changes to those schemes live there and **cannot be delivered by a
PR against `wrf-model/WRF` alone.** A contribution needs a coordinated change to the external
plus the WRF-side Registry, driver and dynamics changes.

Two practical consequences:

- **Running the externals checkout in this working tree will overwrite those three files.** This
  tree carries only the 3 WVT-modified files, not the external's full set, so it is also not
  self-sufficient for a build. Build from the Docker overlay instead.
- The `wrf-4.7.1-base` branch exists because those files are generated/external: it holds them at
  their stock 4.7.1 content so the WVT delta against them is isolable. `git diff wrf-4.7.1-base`
  on that directory is the real change.

## Related documentation

In `wrf-docker-builds/debian/wrf-wps-intel-wvt/`:

- `wvt-porting-notes.md` — the WRF 4.3.3 → 4.7.1 port, per-file, with validation results.
- `wvt-integration-guide.md` — how WVT works and how to add it to a new physics scheme.
- `wvt-source-edge-ringing.md` — numerical analysis of the hard-edged source mask.

In this repo:

- `MULTI_REGION_WIP.md` — the multi-region design record: invariants, stage history, validation.
- `run/README.tracers` — the minimal namelist recipe.
- `test/wvt/` — validation scripts for the multi-region scheme.

## References

Insua-Costa, D. and Miguez-Macho, G. (2018), "A new moisture tagging capability in the Weather
Research and Forecasting model: formulation, validation and application to the 2014 Great
Lake-effect snowstorm", Earth Syst. Dynam., 9, 167–185.

Original implementation: https://github.com/damianinsua/WRF-WVTs
