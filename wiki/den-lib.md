# den.lib

**What it is**: The domain-agnostic core library of Den — provides parametric dispatch primitives, the aspects API, and utility functions that work for any Nix module domain, not just NixOS/Darwin.

**Why it exists**: Den's framework (hosts, homes, batteries) is useful for NixOS/Darwin setups, but the underlying dispatch machinery is general-purpose. `den.lib` is that machinery exposed directly — usable for Terranix, NixVim, system-manager, or any custom Nix module system.

**Relates to**: [[parametric-dispatch]], [[aspect]], [[resolution]], [[den-as-library]], [[context-transformation-pipeline]], [[namespaces]], [[angle-bracket-syntax]]

---

## Core idea

`den.lib` is what Den is built on. The framework (`den.hosts`, `den.ctx`, `den.provides`) is a layer on top of it. You can use `den.lib` without any of the framework machinery.

```nix
# Library-only usage — no den.hosts, no den.ctx, no batteries
aspect = den.lib.parametric {
  terranix.resource.aws_instance.web = { ami = "ami-123"; };
  includes = [
    ({ env, ... }: { terranix.resource.aws_instance.web.tags.Env = env; })
  ];
};

resolved = den.lib.aspects.resolve "terranix" [] (aspect { env = "production"; });
```

## API reference

### Parametric dispatch

| Function | Purpose |
|---|---|
| `den.lib.parametric` | Default constructor — installs a `__functor` that dispatches includes via `canTake.atLeast`; owned configs and statics always included |
| `den.lib.parametric.atLeast` | Like parametric but skips owned configs and statics — only dispatches functions matching atLeast |
| `den.lib.parametric.exactly` | Like atLeast but uses `canTake.exactly` |
| `den.lib.parametric.fixedTo attrs` | Ignores received context — always dispatches with the given fixed attrs |
| `den.lib.parametric.expands attrs` | Extends received context with attrs before dispatch |
| `den.lib.canTake` | `canTake ctx fn` — checks if `fn`'s required args are satisfiable by `ctx` (atLeast mode) |
| `den.lib.canTake.atLeast` | Explicit atLeast mode — all required params present, extras allowed |
| `den.lib.canTake.exactly` | Exact mode — required params match context keys exactly |
| `den.lib.take.atLeast fn ctx` | Apply `fn ctx` only if ctx satisfies atLeast; otherwise return empty |
| `den.lib.take.exactly fn ctx` | Apply `fn ctx` only if ctx satisfies exactly; otherwise return empty |
| `den.lib.perUser fn` | Wraps `fn` to only match `{ host, user }` contexts exactly |
| `den.lib.perHome fn` | Wraps `fn` to only match `{ home }` contexts exactly |
| `den.lib.isFn` | Returns true if value is a function or has `__functor` |

### Aspects API

| Function | Purpose |
|---|---|
| `den.lib.aspects.resolve class [] aspect` | Resolves an aspect for a specific class — walks includes DAG, applies parametric dispatch, deduplicates, returns a `deferredModule` |
| `den.lib.statics aspect` | Extracts only static includes from an aspect (excludes parametric functions and functor) |
| `den.lib.owned aspect` | Extracts only owned configs (class-keyed attributes) — no includes, no functor |
| `den.lib.aspects.adapters.filter pred` | Creates an adapter function that filters aspect includes by predicate during resolution |

### Navigation

| Function | Purpose |
|---|---|
| `den.lib.__findFile` | Enables angle bracket syntax `<namespace/path>` — wire into `_module.args.__findFile` |

## canTake in detail

`canTake` is implemented in `nix/fn-can-take.nix`. It calls `builtins.functionArgs fn` to get the function's declared argument names, then checks them against the provided context attrset.

```nix
# atLeast — the default
canTake { host = x; }             ({ host, ... }: ...)   # true  — host present
canTake { host = x; }             ({ host, user }: ...)  # false — user missing
canTake { host = x; user = y; }   ({ host, ... }: ...)   # true  — extra keys fine

# exactly
canTake.exactly { host = x; }              ({ host }: ...)        # true
canTake.exactly { host = x; user = y; }    ({ host }: ...)        # false — extras
canTake.exactly { host = x; }              ({ host, ... }: ...)   # false — fn has ellipsis
```

The `...` in a function signature means "accept extra keys" — `canTake.exactly` treats a function with `...` as not exactly matching any context with fewer keys.

## parametric variants in detail

