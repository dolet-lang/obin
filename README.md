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
default-target = "windows"
default-profile = "dev"
incremental = true

[targets.windows]
id = "windows/x86_64"
package-require = ["bin/my-game.exe"]

[targets.linux]
id = "linux/x86_64"
package-require = ["bin/my-game"]

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
executable-directory = "bin" # optional
include = ["LICENSE", "config"] # optional project-relative SDK/runtime files
require = ["LICENSE"] # optional final-package contract
```

Dependency values are passed to DOPM as a source path/URL or Git revision.
DOPM resolves them to exact commits in `dopm.lock`, installs sources below
`.dopm/packages`, and Obin passes that project-local root to `doletc`. Every
build asks DOPM to validate the installed commit and clean Git tree; this uses
DOPM's local fast path and does not fetch again. Deleted, replaced, or edited
package directories therefore cannot be hidden by Obin's incremental state.
Changes to the lock/state invalidate incremental application outputs.

## Commands

- `obin init [directory]`
- `obin build [manifest] [--profile name] [--target name] [--out path]`
- `obin run [manifest] [build options] [-- application arguments]`
- `obin package [manifest] [--target name]` (uses the `release` profile by default)
- `obin clean [manifest]`
- `obin doctor [manifest]`

Target names are project aliases. `obin build --target linux` resolves
`targets.linux.id` and passes the canonical `linux/x86_64` target to
`doletc`. Artifacts are isolated under `build/<target>/<profile>/`, and packages
are named `<application>-<version>-<target>` so different operating-system
builds never overwrite each other. A canonical target may also be passed
directly to `--target`.

Select each output explicitly with `--target windows` or `--target linux`.
Explicit targets keep platform-specific SDK preparation, packaging, and logs
independent and prevent an accidental build for an unprepared platform.

`package.include` stages project-relative files or trees at the same relative
path. VCS metadata, hidden cache directories, `.obin`, and `node_modules` are
excluded, and includes are forbidden from escaping either the project or the
package stage. A target may add files with
`targets.<alias>.package-include`. `package.executable-directory` places the
program under a subdirectory such as `bin/`, which is useful for SDK layouts.
`package.require` and `targets.<alias>.package-require` declare paths that must
exist in the final staged bundle. Obin validates them after `after-package`
hooks and refuses to report success for an incomplete package. Package copying
also rejects host-local symbolic links and reparse points so a distributable
cannot accidentally depend on paths from the build machine; a thin SDK should
ship a target-native setup script which creates any required local links.

`clean` is restricted to paths inside the project root. The Frog adapter builds
the pure-Dolet cooker into `.obin/tools`, discovers referenced `.gltf`, `.glb`,
and `.png` files, checks source inputs, cooker protocol, format versions, and
timestamps, and cooks only stale assets. Static PNG literals are discovered by
default. `texture-roots` adds dynamically selected texture directories;
`texture-excludes` skips matching paths. Set `cook-literal-textures = false` to
cook only explicit texture roots. Type-specific patterns override filename
inference when an asset has an ambiguous name. FrogTexture v2 stores GPU-native
BC7/BC5/BC4 (or explicit lossless RGBA8) plus a complete offline mip chain;
Obin validates its header, every mip entry, and the configured texture recipe
before treating it as current. Changing a texture pattern therefore recooks
older outputs automatically without requiring `--force`.
Frog prefers adjacent `.frogmodel` and `.frogtex` files at runtime and retains
source-file fallbacks for development. Release packages omit a PNG only when
its adjacent `.frogtex` passes validation; set `keep-source-textures = true` to
ship both copies for hardware that lacks the selected compressed format.
