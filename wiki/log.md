# Wiki Log

Append-only progress log. Maintained by DAI.

---

## Current state

**Phase**: 1 — Extracting concepts from raw sources
**Last file processed**: none
**Next file to process**: `raw/den-txt/index.txt`

---

_Entries will appear below as sessions complete._

---

## 2026-04-21 — Extract: `raw/den-txt/index.txt`

### Concepts found
- **den**: The top-level framework/library — aspect-oriented, context-driven Nix configuration tool
- **aspect**: A composable attrset containing per-class Nix modules (nixos, darwin, homeManager, etc.) plus a dependency graph via includes/provides
- **dendritic**: The configuration model/pattern where features are the primary unit, not hosts — aspects own cross-cutting concerns across multiple Nix classes
- **context-transformation-pipeline**: The mechanism that traverses infra entities (host → users → homes), activating aspects at each stage to produce a unified config
- **nix-class**: A configuration domain (nixos, darwin, homeManager, system-manager, terranix, etc.) — not invented by Den, exists in the wild
- **den.ctx**: The context type system — declarative context types defining how data flows between pipeline stages
- **den.lib**: The domain-agnostic core library — provides parametric dispatch, canTake, take, aspects API; no NixOS/Darwin assumptions
- **den.aspects**: The attrset of all named aspects in a Den config; each key is an aspect name, value is the aspect definition
- **den.hosts**: Schema declaration for machines — keyed by system then name, produces nixosConfigurations/darwinConfigurations
- **den.homes**: Schema declaration for standalone home-manager configurations — keyed by system then name
- **den.schema**: Base modules applied to all hosts/users/homes — shared metadata and options
- **den.provides** (batteries): Built-in reusable aspects shipped with Den — common patterns like define-user, hostname, primary-user
- **parametric-dispatch**: The mechanism by which aspect functions only activate when the context matches their argument signature — via __functor introspection
- **namespaces**: Scoped aspect libraries (den.ful.<name>) allowing publishing and consuming aspect collections across flakes/non-flakes
- **resolution**: The process of extracting a specific class's config from an aspect bundle, merging into a final evalModules call
- **den-as-library**: Using den.lib alone without the host/homes framework — for custom domains like Terranix, NixVim
- **den-as-framework**: Using the full Den stack — schema + aspects + context pipeline + batteries + output generation
- **flake-parts**: External tool Den works with (or without) — modular flake output composition
- **includes**: The list field on an aspect declaring dependencies on other aspects — forms a DAG
- **provides**: The sub-aspect accessor on an aspect — creates named sub-aspects at aspect.provides.<name> or aspect._.<name>

### Relationships spotted
- den → den.lib: Den is built on top of den.lib; den.lib is the domain-agnostic core
- den → den-as-framework: The framework wraps den.lib with NixOS/Darwin-specific schema and pipeline
- aspect → nix-class: An aspect contains one module per nix-class it targets
- aspect → includes: aspects declare dependencies via includes forming a DAG
- aspect → provides: aspects expose sub-aspects via provides
- den.ctx → context-transformation-pipeline: den.ctx defines the types that make up the pipeline
- context-transformation-pipeline → den.hosts: pipeline is seeded by host/home declarations
- context-transformation-pipeline → parametric-dispatch: pipeline invokes aspect functions via parametric dispatch
- parametric-dispatch → den.lib: canTake/take/parametric are all in den.lib
- den.hosts → den.schema: host declarations are typed/extended by den.schema
- den.provides → aspect: batteries are themselves aspects, included like any other
- namespaces → den.aspects: namespaced aspects live in den.ful.<name> and are referenced in includes
- resolution → nix-class: resolution extracts one class's config from the aspect bundle

### Next file
`raw/den-txt/motivation/index.txt`

---

## 2026-04-21 — Extract: `raw/den-txt/overview/index.txt`

Pure navigation/TOC page — no new concepts. Skipped.

### Next file
`raw/den-txt/motivation/index.txt`

---

## 2026-04-21 — Extract: `raw/den-txt/motivation/index.txt`

### Concepts found
- **flake-aspects**: Den's direct predecessor — dependency-free library focused on composability not wiring; provides parametric aspects via __functor, nested aspects, real dependency declarations between aspects
- **unify**: First known dendritic framework — invented the aspect.class transposition pattern; had stringly-typed module dependencies as a shortcoming
- **blueprint**: Earlier Nix flake wiring tool used before flake-parts; solved wiring not composability
- **snowfall**: Another Nix flake wiring framework; opinionated directory structure approach
- **dendrix**: Planned/ongoing community aspect index — browses aspects on GitHub using import-tree collections instead of flakes, so inputs aren't forced on consumers
- **denful**: Planned LazyVim-style configuration distribution built on Den's sharable namespaces
- **import-tree**: Tool for recursively importing .nix file collections without flakes; used by dendrix so aspects declare only the inputs they need
- **flake-file**: Per-file flake input declarations; used alongside import-tree so module inputs can be propagated to user's flake or npins
- **npins**: Non-flake dependency pinning tool; alternative to flakes for locking inputs
- **with-inputs**: Helper library providing flake-like input management without requiring flakes
- **__functor**: Nix pattern that makes an attrset callable like a function — the mechanism behind parametric aspects
- **feature-composability**: The real problem Den addresses — not syntax or wiring, but making features generic and composable enough to share
- **cross-class-configuration**: Configuring a single concern (e.g. bluetooth) across multiple Nix classes (nixos, darwin, homeManager) in one place
- **den.default**: Special aspect applied to every host, user, and home — global defaults and includes
- **conditional-imports**: Using context args to conditionally include modules — safe in Den because context args are real function args, not _module.args

### Relationships spotted
- flake-aspects → __functor: flake-aspects uses __functor to make aspects parametric
- flake-aspects → parametric-dispatch: flake-aspects introduced parametric dispatch to the dendritic pattern
- den → flake-aspects: Den builds on top of flake-aspects, adds host/user schema and context pipeline
- unify → dendritic: Unify was the first dendritic framework; invented aspect.class transposition
- dendrix → import-tree: dendrix uses import-tree collections to index aspects without forcing flake inputs
- dendrix → namespaces: dendrix is the community-level expression of Den's namespace sharing model
- denful → namespaces: denful is built on Den's sharable namespace system
- __functor → parametric-dispatch: __functor is the Nix mechanism that enables parametric dispatch
- conditional-imports → parametric-dispatch: conditional imports are safe because context args don't depend on config
- den.default → aspect: den.default is itself an aspect — applied globally to all entities
- feature-composability → dendritic: the dendritic pattern is the solution to the feature-composability problem
- cross-class-configuration → aspect: aspects are the mechanism for cross-class configuration

### Next file
`raw/den-txt/explanation/core-principles.txt`

---

## 2026-04-21 — Extract: `raw/den-txt/explanation/core-principles.txt`

### Concepts found
- **aspect-oriented**: Den's first core principle — borrowed from AOP, uses pointcuts to define where config is applied at each context stage
- **pointcut**: AOP term Den adopts — a location in the pipeline where an aspect's configuration gets applied; in Den, `den.ctx.<name>.provides.<name>` is the pointcut
- **context-driven**: Den's second core principle — Den is fundamentally a data transformation pipeline; infra entities are data, aspects are functions applied to that data
- **feature-first**: Design principle — features (aspects) are the primary organizational unit, not hosts or files; hosts select which aspects apply to them, not the other way around
- **host-first**: The traditional Nix approach Den inverts — configurations start from hosts and push modules downward
- **context-driven-dispatch**: Mechanism where aspect functions declare needed context params via argument pattern; functions are silently skipped when context doesn't match — no mkIf, no enable flags needed
- **composition-via-includes**: Aspects form a DAG through includes; provides forms a tree of sub-aspects; real Nix references not stringly-typed tags
- **separation-of-concerns**: Den's four-layer model — Schema (declare entities) / Aspects (configure behavior) / Context (transform data flow) / Batteries (reusable patterns)
- **den.ctx.into**: The transition function on a context type — takes current context data and returns a list of new context values for the next stage; e.g. `den.ctx.host.into.user`
- **owned-config**: Class-keyed attributes on an aspect (nixos, darwin, homeManager) — the direct configuration a specific aspect contributes; always included regardless of context
- **static-aspect**: An aspect that is a plain attrset or a function taking `{ class, aspect-chain }` — evaluated once during resolution, not per-context
- **context-stage**: A named point in the pipeline with a defined data shape (e.g. `host` stage has `{ host }`, `user` stage has `{ host, user }`)
- **wsl-host**: A derived context stage — same data shape as host but different stage name; triggered conditionally when `host.wsl.enable = true`
- **hm-user**: A derived context stage for Home Manager user integration — `{ host, user }` shape, triggered when user has homeManager in classes
- **aspect-dag**: The directed acyclic graph formed by aspects through their includes lists — determines resolution order and dependency chain
- **microvm**: Example of Den extensibility — MicroVM guests as a custom context stage branching from host

### Relationships spotted
- aspect-oriented → pointcut: Den's aspect-oriented nature is implemented via pointcuts at context stages
- pointcut → den.ctx: den.ctx.<name>.provides.<name> is where pointcuts are defined
- context-driven → context-transformation-pipeline: the context-driven principle IS the pipeline
- context-driven-dispatch → parametric-dispatch: context-driven dispatch is the practical application of parametric dispatch
- context-driven-dispatch → __functor: dispatch works by introspecting function args via builtins.functionArgs
- feature-first → dendritic: feature-first is the defining characteristic of the dendritic pattern
- feature-first → host-first: feature-first is explicitly the inversion of host-first
- den.ctx.into → context-stage: into transitions move data from one context-stage to another
- den.ctx.into → context-transformation-pipeline: into transitions are what make the pipeline flow
- owned-config → nix-class: owned configs are keyed by nix-class name
- owned-config → aspect: owned configs are the direct configuration payload of an aspect
- static-aspect → includes: static aspects appear in includes lists; evaluated once not per-context
- wsl-host → den.ctx.into: wsl-host is reached via den.ctx.host.into.wsl-host
- hm-user → den.ctx.into: hm-user is reached via den.ctx.user.into.hm-user
- aspect-dag → includes: the DAG is formed by includes references between aspects
- aspect-dag → resolution: resolution traverses the aspect DAG to collect all configs
- separation-of-concerns → den.schema: schema layer is the "declare entities" concern
- separation-of-concerns → aspect: aspects layer is the "configure behavior" concern
- separation-of-concerns → den.ctx: context layer is the "transform data flow" concern
- separation-of-concerns → den.provides: batteries layer is the "reusable patterns" concern

