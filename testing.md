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

## Assertions

### `assert(condition)`

`assert` requires a boolean Genix expression:

```gb
test "comparison works" {
    let value: int = 12;
    assert(value > 10);
}
```

A false assertion fails the current test.

### `fail(message)`

`fail` deliberately fails the current test and requires a string expression:

```gb
test "invalid branch is unreachable" {
    if false {
        fail("this branch should not execute");
    }
}
```

The bootstrap runner currently validates the message expression but does not yet preserve the custom message in the final failure report.

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

Example output:

```text
Genix Test Runner

✓ addition works
✓ strings compare

2 passed
0 failed
```

A failed suite returns a non-zero process exit status:

```text
Genix Test Runner

✗ addition works
  assertion failed
  at tests/math.gb

0 passed
1 failed
```

This allows `gb test` to be used directly in CI pipelines.

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
normal Genix parser + AST
    ↓
normal static type checker
    ↓
select one test as temporary main
    ↓
fresh interpreter
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

## Bootstrap implementation note

The current pre-alpha implementation represents `assert`/`fail` failure internally using a controlled runtime sentinel based on the interpreter's divide-by-zero failure path.

This is an implementation detail, not intended language semantics. A consequence of the current bootstrap is that a genuine division-by-zero runtime error inside a test can also appear as `assertion failed`.

A later test-runtime revision will replace this with a dedicated test failure intrinsic/result so assertion failures, user `fail(...)` messages, and unrelated runtime errors remain distinct.

## Current limitations

- Tests execute through the interpreter; native test binaries are not generated yet.
- Test blocks are recognized only by `gb test`, not by normal `gb run`, `gb check`, `gb ir`, or `gb build` parsing.
- Test-file imports are not independently resolved yet.
- `fail(message)` does not yet print its custom message.
- Assertion source spans and expression-value diffs are planned improvements.
- Filtering, ignored tests, setup/teardown hooks, benchmarks, snapshots, and parallel execution are not implemented yet.

## Design direction

The current design intentionally keeps testing metadata out of executable Genix IR. Future revisions can add richer test features without forcing native application backends to understand test declarations.

Planned improvements include:

- dedicated assertion/test-failure runtime channel
- exact assertion source spans
- expected/actual value reporting
- test-name filtering
- module imports inside test files
- setup and teardown hooks
- native test execution
- parallel test scheduling
- machine-readable CI output

---

**Genix Testing — pre-alpha developer tooling**
