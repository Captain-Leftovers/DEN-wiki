# Den as a Library

**What it is**: Using `den.lib` directly without any of Den's framework machinery (`den.hosts`, `den.ctx`, `den.provides`) — for Nix domains outside NixOS/Darwin, or for custom pipelines where the full framework is not appropriate.

**Why it exists**: Den's framework is purpose-built for NixOS/Darwin/Home Manager infrastructure. But the underlying parametric dispatch and aspect machinery is domain-agnostic. Library-only usage unlocks Den for Terranix, NixVim, system-manager, or any custom Nix module system.

**Relates to**: [[den-lib]], [[aspect]], [[parametric-dispatch]], [[resolution]], [[context-transformation-pipeline]], [[nix-class]], [[namespaces]]

---

## Core idea

Den is two things layered on top of each other:

```
den.lib          ← domain-agnostic: parametric, canTake, take, aspects.resolve
den framework    ← OS-specific: den.hosts, den.ctx, den.provides, den.schema
```

The framework is entirely optional. You can import and use `den.lib` directly without `inputs.den.flakeModule`, without `den.hosts`, without any batteries.

## The three-step pattern

Library-only usage always follows the same three steps:

```nix
# Step 1: Define an aspect using den.lib.parametric
my-aspect = den.lib.parametric {
  terranix.resource.aws_instance.web = {
    instance_type = "t3.micro";
    ami           = "ami-0abcdef1234567890";
  };
  includes = [
    ({ env, ... }: {
      terranix.resource.aws_instance.web.tags.Environment = env;
    })
    ({ region, ... }: {
      terranix.provider.aws.region = region;
    })
  ];
};

# Step 2: Call the aspect with a context to activate parametric dispatch
bound = my-aspect { env = "production"; region = "us-east-1"; };

# Step 3: Resolve for your custom class
module = den.lib.aspects.resolve "terranix" [] bound;

# Step 4: Pass to your domain's module system
config = terranix.lib.terranixConfiguration {
  nixpkgs = nixpkgs;
  modules = [ module ];
};
```

This is exactly what `den.ctx` automates for NixOS/Darwin hosts — but here you drive it manually.

## When to use library-only

| Situation | Recommendation |
|---|---|
| Managing NixOS / Darwin / HM hosts | Use the full framework |
| Custom Nix domain (Terranix, NixVim, system-manager) | Library-only |
| Custom Nix domain + OS hosts in same flake | Framework for OS, `den.lib` for the custom domain |
| Experimenting with Den's dispatch without committing to the framework | Library-only |
| Implementing your own context pipeline from scratch | Library-only |

## NixVim example

Den used outside NixOS entirely — configuring Neovim via NixVim:

```nix
vim-aspect = den.lib.parametric {
  nixvim.plugins.telescope.enable = true;
  includes = [
    ({ userPrefs, ... }: {
      nixvim.colorscheme = userPrefs.colorscheme;
    })
  ];
};

vim-module = den.lib.aspects.resolve "nixvim" [] (vim-aspect { userPrefs = myPrefs; });

nixvimConfig = inputs.nixvim.lib.makeNixvimConfiguration {
  modules = [ vim-module ];
};
```

`nixvim` here is a custom [[nix-class]] — just a string. Den does not know or care what it means. You define what to do with the resolved module.

## Custom context pipelines

Library-only does not mean no pipeline. You can build your own context pipeline on top of `den.lib`:

```nix
# Define aspects parametrically
server-aspect = den.lib.parametric {
  nginx.virtualHosts."example.com".enable = true;
  includes = [
    ({ server, ... }: {
      nginx.virtualHosts.${server.domain}.forceSSL = server.ssl;
    })
  ];
};

# Drive dispatch manually per entity
servers = [
  { domain = "example.com"; ssl = true; }
  { domain = "staging.example.com"; ssl = false; }
];

modules = map (server:
  den.lib.aspects.resolve "nginx" [] (server-aspect { inherit server; })
) servers;
```

