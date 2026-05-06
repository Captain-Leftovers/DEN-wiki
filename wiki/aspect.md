# Aspect

**What it is**: A composable attrset that owns all configuration for a single feature across every Nix class it touches.

**Why it exists**: Traditional Nix configs scatter a feature across platform-specific files — the NixOS part here, the Home Manager part there. An aspect keeps everything for one concern (bluetooth, gaming, dev-tools) in one place, and makes it generic enough to share with anyone.

**Relates to**: [[nix-class]], [[parametric-dispatch]], [[includes]], [[provides]], [[context-transformation-pipeline]], [[resolution]], [[den.aspects]], [[den.default]], [[aspect-dag]], [[hasAspect]], [[meta.adapter]]

---

## Core idea

An aspect is an attrset keyed by [[nix-class]] names, plus optional `includes` and `provides` fields:

```nix
bluetooth = {
  nixos.hardware.bluetooth.enable = true;
  homeManager.services.blueman-applet.enable = true;
  darwin.networking.bluetooth.enable = true;
};
```

One place. All platforms. Adding this feature to a new host is one line (`includes = [ den.aspects.bluetooth ]`). Removing it is deleting that line.

Aspects can also be **parametric functions** — they receive context (`host`, `user`, `home`) as real function arguments and return the attrset above. This makes them generic: the aspect knows nothing about the caller's specific setup.

```nix
bluetooth = { host, user }: {
  nixos.hardware.bluetooth.enable = true;
  nixos.users.users.${user.userName}.extraGroups = [ "bluetooth" ];
};
```

The context args (`host`, `user`) are **not** `_module.args` or `specialArgs`. They are evaluated before the module system, which makes them safe for conditional imports — something impossible with `_module.args` without triggering Nix infinite recursion.

## Aspect vs module

An aspect is **not** a module — it is a Den-specific container that **holds** modules:

```
┌─────────────────────────────────────────────┐
│  den.aspects.bluetooth  ← ASPECT (wrapper)    │
│  (Den infrastructure, not a NixOS module)     │
├─────────────────────────────────────────────┤
│  nixos = { ... }  ← NixOS module fragment   │
│  homeManager = { pkgs, ... }: { ... }       │
│                  ← Home Manager module       │
│  darwin = { ... }  ← Darwin module fragment │
│  includes = [ ... ]  ← list of aspects      │
│  provides = { ... }  ← named sub-aspects    │
└─────────────────────────────────────────────┘
```

The payload inside each class key is a real module. `homeManager = { pkgs, ... }: { ... }` is a standard Home Manager module that takes normal HM arguments. `nixos = { pkgs, ... }: { ... }` is a standard NixOS module fragment. Den's pipeline routes each payload to the right destination (`lib.nixosSystem`, `home-manager.users.<name>`, etc.).

What the aspect adds is **infrastructure glue**:
- **Routing**: the `nixos` key goes to the OS build, `homeManager` goes to the user's home
- **Dependencies**: `includes = [ den.aspects.dev-tools ]` declares what other aspects this one needs
- **Parametric dispatch**: functions receive `host`, `user`, `home` before the module system runs

### Compare: traditional module vs Den aspect

**Traditional standalone HM module** (`home.nix`):
```nix
{ pkgs, ... }:
{
  programs.git.enable = true;
  home.packages = [ pkgs.htop ];
}
```

**Den equivalent** (same content, wrapped in an aspect):
```nix
den.aspects.alice = {
  homeManager = { pkgs, ... }: {
    programs.git.enable = true;
    home.packages = [ pkgs.htop ];
  };
};
```

The braces inside are identical. The outer wrapper tells Den where to send it. The `includes` list references other **aspects** (Den data structures), not raw modules — Den resolves them, extracts the right class from each, and produces the final merged module for `lib.evalModules`.

## Structure

Every aspect has exactly three kinds of attributes:

| Attribute | Kind | Behaviour |
|---|---|---|
| `nixos`, `darwin`, `homeManager`, etc. | [[nix-class]]-keyed owned config | Always included in resolution for that class — unconditional |
| `includes` | Dependency list | A list of other aspects, static attrsets, or parametric functions forming the [[aspect-dag]] |
| `provides` | Named sub-aspects | Sub-aspects scoped to this aspect, accessible via `aspect._.name` or `aspect.provides.name` |

### Owned configs

Class-keyed attributes are called **owned configs**. They can be plain attrsets or function modules:

```nix
# Attrset form
den.aspects.laptop.nixos.networking.hostName = "laptop";

# Function form (takes normal NixOS module args)
den.aspects.laptop.nixos = { pkgs, ... }: {
  environment.systemPackages = [ pkgs.git ];
};
```

Owned configs are always included — they are not subject to [[parametric-dispatch]].

### Includes

`includes` is a list that can hold three kinds of values:

1. **Static attrset** — `{ nixos.foo = ...; }` — applied unconditionally
2. **Static aspect leaf** — `{ class, aspect-chain }: { ${class}.foo = ...; }` — evaluated once during [[resolution]], receives the class being resolved
3. **Parametric function** — `{ host, user }: { ... }` — evaluated per context via [[parametric-dispatch]]; silently skipped when the context doesn't match the function's arguments

```nix
den.aspects.workstation.includes = [
  den.aspects.dev-tools          # full DAG included transitively
  { nixos.time.timeZone = "UTC"; } # static, always applied
  ({ host, ... }: { nixos.networking.hostName = host.hostName; }) # parametric
];
```

