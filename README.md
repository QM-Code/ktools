# ktools

`ktools/` is a Git superproject that pins a known-good set of sibling repository commits. The root `ktools/` directory now has its own thin `kbuild.py` wrapper for git and batch orchestration, but normal source builds still happen inside the individual repos:

- `kbuild/`
- `kcli/`
- `ktrace/`
- `kconfig/`
- `ki18n/`

If you only need the `kbuild` stack, start here before opening source files.

## Superproject Usage

- Clone the full pinned workspace with:

```bash
git clone --recurse-submodules git@github.com:QM-Code/ktools.git
```

- If you already cloned `ktools/` without submodules:

```bash
git submodule update --init --recursive
```

- Each child repo stays independent. You can still `cd kbuild` or `cd kcli` and commit or push there normally.
- The root wrapper can also fan a command out across the pinned child repos in dependency order:

```bash
./kbuild.py --batch --build dev
./kbuild.py --batch --build-latest
./kbuild.py --batch --clean-all
./kbuild.py --batch --git-sync "Sync all child repos"
```

- With no inline repo list, `--batch` uses `kbuild.json -> batch.repos`.
- To move `ktools` to a newer child-repo commit:

```bash
cd kbuild
git pull
# or make changes, commit, and push
cd ..
git add kbuild
git commit -m "Update kbuild"
git push
```

- The superproject records exact child commits, so other machines get the same known working set.

## Repo Map

- `kbuild/`
  Shared build system used by the other repos. It provides the thin-wrapper bootstrap model, strict `kbuild.json` validation, repo scaffolding, local `vcpkg` support, and demo orchestration.
- `kcli/`
  Base SDK. CLI parsing library with end-user mode, inline `--<root>` mode, aliases, root value handlers, and demo SDKs/executables that exercise those patterns.
- `ktrace/`
  Trace logging SDK layered on top of `kcli`. It consumes `KcliSDK`, exposes its own SDK package, and uses `kcli` to parse `--trace*` inline options.
- `kconfig/`
  JSON config/storage SDK layered on top of both `kcli` and `ktrace`. It consumes `KcliSDK` and `KTraceSDK`, uses `kcli` for `--config*` inline options, and uses `ktrace` for diagnostics and demo visibility.
- `ki18n/`
  JSON internationalization SDK extracted from `kconfig`. It consumes `KConfigSDK` for config-backed asset loading and keeps direct `KcliSDK` and `KTraceSDK` imports in place for planned inline CLI and trace integration.

## Two-Minute Mental Model

- The root `ktools/` repo and every child repo (`kbuild`, `kcli`, `ktrace`, `kconfig`, `ki18n`) have thin `kbuild.py` wrappers.
- That wrapper does not contain the real build logic. It loads the shared implementation from `kbuild/libs/kbuild/` using `./.kbuild.json -> kbuild.root`.
- In a normal `ktools` checkout, the wrapper auto-detects `./kbuild`, `../kbuild`, or `.` as the shared root and writes a local `./.kbuild.json` on first run.
- Manual bootstrap is still available if needed:

```bash
# from /home/karmak/dev/ktools
./kbuild.py --kbuild-root ./kbuild

# from a child repo such as /home/karmak/dev/ktools/kcli
./kbuild.py --kbuild-root ../kbuild
```

- Running `./kbuild.py` with no arguments prints usage. It does not build.
- Normal build commands are:

```bash
./kbuild.py --build-latest
./kbuild.py --build <slot>
./kbuild.py --build-demos [demo ...]
```

- If a repo has a `vcpkg` section in `kbuild.json`, first-time setup is usually:

```bash
./kbuild.py --vcpkg-install
```

- Core SDK output goes to `build/<slot>/sdk/`.
- Demo build output goes to `demo/<demo>/build/<slot>/`.
- Demo build order matters. Earlier demo SDKs can be installed and then consumed by later demos in the same run.
- Unknown CLI options, unknown `kbuild.json` keys, bad dependency prefixes, and invalid build slot names are hard errors by design.

## kbuild

`kbuild/` is the shared orchestrator. The thin wrapper in each consumer repo only validates `./.kbuild.json`, resolves the shared `kbuild` checkout, adds `kbuild/libs` to `sys.path`, and then calls the shared entrypoint.

### What kbuild validates

- `project` and `git` are required in `kbuild.json`.
- `cmake`, `vcpkg`, and `build` are optional.
- Unknown keys fail validation.
- `project.id` must be a valid C/C++ identifier.
- `cmake.sdk.package_name` is required if the repo wants SDK packaging and demo builds.
- `cmake.dependencies.<Package>.prefix` must resolve to a built SDK prefix containing:
  - `include/`
  - `lib/`
  - `lib/cmake/<Package>/<Package>Config.cmake`

