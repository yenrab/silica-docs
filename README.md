# Why Silica?

[Spec](being_revised/silica-specification.md) · [Spec (additional)](being_revised/silica-specification-additional.md) · [Formal verification](being_revised/silica-formal-verification-specification.md) · [Crypto intro](being_revised/crypto-proposal-introduction.md) · [Crypto details](being_revised/crypto-proposal-details.md) · [Crash containment](being_revised/beam_like_crash_containment_design_notes.md)

---

Silica is a functional systems programming language designed for AArch64. It compiles BEAM languages (Erlang, Elixir, Gleam) to native code while preserving their concurrency model. This document explains the backstory, design decisions, and rationale behind the language.

---

## The Backstory: C to Rust and the BEAM

I am a teacher. I was a teacher before I entered the industry, an unofficial teacher when I was in industry, am currently a teacher, and will be an unofficial teacher when I leave my current position at the end of this year.

When teaching people about BEAM languages—how they are memory-safe and run on a virtual machine called the BEAM—I often get asked a natural, sceptical, and correct question:

> **"What language is the BEAM written in?"**

When I respond with "C," I get sceptical looks and often a sceptical follow-up, rightly or wrongly: *"How can you claim your language is memory-safe if the VM is written in C?"* I would attempt to explain this, but no matter how technical my answer, the questioner never seemed satisfied.

Because of this, I decided to produce a Rust version of the BEAM. I was nearing completion of this Rust task when I had one of those *ah-ha* moments.

**The BEAM is a work of genius and inspiration.** Think of the chips and computers available when the BEAM was first created: one CPU, relatively little RAM, very constrained machines. Yet the creators of the BEAM envisioned something very different—they created a virtual machine that *simulates* the chips we have now. I realized much of the code I was engineering from C to Rust was in the BEAM to achieve this simulation.

My realization? **"Why simulate chips that we now have?"**

The VM codebase could be dramatically reduced if it was streamlined to fit modern multi-core chips with their modern capabilities. I also realized that BEAM languages match fairly well in structure and how apps are engineered with these same multi-core chips.

So, on December 24, 2025, I decided to rework the BEAM to match the current chips we have.

The next question I had was: **"In what language should I write this new thing that I startd calling the BEAM shim?"** I could do it in C, C++, Rust, Go, any language really. It would have to be memory-safe, so C, C++, and the like were out. Rust was then a front-runner, being designed as a systems language and the BEAM shim being a system application.

So I considered Rust. I knew that Rust's compiler is built using LLVM. LLVM—and therefore all languages whose compilers are built with it—is based on the PDP-11 computer model. The PDP-11 is from the late 1960s to 70s and is the machine C was designed to run on. Rust and the other LLVM-based languages are written to the PDP-11 model and then go through build cycles to optimize the code to fit current chips.

My thoughts: We have BEAM languages that syntactically and, when engineered following good BEAM practices, behaviorally fit modern chips. If I shoehorn the BEAM shim into the PDP-11 model and then try to optimize the result back out to fit the chips we have, much could be lost.

My next *ah-ha* moment was: **To do this well, a language was needed.** A close-to-the-chip language. A memory-safe language. A systems language. A new language.

With this understanding, and being well versed in different languages from different paradigms, I decided the language would be declarative and functional to reinforce memory safety. This decision included the option for direct memory access, but only using the memory safety features built into modern chips.

In our day, like it or not, LLMs and other AI tools generate a lot of the code being created. The current, commonly used languages were designed for human cognition. That's why we see a lot of syntactic sugar—a great deal of human inference is built into the syntax of these languages. AIs don't do human-style inference, especially at a source code level, as well as humans. So I decided to make the language as inference-free and as explicit syntactically as I could. This decision led to four principles that drive Silica's design:

1. **The language needs to be strongly typed.**
2. **Terseness is not my friend.**
3. **Over verbosity is not my friend.**
4. **Multiple ways to do any one thing is a drawback.**

These principles, the choice of functional and declarative, and the need to bridge between the BEAM languages' syntax and behavior drive and continue to drive the design and implementation of Silica.

---

## Design Decisions

### Memory Safety: Hardware-First, No Escape Hatches

Silica exposes **only** safe memory chip behaviors. There are no raw pointers, no `unsafe` blocks, no pointer arithmetic.

**Why:** Memory corruption in a VM or runtime is fatal. The BEAM survives because Erlang code cannot express memory-unsafe operations—but the VM itself is written in C. Converting to Rust improves safety, yet Rust still allows `unsafe` for performance-critical code. Silica takes a different path:

