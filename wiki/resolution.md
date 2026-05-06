# Resolution

**What it is**: The process of walking an aspect's full dependency DAG, applying parametric dispatch at each stage, and producing a single merged `deferredModule` for a specific Nix class.

**Why it exists**: An aspect is a composable data structure — it references other aspects via `includes`, each of which may reference more. Resolution is what collapses this graph into something a module system like `lib.nixosSystem` can actually consume.

**Relates to**: [[aspect]], [[aspect-dag]], [[parametric-dispatch]], [[nix-class]], [[deferredModule]], [[den-lib]], [[context-transformation-pipeline]], [[deduplication]], [[collectPairs]], [[dedupIncludes]], [[hasAspect]], [[meta.adapter]]

---

## Core idea

Given an aspect and a target class, resolution answers: "what is the complete, merged, deduplicated module for `nixos` (or `darwin`, or `homeManager`) produced by this aspect and everything it depends on?"

```nix
module = den.lib.aspects.resolve "nixos" [] den.aspects.laptop;
```

The result is a `deferredModule` — a standard Nix module ready to pass directly to `lib.nixosSystem`, `darwin.lib.darwinSystem`, or any other module system instantiator.

## The six resolution steps

### 1. Collect all aspects

Den starts from the target aspect and recursively walks every `includes` entry, collecting the full transitive closure of the dependency graph ([[aspect-dag]]).

```nix
# Given:
den.aspects.workstation = {
  includes = [ den.aspects.dev-tools den.aspects.gaming ];
  nixos.services.xserver.enable = true;
};
den.aspects.dev-tools = {
  nixos.environment.systemPackages = [ pkgs.git ];
};
```

Resolution collects `workstation`, `dev-tools`, and `gaming` — the full DAG, depth-first.

### 2. Extract class-specific config

For each collected aspect, Den extracts only the attributes keyed by the target class (`nixos`, `darwin`, `homeManager`, etc.). Class keys that don't match the target are ignored entirely.

An aspect targeting `nixos` on a Darwin host has its `nixos` key skipped — only `darwin` is extracted.

### 3. Evaluate static includes

Functions in `includes` receiving `{ class, aspect-chain }` are **static aspects** — they are evaluated once at this point, not per-context. They receive:
- `class` — the class currently being resolved (`"nixos"`, `"darwin"`, etc.)
- `aspect-chain` — the list of aspects traversed to reach this point (most recent last)

Static aspects can use `class` to produce class-aware config without parametric dispatch:

```nix
# Static aspect — runs once during resolution
{ class, aspect-chain }: {
  ${class}.environment.etc."den-class".text = class;
}
```

### 4. Build context pairs from den.ctx

Den calls `collectPairs` — walks all `into.*` transitions from the entry context (`den.ctx.host { host }`, or `den.ctx.home { home }`) recursively, building a flat list of `{ ctx, ctxDef, source }` pairs representing every pipeline stage that will fire.

This is where the host → users → homes topology is resolved into a concrete list of activation events.

### 5. Apply parametric includes via ctxApply

For each context pair from step 4, Den calls the aspect's `__functor` with that context. `canTake` filters the `includes` list — each function that matches the context shape is called and its result collected.

This is [[parametric-dispatch]] at the resolution stage — every `{ host, user }` function fires once per user, every `{ home }` function fires once per standalone home, etc.

`dedupIncludes` prevents duplicate application:
- **First occurrence** of a context type → `parametric.fixedTo` — owned configs + statics + parametric matches
- **Subsequent occurrences** → `parametric.atLeast` — only parametric matches (owned/statics already applied)

### 6. Merge into deferredModule

All collected config fragments are merged in order into a single `deferredModule`. This is the final output — a standard Nix module that can be passed to any module system instantiator.

```nix
nixosConfigurations.laptop = lib.nixosSystem {
  modules = [ (den.lib.aspects.resolve "nixos" [] den.aspects.laptop) ];
};
```

## den.lib.aspects.resolve signature

```nix
den.lib.aspects.resolve class aspect-chain aspect
```

