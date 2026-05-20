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

**allow-newer:** When transitive deps have stale bounds after a GHC upgrade, read the *first* rejection (not the last), extract `PACKAGE:LIBRARY`, add to cabal.project:

```
allow-newer:
  package-a:base,
  package-b:template-haskell
```

Keep allow-newer minimal. Track upstream issues. Ask if documentation of tracked issues is warranted.

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
