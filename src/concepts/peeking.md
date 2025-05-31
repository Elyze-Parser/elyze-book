# Peeking data

Sometimes you want to look at the next data without consuming it.

Example, you have to match the starting of a parenthesis-delimited expression, and you want to check if one of the next 
characters is a `)`.

If so, you want the contents of the parenthesis to be consumed.

```
5 * ( 1 + 2 )
    ^
    you're here
```

To do that, you have to advance the scanner and check for each step if the scanner matches the close parenthesis.

Then you get the slice of the data between the open parenthesis and the close parenthesis.

```
 1 + 2
```

## Peekable trait

Elyze uses the `Peekable` trait to define peekable data. Peekable data stands for data that you can look at without
consuming it.

```rust
# extern crate elyze;

pub trait Peekable<'a, T> {
    /// Attempt to match the `Peekable` against the current position of the
    /// `Scanner`.
    ///
    /// This method will temporarily advance the position of the `Scanner` to
    /// find a match. If a match is found, the `Scanner` is rewound to the
    /// original position and a `PeekResult` is returned. If no match is found,
    /// the `Scanner` is rewound to the original position and an `Err` is
    /// returned.
    ///
    /// # Arguments
    ///
    /// * `data` - The `Scanner` to use when matching.
    ///
    /// # Returns
    ///
    /// A `PeekResult` if the `Peekable` matches the current position of the
    /// `Scanner`, or an `Err` otherwise.
    fn peek(&self, data: &Scanner<'a, T>) -> ParseResult<PeekResult>;
}
```

This trait defines an unique `peek` method.

This one remains the scanner unchanged and returns a `PeekResult`.

The `PeekResult` is shield by a `ParseResult` because peeking can fail either by recognizing or by accepting the data.
The error is propagated and left for the caller to handle it.

## PeekResult

The `PeekResult` itself is an enumeration.

```rust
# extern crate elyze;
pub enum PeekResult {
    /// The match was successful.
    Found {
        // The last index of the end slice
        end_slice: usize,
        // The size of the start element
        start_element_size: usize,
        // The size of the end element
        end_element_size: usize,
    },
    /// The match was unsuccessful.
    NotFound,
}
```

In its `Found` variant it embeds the last index of the end slice, the size of the start element and the size of the end element.


### Example

Let's implement a Match for the closing parenthesis.

```rust
# extern crate elyze; 
struct CloseParentheses;

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
```

Then define something that will bear the `Peekable` trait.

```rust
struct ParenthesesGroup;
```

Then implement the `Peekable`

```rust
impl<'a> Peekable<'a, u8> for ParenthesesGroup {
    fn peek(&self, scanner: &Scanner<'a, u8>) -> ParseResult<PeekResult> {
        // create an internal scanner allowing to peek data without alterating the original scanner
        let mut inner_scanner = Scanner::new(&scanner.remaining());

        // loop on each byte until we find a close parenthesis
        loop {
            if inner_scanner.is_empty() {
                // we have reached the end without finding a close parenthesis
                break;
            }
            if CloseParentheses.recognize(&mut inner_scanner)?.is_some() {
                // we have found a close parenthesis
                return Ok(PeekResult::Found {
                    // we return the position of the close parenthesis
                    end_slice: inner_scanner.current_position(),
                    // our peeking doesn't include a start element
                    start_element_size: 0,
                    // the size of the end element is a close parenthesis of 1 byte
                    end_element_size: 1,
                });
            }

            // consume the current byte
            inner_scanner.bump_by(1);
        }

        // At this point, we have reached the end of available data without finding a close parenthesis
        Ok(PeekResult::NotFound)
    }
}
```

Its implementation is not perfect, it takes the first close parenthesis and doesn't take into account the case where
there are multiple close parentheses in the case of nested parentheses, for example.

But enough to demonstrate the concept.

```rust
# extern crate elyze;

fn main() -> ParseResult<()> {
    let data = b"7 * ( 1 + 2 )";
    let mut scanner = Scanner::new(data);
    scanner.bump_by(5); // consumes : 7 * (
    let result = ParenthesesGroup.peek(&scanner)?;
    if let PeekResult::Found {
        end_slice,
        end_element_size,
        ..
    } = result
    {
        println!(
            "{:?}",
            // to found the real size of enclosed data, we need to subtract the size of the end element
            String::from_utf8_lossy(&scanner.remaining()[..end_slice - end_element_size]) // 1 + 2
        );
    } else {
        println!("not found");
    }
    println!(
        "scanner: {:?}",
        // the scanner itself remains unchanged
        String::from_utf8_lossy(scanner.remaining()) // scanner: " 1 + 2 )"
    );
    Ok(())
}
```

## Peeking

To stroll a successful peek, Elyze defines a structure called `Peeking`

```rust
pub struct Peeking<'a, T> {
    /// The start of the match.
    pub start_element_size: usize,
    /// The end of the match.
    pub end_element_size: usize,
    /// The length of peeked slice.
    pub end_slice: usize,
    /// The data that was peeked.
    pub data: &'a [T],
}
```

Like you can see, the `Peeking` struct embeds PeekResult::Found and the data slice.

## peek method

This `Peeking` is used by the `peek` method.

```rust
# extern crate elyze;

/// Attempt to match a `Peekable` against the current position of a `Scanner`.
///
/// This function will temporarily advance the position of the `Scanner` to find
/// a match. If a match is found, the `Scanner` is rewound to the original
/// position and a `Peeking` is returned. If no match is found, the `Scanner` is
/// rewound to the original position and an `Err` is returned.
///
/// # Arguments
///
/// * `peekable` - The `Peekable` to attempt to match.
/// * `scanner` - The `Scanner` to use when matching.
///
/// # Returns
///
/// A `Peeking` if the `Peekable` matches the current position of the `Scanner`,
/// or an `Err` otherwise.
pub fn peek<'a, T, P: Peekable<'a, T>>(
    peekable: P,
    scanner: &Scanner<'a, T>,
) -> ParseResult<Option<Peeking<'a, T>>>;
```

This one is a short syntax of using directly the `Peekable::peek` method.

It takes care of the arithmetic data slice for you. 

```rust
# extern crate elyze;
fn main() -> ParseResult<()> {
    let data = b"7 * ( 1 + 2 )";
    let mut scanner = Scanner::new(data);
    scanner.bump_by(5); // consumes : 7 * (
    
    // use peek method instead of ParenthesesGroup.peek
    let result = peek(ParenthesesGroup, &scanner)?;
    if let Some(peeking) = result {
        println!(
            "{:?}",
            // the peek_slice method returns the slice of recognized without the end element
            String::from_utf8_lossy(peeking.peeked_slice()) // 1 + 2
        );
    } else {
        println!("not found");
    }
    println!(
        "scanner: {:?}",
        // the scanner itself remains unchanged
        String::from_utf8_lossy(scanner.remaining()) // scanner: " 1 + 2 )"
    );
    Ok(())
}
```