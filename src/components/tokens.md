# Tokens

Based on the [Recognizable](recognizing.html) trait, a token is the atomic element you want to recognize
in your parse.

The idea behind the token recognition is to create a union type that will allow you to recognize any token
in the same way.

In Rust this union is materialized with an enumeration.

All you need to do is to implement the `Match` trait on your enumeration. And then you can use the
`recognize` function to recognize the token variant.

```rust
# extern crate elyze;
use elyze::errors::ParseResult;
use elyze::matcher::Match;
use elyze::recognizer::recognize;

// define a matching function
fn match_char(c: char, data: &[u8]) -> (bool, usize) {
    match data.get(0) {
        Some(&d) => (d == c as u8, 1),
        None => (false, 0),
    }
}

// create an enumeration of tokens to recognize
enum Token {
    Plus,
    Minus,
    Star,
    Slash,
    LParen,
    RParen,
}

// implement the `Match` trait for the `Token` enum
impl Match<u8> for Token {
    fn is_matching(&self, data: &[u8]) -> (bool, usize) {
        match self {
            Token::Plus => match_char('+', data),
            Token::Minus => match_char('-', data),
            Token::Star => match_char('*', data),
            Token::Slash => match_char('/', data),
            Token::LParen => match_char('(', data),
            Token::RParen => match_char(')', data),
        }
    }

    fn size(&self) -> usize {
        match self {
            Token::Plus => 1,
            Token::Minus => 1,
            Token::Star => 1,
            Token::Slash => 1,
            Token::LParen => 1,
            Token::RParen => 1,
        }
    }
}

// Profit !
fn main() -> ParseResult<()> {
    let data = b"((+-)*/)end";
    let mut scanner = elyze::scanner::Scanner::new(data);
    recognize(Token::LParen, &mut scanner)?;
    recognize(Token::LParen, &mut scanner)?;
    recognize(Token::Plus, &mut scanner)?;
    recognize(Token::Minus, &mut scanner)?;
    recognize(Token::RParen, &mut scanner)?;
    recognize(Token::Star, &mut scanner)?;
    recognize(Token::Slash, &mut scanner)?;
    recognize(Token::RParen, &mut scanner)?;

    print!("{:?}", String::from_utf8_lossy(scanner.remaining())); // "end"

    Ok(())
}
```