### Main commands

- `./kbuild.py --help`
  Print usage. `./kbuild.py` with no args does the same thing.
- `./kbuild.py --kbuild-root <path>`
  Point the thin wrapper at the shared `kbuild/` checkout and write `./.kbuild.json`.
- `./kbuild.py --kbuild-config`
  Create a starter `kbuild.json`.
- `./kbuild.py --kbuild-init`
  Scaffold a new repo from templates.
- `./kbuild.py --build-latest`
  Build the `latest` slot.
- `./kbuild.py --build <slot>`
  Build a named slot such as `dev`, `ci`, or `0.1`.
- `./kbuild.py --build-demos [demo ...]`
  Build demos explicitly. Without names, use `build.demos`.
- `./kbuild.py --clean <slot>`
- `./kbuild.py --clean-latest`
- `./kbuild.py --clean-all`
- `./kbuild.py --vcpkg-install`
  Install/bootstrap repo-local `vcpkg`, sync baseline, then continue the normal build flow.
- `./kbuild.py --vcpkg-sync-baseline`
  Write the current `vcpkg/src` HEAD into `vcpkg/vcpkg.json`.

### Output layout

- Core build tree: `build/<slot>/`
- Core SDK install prefix: `build/<slot>/sdk/`
- Demo build tree: `demo/<demo>/build/<slot>/`
- Demo SDK install prefix: `demo/<demo>/build/<slot>/sdk/` when the demo installs targets
- Repo-local `vcpkg` checkout: `vcpkg/src/`
- Repo-local `vcpkg` caches: `vcpkg/build/`
- `vcpkg` installed tree for the core build: `build/<slot>/installed/<triplet>/`

### Demo orchestration

When `kbuild` configures a demo, it builds `CMAKE_PREFIX_PATH` in this order:

1. The core SDK prefix for the active slot.
2. The core build's `vcpkg` triplet prefix, if the repo uses `vcpkg`.
3. Resolved sibling SDK dependencies from `cmake.dependencies`.
4. Any already-built demo SDK prefixes from earlier demos in the same ordered run.

That is why demo order is part of the real dependency model rather than just presentation.

### Scaffold/templates

`kbuild --kbuild-init` generates the base repo layout:

- root `CMakeLists.txt`
- root `README.md`
- `agent/BOOTSTRAP.md`
- `src/`
- `include/` for SDK repos
- `cmake/00_toolchain.cmake`
- `cmake/10_dependencies.cmake`
- `cmake/20_targets.cmake`
- `cmake/50_install_export.cmake` for SDK repos
- demo scaffold:
  - `demo/bootstrap/`
  - `demo/sdk/alpha`
  - `demo/sdk/beta`
  - `demo/sdk/gamma`
  - `demo/exe/core`
  - `demo/exe/omega`

The current repos started from that scaffold, but they are no longer template-pure. Their `CMakeLists.txt`, demos, tests, and READMEs are now real implementations, not just generated placeholders.

## kcli

`kcli/` is the base SDK in this stack.

### kbuild shape

- `cmake.sdk.package_name = KcliSDK`
- no `cmake.dependencies`
- `vcpkg.dependencies = ["spdlog"]`
- default demo order:
  - `bootstrap`
  - `sdk/alpha`
  - `sdk/beta`
  - `sdk/gamma`
  - `exe/core`
  - `exe/omega`

### Public use case

`kcli::Parser` supports:

- end-user mode: top-level `--option` plus aliases like `-v`
- inline mode: `--<root>` and `--<root>-*`
- optional root value handlers
- command descriptions and inline root help output
- command removal and alias management
- argv compaction after consumed options
- explicit error/warning/unknown-option hooks

### Real demos

- `demo/sdk/alpha`
  Inline parser with required and optional values.
- `demo/sdk/beta`
  Inline parser with numeric and profile options.
- `demo/sdk/gamma`
  Inline parser with a strict flag and tag value.
- `demo/exe/core`
  Consumes `KcliSDK` plus `AlphaSDK`.
- `demo/exe/omega`
  Shows multiple imported inline roots, an app-defined inline root (`--build-*`), end-user options, aliases, and a renamed root (`gamma` exposed as `--renamed`).

### Why it matters to the rest of the stack

- `ktrace` uses `kcli` to implement `--trace*`.
- `kconfig` uses `kcli` to implement `--config*`.
- The demo code in `kcli/` is the clearest concrete reference for how inline roots are intended to be composed.

