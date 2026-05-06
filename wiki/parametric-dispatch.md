# Parametric Dispatch

**What it is**: The mechanism by which Den automatically matches aspect functions to contexts — a function only activates when the context contains all the arguments the function declares.

**Why it exists**: Without parametric dispatch, every aspect function would need explicit `mkIf`, `enable` flags, or `specialArgs` wiring to conditionally apply. Parametric dispatch eliminates all of that — the function's argument signature IS the condition.

**Relates to**: [[aspect]], [[__functor]], [[canTake]], [[parametric-constructor]], [[context-transformation-pipeline]], [[builtins.functionArgs]], [[includes]], [[den.lib]]

---

## Core idea

Den inspects every function in an aspect's `includes` list using `builtins.functionArgs` at evaluation time. If the function requires `{ host, user }` and the current context only has `{ host }`, the function is silently skipped. No error, no fallback — just not called.

```nix
# This runs in any context that has at least a `host` key
({ host, ... }: { nixos.networking.hostName = host.hostName; })

# This only runs when BOTH host and user are present
({ host, user }: { nixos.users.users.${user.userName}.extraGroups = [ "wheel" ]; })

# This only runs in standalone home contexts
({ home }: { homeManager.home.username = home.userName; })

# This runs in every context — it's a plain attrset, not a function
{ nixos.networking.firewall.enable = true; }
```

The context shape IS the condition. No `mkIf`. No `enable` options. No `specialArgs` threading.

## How it works internally

1. Den calls `builtins.functionArgs fn` on each function in `includes`
2. This returns an attrset of `{ argName = isOptional }` pairs
3. `canTake` checks whether the current context satisfies the function's required args
4. If yes → function is called with the context and its result is included
5. If no → function is silently skipped

## canTake

`den.lib.canTake` is the predicate at the core of dispatch:

```nix
# atLeast (default) — all required params must be present; extras are fine
canTake { host = ...; }              ({ host, ... }: ...) # => true
canTake { host = ...; }              ({ host, user }: ...) # => false (user missing)
canTake { host = ...; user = ...; }  ({ host, ... }: ...) # => true (extra keys fine)

# exactly — required params must match context keys exactly
canTake.exactly { host = ...; }              ({ host }: ...) # => true
canTake.exactly { host = ...; foo = ...; }   ({ host, ... }: ...) # => false (extras)
```

Two modes:
- **`canTake.atLeast`** (default) — the function's required params are all present; extra context keys are allowed
- **`canTake.exactly`** — required params match the context keys exactly; neither side has extras

## take

`den.lib.take` wraps `canTake` into helpers for use inside `includes`:

```nix
den.lib.take.atLeast fn ctx   # apply fn only if ctx satisfies atLeast
den.lib.take.exactly fn ctx   # apply fn only if ctx satisfies exactly
```

## perUser and perHome

Convenience wrappers that enforce exact matching for common context shapes:

```nix
den.lib.perUser  ({ host, user }: ...) # only matches { host, user } exactly
den.lib.perHome  ({ home }:      ...) # only matches { home } exactly
```

Equivalent to wrapping with `canTake.exactly` for those specific shapes.

## The parametric constructor

`den.lib.parametric` installs a `[[__functor]]` on an aspect that processes its `includes` list through parametric dispatch when the aspect is called with a context:

```nix
den.lib.parametric {
  nixos.foo = 1;                               # owned config — always included
  includes = [
    { nixos.bar = 2; }                         # static — always included
    ({ host }: { nixos.x = host.hostName; })   # parametric — host contexts only
    ({ user }: { homeManager.y = 1; })         # parametric — user contexts only
  ];
}
```

### Parametric variants

| Constructor | Behaviour |
|---|---|
| `parametric` | Default — owned configs + static includes + atLeast-matched functions |
| `parametric.atLeast` | Skips owned configs and statics — only atLeast-matched functions |
| `parametric.exactly` | Like atLeast but uses `canTake.exactly` |
| `parametric.fixedTo attrs` | Ignores received context — always dispatches with the given fixed attrs |
| `parametric.expands attrs` | Extends the received context with attrs before dispatch |

`parametric.fixedTo` is used internally by [[den.ctx]] when binding a host's own aspect — it fixes the context to `{ host }` so the aspect always receives the right data regardless of what context called it.

`parametric.atLeast` is used for subsequent pipeline occurrences of an aspect to avoid re-applying owned configs (see [[deduplication]]).

## Context-aware battery pattern

The canonical pattern for batteries that need to handle both host+user and standalone home contexts:

```nix
den.lib.parametric.exactly {
  includes = [
    ({ host, user }: { nixos.users.users.${user.userName}.isNormalUser = true; })
    ({ home }:       { homeManager.home.username = home.userName; })
  ];
}
```

The first function runs for every `{ host, user }` pair. The second runs only for standalone `{ home }` contexts. No `mkIf`, no conditionals — the argument pattern is the selector.

## Gotchas

