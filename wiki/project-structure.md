# Project Structure

**What it is**: The convention for organizing `.nix` files in a Den project so `import-tree` auto-loads everything and the mental model of concrete vs reusable aspects stays clear.

**Why it exists**: Den does not enforce a directory structure — `import-tree` recursively loads every `.nix` under `modules/` regardless of path. But without conventions, `den.aspects` and `den.ful` become a flat soup where hosts, users, and reusable features are indistinguishable.

**Relates to**: [[aspect]], [[namespaces]], [[den-schema]], [[import-tree]], [[den.hosts]], [[den.aspects]], [[den.default]], [[den-ctx]]

---

## The one rule

**Folders are for humans. Namespaces are for Den.**

Den auto-imports everything under `modules/`. The folder path does not affect evaluation — only the Nix expressions inside each file matter. You organize files into folders so *you* can find them. Den organizes data into `den.aspects` vs `den.ful.*` so *the module system* can route them correctly.

## How the namespace connects to the files

This is the most common confusion. The answer: **the namespace is declared in one file, populated by many files, and consumed by other files.** There is no directory-level connection. Den merges everything at evaluation time.

Here is exactly what happens:

```
modules/
  namespace.nix         ← says "create a bag called my"
  my/
    bluetooth.nix       ← drops my.bluetooth into the bag
    gaming.nix          ← drops my.gaming into the bag
  hosts/
    laptop.nix          ← reaches into the bag: includes = [ my.bluetooth ];
```

### Step by step

**1. `modules/namespace.nix` creates the empty bag and the module argument:**

```nix
{ inputs, ... }: {
  imports = [ (inputs.den.namespace "my" false) ];
}
```

After this file runs, two things exist in the module system:
- `den.ful.my` — an empty attrset waiting for aspects
- `my` — a module argument alias pointing to `den.ful.my`

Any module loaded after this one can write `my.anything = { ... };` and it gets merged into `den.ful.my.anything`.

**2. `modules/my/bluetooth.nix` drops an aspect into the bag:**

```nix
{
  my.bluetooth = {
    nixos.hardware.bluetooth.enable = true;
  };
}
```

This file does NOT import `namespace.nix` or reference `my/`. It simply writes `my.bluetooth`. Because `den.ful.my` already exists (from step 1), Nix merges this in.

You could move `bluetooth.nix` to `modules/hosts/bluetooth.nix` or `modules/zoo/bluetooth.nix` and it would produce the exact same result. The `my/` folder is for **you** to find it later.

**3. `modules/hosts/laptop.nix` reaches into the bag to use it:**

```nix
{ my, ... }: {
  den.aspects.laptop = {
    includes = [ my.bluetooth ];
  };
}
```

The `my` module argument is the bag created in step 1. This file is a *consumer* of the namespace.

### The directory tree shows nothing because there is nothing to show

Namespaces are **Nix module system values**, not files. You cannot see `den.ful.my` by looking at the directory tree any more than you can see `config.networking.hostName` by looking at files. The file layout is a human convenience. The namespace is an evaluation-time data structure.

**This is normal. This is recommended.** If you created a `my/` directory and tried to mirror the namespace there, you would be working against Den's design. One `namespace.nix` file. Any number of files contributing to it from any folders.

---

## The two buckets

Every value in a Den config falls into one of two buckets:

| Bucket | Purpose | Examples |
|---|---|---|
| **Concrete** (`den.aspects`) | Things that get built — hosts, users, standalone homes | `laptop`, `desktop`, `alice`, `bob` |
| **Reusable library** (`den.ful.<ns>`) | Features pulled into concrete things — never built alone | `bluetooth`, `gaming`, `dev-tools`, `vpn` |

Keep the two buckets separate in your head even though they both end up as `.nix` files. A laptop is a concrete machine; bluetooth support is a reusable feature the laptop *includes*.

---

## Recommended directory layout

When using namespaces, **folder names should match namespace names.** This makes the namespace visible in the file path.

