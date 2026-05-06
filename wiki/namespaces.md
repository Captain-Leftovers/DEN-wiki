# Namespaces

**What it is**: A scoped aspect library under `den.ful.<name>` — a named bucket of reusable [[aspect]]s, separate from `den.aspects`.
**Why it exists**: To organize reusable features into named groups, publish aspect collections for other flakes, and import community/team aspect libraries — without name collisions.
**Relates to**: [[aspect]], [[den.aspects]], [[context-transformation-pipeline]], [[batteries]], [[dendrix]], [[denful]], [[angle-bracket-syntax]], [[flake.denful]], [[example-template]]

---

## The problem namespaces solve

Without namespaces, every aspect lives in one flat bucket: `den.aspects`. That bucket holds both your **concrete machines/users** (things that get built) and your **reusable features** (things machines pull from). This works fine at small scale, but breaks down when:

1. You want to **organize** reusable features separately from machines — "bluetooth support" is a different kind of thing than "my laptop".
2. You want to **publish** a set of aspects so another flake can pull them in.
3. You want to **consume** someone else's published aspect collection.

Namespaces solve all three by creating additional named buckets alongside `den.aspects`.

## Core idea

A namespace is just **a second (or third, or fourth) bag of aspects with a name**:

- `den.aspects` — the default bag (no namespace needed)
- `den.ful.my` — a namespace called `my`
- `den.ful.team` — a namespace called `team`

Each bag works exactly like `den.aspects` — same type, same rules. The only difference is it has a name and lives under `den.ful`.

When you create a namespace called `eg`, you get:

- `den.ful.eg` — the aspects attrset for that namespace
- `eg` — a module argument alias pointing to `den.ful.eg`
- `flake.denful.eg` — a flake output for publishing (only if exported)

## Step-by-step example

### 1. Create the namespace

Create `modules/namespace.nix`:

```nix
{ inputs, ... }: {
  imports = [ (inputs.den.namespace "my" false) ];
}
```

One line. This creates `den.ful.my` (an empty bag) and makes `my` available as a module argument in every module. `false` means local-only — not exposed to other flakes.

### 2. Put aspects in the namespace

Instead of writing `den.aspects.foo`, write `my.foo`:

```nix
# modules/aspects/my/vim.nix
{
  my.vim = {
    homeManager.programs.vim = {
      enable = true;
      defaultEditor = true;
    };
  };
}
```

```nix
# modules/aspects/my/git.nix
{
  my.git = {
    homeManager.programs.git = {
      enable = true;
      userName = "me";
      userEmail = "me@example.com";
    };
  };
}
```

```nix
# modules/aspects/my/dev-tools.nix
{ my, ... }: {
  my.dev-tools = {
    includes = [ my.vim my.git ];
    homeManager.home.packages = [ /* extra stuff */ ];
  };
}
```

Note `dev-tools` uses `{ my, ... }:` to receive the namespace as a function argument, then refers to `my.vim` and `my.git` — the same way you'd refer to `den.aspects.vim` in the default bag.

### 3. Use namespaced aspects in your hosts

Your actual host/machine aspects still live in `den.aspects`. They pull from the namespace via [[includes]]:

```nix
# modules/hosts.nix
{ my, ... }: {
  den.aspects.laptop = {
    includes = [ my.dev-tools ];
    nixos.networking.hostName = "laptop";
  };
}
```

That is the whole loop. The namespace `my` is a **library of reusable parts**. `den.aspects` holds the **concrete machines** that pick from that library.

## Mental model

```
den.aspects           = your actual machines and users (what gets built)
den.ful.<name>        = libraries of reusable parts (what you pick from)
namespace             = the name on a library (my, team, gaming, etc.)
```

Visually:

```
┌───────────────────────────────────────────────┐
│                  Your Flake                   │
│                                               │
│  den.aspects            (concrete machines)   │
│    laptop                 │                   │
│    desktop                │                   │
│    nas                    │ includes          │
│                           ▼                   │
│  den.ful.my             (reusable library)    │
│    vim                                        │
│    git                                        │
│    dev-tools ──includes──▶ vim, git           │
│    bluetooth                                  │
│                                               │
│  den.ful.team           (imported library)    │
│    ┌── from team-config flake                 │
│    │   vpn                                    │
│    │   monitoring                             │
│    └── your local additions                   │
│        custom-alerts                          │
└───────────────────────────────────────────────┘
```

## "But won't den.aspects get crowded?"

If you have 10 machines and 20 users, all 30 live in `den.aspects` — and that is fine. `den.aspects` is flat in the **Nix module system**, not in your filesystem. You can organize the files however you want:

```
modules/
  aspects/
    servers/
      nas.nix              → den.aspects.nas
      media-server.nix     → den.aspects.media-server
      build-server.nix     → den.aspects.build-server
    laptops/
      work.nix             → den.aspects.work
      personal.nix         → den.aspects.personal
    family/
      mom.nix              → den.aspects.mom
      dad.nix              → den.aspects.dad
```

Den auto-imports everything under `modules/`. The folder structure is for **you** — Den does not care about it. The aspect name comes from what you write inside the file (`den.aspects.nas = { ... }`), not from the folder path.

Namespaces are not for splitting machines into groups. Folders do that. Namespaces are for splitting **reusable features** away from concrete machines, and for **sharing across flakes**.

## Three namespace modes

### Local namespace

Created with `false` — not exported to flake outputs. Internal use only.

```nix
{ inputs, ... }: {
  imports = [ (inputs.den.namespace "my" false) ];
}
```

Use this to organize your own reusable aspects into a named scope without publishing them.

### Exported namespace

