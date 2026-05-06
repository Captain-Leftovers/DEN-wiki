# mutual-provider

**What it is**: A Den battery (`den._.mutual-provider`) that enables bidirectional configuration between hosts and users via `.provides.` sub-aspects, allowing a user aspect to push OS-level config into its host and a host aspect to push home-level config into its users.

**Why it exists**: Without it, Den's pipeline is unidirectional — user aspects contribute `nixos` class config upward to the host, but hosts cannot push `homeManager` config into users, and users cannot target specific hosts by name. Mutual-provider makes cross-entity relationships explicit and opt-in.

**Relates to**: [[batteries]], [[den-ctx]], [[context-transformation-pipeline]], [[aspect]], [[home-environments]], [[example-template]]

---

## Core idea

Den's default pipeline flows one way: host context → user contexts → merged back into the host module. User aspects return `nixos.*` config that becomes part of the host's final system configuration. But there is no mechanism for:

- A **host** to say "enable Helix in Alice's home"
- A **user** to say "enable nh system-wide, but only on my laptop"
- A **user** to configure another user on the same machine

`mutual-provider` solves this by scanning `.provides.*` entries on aspects during the `{ host, user }` context stage. It adds extra providers to `den.ctx.user` that look for matching names and return the nested config with correct class forwarding.

The battery is **not automatic** — you must explicitly include it. This keeps evaluations that don't need cross-entity wiring fast and predictable.

---

## How to enable it

### Global — all users on all hosts

```nix
den.ctx.user.includes = [ den._.mutual-provider ];
```

This is the standard placement. It activates mutual-provider for every `{ host, user }` context the pipeline ever creates. Every user aspect can now use `.provides.` to target hosts or other users.

### Global — standalone homes only

```nix
den.ctx.home.includes = [ den._.mutual-provider ];
```

Use this when you manage standalone `den.homes` entries and want host-specific home config without a full NixOS host.

### Scoped — not recommended

Putting `den._.mutual-provider` inside an individual aspect's `includes` (e.g. `den.aspects.alice.includes`) is technically possible but not the supported pattern. The battery registers context-stage providers; `den.ctx.user.includes` is the correct hook.

---

## The four direction patterns

Once enabled, `.provides.` attributes on any aspect become active. There are four patterns covering every host↔user direction.

### 1. User → specific host

User `tux` wants a system-wide program **only** on host `igloo`:

```nix
den.aspects.tux = {
  # Normal config — applies everywhere tux exists
  homeManager.programs.vim.enable = true;

  # ONLY on host "igloo": enable nh system-wide
  provides.igloo = {
    nixos.programs.nh.enable = true;
  };
};
```

During the user context for `tux@igloo`, the mutual-provider scans `tux.provides.igloo`, sees the name matches the current `host.name`, and forwards the `nixos` class into the host module.

### 2. Host → specific user

Host `igloo` wants to force a Home Manager setting on user `alice` only:

```nix
den.aspects.igloo = {
  # Normal host config
  nixos.services.openssh.enable = true;

  # ONLY for user "alice": enable helix in her home
  provides.alice = {
    homeManager.programs.helix.enable = true;
  };
};
```

During the user context for `alice@igloo`, the mutual-provider scans `igloo.provides.alice`, sees `user.name == "alice"`, and forwards the `homeManager` class into alice's home config.

### 3. User → all hosts (conditional)

User `tux` wants to conditionally contribute to whichever host they are on:

```nix
den.aspects.tux = {
  provides.to-hosts = { host, ... }: {
    nixos.programs.nh.enable = host.name == "igloo";
  };
};
```

`to-hosts` is a reserved key. It receives the current `{ host, user }` context and can use `host.name`, `host.system`, or any schema field to decide what to return. The result is forwarded as `nixos` class config.

### 4. Host → all users (conditional)

Host `igloo` wants to conditionally contribute to all its users:

```nix
den.aspects.igloo = {
  provides.to-users = { user, ... }: {
    homeManager.programs.tmux.enable = lib.elem user.userName [ "alice" "bob" ];
  };
};
```

`to-users` is also reserved. It receives each user's context and can filter by `user.name`, `user.userName`, etc. It **excludes** the source user when the source is a user aspect (alice won't receive her own `to-users`), but includes all users when the source is a host aspect.

---

## Complete concrete example

```nix
{
  den.hosts.x86_64-linux.laptop.users.alice = {};
  den.hosts.x86_64-linux.laptop.users.bob = {};

  # Enable mutual-provider globally
  den.ctx.user.includes = [ den._.mutual-provider ];

  # --- ALICE'S ASPECT ---
  den.aspects.alice = {
    homeManager.programs.git.enable = true;

    # Push nh into laptop only
    provides.laptop = {
      nixos.programs.nh.enable = true;
    };

    # Give Bob vim on any shared host
    provides.bob = {
      homeManager.programs.vim.enable = true;
    };
  };

  # --- BOB'S ASPECT ---
  den.aspects.bob = {
    homeManager.programs.helix.enable = true;

    # Push syncthing into laptop only
    provides.to-hosts = { host, ... }: lib.optionalAttrs (host.name == "laptop") {
      nixos.services.syncthing.enable = true;
    };
  };

  # --- LAPTOP'S ASPECT ---
  den.aspects.laptop = {
    nixos.boot.loader.systemd-boot.enable = true;

    # Force kitty on Alice only
    provides.alice = {
      homeManager.programs.kitty.enable = true;
    };

    # Give zsh to all users on this host
    provides.to-users = { user, ... }: {
      homeManager.programs.zsh.enable = true;
    };
  };
}
```