| Variant | Owned configs included? | Static includes? | Parametric matching |
|---|---|---|---|
| `parametric` | ✓ always | ✓ always | atLeast |
| `parametric.atLeast` | ✗ | ✗ | atLeast |
| `parametric.exactly` | ✗ | ✗ | exactly |
| `parametric.fixedTo attrs` | ✓ always | ✓ always | atLeast, but with fixed context |
| `parametric.expands attrs` | ✓ always | ✓ always | atLeast, context extended with attrs |

`parametric.fixedTo` is used internally by [[den-ctx]] when binding a host's aspect — it fixes context to `{ host }` so the aspect always receives the correct data regardless of pipeline stage.

`parametric.atLeast` is used by [[deduplication]] (dedupIncludes) for subsequent occurrences of an aspect — owned configs already applied on first occurrence, only parametric matches needed thereafter.

## Resolving an aspect manually

```nix
# 1. Create an aspect
my-aspect = den.lib.parametric {
  terranix.resource.aws_instance.web = { instance_type = "t3.micro"; };
  includes = [
    ({ env }: { terranix.resource.aws_instance.web.tags.Env = env; })
  ];
};

# 2. Call it with a context to activate parametric dispatch
bound = my-aspect { env = "production"; };

# 3. Resolve for a specific class
module = den.lib.aspects.resolve "terranix" [] bound;

# 4. Pass to your module system
terranixConfig = terranix.lib.terranixConfiguration {
  modules = [ module ];
};
```

This three-step pattern (create → call with context → resolve for class) is exactly what [[den-ctx]] automates for NixOS/Darwin hosts.

## Library-only vs framework

| Use case | Recommendation |
|---|---|
| NixOS / Darwin / Home Manager hosts | Use the full framework — `den.hosts`, `den.ctx`, `den.provides` |
| Custom Nix domain (Terranix, NixVim, system-manager) | Use `den.lib` directly — skip the framework entirely |
| OS config + custom domain in same flake | Use framework for OS, `den.lib` for the custom domain |

The framework is entirely optional. Nothing in `den.lib` references `den.hosts` or `den.ctx`.

## den.lib.fx (experimental)

`den.lib.fx` is an experimental algebraic effects API on a feature branch — not yet on main. It provides:

- `fx.pure value` — wrap a plain value as an effect
- `fx.send "name" payload` — declare an effect request (dependency)
- `fx.bind comp fn` — chain effectful computations
- `fx.bind.fn attrs fn` — convert any plain Nix function into an effectful computation by auto-generating requests from its argument names via `builtins.functionArgs`
- `fx.handle { handlers; state; } comp` — run a computation with handlers; returns `{ state; value; }`

Den is migrating to use `nix-effects` (vic/nix-effects) as its core dependency injection system, replacing the current hand-rolled context threading. The motivation is better context control and deduplication detection. The user-facing aspect syntax is not changing.

See [[algebraic-effects]] for the conceptual background.

## Gotchas

- `den.lib` has no runtime dependencies — it is a standalone library; the `flakeModule` expects to be imported in a module providing `nixpkgs.lib`
- `den.lib.aspects.resolve` takes three arguments: `class`, `aspect-chain` (usually `[]`), and the `aspect` — forgetting the empty list for aspect-chain is a common mistake
- `den.lib.statics` and `den.lib.owned` are for inspection/tooling — in normal usage aspects are resolved via `aspects.resolve`, not extracted manually
- `den.lib.fx` is experimental and only available on the feature branch — do not use in production configs yet

## In the Nix ecosystem

`den.lib` has no direct equivalent in the Nix ecosystem. `callPackage` is the closest analogy for the dispatch pattern, but it is one-shot and nixpkgs-specific. `den.lib.parametric` is a generalized, context-aware, multi-stage dispatch system for any Nix module domain.

The algebraic effects work (`den.lib.fx`) places Den in conversation with research on freer monads and ability-based programming in functional languages — unusual territory for a Nix library.

## Related pages

- [[parametric-dispatch]] — the dispatch mechanism den.lib implements
- [[aspect]] — the data structure den.lib operates on
- [[resolution]] — den.lib.aspects.resolve in detail
- [[den-as-library]] — when and how to use den.lib without the framework
- [[context-transformation-pipeline]] — what den.ctx builds on top of den.lib
- [[angle-bracket-syntax]] — den.lib.__findFile enables this syntax
- [[algebraic-effects]] — the theory behind den.lib.fx
- [[deduplication]] — how parametric.fixedTo and parametric.atLeast prevent duplicate config