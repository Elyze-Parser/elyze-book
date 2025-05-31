# Peek from Visitor

It may be interesting to be able to peek from a Visitor.

You can implement by yourself the `Peekable` trait, or you can let Elyze do it for you.

We just need to say to Elyze that we want it to implement the Peekable trait using the Visitor pattern.

```rust
# extern crate elyze;
use elyze::peek::{DefaultPeekableImplementation, PeekableImplementation};

impl PeekableImplementation for CloseParentheses {
    type Type = DefaultPeekableImplementation;
}
```

This will automatically enable the Peekable trait for CloseParentheses.

```rust
# extern crate elyze;
use elyze::errors::ParseResult;
use elyze::matcher::Match;
use elyze::peek::{peek, Until};
use elyze::scanner::Scanner;

// The Default makes the `CloseParentheses` structure 
// implements the `Visitor` trait
#[derive(Default)]
struct CloseParentheses;

// implementing the `Match` trait needed by Until
impl Match<u8> for CloseParentheses {
    fn is_matching(&self, data: &[u8]) -> (bool, usize) {
        if data[0] == b')' {
            (true, 1)
        } else {
            (false, 0)
        }
    }

    fn size(&self) -> usize {
        1
    }
}

/// Active the Default implementation of Peekable for CloseParentheses
impl PeekableImplementation for CloseParentheses {
    type Type = DefaultPeekableImplementation;
}

fn main() -> ParseResult<()> {
    let data = b"( 7 * ( 1 + 2 ) )";
    let mut scanner = Scanner::new(data);
    scanner.bump_by(7); // consumes : ( 7 * (
    
    // peek the first ")"
    let result = peek(CloseParentheses, &scanner)?;
    if let Some(peeking) = result {
        println!(
            "{:?}",
            // the peek_slice method returns the slice of recognized data without the end element
            String::from_utf8_lossy(peeking.peeked_slice()) // 1 + 2
        );
    } else {
        println!("not found");
    }
    println!(
        "scanner: {:?}",
        // the scanner itself remains unchanged
        String::from_utf8_lossy(scanner.remaining()) // scanner: " 1 + 2 ) )"
    );
    Ok(())
}
```

The result of the peeking is the slice until the first element peeked.