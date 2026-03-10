# C4D 2026+ port TODO

## Current verified state

- [x] Confirm the real SDK integration route: SDK top-level CMake + `MAXON_SDK_CUSTOM_PATHS_FILE`.
- [x] Verify that this repo is discovered as an external SDK module.
- [x] Add a repeatable Windows helper script: `scripts/Configure-HOT4D-C4D2026SDK.ps1`.
- [x] Capture the first real SDK-side blocker from configure.

## Hard blockers

- [ ] Obtain a **complete** Cinema 4D 2026+ C++ SDK extraction containing `frameworks/`.
- [ ] Re-run configure after the SDK payload is complete.
- [ ] Generate missing framework/interface headers:
  - [ ] `source/registration_com_valkaari_hot4D.hxx`
  - [ ] `source/OceanSimulation/OceanSimulation_decl1.hxx`
  - [ ] `source/OceanSimulation/OceanSimulation_decl2.hxx`
- [ ] Confirm whether additional generated implementation glue is required by the SDK tooling.

## First compile pass

- [ ] Run the first full compile from the SDK environment.
- [ ] Capture the first compiler/linker error set.
- [ ] Verify whether `C4D_Falloff` is still supported as used here, or must be migrated to a newer fields/falloff API.
- [ ] Verify MoGraph effector API compatibility for `EffectorData`, `EffectorStrengths`, and registration flags.
- [ ] Verify `VertexColorTag` read/write API signatures against the 2026 SDK.

## Portability / cleanup

- [x] Normalize a few source include paths to be more robust on case-sensitive filesystems.
- [ ] Decide whether `source/C4D_Object/OceanSImulationDesc.cpp` should be renamed for casing consistency.
- [x] Add a source-tree migration/risk note (`docs/MIGRATION-NOTES-C4D2026Plus.md`).
- [x] Add SDK-specific build instructions reflecting the actual 2026 workflow and blocker.

## Validation

- [ ] Test plugin registration in Cinema 4D 2026 and later.
- [ ] Verify object/deformer UI loads correctly.
- [ ] Verify effector registration and MoGraph interaction.
- [ ] Verify ocean simulation output, jacobian map writing, foam behavior, and field/falloff interaction.
