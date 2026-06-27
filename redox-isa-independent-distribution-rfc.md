# RFC: ISA-Independent Package Distribution via Install-Time Compiled Bitcode

**Status:** Draft / Discussion
**Target:** Redox OS package management and toolchain architecture
**Author:** David [draft for internal/Redox circulation]
**Date:** 2026-06-27

---

## Abstract

This RFC proposes that Redox OS adopt **LLVM bitcode as the canonical distribution
format** for packages, with native binaries generated at install time (or first
run) and cached locally. The repository becomes ISA-independent: a single package
build serves x86-64, AArch64, RISC-V, and any future LLVM-supported target without
maintainer action. Native binaries are bound to the producing machine via a local
signing key, giving integrity guarantees without imposing rights-management
restrictions. The model has direct precedent in Wirth's p-code/Juice work, IBM's
System/38 TIMI, Android's ART, and Apple's bitcode-based App Thinning, and offers
specific advantages for long-lived, high-assurance systems facing repeated ISA
migration over their service life.

---

## 1. Motivation

### 1.1 The current packaging model is ISA-coupled

Linux-derived package ecosystems (rpm, deb, and by extension most of the
ports/package infrastructure Redox currently mirrors) bind a package build to a
specific target ISA at build time. This produces several recurring costs:

- Maintainers must build and test separately per architecture (x86-64, armhf,
  arm64, riscv64, ...).
- New architectures start as second-class citizens until enough packages are
  back-filled.
- A package built five years ago cannot benefit from new ISA extensions
  (e.g., newer SIMD width) on hardware that didn't exist when it was built.
- Cross-compilation tooling is a permanent maintenance burden for the packager,
  not the install-time toolchain.

### 1.2 Long-lived and high-assurance systems pay this cost repeatedly

Defense and aerospace programs with multi-decade service lives encounter ISA
obsolescence as a recurring, not one-time, event. Public reporting on the F-22
program, for example, documents multiple mission computer processor transitions
over the airframe's life. Each transition has historically required:

- Re-qualifying (or replacing) the compiler toolchain for the new target,
- Full regression testing of the resulting binaries,
- DO-178C-class re-verification scoped to the new artifacts,
- Schedule and cost impact disproportionate to the fact that the *source*
  software did not change.

Separately, unsupported toolchains on obsolete host platforms (e.g., a vintage
Ada compiler tied to SPARC hardware) become a standing risk: no patches, no bug
fixes, and a hardware-availability problem on top of a software one. The safety
rationale that justified the original toolchain choice does not protect the
program from this decay.

### 1.3 Commercial and general-purpose fragmentation

Outside the defense space, the same problem appears as developer and user
friction: ISVs maintain separate build pipelines per target, app stores carry
multiple binary variants per release, and small/solo developers are effectively
priced out of supporting newer architectures (ARM64 desktop, RISC-V) until
demand justifies the build-farm investment.

---

## 2. Prior Art

This proposal is not novel in concept; it sits in a well-established lineage:

| System | Era | Mechanism |
|---|---|---|
| Wirth's p-code / UCSD p-System | 1971+ | Pascal compiled to a stack-based virtual ISA; one frontend, many interpreters/native back ends |
| ETH Oberon / **Juice** | mid-1990s | Compiled to a portable intermediate form "just above code generation"; final code generation performed at load time — explicitly aimed at network/web software distribution |
| IBM System/38 → AS/400 **TIMI** | 1978+ | Programs compiled to a technology-independent machine interface; translated to native at install/load. Proven at scale during IBM's CISC→PowerPC transition — installed applications transitioned without user or vendor action |
| Java bytecode / JVM | 1995+ | Portable bytecode, primarily JIT'd at runtime rather than AOT-cached at install |
| Android ART (dexopt) | 2014+ | `.dex` compiled to native on install/first-run, cached in `/data/dalvik-cache` |
| Apple App Thinning / bitcode | 2015+ | Server-side recompilation of submitted bitcode for device-specific variants |
| .NET ReadyToRun | 2021+ | IL compiled to native ahead-of-time, cached alongside the assembly |
| Mesa shader cache | ongoing | SPIR-V compiled to native GPU ISA on first use, cached locally |

This proposal is closest in spirit to **TIMI** (install-time AOT, not JIT) and
**Juice** (the IR sits close to the machine boundary, distributed specifically to
decouple software from a hardware target), implemented with **LLVM bitcode** as
the IR rather than a bespoke format, since LLVM already provides mature,
multi-target code generation and is the toolchain Redox already depends on via
Rust.

---

## 3. Design Overview

### 3.1 Canonical artifact

