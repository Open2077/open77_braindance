# open77_braindance

The Cyberpunk 2077 braindance for [Open77](https://open2077.net) scripts, rendered with the game's own
effects only:

- **The launch sequence**: the white-out flash with chromatic aberration and its sound, the dive to
  black, then the braindance look (first person, cyan shimmer) or a plain return to the world. Every
  phase raises an event. While the screen is **black**, your script does what the player must not
  see: teleport them, change their bucket, spawn a set, change the weather.
- **The vision layers**: the braindance editor's **VISUAL**, **THERMAL** and **AUDIO** layers on the
  live world, with a heatmap that outlines every body around the player through walls.

The resource draws **no interface**. Your resource draws its own (layer name, timeline, key hints)
from the exports and events.

Full documentation: **[open2077.net/docs/braindance](https://open2077.net/docs/braindance)**.

## Requirements

- An Open77 server. The launch sequence works on every 2.5 client.
- The vision layers need a client newer than Stable 182 (`Open77.braindance`). On an older client,
  `setLayer` answers `nil` with a reason and nothing changes on screen.

## Install

```sh
git clone https://github.com/Open2077/open77_braindance.git resources/open77_braindance
```

The directory must stay named `open77_braindance`. Add it to `server.jsonc`:

```jsonc
"resources": { "load": [ /* ... */ "open77_braindance" ] }
```

Then `refresh` and `ensure open77_braindance` in the server console. The manifest has
`auto_start false`: it only runs once it is ensured.

Your own resource declares the dependency:

```lua
dependency "open77_braindance >=1.1.0"
```

Calling the exports needs no permission. The permissions belong to `open77_braindance`:
`network.events`, `world.effects`, `screen.effects`, `screen.read`, `input.actions`,
`ui.vanilla.hud`, `perspective.policy` and `players.teleport`.

## Quick start

Braindance transition to another place (server). The resource moves the player while the screen
is black, waits for the world to load, then brings the image back:

```lua
local id, reason = exports.open77_braindance:start(playerId, {
    preset = "transition",
    teleport = { x = 1611.07, y = -775.44, z = 49.98, heading = 90, bucket = 4 },
})
```

Your own work while the screen is black (server):

```lua
local id = exports.open77_braindance:start(playerId, { preset = "braindance", hold = true })

AddEventHandler("open77_braindance:black", function(pid, sequenceId, tag)
    if sequenceId ~= id then return end
    CreateThread(function()
        Open77.players.teleport(pid, { x = 100.0, y = 200.0, z = 30.0 }, { fade = false }):await()
        exports.open77_braindance:resume(pid, sequenceId)
    end)
end)

-- Later: leave the braindance.
exports.open77_braindance:exit(playerId, id)
```

Vision layers (client):

```lua
exports.open77_braindance:setLayer("thermal")                   -- dark world, bodies outlined in red
exports.open77_braindance:setHeat({ npcId = suspect }, "clue")  -- one body in its own colour
exports.open77_braindance:setLayer("off")

AddEventHandler("open77_braindance:layer", function(layer, previous)
    -- show your own layer indicator
end)
```

Vision layers (server):

```lua
exports.open77_braindance:setLayer(playerId, "thermal", { heat = { radius = 40 } })
exports.open77_braindance:setHeat(playerId, { playerId = suspect }, "clue")
AddEventHandler("open77_braindance:layerChanged", function(pid, layer, previous, reason) end)
```

## What is in it

| | |
| --- | --- |
| Presets | `braindance` (overlay until `exit`), `transition`, `glitch` |
| Phases | `flash`, `darkening`, `black`, `revealing`, `active`, `exiting`, `finished`, `cancelled` |
| Server exports | `start`, `resume`, `exit`, `cancel`, `state`, `playEffect`, `stopEffects`, `setLayer`, `setHeat`, `clearHeat`, `layer` |
| Client exports | `start`, `resume`, `exit`, `cancel`, `state`, `active`, `effects`, `presets`, `playEffect`, `stopEffect`, `setLayer`, `layer`, `layers`, `nextLayer`, `setHeat`, `clearHeat`, `heatmap` |
| Server events | `open77_braindance:phase`, `open77_braindance:black`, `open77_braindance:layerChanged` |
| Client events | `open77_braindance:phase`, `open77_braindance:black`, `open77_braindance:layer` |
| Layers | `visual`, `thermal`, `audio`, `off` |

Every option, range and return value is on the [documentation page](https://open2077.net/docs/braindance).
The layers are built on the client API `Open77.braindance` (`setHeatmap`, `setHeat`, `clearHeat`,
`heatmap`, `palette`, `levels`), which any resource with `world.effects` can also call directly.

## Console

```
braindance <playerId> <braindance|transition|glitch> [hold | x y z [heading]]   -- server
braindance <playerId> <exit|resume|cancel> [id]
braindance <playerId> effect <name|stop> [durationMs]
bdlayer <playerId> <off|visual|thermal|audio>
bdlayer <playerId> heat <targetPlayerId> <level>

bd <braindance|transition|glitch|exit|resume|cancel> [id|hold]                   -- client
bdlayer <off|visual|thermal|audio|next> [noeffect]
```

## Tests

The sequence engine and the server half run against a fake platform with any Lua 5.4:

```powershell
./tests/run-lua.ps1 -LuaDll path/to/lua54.dll
./tests/run-lua.ps1 -LuaDll path/to/lua54.dll -Test server_test.lua
```

## Limits

- The look is made of the game's own pieces, not the vanilla braindance editor: there is no rewind,
  and clue scanning is yours to script with `setHeat(target, "clue")`.
- The heatmap uses the same highlight as the scanner; opening the scanner while a layer is on mixes
  both.

## License

[MIT](LICENSE).
