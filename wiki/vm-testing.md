# VM Testing

**What it is**: The practice of booting a Den host configuration inside a QEMU VM before deploying it to physical hardware, using NixOS's built-in `system.build.vm` output.

**Why it exists**: Den's context pipeline produces complex cross-cutting config (host → users → home-manager). A VM lets you verify the entire pipeline — boot, services, user login, HM activation — without risking a broken hardware deployment.

**Relates to**: [[context-transformation-pipeline]], [[den-schema]], [[batteries]], [[home-environments]], [[nix-class]], [[resolution]], [[host.mainModule]], [[host.instantiate]], [[den.hosts]], [[example-template]]

---

## Core idea

Every NixOS configuration automatically gets a `system.build.vm` attribute: a QEMU disk image and a `run-<hostname>-vm` launch script. Den host configurations are ordinary NixOS configurations, so they inherit this for free.

The default Den template exposes this via a flake package:

```nix
# vm.nix (single host — what the default template ships)

# Module arguments from the Den framework flake module:
#   inputs — your flake inputs (including self)
#   den    — the Den module argument (provides den.provides.*)
{ inputs, den, ... }:

{
  # Add the tty-autologin battery to the igloo host aspect.
  # `den.provides.tty-autologin "tux"` is a battery that sets
  #   services.getty.autologinUser = "tux";
  # inside the host's NixOS config, so the VM boots straight to the
  # user shell instead of stopping at a login prompt.
  den.aspects.igloo.includes = [ (den.provides.tty-autologin "tux") ];

  # `perSystem` is flake-parts' way of declaring per-system outputs.
  # `pkgs` is the nixpkgs package set for the current system.
  perSystem = { pkgs, ... }: {
    # Declare a flake package named `vm` (so `nix run .#vm` works).
    packages.vm = pkgs.writeShellApplication {
      # The derivation name; appears in `nix build` output paths.
      name = "vm";

      # `text` is the shell script body that runs when you type `nix run .#vm`.
      text = let
        # Reach into the already-evaluated NixOS configuration for the
        # host named "igloo". `inputs.self` is your flake output.
        # `.config` is the fully-merged NixOS module evaluation result.
        host = inputs.self.nixosConfigurations.igloo.config;
      in ''
        # `host.system.build.vm` is the NixOS-internal QEMU wrapper.
        # It contains a script named `run-igloo-vm` (derived from
        # `host.networking.hostName`, which defaults to "igloo").
        # "$@" forwards extra CLI flags (e.g. `-m 4096`) into QEMU.
        ${host.system.build.vm}/bin/run-${host.networking.hostName}-vm "$@"
      '';
    };
  };
}
```

Run it:

```bash
nix run .#vm
```

The VM boots the **exact** same `deferredModule` that would be switched on the real machine. If it boots and your user logs in, the aspect DAG resolved correctly.

---

## Multi-host setup

The default template hardcodes one host. For a fleet, generate a `.#vm-<name>` per `den.hosts` entry:

```nix
# vm.nix (multi-host — drop-in replacement)

# 1. Module arguments: inputs gives us `self` (our flake), `config` is the
#    fully-merged flake-parts module config, `lib` is nixpkgs lib.
{ inputs, config, lib, ... }:

{
  # ── Part A: Inject tty-autologin into every user on every host ──
  # We want the VM to boot straight into the user shell without a login
  # prompt. The tty-autologin battery is NixOS-only, so it is safe for
  # all hosts that are NixOS (we filter later).
  den.hosts =
    # `lib.mapAttrs` walks `config.den.hosts`, which is keyed by system
    # then hostName. Example shape:
    #   den.hosts.x86_64-linux.laptop.users.alice = {};
    lib.mapAttrs (system: hosts:
      # Inner walk: for each host (e.g. "laptop") under this system.
      lib.mapAttrs (hostName: host: {
        # `host.users` is the attrset of users declared on this host.
        # We inject `tty-autologin` into every user's `aspect.includes`.
        users = lib.mapAttrs (userName: user: {
          aspect.includes = [
            # `tty-autologin "alice"` writes
            #   services.getty.autologinUser = "alice";
            # into the host's NixOS module. This is only evaluated because
            # the user is part of a NixOS host (the filter in Part B
            # guarantees we only build NixOS VMs).
            (inputs.den.legacyPackages.default.provides.tty-autologin userName)
          ];
        }) host.users;
      }) hosts
    ) config.den.hosts;

  # ── Part B: Generate one flake package per host that boots it in QEMU ──
  perSystem = { pkgs, system, ... }: {
    packages =
      # `lib.mapAttrs'` is like `mapAttrs` but lets us control the key
      # name directly. We want package names like `.#vm-laptop`.
      lib.mapAttrs' (hostName: host: {
        # `.#vm-laptop` — the name you type after `nix run`.
        name = "vm-${hostName}";

        value = pkgs.writeShellApplication {
          # Internal name for the derivation; shows up in build logs.
          name = "vm-${hostName}";

          # `text` is the shell script body that runs when you type
          # `nix run .#vm-laptop`.
          text = let
            # `inputs.self.nixosConfigurations.laptop` is the *exact*
            # `nixosConfiguration` Den built for the real host.
            # `.config` reaches into the evaluated NixOS module system.
            hostCfg = inputs.self.nixosConfigurations.${hostName}.config;
          in ''
            # `hostCfg.system.build.vm` is a NixOS-internal path that
            # contains the QEMU disk image + a `run-<hostname>-vm` script.
            # This script is auto-generated by `nixos/modules/virtualisation/qemu-vm.nix`.
            # `"$@"` forwards extra flags (e.g. `-m 4096`) from the CLI.
            ${hostCfg.system.build.vm}/bin/run-${hostCfg.networking.hostName}-vm "$@"
          '';
        };
        # Only generate VM packages for hosts under the *current* system.
        # `config.den.hosts.${system}` is an attrset of hosts for this arch.
        # If no hosts are declared for this system, `or {}` returns an empty
        # attrset so `mapAttrs'` safely does nothing.
      }) config.den.hosts.${system} or {};
  };
}
```

Usage:

```bash
nix run .#vm-laptop      # test laptop config
# ...verify in VM...
nixos-rebuild switch --flake .#laptop