### Next file
`raw/den-txt/explanation/parametric.txt`

---

## 2026-04-21 — Extract: `raw/den-txt/explanation/parametric.txt`

### Concepts found
- **builtins.functionArgs**: Native Nix builtin used by Den to introspect a function's required argument names at evaluation time — the foundation of parametric dispatch
- **canTake**: `den.lib.canTake` — checks if a function's required arguments are satisfiable by a given context attrset; two modes: atLeast (default) and exactly
- **canTake.atLeast**: All of the function's required params must be present in the context; extra context keys are fine — this is the default matching mode
- **canTake.exactly**: The function's required params must match the context keys exactly — no extras allowed on either side
- **take**: `den.lib.take` — wraps canTake into aspect-ready helpers; `take.atLeast fn ctx` and `take.exactly fn ctx`; used inside includes to control activation
- **parametric-constructor**: `den.lib.parametric` — rewrites an aspect's `__functor` to process includes through parametric dispatch; owned configs always included, static includes always included, function includes filtered by context match
- **parametric.atLeast**: Variant — does NOT include owned configs or static includes; only activates functions matching atLeast the context
- **parametric.exactly**: Variant — like atLeast but uses canTake.exactly for matching
- **parametric.fixedTo**: Variant — ignores received context entirely, always dispatches using the given fixed attrs
- **parametric.expands**: Variant — extends the received context with additional attrs before dispatch
- **perUser**: `den.lib.perUser` — wraps a function so it only matches `{host, user}` contexts exactly
- **perHome**: `den.lib.perHome` — wraps a function so it only matches `{home}` contexts exactly
- **anonymous-function-antipattern**: Inlining anonymous functions in includes lists is discouraged — named aspects produce better error traces and improve readability
- **context-aware-battery-pattern**: Using `parametric.exactly` with multiple functions in includes to handle different context shapes (host+user vs home) without any mkIf

### Relationships spotted
- builtins.functionArgs → parametric-dispatch: parametric dispatch is implemented by calling builtins.functionArgs on each function in includes
- canTake → builtins.functionArgs: canTake uses builtins.functionArgs internally to read a function's required params
- canTake → parametric-dispatch: canTake is the predicate that decides whether a function activates for a given context
- canTake.atLeast → canTake: atLeast is the default mode of canTake
- canTake.exactly → canTake: exactly is the strict mode of canTake
- take → canTake: take wraps canTake into convenient helpers for use in aspects
- parametric-constructor → __functor: parametric rewrites the __functor on the aspect attrset
- parametric-constructor → includes: parametric processes the includes list through dispatch
- parametric-constructor → owned-config: owned configs are always included by the default parametric variant
- parametric.atLeast → parametric-constructor: atLeast variant skips owned and static, only dispatches functions
- parametric.exactly → canTake.exactly: exactly variant uses canTake.exactly for matching
- parametric.fixedTo → parametric-constructor: fixedTo ignores context and uses fixed attrs
- parametric.expands → parametric-constructor: expands extends context before dispatch
- perUser → canTake.exactly: perUser is sugar for canTake.exactly matching {host, user}
- perHome → canTake.exactly: perHome is sugar for canTake.exactly matching {home}
- context-aware-battery-pattern → parametric.exactly: the battery pattern uses parametric.exactly to separate host+user vs home concerns
- context-aware-battery-pattern → den.provides: built-in batteries use this pattern extensively

### Next file
`raw/den-txt/explanation/aspects/index.txt`

---

## 2026-04-21 — Extract: `raw/den-txt/explanation/aspects/index.txt`

### Concepts found
- **aspect-as-functor**: Every aspect in Den is an attrset with a `__functor` — making it callable like a function; the functor receives context and returns config
- **functor-pattern**: The Nix `__functor` pattern — any attrset with `__functor` can be called as a function; `__functor = self: arg: ...` receives self (the attrset) and returns a function
- **aspect-structure**: The three kinds of attributes an aspect has — owned configs (class-keyed), includes (dependency list), provides (named sub-aspects)
- **owned-configurations**: Class-keyed direct settings on an aspect (`nixos`, `darwin`, `homeManager`) — applied unconditionally, always part of the resolved output
- **den.lib.aspects.resolve**: The API that resolves an aspect for a specific class — collects owned configs, walks includes recursively, applies parametric dispatch, deduplicates, produces a single deferredModule
- **deferredModule**: The output type of resolution — a single merged Nix module containing all configuration for a specific class, ready to pass to evalModules or nixosSystem
- **aspect-chain**: The sequence of aspects traversed during resolution via includes — available to static aspects as the `aspect-chain` argument
- **recursive-includes-walk**: During resolution, Den walks the includes list recursively — each included aspect's includes are also walked, collecting the full transitive closure

### Relationships spotted
- aspect-as-functor → __functor: every aspect has a __functor installed by den.lib.parametric
- aspect-as-functor → parametric-constructor: den.lib.parametric is what installs the context-aware __functor
- functor-pattern → __functor: the functor-pattern is the general Nix mechanism; __functor is the specific key
- aspect-structure → owned-configurations: owned configs are one of the three aspect attribute kinds
- aspect-structure → includes: includes is one of the three aspect attribute kinds
- aspect-structure → provides: provides is one of the three aspect attribute kinds
- owned-configurations → nix-class: owned configs are keyed by nix-class name
- owned-configurations → resolution: resolution collects owned configs as always-included base config
- den.lib.aspects.resolve → resolution: resolve is the function that performs resolution
- den.lib.aspects.resolve → aspect-dag: resolve walks the DAG formed by includes
- den.lib.aspects.resolve → deferredModule: resolve produces a deferredModule as output
- deferredModule → nix-class: a deferredModule is class-specific — one per class
- recursive-includes-walk → aspect-dag: the recursive walk IS the traversal of the aspect DAG
- aspect-chain → static-aspect: static aspects receive aspect-chain as an argument during resolution

### Next file
`raw/den-txt/explanation/context-pipeline/index.txt`

---

## 2026-04-21 — Extract: `raw/den-txt/explanation/context-pipeline/index.txt`

### Concepts found
- **host-context-step**: The first pipeline step — for each `den.hosts` entry, a `{host}` context is created; `_.host` applies host's aspect with `parametric.fixedTo {host}`; `_.user` applies host's aspect with `parametric.atLeast {host,user}`
- **user-context-step**: The second pipeline step — `into.user` maps each `host.users` entry into a `{host, user}` context; `_.user` applies user's aspect with `parametric.fixedTo {host, user}`
- **derived-context-step**: Battery-registered pipeline extensions — each battery (HM, hjem, maid, WSL) registers its own `into.*` transitions on the host context with activation conditions
- **home-context-path**: Separate pipeline path for standalone `den.homes` entries — goes through `den.ctx.home {home}` → `fixedTo {home}` on home's aspect → `homeConfigurations.<name>`
- **host.mainModule**: Internally computed by resolving the host's aspect with its context — the final module passed to `host.instantiate`
- **host.instantiate**: The builder function on a host — defaults to `lib.nixosSystem`, `darwinSystem`, or `homeManagerConfiguration` depending on class; called with `host.mainModule`
- **intoAttr**: The flake output path for a host/home — defaults to `["nixosConfigurations" name]`, `["darwinConfigurations" name]`, etc.; can be overridden or set to `[]` to skip output
- **flake-output-generation**: `modules/config.nix` — collects all hosts and homes, calls `host.instantiate`, places results into flake outputs via `intoAttr`
- **modules/config.nix**: The file responsible for output generation — bridges the context pipeline results into actual flake outputs
- **perSystem-outputs**: Den aspects can contribute `packages`, `checks`, `devShells` etc. via a `flake-system` context that fans out per system

### Relationships spotted
- host-context-step → parametric.fixedTo: _.host uses fixedTo to bind the host context permanently
- host-context-step → parametric.atLeast: _.user uses atLeast to activate parametric includes needing {host,user}
- user-context-step → den.ctx.into: into.user drives the host→user transition
- derived-context-step → den.ctx.into: each battery registers into.* on the host context
- derived-context-step → den.provides: batteries are what register the derived context transitions
- home-context-path → den.homes: standalone homes trigger the separate home context path
- home-context-path → den.ctx.home: home context path goes through den.ctx.home
- host.mainModule → resolution: mainModule is the result of resolving the host's full aspect
- host.instantiate → den.hosts: every host has an instantiate function (defaulted by class)
- flake-output-generation → host.instantiate: config.nix calls instantiate for each host/home
- flake-output-generation → intoAttr: output is placed at the path specified by intoAttr
- intoAttr → den.hosts: intoAttr is a field on the host schema
- perSystem-outputs → den.ctx: perSystem outputs use a custom flake-system context

### Next file
`raw/den-txt/explanation/effects/index.txt`

---

## 2026-04-21 — Extract: `raw/den-txt/explanation/effects/index.txt`

### Concepts found
- **algebraic-effects**: The theoretical foundation for Den's future dependency injection system — computations that declare what they need from the outside world; handlers provide those dependencies
- **effectful-computation**: A function that asks for something from the outside before it can return a result — the Den aspect `{ host, user }: { ... }` is an effectful computation
- **effect-request**: `fx.send "name" payload` — declares a dependency on an external value; analogous to a function argument name
- **handler**: A function that answers effect requests — receives `{ param, state }` and returns `{ resume, state }`; Den's context pipeline acts as the handler for host/user requests
- **den.lib.fx**: Den's experimental algebraic effects API — available on a feature branch; provides `fx.pure`, `fx.send`, `fx.bind`, `fx.bind.fn`, `fx.handle`
- **fx.pure**: Wraps a plain value in an effect — no dependencies needed
- **fx.bind**: Chains effectful computations — result of one feeds into the next; like Promises/Futures
- **fx.bind.fn**: Converts any plain Nix function into an effectful computation automatically — uses `builtins.functionArgs` to generate effect requests for each arg name
- **fx.handle**: Runs an effectful computation with a set of handlers — produces `{ state, value }`
- **nix-effects**: The external library Den is moving to for its effects system — based on vic/nix-effects, vic/nfx, vic/fx-rs, vic/fx.go
- **dependency-injection**: The pattern at the heart of Den's aspect system — aspects declare what they need (host, user), the pipeline injects the values; effects formalize this
- **mock-testing**: A benefit of the effects model — the same aspect function can be tested with mock handlers instead of real nixpkgs/hosts, bypassing the Nix store
- **aspect-communication**: Future capability enabled by effects — aspects communicating with each other via a higher-level message handler for deduplication, feature detection, aspect replacement

