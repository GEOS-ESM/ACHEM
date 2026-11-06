# AGENTS.md: ACHEM

ACHEM is a GEOS Chemistry child gridded component (MAPL2): sulfur and ammonia
gas and aqueous chemistry for modal aerosol, VOC to secondary organic
aerosol, OCS loss. Branch `develop`, verified against `a10daeb` (2026-11-06).
Fixture: `components.yaml` @ `0f2e0fb`, key `ACHEM`. Documentation only; no
`docs/` files existed before.

## Read first
- `docs/ARCHITECTURE.md`: map, entry points, state, switches, traps, observations.
- `docs/decisions.md`: patterns seen in the code.
- Parent: `../docs/ARCHITECTURE.md` (Chemistry; how children are added, run phases, connections). `../AGENTS.md`.

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
- Built as library `ACHEM` by the parent's CMake; no tests in this repository.