```
modules/
  dendritic.nix          # flake-parts wiring, den.flakeModule, flake-file
  namespace.nix          # creates den.ful.my and den.ful.shared
  defaults.nix           # den.default — stateVersion, global batteries
  vm.nix                 # multi-host VM testing (see [[vm-testing]])

  # Host declarations (which concrete things exist)
  den.nix                # den.hosts.x86_64-linux.laptop.users.alice = {};

  # Concrete machines and users (den.aspects.*)
  hosts/
    laptop.nix           # den.aspects.laptop = { nixos = ...; includes = [ my.bluetooth ]; }
    desktop.nix          # den.aspects.desktop = { ... }
    nas.nix              # den.aspects.nas = { ... }
  users/
    alice.nix            # den.aspects.alice = { homeManager = ...; }
    bob.nix              # den.aspects.bob = { homeManager = ...; }

  # Reusable library — namespace "my" (den.ful.my)
  my/
    bluetooth.nix        # my.bluetooth = { nixos = ...; }
    gaming.nix           # my.gaming = { ...; includes = [ my.steam ]; }
    dev-tools.nix        # my.dev-tools = { ...; includes = [ my.git my.vim ]; }
    git.nix              # my.git = { homeManager.programs.git.enable = true; }
    vim.nix              # my.vim = { homeManager.programs.vim.enable = true; }

  # Reusable library — namespace "shared" (den.ful.shared), e.g. imported or cross-flake
  shared/
    vpn.nix              # shared.vpn = { ... }
    monitoring.nix       # shared.monitoring = { ... }
```

Now the path tells you the namespace: `modules/my/bluetooth.nix` → `my.bluetooth`. `modules/shared/vpn.nix` → `shared.vpn`. No guessing.

### What each file contains

**`modules/dendritic.nix`** — the flake wiring. Happens once, rarely touched after setup.

```nix
{ inputs, ... }: {
  imports = [
    inputs.den.flakeModule
    (inputs.flake-file.flakeModules.dendritic or {})
  ];
  flake-file.inputs = {
    den.url = "github:vic/den";
    home-manager.url = "github:nix-community/home-manager";
    home-manager.inputs.nixpkgs.follows = "nixpkgs";
  };
}
```

**`modules/namespace.nix`** — creates the `my` namespace. One line per namespace.

```nix
{ inputs, ... }: {
  imports = [ (inputs.den.namespace "my" false) ];
}
```

**`modules/defaults.nix`** — global defaults applied to every host, user, and home.

```nix
{ den, ... }: {
  den.default = {
    nixos.system.stateVersion = "25.11";
    homeManager.home.stateVersion = "25.11";
    includes = [
      den.provides.define-user
    ];
  };
}
```

**`modules/den.nix`** — declares which hosts and users exist. This is the "existence" layer; behavior lives in aspects.

```nix
{
  den.hosts.x86_64-linux.laptop.users.alice = {};
  den.hosts.x86_64-linux.desktop.users.alice = {};
  den.hosts.x86_64-linux.desktop.users.bob = {};
}
```

**`modules/hosts/laptop.nix`** — the concrete host aspect. References reusable features from the `my` namespace.

```nix
{ my, ... }: {
  den.aspects.laptop = {
    includes = [ my.bluetooth my.dev-tools ];
    nixos = { pkgs, ... }: {
      networking.hostName = "laptop";
      environment.systemPackages = [ pkgs.firefox ];
    };
  };
}
```

**`modules/users/alice.nix`** — the concrete user aspect.

```nix
{ den, ... }: {
  den.aspects.alice = {
    includes = [
      den.provides.primary-user
      (den.provides.user-shell "fish")
    ];
    homeManager = { pkgs, ... }: {
      home.packages = [ pkgs.htop ];
    };
  };
}
```

**`modules/my/bluetooth.nix`** — a reusable feature in the `my` namespace. The folder name `my/` matches the namespace name so the path itself shows the scope.

```nix
{
  my.bluetooth = {
    nixos.hardware.bluetooth.enable = true;
    homeManager.services.blueman-applet.enable = true;
  };
}
```

---

## Folders vs namespaces: the critical distinction

| | Folders | Namespaces |
|---|---|---|
| **What they organize** | Files on disk | Aspects in the Nix evaluation |
| **Who sees them** | You, the human | Den, the module system |
| **Effect on aspect names** | None — `den.aspects.laptop` is the same whether it lives in `hosts/laptop.nix` or `laptop.nix` | Namespaces change the key path — `my.bluetooth` vs `den.aspects.bluetooth` |
| **When to use** | Group related files for readability | Separate reusable library from concrete machines |

### Example: same config, two folder layouts

**Layout A — by role:**
```
modules/
  hosts/          # concrete machines
  users/          # concrete users  
  my/             # reusable aspects in namespace "my"
  shared/         # reusable aspects in namespace "shared"
```

**Layout B — by environment:**
```
modules/
  personal/       # laptop, alice, bob, bluetooth, gaming
  work/           # desktop, dev-tools, vpn
```

Both produce the exact same `den.aspects` and `den.ful.my`. Den does not know or care. Choose the layout that matches how *you* think about your config.

---

