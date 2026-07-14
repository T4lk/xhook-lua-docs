# Events

## Events

Register with `callbacks.register(name, fn)`; remove with
`callbacks.unregister(name, fn)`. A handler that errors is logged (with a stack
traceback) and skipped — it never takes down the game or the other handlers.

| Event              | Fires                          | Handler signature        |
|--------------------|--------------------------------|--------------------------|
| `on_paint`         | every overlay frame            | `function()`             |
| `on_create_move`   | every user-command tick        | `function(cmd)`          |
| `on_round_start`   | at each round restart          | `function()`             |
| `on_player_death`  | every death (DeathMsg)         | `function(e)`            |
| `on_player_hurt`   | local player takes damage      | `function(e)`            |
| `on_weapon_fire`   | local player fires a shot      | `function(e)`            |
| `on_sound`         | a player/weapon sound played   | `function(e)`            |
| `on_item_pickup`   | local player picks up an item  | `function(e)`            |
| `on_chat`          | a player sends a chat line     | `function(e)`            |
| `on_net_message`   | a hooked user message arrives  | `function(msg)`          |
| `on_round_end`     | round ends (win/draw)          | `function(e)`            |
| `on_bomb_planted`  | C4 planted (de_ maps)          | `function()`             |
| `on_bomb_defused`  | C4 defused (de_ maps)          | `function()`             |
| `on_script_load`   | after scripts (re)load         | `function()`             |
| `on_script_unload` | before a script is dropped     | `function()`             |

**`cmd`** (on_create_move) is a mutable table. The hook runs inside the client's
CreateMove, so these fields are **client-set and take effect** when you write
them: `viewangles` (`Angle`), `buttons` (int), `forwardmove` / `sidemove` /
`upmove` (number), `impulse` (int), `weaponselect` (int — set the outgoing weapon
id for a silent switch), `impact_index` (int), `impact_position` (`Vector`).
Drive movement by writing `forwardmove`/`sidemove`/`upmove` + `buttons`; drive
aim/anti-aim by writing `viewangles`.

`msec`, `lerp_msec`, and `lightlevel` are present but the **engine stamps them
after this hook** — they read ~0 here and writing them has no effect (informational
only). GoldSrc has no `tick_count` / `random_seed` (Source-only).

**`on_player_death`** payload `e`: `attacker` / `victim` (`Entity` or `nil`),
`attacker_index` / `victim_index` (int, 1-based; `attacker_index <= 0` = world),
`weapon` (string), `headshot` (bool).

**`on_player_hurt`** payload `e`: `health` (local player's remaining HP),
`armor`, `damage` (int), `fall` (bool). GoldSrc only networks the Damage message
to the victim, so this fires for **local-player** damage only.

**`on_weapon_fire`** payload `e`: `weapon_id`, `weapon` (name), `clip`. Detected
from the local weapon's clip dropping — **local player** only. **`on_item_pickup`**
payload `e`: `item` (classname). **`on_round_end`** payload `e`: `winner` (`"T"`,
`"CT"`, or `"draw"`).

**`on_sound`** payload `e`: `entity` (`Entity` or nil), `index`, `origin`
(`Vector`), `file`, `volume`, `is_player`, `is_weapon`, `is_footstep`. Fires for
player/weapon sounds resolved to their source — use it for **footstep positions**
and **remote-player gunfire** (unlike `on_weapon_fire`, which is local-only).

**`on_chat`** payload `e`: `entity` (`Entity` or `nil`), `index` (1-based speaker
index; `0` = server/console), `name` (parsed speaker name, may carry a `(CT)` /
`(T)` / `*DEAD*` prefix), `text` (the message). Fires for every incoming chat
line; `entity` is a player handle only when the speaker is a live connected
player.

```lua
callbacks.register("on_player_death", function(e)
    local who = e.attacker and e.attacker:name() or "world"
    engine.log(who .. " killed index " .. e.victim_index ..
               (e.headshot and " (HS)" or "") .. " with " .. e.weapon)
end)
```

---