### Relationships spotted
- algebraic-effects → dependency-injection: algebraic effects are a formal model for dependency injection
- effectful-computation → aspect: Den aspects are effectful computations — they declare host/user as dependencies
- effectful-computation → builtins.functionArgs: fx.bind.fn uses builtins.functionArgs to generate effect requests
- handler → context-transformation-pipeline: Den's context pipeline IS the handler for host/user effect requests
- den.lib.fx → den.lib: den.lib.fx is part of den.lib (on feature branch)
- fx.bind.fn → builtins.functionArgs: fx.bind.fn introspects function args to auto-generate requests
- fx.bind.fn → parametric-dispatch: fx.bind.fn is the effects-based equivalent of parametric dispatch
- nix-effects → den.lib.fx: den.lib.fx is built on nix-effects
- dependency-injection → parametric-dispatch: current parametric dispatch is a form of dependency injection; effects formalize it
- mock-testing → handler: mock testing works by swapping handlers

### Next file
`raw/den-txt/explanation/parametric/index.txt`

---

## 2026-04-21 — Extract: `raw/den-txt/explanation/parametric/index.txt`

Duplicate of `raw/den-txt/explanation/parametric.txt` — same content already extracted. Skipped.

### Next file
`raw/den-txt/explanation/library-vs-framework/index.txt`

---

## 2026-04-21 — Extract: `raw/den-txt/explanation/library-vs-framework/index.txt`

### Concepts found
- **den.lib.statics**:

---

## 2026-04-21 — Extract: `raw/den-txt/explanation/context-system/index.txt`

### Concepts found
- **context**: A named stage in Den's evaluation pipeline — declared under `den.ctx.<name>`; carries provides (aspect contributions), into (transitions to other contexts), and cross-context providers
- **context-type**: The definition of a context stage — `den.ctx.<name>` is a context type with description, provides, and into fields
- **den.ctx.provides**: Maps context names to provider functions — functions that take current context data and return aspect fragments contributed at this stage
- **den.ctx.into**: Maps to functions that take current context data and return a list of new context values — drives the pipeline traversal
- **collectPairs**: Internal Den function — walks all `into.*` transitions recursively from a starting context, building a flat list of `{ ctx, ctxDef, source }` pairs
- **dedupIncludes**: Internal Den function — processes collectPairs output; uses `parametric.fixedTo` for first occurrence of each context type (to bind owned configs once) and `parametric.atLeast` for subsequent occurrences (to prevent duplicate owned configs)
- **hjem-host**: Derived context stage for hjem integration — triggered when `hjem` is enabled on a host; branches from host context
- **hjem-user**: Derived context stage — `{ host, user }` shape; reached from hjem-host per hjem-class user
- **maid-host**: Derived context stage for nix-maid integration — triggered when maid is enabled on a host
- **maid-user**: Derived context stage — `{ host, user }` shape; reached from maid-host per maid-class user
- **custom-context**: A user-defined context type extending the pipeline for domain-specific needs (e.g. cloud infra, custom services); added by defining `den.ctx.<name>` and wiring it via `den.ctx.host.into.<name>`
- **parse-dont-validate**: Design principle Den follows for context types — type-driven design ensures only valid contexts exist; invalid context shapes cannot be constructed

### Relationships spotted
- context → context-type: a context is an instance of a context-type at a given pipeline stage
- context-type → den.ctx: context types are declared under den.ctx.<name>
- den.ctx.provides → context-stage: provides functions fire at a specific context stage and return aspect fragments
- den.ctx.into → context-stage: into functions transition from one context-stage to another
- den.ctx.into → context-transformation-pipeline: into transitions are what drive the pipeline forward
- collectPairs → den.ctx.into: collectPairs recursively walks all into transitions
- collectPairs → context-transformation-pipeline: collectPairs is the internal implementation of the pipeline traversal
- dedupIncludes → collectPairs: dedupIncludes processes the output of collectPairs
- dedupIncludes → parametric.fixedTo: first occurrence of a context type uses parametric.fixedTo
- dedupIncludes → parametric.atLeast: subsequent occurrences use parametric.atLeast to avoid duplicate owned configs
- hjem-host → den.ctx.into: hjem-host is reached via den.ctx.host.into.hjem-host
- hjem-user → hjem-host: hjem-user is reached from hjem-host per user
- maid-host → den.ctx.into: maid-host is reached via den.ctx.host.into.maid-host
- maid-user → maid-host: maid-user is reached from maid-host per user
- custom-context → den.ctx: custom contexts are defined under den.ctx.<name>
- custom-context → den.ctx.into: custom contexts are wired into the pipeline via into transitions on existing contexts
- parse-dont-validate → context-type: Den uses parse-dont-validate to ensure only valid context shapes exist

### Next file
`raw/den-txt/explanation/context-pipeline/index.txt`

---

## 2026-04-21 — Extract: `raw/den-txt/explanation/context-pipeline/index.txt`

### Concepts found
- **host-context**: The entry point of the pipeline — `den.ctx.host { host }` — produced for each `den.hosts.<system>.<name>`; contributes `_.host` (fixedTo host aspect) and `_.user` (atLeast host aspect with host+user)
- **user-context**: `den.ctx.user { host, user }` — produced per user via `into.user`; contributes `_.user` (fixedTo user aspect)
- **home-context**: `den.ctx.home { home }` — produced per `den.homes` entry; standalone path, no host; functions requiring `{ host }` are not activated
- **derived-context**: Any context stage reached via `into.*` from another context — hm-host, hm-user, wsl-host, hjem-host, hjem-user, maid-host, maid-user; each has a condition that must be true
- **host.mainModule**: The internally computed module for a host — result of resolving the host's aspect through its context pipeline; passed to `host.instantiate`
- **host.instantiate**: The builder function called per host — defaults to `lib.nixosSystem` (NixOS), `darwinSystem` (Darwin), `homeManagerConfiguration` (home); places result at `flake.<intoAttr>`
- **flake.intoAttr**: The path in flake outputs where a host/home result is placed — default `[ "nixosConfigurations" name ]` for NixOS; can be overridden or set to `[]` to skip
- **den.ctx.home-path**: Standalone homes go through a separate path — `den.ctx.home { home }` → fixedTo `{ home }` → `homeConfigurations.<name>`; no host context, no host-requiring functions
- **modules/config.nix**: The Den module that drives output generation — collects all hosts and homes, calls instantiate, places results into flake outputs
- **deduplication**: The process ensuring `den.default` configs are not applied multiple times when an aspect appears at multiple pipeline stages — done by `dedupIncludes` using fixedTo for first occurrence and atLeast for subsequent

### Relationships spotted
- host-context → context-transformation-pipeline: host-context is the entry point of the pipeline
- host-context → host.mainModule: host-context resolution produces host.mainModule
- host.mainModule → host.instantiate: mainModule is passed as a module to instantiate
- host.instantiate → deferredModule: instantiate receives the deferredModule as its modules list
- host.instantiate → flake.intoAttr: output is placed at the path defined by intoAttr
- user-context → host-context: user-context is derived from host-context via into.user
- home-context → den.ctx: home-context is a separate root context, not derived from host-context
- derived-context → den.ctx.into: derived contexts are reached via into transitions
- modules/config.nix → host.instantiate: config.nix drives the instantiate call for each host/home
- deduplication → dedupIncludes: deduplication is implemented by dedupIncludes
- den.ctx.home-path → home-context: standalone homes use the home-context path
- flake.intoAttr → den.hosts: intoAttr is a field on each host/home definition

### Next file
`raw/den-txt/explanation/effects/index.txt`

---

## 2026-04-21 — Extract: `raw/den-txt/explanation/effects/index.txt`

### Concepts found
- **algebraic-effects**: The theoretical foundation for Den's future dependency injection system — computations declare what they need from the outside world; handlers provide those dependencies; Den is moving to this model from its current rigid-context system
- **effectful-computation**: A function that asks for something from the outside before it can return a result — a Den aspect `{ host, user }: { ... }` is an effectful computation
- **effect-request**: `fx.send "name" payload` — declares a dependency on an external value; analogous to a function argument
- **handler**: A function that answers effect requests — receives `{ param, state }`, returns `{ resume, state }`; Den's context pipeline acts as the handler for host/user requests
- **den.lib.fx**: Den's experimental algebraic effects API (feature branch) — provides `fx.pure`, `fx.send`, `fx.bind`, `fx.bind.fn`, `fx.handle`
- **fx.pure**: Wraps a plain value in an effect — no external dependencies needed
- **fx.bind**: Chains effectful computations — result of one feeds into the next; analogous to Promises/Futures
- **fx.bind.fn**: Converts any plain Nix function into an effectful computation automatically — uses `builtins.functionArgs` to generate one effect request per argument name
- **fx.handle**: Runs an effectful computation with a set of handlers — produces `{ state, value }`
- **nix-effects**: External library Den is migrating to for its effects system — based on vic/nix-effects, replacing hand-rolled context threading
- **dependency-injection**: The pattern at the core of Den aspects — aspects declare what they need (`host`, `user`), the pipeline injects values; effects formalize and generalize this
- **mock-testing**: Benefit of the effects model — swap handlers to test aspect functions with mock values instead of real nixpkgs/hosts, bypassing the Nix store entirely
- **aspect-communication**: Future capability — aspects communicating with each other via a higher-level message handler for advanced deduplication, feature detection, aspect replacement

### Relationships spotted
- algebraic-effects → dependency-injection: algebraic effects are a formal model for dependency injection
- effectful-computation → aspect: Den aspects are effectful computations
- effectful-computation → builtins.functionArgs: fx.bind.fn uses builtins.functionArgs to auto-generate requests
- handler → context-transformation-pipeline: Den's context pipeline IS the handler for host/user requests
- fx.bind.fn → builtins.functionArgs: fx.bind.fn introspects function args to generate requests
- fx.bind.fn → parametric-dispatch: fx.bind.fn is the effects-based equivalent of parametric dispatch
- nix-effects → den.lib.fx: den.lib.fx is built on nix-effects
- dependency-injection → parametric-dispatch: current parametric dispatch is a form of dependency injection; effects will formalize it
- mock-testing → handler: mock testing works by swapping handlers
- den.lib.fx → den.lib: den.lib.fx is part of den.lib (experimental branch)

### Next file
`raw/den-txt/explanation/library-vs-framework/index.txt`

---

## 2026-04-21 — Extract: `raw/den-txt/explanation/library-vs-framework/index.txt`

