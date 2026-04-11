# WVT Upstream Contribution Guide

## Current State

This repo (`WRF-WVT`) is a local reference of WVT changes against WRF 4.7.1. It has 11 clean commits on the `feature/water-vapor-tracers` branch, each targeting a specific scheme or component. It is NOT a GitHub fork -- it's a local clone at the v4.7.1 tag with our changes on top.

The working development copy lives in the Docker overlay at:
```
/home/mike/git/wrf-repos/wrf-docker-builds/debian/wvt/
```

## Two Concerns

### 1. Ongoing Development (Fixes, Features, New Schemes)

**Workflow:**
1. Make changes to the overlay files in `debian/wvt/`
2. Rebuild Docker images and test
3. Periodically sync changes to this repo:
   ```bash
   cd /home/mike/git/wrf-repos/WRF-WVT
   # Copy updated overlay files
   cp /path/to/debian/wvt/phys/module_cu_kfeta.F phys/
   # Commit with descriptive message
   git add phys/module_cu_kfeta.F
   git commit -m "KF: Fix division by zero in tracer downdraft evaporation"
   ```
4. For bug fixes in existing schemes, add new commits on top (don't amend -- preserves history)
5. For new schemes, add new commits following the existing pattern

### 2. Eventual Upstream Contribution

**Prerequisites before starting:**
1. Open a GitHub issue on `wrf-model/WRF` describing WVT and expressing interest in contributing
2. Get maintainer feedback -- they may have preferences for implementation approach, code style, testing
3. Check if anyone else is working on similar tracer functionality
4. Review WRF's contribution guidelines (CLA, code style, testing requirements)
5. Determine which version to target (likely the latest development branch, not a release)

**Workflow when ready:**
1. Fork `wrf-model/WRF` on GitHub (creates `your-username/WRF`)
2. Clone your fork locally
3. Create a feature branch from their **development branch** (NOT from a release tag):
   ```bash
   git clone https://github.com/your-username/WRF.git WRF-upstream
   cd WRF-upstream
   git checkout -b feature/water-vapor-tracers origin/develop  # or main
   ```
4. Use the commits in this repo as a guide to apply changes to the new version:
   - Port one commit at a time (Registry, WSM6, YSU, KF, etc.)
   - Resolve any conflicts from WRF version changes
   - Test each scheme after porting
5. Push to your fork:
   ```bash
   git push origin feature/water-vapor-tracers
   ```
6. Open a PR against `wrf-model/WRF`'s development branch

**Important:** The PR targets the DEVELOPMENT branch, not a release. Maintainers don't accept changes to released versions.

## Commit Structure Reference

Each commit in this repo targets a specific area, making porting manageable:

```
d711bd3 Registry: Add WVT moisture tracer state variables, tendencies, masks, and namelist options
8dbc18d WSM6: Add moisture tracer mass-fraction tracking
c266531 YSU: Add tracer vertical mixing via qmix infrastructure with surface flux
03327b6 Kain-Fritsch: Add moisture tracer transport through convective mass flux
4bde403 New Tiedtke: Add tracer flux-divergence transport with cloud detrainment
495ee6b Multi-scale KF: Add moisture tracer transport (KF pattern with scale-awareness)
92a460e SMS-3DTKE: Add tracer surface flux injection to implicit and explicit solvers
5c313b6 Drivers: Thread tracer arguments through physics drivers and tendency accumulation
7d0b196 Dynamics: Add tracer source/sink, flux diagnostics, auxinput8 I/O, and diffusion threading
890ed8e Validation: Add namelist consistency checks for WVT scheme combinations
75c02cd Docs: Add WVT tracer README and example namelist
```

When porting to a new WRF version, work through these in order. The Registry and Dynamics commits are most likely to have conflicts (WRF restructures these across versions). The physics scheme commits (WSM6, YSU, KF, etc.) are more self-contained.

## Related Documentation

- `wrf-docker-builds/debian/wrf-wps-intel-wvt/wvt-porting-notes.md` -- Detailed porting notes and validation results
- `wrf-docker-builds/debian/wrf-wps-intel-wvt/wvt-integration-guide.md` -- How to add WVT to new physics schemes
- `wrf-docker-builds/debian/wrf-wps-intel-wvt/sms-3dtke-wvt-status.md` -- SMS-3DTKE implementation details

## Reference

Insua-Costa, D. and Miguez-Macho, G. (2018), "A new moisture tagging capability in the Weather Research and Forecasting model: formulation, validation and application to the 2014 Great Lake-effect snowstorm", Earth Syst. Dynam., 9, 167-185.

Original repository: https://github.com/damianinsua/WRF-WVTs
