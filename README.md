# ArTex — Roblox Game Framework

[![Static Badge](https://img.shields.io/badge/build-v1.0.0-black)](https://github.com/TheRealKr3ative)
![Static Badge](https://img.shields.io/badge/stability-stable-green)

ArTex is a full-stack game framework for Roblox. It handles module registration, lifecycle orchestration, dependency injection, context-aware booting, reactive caching, middleware pipelines, plugin architecture, and first-class Schema and Fabrik integration — all woven into a single coherent system. Define your services, managers, and controllers once. ArTex scans them, orders them, injects their dependencies, and boots them in the right sequence on the right side of the client-server boundary.

---

## Table of Contents

- [Features](#features)
- [Installation](#installation)
- [Quick Start](#quick-start)
- [Core Concepts](#core-concepts)
  - [Architecture](#architecture)
  - [Module Shape](#module-shape)
  - [Context Structure](#context-structure)
  - [Lifecycle](#lifecycle)
  - [Dependency Injection](#dependency-injection)
  - [Priority System](#priority-system)
  - [Middleware](#middleware)
  - [Plugin System](#plugin-system)
  - [Cache](#cache)
  - [Networking](#networking)
  - [Reactive State](#reactive-state)
- [API Reference](#api-reference)
  - [ArTex.configure](#artexconfigure)
  - [ArTex.Inside](#artexinside)
  - [ArTex.priority](#artexpriority)
  - [ArTex.compile](#artexcompile)
  - [ArTex.middleware](#artexmiddleware)
  - [ArTex.install](#artexinstall)
  - [ArTex.open](#artexopen)
  - [ArTex.plug](#artexplug)
  - [ArTex.extend](#artexextend)
  - [Module Access](#module-access)
  - [Events](#events)
  - [Diagnostics](#diagnostics)
  - [Async Utilities](#async-utilities)
  - [Network Utilities](#network-utilities)
  - [Schema Utilities](#schema-utilities)
  - [Reactive Utilities](#reactive-utilities)
  - [Cache API](#cache-api)
  - [Boot Control](#boot-control)
  - [Testing and Studio Utilities](#testing-and-studio-utilities)
  - [Logging](#logging)
  - [Feature Flags](#feature-flags)
  - [Scheduling](#scheduling)
  - [Metrics](#metrics)
  - [Commands and Queries](#commands-and-queries)
  - [Workflows](#workflows)
  - [Global Events](#global-events)
  - [Tracing and Spans](#tracing-and-spans)
  - [Graph Utilities](#graph-utilities)
  - [Registry](#registry)
- [ctx Reference](#ctx-reference)
- [Module Fields Reference](#module-fields-reference)
- [First-Party Plugins](#first-party-plugins)
- [Patterns](#patterns)
  - [Basic Server Boot](#basic-server-boot)
  - [Basic Client Boot](#basic-client-boot)
  - [Custom Network Manager](#custom-network-manager)
  - [Custom Network Controller](#custom-network-controller)
  - [Service with Reactive Cache](#service-with-reactive-cache)
  - [Manager with Signals and Locks](#manager-with-signals-and-locks)
  - [Controller with Latching](#controller-with-latching)
  - [Custom Plugin](#custom-plugin)
  - [Testing with Mocks](#testing-with-mocks)
  - [Feature Flags](#feature-flags-pattern)
  - [Workflows](#workflows-pattern)
- [Exported Types](#exported-types)
- [Contact](#contact)

---

## Features

- **Folder-based scanning:** Point ArTex at your module folders. It finds, requires, and registers everything automatically. No manual registration.
- **Context-aware booting:** Server modules boot on server. Client modules boot on client. Shared modules inject into both. Modules can self-declare their context.
- **Lifecycle engine:** Every module runs through `Warmup → Init → Start` in strict tier order. Each phase is latch-gated — nothing advances until every module in the tier completes.
- **Dependency injection:** Every module receives a fully-built `ctx` table in `.Start()`. No `require` chains. No circular dependencies. No global state.
- **Priority maps:** Define boot order for an entire tier in one place with `ArTex.priority()`. No `BootPriority` field needed on every module.
- **Middleware pipeline:** Intercept any lifecycle phase for any module. Log, profile, validate, or transform — before and after every `Start`.
- **Plugin system:** `ArTex.install()` returns a live handle. Plugins hook into middleware, extend the ArTex API surface, and declare their own plugin dependencies.
- **Reactive cache:** Built-in key-value cache powered by Fabrik atoms. Watch keys, compute derived values, set TTLs, lock keys, run transactions.
- **Schema integration:** `ArTex.channel()`, `ArTex.define()`, `ArTex.subscribe()`, `ArTex.party()` are all first-class on ArTex — no separate Schema require needed.
- **Fabrik integration:** `ArTex.promise()`, `ArTex.signal()`, `ArTex.atom()`, `ArTex.latch()`, `ArTex.batch()` — all Fabrik primitives exposed directly.
- **Knit input integration:** `ArTex.Plugins.Input` wraps Knit's full action registry, context stack, and session system into the ArTex lifecycle.
- **Full diagnostics:** Boot timings, phase traces, module status, slow module warnings, per-tier breakdowns, flame-graph-ready profiler data.
- **Studio tools:** HotReload, DevTools widget, visualizer, spy, benchmark, diff, snapshot — all Studio-only, zero production cost.
- **First-party plugins:** Profiler, Inspector, Guard, HotReload, DevTools, Logger, Analytics, Sandbox, RateLimit, Circuit, Watchdog, Bridge, Compat, and more.

---

## Installation

Place the `ArTex` folder into `ReplicatedStorage`. Require it from your server and client entry scripts.

```lua
local ArTex = require(game:GetService("ReplicatedStorage").ArTex)
```

ArTex requires **Fabrik** and **Schema** as peer dependencies. Place both in `ReplicatedStorage` and pass them into `ArTex.configure()`.

---

## Quick Start

```lua
-- Server.lua (ServerScriptService)
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local ServerStorage     = game:GetService("ServerStorage")

local ArTex = require(ReplicatedStorage.ArTex)

ArTex.configure({
    Schema = require(ReplicatedStorage.Shared.Packages.Schema),
    Fabrik = require(ReplicatedStorage.Shared.Packages.Fabrik),

    NetworkManager    = require(ServerStorage.Modules.Network.NetworkManager),
    NetworkController = require(ReplicatedStorage.Client.Modules.Network.NetworkController),
    Manifest          = require(ReplicatedStorage.Shared.Modules.Manifests),

    Verbosity = "trace",
    Timeout   = 30,

    ContextStructure = {
        Server = {
            Host      = ServerStorage.Modules,
            Structure = { "Services", "Managers", "Utility" },
        },
        Client = {
            Host      = ReplicatedStorage.Client.Modules,
            Structure = { "Controllers", "Services", "Utility" },
        },
        Shared = {
            Host      = ReplicatedStorage.Shared.Modules,
            Structure = { "Packages", "Utility" },
        },
    },
})

ArTex.Inside("Server", "Packages", "Keep", require(ServerStorage.Packages.Keep))
ArTex.Inside("Shared", "Packages", "Fabrik", require(ReplicatedStorage.Shared.Packages.Fabrik))

ArTex.priority({
    Services = { "DataService", "PlayerService" },
    Managers = { "NetworkManager", "CombatManager" },
})

ArTex.compile():next(function()
    print("[ArTex] Server is live")
end)
```

---

## Core Concepts

### Architecture

ArTex organises your codebase into four tiers that boot in strict order:

```
Shared   →   Services   →   Managers   →   Controllers
```

Every tier fully completes all lifecycle phases before the next tier begins. Within a tier, `ArTex.priority()` controls order. Modules in the same tier with the same priority run concurrently.

### Module Shape

A module is any ModuleScript that returns a table. All fields are optional except `tier` is inferred from the folder the module lives in.

```lua
local PlayerService = {}

-- identity
PlayerService.Context      = "Server"       -- self-gate, overrides folder detection
PlayerService.Alias        = "PS"           -- ArTex.get("PS") works as shorthand
PlayerService.Version      = "1.0.0"        -- shown in Inspector and Docs plugin
PlayerService.Description  = "Manages player profiles and archive data"
PlayerService.Tags         = { "Core", "Player" }

-- boot control
PlayerService.NoBoot       = false          -- true = never started by ArTex
PlayerService.Boot         = true           -- Shared modules: opt-in to lifecycle
PlayerService.BootPriority = 1              -- overrides priority map within tier
PlayerService.After        = { "DataService" } -- soft ordering, looser than priority map
PlayerService.Optional     = false          -- true = errors are swallowed regardless of Verbosity
PlayerService.Critical     = false          -- true = any error halts entire compile
PlayerService.Retry        = 0             -- auto-retry Start up to N times before erroring
PlayerService.MaxRestarts   = 3             -- max Watchdog plugin restarts
PlayerService.Timeout       = 30            -- per-module timeout, overrides global
PlayerService.Singleton     = true          -- false = allow multiple instances
PlayerService.Verbosity     = "warn"        -- per-module log level, overrides global

-- inline declarations
PlayerService.Schema        = { ... }       -- auto-registered Schema channels
PlayerService.Exports       = { "GetProfile", "WaitForProfile" } -- method whitelist
PlayerService.Flags         = { "NewPlayerSystem" } -- module only boots if all flags enabled
PlayerService.Store         = { name = "PlayerData", schema = { ... } } -- inline datastore
PlayerService.Schedule      = { interval = 60, fn = "tick" } -- inline recurring task

-- hooks
function PlayerService.Warmup()
    -- synchronous, runs before Init, no ctx yet
    -- set up constants, upvalues, internal tables
end

function PlayerService.Init(ctx)
    -- ctx available, world not yet live
    -- set up signals, atoms, internal state
end

function PlayerService.Start(ctx)
    -- world is live, all modules started
    -- connect events, start reacting
end

function PlayerService.Cleanup()
    -- runs on destroy before connections are cleaned
    -- custom teardown before ArTex disconnects everything
end

return PlayerService
```

### Context Structure

`ArTex.configure()` receives a `ContextStructure` that tells ArTex where your modules live and how to namespace them in `ctx`.

```lua
ContextStructure = {
    Server = {
        Host      = ServerStorage.Modules,
        Structure = { "Services", "Managers", "Utility" },
    },
    Client = {
        Host      = ReplicatedStorage.Client.Modules,
        Structure = { "Controllers", "Services", "Utility" },
    },
    Shared = {
        Host      = ReplicatedStorage.Shared.Modules,
        Structure = { "Packages", "Utility" },
    },
}
```

ArTex scans each `Host` folder, finds all ModuleScripts inside each `Structure` subfolder, and registers them under the matching namespace. A module found at `ServerStorage.Modules.Services.PlayerService` becomes `ctx.Services.PlayerService` on the server.

### Lifecycle

ArTex runs every module through three phases in strict tier order:

```
── Tier: Shared ─────────────────────────────
   Warmup()   all modules, synchronous, no ctx
   Init(ctx)  all modules, ctx available
   Start(ctx) all modules, world live

── Tier: Services ───────────────────────────
   Warmup → Init → Start

── Tier: Managers ───────────────────────────
   Warmup → Init → Start

── Tier: Controllers ─────────────────────────
   Warmup → Init → Start
```

Each phase is latch-gated. Every module in a tier must complete a phase before any module in that tier advances to the next phase. This guarantees that when `Start` runs, every peer module's `Init` has already completed.

### Dependency Injection

Every module's `Start(ctx)` receives the fully-assembled context table. Nothing is required directly inside modules. The full ctx shape:

```lua
-- always present
ctx.ArTex             -- ArTex itself
ctx.Cache             -- built-in reactive cache
ctx.Context           -- "Server" | "Client"
ctx.Env               -- alias for ctx, the full table
ctx.Net               -- NetworkManager on server, NetworkController on client
ctx.Log               -- scoped logger respecting this module's Verbosity
ctx.Module            -- this module's own table
ctx.Name              -- this module's registered name
ctx.Tier              -- this module's tier string
ctx.Version           -- ArTex version string
ctx.Manifest          -- from configure()
ctx.NetworkManager    -- server network facilitator
ctx.NetworkController -- client network facilitator
ctx.Server            -- server metadata: PlaceId, JobId, PlayerCount
ctx.Player            -- LocalPlayer on client, nil on server
ctx.Theme             -- active theme if Themes plugin installed
ctx.i18n              -- locale resolution if I18n plugin installed
ctx.Span              -- current active tracing span if inside traced operation
ctx.Perf              -- performance utilities if Performance plugin installed
ctx.Gateway           -- gateway shorthand if Gateway plugin installed

-- namespaced from ContextStructure
ctx.Services.PlayerService
ctx.Managers.CombatManager
ctx.Controllers.NetworkController
ctx.Packages.Fabrik
ctx.Packages.Keep
ctx.Utility.Promise
```

### Priority System

Instead of setting `BootPriority` on every module, define a priority map once after all `ArTex.Inside()` calls:

```lua
ArTex.priority({
    Services = {
        "DataService",       -- boots first
        "PlayerService",     -- boots second
        "CharacterService",  -- boots third
        -- anything not listed falls to end in scan order
    },
    Managers = {
        "NetworkManager",
        "CombatManager",
        "NotificationManager",
    },
    Controllers = {
        "NetworkController",
        "NotificationController",
        "CharacterController",
        "WatchdogController",
    },
})
```

Priority resolution order:
1. `ArTex.priority()` map — position in array is the priority
2. `Module.BootPriority` — overrides the map for that specific module
3. `Module.After` — soft ordering within the same priority level
4. Folder scan order — final fallback

### Middleware

Middleware intercepts every lifecycle phase for every module. Each middleware receives a context object and a `next` function. You must call `next()` to continue the pipeline.

```lua
-- global — fires for every module every phase
ArTex.middleware(function(ctx, next)
    local t = os.clock()
    next()
    local ms = (os.clock() - t) * 1000
    if ms > 50 then
        warn(string.format("[ArTex] slow: %s.%s took %.2fms", ctx.module, ctx.phase, ms))
    end
end)

-- phase-specific
ArTex.middleware("start", function(ctx, next)
    print("[ArTex] starting", ctx.module)
    next()
end)
```

Middleware `ctx` fields:

| Field | Type | Description |
|---|---|---|
| `module` | `string` | The module name |
| `phase` | `string` | `"warmup"` \| `"init"` \| `"start"` \| `"destroy"` |
| `tier` | `string` | The module's tier |
| `context` | `string` | `"Server"` \| `"Client"` |
| `time` | `number` | `os.clock()` at phase start |

### Plugin System

Plugins extend ArTex itself. `ArTex.install()` returns a live handle with the methods the plugin exposes. Plugins can hook into middleware, extend the ArTex API surface via `artext.extend()`, and declare their own plugin dependencies.

```lua
local MyPlugin = {}
MyPlugin.name     = "MyPlugin"
MyPlugin.version  = "1.0.0"
MyPlugin.requires = { "Profiler" } -- ArTex auto-installs required plugins

function MyPlugin:install(artext, opts)
    -- hook into lifecycle
    artext.middleware("start", function(ctx, next)
        next()
    end)

    -- hook into plugin lifecycle events
    artext.onCompileStart(function() end)
    artext.onCompileEnd(function() end)
    artext.onModuleStart(function(name) end)
    artext.onModuleError(function(name, phase, err) end)
    artext.onCacheChange(function(key, new, old) end)

    -- extend ArTex API surface
    artext.extend("myplugin", {
        report = function() end,
        reset  = function() end,
    })

    -- return handle methods
    return {
        report = function() end,
        reset  = function() end,
    }
end

return MyPlugin
```

### Cache

ArTex ships a built-in reactive cache powered by Fabrik atoms. It lives on `ctx.Cache` and `ArTex.Cache`. Every write fires watchers. TTL keys auto-expire. Computed keys recompute when dependencies change.

```lua
-- basic
ArTex.Cache.set("RoundState", "lobby")
ArTex.Cache.get("RoundState")              -- "lobby"
ArTex.Cache.has("RoundState")             -- true
ArTex.Cache.delete("RoundState")
ArTex.Cache.clear()

-- reactive
ArTex.Cache.watch("RoundState", function(new, old)
    print("state changed:", old, "->", new)
end)

-- derived
ArTex.Cache.computed("IsInCombat", function()
    return ArTex.Cache.get("RoundState") == "combat"
end, { "RoundState" })

-- TTL
ArTex.Cache.ttl("SessionToken", "abc123", 300)
ArTex.Cache.ttlRemaining("SessionToken")
ArTex.Cache.refresh("SessionToken", 300)
ArTex.Cache.onExpire("SessionToken", function() end)
ArTex.Cache.expire("SessionToken")

-- transactions and locking
ArTex.Cache.lock("PlayerHP")
ArTex.Cache.transaction(function()
    local hp     = ArTex.Cache.get("PlayerHP")
    local shield = ArTex.Cache.get("PlayerShield")
    ArTex.Cache.set("PlayerHP",     math.max(0, hp - 25))
    ArTex.Cache.set("PlayerShield", math.max(0, shield - 10))
end)
ArTex.Cache.atomic("PlayerHP", function(current)
    return math.max(0, current - 25)
end)

-- namespacing
local PlayerCache = ArTex.Cache.namespace("Player")
PlayerCache.set("123.HP", 100)
PlayerCache.get("123.HP")

local LocalCache = ArTex.Cache.localNamespace(player)
LocalCache.set("HP", 100)
```

### Networking

ArTex does not ship a NetworkManager or NetworkController. You write your own using ArTex's Schema and Fabrik interfaces, pass them into `configure()`, and ArTex injects them into every module's ctx.

```lua
-- NetworkManager.lua — your code, built on ArTex
local NetworkManager = {}
NetworkManager.Context      = "Server"
NetworkManager.BootPriority = 0

local _ArTex

function NetworkManager.Start(ctx)
    _ArTex = ctx.ArTex

    local T = _ArTex.Type

    NetworkManager.Channels = {
        Player = _ArTex.channel("Player.Service", {
            Signature = T.string(),
            Data      = T.table(),
        }, { mode = "reliable" }),
    }
end

function NetworkManager.Post(channel, player, data) ... end
function NetworkManager.Handle(channel, callback) ... end
function NetworkManager.Subscribe(channel, callback) ... end
function NetworkManager.Broadcast(channel, data) ... end

return NetworkManager
```

### Reactive State

ArTex exposes Fabrik's full reactive primitive suite directly. Atoms, molecules, and organisms are first-class citizens in module code.

```lua
function CombatManager.Init(ctx)
    _ArTex = ctx.ArTex

    -- atom
    self.hp = _ArTex.atom(100)
    self.hp:onChange(function(new)
        if new <= 0 then self:onDeath() end
    end)

    -- molecule
    self.stats = _ArTex.molecule({
        hp      = _ArTex.atom(100),
        shield  = _ArTex.atom(50),
        stamina = _ArTex.atom(100),
    })

    -- derived atom
    self.isDead = self.hp:derive(function(v) return v <= 0 end)

    -- batched mutation
    _ArTex.batch(function()
        self.stats.hp:set(80)
        self.stats.shield:set(40)
    end)
end
```

---

## API Reference

### ArTex.configure

```lua
ArTex.configure(opts: ConfigureOptions)
```

Must be called before any other ArTex method. Can only be called once unless `ArTex.reset()` has been called.

| Field | Type | Default | Description |
|---|---|---|---|
| `Schema` | `Schema` | required | Schema library instance |
| `Fabrik` | `Fabrik` | required | Fabrik library instance |
| `NetworkManager` | `table?` | nil | Server-side network facilitator |
| `NetworkController` | `table?` | nil | Client-side network facilitator |
| `Manifest` | `table?` | nil | Injected as `ctx.Manifest` |
| `ContextStructure` | `table` | required | Folder structure definition |
| `Verbosity` | `string` | `"warn"` | `"trace"` \| `"warn"` \| `"error"` \| `"silent"` |
| `Timeout` | `number` | `30` | Seconds before a phase is considered hung |
| `Strict` | `boolean` | `false` | Turn soft tier warnings into hard errors |
| `TierOrder` | `{ string }?` | nil | Custom tier boot order |

```lua
ArTex.configure({
    Schema = Schema,
    Fabrik = Fabrik,

    NetworkManager    = NetworkManager,
    NetworkController = NetworkController,
    Manifest          = Manifests,

    Verbosity = "trace",
    Timeout   = 30,
    Strict    = false,

    ContextStructure = {
        Server = {
            Host      = ServerStorage.Modules,
            Structure = { "Services", "Managers", "Utility" },
        },
        Client = {
            Host      = ReplicatedStorage.Client.Modules,
            Structure = { "Controllers", "Services", "Utility" },
        },
        Shared = {
            Host      = ReplicatedStorage.Shared.Modules,
            Structure = { "Packages", "Utility" },
        },
    },
})
```

---

### ArTex.Inside

```lua
ArTex.Inside(context: string?, namespace: string?, name: string, value: any)
```

Manually inject a value into ctx. Use for packages and utilities that don't live in a scanned folder.

| Parameter | Description |
|---|---|
| `context` | `"Server"` \| `"Client"` \| `"Shared"` \| `nil` (root, both sides) |
| `namespace` | `"Packages"` \| `"Utility"` \| `nil` (root ctx) |
| `name` | The key on ctx |
| `value` | Any value |

```lua
-- server only, namespaced
ArTex.Inside("Server", "Packages", "Keep", Keep)
-- → ctx.Packages.Keep on server

-- both sides, namespaced
ArTex.Inside("Shared", "Packages", "Fabrik", Fabrik)
-- → ctx.Packages.Fabrik on both

-- both sides, root ctx
ArTex.Inside(nil, nil, "GameVersion", "1.0.0")
-- → ctx.GameVersion on both
```

---

### ArTex.priority

```lua
ArTex.priority(map: { [string]: { string } })
```

Define boot order for entire tiers in one call. Position in the array equals priority. Modules not listed fall to the end in scan order. Call after all `ArTex.Inside()` calls and before `ArTex.compile()`.

```lua
ArTex.priority({
    Services = {
        "DataService",
        "PlayerService",
        "CharacterService",
    },
    Managers = {
        "NetworkManager",
        "CombatManager",
        "NotificationManager",
    },
    Controllers = {
        "NetworkController",
        "NotificationController",
        "CharacterController",
    },
})
```

---

### ArTex.compile

```lua
ArTex.compile(opts?: CompileOptions) -> Promise
```

Scans all registered folders, resolves boot order, runs the full lifecycle, and returns a Fabrik promise that resolves when all modules have started.

| Option | Type | Description |
|---|---|---|
| `dry` | `boolean` | Resolve and print boot order without running any lifecycle |

```lua
-- dry run — prints resolved order without booting
ArTex.compile({ dry = true })

-- real compile
ArTex.compile():next(function()
    print("all modules live")
end):catch(function(err)
    warn("compile failed:", err)
end)
```

---

### ArTex.middleware

```lua
ArTex.middleware(phase?: string, fn: (ctx, next) -> ())
```

Add a middleware function to the lifecycle pipeline. If `phase` is omitted, fires for every phase.

```lua
-- global
ArTex.middleware(function(ctx, next)
    next()
end)

-- phase-specific
ArTex.middleware("start", function(ctx, next)
    next()
end)
```

---

### ArTex.install

```lua
ArTex.install(plugin: Plugin, opts?: table) -> PluginHandle
```

Install a plugin. Returns a live handle with the methods the plugin exposes.

```lua
local profiler  = ArTex.install(ArTex.Plugins.Profiler)
local inspector = ArTex.install(ArTex.Plugins.Inspector)
local guard     = ArTex.install(ArTex.Plugins.Guard, { strict = true })

profiler.report()
profiler.flamegraph()
inspector.graph()
```

---

### ArTex.open

```lua
ArTex.open(pluginName: string, methodName: string, ...) -> any
```

Call a plugin method by name without holding the handle. Useful when the handle isn't in scope.

```lua
ArTex.open("Profiler", "report")
ArTex.open("Inspector", "graph")
```

---

### ArTex.plug

```lua
ArTex.plug(name: string) -> PluginHandle?
```

Retrieve a plugin handle by name after installation.

```lua
local profiler = ArTex.plug("Profiler")
profiler.report()
```

---

### ArTex.extend

```lua
ArTex.extend(name: string, methods: { [string]: (...any) -> any })
```

Extend the ArTex API surface directly. Works from inside modules or plugins. After calling this, `ArTex[name]` is the methods table.

```lua
-- inside CombatManager.Start(ctx)
ctx.ArTex.extend("combat", {
    onHit   = function(fn) return _onHit:connect(fn) end,
    onDeath = function(fn) return _onDeath:connect(fn) end,
})

-- from anywhere after compile
ArTex.combat.onHit(function(attacker, target, damage) end)
```

---

### Module Access

```lua
ArTex.get(name: string) -> any
```
Synchronous module lookup. Safe after compile. Errors if called before `Init` phase or if module not found.

```lua
ArTex.has(name: string) -> boolean
```
Returns true if a module with `name` is registered.

```lua
ArTex.await(name: string) -> Promise
```
Returns a Fabrik promise that resolves when the named module has finished its `Start` phase. Use this inside `Init` or `Start` to wait for a peer without declaring a hard dependency.

```lua
ArTex.modules() -> { { name: string, status: string, tier: string, timing: number } }
```
Returns a list of all registered modules with their current status.

```lua
ArTex.status(name: string) -> string
```
Returns `"pending"` | `"warmup"` | `"init"` | `"starting"` | `"started"` | `"errored"` | `"skipped"` | `"destroyed"` for the named module.

```lua
ArTex.context() -> string
```
Returns `"Server"` or `"Client"` from anywhere.

```lua
ArTex.get("PlayerService")
ArTex.has("CombatManager")
ArTex.await("DataService"):next(function() end)
ArTex.status("PlayerService")   -- "started"
ArTex.context()                 -- "Server"
```

---

### Events

```lua
ArTex.loaded(fn: (modules: { ModuleInfo }) -> ())
```
Fires once when all modules have started. Receives the full module list.

```lua
ArTex.onError(fn: (moduleName: string, phase: string, err: any) -> ())
```
Fires whenever a module errors during any lifecycle phase.

```lua
ArTex.onModuleLoaded(fn: (moduleName: string) -> ())
```
Fires per module as each one finishes its `Start` phase.

```lua
ArTex.onPhaseStart(phase: string, fn: () -> ())
```
Fires before a lifecycle phase begins globally.

```lua
ArTex.onPhaseEnd(phase: string, fn: () -> ())
```
Fires after a lifecycle phase completes globally.

```lua
ArTex.onProgress(fn: (percent: number) -> ())
```
Fires during compile with percent complete.

```lua
ArTex.loaded(function(modules)
    for _, mod in modules do
        print(mod.name, mod.status, mod.timing)
    end
end)

ArTex.onError(function(name, phase, err)
    warn(name, "errored during", phase, err)
end)

ArTex.onModuleLoaded(function(name)
    print(name, "started")
end)

ArTex.onProgress(function(pct)
    print(string.format("boot: %d%%", math.floor(pct * 100)))
end)
```

---

### Diagnostics

```lua
ArTex.report() -> { [string]: number }
```
Returns a flat timing table — module name to milliseconds.

```lua
ArTex.trace() -> { { timestamp: number, module: string, phase: string, duration: number } }
```
Returns the full ordered event log of every lifecycle event.

```lua
ArTex.health() -> { [string]: { status: string, uptime: number, restarts: number } }
```
Returns live health status of every module post-boot.

```lua
ArTex.inspect(name: string) -> ModuleMetadata
```
Returns full metadata for a module: tier, status, timing, tags, alias, context, exports, schema channels, version.

```lua
ArTex.compare(nameA: string, nameB: string) -> { [string]: any }
```
Returns a side-by-side comparison of two modules' metadata.

```lua
ArTex.snapshot() -> StateSnapshot
```
Full point-in-time capture of ctx, cache, module statuses.

```lua
ArTex.diff(snapshot1: StateSnapshot, snapshot2: StateSnapshot) -> { [string]: any }
```
Compares two snapshots and returns what changed.

```lua
ArTex.emit(event: string, data: any)
```
Manually emit a diagnostic event into the trace log.

```lua
ArTex.diagnostics = {
    bootTime     = number,
    moduleCount  = number,
    errorCount   = number,
    skippedCount = number,
    slowest      = string,
    fastest      = string,
    byTier       = { [string]: { count: number, totalTime: number } },
}
```

Sample diagnostics output at `Verbosity = "trace"`:

```
[ArTex] Server | v1.0.0
[ArTex] Resolved 14 modules across 3 tiers
[ArTex] ── Shared ─────────────────────────────
[ArTex]   Fabrik          warmup   0.01ms  ✓
[ArTex]   Manifests       warmup   0.02ms  ✓
[ArTex]   Fabrik          init     0.04ms  ✓
[ArTex]   Fabrik          start    0.03ms  ✓
[ArTex] ── Services ───────────────────────────
[ArTex]   DataService     warmup   0.05ms  ✓
[ArTex]   DataService     init     0.12ms  ✓
[ArTex]   DataService     start    1.23ms  ✓
[ArTex]   PlayerService   start    2.10ms  ✓
[ArTex] ── Managers ───────────────────────────
[ArTex]   NetworkManager  start    0.44ms  ✓
[ArTex]   CombatManager   start    0.88ms  ✓
[ArTex] ────────────────────────────────────────
[ArTex] Boot complete in 5.12ms
```

---

### Async Utilities

```lua
ArTex.promise(fn)             -- Fabrik.promise
ArTex.race(promises)          -- Fabrik.promise.race
ArTex.timeout(promise, secs)  -- Fabrik.promise.timeout
ArTex.signal()                -- Fabrik.signal.new()
ArTex.atom(value)             -- Fabrik.atom
ArTex.molecule(shape)         -- Fabrik.molecule
ArTex.batch(fn)               -- Fabrik.batch
ArTex.latch()                 -- Fabrik.latch.new()
ArTex.interval(secs, fn)      -- Fabrik.thread.interval, auto-cancelled on module destroy
ArTex.debounce(fn, secs)      -- Fabrik.cooldown.debounce
ArTex.throttle(fn, secs)      -- Fabrik.cooldown.throttle
ArTex.once(event, fn)         -- one-shot version of ArTex.loaded / onModuleLoaded
ArTex.measure(fn)             -- runs fn and returns elapsed milliseconds
ArTex.defer(fn)               -- runs after compile settles, before loaded fires
```

---

### Network Utilities

These are thin wrappers that delegate to your configured `NetworkManager` or `NetworkController`. They are available on `ArTex` and on `ctx.Net` — `ctx.Net` automatically resolves to the correct side.

```lua
ctx.Net.on(channel, fn)              -- subscribe shorthand
ctx.Net.send(channel, data)          -- post shorthand
ctx.Net.request(channel, data)       -- invoke shorthand, returns thenable
ctx.Net.broadcast(channel, data)     -- postAll shorthand, server only
ctx.Net.intercept(channel, fn)       -- per-module network intercept
```

---

### Schema Utilities

```lua
ArTex.channel(name, shape, opts)     -- Schema.channel
ArTex.define(name, shape, opts)      -- Schema.define
ArTex.post(name, ...)                -- Schema.post
ArTex.subscribe(name, fn)            -- Schema.subscribe
ArTex.party(name)                    -- Schema.party
ArTex.validate(name, data)           -- Schema.validate, returns (boolean, string?)
ArTex.serialize(name, data)          -- validate + serialize
ArTex.typedef(name, shape)           -- register a reusable type alias
ArTex.Type                           -- Schema.Type factory shorthand
ArTex.route(name, shape, opts)       -- defines channel on both server and client automatically
ArTex.intercept(name, fn)            -- middleware for a specific network channel
ArTex.firewall(rules)                -- block channels by rule
ArTex.allow(channel)                 -- explicitly allow a channel
ArTex.deny(channel)                  -- explicitly deny a channel
```

---

### Reactive Utilities

```lua
ArTex.useState(key, initial)         -- declare reactive state tied to module lifecycle
ArTex.useAtom(initial)               -- Fabrik.atom tied to module lifecycle, auto-cleaned
ArTex.useMolecule(shape)             -- Fabrik.molecule tied to module lifecycle, auto-cleaned
ArTex.bind(key, instance, property)  -- bind cache key to Roblox Instance property
ArTex.unbind(key)                    -- remove a binding
ArTex.bindings                       -- all active bindings
```

---

### Cache API

```lua
-- core
ArTex.Cache.set(key, value)
ArTex.Cache.get(key)
ArTex.Cache.has(key)
ArTex.Cache.delete(key)
ArTex.Cache.clear()
ArTex.Cache.size()
ArTex.Cache.keys(pattern?)
ArTex.Cache.values()
ArTex.Cache.snapshot()

-- mutations
ArTex.Cache.extend(key, field, value)   -- shallow merge into cache table
ArTex.Cache.merge(key, table)           -- deep merge into cache table
ArTex.Cache.clone(key)                  -- deep clone without reference
ArTex.Cache.increment(key, amount?)
ArTex.Cache.append(key, value)          -- append to array cache entry
ArTex.Cache.diff(key, newValue)         -- returns diff before setting

-- reactive
ArTex.Cache.watch(key, fn)              -- fires on every change, returns disconnect fn
ArTex.Cache.subscribe(key, fn)          -- alias for watch
ArTex.Cache.computed(key, fn, deps)     -- derived value, recomputes when deps change
ArTex.Cache.batch(fn)                   -- group mutations, fire watchers once
ArTex.Cache.bind(key, instance, prop)   -- bind to Roblox Instance property

-- TTL
ArTex.Cache.ttl(key, value, seconds)
ArTex.Cache.ttlRemaining(key)
ArTex.Cache.refresh(key, seconds)
ArTex.Cache.expire(key)
ArTex.Cache.onExpire(key, fn)

-- advanced
ArTex.Cache.lock(key)                   -- Fabrik.lock on a key
ArTex.Cache.transaction(fn)             -- atomic multi-key update
ArTex.Cache.atomic(key, fn)             -- read-modify-write atomically
ArTex.Cache.namespace(prefix)           -- scoped cache view
ArTex.Cache.localNamespace(player)      -- per-player scoped cache
ArTex.Cache.persist(key)                -- survives recompile
ArTex.Cache.getOrSet(key, defaultFn)    -- get or compute and store
ArTex.Cache.index(field)                -- create index on a field for fast lookup
ArTex.Cache.find(field, value)          -- find entries by indexed field
ArTex.Cache.evict(pattern)             -- delete all keys matching pattern
ArTex.Cache.tagged(tag)                -- get all keys with a tag
ArTex.Cache.count(fn?)                 -- count entries matching predicate
ArTex.Cache.map(fn)                    -- map all entries to new table
ArTex.Cache.filter(fn)                 -- filter entries by predicate
ArTex.Cache.reduce(fn, initial)        -- reduce all entries to a value
ArTex.Cache.forEach(fn)               -- iterate all entries
ArTex.Cache.search(query)             -- search keys and values by query string
```

---

### Boot Control

```lua
ArTex.precompile(fn)              -- runs before scan, synchronous
ArTex.postcompile(fn)             -- runs after all modules started, alias for defer
ArTex.recompile()                 -- full teardown and reboot, Studio only
ArTex.partial(tier)               -- compile only one tier
ArTex.phase()                     -- current phase name during compile
ArTex.checkpoint(name)            -- named snapshot during compile
ArTex.rollback(name)              -- revert to named checkpoint
ArTex.disable(name)               -- programmatically disable a module before compile
ArTex.enable(name)                -- re-enable a disabled module
ArTex.retry(name)                 -- retry a failed module without full recompile
ArTex.replace(name, module)       -- swap a module before compile
ArTex.freeze()                    -- lock ctx after compile, no further mutations
ArTex.seal()                      -- lock ArTex itself, no further configure/install calls
ArTex.reset()                     -- full teardown to pre-configure state, Studio only
ArTex.reconfigure(opts)           -- update non-structural configure options at runtime
ArTex.config                      -- current configure values
ArTex.config.get(key)             -- read a specific configure value
ArTex.isReady()                   -- boolean, true after compile
ArTex.uptime()                    -- seconds since compile completed
ArTex.time()                      -- os.clock() relative to compile start
ArTex.version                     -- version string e.g. "1.0.0"
ArTex.ready                       -- Fabrik latch, open after compile
```

---

### Testing and Studio Utilities

```lua
ArTex.mock(name, mockModule)      -- replace a module with a mock
ArTex.isMocked(name)              -- true if module is currently mocked
ArTex.isolate(name)               -- run a module with a clean ctx
ArTex.spy(name, method)           -- wrap a method to log all calls, Studio only
ArTex.unspy(name, method)         -- remove spy
ArTex.observe(name, method, fn)   -- non-destructive spy, never wraps
ArTex.pin(name)                   -- prevent disable/mock/replace
ArTex.unpin(name)                 -- remove pin
ArTex.benchmark(name)             -- run Start in isolation, return timing
ArTex.test(name, fn)              -- register a test
ArTex.runTests()                  -- run all tests, returns promise with results
ArTex.expect(value)               -- assertion helper inside tests
ArTex.export(format?)             -- export full state as table or JSON
ArTex.import(data, format?)       -- import state, Studio only
```

---

### Logging

```lua
ArTex.log(level, message)         -- manual log, respects global Verbosity
ArTex.print(message)              -- always prints regardless of Verbosity
ArTex.warn(message)               -- respects Verbosity
ArTex.error(message)              -- respects Verbosity, fires onError signal
ArTex.assert(condition, message)  -- respects Verbosity, cleaner than raw assert
ArTex.protect(fn)                 -- wraps fn in pcall, fires onError on failure

-- ctx.Log is a scoped logger that respects the current module's Verbosity
ctx.Log.trace(message)
ctx.Log.info(message)
ctx.Log.warn(message)
ctx.Log.error(message)
```

---

### Feature Flags

```lua
ArTex.flag(name)                  -- read a feature flag, returns boolean
ArTex.setFlag(name, value)        -- set a feature flag
ArTex.flags                       -- all active flags table
```

Modules declare which flags they require:

```lua
MyModule.Flags = { "NewCombatSystem", "BetaUI" }
-- module only boots if all listed flags are true
```

---

### Scheduling

```lua
ArTex.schedule(interval, fn)      -- recurring task, returns id
ArTex.schedules                   -- all active schedules
ArTex.cancelSchedule(id)          -- cancel a schedule

-- inline on module
MyModule.Schedule = { interval = 60, fn = "tick" }
-- calls MyModule.tick() every 60 seconds, auto-cancelled on destroy
```

---

### Metrics

```lua
ArTex.metric(name, type)          -- register: "counter" | "gauge" | "histogram"
ArTex.metrics                     -- all registered metrics
ArTex.increment(metric, amount?)  -- increment a counter
ArTex.gauge(metric, value)        -- set a gauge
ArTex.histogram(metric, value)    -- record a histogram sample
```

---

### Commands and Queries

```lua
ArTex.command(name, fn)           -- register a write command
ArTex.query(name, fn)             -- register a read query
ArTex.dispatch(name, data)        -- dispatch a command, returns promise
ArTex.ask(name, data)             -- execute a query, returns promise

ArTex.ask("PlayerService.GetProfile", { userId = 123 }):next(function(profile)
    print(profile.Archive.Level)
end)
```

---

### Workflows

```lua
ArTex.workflow(name, steps)        -- define a multi-step async workflow
ArTex.runWorkflow(name, data)      -- execute a workflow, returns promise
ArTex.workflows                    -- all registered workflows
ArTex.cancelWorkflow(id)           -- cancel a running workflow instance
ArTex.workflowStatus(id)           -- "pending" | "running" | "done" | "cancelled" | "failed"

ArTex.workflow("OnboardPlayer", {
    function(ctx) return DataService:LoadProfile(ctx.player) end,
    function(ctx) return PlayerService:AssignSpawn(ctx.player) end,
    function(ctx) return NotificationManager.Send(ctx.player, { Text = "Welcome!" }) end,
})

ArTex.runWorkflow("OnboardPlayer", { player = player }):next(function()
    print("onboarding complete")
end)
```

---

### Global Events

```lua
ArTex.event(name)                 -- register a named global event
ArTex.emit(name, data)            -- fire a named global event
ArTex.on(name, fn)                -- listen to a named global event
ArTex.off(name, fn)               -- remove listener
```

---

### Tracing and Spans

```lua
ArTex.span(name)                  -- open a named profiler span
ArTex.endSpan(name)               -- close a span
ArTex.spans                       -- all active spans

-- inside a traced operation
ctx.Span                          -- current active span
```

---

### Graph Utilities

```lua
ArTex.graph()                     -- full dependency graph as table
ArTex.graph.roots()               -- modules with no dependencies
ArTex.graph.leaves()              -- modules nothing depends on
ArTex.graph.path(from, to)        -- shortest dependency path between two modules
ArTex.graph.cycles()              -- all detected cycles (should be empty)
```

---

### Registry

```lua
ArTex.get(name)                   -- get a module or its alias
ArTex.has(name)                   -- boolean
ArTex.isModule(name)              -- true if name is registered
ArTex.getTagged(tag)              -- all modules with matching tag
ArTex.getTier(name)               -- "Services" | "Managers" | "Controllers" etc
ArTex.getContext(name)            -- "Server" | "Client" | "Shared"
ArTex.tag(name, tags)             -- add tags to a module post-registration
ArTex.untag(name, tags)           -- remove tags
ArTex.schema(name)                -- Schema channels registered by a module
ArTex.exports(name)               -- Exports whitelist of a module
ArTex.find(fn)                    -- first module matching predicate
ArTex.filter(fn)                  -- all modules matching predicate
ArTex.search(query)               -- search module names, tags, aliases
ArTex.clone(name)                 -- clone a module into a new instance
ArTex.inherit(name, overrides)    -- create module inheriting from another
ArTex.fork()                      -- child ArTex instance inheriting parent config
ArTex.instance(name)              -- create or get a named ArTex instance
ArTex.instances                   -- all active instances
ArTex.primary                     -- the default instance
```

---

## ctx Reference

| Key | Type | Description |
|---|---|---|
| `ctx.ArTex` | `ArTex` | ArTex itself |
| `ctx.Cache` | `Cache` | Built-in reactive cache |
| `ctx.Context` | `string` | `"Server"` \| `"Client"` |
| `ctx.Env` | `table` | Alias for ctx, the full table |
| `ctx.Net` | `table` | NetworkManager on server, NetworkController on client |
| `ctx.Log` | `Logger` | Scoped logger, respects module Verbosity |
| `ctx.Module` | `table` | This module's own table |
| `ctx.Name` | `string` | This module's registered name |
| `ctx.Tier` | `string` | This module's tier |
| `ctx.Version` | `string` | ArTex version string |
| `ctx.Manifest` | `table` | From configure() |
| `ctx.NetworkManager` | `table?` | Server network facilitator |
| `ctx.NetworkController` | `table?` | Client network facilitator |
| `ctx.Server` | `table` | PlaceId, JobId, PlayerCount |
| `ctx.Player` | `Player?` | LocalPlayer on client |
| `ctx.Theme` | `table?` | Active theme (Themes plugin) |
| `ctx.i18n` | `table?` | Locale resolution (I18n plugin) |
| `ctx.Span` | `table?` | Current tracing span |
| `ctx.Perf` | `table?` | Performance utilities |
| `ctx.Gateway` | `table?` | Gateway shorthand |
| `ctx.Services.*` | `table` | All services |
| `ctx.Managers.*` | `table` | All managers |
| `ctx.Controllers.*` | `table` | All controllers |
| `ctx.Packages.*` | `table` | All packages |
| `ctx.Utility.*` | `table` | All utilities |

---

## Module Fields Reference

| Field | Type | Default | Description |
|---|---|---|---|
| `Context` | `string?` | inferred | `"Server"` \| `"Client"` \| `"Shared"` |
| `Alias` | `string?` | nil | Short name, `ArTex.get("Alias")` works |
| `Version` | `string?` | nil | Semver string |
| `Description` | `string?` | nil | Shown in Inspector and Docs |
| `Tags` | `{ string }?` | nil | Used with `ArTex.getTagged()` |
| `NoBoot` | `boolean?` | false | Skip entirely |
| `Boot` | `boolean?` | false | Shared modules: opt-in |
| `BootPriority` | `number?` | nil | Overrides priority map |
| `After` | `{ string }?` | nil | Soft ordering within tier |
| `Optional` | `boolean?` | false | Errors swallowed |
| `Critical` | `boolean?` | false | Error halts compile |
| `Retry` | `number?` | 0 | Auto-retry count |
| `MaxRestarts` | `number?` | 3 | Watchdog restart limit |
| `Timeout` | `number?` | global | Per-module timeout |
| `Singleton` | `boolean?` | true | Allow multiple instances |
| `Verbosity` | `string?` | global | Per-module log level |
| `Schema` | `table?` | nil | Inline channel definitions |
| `Exports` | `{ string }?` | nil | Method whitelist for Guard |
| `Flags` | `{ string }?` | nil | Required feature flags |
| `Store` | `table?` | nil | Inline datastore definition |
| `Schedule` | `table?` | nil | Inline recurring task |
| `Warmup` | `function?` | nil | Pre-init synchronous hook |
| `Init` | `function?` | nil | Post-warmup hook, ctx available |
| `Start` | `function?` | nil | Main lifecycle hook |
| `Cleanup` | `function?` | nil | Pre-destroy hook |

---

## First-Party Plugins

| Plugin | Description |
|---|---|
| `ArTex.Plugins.Profiler` | Deep timing breakdowns per module per phase with flame graph data and nested spans |
| `ArTex.Plugins.Inspector` | Live module graph, dependency tree, status of every module, ctx viewer |
| `ArTex.Plugins.Guard` | Strict tier enforcement and Exports whitelist validation |
| `ArTex.Plugins.HotReload` | Studio only — reruns Warmup, Init, Start on module change without restarting |
| `ArTex.Plugins.DevTools` | Studio widget — module graph, cache inspector, signal monitor, network monitor, boot timeline |
| `ArTex.Plugins.Logger` | Structured log output with filtering by module, tier, phase, level |
| `ArTex.Plugins.Analytics` | Boot metrics, session metrics, funnel analysis |
| `ArTex.Plugins.Sandbox` | Isolates a module's ctx to only explicitly allowed modules |
| `ArTex.Plugins.RateLimit` | Per-player, per-channel, and per-method rate limiting |
| `ArTex.Plugins.Circuit` | Circuit breaker — disables a module after N consecutive errors |
| `ArTex.Plugins.Watchdog` | Monitors module health post-boot, restarts unresponsive modules |
| `ArTex.Plugins.Bridge` | Cross-server communication via MessagingService, built on ArTex channels |
| `ArTex.Plugins.Compat` | Import Knit services and controllers directly into ArTex with zero rewrite |
| `ArTex.Plugins.Input` | Knit input library integration — action registry, context stack, sessions on ctx |
| `ArTex.Plugins.Memory` | Tracks memory usage per module pre and post Start |
| `ArTex.Plugins.Replay` | Records all signals and network events, replays them, Studio only |
| `ArTex.Plugins.Audit` | Logs every module method call with caller, args, and timing |
| `ArTex.Plugins.Telemetry` | Sends boot and error data to an external endpoint via HttpService |
| `ArTex.Plugins.Permissions` | Role-based access control on module Exports |
| `ArTex.Plugins.StateSync` | Syncs ArTex Cache to client automatically via Schema |
| `ArTex.Plugins.Rollback` | On compile failure, reverts to last successful compile snapshot |
| `ArTex.Plugins.Queue` | Persistent job queue built on Fabrik.queue, survives recompile |
| `ArTex.Plugins.MockNetwork` | Routes all Schema channels through a mock layer, zero real remotes |
| `ArTex.Plugins.Coverage` | Tracks which module methods were called during a session |
| `ArTex.Plugins.Stress` | Runs modules repeatedly to surface memory leaks and race conditions |
| `ArTex.Plugins.Docs` | Auto-generates markdown documentation from module metadata and Exports |
| `ArTex.Plugins.TimeMachine` | Full state restore to any previous snapshot including cache |
| `ArTex.Plugins.Tracing` | Distributed tracing across server and client with correlated span IDs |
| `ArTex.Plugins.I18n` | Full localization with pluralization and interpolation |
| `ArTex.Plugins.FeatureFlags` | Runtime feature toggles integrated with module boot conditions |
| `ArTex.Plugins.Chaos` | Randomly delays or errors modules to test resilience, Studio only |
| `ArTex.Plugins.Contracts` | Runtime assertion of module Exports input and output types |
| `ArTex.Plugins.Versioning` | Semver module versioning with incompatibility warnings |
| `ArTex.Plugins.Migration` | Data migrations on boot based on version changes |
| `ArTex.Plugins.RBAC` | Role-based access control integrated with PlayerService |
| `ArTex.Plugins.DataBinding` | Two-way binding between Cache values and Roblox Instance properties |
| `ArTex.Plugins.Testing` | Full unit and integration test runner integrated with lifecycle |
| `ArTex.Plugins.Themes` | Theme system for UI modules, integrated with Sleek |
| `ArTex.Plugins.Scheduler` | Recurring task scheduler integrated with module lifecycle |
| `ArTex.Plugins.LiveConfig` | Hot-reload configure() values without recompile |
| `ArTex.Plugins.DependencyGraph` | Visual and programmatic dependency graph tools |
| `ArTex.Plugins.Metrics` | Counters, gauges, and histograms per module |
| `ArTex.Plugins.Streaming` | Streaming-enabled integration, loads modules as instances stream in |
| `ArTex.Plugins.Security` | Security audit of module Exports and network channels |
| `ArTex.Plugins.Gateway` | Single entry point for all cross-service calls |
| `ArTex.Plugins.Workflow` | Multi-step async workflow engine built on Fabrik.queue |
| `ArTex.Plugins.Reactive` | Full reactive state tree built on Fabrik organism, exposed on ctx |
| `ArTex.Plugins.Performance` | Performance budget enforcement per module |
| `ArTex.Plugins.AccessLog` | Logs every module method call with caller, args, timing |
| `ArTex.Plugins.Firewall` | Blocks specific network channels based on configurable rules |
| `ArTex.Plugins.Caching` | HTTP-style response caching for network channels |
| `ArTex.Plugins.Prefetch` | Pre-warms network response cache for likely next requests |
| `ArTex.Plugins.Serialization` | Pluggable serialization formats for cache and network |
| `ArTex.Plugins.GraphSync` | Syncs module dependency graph across server and client |
| `ArTex.Plugins.Hotkeys` | Studio hotkeys for DevTools actions |
| `ArTex.Plugins.Export` | Exports full ArTex state as JSON for offline analysis |
| `ArTex.Plugins.Search` | Full-text search across module names, tags, aliases, cache keys |
| `ArTex.Plugins.MultiInstance` | Multiple named ArTex instances with full isolation |
| `ArTex.Plugins.Changelog` | Tracks API changes between module versions |
| `ArTex.Plugins.Federation` | Multi-server ArTex instances communicating via Bridge |
| `ArTex.Plugins.Compression` | Compresses large cache values automatically |
| `ArTex.Plugins.Keep` | Keep wrapper as a first-party plugin, exposes store on ctx |
| `ArTex.Plugins.DataStore` | DataStore wrapper with automatic retry and schema validation |
| `ArTex.Plugins.Animation` | TweenService wrapper integrated with module lifecycle and Cache atoms |
| `ArTex.Plugins.Sound` | SoundService wrapper with module lifecycle awareness |
| `ArTex.Plugins.Physics` | Physics utilities integrated with module lifecycle |
| `ArTex.Plugins.Pathfinding` | PathfindingService wrapper |
| `ArTex.Plugins.Camera` | Camera manipulation utilities |
| `ArTex.Plugins.LOD` | Level of detail system integrated with module lifecycle |
| `ArTex.Plugins.UI` | Sleek/App integration as a first-party plugin |
| `ArTex.Plugins.Notifications` | NotificationManager and Controller as first-party plugins |
| `ArTex.Plugins.Prompts` | PromptManager and Controller as first-party plugins |

---

## Patterns

### Basic Server Boot

```lua
-- Server.lua
local ArTex = require(ReplicatedStorage.ArTex)

ArTex.configure({
    Schema  = Schema,
    Fabrik  = Fabrik,
    Manifest = Manifests,
    NetworkManager    = require(ServerStorage.Modules.Network.NetworkManager),
    NetworkController = require(ReplicatedStorage.Client.Modules.Network.NetworkController),
    Verbosity = "trace",
    Timeout   = 30,
    ContextStructure = {
        Server = { Host = ServerStorage.Modules,            Structure = { "Services", "Managers" } },
        Client = { Host = ReplicatedStorage.Client.Modules, Structure = { "Controllers" } },
        Shared = { Host = ReplicatedStorage.Shared.Modules, Structure = { "Packages" } },
    },
})

ArTex.Inside("Server", "Packages", "Keep", Keep)

ArTex.priority({
    Services = { "DataService", "PlayerService" },
    Managers = { "NetworkManager", "CombatManager" },
})

ArTex.install(ArTex.Plugins.Profiler)

ArTex.loaded(function(modules)
    print("[ArTex] live —", #modules, "modules")
end)

ArTex.compile():next(function()
    print("boot complete in", ArTex.diagnostics.bootTime, "ms")
end)
```

---

### Custom Network Manager

```lua
-- ServerStorage.Modules.Network.NetworkManager
local NetworkManager = {}
NetworkManager.Context      = "Server"
NetworkManager.BootPriority = 0
NetworkManager.Tags         = { "Network", "Core" }

local _ArTex

function NetworkManager.Start(ctx)
    _ArTex = ctx.ArTex
    local T = _ArTex.Type

    NetworkManager.Channels = {
        Player = _ArTex.channel("Player.Service", {
            Signature = T.string(),
            Data      = T.table(),
        }, { mode = "reliable" }),

        Notification = _ArTex.channel("Notification.Generic", {
            Text     = T.string(),
            Variant  = T.string(true),
            Duration = T.number(true),
        }, { mode = "invoke", timeout = 10 }),
    }
end

function NetworkManager.Post(channel, player, data)
    if typeof(channel.Fire)  == "function" then return channel:Fire(player, data) end
    if typeof(channel.post)  == "function" then return channel.post(player, data) end
    error("[NetworkManager] channel cannot post")
end

function NetworkManager.Handle(channel, callback)
    return _ArTex.subscribe(channel, function(player, data)
        local remit   = data.remit
        local response = callback(player, data)
        if typeof(remit) == "function" then remit(response) end
    end)
end

function NetworkManager.Subscribe(channel, callback)
    return _ArTex.subscribe(channel, callback)
end

function NetworkManager.Broadcast(channel, data)
    if typeof(channel.FireAll) == "function" then channel:FireAll(data) return end
    if typeof(channel.postAll) == "function" then channel.postAll(data) return end
    error("[NetworkManager] channel cannot broadcast")
end

return NetworkManager
```

---

### Service with Reactive Cache

```lua
-- ServerStorage.Modules.Services.PlayerService
local PlayerService = {}
PlayerService.Context     = "Server"
PlayerService.Tags        = { "Core", "Player" }
PlayerService.Exports     = { "GetProfile", "WaitForProfile", "Broadcast" }
PlayerService.Description = "Manages player profiles, archive data, and lifecycle signals"

local _ArTex
local _Network
local _Keep
local _onAdded
local _onRemoving
local _onUpdated

function PlayerService.Init(ctx)
    _ArTex   = ctx.ArTex
    _onAdded    = _ArTex.signal()
    _onRemoving = _ArTex.signal()
    _onUpdated  = _ArTex.signal()
end

function PlayerService.Start(ctx)
    _Network = ctx.NetworkManager
    _Keep    = ctx.Packages.Keep

    local Players = game:GetService("Players")

    local function onPlayerAdded(player)
        _ArTex.protect(function()
            _Keep.StartPlayerSession(player):next(function(stores)
                if not player:IsDescendantOf(Players) then
                    _Keep.EndPlayerSession(player)
                    return
                end

                local account = stores.Default or stores[next(stores)]
                if not account then return end

                local profile = {
                    Player  = player,
                    Archive = account:GetAll(),
                    Folder  = {},
                }

                ctx.Cache.set("Player." .. player.UserId, profile)
                _onAdded:fire(player, profile)

                _Network.Post(_Network.Channels.Player, player, {
                    Signature = "PlayerAdded",
                    Data      = { Archive = profile.Archive },
                })
            end):catch(function(err)
                ctx.Log.error("failed to load " .. player.Name .. ": " .. tostring(err))
            end)
        end)
    end

    local function onPlayerRemoving(player)
        local profile = PlayerService.GetProfile(player)
        if profile then
            _onRemoving:fire(player, profile)
            _Network.Post(_Network.Channels.Player, player, {
                Signature = "PlayerRemoving",
                Data      = {},
            })
            ctx.Cache.delete("Player." .. player.UserId)
        end
        if _Keep.IsSessionActive(player) then
            _Keep.EndPlayerSession(player)
        end
    end

    Players.PlayerAdded:Connect(onPlayerAdded)
    Players.PlayerRemoving:Connect(onPlayerRemoving)
    for _, p in Players:GetPlayers() do task.spawn(onPlayerAdded, p) end
end

function PlayerService.GetProfile(player)
    return _ArTex.env.Cache.get("Player." .. player.UserId)
end

function PlayerService.WaitForProfile(player)
    local profile = PlayerService.GetProfile(player)
    if profile then return _ArTex.promise(function(r) r(profile) end) end
    return _onAdded:promiseWhere(function(p) return p == player end)
        :next(function(_, prof) return prof end)
end

function PlayerService.Broadcast(player, patch)
    _Network.Post(_Network.Channels.Player, player, {
        Signature = "PlayerUpdated",
        Data      = patch,
    })
    _onUpdated:fire(player, patch)
end

function PlayerService.PlayerAdded(fn)    return _onAdded:connect(fn) end
function PlayerService.PlayerRemoving(fn) return _onRemoving:connect(fn) end
function PlayerService.PlayerUpdated(fn)  return _onUpdated:connect(fn) end

function PlayerService.Watch(player, key, fn)
    return _ArTex.env.Cache.watch("Player." .. player.UserId .. "." .. key, fn)
end

return PlayerService
```

---

### Manager with Signals and Locks

```lua
-- ServerStorage.Modules.Managers.CombatManager
local CombatManager = {}
CombatManager.Context = "Server"
CombatManager.Tags    = { "Combat", "Core" }
CombatManager.After   = { "PlayerService" }

local _ArTex
local _Network
local _Players
local _locks  = {}
local _onHit
local _onDeath

function CombatManager.Init(ctx)
    _ArTex  = ctx.ArTex
    _onHit  = _ArTex.signal()
    _onDeath = _ArTex.signal()
end

function CombatManager.Start(ctx)
    _Network = ctx.NetworkManager

    _ArTex.await("PlayerService"):next(function()
        _Players = _ArTex.get("PlayerService")

        local T = _ArTex.Type
        local HitChannel = _ArTex.channel("Combat.Hit", {
            origin = T.Vector3(),
            damage = T.number(),
            target = T.Instance(),
        }, { mode = "reliable" })

        _Network.Handle(HitChannel, function(player, data)
            _ArTex.protect(function()
                CombatManager.processHit(player, data)
            end)
        end)
    end)

    ctx.ArTex.extend("combat", {
        onHit   = function(fn) return _onHit:connect(fn) end,
        onDeath = function(fn) return _onDeath:connect(fn) end,
    })
end

function CombatManager.processHit(player, data)
    local profile = _Players.GetProfile(data.target)
    if not profile then return end

    if not _locks[player.UserId] then
        _locks[player.UserId] = _ArTex.latch()
        _locks[player.UserId]:open()
    end

    _ArTex.batch(function()
        local hp = math.max(0, (profile.Archive.HP or 100) - data.damage)
        _Players.Broadcast(data.target, { HP = hp })
        _onHit:fire(player, data.target, data.damage)
        if hp <= 0 then _onDeath:fire(data.target, player) end
    end)
end

return CombatManager
```

---

### Controller with Latching

```lua
-- ReplicatedStorage.Client.Modules.Controllers.CharacterController
local CharacterController = {}
CharacterController.Context = "Client"
CharacterController.Tags    = { "Character", "Core" }
CharacterController.After   = { "NetworkController" }

local _ArTex
local _Network
local _latch
local _onSpawned

function CharacterController.Init(ctx)
    _ArTex    = ctx.ArTex
    _latch    = _ArTex.latch()
    _onSpawned = _ArTex.signal()
end

function CharacterController.Start(ctx)
    _Network = ctx.NetworkController

    _ArTex.await("PlayerServiceClient"):next(function()
        local PSC = _ArTex.get("PlayerServiceClient")
        PSC.PlayerAdded(function(profile)
            local player = profile.Player
            player.CharacterAdded:Connect(function(character)
                _onSpawned:fire(player, character)
            end)
        end)
    end)

    _latch:open()
end

function CharacterController.awaitReady()
    return _latch:promise()
end

function CharacterController.onSpawned(fn)
    return _onSpawned:connect(fn)
end

return CharacterController
```

---

### Custom Plugin

```lua
local RoundPlugin = {}
RoundPlugin.name    = "RoundTracker"
RoundPlugin.version = "1.0.0"

function RoundPlugin:install(artext, opts)
    local rounds  = 0
    local history = {}

    artext.middleware("start", function(ctx, next)
        if ctx.module == "GameManager" then
            rounds += 1
            table.insert(history, { round = rounds, time = os.time() })
        end
        next()
    end)

    artext.onCacheChange(function(key, new)
        if key == "RoundState" then
            artext.extend("rounds", {
                currentState = function() return new end,
            })
        end
    end)

    artext.extend("rounds", {
        count   = function() return rounds end,
        history = function() return table.clone(history) end,
        reset   = function() rounds = 0 history = {} end,
    })

    return {
        count   = function() return rounds end,
        history = function() return table.clone(history) end,
        reset   = function() rounds = 0 history = {} end,
        report  = function()
            print(string.format("[RoundTracker] %d rounds", rounds))
            for _, r in history do
                print(string.format("  Round %d — %d", r.round, r.time))
            end
        end,
    }
end

return RoundPlugin

-- usage
local rounds = ArTex.install(RoundPlugin)
rounds.report()
ArTex.open("RoundTracker", "report")
ArTex.rounds.count()
```

---

### Testing with Mocks

```lua
if game:GetService("RunService"):IsStudio() then
    ArTex.mock("DataService", {
        Context = "Server",
        Start   = function(ctx) end,
        LoadProfile = function(player)
            return ctx.ArTex.promise(function(resolve)
                resolve({ coins = 100, level = 1 })
            end)
        end,
    })
end

ArTex.test("PlayerService loads profile", function()
    local ps      = ArTex.get("PlayerService")
    local profile = ArTex.expect(ps.GetProfile(mockPlayer)).toBeDefined()
    ArTex.expect(profile.Archive.coins).toBe(100)
end)

ArTex.runTests():next(function(results)
    for _, r in results do
        print(r.name, r.passed and "PASS" or "FAIL", r.error or "")
    end
end)
```

---

### Feature Flags Pattern

```lua
ArTex.setFlag("NewCombatSystem", true)
ArTex.setFlag("BetaUI", false)

-- module only boots if all flags are enabled
local NewCombatManager = {}
NewCombatManager.Flags = { "NewCombatSystem" }
NewCombatManager.Context = "Server"

function NewCombatManager.Start(ctx)
    -- only runs if NewCombatSystem flag is true
end
```

---

### Workflows Pattern

```lua
ArTex.workflow("PurchaseItem", {
    function(data)
        return ArTex.ask("PlayerService.GetProfile", { player = data.player })
    end,
    function(data, profile)
        if profile.Archive.Wallet.Galla < data.price then
            error("Insufficient funds")
        end
        return ArTex.dispatch("PlayerService.DeductGalla", {
            player = data.player,
            amount = data.price,
        })
    end,
    function(data)
        return ArTex.dispatch("InventoryService.AddItem", {
            player = data.player,
            item   = data.item,
        })
    end,
    function(data)
        return ArTex.dispatch("NotificationManager.Send", {
            player = data.player,
            data   = { Text = "Purchase successful!", Variant = "Success" },
        })
    end,
})

ArTex.runWorkflow("PurchaseItem", {
    player = player,
    item   = "SwordOfDestiny",
    price  = 500,
}):next(function()
    print("purchase complete")
end):catch(function(err)
    warn("purchase failed:", err)
end)
```

---

## Exported Types

```lua
export type ConfigureOptions = {
    Schema            : any,
    Fabrik            : any,
    NetworkManager    : any?,
    NetworkController : any?,
    Manifest          : any?,
    ContextStructure  : ContextStructure,
    Verbosity         : ("trace" | "warn" | "error" | "silent")?,
    Timeout           : number?,
    Strict            : boolean?,
    TierOrder         : { string }?,
}

export type ContextStructure = {
    Server : ContextBranch?,
    Client : ContextBranch?,
    Shared : ContextBranch?,
}

export type ContextBranch = {
    Host      : Instance,
    Structure : { string },
}

export type ModuleStatus =
    "pending" | "warmup" | "init" | "starting" |
    "started" | "errored" | "skipped" | "destroyed"

export type ModuleInfo = {
    name    : string,
    status  : ModuleStatus,
    tier    : string,
    context : string,
    timing  : number,
    tags    : { string },
    version : string?,
    alias   : string?,
}

export type MiddlewareContext = {
    module  : string,
    phase   : string,
    tier    : string,
    context : string,
    time    : number,
}

export type PluginHandle = {
    [string] : (...any) -> any,
}

export type Plugin = {
    name     : string,
    version  : string?,
    requires : { string }?,
    install  : (self: Plugin, artext: any, opts: any?) -> PluginHandle?,
}

export type CacheNamespace = {
    set       : (key: string, value: any) -> (),
    get       : (key: string) -> any,
    has       : (key: string) -> boolean,
    delete    : (key: string) -> (),
    watch     : (key: string, fn: (any, any) -> ()) -> () -> (),
    keys      : (pattern: string?) -> { string },
    snapshot  : () -> { [string]: any },
}

export type Logger = {
    trace : (message: string) -> (),
    info  : (message: string) -> (),
    warn  : (message: string) -> (),
    error : (message: string) -> (),
}

export type StateSnapshot = {
    time    : number,
    cache   : { [string]: any },
    modules : { [string]: ModuleStatus },
    ctx     : { [string]: any },
}

export type WorkflowStatus =
    "pending" | "running" | "done" | "cancelled" | "failed"

export type ArTexDiagnostics = {
    bootTime     : number,
    moduleCount  : number,
    errorCount   : number,
    skippedCount : number,
    slowest      : string,
    fastest      : string,
    byTier       : { [string]: { count: number, totalTime: number } },
}
```

All types are exported from `ArTex.Types` and re-exported by the main module.

---

## Contact

| Platform | Handle |
|---|---|
| Roblox | [Kr3ativeKrayon](https://www.roblox.com/users/1911367519/profile) |
| YouTube | [TotallyKr3ative](https://www.youtube.com/channel/UCpNZQoKVclQ74Pk5GmzdQDA) |
| X (Twitter) | [TotallyNotKr3ative](https://x.com/TheRealKr3ative) |
| Email | [TheRealKr3ative@gmail.com](mailto:TheRealKr3ative@gmail.com) |

---

*Last Updated: May 10, 2026*

---