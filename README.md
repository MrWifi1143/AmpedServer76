# Server76 — AMP Generic Module template (v7, battle-tested)

A personal-use AMP Generic template for **Server76**, the stand-alone private Fallout 76 server
(0mega / myfo.online). Verified working end-to-end on AMP 2.8.0.4 "Proteus" / Ubuntu 24.04.
Every fix below was discovered the hard way — this version has them all baked in.

## Files
| File | Purpose |
|------|---------|
| `server76.kvp` | Main template — launch, console, ports, all fixes |
| `server76config.json` | Settings UI (EULA toggle, mode, run-as, optional SteamCMD download) |
| `server76ports.json` | UDP 8000 game + TCP 2102 HTTPS relay (flat list) |
| `server76updates.json` | Install stages: download+extract (7z fallback), optional game files, required-file check |
| `manifest.json` | Repo manifest so AMP's configuration-repository system loads the template |

## Install the template (AMP 2.8+)
AMP 2.8 loads templates from **configuration repositories**, not the old GenericTemplates folder.
1. Put all five files in the root of a public GitHub repo.
2. AMP → Configuration → Instance Deployment → Configuration Repositories → **Add** → `youruser/yourrepo:main` → **Fetch**.
3. Hard-refresh the browser → Create Instance → the template appears (with your manifest's prefix).

## Host prerequisites (one-time, as root)
```
# .NET 10 runtime (Server76 is .NET 10 — NOT 8 as older guides say)
wget https://dot.net/v1/dotnet-install.sh && chmod +x dotnet-install.sh
./dotnet-install.sh --channel 10.0 --runtime dotnet --install-dir /usr/share/dotnet
mkdir -p /etc/dotnet && echo "/usr/share/dotnet" > /etc/dotnet/install_location
ln -sf /usr/share/dotnet/dotnet /usr/bin/dotnet

# extraction + download tools (Ubuntu's p7zip-full ships 7z but often not 7zr — template handles both)
apt-get install -y p7zip-full wget

# firewall
ufw allow 8000/udp && ufw allow 2102/tcp
```

## Game files (you must own Fallout 76)
Only **~2.4 GB** is needed, NOT the full 75 GB install. Place in
`<instance>/Server76/bin/gamedata/Data/`:
- `SeventySix.esm`
- `NW.esm`
- `SeventySix - Localization.ba2`  ← **server hard-requires this; exits silently (code 3) without it**
- `Terrain/Appalachia.btd`
- (`SeventySix.cdx` harmless to include)

Get them via SteamCMD depot download (needs an account owning FO76; Steam Guard requires ONE
interactive login first — `steamcmd +login <user>` — to cache the session):
```
download_depot 1151340 1151343 9006394379407093964
download_depot 1151340 1151342 1065890720657377764
```
If Steam blocks the login with a location-mismatch phishing wall (common on foreign-IP VPSes),
route the server through a Tailscale exit node near your phone for the ONE login, then drop the tunnel.

## Setup order
1. Create instance from the template → run **Update** (downloads + extracts Server76, then prints a
   required-file check so you know exactly what's missing).
2. Place the game files (above). Make sure everything is owned by the amp user:
   `chown -R amp:amp <instance>/Server76`
3. Configuration → enable **Accept EULA** (server refuses to start without it), set **Run As: Internet**.
4. **Start**. First boot builds the world database from the ESMs — takes minutes; be patient.
   Ready line: `World State Service ready` → AMP flips to Running → connect to `<ip>:8000`.

## The fixes baked into v7 (why this template works when naive ones don't)
1. **`UseLinuxIOREDIR=False`** — with the IOREDIR shim on (the old default), the .NET apphost
   can't find its runtime and you get endless `dotnet --headless does not exist / No .NET SDKs
   were found` garbage. All 200+ working CubeCoders templates set this False.
2. **Absolute `--gamedir` / `--serverdata`** via `{{$FullBaseDir}}` — Server76's
   `PathHelper.ResolveCaseInsensitive` throws `ArgumentOutOfRangeException` on relative paths.
   Absolute paths bypass the bug entirely, regardless of AMP's working directory.
3. **EULA wired to `--accept-eula`** — without the flag (or `EULAAccepted:true` in
   `Server76Settings.json`) the server exits code 3 **silently** right after "Saving settings".
4. **`SeventySix - Localization.ba2` required-file check** — the server stats this exact file and
   exits silently if absent (found via strace). The update's final stage now warns loudly.
5. **7zr→7z→7za fallback** — Ubuntu's p7zip packages don't reliably ship `7zr`; Debian's do.
6. **`DOTNET_ROOT` env var** + `/etc/dotnet/install_location` — dual registration so the apphost
   finds .NET 10 under any environment.
7. **Clean CRLF everywhere** — a KVP with mangled line endings is silently skipped or half-parsed
   by AMP, producing bizarre downstream failures. If you edit these files, keep CRLF.

## Debugging tips (earned the hard way)
- Server exits code 3 with no error → it's a silent gate: EULA, or a missing required game file.
  `strace -f -e trace=openat,stat ./Server76.Server --headless --no-tui 2>&1 | tail -60`
  shows the exact file check that failed.
- Server config persists in `Server76/bin/Server76Settings.json` — GameDirectory, EULAAccepted,
  ports, LogLevel (set 0 for max verbosity; logs land in `bin/logs/`).
- Ran it as root by accident? `chown -R amp:amp <instance>` or AMP gets Permission denied on
  `ServerData/Certificates/Server.pem`.
- AMP "Start" re-running update stages instead of launching = AMP can't parse/locate the
  executable — check the instance's `GenericModule.kvp` for mangled line endings.

## Caveats
- **EULA:** developing a private server is fine; *running* one is against Bethesda's EULA. Your call.
- **Not for the CubeCoders repo:** AI-assisted and requires a Steam login for game files — both
  disqualifying there. Personal use / direct sharing only.
