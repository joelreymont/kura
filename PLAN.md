# Kura Implementation Plan

**Status:** Proposed implementation specification  
**Baseline:** RFC-0001 in `README.md`, amended together with this plan  
**Primary target:** x86-64 Omarchy workstation and QEMU/OVMF virtual machine  
**Languages:** Rust and Steel Scheme  
**Store root:** `/kura/store`  
**State root:** `/var/lib/kura`  
**Runtime root:** `/run/kura`

This document turns RFC-0001 into an executable engineering plan. It is written for an implementation agent and is intentionally prescriptive: where the RFC leaves a choice open, this plan selects a v1 design so implementation can proceed without repeatedly reopening architecture.

Normative words such as **must**, **must not**, **should**, and **may** have their usual RFC meanings.

---

## 1. Deliverable

The implementation is complete when Kura can:

1. evaluate a repository of Steel package and system definitions into deterministic Kura IR;
2. lower package definitions into fully resolved derivations whose output paths are known before the build;
3. fetch fixed-output sources and import pinned Arch packages without consulting the ambient host filesystem;
4. build packages as unprivileged users in isolated Linux sandboxes;
5. commit normalized immutable outputs to `/kura/store`;
6. compose package outputs into user profiles and complete FHS system generations;
7. switch a generation through one coherent, journaled root-view transition and roll back through the same backend;
8. boot an x86-64 Omarchy generation from a LUKS2-encrypted Btrfs root under QEMU/OVMF with systemd, UWSM, Hyprland, and Quickshell;
9. retrieve every release output from a signed binary cache;
10. explain and diff every material decision from typed IR rather than by interpreting Scheme again.

The implementation must follow the RFC's vertical milestones. It must not start by porting the full Arch package set or by writing a general-purpose Nix/Guix replacement.

---

## 2. Scope and explicit non-goals

### 2.1 In scope for v1

- Linux only.
- x86-64 as the first executable platform.
- Rust as the sole implementation language for trusted semantics, storage, building, activation, and distribution.
- Steel as an embedded, capability-restricted definition language.
- Input-addressed derivation outputs and a separate content-and-policy identity domain for fixed outputs.
- A local immutable store.
- Fixed-output source objects and foreign package imports whose paths do not depend on mirrors or provenance.
- Arch package archives as pinned foreign bootstrap inputs.
- CMake, Meson, Cargo, Python/PEP 517, generic script, and vendor-binary build plans.
- systemd as PID 1 and service manager.
- A systemd-based initrd with LUKS2 and Btrfs support from the first bootable M5 system.
- Limine as the initial bootloader integration.
- Btrfs as the production root filesystem, with the Kura store, staging area, and trash on one Btrfs subvolume.
- LUKS2 encryption of the production system volume; only the EFI System Partition remains unencrypted.
- FHS generation trees for compatibility.
- User profiles while Arch still hosts the machine.
- Full Kura system generations in a VM.
- Signed repository metadata and binary substitutes.
- Stable, RC, edge, and development channels.

### 2.2 Deferred beyond v1

The following must not block the first bootable release:

- content-addressed output paths;
- cross-distribution compatibility;
- compatibility with Nix derivations, `/nix/store`, Guix packages, or `/gnu/store`;
- an independent package language other than Steel;
- replacing systemd, UWSM, Hyprland, or Quickshell;
- transparent rollback of mutable application databases;
- in-place conversion of arbitrary existing Arch installations;
- Secure Boot enrollment and key management;
- TPM2/FIDO2 unattended unlock, remote unlock, and hibernation-resume integration;
- a native hand-written namespace executor replacing Bubblewrap;
- remote evaluation of arbitrary user Steel code;
- multi-user package policy beyond the permissions described here;
- x86-64 and AArch64 reaching feature parity before the x86-64 VM proof;
- eliminating the binary bootstrap seed down to source-only or stage0 assembly;
- automatic resolution of FHS path conflicts;
- arbitrary post-install scripts.

Deferred work must still be anticipated in versioned interfaces. It must not be partially implemented in ways that weaken v1 invariants.

---

## 3. System invariants

These invariants are the architecture. Code that violates one must be rejected in review even when it appears to make a milestone easier.

### 3.1 Semantic invariants

1. **Rust is the semantic authority.** Steel can compose values but cannot invent unvalidated package, derivation, service, system, source, or store identities.
2. **Evaluation is descriptive.** Evaluating Steel never fetches, builds, reads host state, invokes a subprocess, or mutates the store.
3. **Canonical IR determines identity.** Persistent identities are derived only from canonical Kura IR. Diagnostic formatting, source spans, timestamps, and database row order do not affect identity.
4. **Derivation outputs and fixed outputs have distinct identity domains.** Ordinary outputs are identified by their derivation and output name. Fixed outputs are identified by their expected canonical content digest and materialization policy; mirrors, revisions, signatures, lock-entry names, and fetcher details are acquisition metadata and cannot change the store path.
5. **Resolved derivations are closed.** A derivation names every source, executable, build input, runtime input, environment variable, output, and policy value that can affect its output. Downstream derivations contain a fixed output ID, never its acquisition request.
6. **Output paths are known before execution.** Build processes see their final absolute `/kura/store/...` output paths even though bytes are written to staging directories.
7. **Store objects never change in place.** A valid store path is either absent or immutable. A second realization with different contents is a reproducibility failure, never an overwrite.
8. **No ambient host dependencies.** A build cannot read `/usr`, `/etc`, the user's home, the daemon's environment, or undeclared store paths.
9. **A signed realization binds input identity to output content.** Binary substitution is accepted only when trusted metadata maps the expected output ID to a verified archive content digest.

### 3.2 Deployment invariants

1. **One generation identity defines the active filesystem generation.** `/usr`, `/etc`, and `/opt` must expose the same generation. The preferred backend uses stable links through `/kura/profiles/system/current`; a compatibility spike must prove that layout against systemd hardening, Bubblewrap, and Flatpak. If it fails, Kura uses a journaled bind-mount backend. Mixed-generation root trees are forbidden in either backend.
2. **Root-view switching and process convergence are separate phases.** The selected backend must expose one coherent generation or require boot activation; restarting, reloading, or rebooting processes is planned and journaled separately from that root-view transition.
3. **The booted generation is always retained.** Garbage collection must root both the selected current generation and the generation from which the kernel/userspace booted.
4. **Mutable state is named explicitly.** Every writable path beneath an otherwise immutable generation is backed by `/var` or `/run` through a declared state link.
5. **Production storage is fixed for v1.** Fresh installs and the M5 release VM use one LUKS2 data volume containing Btrfs `@root`, `@kura`, `@var`, and `@home`; `/kura/store`, `/kura/.staging`, and `/kura/.trash` remain in the same `@kura` subvolume.
6. **Secrets never enter IR or store bytes.** IR may contain a secret name and provisioning policy, never secret material.
7. **Rollback does not claim to roll back mutable databases.** Service state compatibility is represented by barriers and migration policy.

### 3.3 Security invariants

1. Package and system Scheme is untrusted input.
2. Source trees and build scripts are untrusted input.
3. Build workers and CDN responses are untrusted until cryptographically and reproducibly verified.
4. The privileged daemon parses and validates all client-supplied IR itself.
5. Only root or an explicitly authorized administrator may activate a system generation.
6. The threat model does not defend against a compromised kernel or root account.

---

## 4. Top-level architecture

```text
                   repository snapshot
                           │
                           ▼
             ┌───────────────────────────┐
             │ kura-eval-worker          │
             │ Steel VM, no capabilities │
             └─────────────┬─────────────┘
                           │ typed Rust values
                           ▼
             ┌───────────────────────────┐
             │ kura-model / kura-ir      │
             │ validate + canonicalize   │
             └─────────────┬─────────────┘
                           │ package/system KIR
                           ▼
             ┌───────────────────────────┐
             │ resolver + lowerer        │
             │ package → derivation DAG  │
             └─────────────┬─────────────┘
                           │ resolved derivations
                 ┌─────────┴──────────┐
                 ▼                    ▼
      ┌────────────────────┐  ┌────────────────────┐
      │ source/import path │  │ substitute path    │
      │ fixed-output fetch │  │ signed Kinfo + KAR │
      └─────────┬──────────┘  └─────────┬──────────┘
                │                       │
                └───────────┬───────────┘
                            ▼
                ┌──────────────────────┐
                │ kurad                │
                │ scheduler + store DB │
                └───────────┬──────────┘
                            │
              ┌─────────────┴──────────────┐
              ▼                            ▼
   ┌──────────────────────┐     ┌──────────────────────┐
   │ kura-sandbox         │     │ profile/system       │
   │ bwrap + cgroup v2    │     │ composition          │
   └──────────┬───────────┘     └──────────┬───────────┘
              │                             │
              ▼                             ▼
       immutable outputs          immutable generation
              │                             │
              └──────────────┬──────────────┘
                             ▼
                 ┌──────────────────────┐
                 │ kura-activator       │
                 │ journal + root view  │
                 └──────────────────────┘
```

### 4.1 Process boundaries

Kura has five security-relevant process classes.

#### `kura`

The unprivileged CLI. It performs command parsing, repository discovery, local display, and client protocol handling. It must not write the system store directly.

#### `kura-eval-worker`

A short-lived, unprivileged evaluator process. It embeds the pinned Steel VM, receives an in-memory module bundle and target parameters, and returns typed KIR or diagnostics. It has no network and no writable filesystem other than bounded anonymous temporary memory.

#### `kurad`

A root-owned daemon. It owns store writes, build scheduling, build-user allocation, substitute registration, garbage collection, system profile mutation, and transaction journals. It must revalidate KIR received from clients.

#### `kura-builder`

A small executable used inside build sandboxes. It interprets a typed `BuildSpec`; it does not evaluate Steel and does not resolve dependencies.

#### `kura-activator`

A narrowly scoped activation helper started by `kurad` before a system switch. It performs the journaled pointer change and service convergence. It survives replacement of the generation containing `kurad` and restarts the daemon last.

Source fetch helpers and archive importers may be separate short-lived processes, but they use typed requests and never share the evaluator process.

---

## 5. Repository and Rust workspace layout

Create this repository shape before implementing features:

```text
kura/
├── AGENTS.md
├── Cargo.toml
├── Cargo.lock
├── LICENSE-APACHE
├── LICENSE-MIT
├── README.md
├── PLAN.md
├── rust-toolchain.toml
├── deny.toml
├── crates/
│   ├── kura-core/
│   ├── kura-ir/
│   ├── kura-model/
│   ├── kura-eval/
│   ├── kura-resolver/
│   ├── kura-source/
│   ├── kura-store/
│   ├── kura-builder-spec/
│   ├── kura-builder/
│   ├── kura-sandbox/
│   ├── kura-engine/
│   ├── kura-profile/
│   ├── kura-system/
│   ├── kura-cache/
│   ├── kura-protocol/
│   ├── kura-cli/
│   ├── kurad/
│   ├── kura-activator/
│   ├── kura-init-select/
│   └── kura-test-support/
├── repo/
│   ├── prelude/
│   ├── packages/
│   ├── systems/
│   ├── hardware/
│   └── bootstrap/
├── schemas/
│   ├── kir-v1.cddl
│   ├── kinfo-v1.cddl
│   └── protocol-v1.cddl
├── docs/
│   ├── adr/
│   ├── architecture/
│   ├── packaging/
│   └── operations/
├── tests/
│   ├── fixtures/
│   ├── golden/
│   ├── sandbox/
│   ├── integration/
│   └── vm/
└── xtask/
```

### 5.1 Crate responsibilities

| Crate | Responsibility | Must not depend on |
| --- | --- | --- |
| `kura-core` | IDs, names, paths, target triples, hashes, domain errors | Steel, SQLite, CLI |
| `kura-ir` | KIR value model, v1 codec, hashing, schemas, golden vectors | Steel, store implementation |
| `kura-model` | Typed package/source/service/system model and validation | daemon, filesystem |
| `kura-eval` | Steel adapter, module bundling, evaluator worker | store writes, network |
| `kura-resolver` | package selection, variants, dependency closure, lowering | daemon process |
| `kura-source` | fixed-output fetch and foreign archive import | Steel |
| `kura-store` | store layout, SQLite state, references, roots, commit, GC, archive | Steel |
| `kura-builder-spec` | typed phases/actions and stable serialization | sandbox implementation |
| `kura-builder` | in-sandbox interpreter for `BuildSpec` | daemon database |
| `kura-sandbox` | Bubblewrap command construction, build users, cgroups, seccomp | Steel |
| `kura-engine` | build DAG scheduling and realization orchestration | CLI rendering |
| `kura-profile` | profile manifests, generations, collision checking | evaluator |
| `kura-system` | FHS composition, service generation, activation planning, images | Steel VM |
| `kura-cache` | KAR/Kinfo, trust metadata, substitution, publication | evaluator |
| `kura-protocol` | framed client-daemon request/event protocol | CLI terminal code |
| `kura-cli` | command UX and rendering | direct store mutation |
| `kurad` | privileged service wiring | Steel evaluation in-process |
| `kura-activator` | activation transaction | package evaluation |
| `kura-init-select` | small M5 systemd-initrd helper that validates and selects the boot generation and root view | Steel, SQLite |
| `kura-test-support` | temporary stores, fixture builders, fault injection | production binaries |

Dependency direction must be enforced with workspace linting and an architecture test.

### 5.2 Toolchain policy

- Pin an exact stable Rust toolchain in `rust-toolchain.toml`.
- Commit `Cargo.lock`.
- Deny unreviewed duplicate crypto, serialization, and SQLite crates with `cargo-deny`.
- Use `unsafe` only in crates that require OS interfaces; each unsafe block must state its invariant.
- Keep the persistent formats independent of `serde`'s derived field order.
- Initially license original Kura code as `MIT OR Apache-2.0` unless the project owner selects another license before the first code import.
- Do not copy implementation code from GPL projects into permissively licensed crates. Prior art may guide behavior; implementations must be original or license-compatible.

