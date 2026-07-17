# Examples

Complete, ready-to-use scripts. Drop any of these in `<settings>\scripts\` as a
`.lua` file and it loads live.

## Watermark

```lua
callbacks.register("on_paint", function()
    local fps = engine.frame_stats().fps
    draw.text(12, 12, draw.color(0, 235, 255),
        ("xhook  |  %s  |  %.0f fps"):format(engine.map_name(), fps))
end)
```

## Box ESP

3D boxes on every player, dimmed when they're behind a wall, with a name tag:

```lua
callbacks.register("on_paint", function()
    for _, p in ipairs(world.get_players()) do
        if p:is_valid() and p:is_alive() and not p:is_local() then
            local mn, mx = p:bbox()
            if mn and mx then
                local col = p:is_enemy() and draw.color(255, 60, 60)
                                          or draw.color(60, 160, 255)
                draw.box_3d(mn, mx, col, 1, { occlude = true })
                local sx, sy = world.world_to_screen(Vector(mx.x, mx.y, mx.z))
                if sx then draw.text(sx, sy - 14, col, p:name()) end
            end
        end
    end
end)
```

## Enemy snaplines

```lua
callbacks.register("on_paint", function()
    local w, h = engine.screen_size()
    for _, p in ipairs(world.get_players()) do
        if p:is_valid() and p:is_alive() and p:is_enemy() then
            local sx, sy = world.world_to_screen(p:origin())
            if sx then draw.line(w * 0.5, h, sx, sy, draw.color(255, 80, 80, 150)) end
        end
    end
end)
```

## Simple bunnyhop

Hold jump; it only fires the jump when you're on the ground:

```lua
local IN_JUMP = 2
callbacks.register("on_create_move", function(cmd)
    local me = engine.local_player()
    if not (me and me:is_alive()) then return end
    if bit.band(cmd.buttons, IN_JUMP) ~= 0 and not me:on_ground() then
        cmd.buttons = bit.band(cmd.buttons, bit.bnot(IN_JUMP))  -- clear jump in the air
    end
end)
```

## Kill counter (net-message hook)

```lua
local kills = 0
net.hook_message("DeathMsg")

callbacks.register("on_net_message", function(msg)
    if msg:name() ~= "DeathMsg" then return end
    local killer = msg:read_byte()          -- byte killer, byte victim, byte hs, string weapon
    local me = engine.local_player()
    if me and killer == me:index() then kills = kills + 1 end
end)

callbacks.register("on_paint", function()
    draw.text(12, 30, draw.color(0, 255, 140), "kills this session: " .. kills)
end)
```

## Custom shader (heat-haze)

Write your own GLSL post-process pass — this warps the whole screen with a wave:

```lua
gfx.shader([[
  #version 120
  uniform sampler2D u_tex;
  uniform float u_time;
  void main(){
    vec2 uv = gl_TexCoord[0].xy;
    uv.x += sin(uv.y * 24.0 + u_time * 3.0) * 0.003;
    gl_FragColor = texture2D(u_tex, uv);
  }
]])
callbacks.register("on_script_unload", function() gfx.shader(nil) end)
```

## World shader (tron grid)

Shade the **map geometry itself** — the `xs_world` function runs inside the
engine's world shader after all lighting, so lightmaps/sun/fog keep working.
Needs the modern world renderer (Visuals → World).

```lua
gfx.world_shader([[
uniform float xs_amt;
vec3 xs_world(vec3 col, vec3 albedo, vec3 light, vec3 wpos, vec3 normal, vec2 uv){
    vec2 g = abs(fract(wpos.xy / 64.0) - 0.5);
    float line = smoothstep(0.46, 0.5, max(g.x, g.y));
    vec3 neon = vec3(0.1, 0.9, 1.0) * (0.6 + 0.4 * sin(xs_time * 2.0));
    return mix(col, col * 0.3 + neon * line, xs_amt);
}
]])
callbacks.register("on_paint", function() gfx.world_uniform("xs_amt", 1.0) end)
callbacks.register("on_script_unload", function() gfx.world_shader(nil) end)
```

## Custom player material (energy rings)

A real GLSL fragment shader on player models, applied through the `"custom"`
chams mode. Needs native Chams enabled (any mode) in Visuals → Chams.

```lua
chams.shader([[
#version 120
varying vec3 v_mpos; varying float v_fresnel;
uniform float u_time; uniform vec4 u_color; uniform float u_alpha;
void main(){
    float bands = pow(0.5 + 0.5 * sin(v_mpos.z * 0.5 - u_time * 5.0), 3.0);
    vec3 c = u_color.rgb * (0.15 + bands * 1.2)
           + vec3(1.0) * pow(clamp(v_fresnel, 0.0, 1.0), 3.0);
    gl_FragColor = vec4(c, u_alpha);
}
]])
callbacks.register("on_paint", function()
    for _, e in ipairs(world.get_players(true)) do
        if e:is_alive() then
            chams.set(e, { mode = "custom", color = { 0, 220, 255, 255 },
                           occluded = "custom", occluded_color = { 255, 60, 60, 255 } })
        end
    end
end)
callbacks.register("on_script_unload", function() chams.shader(nil) end)
```

## Streaming radio

```lua
local h = audio.stream("http://ice1.somafm.com/groovesalad-128-mp3")
audio.set_volume(0.4)

callbacks.register("on_paint", function()
    if audio.is_playing(h) then
        draw.text(12, 48, draw.color(200, 200, 255),
            ("♪ %.0fs"):format(audio.position(h)))
    end
end)
callbacks.register("on_script_unload", function() audio.stop(h) end)
```
