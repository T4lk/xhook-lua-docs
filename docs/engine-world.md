# Engine & World

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
> ### `engine.perf_time() -> number`
> High-resolution monotonic clock in seconds (`QueryPerformanceCounter`, sub-µs
> precision). Use it for smooth interpolation / animation timing where
> `real_time()`'s ~1 ms granularity would judder.

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
> custom knife models add an inspect at a higher index.
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

> ### `world.dlight(origin, radius, r, g, b [, life=0.1 [, decay=0 [, elight=false [, key=0]]]])`
> Spawn a **real engine dynamic light** — it lights the map geometry (engine world
> AND the replacement world renderer) and any models near it. `r/g/b` are 0..255.
> Lights are transient: call it every frame from `on_paint` to keep one alive
> (`life` seconds per pulse; `decay` shrinks radius/sec). `elight=true` makes an
> entity light (models only, no world). `key` pins one engine slot per unique
> value so a moving light **updates in place** instead of stacking — use your own
> ids (1000+i) for persistent movers; `0` grabs any free slot.
> ```lua
> -- warm flickering torch that follows you
> callbacks.register("on_paint", function()
>     local me = engine.local_player(); if not me then return end
>     local flick = 0.9 + 0.1 * math.sin(engine.real_time() * 11)
>     world.dlight(me:origin() + Vector(0,0,40), 260 * flick, 255, 160, 60, 0.1, 0, false, 999)
> end)
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