## ktrace

`ktrace/` is the first sibling-SDK consumer case.

### kbuild shape

- `cmake.sdk.package_name = KTraceSDK`
- `cmake.dependencies.KcliSDK.prefix = ../kcli/build/{version}/sdk`
- `vcpkg.dependencies = ["spdlog"]`
- same ordered demo set as `kcli`

### Public use case

`ktrace` is a trace/logging SDK with:

- compile-time namespace requirement via `KTRACE_NAMESPACE`
- registered channels and color IDs
- `KTRACE(...)` and `KTRACE_CHANGED(...)`
- selector-based enable/disable APIs
- operational log APIs independent of trace-channel enablement
- output formatting options for timestamps, files, and functions

### Real dependency relationship

- `ktrace` links against `KcliSDK`.
- `KTraceSDKConfig.cmake.in` re-exports that dependency with `find_dependency(KcliSDK CONFIG REQUIRED)`.
- The build target files in `ktrace/cmake/20_targets.cmake` are a concrete example of how a scaffolded SDK repo evolves once it grows real internal sources and public dependencies.

### Real CLI embedding pattern

`ktrace/src/ktrace/cli.cpp` uses `kcli::Parser` in inline mode to implement:

- `--trace <selector>`
- `--trace-examples`
- `--trace-namespaces`
- `--trace-channels`
- `--trace-colors`
- `--trace-files`
- `--trace-functions`
- `--trace-timestamps`

That file is the reference implementation for "use `kcli` to bolt a subsystem-specific inline CLI onto an SDK".

### Real demos

- demo SDKs (`alpha`, `beta`, `gamma`) register trace channels in their own namespaces
- demo executables (`core`, `omega`) register local channels, initialize imported demo SDKs, parse `--trace*`, and then emit both local and imported traces

These demos are why `kbuild` demo order matters: the demo SDKs must exist before the demo executables link them.

### Extra note

`ktrace/agent/projects/rpath-packaging.md` documents a current packaging caveat: build-time runtime library directories are also being copied into `INSTALL_RPATH`, which makes the produced SDK trees less relocatable. That is a real follow-up item, not just a theoretical concern.

## kconfig

`kconfig/` is the config/storage layer in this workspace.

### kbuild shape

- `cmake.sdk.package_name = KConfigSDK`
- `cmake.dependencies.KcliSDK.prefix = ../kcli/build/{version}/sdk`
- `cmake.dependencies.KTraceSDK.prefix = ../ktrace/build/{version}/sdk`
- `vcpkg.dependencies = ["spdlog", "nlohmann-json"]`
- same ordered demo set as the other repos

### Public use case

`kconfig` provides:

- config namespace/store registration and merge APIs
- mutable vs read-only stores
- file-backed config loading and persistence helpers
- asset loading
- CLI assignment parsing layered on `kcli`
- trace instrumentation layered on `ktrace`

### Real dependency relationship

- `kconfig` links against both `KcliSDK` and `KTraceSDK`.
- `KConfigSDKConfig.cmake.in` re-exports both dependencies.
- `kconfig/cmake/20_targets.cmake` is the strongest example of a multi-SDK core target in this stack.

### Real CLI embedding pattern

`kconfig/src/kconfig/cli.cpp` uses `kcli::Parser` in inline mode for `--config*`:

- root value handler:
  - `--config '"key"=value'`
- extra commands:
  - `--config-examples`
  - `--config-user <path>`

It parses quoted assignment keys, JSON-like values, fallback string values, and then writes into a mutable config namespace.

### Real demos

- `demo/exe/core`
  Loads runtime JSON files from `demo/exe/core/runtime/`, merges `defaults`, `user`, and `session`, optionally appends CLI overrides, then reads strongly typed config values.
- `demo/exe/omega`
  Adds asset loading, user-config API paths, optional backing-file roundtrip checks, and richer typed reads.
- `demo/sdk/{alpha,beta,gamma}`
  Very small SDK add-ons that ensure the demo layering is real and linkable.

### Important local-source fallback pattern

`kconfig/demo/exe/core/CMakeLists.txt` and `kconfig/demo/exe/omega/CMakeLists.txt` define `ktools_load_demo_sdk(...)`:

- try `find_package(<DemoSDK>)`
- if the package is not installed, fall back to `add_subdirectory(...)` of the local sibling demo SDK source

This is the strongest example in the workspace of a demo that intentionally supports both installed-package and local-source consumption paths.

### Runtime fixtures

The `kconfig` demos depend on checked-in runtime data:

- `defaults.json`
- `user.json`
- `session.json`
- `assets/banner.txt`

