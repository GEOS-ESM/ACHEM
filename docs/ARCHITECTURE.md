# ACHEM: architecture

**Audience:** coding agents and developers working on the `ACHEM` repository.
**Describes:** the MAPL3 line, branch `release/MAPL-v3`.
**Verified against:** `ACHEM` @ `30e63d0` (2026-11-06).
**Placement:** as in the GEOSgcm fixture (`components.yaml` @ `f50622d`), key `ACHEM`.
Other fixtures (for example GEOSadas, GEOSldas) may place this repository
elsewhere; paths inside this repository do not change.

**Build status on this line:** not built; not yet ported to MAPL3. On
`release/MAPL-v3`, `@GEOSchem_GridComp/CMakeLists.txt` calls
`esma_add_subdirectories(GOCART)` only, with the comment "Restore subdirs as
components are ported to MAPL3."; `ACHEM` appears only in a commented-out list
of subdirectories. The Chemistry top-level library (`GEOS_ChemGridComp.F90`,
which adds this component) is present but not compiled: the
`esma_add_library` block that builds it is commented out in `../CMakeLists.txt`.
Build dependency `Chem_Shared` (from the Chemistry `Shared`
directory) is not built on this line either. The code in this repository is the MAPL2
code, kept for a later port; it differs from `develop` only in
`GACL_ConstantsMod.F90`, which imports `MAPL_Constants` instead of
`MAPL_ConstantsMod` (the other changes there are trailing whitespace).

Parent: the Chemistry component, see
`GEOSchem_GridComp/docs/ARCHITECTURE.md` (`../../docs/ARCHITECTURE.md` in the fixture)
(how children are added, run phases, connections, the
`AERO` state). This file does not repeat it. Previous decisions:
[decisions.md](decisions.md).

## 1. Placement and map of the repository

In the fixture the repository is checked out at
`src/Components/@GEOSgcm_GridComp/GEOSagcm_GridComp/GEOSphysics_GridComp/@GEOSchem_GridComp/@ACHEM`.
It is a single gridded component, library `ACHEM`, plus helper modules and one
generated chemistry solver (`CMakeLists.txt`, `esma_add_library`).

| Path | Content |
|---|---|
| `ACHEM_GridCompMod.F90` (3,870 lines) | The gridded component `ACHEM_GridCompMod` (public: `SetServices` only) and its routines `Initialize_`, `Run_`, `Finalize_`, `extract_`, `GetVolcDailyTables`, `gas_chemistry`, `aqu_chemistry_fast`, `aqu_chemistry`, `ocs_chemistry`, `solar_zenith_angle` |
| `GACL_ConstantsMod.F90` | Physical constants and molecular weights (`pi`, `g_earth`, `N_avog`, `mw_*`) |
| `GACL_EmissionsMod.F90` | `NH3_Emissions`, `SO2_Emissions`, `DMS_Emissions`, `SOAG_Emissions`, `VOC_Emissions` (public) |
| `GACL_ReactionRatesMod.F90` | Henry's law function `henry` and constants `H_*_298`, `E_R_*` used by the aqueous scheme |
| `GACL_DryDepositionMod.F90` | `DepositionVelocity` (generic over aerosol and gas) |
| `kpp/gas/` | Output of the Kinetic PreProcessor (KPP) for the gas-phase sulfur and ammonia scheme: `.kpp`, `.def`, `.spc`, `.eqn` sources and the generated `kpp_achem_gas_*.f90` |
| `ACHEM_GridComp.rc` | Private configuration (switches, factors) |
| `ACHEM_Registry.rc` | Registry for the automatic code generator (ACG) |
| `ACHEM_ExtData.rc`, `ACHEM_ExtData.yaml` | Emission and climatology files for the `ExtData` component |
| `AMIP/`, `AMIP.20C/` | Variant resource files, section 5 |
| `CMakeLists.txt`, `README.md` (one line), `CHANGELOG.md` | build, title, history |