| Argument | Type | Description |
|---|---|---|
| `class` | str | The Nix class to resolve for — `"nixos"`, `"darwin"`, `"homeManager"`, etc. |
| `aspect-chain` | list | Accumulator for the chain of aspects traversed — pass `[]` at the call site |
| `aspect` | aspect | The aspect to resolve |

The `aspect-chain` argument is used internally to pass context to static aspects (the `{ class, aspect-chain }` functions). Always pass `[]` when calling from outside.

## deferredModule

The output of resolution is a `deferredModule` — a Nix value that behaves as a standard module when passed to `lib.evalModules`, `lib.nixosSystem`, `darwin.lib.darwinSystem`, or `home-manager.lib.homeManagerConfiguration`.

It is "deferred" because it has not yet been evaluated against a specific `pkgs` or `config` — it is a module definition, not a realized configuration. The module system evaluates it lazily when building the final system.

```nix
# All of these work:
lib.nixosSystem        { modules = [ deferredModule ]; }
darwin.lib.darwinSystem { modules = [ deferredModule ]; }
home-manager.lib.homeManagerConfiguration { modules = [ deferredModule ]; }
```

## entity.resolved

Every declared entity (host, user, home) has a `resolved` field — the aspect produced by running it through its context pipeline. Den uses this internally to compute `host.mainModule`:

```nix
host.mainModule = den.lib.aspects.resolve host.class [] host.resolved;
```

To filter which aspects participate in resolution for a specific context, set `meta.adapter` on the context node:

```nix
den.ctx.host.meta.adapter = inherited:
  den.lib.aspects.adapters.filter (a: a.name != "debug-tools") inherited;
```

Adapters compose — context adapters are outermost, aspect-level adapters are innermost.

## Resolution vs the pipeline

These are two related but distinct processes:

| | Resolution | Pipeline |
|---|---|---|
| What it does | Collapses an aspect DAG into a module for one class | Walks host→users→homes topology, activating aspects |
| Input | An aspect + a class name | A host/home declaration |
| Output | A `deferredModule` | A built `nixosConfiguration` / `darwinConfiguration` |
| When it runs | During `collectPairs` / `ctxApply` | Driven by `modules/config.nix` at evaluation time |
| Key function | `den.lib.aspects.resolve` | `den.ctx.host { host }` |

The pipeline drives resolution — it calls `aspects.resolve` for each host after walking the full context topology.

## Gotchas

- `den.lib.aspects.resolve` takes **three** arguments — forgetting the middle `[]` argument for aspect-chain is a common mistake causing a confusing error
- Resolution is **eager** for all declared entities at evaluation time — there is no lazy on-demand evaluation per host
- Adapters set on `den.ctx.host.meta.adapter` apply to **all** hosts in the config globally — use per-aspect adapters if you need per-host filtering
- The `aspect-chain` passed to static aspects accumulates in depth-first order — it is a list of the aspects traversed to reach the current point, useful for cycle detection and debugging
- `deferredModule` is class-specific — a module resolved for `"nixos"` cannot be passed to `darwin.lib.darwinSystem` and vice versa

## In the Nix ecosystem

Standard NixOS module evaluation has no equivalent of resolution. `lib.evalModules` takes a flat list of modules and merges them. Den's resolution is the step that produces that flat list — collapsing a recursive DAG of parametric, context-aware aspect functions into the flat module list that `evalModules` expects.

This is why Den does not call `lib.evalModules` directly — it builds the modules list first through resolution, then hands it to the standard NixOS API.

## Related pages

- [[aspect]] — the data structure resolution operates on
- [[aspect-dag]] — the dependency graph resolution traverses
- [[parametric-dispatch]] — how function includes are filtered during ctxApply
- [[deferredModule]] — the output type of resolution
- [[den-lib]] — den.lib.aspects.resolve is the resolution API
- [[context-transformation-pipeline]] — the pipeline that drives resolution for each host
- [[collectPairs]] — internal function that walks into.* transitions before resolution
- [[dedupIncludes]] — internal function that prevents duplicate owned config application
- [[deduplication]] — the problem dedupIncludes solves during resolution
- [[meta.adapter]] — how to filter which aspects participate in resolution