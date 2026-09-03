# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A work-in-progress **decompilation** of Pokémon Platinum (US, NDS) — matching C source that
recompiles byte-identical to the retail ROM. It is not a from-scratch reimplementation: correctness
is measured by SHA1-matching the original binary, not by conventional unit tests. Most of the
codebase is un-obfuscated C already; the primary ongoing work is *documenting* undocumented
functions/structs (renaming address-based symbols to meaningful names) and improving tooling, not
writing new game logic.

## Build commands

Builds require the Metroskrew (MWCC) toolchain, `gcc-arm-none-eabi`, Meson/Ninja, and other deps —
see `INSTALL.md` for full per-OS setup. Once dependencies are installed:

```bash
make            # full build: sets up release config, builds ROM, runs `make check`
make rom        # build build/pokeplatinum.us.nds without the checksum-verifying test suite
make check      # build + run meson tests (SHA1 checksum matches against the retail ROM)
make debug      # build a GDB-debuggable ROM (build/debug.nef, overlay.map)
make format     # run clang-format via the ninja `clang-format` target
make configure  # just run meson setup/configure, no build
make update     # update subprojects (needed after pulling changes that bump SDK/tool versions)
make clean      # ninja clean + remove build/res
make distclean  # remove the whole build/ directory
```

- Build revision defaults to Rev 1 (the patched retail cartridge). Build Rev 0 instead with
  `ROM_REVISION=0 make`.
- `make check` is the CI gate and the standard way to verify a change didn't break the match —
  it re-links and diffs checksums against `platinum.us/*.sha1`. Any change to C/assembly/data must
  still produce a matching ROM.
- There is no traditional test framework/single-test runner — "tests" are meson's SHA1 checksum
  comparisons (`SBIN Checksums`, `Filesystem Checksums`, `ROM Checksum`) run via `meson test -C build`.
- Formatting is enforced in CI via `cpp-linter` against `.clang-format` (only `**/*.c`, `**/*.h`
  under `include/`, `src/`, `lib/`; `tools/`, `subprojects/`, `.s`/`.inc` files are excluded). Use
  `pre-commit install` (see `CONTRIBUTING.md`) to format automatically on commit.

## High-level architecture

**Build system:** Meson (`meson.build` at root) driven through a thin `Makefile` wrapper. Meson
pulls in the proprietary/vendored Nintendo SDK pieces (`NitroSDK`, `NitroSystem`, `NitroWiFi`,
`NitroDWC`) and internal libs (`lib/crypto`, `lib/gds`, `lib/spl`, `lib/ppwlobby`) as dependencies/
subprojects, compiles the ARM9 `main` executable with the Metroskrew MWCC compiler, then packs it
plus the ARM7 binary and NitroFS filesystem (`res/`) into `pokeplatinum.us.nds` via the in-tree
`nitrorom` tool.

**ARM9 binary layout — main + overlays:** `src/` (and mirrored `include/`) is split between
top-level modules (always resident: `field/`, `battle/`, `applications/`, etc.) and `overlayNNN/`
directories, which correspond to the DS overlay system (code paged in/out of a fixed memory region
at runtime, e.g. per-battle-facility, per-minigame). When investigating a function, check whether it
belongs in an overlay module (naming convention `ovNN_...`) vs. the main resident binary — this
affects where declarations/definitions and includes live (`include/overlayNNN/` mirrors
`src/overlayNNN/`).

**Undocumented-symbol convention:** files/functions not yet reverse-engineered keep their original
address as the name, e.g. `src/overlay006/ov6_0223E140.c` (file named by its load address) containing
functions like `ov6_0223E140`. Documenting a function means giving the file and function a real name
and moving it out of the address-named file — this is the majority of ongoing contribution work (see
`CONTRIBUTING.md`'s "living issue" of small documentation targets). `include/struct_decls/` holds
forward declarations, `include/struct_defs/` holds full struct definitions — this split exists
because many structs are still only partially reverse-engineered.

**Generated headers from plain text (`generated/`):** many enums/constants are defined once as plain
`.txt` files in `generated/` (e.g. `generated/days_of_week.txt`) and expanded at build time into C
headers under `build/generated/` that are valid *both* as a C `enum` (when
`POKEPLATINUM_GENERATED_ENUM` is defined, used by C sources) and as `#define` constants (for the
assembler/scripting engine, which can't parse enums). If an `#include` can't be found in the source
tree, check `generated/*.txt` first — the real header is synthesized into `build/generated/`.

**Data-file processing pipeline (`tools/dataproc` + `res/`):** game data (species, moves, items,
trainers, Battle Frontier pools, NPC trades, etc.) is authored as human-editable JSON/text under
`res/<category>/` and compiled into game NARCs/headers by dedicated CLI tools in `tools/dataproc`
(`speciesproc`, `moveproc`, `itemproc`, `trainerproc`, `frontierproc`, `npctradeproc`, `dexproc`,
`resdatproc`), all built on the shared `lib/dataproc.c` interface and `src/common.c` helpers. Each
tool follows the same shape: `enums[]` (constants loaded from a header for referencing in JSON),
`archives[]` (output NARCs), `headers[]` (generated from `tools/dataproc/data/*.h.template`), and
`textbanks[]`. See `docs/datafiles/overview.md` for the full pattern and per-tool input/output table,
and `docs/datafiles/*.md` for format specifics on Pokémon/trainers/items/moves.

**Custom scripting engine:** field/battle event scripts are a bytecode format assembled from
`asm/macros/*.inc` (`scrcmd.inc`, `frscrcmd.inc`, `btlcmd.inc`, `btlanimcmd.inc`, `aicmd.inc`,
`movement.inc`, `pokemon_anim_cmd.inc`) — these `.inc`/`.s` files are intentionally excluded from
clang-format (see `.clang-format-ignore`).

**Other tools of note (`tools/`):** `metroskrew` is the vendored MWCC compiler toolchain (fetched via
`tools/devtools/get_metroskrew.sh`, not committed); `nitrorom` packs the final `.nds`; `nitroarc`
builds NARC archives; `nitrogfx`/`nitrobtx` handle graphics conversion; `asmdiff`/`m2ctx` assist
matching work; `fixrom`/`postconf` are build-time fixups; `ordergen` derives link order.

## Naming conventions (from `CONTRIBUTING.md`)

```c
// Structs: PascalCase, with both a tag and typedef.
typedef struct MyStruct {
    u32 myField; // camelCase members
} MyStruct;

// Enums: PascalCase name, NOT typedef'd. UPPER_SNAKE_CASE members;
// explicitly assign only the first member.
enum MyEnum {
    MY_ENUM_MEMBER = 0,
    MY_ENUM_MEMBER,
};

// Functions: PascalCase, camelCase parameters/locals.
static int MyFunction(int myParameter) { int myLocalVariable; }

// Public API of a module is typically prefixed Module_Name.
int Module_MyExternalFunction(int myParameter) { }
```

## Working notes

- A correct change must still produce a byte-identical ROM (`make check` green) — this is the
  binding correctness constraint for anything touching compiled code, unlike typical application
  repos where behavior-preserving refactors are the bar.
- `docs/bugs_and_glitches.md` documents *known, intentional-to-preserve* bugs from the original game
  and where in the (matched) source they live — don't "fix" these unless explicitly asked to, since
  the build must still match retail behavior by default.
- Two ROM revisions are supported (Rev 0 = original retail, Rev 1 = later patched cartridge); code
  differences between them are typically gated with `#if POKEPLATINUM_REVISION == 0/1`.
