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

### `phys/physics_mmm/` is an external — and the WVT changes live in a fork of it

WRF routes WSM6, YSU and New Tiedtke through `phys/physics_mmm/`, which is **not part of the
`wrf-model/WRF` repository**. It is pulled by `tools/manage_externals` from what
`arch/Externals.cfg` points at. Upstream that is `NCAR/MMM-physics` @ tag `20240626-MPASv8.2`;
here it is:

```
repo_url = https://github.com/mullenkamp/MMM-physics.git
branch   = feature/water-vapor-tracers
```

That branch is cut **from the pinned tag**, so the physics baseline is byte-identical to what WRF
4.7.1 expects — it adds the WVT changes and nothing else (`mp_wsm6.F90` +1100, `cu_ntiedtke.F90`
+440, `bl_ysu.F90` +12). MMM-physics `main` has since moved 19 commits ahead and altered
`bl_ysu.F90`; it is deliberately not used.

**This repo tracks no files under `phys/physics_mmm/`**, exactly as upstream does. To build:

```bash
git clone -b feature/water-vapor-tracers https://github.com/mullenkamp/WRF.git
cd WRF
./tools/manage_externals/checkout_externals -e arch/Externals.cfg   # note: -e, it is not at the root
./configure && ./compile em_real
```

**Consequences for an upstream PR.** The WVT work is in two halves and both are required: the
Registry, driver and dynamics changes here, and the three physics schemes in the external. A PR
against `wrf-model/WRF` alone cannot deliver the WSM6/YSU/New Tiedtke changes — those need a
coordinated PR to `NCAR/MMM-physics`, and the `arch/Externals.cfg` change here would be reverted to
point back at whatever tag upstream settles on.

**One historical wrinkle:** the `wvt-4.7.1-single-region` tag predates this restructuring and still
carries the three files as tracked content. A clone *at that tag* therefore hits the old conflict —
`checkout_externals` refuses to populate a directory that already exists with tracked files in it.
The tag is a provenance marker, not a build target.

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
