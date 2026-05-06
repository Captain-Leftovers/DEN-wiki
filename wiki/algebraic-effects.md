# Algebraic Effects

**What it is**: A functional programming pattern where computations declare their external dependencies as "effect requests" that handlers fulfill — the theoretical foundation for Den's future dependency injection system (`den.lib.fx`).

**Why it exists**: Den's current parametric dispatch is a form of dependency injection, but it is hand-rolled and rigid. Algebraic effects formalize and generalize this — making dependency injection transparent, testable, and composable across arbitrary Nix configurations.

**Relates to**: [[parametric-dispatch]], [[den-lib]], [[aspect]], [[context-transformation-pipeline]], [[builtins.functionArgs]]

---

## Core idea

An effectful computation is a function that declares what it needs from the outside world before it can produce a result. You already use this pattern in Nix:

```nix
myPackage = { stdenv, jq }: stdenv.mkDerivation { ... };
```

`myPackage` is a pure function that cannot do its job until `stdenv` and `jq` are provided. `callPackage` acts as the handler — it finds the right values and injects them.

Den aspects work the same way:

```nix
{ host, user }: {
  nixos.users.users.${user.userName}.shell = pkgs.fish;
}
```

This aspect declares `host` and `user` as dependencies. The [[context-transformation-pipeline]] acts as the handler, providing the correct values for each machine and user.

Algebraic effects are a formal model for this pattern — making the dependency/handler relationship explicit, composable, and swappable.

## den.lib.fx API

`den.lib.fx` is Den's experimental effects API, available on a feature branch (not yet on `main`). Access it via:

```bash
nix repl "github:vic/den?dir=templates/ci"
# then: :a den.lib
```

### `fx.pure value`

Wraps a plain value as an effect — no external dependencies needed:

```nix
fx.handle { handlers = {}; } (fx.pure 42)
# => { state = null; value = 42; }
```

### `fx.send "name" payload`

Declares a dependency on an external value — an effect request:

```nix
fx.send "hostName" null   # "I need a hostName to continue"
```

Requests are like ordering food. You say what you want; the handler produces it.

### `fx.handle { handlers; state; } computation`

Runs a computation with a set of handlers:

```nix
fx.handle {
  handlers.hostName = { param, state }: {
    resume = "igloo";
    inherit state;
  };
  state = {};
} (fx.send "hostName" null)
# => { value = "igloo"; state = {}; }
```

A handler receives `{ param, state }` and returns `{ resume, state }`. `param` is the payload sent with the request. `resume` is the answer.

### `fx.bind comp fn`

Chains effectful computations — the result of one feeds into the next:

```nix
fx.bind (fx.send "pkgs" null)
  (pkgs: fx.bind (fx.send "hostName" null)
    (hostName: fx.pure (pkgs.writeText "motd" "Welcome to ${hostName}")))
```

Analogous to Promises or Futures — sequential async operations, but in pure Nix.

### `fx.bind.fn attrs fn`

Converts any plain Nix function into an effectful computation automatically. It:

1. Calls `builtins.functionArgs fn` to get argument names
2. Generates one `fx.send "<argName>"` per argument
3. Binds all handler responses into an attrset
4. Calls the original function with that attrset
5. Wraps the result in `fx.pure`

```nix
myMotd = { pkgs, hostName }:
  pkgs.writeText "motd" "Welcome to ${hostName}!";

fx.handle {
  handlers = {
    pkgs     = { param, state }: { resume = import <nixpkgs> {}; inherit state; };
    hostName = { param, state }: { resume = "igloo"; inherit state; };
  };
  state = {};
} (fx.bind.fn {} myMotd)
# => { value = «derivation .../motd»; state = {}; }
```

The function `myMotd` is completely pure — it never mentions effects. `fx.bind.fn` turns it into an effectful computation by inspecting its arguments.

## The connection to parametric dispatch

Den's current [[parametric-dispatch]] is already a form of algebraic effects — it just uses `builtins.functionArgs` directly rather than a formal effects system:

