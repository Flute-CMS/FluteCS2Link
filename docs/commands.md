# Console commands

All commands are registered with `FCVAR_GAMEDLL`. They are available from the server console and via RCON, but **not exposed to clients**.

---

## `mm_getinfo`

Dumps a full report — server info + all connected real players — as a single JSON line. Bots are filtered out.

**Usage:**
```
mm_getinfo
```

**Response:** see [schema.md — Server info](schema.md#server-info)

**Size:** ~160 bytes skeleton + ~200 bytes per player. A 60-player server produces ~12 KB.

Internally the output is chunked into 1024-byte pieces before being written to the console. RCON clients must reassemble multi-packet responses — any modern RCON library does this automatically.

---

## `mm_getinfo_slot <N>`

Dumps the JSON for a single player by slot number.

**Usage:**
```
mm_getinfo_slot 2
```

**Response:**
- Slot occupied by a real player → player object, see [schema.md — Player](schema.md#player)
- Slot empty, bot, or invalid → `{}`

**When to use:** admin dashboards polling one player several times per second. Much cheaper than running full `mm_getinfo`.

---

## `mm_getinfo_file <path>`

Atomically writes the full JSON (identical to `mm_getinfo` output) to a file at the given path.

**Usage:**
```
mm_getinfo_file /var/www/html/stats.json
```

**Console reply:**
- `mm_getinfo_file: ok` — success
- `mm_getinfo_file: failed` — open/write/rename failed

**Atomicity:** the file is first written to `<path>.tmp`, then `rename(2)`'d into place. Readers never see a partial or corrupt JSON.

**Typical use:**
- a cron job (or external scheduler) invokes the command via RCON every N seconds
- nginx serves the file as a static asset: `location = /stats.json { alias /var/www/html/stats.json; }`
- the web frontend does `fetch('/stats.json')` instead of speaking RCON

**Permissions:** the server process (`steam` inside `joedwards32/cs2`) needs write access to the target directory.

---

## Edge cases

| Situation | Behavior |
|---|---|
| Command invoked by a client | Silently unavailable (no `FCVAR_CLIENTCMD_CAN_EXECUTE`) |
| `mm_getinfo_slot` with no arg | Console: `usage: mm_getinfo_slot <slot>` |
| `mm_getinfo_slot` with a non-numeric arg | `mm_getinfo_slot` returns `{}` |
| `mm_getinfo_file` with no arg | Console: `usage: mm_getinfo_file <path>` |
| Slot out of range (≥ `maxClients`) | `mm_getinfo_slot` returns `{}` |
| Empty server | `mm_getinfo` returns a JSON with `players: []`, `player_count: 0` |
| Nickname contains `%`, `"`, `\`, emoji, Cyrillic | Correctly escaped by `nlohmann::json`; invalid UTF-8 bytes are stripped |
