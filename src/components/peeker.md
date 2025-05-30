# Peeker

The `Peeker` is like the `Acceptor` but it doesn't consume the data.

The other difference is that the `Peeker` returns a `Peeking` so the constraint to have 
a homogeneous data type isn't required.

This implies that any kind of `Visitor` can be used as `Peekable`.

Peeking a variation of elements is more complex than accepting it. You want the shortest possible 
data slice.

```
7 * ( 1 + 2 )
```

If you are peeking "*" or "+".

Following the order of registration, the peek will be either:
- `*` -> `+`  : `7 `
- `+` -> `*`  : `7 * ( 1 `

We want that the peeking always returns the shortest possible data slice so `7 ` independently of the order of registration.

Inverting the order of registration may fail the peeking with this slice:
```
( 1 + 2 ) * 7
```

To allow this behavior, we need to register all Peekables, then execute the peeking for each 
Peekable. If the peeked length is shorter than the internal state, this state is replaced. Otherwise, 
the internal state is left unchanged.

## Example

Let's take this selection of tokens:

```rust
enum OperatorTokens {
    Plus,
    Times,
}
```

`OperatorTokens` implements `Visitior`, so it's the case of `OperatorTokens::Plus` and `OperatorTokens::Times`.

You can then use the [Until](/concepts/peeking.html#until) with it to create a `Peekable` variant.

```rust
# extern crate elyze;

fn main() -> ParseResult<()> {
    let data = b"7 * ( 1 + 2 )";
    let scanner = Scanner::new(data);
    // create a peeker with a scanner
    let slice = Peeker::new(&scanner)
        // register the peekable until `OperatorTokens::Plus`
        .add_peekable(Until::new(OperatorTokens::Times))
        // register the peekable until `OperatorTokens::Plus`
        .add_peekable(Until::new(OperatorTokens::Plus))
        // peek the scanner for the first `OperatorTokens`
        .peek()?;
    // if found
    if let Some(slice) = slice {
        // the slice is the shortest possible
        println!("{:?}", String::from_utf8_lossy(slice.peeked_slice())); // "7 "
    }
    Ok(())
}
```

If we want something more reusable, we can create a struct that implements `Peekable` and put it inside the `Peeker`.

```rust
# extern crate elyze;

// define a struct that implements `Peekable`
struct FirstOperator;

// implement `Peekable` for `FirstOperator`
impl<'a> Peekable<'a, u8> for FirstOperator {
    fn peek(&self, scanner: &Scanner<'a, u8>) -> ParseResult<PeekResult> {
        Peeker::new(scanner)
            .add_peekable(Until::new(OperatorTokens::Plus))
            .add_peekable(Until::new(OperatorTokens::Times))
            .peek()
            // convert the `Peeking` into a `PeekResult`    
            .map(Into::into)
    }
}
```

Which simplifies the code.

```rust
# extern crate elyze;
fn main() -> ParseResult<()> {
    let data = b"7 * ( 1 + 2 )";
    let scanner = Scanner::new(data);
    let slice = peek(FirstOperator, &scanner)?;
    if let Some(slice) = slice {
        println!("{:?}", String::from_utf8_lossy(slice.peeked_slice())); // "7 "
    }

    let data = b"7 * ( 1 + 2 )";
    let scanner = Scanner::new(data);
    let slice = peek(FirstOperator, &scanner)?;
    if let Some(slice) = slice {
        println!("{:?}", String::from_utf8_lossy(slice.peeked_slice())); // "7 "
    }

    let data = b"1 + 2 * 7";
    let scanner = Scanner::new(data);
    let slice = peek(FirstOperator, &scanner)?;
    if let Some(slice) = slice {
        println!("{:?}", String::from_utf8_lossy(slice.peeked_slice())); // "1 "
    }

    Ok(())
}
```
