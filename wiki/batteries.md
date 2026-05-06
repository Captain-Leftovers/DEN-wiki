# Batteries

**What it is**: Den's built-in library of reusable aspects shipped under `den.provides.*` (aliased `den._.*`) — pre-built parametric aspects handling common cross-platform configuration patterns.

**Why it exists**: Every Den setup needs the same handful of things: create user accounts, set hostnames, mark a primary user, set a shell. Without batteries you'd write these from scratch every time. Batteries are the answer to "what do I include to get a working system?"

**Relates to**: [[aspect]], [[parametric-dispatch]], [[context-transformation-pipeline]], [[den-ctx]], [[den-schema]], [[includes]], [[den.default]], [[home-environments]], [[mutual-provider]]

---

## Core idea

Batteries are aspects. They live in `den.provides.*` (or the `den._.*` alias) and are included exactly like any other aspect:

```nix
den.aspects.alice.includes = [
  den.provides.primary-user
  (den.provides.user-shell "fish")
  (den.provides.unfree [ "vscode" ])
];
```

Some batteries take arguments (like `user-shell` and `unfree`). Some are plain aspect references. All are implemented using [[parametric-dispatch]] — they fire only in the contexts they were designed for.

## Usage patterns

### Global — apply to all entities

```nix
den.default.includes = [
  den.provides.define-user
  den.provides.hostname
  den.provides.inputs'
];
```

### Per-aspect — apply to specific hosts or users

```nix
den.aspects.alice.includes = [
  den.provides.primary-user
  (den.provides.user-shell "zsh")
];
```

### Composed into a new aspect

```nix
den.aspects.admin-user = den.lib.parametric {
  includes = [
    den.provides.primary-user
    (den.provides.user-shell "fish")
    { nixos.security.sudo.wheelNeedsPassword = false; }
  ];
};
```

Batteries compose freely with regular aspects and with each other.

## System batteries

### `den.provides.define-user` / `den._.define-user`

Creates the user account and home directory on every platform.

| Platform | Effect |
|---|---|
| **NixOS** | `users.users.<name>.isNormalUser = true; users.users.<name>.home = "/home/<name>"` |
| **Darwin** | `users.users.<name>.home = "/Users/<name>"` |
| **Home Manager (standalone)** | `home.username = "<name>"; home.homeDirectory = "/home/<name>"` |

Declare the user once in `den.hosts` or `den.homes`, include this battery once in `den.default`, and the user exists everywhere they should.

```nix
den.default.includes = [ den.provides.define-user ];
```

### `den.provides.hostname` / `den._.hostname`

Sets the system hostname from `den.hosts.<name>.hostName`.

- Works on NixOS, Darwin, WSL
- Uses [[parametric-dispatch]] to read `host.hostName` from the schema

```nix
den.default.includes = [ den.provides.hostname ];
```

### `den.provides.os-user` / `den._.os-user`

Automatically enabled by Den internally. Forwards user settings into `host.users.users.<userName>` on NixOS and Darwin.

### `den.provides.primary-user` / `den._.primary-user`

Marks a user as the primary/admin account.

| Platform | Effect |
|---|---|
| **NixOS** | Adds `wheel` and `networkmanager` to `users.users.<name>.extraGroups`; ensures `isNormalUser` |
| **Darwin** | Sets `system.primaryUser = "<name>"` |
| **WSL** | Sets `defaultUser = "<name>"` |

Only the user that includes this battery gets admin privileges. If two users exist on a host, only the one with `primary-user` gets `sudo`.

```nix
den.aspects.alice.includes = [ den.provides.primary-user ];
```

### `den.provides.user-shell "shell"` / `den._.user-shell`

Sets the login shell at both OS and Home Manager levels.

| Platform | Effect |
|---|---|
| **NixOS** | `users.users.<name>.shell = pkgs.<shell>; programs.<shell>.enable = true` |
| **Darwin** | Shell assignment (Darwin equivalent) |
| **Home Manager** | `programs.<shell>.enable = true` |

One line sets the shell everywhere. In a traditional setup you set `users.users.alice.shell` in NixOS config AND `programs.fish.enable` in Home Manager config, keeping them in sync manually. The battery does both via [[parametric-dispatch|parametric dispatch]].

```nix
den.aspects.alice.includes = [ (den.provides.user-shell "fish") ];
```

### Why `den.default.includes` works everywhere

`den.default` is an aspect processed at **every** pipeline stage. Batteries placed here use `parametric.exactly` internally to provide multiple context-specific implementations in one aspect:

```nix
# Simplified: what define-user does internally
den.lib.parametric {
  includes = [
    # NixOS/Darwin host-bound user
    ({ host, user }: {
      nixos.users.users.${user.userName}.isNormalUser = true;
      nixos.users.users.${user.userName}.home = user.homeDirectory;
    })
    # Standalone home
    ({ home }: {
      homeManager.home.username = home.userName;
      homeManager.home.homeDirectory = home.homeDirectory;
    })
  ];
}
```

Den routes each implementation to the right context automatically. You include the battery once; it adapts to wherever it runs.

### `den.provides.mutual-provider` / `den._.mutual-provider`

Allows hosts and users to contribute configuration to each other via `.provides.`. Without this battery, aspects only flow down the pipeline — with it, a user can push config into its host and vice versa.

See [[mutual-provider]] for the full guide with examples, direction patterns, and gotchas. Brief usage:

```nix
# Enable globally for all user contexts
den.ctx.user.includes = [ den._.mutual-provider ];

# User contributing to a specific host
den.aspects.alice.provides.my-laptop = { host, ... }: {
  nixos.programs.nh.enable = host.name == "my-laptop";
};

# Host contributing to a specific user
den.aspects.my-laptop.provides.alice = { user, ... }: {
  homeManager.programs.helix.enable = user.name == "alice";
};
```

