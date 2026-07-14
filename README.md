# Lua API Reference

- **Where scripts go:** `<settings>\scripts\*.lua` (the folder your configs live
  in; created automatically on first run). Copy the samples from the repo
  [`scripts/`](../scripts) folder to get started.
- **Hot-reload:** edit and save — the file reloads live (polled ~4×/sec). New
  files are picked up automatically; deleting a file unloads it.
- **Sandbox:** each script runs in its own environment with only
  `string`, `table`, `math`, `coroutine` and the namespaces below. `io`, `os`,
  `package`, `debug`, `load`/`dofile` are **not** available. CPU
  (per-handler wall-clock) and memory are capped; a runaway loop is aborted.
- **Modules / parent-child (`require`):** a script can pull in other Lua files with
  `require("name")`, which loads and runs `<settings>\scripts\lib\name.lua` **once**
  in the calling script's environment and returns its value. Files under
  `scripts\lib\` are **not** auto-loaded on their own — they only run when a parent
  requires them — so one "main" script can compose several children, and
  enabling/loading the main brings them all. A child's callbacks/tasks are **owned
  by the parent**, so they unload when the parent does.
  ```lua
  -- main.lua (in scripts\)
  require("radar")              -- runs scripts\lib\radar.lua (side effects)
  local util = require("util") -- scripts\lib\util.lua returns a table
  print(util.add(2, 3))
  ```
- **Persistence:** the `state` table survives hot-reloads. Everything else in a
  script is re-created on reload.

### Recently added

- **Entities:** `entity:class()`, `entity:prop(name)` / `set_prop(name, val)`
  (raw `entity_state_t` fields incl. `iuser*`/`fuser*`/`vuser*`),
  `entity:backtrack_records()`, `entity:set_velocity(v)` / `set_origin(v)`
  (local player, client-side prediction).
- **Net:** `net.hook_message(name)` + the `on_net_message(msg)` event with a
  `NetMessage` reader (`read_byte`/`short`/`long`/`float`/`coord`/`angle`/`string`).
- **usercmd:** `on_create_move`'s `cmd` now exposes `weaponselect`, `lerp_msec`,
  `lightlevel`, `impact_index`, `impact_position`.
- **Post-processing (`gfx.*`):** toggle/adjust the real engine effects
  (`gfx.set`/`get`/`reset` — bloom, DoF, godrays, colour grade incl. chromatic
  aberration, …), a screen pulse (`gfx.flash`), and **write your own GLSL pass**
  (`gfx.shader` + `gfx.uniform`).
- **3D drawing:** occlusion opts on `box_3d`/`line_3d`/`circle_3d`
  (`{ occlude = true }`), filled `box_3d_filled` / `triangle_3d_filled` /
  `polygon_3d_filled`, and `draw.framebuffer_read(x,y,w,h)`.
- **Math:** `Quaternion` and `Matrix3x4` usertypes; `engine.frame_stats()`.

See [`scripts/xhook_selftest.lua`](../scripts/xhook_selftest.lua) for a live demo
of all of the above.

---

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
player. See `scripts/osrs_chat.lua`.

```lua
callbacks.register("on_player_death", function(e)
    local who = e.attacker and e.attacker:name() or "world"
    engine.log(who .. " killed index " .. e.victim_index ..
               (e.headshot and " (HS)" or "") .. " with " .. e.weapon)
end)
```

---

## `engine.*`

> ### `engine.log(...)`  ·  `engine.print(...)`  ·  `print(...)`
> Print tab-joined arguments to the game console (prefixed `[lua]`).
> ```lua
> engine.log("health:", engine.local_player():health())
> ```

> ### `engine.notify(title, message)`
> Push an on-screen notification.
> ```lua
> engine.notify("my script", "loaded")
> ```

> ### `engine.time() -> number`
> Client time in seconds (game clock). Use for timing in-game logic.

> ### `engine.real_time() -> number` · `engine.tickcount() -> integer`
> Wall-clock seconds / milliseconds since boot (advance even paused).

> ### `engine.frame_time() -> number`
> Duration of the last frame, in seconds.
> ### `engine.frame_stats() -> table`
> `{ fps, avg_fps, min_fps, max_fps, frame_ms }` — instant FPS plus a rolling
> average / min / max over the last ~90 frames, and the last frame time in ms. For
> an in-script FPS/perf readout.

> ### `engine.is_in_game() -> boolean`
> True when connected to a server and playing (not in a menu).

> ### `engine.max_players() -> integer`
> Server player slot count.

> ### `engine.map_name() -> string`
> Current level name (e.g. `"de_dust2"`), or `""`.

> ### `engine.screen_size() -> number, number`
> Overlay width, height in pixels.
> ```lua
> local w, h = engine.screen_size()
> ```

> ### `engine.view_angles() -> Angle`
> The local player's current view angles.

> ### `engine.local_weapon() -> table | nil`
> The local player's active weapon: `{ id, name, clip, in_reload, shots_fired,
> spread, accuracy, next_attack, range, silenced }`. `nil` when not in-game.

> ### `engine.local_player() -> Entity | nil`
> A handle to the local player, or `nil` when not in game.

> ### `engine.play_weapon_anim(sequence [, duration=3.0] [, framerate=1.0])`
> Force the **viewmodel** (the gun/knife in your hands) to play `sequence` for
> `duration` seconds at `framerate`× speed — **without switching weapons**. The
> sequence is re-asserted every render frame (so the weapon's own animation can't
> overwrite it), then control is handed back when the window elapses. `sequence`
> indices are **model-dependent**; discover them with `engine.viewmodel_sequence()`.
> Vanilla `v_knife.mdl` has its idle at sequence `0` (no dedicated inspect); many
> custom knife models add an inspect at a higher index. See `scripts/knife_inspect.lua`.
> ```lua
> -- Replay the knife idle/inspect on a key press
> if input.is_key_pressed(0x46) then engine.play_weapon_anim(0, 3.0) end  -- F
> ```

> ### `engine.stop_weapon_anim()`
> Drop the override immediately; the weapon's own animation resumes.

> ### `engine.viewmodel_sequence() -> integer`
> The sequence the game is **currently** playing on the viewmodel, or `-1` when
> there's no viewmodel. Hold a weapon and watch this to find the index you want.

> ### `engine.set_viewmodel_offset(forward, right, up)`
> Shift the viewmodel along the current **view axes**, in units (same axes as the
> built-in Visuals › Viewmodel offset). Latched — re-applied every render frame
> until you change it or call `engine.reset_viewmodel()`.

> ### `engine.set_viewmodel_angles(pitch, yaw, roll)`
> Rotate the viewmodel by this many **degrees** on each axis (pitch = tilt up/down,
> yaw = swing left/right, roll = bank). Also latched. Drive it every frame (from
> `on_create_move` / `on_paint`) to build your own sway/animation:
> ```lua
> -- Bank the gun with your strafe velocity + a gentle idle bob.
> callbacks.register("on_paint", function()
>     local me = engine.local_player(); if not me then return end
>     local vel  = me:velocity()
>     local roll = util.clamp(vel:length2d() * 0.02, -8, 8)
>     local bob  = math.sin(engine.real_time() * 2) * 1.5
>     engine.set_viewmodel_angles(bob * 0.3, 0, roll)
>     engine.set_viewmodel_offset(0, 0, bob * 0.2)
> end)
> ```

> ### `engine.reset_viewmodel()`
> Clear the position + angle offset (back to the game's own placement). Overrides
> are also cleared automatically whenever scripts reload or unload.

> ### `engine.exec(command)`
> Run a console command (a newline is appended for you).
> ```lua
> engine.exec("say hello")
> ```

> ### `engine.get_cvar(name) -> number`  ·  `engine.get_cvar_string(name) -> string`
> Read a game cvar as a float / string.

> ### `engine.register_cvar(name, default [, flags])`
> Register a console variable, then read/write it with `get_cvar` / `set_cvar`.
> No-op if it already exists.
> ### `engine.add_command(name, fn)`
> Register a console command; `fn(args)` receives an array of the console
> arguments (`args[1]` is the command name).
> ```lua
> engine.add_command("lua_hello", function(args) engine.log("hi " .. (args[2] or "")) end)
> ```
> ### `engine.play_sound(file [, volume=1])`
> Play a WAV from `<settings>\sounds\` (or an absolute path), volume 0–1.
> ### `engine.say(text [, team])` · `engine.get_clipboard()` · `engine.set_clipboard(text)`
> Send a chat message (team chat if `team` is true); read/write the OS clipboard.

> ### `engine.set_cvar(name, value)`
> Set a cvar. `value` may be a number or a string.
> ```lua
> engine.set_cvar("cl_righthand", 0)
> ```

---

## `world.*`

> ### `world.get_players([enemies_only]) -> Entity[]`
> Array of connected **player** handles (excludes the local player).
> Pass `true` to return only players on the other team.
> ```lua
> for _, e in ipairs(world.get_players(true)) do
>     if e:is_alive() then engine.log(e:name()) end
> end
> ```

> ### `world.get_player(index) -> Entity | nil`
> Handle for the player at `index` (1..max_players), or `nil` if not connected.

> ### `world.world_to_screen(pos) -> number, number | nil`
> Project a world `Vector` to screen `x, y`. Returns `nil` if the point is
> behind the camera / off-screen — always check before drawing.
> ```lua
> local x, y = world.world_to_screen(ent:origin())
> if x then draw.text(x, y, draw.color(255,255,255), "here") end
> ```

> ### `world.bomb() -> table | nil`
> Planted-C4 state on de_ maps: `{ planted, defused, plant_time }` (or `nil` when
> no bomb is down). Compute the countdown from `plant_time` and `engine.time()`.

> ### `world.box_to_screen(mins, maxs) -> x, y, w, h | nil`
> Project a world AABB (e.g. from `ent:bbox()`) to a 2D screen rectangle. Returns
> `nil` unless **all** corners are on screen — the correct way to draw a 2D box
> (avoids the box exploding across the screen when a corner is behind the eye).

> ### `world.get_entities([model_substring]) -> Entity[]`
> Non-player entities read live from the engine (grenades, dropped weapons, the
> planted C4, hostages…) as full **Entity** handles — use `:origin()`,
> `:model_name()`, `:distance()`, etc. Pass a substring to filter by model name.
> ```lua
> for _, e in ipairs(world.get_entities("grenade")) do
>     local x, y = world.world_to_screen(e:origin())
>     if x then draw.circle_filled(x, y, 3, draw.color(255,210,90)) end
> end
> ```

### World traces

Traces run through the engine event API — the same path the aim/autowall code
uses — and push/pop the engine's player-prediction states internally, so they
are safe to call from either `on_paint` or `on_create_move`. Points are `Vector`s.

> ### `world.trace_line(start, end [, mask [, ignore]]) -> trace`
> Point trace from `start` to `end`. `mask` is one of the `world.MASK_*`
> constants (default `MASK_NORMAL` — hits world + players). `ignore` is an
> `Entity` or integer index to skip (default: skip nothing).

> ### `world.trace_hull(start, end [, hull [, mask [, ignore]]]) -> trace`
> Like `trace_line` but sweeps a hull. `hull` is `world.HULL_POINT` /
> `HULL_REGULAR` (default) / `HULL_DUCKED`.

> ### `world.trace_bullet(from, to) -> number`
> **Autowall** — the damage a bullet from your current weapon would deal landing
> at `to`, accounting for wall penetration and range falloff (0 = fully blocked).
> ```lua
> if world.trace_bullet(me:eye_position(), enemy:head_position()) > 30 then ... end
> ```

> ### `world.is_visible(from, to [, ignore]) -> boolean, Vector`
> Convenience line-of-sight check (glass-ignoring point trace). Returns whether
> the path is clear plus the trace end position. Pass the target's `Entity`/index
> as `ignore` so its own body doesn't count as blocking.
> ```lua
> local me = engine.local_player()
> for _, e in ipairs(world.get_players(true)) do
>     if e:is_alive() and world.is_visible(me:eye_position(), e:head_position(), e) then
>         local x, y = world.world_to_screen(e:head_position())
>         if x then draw.text(x, y, draw.color(0,255,0), "visible", draw.flags.CENTER_X) end
>     end
> end
> ```
>
> **Trace result table** (`world.trace_line` / `trace_hull`):
> | field | type | meaning |
> |-------|------|---------|
> | `fraction` | number | 0..1 fraction of the path travelled (1 = nothing hit) |
> | `hit` | boolean | `fraction < 1` |
> | `endpos` | Vector | final position |
> | `normal` | Vector | surface normal at impact |
> | `plane_dist` | number | impact plane distance |
> | `entity` | Entity? | player hit, if any |
> | `ent_index` | integer | raw entity index from the trace |
> | `hitgroup` | integer | hitgroup struck |
> | `allsolid` / `startsolid` | boolean | started inside solid |
> | `in_open` / `in_water` | boolean | end-point medium |
>
> **Constants:** `world.HULL_POINT` · `HULL_REGULAR` · `HULL_DUCKED` ·
> `world.MASK_NORMAL` · `MASK_WORLD` · `MASK_STUDIO_BOX` · `MASK_STUDIO_IGNORE` · `MASK_GLASS_IGNORE`

---

## `draw.*` (valid inside `on_paint`)

Colours are packed integers from `draw.color`.

> ### `draw.color(r, g, b [, a=255]) -> integer`
> Build a colour (channels 0–255).

> ### `draw.color_hsv(h, s, v [, a=1]) -> integer`
> Build a colour from HSV (all 0–1). Handy for rainbows:
> ```lua
> local hue = (engine.real_time() * 0.2) % 1
> draw.text(x, y, draw.color_hsv(hue, 0.8, 1.0), "rainbow")
> ```

> ### `draw.text(x, y, color, text [, flags])`
> `flags` from `draw.flags`: `SHADOW` (default), `OUTLINE`, `CENTER_X`,
> `CENTER_Y`, `NONE` — combine with `|`.
> ```lua
> draw.text(x, y, draw.color(255,255,255), "hi", draw.flags.CENTER_X | draw.flags.SHADOW)
> ```

> ### `draw.line(x1, y1, x2, y2, color)`
> ### `draw.rect(x, y, w, h, color [, rounding=0] [, thickness=1])` — outline
> ### `draw.rect_filled(x, y, w, h, color [, rounding=0])`
> ### `draw.circle(x, y, radius, color [, segments=24] [, thickness=1])`
> ### `draw.circle_filled(x, y, radius, color [, segments=24])`
> ### `draw.triangle(x1,y1, x2,y2, x3,y3, color [, thickness=1])`
> ### `draw.triangle_filled(x1,y1, x2,y2, x3,y3, color)`
> ### `draw.line_3d(a, b, color [, thickness=1] [, opts])`
> Draw a line between two **world** `Vector`s; skipped if either end is
> off-screen. Great for tracers / bounding volumes. `opts` — see the occlusion
> note under `draw.box_3d` (tested at the line midpoint).
> ### `draw.circle_3d(center, radius, color [, segments=32] [, thickness=1] [, opts])`
> A world-space horizontal ring (range / FOV indicators on the ground). `opts` —
> occlusion-tested at the ring centre.
> ### `draw.text_3d(worldpos, color, text [, flags])`
> `world_to_screen` + text in one call; skipped when off-screen.
> ### `draw.polyline({x1,y1, x2,y2, ...}, color [, thickness=1] [, closed=false])`
> Connect a flat list of screen points into a single stroked path with round
> joins — **one GPU call** for the whole line, not one per segment. Pass
> `closed = true` to join the last point back to the first. Ideal for curved
> trails / slider bodies where faking the line with many `circle_filled` calls
> would tank the frame rate. Ends are flat (butt) caps; add a `circle_filled` at
> each end if you want rounded caps.
> ### `draw.polygon({x1,y1, ...}, color [, thickness=1])` · `draw.polygon_filled({...}, color)`
> Closed outline / convex fill of a point list (points clockwise for the fill).
> ### `draw.arc(x, y, radius, start_deg, end_deg, color [, thickness=1] [, segments=32])`
> Circular arc outline (progress rings, HP arcs). `draw.pie(...)` fills the sector.
> ### `draw.capsule(x1, y1, x2, y2, radius, color [, segments=16])`
> Filled pill/capsule: two round caps of `radius` joined by a filled body between
> the two points. A cheap rounded thick line — good for slider segments, sci-fi
> bars, or any stadium shape. `segments` controls the smoothness of each cap.
> ### `draw.push_clip(x, y, w, h)` · `draw.pop_clip()`
> Clip subsequent draws to a rectangle (mask overflow). Always pair them. If you
> forget a `pop_clip` (or leave a `Begin`/style push open), the layer now unwinds
> it for you after `on_paint` returns, so a leaked clip no longer clips away the
> menu/ESP/HUD drawn later in the frame — but keep them paired: the safety net
> only covers the boundary between your handler and the rest of the frame, not
> imbalances *within* your own drawing.
> ### `draw.blur(x, y, w, h [, rounding] [, alpha] [, dim])`
> Frosted-glass backdrop of the game behind a rect (the menu's frost). Needs HUD
> blur enabled in Visuals; captures on the next frame, so the first draw is empty.
> ### `draw.glow(x, y, w, h, color [, size=14])`
> Soft glow/shadow around a **rect**.
> ### `draw.glow_circle(cx, cy, radius, color [, size=14])`
> Soft glow around a **round** shape (radar rim, orb). Use this instead of
> `draw.glow` for circles — the rect glow can't close into a circle, so it leaves
> a squarish halo. Draws concentric rings that fade outward and match the circle.
> ### `draw.rgba(color) -> r, g, b, a`  ·  `draw.lerp_color(a, b, t) -> color`
> Unpack a packed colour into channels; interpolate two colours.
> ### `draw.box_3d(mins, maxs, color [, thickness=1] [, opts])`
> Draw the 12 edges of a world-space AABB (two corner `Vector`s). Each edge is
> drawn only when both its endpoints are on screen. Ideal 3D player boxes:
> ```lua
> local o = ent:origin()
> draw.box_3d(o + Vector(-16,-16,0), o + Vector(16,16,72), draw.color(255,0,0))
> ```
> **Occlusion** — pass an `opts` table to auto-dim/hide when the box centre is
> behind world geometry from your eye (depth-correct ESP, no render hook):
> `{ occlude = true }` dims to 30% alpha when occluded; `occlude_alpha = 0.5`
> sets the fade; `hide_occluded = true` skips it entirely. Backed by the same
> trace as `world.is_visible`. Works on `line_3d` (midpoint) and `circle_3d`
> (centre) too.
> ```lua
> draw.box_3d(mn, mx, draw.color(0,255,120), 1, { occlude = true, occlude_alpha = 0.25 })
> ```
> ### `draw.box_3d_filled(mins, maxs, color [, opts])`
> Solid world-space AABB — the 6 faces filled (use a translucent colour). Same
> `opts` occlusion table as `draw.box_3d`.
> ### `draw.triangle_3d_filled(a, b, c, color)`
> Filled world-space triangle from three world `Vector`s (skipped unless all three
> are on screen). Building block for filled 3D shapes.
> ### `draw.polygon_3d_filled({v1, v2, ...}, color)`
> Filled convex world-space polygon from a list of world `Vector`s (triangle fan);
> skipped if any vertex is off-screen.
> ### `draw.framebuffer_read(x, y, w, h) -> r, g, b`
> Average colour (0..255) of the **rendered frame** in a screen rect — for dynamic
> theming (tint your UI to what's on screen). It's a GPU→CPU readback so it stalls
> the pipeline: keep the rect modest, avoid calling it every frame on a huge area.
> Rects over 200px/side sample the central 200×200 window.
> ```lua
> local r, g, b = draw.framebuffer_read(cx - 40, cy - 40, 80, 80)
> draw.rect_filled(10, 10, 60, 60, draw.color(r, g, b))   -- swatch of the crosshair area
> ```
> ### `draw.rect_gradient(x, y, w, h, col_tl, col_tr, col_br, col_bl)`
> Four-corner gradient fill. Repeat a colour on adjacent corners for a simple
> vertical/horizontal fade.
> ### `draw.gradient_text(x, y, text, col_start, col_end [, flags])`
> Text whose colour fades from `col_start` to `col_end` across its length.
> ### `draw.icon(name_or_id, x, y, size [, color])`
> Draw a built-in menu icon as a `size`×`size` square. `name` is a key of
> `draw.icons` (`"rage"`, `"legit"`, `"visuals"`, `"kreedz"`, `"misc"`,
> `"configs"`, `"console"`, `"defuser"`, `"c4"`, `"headshot"`, `"noscope"`,
> `"penetrate"`, `"blind"`, `"smoke"`, `"inair"`); most are white silhouettes, so
> pass a `color` to tint. `draw.icons` is a `{name = id}` table for discovery.
> ### `draw.weapon_icon(weapon_id, x, y, size [, color])`
> Draw the weapon icon for a game weapon id (e.g. `engine.local_weapon().id`).
> ### `draw.load_image(filename) -> Image | nil`
> Load a PNG into a texture (deduped by path). `filename` is relative to
> `<settings>\scripts\images\` or an absolute path; `nil` if missing. **Call from
> an `on_paint` handler** (texture creation needs the render thread). The handle
> has `:width()`, `:height()`, `:size()`.
> ### `draw.image(image, x, y [, w, h] [, color])`
> Draw a loaded `Image`; `w`/`h` default to its native size, `color` tints it.
> ### `draw.image_rotated(image, cx, cy, w, h, angle_deg [, color])`
> Draw a loaded `Image` rotated `angle_deg` degrees (clockwise) around its centre
> `(cx, cy)`. `w`/`h` are the unrotated width/height; `color` tints it. Use for
> spinning sprites — slider balls, spinner discs, radar sweeps.
> ### `draw.create_font(file, size) -> Font`
> Load a TTF from `<settings>\scripts\fonts\` (or an absolute path) at `size` px.
> The atlas rebuilds on the next frame, so the font is ready ~1 frame later
> (`font:ready()`); `font:size()` returns its pixel size.
> ### `draw.text_font(font, x, y, color, text [, size])`
> Draw text in a custom font (falls back to the default until the font is ready).
> ```lua
> local logo
> callbacks.register("on_paint", function()
>     logo = logo or draw.load_image("logo.png")   -- lazy, render thread
>     if logo then draw.image(logo, 20, 20, 64, 64) end
> end)
> ```
> ### `draw.text_size(text) -> number, number`
> Measure `text`, returning width, height in pixels.

---

## `input.*`

> ### `input.is_key_down(vk) -> boolean`
> True while a virtual-key (Win32 VK code) is held.

> ### `input.is_key_pressed(vk) -> boolean`
> True on the frame the key transitions up→down (edge). Great for toggles.
> ```lua
> if input.is_key_pressed(0x2D) then enabled = not enabled end  -- INSERT
> ```

> ### `input.cursor_position() -> number, number`
> Mouse `x, y` in overlay space.

> ### `input.mouse_delta() -> dx, dy`  ·  `input.scroll() -> number`
> Mouse movement since last frame; wheel this frame.

## `network.*` — async HTTP

`network.get(url, callback)` · `network.post(url, body, callback [, content_type])`
— non-blocking HTTP on a worker thread; the callback fires on the main thread the
frame the response arrives: `callback(ok, status, body)`.
```lua
network.get("https://api.github.com", function(ok, status, body)
    if ok then engine.log("status " .. status .. ", " .. #body .. " bytes") end
end)
```

## `gfx.*` — engine post-processing

Drive the **real** GLSL post-process suite (the same passes as Visuals →
Post-process) from Lua — not faked with overlay draws.

- `gfx.set(name, value)` — enable/adjust an effect. `value` is a boolean (for
  toggles) or a number (for parameters). The original is snapshotted on first
  touch and **restored automatically when the script unloads** (or via
  `gfx.reset()`), so a script can never leave an effect stuck on.
- `gfx.get(name) -> boolean|number`.
- `gfx.reset()` — restore everything this session's scripts changed.
- `gfx.flash(r, g, b [, strength=0.6])` — fire-and-forget GPU screen pulse:
  coloured vignette + expanding shockwave ring + chromatic pulse. `r/g/b` 0..255.
- `gfx.shader(glsl_frag)` — **write your own shader**: install a GLSL 120
  fragment shader as the final post-process pass (heat-haze, glitch, CRT, whatever).
  Pass `nil`/`""` to remove it (also auto-removed on unload). It samples the fully
  composited scene. Interface provided:
  `uniform sampler2D u_tex;` (uv = `gl_TexCoord[0].xy`), `uniform vec2 u_resolution;`,
  `uniform float u_time;` (seconds). Compiled on the render thread; a compile error
  just no-ops (enable `pp_debug` to see the log).
- `gfx.uniform(name, a [, b [, c]])` — set a custom `float` / `vec2` / `vec3`
  uniform on your shader.

```lua
gfx.shader([[
  #version 120
  uniform sampler2D u_tex; uniform float u_time; uniform float u_amount;
  void main(){
    vec2 uv = gl_TexCoord[0].xy;
    uv.x += sin(uv.y*28.0 + u_time*3.0) * 0.004 * u_amount;  // heat-haze warp
    gl_FragColor = texture2D(u_tex, uv);
  }
]])
gfx.uniform("u_amount", 1.0)
```

Names — toggles: `bloom`, `dof`, `motionblur`, `ssao`, `ssr`, `godrays`, `fxaa`,
`fog`, `outline`, `grade`, `tonemap`. Parameters: `bloom_intensity`,
`bloom_threshold`, `dof_strength`, `motionblur_amount`, `godrays_intensity`, and
the colour-grade set `exposure`, `contrast`, `saturation`, `temperature`,
`vignette`, `chromatic` (aberration), `grain`, `sharpen` (these need `grade` on).

> These are the **same cvars** the Visuals menu uses, and effects self-disable if
> the GPU lacks the needed extensions. DoF/SSAO/SSR/godrays need the depth buffer.

```lua
gfx.set("bloom", true); gfx.set("bloom_intensity", 1.2)
gfx.set("grade", true); gfx.set("chromatic", 0.35); gfx.set("vignette", 0.4)
gfx.flash(0, 255, 200, 0.7)   -- cyan screen pulse
```

## `net.*` — user-message hooks

Read GoldSrc user messages (server → client) from Lua.

- `net.hook_message(name) -> bool` — start firing `on_net_message` whenever the
  named user message arrives (e.g. `"DeathMsg"`, `"SayText"`, `"Health"`,
  `"TextMsg"`, `"ScreenShake"`, `"Radar"`, …). Returns `false` if the message
  isn't registered yet — user messages only exist once you're **in a game**, so
  hook on first use / retry until it returns `true`. Layers on top of any existing
  handler (the cheat's own or the engine's), which still runs after yours.
- `net.unhook_message(name)` — stop. All hooks are auto-removed on script
  reload/unload.

**`msg`** (the `on_net_message` payload) is a **NetMessage reader**, valid **only
inside the callback** (a stashed handle errors afterwards). Methods:
`:name()`, `:read_char()`, `:read_byte()`, `:read_short()`, `:read_word()`,
`:read_long()`, `:read_float()`, `:read_coord()`, `:read_angle()`,
`:read_string()`, `:eof()`. Each reader has its own read cursor, so reading here
never disturbs the game's own parse of the same message.

```lua
net.hook_message("DeathMsg")
callbacks.register("on_net_message", function(msg)
    if msg:name() ~= "DeathMsg" then return end
    local killer, victim = msg:read_byte(), msg:read_byte()
    local headshot, weapon = msg:read_byte() ~= 0, msg:read_string()
    engine.log(("%d killed %d with %s%s"):format(
        killer, victim, weapon, headshot and " (HS)" or ""))
end)
```

## `db.*` — persistent key-value store

`db.set(key, value)`, `db.get(key [, default])`, `db.delete(key)`, `db.all()`.
Values are JSON-serialisable (tables/strings/numbers/booleans), persisted to a
`db.json` in **your script's own folder** (`<settings>\scripts\data\<script>\`) —
per-script and isolated, survives reloads and restarts.
```lua
db.set("kills", (db.get("kills", 0)) + 1)
```

---

## `features.*` — drive xhook's own features

Read and write the client's built-in feature values (the same bool/int/float
targets the menu and key-binds control) by their stable **registry id**. Writes
go straight to the feature; use this to let a script toggle the ragebot, flip
thirdperson, change a distance, etc.

> ### `features.list([category]) -> string[]`
> Every registry id, optionally filtered to one `category` (e.g. `"Ragebot"`,
> `"Kreedz"`, `"Visuals"`). Ids look like `"kreedz.bhop"`, `"rage.forcebody"`,
> `"visuals.tp_dist"`.

> ### `features.info(id) -> table | nil`
> `{ name, category, type, min, max, items? }`. `type` is
> `"bool"`/`"int"`/`"float"`/`"multi"`; `items` (option labels) is present only
> for dropdown-backed values.

> ### `features.get(id) -> boolean | number | boolean[] | nil`
> Current value. Booleans → `boolean`, int/float → `number`, `multi` → an array
> of booleans. `nil` if the id is unknown.

> ### `features.set(id, value) -> boolean`
> Write `value` (bool→boolean, int/float→number clamped to `min`/`max`,
> multi→array of booleans). Returns `true` if it was written; raises on an
> unknown id.
> ```lua
> -- Enable bhop while a key is held, restore when released.
> callbacks.register("on_create_move", function()
>     features.set("kreedz.bhop", input.is_key_down(0x20)) -- SPACE
> end)
> ```

> ⚠️ Values are shared with the menu/binds and read on the game thread. A script
> and a key-bind fighting over the same id will stomp each other each frame.

---

## `ui.*` — your own menu controls

Declare controls once; they render as a **sub-tab under the Scripting menu tab**
and keep their value. Read a control anywhere with `:get()`. Cards auto-size to
the number of controls in them, laid out in two balanced columns.

Control values **persist** to `<settings>\scripts\config\<script>.cfg` — they
survive reloads and restarts automatically (no API needed). Each script also has
an **enable/disable** toggle in the *Scripts* sub-tab; a disabled script stays on
disk but doesn't run.

> ### `ui.tab(name) -> Tab`
> Create (or fetch) a control tab shown in the Scripting sidebar.

> ### `Tab:card(name) -> Card`
> A titled group-box on that tab.

> ### Card control builders — each returns a **Control** handle
> | Method | Returns via `:get()` |
> |--------|----------------------|
> | `card:checkbox(label, default [, on_change])` | boolean |
> | `card:slider_int(label, default, min, max [, on_change])` | integer |
> | `card:slider_float(label, default, min, max [, on_change])` | number |
> | `card:combo(label, {items}, default_index [, on_change])` | integer (1-based) |
> | `card:color(label, {r,g,b,a} [, on_change])` | `{r,g,b,a}` 0–255 |
> | `card:button(label [, on_click])` | — (fires `on_click`) |
> | `card:keybind(label [, default_vk] [, mode] [, on_change])` | integer vk |
> | `card:hotkey(label [, default_vk] [, on_change])` | integer vk (toggle keybind) |
> | `card:multiselect(label, {items} [, {default_bools}] [, on_change])` | boolean[] |
> | `card:text_input(label [, default] [, on_change])` | string |
> | `card:label(text)` · `card:separator()` | — |
>
> `on_change` / `on_click` receive the new value (button gets none) and fire when
> the user edits the control in-menu. **Keybind** `mode` is `"hold"` (default),
> `"toggle"`, or `"always"`; click the button then press a key to bind (Esc
> cancels, right-click clears; mouse4/5 work). `default_vk` is a Win32 virtual-key
> code (e.g. `0x20` = Space).

> ### `Control:get()` · `:set(value)` · `:active()` · `:set_visible(bool)` · `:label()`
> Read / write the value from anywhere (e.g. in `on_paint` / `on_create_move`).
> `:active()` is for **keybinds** — true while the key is held (or toggled on).
> `:set_visible(false)` hides a control and reflows its card (for conditional UI).
> A handle kept across a reload becomes inert (`:get()` returns `nil`).
> ```lua
> local key = card:keybind("Aim key", 0x06, "hold")   -- mouse5
> callbacks.register("on_create_move", function(cmd)
>     if key:active() then do_aim(cmd) end
> end)
> ```
> ```lua
> local tab = ui.tab("My Script")
> local c   = tab:card("Aimbot")
> local en  = c:checkbox("Enabled", false)
> local fov = c:slider_int("FOV", 90, 0, 180)
> callbacks.register("on_create_move", function(cmd)
>     if en:get() then aim(cmd, fov:get()) end
> end)
> ```

---

## `callbacks.*`

> ### `callbacks.register(event, fn)`
> Register `fn` for `event` (see the events table). Registrations are owned by
> the script and dropped automatically on reload/unload.

> ### `callbacks.unregister(event, fn) -> integer`
> Remove a previously-registered handler; returns how many were removed.

---

## `task.*` (coroutine scheduler)

> ### `task.spawn(fn)`
> Run `fn` as a cooperative coroutine. It runs immediately until it finishes or
> calls `task.wait`.

> ### `task.wait(seconds)`
> **Only valid inside a `task.spawn` body.** Yields; the scheduler resumes the
> task ~`seconds` later. Loops that sleep won't block a frame.
> ```lua
> task.spawn(function()
>     while true do engine.log("tick"); task.wait(1.0) end
> end)
> ```

> ### `task.defer(fn)`
> Alias of `task.spawn` — run a one-shot coroutine.

---

## Usertypes

Vector also has `:to_screen() -> x,y|nil` (world→screen) and `:rotate_yaw(deg)`
(rotate about Z).

### `Vector(x, y, z)`
Fields `.x .y .z` (read/write). Operators `+ - * /` (scalar or componentwise),
unary `-`, `==`, `tostring`.
Methods: `:length()`, `:length2d()`, `:dot(v)`, `:distance(v)`,
`:normalized()`, `:cross(v)`, `:unpack() -> x,y,z`, `:angle_between(v)`,
`:lerp(to, t)`, `:to_angles() -> Angle` (direction → view angles).
```lua
local a = Vector(1, 0, 0)
local b = a + Vector(0, 2, 0)
engine.log(b:length())        -- 2.236...
```

### `Angle(pitch, yaw, roll)`
Fields `.p/.pitch/.x`, `.y/.yaw`, `.r/.roll/.z` (read/write).
Methods: `:forward() -> Vector`, `:vectors() -> forward, right, up`.
```lua
local fwd = engine.view_angles():forward()
```

### `Quaternion(x, y, z, w)`
Fields `.x/.y/.z/.w` (read/write). Build from Euler with
`Quaternion.from_angles(Angle)`. Methods: `:length()`, `:normalize()`,
`:conjugate()`, `:dot(q)`, `:mul(q)` (also `q1 * q2`), `:rotate(Vector) -> Vector`,
`:to_angles() -> Angle`, `:slerp(q, t) -> Quaternion`.
```lua
local q = Quaternion.from_angles(engine.view_angles())
local fwd = q:rotate(Vector(1, 0, 0))        -- same as the angle's forward
```

### `Matrix3x4(...)`
A 3×4 transform (rotation + translation). Construct from 12 row-major numbers, no
args for identity, or `Matrix3x4.from_angles(Angle [, origin])`. Methods:
`:get(r,c)` / `:set(r,c,v)` (r 0..2, c 0..3), `:origin() -> Vector`,
`:forward()` / `:right()` / `:up() -> Vector`, `:transform(Vector) -> Vector`
(rotate + translate a point), `:rotate(Vector) -> Vector` (rotation only). Pairs
naturally with `entity:bone_matrix(i)` (which returns the 12 numbers).
```lua
local m = Matrix3x4.from_angles(ent:angles(), ent:origin())
local muzzle = m:transform(Vector(20, 0, 4))  -- a point in the entity's local space
```

### `Entity` (a validated handle)
Never holds a raw pointer — every accessor re-validates, so a handle kept across
a death/disconnect just reports invalid instead of crashing.
Methods: `:is_valid()`, `:index()`, `:name()`, `:origin() -> Vector`,
`:eye_position() -> Vector`, `:velocity() -> Vector`, `:angles() -> Angle`,
`:health()`, `:armor()`, `:team()`, `:money()`, `:speed()`, `:fov()`,
`:sequence()`, `:model_name()`, `:is_alive()`, `:is_player()`, `:is_local()`,
`:is_enemy()`, `:in_pvs()`, `:is_dormant()`, `:is_scoped()`, `:on_ground()`,
`:is_ducking()`, `:in_water()`, `:on_ladder()`, `:has_c4()`, `:has_defuser()`,
`:has_shield()`, `:distance_to(vec)`, `:distance()` (to local player),
`:head_position() -> Vector`, `:hitbox_position(i) -> Vector` (0-based hitbox
index), `:bbox() -> mins, maxs` (two world `Vector`s — a crouch-aware AABB ready
for `draw.box_3d`, matching the native ESP box), `:prev_origin() -> Vector`,
`:is_dead()`, `:class_id()`, `:fall_velocity()`, `:gait_yaw()`,
`:sequence_frame()`, `:observer_state()`, `:weapon_model()`,
`:frags()`, `:deaths()`, `:ping()` (scoreboard / latency).

**Type & raw props** (Phase 01):
- `:class() -> string` — coarse type from the model name (GoldSrc has no
  client-side classname): `"player"`, `"weapon"`, `"grenade"`, `"c4"`,
  `"hostage"`, `"chicken"`, `"brush"` (a `*N` BSP submodel — func_door/wall/…),
  `"viewmodel"`, `"world"`, or `"unknown"`.
- `:prop(name) -> number | Vector` — read a field of the engine entity's
  `curstate` (`entity_state_t`). Raises on an unknown name. Useful names:
  `rendermode`, `renderamt`, `renderfx`, `rendercolor_r/g/b`, `effects`,
  `movetype`, `body`, `skin`, `sequence`, `frame`, `colormap`, `owner`,
  `aiment`, `weaponmodel`, `gaitsequence`, `origin`/`angles`/`velocity`/`mins`/
  `maxs` (Vectors), and the mod scratch fields `iuser1..4`, `fuser1..4`,
  `vuser1..4` (where CS servers stash custom per-entity data — the closest
  GoldSrc equivalent of Source netvars).
- `:set_prop(name, value)` — write the client `curstate` copy. The engine
  overwrites most fields on the next server update, so this is mainly useful for
  local render fields (`rendermode`/`renderamt`/`rendercolor_*`) between updates.

```lua
local e = world.get_entities("grenade")[1]
if e and e:class() == "grenade" then
    print("owner slot:", e:prop("owner"), "iuser1:", e:prop("iuser1"))
end
```

**Backtrack** (Phase 04):
- `:backtrack_records() -> table[]` — the quantised rewind candidates the
  ragebot / backtrack-chams use for this player. Each entry is
  `{ origin = Vector, time = number (server animtime), latency = number (fake-lag
  seconds this record would request) }`. Read-only view of the existing lag-comp
  history; empty for non-players / when no history exists. Handy for drawing
  backtrack points (`draw.box_3d` at each `origin`).

**Local movement override** (Phase 04, local player only):
- `:set_velocity(Vector)` / `:set_origin(Vector)` — queue a **client-side
  prediction** velocity/teleport applied after movement runs. ⚠️ Prediction only:
  the server re-simulates from your usercmd, so it **rubberbands** on lag-comp
  servers — for real, server-respected movement drive `cmd.forwardmove` /
  `sidemove` / `buttons` from `on_create_move` (Phase 02) instead. Effects last one
  tick, so call every tick for a continuous hold. Errors on a non-local entity.

```lua
-- client-side "speed" feel: re-apply a boosted horizontal velocity each tick
callbacks.register("on_create_move", function()
    local me = engine.local_player()
    if not (me and me:is_alive()) then return end
    local v = me:velocity()
    me:set_velocity(Vector(v.x * 1.5, v.y * 1.5, v.z))
end)
```

**Per-hitbox geometry** (`i` = 0..hitbox count): `:hitbox_min(i)` / `:hitbox_max(i)`
(world AABB), `:obb_min(i)` / `:obb_max(i)` (oriented box), `:hitbox_points(i)`
(`Vector[]` multipoint samples), `:bone_position(i)` (`Vector`), `:bone_matrix(i)`
(12 numbers, row-major 3×4).

> ### `engine.hit_stats() -> table`
> `{ hits, misses, last_damage, last_hitgroup, last_hitbox, last_headshot,
> last_victim, last_weapon_id, last_time }` from the hit-registration tracker.

**Dormant** enemies (out of your PVS) are still returned by `world.get_players`
with their **last-known** position/health; check `:is_dormant()` and draw them
dimmed. `:bbox()` falls back to the collision hull. Use `:dormant_time()`
(seconds since last seen) to **fade out / drop** stale dormant players instead of
showing them forever.

## `util.*`

`util.clamp(v, lo, hi)`, `util.lerp(a, b, t)`, `util.round(v)`, `util.sign(v)`,
`util.map(v, in_lo, in_hi, out_lo, out_hi)`, `util.normalize_yaw(deg)` (wrap to
±180), `util.approach(cur, target, delta)` (step toward a value — for smoothing).

`Angle` also has `:normalize()` (wrap to ±180), `:delta(other)` (normalised
difference — for aim deltas), and `:length()` (pitch/yaw magnitude).

## `bit.*`  ·  `files.*`

`bit.band/bor/bxor(...)`, `bit.bnot(x)`, `bit.lshift/rshift/arshift(x, n)`.

`files.read(name) -> string|nil`, `files.write(name, data) -> bool`,
`files.exists(name) -> bool`, `files.delete(name)`, `files.mkdir(dir) -> bool`,
`files.list(dir) -> { {name, dir=bool}, ... }`, `files.script_dir() -> string`.

**Each script gets its own private folder** — `files.*` is sandboxed to
`<settings>\scripts\data\<your-script>\`, so two scripts writing the same
filename never collide. Subfolders work (`files.write("saves/slot1.json", ...)`
auto-creates parents); `files.list(".")` lists your folder; `files.list("saves")`
lists a subdir. Absolute paths, drive letters, and `..` are rejected.
`files.script_dir()` returns your folder's name (informational — you don't need
it, paths are already scoped). Your `db.*` store lives here too, so it's
per-script as well.

`json.encode(value) -> string` · `json.decode(string) -> value` — round-trip Lua
tables/strings/numbers/booleans. Pair with `files.*` for persistent storage:
```lua
files.write("save.json", json.encode(state))          -- persist
local data = json.decode(files.read("save.json") or "{}")  -- restore
```

---

## `memory.*` — raw memory (trusted "Native" tier)

> **Locked by default.** This namespace only exists after you turn on **Raw memory
> access** in *Scripting → Scripts* (the toggle next to *Reload all*). It breaks the
> sandbox — raw peek/poke and native calls — so only turn it on for scripts you
> trust. Toggling reloads all scripts. In a normal (sandboxed) script `memory` is
> `nil`, so guard with `if not memory then return end`.

x86 build: addresses are 32-bit and fit losslessly in a Lua integer. Every read /
write is **SEH-guarded** — a bad address yields `nil` (reads) or `false` (writes)
instead of crashing the game.

| Call | Returns |
|---|---|
| `memory.module_base([name])` | base address of a loaded module (`name` omitted → the game exe); `0` if not loaded |
| `memory.module_size([name])` | `SizeOfImage` of the module, `0` if not loaded |
| `memory.find_pattern(module, "AA BB ?? CC")` | first matching address, or `nil` (`?`/`??` = wildcard byte) |
| `memory.read_int/uint/short/byte/float/double/ptr(addr)` | typed value, or `nil` on fault |
| `memory.read_bytes(addr, n)` | `n`-byte string (≤ 1 MiB), or `nil` |
| `memory.read_string(addr[, max])` | string up to a NUL or `max` bytes (default 256) |
| `memory.write_int/short/byte/float/double(addr, v)` | `true` on success, `false` on fault |
| `memory.write_bytes(addr, str)` | `true`/`false` |
| `memory.alloc(size)` | RWX allocation address, or `nil` |
| `memory.protect(addr, size, prot)` | previous protection, or `nil` (`prot` = a `memory.PAGE_*` constant) |

Page constants: `memory.PAGE_NOACCESS / PAGE_READONLY / PAGE_READWRITE / PAGE_EXECUTE
/ PAGE_EXECUTE_READ / PAGE_EXECUTE_READWRITE`.

```lua
if not memory then return end
local base = memory.module_base("hw.dll")
local addr = memory.find_pattern("hw.dll", "55 8B EC 83 EC ??")
if addr then engine.log(string.format("found @ 0x%08X", addr)) end
```

### Writing memory safely

Reads are cheap and forgiving; writes need three habits so a patch is *effective*
and *reversible*:

1. **Unlock the page first.** Code/constants live in read-only sections (`.text`,
   `.rdata`), so a raw `write_*` there just returns `false`. Call
   `memory.protect(addr, size, memory.PAGE_EXECUTE_READWRITE)` first, and keep the
   returned old protection to put back later.
2. **Verify before you write.** Read the value and confirm it's what you expect
   (e.g. a known constant) before flipping protection or writing — never patch an
   address you haven't validated on *this* build.
3. **Restore on unload.** Save the original bytes (`memory.read_bytes`) and the old
   protection, then undo them from an `on_script_unload` handler so a hot-reload or
   disable leaves the engine exactly as it was.

```lua
-- Snapshot → verify → unlock → write → restore.
local A = memory.find_pattern("hw.dll", "...")
local orig_bytes = memory.read_bytes(A, 8)
local orig_prot  = memory.protect(A, 8, memory.PAGE_EXECUTE_READWRITE)
memory.write_double(A, newValue)

callbacks.register("on_script_unload", function()
    memory.write_bytes(A, orig_bytes)      -- exact restore
    memory.protect(A, 8, orig_prot)        -- re-lock the page
end)
```

> **`find_pattern` returns only the *first* match.** Modules often carry several
> identical copies of a constant, and only one is referenced by code — patching a
> dead copy silently does nothing. When it matters, snapshot the module with
> `read_bytes` (≤ 1 MiB/call, so chunk it), enumerate every copy yourself, and pick
> the one whose absolute address appears most often as a disp32 reference.
> `speed_control.lua` does exactly this.

### Runnable demos

| Script | Shows |
|---|---|
| [`native_probe.lua`](../scripts/native_probe.lua) | module base/size + a signature scan, logged to console |
| [`memory_watch.lua`](../scripts/memory_watch.lua) | read-only inspector: live-read any address/signature as int/float/hex/string |
| [`speed_control.lua`](../scripts/speed_control.lua) | a **write** demo — reversible speedhack that patches the `1000.0` time constant, with the referenced-copy scan above |

---

## The `state` table

A per-script table that **persists across hot-reloads**. Stash tuned values or
counters here so editing the file doesn't reset them.
```lua
state.count = (state.count or 0) + 1
```

## `audio.*` (streaming radio + sound)

Streams internet radio (http/https mp3/aac) or plays local files, on a background
decode thread (Windows Media Foundation → waveOut). Handles are integers; `0` =
failed / "all".

> ### `audio.stream(url) -> handle` · `audio.play_file(path) -> handle`
> Open + start playing a stream URL or a local/downloaded file. Returns a handle.
> ### `audio.stop([handle])` · `audio.pause(handle, bool)`
> Stop one handle (or **all** if omitted); pause/resume.
> ### `audio.set_volume(0..1)` · `audio.volume() -> number`
> Global output volume.
> ### `audio.is_playing(handle) -> bool` · `audio.position(handle) -> seconds`
> Playing state and playback position (for a progress bar).
> ### `audio.tags(handle) -> { title, artist, bitrate }`
> Metadata table. `bitrate` (kbps) is populated; `title`/`artist` are best-effort
> (live Shoutcast/ICY now-playing is a follow-up, currently empty).
> ### `audio.spectrum(handle [, bins=32]) -> number[]`
> FFT magnitudes (0..1) of the current audio — drive an EQ/visualizer.
> ### `audio.load_sound(path) -> id` · `audio.play_sound(path_or_id [, gain=1])`
> One-shot SFX: `load_sound` caches a file and returns an id; `play_sound` fires a
> voice from a path or a cached id (cheap to spam for hit sounds / UI clicks).
> ```lua
> local h = audio.stream("http://example.com/stream.mp3")
> audio.set_volume(0.5)
> callbacks.register("on_paint", function()
>   for i, m in ipairs(audio.spectrum(h, 24)) do
>     draw.rect_filled(20 + i*6, 200 - m*80, 4, m*80, draw.color(0,200,255))
>   end
> end)
> ```

## `input.*` / `engine.*` additions

> ### `input.is_menu_open() -> bool`
> Is the cheat menu open (cursor free)? Gate clickable in-world HUD on this so
> clicks don't hijack gameplay.
> ### `input.is_mouse_down(button) -> bool` · `input.mouse_clicked(button) -> bool`
> `button`: 0 = left, 1 = right, 2 = middle. `mouse_clicked` is edge-triggered.
> ### `engine.settings_path() -> string`
> The cheat's base settings/data directory (trailing backslash) — build absolute
> paths from this instead of relative `..\scripts\data\`.
> ### `engine.set_thirdperson(bool)` · `engine.is_thirdperson() -> bool`
> Toggle / read the thirdperson camera (so an aura script can enable it itself).
> ### `draw.text_scaled(x, y, color, text, size [, center])`
> The **default** font at an arbitrary pixel `size` — big readouts (speedometer)
> without shipping a TTF. `center` (bool) centres on `x`. Large sizes soften; use
> `draw.create_font` for crisp huge text.
> ### `task.delay(s, fn) -> cancel` · `task.interval(s, fn) -> cancel`
> Run `fn` once after `s` seconds, or every `s` seconds. Returns a **cancel**
> function — call it to stop the timer.

## Full example

See the ready-to-run scripts in [`scripts/`](../scripts):
`hello.lua`, `box_esp.lua`, `scheduler.lua`, `movement.lua`.
