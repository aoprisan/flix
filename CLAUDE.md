# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Flix is a statically typed functional, imperative, and logic programming language that compiles to JVM bytecode. The compiler and standard library are written in Scala 2.13.17, targeting Java 21.

## Build System

The project uses **Mill 1.1.2** (wrapper: `./mill`). JVM heap is set to 4GB in `.mill-jvm-opts`.

### Common Commands

- `./mill flix.compile` — compile the compiler
- `./mill flix.test` — run all tests (uses ScalaTest, ~20 min)
- `./mill flix.assembly` — build a fat JAR
- `./mill flix.testFuzzerSuite` — run fuzzer tests
- `./mill flix.testIDECompletion` — run IDE completion tests
- `./mill flix.testPackageManager` — run package manager tests

### Running a Single Test Suite

Use Mill's test selector: `./mill flix.test <fully.qualified.SuiteName>`

For custom test tasks, the build invokes ScalaTest's runner directly with `-s <suite>` (see `build.mill` for examples).

## Compiler Architecture

The compiler has two main phases orchestrated in `ca.uwaterloo.flix.api.Flix`:

**Frontend (`check()`)** — produces `TypedAst.Root`:
Reader → Lexer → Parser2 → Weeder2 → Desugar → Namer → Resolver → Kinder → Deriver → Typer → EntryPoints → Instances → PredDeps → Stratifier → PatMatch2 → Redundancy → Safety → Terminator → Dependencies

**Backend (`codeGen()`)** — produces JVM bytecode:
TreeShaker1 → Specialization (monomorphization) → LambdaDrop → Optimizer → Simplifier → ClosureConv → LambdaLift → TreeShaker2 → EffectBinder → TailPos → Eraser → Reducer → JvmBackend → JvmWriter → JvmLoader

The frontend supports incremental compilation via `ChangeSet` and cached ASTs. The backend explicitly nulls intermediate ASTs to reduce memory pressure.

**AST progression:** ParsedAst → KindedAst → TypedAst → LiftedAst → ErasedAst → BytecodeAst

Each compiler phase lives in `main/src/ca/uwaterloo/flix/language/phase/`. Each error type lives in `main/src/ca/uwaterloo/flix/language/errors/` and corresponds to a phase.

## Source Layout

- `main/src/ca/uwaterloo/flix/api/` — public API (`Flix.scala`, `Bootstrap.scala`, LSP server)
- `main/src/ca/uwaterloo/flix/language/ast/` — AST definitions, `Type.scala`, `Kind.scala`, `Symbol.scala`
- `main/src/ca/uwaterloo/flix/language/phase/` — all compiler phases (~109 files)
- `main/src/ca/uwaterloo/flix/language/errors/` — error types per phase
- `main/src/ca/uwaterloo/flix/language/phase/jvm/` — JVM bytecode generation
- `main/src/ca/uwaterloo/flix/language/phase/unification/` — type unification
- `main/src/ca/uwaterloo/flix/tools/` — REPL, benchmarking, package manager
- `main/src/library/` — Flix standard library (187 `.flix` files)
- `main/test/` — all tests (Scala test suites + Flix test files)
- `examples/` — example Flix programs

## Code Style (from docs/STYLE.md)

### Scala
- Prefer functional style; local mutability is OK
- Avoid inheritance; prefer algebraic data types
- Use `mapN` over `for ... yield` with `Validation`
- No shadowed or unused variables
- Scalac uses `-Xfatal-warnings`
- Common visitor methods: `visitExp`, `visitPat`, etc.
- Variable naming: abbreviate long names (`eff`, `tparam`); name expressions `exp1`, `exp2`, `exp3`

### Flix
- 4-space indentation
- Doc comments use `///`
- Trait instance order: Eq, Order, ToString
- Subject-last argument order (to support `|>`)
- Avoid unnecessary lambdas (e.g., `List.map(String.toLowerCase)` not `List.map(s -> ...)`)
- Variable names: `o` for Option, `l` for List; type vars: `a`, `b`, `c`; effect vars: `ef`, `ef1`

### JVM Bytecode Generation
- Prefer public fields over getters/setters
- Prefer direct field init over constructor args
- Classes must be final

## Commit Convention

Use [Semantic Commit Messages](https://gist.github.com/joshbuchea/6f47e86d2510bce28f8e7f42ae84c716) with tags: `feat`, `fix`, `docs`, `style`, `refactor`, `test`, `chore`. Summary should be a single clear sentence, all lowercase except names.
