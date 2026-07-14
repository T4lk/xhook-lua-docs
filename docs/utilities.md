# Utilities & Memory

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

**Example — read the engine module base and scan for a pattern:**

```lua
local base = memory.module_base("hw.dll")
engine.log(("hw.dll base = %X"):format(base))

-- find a signature and read the dword at that address
local addr = memory.find_pattern("hw.dll", "55 8B EC 83 EC ??")
if addr then
    engine.log(("found @ %X, dword = %d"):format(addr, memory.read_int(addr)))
end
```

> ⚠️ `memory.*` is the trusted **Native** tier — reads and (especially) writes are
> unguarded. A bad write crashes the game. Snapshot with `read_bytes` and verify an
> address before writing to it.

---

