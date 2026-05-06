# hasAspect

**What it is**: A predicate on resolved host, user, or home objects (`host.hasAspect`, `user.hasAspect`, `home.hasAspect`) that tests whether a specific aspect reference appears in that entity's fully resolved aspect tree.

**Why it exists**: To let aspect class-config modules branch on the structural presence of companion aspects at evalModules time — after the aspect DAG has been resolved and frozen — without causing infinite recursion.

**Relates to**: [[aspect]], [[resolution]], [[meta.adapter]], [[host.resolved]], [[context-transformation-pipeline]], [[parametric-dispatch]]

---

## Core idea

Once Den has resolved a host's aspect tree, the resulting `host.resolved` object exposes `.hasAspect <ref>` — a boolean that answers "was this aspect included in my tree?"

```nix
den.aspects.example-impermanence = { host, ... }: {
  nixos = { lib, ... }:
    lib.mkMerge [
      (lib.mkIf (host.hasAspect den.aspects.example-zfs-root) {
        environment.etc."impermanence-backend".text = "zfs";
      })
      (lib.mkIf (host.hasAspect den.aspects.example-btrfs-root) {
        environment.etc."impermanence-backend".text = "btrfs";
      })
    ];
};
```

Because `host.hasAspect` reads from the **frozen** resolved tree, and the `nixos = ...` body runs during `evalModules`, there is no cycle. The tree shape is already known; the body only consumes it.

---

## Details

### Where it works

`hasAspect` is available on the context objects passed to parametric aspect functions:

| Object | Available in |
|--------|--------------|
| `host.hasAspect` | `{ host, ... }:` parametric functions, and inside host class bodies |
| `user.hasAspect` | `{ host, user, ... }:` parametric functions, and inside user class bodies |
| `home.hasAspect` | `{ home, ... }:` parametric functions, and inside home class bodies |

### What it reads

`host.hasAspect` tests membership in the **resolved** aspect tree — the transitive closure of `includes` after `meta.adapter` filters have been applied. It does **not** test whether the aspect was declared in the original `den.aspects` attrset; it tests whether it survived resolution.

### Companion pattern: meta.adapter

`hasAspect` and `meta.adapter` are complementary:

| Tool | Question it answers | When it runs |
|------|---------------------|--------------|
| `hasAspect` | "Is aspect X in my resolved tree?" | evalModules time (read) |
| `meta.adapter` | "Should aspect Y be in the tree?" | Resolution time (write) |

Use `hasAspect` when you want to **react** to the presence of another aspect. Use `meta.adapter` (e.g. `oneOfAspects`) when you want to **decide** which aspects are present.

---

## Gotchas

### Anti-pattern: hasAspect in includes

The most common mistake is using `hasAspect` inside an `includes` list to decide what to pull in:

```nix
# BROKEN — infinite recursion
den.aspects.broken = { host, ... }: {
  includes = if host.hasAspect den.aspects.example-zfs-root
    then [ den.aspects.example-zfs-impermanence ]
    else [ den.aspects.example-btrfs-impermanence ];
};
```

Why it cycles:

1. `host.resolved` is computed by walking `includes` and applying `meta.adapter`.
2. `broken`'s `includes` depends on `host.hasAspect`.
3. `host.hasAspect` depends on `host.resolved`.
4. `host.resolved` depends on `broken`'s `includes`.

That is a back-edge into the tree from a position the tree itself has to read first to know its own shape — a classic fixed-point with no fixed point. Nix reports infinite recursion.

**Fix**: Use `meta.adapter` instead. See [[meta.adapter]].

### hasAspect does not work in top-level aspect definitions

`hasAspect` is only available on resolved context objects. You cannot call it in a plain module body outside of a parametric function or class-config module:

```nix
# BROKEN — host is not in scope here
{
  den.aspects.foo.includes = if host.hasAspect ... then ... else ...;
}
```

### Aspect references must be exact

`hasAspect` expects the actual aspect attrset reference (`den.aspects.foo`), not a string name. Passing a string will fail or return false.

---

## In the Nix ecosystem

`hasAspect` replaces the traditional `cfg.enable` toggle pattern. Instead of checking whether a user set `myModule.enable = true`, you check whether the aspect that **owns** that concern is structurally present in the host's tree. This makes dependencies explicit and traceable.

In the NixOS module system, this is analogous to asserting `config.services.postgresql.enable` from another module — but at the Den aspect layer, where the "modules" are aspects and the "options" are structural presence.
