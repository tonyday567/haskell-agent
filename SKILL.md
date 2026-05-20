---
name: haskell-agent
description: Haskell release protocol, development conventions, and deep patterns for the ~/haskell/ ecosystem
---

# haskell-agent

The canonical Haskell protocol — release checklist, house style, dependency management, and structural patterns from ~37 packages in `~/haskell/`.

Load this skill when working on any Haskell package under `~/haskell/` — especially during release prep, new project setup, or when conventions questions arise.

## Source

- Local: `~/other/haskell-agent/protocol.md`
- GitHub: `https://github.com/tonyday567/haskell-agent`
- Symlink: `~/mg/buff/haskell-protocol.md` → the above

## When to load

- Starting a new Haskell project
- Preparing a release (the full 8-phase checklist)
- Resolving dependency issues (allow-newer, bound rot)
- Checking conventions (import style, export lists, module structure)
- Benchmarking compiled Haskell (perf pattern)
- Working with dual representations (Circuit/Hyper, run vs reify)
- Setting up CI for a Haskell package

## Protocol sections

1. **Quick Start** — ghcup, cabal init, tooling install
2. **Conventions** — imports, exports, module structure, doctests
3. **House Style** — emergent patterns from ~37 packages (bool over if, import aliases, Category-based architecture, etc.)
4. **Build Performance** — cabal build as #1 wait-point
5. **Core Library Recipes** — per-library cards as tested recipes
6. **Release Protocol** — 8 gated phases: Environment → Init → Dependencies → Code Quality → Docs/Testing → Release Prep → Verification → Publishing
7. **Markdown Cards** — design docs with repl-verifiable code blocks
8. **Deep Patterns** — dual representations, tensor awareness, Either convention, perf pattern
9. **Dev Toolkit** — typed holes, HLS typecheck, module type check
10. **Default Files** — .hlint.yaml, .ghci, .gitignore, CI template

## Key references within protocol

- **allow-newer** — canonical minimal block is `tdigest:base`; ghost pins table; internal bound rot; rationalization workflow
- **harpie replaces numhask-array** — bump bounds accordingly
- **Either convention** — `Left` = feedback (continue), `Right` = exit; fixed by `Trace` class
- **Perf pattern** — compile don't interpret, tight loops, percentiles not averages, NOINLINE on entry points
- **Dual representations** — `run` for Hyper, `reify` for Circuit; not interchangeable

## Companion files

- `~/self/buff/haskell-survey.md` — haskell-survey: full idiom mining, hardening log, ~37 library surveys
- `~/self/buff/haskell-allow-newer.md` — upstream blocker tracking, repo-by-repo status
- `~/haskell/circuits/SKILL.md` — circuits-specific agent field guide
- `~/haskell/circuits-meter/SKILL.md` — perf benchmarking conventions
