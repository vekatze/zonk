# zonk

`zonk` is a parser combinator for the [Neut](https://vekatze.github.io/neut/) programming language.

## Installation

```sh
neut get zonk https://github.com/vekatze/zonk/raw/main/archive/0-1-52.tar.zst
```

## Types

### Main Definitions

```neut
// An opaque type that holds information needed to execute parsers
data zonk-kit

// An opaque type that holds expected items
data expected-item

// The type of parsers
inline zonk(a) {
  (&zonk-kit) -> either(expected-item, a)
}

// Creates a zonk-kit for given text
define make-zonk-kit(input-stream: text): zonk-kit

// Converts an opaque `expected-item` to a parse error
define make-parse-error(k: &zonk-kit, expected: expected-item): parse-error

// Converts a parse error into a human-readable text
define report(e: error): text
```

### Parser Combinators

```neut
// Succeeds only at the end of the input.
inline end-of-input: zonk(unit)

// Succeeds only when the head rune of the remaining stream satisfies `p`.
// The variable `label` is used when reporting errors.
inline satisfy(label: &text, p: (rune) -> bool): zonk(rune)

// Reads any single rune from the remaining stream.
inline any-rune: zonk(rune)

// Succeeds only when the head of the remaining stream is equal to `t`.
inline chunk(t: &text): zonk(unit)

// Consumes the input stream while `!p` is true.
inline take-while(!p: (rune) -> bool): zonk(text)

// Discards the input stream while `!p` is true.
inline drop-while(!p: (rune) -> bool): zonk(unit)

// Skips ascii spaces.
inline ascii-space: zonk(unit)

// Executes `p`, overriding the (possible) error message with `l`.
inline label<a>(l: &text, p: zonk(a)): zonk(a)

// `attempt(p)` is the same as `p` if `p` succeeds.
// `attempt(p)` rewinds the input stream if `p` fails.
inline attempt<a>(p: zonk(a)): zonk(a)

// `look-ahead(p)` rewinds the input stream if `p` succeeds.
// `look-ahead(p)` is the same as `p` if `p` fails.
inline look-ahead<a>(p: zonk(a)): zonk(a)

// `optional(p)` is the same as `p` if `p` succeeds.
// `optional(p)` suppresses the error of `p` and results in none if `p` fails.
inline optional<a>(p: zonk(a)): zonk(?a)

// Executes `p1`. If it succeeds, `branch(p1, p2)` returns the result of `p1`.
// If it fails, executes p2.
inline branch<a>(p1: zonk(a), p2: zonk(a)): zonk(a)

// Succeeds only when `p` can parse the head of the input stream.
// `not-followed-by` does not consume the input stream.
// The variable `label` is used when reporting errors.
inline not-followed-by<a>(label: &text, p: zonk(a)): zonk(unit)

// Tries given parsers one by one.
// The variable `label` is used when reporting errors.
inline choice<a>(label: &text, candidates: list(zonk(a)), fallback: zonk(a)): zonk(a)

// Parses the input stream using `!p` iteratively until it fails.
// This parser never fails.
inline many<a>(!p: zonk(a)): zonk(list(a))

// Parses the input stream using `!p` iteratively until it fails.
// This parser succeeds only if `!p` successfully parse the input stream at least once.
inline some<a>(!p: zonk(a)): zonk(list(a))
```

### Presets

```neut
// A regex type
data regex {
| Any(label: &text, runes: list(rune))
| Chunk(chunk-text: &text)
| Choose(label: &text, candidates: list(regex), fallback: regex)
| Join(components: list(regex))
| Repeat(component: regex)
| End-Of-Input
}

// Returns True only if the regex `r` recognizes `input`.
inline recognize(r: regex, input: text): bool

// Skips space characters.
define skip-space(): zonk(unit)

// Parses a symbol.
inline read-symbol(): zonk(text)

// Parses an integer.
inline read-int(): zonk(int)

// Parses a float.
inline read-float(): zonk(float)

// Parses integers separated by space characters and stores them into a list.
define read-int-list(size: int): zonk(list(int))

// Parses integers separated by space characters and stores them into a vector.
define read-int-vector(size: int): zonk(vector(int))

// A float-version of `read-int-list`.
define read-float-list(size: int): zonk(list(float))

// A float-version of `read-int-vector`.
define read-float-vector(size: int): zonk(vector(float))
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
define get-point(k: &zonk-kit): point
```

## Example

Executing `zen` in the following example should output `pass`:

```neut
inline _sample-parser: zonk(unit) {
  // constructs a parser
  function (k) {
    // accepts: foo
    try _ = chunk("foo")(k);
    // accepts: (buz|test|bar)
    try _ = choice("buz, test, or bar", [chunk("buz"), chunk("test")], chunk("bar"))(k);
    // accepts: (qux)+
    try _ = some(chunk("qux"))(k);
    // accepts: (yo)*
    try _ = many(chunk("yo"))(k);
    // accepts: end-of-input
    try _ = end-of-input(k);
    Right(Unit)
  }
}

define zen(): unit {
  pin k = make-zonk-kit(*"foobarquxquxyoyoyoyo");
  match _sample-parser(k) {
  | Right(_) =>
    print("pass\n")
  | Left(_) =>
    let err = make-parse-error(k, e);
    pin err' = report(err);
    print("fail: ");
    print-line(err');
  }
}
```

If you rewrite the input as `fooXXXXquxquxyoyoyoyo`, the output will be as follows:

```text
fail: parse error at row 1, column 4:
expected:
  buz, test, or bar
found:
  "XXXX"
```
