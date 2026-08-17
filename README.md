# Schedule I Dedicated Server for AMP

An [AMP (CubeCoders)](https://cubecoders.com/AMP) deployment template that runs a **Schedule I dedicated server** using the [S1DedicatedServers mod](https://github.com/ifBars/S1DedicatedServers) by ifBars. It wraps the official S1DS Docker image (`ghcr.io/ifbars/s1dedicatedservers`), which ships the whole Wine + Xvfb + MelonLoader + SteamCMD stack. AMP gives you start/stop, console, player list, scheduling and backups, while upstream maintains the tricky parts.

> **Note:** This is a community template. It is not affiliated with CubeCoders or the S1DedicatedServers authors.

## Requirements

- AMP 2.6+ on **Linux** with an ADS ("Controller") instance
- **Docker installed on the host**, and the AMP system user (usually `amp`) in the `docker` group
- A **Steam account that owns Schedule I** (the server download needs a real login, use a dedicated account if possible)
- `setfacl` available on the host (package `acl`; preinstalled on most distros)
- ~15 GB disk for the game install, ~4 GB RAM for an idle server (more with players)

## Server setup

1. In AMP: **Configuration → Instance Deployment → Configuration Repositories → Add**, enter:
   ```
   Wasilewskiii/s1ampserver:main
   ```
   then fetch/refresh. "Schedule I Dedicated Server (S1DS)" appears in the Create Instance application list.
2. **Create the instance without a container.** It must run directly on the host so it can reach the Docker daemon. AMP shows an advisory about this during creation. If you create it inside a container anyway, the first start stops with a clear error telling you to recreate it on the host.
3. Open the instance → **Configuration → Schedule I** and set:
   - **Steam Username / Steam Password**: the account that owns the game (stored only in a `chmod 600` env file inside the instance; avoid single quotes in credentials)
   - **Game Runtime**: `IL2CPP` (recommended: players stay on the game's default Steam branch) or `Mono` (uses the `alternate` branch)
   - **Docker Image Tag**: `latest`, or pin a version for controlled upgrades
4. **Start** the server and watch the console:
   - If Steam Guard prompts, stop the server, enter the code in **Steam Guard Code** (email codes are sent only after a login attempt), start again, and clear the field once logged in. The session is cached afterwards.
   - First boot downloads ~7 GB via SteamCMD and (on IL2CPP) runs a one-time Cpp2IL code generation, expect 15 to 30 minutes. Later starts are fast.
5. Server name, password and max players are normal AMP settings in the same section and get written into `server_config.toml` on every start. Anything more exotic can be edited directly in `s1ds/game/UserData/server_config.toml` via the AMP file manager.

### Ports

| Port | Protocol | Purpose |
|------|----------|---------|
| 38465 | UDP | Game traffic |
| 38465 | TCP | S1DS status query |
| 27018 | UDP | Steam game server query (`steamGameServerQueryPort` in `server_config.toml` must match) |

AMP registers the host firewall rules automatically. Forward these on your router if the server is behind NAT.

### Saves, backups, updating

- World data and config live in `s1ds/game/UserData/`, which is the part worth backing up. AMP's Backups tab works normally.
- To import an existing world, follow the [official save-path guide](https://github.com/ifBars/S1DedicatedServers/blob/master/docfx/docs/configuration/save-path.md) and never overwrite the server's `Players/Player_0` data.
- **Updating**: run the AMP Update action to `docker pull` the configured image tag. Game updates are pulled by SteamCMD on every server start.

## Player setup (what your friends must install)

Players **cannot join with an unmodded game**. Each player needs, one time:

1. **Schedule I on the default Steam branch**.
2. **MelonLoader**: installer from [MelonLoader releases](https://github.com/LavaGang/MelonLoader/releases). Pick **0.7.2 or newer** (avoid 0.7.1, known broken). Point it at the Schedule I install folder.
3. **The S1DS client mod**: from [S1DedicatedServers releases](https://github.com/ifBars/S1DedicatedServers/releases) download **`Il2cpp_Client.zip`** (or `Mono-Client.zip` if the server runs the Mono runtime) and extract it into the Schedule I game folder, so that `DedicatedServerMod_Il2cpp_Client.dll` ends up in the game's `Mods\` folder.

Then launch the game normally. A MelonLoader console opens alongside and lists the loaded mods. The mod adds a **join option to the main menu**: enter the server address as `IP:38465` and connect.

Keep the client mod on the **same S1DS release as the server**. The server verifies client mods at join and version mismatches are the most common "can't connect" cause.

## Known limitations

- AMP's CPU/memory graphs for this instance show the tiny `docker` client process, not the game. Check real usage with `docker stats s1ds-server` on the host. Cap the container's resources with the Memory/CPU Limit settings.
- The mod currently advertises but does not enforce `serverPassword` at join with the SteamGameServer auth provider (as of v1.0.5).

## Troubleshooting

- **"Docker daemon not reachable" on start**: the instance was created inside an AMP container, or the AMP user is not in the `docker` group. See step 2 of Server setup.
- **`STEAM_USER and STEAM_PASS must be provided`**: Steam credentials not set in the instance configuration.
- **Steam Guard loop**: email codes expire; trigger a fresh login attempt, then use the newest code promptly.
- **Players time out**: check the three ports above are open/forwarded, and that `server_config.toml` ports match the template's port settings.

## Future ideas

- A second, AMP-native template variant that runs the game inside AMP's own Wine container (`cubecoders/ampbase:wine-stable`, like the Abiotic Factor template) instead of wrapping the upstream Docker image. That would give correct CPU/RAM graphs and AMP's built-in container resource limits, at the cost of maintaining the Wine, MelonLoader and .NET setup in the template itself rather than relying on the upstream image.

## Credits

- [ifBars and contributors](https://github.com/ifBars/S1DedicatedServers): the S1DedicatedServers mod and Docker image
- [CubeCoders](https://cubecoders.com/AMP): AMP and the Generic module template system
