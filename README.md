# Kura

**Reproducible builds, immutable packages, and atomic upgrades for Omarchy.**

Kura replaces the Arch/pacman foundation under Omarchy with a functional substrate that Omarchy owns: an immutable store, sandboxed reproducible builds, atomic system generations, rollback, and our own packages, build farm, and releases. Linux, systemd, UWSM, Hyprland, and Quickshell are unchanged.

Steel Scheme describes packages and systems. Rust owns their semantics and executes them. A typed, canonical IR sits between the two.

---

# RFC-0001: Kura

**Scope:** Package management, builds, system composition, deployment, updates, rollback, binary distribution
**Languages:** Rust, Steel Scheme

## 1. Summary

Kura takes the ideas that Guix and Nix have proven—immutable package outputs, sandboxed reproducible builds, hashed system closures, atomic generations, rollback, garbage collection, binary substitutes, declarative system composition—and builds only what Omarchy needs, without inheriting anyone else's politics, governance, language, or compatibility obligations.

It does not adopt Guix System, Shepherd, GNU package policy, `/gnu/store`, or Nix's language, store layout, or flakes.

This is not a general-purpose distribution framework. It is a foundation that can change as fast as Omarchy does.

## 2. Motivation

Omarchy owns the user experience but inherits its system model from Arch. An update coordinates mutable package transactions, ALPM hooks, AUR packages, migrations, Snapper snapshots, conflicts, restart state, config changes, hardware packages, and systemd services. Most of that exists because packages mutate a shared filesystem in place.

Snapper gives you a way back, but only at the filesystem level. It has no notion of a coherent system, and a snapshot can't be built or tested before you're already running it.

Kura builds the next system separately:

```
system definition → build graph → immutable outputs → generation N+1 → atomic activation
```

Generation N is untouched. Rollback means switching back to it.

## 3. Why not Guix or Nix?

**Guix** proves the architecture and is the closest reference. It is also a GNU project: deblobbed kernel, no proprietary firmware or drivers, no vendor binaries, Shepherd instead of systemd. Omarchy needs NVIDIA, firmware, Steam, and whatever else users run. Guix is prior art. We reuse what's legally reusable and depend on nothing upstream.

**Nix** has the same ideas and a bigger ecosystem, and we're still not using it. The language is an abomination: untyped, lazy, with error messages that point nowhere. The non-FHS store fights every proprietary binary. Flakes are still experimental after five years. And the project has spent two years in political fights that already produced two forks, Lix and Snix. Building on Nix means inheriting the language, the store, and the politics.

Kura keeps the functional model and drops everything else.

## 4. Architecture

```
Steel Scheme          packages, systems, transforms
      ↓
Rust semantic model   typed constructors, validation, canonicalization
      ↓
Kura IR               versioned, deterministic, hashable, signable
      ↓
Rust engine           build, store, deploy
```

### Steel Scheme

Steel is a Rust-native embeddable Scheme, already used as the extension language for the Helix editor. It is the programmable surface for package definitions, variants, dependency composition, platform selection, system descriptions, services, hardware profiles, and local overrides.

```scheme
(define-package hyprland
  (version "0.52.0")
  (source (github "hyprwm" "Hyprland" #:revision version #:hash "..."))
  (depends wayland libinput pixman)
  (build (cmake)))
```

Scheme because package composition needs macros, higher-order functions, and a REPL. Steel because it's a Rust crate: no FFI boundary, it embeds in the `kura` binary, and we can fork it. It is young and nobody has run it at nixpkgs scale, but evaluation happens on repository infrastructure and is cached as IR, so evaluator speed is not on the user's critical path.

### Rust semantic model

```rust
struct Package {
    name: PackageName,
    version: Version,
    source: Source,
    inputs: Vec<Dependency>,
    outputs: Vec<Output>,
    build: BuildPlan,
}
```

Steel calls Rust-backed constructors rather than handing untyped records to the engine. Invalid data fails at the language boundary. Rust is the semantic authority even though Scheme is dynamic.

### Kura IR

Validated Rust objects canonicalize into Kura IR: packages, sources, dependencies, build plans, services, systems, images, deployment state. Deterministic, serializable, hashable, inspectable, independent of Steel.

Repository infrastructure evaluates Scheme into signed IR ahead of time. Clients consume the signed index without evaluating the repository. Steel is still in the client binary, so local overrides are evaluated locally and composed with upstream IR.

## 5. Deterministic evaluation

Package evaluation is pure. The Steel environment for package code is capability-restricted: values, collections, macros, modules, constructors, target predicates, transformations. No network, filesystem, environment, clock, randomness, subprocesses, or native libraries.

```scheme
(git-source "https://github.com/..." #:revision "..." #:hash "...")
```

describes a source. It does not fetch one.

## 6. Store and build model

The store lives at `/kura/store`. Paths are input-addressed: the hash of the canonical IR for a build, including all inputs. Content-addressed outputs come later.

```
/kura/store/<hash>-hyprland-0.52.0
```

Objects are immutable and reference their dependencies explicitly. The Rust engine owns identity, reference tracking, realization, sandboxing, scheduling, logs, garbage collection, substitution, remote builds, and verification.

Kura runs its own builders, signing keys, caches, release pipeline, and CDN. There is no binary compatibility with Guix or Nix.

### Bootstrap