- **`...` matters** — `({ host, ... }: ...)` uses atLeast matching (extra keys allowed); `({ host }: ...)` uses exactly matching (no extra keys). Forgetting `...` is a common source of a function not activating when expected.
- **Anonymous functions are an anti-pattern** — inline lambdas in `includes` produce cryptic Nix error traces. Use named aspects instead: `den.aspects.foo = { host }: { ... };` then reference `den.aspects.foo` in includes.
- **Owned configs bypass dispatch** — if you want class-specific config to be conditional, put it in a parametric function in `includes`, not as a top-level class key on the aspect.
- **`den.default` parametric includes run at every pipeline stage** — use `den.lib.take.exactly` to restrict a function to one specific context shape when it would otherwise match too broadly.

## In the Nix ecosystem

Standard NixOS modules have no equivalent mechanism. Conditional module application requires `lib.mkIf`, `specialArgs`, or `_module.args` — all of which couple the module to the evaluation context. Den's parametric dispatch decouples them entirely.

### `mkIf` — conditional module application

**Traditional**: A module is always imported; `lib.mkIf` makes it produce empty config when the condition is false:

```nix
{ config, lib, ... }:
{
  config = lib.mkIf (config.networking.hostName == "laptop") {
    hardware.bluetooth.enable = true;
  };
}
```

The module runs every evaluation. The condition is late-bound (needs `config`), so it can't be used for conditional `imports` without triggering infinite recursion.

**Den**: The function argument signature IS the guard. A function requiring `{ user, ... }` never runs in a `{ host }` context:

```nix
# Only runs during user context — no mkIf, no condition
({ user, ... }: {
  homeManager.programs.git.enable = true;
})

# Only runs for laptops — host context provides host.hostName
({ host, ... }: lib.optionalAttrs (host.hostName == "laptop") {
  nixos.hardware.bluetooth.enable = true;
})
```

Context data is pre-module-system, so it is safe for conditional imports.

### `lib.mkMerge` — combining conditional fragments

**Traditional**: Multiple conditional pieces are merged manually:

```nix
{
  config = lib.mkMerge [
    (lib.mkIf isLaptop  { hardware.bluetooth.enable = true; })
    (lib.mkIf hasNvidia { hardware.nvidia.enable = true; })
    (lib.mkIf isDev     { services.postgresql.enable = true; })
  ];
}
```

**Den**: Each concern is a separate aspect in `includes`. The aspect resolution DAG merges them naturally:

```nix
den.aspects.laptop.includes = [
  den.aspects.bluetooth
  den.aspects.nvidia
];
```

No `mkMerge`, no nested attrset merging. Aspects depend on aspects; Den resolves and merges.

### `specialArgs` / `_module.args` — threading data into modules

**Traditional**: Data the module needs must be threaded through `specialArgs` or `extraSpecialArgs` at the `nixosSystem` / `home-manager` call site:

```nix
lib.nixosSystem {
  specialArgs = { inherit inputs hostname; };
  modules = [ ./configuration.nix ];
}

# Then in configuration.nix
{ inputs, hostname, pkgs, ... }: {
  environment.systemPackages = [ inputs.some-flake.packages.${pkgs.system}.tool ];
}
```

This is fragile: add a new variable, touch three call sites (host, HM, maybe Darwin).

**Den**: Context data (`host`, `user`, `home`) and batteries (`inputs'`, `self'`) arrive as function arguments automatically because the pipeline knows what context it is in:

```nix
# host context gives you `host` automatically
({ host, ... }: {
  nixos.networking.hostName = host.hostName;
})

# inputs' battery gives you system-qualified inputs in any context
den.default.includes = [ den._.inputs' ];
# Now any aspect can use:
({ inputs', ... }: {
  nixos.environment.systemPackages = [ inputs'.some-flake.packages.tool ];
})
```

No `specialArgs` wiring, no `extraSpecialArgs`, no `_module.args` declarations.

### Summary table

| Traditional pattern | Den replacement |
|---|---|
| `lib.mkIf condition { ... }` | Parametric function with context args: `({ host, user }: ...)` |
| `lib.mkMerge [ a b c ]` | Aspect `includes` list merging through resolution |
| `specialArgs = { inherit inputs; }` | Context-aware function args + batteries like `inputs'` |
| `extraSpecialArgs` for HM | Same — `inputs'` works in user/home contexts automatically |
| Manual `home-manager.users.alice = ...` wiring | HM battery forwards `homeManager` class automatically |

`callPackage` in nixpkgs is the closest analogy — it also inspects function arguments and injects matching values. The key difference: `callPackage` is one-shot and nixpkgs-specific. Den's dispatch runs per-context across an entire infrastructure pipeline and works for any Nix module domain.

## Related pages

- [[aspect]] — parametric dispatch is what makes aspects callable with a context
- [[__functor]] — the Nix mechanism that makes an attrset callable; installed by `den.lib.parametric`
- [[canTake]] — the predicate that checks argument compatibility
- [[parametric-constructor]] — `den.lib.parametric` and its variants
- [[includes]] — the list that parametric dispatch filters through
- [[context-transformation-pipeline]] — the pipeline that calls aspects with context data
- [[den.lib]] — the library where canTake, take, parametric live
- [[builtins.functionArgs]] — the native Nix builtin that makes argument introspection possible
- [[deduplication]] — how fixedTo and atLeast variants prevent duplicate owned configs