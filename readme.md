# zonk

`zonk` is a parser combinator for the [Neut](https://vekatze.github.io/neut/) programming language.

## Installation

```sh
neut get zonk https://github.com/vekatze/zonk/raw/main/archive/0-1-34.tar.zst
```

## Types

### Main Definitions

```neut
// A monadic parser.
data parser(a)

// Monadic bind.
inline zonk<a, b>(p: parser(a), k: (a) -> parser(b)): parser(b)

// Monadic return.
inline return<a>(x: a): parser(a)

// Creates a parser state from the input string.
// Use this with `run`.
define new-state(input: text): cell(state)

// Executes parser, setting `st` as the initial input stream.
inline run<a>(p: parser(a), st: &cell(state)): except(error, a)

// Converts an error into a human-readable text.
define report(e: error): text
```

### Parser Combinators

```neut
// Succeeds only at the end of the input.
inline end-of-input: parser(unit)

// Succeeds only when the head rune of the remaining stream satisfies `p`.
inline satisfy(p: (rune) -> bool): parser(rune)

// Reads any single rune from the remaining stream.
inline any-rune: parser(rune)

// Succeeds only when the head of the remaining stream is equal to `t`.
inline chunk(t: &text): parser(unit)

// Consumes the input stream while `!p` is true.
inline take-while(!p: (rune) -> bool): parser(text)

// Discards the input stream while `!p` is true.
inline drop-while(!p: (rune) -> bool): parser(unit)

// Skips ascii spaces.
inline ascii-space: parser(unit)

// Executes the parser `p`, overriding the (possible) error message with `l`.
inline label<a>(l: &text, p: parser(a)): parser(a)

// Executing the parser `p`, suppressing any possible error messages.
inline hidden<a>(p: parser(a)): parser(a)

// `attempt(p)` is the same as `p` if `p` succeeds.
// `attempt(p)` rewinds the input stream if `p` fails.
inline attempt<a>(p: parser(a)): parser(a)

// `look-ahead(p)` rewinds the input stream if `p` succeeds.
// `look-ahead(p)` is the same as `p` if `p` fails.
inline look-ahead<a>(p: parser(a)): parser(a)

// A parser that always fails, reporting the expected inputs.
define fail<a>(expected: list(tag)): parser(a)

// `optional(p)` is the same as `p` if `p` succeeds.
// `optional(p)` suppresses the error of `p` and results in none if `p` fails.
inline optional<a>(p: parser(a)): parser(?a)

// Executes `p1`. If it succeeds, `branch(p1, p2)` returns the result of `p1`.
// If it fails, executes p2.
inline branch<a>(p1: parser(a), p2: parser(a)): parser(a)

// Succeeds only when `p` can parse the head of the input stream.
// `not-followed-by` does not consume the input stream.
inline not-followed-by<a>(p: parser(a)): parser(unit)

// Tries list on parsers one by one.
inline choice<a>(candidates: list(parser(a)), fallback: parser(a)): parser(a)

// Parses the input stream using `!p` iteratively until it fails.
// This parser never fails.
inline many<a>(!p: parser(a)): parser(list(a))

// Parses the input stream using `!p` iteratively until it fails.
// This parser succeeds only if `!p` successfully parse the input stream at least once.
inline some<a>(!p: parser(a)): parser(list(a))
```

### Presets

```neut
// A regex type
data regex {
| Any(list(rune))
| Chunk(&text)
| Choose(candidates: list(regex), fallback: regex)
| Join(list(regex))
| Repeat(regex)
}

// Returns True only if the regex `r` recognizes `input`.
inline recognize(r: regex, input: text): bool

// Skips space characters.
define skip-space(): parser(unit)

// Parses a symbol.
inline read-symbol(): parser(text)

// Parses an integer.
inline read-int(): parser(int)

// Parses a float.
inline read-float(): parser(float)

// Parses integers separated by space characters and stores them into a list.
define read-int-list(size: int): parser(list(int))

// Parses integers separated by space characters and stores them into a vector.
define read-int-vector(size: int): parser(vector(int))

// A float-version of `read-int-list`.
define read-float-list(size: int): parser(list(float))

// A float-version of `read-int-vector`.
define read-float-vector(size: int): parser(vector(float))
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
inline get-point(): parser(point)
```

## Example

Executing `zen` in the following example should output `pass`:

```neut
inline sample-parser: parser(unit) {
  // construct a parser
  with zonk {
    // accepts: foo
    bind _ = chunk("foo") in
    // accepts: (buz|test|bar)
    bind _ = choice([chunk("buz"), chunk("test")], chunk("bar")) in
    // accepts: (qux)+
    bind _ = some(chunk("qux")) in
    // accepts: (yo)*
    bind _ = many(chunk("yo")) in
    // accepts: end-of-input
    bind _ = end-of-input in
    return(Unit)
  }
}

define zen(): unit {
  // construct an input
  let some-input = new-state(*"foobarquxqux") in
  let result on some-input =
    // run the parser
    match run(sample-parser, some-input) {
    | Pass(_) =>
      print("pass\n")
    | Fail(_) =>
      print("fail\n")
    }
  in
  let _ = some-input in
  result
}
```