### Concepts found
- **den.lib.statics**: `den.lib` function — extracts only static includes from an aspect (excludes parametric functions and functor)
- **den.lib.owned**: `den.lib` function — extracts only owned configs from an aspect (no includes, no functor)
- **den.lib.isFn**: `den.lib` function — checks if a value is a function or has `__functor`
- **den.lib.__findFile**: `den.lib` function — enables angle bracket syntax `<namespace/path>` for deep aspect path resolution
- **terranix**: Example non-OS Nix module system — used to demonstrate den.lib used outside NixOS/Darwin context
- **nixvim**: Example non-OS Nix module system — another domain where den.lib can be used standalone
- **system-manager**: Another non-OS Nix class — Den framework supports it alongside nixos/darwin
- **library-only-usage**: Using `den.lib` without any of the `den.hosts`/`den.aspects` framework machinery — for custom Nix domains like Terranix, NixVim
- **framework-usage**: Using the full Den stack — `den.hosts`/`den.homes` schema + `den.ctx` pipeline + `den.provides` batteries + output generation in `modules/config.nix`
- **angle-bracket-syntax**: `<namespace/path>` shorthand for referencing deep aspect paths — enabled by setting `_module.args.__findFile = den.lib.__findFile`

### Relationships spotted
- den.lib.statics → den.lib: statics is a utility function in den.lib
- den.lib.owned → den.lib: owned is a utility function in den.lib
- den.lib.isFn → den.lib: isFn is a utility function in den.lib
- den.lib.__findFile → angle-bracket-syntax: __findFile is the mechanism behind angle bracket syntax
- angle-bracket-syntax → namespaces: angle brackets provide shorthand for referencing namespaced aspects
- library-only-usage → den.lib: library-only usage means using just den.lib
- library-only-usage → terranix: terranix is a primary use case for library-only Den
- framework-usage → den.hosts: framework usage requires den.hosts/den.homes
- framework-usage → den.ctx: framework usage uses the full den.ctx pipeline
- framework-usage → den.provides: framework usage leverages batteries

### Next file
`raw/den-txt/reference/schema.txt`

---

## 2026-04-21 — Extract: `raw/den-txt/reference/schema.txt`

### Concepts found
- **den.schema.conf**: Base module applied to ALL entities — host, user, and home; the shared root schema
- **den.schema.host**: Base module applied to all hosts — imports conf; adds host-specific options
- **den.schema.user**: Base module applied to all users — imports conf; adds user-specific options including `classes`
- **den.schema.home**: Base module applied to all homes — imports conf; adds home-specific options
- **freeform-type**: The type used for host/user/home objects — allows any arbitrary attribute to be assigned as metadata alongside defined options
- **host.resolved**: Auto-derived field on every host — the aspect produced by running the host through its context pipeline; used internally by mainModule
- **user.resolved**: Auto-derived field on every user — same concept as host.resolved but for user context
- **home.resolved**: Auto-derived field on every home — same concept for home context
- **meta.adapter**: Field on a context node or aspect — a function that controls/filters how includes are resolved within that context transitively; context adapters are outermost, aspect adapters are innermost
- **den.lib.aspects.adapters.filter**: Utility for creating adapters that filter aspects by name or meta.provider during resolution
- **user.classes**: The list of home management Nix classes a user participates in — defaults to `[ "homeManager" ]`; controls which derived contexts are activated (hm-user, hjem-user, maid-user)
- **host.class**: The primary OS class of a host — auto-derived from platform (`x86_64-linux` → `"nixos"`, `aarch64-darwin` → `"darwin"`)
- **host.aspect**: The name of the primary aspect for a host — defaults to the host's attrset key name
- **user.aspect**: The name of the primary aspect for a user — defaults to the user's attrset key name

### Relationships spotted
- den.schema.conf → den.schema.host: host schema imports conf
- den.schema.conf → den.schema.user: user schema imports conf
- den.schema.conf → den.schema.home: home schema imports conf
- freeform-type → den.hosts: host definitions use freeformType allowing arbitrary metadata
- freeform-type → den.schema: schema options are layered on top of freeformType
- host.resolved → host-context: resolved is produced by running the host through its context pipeline
- meta.adapter → resolution: adapters control which aspects are included during resolution
- meta.adapter → den.lib.aspects.adapters.filter: adapters are typically built with the filter utility
- user.classes → hm-user: having "homeManager" in classes activates the hm-user derived context
- user.classes → hjem-user: having "hjem" in classes activates the hjem-user derived context
- host.class → nix-class: host.class is the resolved nix-class for the host platform
- host.aspect → den.aspects: the host.aspect name is the key in den.aspects that holds the host's primary aspect

### Next file
`raw/den-txt/reference/aspects.txt`

---

## 2026-04-21 — Extract: `raw/den-txt/reference/aspects.txt`

### Concepts found
- **auto-generation**: Den automatically creates `den.aspects` entries for every declared host/user/home — you do not need to declare them manually unless adding shared/generic aspects
- **den.ful**: `attrsOf aspectsType` — namespaced aspect collections; each key is a namespace name, value is a full aspects collection; populated by `den.namespace` or by merging upstream `denful` flake outputs
- **flake.denful**: Raw flake output type — for publishing namespaces; set automatically by `den.namespace`; consumed by downstream flakes
- **meta.provider**: Tracks structural origin of an aspect as a path — top-level aspects have `[]`; aspects from `foo.provides.bar` have `["foo"]`; deeply nested `foo._.bar._.baz` has `["foo" "bar"]`
- **cross-context-forwarding**: Pattern using `den._.forward` — pulls configuration from one entity's `.resolved` into another entity without manual wiring; used to e.g. collect SSH host keys from all peers
- **ctxApply**: Internal function — applies parametric includes to a context during class resolution (step 5 of resolution)
- **den._.forward**: Battery that forwards aspect configuration from one entity's resolved context into another — used for separation of concerns and custom Nix classes
- **aspect-custom-submodule**: Pattern where an aspect uses function-args style `{ config, ... }:` to define custom `options` and `config` values within the aspect — makes `imports` not be interpreted as a Nix class
- **aspect-fixed-point**: Aspects are fixed-point — they can reference themselves via `{ config, ... }:` where `config` represents the aspect's own module config

### Relationships spotted
- auto-generation → den.hosts: Den auto-generates aspects from host declarations
- auto-generation → den.aspects: generated aspects are placed in den.aspects
- den.ful → namespaces: den.ful is the internal representation of namespaces
- den.ful → flake.denful: flake.denful exports the namespace for downstream consumption
- meta.provider → provides: aspects accessed via .provides.* have meta.provider set to track origin
- meta.provider → meta.adapter: adapters can filter by meta.provider to select aspects by origin
- cross-context-forwarding → host.resolved: forwarding reads from the source entity's .resolved
- cross-context-forwarding → den._.forward: the forward battery implements cross-context forwarding
- ctxApply → parametric-dispatch: ctxApply applies parametric dispatch during resolution
- ctxApply → resolution: ctxApply is step 5 of class resolution
- aspect-fixed-point → aspect-custom-submodule: fixed-point and custom submodule both use function-args style

### Next file
`raw/den-txt/reference/ctx.txt`

---

## 2026-04-21 — Extract: `raw/den-txt/reference/ctx.txt`

### Concepts found
- **den.ctx.modules**: Field on a context type — additional modules merged directly into the resolved output (beyond what aspects contribute)
- **den.ctx.includes**: Field on a context type — aspect includes attached to the context type itself; used by batteries to inject behavior at specific pipeline stages without modifying aspect definitions
- **den.ctx.host._.user**: Provider on host context — applies the host's aspect with `atLeast { host, user }` for every user; activates parametric includes that need both host and user
- **den.ctx.user._.user**: Provider on user context — applies the user's aspect with `fixedTo { host, user }` binding the specific user
- **den.ctx.home._.home**: Provider on home context — applies the home's aspect with `fixedTo { home }`
- **den.ctx.hm-host.provides.hm-host**: Provider — imports the Home Manager OS module (`home-manager.nixosModules.home-manager` or darwin equivalent)
- **den.ctx.hm-user._.hm-user**: Provider — forwards the `homeManager` class content to `home-manager.users.<userName>`
- **den.ctx.wsl-host.provides.wsl-host**: Provider — imports WSL NixOS module, creates wsl class forward
- **ctx-meta-adapter-composition**: Context adapters compose with aspect-level adapters — context adapters are outermost (applied first), aspect adapters are innermost (applied last)

### Relationships spotted
- den.ctx.modules → deferredModule: modules on a context type are merged into the final deferredModule
- den.ctx.includes → den.ctx: includes on a context type inject behavior at that pipeline stage
- den.ctx.includes → batteries: batteries use ctx.includes to hook into pipeline stages
- den.ctx.host._.user → parametric-dispatch: atLeast dispatch activates host-aspect parametric functions that need both host and user
- den.ctx.hm-host.provides.hm-host → hm-host: the hm-host provider is what imports the HM OS module
- den.ctx.hm-user._.hm-user → hm-user: the hm-user provider forwards homeManager class content
- ctx-meta-adapter-composition → meta.adapter: adapters on contexts compose with adapters on aspects

### Next file
`raw/den-txt/reference/batteries.txt`

---

## 2026-04-21 — Extract: `raw/den-txt/reference/batteries.txt`

