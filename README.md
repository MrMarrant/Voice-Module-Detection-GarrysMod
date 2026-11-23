# Voice-Module-Detection-GarrysMod

This module was originally created for the [SCP-2521](https://steamcommunity.com/sharedfiles/filedetails/?id=3843598891) addon, but you are free to use it however you like.

## Installation

> The module is installed **on the server only**. Players don't need to download anything.

### 1. Identify the server type

In the server console, type:

```
lua_run print(jit.os, jit.arch)
```

| Output | Server type |
|---|---|
| `Windows x64` | Windows 64-bit |
| `Windows x86` | Windows 32-bit |
| `Linux x64` | Linux 64-bit |
| `Linux x86` | Linux 32-bit |

32-bit is Garry's Mod's default branch, 64-bit is the **x86-64** branch.

### 2. Pick the file

Download the file matching your server int the [Releases section](https://github.com/MrMarrant/Voice-Module-Detection-GarrysMod/releases):

| Server | File |
|---|---|
| Windows 64-bit | `gmsv_scp_2521_voice_win64.dll` |
| Windows 32-bit | `gmsv_scp_2521_voice_win32.dll` |
| Linux 64-bit | `gmsv_scp_2521_voice_linux64.dll` |
| Linux 32-bit | `gmsv_scp_2521_voice_linux.dll` |

**On a 64-bit server, two versions are available:**
- **base**: lightweight, fine for most servers.
- **small**: understands much better, but uses more memory and CPU.

32-bit servers only have the **base** version.

### 3. Install

1. Copy the file into `garrysmod/lua/bin/`. Create the `bin` folder if it doesn't exist.
2. **Don't rename** the file.
3. Restart the server.

### Troubleshooting

- **Nothing in the console on startup**: check the folder (`lua/bin/`), the file name, and that it matches your server type (step 1).
- **The module refuses to start on a 64-bit server**: the machine's CPU is too old.
- **The module stopped working after a Garry's Mod update**: wait for an update of the module.

> **Privacy:** voice is never recorded, only the text is used. Still, let your players know that their voice is analysed (GDPR).

## Configuration (server convars, saved)

| Convar | Default | Purpose |
|---|---|---|
| `scp_2521_voice` | `1` | Enables or disables voice detection (applied immediately) |
| `scp_2521_voice_language` | `auto` | `auto` = server language (`gmod_language`), otherwise a two-letter code (`en`, `fr`…). If no language is found: English |
| `scp_2521_voice_threads` | `2` | CPU cores used for transcription (applied the next time the module starts) |

## Choosing the model

The model is chosen at build time and embedded in the file. `scp_2521_voice_status` shows which one is installed.

| Model | Size | Extra RAM | Compute | French | Platforms |
|---|---|---|---|---|---|
| `base` (default) | ~57 MB | ~250 MB (measured: 215 MB) | baseline | weak | all |
| `small` | ~190 MB | ~500 MB (estimate, not measured) | ~3× more | much better | 64-bit only |

`small` is rejected when building for 32-bit. A 32-bit srcds only has 4 GB of address space and often runs short of memory with a large map. ggml also runs without AVX there, so transcription would no longer keep up with players.

### Lua API (global table `scp_2521_voice`, server)

| Function | Returns |
|---|---|
| `Start(language, threads)` | `true, method` or `false, reason`. Loads the model (≈ 1 s) and installs the detour |
| `Stop()` | Removes the detour and stops the threads. Also called when the module is unloaded |
| `SetLanguage(language)` | `boolean` |
| `Poll(max)` | `{ { steamid = "7656…", text = "…" }, … }` |
| `GetStats()` | Diagnostic counters, or `nil` if the module is stopped |
| `VERSION` | `"1.0.0"` |
| `MODEL` | Embedded Whisper model: `"base"` or `"small"` |


#### Examples

**Load the module only if it is installed:**

```lua
if util.IsBinaryModuleInstalled("scp_2521_voice") then
    require("scp_2521_voice") -- creates the global table scp_2521_voice
end
```

**Start and stop transcription:**

```lua
-- language: Whisper code ("en", "fr"…), threads: clamped between 1 and the number of CPU cores
local ok, detail = scp_2521_voice.Start("en", 2) -- loads the model, takes about 1 s
if not ok then
    ErrorNoHalt("Voice transcription disabled: " .. tostring(detail) .. "\n") -- invalid language, old CPU, signature not found…
    return
end
print("Voice transcription enabled, engine hook: " .. tostring(detail)) -- detail is nil if it was already started

-- Later (map change, convar turned off…): no effect if it is not started
scp_2521_voice.Stop()
```

**Change the language without restarting:**

```lua
if not scp_2521_voice.SetLanguage("fr") then
    print("Invalid language code, or module stopped")
end
```

**Read the transcripts** (on a timer: `Poll` never waits, it returns an empty table when there is nothing new):

```lua
timer.Create("my_addon_voice_poll", 0.25, 0, function()
    for _, entry in ipairs(scp_2521_voice.Poll(16)) do -- at most 16 per call (1 to 64)
        local ply = player.GetBySteamID64(entry.steamid)
        if IsValid(ply) then
            print(ply:Nick() .. " said: " .. entry.text)
        end
    end
end)
```

**Check the module state:**

```lua
print(scp_2521_voice.VERSION, scp_2521_voice.MODEL) -- "1.0.0", "base"

local stats = scp_2521_voice.GetStats() -- nil if the module is stopped
if stats then
    print("packets: " .. stats.packets .. ", transcripts: " .. stats.transcripts)
    -- also: packetsDropped, decodeErrors, utterances, utterancesDropped, silentUtterances
end
```