---

## 6. Domain model

The high-level model must distinguish definitions from realized artifacts. Do not use one `Package` structure for every stage.

### 6.1 Core identities

```rust
pub struct Digest256([u8; 32]);

pub struct PackageName(String);
pub struct Version(String);
pub struct OutputName(String);
pub struct Platform {
    pub arch: Architecture,
    pub os: OperatingSystem,
    pub abi: Abi,
    pub cpu_level: CpuLevel,
}

pub struct PackageId {
    pub repository: RepositoryId,
    pub name: PackageName,
    pub variant: VariantId,
}

pub struct DerivationId(Digest256);
pub struct MaterializationPolicyId(Digest256);

/// One 256-bit path identity with disjoint derivation-output and fixed-output hash domains.
pub struct OutputId(Digest256);

pub enum OutputOrigin {
    Derivation {
        derivation: DerivationId,
        output: OutputName,
    },
    Fixed {
        kind: FixedOutputKind,
        expected_content: ContentDigest,
        materialization_policy: MaterializationPolicyId,
    },
}

pub struct StorePath {
    pub output_id: OutputId,
    pub label: StoreLabel,
}
pub struct ContentDigest(Digest256);
```

Constructors must validate allowed characters, lengths, normalization, and reserved names. Raw strings must not cross crate boundaries where a domain type exists.

### 6.2 Definition-stage objects

- `PackageDefinition`
- `SourceDefinition`
- `DependencySpec`
- `Variant`
- `BuildPlan`
- `OutputDefinition`
- `ServiceDefinition`
- `SystemDefinition`
- `HardwareProfile`
- `ProfileDefinition`
- `SecretReference`
- `StatePathDefinition`

These can contain unresolved package references and target predicates.

### 6.3 Resolution-stage objects

- `ResolvedPackage`
- `ResolvedDependency`
- `ResolvedSource`
- `ResolvedBuildPlan`
- `ResolvedService`
- `ResolvedSystem`

Every package reference is pinned to a repository snapshot, package definition digest, variant, platform, and selected output.

### 6.4 Execution-stage objects

- `Derivation`
- `DerivationOutput`
- `FixedOutputRequest`
- `BuildSpec`
- `SandboxPolicy`
- `ResourceRequest`
- `Realization`
- `StoreObjectInfo`
- `Closure`

A derivation contains no unresolved predicates, macros, package names requiring lookup, or implicit defaults.

### 6.5 Deployment-stage objects

- `ProfileManifest`
- `GenerationManifest`
- `FhsTree`
- `SystemManifest`
- `ServiceManifest`
- `ActivationPlan`
- `ActivationTransaction`
- `BootEntry`
- `ImageManifest`

### 6.6 Origin information

Every definition-stage value carries an optional non-semantic origin:

```rust
pub struct Origin {
    pub module_digest: Digest256,
    pub module_name: String,
    pub byte_start: u32,
    pub byte_end: u32,
    pub expansion_stack: Vec<MacroFrame>,
}
```

Origins are serialized in diagnostic sidecars but excluded from all semantic hashes. `kura explain` uses them to point back to Steel definitions.

---

## 7. Kura IR

Kura IR, abbreviated KIR, is the stable boundary between evaluation and execution.

### 7.1 Encoding

Use a Kura-owned deterministic CBOR subset for v1. Do not hash JSON, Rust `Debug`, `bincode`, or a generic `serde_cbor` representation.

The encoder must support only:

- unsigned and signed integers;
- byte strings;
- UTF-8 text strings normalized to NFC;
- fixed-length arrays;
- explicitly ordered maps where unavoidable;
- booleans and explicit null values.

It must reject:

- floats;
- indefinite-length values;
- duplicate map keys;
- non-normalized text;
- unknown semantic tags;
- integers outside the supported range;
- trailing bytes.

Semantic structures should use arrays with numeric type and schema identifiers rather than maps:

```text
[
  0x4b4952,          # "KIR" marker
  1,                 # KIR major version
  20,                # object kind: derivation
  [
    1,               # derivation schema version
    "hyprland",
    "0.52.0",
    [...],
    ...
  ]
]
```

`schemas/kir-v1.cddl` documents the format, but the Rust decoder remains authoritative.

### 7.2 Canonical ordering

- Lists preserve order only where order is semantically meaningful.
- Set-like fields are sorted by their domain key.
- Environment maps are sorted by normalized variable name.
- Dependency sets are sorted by output ID, then mount role.
- Service enablement and file entries are sorted by absolute path.
- Extension maps are sorted by canonical encoded key bytes.
- The encoder must produce one byte string for one semantic value.

### 7.3 Hash domains

Use SHA-256 for persistent v1 identities and lowercase unpadded RFC 4648 Base32 for path text. `OutputId` is a 256-bit digest produced by one of two disjoint domains.

```text
derivation_id =
  SHA256("kura:derivation:v1\0" || canonical_derivation_bytes)

materialization_policy_id =
  SHA256(
    "kura:materialization-policy:v1\0" ||
    canonical_materialization_policy_bytes
  )

derivation_output_id =
  SHA256(
    "kura:derivation-output:v1\0" ||
    derivation_id ||
    "\0" ||
    utf8(output_name)
  )

fixed_output_id =
  SHA256(
    "kura:fixed-output:v1\0" ||
    canonical_fixed_output_identity_bytes
  )

package_definition_id =
  SHA256("kura:package-definition:v1\0" || canonical_package_bytes)

system_id =
  SHA256("kura:system:v1\0" || canonical_system_bytes)
```

`canonical_fixed_output_identity_bytes` is KIR containing exactly:

```text
[
  fixed_output_kind,
  digest_algorithm,
  expected_canonical_content_digest,
  materialization_policy
]
```

The expected digest covers the canonical payload or tree representation defined by the materialization policy, not a mirror URL, Git revision, transport archive, signature, fetch log, or the compressed KAR used for distribution. `materialization_policy` is versioned semantic data such as `raw-file-v1`, archive format and strip count plus `canonical-tree-v1`, Git submodule policy, or `arch-package-import-v1`. Changing any policy that can change the final tree changes the fixed output ID.

A separate operational request may be identified as:

```text
fixed_output_request_id =
  SHA256(
    "kura:fixed-output-request:v1\0" ||
    canonical_acquisition_request_bytes
  )
```

That request may contain URLs, mirrors, revision/tag hints, signature locations, and trust metadata. It is used for fetching, diagnostics, and cache coordination only. It is never embedded in a downstream derivation and never determines a store path. Adding a mirror or changing a revision hint while retaining the same expected content and materialization policy therefore causes no rebuild.

`ForeignPackage` imports use the fixed-output domain too: their expected digest is the canonical imported package-tree digest and their materialization policy is the exact versioned foreign-package importer. Archive digests and signatures are verified acquisition metadata, not path identity.

The full 256-bit output ID is used in the store path. Do not truncate it.

```text
/kura/store/<52-base32-chars>-<sanitized-label>
```

Examples:

```text
/kura/store/6qs...52chars...p7-hyprland-0.52.0
/kura/store/f4n...52chars...2a-mesa-25.2.1-dev
```

The human-readable suffix is not authoritative. Parsing a store path always reads and validates the hash component first.

### 7.4 Semantic and non-semantic fields

Semanticity is relative to the identity being computed. Kura must not use one catch-all serialization for package definitions, acquisition requests, derivations, and store paths.

A derivation ID includes:

- package name/version used for diagnostics and builder behavior where declared;
- platform;
- builder output ID and arguments;
- declared environment;
- exact input `OutputId` values and mount roles;
- output names;
- build actions;
- sandbox capabilities;
- explicit runtime references;
- build-system lowering version.

For a fixed source, the derivation contains only `fixed_output_id`. It does not contain URLs, revision hints, signature locations, `fixed_output_request_id`, or `package_definition_id` merely because the recipe or lock entry changed.

A fixed-output identity includes only the fields in §7.3. Its separate acquisition-request identity may include mirror/source URLs, mirror order, Git revision or tag hints, detached-signature locations, lock-entry location, and trust requirements. Those fields are semantic to `fixed_output_request_id` but non-semantic to `fixed_output_id`, `derivation_id`, and downstream store paths.

The following remain non-semantic to every persistent build/store identity and belong in origin or operational sidecars:

- source spans and comments;
- evaluation, download, and build timestamps;
- TLS and signature verification logs;
- log location and worker hostname;
- display formatting and channel name;
- database row IDs.

A mirror-only edit is allowed to change `package_definition_id`, repository snapshot ID, and `fixed_output_request_id`. It must not change `fixed_output_id`, `derivation_id`, or any downstream `OutputId`.

### 7.5 Compatibility

- Every persistent object includes a major schema version.
- A binary must read its current major version and the immediately preceding major version.
- Writers emit only the current version.
- Migration is decode-old, validate, encode-new; old bytes remain available for audit.
- Golden byte vectors and hashes must be checked in.
- CI must verify vectors on x86-64 and AArch64 and across debug/release builds.

### 7.6 Diagnostic rendering

Provide deterministic, human-readable JSON and text renderers, but label them as views:

```bash
kura show --json hyprland
kura show --kir-hex hyprland
kura hash --explain hyprland
```

`kura hash --explain` prints the exact semantic fields and domain separator that produced an identity.

---

## 8. Steel evaluation

### 8.1 Version pin and adapter

Pin Steel to an exact Git commit. All Steel APIs must be hidden behind `crates/kura-eval/src/steel_adapter.rs`. No other crate imports Steel.

The adapter owns:

- VM creation;
- built-in allowlisting;
- module loading;
- Rust type registration;
- error conversion;
- source location capture;
- evaluation limits;
- extraction of root definitions.

A Steel upgrade is one atomic change containing compatibility tests and regenerated evaluator fixtures.

### 8.2 Capability model

The evaluator VM exposes only:

- numbers, booleans, strings, symbols;
- immutable lists, vectors, maps, and sets;
- lexical functions and macros;
- pure collection transforms;
- Kura constructors and accessors;
- target/platform predicates;
- module import from the supplied in-memory module bundle;
- deterministic formatting needed for names.

It must not expose:

- filesystem APIs;
- environment access;
- process execution;
- sockets or HTTP;
- clocks or dates;
- randomness;
- dynamic native libraries;
- Git access;
- host locale;
- JIT compilation;
- arbitrary Rust FFI registration;
- mutable global process state.

Do not assume that disabling a Steel feature is sufficient. Create the VM from an explicit allowlist and run it inside an OS sandbox as defense in depth.

### 8.3 Module bundle

The CLI or repository service creates a `ModuleBundle` before starting the worker:

```rust
pub struct ModuleBundle {
    pub root_module: ModuleId,
    pub modules: BTreeMap<ModuleId, ModuleSource>,
    pub repository_snapshot: RepositorySnapshotId,
    pub target: Platform,
    pub feature_flags: BTreeSet<FeatureName>,
}
```

A module source contains normalized relative name, UTF-8 bytes, and digest. Import resolution occurs only within this map. The worker does not open repository files itself.

Reject:

- absolute module paths;
- `..` path components;
- duplicate normalized paths;
- invalid UTF-8;
- modules whose supplied digest does not match their bytes;
- import cycles not supported by the Kura prelude;
- modules outside the snapshot.

### 8.4 Rust-backed values

Steel code must work with opaque immutable Rust values such as:

- `KuraPackage`
- `KuraSource`
- `KuraDependency`
- `KuraBuildPlan`
- `KuraService`
- `KuraSystem`
- `KuraTransform`

Constructors validate their immediate arguments and return a new value. Mutation is represented by functional transforms.

Example public surface:

```scheme
(define hyprland
  (package
    (name "hyprland")
    (version "0.52.0")
    (source
      (github-source
        (owner "hyprwm")
        (repository "Hyprland")
        (revision version)
        (sha256 "...")))
    (dependencies wayland libinput pixman)
    (build (cmake-build))
    (outputs "out" "dev")))
```

The shipped prelude may provide `define-package` syntax, but the macro must lower to Rust constructors. It must not create untyped association lists that Rust later guesses how to interpret.

### 8.5 Errors and source spans

Errors must contain:

- stable error code;
- human message;
- definition kind/name;
- module and byte span;
- macro expansion stack where available;
- suggested correction;
- related origin spans for conflicts.

Example:

```text
KURA-EVAL-021: dependency output does not exist
  package: quickshell
  requested: qt6-base:headers
  available: out, dev
  at repo/packages/quickshell.scm:18:13
```

Patch Steel locally if necessary to expose current call-site spans to registered Rust functions. Do not accept opaque "native function failed" errors as the permanent interface.

### 8.6 Evaluation limits

The worker must have:

- maximum module bytes;
- maximum object count;
- maximum string/vector size;
- maximum recursion depth;
- maximum VM instruction count;
- cgroup memory and CPU limits;
- a parent watchdog;
- deterministic error codes for limit exhaustion.

Wall-clock timeout is a safety backstop, not part of semantic behavior. Add a VM fuel counter so infinite evaluation fails at the same instruction boundary.

### 8.7 Evaluation outputs

Repository evaluation returns:

```rust
pub struct EvaluationResult {
    pub snapshot: RepositorySnapshotId,
    pub target: Platform,
    pub roots: Vec<DefinitionRef>,
    pub objects: Vec<KirObject>,
    pub origins: OriginTable,
    pub diagnostics: Vec<Diagnostic>,
}
```

The repository service signs the semantic objects and a separate origin bundle. Clients may discard origins without changing identities.

### 8.8 Local overrides

Clients load signed upstream KIR into Rust values, expose those values to a local evaluator, and apply local Steel transforms. The local evaluator emits new KIR whose inputs include the exact upstream object IDs.

Local overrides never rewrite or shadow signed upstream bytes in place.

---

## 9. Resolution and lowering

### 9.1 Repository snapshot

A `RepositorySnapshot` pins:

- repository identity;
- KIR version;
- module tree digest;
- package definition IDs;
- transform IDs;
- target set;
- prelude version;
- Steel engine version;
- resolver version.

