# Wiki Index

**Phase**: 2 — Synthesizing concept pages

---

## Den Core Concepts

- [[aspect]] — What an aspect is: the primary unit of Den, owning all cross-platform config for one feature
- [[parametric-dispatch]] — How Den matches aspect functions to contexts using argument introspection — no mkIf needed
- [[nix-class]] — What a Nix class is (nixos, darwin, homeManager, etc.) and how Den uses class names to route config
- [[context-transformation-pipeline]] — How Den walks host→users→homes, activates aspects at each stage, and produces final configs
- [[den-ctx]] — The declarative context type definitions that implement the pipeline — into transitions, providers, includes, meta.adapter
- [[den-motivation]] — Why Den was built, what came before it, and the ecosystem tools (flake-aspects, import-tree, dendrix, denful)
- [[algebraic-effects]] — den.lib.fx: effectful computations, handlers, fx.bind.fn, mock testing, and Den's migration to nix-effects
- [[resolution]] — How an aspect DAG is collapsed into a deferredModule: the six steps, collectPairs, dedupIncludes, and the resolve API
- [[hasAspect]] — Querying the resolved aspect tree from inside class-config modules: cycle-safe structural branching
- [[meta.adapter]] — Filtering and transforming aspect resolution: oneOfAspects, excludeAspect, substituteAspect, and adapter composition
- [[den-as-library]] — Using den.lib without the framework: custom Nix domains (Terranix, NixVim), the three-step pattern, mixing library and framework

## Den Framework

- [[den-schema]] — How to declare hosts, users, and homes; den.schema shared options; freeform attributes; auto-generation of aspects
- [[batteries]] — Den's built-in reusable aspects (den.provides.*): define-user, hostname, primary-user, user-shell, HM/hjem/maid integration
- [[mutual-provider]] — Bidirectional host↔user configuration via `.provides.`: user→host, host→user, user→user, standalone homes
- [[namespaces]] — Scoped aspect libraries (den.ful.*): local, exported, and imported; angle bracket syntax; foundation of Den's sharing story
- [[vm-testing]] — Boot any host config in a QEMU VM before hardware deployment; multi-host `.#vm-<name>` pattern
- [[project-structure]] — How to organize files: concrete vs reusable, folders vs namespaces, common mistakes
- [[example-template]] — Den's advanced feature showcase: cross-platform hosts, namespaces, angle brackets, mutual providers, CI checks

## Home Environments

- [[home-environments]] — Home Manager, hjem, nix-maid integration; standalone homes; host-bound homes; separate host+user pattern

## Ecosystem & Tooling

- [[den-lib]] — The domain-agnostic core library: parametric variants, canTake, take, aspects.resolve, statics, owned, __findFile, den.lib.fx

## Nix Fundamentals

_Pages will appear here as Phase 2 synthesis completes._