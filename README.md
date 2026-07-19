# Obin

Obin is application tooling for the Dolet language. It keeps the compiler and
the package manager focused: Obin reads an application manifest, invokes
`doletc`, delegates declared package installation to DOPM, runs optional tool
adapters such as Frog's asset cooker, and stages distributable applications.

Obin itself is written in Dolet and has no Node.js dependency.

## Quick start

```powershell
obin init my-app
cd my-app
obin run
obin package
```

An explicit manifest may be passed to every project command:

```powershell
obin build "path/to/dolet.toml" --profile release
```

Both `obin.toml` and `dolet.toml` are recognized. `obin.toml` is preferred when
both exist.

## Manifest

```toml
[application]
name = "my-game"
version = "0.1.0"
entry = "src/main.dlt"
type = "game" # console, gui, or game

[build]
target = "windows"
default-profile = "dev"
incremental = true

[profiles.dev]
optimization = 0
no-console = false

[profiles.release]
optimization = 3
no-console = true

[tools]
# doletc = "C:/optional/custom/doletc.exe"
# dopm = "C:/optional/custom/dopm.exe"

[packages]
auto-sync = true

[dependencies]
# frog = "*"

[assets]
roots = ["assets"]
destination = "assets"

[tool.frog]
enabled = true
cook-models = true
cook-textures = true
texture-roots = ["assets/textures"]
texture-excludes = ["@channels="]
# Optional type overrides; common names are inferred automatically.
texture-normal-patterns = ["_n.png", "normal.png"]
texture-metallic-roughness-patterns = ["metallicRoughness.png"]
texture-occlusion-patterns = ["_ao.png"]
texture-data-patterns = ["lookup/"]
texture-color-patterns = ["albedo/"]
texture-lossless-patterns = ["ui/pixel-art/"]
# keep-source-textures = true # optional PNG fallback inside release packages
lods = [0.45, 0.22]
# cooker = "tools/custom-frog-cooker.exe"

[hooks]
# before-build = ["scripts/generate.exe"]
# after-build = ["scripts/check.exe"]
# after-package = ["scripts/sign.exe"]

[package]
directory = "dist"
include-assets = true
```

Dependency version strings are recorded by Obin, but the current DOPM source
only exposes `install <name>` and does not resolve versions. Obin therefore does
not pretend that version resolution exists: it delegates package names exactly
as supported and avoids rerunning DOPM until the dependency declaration changes.

## Commands

- `obin init [directory]`
- `obin build [manifest] [--profile name] [--target name] [--out path]`
- `obin run [manifest] [build options] [-- application arguments]`
- `obin package [manifest]` (uses the `release` profile by default)
- `obin clean [manifest]`
- `obin doctor [manifest]`

`clean` is restricted to paths inside the project root. The Frog adapter builds
the pure-Dolet cooker into `.obin/tools`, discovers referenced `.gltf`, `.glb`,
and `.png` files, checks source inputs, cooker protocol, format versions, and
timestamps, and cooks only stale assets. Static PNG literals are discovered by
default. `texture-roots` adds dynamically selected texture directories;
`texture-excludes` skips matching paths. Set `cook-literal-textures = false` to
cook only explicit texture roots. Type-specific patterns override filename
inference when an asset has an ambiguous name. FrogTexture v2 stores GPU-native
BC7/BC5/BC4 (or explicit lossless RGBA8) plus a complete offline mip chain;
Obin validates its header and every mip entry before treating it as current.
Frog prefers adjacent `.frogmodel` and `.frogtex` files at runtime and retains
source-file fallbacks for development. Release packages omit a PNG only when
its adjacent `.frogtex` passes validation; set `keep-source-textures = true` to
ship both copies for hardware that lacks the selected compressed format.