**Result when building `laptop`**:

| Entity | Normal config | Mutual config received |
|--------|--------------|------------------------|
| `laptop` (host) | `systemd-boot` | `nh` (from alice), `syncthing` (from bob) |
| `alice` (user) | `git` | `kitty` (from laptop), `zsh` (from laptop), `vim` (from alice→bob, no — alice does not receive her own peer config) |
| `bob` (user) | `helix` | `vim` (from alice), `zsh` (from laptop) |

---

## User peers: one user configuring another

Because `.provides.` is just named sub-aspects, you can target other users on the same machine:

```nix
den.aspects.alice = {
  # Give bob vim on any shared host
  provides.bob = {
    homeManager.programs.vim.enable = true;
  };

  # Give carl and david tmux on any shared host
  provides.to-users = { user, ... }: {
    homeManager.programs.tmux.enable = lib.elem user.userName [ "carl" "david" ];
  };
};
```

`to-users` excludes the source user. Alice will not receive `tmux` from her own `to-users`, but `bob`, `carl`, and `david` will if they exist on the same host.

---

## Standalone homes (non-NixOS hosts)

When you manage standalone Home Manager configs with `den.homes`, mutual-provider works there too:

```nix
den.ctx.home.includes = [ den._.mutual-provider ];

den.homes.x86_64-linux."tux@igloo" = {};
den.homes.x86_64-linux."tux@iceberg" = {};

den.aspects.tux = {
  # Applies to tux on BOTH homes
  homeManager.programs.vim.enable = true;

  # ONLY on the igloo home
  provides.igloo = {
    homeManager.programs.helix.enable = true;
  };

  # ONLY on the iceberg home
  provides.iceberg = {
    homeManager.programs.emacs.enable = true;
  };
};
```

This is useful when `igloo` and `iceberg` are not NixOS machines you control — you get host-specific home config without a full host declaration.

---

## Under the hood: how it hooks into the pipeline

Without mutual-provider, the user context stage runs:

```
user context → _.user (the user's own aspect) → back to host
```

With `den.ctx.user.includes = [ den._.mutual-provider ]`, the battery contributes additional provider functions to `den.ctx.user._`:

```
user context
  ├─→ _.user                         (user's own aspect — always present)
  ├─→ _.mutual-from-host            (scans host.aspect.provides.<this-user>)
  ├─→ _.mutual-from-user-to-host    (scans user.aspect.provides.<this-host>)
  ├─→ _.mutual-from-user-to-users    (scans user.aspect.provides.to-users)
  ├─→ _.mutual-from-user-to-hosts    (scans user.aspect.provides.to-hosts)
  └─→ _.mutual-from-host-to-users   (scans host.aspect.provides.to-users)
       → back to host
```

Each provider receives `{ host, user }`, checks whether the aspect has a matching `.provides.` entry, and if so returns the nested config with class forwarding applied (`nixos` → host module, `homeManager` → user's home).

**Without the include**, these provider functions are never added, so `.provides.` attributes are silently ignored.

---

## Gotchas

- **Must be included to have any effect**: Writing `provides.igloo.nixos.programs.nh.enable = true` without `mutual-provider` in `den.ctx.user.includes` is silently ignored. There is no error.
- **Global placement is the idiomatic pattern**: Put it in `den.ctx.user.includes` (and/or `den.ctx.home.includes`) once. Scoping it to individual aspect `includes` is not the supported pattern because the battery registers context-stage providers.
- **Class forwarding still applies**: If you put `homeManager.programs.helix` inside `provides.igloo.nixos`, it will not work. The class name (`nixos`, `darwin`, `homeManager`) must match the target context.
- **`.provides.` is not special syntax**: It is just a nested attribute path on an aspect. `den.aspects.tux.provides.igloo` is a sub-aspect named `provides.igloo`. The mutual-provider battery knows to look for these names during the user context stage.
- **No circular dependency risk**: The pipeline resolves `.provides.` before `lib.evalModules` runs, so mutual contributions are collected statically. There is no infinite loop between host and user configs.

---

## In the Nix ecosystem

Standard NixOS + Home Manager setups have no equivalent concept. If you want host-specific home config, you manually thread `specialArgs` or use `imports` with conditionals. If you want a user to enable a system-wide package only on one machine, you duplicate the host config or use `mkIf` with a hostname check inside the host module.

Den's mutual-provider makes these relationships **declarative and local to the entity that owns the concern**: Alice says "I want nh on my laptop" inside `alice`'s aspect; the laptop says "I want kitty for Alice" inside `laptop`'s aspect. Neither needs to know about the other's internal structure.
