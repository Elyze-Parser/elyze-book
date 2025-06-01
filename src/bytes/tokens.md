# Tokens

There are some patterns that we can recognize on bytes. All common symbols are grouped in a `Token` enumeration.

```rust
pub enum Token {
    /// The "(" character
    OpenParen,
    /// The `)` character
    CloseParen,
    /// The `,` character
    Comma,
    /// The `;` character
    Semicolon,
    /// The `:` character
    Colon,
    /// The whitespace character
    Whitespace,
    /// The `>` character
    GreaterThan,
    /// The `<` character
    LessThan,
    /// The `!` character
    Exclamation,
    /// The `'` character
    Quote,
    /// The `"` character
    DoubleQuote,
    /// The `=` character
    Equal,
    /// The `+` character
    Plus,
    /// The `-` character
    Dash,
    /// The `/` character
    Slash,
    /// The `*` character
    Star,
    /// The `%` character
    Percent,
    /// The `&` character
    Ampersand,
    /// The `|` character
    Pipe,
    /// The `^` character
    Caret,
    /// The `~` character
    Tilde,
    /// The `.` character
    Dot,
    /// The `?` character
    Question,
    /// The `@` character
    At,
    /// The `#` character
    Hash,
    /// The `$` character
    Dollar,
    /// The `\\` character
    Backslash,
    /// The `_` character
    Underscore,
    /// The `#` character
    Sharp,
    /// The `\n` character
    Ln,
    /// The `\r` character
    Cr,
    /// The `\t` character
    Tab,
    /// The `\r\n` character
    CrLn,
}
```

This one already implements the `Match`, `Recognizable`,`Visitor` and `Peekable` traits.

```rust
# extern crate elyze;
use elyze::bytes::token::Token;
use elyze::errors::{ParseError, ParseResult};
use elyze::peek::{peek, Last};
use elyze::recognizer::{recognize, Recognizer};
use elyze::scanner::Scanner;
use elyze::visitor::Visitor;

fn main() -> ParseResult<()> {
    let data = b"+-*";

    // use recognize
    let mut scanner = Scanner::new(data);
    let recognized = recognize(Token::Plus, &mut scanner)?;
    assert_eq!(recognized, Token::Plus);

    // use the recognizer
    let mut scanner = Scanner::new(data);
    let recognized = Recognizer::new(&mut scanner)
        .try_or(Token::Dash)?
        .try_or(Token::Plus)?
        .try_or(Token::Star)?
        .finish()
        .ok_or(ParseError::UnexpectedToken)?;
    assert_eq!(recognized, Token::Plus);

    // use the visitor
    let mut scanner = Scanner::new(data);
    let accepted = Token::accept(&mut scanner)?;
    assert_eq!(accepted, Token::Plus);

    // use peek
    let mut scanner = Scanner::new(data);
    let peeked = peek(Token::Dash, &mut scanner)?;
    if let Some(peeked) = peeked {
        assert_eq!(peeked.peeked_slice(), b"+");
    }

    // last token
    let data = b" 8 + ( 7 * ( 1 + 2 ) )";
    let mut scanner = Scanner::new(data);
    let peeked = peek(Last::new(Token::CloseParen), &mut scanner)?;
    if let Some(peeked) = peeked {
        assert_eq!(peeked.peeked_slice(), b" 8 + ( 7 * ( 1 + 2 ) ");
    }

    Ok(())
}
```

## Separated List

By playing will all these implementations, we can build a separated list of tokens non including the comma token.

```rust
# extern crate elyze;

// define a structure to implement Peekable
// using the Visitor pattern excluding the comma token
struct AnyTokenExceptComma;

// Enable the Peekable trait using the Visitor pattern
impl PeekableImplementation for AnyTokenExceptComma {
    type Type = DefaultPeekableImplementation;
}

// Define the PeekSize trait
impl PeekSize<u8> for AnyTokenExceptComma {
    fn peek_size(&self) -> usize {
        // The size is not important can be default to 0
        0
    }
}

// Define the Visitor trait for the AnyTokenExceptComma structure
// excluding the comma token
impl<'a> Visitor<'a, u8> for AnyTokenExceptComma {
    fn accept(scanner: &mut Scanner<'a, u8>) -> ParseResult<Self> {
        let token = Token::accept(scanner)?;
        match token {
            Token::Comma => Err(ParseError::UnexpectedToken),
            _ => Ok(AnyTokenExceptComma),
        }
    }
}

// Define a structure to implement Visitor
#[derive(Debug, PartialEq)]
struct TokenData(Token);

impl<'a> Visitor<'a, u8> for TokenData {
    fn accept(scanner: &mut Scanner<'a, u8>) -> ParseResult<Self> {
        let token = Token::accept(scanner)?;
        match token {
            Token::Comma => Err(ParseError::UnexpectedToken),
            _ => Ok(TokenData(token)),
        }
    }
}

// Define a structure to implement Visitor for the separator
struct SeparatorComma;

impl<'a> Visitor<'a, u8> for SeparatorComma {
    fn accept(scanner: &mut Scanner<'a, u8>) -> ParseResult<Self> {
        recognize(Token::Comma, scanner)?;
        Ok(SeparatorComma)
    }
}
```

Then we can build the separated list

```rust
# extern crate elyze;
fn main() -> ParseResult<()> {
    let data = b"*,-,+,/,";
    let scanner = Scanner::new(data);
    // clean up the data of its trailing comma
    let mut data_scanner =
        get_scanner_without_trailing_separator(AnyTokenExceptComma, Token::Comma, &scanner)?;
    assert_eq!(data_scanner.data(), b"*,-,+,/"); // data without a trailing comma
    // accept the separated list
    let list = SeparatedList::<u8, TokenData, SeparatorComma>::accept(&mut data_scanner)?;
    assert_eq!(
        list.data,
        vec![
            TokenData(Token::Star),
            TokenData(Token::Dash),
            TokenData(Token::Plus),
            TokenData(Token::Slash),
        ]
    );
    Ok(())
}
```