Created with `true` — exposed as `flake.denful.<name>`. Downstream flakes can import it.

```nix
{ inputs, ... }: {
  imports = [ (inputs.den.namespace "eg" true) ];
}
```

This sets `flake.denful.eg` automatically. Other flakes that point at yours can merge this namespace into their own `den.ful.eg`.

### Imported namespace

Merges aspects from one or more upstream flakes into your local `den.ful.<name>`:

```nix
{ inputs, ... }: {
  imports = [ (inputs.den.namespace "shared" [ inputs.team-config ]) ];
}
```

Instead of `true`/`false`, you pass **a list of flake inputs**. Den looks at each input's `flake.denful.shared` and merges those aspects into your local `den.ful.shared`. You can then use upstream aspects and add your own — local definitions merge on top:

```nix
{ shared, ... }: {
  # Use upstream aspects
  den.aspects.laptop.includes = [ shared.vpn shared.monitoring ];

  # Add your own to the same namespace
  shared.custom-alerts = {
    nixos.services.prometheus.alertmanager.enable = true;
  };
}
```

## Architecture

```
upstream flake  ──→  flake.denful.shared
local modules   ──→  den.ful.shared  ←──  merged result
                          ↓
              den.aspects.*.includes
```

## Angle bracket syntax

Optional shorthand. When namespaces are enabled, you can use `<namespace/path>` for deep aspect paths. Enable it by setting `__findFile`:

```nix
{ den, ... }: {
  _module.args.__findFile = den.lib.__findFile;
}
```

Then reference aspects with angle brackets:

```nix
den.aspects.laptop.includes = [ <my/dev-tools> ];
# equivalent to:
den.aspects.laptop.includes = [ den.ful.my.dev-tools ];
```

The `/` separator navigates the attrset path. This is purely syntactic sugar — it resolves to the same value at evaluation time. See [[angle-bracket-syntax]] for full resolution rules.

## Do you even need namespaces?

**Maybe not.** If you are one person, not sharing configs with anyone, you can put reusable features directly in `den.aspects`:

```nix
den.aspects.bluetooth = {
  nixos.hardware.bluetooth.enable = true;
};

den.aspects.laptop = {
  includes = [ den.aspects.bluetooth ];
};
```

Namespaces become worthwhile when:

1. You want a **clean separation** between "things that get built" and "reusable building blocks" — so `den.aspects` only has real hosts/users and `den.ful.my` has the library.
2. You want to **export** aspects for another flake to consume.
3. You want to **import** someone else's aspect library.

If none of those apply, skip namespaces. You can always add one later without restructuring — move `den.aspects.bluetooth` to `my.bluetooth` and update the references.

## Quick reference

| You write | What it means |
|---|---|
| `inputs.den.namespace "foo" false` | Create local namespace `foo` |
| `inputs.den.namespace "foo" true` | Create + export namespace `foo` |
| `inputs.den.namespace "foo" [ inputs.x ]` | Import `foo` from flake `x`, merge with local |
| `foo.bar = { ... }` | Define aspect `bar` in namespace `foo` |
| `{ foo, ... }: foo.bar` | Access aspect `bar` from namespace `foo` |
| `<foo/bar>` | Angle-bracket shorthand for `den.ful.foo.bar` |

## Relationship to dendrix

[[dendrix]] is the community-level expression of the namespace model. It is a browsable index of aspects published by people on GitHub, using `import-tree` collections rather than flakes. This means:

- Consumers don't have to download all of a publisher's flake inputs
- Aspects declare only the inputs they need via `flake-file`
- Those inputs propagate to the consumer's flake or npins setup

The namespace system in Den is the local mechanism. Dendrix is the community discovery layer built on top of it.

## Relationship to denful

[[denful]] is a planned LazyVim-style configuration distribution built on Den's namespace system. Users would include from it the way editor users include from editor distributions — pulling in curated, tested, shareable aspect collections rather than writing everything from scratch.

## Gotchas

- The `den.namespace` function returns a **module** — it must be added to `imports`, not to `includes`
- The namespace module arg (`eg`) is only available in modules loaded after the namespace is created — use `imports = [ (inputs.den.namespace ...) ]` early in your module tree
- Merging upstream namespaces is additive — if both local and upstream define `shared.vim`, the results are merged like any NixOS module option (last write wins for non-list types)
- [[angle-bracket-syntax]] requires `_module.args.__findFile = den.lib.__findFile` — forgetting this makes `<eg/desktop>` resolve via the normal NIX_PATH lookup, not Den's namespace, which will fail or return wrong results
- `flake.denful` is only set when `den.namespace "name" true` is used — local-only namespaces (`false`) do not appear in flake outputs

## In the Nix ecosystem

The Nix ecosystem has historically lacked a good module sharing story. NixOS modules in flakes can be imported but carry all their flake inputs. The NUR (Nix User Repository) provides packages but not configuration. Den's namespace system is the first attempt at a typed, composable configuration sharing layer that works across flakes and non-flakes.

The `flake.denful` output type is Den's proposed standard for publishing aspect libraries — the equivalent of a npm package or a Haskell library, but for Nix configurations.

## Related pages

- [[aspect]] — namespaces are collections of aspects
- [[den.aspects]] — namespaced aspects in den.ful are separate from top-level den.aspects
- [[angle-bracket-syntax]] — shorthand syntax for referencing namespaced aspects
- [[flake.denful]] — the flake output type for publishing namespaces
- [[dendrix]] — community index of published aspects built on the namespace model
- [[denful]] — planned LazyVim-style distribution built on namespaces
- [[batteries]] — batteries are aspects that could be published as namespaces
- [[project-structure]] — how to organize files that define namespaces and concrete aspects
