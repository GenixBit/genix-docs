# Genix Standard Library

The official standard library is maintained in `genix-stdlib` and is loaded by the Genix compiler as ordinary `.gb` modules.

> Status: pre-alpha / stdlib 0.0.1.

## Setup during source development

Point the compiler at a stdlib checkout or installation:

```bash
export GENIX_STDLIB=/path/to/genix-stdlib
```

Native builds also require the runtime:

```bash
export GENIX_RUNTIME=/path/to/genix-runtime
```

## Import resolution

Given:

```gb
import math;
```

Genix searches in this order:

1. A project-local `math.gb` beside the project entry source.
2. `GENIX_STDLIB/modules/math.gb`.

Project-local modules intentionally take precedence.

## Compatibility metadata

`genix-stdlib/COMPATIBILITY` currently contains:

```text
GENIX_STDLIB_VERSION=0.0.1
GENIX_LANGUAGE_VERSION=0.0.1
GENIX_RUNTIME_ABI=1
```

The compiler validates the stdlib language version and runtime ABI before loading an official module.

## `io`

```gb
import io;

fn main() {
    io.println("Hello");
    io.print_int(42);
    io.print_float(3.14);
    io.print_bool(true);
}
```

Current API:

```text
io.println(text: string)
io.print_int(value: int)
io.print_float(value: float)
io.print_bool(value: bool)
```

The first version is implemented in Genix on top of the language `print(...)` primitive. Input APIs are deferred until the native intrinsic/FFI contract is defined.

## `math`

Current API:

```text
math.abs_int(value: int) -> int
math.abs_float(value: float) -> float
math.min_int(a: int, b: int) -> int
math.max_int(a: int, b: int) -> int
math.clamp_int(value: int, minimum: int, maximum: int) -> int
math.square(value: float) -> float
```

Example:

```gb
import math;

fn main() {
    let distance: int = math.abs_int(-42);
    let bounded: int = math.clamp_int(120, 0, 100);
    let area: float = math.square(12.0);

    print(distance);
    print(bounded);
    print(area);
}
```

## `string`

Current API:

```text
string.concat(left: string, right: string) -> string
string.equals(left: string, right: string) -> bool
string.not_equals(left: string, right: string) -> bool
```

Example:

```gb
import string;

fn main() {
    let message: string = string.concat("Hello ", "Genix");
    print(message);
    print(string.equals(message, "Hello Genix"));
}
```

## Architecture

Most standard-library APIs should remain ordinary Genix code:

```text
Application
    ↓
Genix stdlib `.gb` modules
    ↓
Language primitives / compiler intrinsics
    ↓
Genix Runtime ABI
    ↓
Operating system
```

Platform-independent logic belongs in `.gb` modules. OS access and operations that require native services should cross a documented intrinsic/FFI boundary into `genix-runtime`.

## Planned next modules

After the native intrinsic/FFI layer is available, the next useful additions are expected to include:

- `io.input`
- `process`
- `fs`
- `path`
- `time`
- `collections`
- `json`
- `net`
- `http`
- `test`

APIs remain experimental until covered by compiler, runtime, and stdlib compatibility tests.
