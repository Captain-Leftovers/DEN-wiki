# Home Environments

**What it is**: Den's integration layer for user-level configuration tools — Home Manager, hjem, and nix-maid — plus standalone homes for machines without root access.

**Why it exists**: NixOS and nix-darwin manage the OS. Home environments manage the user's personal config (dotfiles, user packages, programs). Den makes these two concerns composable: a user's homeManager aspect config flows into the right place automatically without manual wiring.

**Relates to**: [[aspect]], [[nix-class]], [[context-transformation-pipeline]], [[den-ctx]], [[batteries]], [[den-schema]], [[parametric-dispatch]]

---

## Core idea

In Den, home environment config lives in an aspect under the matching class key — `homeManager`, `hjem`, or `maid`:

```nix
den.aspects.alice = {
  homeManager = { pkgs, ... }: {
    home.packages = [ pkgs.htop ];
    programs.git.enable = true;
  };
  hjem = {
    files.".config/foo".text = "bar";
  };
};
```

Den routes each class to the right destination automatically via derived context stages in the [[context-transformation-pipeline]]. You never manually wire `home-manager.users.alice = ...`.

## Enabling home environments

All home integrations are opt-in via `user.classes`. The default is `[ "homeManager" ]`.

```nix
# Per user
den.hosts.x86_64-linux.igloo.users.alice.classes = [ "homeManager" "hjem" ];

# For all users via schema
den.schema.user = { lib, ... }: {
  config.classes = lib.mkDefault [ "homeManager" ];
};
```

A derived context stage only activates when:
1. At least one user has the matching class in `user.classes`
2. The corresponding input exists (`inputs.home-manager`, `inputs.hjem`, `inputs.nix-maid`)

## Home Manager

### How it works

The HM battery registers two derived context stages:

```
den.ctx.host
  └─ into.hm-host  (if host has ≥1 HM user)
       ├─ imports home-manager.nixosModules.home-manager
       └─ into.hm-user  (per HM-class user)
            └─ forwards homeManager class → home-manager.users.<userName>
```

1. `hm-host` stage: imports the HM OS module into the host's NixOS config
2. `hm-user` stage: takes the `homeManager` class content from the user's aspect and merges it into `home-manager.users.<userName>`

### Requirements

- `inputs.home-manager` must exist in flake inputs (or `host.home-manager.module` must be set)
- At least one user must have `"homeManager"` in their `classes`

### Configuration

```nix
den.aspects.alice = {
  homeManager = { pkgs, ... }: {
    home.packages = [ pkgs.htop ];
    programs.git = {
      enable = true;
      userName = "Alice";
    };
  };
};
```

The `homeManager` value is a standard Home Manager module. Den forwards it to `home-manager.users.alice` automatically.

### Home Manager as NixOS module vs standalone

There are two ways to use Home Manager. Den handles both; the difference is in where you declare the user.

| | **As NixOS module** (host-bound) | **Standalone** |
|---|---|---|
| **Declaration** | `den.hosts.x86_64-linux.igloo.users.alice = {}` | `den.homes.x86_64-linux.alice = {}` |
| **Build command** | `nixos-rebuild switch --flake .#igloo` | `home-manager switch --flake .#alice` |
| **Who builds it** | NixOS system build | User runs `home-manager` CLI |
| **Den pipeline** | `host → user → hm-host → hm-user` | `home` (shorter, no host context) |
| **OS-level access** | `osConfig` module arg available | `osConfig` is empty module |
| **VM testing** | `nix run .#vm-<host>` tests the full host+HM pipeline before hardware deploy | `home-manager build --flake .#<name>` (no VM) |
| **Use case** | You manage the OS; user homes are part of system | You don't have root; or you want faster home rebuilds |

**As NixOS module** is Den's default when you declare a user under `den.hosts`. The HM battery registers `hm-host` and `hm-user` derived contexts automatically:

```nix
den.hosts.x86_64-linux.igloo.users.alice = {
  classes = [ "homeManager" ];  # default, usually omitted
};

den.aspects.alice = {
  homeManager = { pkgs, ... }: {
    programs.git.enable = true;
  };
};
```

Den imports `home-manager.nixosModules.home-manager` into the NixOS config and forwards the `homeManager` class content to `home-manager.users.alice`. You never write `home-manager.users.alice = ...` manually.

**Standalone** is when you declare a home under `den.homes`:

```nix
den.homes.x86_64-linux.alice = {};

den.aspects.alice = {
  homeManager = { pkgs, ... }: {
    programs.git.enable = true;
  };
};
```

This produces `homeConfigurations.alice`. No NixOS module is imported; no host context exists. Functions requiring `{ host }` are never activated.

### Custom HM module per host

Override which HM module is used per host:

```nix
den.hosts.x86_64-linux.laptop.home-manager.module =
  inputs.home-manager-unstable.nixosModules.home-manager;
```

## hjem

hjem is a lightweight alternative to Home Manager for managing user files and dotfiles.

### Requirements

- `inputs.hjem` must exist
- User must have `"hjem"` in `classes`

### Enabling

