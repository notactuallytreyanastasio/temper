---
title: Verifying a Backend
---

# Knowing a backend works

A backend can pass every functional test and still translate ordinary programs
wrongly. This page is about the gap between a green suite and a working
backend, and how to close it. The examples come from the Blimp backend, which
reached 65 of 65 functional tests before its first program written outside the
suite failed five of its six tests
([f01fe0d](https://github.com/notactuallytreyanastasio/temper-blimp/commit/f01fe0d)).

## What the functional test matrix records

`functional-test-matrix.md` is written by `AllTestsTest.updateFunctionalTestMatrix`.
It does not run anything. Each cell is the test's *disposition* for that
backend, computed from the status lists in `FunctionalTestStatus.kt`: a tick
means no one has recorded an issue that skips the test for that backend.
The results come from your backend's `FunctionalTestRunner` subclass, which
actually builds and runs each test. Run that, and read its output, before
changing the lists.

The lists read backwards. `onlyPasses(issue, tests...)` attaches the issue to
every test *not* named, so naming a test turns it on. Each issue check applies
to exactly one backend. So a test name added to the wrong backend's list does
not switch anything off for yours. It switches that test *on* for the other
backend, whose toolchain you may never have run. Edit these lists one line at
a time and read the diff for your own backend's block. A replace-all anchored
on a line that appears in two backends' lists made this mistake twice in the
Blimp series.

Regenerating the matrix rewrites every row. If other backends' cells were
stale, they change too; say so in your commit message rather than letting it
look like part of your change
([871c002](https://github.com/notactuallytreyanastasio/temper-blimp/commit/871c002)).

## Tests that a passing program cannot fool

A functional test proves that the paths it takes are right. It says nothing
about the paths it does not take. A program that never calls `length` passes
whether or not your backend shadowed `length`. Each of these catches something
the others miss:

- **Assertions on generated text.** Call the translator directly and compare
  its output with what you expect. This is the only way to pin how names are
  sanitized, how a setter is spelled, or what order members come out in. Keep
  the inputs free of anything that pulls in support code, or the expectation
  fills with helpers
  ([3bd7dd6](https://github.com/notactuallytreyanastasio/temper-blimp/commit/3bd7dd6)).
- **Programs written outside the suite.** The suite is finite and the backend's
  author has, by definition, made it pass. Write a small real program: a parser,
  a calculator, a game. Run it on your backend and on two mature ones, and
  compare. That is how the Blimp backend found that its object fields were read
  stale, a bug present since the first day
  ([14a53b2](https://github.com/notactuallytreyanastasio/temper-blimp/commit/14a53b2)).
- **A reference implementation, if there is one.** When your output has to
  agree with a tool that already exists, diff against the tool over every input
  you have, instead of testing cases you thought of. A Blimp syntax highlighter
  written in Temper was checked this way: a dumper built from Blimp's own
  lexer, compared token for token over every `.blimp` file in the repository.
- **`semantics/broken`.** It checks that a backend degrades on code the
  frontend rejected, rather than crashing the compiler.

## Evidence that a fix fixed something

- **Revert it and watch it fail.** A fix is proven when removing it flips the
  output from right back to wrong. A test that passes both with and without your
  change tests nothing.
- **Compiling is not running.** Type-checking, or a clean build of one target,
  says nothing about behaviour. It also says nothing about the other build
  targets, which may not even compile.
- **Know which binary you ran.** If your target's tools are built from source,
  check what is on the `PATH`. In the Blimp series, the interpreter on the
  `PATH` was a symlink into the build directory, so every transcript ran
  whatever was last built. It was also a Debug build: the functional suite ran
  2.5 times slower than on an optimized build, and the game's render 12 times
  ([707871e](https://github.com/notactuallytreyanastasio/temper-blimp/commit/707871e),
  [43579b0](https://github.com/notactuallytreyanastasio/temper-blimp/commit/43579b0)).
- **Check that your check can fail.** Before trusting a comparison that
  reports no differences, corrupt one input and confirm it reports one.
- **Watch exit statuses in pipelines.** A shell pipeline reports its last
  command's status, so `run-tests | tail` succeeds whatever the tests did.

## Failures that do not announce themselves

The most expensive bugs in a backend produce plausible output, not errors.
When the target does any of the following, write a test that would notice:

- **Duplicate definitions are allowed**, and the later one wins. A name
  collision then shows up as the wrong function running
  ([f01fe0d](https://github.com/notactuallytreyanastasio/temper-blimp/commit/f01fe0d)).
- **Out-of-range arguments are clamped.** Blimp's `tcp_connect` folded port
  131071 into 65535 and connected. Temper's `std/net` sent an `https` URL as
  plain text on port 80 and could get back a plausible 200
  ([f624aae](https://github.com/notactuallytreyanastasio/temper-blimp/commit/f624aae)).
- **Failure is a return value.** Blimp's `write_file` answers `false`. If the
  test runner ignores it, a run whose report was never written looks green
  ([2e9c666](https://github.com/notactuallytreyanastasio/temper-blimp/commit/2e9c666)).
- **A missing header means "anything".** An absent `Accept-Encoding` permits
  every content coding, so a client that cannot decompress must ask for
  `identity` explicitly
  ([9b8fed9](https://github.com/notactuallytreyanastasio/temper-blimp/commit/9b8fed9)).
- **Memory is never reclaimed.** On an arena-allocated runtime, an operation
  that copies on every call is quadratic in memory even when it is fast. Measure
  peak memory as well as time. `/usr/bin/time -l` on macOS or `-v` on Linux
  prints it.

And in your own source: a doc comment inserted in the wrong place compiles
fine and describes the wrong declaration. Run the module's linters as a step
of their own
([bbfcea5](https://github.com/notactuallytreyanastasio/temper-blimp/commit/bbfcea5)).
