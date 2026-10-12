# Access Manager

**English** | **[Русский](https://github.com/Nord1cWarr1or/Access-Manager/blob/main/README.ru.md)**

Access Manager is an AMX Mod X plugin for Counter-Strike 1.6 that loads admin accounts from a MySQL database or a local INI file and authorizes players by SteamID, IP, or in-game name. It checks passwords, merges flags when several accounts match the same player, and drops accounts once they expire. Other plugins read access data through the natives and forwards below.

## Features

- Two account sources: the `amx_amxadmins` MySQL tables or `am_users.ini` with `am_local_mode "1"`.
- Authorization by SteamID, IP, or name, with a password check and an optional kick on failure.
- Several matching accounts merge into one flag set; each account keeps its own expiry date.
- Re-authorizes the player after a name change (ReAPI hookchain).
- Every SQL load writes `am_users_sql_backup.ini`; the backup is read at startup, so accounts are already in place before the query returns.
- Reuses the database connection of Fresh Bans or Lite Bans when they are installed.
- Localized messages via `data/lang/access_manager.txt`.

## Requirements

- Amx Mod X with the ReAPI module — the name-change hook runs through `RG_CBasePlayer_SetClientUserInfoName`.
- `sqlx` module — needed only in database mode; local mode works without it.

### Optional dependencies

- Fresh Bans or Lite Bans — without them the plugin opens its own connection from `am_host` / `am_user` / `am_password` / `am_database`.
- AmxBans RBS by SKAJIbnEJIb and Admin Load by Fant0M — the plugin exports `amxbans_get_expired()` and `adminload_get_expired()` so these panels work unchanged.

## Installation

1. Make sure ReAPI is installed on the server.
2. Compile `scripting/AccessManager.sma`:
   ```bash
   amxxpc AccessManager.sma -oAccessManager.amxx
   ```
3. Copy `AccessManager.amxx` to `addons/amxmodx/plugins/`.
4. Add `AccessManager.amxx` to `addons/amxmodx/configs/plugins.ini`.
5. Copy `configs/AccessManager.cfg` and `configs/am_users.ini` to `addons/amxmodx/configs/`, and `data/lang/access_manager.txt` to `addons/amxmodx/data/lang/`.
6. Set the database cvars in `AccessManager.cfg`, or set `am_local_mode "1"` to use only `am_users.ini`.
7. Change the map or restart the server.

## Commands

| Command | Access | Description |
|---------|--------|-------------|
| `amx_reloadadmins` | `ADMIN_CFG` | Reload accounts from the database or the INI file. Players without the flag can use it if `am_allow_reload_to_anyone` is `1`, subject to the cooldown. |

## CVars

| Cvar | Default | Description |
|------|---------|-------------|
| `am_local_mode` | `0` | `1` — read accounts from `am_users.ini` only, no database. |
| `am_allow_reload_to_anyone` | `0` | Allow `amx_reloadadmins` to players without the access flag. |
| `am_reloadadmins_cooldown` | `30` | Cooldown in seconds for reloads by unprivileged players. |
| `am_long_query_time` | `10` | Queries slower than this (seconds) are logged as long. |
| `am_host` | `""` | MySQL host. |
| `am_user` | `""` | MySQL user. |
| `am_password` | `""` | MySQL password. |
| `am_database` | `""` | MySQL database name. |
| `am_server_ip` | `127.0.0.1:27015` | Server address as `IP:PORT`, matched against `amx_serverinfo.address`. |
| `am_prune_expired` | `0` | Delete expired accounts created through `AccessManager_AddAccess()`. Requires the `ADD_ACCESS_FEATURE` define and an `ingame` column in `amx_amxadmins`. |
| `amx_password_field` | `_pw` | Client info key that carries the password (stock AMX Mod X cvar, bound by the plugin). |
| `amx_default_access` | `z` | Flags for players who match no account (stock AMX Mod X cvar, bound by the plugin). |

## Natives and forwards

Include the API and call it:

```pawn
#include <access_manager>

public client_admin(pPlayer, bitAccessFlags) {
    // fires after the player is authorized
}

new szNick[MAX_ACCOUNT_NAME_LENGTH];
if (AccessManager_GetNickName(pPlayer, szNick, charsmax(szNick))) {
    // account nickname from the database or am_users.ini
}
```

| Native | Description |
|--------|-------------|
| `AccessManager_GetNickName(pPlayer, szBuffer[], iMaxLen)` | Nickname of the account the player is authorized with. Returns the copied length, or `0` if the player is offline or has no account. |
| `AccessManager_GetNickNameByAuthID(const szAuthID[], szBuffer[], iMaxLen)` | Nickname looked up by authid (SteamID, IP, or name as stored). Returns the copied length, or `0` when nothing matches. |
| `AccessManager_ReAuthPlayer(pPlayer)` | Run authorization again for a connected player. Returns `1` on success, `0` if the player is offline. |
| `admin_expired(pPlayer)` | Expiry timestamp of the player's account, `0` if it never expires. |
| `amxbans_get_expired(pPlayer)` | Same value, alias for AmxBans RBS. |
| `adminload_get_expired(pPlayer)` | Same value, alias for Admin Load. |
| `AccessManager_AddAccess(pPlayer, const szAuthID[], const szFlags[], iMinutes, iCaller)` | Grant temporary access. Compiled only with the `ADD_ACCESS_FEATURE` define; returns a request id, or `-1` when no account source is available. |

| Forward | Description |
|---------|-------------|
| `client_admin(pPlayer, bitAccessFlags)` | A player has been authorized; carries the merged flags. |
| `amxx_admin_access(pPlayer, bitAccessFlags, iExpireTimeStamp)` | Same moment, with the expiry timestamp. |
| `AccessManager_AccessAdded(iUserID, iCallerID)` | `AccessManager_AddAccess()` finished; compiled only with `ADD_ACCESS_FEATURE`. |

## Technical notes

- Auth flags in `am_users.ini`: `a` — kick on a wrong password, `c` — the account key is a SteamID, `d` — an IP, `e` — no password check.
- The nickname field identifies the account; it takes no part in authorization.
- The plugin first tries the Fresh Bans (`fbans_sql_connected`) or Lite Bans (`lite_bans_sql_init`) tuple and falls back to its own cvars.
- `AccessManager_AddAccess()` and `am_prune_expired` are compile-time optional: uncomment `ADD_ACCESS_FEATURE` in the source, add an `ingame` int column to `amx_amxadmins`, and recompile.

## Credits

- **mx?!** — author.
- **Nordic Warrior** — added the `AccessManager_GetNickName()` and `AccessManager_GetNickNameByAuthID()` natives, wrote this README.
- **Neygomon** — «Admin Loader», some authorization mechanics taken from it.

## License

TODO: no `LICENSE` file in the repository yet.