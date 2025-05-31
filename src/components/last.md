# Last

If you want the last element peekable in the data, you can use the `Last` modifier.

The `Last` modifier takes a `Peekable` as argument. So it may be a `Visitor`.

The code remains the same. But the behavior is different.

Instead of returning at the first peeked element, it will advance an internal scanner and re-apply the peek operation 
until reaching the end of the data.

The last element peeked is returned.

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