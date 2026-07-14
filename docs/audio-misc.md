# Audio & State

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

> ### `audio.stream(url) -> handle` · `audio.play_file(path [, rate]) -> handle`
> Open + start playing a stream URL or a local/downloaded file. Returns a handle.
> `rate` (play_file) is a playback-speed multiplier: `1.0` normal, `1.5` faster +
> higher-pitched (Nightcore/Double-Time), `0.75` slower + lower (Half-Time). It's
> set when the device opens, so it only applies to a **fresh** `play_file`.
> `audio.position` still reports content-time, so anything synced to the audio stays
> in sync regardless of rate.
> ### `audio.stop([handle])` · `audio.pause(handle, bool)`
> Stop one handle (or **all** if omitted); pause/resume.
> ### `audio.set_volume(0..1)` · `audio.volume() -> number`
> Global output volume.
> ### `audio.is_playing(handle) -> bool` · `audio.position(handle) -> seconds`
> Playing state and playback position (for a progress bar).
> ### `audio.duration(handle) -> seconds | nil`
> Total content length (read from the file's metadata when it opens), or `nil` if
> unknown. Pair with `audio.position` for a real progress bar and clean end-of-track
> logic — no more `is_playing` + wall-clock guessing.
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

