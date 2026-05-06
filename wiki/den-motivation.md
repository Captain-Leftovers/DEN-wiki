# Den Motivation

**What it is**: The history, design philosophy, and social goal behind Den — why it was built, what came before it, and what it is trying to achieve for the Nix ecosystem.

**Why it exists**: Den was not built to be another configuration wiring tool. It was built to solve a specific unsolved problem: making Nix configurations generic enough to actually share with other people.

**Relates to**: [[aspect]], [[parametric-dispatch]], [[namespaces]], [[den-lib]], [[flake-aspects]], [[dendrix]], [[denful]], [[import-tree]], [[context-transformation-pipeline]]

---

## The core motivation

> Den was built for being able to share re-usable (parametric) cross-class Nix configurations.
> The purpose of Den is social more than purely technical.

The dominant pattern in the Nix community is copy-paste. People find a configuration they like, copy it, and adapt it. Modules are tangled with the original author's hostnames, usernames, and flake inputs. Evaluating someone else's module often pulls in all of their dependencies. Nothing is truly reusable.

Den is a bet that this can change — that Nix configurations can be shared the way libraries are shared, as generic functions that work for anyone.

## What came before

Den's author (vic) used a sequence of tools before creating Den:

| Tool | Role | What it solved | What it left unsolved |
|---|---|---|---|
| custom in-flake libs | Ad-hoc wiring | Nothing special | Everything |
| blueprint | Flake scaffolding | Cleaner flake structure | No composability model |
| snowfall | Opinionated directory layout | Predictable module loading | No composability model |
| flake-parts | Modular flake outputs | Composable flake output types | Aspects were flat, stringly-typed |
| Unify | First dendritic framework | `aspect.class` transposition | No parametric aspects, no real dependency graphs |
| flake-aspects | Parametric aspect composition | Parametric aspects, nested aspects, real deps | No host/user schema, no context pipeline |

Every tool solved the **wiring** problem — how to load modules and produce flake outputs cleanly. None solved the **composability** problem — how to make a feature generic enough for someone else to use.

## The insight: functions are the most composable things

The re-usability problem was not about syntax or file layout. It was about feature composability. In functional programming, the most composable unit is a function.

A regular NixOS module can be a function:
```nix
{ pkgs, ... }: { environment.systemPackages = [ pkgs.git ]; }
```

A dendritic aspect in Den is also a function — one that returns config for multiple classes:
```nix
{ host, user }: {
  nixos  = { pkgs, ... }: { users.users.${user.userName}.shell = pkgs.fish; };
  darwin = { ... }: { };
}
```

This function knows nothing about the caller's specific setup. It is fully generic. Anyone can include it.

The critical technical detail: `host` and `user` here are **real function arguments**, not `_module.args` or `specialArgs`. They do not depend on `config`. This makes them safe for conditional imports — something impossible with `_module.args` without triggering Nix infinite recursion.

## flake-aspects

The exploration of this idea produced [[flake-aspects]] — a dependency-free library (usable with or without flakes) focused on composability, not wiring.

flake-aspects introduced:
- **Parametric aspects** via the Nix `__functor` pattern — aspects callable as functions
- **Nested aspects** — unlike the flat `flake.modules` structure that forced names like `flake.modules.nixos."host/desktop"`
- **Real dependency declarations** — `includes = [ den.aspects.base ]` instead of stringly-typed `modules = [ "base" "gaming" ]`

Some people use flake-aspects alone without Den's framework layer. Den is the opinionated layer on top.

## What Den adds on top of flake-aspects

flake-aspects handles composability. Den answers the remaining questions specific to NixOS/Darwin infrastructure:

- How to declare machines and users as structured data: `den.hosts.<arch>.<name>.users.<user>`
- How to define shared schema options for all hosts or all users: `den.schema.host.options.vpn-group = lib.mkOption ...`
- How to apply features globally: `den.default.includes = [ (den._.unfree ["vscode"]) ]`
- How a host can affect its users' config and vice versa
- How to mix in aspects from remote sources: `den.namespace "omfnix" [ inputs.remote ]`
- How to publish aspects for the community: `{ omfnix, ... }: { omfnix.niri.nixos = ...; }`