Built from `kpp/gas/`: `Precision, Parameters, Global, Function, JacobianSP,
Jacobian, HessianSP, Hessian, StoichiomSP, Stoichiom, Rates, Monitor, Util,
LinearAlgebra, Integrator` (`CMakeLists.txt`, `srcs`). Not built: `Main`,
`Model`, `Initialize`, `mex_*`, `Makefile_kpp_achem_gas`. The generated
`.f90` files were not read for this document. `*.rc` and `*.yaml` files in
all directories are installed under `etc/<dir>` (`CMakeLists.txt`, `rc_files`).

## 2. Purpose
ACHEM ("aerosol chemistry") is a GEOS Chemistry child that holds a small
chemistry package: gas-phase and aqueous-phase sulfur and ammonia chemistry
for modal aerosol schemes, oxidation of volatile organic compounds (VOC) to
secondary organic aerosol, and carbonyl sulfide (OCS) loss producing SO2
(`ACHEM_GridCompMod.F90`, module header; Run_). *Inferred:* "gas and aqueous
phase chemistry" in the parent's table is the same scheme.

## 3. Place in the hierarchy
- Parent: `GEOSchem_GridComp`, which adds ACHEM with
  `MAPL_AddChild(GC, NAME='ACHEM', SS=AChem_SetServices)` when
  `ENABLE_ACHEM` is true (`GEOS_ChemGridComp.F90`, SetServices). The shipped
  `GEOS_ChemGridComp.rc` sets it `.TRUE.`.
- Children: none.
- Run phases: one (`ESMF_METHOD_RUN` only). The parent calls it in
  Run2: Run1 runs only children with `NPHASE > 1`, and Run2 uses `IPHASE = 2` if `NPHASE > 1`, else 1 (`GEOS_ChemGridComp.F90`, Run1, Run2). The component returns at once unless
  its own run alarm rings, and turns the alarm off when it does (`ACHEM_GridCompMod.F90`, Run_, `ESMF_AlarmIsRinging`, `ESMF_AlarmRingerOff`);
  the interval comes from `ACHEM_DT` (parent document).
- Connections made by the parent (all in `GEOS_ChemGridComp.F90`, SetServices):
  - to ACHEM from `CHEMENV`: `AIRDENS`, `DELP`, `CN_PRCP`, `NCN_PRCP`;
  - from `PCHEM`: `O3`; from `STRATCHEM` or `GMICHEM`: `O3P`, `OH` (as `OHSTRAT`), `O3CHEM`/`OX` (as `O3`), `OCS_JRATE`; from `GMICHEM`: `OH`, `H2O2`, `NO3`;
  - from ACHEM to `MAM` (`ACHEM::SO2`, `ACHEM::H2SO4`, `ACHEM::NH3`, `ACHEM::SOAG` renamed `SOA_GAS`, `pSO4_aq`, `pNH4_aq` and tendency fields), `MATRIX` (`SO2`, `H2SO4`, `NH3`, `SOAG`, `pSO4_aq`, `pNH4_aq`), `CARMA` (`pSOA_ANTHRO_VOC`, `pSOA_BIOB_VOC`), `GOCART2G` (`pSOA_ANTHRO_VOC`, `pSOA_BIOB_VOC`, and `pSO2_OCS` when `chemReg%doing_OCS`).
- ACHEM calls no sibling directly. It uses `DryDepositionGOCART` from the shared `Chem_Shared` library (`ACHEM_GridCompMod.F90`, `use DryDepositionMod`).

