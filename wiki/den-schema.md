# Den Schema

**What it is**: The declaration layer of Den — `den.hosts`, `den.homes`, and `den.schema` define what machines, users, and homes exist in your infrastructure as structured data that the pipeline traverses.

**Why it exists**: Den separates *what exists* (schema) from *what it does* (aspects). Your machines are data first. The pipeline reads that data and activates the right aspects — nothing is hardcoded into the config logic.

**Relates to**: [[context-transformation-pipeline]], [[aspect]], [[nix-class]], [[den.ctx]], [[resolution]], [[freeform-type]], [[auto-generation]], [[batteries]], [[meta.adapter]]

---

## Core idea

Den's schema layer is three options:

| Option | Purpose | Produces |
|---|---|---|
| `den.hosts` | Declare machines and their users | `nixosConfigurations`, `darwinConfigurations` |
| `den.homes` | Declare standalone home environments | `homeConfigurations` |
| `den.schema` | Define shared options/defaults for all entities | Nothing directly — metadata aspects read |

These are not aspects. They are structured data declarations. The pipeline reads them; aspects act on them.

## den.hosts

Keyed by `<system>.<name>`:

```nix
den.hosts.x86_64-linux.laptop = {
  users.alice = { };
};

den.hosts.aarch64-darwin.mac = {
  users.alice = { };
  brew.apps = [ "iterm2" ];   # freeform attribute
};
```

### Host options

| Option | Type | Default | Description |
|---|---|---|---|
| `name` | str | attr key | Config name |
| `hostName` | str | `name` | Network hostname |
| `system` | str | parent key | Platform — `x86_64-linux`, `aarch64-darwin`, etc. |
| `class` | str | auto | `"nixos"` or `"darwin"` — derived from system |
| `aspect` | str | `name` | Primary aspect name — which `den.aspects.<name>` to use |
| `users` | attrsOf userType | `{}` | User accounts on this host |
| `instantiate` | raw | auto | Builder function — `lib.nixosSystem`, `darwinSystem`, etc. |
| `intoAttr` | listOf str | auto | Flake output path |
| `resolved` | raw | auto | Aspect produced by running the host through its context pipeline |
| `*` | — | — | Any freeform attribute — accessible in aspects via `host.<attr>` |

### instantiate defaults

| Class | Default |
|---|---|
| `nixos` | `inputs.nixpkgs.lib.nixosSystem` |
| `darwin` | `inputs.darwin.lib.darwinSystem` |
| `systemManager` | `inputs.system-manager.lib.makeSystemConfig` |

### intoAttr defaults

| Class | Default |
|---|---|
| `nixos` | `[ "nixosConfigurations" name ]` |
| `darwin` | `[ "darwinConfigurations" name ]` |
| `systemManager` | `[ "systemConfigs" name ]` |

Set `intoAttr = []` to skip placing the result in flake outputs entirely.

## den.hosts users

Each host has a `users` attrset. Users are declared inside the host:

```nix
den.hosts.x86_64-linux.laptop.users = {
  alice = { };
  bob.classes = [ "homeManager" "hjem" ];
};
```

### User options

| Option | Type | Default | Description |
|---|---|---|---|
| `name` | str | attr key | User config name |
| `userName` | str | `name` | System account name |
| `classes` | listOf str | `[ "homeManager" ]` | Home environment classes — controls which derived contexts activate |
| `aspect` | str | `name` | Primary aspect name |
| `resolved` | raw | auto | Aspect produced by running the user through its context pipeline |
| `*` | — | — | Any freeform attribute |

`classes` is the most important user option — it drives which derived context stages fire. `[ "homeManager" ]` activates `hm-user`. `[ "hjem" ]` activates `hjem-user`. Multiple classes activate multiple stages simultaneously.

## den.homes

Standalone home-manager configurations — for machines without root access, or for managing a home independently from its host:

```nix
den.homes.x86_64-linux.alice = { };
den.homes.x86_64-linux."alice@laptop" = { };  # bound to specific hostname
```

### Home options

| Option | Type | Default | Description |
|---|---|---|---|
| `name` | str | attr key | Home config name |
| `userName` | str | `name` | System account name |
| `system` | str | parent key | Platform |
| `class` | str | `"homeManager"` | Home management class |
| `aspect` | str | `name` | Primary aspect name |
| `pkgs` | raw | `inputs.nixpkgs.legacyPackages.${system}` | Nixpkgs instance |
| `instantiate` | raw | `inputs.home-manager.lib.homeManagerConfiguration` | Builder |
| `intoAttr` | listOf str | `[ "homeConfigurations" name ]` | Output path |
| `resolved` | raw | auto | Aspect from context pipeline |
| `*` | — | — | Any freeform attribute |

