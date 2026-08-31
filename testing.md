# Testing in Genix

Genix includes a first-class test command for interpreter-based development testing:

```bash
gb test
```

The current test system is part of the pre-alpha developer toolchain. Test syntax is handled by the `gb test` frontend and does not participate in normal application builds or Genix IR generation.

## Project layout

`gb new` creates a starter test automatically:

```text
hello-genix/
├── genix.toml
├── src/
│   └── main.gb
└── tests/
    └── smoke.gb
```

The generated test is:

```gb
test "arithmetic works" {
    assert(2 + 2 == 4);
}
```

`gb test` recursively discovers `.gb` files under `tests/`.

## Test blocks

A test is declared with a quoted name and a block:

```gb
test "addition works" {
    assert(10 + 20 == 30);
}
```

Multiple tests may exist in one file:

```gb
test "addition works" {
    assert(2 + 2 == 4);
}

test "strings compare" {
    assert("Genix" == "Genix");
}
```

Test names are shown directly in runner output, so names should describe behavior rather than implementation details.

## Assertions and explicit failures

### `assert(condition)`

`assert` requires a boolean Genix expression:

```gb
test "comparison works" {
    let value: int = 12;
    assert(value > 10);
}
```

A false assertion fails the current test with test diagnostic `T0001` and points at the original assertion expression.

```text
✗ comparison works
error[T0001]: assertion failed
 --> tests/math.gb:3:12
   |
 3 |     assert(value > 20);
   |            ^^^^^^^^^^ assertion evaluated to false
```

### `fail(message)`

`fail` deliberately fails the current test and requires a string expression:

```gb
test "invalid branch is unreachable" {
    if true {
        fail("unexpected branch");
    }
}
```

The custom message is preserved and reported using `T0002`:

```text
✗ invalid branch is unreachable
error[T0002]: test failed: unexpected branch
 --> tests/branch.gb:3:9
   |
 3 |         fail("unexpected branch");
   |         ^^^^^^^^^^^^^^^^^^^^^^^^^^ explicit test failure
```

### `pass()`

`pass()` explicitly performs no operation and may be useful in successful match branches:

```gb
test "missing file returns Err" {
    let result: Result<string,string> = fs.try_read_text("missing.txt");

    match result {
        Ok(_) => {
            fail("expected Err");
        }
        Err(_) => {
            pass();
        }
    }
}
```

## Test diagnostic codes

The test runner has a separate diagnostic family from compiler errors:

| Code | Meaning |
|---|---|
| `T0001` | `assert(condition)` evaluated to false |
| `T0002` | Explicit `fail(message)` |

Compiler diagnostics continue to use the `E....` families. Runtime failures remain runtime errors rather than being converted into `T0001` or `T0002`.

For example:

```gb
test "runtime failure remains runtime" {
    let value: int = 1 / 0;
    print(value);
}
```

is reported as a runtime failure:

```text
✗ runtime failure remains runtime
  runtime error: division by zero
  at tests/runtime.gb:1:1
```

The exact fallback test-block location can become more precise when runtime expression spans are generalized, but the failure category is already distinct from assertion diagnostics.

## Helper functions

Test files may define ordinary helper functions outside test blocks:

```gb
fn double(value: int) -> int {
    return value * 2;
}

test "double returns twice the value" {
    assert(double(4) == 8);
}
```

Helpers are type-checked together with the test suite.

Test helper files cannot define `fn main()`. The runner owns the temporary entry point used to isolate and execute each test.

## Running project tests

From a project root:

```bash
gb test
```

Or provide a project path:

```bash
gb test path/to/project
```

Successful example:

```text
Genix Test Runner

✓ addition works
✓ strings compare

2 passed
0 failed
```

A failed suite returns a non-zero process exit status, making `gb test` directly usable in CI.

## Standalone test files

A single test file may be executed without a `genix.toml` project:

```bash
gb test tests/math.gb
```

The file may contain helper functions and test blocks, but not `fn main()`.

## Isolation model

Each test executes with a fresh interpreter instance.

Conceptually:

```text
test source
    ↓
test frontend
    ↓
record original assertion/fail spans
    ↓
normal Genix parser + AST
    ↓
normal static type checker
    ↓
select one test as temporary main
    ↓
fresh interpreter
    ↓
runner classifies test failure vs runtime error
```

A test therefore does not retain local variables or interpreter scopes from a previously executed test.

The application AST and test helper functions are cloned for each test run. Test blocks themselves remain outside the normal application IR/native build pipeline.

## Relationship to application code

For project tests, `gb test` first loads and type-checks the normal Genix project. This makes the project's functions and already-loaded modules available to the testing program.

The current bootstrap test loader strips import declarations from files under `tests/`. Test files should therefore rely on modules imported by the project entry until the dedicated test module loader is generalized.

## Static checking

The complete test suite is passed through the standard Genix type checker before execution.

For example, this fails before the test starts:

```gb
test "bad type" {
    let count: int = "wrong";
}
```

The testing frontend reuses the normal parser and semantic checker rather than implementing a second Genix language.

## Dedicated test-failure channel

The old bootstrap implementation used division by zero as an assertion sentinel. That mechanism has been removed.

The current pre-alpha runner assigns each `assert` / `fail` site an internal site ID and records its original source span. A failed site enters a private per-run test trap, which the runner decodes back into `T0001` or `T0002` plus the original source location.

Conceptually:

```text
assert(expression)
      ↓ false
private test site ID
      ↓
test-only failure trap
      ↓
T0001 + original expression span
```

```text
fail(message)
      ↓
private test site ID + message
      ↓
test-only failure trap
      ↓
T0002 + original fail span + message
```

The trap key is unique for each runner invocation, so unrelated interpreter errors are not classified as test failures.

This is still bootstrap infrastructure: internally, the private trap currently piggybacks on the existing host file-read failure path. It is not part of the public Genix filesystem API or Runtime ABI contract. A future structured interpreter/test runtime signal can replace that mechanism without changing `T0001` / `T0002` user-facing behavior.

## Current limitations

- Tests execute through the interpreter; native test binaries are not generated yet.
- Test blocks are recognized only by `gb test`, not by normal `gb run`, `gb check`, `gb ir`, or `gb build` parsing.
- Test-file imports are not independently resolved yet.
- Runtime failures currently fall back to the test-block source position rather than an exact runtime-expression span.
- Assertion value diffs such as expected/actual are not implemented yet.
- Filtering, ignored tests, setup/teardown hooks, benchmarks, snapshots, and parallel execution are not implemented yet.
- The private pre-alpha test trap should eventually become a structured interpreter/runtime test signal.

## Design direction

The current design intentionally keeps testing metadata out of executable Genix IR. Future revisions can add richer test features without forcing native application backends to understand test declarations.

Planned improvements include:

- structured interpreter-level test failure signaling
- expected/actual value reporting
- exact runtime-error expression spans
- test-name filtering
- module imports inside test files
- setup and teardown hooks
- native test execution
- parallel test scheduling
- machine-readable CI output

---

**Genix Testing — pre-alpha developer tooling**
