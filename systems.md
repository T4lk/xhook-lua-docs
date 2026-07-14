# Net · DB · Features · UI

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