## Why `my`? Can I name the namespace `features`?

**You can name it anything.** Den does not care. But `my` is the convention because namespace names should answer **"whose?"** (scope/ownership), not **"what kind?"** (content type).

| Name | Reads as | Risk |
|---|---|---|
| `my` | "My personal library" | None — generic, stays correct forever |
| `features` | "Things that are features" | Drift — what if you later add non-feature things? What if you import someone else's features? |
| `infra` | "My infrastructure library" | Good if you plan to publish infra configs |
| `shared` | "Shared across my flakes" | Good if you consume this namespace in multiple flakes |

The `features/` folder says **"these files define features"** — a content-type label. The `den.ful.my` namespace says **"these are my aspects, not imported ones"** — an ownership label. They are orthogonal concerns.

### Why matching them feels wrong in practice

Suppose you name the namespace `features` because your folder is `features/`:

```nix
# modules/namespace.nix
{ inputs, ... }: {
  imports = [ (inputs.den.namespace "features" false) ];
}
```

Six months later you add work-specific VPN configs in `modules/work/` but you want them in the same namespace so `den.aspects.laptop` can pull from one place. Now `den.ful.features` contains VPN configs, and the name is a lie. Or you create `work/` namespace and now your laptop aspect references two namespaces:

```nix
den.aspects.laptop.includes = [ features.bluetooth work.vpn ];
```

Meanwhile with `my`:

```nix
# modules/features/bluetooth.nix — still in features/ folder
my.bluetooth = { ... };

# modules/work/vpn.nix — in work/ folder, same namespace
my.vpn = { ... };

# modules/hosts/laptop.nix — one namespace to pull from
den.aspects.laptop.includes = [ my.bluetooth my.vpn ];
```

The namespace is a **scope boundary** (my stuff), not a folder mirror. `my` is the convention because it is the shortest way to say "locally owned, not imported."

### The example template uses `eg` — and also has an `eg/` folder

The Den example template ships with `eg` as the namespace name and an `eg/` folder inside `modules/aspects/`:

```
modules/
  aspects/
    eg/               ← folder named "eg"
      autologin.nix   # eg.autologin
      routes.nix      # eg.routes
      vm.nix          # eg.vm
      xfce-desktop.nix # eg.xfce-desktop
```

This is a **teaching choice**, not a recommendation. The template authors named both the namespace and the folder `eg` so readers could instantly see the connection: *"open the `eg/` folder to find the `eg` namespace aspects."* It reduces cognitive load in a tutorial.

For real configs, this coupling is the semantic drift I described above. If you name the namespace after a folder, any aspect that belongs in the same namespace but lives elsewhere becomes confusing. The example template gets away with it because it is tiny and self-contained.

**Use the [[example-template]] as a learning reference, not as a structural blueprint.**

### Real-world naming

| Scenario | Namespace name | Why |
|---|---|---|
| Personal, one flake | `my` | Short, clear scope |
| Publishing infra configs | `infra` | Describes what others consume |
| Multiple flakes sharing one library | `shared` | Explicit cross-flake scope |
| Team/org namespace | `acme` (your org) | Consumer sees the source |

Use `eg` only if you are literally writing a tutorial.

---

### Why not just use a plain attrset like `features.bluetooth`?

You could write this:

```nix
# modules/features/bluetooth.nix
features.bluetooth = {
  nixos.hardware.bluetooth.enable = true;
};

# modules/hosts/laptop.nix
den.aspects.laptop = {
  includes = [ features.bluetooth ];
};
```

This works at the Nix level — `features.bluetooth` is a plain attrset, and `includes` can reference it. **But it is not a Den namespace, and you lose everything Den namespaces are designed for.**

| | `features.bluetooth` (plain attrset) | `my.bluetooth` (Den namespace) |
|---|---|---|
| **Aspect type** | Plain attrset — no `__functor`, no parametric dispatch, no aspect metadata | Proper aspect — gets `__functor` installed, parametric dispatch, `meta.name`, `meta.loc` |
| **Includes validation** | Works by accident — Den treats it as a static attrset | First-class citizen — validated as an aspect by the aspect type system |
| **Publishing** | Cannot be exported to other flakes | Exported as `flake.denful.my` automatically |
| **Importing** | Cannot consume upstream aspect libraries | Can merge `den.ful.shared` from another flake |
| **Angle brackets** | `<features/bluetooth>` does not work | `<my/bluetooth>` resolves correctly via `den.lib.__findFile` |
| **Community tooling** | Unknown convention — no ecosystem support | Standard Den pattern — documented, expected, supported |
| **Mental model** | "I invented a random attrset to group things" | "This is a Den library, separate from concrete machines" |