No `den.ctx`, no `den.hosts` — just `den.lib.parametric` and `den.lib.aspects.resolve` applied to your own data structures.

## What den.lib provides for library use

| Function | Use in library-only context |
|---|---|
| `den.lib.parametric` | Wrap an aspect with context-aware `__functor` |
| `den.lib.parametric.fixedTo attrs` | Pin an aspect to a fixed context |
| `den.lib.parametric.exactly` | Strict context matching — `{ home }` only, etc. |
| `den.lib.canTake ctx fn` | Check if context satisfies function's args |
| `den.lib.take.atLeast fn ctx` | Conditionally apply a function |
| `den.lib.take.exactly fn ctx` | Strict conditional application |
| `den.lib.aspects.resolve class [] aspect` | Collapse aspect DAG to a deferredModule |
| `den.lib.statics aspect` | Extract static includes for inspection |
| `den.lib.owned aspect` | Extract owned configs for inspection |
| `den.lib.isFn value` | Check if value is a function or has `__functor` |
| `den.lib.__findFile` | Enable angle bracket namespace syntax |

## Mixing library and framework

In a single flake, you can use the framework for OS hosts and `den.lib` directly for a custom domain:

```nix
{ inputs, den, ... }: {
  imports = [ inputs.den.flakeModule ];  # framework for OS

  # Normal framework usage for hosts
  den.hosts.x86_64-linux.laptop.users.alice = { };
  den.aspects.laptop.nixos.networking.hostName = "laptop";

  # Library-only usage for a custom domain alongside
  flake.terranixConfigurations.prod =
    let
      aspect = den.lib.parametric {
        terranix.resource.aws_instance.app = { instance_type = "t3.small"; };
      };
      module = den.lib.aspects.resolve "terranix" [] (aspect { env = "prod"; });
    in
    terranix.lib.terranixConfiguration { modules = [ module ]; };
}
```

The framework and library usage are fully independent — they share `den.lib` internally but do not interfere.

## The nvf-standalone template

Den ships a `nvf-standalone` template demonstrating library-only usage for NixVim (NVF) without any NixOS involvement:

```bash
nix flake init -t github:vic/den#nvf-standalone
```

This is the canonical reference for library-only Den usage. The CI test `den-as-lib.nix` also serves as a working example.

## Gotchas

- Den has no runtime dependencies — `den.lib` is available as soon as you import the Den flake, without needing `inputs.den.flakeModule`
- `den.lib.aspects.resolve` takes three arguments — the commonly forgotten middle argument is the aspect-chain accumulator, always pass `[]`
- Nothing in `den.lib` references `nixpkgs` or any OS-specific tooling — it is truly domain-agnostic
- Custom classes are just strings — `den.lib` does not validate class names; an aspect with `{ myCustomClass.foo = 1; }` is perfectly valid
- You are responsible for driving the pipeline manually in library-only usage — `den.ctx` is not available unless you import `inputs.den.flakeModule`

## In the Nix ecosystem

Most Nix configuration tools are tightly coupled to NixOS or to the nixpkgs ecosystem. `den.lib` is unusual in being genuinely domain-agnostic — the same parametric dispatch and aspect DAG machinery that powers NixOS host management can be applied to infrastructure-as-code (Terranix), editor configuration (NixVim), or any other Nix module system.

This positions Den as a general-purpose Nix configuration composition library, not just a NixOS framework.

## Related pages

- [[den-lib]] — the full API reference for den.lib
- [[aspect]] — the primary data structure library-only usage manipulates
- [[parametric-dispatch]] — the dispatch mechanism library-only usage relies on
- [[resolution]] — den.lib.aspects.resolve in detail
- [[nix-class]] — custom class names work the same as built-in ones
- [[context-transformation-pipeline]] — what the framework builds on top of den.lib; library-only usage replaces this with manual driving
- [[namespaces]] — den.lib.__findFile enables angle bracket syntax in library-only setups too