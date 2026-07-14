# Lua API Reference

Script the xhook CS 1.6 cheat in Lua — draw ESP, read entities, hook net messages,
write GLSL shaders, stream audio, and more.

## Your first script

Drop a `.lua` file in `<settings>\scripts\` and it loads automatically. This one
draws text and a box every frame:

```lua
callbacks.register("on_paint", function()
    draw.text(20, 20, draw.color(0, 235, 255), "hello from Lua")
    draw.rect(20, 44, 120, 60, draw.color(0, 235, 255), 4, 1)
end)
```

Save the file — it reloads live while the game runs.

## How it works

- **Where scripts go:** `<settings>\scripts\*.lua` (the same folder your configs
  live in; created automatically on first run).
- **Hot-reload:** edit and save — the file reloads live (polled ~4×/sec). New files
  are picked up automatically; deleting a file unloads it.
- **Sandbox:** each script runs in its own environment with only `string`, `table`,
  `math`, `coroutine` and the namespaces in these docs. `io`, `os`, `package`,
  `debug`, `load`/`dofile` are **not** available. CPU (per-handler wall-clock) and
  memory are capped; a runaway loop is aborted.
- **Persistence:** the `state` table survives hot-reloads. Everything else in a
  script is re-created on reload.

## Modules / parent-child scripts (`require`)

A script can pull in other Lua files with `require("name")`, which loads and runs
`<settings>\scripts\lib\name.lua` **once** in the calling script's environment and
returns its value. Files under `scripts\lib\` are **not** auto-loaded on their own —
they only run when a parent requires them — so one "main" script can compose several
children, and enabling the main brings them all. A child's callbacks/tasks are
**owned by the parent**, so they unload when the parent does.

```lua
-- main.lua  (in scripts\)
require("radar")               -- runs scripts\lib\radar.lua (side effects)
local util = require("util")   -- scripts\lib\util.lua returns a table
print(util.add(2, 3))
```

## What's here

- **[Events](events.md)** — the callbacks you hook (`on_paint`, `on_create_move`, …)
- **[Engine & World](engine-world.md)** — `engine.*`, `world.*`, traces, players
- **[Drawing & Effects](drawing.md)** — `draw.*`, 3D, `gfx.*` post-processing + shaders
- **[Net · DB · Features · UI](systems.md)** — net-message hooks, storage, your own menu
- **[Callbacks · Tasks · Types](types.md)** — scheduler, `Vector`/`Angle`/`Quaternion`/`Entity`
- **[Utilities & Memory](utilities.md)** — `util`, `bit`, `files`, raw `memory.*`
- **[Audio & State](audio-misc.md)** — streaming radio, one-shot SFX, the `state` table
- **[Examples](examples.md)** — full ready-to-use scripts
