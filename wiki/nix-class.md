# Nix Class

**What it is**: A named configuration domain in Nix — `nixos`, `darwin`, `homeManager`, `hjem`, `maid`, or any custom name — that identifies which module system a piece of config belongs to.

**Why it exists**: Different platforms and tools have separate module systems that cannot be mixed directly. Nix classes give Den a uniform way to label and route configuration to the right system without hardcoding platform assumptions.

**Relates to**: [[aspect]], [[owned-config]], [[resolution]], [[den.hosts]], [[den.homes]], [[nix-class]], [[context-transformation-pipeline]], [[den.lib.aspects.resolve]]

---

## Core idea

A Nix class is just a string name that Den uses as a key. When you write:

```nix
bluetooth = {
  nixos.hardware.bluetooth.enable = true;
  homeManager.services.blueman-applet.enable = true;
  darwin.networking.bluetooth.enable = true;
};
```

`nixos`, `homeManager`, and `darwin` are Nix classes. The values under each key are standard modules for that system — a NixOS module, a Home Manager module, a nix-darwin module.

**Nix classes are not invented by Den.** They exist independently in the wild. Den just uses them as a consistent labeling scheme to route config to the right place.

## Built-in classes

| Class | System | How it gets there |
|---|---|---|
| `nixos` | NixOS | Auto-detected from `x86_64-linux`, `aarch64-linux`, etc. |
| `darwin` | nix-darwin | Auto-detected from `aarch64-darwin`, `x86_64-darwin` |
| `homeManager` | Home Manager | Opt-in via `user.classes = [ "homeManager" ]` |
| `hjem` | hjem | Opt-in via `user.classes = [ "hjem" ]` |
| `maid` | nix-maid | Opt-in via `user.classes = [ "maid" ]` |
| `user` | Built-in NixOS/Darwin user | Always present; forwards to `users.users.<name>` |
| `system-manager` | system-manager | Used like nixos for non-NixOS Linux |
| `packages` | Flake outputs | Custom class for contributing to `flake.packages` |

## How Den uses classes

### In aspects — owned configs

Class names key the owned config attributes of an [[aspect]]:

```nix
den.aspects.laptop = {
  nixos.networking.hostName = "laptop";         # NixOS module
  darwin.nix-homebrew.enable = true;             # Darwin module
  homeManager.programs.starship.enable = true;   # HM module
};
```

Each value is a regular module for that system — attrset or function form. Den routes each to the correct module system during [[resolution]].

### In host declarations — host.class

Every host has a `class` field auto-derived from its platform:

```nix
den.hosts.x86_64-linux.laptop = { ... };   # class = "nixos"
den.hosts.aarch64-darwin.mac  = { ... };   # class = "darwin"
```

This determines which `instantiate` function is used (`lib.nixosSystem` vs `darwin.lib.darwinSystem`) and which class key is extracted from aspects during resolution.

### In user declarations — user.classes

Users declare which home environment classes they participate in:

```nix
den.hosts.x86_64-linux.laptop.users.alice = {
  classes = [ "homeManager" "hjem" ];
};
```

Each class in this list activates the corresponding derived context stage in the [[context-transformation-pipeline]] — `hm-user`, `hjem-user`, etc.

### In resolution

`den.lib.aspects.resolve "nixos" aspect` extracts only the `nixos`-keyed owned configs from the aspect and its entire `includes` DAG, merges them, and returns a single `deferredModule` for that class. See [[resolution]].

## Custom classes

Den does not restrict you to the built-in class names. Any string can be a class:

```nix
# Aspect contributing to a custom "terranix" class
den.aspects.web-server = den.lib.parametric {
  terranix.resource.aws_instance.web = { ami = "..."; };
};

# Resolve for the custom class
module = den.lib.aspects.resolve "terranix" [] aspect;
```

This is the basis of [[den-as-library]] usage — Den's aspect and dispatch machinery works for any Nix module domain, not just NixOS/Darwin.

The `packages` class works the same way — aspects can produce `flake.packages` entries by using `packages` as a class key.

## Gotchas

- `host.class` (singular) is the OS class — `"nixos"` or `"darwin"`. `user.classes` (plural) is the list of home environment classes. They are different fields.
- Class names are case-sensitive strings — `homeManager` not `home-manager` or `HomeManager`.
- A class key on an aspect is treated as an owned config only if it does not look like a module (`imports`, `config`, `options` top-level) — in that case Den interprets the attribute as a submodule definition. Use function-args style `{ config, ... }:` to avoid this ambiguity.
- Adding a new class to `user.classes` activates a new derived context — but only if a matching battery or context type is registered. Unrecognized class names are ignored by the pipeline.

## In the Nix ecosystem

The concept of classes pre-dates Den. NixOS modules, Darwin modules, and Home Manager modules are all separate module systems that happen to use the same `lib.evalModules` machinery. The `nixosModules`, `darwinModules`, and `homeManagerModules` flake output types reflect this natural separation.

Den's contribution is treating these as rows in a matrix rather than separate trees — an [[aspect]] owns one row, containing all class-specific config for one feature.

## Related pages

- [[aspect]] — aspects are keyed by class names for their owned configs
- [[owned-config]] — class-keyed attributes on an aspect; always included in resolution
- [[resolution]] — extracts one class's config from an aspect's full DAG
- [[den.hosts]] — host.class is auto-derived from the host's platform
- [[context-transformation-pipeline]] — user.classes controls which derived contexts activate
- [[den-as-library]] — custom classes enable Den outside NixOS/Darwin
- [[den.lib.aspects.resolve]] — the function that resolves an aspect for a specific class