| Current (parametric dispatch) | Future (algebraic effects) |
|---|---|
| `builtins.functionArgs fn` introspects args | `fx.bind.fn` auto-generates requests from args |
| `canTake ctx fn` checks if context satisfies args | Handler system decides what to provide |
| Pipeline calls matching functions with context | Handlers answer effect requests |
| Manual deduplication in `dedupIncludes` | Centralized handler tracks what has been provided |

The effects model generalizes this: instead of a fixed context attrset driving dispatch, any handler can respond to any request — enabling advanced features like aspect-to-aspect communication, dynamic feature detection, and testable configuration generation.

## Mock testing — a key benefit

The handler/computation separation makes the same aspect function testable with mock data:

```nix
# Production: real nixpkgs and real host
fx.handle {
  handlers.pkgs     = { ... }: { resume = import <nixpkgs> {}; inherit state; };
  handlers.hostName = { ... }: { resume = "igloo"; inherit state; };
  state = {};
} (fx.bind.fn {} myConfig)

# Test: mock everything — no Nix store, no downloads
fx.handle {
  handlers.pkgs     = { ... }: { resume = { writeText = n: t: "MOCK:${t}"; }; inherit state; };
  handlers.hostName = { ... }: { resume = "test-host"; inherit state; };
  state = {};
} (fx.bind.fn {} myConfig)
```

Same computation. Different behavior. This bypasses the Nix store entirely for unit testing.

## Why Den is migrating to this model

Den's current effects system (as of v0.x) is described as "half-baked, rigid-context, manually-threaded." The migration to `nix-effects` (`vic/nix-effects`, based on `vic/nfx`) aims to:

- **Better context control** — handlers can respond dynamically rather than requiring a fixed context shape
- **Better dedup detection** — a centralized handler tracks what has been provided, enabling smarter deduplication than the current `dedupIncludes` approach
- **Aspect communication** — aspects can communicate with each other via higher-level message handlers (feature detection, aspect replacement, advanced coordination)
- **Formal foundation** — replaces home-grown hacks with a professional, tested, ability-based system

The user-facing aspect syntax is **not changing**. The effects system is an implementation detail under the hood.

## nix-effects lineage

Den's effects implementation traces through:

```
vic/fx.go → vic/fx-rs → vic/nfx → vic/nix-effects → den.lib.fx
```

The design is rooted in research on "freer monads" and "ability-based programming" in functional languages. The library is small — its repository appears larger because of extensive inline testing and documentation.

## Gotchas

- `den.lib.fx` is **experimental** and only on a feature branch — do not use in production configs today
- The effects migration does not change the user-facing API — aspects still look like `{ host, user }: { ... }` 
- `callPackage` is the closest existing Nix analogy but is one-shot and nixpkgs-specific; `den.lib.fx` handles arbitrary multi-stage communication
- The mathematical terminology in nix-effects docs ("freer monads", "abilities") can be ignored for practical use — the pattern is just: declare what you need, provide a handler that supplies it

## In the Nix ecosystem

Algebraic effects are unusual in the Nix world. Most Nix evaluation is pure and lazy — side effects and dependency injection happen through the module system's `_module.args` and `specialArgs`, which are baked into `lib.evalModules`. 

Den's effects work is exploring a more principled alternative to this approach — one where dependency injection is explicit, inspectable, and not coupled to the module system's evaluation order. This is genuinely novel territory for Nix configuration tooling.

## Related pages

- [[parametric-dispatch]] — the current implementation that effects will eventually replace/formalize
- [[den-lib]] — den.lib.fx is part of den.lib (experimental branch)
- [[aspect]] — aspects are effectful computations — they declare host/user as dependencies
- [[context-transformation-pipeline]] — the pipeline is the current "handler" for host/user requests
- [[builtins.functionArgs]] — the Nix builtin that both parametric dispatch and fx.bind.fn use for argument introspection