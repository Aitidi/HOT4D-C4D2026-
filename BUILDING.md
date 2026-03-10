# Building HOT4D for Cinema 4D 2026 and later

This repository is an early adaptation pass of the original HOT4D plugin toward the Cinema 4D 2026 and later SDK.

## Actual SDK-backed build approach

The Cinema 4D 2026 SDK ships a **top-level CMake project** which discovers plugin/modules from either:

- the SDK-local `plugins/` folder (`MAXON_SDK_MODULES_DIR`), or
- a `MAXON_SDK_CUSTOM_PATHS_FILE` text file listing external module roots.

This HOT4D repo already has the expected module layout for that workflow:

- `project/projectdefinition.txt` declares a `DLL` plugin target.
- `APIS=cinema.framework;math.framework;core.framework;misc.framework;` opts into the modern framework-based SDK.
- `source/framework_registration.cpp` includes generated registration glue.
- `source/OceanSimulation/OceanSimulation_decl.h` includes generated Maxon interface declaration headers.

So the intended build is:

1. point the SDK's top-level CMake at this repo via `MAXON_SDK_CUSTOM_PATHS_FILE`,
2. let the SDK generate module-specific build files and generated `*.hxx` glue,
3. build the generated Visual Studio/Xcode/Ninja project from the SDK build tree.

## Helper script

A Windows helper is included to drive that setup:

- `scripts/Configure-HOT4D-C4D2026SDK.ps1`

Example:

```powershell
pwsh -File .\scripts\Configure-HOT4D-C4D2026SDK.ps1 `
  -SdkRoot 'C:\path\to\Cinema_4D_CPP_SDK_2026_1_0' `
  -Build
```

What it does:

- locates `cmake.exe` (PATH, standalone install, or Visual Studio bundled CMake),
- validates that the SDK root looks complete,
- writes `custom_paths_hot4d.txt` into the SDK root with this repo as an external module,
- runs the SDK top-level CMake configure,
- optionally runs the build.

## What was verified in the first real SDK-backed pass

A real configure attempt was run against the provided SDK root using Visual Studio 2026 CMake:

```powershell
cmake -S <SDK_ROOT> -B <SDK_ROOT>\_build_hot4d_vs18 \
  -G "Visual Studio 18 2026" -A x64 \
  -D MAXON_SDK_CUSTOM_PATHS_FILE=<SDK_ROOT>\custom_paths_hot4d.txt
```

That configure **successfully reached SDK module discovery** and recognized this repo as an external module:

- `Reading path definitions: ...custom_paths_hot4d.txt`
- `Read module paths: .../repo`
- `Updating CMake file for 'repo'`

This confirms the integration route is correct.

## Current hard blocker from the provided SDK payload

The configure then failed before code compilation because the provided SDK root is incomplete:

- expected path missing: `frameworks/cinema.framework/project/projectdefinition.txt`
- in fact, the provided SDK root currently has `CMakeLists.txt`, `cmake/`, and `docs/`, but **no `frameworks/` tree at all**.

Error excerpt:

```text
Project Definition File does not exist at location
'C:/.../Cinema_4D_CPP_SDK_2026_1_0/frameworks/cinema.framework/project/projectdefinition.txt'
```

Because the SDK frameworks are absent, CMake cannot finish loading the SDK itself, so it cannot yet:

- generate HOT4D's missing registration/interface `*.hxx` files,
- compile plugin sources,
- surface the first real C++ API migration errors.

## Expected SDK prerequisites

A full build requires all of the following:

1. A **complete Cinema 4D 2026 and later C++ SDK** extraction.
2. The SDK `frameworks/` tree and its framework `projectdefinition.txt` files.
3. The SDK top-level CMake tooling.
4. A supported compiler toolchain (Visual Studio 2026/2022 on Windows).

## Generated files still expected once the SDK is complete

When configured from a complete SDK, this module is expected to generate and/or consume helper headers such as:

- `source/registration_com_valkaari_hot4D.hxx`
- `source/OceanSimulation/OceanSimulation_decl1.hxx`
- `source/OceanSimulation/OceanSimulation_decl2.hxx`

Those are not meant to be hand-authored.

## Recommended bring-up workflow

1. Verify the SDK extraction contains `frameworks/cinema.framework/` and sibling frameworks.
2. Run `scripts/Configure-HOT4D-C4D2026SDK.ps1`.
3. Build from the generated SDK build tree.
4. Capture the first compiler/linker error set.
5. Fix only the errors justified by compiler output or clearly documented SDK conventions.

## Additional migration notes

For a module-by-module source-tree map and upgrade-risk assessment, see:

- `docs/MIGRATION-NOTES-C4D2026Plus.md`

## Scope of this pass

This pass focused on:

- verifying the real SDK integration path,
- proving that external-module discovery works through `MAXON_SDK_CUSTOM_PATHS_FILE`,
- identifying the first concrete SDK-side blocker,
- adding a repeatable configure helper and updated build notes.
