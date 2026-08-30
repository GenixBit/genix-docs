# Native Intrinsics and Host Services

> Status: pre-alpha bootstrap design.

Genix keeps normal application code platform-neutral. OS-facing standard-library functions are exposed as ordinary Genix APIs, while the compiler recognizes a deliberately small set of canonical stdlib calls as **host intrinsics**.

Users do not write private intrinsic names or C declarations.

## Current public APIs

```gb
import io;
import fs;
import process;

fn main() {
    let name: string = io.input("Your name: ");

    fs.write_text("hello.txt", "Hello " + name);
    let text: string = fs.read_text("hello.txt");
    io.println(text);

    let home: string = process.env("HOME");
    io.println(home);
}
```

Current host-backed APIs:

```text
io.input(prompt: string) -> string
fs.read_text(path: string) -> string
fs.write_text(path: string, text: string)
process.env(name: string) -> string
process.exit(code: int)
```

## Interpreter path

`gb run` recognizes the canonical stdlib function names and performs the equivalent host operation through Rust:

```text
Genix stdlib call
      ↓
interpreter intrinsic dispatch
      ↓
stdin / filesystem / environment / process
```

## Native path

`gb build` keeps the call represented in typed Genix IR. The C11 backend recognizes the same canonical function and lowers it to the Genix Runtime ABI:

```text
io.input      → gb_input
fs.read_text  → gb_fs_read_text
fs.write_text → gb_fs_write_text
process.env   → gb_env_get
process.exit  → gb_process_exit
```

This means the same `.gb` source has matching behavior in interpreted and compiled modes.

## Why this is not the final FFI

The bootstrap intrinsic boundary is intentionally narrow. It exists so the official stdlib can provide essential host functionality before Genix has attributes, foreign declarations, richer result types, and stable package metadata.

A future general FFI should support concepts such as:

```text
native function declarations
library/package linkage
calling conventions
safe/unsafe boundaries
platform conditions
native type mapping
structured build metadata
```

Those capabilities should not be simulated by adding arbitrary compiler special cases. New bootstrap intrinsics should therefore be rare and limited to foundational official stdlib functionality.

## Current error model

Genix does not yet have stable `Result`, `Option`, or exception semantics. Therefore:

- missing environment variables return an empty string;
- native filesystem failures currently panic through the runtime;
- interpreter filesystem failures return a Genix execution error;
- `process.exit` terminates immediately with the requested status.

These behaviors are transitional and should move to structured error types when the language supports them.

## Runtime compatibility

The current host services are additive Runtime ABI v1 symbols. The compiler validates `GENIX_RUNTIME_ABI_VERSION` before native compilation, and the stdlib declares its expected ABI in `COMPATIBILITY`.

---

**Genix — by GenixBit**
