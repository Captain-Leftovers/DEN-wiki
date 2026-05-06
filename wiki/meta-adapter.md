# meta.adapter

**What it is**: A function field on a Den context type or aspect that filters or transforms the list of aspects during the resolution tree walk, controlling which aspects from the `includes` DAG are actually included in the resolved output.

**Why it exists**: To make structural decisions about the aspect tree ("which aspects should be in the tree?") at resolution time, with full visibility of the tree shape, without causing the infinite recursion that would result from using `hasAspect` inside `includes`.

**Relates to**: [[aspect]], [[resolution]], [[hasAspect]], [[den-ctx]], [[den.lib.aspects.adapters]], [[meta.provider]], [[context-transformation-pipeline]]

---

## Core idea

During resolution, Den walks the `includes` DAG recursively. At each node, if a `meta.adapter` is present, it is applied to the list of aspects that the node wants to include. The adapter can keep, discard, reorder, or replace aspects. The result becomes the effective includes list for that subtree.

```nix
den.aspects.example-secrets-bundle = {
  includes = [
    den.aspects.example-agenix-rekey
    den.aspects.example-sops-nix
  ];
  meta.adapter = den.lib.aspects.adapters.oneOfAspects [
    den.aspects.example-agenix-rekey  # preferred when present
    den.aspects.example-sops-nix      # fallback
  ];
};
```

`oneOfAspects` keeps the first aspect from the list that is actually present in the resolved tree and tombstones the rest. This lets you declare "try agenix-rekey, but if it's not available, fall back to sops-nix" without any conditional `includes` logic.

---

## Details

### Where adapters live

Adapters can be set on **context types** or on **individual aspects**:

```nix
# Context adapter — applies to ALL hosts
den.ctx.host.meta.adapter = inherited:

# Aspect adapter — applies only to this aspect's resolution subtree
den.aspects.foo.meta.adapter = inherited:
```

Context adapters are **outermost** (applied first). Aspect adapters are **innermost** (applied last, closest to the leaf). They compose transitively: a context adapter affects every aspect resolved within that context, and an aspect adapter affects that aspect and everything it includes.

### Standard adapter helpers

Den provides a library of adapter constructors in `den.lib.aspects.adapters`:

| Helper | What it does |
|--------|--------------|
| `oneOfAspects [ a b c ]` | Keep the first present aspect; tombstone the rest |
| `excludeAspect <ref>` | Tombstone a specific aspect by reference |
| `substituteAspect <old> <new>` | Swap one aspect for another |
| `filter predicate` | Custom predicate-based filtering |
| `filterIncludes predicate` | Filter only the `includes` list, leaving owned configs intact |

All of these operate during the **resolve tree walk**, before evalModules begins.

### OneOfAspects in depth

`oneOfAspects` is the most common adapter. It solves the "prefer A, fallback to B" problem:

```nix
den.aspects.example-secrets-bundle = {
  includes = [
    den.aspects.example-agenix-rekey
    den.aspects.example-sops-nix
  ];
  meta.adapter = den.lib.aspects.adapters.oneOfAspects [
    den.aspects.example-agenix-rekey
    den.aspects.example-sops-nix
  ];
};
```

Both candidates remain in the `includes` list at definition time. `oneOfAspects` tombstones the loser **during resolution** rather than at module-definition time. This is safe because the adapter runs with full structural visibility and operates on the tree from the outside.

### Composing adapters

Adapters compose naturally because each is just a function `aspectList → aspectList`. You can write your own:

```nix
meta.adapter = aspectList:
  lib.filter (a: a != den.aspects.some-debug-thing) aspectList;
```

Or combine helpers:

```nix
meta.adapter = den.lib.composeExtensions
  (den.lib.aspects.adapters.excludeAspect den.aspects.debug)
  (den.lib.aspects.adapters.oneOfAspects [ den.aspects.foo den.aspects.bar ]);
```

---

## Gotchas

### Do not read from the tree you are writing

`meta.adapter` decides structure. `hasAspect` reads structure. Mixing them in the same direction causes cycles:

- **OK**: `meta.adapter` (write) → resolution → `host.hasAspect` (read) inside `nixos = ...`
- **BROKEN**: `host.hasAspect` (read) → `includes` list (write) → resolution → `host.resolved`

See [[hasAspect]] for the anti-pattern and why it breaks.

### Adapters are silent

A tombstoned aspect does not produce an error or warning; it simply disappears from the resolved tree. If you expected an aspect to be present and it is not, check whether an upstream adapter filtered it out.

### Context adapters are broad

Setting `den.ctx.host.meta.adapter` affects **every** host in the configuration. If you need per-host filtering, set the adapter on the host's primary aspect instead:

```nix
den.aspects.igloo.meta.adapter = ...;
```

---

## In the Nix ecosystem

`meta.adapter` is Den's mechanism for feature selection and conflict resolution. It replaces manual `mkIf cfg.enable` toggles for structural aspect relationships. Where traditional NixOS modules use boolean options to turn features on and off, Den uses structural presence in the aspect tree — and `meta.adapter` is the safe, cycle-free way to control that presence.