### Concepts found
- **den._.define-user**: Battery — creates OS-level user account (`users.users.<name>`) with `isNormalUser` and home dir; sets `home.username`/`home.homeDirectory` for HM; works in host-user and standalone home contexts
- **den._.hostname**: Battery — sets system hostname from `den.hosts.<name>.hostName`; works on NixOS/Darwin/WSL
- **den._.os-user**: Battery — user class automatically enabled by Den to forward settings into `host.users.users.<userName>`; forwards to NixOS/Darwin classes
- **den._.primary-user**: Battery — marks a user as primary/admin; NixOS: adds wheel+networkmanager groups; Darwin: sets `system.primaryUser`; WSL: sets `defaultUser`
- **den._.user-shell**: Battery — sets login shell at both OS and HM levels; enables `programs.<shell>.enable` on OS and HM; sets `users.users.<name>.shell`; takes shell name as argument
- **den._.mutual-provider**: Battery — allows hosts and users to contribute config to each other via `.provides.`; user can target specific host by name and vice versa
- **den._.tty-autologin**: Battery — configures TTY1 auto-login via `services.getty.autologinUser`; NixOS only
- **den._.wsl**: Battery — WSL-specific activation; enables `den.ctx.wsl-host` when `host.wsl.enable = true`; includes NixOS-WSL module
- **den._.import-tree**: Battery — provides `inputs.import-tree` helpers to import trees of non-dendritic legacy modules; auto-detects class from directory names (`_nixos/`, `_darwin/`, `_homeManager/`)
- **den._.inputs'**: Battery — exposes flake-parts `inputs'` (system-qualified inputs) into aspect modules; requires flake-parts
- **den._.self'**: Battery — exposes flake-parts `self'` (system-qualified self outputs) into aspect modules; requires flake-parts
- **den._.unfree**: Battery — allows specific unfree packages by name via `nixpkgs.config.allowUnfreePredicate`; works for any class
- **den._.insecure**: Battery — allows specific insecure packages by name; works for any class
- **home-manager-battery**: The HM integration batteries — define `den.ctx.hm-host` and `den.ctx.hm-user`; merge HM config into host OS; bridge `home-manager.users.<name>` with user's `homeManager` class
- **hjem-battery**: The hjem integration batteries — integrate hjem as alternative to HM; set `hjem.users.<name>` from user's hjem class
- **maid-battery**: The nix-maid integration batteries — merge maid config into `users.users.<name>.maid`

### Relationships spotted
- den._.define-user → den.provides: define-user is a battery in den.provides
- den._.define-user → user-context: works in host-user context and standalone home context
- den._.primary-user → user-context: primary-user battery activates in user context
- den._.mutual-provider → provides: mutual-provider enables cross-aspect provides between host and user aspects
- den._.wsl → wsl-host: wsl battery activates the wsl-host derived context
- den._.import-tree → import-tree: the battery wraps the import-tree tool
- den._.inputs' → flake-parts: inputs' battery requires flake-parts to be present
- home-manager-battery → hm-host: HM battery defines and activates hm-host context
- home-manager-battery → hm-user: HM battery defines and activates hm-user context
- hjem-battery → hjem-host: hjem battery defines hjem-host context
- maid-battery → maid-host: maid battery defines maid-host context

### Next file
`raw/den-txt/reference/output.txt`

---

## 2026-04-21 — Extract: `raw/den-txt/reference/output.txt`

### Concepts found
- **outputs.nix**: Den module — when flake-parts is absent, Den defines its own `options.flake` so output generation works identically with or without flake-parts
- **flake-parts-compatibility**: Den works with or without flake-parts — outputs.nix provides the same `options.flake` interface either way
- **custom-instantiate**: Override `host.instantiate` to use a different builder (e.g. `inputs.nixos-unstable.lib.nixosSystem`) or add `specialArgs`
- **custom-intoAttr**: Override `host.intoAttr` to place output at a custom flake path; set to `[]` to skip output placement entirely
- **packages-class**: Aspects can contribute to `flake.packages` and other per-system outputs by using a `packages` class in an aspect — requires defining `den.ctx.flake-system` and `den.ctx.flake-packages` contexts
- **den.flakeOutputs.packages**: Import to enable package output merging when not using flake-parts and having multiple packages

### Relationships spotted
- outputs.nix → modules/config.nix: outputs.nix and config.nix together drive output generation
- flake-parts-compatibility → flake-parts: Den replicates flake-parts' options.flake when flake-parts is absent
- custom-instantiate → host.instantiate: override host.instantiate to customize the builder
- custom-intoAttr → flake.intoAttr: custom-intoAttr is the pattern of overriding intoAttr
- packages-class → nix-class: packages is a custom nix-class used for flake output generation
- packages-class → aspect: aspects use the packages class to contribute flake outputs

### Next file
`raw/den-txt/guides/configure-aspects.txt`

---

## 2026-04-21 — Extract: `raw/den-txt/guides/configure-aspects.txt`

### Concepts found
- **aspect-meta.name**: Default meta field — `"den.aspects.igloo"` derived from the aspect's location path
- **aspect-meta.loc**: Default meta field — the location array `[ "den" "aspects" "igloo" ]` from which the name is derived
- **aspect-meta.file**: Default meta field — the last file where this aspect was defined
- **aspect-meta.self**: Default meta field — a reference to the aspect module's own `config`
- **incremental-features**: Dendritic pattern — any file can contribute to any aspect; aspects are open and composable; not monolithic; reflects Den's non-monolithic design philosophy
- **aspect-auto-creation**: Den creates a `parametric` aspect for each host and user automatically — you extend these, you do not replace them

### Relationships spotted
- aspect-meta.name → aspect: meta.name is a field on every aspect's meta submodule
- aspect-meta.loc → aspect-meta.name: meta.name is derived from meta.loc
- aspect-meta.self → aspect-fixed-point: meta.self enables the fixed-point self-reference pattern
- incremental-features → dendritic: incremental features is a core property of the dendritic pattern
- incremental-features → den.aspects: any module can contribute to any aspect at any time
- aspect-auto-creation → auto-generation: auto-creation is the same as auto-generation from host/user declarations

### Next file
`raw/den-txt/guides/declare-hosts.txt`

---

## 2026-04-21 — Extract: `raw/den-txt/guides/declare-hosts.txt`

### Concepts found
- **standalone-home**: A `den.homes` entry — for machines without root access or HM-only setups; produces `homeConfigurations.<name>`; no host required
- **host-bound-home**: A standalone home named `"user@hostname"` — `home-manager` CLI auto-selects it when `whoami` and `hostname` match; hostname does not need to be a `den.hosts` entry
- **separate-host-user**: Pattern of declaring both `den.hosts.*.igloo.users.tux` and `den.homes.*."tux@igloo"` — allows independent activation and faster rebuilds since they don't need to be rebuilt together; host config readable from user via `osConfig`

### Relationships spotted
- standalone-home → den.homes: standalone homes are declared under den.homes
- standalone-home → home-context: standalone homes use the home-context pipeline
- host-bound-home → standalone-home: host-bound homes are a naming convention on standalone homes
- separate-host-user → standalone-home: the pattern uses a standalone home alongside a host-managed user
- separate-host-user → den._.mutual-provider: mutual-provider enables host-specific config in standalone homes

### Next file
`raw/den-txt/guides/home-manager.txt`

---

## 2026-04-21 — Extract: `raw/den-txt/guides/home-manager.txt`

### Concepts found
- **home-manager**: External tool — user environment manager for NixOS/Darwin; integrated into Den as the `homeManager` Nix class; opt-in per user via `classes`
- **hjem**: Lightweight alternative home environment manager — integrated as `hjem` Nix class; opt-in; requires `inputs.hjem`
- **nix-maid**: Another alternative user environment manager — integrated as `maid` Nix class; NixOS only; requires `inputs.nix-maid`
- **osConfig**: Module argument available in standalone home configurations — provides read access to the host's NixOS config from within the user's home config
- **home-manager.useGlobalPkgs**: HM option settable at the `hm-host` context stage — shares nixpkgs instance between OS and HM modules
- **home-manager.useUserPackages**: HM option settable at the `hm-host` context stage — installs HM packages via the OS package manager

### Relationships spotted
- home-manager → hm-host: the hm-host context imports the home-manager OS module
- home-manager → hm-user: the hm-user context forwards homeManager class to home-manager.users
- hjem → hjem-host: hjem integration activates the hjem-host context
- nix-maid → maid-host: nix-maid integration activates the maid-host context
- osConfig → separate-host-user: osConfig is available in the separate-host-user pattern
- home-manager.useGlobalPkgs → hm-host: useGlobalPkgs is configured at hm-host stage

### Next file
`raw/den-txt/guides/namespaces.txt`

---

## 2026-04-21 — Extract: `raw/den-txt/guides/namespaces.txt`

### Concepts found
- **local-namespace**: A namespace created with `inputs.den.namespace "name" false` — not exported to flake outputs; consumed internally only
- **exported-namespace**: A namespace created with `inputs.den.namespace "name" true` — exposed as `flake.denful.<name>`; consumable by downstream flakes
- **imported-namespace**: Merging aspects from upstream flakes — `inputs.den.namespace "shared" [ inputs.team-config ]`; merges `flake.denful.shared` from each source into local `den.ful.shared`
- **namespace-module-arg**: When a namespace is created, a module argument alias is automatically created — e.g. creating namespace `eg` makes `eg` available as a module arg pointing to `den.ful.eg`

### Relationships spotted
- local-namespace → namespaces: local namespace is one mode of the namespace system
- exported-namespace → namespaces: exported namespace is another mode
- exported-namespace → flake.denful: exported namespaces appear in flake.denful
- imported-namespace → namespaces: imported namespace merges upstream into local den.ful
- imported-namespace → dendrix: the import pattern is how dendrix distribution would work
- namespace-module-arg → namespaces: module arg alias is automatically created with namespace
- angle-bracket-syntax → namespace-module-arg: angle brackets reference namespaced aspects via the module arg

### Next file
`raw/den-txt/guides/batteries.txt`

---

## 2026-04-21 — Extract: `raw/den-txt/guides/batteries.txt`

### Concepts found
- **global-battery-pattern**: Applying batteries via `den.default.includes` — affects all hosts, users, and homes
- **per-aspect-battery-pattern**: Applying batteries in a specific aspect's `includes` — affects only that aspect's entities
- **battery-composition**: Batteries compose with regular aspects via `den.lib.parametric { includes = [...] }` — batteries are aspects and can be mixed freely

### Relationships spotted
- global-battery-pattern → den.default: global batteries go in den.default.includes
- per-aspect-battery-pattern → includes: per-aspect batteries go in a specific aspect's includes
- battery-composition → parametric-constructor: battery composition uses den.lib.parametric

### Next file
`raw/den-txt/guides/from-zero-to-den.txt`

---

## 2026-04-21 — Extract: `raw/den-txt/guides/from-zero-to-den.txt`

### Concepts found
- **noflake-template**: Den template for stable Nix without flakes — uses npins + import-tree + with-inputs + lib.evalModules; result of the "from zero to den" guide
- **den.flakeModule**: The flake module import that enables Den framework features — provides `den` module argument, all `den.*` options, and `nixosConfigurations` outputs
- **modules-directory**: The conventional `./modules/` directory in a Den project — loaded recursively by import-tree; any `.nix` file in it contributes to the config; no manual imports needed
- **underscore-prefix**: Convention for import-tree — directories/files starting with `_` are NOT auto-loaded; used to place non-dendritic legacy modules (`_nixos/`) alongside dendritic ones
- **default.nix-entrypoint**: The minimal Den entry point for noflake setups — imports npins sources, uses with-inputs for input management, calls `lib.evalModules` with import-tree, reads `.config.flake` for output compatibility

