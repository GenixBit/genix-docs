# Genix Documentation

Official documentation and evolving language specification for **Genix**, the `.gb` programming language by **GenixBit**.

> Status: pre-alpha. Language syntax, IR, and compiler behavior documented here may change before the first stable release.

## Current documentation

- `control-flow.md` — mutability, comparisons, boolean logic, `if` / `else`, `while`, and lexical block scope
- `functions-and-types.md` — functions, parameters, returns, explicit types, type inference, static checking, and numeric widening
- `projects-and-modules.md` — `genix.toml`, `gb new`, multi-file projects, imports, module namespaces, and project checking
- `intermediate-representation.md` — typed Genix IR, lowering, explicit casts, backend contract, and `gb ir`
- `native-compilation.md` — native `gb build`, C11 backend, debug/release builds, type mapping, and compiler requirements

## Current project flow

```bash
gb new hello-genix
cd hello-genix
gb run
gb check
gb ir
gb build
./build/hello-genix
```

Optimized build:

```bash
gb build --release
```

## Current compiler pipeline

```text
Genix project
    ↓
module loader
    ↓
lexer
    ↓
parser
    ↓
AST
    ↓
static type checker
    ↓
typed Genix IR
    ├── C11 backend → native executable
    ├── LLVM backend → planned
    └── WebAssembly backend → planned
```

`gb run` currently executes the checked AST through the interpreter. `gb build` lowers the checked AST to typed Genix IR and sends that IR to the native C11 backend.

## Genix IR

The IR is the compiler boundary between language semantics and target-specific backends. It currently records resolved function names, typed variables, typed expressions, structured control flow, and explicit safe `int → float` casts.

Inspect it with:

```bash
gb ir
gb ir path/to/project
gb ir examples/functions.gb
```

## Documentation roadmap

This repository will continue to cover:

- Getting started
- Language tour
- Syntax and grammar
- Type system
- Variables and constants
- Control flow
- Functions
- Modules and imports
- Genix IR and optimization passes
- Native compiler backends
- Target triples and cross-compilation
- Error handling
- Concurrency
- Standard library reference
- Compiler and CLI usage
- Language specification
- Design decisions and compatibility notes

## Hello, Genix

```gb
fn main() {
    print("Hello from Genix!");
}
```

## Typed function example

```gb
fn add(a: int, b: int) -> int {
    return a + b;
}

fn main() {
    let result: int = add(10, 20);
    print(result);
}
```

Genix source files use the `.gb` extension.

## Core identity

| Item | Value |
|---|---|
| Language | Genix |
| Source extension | `.gb` |
| Primary CLI | `gb` |
| Compiler name | `gbc` |
| Intermediate representation | Genix IR |
| Creator | GenixBit |

## Documentation policy

Experimental proposals should be clearly marked as experimental. Stable behavior should only be described as final after it has corresponding compiler tests and an accepted language specification.

---

**Genix — by GenixBit**
