# SOLDER

**Stack Oriented Language for Deep Embedded Runtimes.**

A Forth in the broad sense — no syntax, postfix over a data stack, a dictionary of words that can be
extended from the keyboard — written in [sysl](https://sysl.sh) so that it can live in a
microcontroller's flash and be talked to over a serial line. This repository is the **language** and
nothing else. It reads text and answers with text, and knows nothing about a board, a register or a
terminal; what drives it is a separate program.

```
5 3 + PR                       // 8
"hello" PR                     // hello
DEF SQUARE DUP * END
7 SQUARE PR                    // 49
[] 1 , 2 , 3 , CONSTANT XS
XS LENGTH PR                   // 3
```

## What it adds to Forth

**A typed cell.** Forth's cell is a machine word and its meaning is whatever the programmer
remembers. SOLDER's is a tagged value: two widths of integer, a double, a string, an array, an object,
a definition, a colour, a moment, a coordinate, a complex number. `PR` prints what it was given
because it knows what it was given.

**Reference counting rather than a garbage collector.** An array that stops being referred to is freed
at the point it stops. There is no collection to schedule, no pause to budget for, and no allocator
behaviour that differs between the bench and the field.

## Why it exists

SOLDER is a port of [Metal](https://github.com/edadma/metal), the same author's abandoned C project:
about 17,000 lines doing this job, cross-built for Linux, Windows and two Pi boards. **That makes this
the one piece of evidence about sysl that changes a single variable** — the language, the person and
the design are the same, and only the implementation language is different.

The comparison lands hardest on the cell. `src/cell.c` in the original is 236 lines of hand-written
`retain` and `release`, a `switch` over which types own storage, called by hand at every assignment,
every stack push and every array teardown. In `sh/sysl/solder/cell.sysl` that file has no counterpart
at all: a cell is a data enum whose payload is reachable only through the pattern that names it, and
the counting is the language's.

## Using it

```
dependencies {
  solder { git = "github.com/sysl-lang/solder", version = "0.1.0" }
}
```

`requires { heap = true }`. The dictionary grows as words are defined and a string is as long as the
person at the keyboard typed; the board this is aimed at links a heap whether or not a program touches
it, so the capability costs nothing that is not already spent.

## Tests

```
sysl test .
```

## Status

**In progress.** The cell and the interpreter's state are here; the reader, the inner interpreter and
the vocabulary are being written. Nothing is tagged yet.

## Licence

ISC — see `LICENSE`.