### Relationships spotted
- noflake-template → npins: noflake template uses npins for dependency pinning
- noflake-template → import-tree: noflake template uses import-tree to load modules
- noflake-template → with-inputs: noflake template uses with-inputs for input management
- den.flakeModule → den: importing den.flakeModule enables the den module argument
- modules-directory → import-tree: import-tree loads all .nix files from the modules directory
- underscore-prefix → import-tree: import-tree ignores underscore-prefixed directories
- default.nix-entrypoint → noflake-template: the entry point is the core of the noflake approach

### Next file
`raw/den-txt/tutorials/overview.txt`

---

## 2026-04-21 — Extract: `raw/den-txt/tutorials/overview.txt`

### Concepts found
- **minimal-template**: Smallest possible Den setup — one host, one user, no extra dependencies; flakes, no flake-parts, no HM
- **default-template**: Recommended starting point — flakes + flake-parts + Home Manager + VM testing + dendritic flake-file
- **example-template**: Feature showcase — namespaces, angle brackets, cross-platform NixOS + Darwin, providers
- **microvm-template**: Demonstrates Den extensibility — MicroVM virtualization as a custom context/class
- **nvf-standalone-template**: Demonstrates Den outside NixOS/Darwin — Den used with NixVim (NVF) standalone
- **flake-parts-modules-template**: Demonstrates integrating Den aspects with third-party flake-parts perSystem submodules
- **bogus-template**: Bug reproduction template — minimal isolated config for bug reports using nix-unit
- **ci-template**: Den's own test suite — comprehensive tests covering every Den feature; best learning resource
- **nix-unit**: Testing tool used by Den's CI/bogus template for unit testing Nix configurations

### Relationships spotted
- minimal-template → den: minimal template shows the bare minimum Den setup
- default-template → flake-parts: default template uses flake-parts
- default-template → home-manager: default template includes HM integration
- example-template → namespaces: example template showcases namespace usage
- microvm-template → custom-context: microvm uses a custom context to model VM guests
- nvf-standalone-template → library-only-usage: nvf template uses Den outside NixOS/Darwin
- ci-template → nix-unit: CI template uses nix-unit for testing
- bogus-template → nix-unit: bogus template uses nix-unit for bug reproduction

### Next file
`raw/den-txt/community/index.txt`

---

## 2026-04-21 — Extract: `raw/den-txt/community/index.txt`

### Concepts found
- **vic/vix**: Den author's personal infra — the reference real-world Den config; available on GitHub
- **dendritic-design**: The broader pattern/philosophy that inspired Den — "Flipping the Configuration Matrix" by Pol Dellaiera is a key reference; features over hosts

### Relationships spotted
- dendritic-design → dendritic: dendritic-design is the formal articulation of the dendritic pattern
- dendritic-design → feature-first: dendritic design is the same as feature-first

### Next file
`raw/den-txt/releases.txt`

---

## 2026-04-21 — Extract: `raw/den-txt/releases.txt`

### Concepts found
- **den-v0**: Den is currently at v0.x — not unstable (>180 CI tests per feature), but fast-paced and being shaped by users; not yet at a definitive form
- **den-main-branch**: Bleeding-edge — `github:vic/den`; PRs merged here; stable in the sense every PR passes CI; docs at den.oeiuwq.com
- **den-latest-tag**: Follows the latest created release tag — `github:vic/den/latest`; for users who want to move only between release points
- **den-standalone**: Den has no runtime dependencies — standalone library; `flakeModule` expects to be imported in a module providing nixpkgs lib; bundles its own aspect types originally based on flake-aspects but now optimized

### Relationships spotted
- den-v0 → den: Den is currently at the v0 series
- den-main-branch → den: main branch is the primary development target
- den-latest-tag → den: latest tag tracks the most recent release
- den-standalone → flake-aspects: Den originally based its aspect types on flake-aspects but has since optimized them independently

### Phase 1 complete — all raw files processed

**Total concepts extracted**: ~160
**Total relationships mapped**: ~140
**Ready for Phase 2: synthesize concept pages**

---

## 2026-04-21 — Synthesize: `aspect`

**Page created**: `wiki/aspect.md`
**Sources drawn from**: `index.txt`, `core-principles.txt`, `aspects/index.txt`, `parametric.txt`, `configure-aspects.txt`, `reference/aspects.txt`
**Pages linked to**: [[nix-class]], [[parametric-dispatch]], [[includes]], [[provides]], [[context-transformation-pipeline]], [[resolution]], [[den.aspects]], [[den.default]], [[aspect-dag]], [[aspect-fixed-point]], [[incremental-features]]
**Pages updated with backlink**: none yet — linked pages do not exist yet

### Next concept to synthesize
`parametric-dispatch` — second most foundational; required by aspect, context pipeline, batteries, and everything else

---

## 2026-04-21 — Synthesize: `parametric-dispatch`

**Page created**: `wiki/parametric-dispatch.md`
**Sources drawn from**: `explanation/parametric.txt`, `explanation/parametric/index.txt`, `explanation/aspects/index.txt`, `explanation/core-principles.txt`, `reference/aspects.txt`, `reference/ctx.txt`
**Pages linked to**: [[aspect]], [[__functor]], [[canTake]], [[parametric-constructor]], [[context-transformation-pipeline]], [[builtins.functionArgs]], [[includes]], [[den.lib]], [[deduplication]], [[den.ctx]]
**Pages updated with backlink**: `aspect.md` — already links to [[parametric-dispatch]]

### Next concept to synthesize
`nix-class` — foundational concept referenced by aspect, owned-config, resolution, context pipeline; needed before writing context-transformation-pipeline

---

## 2026-04-21 — Synthesize: `nix-class`

**Page created**: `wiki/nix-class.md`
**Sources drawn from**: `index.txt`, `core-principles.txt`, `aspects/index.txt`, `configure-aspects.txt`, `reference/schema.txt`, `explanation/library-vs-framework/index.txt`, `declare-hosts.txt`
**Pages linked to**: [[aspect]], [[owned-config]], [[resolution]], [[den.hosts]], [[context-transformation-pipeline]], [[den-as-library]], [[den.lib.aspects.resolve]]
**Pages updated with backlink**: `aspect.md` — already links to [[nix-class]]; `parametric-dispatch.md` — already references nix-class contextually

### Next concept to synthesize
`context-transformation-pipeline` — the pipeline that ties aspects, parametric dispatch, and nix-classes together into a working config; central to understanding how Den actually runs

---

## 2026-04-21 — Synthesize: `context-transformation-pipeline`

**Page created**: `wiki/context-transformation-pipeline.md`
**Sources drawn from**: `explanation/context-pipeline/index.txt`, `explanation/context-system/index.txt`, `explanation/core-principles.txt`, `reference/ctx.txt`, `index.txt`
**Pages linked to**: [[aspect]], [[nix-class]], [[parametric-dispatch]], [[context-stage]], [[den.ctx]], [[den.hosts]], [[den.homes]], [[resolution]], [[deduplication]], [[collectPairs]], [[dedupIncludes]], [[host.mainModule]], [[host.instantiate]], [[den.default]], [[deferredModule]]
**Pages updated with backlink**: `aspect.md` — already links to [[context-transformation-pipeline]]; `parametric-dispatch.md` — already links to [[context-transformation-pipeline]]; `nix-class.md` — already links to [[context-transformation-pipeline]]

### Next concept to synthesize
`den.ctx` — the declarative context type definitions that declare the pipeline structure; directly implements the pipeline using into/provides/includes

---

## 2026-04-21 — Synthesize: `den-schema`

**Page created**: `wiki/den-schema.md`
**Sources drawn from**: `reference/schema.txt`, `guides/declare-hosts.txt`, `guides/from-zero-to-den.txt`, `explanation/core-principles.txt`, `guides/configure-aspects.txt`
**Pages linked to**: [[context-transformation-pipeline]], [[aspect]], [[auto-generation]], [[nix-class]], [[freeform-type]], [[den.default]], [[batteries]], [[incremental-features]]
**Pages updated with backlink**: `context-transformation-pipeline.md` — already links to [[den.hosts]] and [[den.homes]]; `aspect.md` — already links to [[den.aspects]]

### Next concept to synthesize
`batteries` — den.provides.* built-in reusable aspects; covers all individual batteries, usage patterns, and how batteries register context stages

---

## 2026-04-21 — Synthesize: `batteries`

**Page created**: `wiki/batteries.md`
**Sources drawn from**: `reference/batteries.txt`, `guides/batteries.txt`, `explanation/context-pipeline/index.txt`, `reference/ctx.txt`, `guides/home-manager.txt`
**Pages linked to**: [[aspect]], [[parametric-dispatch]], [[context-transformation-pipeline]], [[den-ctx]], [[den-schema]], [[includes]], [[den.default]], [[home-environments]]
**Pages updated with backlink**: `aspect.md` — already links to [[den.provides]]; `context-transformation-pipeline.md` — already links to [[batteries]]; `den-ctx.md` — already links to [[batteries]]

### Next concept to synthesize
`namespaces` — den.ful, local/exported/imported namespaces, angle-bracket syntax, dendrix relationship

---

## 2026-04-21 — Synthesize: `namespaces`

**Page created**: `wiki/namespaces.md`
**Sources drawn from**: `guides/namespaces.txt`, `reference/aspects.txt`, `motivation/index.txt`, `community/index.txt`
**Pages linked to**: [[aspect]], [[den.aspects]], [[angle-bracket-syntax]], [[flake.denful]], [[dendrix]], [[denful]], [[batteries]]
**Pages updated with backlink**: `aspect.md` — already links to [[namespaces]] contextually; `batteries.md` — already links to [[namespaces]] contextually

### Next concept to synthesize
`home-environments` — home-manager, hjem, nix-maid integration; standalone homes; host-bound homes; separate host+user pattern; osConfig

---

## 2026-04-21 — Synthesize: `home-environments`

**Page created**: `wiki/home-environments.md`
**Sources drawn from**: `guides/home-manager.txt`, `reference/batteries.txt`, `explanation/context-pipeline/index.txt`, `guides/declare-hosts.txt`, `reference/ctx.txt`
**Pages linked to**: [[aspect]], [[nix-class]], [[context-transformation-pipeline]], [[den-ctx]], [[batteries]], [[den-schema]], [[parametric-dispatch]]
**Pages updated with backlink**: `batteries.md` — already links to [[home-environments]]; `context-transformation-pipeline.md` — already links to hm-host/hm-user contexts; `den-schema.md` — already links to user.classes

