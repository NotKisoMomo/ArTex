# ArTex

<div align="center">

[![Version](https://img.shields.io/badge/version-v1.0.0-6C3EF4?style=for-the-badge&logoColor=white)](https://github.com/NotKisoMomo/ArTex/releases)
[![Stability](https://img.shields.io/badge/stability-stable-22c55e?style=for-the-badge)](https://github.com/NotKisoMomo/ArTex)
[![License](https://img.shields.io/badge/license-MIT-3b82f6?style=for-the-badge)](https://github.com/NotKisoMomo/ArTex/blob/main/LICENSE)
[![Roblox](https://img.shields.io/badge/Roblox-Model-e11d48?style=for-the-badge&logo=roblox&logoColor=white)](https://www.roblox.com/library)

<br/>

[![Get Latest](https://img.shields.io/badge/%E2%86%93%20Get%20Latest-6C3EF4?style=for-the-badge)](https://github.com/NotKisoMomo/ArTex/releases/latest)
[![Wiki](https://img.shields.io/badge/Wiki-Docs-0f172a?style=for-the-badge&logo=gitbook&logoColor=white)](https://github.com/NotKisoMomo/ArTex/wiki)

</div>

---

ArTex is a full-stack game framework for Roblox. Define your services, managers, and controllers once -- ArTex scans them, resolves their boot order, injects their dependencies, and runs them in strict tier sequence on the correct side of the client-server boundary.

Schema and Fabrik ship pre-installed. No boilerplate. No manual registration. No require chains.

---

## Install

Place the `ArTex` folder into `ReplicatedStorage` and require it from your server and client entry scripts.

```lua
local ArTex = require(game:GetService("ReplicatedStorage").ArTex)
```

Drop `Fabrik` and `Schema` as direct children of the `ArTex` ModuleScript. They are available on `ctx.Packages.Fabrik` and `ctx.Packages.Schema` automatically -- no configuration needed.

---

## At a Glance

```lua
-- Server.lua
ArTex.configure({
    NetworkManager    = require(ServerStorage.Modules.Network.NetworkManager),
    NetworkController = require(ReplicatedStorage.Client.Modules.Network.NetworkController),
    ContextStructure  = {
        Server = { Host = ServerStorage.Modules,            Structure = { "Services", "Managers" } },
        Client = { Host = ReplicatedStorage.Client.Modules, Structure = { "Controllers" } },
        Shared = { Host = ReplicatedStorage.Shared.Modules, Structure = { "Packages" } },
    },
})

ArTex.priority({
    Services = { "DataService", "PlayerService" },
    Managers = { "NetworkManager", "CombatManager" },
})

ArTex.compile():next(function()
    print("live in", ArTex.diagnostics.bootTime, "ms")
end)
```

```lua
-- PlayerService.lua
local PlayerService   = {}
PlayerService.Context = "Server"
PlayerService.After   = { "DataService" }

local _ArTex
local _Network

function PlayerService.Start(ctx)
    _ArTex   = ctx.ArTex
    _Network = _ArTex.resolve("NetworkManager")

    ctx.GetService("DataService").PlayerAdded(function(player, data)
        ctx.Cache.set("Profile." .. player.UserId, data)
    end)
end

return PlayerService
```

---

## Why ArTex

| | Knit | Nevermore | ArTex |
|---|---|---|---|
| Auto folder scanning | ✗ | ✗ | ✓ |
| Dependency injection via ctx | ✗ | ✗ | ✓ |
| Three-phase latch-gated boot | ✗ | ✗ | ✓ |
| Built-in reactive cache | ✗ | ✗ | ✓ |
| Priority map | ✗ | ✗ | ✓ |
| Schema + Fabrik pre-installed | ✗ | ✗ | ✓ |
| Sync module resolution | ✗ | ✗ | ✓ |
| Plugin system | ✗ | ✗ | ✓ |
| Cycle detection | ✗ | ✗ | ✓ |
| Client table auto-wiring | ✓ | ✗ | ✓ |

---

## Documentation

Full documentation lives in the [Wiki](https://github.com/TheRealKr3ative/ArTex/wiki).

| Page | Description |
|---|---|
| [Getting Started](https://github.com/NotKisoMomo/ArTex/wiki/Getting-Started) | Install, configure, first boot |
| [Core Concepts](https://github.com/NotKisoMomo/ArTex/wiki/Core-Concepts) | Architecture, lifecycle, tiers, ctx |
| [API Reference](https://github.com/NotKisoMomo/ArTex/wiki/API-Reference) | Every method on ArTex |
| [Module Fields](https://github.com/NotKisoMomo/ArTex/wiki/Module-Fields) | All declarative module fields |
| [Patterns](https://github.com/NotKisoMomo/ArTex/wiki/Patterns) | Real-world usage examples |
| [Plugins](https://github.com/NotKisoMomo/ArTex/wiki/Plugins) | Plugin system and first-party plugins |
| [Migration](https://github.com/NotKisoMomo/ArTex/wiki/Migration) | Moving from Knit or Nevermore |

---

## Contact

| Platform | Handle |
|---|---|
| Roblox | [Kr3ativeKrayon](https://www.roblox.com/users/1911367519/profile) |
| YouTube | [TotallyKr3ative](https://www.youtube.com/channel/UCpNZQoKVclQ74Pk5GmzdQDA) |
| X | [TotallyNotKr3ative](https://x.com/TheRealKr3ative) |
| Email | [TheRealKr3ative@gmail.com](mailto:TheRealKr3ative@gmail.com) |

---

*Built by [Ternary Productions](https://github.com/NotKisoMomo)*
    },
})

ArTex.priority({
    Services = { "DataService", "PlayerService" },
    Managers = { "NetworkManager", "CombatManager" },
})

ArTex.compile():next(function()
    print("live in", ArTex.diagnostics.bootTime, "ms")
end)
```

```lua
-- PlayerService.lua
local PlayerService   = {}
PlayerService.Context = "Server"
PlayerService.After   = { "DataService" }

local _ArTex
local _Network

function PlayerService.Start(ctx)
    _ArTex   = ctx.ArTex
    _Network = _ArTex.resolve("NetworkManager")

    ctx.GetService("DataService").PlayerAdded(function(player, data)
        ctx.Cache.set("Profile." .. player.UserId, data)
    end)
end

return PlayerService
```

---

## Why ArTex

| | Knit | Nevermore | ArTex |
|---|---|---|---|
| Auto folder scanning | ✗ | ✗ | ✓ |
| Dependency injection via ctx | ✗ | ✗ | ✓ |
| Three-phase latch-gated boot | ✗ | ✗ | ✓ |
| Built-in reactive cache | ✗ | ✗ | ✓ |
| Priority map | ✗ | ✗ | ✓ |
| Schema + Fabrik pre-installed | ✗ | ✗ | ✓ |
| Sync module resolution | ✗ | ✗ | ✓ |
| Plugin system | ✗ | ✗ | ✓ |
| Cycle detection | ✗ | ✗ | ✓ |
| Client table auto-wiring | ✓ | ✗ | ✓ |

---

## Documentation

Full documentation lives in the [Wiki](https://github.com/TheRealKr3ative/ArTex/wiki).

| Page | Description |
|---|---|
| [Getting Started](https://github.com/TheRealKr3ative/ArTex/wiki/Getting-Started) | Install, configure, first boot |
| [Core Concepts](https://github.com/TheRealKr3ative/ArTex/wiki/Core-Concepts) | Architecture, lifecycle, tiers, ctx |
| [API Reference](https://github.com/TheRealKr3ative/ArTex/wiki/API-Reference) | Every method on ArTex |
| [Module Fields](https://github.com/TheRealKr3ative/ArTex/wiki/Module-Fields) | All declarative module fields |
| [Patterns](https://github.com/TheRealKr3ative/ArTex/wiki/Patterns) | Real-world usage examples |
| [Plugins](https://github.com/TheRealKr3ative/ArTex/wiki/Plugins) | Plugin system and first-party plugins |
| [Migration](https://github.com/TheRealKr3ative/ArTex/wiki/Migration) | Moving from Knit or Nevermore |

---

## Contact

| Platform | Handle |
|---|---|
| Roblox | [Kr3ativeKrayon](https://www.roblox.com/users/1911367519/profile) |
| YouTube | [TotallyKr3ative](https://www.youtube.com/channel/UCpNZQoKVclQ74Pk5GmzdQDA) |
| X | [TotallyNotKr3ative](https://x.com/TheRealKr3ative) |
| Email | [TheRealKr3ative@gmail.com](mailto:TheRealKr3ative@gmail.com) |

---

*Built by [Plinko Labs](https://github.com/TheRealKr3ative)*

---
