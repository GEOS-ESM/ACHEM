# ACHEM: previous architectural decisions

**Audience:** coding agents and developers working on the `ACHEM` repository.
**Describes:** the MAPL3 line, branch `release/MAPL-v3`.
**Verified against:** `ACHEM` @ `30e63d0` (2026-11-06).
**Placement:** as in the GEOSgcm fixture (`components.yaml` @ `f50622d`).
Other fixtures (for example GEOSadas, GEOSldas) may place this repository
elsewhere; paths inside this repository do not change.

Only what is easy to see in the code now. Entries grow over time. Context:
[ARCHITECTURE.md](ARCHITECTURE.md).

### Specs created at run time from switches
- Pattern: `SetServices` reads `ACHEM_GridComp.rc` and adds import, internal and export specs only for the enabled parts (`mam_chem`, `voc_chem`, `ocs_chem`) (`ACHEM_GridCompMod.F90`, SetServices; `ACHEM_GridComp.rc`). [Verified]
- Likely reason: one build serves configurations that need only some of the chemistry, without unused imports needing data. [Inferred]
- Question for maintainers: the parent connects some of these fields whenever ACHEM is enabled; is each combination of switches and parent keys meant to work?

### Private configuration file read in SetServices
- Pattern: the component loads `ACHEM_GridComp.rc` itself with `ESMF_ConfigLoadFile`, not through MAPL resource keys (`ACHEM_GridCompMod.F90`, SetServices). [Verified]
- Likely reason: older component style, cloned from MAM in 2012 (`ACHEM_GridCompMod.F90`, `REVISION HISTORY`). [Inferred]

### Tracers kept in the internal state, named with the component prefix
- Pattern: species are internal specs named `ACHEM::<species>`, friendly to Dynamics, Turbulence, Moist and (some) MAM; the parent renames them on connection to MAM and MATRIX (`ACHEM_GridCompMod.F90`, SetServices; `GEOS_ChemGridComp.F90`, SetServices). [Verified]
- Likely reason: Dynamics, Turbulence and Moist handle these as friendly tracers without knowing the species. [Inferred]

### Gas-phase chemistry by a KPP-generated integrator
- Pattern: the sulfur and ammonia gas scheme is defined in `kpp/gas/kpp_achem_gas.eqn` and `.spc` and compiled from the generated Fortran; `gas_chemistry` calls it (`CMakeLists.txt`, `srcs`; `ACHEM_GridCompMod.F90`, `gas_chemistry`). [Verified]
- Likely reason: a stiff solver generated from a short reaction list. [Inferred]
- Question for maintainers: which KPP version and command produced the checked-in files?

### Fast aqueous scheme in use; full solver kept
- Pattern: Run_ calls `aqu_chemistry_fast`; the call to `aqu_chemistry` is commented out and the routine stays in the file (`ACHEM_GridCompMod.F90`, Run_). [Verified]
- Likely reason: run-time cost. [Inferred]

### Fixed OH, NO3 and H2O2 climatology with imposed diurnal cycle
- Pattern: oxidants are imports (from ExtData or from `GMICHEM`/`STRATCHEM`); ACHEM scales OH and NO3 by solar zenith angle when `apply_diurnal_cycle`, and refreshes an internal H2O2 copy every three hours (`ACHEM_GridCompMod.F90`, Run_). [Verified]
- Likely reason: climatological oxidants are daily means. [Inferred]

### Helper modules shared by emission, rates and deposition
- Pattern: `GACL_EmissionsMod`, `GACL_ReactionRatesMod`, `GACL_DryDepositionMod`, `GACL_ConstantsMod` hold numerical code and constants separate from the component (`CMakeLists.txt`, `srcs`). [Verified]