The unit of distribution is **LLVM bitcode**, not a native ELF binary. Rust
(Redox's primary implementation language) already emits this as a natural
compiler intermediate stage; C/C++ via clang likewise.

### 3.2 Install-time compilation

```
Package: myapp.pkg
  ├── myapp.bc        ← LLVM bitcode (the certified/signed artifact)
  └── manifest.toml   ← deps, capabilities, distribution tier, signatures

Install time:
  llc -O2 -march=native myapp.bc -o /usr/lib/native-cache/myapp

Run time:
  Loader checks native-cache first
  Falls back to on-demand JIT/compile if cache missing or invalidated
```

### 3.3 Cache invalidation

The native cache entry is regenerated when any of the following change:

- CPU microarchitecture (new machine, or a machine class with new ISA extensions)
- `llc`/toolchain version (improved codegen available)
- Package or bitcode signature verification failure
- Explicit user/administrator request

### 3.4 AOT vs JIT

Install-time AOT is preferred over first-run JIT for Redox's target hardware
class. Android's experience with ART is instructive here: pure first-run
compilation produced unacceptable latency on constrained hardware, which pushed
Android toward install-time compilation. Capable desktop/server hardware (the
present Redox target) does not need the JIT fallback as the default path, but it
remains useful as a degraded-mode option (e.g., disk space pressure, cache
unavailable).

---

## 4. Security and Trust Model

### 4.1 Layered chain of trust

```
Vendor key      → signs the bitcode artifact
Repo key        → signs the expected native-output hash for a given target
Compiler key     → signs the binary actually produced on this machine
Device/TPM key  → binds the signed binary to this specific hardware instance
```

### 4.2 What each layer defends against

| Layer | Threat addressed |
|---|---|
| Vendor signature on bitcode | Upstream supply-chain compromise (the artifact a vendor can poison is auditable IR, not an opaque native blob) |
| Repo signature on expected hash | Compromised mirror, MITM during download, build-farm compromise (cf. SolarWinds, XZ Utils) |
| Local compiler key on native binary | Post-install tampering — in-place patching, GOT/PLT hooking, rootkit injection into installed executables |
| TPM-bound device key | Binary lifted from one machine and run on another; ties the signing operation to hardware that malware (even with root) cannot replicate |

### 4.3 Verification at load time

The loader checks, before execution:

1. Is the binary signed by a compiler key recognized on *this* machine?
2. Does the signature match the file's current on-disk state?

Any failure triggers recompilation from bitcode (the recovery path is always
available) or rejection of execution.

### 4.4 This is integrity, not rights management

The bitcode package is freely copyable — nothing in this model restricts
distribution or installation on any number of machines. The *native binary
cache* is bound to its producing machine, but this is an optimization artifact
that is useless elsewhere by construction, not a copy-protection mechanism
enforced against the user. There is nothing to defeat, because nothing is being
restricted — the bitcode, the actual portable artifact, was never locked down.

### 4.5 User sovereignty

Critically, the keys in this model belong to the user/operator, not a vendor or
platform owner:

- Verification can be disabled locally.
- The user can supply their own compiler key.
- The user can recompile any package from bitcode with their own flags.
- The user can inspect bitcode before compiling it.
- Running unsigned output is the user's choice to make, not a vendor's to
  forbid.

This is the opposite posture from Secure Boot under a third-party root of
trust, Apple notarization/Gatekeeper, or locked Android bootloaders — and it is
consistent with Redox's existing capability-based, user-directed security
philosophy.

---

## 5. Legacy and Compatibility Tiers

Not all software can or should compile cleanly to bitcode on day one. A tiered
fallback keeps this explicit and contained rather than blocking adoption:

| Tier | Description | Handling |
|---|---|---|
| 1 | Standard C/C++ (GCC-compiled today) | Recompile through clang → bitcode. Default and preferred path. |
| 2 | GCC-only extensions | `DragonEgg` or intermediate translation; flagged explicitly as tier 2 in the repo. |
| 3 | Binary-only / proprietary | Shipped as an architecture-specific native blob (analogous to proprietary driver handling on Linux, or Proton-style compatibility shims). Lives outside the native-cache system entirely. |
| 4 | Heavy inline assembly | Prefer rewriting hot paths with LLVM intrinsics where equivalents exist; otherwise isolate as a small tier-3 shim library. |

Manifest metadata makes the tier explicit per package:

```toml
[package]
name = "someapp"
distribution = "bitcode"      # tier 1 - preferred
# distribution = "native-x86" # tier 3 - legacy blob
# distribution = "source"     # compile on device
```

The repository as a whole remains ISA-independent; legacy packages are an
explicitly contained exception, not a structural compromise of the model.

---

## 6. Applicability and Scope

### 6.1 Where this model is a strong fit

- **Desktop/server Redox targets** generally: ISA-independent repo, automatic
  benefit from new hardware, no per-architecture build silos.
- **Commercial software distribution**: one build pipeline, one signed
  artifact, no architecture-specific app store variants; removes the
  combinatorial maintenance burden (x86-64 / arm64 / riscv64 / musl-vs-glibc)
  that disproportionately hurts small developers.
- **High-assurance systems with long service lives and recurring ISA
  migration** (the F-22-class case): the certified artifact becomes the
  bitcode; an ISA transition becomes a toolchain (compiler backend)
  qualification event rather than a full software re-verification event. This
  is structurally similar to how model-based design tools generate certified C
  from a verified Simulink/Embedded Coder model — the model is verified, the
  generated code is a qualified-toolchain output — applied one layer lower, at
  the IR/native boundary.

### 6.2 Where this model does not apply

Deeply embedded targets (e.g., Cortex-M class) break several of this model's
assumptions and should not be in scope for this RFC:

- Insufficient flash for a two-copy (bitcode + native cache) footprint.
- No realistic on-device compute budget for `llc` at install time; firmware is
  flashed, not installed.
- Determinism requirements: a compile-on-load step introduces output
  variability that conflicts with certification regimes (e.g., DO-178C) that
  require knowing exactly what binary is running. Reproducible-build
  discipline (fixed `llc` flags, deterministic codegen) is necessary if this
  model is ever extended toward certified output, and is a prerequisite, not
  an afterthought.

The clean position for this RFC is: **bitcode distribution is a
desktop/server/host-class OS concept.** Deeply embedded firmware needs a
separate answer and is explicitly out of scope here.

---

## 7. Open Questions / Risks

1. **Trusted base size.** The install-time translator (`llc` + signing
   pipeline) runs with package-level trust and is therefore part of the
   trusted computing base. Its own attack surface needs to be bounded —
   ideally via Redox's existing capability isolation — independent of the
   trust placed in the bitcode itself.
2. **Determinism/reproducibility.** For any future high-assurance application
   of this model, `llc` output must be reproducible given fixed inputs and
   flags, to support the "qualified toolchain → trusted output" certification
   argument in Section 6.1. This should be validated explicitly, not assumed.
3. **Non-LLVM language frontends.** Ada is the obvious case of interest given
   defense-sector prevalence. GNAT-LLVM (an LLVM backend for GNAT) is under
   active development but is not yet a primary, broadly-supported target;
   its maturity directly gates how far this model can be pushed into Ada-based
   programs.
4. **Install-time compilation cost.** Heavier optimization at install time
   (the LLVM IR tradeoff vs. a leaner Juice-style "just above codegen" IR)
   is paid once and cached, which should be acceptable for the target
   hardware class, but should be measured rather than assumed for very large
   packages.
5. **Tier-3/4 software's relationship to the integrity model.** Legacy native
   blobs sit outside the native-cache signing chain by definition; the
   security posture for that tier needs to be specified explicitly rather than
   left implicit.

---

## 8. Relationship to Existing Work

This RFC builds on and formalizes the author's prior Redox proposal advocating
LLVM bitcode as an ISA-independent package format, which has an existing
working proof-of-concept and has had a positive initial reception internally.
This document extends that base proposal with the install-time native-cache
mechanism, the layered signing/trust model, the legacy compatibility tiers, and
the applicability analysis for high-assurance/long-lived systems.

---

## 9. Summary

| Property | Outcome |
|---|---|
| Repository | ISA-independent; one build serves all current and future LLVM targets |
| Install-time step | `llc` compiles bitcode → native, cached locally |
| Trust chain | Vendor (bitcode) → Repo (expected hash) → Local compiler key (native binary) → Device/TPM binding |
| Rights model | Integrity only — bitcode remains freely copyable; native cache is a non-transferable optimization artifact, not a restriction |
| User control | Full — user holds the keys, can disable verification, recompile, or run unsigned at their own risk |
| Best fit | Desktop/server Redox, commercial distribution, long-lived/high-assurance systems facing recurring ISA migration |
| Out of scope | Deeply embedded/firmware targets without install-time compute or flash budget, and without resolved determinism guarantees for certification |

---

## References (background, non-exhaustive)

- N. Wirth, UCSD p-System / p-code, 1971 and follow-on work.
- ETH Zürich, Oberon System / **Juice**, mid-1990s — portable intermediate
  representation for network software distribution.
- IBM System/38 and AS/400 — Technology Independent Machine Interface (TIMI).
- LLVM Project — bitcode format and multi-target code generation
  infrastructure.
- Reproducible Builds project — deterministic build verification.
- Public reporting on F-22 mission computer processor lineage (illustrative of
  recurring ISA migration cost in long-lived defense programs).