## 4. Entry points
- **SetServices:** allocates the private `AChem_State`, reads `ACHEM_GridComp.rc` (fixed file name, loaded with `ESMF_ConfigLoadFile`), derives `mam_chem = gas_chemistry .or. aqueous_chemistry`, registers Initialize, Run and Finalize with `MAPL_GridCompSetEntryPoint`, stores the state with `ESMF_UserCompSetInternalState` under `AChem_State`, adds import, internal and export specs depending on the switches, adds timers (`TOTAL`, `RUN`, `-EMISSIONS`, `-CHEMISTRY`, `--CHEMISTRY_GAS`, `--CHEMISTRY_AQUEOUS`, `--CHEMISTRY_VOC`, `--CHEMISTRY_OCS`, `INITIALIZE`) and calls `MAPL_GenericSetServices`.
- **Initialize_:** `MAPL_GenericInitialize`; `extract_`; resets the volcanic-emission table (`nymd_volcanic_emiss = -1`); if aqueous chemistry is on, allocates the private H2O2 buffer `self%h2o2` and fills it from the `H2O2` import (the code has `using_GMI_H2O2 = .false.`, "coupling with GMI is not implemented").
- **Run_** (one phase), in order, each part guarded by the switches:
  1. Emissions (timer `-EMISSIONS`): daily volcanic table via `GetVolcDailyTables` (when the date changes); an OH and NO3 diurnal cycle from the solar zenith angle when `apply_diurnal_cycle` and `mam_chem` or `voc_chem`; then `NH3_Emissions`, `SO2_Emissions`, `DMS_Emissions`, `SOAG_Emissions`, `VOC_Emissions` add mass to the internal tracers.
  2. Dry deposition of the lowest layer (`DryDepositionGOCART`, `DepositionVelocity`, exponential loss over the time step) for the `mam_chem` species.
  3. Floor tracers at `tiny(0.0)`.
  4. Chemistry (timer `-CHEMISTRY`): gas phase `gas_chemistry` (calls the KPP integrator), aqueous phase `aqu_chemistry_fast`, VOC oxidation by OH, OCS loss `ocs_chemistry`.
  5. Diagnostics exports (concentrations, column densities, tendencies, surface values). H2O2 buffer refresh: when aqueous chemistry is on, `self%h2o2` is reset from the `H2O2` import at model hours divisible by 3 (Run_).
- **Finalize_:** `MAPL_GenericFinalize`, then deallocates the volcano tables, the H2O2 buffer and the private state.
- `aqu_chemistry` (the full solver using `aqueous_chemistry_solver_max_dt`) is compiled but its call in Run_ is commented out; `aqu_chemistry_fast` is called. `extract_` is the usual helper (grid bounds, date, time step).

