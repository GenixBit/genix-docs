# Genix Intermediate Representation

Genix IR is the typed, backend-neutral representation produced after parsing, module resolution, and static type checking.

> Status: pre-alpha. The IR is an internal compiler interface and may change frequently before the first stable compiler release.

## Why Genix IR exists

The parser AST reflects source syntax. Native backends should not need to repeat language-semantic work such as type inference, numeric compatibility, or module-name resolution.

The compiler therefore lowers the checked AST into Genix IR:

```text
.gb source
    ↓
Lexer
    ↓
Parser
    ↓
AST
    ↓
Static Type Checker
    ↓
Genix IR
    ↓
Backends
```

Current backend architecture:

```text
Typed Genix IR
    ├── C11 backend         implemented
    ├── LLVM backend        planned
    └── WebAssembly backend planned
```

## What the IR resolves

Genix IR currently records:

- Fully resolved function names
- Typed function parameters and return values
- Explicit variable types, including inferred source variables
- A type on every expression
- Structured `if`, `while`, blocks, calls, and returns
- Explicit safe widening conversions from `int` to `float`

For example:

```gb
fn average(a: float, b: float) -> float {
    return (a + b) / 2.0;
}

fn main() {
    let result: float = average(10, 20);
    print(result);
}
```

The source arguments `10` and `20` are `int` literals. Because `average` expects `float`, IR lowering inserts explicit float casts before backend generation.

Conceptually:

```text
average(cast<float>(10:int), cast<float>(20:int)):float
```

This means a C, LLVM, or WebAssembly backend does not need to rediscover the widening rule.

## Module resolution

Given:

```gb
import math;

fn main() {
    let result = math.twice(21);
    print(result);
}
```

module functions are represented with qualified names such as:

```text
math.add
math.twice
```

Internal calls inside `math.gb` are also rewritten to the same namespace before type checking and IR lowering.

## Inspecting IR

The developer CLI can print typed IR:

```bash
gb ir
```

For another project:

```bash
gb ir path/to/project
```

For a single file:

```bash
gb ir examples/functions.gb
```

The textual output is intended for compiler development and diagnostics. It is not currently a stable serialization format.

## IR data model

The first IR version contains:

```text
Program
└── Function
    ├── typed parameters
    ├── return type
    └── statements
        ├── typed variable definitions
        ├── assignments
        ├── calls
        ├── print
        ├── returns
        ├── if / else
        ├── while
        └── blocks
```

Every expression carries its result type and a backend-neutral expression kind.

Current expression kinds include:

- Integer literal
- Float literal
- Boolean literal
- String literal
- Variable reference
- Function call
- Explicit cast
- Unary operation
- Binary operation

## Backend contract

The C11 backend now consumes `ir::Program`; it does not consume the parser AST directly.

The intended architectural rule is:

> Language semantics belong in frontend analysis and IR lowering. Target-specific representation belongs in backends.

This boundary will allow future backends to share the same checked program semantics.

## Next IR work

Planned work includes:

- Source-span metadata
- Stable symbol IDs instead of string-only symbol names
- Basic blocks / lower-level control-flow representation
- Constant folding
- Dead-code elimination
- Simple function inlining
- Runtime intrinsic representation
- Target-independent string/runtime operations
- IR verification pass
- Optional textual `.gir` serialization for debugging
- LLVM lowering
- WebAssembly lowering

---

**Genix — by GenixBit**
