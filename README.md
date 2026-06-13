# capa_test

Assertion and reporting helpers for `capa test` suites. A
`Tester` accumulates pass/fail counts, every assertion prints
one deterministic line, and `finish()` prints a stable one-line
summary, then `panic`s when anything failed, so the test
process exits non-zero exactly as the
[`capa test` result contract](https://github.com/nelsonduarte/capa-language/blob/main/docs/testing.md)
requires.

Capability surface: **Stdio only**, passed explicitly to each
printing helper. Creating a `Tester` needs no capability at
all; the library cannot touch the filesystem, the network, the
clock, or anything else. `capa --manifest` proves it (see
[Audit claim](#audit-claim)).

## Quick start

In your project (the one with a `capa.toml`), declare the
library as a dev-dependency. Dev-dependencies are vendored only
when your project is the install root; consumers of *your*
package never fetch them:

```bash
capa add --dev capa_test \
    --git https://github.com/nelsonduarte/capa_test --tag v0.1.0 \
    --verify-key 6C1D222D491FB88031E041A536CFB426101AA24B
capa install
```

That writes, under `[dev-dependencies]` in your `capa.toml`:

```toml
[dev-dependencies]
capa_test = { git = "https://github.com/nelsonduarte/capa_test", tag = "v0.1.0", verify_key = "6C1D222D491FB88031E041A536CFB426101AA24B" }
```

Write a test, a plain Capa program under `tests/`, named
`tests/test_*.capa`:

```capa
// tests/test_greeting.capa
import capa_test.testing

fun main(stdio: Stdio)
    let t = tester()
    t.eq_str(stdio, "greeting_shape", greet("capa"), "hello capa")
    t.check(stdio, "non_empty", not greet("x").is_empty())
    t.finish(stdio)
```

Run the suite, then run it on both backends with a stdout
parity diff:

```bash
capa test          # Python backend
capa test --both   # Python + Wasm, byte-identical stdout required
```

A passing run looks like:

```
capa test: 1 file(s) under /your/project/tests [backend: python+wasm]
test_greeting.capa ... ok (0.5s)
1 test(s): 1 passed, 0 failed
```

## The exit-code contract and `panic`

`capa test` runs each test file as a plain Capa program: exit 0
is a pass, anything else is a failure. `main`'s return value is
ignored, so the only way to fail is to **abort**. This library
fails the honest way: each assertion only prints and counts;
`finish()` ends the suite with

```
capa_test: <passed> passed, <failed> failed
```

and, when `failed > 0`, calls `panic("<failed> assertion(s)
failed")`. `panic` writes `panic: <message>` to stderr and
exits non-zero on every backend, which `capa test` reports as
FAILED and propagates as its own exit code 1. If your test
never reaches `finish()` (a runtime error escaped `main`), the
process aborts anyway and the test still fails; you lose the
summary line, not the failure.

## API surface

One module, `capa_test.testing`:

```capa
pub type Tester { passed: Int, failed: Int }

pub fun tester() -> Tester   // fresh reporter, zeroed counts
```

`Tester` has reference semantics: pass it to helper functions
freely, the counts are shared. All methods below print through
the `Stdio` you pass them; passing assertions print
`ok <name>`, failing ones print `FAIL <name>: <detail>` and
bump `failed`.

```capa
impl Tester
    // truth
    pub fun check(self, stdio: Stdio, name: String, cond: Bool)

    // equality, with "expected X, got Y" failure detail
    pub fun eq_str(self, stdio: Stdio, name: String, got: String, want: String)
    pub fun eq_int(self, stdio: Stdio, name: String, got: Int, want: Int)
    pub fun eq_bool(self, stdio: Stdio, name: String, got: Bool, want: Bool)
    pub fun eq_str_list(self, stdio: Stdio, name: String, got: List<String>, want: List<String>)

    // String shape checks
    pub fun ne_str(self, stdio: Stdio, name: String, got: String, banned: String)
    pub fun contains(self, stdio: Stdio, name: String, text: String, sub: String)
    pub fun starts_with(self, stdio: Stdio, name: String, text: String, prefix: String)

    // unconditional failure, for branches a test must not reach
    pub fun fail(self, stdio: Stdio, name: String, message: String)

    // summary line + panic when failed > 0
    pub fun finish(self, stdio: Stdio)
```

The counts are public fields (`t.passed`, `t.failed`), so a
suite can assert on a scratch reporter's totals; this library's
own tests do exactly that.

### Why per-type equality and not a generic `assert_eq<T>`?

Generic `==` itself type-checks and runs correctly on both
backends. The *failure message* is the problem: rendering a
generic `T` with `${x}` is not portable today. Through a type
parameter the Python backend prints `True` where the Wasm
backend prints `true`, and the Wasm backend rejects
interpolating lists and options outright. A test library whose
failure messages diverge across backends would break the very
`capa test --both` parity it exists to support, and a generic
`assert_eq` without expected/actual in the message is a worse
API than typed variants with correct rendering. So: `eq_str`,
`eq_int`, `eq_bool`, `eq_str_list`. Float equality is
deliberately absent from v0.1 (exact `==` on floats is a
footgun; an epsilon-based `close_to` can come later if real
suites need it).

## Determinism

The library adds no nondeterminism of its own: no timing, no
reordering, lines appear in call order, and the summary line is
a pure function of the counts. A suite built on it passes
`capa test --both` whenever the code under test is itself
deterministic.

## Seeing a failure end to end

[`demo/failing_suite/`](./demo/failing_suite/) is a separate
tiny project (own `capa.toml`, this library as a path
dev-dependency) whose single test fails on purpose. It is kept
outside `tests/` precisely so the library's own suite stays
green. From that directory, `capa test --both` prints:

```
capa test: 1 file(s) under .../demo/failing_suite/tests [backend: python+wasm]
test_deliberate_failure.capa ... FAIL (0.6s)
  exit code 1 [python]
  --- captured stdout ---
  ok this_one_passes
  FAIL answer_is_42: expected 42, got 41
  FAIL greeting_shape: expected 'hello capa', got 'hello wasm'
  capa_test: 1 passed, 2 failed
  --- captured stderr ---
  panic: 2 assertion(s) failed
  exit code 1 [wasm]
  --- captured stdout ---
  ok this_one_passes
  FAIL answer_is_42: expected 42, got 41
  FAIL greeting_shape: expected 'hello capa', got 'hello wasm'
  capa_test: 1 passed, 2 failed
  --- captured stderr ---
  panic: 2 assertion(s) failed
1 test(s): 0 passed, 1 failed
```

and exits 1. The two backends agree byte for byte: the same
captured stdout, and the same single-line `panic: 2
assertion(s) failed` on stderr with no host traceback on
either side.

## Tests

The library is tested with itself: the three suites under
[`tests/`](./tests/) build their assertions out of `Tester`,
exercise the failure paths on a scratch reporter (whose FAIL
lines are part of the expected, deterministic stdout), and
assert on its counts. From the repository root:

```bash
capa test
capa test --both
```

Current output of `capa test --both`:

```
capa test: 3 file(s) under .../capa_test/tests [backend: python+wasm]
test_assertions.capa ... ok (0.56s)
test_reporting.capa ... ok (0.51s)
test_strings.capa ... ok (0.52s)
3 test(s): 3 passed, 0 failed
```

## Audit claim

`capa --manifest testing.capa`, condensed to the capability
columns:

```
tester:               declared=[]        reachable=[]
Tester.check:         declared=['Stdio'] reachable=['Stdio']
Tester.eq_str:        declared=['Stdio'] reachable=['Stdio']
Tester.eq_int:        declared=['Stdio'] reachable=['Stdio']
Tester.eq_bool:       declared=['Stdio'] reachable=['Stdio']
Tester.eq_str_list:   declared=['Stdio'] reachable=['Stdio']
Tester.ne_str:        declared=['Stdio'] reachable=['Stdio']
Tester.contains:      declared=['Stdio'] reachable=['Stdio']
Tester.starts_with:   declared=['Stdio'] reachable=['Stdio']
Tester.fail:          declared=['Stdio'] reachable=['Stdio']
Tester.finish:        declared=['Stdio'] reachable=['Stdio']
render_list:          declared=[]        reachable=[]
```

No function crosses `unsafe`, no function reaches any
capability beyond the `Stdio` you hand it, and the
constructor reaches none at all.

## Honest limitations (v0.1)

- **No generic `assert_eq`.** See the per-type rationale above.
- **No float assertions.** Exact float `==` is a footgun;
  deferred until an epsilon-based design is warranted.
- **No test isolation or runners.** This is an assertion
  library; discovery, process isolation, parity diffing and
  exit-code aggregation are `capa test`'s job, not duplicated
  here.
- **Output is the protocol.** There is no machine-readable
  report format; `capa test --both` already diffs stdout, and
  the `ok`/`FAIL` line shapes are stable.

## License

MIT. See [`LICENSE`](./LICENSE). Release tags are GPG-signed;
see [`SECURITY.md`](./SECURITY.md) for the fingerprint and
verification instructions.
