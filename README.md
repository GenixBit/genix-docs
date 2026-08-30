# Genix Documentation

Official documentation and evolving language specification for **Genix**, the `.gb` programming language by **GenixBit**.

> Status: pre-alpha. Language syntax and behavior documented here may change before the first stable release.

## Current documentation

- `control-flow.md` — mutability, comparisons, boolean logic, `if` / `else`, `while`, and lexical block scope
- `functions-and-types.md` — functions, parameters, returns, explicit types, type inference, static checking, and numeric widening

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
- Error handling
- Concurrency
- Standard library reference
- Compiler and CLI usage
- Language specification
- Design decisions and compatibility notes

## Hello, Genix

```gb
fn main() {
    print("Hello from Genix!")
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
| Creator | GenixBit |

## Documentation policy

Experimental proposals should be clearly marked as experimental. Stable behavior should only be described as final after it has corresponding compiler tests and an accepted language specification.

---

**Genix — by GenixBit**