## The ecosystem tools

Den's sharing vision depends on a set of supporting tools, all by the same author:

### [[flake-aspects]]
The dependency-free parametric aspect library Den is built on. Handles `__functor`, nested aspects, and real dependency declarations. Can be used standalone without Den's framework.

### [[import-tree]]
Recursively imports all `.nix` files from a directory tree without requiring a manual imports list. Used by Den projects to auto-load the `modules/` directory. The `_` prefix convention tells import-tree to skip a directory (used to isolate non-dendritic legacy modules).

### [[with-inputs]]
Provides flake-like input management without requiring Nix flakes. Allows non-flake Den setups to have the same input propagation semantics as flake setups. Makes modules portable between flake and non-flake environments — switching to flakes later only requires changing the entrypoint, not any module.

### [[npins]]
A non-flake dependency pinning tool — the equivalent of `flake.lock` for non-flake setups. Used in the `noflake` template and the "From Zero to Den" guide.

### [[flake-file]]
Per-file flake input declarations. Allows modules in an `import-tree` collection to declare only the inputs they actually need, rather than inheriting all inputs from a parent flake. Used by `dendrix` so community aspects don't force consumers to download unneeded inputs.

### [[dendrix]]
A planned/ongoing community index of Den aspects published on GitHub. Uses `import-tree` collections rather than flakes — aspects declare only what they need via `flake-file`, and those inputs propagate to the consumer's flake or npins setup.

The goal: make the variety of aspects people produce *visible*, motivating reuse over copy-paste. Dendrix is to Nix configurations what the NUR (Nix User Repository) is to packages — but for reusable configuration.

### [[denful]]
A planned LazyVim-style configuration distribution built on Den's namespace system. Users would include from it the way editor users include from editor distributions — curated, tested, shareable aspect collections rather than writing everything from scratch.

## Den's place in the ecosystem

Den is unique among Nix configuration frameworks in that its primary goal is social rather than technical:

| Framework | What it solves |
|---|---|
| snowfall, blueprint | Cleaner flake structure and module loading |
| flake-parts | Composable flake output types |
| Unify | Clean dendritic organization within one config |
| **Den** | **Making configurations generic and shareable across people** |

The others make your own config cleaner. Den makes it possible to share with others.

## Versioning

Den is at the v0.x series — not unstable (>180 CI tests per feature), but fast-paced. v0.x means the design is still being shaped by its users. Now is the time for users to influence Den's direction.

- **main branch** (`github:vic/den`) — bleeding edge, every PR passes CI
- **latest tag** (`github:vic/den/latest`) — follows the most recent release tag
- **versioned** (`github:vic/den/<version>`) — pinned to a specific release

Den has no runtime dependencies. Its `flakeModule` expects to be imported in a module providing `nixpkgs.lib`. It bundles its own aspect types, originally based on flake-aspects but now optimized independently.

## Gotchas

- Den's predecessor flake-aspects is still usable standalone — Den is not a replacement, it is a framework layer on top
- dendrix and denful are ongoing/planned work, not finished products — the vision is clear but the tooling is still being built
- The "social" framing is intentional: Den is designed around *giving* to others, not just making your own config work
- Den's aspect types were originally based on flake-aspects but have since diverged — they are now maintained independently within Den

## Related pages

- [[aspect]] — the primary unit of Den's sharing model
- [[parametric-dispatch]] — the technical mechanism that makes aspects generic
- [[namespaces]] — the sharing mechanism built on top of parametric aspects
- [[den-lib]] — the core library that flake-aspects evolved into inside Den
- [[context-transformation-pipeline]] — what Den adds on top of flake-aspects
- [[flake-aspects]] — Den's direct predecessor; still usable standalone
- [[dendrix]] — community index of published aspects
- [[denful]] — planned configuration distribution
- [[import-tree]] — recursive module loading used by Den projects
- [[with-inputs]] — flake-like inputs for non-flake setups