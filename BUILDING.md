# Building HOT4D for Cinema 4D 2026 and later

This repository is an early adaptation pass of the original HOT4D plugin toward the Cinema 4D 2026 and later SDK.

## Actual SDK-backed build approach

The Cinema 4D 2026 SDK ships a **top-level CMake project** which discovers plugin/modules from either:

- the SDK-local `plugins/` folder (`MAXON_SDK_MODULES_DIR`), or
- a `MAXON_SDK_CUSTOM_PATHS_FILE` text file using SDK custom-path commands such as `MODULE <path>` for external module roots.

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

1. A **complete extended Cinema 4D 2026 and later C++ SDK** extraction from the Maxon developer downloads.
2. The SDK `frameworks/` tree and its framework `projectdefinition.txt` files.
3. The SDK top-level CMake tooling.
4. A supported compiler toolchain (Visual Studio 2026/2022 on Windows).

The SDK documentation explicitly says the extended SDK ships with the necessary SDK components, while the `sdk.zip` shipped with Cinema 4D is insufficient. The payload inspected for this bring-up does not match that expectation because it has `CMakeLists.txt`, `cmake/`, and `docs/`, but no `frameworks/` directory at all.

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

## Lessons learned during the C4D 2026 port

### 1. `Ocean().Create()` failure was a Maxon registration bootstrap problem, not an ocean algorithm problem

The most important debugging result of this port is that a failure like:

```text
OceanSimulation::Ocean().Create() -> NULL_RETURN_REASON::NULLPTR
```

can happen even when the implementation class itself is valid.

What we observed:

- `OceanImplementation::GetClass()` worked.
- `RunOceanImplementationSelfTest()` succeeded.
- `System::FindDefinition(CLASS, ...)` and `System::FindDefinition(COMPONENT, ...)` were initially missing.
- The real ocean simulation code was fine; the public Maxon registration path was not.

Meaning:

- local/private class registration may still work,
- but `MAXON_DECLARATION(...).Create()` can fail if the generated registration bootstrap is wired incorrectly.

### 2. The generated `register.cpp` conflicted with HOT4D's explicit bootstrap path

The critical fix was not inside the ocean algorithm.
It was in how the target consumed generated Maxon registration units.

What finally worked:

- keep `source/framework_registration.cpp`,
- make it include `register.hxx` through the non-framework path,
- keep generated `interface_registration.cpp`,
- **do not also compile generated `register.cpp` for this target**.

For this project, compiling both paths at once left `Ocean().Create()` broken.
After switching to the corrected bootstrap arrangement:

- `Ocean().Create()` succeeded,
- `RunOceanImplementationSelfTest()` succeeded,
- the plugin could create the ocean object through the declaration path again.

### 3. Make registration fixes reproducible at the project level, not by hand-editing the build tree

A temporary `.vcxproj` edit can prove a theory, but it is not a finished fix.
The durable solution was to add a project-local `project/CMakeLists.txt` which overrides the auto-generated target settings for HOT4D.

That file now captures the important rule for this plugin:

- compile `framework_registration.cpp`,
- compile generated `interface_registration.cpp`,
- skip generated `register.cpp`.

This makes reconfigure/rebuild stable and prevents the fix from being lost the next time CMake regenerates the project.

### 4. Deformer playback in C4D 2026 required explicit execution-chain registration

A major runtime symptom was:

- the deformer only updated when parameters changed,
- playback did not continuously animate the ocean.

The effective fix was to register the deformer into the execution pipeline using:

- `OBJECT_CALL_ADDEXECUTION`,
- `AddToExecution()`,
- `Execute()`

and to schedule evaluation in the proper animation-related execution priority.

Without that integration, the ocean can appear "stuck" even when the simulation code is otherwise correct.

### 5. Do not write `BaseSelect`/`SelectionTag` state from the parallel deformation loop

A C4D 2026 stability issue showed up in the Point Selection/Jacobian path.
The unsafe pattern was:

- computing point-selection state in `ParallelFor`, and
- directly mutating `SelectionTag` / `BaseSelect` from that parallel region.

That can crash.
The safe pattern is:

1. compute per-point selection flags in parallel,
2. write the selection back to the tag on a single thread afterward.

### 6. Foam/Jacobian tag creation must target the deformed point object, not the deformer itself

When HOT4D needs to auto-create helper tags such as `Jminus` and `Foam`, the creation target matters.
The robust logic for the current object hierarchy is:

- prefer the parent object (`GetUp()`), because HOT4D usually sits as a child deformer under the point object,
- only fall back to `GetDown()` if the parent is not a usable point object.

Creating the tags on the deformer itself or on the wrong hierarchy node leads to failures or misleading UI state.

### 7. Temporary diagnostic probes should be removed once the real issue is understood

A temporary `MiniProbe` declaration/class probe was useful while isolating the registration problem, but it was not part of HOT4D's real feature set.
Once the Maxon registration issue was understood and fixed, the probe was removed.

Takeaway:

- use minimal probes aggressively during diagnosis,
- but delete them once the real system is working again.

### 8. Current known state after the fix

At the end of this pass, the important practical state is:

- HOT4D builds successfully against the current configured SDK environment,
- `Ocean().Create()` succeeds,
- the self-test succeeds,
- the registration bootstrap fix is reproducible,
- the deformer updates correctly during playback,
- Jacobian/foam and point-selection crash fixes are in place.

There may still be cosmetic warnings and deeper registration-theory questions worth revisiting later, but the main migration blockers described above have been resolved.