### Next concept to synthesize
`den-lib` — the domain-agnostic core library; parametric variants, canTake, take, statics, owned, isFn, __findFile, aspects API; when to use library-only vs framework

---

## 2026-04-21 — Synthesize: `den-lib`

**Page created**: `wiki/den-lib.md`
**Sources drawn from**: `explanation/library-vs-framework/index.txt`, `explanation/parametric.txt`, `explanation/aspects/index.txt`, `explanation/effects/index.txt`, `reference/aspects.txt`
**Pages linked to**: [[parametric-dispatch]], [[aspect]], [[resolution]], [[den-as-library]], [[context-transformation-pipeline]], [[angle-bracket-syntax]], [[algebraic-effects]], [[deduplication]], [[den-ctx]]
**Pages updated with backlink**: `parametric-dispatch.md` — already links to [[den.lib]]; `aspect.md` — already links to [[parametric-dispatch|den.lib.parametric]]

### Next concept to synthesize
`den-motivation` — why Den was built, the history from flake-parts through flake-aspects to Den, the social goal, dendrix, denful; covers the ecosystem tools (import-tree, with-inputs, npins, flake-aspects)

---

## 2026-04-21 — Synthesize: `den-motivation`

**Page created**: `wiki/den-motivation.md`
**Sources drawn from**: `motivation/index.txt`, `releases.txt`, `community/index.txt`, `index.txt`
**Pages linked to**: [[aspect]], [[parametric-dispatch]], [[namespaces]], [[den-lib]], [[flake-aspects]], [[dendrix]], [[denful]], [[import-tree]], [[with-inputs]], [[npins]], [[flake-file]], [[context-transformation-pipeline]]
**Pages updated with backlink**: `aspect.md` — already links contextually; `namespaces.md` — already links to [[dendrix]] and [[denful]]; `den-lib.md` — already links to [[flake-aspects]]

### Next concept to synthesize
`algebraic-effects` — den.lib.fx, effectful computations, handlers, fx.bind.fn, the connection to parametric dispatch, Den's migration to nix-effects

---

## 2026-04-21 — Synthesize: `algebraic-effects`

**Page created**: `wiki/algebraic-effects.md`
**Sources drawn from**: `explanation/effects/index.txt`, `explanation/parametric.txt`, `releases.txt`
**Pages linked to**: [[parametric-dispatch]], [[den-lib]], [[aspect]], [[context-transformation-pipeline]], [[builtins.functionArgs]]
**Pages updated with backlink**: `den-lib.md` — already links to [[algebraic-effects]]; `parametric-dispatch.md` — already links to [[context-transformation-pipeline]]

### Next concept to synthesize
`resolution` — how den.lib.aspects.resolve works; collectPairs, dedupIncludes, deferredModule output; the full resolution order

---

## 2026-04-21 — Synthesize: `resolution`

**Page created**: `wiki/resolution.md`
**Sources drawn from**: `explanation/aspects/index.txt`, `explanation/context-pipeline/index.txt`, `explanation/context-system/index.txt`, `reference/aspects.txt`, `reference/schema.txt`
**Pages linked to**: [[aspect]], [[aspect-dag]], [[parametric-dispatch]], [[nix-class]], [[deferredModule]], [[den-lib]], [[context-transformation-pipeline]], [[deduplication]], [[collectPairs]], [[dedupIncludes]], [[meta.adapter]]
**Pages updated with backlink**: `aspect.md` — already links to [[resolution]]; `den-lib.md` — already links to [[resolution]]; `context-transformation-pipeline.md` — already links to [[collectPairs]] and [[dedupIncludes]]

### Next concept to synthesize
`den-as-library` — using den.lib without the framework; custom Nix domains (Terranix, NixVim); when library-only is the right choice; the three-step pattern

---

## 2026-04-21 — Synthesize: `den-as-library`

**Page created**: `wiki/den-as-library.md`
**Sources drawn from**: `explanation/library-vs-framework/index.txt`, `tutorials/overview.txt`, `explanation/parametric.txt`, `reference/aspects.txt`
**Pages linked to**: [[den-lib]], [[aspect]], [[parametric-dispatch]], [[resolution]], [[nix-class]], [[context-transformation-pipeline]], [[namespaces]]
**Pages updated with backlink**: `den-lib.md` — already links to [[den-as-library]]; `resolution.md` — already links contextually; `nix-class.md` — already links to [[den-as-library]]

### Phase 2 status
Core concept pages complete. 13 pages written covering all major Den concepts.
Remaining concepts from Phase 1 log that could become stub or small pages:
- `aspect-dag` — referenced by aspect and resolution; short page on DAG structure
- `deduplication` — referenced by pipeline and resolution; short page on the problem/solution
- `dendritic` — the pattern/philosophy; could be covered inline in den-motivation
- `incremental-features` — short concept; likely covered sufficiently in aspect.md
- `feature-first` — design principle; covered in context within den-motivation and aspect.md
These will be written as questions arise in Phase 3, not proactively.

---

## 2026-04-21 — Synthesize: `den-ctx`

**Page created**: `wiki/den-ctx.md`
**Sources drawn from**: `reference/ctx.txt`, `explanation/context-system/index.txt`, `explanation/context-pipeline/index.txt`, `explanation/core-principles.txt`
**Pages linke.d to**: [[context-transformation-pipeline]], [[context-stage]], [[aspect]], [[parametric-dispatch]], [[batteries]], [[custom-context]], [[meta.adapter]], [[den.ctx.into..]], [[den.ctx.provides]]
**Pages updated with backlink**: `context-transformation-pipeline.md` — already links to [[den.ctx]]

### Next concept to synthesize
`den-schema` — covers den.schema, den.hosts, den.homes, freeform-type, host.class, user.classes; the declaration layer that seeds the pipeline

---

## 2026-04-21 — Grow: `provides` and `auto-generation` clarifications

**Question asked**: Does `provides` get included when you include the parent aspect? And why does Den auto-create aspects for hosts/users?

**Pages updated**:
- `aspect.md` — expanded `provides` section to make explicit that provides sub-aspects are NOT included when including the parent; added before/after examples showing the three inclusion options. Expanded `auto-generation` section with the "why" — the auto-created aspect is a shared hook point that lets any file contribute to a host/user's config without a central import list; added multi-file example showing three files contributing to the same aspect; clarified the existence vs behavior separation.

---

## 2026-04-25 — Grow: `namespaces` page rewrite

**Question asked**: What are namespaces, how do they work, and do you really need them? Also: with 10 machines and 20 users, isn't den.aspects too crowded — shouldn't you split into folders like traditional configs?

**Pages updated**:
- `namespaces.md` — major rewrite. Added "The problem namespaces solve" section framing the motivation before any syntax. Added full step-by-step walkthrough (create namespace, populate it, use from hosts). Added mental model diagram showing den.aspects vs den.ful. Added "But won't den.aspects get crowded?" section explaining that den.aspects is flat in the module system but not in the filesystem — folders organize for humans, Den auto-imports everything. Added "Do you even need namespaces?" section with honest guidance that single-user configs may not need them. Added quick reference table. Preserved all existing technical detail (three modes, architecture, angle brackets, dendrix/denful relationship, gotchas, ecosystem context).

---

## 2026-04-26 — Grow: Clarifying questions (parametric dispatch, batteries, aspect vs module, HM modes)

**Questions asked**:
1. Detailed explanation of "You don't need mkIf, lib.mkMerge, or manual specialArgs threading"
2. What define-user, primary-user, user-shell do on each platform
3. "Home Manager as a NixOS module" — what that means in Den
4. Is a user aspect a module?

**Pages updated**:
- `parametric-dispatch.md` — expanded "In the Nix ecosystem" section with concrete before/after comparisons: `mkIf` vs parametric function guards, `mkMerge` vs aspect `includes`, `specialArgs`/`extraSpecialArgs` vs context args + `inputs'` battery. Added summary table.
- `aspect.md` — added new "Aspect vs module" section with structure diagram showing aspect as container and `nixos`/`homeManager`/`darwin` as payload modules. Added compare block showing traditional standalone HM module vs Den equivalent.
- `batteries.md` — expanded `define-user`, `primary-user`, and `user-shell` sections with platform effect tables (NixOS/Darwin/HM/WSL). Added "Why den.default.includes works everywhere" subsection explaining `parametric.exactly` with simplified internal implementation.
- `home-environments.md` — added "Home Manager as NixOS module vs standalone" comparison table and expanded explanation of what Den does in each mode (imports HM module automatically, forwards `homeManager` class content, produces `homeConfigurations`).

---

## 2026-04-26 — Synthesize: `vm-testing`

**Page created**: `wiki/vm-testing.md`
**Sources drawn from**: `raw/den-txt/tutorials/default/index.txt`, `raw/den-txt/guides/migrate/index.txt`, `raw/den-txt/tutorials/overview/index.txt`
**Pages linked to**: [[context-transformation-pipeline]], [[den-schema]], [[batteries]], [[home-environments]], [[nix-class]], [[resolution]], [[host.mainModule]], [[host.instantiate]], [[den.hosts]]
**Pages updated with backlink**: `index.md` — added vm-testing entry under Den Framework; `home-environments.md` — added VM testing row to HM comparison table and related-pages link

### Concepts covered
- **Multi-host VM testing**: Generating `.#vm-<hostname>` per `den.hosts` entry using `lib.mapAttrs` in `perSystem`, instead of the single-host `.#vm` from the default template. Added line-by-line comments in both single-host and multi-host snippets.
- **Identical pipeline guarantee**: The VM runs the exact same resolution + context pipeline as the real host, unlike traditional separate `vm.nix` test configs
- **Workflow**: edit → `nix run .#vm-<host>` → verify → `nixos-rebuild switch --flake .#<host>` → rotate to next host
- **VM-specific batteries**: `tty-autologin` for instant user-shell access, `hostname` for script name matching
- **Hardware-gated aspects**: Using parametric dispatch to skip GPU/bluetooth/firmware config in QEMU
- **Standalone homes limitation**: `den.homes` cannot use `system.build.vm`; must test via `home-manager build`

---

## 2026-04-26 — Synthesize: `project-structure`

