# Kevbot AMP Templates

CubeCoders AMP `GenericModule` templates for game servers. Source of truth lives here in the ops workspace; the directory is also a separate git repo pushed to `github.com/Kdehner/KevbotAmpTemplates` so AMP can fetch from it as a Configuration Repository.

The `amp-templates/` path is gitignored from the parent `ops/` repo so the two histories stay independent.

## How AMP consumes this

1. AMP UI → Configuration → Configuration Repositories → add `Kdehner/KevbotAmpTemplates:main`
2. AMP fetches `manifest.json` from the repo root, then any per-template files referenced by it
3. When you create a new instance from a template, AMP renders the kvp + config files into the instance directory at `/home/amp/.ampdata/instances/<Name>/`
4. AMP starts the instance in a Docker container (image `cubecoders/ampbase:debian` for plain Linux, `:wine-stable` for Wine/Proton games), bind-mounting the instance dir to `/AMP`

## Repo layout

```
amp-templates/
├── manifest.json        ← repo-level metadata, lists all templates
├── README.md            ← this file
└── <prefix>.kvp         ← one set per template:
    <prefix>config.json
    <prefix>metaconfig.json
    <prefix>ports.json
    <prefix>updates.json
```

The `prefix` matches the `App.AppName` in the kvp (e.g. `dmp`, lowercase, no spaces).

## Required files per template

### `<prefix>.kvp` — main definition (dotted key=value)

Sets `Meta.*` (display info, container requirements), `App.*` (executable, working dir, args), `Console.*` (regex patterns for ready/join/leave/chat detection).

Key fields that bit us:

| Field | What we learned |
|---|---|
| `Meta.SpecificDockerImage` | `cubecoders/ampbase:debian` is fine for Linux-native binaries; use `:wine-stable` for Windows-only games |
| `Meta.ContainerPolicy=RequiredOnLinux` | Forces docker isolation, matches how every other template on this host runs |
| `Meta.ExtraContainerPackages` | JSON array of apt packages installed at container boot via `AMP_CONTAINER_DEPS` in `ampstart.sh`. Use this for runtime deps (e.g. `["mono-runtime", "libmono-system-numerics4.0-cil", "libmono-system-runtime-serialization4.0-cil"]` for a Mono game). |
| `App.RootDir` | Relative to `/AMP`. Where `App.UpdateSources` extracts to. |
| `App.BaseDirectory` | The "logical" application root, used for `{{$FullBaseDir}}` substitution. |
| `App.WorkingDir` | **Relative to `RootDir`**, not `/AMP`. CWD when AMP launches the executable. If empty, AMP may strip relative arg paths. |
| `App.ExecutableLinux` | Absolute path (`/usr/bin/mono`) or relative to RootDir (`.proton/proton`). Affects how args are forwarded — see gotcha below. |
| `App.LinuxCommandLineArgs` | **Only forwarded when `ExecutableLinux` is a relative path.** When absolute, AMP silently drops this and only forwards `App.CommandLineArgs`. |
| `App.CommandLineArgs` | Always forwarded. Use `{{$FullBaseDir}}foo.exe` to give an absolute path to your assembly. |
| `App.AdminMethod=STDIO` | Standard for console-driven servers. Use `RCON` if the game has an RCON protocol. |
| `App.ExitMethod=String` + `App.ExitString` | The string AMP types to gracefully shut down (e.g. `/quit`). |
| `Console.AppReadyRegex` | When AMP marks the instance "Ready". Match the line the server prints once accepting connections. |
| `App.Ports` | `@IncludeJson[<prefix>ports.json]` — gets inlined as a JSON array. |
| `App.UpdateSources` | `@IncludeJson[<prefix>updates.json]` — likewise. |

### `<prefix>config.json` — UI config schema

JSON array of nodes describing the user-editable settings shown in AMP's Configuration tab. Each node has:
- `DisplayName`, `Description`, `Keywords`
- `Category`, `Subcategory` — see icon gotcha below
- `FieldName` (`$ApplicationPort1`, `$MaxUsers`, etc. — these are AMP variables; or arbitrary `$Custom` for AppSettings)
- `ParamFieldName` — the key written into the actual config file (matches `<prefix>metaconfig.json` regex)
- `InputType` — `text`, `number`, `checkbox`, `enum`, `password`
- `EnumValues` — for enum inputs
- `IncludeInCommandLine` — true if this should also appear in `{{$FormattedArgs}}`

### `<prefix>metaconfig.json` — server config file mapping

Tells AMP how to read/write the game's actual config file. Single-element JSON array:
```json
[{
  "ConfigFile": "Config/Settings.txt",
  "AutoMap": true,
  "Importable": true,
  "ConfigType": "ini",
  "ConfigFormatRegex": "^(?<key>.+?)=(?<value>.*?)$"
}]
```

### `<prefix>ports.json` — network ports
```json
[{"Protocol": "TCP", "Port": 6702, "Ref": "ApplicationPort1",
  "Name": "Game Port", "Description": "..."}]
```

### `<prefix>updates.json` — download sources
```json
[{"UpdateStageName": "...", "UpdateSourcePlatform": "All",
  "UpdateSource": "FetchURL",
  "UpdateSourceData": "https://...",
  "UpdateSourceArgs": "filename.zip",
  "UpdateSourceTarget": "{{$FullRootDir}}",
  "UnzipUpdateSource": true,
  "DeleteAfterExtract": true,
  "OverwriteExistingFiles": false,
  "SkipOnFailure": false}]
```

## Adding a new template

