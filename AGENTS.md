# Repository Guidelines

## Project Structure

This is a C/C++ and ARM assembly decompilation of Pokémon Platinum. Game logic
and engine code live in `src/`; public headers are in `include/`; assembly is
in `asm/`; graphics, data, and other source assets are in `res/`. Generated
source inputs are kept in `generated/`. Build output belongs in `build/` and
should not be committed. Repository tools and vendored build helpers are under
`tools/` and `subprojects/`.

## Build, Test, and Development Commands

- `make build` creates the local build directory.
- `make` builds the default Rev 1 ROM and runs the checks configured by the
  repository.
- `make check` builds `build/pokeplatinum.us.nds` and runs the Meson test suite.
- `make format` runs the repository's `clang-format` target.
- `make clean` removes generated build products; `make purge` removes fetched
  build dependencies and the build directory.

Follow `INSTALL.md` for platform packages and required ROM inputs. A 32-bit
runtime is required on Linux for the bundled Metrowerks tools.

## Coding Style and Naming

Use the repository's `.editorconfig` and `.clang-format`; run `make format`
before submitting C/C++ changes. Use four-space indentation in C/C++ where the
formatter does not decide otherwise. Structures, enums, and functions use
`PascalCase`; variables and parameters use `camelCase`; enum members use
`UPPER_SNAKE_CASE`. Public module functions generally use a `Module_` prefix.
Match surrounding conventions in assembly, scripts, and data files.

## Testing Guidelines

Run `make check` after source or asset changes. It verifies the build and runs
Meson tests. Install the configured `pre-commit` hooks when possible so
formatting checks run automatically. Before a pull request, confirm the build
produces the expected ROM hash documented in `README.md`.

## Commits and Pull Requests

Write short, imperative commit subjects; scoped prefixes such as `tools:` are
common, and issue or pull-request references may be included (for example,
`Document overlay76 behavior (#1259)`). Keep each commit focused. Pull
requests should explain the change, identify relevant testing (`make check` and
formatting), and link the related issue or discussion when one exists.