- **Memory Tagging Extensions (MTE)** — Hardware-validated tags on every allocation; tag mismatch triggers a deterministic fault instead of silent corruption.
- **Pointer Authentication (PAC)** — Prevents pointer forgery.
- **Region-based ownership** — Safe memory management without garbage collection.
- **Zero unsafe operations** — The language does not support them.

Faults become LP-level crashes (BEAM-style "let it crash") rather than process-wide corruption. The goal is to constrain the fault model so that most failures are recoverable.

---

### Encryption Safety: Compiler-Enforced Constant-Time

Cryptographic code is unsafe by default in most languages. Side-channel bugs (timing leaks, control-flow leaks, secret-dependent branching) slip through code review.

**Why:** Silica's crypto proposal moves safety into the language:

- **Secret vs. public labels** — The compiler tracks secrecy; secrets cannot accidentally become public.
- **Constant-time operations** — Secret comparisons return `CtMask`, not booleans; no timing-based leaks.
- **No secret-dependent branching** — Secrets cannot control `case` expressions, guards, or recursion depth.
- **Protected secret storage** — Zeroization on scope exit, no logging, no accidental copies.

The compiler rejects insecure patterns at compile time. Security shifts from "hope developers follow guidelines" to "the compiler prevents mistakes."

---

### Functional, Declarative Programming

Silica is functional and declarative: immutable data, pattern matching, recursion (no loops), and explicit effect tracking.

**Why:** A declarative, functional language reinforces memory safety. This aligns with BEAM semantics and enables:

- **Actor-based concurrency** — Message passing, no shared mutable state.
- **Effect tracking** — Side effects are declared on sequence blocks (`sequence proc[device_io, concurrency]`); the compiler aggregates them.
- **Predictable control flow** — Easier to reason about, audit, and verify.
- **Crash containment** — "Let it crash" at the lightweight-process level; failures are isolated.

---

### Explicitness Over Terseness: LLM Code Generation

Silica favors explicit syntax over terse shortcuts. Function signatures, bindings, and pattern matches require explicit type annotations. Sequence blocks use `sequence ... produces pure ... end` instead of implicit returns.

**Why:** LLMs and other AI tools generate a lot of code today. Common languages were designed for human cognition—syntactic sugar and built-in inference. AIs don't do human-style inference at the source level as well as humans. Silica is designed to be as inference-free and explicit as possible. Terse, ambiguous syntax leads to:

- Wrong type inference
- Accidental effect leakage
- Unclear control flow

Explicit design choices reduce LLM mistakes:

| Choice | Benefit |
|--------|---------|
| Explicit types (`x: int64 <- 42`) | No inference ambiguity; LLMs and humans see types clearly |
| Distinct operators (`<-` bind, `=` equality, `->` return) | Unambiguous parsing |
| `sequence proc[ε] ... produces pure x end` | Fixed shape; harder to accidentally return effectful calls |
| Concrete types (`OptionInt` not `Option<T>`) | No generic inference complexity |
| Catch-all patterns require types (`_: int64 -> 0`) | Exhaustiveness and clarity |

The goal is **LLM-friendly and human-readable**: code that both machines and humans can parse, analyze, and verify reliably.

Four principles guide this design:

1. **Strongly typed** — No inference ambiguity.
2. **Terseness is not my friend** — Explicit over cryptic.
3. **Over verbosity is not my friend** — Clear without bloat.
4. **One way to do it** — Multiple ways to do any one thing lead to poorly generated code.

---

## Summary

| Concern | Silica's Approach |
|---------|-------------------|
| Memory safety | Hardware (MTE, PAC) + no unsafe escape hatches |
| Encryption safety | Compiler-enforced constant-time, secret labels |
| Concurrency | Functional, actor-based, BEAM-compatible |
| LLM code generation | Explicit types, unambiguous syntax, structured patterns |
| Syntax principles | Strongly typed; neither terse nor verbose; one way to do it |

---

## Related Documents

All of these documents are undergoing revisions as the Silica compiler is being developed.

| Document | Description | Stability |
|----------|-------------|-----------|
| [silica-specification.md](being_revised/silica-specification.md) | Core language specification | Mostly stable |
| [silica-specification-additional.md](being_revised/silica-specification-additional.md) | Additional specification material | Mostly stable |
| [silica-formal-verification-specification.md](being_revised/silica-formal-verification-specification.md) | Formal verification specification | Unstable |
| [crypto-proposal-introduction.md](being_revised/crypto-proposal-introduction.md) | Crypto-safe language primitives | Unstable |
| [crypto-proposal-details.md](being_revised/crypto-proposal-details.md) | Crypto proposal details | Unstable |
| [beam_like_crash_containment_design_notes.md](being_revised/beam_like_crash_containment_design_notes.md) | BEAM-style fault isolation in native binaries | Unstable |



