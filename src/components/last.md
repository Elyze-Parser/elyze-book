# Last

Works as [Until](until.html), but returns the last element instead of the first one.

Because peeking the last element is harder than taking the first one, we need to give more insight to Elyze to make it work. 

Not that much. We just need to say to Elyze that we want it to implement the Peekable trait using the Visitor pattern.

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
#[derive(Default)] // Enable the Visitor implementation for CloseParentheses
struct CloseParentheses;

/// Enable the PeekSize and Recognizable implementation for CloseParentheses
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
    let data = b"8 / ( 7 * ( 1 + 2 ) )";
    let mut scanner = Scanner::new(data);
    // consumes : "8 / ( " to reach the start of the enclosed data
    scanner.bump_by(b"8 / (".len());
    // because CloseParentheses implements the Peekable trait, we can peek it with the modifier Last
    let result = peek(Last::new(CloseParentheses), &scanner)?;
    if let Some(peeking) = result {
        println!(
            "{:?}",
            // the peek_slice method returns the all enclosed data not the first occurrence of ")" -> "7 * ( 1 + 2 "
            String::from_utf8_lossy(peeking.peeked_slice()) //  7 * ( 1 + 2 )
        );
    }
    Ok(())
}
```