```nix
# Per user
den.hosts.x86_64-linux.laptop.users.alice.classes = [ "hjem" ];

# For all users on all hosts
den.schema.host.hjem.enable = true;
```

### Configuration

```nix
den.aspects.alice = {
  hjem = {
    files.".config/foo/config".text = "setting = true";
  };
};
```

The `hjem` class content is forwarded to `hjem.users.<userName>` automatically.

## nix-maid

nix-maid is another user environment manager for NixOS only.

### Requirements

- `inputs.nix-maid` must exist
- Host class must be `"nixos"` — not available on Darwin
- User must have `"maid"` in `classes`

### Enabling

```nix
den.hosts.x86_64-linux.laptop.users.alice.classes = [ "maid" ];
```

### Configuration

```nix
den.aspects.alice = {
  maid = {
    # nix-maid configuration
  };
};
```

## Multiple environments simultaneously

A user can participate in multiple home environments at once:

```nix
den.hosts.x86_64-linux.laptop.users.alice = {
  classes = [ "homeManager" "hjem" ];
};
```

Both `homeManager` and `hjem` class content from `den.aspects.alice` are forwarded to their respective targets. The contexts activate in parallel — `hm-user` and `hjem-user` both fire.

## Standalone homes

For machines without root access, or to manage a home configuration independently from a host:

```nix
den.homes.x86_64-linux.alice = { };
```

Produces `homeConfigurations.alice`. Build and activate with the `home-manager` CLI.

Standalone homes go through a shorter pipeline — `den.ctx.home { home }` — with no host context. Functions requiring `{ host }` never activate. Functions requiring `{ home }` run instead.

### Host-bound standalone homes

Naming a home `"user@hostname"` makes the `home-manager` CLI auto-select it:

```nix
den.homes.x86_64-linux."alice@igloo" = { };
```

When `whoami == alice` and `hostname == igloo`, `home-manager` automatically picks `homeConfigurations."alice@igloo"`. The hostname does not need to be a `den.hosts` entry — it can be an externally managed machine.

Den automatically sets `home.userName` correctly so batteries like `define-user` work.

### Host-specific config in standalone homes

Use [[mutual-provider]] to push host-specific config into a standalone home:

```nix
den.ctx.user.includes = [ den._.mutual-provider ];

# Config only when on igloo
den.aspects.alice.provides.igloo = { host, ... }: {
  homeManager.programs.ssh.matchBlocks."igloo" = { ... };
};
```

### Separate host + user management

Declare both a host-managed user AND a standalone home for the same person:

```nix
den.hosts.x86_64-linux.igloo.users.alice = { };    # OS config
den.homes.x86_64-linux."alice@igloo"    = { };    # home config
```

Benefits:
- Host and home can be activated independently — faster rebuilds
- OS config changes don't force a home rebuild and vice versa
- Host config is still readable from the home via the `osConfig` module argument

## osConfig

In standalone home configurations, the `osConfig` module argument provides read-only access to the host's NixOS configuration. Available in `homeManager` modules when the home is built alongside the host.

```nix
den.aspects.alice.homeManager = { osConfig, ... }: {
  programs.ssh.matchBlocks."work" = {
    hostname = osConfig.networking.hostName;
  };
};
```

## Gotchas

- Home integrations are **opt-in** — Den does not enable any home environment by default. You must set `user.classes` explicitly or set a default via `den.schema.user`.
- The default value of `user.classes` is `[ "homeManager" ]` in the schema — but only if the HM battery is loaded AND `inputs.home-manager` exists. If HM is not in your inputs, the derived context never fires regardless of `classes`.
- `hjem` and `nix-maid` are not available on Darwin (`nix-maid` is NixOS only; check `hjem` docs for Darwin support status).
- `home.stateVersion` must be set — either globally via `den.default.homeManager.home.stateVersion` or per user in their aspect.
- `osConfig` is only available when the home is built as part of the host system (via `hm-user` context). In standalone home builds, `osConfig` is an empty module.

## In the Nix ecosystem

Home Manager is the dominant user environment manager in the Nix ecosystem. Den does not replace it — it integrates it as one [[nix-class]] among several. The same aspect can configure NixOS options AND Home Manager options in one place, making home environment config a natural extension of host config rather than a separate concern wired in manually.

hjem and nix-maid exist as lighter-weight alternatives. Den's multi-class user model means you can try multiple approaches simultaneously or migrate incrementally.

## Related pages

- [[aspect]] — home environment config lives in class-keyed owned configs on user aspects
- [[nix-class]] — homeManager, hjem, maid are Nix classes activated via user.classes
- [[context-transformation-pipeline]] — hm-host, hm-user, hjem-host, hjem-user, maid-host, maid-user are derived context stages
- [[den-ctx]] — the HM/hjem/maid batteries register context stages via den.ctx
- [[den-schema]] — user.classes controls which home environment contexts activate
- [[batteries]] — the HM, hjem, and maid batteries implement the integration
- [[parametric-dispatch]] — home context functions use {home} as the context shape; host+user functions use {host, user}
- [[vm-testing]] — boot host-bound configs in QEMU before hardware deployment