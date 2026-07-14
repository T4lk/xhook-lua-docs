# Callbacks · Tasks · Types

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

**Type & raw props**:
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

**Backtrack**:
- `:backtrack_records() -> table[]` — the quantised rewind candidates the
  ragebot / backtrack-chams use for this player. Each entry is
  `{ origin = Vector, time = number (server animtime), latency = number (fake-lag
  seconds this record would request) }`. Read-only view of the existing lag-comp
  history; empty for non-players / when no history exists. Handy for drawing
  backtrack points (`draw.box_3d` at each `origin`).

**Local movement override** (local player only):
- `:set_velocity(Vector)` / `:set_origin(Vector)` — queue a **client-side
  prediction** velocity/teleport applied after movement runs. ⚠️ Prediction only:
  the server re-simulates from your usercmd, so it **rubberbands** on lag-comp
  servers — for real, server-respected movement drive `cmd.forwardmove` /
  `sidemove` / `buttons` from `on_create_move` instead. Effects last one
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