## 5. State
- **Private state** `AChem_State` (type with `ESMF_Config`, switches, VOC and OCS constants, volcano tables, H2O2 buffer) (`ACHEM_GridCompMod.F90`).
- **Imports**, written by hand with `MAPL_AddImportSpec` in SetServices. Always: `AIRDENS`, `DELP`, `PLE`, `T`. If `mam_chem`: `AREA`, `ZLE`, `QLTOT`, `FCLD`, `U10N`, `V10N`, `FROCEAN`, `LWI`, `USTAR`, `SH`, `Z0H`, `ZPBL`, `TS`, `H2O2`, `OH`, `NO3`, `O3`, `DMS_CONC_OCEAN`, the SO2 emission fields (`SO2_EMIS_FIRES`, `_NONENERGY`, `_ENERGY`, `_SHIPPING`, `_AIRCRAFT_LTO/CDS/CRS`), `NH3_EMIS`, `NH3_EMIS_FIRE`, `SOAG_EMIS`. If `voc_chem`: `CO_BIOMASS_VOC`, `CO_BF_VOC`, `CO_FS_VOC`, and `OH` when `mam_chem` is off. If `ocs_chem`: `TROPP`, `OHSTRAT`, `O3P`, `OCS_JRATE`. Several imports are marked `RESTART = MAPL_RestartSkip`.
- **Internal state**, `MAPL_AddInternalSpec`, names prefixed with the component name (`ACHEM::`): `DMS`, `MSA`, `SO2`, `H2SO4`, `NH3`, `SOAG` (if `mam_chem`); `OCS` (if `ocs_chem`); `VOC`, `VOCbiob` (if `voc_chem`). All are in `mol mol-1` and friendly to `DYNAMICS:TURBULENCE:MOIST`; `SO2`, `H2SO4`, `NH3`, `SOAG`, `OCS` are also friendly to `MAM` (`ACHEM_GridCompMod.F90`, SetServices, `FRIENDLYTO`). The tracers marked friendly to `DYNAMICS`, `TURBULENCE` and `MOIST` are gathered by the parent (parent document, `CHEM_TRACERS`); *Inferred:* they are then transported, mixed and acted on by those components.
- **Exports:** the `OH`/`NO3` diurnally scaled fields, emission, concentration, column-density, deposition and tendency diagnostics (`ACHEM_Registry.rc`, `ExportSpec`, included in code through the generated `ACHEM_ExportSpec___.h`, guarded by `mam_chem`); with `voc_chem` also `pSOA_ANTHRO_VOC`, `pSOA_BIOB_VOC` and their `_MMRday` forms; with `ocs_chem` the `pSO2_OCS*`, `lOCS*`, `pScl_OCS` fields (SetServices).
- **Restarts:** the internal tracers are checkpointed through MAPL. The parent template lists `ACHEM_INTERNAL_RESTART_FILE` and `ACHEM_IMPORT_RESTART_FILE` (parent document, restarts). The H2O2 buffer and volcano tables are not checkpointed (they live in the private state).
- **Who consumes the exports:** the connections made by the parent (above). *Unverified:* `HISTORY` collections of the other diagnostics (`ACHEM_History___.rc` is generated).
- **External data:** `ACHEM_ExtData.rc` (legacy) and `ACHEM_ExtData.yaml` give sources for emissions (QFED fires, HTAP anthropogenic and aviation, EDGAR NH3, CEDS CO, CAM SOAG), DMS concentration climatology (Lana 2011) and the `OH`, `NO3`, `H2O2` climatology (MERRA2 GMI) (`ACHEM_ExtData.yaml`, `Collections`).

## 6. Configuration that changes structure
Read from `ACHEM_GridComp.rc` in `SetServices` (not through the parent):
- `gas_chemistry:`, `aqueous_chemistry:`: code defaults `.true.`; shipped file sets both `.false.`. Either true makes `mam_chem` true and adds most imports, exports and internal tracers.
- `voc_chemistry:` (code default `.false.`, shipped `.true.`) with `voc_BiomassBurnFactor`, `voc_AnthroFactor`, `voc_MW`, `soa_MW`.
- `ocs_chemistry:` (default and shipped `.false.`), `ocs_surface_vmr:`.
- `apply_diurnal_cycle:` (default `.true.`, not in the shipped file); `aqueous_chemistry_solver_max_dt:`; `aviation_vertical_layers:` (four heights, required); `volcanoes:` (resource file; default `/dev/null`).
- *Variants:* `AMIP/` and `AMIP.20C/` hold an `ACHEM_GridComp.rc` that differs from the top-level file only in `volcanoes:` (a daily template `so2_volcanic_emissions_Carns.%y4%m2%d2.rc` instead of the degassing-only file) and whitespace (`diff` of `AMIP/ACHEM_GridComp.rc` and `AMIP.20C/ACHEM_GridComp.rc` against `ACHEM_GridComp.rc`). Their `ACHEM_ExtData.rc` differ in fire sources: `AMIP` uses QFED v2.6r1, `AMIP.20C` uses HFED v1.0 for SO2, NH3 and CO fire emissions. `AMIP/` also has `ACHEM_ExtData.yaml`; `AMIP.20C/` has none. *Inferred:* the setup scripts copy the variant that fits the experiment; not checked here.
- Parent switches: `ENABLE_ACHEM`; `ENABLE_MAM`, `ENABLE_MATRIX`, `ENABLE_PCHEM`, `ENABLE_STRATCHEM`, `ENABLE_GMICHEM`, `ENABLE_GOCART2G`, `ENABLE_CARMA` select which of the connections above exist.