Package lookup must always be relative to a snapshot. `latest` is a channel operation outside semantic resolution.

### 9.2 Dependency roles

Represent dependency purpose explicitly:

```rust
pub enum DependencyRole {
    Build,
    Host,
    Target,
    Runtime,
    Propagated,
    Test,
    Tool,
}
```

For native x86-64 builds, host and target may initially coincide, but the distinction must be preserved for cross-compilation later.

Each dependency selects named outputs.

### 9.3 Variants

Variants are pure transforms with explicit dimensions:

- platform;
- feature set;
- build type;
- hardware capability;
- license/source availability;
- debug/development output selection.

Resolution must reject two transforms that assign incompatible values to the same dimension. Variant order must not silently change results.

### 9.4 Package to derivation

Lowering consists of deterministic passes:

```text
PackageDefinition
  → apply explicit transforms
  → select target branches
  → resolve dependency references
  → resolve source objects
  → select outputs
  → expand build-system defaults
  → construct BuildSpec
  → calculate input closure
  → validate sandbox and metadata policy
  → canonical Derivation
```

Each pass has a stable version included in KIR or in the selected builder identity.

Before constructing a derivation, the lowerer separates each fixed source into:

- a semantic `OutputId` derived from expected canonical content and materialization policy; and
- a non-closure acquisition request containing mirrors, revision hints, signature locations, and other provenance.

The derivation contains only the fixed output ID/store path. It does not carry the enclosing package-definition digest solely as a recipe provenance token. A mirror edit may change package/repository metadata and an acquisition request ID, but it must leave the derivation ID and every downstream output path unchanged.

### 9.5 Build-system lowering

High-level helpers such as `(cmake-build)` are definition conveniences. Rust lowers them into a typed `BuildSpec`.

The lowerer must never emit an implicit executable name such as `"cmake"`. It emits an exact executable reference to a selected store output and an absolute path within that output.

### 9.6 Why-rebuild diff

Keep pass-level intermediate summaries so `kura diff` can identify the first semantic difference:

```text
hyprland output changed because:
  source.revision:
    old 6e01...
    new 8a94...
No dependency or build-plan fields changed.
```

This comparison works on typed KIR, not Scheme source text.

---

## 10. Sources and fixed-output objects

### 10.1 Source model

Initial source kinds:

```rust
pub enum Source {
    HttpFile {
        urls: Vec<Url>,
        sha256: Digest256,
        file_name: FileName,
    },
    Archive {
        file: FixedFileRef,
        format: ArchiveFormat,
        strip_components: u16,
        tree_sha256: Digest256,
    },
    GitTree {
        remote: Url,
        revision: GitCommit,
        submodules: SubmodulePolicy,
        tree_sha256: Digest256,
    },
    LocalTree {
        snapshot_entry: RepositoryPath,
        tree_sha256: Digest256,
    },
    ForeignPackage {
        lock_entry: ForeignLockId,
    },
}
```

Every source names an expected digest for the canonical payload or tree representation defined by its materialization policy. URL, mirror order, revision, tag, TLS result, detached-signature location, repository-relative path, and display file name are acquisition or provenance data. They are not path identity.

A raw `HttpFile` store object always contains its bytes at the canonical relative path `payload`; `file_name` is a display and installation hint. This prevents a harmless rename from changing the fixed object's bytes or identity.

### 10.2 Fixed-output identity

Every source lowers to `FixedOutputIdentity` plus a separate `FixedOutputRequest`. The identity uses the §7.3 fixed-output hash domain.

| Source kind | Expected canonical content digest | Materialization policy | Excluded from identity |
| --- | --- | --- | --- |
| `HttpFile` | SHA-256 of raw `payload` bytes | `raw-file-v1` | URLs, mirror order, file name, TLS data |
| `Archive` | SHA-256 of the normalized extracted tree | `archive-tree-v1(format, strip_components, canonical-tree-v1)` | archive URL and mirror order; the referenced transport object's acquisition data |
| `GitTree` | SHA-256 of the normalized checkout tree | `git-tree-v1(submodules, canonical-tree-v1)` | remote and revision/tag hint |
| `LocalTree` | SHA-256 of the normalized tree | `local-tree-v1(canonical-tree-v1)` | repository path and module location |
| `ForeignPackage` | SHA-256 of the normalized imported package mini-root | `arch-package-import-v1` or another named importer policy | archive URLs, archive digest, signature URL/fingerprint, lock-entry ID, snapshot date |

The transport archive digest is still mandatory when an archive is downloaded, and Git commits remain mandatory provenance pins. They protect acquisition and auditing. The final canonical content digest plus materialization policy determines the store path and downstream closure.

Changing a mirror, reordering URLs, moving a local source within the repository, or changing a Git revision that resolves to the same expected canonical tree must not change `fixed_output_id`, a downstream derivation, or a store path.

### 10.3 Fetch architecture

Fetching is not a normal unrestricted build.

1. `kurad` validates a `FixedOutputRequest` and its associated `FixedOutputIdentity`.
2. A fetch helper receives network access and only a bounded download directory.
3. Transport bytes are streamed through SHA-256 and checked against the acquisition digest where applicable.
4. Archive extraction or source materialization occurs in a separate no-network sandbox.
5. Extraction rejects absolute paths, `..`, device nodes, unsafe hardlinks, and escaping symlinks.
6. The resulting file/tree is normalized and hashed using the identity's canonical-content algorithm.
7. Only an exact expected canonical content digest is committed at the precomputed fixed output path.
8. Mismatches are quarantined with diagnostics.

Mirrors are attempted in declared order. Mirror selection and mirror-list edits do not affect object identity.

### 10.4 Canonical source trees

Normalize source tree metadata before hashing:

- paths sorted by raw UTF-8 bytes after normalization policy;
- uid/gid represented logically as zero;
- every mtime set to Unix timestamp `0`; v1 has no per-source alternative;
- directory permissions normalized unless explicitly required by the versioned materialization policy;
- no host xattrs;
- symlink target bytes preserved;
- hardlinks represented canonically;
- no sockets or device files.

The fixed value `0` applies to source-object canonicalization. A build's separately declared `SOURCE_DATE_EPOCH` may carry an upstream release epoch and is part of the derivation environment.

### 10.5 Git sources

A Git source must pin a full commit ID and expected canonical tree digest. Submodule commits are explicit. Git LFS objects must be fixed-output inputs, not fetched during a build.

Repository tags are never trust anchors. The remote and commit establish provenance and acquisition; the expected canonical tree digest and materialization policy establish store identity.

### 10.6 Source provenance

Operational metadata records:

- fixed output ID and fixed output request ID;
- all attempted URLs;
- TLS result;
- download timestamp;
- upstream revision/tag;
- transport archive digest;
- signature result where available;
- fetcher/tool version;
- canonical content digest.

None of the acquisition fields changes a fixed output's identity. Only its expected canonical content digest or materialization policy does.

---

## 11. Foreign Arch bootstrap

Kura must not import `/usr` from the running machine. Bootstrap inputs come from exact Arch package archives.

### 11.1 Lock file

Create target-specific lock files:

```text
repo/bootstrap/arch-x86_64.lock
repo/bootstrap/arch-aarch64.lock
```

Each entry contains:

```rust
pub struct ArchLockEntry {
    pub name: PackageName,
    pub version: String,
    pub architecture: Architecture,
    pub archive_urls: Vec<Url>,
    pub archive_sha256: Digest256,
    pub tree_sha256: Digest256,
    pub signature_urls: Vec<Url>,
    pub signing_fingerprint: Option<Fingerprint>,
    pub dependencies: Vec<ArchDependency>,
    pub snapshot_date: DateOnly,
    pub redistribution: RedistributionPolicy,
}
```

The lock is generated by a dedicated command and committed. Package evaluation consumes the lock as data; it does not query mirrors.

### 11.2 Import pipeline

For each package:

1. fetch the archive into the bounded transport cache and retrieve the detached signature as acquisition/trust metadata;
2. verify the archive SHA-256;
3. verify signature against the pinned bootstrap keyring;
4. parse `.PKGINFO`, `.BUILDINFO`, and `.MTREE`;
5. reject unsafe archive entries;
6. extract into a package mini-root;
7. do not run install scripts or ALPM hooks;
8. normalize metadata;
9. calculate and verify the canonical imported mini-root digest against `tree_sha256`;
10. record package metadata, archive digest, signature result, and canonical tree digest;
11. commit at the precomputed fixed output path.

### 11.3 Foreign-package identity

An Arch import is a fixed output, not an ordinary derivation output. Its `OutputId` is computed from:

```text
fixed_output_kind = foreign-package
expected_canonical_content_digest = tree_sha256
materialization_policy = arch-package-import-v1
```

The package name/version, lock-entry ID, mirror list, archive digest, signature locations, signing fingerprint, repository snapshot date, and importer executable path are recorded for acquisition, trust, and provenance but do not enter the store-path hash. The importer policy version does enter the hash because a semantic importer change can produce a different canonical mini-root.

Changing an Arch mirror or resigning/recompressing an archive that imports to the same expected mini-root therefore leaves every downstream derivation unchanged.

### 11.4 Bootstrap SDK

Compose selected foreign packages into an immutable bootstrap FHS tree:

```text
/kura/store/<hash>-arch-bootstrap-sdk/root/usr
/kura/store/<hash>-arch-bootstrap-sdk/root/etc
```

Build sandboxes mount this tree as `/usr` and selected files from `/etc`. The ambient host `/usr` is hidden.

The SDK must pin at least:

- shell and core utilities;
- binutils;
- compiler and linker;
- glibc and headers;
- pkg-config;
- make/ninja;
- CMake/Meson/Python;
- patch, tar, gzip, xz, zstd;
- Git only for controlled source tooling, never package builds;
- Rust/Cargo for Cargo packages;
- package-specific foreign libraries until Kura-built replacements exist.

### 11.5 Bootstrap trust accounting

`kura bootstrap explain` must list every binary seed in a derivation's transitive closure. M5 reduces that closure by rebuilding the toolchain; it does not pretend the historical bootstrap seed never existed.

---

## 12. Package output layout

Every package output is a mini-root:

```text
<output>/
├── usr/
│   ├── bin/
│   ├── lib/
│   ├── include/
│   ├── share/
│   └── libexec/
├── etc/
└── opt/
```

Outputs must not place files directly under arbitrary host paths.

Named output conventions:

- `out`: runtime programs and data;
- `dev`: headers, static libraries, pkg-config/CMake metadata;
- `doc`: large documentation;
- `debug`: split debug symbols;
- package-specific outputs only when justified.

Builds normally configure `--prefix=/usr` and install through `DESTDIR=$out`. This makes imported Arch packages and Kura-built packages compose through the same FHS machinery.

### 12.1 Output splitting

A deterministic fixup pass may move files from staging `out` to `dev`, `doc`, or `debug`. Split rules are part of the derivation.

Moving a file after it has been referenced by another file must preserve and rescan references.

### 12.2 Normalized metadata

Before commit:

- owner/group become logical `0:0`;
- umask-equivalent modes are enforced;
- mtimes are normalized;
- directory order is irrelevant;
- xattrs are allowlisted;
- Linux capabilities and setuid bits require explicit package metadata;
- device nodes, sockets, and FIFOs are rejected unless a narrowly defined output type permits them;
- writable bits on store directories are removed after commit.

The privileged store commit step applies approved ownership, capabilities, and xattrs. Build users do not gain privilege to create them directly.

---

## 13. Build specification and builder

### 13.1 Typed build actions

`BuildSpec` is a versioned execution language:

```rust
pub struct BuildSpec {
    pub version: BuildSpecVersion,
    pub phases: Vec<Phase>,
    pub environment: BTreeMap<EnvName, String>,
    pub working_directory: SandboxPath,
    pub outputs: Vec<BuildOutput>,
    pub fixups: Vec<Fixup>,
}

pub struct Phase {
    pub name: PhaseName,
    pub condition: PhaseCondition,
    pub actions: Vec<Action>,
}

pub enum Action {
    CreateDirectory { path: SandboxPath, mode: FileMode },
    Remove { path: SandboxPath, recursive: bool },
    Copy { from: SandboxPath, to: SandboxPath, mode: CopyMode },
    Move { from: SandboxPath, to: SandboxPath },
    ReplaceText { path: SandboxPath, replacements: Vec<Replacement> },
    ApplyPatch { patch: InputPath, strip: u8, directory: SandboxPath },
    Run {
        program: InputExecutable,
        args: Vec<Argument>,
        cwd: SandboxPath,
        env: BTreeMap<EnvName, String>,
        allowed_exit_codes: BTreeSet<i32>,
    },
    AssertPath { path: SandboxPath, kind: PathKind },
    AssertNoMatch { root: SandboxPath, pattern: BytePattern },
}
```

The builder resolves `InputExecutable` to an exact mounted store path. A bare executable name is invalid.

### 13.2 Standard phases

The default phase sequence is:

1. `unpack`
2. `patch`
3. `configure`
4. `build`
5. `check`
6. `install`
7. `split`
8. `fixup`
9. `validate`

A package can omit or extend phases but cannot mutate the execution model.

### 13.3 Build-system adapters

Implement in this order:

1. `script-build` for fixtures and escape hatches;
2. `cmake-build`;
3. `meson-build`;
4. `cargo-build`;
5. `python-build` using PEP 517;
6. `vendor-binary-build`;
7. `autotools-build` for toolchain bootstrap.

Each adapter expands to typed actions in Rust. Defaults such as `-DCMAKE_BUILD_TYPE=Release` are visible in KIR.

### 13.4 Script escape hatch

`RunScript` is allowed only with:

- an explicitly selected shell output;
- script bytes embedded in KIR or a fixed source;
- an explicit working directory;
- no interpolation performed by the daemon;
- all environment variables declared;
- a lint warning requiring package-level justification.

No build phase may use `/bin/sh` implicitly.

### 13.5 Deterministic environment

The sandbox starts with an empty environment and adds only declared values plus fixed Kura defaults:

```text
HOME=/build/home
PATH=<explicit native tool paths>
LANG=C.UTF-8
LC_ALL=C.UTF-8
TZ=UTC
SOURCE_DATE_EPOCH=<declared source epoch>
PYTHONHASHSEED=0
ARFLAGS=crD
KURA_BUILD_TOP=/build
KURA_OUTPUT_OUT=<final store path>
```

Also enforce:

- hostname `kura-builder`;
- fixed user/group names and IDs inside the namespace;
- fixed umask `022`;
- no `-march=native`;
- target CPU level from `Platform`;
- a stable logical parallel job count;
- deterministic GNU `ar`/`ranlib` mode through the selected toolchain adapter; `ARFLAGS=crD` is the v1 default.

Output normalization does not excuse embedded timestamps or nondeterministic generated content. Reproducibility tests must still compare canonical content digests.

### 13.6 Builder identity

The exact `kura-builder` output ID and `BuildSpec` version are derivation inputs. Changing builder semantics rebuilds affected outputs.

---

## 14. Sandboxing

### 14.1 Backend

Use Bubblewrap for the first production backend. Rust owns a typed `SandboxPlan` and renders Bubblewrap arguments. Do not let package definitions supply raw Bubblewrap arguments.

The trait allows a future native backend:

```rust
pub trait SandboxBackend {
    fn spawn(
        &self,
        plan: &SandboxPlan,
        io: SandboxIo,
    ) -> Result<SandboxChild, SandboxError>;
}
```

### 14.2 Filesystem view

A build sandbox contains only:

- read-only declared input store paths at their exact absolute locations;
- writable output staging directories bind-mounted at their final absolute store paths;
- a writable tmpfs `/build`;
- a writable tmpfs `/tmp`;
- a minimal read-only `/proc`;
- a minimal `/dev` containing null, zero, random, urandom, tty as needed;
- an empty `/home` except `/build/home`;
- generated `/etc/passwd`, `/etc/group`, `/etc/hosts`, and resolver files appropriate to policy;
- an FHS build environment mounted at `/usr` where required.

Hide the host root. Do not bind the entire `/kura/store`; mount only the declared closure.

### 14.3 Namespaces and privilege

Use new mount, PID, IPC, UTS, cgroup, and network namespaces. The build process runs as a dedicated unprivileged build user.

Create a fixed pool such as:

```text
kura-builder01 … kura-builder32
```

Accounts have no login shell and no home. `kurad` allocates one per concurrent build and cleans its processes before reuse.

Use `no_new_privs`. Deny privilege escalation, mount operations from the build, BPF, keyring manipulation, ptrace outside the sandbox, and other unnecessary high-risk syscalls with seccomp.

### 14.4 Network policy

Default is no network namespace interface other than loopback, with loopback down unless a test explicitly requires it.

Only fixed-output fetch helpers receive network capability. A normal derivation with `network = true` is invalid in v1.

### 14.5 Resource policy

Place every evaluation and build process in its own cgroup v2 subtree. Set:

- memory high/max;
- pids max;
- CPU weight/quota where needed;
- I/O weight;
- wall-clock watchdog;
- output-byte limit;
- log-byte limit.

Resource limits are operational policy, not semantic inputs, unless a package explicitly declares a resource-dependent algorithm that must be stable.

### 14.6 Build termination

On cancellation or failure:

1. freeze/cancel the cgroup;
2. kill all remaining processes;
3. wait for empty cgroup;
4. unmount sandbox;
5. release build user;
6. mark staging paths abandoned;
7. persist final structured error;
8. remove staging asynchronously after journal recovery can see it.

### 14.7 M2 versus M3

M2 may use the same Bubblewrap backend with a reduced policy, but it must never run package scripts as root or against the ambient host root. M3 is complete when isolation, cgroups, build-user pooling, network denial, structured logs, and reproducibility enforcement are mandatory rather than experimental.

---

## 15. Store design

### 15.1 Filesystem layout

```text
/kura/
├── store/
├── profiles/
│   ├── system/
│   └── per-user/
├── .staging/
└── .trash/

/var/lib/kura/
├── db.sqlite
├── activation/
├── boot/
├── objects/
├── quarantine/
└── state/

/var/cache/kura/
├── downloads/
├── archives/
└── logs/

/run/kura/
├── daemon.sock
├── builds/
└── transactions/
```

`/kura/store`, `.staging`, and `.trash` must be on the same filesystem so commit and trash renames are atomic. On Btrfs they must also be directories in the same subvolume: rename across Btrfs subvolumes fails with `EXDEV`.

Production installation uses this storage layout:

```text
GPT disk
├── EFI System Partition                 unencrypted FAT
└── LUKS2 system partition
    └── Btrfs filesystem
        ├── @root   → /
        ├── @kura   → /kura
        ├── @var    → /var
        └── @home   → /home
```

`store`, `.staging`, and `.trash` all live beneath `@kura`; none is a nested subvolume. Kura generations replace Snapper as the system rollback mechanism. Optional snapshots may later protect mutable `@var` or `@home`, but they are not part of package/system rollback.

The M5 QEMU release proof uses this LUKS2/Btrfs layout and the same systemd-based initrd architecture as production. It may use a known test passphrase supplied by the harness, but it must exercise the real unlock path. M6 extends hardware coverage and installer/recovery UX; it does not introduce disk encryption for the first time.

### 15.2 Ownership

- `/kura/store` is root-owned and not writable by build users.
- Build users write only staging directories bind-mounted into their sandbox.
- Only `kurad` commits, trashes, or deletes store objects.
- Profile pointer updates are performed through daemon transactions.

### 15.3 SQLite state

Use SQLite in WAL mode with foreign keys and full synchronous durability for state transitions that accompany filesystem renames.

Minimum tables:

```text
derivations
fixed_output_identities
fixed_output_requests
outputs
store_objects
references
realisations
builds
output_locks
roots
leases
profiles
generations
activation_transactions
substitute_records
```

Operational timestamps may exist in the DB but never affect object identity.

### 15.4 Realization lifecycle

```text
unknown
  → planned
  → building | fetching | substituting
  → normalizing
  → committing
  → valid

failure paths:
  → failed
  → quarantined
  → abandoned
```

A per-output lock prevents duplicate builds, fixed-output materializations, or substitutions while allowing multiple clients to subscribe to one realization event stream.

### 15.5 Crash-safe commit

For each output:

1. verify expected output staging mount exists;
2. stop all build processes;
3. normalize files and metadata;
4. perform fixups;
5. scan and validate references;
6. construct canonical archive metadata and content digest;
7. begin DB transaction and record `committing`;
8. fsync files and staging tree;
9. atomically rename staging directory to final store path;
10. fsync `/kura/store`;
11. insert object, references, and realization rows;
12. mark `valid`;
13. commit DB transaction.

A crash between filesystem rename and DB commit leaves an unregistered path. Startup recovery recomputes metadata and either registers it when it matches the journal or moves it to quarantine. It never assumes an unexplained path is valid.

### 15.6 Existing output

When the expected store path already exists:

- validate its store record;
- compare the expected `OutputOrigin` and `OutputId`;
- if a new build produced the same canonical content digest, discard staging and record an equivalent realization attempt;
- if contents differ, quarantine the new output and emit a reproducibility failure;
- never replace the existing valid path.

### 15.7 Reference scanning

Scan every regular file and symlink target for literal store path references matching:

```text
/kura/store/<52 lowercase base32 chars>-
```

The final reference set is:

```text
discovered references
∪ explicit runtime references
∪ references required by output metadata
```

A discovered reference outside the declared input/output closure is a build failure. Explicit runtime references support `dlopen`, plugin search, data lookup, and other references that are not literal bytes.

Reference scanning must be byte-oriented and must not assume UTF-8.

### 15.8 Canonical archive: KAR v1

Kura Archive v1 represents one normalized store object. Entries are sorted by path and contain:

- relative path;
- type;
- mode;
- logical uid/gid;
- normalized mtime;
- file bytes or symlink target;
- hardlink identity;
- sorted allowed xattrs;
- approved capabilities.

The archive stream is hashed before compression. Distribution uses deterministic Zstandard compression, but the semantic content digest covers the uncompressed canonical KAR stream.

Reject absolute paths, escaping links, duplicate paths, unsupported xattrs, and inconsistent hardlinks during import.

### 15.9 Verification

`kura store verify` supports:

- metadata-only validation;
- reference validation;
- full KAR content rehash;
- closure validation;
- repair from trusted substitutes.

Verification never silently changes a valid path. Repair moves a corrupt object to quarantine first.

### 15.10 Roots and leases

Root kinds:

- current profile generations;
- retained previous generations;
- current system generation;
- booted system generation;
- bootloader entries;
- explicit user roots;
- active build inputs/outputs;
- expiring leases;
- release publication roots.

Build leases are created before scheduling and renewed while a client/build owns them.

### 15.11 Garbage collection

GC is mark-and-sweep:

1. snapshot roots and valid references under transaction;
2. mark transitive closure;
3. optionally scan running process executable/maps/fds for additional conservative roots;
4. select unmarked objects;
5. atomically rename each object to `.trash/<transaction>/...`;
6. remove DB validity and references transactionally;
7. delete trash out of the critical section.

Default policy retains at least the current, booted, and several recent generations. Aggressive GC requires explicit user action.

---

## 16. ELF, script, and runtime fixups

### 16.1 Kura-built ELF

For dynamically linked Kura-built ELF files:

- set the ELF interpreter to the exact glibc loader store path;
- set `RUNPATH`, not global `LD_LIBRARY_PATH`, to exact dependency output library directories;
- preserve `$ORIGIN` only when package policy declares it;
- remove build-directory RPATHs;
- reject references to ambient `/usr/lib` unless the package is explicitly marked FHS-runtime-dependent;
- split debug sections deterministically when a `debug` output exists.

Use a Kura-built, versioned ELF fixup tool. Its output ID is part of the derivation.

### 16.2 Scripts

Rewrite shebangs to exact store executables for Kura-built packages:

```text
#!/usr/bin/python
```

becomes an exact selected Python output path, or a generated wrapper when multiple arguments are required.

Reject `/usr/bin/env` in committed Kura-built scripts unless package policy explicitly keeps it for user-controlled environments.

### 16.3 Foreign binaries

Foreign/vendor binaries remain unpatched by default. They run against the generation's FHS tree:

- `/lib` and `/lib64` link into `/usr/lib`;
- `/usr/lib`, `/usr/share`, and related paths come from the composed generation;
- wrappers set only application-specific environment required by the vendor;
- no global `LD_LIBRARY_PATH`.

A vendor package may opt into ELF patching only when redistribution and integrity policy permit it.

### 16.4 Validation

Fixup validation must detect:

- references to `/build`, staging paths, or user homes;
- undeclared store references;
- non-existent interpreters;
- RPATH/RUNPATH outside declared policy;
- world-writable files;
- setuid/capability use without approval;
- absolute symlinks outside allowed FHS targets;
- embedded archive members whose policy requires further scanning.

---

## 17. FHS composition

### 17.1 Composer input

The composer receives ordered semantic inputs but does not use ordering as hidden conflict priority:

```rust
pub struct FhsComposition {
    pub packages: Vec<SelectedOutput>,
    pub files: Vec<SystemFile>,
    pub collision_rules: Vec<CollisionRule>,
    pub generators: Vec<IndexGenerator>,
}
```

### 17.2 Merge rules

- Directories merge recursively.
- A non-directory path with one provider becomes a symlink to that provider's store path.
- Two non-directory providers conflict by default.
- Byte- and metadata-identical files may be co-owned only when the composer can prove identity and records all owners.
- A replacement requires an explicit rule naming path, preferred provider, and replaced provider.
- Wildcard replacement rules are forbidden in v1.
- A directory versus non-directory collision always fails.
- Case-colliding names are rejected even on a case-sensitive host to preserve portability of repository metadata. This rule has no override in v1; such a composition is invalid.

Example diagnostic:

```text
KURA-COMPOSE-004: path collision at /usr/bin/foo
  provider 1: package-a:out → /kura/store/.../usr/bin/foo
  provider 2: package-b:out → /kura/store/.../usr/bin/foo
Add an explicit collision rule; package order is not a priority.
```

### 17.3 Generated indexes

Some FHS artifacts cannot be simple symlinks. Model deterministic generation steps for:

- desktop database;
- MIME database;
- GSettings schemas;
- icon caches;
- linker cache where required for foreign binaries;
- systemd unit enablement links;
- font configuration snippets;
- initramfs module metadata where applicable.

These generators produce a new immutable FHS tree output. Their executable identities and inputs are hashed.

### 17.4 Root tree

A system generation store object contains:

```text
<system>/root/usr
<system>/root/etc
<system>/root/opt
<system>/manifest.kir
<system>/activation.kir
<system>/boot/
```

The preferred `SymlinkRoot` backend creates stable links at installation:

```text
/usr    -> /kura/profiles/system/current/root/usr
/etc    -> /kura/profiles/system/current/root/etc
/opt    -> /kura/profiles/system/current/root/opt
/bin    -> usr/bin
/sbin   -> usr/bin
/lib    -> usr/lib
/lib64  -> usr/lib
```

`/kura` must be available before PID 1. In v1 it lives on the root filesystem. Under `SymlinkRoot`, only `/kura/profiles/system/current` changes during a live filesystem switch.

### 17.5 Root-exposure compatibility gate

The symlink layout is not considered production-safe until an early spike proves it against mechanisms that commonly mount or harden `/usr` and `/etc`. The spike must test at least:

- systemd units using `ProtectSystem=full`, `ProtectSystem=strict`, `ReadOnlyPaths=`, `BindReadOnlyPaths=`, `RootDirectory=`, and dynamic users;
- system and user services before and after a live generation switch;
- Bubblewrap tools whose sandbox plan bind-mounts `/usr`;
- Flatpak application launch, portal helpers, and namespace setup;
- initramfs and early systemd behavior;
- read-only remounts and nested mount namespaces;
- a process that opened the old `/usr` before activation while a new process starts afterward.

