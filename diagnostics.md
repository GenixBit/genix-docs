# Genix Compiler Diagnostics

Genix compiler diagnostics are designed to be stable, source-aware, and useful from the command line, editors, CI systems, and future language-server tooling.

> Status: pre-alpha. Error codes are now part of the developer-facing diagnostics contract, but individual wording may still improve before Genix 1.0.

## Diagnostic format

A compiler error can include:

```text
error[E0201]: initializer for 'age' expected int, found string
 --> src/main.gb:2:20
   |
 2 |     let age: int = "twenty";
   |                    ^^^^^^^^ type mismatch
  = help: change the expression or annotation so the types are compatible
```

The diagnostic model contains:

- stable error code
- concise message
- source filename
- line and column
- source span
- primary label
- optional help text

Lexer and parser errors use exact token/source spans. The first diagnostics implementation maps existing type-checker failures back to the most relevant source location through the frontend diagnostics adapter. Expression-level typed source maps will refine this further in later compiler revisions without changing the public error-code model.

## Error-code families

### Lexer — `E000x`

| Code | Meaning |
|---|---|
| `E0001` | Unexpected character or invalid single-character operator |
| `E0002` | Unterminated string literal |
| `E0003` | Invalid numeric literal |

Example:

```gb
fn main() {
    @
}
```

produces an `E0001` diagnostic at the invalid character.

### Parser and syntax — `E010x`

| Code | Meaning |
|---|---|
| `E0100` | General syntax error or expected expression/identifier/token |
| `E0101` | Invalid or unknown type syntax |
| `E0102` | Unsupported `Option` / `Result` type shape |
| `E0103` | Invalid `match` pattern |
| `E0104` | Invalid `Some` / `Ok` / `Err` / `None` constructor use |

Parser diagnostics retain the exact token span and the source filename supplied by the frontend.

### Type checking — `E020x`

| Code | Meaning |
|---|---|
| `E0201` | Type mismatch or incompatible operator operands |
| `E0202` | Undefined variable or function |
| `E0203` | Mutation of an immutable binding |
| `E0204` | Invalid, missing, or non-guaranteed return |
| `E0205` | Invalid, duplicate, or non-exhaustive `match` handling |
| `E0206` | Invalid `?` error propagation |
| `E0207` | Function-call signature / argument mismatch |
| `E0208` | Invalid value use, wrapper context, or `void` use |
| `E0209` | Duplicate declaration, parameter, or function |
| `E0210` | Invalid or missing `fn main()` entry point |
| `E0299` | Unclassified pre-alpha type-checker error |

`E0299` is a compatibility fallback. New recurring error classes should receive a dedicated stable code instead of relying on the fallback.

## Type mismatch example

```gb
fn main() {
    let age: int = "twenty";
}
```

The CLI reports `E0201` and points at the initializer value.

## Undefined names

```gb
fn main() {
    print(username);
}
```

Undefined variables and functions are classified as `E0202`.

## Mutability

```gb
fn main() {
    let count = 0;
    count = 1;
}
```

Reassigning an immutable `let` binding is `E0203`; the help text recommends `mut` when reassignment is intentional.

## Match exhaustiveness

```gb
fn main() {
    let value: Option<string> = None;

    match value {
        Some(text) => {
            print(text);
        }
    }
}
```

An `Option<T>` match must handle both `Some(...)` and `None`; a `Result<T,string>` match must handle both `Ok(...)` and `Err(...)`. Violations are `E0205`.

## `?` propagation

`?` diagnostics use `E0206`. The operator currently requires a `Result<T,string>` expression and a surrounding function that also returns a compatible `Result`.

## CLI behavior

These commands use the diagnostics renderer for direct `.gb` source files:

```bash
gb check src/main.gb
gb run src/main.gb
gb ir src/main.gb
```

The project/module system continues to share the same lexer, parser, and type checker. Source mapping for merged multi-file type-checking will be expanded as the module source-map layer matures.

## CI contract

`genix-lang` CI intentionally compiles invalid programs and verifies that diagnostics include:

```text
error[E....]
filename:line:column
^
help:
```

The suite currently checks representative lexer, parser, and type errors so diagnostics cannot silently regress to unstructured strings.

## Design direction

The diagnostics model is intended to support future:

- warnings and lints
- multiple labels per diagnostic
- secondary source locations
- notes and suggestions
- machine-readable JSON diagnostics
- IDE/LSP diagnostics
- quick fixes
- macro / generated-source traces if those language features are added

The executable AST and Genix IR remain independent from CLI rendering concerns. Source metadata belongs to the frontend/source-map layer so interpreters and backends do not need to carry presentation-specific diagnostic state.

---

**Genix — by GenixBit**
