# Drawing & Effects

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
> ### `draw.text_sized(x, y, size, color, text)`
> Text at an explicit **pixel size** — for distance-scaled labels (far cards get
> smaller text), big score numbers, etc. (`draw.text_scaled` is the same effect with
> a different argument order.)

> ### `draw.line(x1, y1, x2, y2, color)`
> ### `draw.rect(x, y, w, h, color [, rounding=0] [, thickness=1])` — outline
> ### `draw.rect_filled(x, y, w, h, color [, rounding=0])`
> ### `draw.circle(x, y, radius, color [, segments=24] [, thickness=1])`
> ### `draw.circle_filled(x, y, radius, color [, segments=24])`
> ### `draw.triangle(x1,y1, x2,y2, x3,y3, color [, thickness=1])`
> ### `draw.triangle_filled(x1,y1, x2,y2, x3,y3, color)`
> ### `draw.triangles({x1,y1, x2,y2, ...}, color)`
> A raw **triangle list** from a flat point table (every 3 points = one filled
> triangle). Solid, **non-anti-aliased** fill — no edge fringe, so adjacent
> triangles don't double-blend at shared edges. Build a clean slider tube / arbitrary
> mesh in one call (emit a strip as `v0,v1,v2, v1,v2,v3, …`). Unlike `polygon_filled`
> it isn't limited to a convex shape.
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
> ### `draw.image_quad(image, x1,y1, x2,y2, x3,y3, x4,y4 [, color])`
> Draw an `Image` warped onto **four arbitrary corners** — genuinely
> perspective-tilted, not just axis-aligned scaling. Corners map to the image's UVs
> in order **TL, TR, BR, BL**. Project the four corners of a card / playfield / note
> with `world_to_screen` and pass them here for a real 3D-tilted quad.
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
- `gfx.world_shader(glsl_fn)` — **shade the WORLD itself** (not the screen): splice
  a GLSL 120 function into the replacement world renderer's fragment shader. You
  supply exactly one function:
  `vec3 xs_world(vec3 col, vec3 albedo, vec3 light, vec3 wpos, vec3 normal, vec2 uv)`
  — it runs as the *last* step of the engine world shader (after lightmaps, sun,
  dynamic lights, tint and fog all did their thing) and returns the final surface
  colour. `uniform float xs_time;` (seconds) is predeclared — do **not** redeclare
  it; declare any extra uniforms yourself and feed them with `gfx.world_uniform`.
  Pass `nil`/`""` to remove (auto-removed on unload). Compile failure keeps the
  stock look. Needs the modern world renderer (Visuals → World) to be active.
- `gfx.world_uniform(name, a [, b [, c]])` — `float` / `vec2` / `vec3` uniform on
  the world function.

```lua
-- world: pulsing tron grid painted onto every surface, lighting intact
gfx.world_shader([[
vec3 xs_world(vec3 col, vec3 albedo, vec3 light, vec3 wpos, vec3 normal, vec2 uv){
    vec2 g = abs(fract(wpos.xy / 64.0) - 0.5);
    float line = smoothstep(0.47, 0.5, max(g.x, g.y));
    return col * 0.35 + vec3(0.1, 0.9, 1.0) * line * (0.6 + 0.4 * sin(xs_time * 2.0));
}
]])
```

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

## `chams.*` — script-driven player materials

Bridge to the native GPU chams pass: assign a material **per player, per frame**
(call from `on_paint`; stop calling → native menu chams resume). Needs native
Chams enabled for the category in Visuals → Chams.

- `chams.set(player, { mode=, color=, occluded=, occluded_color=, wireframe=, hide= })`
  — `player` is an Entity or index. `mode`/`occluded` accept a mode name (see
  `chams.modes`) or id; `color`/`occluded_color` are `{r,g,b,a}` 0..255.
- `chams.clear(player)` — hand the player back to the native chams.
- `chams.modes` — table of `name = id` for every material (`flat`, `glow`,
  `galaxy`, `dragon`, … and `custom`, see below).
- `chams.shader(glsl_frag)` — **write your own player material**: install a full
  GLSL 120 fragment shader for the `"custom"` mode, then use it with
  `chams.set(e, { mode = "custom", ... })`. It links against the GPU-skinning
  vertex shader — declare only the varyings you use:
  `varying vec2 v_uv;` (skin uv), `varying vec3 v_light;` (engine light),
  `varying float v_fresnel;` (1 at silhouette), `varying vec2 v_chromeUV;`,
  `varying vec3 v_vpos; / v_vnormal;` (view-space), `varying vec3 v_wpos;`
  (world), `varying vec3 v_mpos;` (model-space, body-anchored patterns),
  `varying float v_up;` (0 feet .. 1 head) — plus the standard uniforms you want:
  `uniform sampler2D u_tex;` (the real skin, when the model has one),
  `uniform float u_time;`, `uniform vec4 u_color;` (the chams.set colour, 0..1),
  `uniform float u_alpha;`. Pass `nil`/`""` to remove (auto-removed on unload).
  While none is installed / compile failed, `"custom"` draws flat colour.
- `chams.uniform(name, a [, b [, c]])` — custom `float` / `vec2` / `vec3` uniform
  on your material shader.

```lua
chams.shader([[
#version 120
varying vec3 v_mpos; varying float v_fresnel;
uniform float u_time; uniform vec4 u_color; uniform float u_alpha;
void main(){
    float bands = 0.5 + 0.5 * sin(v_mpos.z * 0.35 - u_time * 4.0);   // energy rings
    vec3 c = u_color.rgb * bands + vec3(1.0) * pow(v_fresnel, 3.0);
    gl_FragColor = vec4(c, u_alpha);
}
]])
callbacks.register("on_paint", function()
    for _, e in ipairs(world.get_players(true)) do
        if e:is_alive() then chams.set(e, { mode = "custom", color = {0,220,255,255} }) end
    end
end)
```