### host-bound homes

Naming a home `"user@hostname"` makes the `home-manager` CLI auto-select it when `whoami == user` and `hostname == hostname`. The hostname does not need to be a `den.hosts` entry — it can be an externally managed machine.

## den.schema

`den.schema.*` defines shared options and defaults applied to all entities of a given kind. It is not an aspect — it defines the *metadata* that aspects later read.

```nix
# Applied to all hosts, users, and homes
den.schema.conf = { lib, ... }: {
  options.owner = lib.mkOption { type = lib.types.str; default = "unknown"; };
};

# Applied to all hosts only
den.schema.host = { host, lib, ... }: {
  options.hardened = lib.mkEnableOption "hardened profile";
  config.hardened  = lib.mkDefault true;
};

# Applied to all users only
den.schema.user = { user, lib, ... }: {
  config.classes = lib.mkDefault [ "homeManager" ];
};
```

| Option | Type | Applied to |
|---|---|---|
| `den.schema.conf` | deferredModule | All hosts, users, and homes |
| `den.schema.host` | deferredModule | All hosts (imports conf) |
| `den.schema.user` | deferredModule | All users (imports conf) |
| `den.schema.home` | deferredModule | All homes (imports conf) |

Schema options become attributes on the entity — `host.hardened`, `user.classes` — readable in aspects and batteries.

## Freeform attributes

Host, user, and home types all use `freeformType` — any attribute not matching a defined option passes through as raw metadata:

```nix
den.hosts.x86_64-linux.laptop = {
  users.alice = { };
  gpu = "nvidia";          # custom freeform attribute
  location = "homelab";    # another freeform attribute
};
```

Access freeform attributes in aspects via the context arg:

```nix
({ host, ... }: lib.optionalAttrs (host ? gpu) {
  nixos.hardware.nvidia.enable = true;
})
```

## Auto-generation of aspects

When you declare a host or user, Den automatically creates a `parametric` aspect for it in `den.aspects`:

```nix
# This declaration:
den.hosts.x86_64-linux.igloo.users.tux = {
  classes = [ "homeManager" ];
};

# Auto-generates:
den.aspects.igloo = parametric { nixos = {}; };
den.aspects.tux   = parametric { homeManager = {}; };
```

You extend these auto-generated aspects — you never replace them. Any module file can contribute to any aspect at any time. See [[auto-generation]] and [[incremental-features]].

## entity.resolved

Every entity has a `resolved` field — the aspect produced by running it through its context pipeline. Used internally by Den to produce `host.mainModule`. 

To filter which aspects are included during resolution, set `meta.adapter` on the context node:

```nix
den.ctx.host.meta.adapter = inherited:
  den.lib.aspects.adapters.filter (a: a.name != "unwanted") inherited;
```

## Gotchas

- `den.schema.*` is NOT an aspect — it defines metadata options on the entity objects themselves, not configuration modules. Aspects read from these options; they don't live inside them.
- `host.class` (singular, auto-derived) vs `user.classes` (plural, explicitly set) — a common source of confusion.
- Default `user.classes` is `[ "homeManager" ]` — if you don't want HM, you must explicitly override this, either per user or via `den.schema.user`.
- `host.aspect` defaults to the host's attrset key name — so `den.hosts.x86_64-linux.laptop` auto-uses `den.aspects.laptop`. If the key name doesn't match an existing aspect, Den creates one automatically.
- Freeform attributes are not type-checked — assigning a typo'd attribute name silently succeeds and simply becomes inaccessible under the wrong name.

## In the Nix ecosystem

The separation of schema (what exists) from behavior (aspects) is Den's equivalent of the entity-component separation in game engines or the data/behavior split in functional programming. Most Nix frameworks treat the host config file as both declaration and behavior. Den makes them distinct layers.

`den.schema` plays the role that `flake-parts` options play in a flake-parts setup — a place to define typed, shared metadata accessible across all modules. The difference is that Den's schema is entity-specific (per host, per user, per home), not global.

## Related pages

- [[context-transformation-pipeline]] — the pipeline that reads schema data and traverses it
- [[aspect]] — what gets activated based on schema declarations
- [[auto-generation]] — Den creates aspects automatically from host/user declarations
- [[nix-class]] — host.class and user.classes are the key schema fields driving the pipeline
- [[freeform-type]] — how arbitrary metadata attributes work on host/user/home objects
- [[den.default]] — the global aspect applied to all entities regardless of schema
- [[batteries]] — batteries like define-user and hostname read from schema fields (host.hostName, user.userName)
- [[incremental-features]] — any file can extend auto-generated aspects from schema declarations
- [[project-structure]] — where to place host declarations and user declarations in your project