The test matrix must run on the oldest and newest supported kernels/systemd versions. It records mount tables and verifies that hardening applies to the intended generation rather than the symlink inode or an unexpected target.

If any required behavior is unreliable, v1 selects `BindMountRoot` instead:

- `/usr`, `/etc`, and `/opt` are real empty mount-point directories;
- the selected generation trees are read-only bind mounts;
- `/kura/profiles/system/current` remains the desired-generation identity and recovery anchor;
- activation uses a journaled mount-switch transaction rather than claiming that one symlink rename alone changes the live root view;
- boot activation mounts one coherent generation before PID 1;
- live activation must either use a proven new-mount-API replacement sequence or be rejected as boot-only when an all-or-nothing root switch cannot be guaranteed.

`ADR-0007` freezes `SymlinkRoot` or `BindMountRoot` only after this spike. No later component may assume the backend directly; it uses the `RootExposureBackend` interface.

---

## 18. Immutable `/etc` and mutable state

### 18.1 System file types

```rust
pub enum SystemFile {
    Text {
        path: EtcPath,
        contents: String,
        mode: FileMode,
    },
    Source {
        path: EtcPath,
        source: StoreFileRef,
        mode: FileMode,
    },
    Symlink {
        path: EtcPath,
        target: AbsolutePath,
    },
    PersistentStateLink {
        path: EtcPath,
        backing: VarPath,
        initializer: StateInitializer,
        mode: FileMode,
        owner: UserGroup,
    },
    RuntimeStateLink {
        path: EtcPath,
        backing: RunPath,
    },
    SecretLink {
        path: EtcPath,
        secret: SecretReference,
    },
}
```

### 18.2 Required mutable mappings

The base system must explicitly account for at least:

- `/etc/machine-id`;
- `/etc/passwd`;
- `/etc/group`;
- `/etc/shadow`;
- `/etc/gshadow`;
- `/etc/resolv.conf`;
- SSH host keys;
- NetworkManager system connections;
- CUPS or other services that mutate configuration;
- hardware calibration state where relevant;
- generated locale/timezone state if users can change it outside a rebuild.

Back persistent files under `/var/lib/kura/etc/...` or the service's canonical `/var/lib/...` path. Back runtime files under `/run/...`. `/etc/mtab` is not mutable state; the generation emits it as an immutable `SystemFile::Symlink` to `/proc/self/mounts`.

### 18.3 Users and groups

System definitions emit `sysusers.d` declarations. Activation:

1. ensures mutable passwd/group/shadow backing files exist;
2. validates root and administrator accounts;
3. runs the generation's exact `systemd-sysusers`;
4. records changes;
5. refuses destructive UID/GID reassignment without an explicit migration.

Human user home directories remain in `/home`.

### 18.4 Temporary and state directories

Services emit `tmpfiles.d` entries for required `/run`, `/var`, cache, log, and state paths. Activation runs the generation's exact `systemd-tmpfiles` with the selected configuration.

### 18.5 Secrets

A secret definition contains only:

- stable secret name;
- target service/path;
- provider type;
- ownership/mode;
- required/optional policy;
- rotation behavior.

Provisioners write bytes to `/run/credentials`, `/run/kura/secrets`, or an explicitly approved `/var/lib` secret store. Prefer systemd credentials for services.

A build, KIR dump, cache object, or `kura diff` must never contain secret bytes.

---

## 19. Profiles and generations

### 19.1 User profile layout

```text
/kura/profiles/per-user/<uid>/<profile>/
├── current -> generations/17
├── generations/
│   ├── 16 -> /kura/store/<hash>-profile
│   └── 17 -> /kura/store/<hash>-profile
└── lock
```

A profile store object contains a manifest and symlink trees such as `bin`, `share`, and optional FHS runtime root.

### 19.2 Profile transaction

1. resolve desired packages;
2. realize all outputs;
3. compose and validate profile object;
4. create next numeric generation link;
5. fsync profile directory;
6. atomically replace `current`;
7. update roots;
8. emit shell integration instructions or notification.

A failed build never changes `current`.

### 19.3 Arch-hosted M4 mode

While Arch hosts the machine:

- Kura does not replace global `/usr` or `/etc`;
- user profile executables use direct store interpreters/RUNPATH;
- profile `bin` is added to the user's path explicitly;
- FHS-only foreign applications launch through `kura run`, which creates a mount namespace using the profile's composed FHS root;
- Omarchy remains at `/usr/share/omarchy`;
- `$OMARCHY_PATH`, UWSM, Hyprland, Quickshell, and user migrations remain unchanged.

Do not solve M4 by exporting a global `LD_LIBRARY_PATH`.

### 19.4 Rollback

`kura profile rollback` selects the previous retained generation and atomically replaces `current`. It does not rebuild.

Generation deletion and GC are separate explicit operations.

---

## 20. System and service model

### 20.1 System definition

A resolved system includes:

```rust
pub struct ResolvedSystem {
    pub name: SystemName,
    pub platform: Platform,
    pub hostname: Hostname,
    pub kernel: SelectedOutput,
    pub initrd: InitrdDefinition,
    pub bootloader: BootloaderDefinition,
    pub packages: Vec<SelectedOutput>,
    pub services: Vec<ResolvedService>,
    pub users: Vec<UserDefinition>,
    pub groups: Vec<GroupDefinition>,
    pub files: Vec<SystemFile>,
    pub state_paths: Vec<StatePathDefinition>,
    pub secrets: Vec<SecretReference>,
    pub hardware: HardwareFacts,
    pub activation_policy: ActivationPolicy,
}
```

### 20.2 Hardware facts

Steel cannot inspect `/sys`, PCI, DMI, or devices.

`kura hardware scan` is a Rust command that emits typed facts:

```text
GPU vendor/device
CPU architecture and level
storage controllers
network devices
audio devices
laptop vendor/model
firmware requirements
virtualization environment
```

The user commits or selects the resulting hardware profile. Repository transforms consume the facts as explicit input.

Auto-detection at installer time produces a reviewed profile before system realization.

### 20.3 Service definition

A service may emit:

- systemd system units;
- systemd user units;
- enablement links;
- D-Bus policy;
- udev rules;
- sysusers declarations;
- tmpfiles declarations;
- sysctl snippets;
- environment.d snippets;
- state paths;
- credential references;
- health checks;
- activation/restart policy.

```rust
pub struct ServiceDefinition {
    pub id: ServiceId,
    pub package_outputs: Vec<SelectedOutput>,
    pub units: Vec<UnitDefinition>,
    pub state: Vec<StatePathDefinition>,
    pub users: Vec<UserDefinition>,
    pub secrets: Vec<SecretReference>,
    pub activation: ServiceActivation,
}
```

### 20.4 Unit generation

Prefer structured unit sections and directives. Allow a raw unit fragment only behind a validated escape hatch.

Validation must catch:

- duplicate sections/directives where invalid;
- executable paths not in the closure;
- undeclared users/groups;
- writable paths not declared as state;
- secret paths embedded directly;
- conflicting unit ownership;
- invalid enablement targets.

### 20.5 Restart classes

Each service declares one:

- `Reload`;
- `Restart`;
- `TryRestart`;
- `RestartIfRunning`;
- `ReexecManager`;
- `RebootRequired`;
- `NoAction`;
- custom typed transition with a health check.

The activation planner computes changes from old and new service manifests, not from package names alone.

---

## 21. System generation selection and activation

### 21.1 Generation selection and root exposure

Use this generation identity layout regardless of root-exposure backend:

```text
/kura/profiles/system/
├── current -> generations/42
├── previous -> generations/41
├── booted -> generations/41
├── generations/
│   ├── 41 -> /kura/store/<hash>-omarchy-system
│   └── 42 -> /kura/store/<hash>-omarchy-system
└── lock
```

`current` is the authoritative desired live generation and recovery anchor. `/usr`, `/etc`, and `/opt` must always expose that same generation.

With `SymlinkRoot`, the activator creates `current.next`, atomically renames it over `current`, and fsyncs the profile directory. One rename changes all three stable links.

With `BindMountRoot`, pointer replacement and mount exposure are a journaled transaction. The activator must never perform unrelated independent bind mounts and then report success. If the selected kernel cannot provide a proven coherent live transition, the planner marks full-system activation `boot`-only.

The implementation of §21.4 is parameterized by `RootExposureBackend`; it must not assume symlinks after `ADR-0007` selects the backend.

### 21.2 Activation modes

Expose:

```text
kura system activate --mode=switch
kura system activate --mode=boot
kura system activate --mode=test
```

- `switch`: atomically update the selected generation, commit the coherent root-view transition through `RootExposureBackend`, and converge services. The planner rejects this mode when the selected backend cannot guarantee a safe live transition.
- `boot`: install/select the generation for the next boot without changing the live current pointer or root mounts.
- `test`: install a boot entry or temporary selection that does not become the persistent default.

The planner may require `boot` when the diff crosses a reboot barrier such as kernel, initrd, dynamic loader, PAM, core systemd, or incompatible mutable-state schema changes. `--force-live` is a developer-only escape hatch and must print the exact risks.

### 21.3 Transaction journal

Journal states:

```text
prepared
boot-artifacts-installed
pre-switch-complete
root-view-switched
services-converging
health-checking
active
degraded
rolled-back
aborted
```

Journal records include old/new generation, mode, planned actions, completed action IDs, boot entry state, and error. Secret values are excluded.

### 21.4 Activation sequence

For a live-safe `switch`:

1. acquire global activation lock;
2. verify new generation and closure;
3. create generation link and GC roots;
4. compute and persist activation plan;
5. provision required secrets;
6. run non-destructive pre-switch checks;
7. ensure users/groups and persistent state backing are viable;
8. ask the selected `RootExposureBackend` to prepare the new coherent root view;
9. commit the backend transition: one `current` rename for `SymlinkRoot`, or the journaled mount transition for `BindMountRoot`;
10. fsync the profile directory, persist mount state where applicable, and record `root-view-switched`;
11. run exact new-generation `systemd-sysusers` and `systemd-tmpfiles` as planned;
12. run `systemctl daemon-reload`;
13. execute reload/restart actions in dependency order;
14. restart `kurad` last;
15. run service and desktop health checks;
16. mark active or degraded;
17. retain old generation as `previous`.

A service failure after the root-view switch does not erase the journal. Automatic rollback is allowed only for a transition explicitly marked rollback-safe. Otherwise Kura reports a degraded generation and offers a deterministic rollback command.

### 21.5 Rollback sequence

1. validate target retained generation;
2. compute reverse service/state barriers;
3. refuse live rollback across an incompatible state barrier unless using boot mode;
4. ask `RootExposureBackend` to prepare and commit the reverse coherent root view;
5. update `current` as part of that backend transaction and persist the rollback journal;
6. daemon-reload and converge services;
7. update boot default as requested;
8. retain the failed generation for diagnosis.

### 21.6 Booted versus current

`booted` identifies the generation selected by the initramfs before PID 1. It remains a GC root even after a live switch.

`kura system status` displays:

```text
booted:  41
current: 42
default boot: 42
state: degraded|active|reboot-required
```

This prevents the false claim that every running process belongs to the current filesystem generation.

### 21.7 Mutable-state barriers

Services can declare:

```rust
pub enum StateCompatibility {
    BackwardCompatible,
    ForwardCompatible,
    Bidirectional,
    MigrationRequired(MigrationId),
    Irreversible(MigrationId),
}
```

An irreversible migration prevents automatic live rollback. Kura must surface this before activation.

---

## 22. Boot and image generation

### 22.1 Limine integration

Keep Limine for continuity with Omarchy.

Each generation owns boot metadata. Deployment copies or installs generation-specific kernel/initrd/UKI artifacts to the ESP under hash-qualified names and writes a boot entry containing the exact system store path.

Conceptual entry:

```text
Omarchy Kura generation 42
  kernel: /EFI/Linux/kura-42-<hash>.efi
  cmdline: rd.luks.uuid=luks-<uuid> root=UUID=<unlocked-btrfs-uuid> rootflags=subvol=@root kura.system=/kura/store/<hash>-omarchy-system
```

Exact command-line syntax is owned by the selected systemd/cryptsetup initrd implementation, not hand-concatenated by package recipes. Boot artifacts are protected by roots until their entries are removed.

### 22.2 M5 systemd-based initrd

M5 uses a systemd-based initrd from the first bootable proof. Do not implement a one-off PID 1 that assumes a plain root filesystem and then replace it in M6.

The initrd contains systemd, udev, `systemd-cryptsetup`, cryptsetup/libcryptsetup, device-mapper support, Btrfs support, the required keyboard and storage drivers, and a small Rust helper named `kura-init-select`.

Boot flow:

1. mount initrd virtual filesystems and start udev;
2. discover the Kura data partition;
3. unlock its LUKS2 container through `systemd-cryptsetup`; v1 requires passphrase/recovery-key unlock, while TPM2/FIDO2 and remote unlock are deferred;
4. mount Btrfs `@root` at `/sysroot`, then mount `@kura`, `@var`, and `@home` at their final paths below `/sysroot`;
5. verify that the requested `kura.system` store path and its closure exist;
6. run `kura-init-select` to select the requested generation as both `current` and `booted` and establish the root-exposure backend frozen by `ADR-0007`;
7. switch root to the selected system's `/usr/lib/systemd/systemd`.

Build required virtio, storage, device-mapper, crypto, input, and Btrfs drivers into the M5 kernel or include them in the initrd. The QEMU harness may supply a fixed test passphrase, but the image must not contain a plaintext key file that bypasses `systemd-cryptsetup`.

### 22.3 VM image

Produce a GPT VM image with the production storage shape:

- an unencrypted EFI System Partition;
- a LUKS2 Kura data partition;
- Btrfs `@root`, `@kura`, `@var`, and `@home` subvolumes inside the unlocked container;
- generation-specific boot artifacts;
- a known test user or first-boot provisioning path;
- no embedded production secrets.