## 7. Dependencies
`CMakeLists.txt`: `Chem_Shared`, `MAPL`, `GMAO_mpeu` (`m_StrTemplate`), `ESMF::ESMF`, `NetCDF::NetCDF_Fortran`; compile definitions `MAPL_MODE`, `GEOS5` (so `GACL_ConstantsMod` takes constants from the `MAPL_Constants` module on this line; `MAPL_ConstantsMod` on `develop`). ACG generates `ACHEM_ExportSpec___.h`, `ACHEM_GetPointer___.h` and `ACHEM_History___.rc` from `ACHEM_Registry.rc` (`new_esma_generate_automatic_code`). The `GACL_` prefix: *Unverified:* "Global Atmospheric Chemistry Library" or similar; not stated in the code.

## 8. Known traps
- **Shipped configuration is VOC only.** With the shipped `ACHEM_GridComp.rc`, `mam_chem` is false, so the sulfur and ammonia tracers, most imports (including `O3`, `H2O2`, `NO3`) and most exports do not exist. The parent still names some of them in `MAPL_AddConnectivity` with the shipped `GEOS_ChemGridComp.rc` (`ENABLE_PCHEM`, `ENABLE_ACHEM` true): `PCHEM` `O3` to ACHEM `O3`, and `GMICHEM` `OH`, `H2O2`, `NO3` to ACHEM when `ENABLE_GMICHEM` is on (`GEOS_ChemGridComp.F90`, SetServices); ACHEM adds `O3`, `H2O2`, `NO3` only inside the `mam_chem` branch (`ACHEM_GridCompMod.F90`, SetServices). The same holds for `ACHEM::SO2` and the other names sent to `MAM` when `ENABLE_MAM` is true. How MAPL reacts to a connection to a missing spec was not checked.
- Switch values in the code defaults (`.true.`) and in the shipped file (`.false.`) differ; the file wins.
- `ACHEM_Registry.rc` lists imports and internal specs, but imports and internal specs in code come from hand-written `MAPL_Add*Spec` calls; some registry imports (for example `RH2`, `Q`, `QCTOT`, `QL`) have no spec in `SetServices`. Only the export specification is included from generated code (`#include "ACHEM_ExportSpec___.h"`).
- The generated `kpp/gas/*.f90` should be regenerated from the `.kpp`/`.eqn`/`.spc` sources, not edited. *Unverified:* the KPP version and command used; `Makefile_kpp_achem_gas` is present.
- Internal tracer names contain `::` (`ACHEM::SO2`); the parent renames them on connection.

## 9. Decisions and observations
Decisions: [decisions.md](decisions.md).

Observations (noted, not fixed):
- `ACHEM_ExtData.yaml` (top level) names the fire SO2 export `SO2_EMIS_FIRE`; the import in code and `ACHEM_ExtData.rc`, `AMIP/ACHEM_ExtData.yaml` is `SO2_EMIS_FIRES`. Question for maintainers: is the top-level yaml entry matched to the import?
- `NH3_EMIS_OCEAN` is supplied in both ExtData files, but no import of that name appears in `ACHEM_GridCompMod.F90`. Question: is it read by another component?
- `ACHEM_Registry.rc` header calls the component "SulfateChem".
- `aqu_chemistry` is not called; the configuration key `aqueous_chemistry_solver_max_dt` is read, and is passed only in the commented-out call (Run_).
- The `combo_chemistry` keys are in `#if (0)` blocks (SetServices).
- Initialize_ comment says "coupling with GMI is not implemented" while the parent connects GMICHEM `H2O2` to ACHEM; the buffer is therefore refilled from the same import (Initialize_, Run_).
