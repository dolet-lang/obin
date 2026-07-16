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
the pure-Dolet cooker into `.obin/tools`, discovers referenced `.gltf` files,
checks source buffers and cooker timestamps, and cooks only stale assets.