### `den.provides.tty-autologin "user"` / `den._.tty-autologin`

Configures TTY1 auto-login via `services.getty.autologinUser`. NixOS only.

```nix
den.aspects.my-laptop.includes = [ (den.provides.tty-autologin "alice") ];
```

### `den.provides.wsl` / `den._.wsl`

WSL-specific activation. Enables `den.ctx.wsl-host` when `host.wsl.enable = true` and imports the NixOS-WSL module.

### `den.provides.forward` / `den._.forward`

Forwards aspect configuration from one entity's `.resolved` into another entity. Used for cross-context forwarding and creating custom Nix classes.

```nix
# Collect SSH host keys from all other hosts into this host's config
den.aspects.my-host.includes = [
  ({ host }:
    den._.forward {
      each       = lib.filter (h: h != host) (lib.attrValues den.hosts.${host.system});
      fromClass  = _: "ssh-host-key";
      intoClass  = _: host.class;
      intoPath   = _: [];
    }
  )
];
```

### `den.provides.import-tree` / `den._.import-tree`

Imports trees of non-dendritic `.nix` files into Den, auto-detecting class from directory names (`_nixos/`, `_darwin/`, `_homeManager/`). Requires `inputs.import-tree`.

```nix
den.aspects.laptop.includes = [ (den.provides.import-tree ./legacy-modules) ];
```

### `den.provides.unfree [ "pkg" ]` / `den._.unfree`

Allows specific unfree packages by name via `nixpkgs.config.allowUnfreePredicate`. Works for any class (nixos, darwin, homeManager).

```nix
den.aspects.laptop.includes = [ (den.provides.unfree [ "nvidia-x11" "steam" ]) ];
```

### `den.provides.insecure [ "pkg" ]` / `den._.insecure`

Allows specific insecure packages by name. Same pattern as unfree.

## Flake-parts batteries

These only activate when `inputs.flake-parts` is present.

### `den.provides.inputs'` / `den._.inputs'`

Exposes flake-parts `inputs'` (system-qualified inputs) as a module argument in aspect modules. Works in host, user, and home contexts.

```nix
den.default.includes = [ den.provides.inputs' ];
# Now aspects can use: { inputs', ... }: { nixos.environment.systemPackages = [ inputs'.some-flake.packages.tool ]; }
```

### `den.provides.self'` / `den._.self'`

Exposes flake-parts `self'` (system-qualified self outputs) as a module argument.

## Home environment batteries

These batteries register the derived context stages for home environment integration. They are activated automatically when the relevant input is present and `user.classes` includes the matching class.

### Home Manager battery

- Defines `den.ctx.hm-host` — activates when at least one host user has `"homeManager"` in their classes
- Imports `home-manager.nixosModules.home-manager` (or darwin equivalent) at the hm-host stage
- Defines `den.ctx.hm-user` — forwards `homeManager` class content to `home-manager.users.<userName>`
- Exposes `home-manager.useGlobalPkgs` and `home-manager.useUserPackages` at the hm-host stage

### hjem battery

- Defines `den.ctx.hjem-host` and `den.ctx.hjem-user`
- Sets `hjem.users.<name>` from the user's `hjem` class content
- Requires `inputs.hjem`

### nix-maid battery

- Defines `den.ctx.maid-host` and `den.ctx.maid-user`
- Merges maid config into `users.users.<name>.maid`
- Requires `inputs.nix-maid`; NixOS only

## How batteries register context stages

Batteries are not special-cased in Den's core. They register their context stages by contributing to `den.ctx` as regular module options:

```nix
# Simplified: what the HM battery does internally
den.ctx.host.into.hm-host = { host }:
  lib.optional (host.home-manager.enable && hasHmUsers host) { inherit host; };

den.ctx.hm-host.provides.hm-host = { host }: {
  nixos = { ... }: { imports = [ inputs.home-manager.nixosModules.home-manager ]; };
};
```

This is why batteries must be imported to have effect — if the battery module is not loaded, its `into.*` transition never exists and the derived context never fires.

## Gotchas

- `den._` is an alias for `den.provides` — they are identical; `den._` is just shorter to type
- Batteries that take arguments (`user-shell`, `unfree`, `tty-autologin`) return an aspect when called — wrap them in parentheses in the includes list: `includes = [ (den.provides.user-shell "fish") ]`
- Home environment batteries (HM, hjem, maid) are activated by `user.classes` — the battery must be imported AND the user must declare the matching class
- `den.provides.define-user` works in both `{host, user}` and `{home}` contexts because it uses `parametric.exactly` with two separate includes functions — one per context shape
- Adding `mutual-provider` to `den.ctx.user.includes` affects ALL user contexts globally — use it in specific aspect includes if you only want it for certain users

## In the Nix ecosystem

Batteries are Den's equivalent of NixOS modules like `programs.git.enable` — pre-packaged configuration patterns with sensible defaults. The difference is that batteries are cross-platform (one battery for nixos + darwin + homeManager), parametric (they read host/user data), and composable (they're just aspects).

The name "batteries" comes from "batteries included" — Den ships with enough common patterns that most setups don't need to rewrite the basics.

## Related pages

- [[aspect]] — batteries are aspects; they follow all the same rules
- [[parametric-dispatch]] — batteries use parametric.exactly to target specific context shapes
- [[den.default]] — the common place to put batteries that apply globally
- [[includes]] — batteries are added to includes lists like any other aspect
- [[context-transformation-pipeline]] — home environment batteries register derived context stages
- [[den-ctx]] — batteries hook into the pipeline via den.ctx includes and into transitions
- [[home-environments]] — the HM, hjem, and maid batteries in detail
- [[den-schema]] — batteries read from schema fields like host.hostName and user.userName