The image manifest, partition sizes, Btrfs layout, selected closure, and installed files are deterministic inputs. LUKS2 formatting uses fresh cryptographic randomness, so the final encrypted ciphertext image is not required to be bit-for-bit reproducible. Tests compare the signed image manifest and installed closure, then boot and unlock the actual encrypted image. A separate unencrypted fixture may exist for fast low-level tests, but it cannot satisfy M5 or release acceptance.

Use QEMU with OVMF in CI. A screenshot or serial-only boot is insufficient for final acceptance; the graphical session must be exercised.

### 22.4 Boot rollback

Keep several bootable generation entries. Selecting an old entry causes the initramfs to set both `current` and `booted` to that generation before PID 1, avoiding a mixed early boot. On an encrypted install, rollback uses the same unlocked LUKS2/Btrfs volume and does not duplicate secret material in a generation.

---

## 23. Binary cache, repository metadata, and build farm

### 23.1 Objects

Publish:

- canonical KIR repository snapshot;
- `Kinfo` realization record;
- compressed KAR archive;
- build attestation;
- source/provenance metadata where distributable;
- release/system manifests.

### 23.2 Kinfo v1

```rust
pub enum KinfoOrigin {
    Derivation {
        derivation: DerivationId,
        output_name: OutputName,
    },
    Fixed {
        kind: FixedOutputKind,
        expected_content: ContentDigest,
        materialization_policy: MaterializationPolicyId,
    },
}

pub struct Kinfo {
    pub version: u16,
    pub origin: KinfoOrigin,
    pub output_id: OutputId,
    pub store_path: StorePath,
    pub content_digest: ContentDigest,
    pub archive_digest: Digest256,
    pub archive_size: u64,
    pub references: Vec<OutputId>,
    pub platform: Platform,
    pub build_attestations: Vec<AttestationRef>,
    pub redistribution: RedistributionPolicy,
}
```

Canonicalize and sign the realization record. For fixed outputs, `origin` binds the expected canonical content and materialization-policy identity without importing acquisition URLs into the path. The archive is also covered by repository target metadata.

### 23.3 Trust metadata

Use a TUF-style role model:

- offline root keys;
- delegated targets/release keys;
- snapshot metadata binding one coherent repository version;
- frequently refreshed timestamp metadata;
- channel delegations for stable, RC, edge, and development;
- key thresholds for production promotion.

Clients pin root metadata and enforce expiration, version monotonicity, target hashes, and snapshot consistency.

A CDN is an untrusted byte transport.

### 23.4 Substitution

For an expected output:

1. obtain current trusted channel metadata;
2. find the exact realization for the expected output ID, whether derivation-backed or fixed;
3. verify Kinfo signature and metadata;
4. verify expected output ID/store path;
5. download archive;
6. verify compressed target digest;
7. decode KAR safely;
8. verify canonical content digest and references;
9. commit through the same store transaction as a local build;
10. record the trusted realization source.

A substitute with the right path name but the wrong origin identity, expected output ID, or content is rejected.

### 23.5 Build workers

Workers receive:

- derivation KIR;
- referenced input identities;
- trusted source/substitute endpoints;
- sandbox policy;
- builder identity.

Workers return:

- KAR archive;
- Kinfo draft;
- structured logs;
- builder/sandbox versions;
- source and input digests;
- in-toto-style attestation data;
- reproducibility result if paired.

Workers do not hold release signing keys.

### 23.6 Reproducibility gate

Critical production outputs must be built on at least two independent clean workers. Promotion requires equal canonical content digests.

For packages not yet reproducible, metadata may explicitly mark `reproducibility = unverified`, but such packages cannot satisfy the final production success criterion unless they are vendor fixed outputs whose upstream bytes are themselves the artifact.

### 23.7 Promotion

Stable, RC, edge, and development are signed references to immutable snapshots. Stable/RC/edge promotion reuses the same KIR and artifacts; it does not rebuild. Development is advanced automatically by repository CI, but every value it names is still an immutable signed snapshot.

### 23.8 Redistribution policy

Every source/output records:

- license;
- source availability;
- redistribution permission;
- vendor origin;
- signature/provenance;
- security status.

Publication refuses artifacts whose policy forbids redistribution. Local recipes may still fetch user-authorized vendor bytes by digest.

---

## 24. Daemon protocol and authorization

### 24.1 Transport

Use a Unix domain socket:

```text
/run/kura/daemon.sock
```

Messages are length-prefixed, versioned CBOR envelopes. Protocol encoding is separate from KIR and may evolve independently.

Handshake:

```rust
ClientHello {
    protocol_min,
    protocol_max,
    cli_version,
    capabilities,
}
ServerHello {
    selected_protocol,
    daemon_version,
    store_version,
    capabilities,
}
```

Support the current and immediately previous protocol version.

### 24.2 Authentication

Read peer credentials from the Unix socket.

Policy:

- read/query/status: any local user;
- build/fetch into shared store: members of `kura-build` or policy-authorized users;
- mutate own user profile: that UID;
- inspect another user's profile: root unless explicitly shared;
- GC unrooted shared store: administrator;
- system activation/rollback/channel root: root or administrator policy;
- cache key and publication operations: dedicated release service, not workstation users.

Use a small explicit authorization module, not command-name string matching.

### 24.3 Request/event model

Long operations return events:

```rust
pub enum Event {
    Accepted { transaction: TransactionId },
    Plan { summary: PlanSummary },
    WaitingForInput { output: OutputId },
    DownloadProgress { ... },
    BuildStarted { ... },
    LogChunk { stream: LogStream, bytes: Vec<u8> },
    OutputCommitted { ... },
    ActivationStep { ... },
    Warning { diagnostic: Diagnostic },
    Finished { result: OperationResult },
    Failed { diagnostic: Diagnostic },
}
```

Clients may disconnect without canceling a shared build. Explicit cancellation releases only that client's interest; the daemon cancels when no owner/lease remains and policy permits.

### 24.4 Daemon upgrade

Activation starts `kura-activator` before switching. The activator:

- holds the transaction journal and result pipe;
- switches the generation;
- converges services;
- restarts `kurad` last;
- waits for the new daemon handshake;
- reports final status.

The CLI can reconnect and query by transaction ID.

---

## 25. Command-line interface

Implement commands in this order.

### 25.1 Evaluation and inspection

```text
kura eval <file-or-attribute>
kura show <attribute>
kura hash <attribute>
kura graph <attribute>
kura explain <attribute-or-store-path>
kura diff <old> <new>
```

### 25.2 Build and store

```text
kura build <attribute>
kura path <attribute>
kura log <build-or-store-path>
kura store info <path>
kura store verify [path]
kura store references <path>
kura store referrers <path>
kura gc
kura gc --dry-run
kura root add|remove|list
```

### 25.3 Profiles

```text
kura profile install <attributes...>
kura profile remove <names...>
kura profile list
kura profile history
kura profile diff <generation-a> <generation-b>
kura profile rollback
kura run <attribute> -- <args...>
```

### 25.4 Systems

```text
kura system build <system>
kura system diff <old> <new>
kura system activate <system> --mode=switch|boot|test
kura system rollback [generation]
kura system generations
kura system status
kura image build <system>
kura hardware scan
```

### 25.5 Distribution

```text
kura channel list|set|update
kura cache info
kura cache publish
kura substitute query
kura repository evaluate
kura release promote
```

### 25.6 Diagnostics

```text
kura doctor
kura transaction show <id>
kura activation recover
kura bootstrap explain
```

All mutating commands support `--dry-run` where meaningful. Dry runs produce the same typed plan used for execution.

---

## 26. Explainability and observability

### 26.1 Structured logs

Every operation has a transaction ID. Persist JSONL events and a rendered text log. Records include semantic IDs, not only display names.

Build logs distinguish stdout/stderr and preserve raw bytes safely.

### 26.2 `kura explain`

Required queries:

- why a package is in a closure;
- why an output rebuilt;
- why a path collision occurred;
- why a service will restart;
- why activation requires a reboot;
- why a substitute was rejected;
- which bootstrap binaries remain trusted;
- why an object is GC-rooted;
- where a definition originated in Steel.

### 26.3 `kura diff`

System diff sections:

```text
packages
FHS paths
systemd units
enabled services
users/groups
state paths
secrets by name only
kernel/initrd/bootloader
hardware facts
activation actions
reboot and rollback barriers
```

### 26.4 `kura doctor`

Checks:

- store and DB consistency;
- profile links and generation roots;
- current/booted/default generation coherence;
- activation journals;
- boot entries and ESP artifacts;
- daemon protocol/version;
- build user cleanup;
- cgroup availability;
- sandbox capability;
- root trust metadata and clock sanity;
- pending state migrations;
- `/usr` and `/etc` link layout.

---

## 27. Testing strategy

### 27.1 Unit tests

Every crate needs unit coverage for validation, parsing, state transitions, and failure paths.

Highest priority:

- name/path sanitization;
- hash domain separation;
- canonical ordering;
- collision detection;
- derivation closure;
- activation state machine;
- archive path safety;
- authorization.

### 27.2 Golden tests

Check in:

- Steel source → package KIR;
- package KIR → derivation KIR;
- KIR bytes and SHA-256;
- store path vectors for both derivation-output and fixed-output domains;
- mirror/revision/lock-location edits that change acquisition KIR but preserve fixed output IDs and downstream derivation IDs;
- materialization-policy changes that necessarily change fixed output IDs;
- BuildSpec renderings;
- Kinfo bytes;
- profile/system manifests;
- systemd unit renderings;
- activation plans;
- KAR archives for small fixture trees.

A formatting or dependency upgrade that changes golden bytes requires an explicit schema/version decision.

### 27.3 Property tests

Use property testing for:

- KIR encode/decode round trips;
- canonical map/set ordering;
- path normalization;
- archive extraction;
- fixed-output invariance under arbitrary mirror ordering and provenance-only edits;
- fixed-output sensitivity to expected-content and materialization-policy changes;
- profile collision commutativity;
- reference graph mark/sweep;
- transaction state transitions;
- protocol framing.

### 27.4 Fuzzing

Fuzz:

- KIR decoder;
- protocol decoder;
- KAR decoder;
- Arch package metadata parser;
- archive extraction;
- store path parser;
- systemd unit parser/renderer;
- evaluator boundary value conversion.

Fuzz targets must run without network and with bounded memory.

### 27.5 Fault injection

`kura-test-support` exposes named crash points around:

- staging fsync;
- store rename;
- DB commit;
- profile generation creation;
- root-exposure backend commit, including `current` rename or mount transition;
- boot artifact install;
- daemon-reload;
- individual service restart;
- daemon replacement.

Integration tests kill processes at every crash point and verify recovery produces one coherent state.

### 27.6 Sandbox tests

Attempt to:

- read host `/etc/shadow`;
- enumerate undeclared store paths;
- write an input;
- escape output mounts;
- create a device;
- access the network;
- fork past pids limit;
- consume excess memory;
- ptrace another build;
- retain a process after cancellation.

Tests pass only when access is denied and cleanup is complete.

### 27.7 Reproducibility tests

Build each M1 package twice:

- fresh staging;
- different build users;
- different temporary path names;
- separate worker VMs;
- varied scheduling/order;
- eventually varied physical workers.

Compare canonical KAR content digests. On mismatch, produce a path- and metadata-level diff.

### 27.8 VM tests

Under QEMU/OVMF:

1. boot generation N;
2. verify systemd reaches graphical target;
3. verify UWSM starts;
4. launch Hyprland and Quickshell;
5. verify terminal;
6. verify NetworkManager connectivity through a test network;
7. verify PipeWire devices/graph;
8. verify portals and notifications;
9. build/select generation N+1;
10. activate or reboot as planned;
11. verify N+1;
12. roll back to N;
13. verify N remains bootable;
14. corrupt a cache download and verify rejection;
15. interrupt activation at injected points and verify recovery.

Use serial logs, systemd state, process assertions, Wayland automation, and screenshots. Do not use screenshots as the sole oracle.

The M5 release matrix creates a LUKS2/Btrfs disk, boots the production systemd-based initrd, unlocks it through the configured test credential path, verifies the `@kura` subvolume constraint, updates, rolls back, and recovers from interrupted activation. Faster unencrypted fixtures may supplement this matrix but cannot replace it.

---

## 28. M1 package prototype

M1 validates Steel, Rust constructors, KIR, resolution, and build-plan lowering. Actual store builds complete in M2.

### 28.1 Port order

1. internal fixture packages;
2. Wayland;
3. Alacritty;
4. UWSM;
5. Neovim;
6. PipeWire;
7. Hyprland;
8. Quickshell;
9. Mesa;
10. NVIDIA;
11. Obsidian as the proprietary desktop application.

Obsidian is selected because it is already part of the current Omarchy package set and exercises vendor-binary policy. Publication remains disabled unless redistribution rights are confirmed.

### 28.2 Package-specific proof

| Package | Main proof | M1 acceptance |
| --- | --- | --- |
| Wayland | simple Meson package, generated protocol data | deterministic source, dependency, outputs, Meson BuildSpec |
| Alacritty | Cargo/vendor dependencies | locked Cargo sources, exact Rust tool input, no network build |
| UWSM | Python packaging/scripts | PEP 517 plan and exact shebang fixup |
| Neovim | CMake and runtime data/plugins | multiple inputs, tests, runtime references |
| PipeWire | Meson, plugins, systemd/user units | service metadata and split outputs |
| Hyprland | CMake, Wayland stack, generated protocols | complex C++ dependencies and desktop runtime |
| Quickshell | CMake/Qt and QML resources | Qt plugins, QML paths, service/session integration |
| Mesa | large Meson graph, LLVM, drivers | feature variants and large dependency closure |
| NVIDIA | fixed vendor archive, userspace/kernel split | proprietary metadata and fixed-output import |
| Obsidian | vendor Electron application | FHS wrapper, desktop file, publication prohibition |

### 28.3 NVIDIA scope

