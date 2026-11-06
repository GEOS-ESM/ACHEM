# AGENTS.md: ACHEM

ACHEM is a GEOS Chemistry child gridded component (MAPL2 code): sulfur and ammonia
gas and aqueous chemistry for modal aerosol, VOC to secondary organic
aerosol, OCS loss. Branch `release/MAPL-v3`, verified against `30e63d0` (2026-11-06).
Fixture: `components.yaml` @ `f50622d`, key `ACHEM`.

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

## Read first
- `docs/ARCHITECTURE.md`: map, entry points, state, switches, traps, observations.
- `docs/decisions.md`: patterns seen in the code.
- Parent: `../docs/ARCHITECTURE.md` (Chemistry; how children are added, run phases, connections). `../AGENTS.md` (on this line the parent's documents are not written yet; the MAPL2-line versions are on its `develop` branch).

## Where things are
- `ACHEM_GridCompMod.F90`: whole component; public `SetServices` only. Routines: `Initialize_`, `Run_` (one phase), `Finalize_`, `gas_chemistry`, `aqu_chemistry_fast`, `ocs_chemistry`.
- `GACL_*Mod.F90`: constants, emissions, reaction rates, dry deposition helpers.
- `kpp/gas/`: generated KPP solver. Edit the `.kpp`, `.eqn`, `.spc` sources, not the `.f90`. `Main`, `Model`, `Initialize`, `mex_*` are not built.
- `ACHEM_GridComp.rc`: private switches (`gas_chemistry`, `aqueous_chemistry`, `voc_chemistry`, `ocs_chemistry`).
- `ACHEM_Registry.rc`: ACG input; only the export specs and pointer code are generated.
- `ACHEM_ExtData.rc`/`.yaml`; `AMIP/`, `AMIP.20C/` variants (different volcano file, fire emission sources).

## Rules and traps
- Imports, internal tracers and most exports exist only for enabled switches. The shipped file has gas and aqueous chemistry off, so only the VOC part exists; the parent still connects `PCHEM` `O3` (and `GMICHEM` `H2O2`, `NO3`) to ACHEM imports that exist only with `mam_chem`, and `ACHEM::SO2` etc. to MAM when `ENABLE_MAM` is on.
- Tracers are internal specs named `ACHEM::<name>`, friendly to Dynamics, Turbulence, Moist (and MAM for some). The parent renames them on connection.
- Run_ exits unless ACHEM's own run alarm rings (`ACHEM_DT`), and turns the alarm off when it does (`ESMF_AlarmRingerOff`).
- `aqu_chemistry` is compiled but not called; `aqu_chemistry_fast` is.
- Do not edit generated files or `CHANGELOG.md` by hand for documentation work.
- Every statement in `docs/` carries a citation or an *Inferred:* / *Unverified:* label; keep that when editing.

## Build and test
- Not built on this line (see Build status). The library `ACHEM` is defined in this repository's `CMakeLists.txt`; no tests in this repository.
