# den.ctx

**What it is**: The attrset of context type definitions that declare the structure of Den's evaluation pipeline — each entry defines a named stage, what it contributes, and how it transitions to other stages.

**Why it exists**: The pipeline needs a declarative description of its topology. `den.ctx` is that description — it defines what data exists at each stage, which aspects get activated, and which transitions are possible. Everything in the pipeline flows from these definitions.

**Relates to**: [[context-transformation-pipeline]], [[context-stage]], [[aspect]], [[parametric-dispatch]], [[den.ctx.into]], [[den.ctx.provides]], [[batteries]], [[custom-context]], [[meta.adapter]], [[hasAspect]]

---

## Core idea

`den.ctx` is a `lazyAttrsOf ctxType` — a map of names to context type definitions. Each definition has four fields:

```nix
den.ctx.host = {
  description = "OS host context";          # human-readable label

  _.host = { host }:                        # provider: activates an aspect with this context
    parametric.fixedTo { inherit host; }
    den.aspects.${host.aspect};

  into.user = { host }:                     # transition: produces new context values
    map (user: { inherit host user; })
    (lib.attrValues host.users);

  includes = [ ... ];                       # aspect includes injected at this stage
  modules  = [ ... ];                       # raw modules merged into resolved output
};
```

The `_` field (alias: `provides`) maps names to provider functions. The `into` field maps names to transition functions. Together they define the full pipeline topology.

## Built-in context types

### `den.ctx.host` — `{ host }`

Entry point. Produced for each `den.hosts.<system>.<name>`.

| Provider/Transition | What it does |
|---|---|
| `_.host` | `parametric.fixedTo { host }` on host's aspect — binds owned configs |
| `_.user` | `parametric.atLeast { host, user }` on host's aspect — activates cross-user parametric includes |
| `into.user` | One `{ host, user }` per `host.users` entry |
| `into.hm-host` | `{ host }` hm-host — if HM enabled and has HM users |
| `into.wsl-host` | `{ host }` wsl-host — if `host.wsl.enable` on NixOS |
| `into.hjem-host` | `{ host }` hjem-host — if hjem enabled |
| `into.maid-host` | `{ host }` maid-host — if nix-maid enabled |

### `den.ctx.user` — `{ host, user }`

Produced per user via `into.user` from host context.

| Provider/Transition | What it does |
|---|---|
| `_.user` | `parametric.fixedTo { host, user }` on user's aspect |
| `into.default` | Identity transition — passes through |

### `den.ctx.home` — `{ home }`

Produced for each `den.homes.<system>.<name>`. Separate root — not derived from host context.

| Provider/Transition | What it does |
|---|---|
| `_.home` | `parametric.fixedTo { home }` on home's aspect |

### `den.ctx.hm-host` — `{ host }`

Registered by the Home Manager battery. Same data shape as host context but different stage name.

| Provider/Transition | What it does |
|---|---|
| `provides.hm-host` | Imports `home-manager.nixosModules.home-manager` (or darwin equivalent) |
| `into.hm-user` | One `{ host, user }` per HM-class user |

### `den.ctx.hm-user` — `{ host, user }`

Registered by the Home Manager battery.

| Provider/Transition | What it does |
|---|---|
| `_.hm-user` | Forwards `homeManager` class content to `home-manager.users.<userName>` |

### `den.ctx.wsl-host` — `{ host }`

Registered by the WSL battery.

| Provider/Transition | What it does |
|---|---|
| `provides.wsl-host` | Imports NixOS-WSL module; creates `wsl` class forward |

## The `_` vs `provides` distinction

`_` and `provides` are aliases for the same field. Both map context names to provider functions. The provider function receives the current context data and returns aspect fragments contributed at this stage.

```nix
# These are identical:
den.ctx.host._.host    = { host }: ...;
den.ctx.host.provides.host = { host }: ...;
```

The `_` shorthand is preferred in practice for brevity.

## `into` transitions

`into` functions take the current context data and return a **list** of new context values for the next stage:

```nix
# Returns one {host, user} per user — or [] if no users
den.ctx.host.into.user = { host }:
  map (user: { inherit host user; }) (lib.attrValues host.users);

# Returns a singleton list or [] based on condition
den.ctx.host.into.wsl-host = { host }:
  lib.optional host.wsl.enable { inherit host; };
```

Returning `[]` means the transition does not fire. This is how conditional derived contexts work — the condition is in the `into` function, not in a flag.

## `includes` on a context type

Context types can have an `includes` field — aspect includes injected at that pipeline stage globally, without modifying any individual aspect. Batteries use this to hook into the pipeline:

```nix
den.ctx.user.includes = [ den._.mutual-provider ];
```

This adds `mutual-provider` to every user context stage without requiring every user aspect to declare it. See [[mutual-provider]] for the full guide on bidirectional host↔user configuration.

## `meta.adapter`

A context type can declare a `meta.adapter` — a function that filters or transforms which aspects are resolved within that context, applied transitively:

```nix
den.ctx.host.meta.adapter = inherited:
  den.lib.aspects.adapters.filter (a: a.name != "debug-tools") inherited;
```

Context adapters are **outermost** — they apply before any aspect-level adapters. This means a context-level filter cannot be overridden by an aspect inside that context.

## Defining custom context types

Any new pipeline stage is three declarations:

```nix
# 1. Define the context type
den.ctx.gpu = {
  description = "GPU-enabled host";
  _.gpu = { host }: {
    nixos.hardware.nvidia.enable = true;
  };
};

# 2. Wire it into an existing context via into
den.ctx.host.into.gpu = { host }:
  lib.optional (host ? gpu) { inherit host; };

# 3. Optionally contribute aspect config at this stage
den.aspects.my-gpu-setup = { host }: {
  nixos.hardware.nvidia.package = host.gpu.package;
};
```

This is how all batteries (HM, WSL, hjem, maid) register their context stages — they are not special-cased in Den's core.

## Gotchas

- `into` functions must return a **list**, even for a single value — `[ { inherit host; } ]` not `{ inherit host; }`. Returning an attrset instead of a list is a silent bug.
- Context type names are arbitrary strings — but they must match exactly between the `into.<name>` transition and the `den.ctx.<name>` definition.
- `_.user` on `den.ctx.host` is not the same as `_.user` on `den.ctx.user` — the former activates the host aspect with both host+user args; the latter activates the user's own aspect fixed to that user.
- Batteries register their `into.*` transitions as side effects of being imported — if a battery's module is not imported, its derived context never exists.

## In the Nix ecosystem

`den.ctx` has no direct equivalent elsewhere in the Nix ecosystem. The closest concept is flake-parts' `perSystem` — a named evaluation scope with data injected at that scope. Den generalizes this idea into a full topology (not just per-system) and makes the transitions user-extensible.

The key difference from standard NixOS module evaluation: `lib.evalModules` is a one-shot merge. `den.ctx` is a traversal — it walks relationships between entities and accumulates config along the way.

## Related pages

- [[context-transformation-pipeline]] — the full pipeline that den.ctx implements
- [[context-stage]] — a named point in the pipeline with a defined data shape
- [[aspect]] — what gets activated at each context stage
- [[parametric-dispatch]] — how aspect functions are matched to context data
- [[batteries]] — how batteries register their own context types via into transitions
- [[custom-context]] — the pattern for adding new pipeline stages
- [[meta.adapter]] — filtering which aspects resolve within a context
- [[den.ctx.into]] — the transition functions that drive the pipeline forward
- [[den.ctx.provides]] — the provider functions that activate aspects at each stage