The namespace system is not just about grouping files. It is Den's **type system for reusable aspect libraries**. Using `features.bluetooth` is like using a plain Python dict instead of a dataclass — it holds data, but the surrounding tooling doesn't know what it means.

### When folders are enough (no namespace at all)

If you only want to separate files visually and do not plan to publish or import aspect libraries, skip namespaces entirely. Put reusable aspects directly in `den.aspects`:

```nix
# modules/features/bluetooth.nix
den.aspects.bluetooth = {
  nixos.hardware.bluetooth.enable = true;
};

# modules/hosts/laptop.nix
den.aspects.laptop = {
  includes = [ den.aspects.bluetooth ];
};
```

This is simpler, fully supported, and requires zero namespace boilerplate. The `features/` folder is purely for your editor. Use this until you need:
- Exporting aspects to other flakes
- Importing someone else's aspect library
- A strict separation between "concrete machines" (`den.aspects`) and "reusable library" (`den.ful.my`) |

Namespaces become necessary when you want the **semantic separation** enforced by Den's type system, not just folder organization.

---

## Common mistakes

### Mistake 1: Nesting aspects inside host files

```nix
# BAD — bluetooth is hidden inside laptop.nix, not reusable
# modules/hosts/laptop.nix
{
  den.aspects.laptop = {
    nixos.hardware.bluetooth.enable = true;  # buried inside a host
  };
}
```

```nix
# GOOD — bluetooth is a standalone aspect, laptop includes it
# modules/features/bluetooth.nix
den.aspects.bluetooth = {
  nixos.hardware.bluetooth.enable = true;
};

# modules/hosts/laptop.nix
den.aspects.laptop = {
  includes = [ den.aspects.bluetooth ];
};
```

### Mistake 2: Declaring hosts inside aspect files

```nix
# BAD — host declaration mixed with aspect behavior
# modules/hosts/laptop.nix
{
  den.hosts.x86_64-linux.laptop.users.alice = {};  # declaration
  den.aspects.laptop = { ... };                     # behavior
}
```

```nix
# GOOD — separation of concerns
# modules/den.nix
den.hosts.x86_64-linux.laptop.users.alice = {};

# modules/hosts/laptop.nix
den.aspects.laptop = { ... };
```

`den.hosts` is the **schema declaration** (what exists). `den.aspects.*` is the **behavior definition** (what it does). Keep them in separate files so any file can contribute to an aspect without knowing about host declarations.

### Mistake 3: Creating a namespace for every folder

```nix
# BAD — namespaces mirror folders pointlessly
# modules/hosts/namespace.nix
{ inputs, ... }: {
  imports = [ (inputs.den.namespace "hosts" false) ];
}
# modules/users/namespace.nix
{ inputs, ... }: {
  imports = [ (inputs.den.namespace "users" false) ];
}
```

Namespaces are not folders. One namespace called `my` (or `shared`, or `infra`) is usually enough for personal configs. Splitting into `hosts`/`users` namespaces adds indirection with no benefit.

---

## Multi-user, multi-host scaling

With 10 hosts and 20 users, `den.aspects` holds 30 entries — and that is fine. The filesystem keeps them organized:

```
modules/
  hosts/
    servers/
      nas.nix
      build.nix
      proxy.nix
    laptops/
      work.nix
      personal.nix
    desktops/
      gaming-rig.nix
  users/
    family/
      alice.nix
      bob.nix
    service/
      backup.nix
      ci.nix
```

Each `.nix` file defines one aspect (or a small related group). `import-tree` flattens them all at evaluation time. The folder nesting is purely for your editor and `git grep`.

---

## In the Nix ecosystem

Traditional NixOS configs organize by "layer" — `hardware/`, `networking/`, `users/`, `services/` — each directory contains NixOS modules for that layer. Den inverts this: organize by **concern** (aspect) instead of layer. A `bluetooth.nix` contains NixOS *and* Home Manager config for bluetooth in one file. The `hosts/` and `users/` directories are the *consumers* of those concerns, not the *owners*.

This is the dendritic (feature-first) pattern: features own cross-cutting config, hosts pick which features apply.

---

## Related pages

- [[aspect]] — what goes inside each `.nix` file
- [[namespaces]] — when and how to use `den.ful.*`
- [[den-schema]] — `den.hosts` and `den.homes` declarations
- [[den.default]] — global defaults applied to all entities
- [[vm-testing]] — VM runner configuration
- [[import-tree]] — the tool that recursively loads `modules/`
