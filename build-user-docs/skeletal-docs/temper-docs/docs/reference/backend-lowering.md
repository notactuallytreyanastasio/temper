---
title: Lowering onto a Smaller Language
---

# Lowering Temper onto a language that lacks what Temper assumes

A backend for Java or Python mostly maps constructs one to one. A backend for a
smaller language cannot. This page collects what that takes, using the Blimp
backend as the worked example. Blimp is an actor language with no `while`, no
`if` statement, no module system, no regex library, no mutable arrays, and
`and`/`or` operators that evaluate both sides. Every example below is real
output from `temper build -b blimp`, and the links go to the commit that made
each decision. The full series is
[temper-blimp](https://github.com/notactuallytreyanastasio/temper-blimp).

## Where a missing feature goes

Every gap between Temper and the target has to be filled somewhere, and there
are three places:

1. **The translator.** Structural rewrites: turning a loop into recursion, a
   class into whatever the target has for objects.
2. **Support code written in the target language.** A library of helpers
   spliced into or shipped with every translated program. The Blimp backend
   calls its library `temper-core`, and it is plain Blimp.
3. **The target's runtime.** If the target's implementation is yours to change,
   and some things can only be fixed there.

Default to the second. Code written in the target language can be read, tested
and stepped through with the target's own tools, and it keeps the translator
small. The Blimp backend wrote its helpers "in plain Blimp rather than as a Zig
builtin so it reads, tests and steps through with the same tools"
([242f061](https://github.com/notactuallytreyanastasio/temper-blimp/commit/242f061)).

Go to the runtime only when the target cannot express the fix. Temper's
integers wrap on overflow, and Blimp's interpreter used Zig's checked
arithmetic, so `9223372036854775807 + 1` killed the process. Support code
cannot detect overflow, because detecting it means doing the arithmetic that
panics; the fix went into the interpreter
([6b2f899](https://github.com/notactuallytreyanastasio/temper-blimp/commit/6b2f899)).

When the translator needs an awkward workaround, look underneath before
writing it. The Blimp backend once grew a pass that hoisted every message
argument into a local, to work around a crash when an argument spawned an actor.
The real cause was in the runtime, which kept actor entries in a growable array
and held a pointer into it across the send. The workaround was deleted a
minute after the runtime fix
([690818b](https://github.com/notactuallytreyanastasio/temper-blimp/commit/690818b),
[1ef6a24](https://github.com/notactuallytreyanastasio/temper-blimp/commit/1ef6a24)).

## What the frontend has already done for you

Before planning a lowering, check what shape it arrives in. The frontend
simplifies control flow before a backend sees it:

- **There are no mid-function `return`s.** `return foo()` becomes an assignment
  to a return variable followed by `break` to a label around the whole body.
  The function then ends with one `return`.
- **There is no `for`, `do`, or `continue` in the usual sense.** Loops arrive as
  `while`. A `continue` that must still run the loop's increment arrives as a
  `break` out of a labelled block wrapped around the body, with the increment
  after that block.
- **With `CoroutineStrategy.TranslateToRegularFunction`, there are no
  generators.** They arrive as state machines over ordinary functions.

So "early return" and "continue" are really one problem: a `break` to a label.

## Loops without `while`

Without a loop construct, a `while` becomes a function that calls itself in
tail position. Its parameters are every local the loop reads or writes, and it
returns them in a list, led by a tag that says how the loop ended. The caller
unpacks the list by position.

This Temper:

```temper inert
let fib(n: Int): Int {
  var a = 0;
  var b = 1;
  var i = 0;
  while (i < n) {
    let t = a + b;
    a = b;
    b = t;
    i += 1;
  }
  return a;
}
```

becomes this Blimp:

```text
def blimp_loop_0(a__4: Any, b__5: Any, i__6: Any, n__2: Any) -> Any do
  case i__6 < n__2 do
    true -> blimp_signal_1 = case true do
      true -> t__7 = temper_int32(a__4 + b__5)
      a__4 = b__5
      b__5 = t__7
      i__6 = temper_int32(i__6 + 1)
      [:fall, a__4, b__5, i__6, n__2]
    end
    a__4 = elem(blimp_signal_1, 1)
    b__5 = elem(blimp_signal_1, 2)
    i__6 = elem(blimp_signal_1, 3)
    n__2 = elem(blimp_signal_1, 4)
    case elem(blimp_signal_1, 0) do
      :escape -> blimp_signal_1
      :break ->[:done, a__4, b__5, i__6, n__2]
      _ -> blimp_loop_0(a__4, b__5, i__6, n__2)
    end
    _ ->[:done, a__4, b__5, i__6, n__2]
  end
end
def fib__1(n__2: Any) -> Any do
  a__4 = 0
  b__5 = 1
  i__6 = 0
  blimp_carried_2 = blimp_loop_0(a__4, b__5, i__6, n__2)
  a__4 = elem(blimp_carried_2, 1)
  b__5 = elem(blimp_carried_2, 2)
  i__6 = elem(blimp_carried_2, 3)
  n__2 = elem(blimp_carried_2, 4)
  a__4
end
```

The details that matter:

- **The carried set is computable exactly** when the loop function is top-level
  and closes over nothing: it is the locals the loop body mentions.
- **The tag goes first and the carried values after it,** so carried value `i`
  is always at `i + 1` whatever the tag. Blimp needed a third tag, `:escape`,
  for a `break` aimed at a loop further out. It carries its destination and
  payload after the carried values, so the offsets never change
  ([0da2c77](https://github.com/notactuallytreyanastasio/temper-blimp/commit/0da2c77)).
- **Track whether each path has already ended.** Otherwise the fall-through tag
  at the end of a body overwrites a branch's `:break`, and every loop reports
  `:fall` ([67cc88e](https://github.com/notactuallytreyanastasio/temper-blimp/commit/67cc88e)).
- **The tail call must really be in tail position.** In Blimp, binding the
  result of a `case` to a name and returning the name makes the call a
  non-tail call. Each iteration then keeps a live frame, and a regex search
  over 2,000 characters took 52 seconds instead of under one
  ([07d32e8](https://github.com/notactuallytreyanastasio/temper-blimp/commit/07d32e8)).

## Branches whose assignments do not escape

In Blimp, an assignment inside a `case` arm does not reliably reach the code
after the `case`. So a lowered `if` hands the names it assigns out as the
value of the `case`, and the caller rebinds them. That is the same
list-in, list-out shape as the loop.

Two traps:

- **Decide the shared names before translating either arm.** Take a snapshot of
  the enclosing scope first, and emit `name = nil` ahead of the `case` for each
  name the snapshot lacks. If you ask the scope after translating, a name only
  one arm declared is already in it, and the other arm packages a name the
  target never bound.
- **Find out where the target accepts your construct.** A Blimp `case` is valid
  as a statement or on the right of an assignment, and nowhere else: not after
  `reply`, not as a call argument, not in a list literal. The Blimp backend
  found this one form at a time by running the interpreter, and now gives a
  `case` a name before using it anywhere else
  ([ba2e76c](https://github.com/notactuallytreyanastasio/temper-blimp/commit/ba2e76c)).

## Early exits as continuations

With the frontend's rewrite, an early `return` is a `break` to a labelled block.
Without a `goto`, the Blimp backend splits the function body at the first
statement containing an exit. Everything before it is emitted as is, the rest
of the body becomes a *continuation* function, and a `break` becomes a call to
that function.

A labelled block's continuation has two users, and both must get it. A
`break L` jumps to it, and so does reaching the end of the block normally.
Wire it only to the `break` and nothing crashes. You get two nearly identical
continuation functions, only one of which runs the loop's increment, so
iteration zero repeats forever
([d14fa72](https://github.com/notactuallytreyanastasio/temper-blimp/commit/d14fa72)).

## Classes onto whatever the target has

Blimp has actors, not objects, so a class becomes an actor. Fields become
`state`, methods become `on` handlers, and `new` becomes `spawn` plus a
constructor message:

```text
actor Counter__0 do
  state count__6: Any :: nil
  on :__temper_types do
    reply[:Counter]
  end
  on :bump do
    t___12 = temper_int32(count__6 + 1)
    count__6 = t___12
    become count__6: count__6
    reply count__6
  end
  on :__new(count__10: Any) do
    count__6 = count__10
    become count__6: count__6
    reply nil
  end
  ...
end
```

Check the target's semantics against Temper's before trusting a mapping like
this. A Blimp handler's state is fixed when the message arrives, and `become`
publishes to the *next* message. That is fine until a method calls another
method on the same object and then reads a field. The Blimp backend passed all
65 functional tests with this bug, because no test did that. Its first program
from outside the suite, a 130-line expression calculator, failed five of
six tests. The fix moves
exactly the fields that can go stale into a one-value cell actor, and no
others ([14a53b2](https://github.com/notactuallytreyanastasio/temper-blimp/commit/14a53b2)).

A downcast needs a runtime answer to "what type is this?". With no class
hierarchy to ask, each actor answers `:__temper_types` with its type and all
its supertypes, breadth-first.

## Operators whose namesakes disagree

A Temper operation and a target builtin with the same name often mean
different things. Each difference needs support code or a support-network
entry. From the Blimp backend:

| Temper expects | Blimp does | Fix |
| ---- | ---- | ---- |
| `Int` wraps at 32 bits | 64-bit integers | every arithmetic result goes through `temper_int32` |
| `&&` and `\|\|` skip the right side | `and`/`or` evaluate both sides | lower to `case`, which evaluates only the branch it takes |
| `slice(a, b)` takes an end index | `slice` takes a length | `temper_slice` |
| Float division by zero gives infinity or NaN | raises an error | `temper_float_div` |
| A stable sort | the insertion helper compared with `<=`, reordering equal keys | compare with a strict `<` |
| `"a${b}c"` concatenates any number of pieces | `concat` is variadic but needs two or more | one piece is the value itself; zero pieces is `""` |

The eager-`and` row applies to your own support code too. The Blimp regex
parser contained `not(nil?(cls)) and elem(cls, 0) == :cls`, the guard idiom that
short-circuiting languages allow. In Blimp it still called `elem(nil, 0)`, and a
character-set feature the series believed it had shipped never ran
([82a5e87](https://github.com/notactuallytreyanastasio/temper-blimp/commit/82a5e87)).

## Names

Translated names can collide with the target's keywords and builtins, and the
failure may not be an error. A library exporting `length` became
`def length` in Blimp, where a definition silently beats a builtin. Its support
code called `length` in tail position, so it called itself forever in constant
stack: no overflow, no crash, a test run that looked deadlocked
([811a727](https://github.com/notactuallytreyanastasio/temper-blimp/commit/811a727)).

- Take the reserved set from the target's real registry, and confirm a name is
  a builtin by calling it, not by searching the source for it. A search for
  `register` in the Blimp interpreter also matched actor templates registered
  by unit tests
  ([35a2175](https://github.com/notactuallytreyanastasio/temper-blimp/commit/35a2175)).
- Generate names from one generator per `translate` call. If every module of
  a library lands in one output file, per-module counters collide (see
  *Lifecycle of a Backend*).
- If the target dispatches on a name alone, members with the same name
  collide too. Blimp picks the first handler with a matching atom, so a setter
  and a method that spell the same atom silently shadow each other
  ([9801f49](https://github.com/notactuallytreyanastasio/temper-blimp/commit/9801f49)).

## Libraries the target does not have

A `@connected` declaration can be answered by the target's library, by support
code, or by the declaration's own Temper body. When the target has nothing,
the answer is code written in the target. The Blimp backend's regex support is
an 880-line engine written in Blimp, a parser for the pattern dialect Temper's
`RegexFormatter` emits plus a continuation-passing matcher
([74986f6](https://github.com/notactuallytreyanastasio/temper-blimp/commit/74986f6)).

Check the costs of the target's data structures before you choose a
representation. Blimp's lists are immutable, so appending copies. A bit vector
stored as one list cost 1.58 GB for 14,000 single-bit sets, and storing it as
256-bit chunks raised the limit about fifteen times
([14062d2](https://github.com/notactuallytreyanastasio/temper-blimp/commit/14062d2)).
The same copying still affects the Blimp backend's `ListBuilder` and
`StringBuilder`. 20,000 appends cost 1.6 GB and 2.0 GB respectively, because
the interpreter frees nothing until the program exits.

## When you cannot translate something

There are two different situations:

- **A shape your backend has not learned yet.** Use Kotlin's `TODO()` with the
  offending node. The build stops with a location, and the location tells you
  what to implement next. Never emit a plausible-looking fallback: wrong output
  hides, while a crash is a work item.
- **Code the frontend already rejected.** The functional test
  `semantics/broken` feeds every backend deliberately broken Temper and expects
  the program to fail when it runs, not the compiler to crash. The frontend
  hands you garbage nodes here. Lower them to a call that fails at run time
  with the position, so the program runs up to the bad line and stops there
  ([21d8d5b](https://github.com/notactuallytreyanastasio/temper-blimp/commit/21d8d5b)).
