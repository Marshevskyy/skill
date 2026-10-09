---
name: software-design
description: >
  Design rules for writing, reviewing, and refactoring code, distilled from
  John Ousterhout's "A Philosophy of Software Design". Use when making a design
  decision: shaping a new module, class, or API; splitting, merging, or
  refactoring code; deciding how to handle errors and special cases; or
  reviewing a PR for design quality. Trigger on phrases like "too complex",
  "how to split this", "review the design", "refactor", "clean this up",
  "interface design", "deep module", "information leakage".
---

# Software design

The only goal is to **manage complexity**: anything in the structure of the
code that makes it hard for a first-time reader to understand or change. It
accumulates from small, individually reasonable decisions, so nothing is too
small to matter. Two moves fight it: **eliminate** (fewer concepts, fewer
special cases) and **encapsulate** (deep modules hiding decisions behind small
interfaces).

## Decide what matters first

An abstraction is a split between the few things that matter (exposed in the
interface) and everything else (hidden). Before designing or reviewing, state
in one sentence what matters here. Something matters if it has **leverage**
(many things depend on it or are simplified by knowing it) or **generality**
(it is used from many places). Keep that set small and make it prominent in
interfaces, names, and the top of files. Two failure modes: exposing so much
that nothing stands out, and burying something important.

## Decision rules

Apply in order when two designs both work.

1. **Fewer things a reader must hold in their head.** A caller should use the
   module without reading its body.
2. **Deeper over shallower.** Value = functionality delivered − interface
   learned. One small interface hiding a lot beats many small pieces. Never
   split a function because of line count.
3. **Simple interface over simple implementation.** Modules have more callers
   than maintainers. Pull complexity into the module instead of pushing it out
   as flags, setup steps, or exceptions.
4. **Define errors and special cases out of existence.** Before a throw or an
   `if (edgeCase)`, redefine the operation so the case is normal (`unset`
   guarantees "not present" rather than failing when absent; no selection is a
   range with `start == end`). Only then mask low, aggregate high, or crash.
5. **Same knowledge in one place.** A decision encoded in two modules: merge
   them, or extract it into a module with a *simpler* interface than either.
6. **General interface, current functionality.** Shape the interface so more
   than one caller could use it; implement only today's need.
7. **Different layer, different abstraction.** Adjacent layers exposing the
   same concept means one should go.

## Red flags

Use these names verbatim; naming the flag is most of the diagnosis.

| Flag | Looks like |
|---|---|
| **Shallow Module** | Interface nearly as complex as what it hides; one-line wrappers. |
| **Information Leakage** | Same design decision visible in two modules. Most expensive to leave in. |
| **Temporal Decomposition** | Split by *when* things run, not by what they know. |
| **Overexposure** | Common path forces callers to learn rare options. |
| **Pass-Through Method** | Forwards arguments, adds nothing. |
| **Repetition** | Non-trivial logic repeated; a missing abstraction. |
| **Special-General Mixture** | Special-case logic inside a general mechanism. |
| **Conjoined Methods** | Cannot be understood separately. |
| **Comment Repeats Code** | A stranger could write it from the code alone. |
| **Implementation Contaminates Interface** | Interface docs say *how*, not *what*. |
| **Vague Name** | `data`, `result`, `info`, `tmp`, `handle`, `process`. |
| **Hard to Pick Name** | Probably two things in one. |
| **Hard to Describe** | A short, complete interface comment is hard to write → design is wrong. The master test. |
| **Nonobvious Code** | A first-time reader's guess is wrong. |

Also from the book: **False Abstraction** (interface omits what callers need,
so they read the body anyway) and **Tactical Tornado** (each patch works, the
design gets worse).

## Comments and names

Default to **no comment**. A comment earns its place only by saying what the
code cannot.

- **Interface comment** on an exported thing when its name and types do not
  already say what it guarantees — units, boundaries, null semantics,
  ownership, side effects. One or two sentences, never the *how*. If it needs
  more, fix the design. Where the surrounding file has no interface comments,
  match it unless the guarantee would otherwise be missed.
- **Body comment** only for a constraint the code cannot express: a
  cross-module or ordering invariant the type system cannot encode (make it a
  type or an assertion first when it can), a deliberate deviation and what it
  protects, a workaround and its cause. Never narrate the next line.
- **Rationale goes in the PR description**, not in code: alternatives
  considered, trade-offs, benchmarks, and how the code differs from before.
- Names are **precise** (guessable in isolation) and **consistent** (one name
  per concept, never reused).

## When generating code

1. Find the concept that deserves a boundary; write its interface comment
   first as a design test. Long or vague means the boundary is wrong.
2. Sketch a second, materially different design. Compare interface size,
   special cases, and what callers must know. Pick one and give the reason in
   your reply or the PR description.
3. Implement the narrowest strong interface. Make every caller simpler, even
   if the implementation grows.
4. Run the review checklist on your own diff before finishing.

## When modifying existing code

1. **Match local conventions.** Anything that looks like a convention is one.
   Do not add a second way of doing something, and do not change a convention
   inside an unrelated change.
2. **Leave the design as if the change had been known from the start.** A
   flag, branch, or parameter added to fit the new case into the old shape is
   how codebases rot. Reshape the module instead, within its conventions.
3. **Stay within the change's blast radius.** If the proper fix reaches
   further, do the minimal clean version and propose the larger refactor
   separately, with the reason.
4. **Comments and names travel with the code.** Everything you touch must
   still be correctly described. If a deadline forces a shortcut, name the
   debt in the PR description.

## Review checklist (for PRs and for your own diff)

Report in this order, using flag names; skip items with nothing to say.

1. **What matters?** Does the interface expose exactly that and hide the rest?
2. **Obvious?** Where would a first-time reader guess wrong?
3. **Deep?** Can the interface comment alone let someone call it?
4. **Leakage?** Is any knowledge encoded twice?
5. **Layers?** Pass-through methods, variables, or decorators?
6. **Special cases?** Any `if (noX)` branch an always-present-X design removes?
7. **Errors?** Can any throw be defined away, masked, or aggregated?
8. **Names and comments?** Vague, inconsistent, or repeating the code?
9. **Direction?** Better or worse than if the requirement had been known up
   front?

State the concrete fix, not just the flag: "merge `parseConfig` and
`validateConfig`; both know the schema", not "information leakage".

## Defaults to override

Mainstream training gives you these instincts. Do not follow them without a
specific reason.

- Splitting a function because it is long. Length is not a reason.
- Extracting a "helper" used once.
- Wrapping a library without simplifying it.
- Adding a boolean mode parameter.
- Adding a config option the code could compute itself.
- Throwing where redefining the operation removes the error.
- Implementation inheritance. Prefer composition.
- Widening or leaking an interface for test convenience.
- Adopting a pattern or framework that does not reduce complexity in *this*
  codebase.

