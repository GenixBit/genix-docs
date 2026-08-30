# Genix Runtime ABI

Native Genix programs use a separate low-level runtime provided by the `genix-runtime` repository.

> Status: pre-alpha. Current runtime ABI version: **1**.

## Compiler/runtime boundary

```text
.gb source
   ↓
parser + type checker
   ↓
typed Genix IR
   ↓
native backend
   ↓
generated application code
   +
Genix Runtime ABI
   ↓
native executable
```

The compiler does not embed its own private string/output runtime anymore. Generated native code includes:

```c
#include <genix/runtime.h>
```

and links against the runtime implementation.

## Runtime discovery

For development, set:

```bash
export GENIX_RUNTIME=/path/to/genix-runtime
```

Then:

```bash
gb build
gb build --release
```

The runtime directory must contain:

```text
include/genix/runtime.h
src/runtime.c
```

## ABI v1 services

Lifecycle:

```c
void gb_runtime_init(void);
void gb_runtime_shutdown(void);
```

Memory and failure handling:

```c
void* gb_alloc(size_t size);
_Noreturn void gb_panic(const char* message);
```

Strings:

```c
char* gb_string_concat(const char* left, const char* right);
bool gb_string_equal(const char* left, const char* right);
```

Output:

```c
void gb_print_int(int64_t value);
void gb_print_float(double value);
void gb_print_bool(bool value);
void gb_print_string(const char* value);
```

## Generated lifecycle

A native Genix executable currently follows this structure conceptually:

```c
int main(void) {
    gb_runtime_init();
    gb_fn_main();
    gb_runtime_shutdown();
    return 0;
}
```

## Memory model status

The bootstrap runtime tracks `gb_alloc()` allocations and releases them during runtime shutdown. String concatenation uses this allocator.

This solves immediate lifetime management for generated temporary strings but is **not the final Genix ownership model**. Future language work may introduce owned strings, reference-counted values, regions, or another explicit memory-safety model.

## Primitive ABI mapping

| Genix | Runtime/native ABI |
|---|---|
| `int` | `int64_t` |
| `float` | `double` |
| `bool` | `bool` |
| `string` | `const char*` |
| void | `void` |

## Compatibility

During pre-alpha development, compiler and runtime changes may be breaking. Public ABI changes should increment `GENIX_RUNTIME_ABI_VERSION` and update both repositories together.

Compiler CI validates against the current external runtime repository and executes generated native binaries to catch ABI drift.

---

**Genix — by GenixBit**
