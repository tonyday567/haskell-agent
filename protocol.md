# Haskell Protocol

A release checklist, development standards, and toolkit for Haskell packages. Written for humans and agents alike — same list, same commands.

## Quick Start

Install Haskell via [ghcup](https://www.haskell.org/ghcup/). Install everything.

```bash
ghcup list -c set -r
```

Create a new project:

```bash
mkdir mylib && cd mylib
cabal init --minimal --simple --overwrite --lib --tests --language=GHC2024 --license=BSD-2-Clause -p mylib
cabal build && cabal test
```

Install extra tooling:

```bash
cabal install ormolu hlint hkgr cabal-gild --allow-newer --overwrite-policy=always
```

cabal-docspec (external runner, not on Hackage):

```bash
git clone https://github.com/phadej/cabal-extras
cd cabal-extras/cabal-docspec
cabal install cabal-docspec:exe:cabal-docspec --overwrite-policy=always
```

Tools are expected in `~/.cabal/bin/`. Verify:

```bash
which hlint ormolu hkgr cabal-gild cabal-docspec
```

---

## Conventions

These are the defaults. Deviate when you have a reason, but start here.

**Language & tooling:**
- Language edition: GHC2024
- Cabal stanzas: copy from `~/haskell/numhask-space` (GHC2024 backport, ghc-options)
- CI workflow: copy from `~/haskell/numhask-space/.github/workflows/haskell-ci.yml`
- tested-with: GHC 9.14, 9.12, 9.10 (last three)

**Testing:**
- Doctests in haddocks, no separate test stanza
- Run via `cabal-docspec`

**Import style — development phase:**
- Import whole modules, let the compiler catch name clashes
- Don't maintain tight import lists (`import Data.Foo (bar, baz)`) — they cause cascading bugs
- When using Category/Arrow: `import Prelude hiding (id, (.))`

```haskell
import Prelude hiding (id, (.))
import Control.Category (id, (.))
```

**Export lists — development vs publication:**
- Development: implicit exports (no export list). Compiler warns on unused bindings.
- Publication: explicit export list (`module Foo (bar, baz) where`)
- Adding new code to a published module? Remove the export list first.

```haskell
-- Development (implicit):
module Agent.Fork where

-- Publication (explicit):
module Agent.Fork (PiConfig(..), piChannel) where
```

**Module structure:**

```haskell
-- | One-line summary of module.
--
-- Design philosophy and intentions.
module MyModule where

import Data.Text

-- $setup
-- >>> import MyModule

-- | Haddock for a definition.
--
-- >>> add 2 3
-- 5
add :: Int -> Int -> Int
add x y = x + y
```

- Module header: overview, design philosophy
- Functions/types: clear doc comments
- Aim for 100% haddock coverage
- Setup block for shared doctest imports

Verify haddocks:

```bash
cabal haddock
```

Run doctests:

```bash
cabal-docspec                     # all
cabal-docspec -m Module.Name      # single module
```

---

## House Style

Recurring patterns across ~37 packages in `~/haskell/`. These are emergent conventions, not rules. Full survey at `~/self/buff/haskell-survey.md`.

**Combinators over syntax:**
- `Data.Bool.bool` over if-then-else
- `BlockArguments` for cleaner do-style
- `\case` for total functions on enums

**Names:**
- Predictable qualified aliases: `Data.Text as Text`, `Data.ByteString as BS`, `Data.Map.Strict as Map`, `Data.Sequence as Seq`
- `Prelude qualified as P` when replacing Prelude entirely
- `Union`/`Intersection` newtypes for semigroup choice

**Structure:**
- Category/Arrow/Profunctor as architecture (circuits, box, mealy, mnet)
- Data-driven design: strategy as plain data, sum types for policy
- Pattern synonyms with COMPLETE pragmas for view patterns
- Newtype wrappers with maximal instances

**Optics & labels:**
- `optics-core` with `#field` OverloadedLabels syntax
- Profunctor literacy: `dimap`, `lmap`, `rmap`, `strong`, `choice`

**Display:**
- `prettyprinter` for rendered output
- Module identity: non-mechanical openers (epigraphs, laws, diagrams)

**Prelude replacement:** `NoImplicitPrelude` + explicit imports, or `RebindableSyntax` + custom Prelude. Pick one.

**Mealy:** `Data.Mealy` machines are the signal-processing backbone across mealy, chart-svg, semcons, perf.

---

## Build Performance

`cabal build` is the #1 wait-point for agents. Compilation time is a natural bottleneck.

Mitigations:
- Incremental builds (already default in cabal)
- Caching strategy in CI/local workflows
- Parallel compilation flags
- Consider feature cost vs compile time when designing libraries

---

## Core Library Recipes

Per-library cards as tested recipes. Each card documents one library with import qualifiers, 5-10 essential combinators, common gotchas, and doctest-validated examples.

The workflow:
1. Start a project — `cabal init`, add dependencies
2. Open the recipe card — read the import conventions and patterns
3. Test in a fresh repl — `cabal repl`, import as the card specifies, run examples
4. Discover friction — existing code clashes with introduced encodings
5. Resolve immediately — the card shows the right way

Candidate libraries: containers, text, bytestring, profunctors, kan-extensions, aeson, lens, mtl, unordered-containers, vector.

Cards live in `buff/haskell-<library>.md` or as skills.

---

## Release Protocol

The publishing checklist. Run top to bottom. Each section gates the next.

### 1. Environment Check

⊢ ghcup version check

```bash
ghcup list -c set -r
```

Expected: ghc 9.14+, cabal 3.16+, hls 2.13+

⊢ cabal update

```bash
cabal update
```

⬡ toolchain current → proceed

---

### 2. Initialization

⊢ Capture project state:

◆ repo location and branch
◆ library version in .cabal
◆ github tags (existing releases)
◆ hackage version (current published)

⊢ Write a clear statement of what this release delivers.

**New libraries (0.1.0.0):**
- Skip versioning checks — start at 0.1.0.0
- Skip upstream publishing checks
- Skip or start fresh ChangeLog
- Update tested-with to your supported GHCs
- All other sections apply

---

### 3. Dependency Management

⊢ Inspect cabal.project and cabal.project.local

Default pattern:

```
packages:
  <name>.cabal

write-ghc-environment-files: always
```

**Local dependency pattern:** If referencing local packages, commit their working state to a branch, push, and reference by branch:

```
source-repository-package
  type: git
  location: https://github.com/<user>/<dep>.git
  branch: haskell-9.14-working
```

⊢ Check for outdated deps:

```bash
cabal outdated
cabal gen-bounds
```

**allow-newer:** When transitive deps have stale bounds after a GHC upgrade, read the *first* rejection (not the last), extract `PACKAGE:LIBRARY`, add to cabal.project.

**Canonical minimal block (GHC 9.14).** Most repos only need this:

```
allow-newer:
  tdigest:base
```

The single dominant blocker: `tdigest => base <4.22`. Issue: [phadej/haskell-tdigest#50](https://github.com/phadej/haskell-tdigest/issues/50). When you see `*:base` in allow-newer, it is almost always masking `tdigest:base` — replace the wildcard.

**Ghost pins.** These wildcards were carried across many repos but mask nothing. Test without them:

| Pin | Reality |
|-----|---------|
| `*:template-haskell` | No actual template-haskell blockers found |
| `*:containers` | No actual containers blockers found |
| `*:bytestring` | No actual bytestring blockers found |
| `*:transformers` | No actual transformers blockers found |
| `tree-diff:base` | tree-diff is not a direct dep; transient solver appearance |

**Internal bound rot.** Our own packages accumulate stale upper bounds on GHC 9.14:

| Bound | Fix |
|-------|-----|
| `containers <0.8` | bump to `<0.9` |
| `Cabal-syntax <3.15` | bump to `<3.17` |
| `ghc <9.10` | bump to `<9.15` |
| `time <1.15` | bump to `<1.16` |

**Rationalization workflow:**

1. Branch: `git checkout -b allow-newer-fix`
2. Strip all allow-newer from `cabal.project`
3. `cabal clean && cabal build`
4. Read the **first** rejection
5. Add back the **minimal** specific pin
6. If "unknown package", add local dep paths
7. If internal bounds, bump them in `.cabal`
8. Verify: `cabal build --ghc-options=-Wunused-packages`
9. Commit

**harpie replaces numhask-array.** `numhask-array` is gone. `~/haskell/harpie` is the replacement. Bump bounds accordingly.

Full tracking at `~/self/buff/haskell-allow-newer.md`.

⬡ dependencies resolved → proceed

---

### 4. Code Quality Checks

⊢ Build clean with unused-package warnings:

```bash
cabal clean && cabal build --ghc-options=-Wunused-packages
```

⊢ Format cabal file:

```bash
cabal-gild -m check --input=<package>.cabal
cabal-gild --io=<package>.cabal
```

⊢ Format Haskell (ormolu):

```bash
ormolu --mode check $(git ls-files '*.hs')
ormolu --mode inplace $(git ls-files '*.hs')
```

⊢ Lint:

```bash
hlint .
```

⬡ all checks pass → proceed

---

### 5. Documentation & Testing

⊢ readme.md — should cover:
- Badges (Hackage, CI)
- Title and brief description
- Architecture or design overview
- Usage with code examples
- Related work / references

No org-mode. Plain markdown with code blocks.

⊢ Haddock documentation — aim for 100% coverage:

```bash
cabal haddock
```

⊢ Doctests:

```bash
cabal-docspec
```

⊢ Verify library installs:

```bash
cabal install
```

⬡ docs and tests pass → proceed

---

### 6. Release Preparation

⊢ Update ChangeLog.md with release notes.

⊢ Bump version in .cabal — follow [PVP](https://pvp.haskell.org/).

⬡ version and changelog updated → proceed

---

### 7. Verification

⊢ CI workflow — copy template if not present:

```bash
cp ~/haskell/numhask-space/.github/workflows/haskell-ci.yml .github/workflows/
```

⊢ Update tested-with in .cabal:

```cabal
tested-with:
  ghc ==9.14.1,
  ghc ==9.12.2,
  ghc ==9.10.1
```

⊢ Branch, commit, push:

```bash
git checkout -b feature/release-X.Y.Z
git add -A
git commit -m "release X.Y.Z: changelog, version, CI"
git push origin feature/release-X.Y.Z
```

⊢ Verify CI passes on GitHub Actions:
- hlint
- ormolu
- build (GHC 9.14, 9.12, 9.10 + macos/windows on 9.14)
- cabal-docspec

⊢ Stack checks (optional):

```bash
stack init --resolver nightly --ignore-subdirs
stack build --resolver nightly --haddock --test --bench --no-run-benchmarks
```

⬡ CI green and commits verified → proceed

---

### 8. Publishing

⊢ Merge to main:

```bash
git checkout main
git pull origin main
git merge feature/release-X.Y.Z
git push origin main
```

⊢ Final check on main:

```bash
cabal clean && cabal build && cabal-docspec
```

⊢ Tag:

```bash
git tag vX.Y.Z
```

⊢ Verify package metadata:

```bash
cabal check
```

⊢ Source distribution:

```bash
cabal sdist
```

⊢ Push tag:

```bash
git push origin vX.Y.Z
```

⊢ Publish to Hackage:

```bash
cabal upload --publish dist-newstyle/sdist/<package>-X.Y.Z.tar.gz
```

⊢ Verify on Hackage. If haddocks fail:

```bash
cabal haddock --builddir=docs --haddock-for-hackage --enable-doc
cabal upload -d --publish docs/*-docs.tar.gz
```

◉ package published → release complete

---

## Tool Versions

Current recommended (verify with `ghcup list -c set -r`):
- GHC: 9.14.1 (latest) or 9.12+ (stable)
- Cabal: 3.16+
- HLS: 2.13+
- Stack: 3.9+

---

## Markdown Cards

Cards are design documents — prose and Haskell code together in one file. They're meant to be read, not just tested.

**What cards contain:**
- Prose narrative (design explanation, reasoning)
- Haskell fence blocks — pasteable into `cabal repl`
- Alternative implementations
- Mermaid diagrams

**Principles:**
- Repl-verifiable. Paste code blocks into `cabal repl` and they work.
- Pleasant to read. Not a wall of code.
- Pleasant to copy/paste.
- Not too long. If sprawling, split it.
- Not too polished. Rough edges encourage participation.

**Working with .md cards in repl:**

GHCi only recognizes `.hs` and `.lhs`. Paste code blocks directly. For multiline use `:{` / `:}`.

Cards needing extra packages declare them at the top:

```bash
cabal repl -b yaya     # for cards that need yaya
```

The dependency lives in the command, not in `.cabal`.

Cards live in `examples/` or `buff/`. They are NOT integrated into cabal build stanzas.

---

## Deep Patterns

Cross-library lessons from `~/haskell/`. These aren't conventions — they're structural insights that recur.

### Dual Representations

When a library provides two representations of the same thing (initial and final encoding), know which elimination form goes with which type. `circuits` has `Circuit` (GADT, inspectable) and `Hyper` (final, coinductive). `reify :: Circuit arr t x y -> arr x y` interprets a Circuit. `run :: Hyper a a -> a` ties the self-referential knot. They are not interchangeable — calling `run` on a Circuit is a type error. This is the most common bug in example cards.

General lesson: when there's a dual, name the eliminators distinctly. Don't overload.

### Tensor Awareness

When a type is parametric in its tensor (like `Circuit arr t`), identical constructors mean different things under `(,)` vs `Either`. The compiler won't stop you from using the wrong one — you'll get a puzzling type error deep inside a `Knot` or `reify`. Pin the tensor explicitly with a type annotation.

### Either Convention

`Trace (->) Either` uses `Left` = feedback (continue), `Right` = exit. User-facing code often uses the opposite convention. When a loop behaves strangely — exiting immediately when it should loop, or looping forever — check which branch you're returning. The convention is fixed by the class, not configurable.

### Perf Pattern

For benchmarking compiled Haskell:

1. **Compile, don't interpret.** Repl timing is 10-100x slower and the compilation strategy is different. Always `cabal run` with `-O2`.
2. **Tight loop.** Measure one thing, many times. Read clock, run action, force NF, read clock. No allocation in the measurement.
3. **Warmup.** First measurements are cold (L2 miss, GC nursery). Warmup 500-1000 iterations.
4. **Percentiles, not averages.** Report p50. Averages skewed by GC pauses. The p50 tells you what the hot path actually costs.
5. **NOINLINE on entry points.** Without it, GHC can inline the entire benchmark loop, constant-fold it, and measure nothing.
6. **RTS options.** `+RTS -s` for allocation/GC stats. `-A64M` for large allocation area to minimize GC frequency.

Reference: `~/haskell/circuits-meter/SKILL.md`.

---

## Dev Toolkit

Rough-edged tools for when you're deep in primops or need offline navigation.

### Typed Holes for Primop Integration

When wrapping raw primops, let GHC tell you the type rather than guessing:

```haskell
{-# LANGUAGE MagicHash, UnboxedTuples, ScopedTypeVariables, RankNTypes #-}
import GHC.Exts

data Tag a = Tag (PromptTag# a)

myWrapper :: forall a b. Tag a -> ((IO b -> IO a) -> IO a) -> IO b
myWrapper (Tag t) f = IO (control0# t _hole)
```

Compile with typed holes:

```bash
ghc -fdefer-typed-holes -fno-diagnostics-show-caret myfile.hs
```

GHC outputs the exact type needed. Use this for primops with complex unlifted/lifted boundaries (catch#, control0#, etc.).

### HLS Quick Typecheck

```bash
cd ~/haskell/<project>
haskell-language-server-wrapper typecheck
```

Runs full diagnostics without an editor. ~3.5s warm cache.

### Module Type Check Technique

For refactoring: comment out all top-level type signatures, let GHC infer, compare with the original signatures. Reveals hidden polymorphism, structural flaws, and design-vs-reality gaps.

```haskell
-- myFunc :: [a] -> Int
myFunc = length           -- GHC infers [a] -> Int, matches
```

---

## Default Files

### .hlint.yaml

```yaml
- ignore: {name: Use if}
- ignore: {name: Use bimap}
- ignore: {name: Eta reduce}
```

### .ghci

```
:set -Wno-type-defaults
```

### .gitignore

```
/.stack-work/
/dist-newstyle/
stack.yaml.lock
**/.DS_Store
cabal.project.local*
/.hie/
.ghc.environment.*
/.hkgr/
```

### CI Template

Copy from `~/haskell/numhask-space/.github/workflows/haskell-ci.yml`.

### Cabal Stanzas

Copy from `~/haskell/numhask-space`. Covers:
- GHC2024 backport (with GHC2021 and Haskell2010 fallbacks for older GHCs)
- ghc-options (`-Wall -Wcompat -Wincomplete-record-updates -Wincomplete-uni-patterns -Wredundant-constraints`)
- ghc-options-exe (`-fforce-recomp -funbox-strict-fields -rtsopts -threaded -with-rtsopts=-N`)