These files are part of the actual use case, not just test noise. If a change touches `kconfig` demo behavior, inspect the runtime fixtures as part of the task.

## ki18n

`ki18n/` owns the extracted i18n layer from this stack.

### kbuild shape

- `cmake.sdk.package_name = Ki18nSDK`
- `cmake.dependencies.KConfigSDK.prefix = ../kconfig/build/{version}/sdk`
- `cmake.dependencies.KcliSDK.prefix = ../kcli/build/{version}/sdk`
- `cmake.dependencies.KTraceSDK.prefix = ../ktrace/build/{version}/sdk`
- no `vcpkg` section
- same ordered demo set as the other repos

### Public use case

`ki18n` provides:

- config-backed language selection and fallback loading
- flattened key lookup over nested string JSON
- missing-key fallback vs required-key APIs
- token replacement helpers for translated strings

### Real dependency relationship

- `ki18n` links against `KConfigSDK` and uses `kconfig::store` plus `kconfig::asset` for config-backed string loading.
- It keeps direct `KcliSDK` and `KTraceSDK` imports available for the planned inline-root and trace logging work.

### Real demos

- `demo/exe/core`
  Shows the basic config-backed language load path.
- `demo/exe/omega`
  Carries the fuller config, i18n, asset, and backing-file flow that used to live in `kconfig`.

## Cross-Repo Build Order

If you need a working full stack in one slot, build in dependency order and use the same slot name in every repo.

Example with `dev`:

```bash
cd /home/karmak/dev/ktools/kcli
./kbuild.py --build dev --vcpkg-install

cd /home/karmak/dev/ktools/ktrace
./kbuild.py --build dev --vcpkg-install

cd /home/karmak/dev/ktools/kconfig
./kbuild.py --build dev --vcpkg-install

cd /home/karmak/dev/ktools/ki18n
./kbuild.py --build dev
```

Why the shared slot matters:

- `ktrace` resolves `../kcli/build/{version}/sdk`
- `kconfig` resolves both:
  - `../kcli/build/{version}/sdk`
  - `../ktrace/build/{version}/sdk`
- `ki18n` resolves:
  - `../kcli/build/{version}/sdk`
  - `../ktrace/build/{version}/sdk`
  - `../kconfig/build/{version}/sdk`

If the slot names do not match, the dependency prefixes do not resolve.

## Testing Conventions

- `kcli`
  Demo shell cases live under `demo/exe/*/cmake/tests/demo_cli_cases.sh`.
- `ktrace`
  Demo shell cases live under `demo/exe/*/cmake/tests/trace_cli_cases.sh`.
  It also has core CTest coverage under `ktrace/cmake/tests/`.
- `kconfig`
  Demo shell cases live under `demo/exe/*/cmake/tests/demo_config_cases.sh`.
  These tests depend on the checked-in runtime fixtures in the demo runtime directories.
- `ki18n`
  Demo shell cases live under `demo/exe/*/cmake/tests/demo_i18n_cases.sh`.
  These tests depend on the checked-in config and string fixtures in the demo runtime directories.

Across all four repos, the demos are the main end-to-end validation surface.

## Known Gotchas

- `./kbuild.py` with no args prints help. It does not build.
- `--rebuild` does not exist. Use:
  - `./kbuild.py --clean <slot>`
  - `./kbuild.py --clean-latest`
  - `./kbuild.py --clean-all`
- Always run `kbuild.py` from the repo root it lives in.
- Do not build `ktrace` before `kcli` in the same slot.
- Do not build `kconfig` before both `kcli` and `ktrace` in the same slot.
- Do not build `ki18n` before `kconfig` in the same slot.
- For demo failures, check:
  - missing sibling SDK package configs under `build/<slot>/sdk/lib/cmake/...`
  - missing `vcpkg` setup
  - wrong slot name
  - demo order
- The current repos are not pristine template outputs. Read the actual repo CMake and demo sources before assuming scaffold defaults.

## Where to Look First

If you need the fastest path to operational context:

- `kbuild/docs/kbuild.md`
- `kcli/kbuild.json`
- `ktrace/kbuild.json`
- `kconfig/kbuild.json`
- `ki18n/kbuild.json`
- `kcli/demo/exe/omega/src/main.cpp`
- `ktrace/src/ktrace/cli.cpp`
- `kconfig/src/kconfig/cli.cpp`
- `ki18n/src/ki18n/i18n.cpp`
- `ki18n/demo/exe/omega/src/main.cpp`

That set gives you the real build model, dependency layering, and the highest-signal cross-repo use cases without reopening the entire tree.
