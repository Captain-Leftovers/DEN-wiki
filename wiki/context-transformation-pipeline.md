# Context Transformation Pipeline

**What it is**: The mechanism Den uses to walk your declared infrastructure (hosts → users → homes), activate the right aspects at each stage, and produce a unified configuration for every entity.

**Why it exists**: Config for a host depends on what users it has. Config for a user depends on which host they're on. Config for a home depends on whether it's standalone or host-managed. The pipeline makes these relationships automatic — you declare the data, Den does the traversal.

**Relates to**: [[aspect]], [[nix-class]], [[parametric-dispatch]], [[context-stage]], [[den.ctx]], [[den.hosts]], [[den.homes]], [[resolution]], [[deduplication]], [[collectPairs]], [[dedupIncludes]], [[host.mainModule]], [[host.instantiate]], [[hasAspect]], [[meta.adapter]]

---

## Core idea

Den's pipeline is a data transformation — your infrastructure declarations are input data, and a merged NixOS/Darwin/HM configuration is the output. The pipeline walks the topology automatically:

```
den.hosts.x86_64-linux.laptop   →   {host}   →   {host, user} per user
                                          ↘   {host} hm-host   →   {host, user} hm-user
                                          ↘   {host} wsl-host
den.homes.x86_64-linux.alice    →   {home}
```

At each stage, the appropriate aspects are activated with the current context data. A function declared as `{ host, user }: { ... }` only activates when both `host` and `user` are present in the context — never in a `{host}` or `{home}` context. This is [[parametric-dispatch]] at work.

## The six pipeline steps

### 1. Host context

For each entry in `den.hosts.<system>.<name>`, Den creates a `{host}` context and fires two providers:

- `_.host` — applies the host's own aspect with `parametric.fixedTo { host }` — binds owned configs and static includes to this specific host
- `_.user` — applies the host's aspect with `parametric.atLeast { host, user }` — activates parametric includes that need both host and user, for every user on the host

### 2. User context

`den.ctx.host.into.user` maps each `host.users` entry into a `{host, user}` context:

```nix
den.ctx.host.into.user = { host }:
  map (user: { inherit host user; }) (lib.attrValues host.users);
```

The user context fires `_.user` — applies the user's own aspect with `parametric.fixedTo { host, user }`.

### 3. Derived contexts

Batteries register additional `into.*` transitions on the host context. Each fires conditionally:

| Transition | Condition | Produces |
|---|---|---|
| `into.hm-host` | `host.home-manager.enable && hasHmUsers` | `{host}` hm-host |
| `into.hm-user` | per HM-class user on hm-host | `{host, user}` hm-user |
| `into.wsl-host` | `host.class == "nixos" && host.wsl.enable` | `{host}` wsl-host |
| `into.hjem-host` | `host.hjem.enable && hasHjemUsers` | `{host}` hjem-host |
| `into.hjem-user` | per hjem-class user | `{host, user}` hjem-user |
| `into.maid-host` | `host.nix-maid.enable && hasMaidUsers` | `{host}` maid-host |
| `into.maid-user` | per maid-class user | `{host, user}` maid-user |

Each derived context imports the appropriate OS-level module (e.g. `home-manager.nixosModules.home-manager`) and forwards the relevant class config to the right target (e.g. `home-manager.users.<userName>`).

### 4. collectPairs

Internally, `collectPairs` walks all `into.*` transitions recursively from the starting context, building a flat list of `{ ctx, ctxDef, source }` pairs — one entry per context stage reached.

### 5. dedupIncludes

`dedupIncludes` processes the pairs from `collectPairs` to prevent the same owned config from being applied multiple times when an aspect appears at multiple pipeline stages:

- **First occurrence** of a context type → `parametric.fixedTo` — includes owned configs + statics + parametric matches
- **Subsequent occurrences** → `parametric.atLeast` — includes only parametric matches (owned/statics already applied)

This is why `den.default` settings like `stateVersion` are not duplicated even though `den.default` is processed at every stage.

### 6. Output

`modules/config.nix` collects the results and calls `host.instantiate` for each host and home:

```nix
# For hosts
host.instantiate {
  modules = [ host.mainModule { nixpkgs.hostPlatform = host.system; } ];
}

# For homes
home.instantiate {
  pkgs = home.pkgs;
  modules = [ home.mainModule ];
}
```

`host.mainModule` is the `[[deferredModule]]` produced by resolving the host's full aspect DAG through its context pipeline. The result is placed at `flake.<intoAttr>` — by default `nixosConfigurations.<name>`, `darwinConfigurations.<name>`, or `homeConfigurations.<name>`.

## Standalone homes — a separate path

`den.homes` entries go through a shorter pipeline:

```
den.homes.x86_64-linux.alice → den.ctx.home {home} → fixedTo {home} aspects.alice → homeConfigurations.alice
```

There is no host context. Functions requiring `{ host }` are never activated. Functions requiring `{ home }` run instead. This makes standalone homes safe to build independently of any host.

## Custom contexts

The pipeline is open for extension. Adding a new derived context is three lines:

```nix
den.ctx.gpu = {
  description = "GPU-enabled host";
  _.gpu = { host }: { nixos.hardware.nvidia.enable = true; };
};
den.ctx.host.into.gpu = { host }:
  lib.optional (host ? gpu) { inherit host; };
```

This creates a new pipeline stage that only activates for hosts with a `gpu` attribute. Any battery or user config can contribute to it via the `gpu` class on an aspect.

## Gotchas

- The pipeline runs at **evaluation time**, not lazily — all declared hosts and homes are traversed when the flake is evaluated
- `den.default` parametric functions run at every context stage — use `den.lib.take.exactly` when a function should only run at one specific stage
- `host.class` (`"nixos"`, `"darwin"`) is the OS class; `user.classes` is the list of home environment classes — they control different parts of the pipeline
- Derived contexts (hm-host, wsl-host, etc.) are registered by batteries — if a battery is not included, its context stage never activates even if the host flag is set
- The `into.*` functions return a **list** — returning an empty list `[]` means the transition does not fire; returning `[ { inherit host; } ]` fires once

## In the Nix ecosystem

Standard NixOS configurations have no concept of a pipeline. Host config, user config, and home config are wired manually — you import home-manager modules, thread `specialArgs`, and manage the relationships yourself. Den's pipeline makes the topology declarative: you declare what exists, Den handles the traversal and wiring.

The pipeline is also the reason `host` and `user` context args are safe for conditional imports — they are resolved during the pipeline traversal, before `lib.evalModules` runs, so there is no circular dependency on `config`.

## Related pages

- [[aspect]] — aspects are what get activated at each pipeline stage
- [[parametric-dispatch]] — the mechanism that routes context data to matching functions
- [[nix-class]] — user.classes controls which derived context stages activate
- [[den.ctx]] — the context type definitions that declare the pipeline structure
- [[context-stage]] — a named point in the pipeline with a defined data shape
- [[collectPairs]] — internal function that walks all into.* transitions recursively
- [[dedupIncludes]] — internal function that prevents duplicate owned config application
- [[deduplication]] — the problem dedupIncludes solves
- [[host.mainModule]] — the deferredModule produced by running a host through the pipeline
- [[host.instantiate]] — the builder function called with mainModule to produce the final config
- [[den.hosts]] — the source data that seeds the pipeline
- [[den.homes]] — standalone homes that go through a shorter pipeline path
- [[den.default]] — the global aspect processed at every pipeline stage