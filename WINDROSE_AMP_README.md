# Windrose + WindrosePlus — AMP Linux Template

AMP Generic Module template for running a Windrose Dedicated Server with WindrosePlus on AMP (Linux), using Wine.

---

## Files

| File | Purpose |
|------|---------|
| `windrose.kvp` | Main AMP module config (start command, ports, update sources, console regex) |
| `windroseconfigjson.json` | Settings manifest — exposes server name, player slots, WindrosePlus multipliers and RCON config in the AMP UI |
| `windrosemetaconfig.json` | Maps AMP settings to `ServerDescription.json` and `windrose_plus.json` on disk |

Drop all three files into the top-level directory of your AMP Generic Module instances folder (same level as other templates like `abiotic-factor.kvp`).

---

## How it works

The Windrose Dedicated Server is **Windows-only**. This template runs it on Linux via **Wine**, the same approach used by the Abiotic Factor community template (which also uses UE4 and UE4SS). AMP's `cubecoders/ampbase:wine-stable` Docker image provides a pre-configured Wine environment.

**Update flow:**
1. SteamCMD downloads the Windrose Dedicated Server (App ID `4129620`, anonymous login) into `./windrose/4129620/`
2. A second update stage pulls the latest WindrosePlus release zip from GitHub and extracts it into the same directory
3. AMP then calls `wine WindroseServer.exe` to start the server

---

## Known limitation — PAK rebuild step

WindrosePlus requires a PAK rebuild step before each launch whenever multipliers or `.ini` files change. On Windows this is handled by `StartWindrosePlusServer.bat`, which calls `WindrosePlus-BuildPAK.ps1` before launching the exe.

**AMP does not run pre-launch scripts natively in the Generic Module.** There are two ways to handle this:

### Option A — Wrapper shell script (recommended for Linux)

Create a file at `./windrose/4129620/start_windrose.sh` and point AMP's `ExecutableLinux` at it instead of `/usr/bin/wine`:

```bash
#!/bin/bash
# WindrosePlus PAK rebuild before launch
# repak and retoc are Windows PE binaries — run them under Wine too
GAMEDIR="$(cd "$(dirname "$0")" && pwd)"

cd "$GAMEDIR"

echo "[AMP] Running WindrosePlus PAK rebuild..."
wine "$GAMEDIR/windrose_plus/tools/repak.exe" pack \
    "$GAMEDIR/windrose_plus/overrides" \
    "$GAMEDIR/R5/Content/Paks/WindrosePlus_override_P.pak" 2>/dev/null || true

echo "[AMP] Starting WindroseServer..."
exec wine "$GAMEDIR/WindroseServer.exe"
```

Then in `windrose.kvp` change:
```
App.ExecutableLinux=./windrose/4129620/start_windrose.sh
App.LinuxCommandLineArgs=
```

Mark it executable after the update step using an `UpdateSources` `SetExecutableFlag` entry:
```json
{
  "UpdateStageName": "Mark start script executable",
  "UpdateSourcePlatform": "Linux",
  "UpdateSource": "SetExecutableFlag",
  "UpdateSourceArgs": "4129620/start_windrose.sh"
}
```

> **Note:** The full `WindrosePlus-BuildPAK.ps1` script uses `repak.exe` and `retoc.exe` (bundled in the WindrosePlus release). These are native Windows PE binaries and need Wine to run on Linux. The wrapper above calls repak directly for simplicity. If you need the full PAK logic (multiple override files, stale PAK removal), mirror the PowerShell script's logic in bash, calling `wine repak.exe` and `wine retoc.exe` for each step.

### Option B — Skip multiplier PAK rebuilds, use live config only

RCON password, admin Steam IDs, and feature flags (`live_map`, `cpu_optimization`) in `windrose_plus.json` are read live and take effect without a PAK rebuild. If you only need those settings and not multipliers/`.ini` overrides, you can skip the rebuild step entirely. The server just won't apply multiplier or stat overrides.

---

## WindrosePlus install step

The WindrosePlus update source in `windrose.kvp` pulls the release zip from GitHub. AMP's GitHub update source extracts the zip into the target path. However, the mod still needs `install.ps1` to wire up `mods.txt` and the dashboard launcher.

After the first `Update` in AMP, run this manually once via SSH or the AMP console:

```bash
# Inside the AMP instance's root dir (where ./windrose/ is)
cd ./windrose/4129620
wine cmd /c "powershell -NoProfile -ExecutionPolicy Bypass -File install.ps1"
```

Or on Linux without PowerShell, you can wire it up manually:
```bash
# Create mods.txt if it doesn't exist
MODSDIR="./R5/Binaries/Win64/ue4ss/Mods"
mkdir -p "$MODSDIR"
echo "WindrosePlus : 1" >> "$MODSDIR/mods.txt"
```

---

## Port mapping in AMP

| Port | Protocol | Purpose |
|------|----------|---------|
| 7777 | UDP | Main game connection |
| 27015 | UDP | Server query (WindrosePlus adds query responder) |
| 27016 | TCP | WindrosePlus web dashboard and RCON |

Make sure all three are forwarded/opened in your firewall and AMP port config.

---

## First-run checklist

- [ ] AMP instance created with this template
- [ ] Run Update — SteamCMD downloads Windrose DS, GitHub stage pulls WindrosePlus
- [ ] SSH in and run the WindrosePlus install step (mods.txt wiring)
- [ ] Set RCON password in AMP settings (not blank, not "changeme")
- [ ] Set Server Name and optional Invite Code
- [ ] Adjust multipliers as needed
- [ ] Start the instance — Wine will spin up the server
- [ ] Check AMP console output for `LogWorld: Bringing World up for play`
- [ ] Access the WindrosePlus dashboard at `http://<your-server-ip>:27016`

---

## Troubleshooting

- **Server crashes on startup** — Check `R5/Binaries/Win64/ue4ss/UE4SS-settings.ini`. Only `HookProcessInternal` and `HookEngineTick` should be enabled.
- **No RCON / dashboard** — Confirm RCON password is set and port 27016 is open.
- **Wine crashes / missing DLL errors** — Make sure the `cubecoders/ampbase:wine-stable` Docker image is in use and Wine is fully initialized. Try running `wineboot` in the instance container first.
- **Map shows no data** — A player needs to join at least once to trigger the terrain export.
- **Multipliers not applying** — The PAK rebuild step didn't run. Follow Option A above or manually run `wine repak.exe` before restarting.
- **`mods.txt` missing WindrosePlus** — Run the install step from the First-run checklist.

---

## References

- [WindrosePlus GitHub](https://github.com/HumanGenome/WindrosePlus)
- [AMP Generic Module wiki](https://github.com/CubeCoders/AMP/wiki/Configuring-the-'Generic'-AMP-module)
- [AMP Community Templates](https://github.com/Greelan/AMPTemplates)
- [Windrose Dedicated Server guide (Steam)](https://steamcommunity.com/sharedfiles/filedetails/?id=3706337486)
- [Windrose Steam App ID: 4129620](https://steamdb.info/app/4129620/)