1. **Copy the DMP files** as a starting point: `cp dmp.kvp newgame.kvp` and the four matching json files
2. **Edit `<prefix>.kvp`**: change `App.AppName`, `Meta.DisplayName`, `Meta.AppConfigId` (generate a fresh UUID), `App.RootDir`, `App.BaseDirectory`, executable, args
3. **Add runtime deps** to `Meta.ExtraContainerPackages` if the binary needs anything beyond a bare Debian (e.g. mono, java, dotnet, libssl1.1)
4. **Update `<prefix>updates.json`** with the download URL
5. **Update `<prefix>ports.json`** with whatever ports the game listens on
6. **Update `<prefix>config.json`** with the user-tunable settings
7. **Edit `manifest.json`** and add the new template to the list (if AMP requires per-template entries — currently we have one repo-level manifest)
8. **Test locally**: commit, push, and in AMP refresh the configuration repository, then create a test instance
9. **Iterate via `docker exec` patches** while debugging (see below), then sync the working values back to source and push

## Critical gotchas (learned the hard way)

### Args don't pass when ExecutableLinux is absolute
If `App.ExecutableLinux=/usr/bin/foo`, AMP silently **drops** `App.LinuxCommandLineArgs`. Put the assembly path / args in `App.CommandLineArgs` instead, with `{{$FullBaseDir}}` for the absolute path. Symptom: process launches with argc=0.

### WorkingDir is relative to RootDir
`App.WorkingDir=DMPServer` with `App.RootDir=./dmp/` resolves to `/AMP/dmp/DMPServer`. Empty WorkingDir means cwd defaults to RootDir, which may not be where your binary lives. AMP will strip relative path arguments it can't resolve from cwd.

### Icon names must exist in AMP's bundled font
`Category` and `Subcategory` use `Name:icon` (or `Name:icon:order`) format where `icon` is a Material Icon name. AMP ships a subset — `rocket_launch` is **not** in it (causes "Syntax error, unrecognized expression: unsupported pseudo: rocket_launch" in the UI). Confirmed working icons from other templates: `stadia_controller`, `joystick`, `videogame_asset`, `extension`, `skull`, `dns`, `settings`, `build`, `system_update_alt`, `download`, `storage`, `share`, `bug_report`, `cleaning_services`, `speed`.

### Don't put `:N` order suffix on top-level Category
Three-part `Name:icon:N` works for `Subcategory` but renders the `:1` as junk text on top-level `Category`. Use two-part `Name:icon` for Category.

### File ownership in containers
The container runs as user `amp` (uid 1001) inside, but `ampstart.sh` runs as root for setup. If the update extraction or any prior run leaves files owned by root, AMP can't write to them and config merging fails with `UnauthorizedAccessException`. Quick fix: `docker exec -u root <container> chown -R amp:amp /AMP/<rootdir>/`.

### kvp is loaded once at AMP-process startup
AMP reads `/AMP/GenericModule.kvp` when the AMP process boots inside the container. Editing the file via `docker exec` does **not** take effect until you `docker restart <container>`. Clicking "Restart" in the AMP UI only restarts the application, not AMP itself.

### AMP rewrites the kvp on its own startup
On AMP-process startup, some kvp fields get re-derived from configmanifest sources and overwrite your edits. Empirically: `WorkingDir` and `LinuxCommandLineArgs` survive; `ExecutableLinux` does not. If you must override `ExecutableLinux` for debugging, replace `/usr/bin/<binary>` with a wrapper script in the container instead of editing the kvp.

### configmanifest.json is rendered, not authoritative
The `<prefix>config.json` file in this repo is the source. When AMP installs the template into an instance it renders `/AMP/configmanifest.json` from it. Editing the source and pushing won't affect existing instances until you click "Update Application" (or delete + recreate the instance).

## Debugging recipes

### See exactly what AMP invokes a binary with
Replace the real binary with a tracing wrapper:

```bash
docker exec -u root <container> bash -c '
mv /usr/bin/mono /usr/bin/mono.real
cat > /usr/bin/mono << "EOF"
#!/bin/bash
{
  echo "=== pid=$$ cwd=$(pwd) argc=$# ==="
  i=0; for a in "$@"; do echo "ARG[$i]=$a"; i=$((i+1)); done
  echo "PARENT=$(tr "\0" " " < /proc/$PPID/cmdline)"
  echo "---"
} >> /tmp/trace.log
exec /usr/bin/mono.real "$@"
EOF
chmod +x /usr/bin/mono'
```

Click Start in AMP, then `docker exec <container> cat /tmp/trace.log`. Restore with `mv /usr/bin/mono.real /usr/bin/mono` when done.

### Iterate without re-pushing
1. Patch `/AMP/GenericModule.kvp` (or `/AMP/configmanifest.json`) inside the container with `docker exec sed`
2. `docker restart <container>` to make AMP reload
3. Test
4. When satisfied, sync the change back to this repo, commit, push

### Compare against a known-working template
The host runs Astroneer, Valheim, ARK, ASKA, 7 Days, etc. — all useful references. Inspect their kvp/manifest:
```bash
docker exec AMP_FWBValheim01 grep -E "^App\." /AMP/GenericModule.kvp
docker exec AMP_KMARK01 head -200 /AMP/configmanifest.json
```

### Reset world / change game mode (DMP specifically)
DMP's world state lives in `Universe/`. Settings are key=value INI in `Config/Settings.txt` (`gameMode=SANDBOX|SCIENCE|CAREER`).

```bash
# With the application STOPPED:
docker exec -u amp <container> bash -c '
cd /AMP/dmp/DMPServer
mv Universe Universe.bak.$(date +%Y%m%d-%H%M%S)
mkdir Universe
sed -i "s|^gameMode=.*$|gameMode=CAREER|" Config/Settings.txt'
```

## Current templates

- **dmp** — DarkMultiPlayer for Kerbal Space Program. SpaceDock mod 11. Mono-based, port 6702/TCP, console-driven (`/quit` to exit, "Ready!" line on startup).
