# zonk

`zonk` is a parser combinator for the [Neut](https://vekatze.github.io/neut/) programming language.

## Installation

```sh
neut get zonk https://github.com/vekatze/zonk/raw/main/archive/0.3.2.tar.zst
```

## Types

### Main Definitions

```neut
// Holds an item expected by a parser.
data expected-item

// An opaque type that holds information needed to execute parsers
data zonk-kit

// A fully elaborated parse error with position information.
data parse-error {
| Parse-Error(
    found: string,
    expected: expected-item,
    error-location: point,
  )
}

// The type of parsers
alias zonk(a) {
  @(&zonk-kit) -> either(unit, a)
}

// The result type of a parser.
alias parsed(a) {
  either(unit, a)
}

// Creates a zonk-kit for the given input string
define make-zonk-kit@(input-stream: string) -> zonk-kit

// Builds a parse error from the kit's current failure state
define make-parse-error@(k: &zonk-kit) -> parse-error

// Converts a parse error into a human-readable string
define report(e: parse-error) -> string
```

### Parser Combinators

```neut
// Fails with `label` as the expected input.
inline report-unexpected-input<a>(k: &zonk-kit, label: text) -> parsed(a)

// Succeeds only at the end of the input.
constant end-of-input: zonk(unit)

// Succeeds only when the head rune of the remaining stream satisfies `p`.
// The variable `label` is used when reporting errors.
inline satisfy@(label: text, p: (rune) -> bool) -> zonk(rune)

// Reads any single rune from the remaining stream.
constant any-rune: zonk(rune)

// Succeeds only when the head of the remaining stream is equal to `t`.
inline chunk@(chunk-text: text) -> zonk(unit)

// Consumes the input stream while `p` is true.
inline take-while@(p: (rune) -> bool) -> zonk(string)

// A non-empty version of `take-while`.
inline take-while-1@(p: (rune) -> bool) -> zonk(string)

// Discards the input stream while `p` is true.
inline drop-while@(p: (rune) -> bool) -> zonk(unit)

// Skips ASCII spaces.
constant ascii-space: zonk(unit)

// Executes `p`, overriding the (possible) error message with `l`.
inline label@<a>(l: text, p: zonk(a)) -> zonk(a)

// `attempt(p)` is the same as `p` if `p` succeeds.
// `attempt(p)` rewinds the input stream if `p` fails.
inline attempt@<a>(p: zonk(a)) -> zonk(a)

// `look-ahead(p)` rewinds the input stream if `p` succeeds.
// `look-ahead(p)` is the same as `p` if `p` fails.
inline look-ahead@<a>(p: zonk(a)) -> zonk(a)

// `optional(p)` is the same as `p` if `p` succeeds.
// `optional(p)` suppresses the error of `p` and results in none if `p` fails.
// Note that `optional(p)` does not rewind the input stream automatically.
inline optional@<a>(p: zonk(a)) -> zonk(?a)

// Executes `p1`. If it succeeds, `branch(p1, p2)` returns the result of `p1`.
// If it fails, executes `p2`.
// Note that `branch(p1, p2)` does not rewind the input stream automatically.
inline branch@<a>(p1: zonk(a), p2: zonk(a)) -> zonk(a)

// Succeeds only when `p` can parse the head of the input stream.
// `not-followed-by` does not consume the input stream.
// The variable `label` is used when reporting errors.
inline not-followed-by@<a>(label: text, p: zonk(a)) -> zonk(unit)

// Tries given parsers one by one.
// The variable `label` is used when reporting errors.
// Note that each candidate should be wrapped in `attempt` if it may consume input before failing.
inline choice@<a>(label: text, candidates: list(zonk(a)), fallback: zonk(a)) -> zonk(a)

// Parses the input stream using `!p` iteratively until it fails.
// This parser never fails.
inline many<a>(!p: zonk(a)) -> zonk(list(a))

// Parses the input stream using `!p` iteratively until it fails.
// This parser succeeds only if `!p` successfully parses the input stream at least once.
inline some@<a>(!p: zonk(a)) -> zonk(list(a))

// Consumes runes while `p` holds and interprets the consumed string.
inline scan-while@<a>(label: text, p: (rune) -> bool, interpret: (&string) -> +?a) -> zonk(a)
```

### Presets

```neut
// A regex type
data regex {
| Any(label: text, runes: list(rune))
| Chunk(chunk-text: text)
| Choose(label: text, candidates: list(regex), fallback: regex)
| Join(components: list(regex))
| Repeat(regex)
| End-Of-Input
}

// Returns True only if the regex `r` recognizes `input`.
inline recognize(r: regex, input: string) -> bool

// Skips space characters.
constant skip-space: zonk(unit)

// Parses a symbol.
inline read-symbol() -> zonk(string)

// Parses an integer.
constant read-int: zonk(int)

// Parses a float.
constant read-float: zonk(float)

// Parses integers separated by space characters and stores them into a list.
define read-int-list(size: int) -> zonk(list(int))

// Parses values separated by space characters and stores them into a vector.
inline read-vector<a>(size: int, !p: zonk(a)) -> zonk(vector(a))

// Parses integers separated by space characters and stores them into a vector.
define read-int-vector(size: int) -> zonk(vector(int))

// A float-version of `read-int-list`.
define read-float-list(size: int) -> zonk(list(float))

// A float-version of `read-int-vector`.
define read-float-vector(size: int) -> zonk(vector(float))
```

### Utils

```neut
// A type to represent a specific position in an input stream
data point {
| Point(
    row: int,
    column: int,
  )
}

// Gets the current reading position.
inline get-point(k: &zonk-kit) -> point

// Converts a point into human-readable text.
define show-point(p: point) -> string
```

## Example

Executing `zen` in the following example should output `pass`:

```neut
import {
  core::string.io {print-line},
  this::parse {choice, chunk, end-of-input, make-parse-error, make-zonk-kit, many, report, some, zonk},
}

constant sample-parser: zonk(unit) {
  // constructs a parser
  @(k) => {
    // accepts: foo
    try _ = chunk@("foo")@(k);
    // accepts: (baz|test|bar)
    try _ = choice@("baz, test, or bar", List::[chunk@("baz"), chunk@("test")], chunk@("bar"))@(k);
    // accepts: (qux)+
    try _ = some@(chunk@("qux"))@(k);
    // accepts: (yo)*
    try _ = many(chunk@("yo"))@(k);
    // accepts: end-of-input
    try _ = end-of-input@(k);
    Right(Unit)
  }
}

define zen() -> unit {
  pin k = make-zonk-kit@(*"foobarquxquxyoyoyoyo");
  match sample-parser@(k) {
  | Right(_) =>
    print("pass\n")
  | Left(_) =>
    let err = make-parse-error@(k);
    pin message = report(err);
    print("fail: ");
    print-line(message)
  }
}
```

If you rewrite the input as `fooXXXXquxquxyoyoyoyo`, the output will be as follows:

```text
fail: parse error at row 1, column 4:
expected:
  baz, test, or bar
found:
  "XXXX"
```