M1 models and imports the vendor archive and userspace outputs. Kernel module build, initramfs integration, module signing, and physical-hardware selection belong to M6. The M5 QEMU proof uses virtio graphics and does not require an NVIDIA kernel module.

### 28.4 Cargo source policy

Cargo packages must use a generated, committed dependency lock translated into fixed-output source objects. `cargo` runs offline. The crates.io index and network are not available in the build sandbox.

### 28.5 M1 exit criteria

- all ten package definitions evaluate;
- invalid variants/dependencies fail at the Rust boundary with source spans;
- KIR bytes and hashes are stable;
- package graphs are inspectable without Steel;
- every build plan contains exact executable/input references;
- no evaluator capability test can read host state;
- `kura diff` explains recipe changes;
- changing only mirrors, revision hints, or lock locations leaves fixed output IDs and downstream derivation IDs unchanged.

---

## 29. M2 local store

### 29.1 Work

- implement source fetch and canonical trees;
- implement Arch lock/import;
- compose bootstrap SDK;
- implement store DB and paths;
- implement BuildSpec interpreter;
- implement minimal Bubblewrap backend;
- implement realization scheduler;
- implement normalization/fixups/references/KAR;
- build M1 package set using foreign toolchain and dependencies.

### 29.2 Exit criteria

- store output paths are known before builds;
- no M1 build reads ambient host `/usr`;
- all outputs are immutable and registered with references;
- repeated realization reuses a valid path;
- differing second realization is reported, never overwrites;
- crash tests pass for commit boundaries;
- `kura store verify` validates every M1 output;
- fixed sources and foreign imports keep the same paths across provenance-only acquisition edits;
- GC preserves declared roots and removes unrooted fixture outputs.

---

## 30. M3 sandboxed reproducible builds

### 30.1 Work

- mandatory isolated namespace plan;
- build-user pool;
- cgroup v2 controls;
- no-network enforcement;
- seccomp/no-new-privileges;
- structured logs and cancellation;
- reproducibility worker pair;
- nondeterminism diff tooling;
- package patches needed for deterministic outputs.

### 30.2 Exit criteria

- sandbox escape test suite passes;
- no normal derivation can reach the network;
- killed builds leave no processes or mounts;
- all M1 packages build twice with equal canonical content digests;
- source fetch is the only network-capable operation;
- output references are a subset of declared closure plus explicit runtime refs.

---

## 31. M4 profiles and generations on Arch

### 31.1 Work

- per-user profile manifests/generations;
- atomic profile switching and rollback;
- GC roots and retention;
- `kura run` FHS namespace for foreign apps;
- Omarchy CLI integration pilot;
- package/profile diff and explain;
- channel mapping without replacing current Omarchy update flow.

### 31.2 Omarchy integration constraints

Current Omarchy separates runtime and settings content across `/usr/share/omarchy`, `/etc`, `/usr/lib`, `/usr/share`, and `/etc/skel`, and has per-user finalization and migrations. M4 must preserve those semantics rather than prematurely translating them into a full Kura system.

Pilot integration should:

- package Omarchy runtime/settings as Kura outputs;
- expose them through a user/developer profile where possible;
- preserve `$OMARCHY_PATH=/usr/share/omarchy`;
- keep current user-home seed/finalize/migration behavior;
- keep Arch's global system and boot path unchanged;
- replace selected package helper calls only in an explicit experimental channel.

### 31.3 Exit criteria

- a developer can install, update, diff, and roll back the M1 profile on an Arch Omarchy machine;
- Arch remains bootable and owns global `/usr` and `/etc`;
- Kura-built apps run without global `LD_LIBRARY_PATH`;
- Obsidian or another foreign app runs through the FHS wrapper;
- profile rollback is one pointer replacement;
- GC does not remove current, previous, or running-profile closures.

---

## 32. M5 bootable Omarchy generation

### 32.1 Toolchain bootstrap stages

#### Stage 0: pinned foreign seed

Use exact imported Arch toolchain, libc, shell, core utilities, Python, Rust, and build tools.

#### Stage 1: Kura-built foundation

Build with stage 0:

- binutils;
- GCC bootstrap/full compiler;
- Linux headers;
- glibc;
- shell and essential utilities;
- make, patch, tar, xz, zstd;
- pkg-config;
- CMake, Ninja, Meson, Python;
- LLVM/Clang where required;
- Rust toolchain from a pinned bootstrap compiler.

#### Stage 2: self-rebuild

Use stage 1 to rebuild the foundation. Then rebuild Kura itself with the Kura-built Rust toolchain.

#### Stage 3: bootstrap comparison

Rebuild critical toolchain packages once more with stage 2 and compare canonical contents. Differences become explicit bootstrap reproducibility work; they may not be hidden.

"Built by Kura" means every runtime toolchain output was produced by Kura from a declared seed. It does not claim a source-only trust chain to machine code.

### 32.2 Base system closure

Build:

- kernel and firmware selection;
- glibc and core userland;
- cryptsetup/libcryptsetup, device-mapper userspace, and Btrfs tools;
- systemd, udev, and the systemd-based initrd;
- D-Bus;
- PAM and login stack;
- NetworkManager;
- PipeWire/WirePlumber;
- graphics stack and Mesa;
- UWSM;
- Hyprland;
- Quickshell;
- portals;
- terminal, browser, and editor;
- Omarchy runtime/settings;
- Limine and boot artifacts;
- `kurad`, CLI, activator, and initramfs.

### 32.3 Omarchy content mapping

Translate the current Omarchy file layout into:

- package outputs for runtime binaries and shell;
- immutable `/etc` files;
- service definitions;
- systemd user/system units;
- `/etc/skel` seed content;
- first-user finalization service;
- existing per-user migrations;
- themes, applications, icons, fonts, and desktop data;
- bootloader and hardware profiles.

Do not drop current user migrations merely because the system generation is immutable. They mutate user homes and remain a distinct mechanism.

### 32.4 Exit criteria

A QEMU/OVMF image:

- boots through Limine;
- unlocks a LUKS2 data partition and mounts the specified Btrfs subvolumes;
- selects an exact Kura generation before host PID 1;
- reaches systemd graphical target;
- starts UWSM, Hyprland, and Quickshell;
- has networking, audio, portals, notifications, terminal, browser, and editor;
- installs generation N+1 without mutating N;
- activates N+1 through the guarantees of the root-exposure backend or stages it for reboot according to the plan;
- rolls back to N;
- keeps both bootable;
- contains no runtime dependency on the ambient Arch host.

---

## 33. M6 binary cache and installer

### 33.1 Work

- production Kinfo/KAR service;
- TUF-style trust metadata;
- build workers and attestations;
- double-build reproducibility gate;
- stable/RC/edge promotion and signed development-channel publication;
- installer image, hardware scan, broader initrd hardware coverage, recovery-key UX, and creation of the already-proven LUKS2/Btrfs layout;
- broader firmware/NVIDIA support;
- generation-aware Limine UI;
- release operations and key rotation;
- disaster recovery;
- CDN publication and cache monitoring.

### 33.2 Installer strategy

Support fresh installs first. An Arch-to-Kura conversion tool is a later project.

Installer flow:

1. boot trusted installer image;
2. scan hardware;
3. select channel/system snapshot;
4. create the EFI System Partition, a LUKS2 system volume, and the Btrfs `@root`, `@kura`, `@var`, and `@home` subvolumes;
5. enroll and verify the configured passphrase/recovery-key path without placing key material in the store or generation;
6. populate the store entirely from signed substitutes;
7. create deployment root exposure and mutable state roots;
8. generate the machine-specific hardware/system definition;
9. install the M5-proven systemd initrd and generation boot artifacts;
10. create/provision the first user;
11. reboot, unlock the encrypted volume, and run acceptance checks.

### 33.3 Exit criteria

- production install requires no local package builds;
- every production object comes from trusted signed metadata;
- corrupt, stale, replayed, or mixed snapshot metadata is rejected;
- stable promotion does not rebuild;
- an encrypted QEMU image and at least one supported physical reference machine install, unlock, update, roll back, and recover from an interrupted update;
- `/kura/store`, `.staging`, and `.trash` are verified to share the `@kura` subvolume on every production install;
- signing-key rotation and cache restore are rehearsed.

---

## 34. Issue-ready work breakdown

Use one issue per item. Do not combine milestone-spanning work in a single pull request.

### Foundation and M1

- **KUR-001** Create Rust workspace, CI, licenses, `AGENTS.md`, retain `PLAN.md` as the normative implementation specification, and add the architecture dependency test.
- **KUR-002** Implement `kura-core` domain types, path/name validation, Base32, and distinct SHA-256 domains for derivation outputs and fixed outputs.
- **KUR-003** Implement KIR v1, canonical fixed-output identity/materialization-policy encoding, decoder, schemas, and golden vectors proving mirror edits do not change output IDs.
- **KUR-004** Implement package/source/dependency/output Rust model and validation.
- **KUR-005** Pin Steel and implement isolated adapter plus Rust-backed opaque values.
- **KUR-006** Implement in-memory module bundle, origin tracking, evaluation limits, diagnostics.
- **KUR-007** Implement Steel prelude and package/build-system macros.
- **KUR-008** Implement repository snapshot, resolver, variants, dependency roles.
- **KUR-009** Implement package-to-BuildSpec lowering and `eval/show/hash/graph/diff/explain`.
- **KUR-010** Port fixtures and the ten M1 package definitions.
- **KUR-SPIKE-001** Test symlinked `/usr`, `/etc`, and `/opt` against systemd hardening, Bubblewrap, Flatpak, and mount namespaces; select `SymlinkRoot` or `BindMountRoot` in `ADR-0007` before system composition begins.

### M2

- **KUR-011** Implement fixed-output identities and acquisition requests, canonical raw-file/tree materialization, mirror-invariance tests, and download helpers.
- **KUR-012** Implement Arch lock format with canonical imported-tree digest, signature verification, fixed-output foreign-package identity, and safe importer.
- **KUR-013** Implement bootstrap FHS SDK composer.
- **KUR-014** Implement store filesystem layout, SQLite schema, roots, and recovery.
- **KUR-015** Implement BuildSpec v1 and `kura-builder`.
- **KUR-016** Implement minimal sandbox and build scheduler.
- **KUR-017** Implement output normalization, ELF/shebang fixups, and reference scanning.
- **KUR-018** Implement KAR v1, store commit, verification, and quarantine.
- **KUR-019** Build and validate all M1 outputs.
- **KUR-020** Implement GC, leases, and process-safe retention policy.

### M3

- **KUR-021** Harden Bubblewrap sandbox and filesystem declaration.
- **KUR-022** Implement build-user pool, cgroup v2, seccomp, cancellation cleanup.
- **KUR-023** Implement structured build events/log storage.
- **KUR-024** Implement reproducibility pair builds and canonical output diff.
- **KUR-025** Make M1 package outputs reproducible and enforce the gate.

### M4

- **KUR-026** Implement profile manifest and FHS/profile composer.
- **KUR-027** Implement per-user generations, atomic switching, rollback, roots.
- **KUR-028** Implement `kura run` FHS namespace.
- **KUR-029** Package Omarchy runtime/settings experimentally.
- **KUR-030** Integrate experimental Omarchy package/profile commands and channels.
- **KUR-031** Complete Arch-hosted end-to-end acceptance suite.

### M5

- **KUR-032** Implement service/system model, unit/file/state validation.
- **KUR-033** Implement immutable `/etc` state-link model, sysusers/tmpfiles, secrets.
- **KUR-034** Implement full FHS system composer and generated indexes.
- **KUR-035** Implement activation planner, journal, `RootExposureBackend`, coherent root-view switch, and rollback.
- **KUR-036** Implement Limine generation deployment and boot roots.
- **KUR-037** Implement the M5 systemd-based initrd, `kura-init-select`, LUKS2/Btrfs VM storage layout, and encrypted QEMU boot acceptance.
- **KUR-038** Build Kura-owned C/C++ foundation toolchain.
- **KUR-039** Build Kura-owned Rust toolchain and rebuild Kura.
- **KUR-040** Port base system and current Omarchy closure.
- **KUR-041** Complete encrypted QEMU graphical update/rollback and interrupted-boot acceptance.

### M6

- **KUR-042** Implement Kinfo and substitute client/server.
- **KUR-043** Implement TUF-style root/targets/snapshot/timestamp metadata.
- **KUR-044** Implement build worker protocol and attestations.
- **KUR-045** Implement reproducibility promotion gate and channel snapshots.
- **KUR-046** Implement the LUKS2/Btrfs installer around the M5-proven initrd, hardware scan, recovery-key workflow, broader hardware coverage, and fresh deployment.
- **KUR-047** Add NVIDIA/kernel/initramfs hardware integration.
- **KUR-048** Production operations, key rotation, disaster recovery, monitoring.

---

## 35. Initial pull-request sequence

The first implementation agent should make these pull requests in order.

### PR 1 — Workspace and guardrails

- workspace and crate skeletons;
- pinned Rust;
- licenses;
- CI for fmt, clippy, unit tests, deny;
- `AGENTS.md`;
- dependency-direction test;
- no functional CLI beyond `kura --version`.

### PR 2 — Core identities

- validated names and paths;
- platform model;
- separate SHA-256/domain separators for derivation outputs and fixed outputs;
- canonical fixed-output materialization policy IDs;
- Base32 store hash;
- store label sanitizer;
- unit/property tests.

### PR 3 — KIR v1

- owned value model;
- fixed-output identity and acquisition-request KIR;
- encoder/decoder;
- CDDL;
- golden vectors;
- diagnostic JSON renderer;
- fuzz target.

### PR 4 — Semantic model

- package, source, dependency, output, build-plan types;
- validation errors and origin sidecar;
- package KIR conversion;
- fixtures without Steel.

### PR 5 — Steel boundary

- exact Steel pin;
- adapter-only dependency;
- allowlisted VM;
- Rust type registration;
- evaluator worker process;
- capability denial tests;
- source-span/error bridge.

### PR 6 — Definition prelude