**Page created**: `wiki/project-structure.md`
**Sources drawn from**: `raw/den-txt/guides/declare-hosts/index.txt`, `raw/den-txt/guides/configure-aspects/index.txt`, `raw/den-txt/tutorials/default/index.txt`, `raw/den-txt/tutorials/example/index.txt`, `raw/den-txt/guides/migrate/index.txt`
**Pages linked to**: [[aspect]], [[namespaces]], [[den-schema]], [[import-tree]], [[den.hosts]], [[den.aspects]], [[den.default]], [[den-ctx]], [[vm-testing]], [[batteries]]
**Pages updated with backlink**: `index.md` — added `project-structure` under Den Framework

### Concepts covered
- **Folders vs namespaces distinction**: folders organize files for humans; namespaces organize aspects for Den
- **Two-bucket mental model**: `den.aspects` = concrete machines/users; `den.ful.<ns>` = reusable feature library
- **Recommended directory layout**: dendritic.nix, namespace.nix, defaults.nix, vm.nix, hosts/, users/, features/, den.nix
- **File-by-file explanation**: what each file contains and why
- **Common mistakes**: nesting aspects inside hosts, mixing declarations with behavior, creating namespaces per folder
- **Multi-user multi-host scaling**: folder nesting for editors/git, flat evaluation for Den
- **Namespace-to-files connection**: Added explicit step-by-step section showing how `namespace.nix` creates the bag, `features/*.nix` drops aspects into it, and `hosts/*.nix` consumes it — addressing the common confusion that directory structure should mirror namespaces
- **Why not plain attrsets**: Added comparison table (`features.bluetooth` vs `my.bluetooth`) explaining that namespaces are Den's type system for reusable libraries, not just grouping — plain attrsets lose aspect machinery, publishing, angle brackets, and ecosystem conventions
- **Namespace naming convention**: Added "Why `my`? Can I name it `features`?" section — explains namespace names describe scope/ownership (`my`, `infra`, `shared`), not content type (`features`), and why matching namespace to folder names creates semantic drift when a namespace receives contributions from multiple folders
- **Example template `eg` caveat**: Explicitly acknowledged that the example template uses `eg` as both namespace and folder name, and explained this is a **teaching choice for clarity** — not a structural recommendation for real configs

### Next concept to synthesize
`mutual-provider` — already completed below.

---

## 2026-04-26 — Synthesize: `mutual-provider`

**Page created**: `wiki/mutual-provider.md`
**Sources drawn from**: `guides/mutual/index.txt`, `reference/batteries/index.txt`, `reference/ctx/index.txt`, `tutorials/ci/index.txt`
**Pages linked to**: [[batteries]], [[den-ctx]], [[context-transformation-pipeline]], [[aspect]], [[home-environments]]
**Pages updated with backlink**:
- `batteries.md` — trimmed the `mutual-provider` section to a brief summary + link to the new page; added [[mutual-provider]] to Relates to
- `den-ctx.md` — added inline link to [[mutual-provider]] in the `includes` section where `den._.mutual-provider` is shown as the example
- `home-environments.md` — added inline wiki-link to [[mutual-provider]] in the standalone-homes section
- `index.md` — added `mutual-provider` as a new top-level entry under Den Framework

### Concepts covered
- **mutual-provider battery**: What it is, why it is opt-in, and the exact mechanism (context-stage providers scanning `.provides.*` during the `{ host, user }` context)
- **Four direction patterns**: User→specific host (`provides.<hostname>`), Host→specific user (`provides.<username>`), User→all hosts (`provides.to-hosts`), Host→all users (`provides.to-users`)
- **User peers**: One user configuring another user on the same machine via `provides.<username>` and `provides.to-users`
- **Standalone homes**: Using `den.ctx.home.includes` for host-specific home config on non-NixOS machines
- **Complete concrete example**: A three-entity setup (laptop, alice, bob) showing all four directions and the final merged result table
- **Under the hood**: How the battery hooks into `den.ctx.user._` as additional providers, and why `.provides.` is silently ignored without the include
- **Global vs scoped placement**: Why `den.ctx.user.includes` is the idiomatic pattern; why per-aspect `includes` is not the supported hook
- **Gotchas**: Must-include requirement, class forwarding rules, no-circular-dependency guarantee, `.provides.` being plain nested attributes rather than special syntax

---

## 2026-04-28 — Extract: `raw/den-example-template/`

### Files read
- `flake.nix` — auto-generated by flake-file; dendritic pattern
- `modules/den.nix` — host/home declarations (igloo, apple, alice); `den.schema.user.classes`
- `modules/dendritic.nix` — wires `flake-file.flakeModules.dendritic` and `den.flakeModules.dendritic`
- `modules/inputs.nix` — per-file flake inputs (home-manager, darwin, optional WSL)
- `modules/namespace.nix` — creates `eg` namespace, enables `__findFile` for angle brackets
- `modules/nh.nix` — `den.lib.nh.denPackages` exposing per-host flake apps
- `modules/tests.nix` — CI checks verifying mutual-provider wiring (`nh`, `tmux`, `emacs-nox`)
- `modules/vm.nix` — `eg.vm.provides.gui` includes for VM, `writeShellApplication` wrapper
- `modules/aspects/defaults.nix` — `den.default` state versions, global includes with `<den/...>` batteries, `perHost`/`perUser`/`perHome` helpers
- `modules/aspects/alice.nix` — user aspect with `includes` (customEmacs, cooper, setHost, eg.autologin, batteries), `nixos`, `homeManager`, `provides.igloo`, context-aware `cooper` and `setHost`
- `modules/aspects/igloo.nix` — host aspect with system packages + `provides.alice`
- `modules/aspects/hasAspect-examples.nix` — **major new pedagogical file**: demonstrates `host.hasAspect` vs `meta.adapter` / `oneOfAspects`, and the anti-pattern of using `hasAspect` in `includes`
- `modules/aspects/eg/autologin.nix` — context-aware parametric aspect for display-manager autologin
- `modules/aspects/eg/ci-no-boot.nix` — CI-only aspect disabling boot
- `modules/aspects/eg/vm.nix` — namespace aspect combining VM sub-features via `provides.gui` / `.tui`
- `modules/aspects/eg/vm-bootable.nix` — installer CD modules for VM bootable images
- `modules/aspects/eg/xfce-desktop.nix` — XFCE desktop configuration
- `.github/workflows/test.yml` — GitHub Actions running `nix flake check` on Ubuntu + macOS, injecting `_module.args.CI = true`

### Concepts found
- **hasAspect**: Predicate on resolved host/user/home objects (`host.hasAspect`) testing whether an aspect appears in the entity's resolved tree; usable only inside class-config module bodies (e.g. `nixos = ...`) because the tree is frozen by evalModules time
- **oneOfAspects**: `den.lib.aspects.adapters.oneOfAspects` — a `meta.adapter` that keeps the first present aspect from a list and tombstones the rest; used for "prefer A, fallback to B" structural decisions
- **den.lib.aspects.adapters**: Library namespace providing `oneOfAspects`, `excludeAspect`, `substituteAspect`, `filter`, `filterIncludes` — all operate during the resolve tree walk
- **meta.adapter (practical)**: Field on aspect or context controlling which aspects from the `includes` DAG are resolved transitively; operates ON the tree during resolution, cycle-safe
- **hasAspect-includes-antipattern**: Using `host.hasAspect` inside an `includes` list causes infinite recursion because `resolved` depends on `includes`, and `includes` depends on `resolved`
- **context-aware-user-description**: `cooper` pattern — aspect emitting config anytime `{ user }` context exists, via parametric dispatch
- **den.lib.perHost**: Helper wrapping a function to only match `{host}` contexts exactly
- **den.lib.perUser**: Helper wrapping a function to only match `{host, user}` contexts exactly
- **den.lib.perHome**: Helper wrapping a function to only match `{home}` contexts exactly
- **den.default**: Special global aspect applied to every host, user, and home; holds state versions and global includes
- **den.schema.user.classes**: `lib.mkDefault [ "homeManager" ]` to opt-in all users to Home Manager
- **flake-file**: Tool auto-generating `flake.nix` from per-module input declarations
- **dendritic-flake-pattern**: `modules/dendritic.nix` imports both `flake-file` and `den` dendritic modules
- **denPackages**: `den.lib.nh.denPackages` generating per-host flake apps for `nh` building
- **checkCond**: Pattern in `tests.nix` for writing conditional `nix flake check` assertions
- **vm-installer-cd**: Using `modulesPath + "/installer/cd-dvd/installation-cd-${variant}.nix"` to create bootable VM images
- **autologin-context-aware**: Parametric aspect that only activates when `{ user }` is present, wiring `services.displayManager.autoLogin.user`
- **primary-user-battery**: `<den/primary-user>` used in alice.nix — marks alice as admin always
- **user-shell-battery**: `<den/user-shell>` used in alice.nix — sets default shell to fish
- **mutual-provider-practical**: `alice.provides.igloo` enables `programs.nh`, `igloo.provides.alice` enables `programs.tmux`

### Relationships spotted
- hasAspect → host.resolved: hasAspect queries the resolved aspect tree
- hasAspect → evalModules: safe because class body runs at evalModules time, after tree resolution
- hasAspect → meta.adapter: complementary tools — hasAspect reads structure, meta.adapter writes structure
- oneOfAspects → meta.adapter: oneOfAspects is a specific meta.adapter implementation
- oneOfAspects → den.lib.aspects.adapters: part of the adapters library
- hasAspect-includes-antipattern → hasAspect: the anti-pattern is a misuse of hasAspect
- den.lib.perHost → parametric-dispatch: helpers for exact context matching
- den.lib.perUser → parametric-dispatch: helpers for exact context matching
- den.lib.perHome → parametric-dispatch: helpers for exact context matching
- context-aware-user-description → parametric-dispatch: uses parametric dispatch
- den.default → aspect: den.default is a special global aspect
- flake-file → dendritic: flake-file generates flake.nix; dendritic is the pattern
- denPackages → den.lib.nh: denPackages is part of den.lib.nh
- denPackages → perSystem-outputs: generates per-system flake packages

### Discrepancies with existing wiki pages
- `example-template.md` references `eg/routes.nix` and `<eg/routes>` which do not exist in the raw template
- `example-template.md` shows `eg.vm-bootable._.gui` but raw uses `eg.vm-bootable.provides.gui`
- `example-template.md` mutual provider examples use `programs.helix.enable` but raw uses `programs.tmux.enable` and `programs.nh.enable`
- `example-template.md` does not cover `hasAspect-examples.nix`, `dendritic.nix`, `inputs.nix`, `nh.nix`, or context-aware aspects (`cooper`, `autologin`, `setHost`)

### Next steps
Update `example-template.md` to match raw files; synthesize `hasAspect` and `meta-adapter` concept pages.