# Genix Native Compilation

Genix can compile a project into a host-native executable with `gb build`.

> Status: pre-alpha. The current native backend is a bootstrap C11 backend and is expected to evolve.

## Build a project

From a directory containing `genix.toml`:

```bash
gb build
```

For an optimized build:

```bash
gb build --release
```

You may also pass a project path:

```bash
gb build path/to/project
gb build path/to/project --release
```

## Output

For a project named `hello-genix`, a native build currently writes:

```text
build/
├── hello-genix.c
└── hello-genix
```

On Windows-like environments supported by a compatible C compiler, the executable name uses `.exe`.

## Compilation pipeline

```text
.gb project
    ↓
module loading
    ↓
lexing
    ↓
parsing
    ↓
AST
    ↓
static type checking
    ↓
C11 code generation
    ↓
system C compiler
    ↓
native executable
```

The generated C file is intentionally retained in the build directory so the early backend can be inspected and debugged.

## Build profiles

### Debug

```bash
gb build
```

The current backend invokes the C compiler with:

```text
-std=c11 -O0 -g
```

### Release

```bash
gb build --release
```

The current backend invokes the C compiler with:

```text
-std=c11 -O2
```

## Selecting a C compiler

Genix checks the `CC` environment variable first. If it is not set, it tries:

```text
cc
clang
gcc
```

If no compatible compiler is found, `gb build` reports an actionable error instead of silently falling back to interpretation.

## Native type mapping

| Genix type | C11 representation |
|---|---|
| `int` | `int64_t` |
| `float` | `double` |
| `bool` | `bool` |
| `string` | `const char*` |
| no return value | `void` |

## Functions and modules

Genix function names and module-qualified names are converted to safe internal C identifiers. For example:

```gb
math.twice(21)
```

is lowered to an internal generated C function name. This keeps Genix module syntax separate from the backend representation.

## Runtime support

The bootstrap generated runtime currently includes support required by the implemented language subset, including:

- string concatenation
- string comparison
- typed printing
- basic out-of-memory failure handling

This runtime is deliberately small. Long-term runtime services should move into the `genix-runtime` repository with a defined ABI.

## Current limitations

- Builds target the host platform only.
- Cross-compilation and target triples are not implemented.
- A C11 compiler is currently required for native builds.
- String allocation is a bootstrap implementation; the final memory-safety and ownership model is not designed yet.
- The C11 backend is not intended to prevent a future LLVM backend.

## Why start with C11?

The C11 backend gives Genix a small, auditable path from typed AST to a real executable without coupling the language frontend immediately to a large backend framework. It also makes backend behavior easy to inspect during the pre-alpha phase.

Once Genix IR is introduced, backend selection can evolve toward:

```text
Genix AST
   ↓
Genix IR
   ├── C11 backend
   ├── LLVM backend
   └── future WebAssembly backend
```

---

**Genix — by GenixBit**
