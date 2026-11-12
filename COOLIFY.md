# Running this on Coolify with playit.gg

This fork is set up for a Coolify **Application** (Build pack: *Docker Compose*) with
playit.gg providing the public address. Unlike Terraria, Project Zomboid is **UDP** and
speaks on **two** ports that the server has to listen on using the exact numbers the
tunnel exposes.

## 1. Create the playit.gg tunnel

1. In the playit dashboard, add a tunnel and pick the **Project Zomboid** type
   (not "Custom UDP" / "Custom TCP").
2. playit assigns **two sequential public ports**, e.g. `10233` and `10234`. Write them
   down.
3. Set the tunnel's **Local IP** to the Docker host as seen from the playit container
   (the same address that works for Terraria, typically the bridge gateway or the host's
   LAN IP). `127.0.0.1` does **not** work: inside the playit container it points at
   playit itself.
4. The tunnel's local ports must be the same two numbers. The server is configured to
   listen on the public ports for exactly this reason.

If you delete and recreate the tunnel you get new public ports. Update `PZ_PORT` /
`PZ_UDPPORT` and redeploy, or nobody can join.

## 2. Create the Coolify application

- **Source:** this Git repository, branch `main`
- **Build pack:** Docker Compose
- **Docker Compose Location:** `/docker-compose.yml`
- **Domains:** none. This is a UDP game server, not an HTTP service.

Environment variables (bulk paste, then adjust the ports):

```env
ADMINUSERNAME=admin
ADMINPASSWORD=pick-a-strong-one
SERVERNAME=zomboid
SERVERPRESET=Apocalypse
DISPLAYNAME=Fellbach Z
MEMORY=4G
PUBLIC=false
PZ_PORT=10233
PZ_UDPPORT=10234
PZ_DATA_PATH=/data/coolify/zomboid
PZ_WORKSHOP_PATH=/data/coolify/zomboid-workshop
```

`ADMINPASSWORD` is only needed for the first boot. Once the server has generated
`Server/zomboid.ini`, delete the variable from Coolify and redeploy. It is passed as a
command line argument, and the server logs its arguments in clear text, so leaving it
set means the password is printed on every start.

## 3. Join

In the game: **Join → add server** with `147.x.x.x:10233` (the first playit port) and
**"Use Steam Relay" unchecked**.

A "port 16262 is closed / performance may be limited" warning means the second port is
not aligned end to end. Check that `PZ_UDPPORT`, the published port and the tunnel's
second port are all the same number.

## 4. Server and gameplay settings

Everything persists in `PZ_DATA_PATH`. The interesting files are:

| File | Contents |
| --- | --- |
| `Server/<SERVERNAME>.ini` | PvP, max players, safety system, password, mods, map list |
| `Server/<SERVERNAME>_SandboxVars.lua` | Zombie population, loot rarity, XP rates, day length, water/electricity shutoff, respawn |
| `Server/<SERVERNAME>_spawnregions.lua` | Spawn regions (mod maps append themselves) |
| `db/` | Player accounts and characters |
| `Saves/` | World saves |

Edit them over SSH, then restart the container in Coolify:

```bash
nano /data/coolify/zomboid/Server/zomboid_SandboxVars.lua
```

Two rules:

- Keep `SERVERPRESETREPLACE=false`. Setting it to `true` overwrites
  `SandboxVars.lua` from the preset on every start and throws your edits away.
- If you edit `Mods=` / `WorkshopItems=` in the ini by hand, set
  `SELF_MANAGED_MODS=true`. Otherwise the entrypoint rewrites both lines from
  `MOD_IDS` / `WORKSHOP_IDS` on every start, and with those empty it clears them.

`SERVERNAME` is fixed after the first boot: changing it makes the server look for a
different data set and start a fresh world (the old one is kept under the old name).