Unlike stringly-typed tags (`modules = [ "base" "gaming" ]`), includes are real Nix references — type-checked by the module system.

### Provides

`provides` creates named sub-aspects scoped to the parent. The critical thing to understand: **`provides` sub-aspects are NOT included when you include the parent aspect**. They are opt-in extras that must be explicitly requested.

```nix
den.aspects.gaming = {
  nixos.programs.steam.enable = true;        # included when you include gaming
  homeManager.programs.discord.enable = true; # included when you include gaming

  provides.emulation = {
    nixos.programs.retroarch.enable = true;  # NOT included unless explicitly requested
    nixos.hardware.opengl.enable = true;
  };
};
```

```nix
# Gets steam + discord only — emulation is NOT pulled in
den.aspects.laptop.includes = [ den.aspects.gaming ];

# Gets ONLY emulation — not steam or discord
den.aspects.laptop.includes = [ den.aspects.gaming._.emulation ];

# Gets everything
den.aspects.laptop.includes = [
  den.aspects.gaming
  den.aspects.gaming._.emulation
];
```

Think of `provides` as opt-in modular extras. The main aspect is the common baseline; `provides` sub-aspects are things you can pick from selectively.

## Auto-generation

Den automatically creates a `parametric` aspect for every declared host and user. Given:

```nix
den.hosts.x86_64-linux.igloo.users.tux = {
  classes = [ "homeManager" ];
};
```

Den produces:

```nix
den.aspects.igloo = parametric { nixos = {}; };
den.aspects.tux   = parametric { homeManager = {}; };
```

You extend these auto-created aspects — you do not replace them. Any file can contribute to any aspect at any time ([[incremental-features]]).

### Why does Den do this?

The auto-generated aspect is a **shared hook point** — it's what allows any file anywhere in your config to contribute configuration for a specific host or user, without a central import list.

```nix
# file: modules/hosts/igloo.nix
den.aspects.igloo = {
  includes = [ den.provides.hostname ];
  nixos.networking.firewall.enable = true;
};

# file: modules/users/tux.nix
den.aspects.tux = {
  homeManager.programs.git.enable = true;
};

# file: modules/shared/dev.nix
den.aspects.igloo = {
  nixos.environment.systemPackages = [ pkgs.git ];  # third file also contributing
};
```

All three files contribute to `igloo`'s aspect. Den merges them. No file needs to know about the others. No central `imports = [ ... ]` list to maintain.

This is the separation between **declaring existence** (`den.hosts`) and **defining behavior** (`den.aspects`). The host declaration says "this machine exists." The aspect is where any part of your config can say "and here's what it should do." The auto-generated aspect is the bridge between the two.

## The __functor

Every aspect has a `__functor` installed by [[parametric-dispatch|den.lib.parametric]]. This makes the attrset callable:

```nix
# Calling an aspect with a context activates parametric dispatch
result = den.aspects.laptop { host = ...; user = ...; };
```

The functor filters the `includes` list based on which functions' argument signatures match the given context, then returns the matching config fragments. Owned configs and static includes are always returned.

## Meta-data

Every aspect has a `meta` submodule (freeformType):

| Field | Value |
|---|---|
| `meta.name` | `"den.aspects.igloo"` — derived from location |
| `meta.loc` | `[ "den" "aspects" "igloo" ]` |
| `meta.file` | Last file where this aspect was defined |
| `meta.self` | Reference to the aspect's own module config |

`meta.self` enables the [[aspect-fixed-point]] pattern — an aspect referencing its own config values.

`meta.adapter` controls which aspects from the `includes` DAG are actually resolved — a function that filters or transforms the aspect list transitively.

`meta.provider` tracks structural origin — top-level aspects have `[]`; `foo.provides.bar` has `["foo"]`.

## Gotchas

- Anonymous functions in `includes` are an anti-pattern — use named aspects instead for readable error traces
- Owned configs are **always** included regardless of context — use parametric functions in `includes` if you need conditional behaviour
- `den.default` is a special aspect applied globally to all hosts, users, and homes — its owned configs are deduplicated across pipeline stages; its parametric includes run at every stage
- Aspects are fixed-point — `{ config, ... }:` in an aspect gives access to the aspect's own resolved config, enabling self-referential values

## In the Nix ecosystem

The aspect concept is Den's answer to the **configuration matrix problem**: features × platforms. Traditional NixOS configs fill this matrix column by column (all NixOS together, all HM together). Aspects fill it row by row (all bluetooth together).

The `aspect.class` transposition pattern was first introduced by **Unify** (the first dendritic framework). Den extends it with parametric functions, real dependency graphs, and a namespace system for sharing.

Compared to plain NixOS modules: a NixOS module is one cell in the matrix. An aspect is one row.

## Related pages

- [[nix-class]] — the class names that key owned configs (nixos, darwin, homeManager, etc.)
- [[parametric-dispatch]] — how aspect functions are matched to contexts
- [[includes]] — how the dependency DAG is declared and traversed
- [[provides]] — how sub-aspects are created and accessed
- [[aspect-dag]] — the directed acyclic graph formed by includes
- [[resolution]] — how an aspect is resolved into a deferredModule for a specific class
- [[den.aspects]] — the attrset holding all named aspects in a Den config
- [[den.default]] — the global aspect applied to all entities
- [[context-transformation-pipeline]] — the pipeline that feeds context data into aspect functions
- [[aspect-fixed-point]] — the pattern for self-referential aspects
- [[incremental-features]] — any file can contribute to any aspect at any time
- [[project-structure]] — how to organize files that define aspects