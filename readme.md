# zonk

`zonk` is a parser combinator for the [Neut](https://vekatze.github.io/neut/) programming language.

## Example

Executing `zen` in the following example should output `pass`:

```neut
constant sample-parser: parser(unit) {
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
