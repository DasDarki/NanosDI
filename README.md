# NanosDI

Dependency injection for [nanos world](https://nanos.world) packages. Written in [Nebra](https://nebra-lang.github.io), usable from Nebra with
annotations and from plain Lua with a small API.

```nebra
import { Service } from "nanos-di"
import type { AccountService } from "./AccountService"

@Service
export class PlayerService extends Service
  @Inject
  accounts: AccountService

  override function OnStart(): void
    print(self.accounts:count())
  end

  override function OnStop(): void
    print("stopping")
  end
end
```

```lua
local PlayerService = NanosDI.service("PlayerService", { accounts = "AccountService" })

function PlayerService:OnStart()
    print(self.accounts:count())
end
```

## Installation

1. Build with `nebra build` and copy `out` to `Packages/nanos-di` of your server.
2. Add `"nanos-di"` to `packages_requirements` in your `Package.toml`.
3. For a Nebra package, also add the dependency for the types and annotations.

## How it works

Services only register while the packages load. When all packages are loaded, the container runs these steps once:

1. Checks every injection and reports all problems at once. If one fails, nothing is created.
2. Creates every service.
3. Assigns every `@Inject` field.
4. Calls `OnStart`, dependencies first.

Because creating and injecting are separate steps, services can depend on each other in a cycle. On unload, `OnStop` runs in reverse order.
A service whose `OnStart` fails is logged, gets no `OnStop`, and does not stop the others.

## Rules

**Use `import type` for injected types.** Nebra rejects regular imports that form a cycle, and the container only needs the type's name.

**Import every service file once**, for example in your `Index.neb`. A file that is only reached through `import type` never runs, so its
service would never register.

```nebra
import "./services/AccountService"
import "./services/PlayerService"
```

**Do not use injected fields in the constructor.** They are assigned after all services exist. Use `OnStart` instead.

**A service's constructor takes no parameters**, because the container creates it.

**Services are found by name.** The name is the class name, so it has to be unique across all packages. Set another one with
`@Service(name = "...")`. If you renamed an import with `as`, use `@Inject(name = "...")`.

## Injection

| Field | Result |
|---|---|
| `@Inject x: Foo` | The service `Foo`. Startup fails without one. |
| `@Inject x: Foo?` | The service `Foo`, or `nil` without one. |
| `@Inject x: SomeInterface` | The only service that implements the interface. Fails when several do. |
| `@Inject(name = "Foo") x: any` | The service or value registered as `Foo`. |

`@Inject` needs `@Service` on its class. An abstract `@Service` class passes its injected fields on to the services extending it and is not
created itself. Extending `Service` is optional, it only declares `OnStart` and `OnStop` for you.

In Lua, a name ending in `?` is optional: `NanosDI.service("Foo", { cache = "Cache?" })`.

## API

| Function | Description |
|---|---|
| `NanosDI.service(name, injections)` | Lua: creates and registers a service class. |
| `NanosDI.register(name, class, injections, isAbstract)` | Registers an existing class with a `new` function. |
| `NanosDI.provide(name, value)` | Makes an existing value injectable, such as an `EntityManager`. |
| `NanosDI.get(classOrName)` | Returns a service, for code that is not a service. Only after services are resolved. |
| `NanosDI.onReady(callback)` | Runs the callback once services are started, or right away if they already are. |
| `NanosDI.isReady()` | Returns whether the services are resolved. |

Top-level code of a file runs while packages load, before anything is resolved. Put code that needs services into `OnStart` or `onReady`.

## Reloading

Registering after startup, for example through `package reload`, resolves again on the next tick. A reloaded service is stopped and replaced,
and every service injecting it receives the new instance. Other running services are not restarted. Reloading `nanos-di` itself needs
`package reload all`, since it holds the registrations.
