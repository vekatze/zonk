# zonk

`zonk` is a parser combinator for the [Neut](https://vekatze.github.io/neut/) programming language.

## Installation

```sh
neut get zonk https://github.com/vekatze/zonk/raw/main/archive/0-2-12.tar.zst
```

## Types

### Main Definitions

```neut
// An opaque type that holds expected items
data expected-item

// An opaque type that holds information needed to execute parsers
data zonk-kit(c)

// A fully elaborated parse error with position information.
data parse-error {
| Parse-Error(
    found: string,
    expected: expected-item,
    error-location: point,
  )
}

// The type of parsers
alias zonk(c, a) {
  (&zonk-kit(c)) -> either(expected-item, a)
}

// Creates a zonk-kit for given string and context
define make-zonk-kit<c>(input-stream: string, context: c) -> zonk-kit(c)

// Gets the user-supplied context
define get-context<c>(k: &zonk-kit(c)) -> &c

// Converts an opaque `expected-item` to a parse error
define make-parse-error<c>(k: &zonk-kit(c), expected: expected-item) -> parse-error

// Converts a parse error into a human-readable string
define report(e: parse-error) -> string
```

### Parser Combinators

```neut
// Succeeds only at the end of the input.
constant end-of-input<c>: zonk(c, unit)

// Succeeds only when the head rune of the remaining stream satisfies `p`.
// The variable `label` is used when reporting errors.
inline satisfy<c>(label: &string, p: (rune) -> bool) -> zonk(c, rune)

// Reads any single rune from the remaining stream.
constant any-rune<c>: zonk(c, rune)

// Succeeds only when the head of the remaining stream is equal to `t`.
inline chunk<c>(t: &string) -> zonk(c, unit)

// Consumes the input stream while `!p` is true.
inline take-while<c>(!p: (rune) -> bool) -> zonk(c, string)

// A non-empty version of `take-while`.
inline take-while-1<c>(!p: (rune) -> bool) -> zonk(c, string)

// Discards the input stream while `!p` is true.
inline drop-while<c>(!p: (rune) -> bool) -> zonk(c, unit)

// Skips ascii spaces.
constant ascii-space<c>: zonk(c, unit)

// Executes `p`, overriding the (possible) error message with `l`.
inline label<c, a>(l: &string, p: zonk(c, a)) -> zonk(c, a)

// `attempt(p)` is the same as `p` if `p` succeeds.
// `attempt(p)` rewinds the input stream if `p` fails.
inline attempt<c, a>(p: zonk(c, a)) -> zonk(c, a)

// `look-ahead(p)` rewinds the input stream if `p` succeeds.
// `look-ahead(p)` is the same as `p` if `p` fails.
inline look-ahead<c, a>(p: zonk(c, a)) -> zonk(c, a)

// `optional(p)` is the same as `p` if `p` succeeds.
// `optional(p)` suppresses the error of `p` and results in none if `p` fails.
// Note that `optional(p)` does not rewind the input stream automatically.
inline optional<c, a>(p: zonk(c, a)) -> zonk(c, ?a)

// Executes `p1`. If it succeeds, `branch(p1, p2)` returns the result of `p1`.
// If it fails, executes `p2`.
// Note that `branch(p1, p2)` does not rewind the input stream automatically.
inline branch<c, a>(p1: zonk(c, a), p2: zonk(c, a)) -> zonk(c, a)

// Succeeds only when `p` can parse the head of the input stream.
// `not-followed-by` does not consume the input stream.
// The variable `label` is used when reporting errors.
inline not-followed-by<c, a>(label: &string, p: zonk(c, a)) -> zonk(c, unit)

// Tries given parsers one by one.
// The variable `label` is used when reporting errors.
// Note that each candidate should be wrapped in `attempt` if it may consume input before failing.
inline choice<c, a>(label: &string, candidates: list(zonk(c, a)), fallback: zonk(c, a)) -> zonk(c, a)

// Parses the input stream using `!p` iteratively until it fails.
// This parser never fails.
inline many<c, a>(!p: zonk(c, a)) -> zonk(c, list(a))

// Parses the input stream using `!p` iteratively until it fails.
// This parser succeeds only if `!p` successfully parses the input stream at least once.
inline some<c, a>(!p: zonk(c, a)) -> zonk(c, list(a))
```

### Presets

```neut
// A regex type
data regex {
| Any(label: &string, runes: list(rune))
| Chunk(chunk-text: &string)
| Choose(label: &string, candidates: list(regex), fallback: regex)
| Join(components: list(regex))
| Repeat(component: regex)
| End-Of-Input
}

// Returns True only if the regex `r` recognizes `input`.
inline recognize(r: regex, input: string) -> bool

// Skips space characters.
constant skip-space<c>: zonk(c, unit)

// Parses a symbol.
inline read-symbol<c>() -> zonk(c, string)

// Parses an integer.
constant read-int<c>: zonk(c, int)

// Parses a float.
constant read-float<c>: zonk(c, float)

// Parses integers separated by space characters and stores them into a list.
define read-int-list<c>(size: int) -> zonk(c, list(int))

// Parses values separated by space characters and stores them into a vector.
inline read-vector<c, a>(size: int, !p: zonk(c, a)) -> zonk(c, vector(a))

// Parses integers separated by space characters and stores them into a vector.
define read-int-vector<c>(size: int) -> zonk(c, vector(int))

// A float-version of `read-int-list`.
define read-float-list<c>(size: int) -> zonk(c, list(float))

// A float-version of `read-int-vector`.
define read-float-vector<c>(size: int) -> zonk(c, vector(float))
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
define get-point<c>(k: &zonk-kit(c)) -> point
```

## Example

Executing `zen` in the following example should output `pass`:

```neut
import {
  core.string.io {print-line},
  this.make-parse-error {make-parse-error},
  this.make-zonk-kit {make-zonk-kit},
  this.parse {choice, chunk, end-of-input, many, some, zonk},
  this.error {report},
}

constant sample-parser: zonk(unit, unit) {
  // constructs a parser
  (k) => {
    // accepts: foo
    try _ = chunk("foo")(k);
    // accepts: (baz|test|bar)
    try _ = choice("baz, test, or bar", List[chunk("baz"), chunk("test")], chunk("bar"))(k);
    // accepts: (qux)+
    try _ = some(chunk("qux"))(k);
    // accepts: (yo)*
    try _ = many(chunk("yo"))(k);
    // accepts: end-of-input
    try _ = end-of-input(k);
    Right(Unit)
  }
}

define zen() -> unit {
  pin k = make-zonk-kit(*"foobarquxquxyoyoyoyo", Unit);
  match sample-parser(k) {
  | Right(_) =>
    print("pass\n")
  | Left(e) =>
    let err = make-parse-error(k, e);
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