- Kura module bundle;
- package/source/dependency macros;
- build-system constructors;
- small fixture packages;
- `kura eval` and `kura show`.

### PR 7 — Resolver and derivations

- repository snapshot;
- package lookup;
- variants and dependency roles;
- BuildSpec lowering;
- derivation/output hashes;
- `kura graph`, `hash`, `diff`, `explain`.

### PR 8 — Root-exposure compatibility spike

- conventional-root and symlink-root test images;
- systemd hardening matrix;
- Bubblewrap and Flatpak matrix;
- initrd, daemon-reexec, and live-switch checks;
- measured `ADR-0007` decision selecting `SymlinkRoot` or `BindMountRoot`.

### PR 9 — M1 recipes

- running `hello-kura` fixture at its M1 gate;
- ten package files;
- evaluation tests;
- expected graphs and derivation goldens;
- no store build yet.

### PR 10 — Store skeleton

- `kurad`;
- protocol handshake;
- SQLite schema including fixed-output identities and acquisition requests;
- store paths;
- roots;
- transaction/recovery fixtures.

### PR 11 — Sources and foreign imports

- downloader and acquisition fallback;
- archive safety;
- canonical trees;
- Arch lock/import;
- bootstrap SDK.

Subsequent PRs follow the issue breakdown. No PR should both change a persistent format and add an unrelated package port.

---

## 36. Agent implementation rules

The implementation agent must:

1. read the RFC and this plan before each milestone;
2. create or update an ADR before deviating from a selected architecture;
3. add tests in the same pull request as behavior;
4. use fixture derivations before real packages;
5. keep persistent format changes explicit and versioned;
6. never hide nondeterminism by accepting a new content hash for an existing output ID;
7. never add evaluator or build capabilities merely to make one package pass;
8. prefer a package patch or explicit declaration over a global compatibility hack;
9. preserve structured diagnostics and origin information;
10. keep commits atomic and milestone-local;
11. run focused tests plus the workspace test suite;
12. record unresolved risks in an issue rather than silently weakening validation.

A pull request is not complete when it merely builds on the author's machine. Its acceptance criteria and fault paths must pass.

---

## 37. Required ADRs

Create these before or with the corresponding implementation:

- `ADR-0001`: Rust/Steel trust boundary.
- `ADR-0002`: KIR v1 deterministic encoding.
- `ADR-0003`: store identity domains for derivation outputs and fixed outputs.
- `ADR-0004`: pinned Arch foreign bootstrap.
- `ADR-0005`: Bubblewrap sandbox backend.
- `ADR-0006`: package mini-roots and FHS composition.
- `ADR-0007`: root exposure after the symlink/systemd-hardening spike (`SymlinkRoot` versus `BindMountRoot`).
- `ADR-0008`: immutable `/etc` with typed state links.
- `ADR-0009`: activation modes and mutable-state barriers.
- `ADR-0010`: KAR/Kinfo and TUF-style repository trust.
- `ADR-0011`: Limine, the LUKS2/Btrfs production storage layout, and the M5 systemd-based initrd strategy.
- `ADR-0012`: bootstrap/self-hosting definition.

ADRs state context, decision, rejected alternatives, consequences, and migration path.

---

## 38. Principal risks and mitigations

### 38.1 Steel is pre-1.0

**Risk:** API or behavior changes break evaluation or deterministic output.

**Mitigation:** exact commit pin, one adapter crate, no direct use elsewhere, evaluator golden tests, fork only when required, KIR independence from Steel.

### 38.2 Self-hosting expands the package graph

**Risk:** M5 turns into a complete distribution rewrite before architecture is proven.

**Mitigation:** foreign inputs through M4, x86-64 VM first, staged toolchain, explicit base closure, no physical-hardware requirement until the VM boots.

### 38.3 FHS compatibility creates mixed-generation behavior

**Risk:** running processes use old mapped libraries but new `/usr` paths after a switch.

**Mitigation:** exact RUNPATH for Kura-built software, service restart planning, boot-mode barriers for core changes, `booted` versus `current` reporting, retained old generations.

### 38.4 Immutable `/etc` conflicts with real software

**Risk:** programs expect to write configuration under `/etc`.

**Mitigation:** typed state links, service-specific writable declarations, validation and tests, no ad hoc remounting of the whole `/etc` writable.

### 38.5 Build isolation is security-sensitive

**Risk:** a malicious source escapes and attacks the host/store.

**Mitigation:** audited Bubblewrap backend, root-owned typed policy, build users, namespaces, cgroups, seccomp, no network, dedicated escape suite, no package-supplied raw sandbox flags.

### 38.6 Input-addressed outputs can be nondeterministic

**Risk:** two builders produce different bytes for one expected path.

**Mitigation:** first valid object is never overwritten, mismatches quarantined, double builds, canonical diffs, release reproducibility gate.

### 38.7 Store and activation span DB and filesystem

**Risk:** crashes leave split-brain state.

**Mitigation:** explicit journals, same-filesystem atomic renames, fsync boundaries, startup recovery, kill-at-every-step tests.

### 38.8 Mutable data prevents safe rollback

**Risk:** an application upgrades a database incompatible with the old generation.

**Mitigation:** state compatibility metadata, pre-activation warning, boot-only barriers, explicit migrations, no claim of automatic data rollback.

### 38.9 Proprietary redistribution

**Risk:** the cache publishes NVIDIA or commercial application bytes without permission.

**Mitigation:** first-class redistribution policy, publication denial by default, local fixed-output recipes, legal review before promotion.

### 38.10 Daemon replacement during activation

**Risk:** the component performing the switch replaces itself and loses transaction status.

**Mitigation:** pre-spawned activator, journaled transaction, daemon restart last, protocol reconnect by transaction ID.

### 38.11 Encrypted-root and filesystem topology can fail before PID 1

**Risk:** LUKS2 unlocking, device-mapper, Btrfs subvolume mounting, recovery prompts, or `/kura` availability fails before the selected generation can start host PID 1. A careless layout could also place `.staging` or `.trash` in another Btrfs subvolume and break atomic commit/GC.

**Mitigation:** M5 already uses the production systemd/cryptsetup initrd architecture and an encrypted QEMU disk; every release test exercises passphrase unlock, subvolume mount order, wrong-credential recovery, and `@kura` mount/subvolume validation. M6 broadens hardware and installer UX rather than introducing encryption. Secrets and recovery keys never enter KIR or the store.

### 38.12 Symlinked `/usr`, `/etc`, and `/opt` may violate systemd or sandbox assumptions

**Risk:** `ProtectSystem=`, bind mounts, Bubblewrap, Flatpak, or nested mount namespaces may treat symlinked `/usr` and `/etc` differently from conventional real directories, causing hardening gaps or runtime failures.

**Mitigation:** `KUR-SPIKE-001` is a precondition for system composition; test the full systemd/sandbox matrix on supported kernels; freeze the result in `ADR-0007`; keep all higher layers behind `RootExposureBackend`; fall back to real mount-point directories and a journaled bind-mount backend, making unsafe live transitions boot-only rather than weakening atomicity.

---

## 39. Definition of done for the first Kura release

The release is done only when all conditions below are true.

### Architecture

- KIR v1 and both derivation-output and fixed-output identity domains are frozen and documented, including foreign-package import policy.
- Steel can be removed from a client build path after signed IR is available.
- Every store output can be explained from either its derivation and inputs or its fixed-output identity and acquisition provenance.
- No production package build has network access.
- No store object is mutable.

### Build and store

- M1 packages build reproducibly or are verified fixed vendor outputs.
- Store crash recovery and GC tests pass.
- Substitutes and local builds commit through the same validation path.
- Full store verification passes on a release image.

### System

- One generation identity selects one coherent FHS root view through the backend frozen by `ADR-0007`.
- `/etc` mutable paths are explicitly declared.
- Booted/current/default generations are visible and rooted.
- Activation and rollback survive fault injection.
- Reboot barriers and state barriers are enforced.
- The production installer uses LUKS2/Btrfs and proves encrypted boot, rollback, and same-subvolume store transactions.

### Omarchy

- systemd, UWSM, Hyprland, and Quickshell run unchanged in role;
- Omarchy runtime/settings, themes, commands, and user migrations are present;
- networking, audio, portals, notifications, terminal, browser, and editor pass VM acceptance;
- generation N and N+1 remain separately bootable.

### Distribution

- release metadata rejects stale/mixed snapshots;
- all production artifacts are fetched from the signed Omarchy cache on a fresh install;
- critical outputs pass independent double-build comparison;
- stable promotion reuses immutable artifacts;
- signing-key recovery and rotation procedures are documented and tested.

---

## Appendix A — Representative Rust interfaces

```rust
pub trait CanonicalKir {
    const KIND: KirKind;
    const VERSION: u16;

    fn encode_kir(&self, encoder: &mut KirEncoder) -> Result<(), KirError>;

    fn canonical_bytes(&self) -> Result<Vec<u8>, KirError> {
        let mut encoder = KirEncoder::new();
        encoder.envelope(Self::KIND, Self::VERSION, |e| self.encode_kir(e))?;
        Ok(encoder.finish())
    }
}

pub enum RealizationPlan {
    Derivation(Derivation),
    FixedOutput {
        identity: FixedOutputIdentity,
        request: FixedOutputRequest,
    },
}

pub trait Store {
    fn query(&self, output: &OutputId) -> Result<Option<StoreObjectInfo>, StoreError>;
    fn begin_realization(
        &self,
        plan: &RealizationPlan,
    ) -> Result<RealizationTransaction, StoreError>;
    fn verify(
        &self,
        path: &StorePath,
        mode: VerifyMode,
    ) -> Result<VerificationReport, StoreError>;
    fn roots(&self) -> Result<Vec<Root>, StoreError>;
    fn collect(&self, request: GcRequest) -> Result<GcPlan, StoreError>;
}

pub trait Evaluator {
    fn evaluate(
        &self,
        bundle: ModuleBundle,
        limits: EvaluationLimits,
    ) -> Result<EvaluationResult, EvaluationError>;
}

pub trait Resolver {
    fn resolve_package(
        &self,
        snapshot: &RepositorySnapshot,
        request: PackageRequest,
    ) -> Result<ResolvedPackage, ResolutionError>;

    fn lower(
        &self,
        package: &ResolvedPackage,
    ) -> Result<DerivationGraph, LoweringError>;
}

pub trait ActivationBackend {
    fn prepare(
        &self,
        old: &SystemManifest,
        new: &SystemManifest,
        mode: ActivationMode,
    ) -> Result<ActivationPlan, ActivationError>;

    fn execute(
        &self,
        plan: ActivationPlan,
        journal: &mut ActivationJournal,
    ) -> Result<ActivationResult, ActivationError>;
}
```

---

## Appendix B — Representative Steel system

```scheme
(require kura/prelude)
(require packages/linux)
(require packages/omarchy)
(require services/networking)
(require services/desktop)

(define workstation
  (omarchy-system
    (name "workstation")
    (hostname "workstation")
    (platform x86_64-linux)
    (kernel linux)
    (hardware
      (hardware-profile
        (gpu 'intel)
        (firmware intel-firmware)))
    (packages
      foot
      chromium
      neovim)
    (services
      network-manager
      bluetooth
      pipewire
      printing
      docker
      uwsm
      hyprland
      quickshell)
    (users
      (user
        (name "omarchy")
        (groups "wheel" "video" "audio")))
    (state
      (persistent-state "/etc/NetworkManager/system-connections"
                        "/var/lib/kura/etc/NetworkManager/system-connections"))
    (secrets
      (service-secret
        (name "example-token")
        (service "example.service")))))

(export-system workstation)
```

This code describes values. It does not read hardware, fetch sources, create users, or write files during evaluation.

---

## Appendix C — Activation journal example

```json
{
  "transaction": "01J6...",
  "old_generation": 41,
  "new_generation": 42,
  "mode": "switch",
  "state": "services-converging",
  "root_view_switched": true,
  "root_exposure_backend": "symlink-root",
  "completed": [
    "verify-closure",
    "install-generation-link",
    "prepare-state",
    "switch-root-view",
    "daemon-reload",
    "restart-NetworkManager.service"
  ],
  "pending": [
    "restart-pipewire-user-services",
    "restart-kurad.service",
    "health-check-graphical-session"
  ],
  "reboot_required": false,
  "rollback_barrier": null
}
```

The real journal uses the versioned protocol/KIR representation and excludes secret values.

---

## Appendix D — First executable fixture

Before Wayland, implement this fixture:

```scheme
(define-package hello-kura
  (version "1.0")
  (source
    (local-source "fixtures/hello-kura"
                  #:tree-sha256 "..."))
  (build
    (script-build
      (run bash "-c"
           "mkdir -p $KURA_OUTPUT_OUT/usr/bin &&
            printf '#!/bin/sh\necho hello-kura\n' \
              > $KURA_OUTPUT_OUT/usr/bin/hello-kura &&
            chmod 755 $KURA_OUTPUT_OUT/usr/bin/hello-kura"))))
```

This is a running vertical fixture whose acceptance criteria accrete by milestone. Do not block M1 on M2–M4 behavior.

**M1:**

- Steel returns a typed package;
- package KIR, fixed-output identity, and derivation hash match golden vectors;
- changing the local repository path without changing the expected canonical tree digest leaves the fixed output ID and derivation ID unchanged;
- the shell is represented as an exact selected input in the lowered `BuildSpec`.

**M2:**

- the source materializes as the expected fixed canonical tree;
- output is normalized and committed at the precomputed derivation-output path;
- the shebang is fixed to a selected store shell;
- GC preserves the output while explicitly rooted and deletes it after root removal.

**M3:**

- the build cannot see host `/usr` or the network;
- a clean second build has the same canonical content digest;
- cancellation leaves no process, mount, staging path, or build-user contamination.

**M4:**

- a profile can select, run, update, and roll back the fixture;
- profile roots preserve the closure and profile removal makes it collectible.

This one fixture exercises the complete vertical path before the real package graph obscures defects, while each milestone remains independently finishable.