To build packages, Kura needs a compiler, linker, libc, and build tools that Kura itself built. It doesn't have those yet. Through Milestone 4, Kura imports Arch's toolchain and libraries as foreign inputs: hashed by content and immutable once imported, but not built by Kura. Milestone 5 builds the toolchain with Kura, after which foreign inputs are limited to firmware and vendor binaries.

## 7. The desktop doesn't change

Kura does not touch the desktop architecture: Linux, systemd, logind, systemd user services, UWSM, Hyprland, Quickshell.

```scheme
(omarchy-system
  (hostname "workstation")
  (kernel linux)
  (hardware (auto-detect))
  (desktop omarchy-desktop)
  (services network-manager bluetooth printing docker))
```

This evaluates to a closure containing kernel, initrd, bootloader entries, `/etc`, systemd units, firmware, drivers, packages, the UWSM session, and the Omarchy desktop. Activation creates a generation and atomically makes it current. Old generations stay bootable.

## 8. FHS compatibility

Each generation contains a real FHS tree—`/usr/bin`, `/usr/lib`, `/usr/share`, `/etc`—as a store object: a directory of symlinks into the store, built like any other output. On the running system, `/usr` and `/etc` are symlinks to the current generation's tree. Switching generations swaps two symlinks.

Kura-built packages link to store paths directly via `RUNPATH`. Foreign binaries—NVIDIA, Steam, JetBrains, Electron apps, anything with `/usr/lib` hardcoded—find what they expect where they expect it, with an unmodified dynamic linker and no patching. Two packages that provide the same path conflict at composition time, in Rust, not during install on the user's machine.

Package metadata covers source availability, license, redistribution rights, provenance, reproducibility, vendor origin, architecture support, and security status. Open-source packages, proprietary drivers, firmware, and commercial binaries are all first-class where legally permitted.

## 9. Mutable state

The store, `/usr`, and `/etc` are immutable per generation. Everything else is explicitly mutable:

- `/var` and `/home` persist across generations.
- Users, groups, and tmpfiles are declared in the system definition and reconciled at activation via `sysusers.d` and `tmpfiles.d`.
- Secrets never enter the store. They are provisioned at activation into `/run` or `/var`.
- There are no machine-local `/etc` edits. Overrides go in the system definition.

## 10. AI-assisted maintenance

Most package and infrastructure maintenance will be done by agents:

```
upstream release → agent updates recipe → Steel eval → Rust validation
→ sandbox build → package tests → VM integration tests
→ provenance and reproducibility checks → promotion
```

The Rust compiler gives agents immediate structural feedback. Steel keeps recipe edits small. The IR is what both humans and agents inspect when something breaks; `kura explain` and `kura diff` work on IR, not Scheme source.

Agents speed up maintenance. They don't replace deterministic builds or validation.

## 11. Milestones

Vertical proofs, not a rewrite.

**M1 — Steel package prototype.** Rust types for `Package`, `Source`, `Dependency`, `Output`, `BuildPlan`, exposed to Steel. Port ten packages that cover the build systems and package kinds Omarchy has: `hyprland` (CMake), `quickshell` (CMake/Qt), `uwsm` (Python), `alacritty` (Cargo), `neovim` (CMake), `pipewire` (Meson), `mesa` (Meson), `wayland` (Meson), `nvidia` (vendor driver), and one proprietary desktop app. Validates the language and semantic boundary.

**M2 — Local store.** Store paths, realization, reference tracking, local builds, garbage collection. Build the M1 set into `/kura/store` using Arch's toolchain as foreign inputs.

**M3 — Sandboxed builds.** Namespaces, filesystem isolation, controlled networking, build users, resource limits, structured logs. Done when the M1 set builds bit-for-bit identically twice.

**M4 — Profiles and generations.** Package profiles, atomic generation switching, rollback, GC roots. Kura manages packages on developer machines while Arch still hosts.

**M5 — Bootable Omarchy generation.** Kura builds its own toolchain. Generate a full system—kernel, initrd, systemd, `/etc`, UWSM, Hyprland, Quickshell, Omarchy shell—and boot it in a VM. First point at which Kura can replace Arch.

**M6 — Binary cache and installer.** Signed substitutes, builders, release promotion, hardware profiles, installer generation. Only then does Kura become the default install.

## 12. Success criteria

The first release proves the architecture, not package coverage. It must:

1. boot a VM from a Kura-built image;
2. start systemd and UWSM;
3. launch Hyprland and Quickshell;
4. have working networking, audio, portals, notifications, terminal, browser, and editor;
5. switch between generations;
6. update atomically from N to N+1;
7. roll back to N;
8. pull every production package from a signed Omarchy binary cache.

After that, package coverage and hardware support are engineering work, not architectural risk.

## 13. Conclusion

Kura gives Omarchy the whole path from source definition to running system, with no other distribution's update semantics, packaging policy, release cadence, infrastructure, or politics in the loop, and without rewriting the parts of Omarchy that already work.

> Keep Scheme where programmability matters. Keep Rust where correctness matters. Make the boundary typed, deterministic, and inspectable.

This is the path from Arch-based Omarchy to an independent functional distribution, and there is no flag day.

---

## Why "Kura"?

*Kura* (蔵) is Japanese for a storehouse: a place where valuable things are kept and retrieved when needed. It names the central abstraction—an immutable store of packages and complete system closures—and the project's role beneath Omarchy. Short, pronounceable, and not borrowed from Guix or Nix.