nix run .#vm-workstation # test workstation config
# ...verify in VM...
nixos-rebuild switch --flake .#workstation --target-host workstation
```

Each VM is an independent QEMU process. You can keep one open while editing another host, or run them side-by-side if your machine has the RAM.

---

## Why this works with Den

Standard NixOS VM testing means writing a separate `configuration.nix` for the VM. With Den you don't need a separate file because:

- The same `den.hosts.<name>` declaration produces both `nixosConfigurations.<name>` and the VM package.
- The same [[resolution]] pipeline runs — same [[aspect-dag]], same [[parametric-dispatch]], same [[dedupIncludes]].
- The same [[context-transformation-pipeline]] fires: host context → user contexts → HM contexts.

If an aspect fails to resolve, the VM build fails with the same error the real deployment would hit. If HM user services conflict, you see it in the VM boot log.

---

## Workflow: edit → VM → deploy → next host

| Step | Action | Command |
|---|---|---|
| 1 | Edit aspects | `nvim modules/workstation.nix` |
| 2 | Build VM | `nix run .#vm-workstation` |
| 3 | Verify inside VM | Check services, open terminal, test HM programs |
| 4 | Deploy to hardware | `nixos-rebuild switch --flake .#workstation --target-host ...` |
| 5 | Rotate to next host | `nix run .#vm-laptop` |

You can run steps 2–4 in parallel for unrelated hosts if your builder has enough cores. The VM is just another Nix derivation; it caches like everything else.

---

## Batteries useful for VM testing

- **`den.provides.tty-autologin <user>`** — skips the login prompt in the VM so you land straight in the user shell. NixOS only.
- **`den.provides.define-user`** — already creates the OS account; combined with tty-autologin you get instant feedback on the user environment.
- **`den.provides.hostname`** — ensures `networking.hostName` matches the host name; avoids mismatched `run-*-vm` script names.

Avoid hardware-only aspects in the VM target if they break the QEMU environment (e.g. NVIDIA drivers, bluetooth firmware, custom kernel patches). Use [[parametric-dispatch]] to gate them:

```nix
{ host, ... }: lib.mkIf (host ? hardware.gpu) {
  nixos.services.xserver.videoDrivers = [ "nvidia" ];
}
```

Since the VM context lacks `hardware.gpu`, the driver config is silently skipped.

---

## Gotchas

- **`networking.hostName` must match the flake output name** for the `run-<hostname>-vm` script to exist. If you override `hostName` in the host aspect, use that value in the VM wrapper, not the attrset key.
- **Standalone homes (`den.homes`) cannot be tested with `system.build.vm`**. They produce `homeConfigurations`, not NixOS systems. Test those with `home-manager build --flake .#user@host`.
- **First VM boot can be slow** if the closure is large. Subsequent runs are cached. Use `nix run .#vm-laptop -- -m 4096` to give QEMU more RAM.
- **VMs share the host nix store** via 9P. This is fast but means the VM sees the same store path layout. It is not an isolated install test.
- **SSH host keys are regenerated** in the VM. If your config references `host.hostKeys` or mutual-provider host-key forwarding, the VM will have different keys than the real machine.
- **The multi-host `mapAttrs` snippet assumes all `den.hosts` entries are NixOS**. Darwin hosts do not have `system.build.vm`. Filter by `host.class == "nixos"` before generating packages.

---

## In the Nix ecosystem

NixOS's `system.build.vm` is a standard feature of `nixos/modules/virtualisation/qemu-vm.nix`. It is not Den-specific. What Den adds is the guarantee that the VM runs the **identical** aspect-resolution result as the real host — no manual module list duplication, no "test config" divergence.

In traditional NixOS setups, users maintain a `vm.nix` that imports a subset of modules. In Den, the VM is a view onto the same host declaration, so the test surface is the full pipeline.

---

## Related pages

- [[context-transformation-pipeline]] — the pipeline the VM exercises end-to-end
- [[den-schema]] — where `den.hosts` is declared
- [[batteries]] — `tty-autologin`, `hostname`, `define-user`
- [[home-environments]] — standalone home testing (different path)
- [[resolution]] — how the VM's module is produced
- [[host.mainModule]] — the internally computed module passed to the builder
