# SOLDER

**Stack Oriented Language for Deep Embedded Runtimes.**

A Forth in the broad sense — no syntax, postfix over a data stack, a dictionary of words that can be
extended from the keyboard — written in [sysl](https://sysl.sh) so that it can live in a
microcontroller's flash and be talked to over a serial line. This repository is the **language** and
nothing else. It reads text and answers with text, and knows nothing about a board, a register or a
terminal; what drives it is a separate program.

```
5 3 + PR                          // 8
"hello" PR                        // hello
DEF SQUARE DUP * END
7 SQUARE PR                       // 49
DEF COUNT 5 0 DO I PR SPACE LOOP END
COUNT                             // 0 1 2 3 4
[] 1 , 2 , 3 , CONSTANT XS
XS LENGTH PR                      // 3
{} "x" 1 PUT "y" 2 PUT PR         // {x: 1 y: 2}
HEX FF PR                         // ff
3.14159 "{>8.2}" PRINTF           //     3.14
```

## What it adds to Forth

**A typed cell.** Forth's cell is a machine word and its meaning is whatever the programmer
remembers. SOLDER's is a tagged value: two widths of integer, a double, a string, an array, an object,
a definition, a variable, a colour, a moment, a coordinate, a complex number. `PR` prints what it was
given because it knows what it was given, and `+` on two strings joins them because it can tell.

**Reference counting rather than a garbage collector.** An array that stops being referred to is freed
at the point it stops. There is no collection to schedule, no pause to budget for, and no allocator
behaviour that differs between the bench and the field.

**Refusals instead of a `longjmp`.** Every word answers a value or a `Fault`, so a mistake at the
keyboard leaves the interpreter usable and the *next* line still works. A `Fault` is either
`Bad(message)` — the sentence to print — or `More`, which means the text stopped in the middle of
something and a console should ask for another line rather than complain.

## Why it exists

SOLDER is a port of Metal, the same author's abandoned C project: about 17,000 lines doing this job,
cross-built for Linux, Windows and two Pi boards. **That makes this the one piece of evidence about
sysl that changes a single variable** — the language, the person and the design are the same, and only
the implementation language is different.

The comparison lands hardest on the cell. `src/cell.c` in the original is 236 lines of hand-written
`retain` and `release`, a `switch` over which types own storage, called by hand at every assignment,
every stack push and every array teardown. In `sh/sysl/solder/cell.sysl` that file has no counterpart
at all: a cell is a data enum whose payload is reachable only through the pattern that names it, and
the counting is the language's.

Three of C's pointers are gone with it, each replaced by something that names what it means rather
than where it is:

| C | here |
|---|---|
| `CELL_POINTER` — a `cell_t*` at a variable's storage | `Var(slot)`, an index into a table the interpreter owns |
| `CELL_RETURN` — a `cell_t*` into the middle of a code array | a return stack of frames, each naming the body it is inside |
| `INDEX` — a `cell_t*` at an array element | `INDEX@` and `INDEX!`, which were already there beside it |

Where the port does something the original does not, it is written down at the point it happens.
There are five so far: `HEX FF` is two hundred and fifty-five rather than the float fifteen (`F` is a
digit in base sixteen, and the original tests for a width suffix before it tries the base); `GET`
exists, the original having a `PUT` and no way to read a key back; `+` joins two strings; a control
word that closes the wrong thing is refused by name rather than patching a cell that was never a
branch; and **a colour, a moment, a coordinate and a complex number can be built at all** — the
original has all four in its cell and no word in front of any of them, so they were reachable from C
and unreachable from the language.

**Twenty-one of the original's words are not here, and all but one of them cannot be.**

**Sixteen are memory introspection, and they are test scaffolding rather than language**: `REFCOUNT`,
`REFCOUNT-EXPECT`, `CELL-SHARED?`, `FORCE-COLLECT`, `CELL-ADDR`, `MEM-STATS` and their neighbours are
registered by `src/test.c` and `src/test_refcount.c`, and Metal's own C tests drive them —
`TEST_INTERPRET("REFCOUNT")` appears thirteen times. **They exist because the counting is
hand-written and can be wrong**, so the suite has to interrogate it from inside the language. Here
the counting is the compiler's, so there is nothing for such a test to assert; and there is no
address a program can see, and no intern pool for `INTERN-COUNT` or `STRING-SAME-PTR?` to answer
about, since a `string` is already shared.

**Three are `(DO)`, `(LOOP)` and `(+LOOP)`** — the runtime halves of the counted loop. `DO` in the
original compiles a reference to `(DO)` and so has to `find_word` it, which means the dictionary
publishes a word that corrupts the return stack if anybody types it. Here the compiling word plants
the runtime directly and it needs no name.

**`INDEX` is the pointer** named in the table above.

The one that could exist and does not is `TEST`, the original's runner for its own C test files;
SOLDER's tests are `@test` functions that `sysl test .` runs.

**`DEBUG-ON` and `DEBUG-OFF` are here and do something different.** In the original they set a global
that makes `debug(…)` calls scattered through the C print `[DEBUG file.c:123] …`, and the interpreter's
contribution to that is `executing cell type: 4` — a tag number rather than a word. Here they echo
each *token* with the stack it left behind, in the same notation `.S` prints:

```
solder> DEBUG-ON 1 2 +
tracing on
[DEBUG-ON] <0>
[1] <1> 1
[2] <2> 1 2
[+] <1> 3
```

A token that went into a definition says so instead — `[2 compiled]` — and the two are told apart by
**which branch the reader took**, not by what mode it was in: `DEF` is read while not compiling and
`END` while compiling, and both of them *run*, being immediate.

The trace stops at the token, and that is a limit rather than a preference: a compiled cell has
deliberately forgotten which name it came from, which is the fold that removed C's `word_idx`. What
the original prints from inside a definition is a tag number, so nothing readable is lost.

## Vocabulary

| | |
|---|---|
| stack | `DUP` `DROP` `SWAP` `OVER` `2DUP` `ROT` `-ROT` `?DUP` `DEPTH` `PICK` `ROLL` `.S` |
| arithmetic | `+` `-` `*` `/` `%` `NEGATE` `1+` `1-` `2*` `2/` `ABS` `SIGNUM` `MIN` `MAX` `/MOD` `*/` `*/MOD` |
| comparison | `=` `!=` `<` `>` `<=` `>=` `0=` `0!=` `0<` `0>` `0<=` `0>=` |
| logic and bits | `AND` `OR` `NOT` `TRUE` `FALSE` `&` `\|` `^` `~` `<<` `>>` `>>>` |
| control | `IF` `ELSE` `THEN` `BEGIN` `AGAIN` `UNTIL` `WHILE` `REPEAT` `DO` `LOOP` `+LOOP` `I` `J` `UNLOOP` `CASE` `OF` `ENDOF` `ENDCASE` |
| definitions | `DEF` `END` `EXIT` `'` `[']` `EXECUTE` `CONSTANT` `VARIABLE` `@` `!` `+!` `1+!` `1-!` |
| arrays and objects | `[]` `,` `LENGTH` `INDEX@` `INDEX!` `{}` `PUT` `GET` `HAS?` `KEYS` `VALUES` `ENTRIES` `FROM-ENTRIES` |
| floating point | `SIN` `COS` `TAN` `ASIN` `ACOS` `ATAN` `ATAN2` `EXP` `LN` `LOG10` `**` `SQRT` `FLOOR` `CEIL` `ROUND` `TRUNC` `FMOD` `PI` `E` |
| printing and the base | `PR` `CR` `SPACE` `EMIT` `WORDS` `HELP` `DECIMAL` `HEX` `BINARY` `OCTAL` `BASE` |
| the trace | `DEBUG-ON` `DEBUG-OFF` |
| the four other cells | `RGB` `RED` `GREEN` `BLUE` `DATETIME` `EPOCH` `TZ` `COORD` `LON` `LAT` `COMPLEX` `RE` `IM` |
| text | `FORMAT` `PRINTF` `STRING-EMPTY?` |
| values | `NULL` `UNDEFINED?` `BL` `BYE` |

`FORMAT` is a template with `{}` in it, filled from the stack — **not `printf`**, because a cell
already knows what it is, so everything inside the braces is about layout rather than about type:
`{>8.2}` is right-aligned in eight columns to two decimal places, `{04x}` is zero-padded hexadecimal,
and `{{` is a literal brace. `PRINTF` is `FORMAT` and `PR`.

**A control structure may be typed at the prompt.** An opening word with no `DEF` around it builds an
anonymous definition, which runs the moment its structure closes and is never filed — so
`5 0 DO I PR LOOP` is a thing to type rather than a thing to define first. It may be typed over
several lines, exactly as a named definition may, and `pending(vm)` is true while either is waiting.

A number takes its width from its value — an integer that fits in 32 bits is one, and one that does
not is a 64-bit integer — and `L` or `F` after it says which was meant where that matters. Arithmetic
narrows back only if both operands were narrow, so a loop that crosses 2³¹ does not change type
underneath itself.

## Using it

```
dependencies {
  solder { git = "github.com/sysl-lang/solder", version = "0.4.0" }
}
```

```sysl
import sh.sysl.solder.*

val vm = solder()

run(vm, "DEF SQUARE DUP * END 7 SQUARE PR") match
    Ok(_) -> print(drain(vm))
    Err(f) -> print(describe(f))
```

An interpreter is a value and carries the session: words defined by one call to `run` are there for
the next, and a definition may be typed across several lines — `pending(vm)` is what a console asks to
decide between a prompt and a continuation prompt. `drain(vm)` takes what has been printed and leaves
nothing behind; `transcript(vm)` reads the whole session without disturbing it. `reset(vm)` abandons a
half-built definition and clears the stacks, and deliberately keeps the dictionary.

`requires { heap = true }`. The dictionary grows as words are defined and a string is as long as the
person at the keyboard typed; the board this is aimed at links a heap whether or not a program touches
it, so the capability costs nothing that is not already spent.

## Tests

```
sysl test .
```

A hundred and seventeen of them, and nearly all are asserted through the transcript — what a person
typing would see — rather than through the stack, because the transcript is what is promised.

## The consoles

The read-run-print loop is `session`, in this package, because there is more than one console and a
loop written out per platform is copies that drift. A console names its streams and prints its
banner, and that is all it does.

| | |
|---|---|
| [**solder-host**](https://github.com/sysl-lang/solder-host) | at a terminal, over `sysl.term.edit` |
| [**solder-pico2**](https://github.com/sysl-lang/solder-pico2) | on a Raspberry Pi Pico 2 W, over USB serial — 405 KB of flash and 5.3 KB of static RAM, and no C in the project at all |

The two differ in their first few lines and nowhere else, which is what putting the loop in the
package bought.

## Status

The language runs, and is tagged `v0.4.0`. The board image builds and links; nothing has been run on
real silicon yet.

## Licence

ISC — see `LICENSE`.
