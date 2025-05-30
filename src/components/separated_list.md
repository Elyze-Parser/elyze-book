# Separated list

The `SeparatedList` component is a list of objects separated by a separator.

It is built using to Visitor. One is the element of the list to accept, and the other is the separator.

The `SeparatedList` will try to accept the `element` if it's a success push it into the list. Then, try to accept the separator. 
And repeat the process until a failure. 

If one of the Visitor (element or separator) fails, it will return the list of elements accepted.

```rust
# extern crate elyze;

// Define how to match a number
struct TokenNumber;

impl Match<u8> for TokenNumber {
    fn is_matching(&self, data: &[u8]) -> (bool, usize) {
        let mut pos = 0;
        while pos < data.len() && data[pos].is_ascii_digit() {
            pos += 1;
        }
        (pos > 0, pos)
    }
    fn size(&self) -> usize {
        0
    }
}

#[derive(Debug)]
struct NumberData(usize);

impl<'a> Visitor<'a, u8> for NumberData {
    fn accept(scanner: &mut Scanner<u8>) -> ParseResult<Self> {
        let slice = recognize_slice(TokenNumber, scanner)?;
        let number = std::str::from_utf8(slice)?.parse::<usize>()?;
        Ok(NumberData(number))
    }
}

// Define how to match a separator
#[derive(Debug)]
struct Separator;

impl<'a> Visitor<'a, u8> for Separator {
    fn accept(scanner: &mut Scanner<u8>) -> ParseResult<Self> {
        recognize(Token::Tilde, scanner)?;
        recognize(Token::Tilde, scanner)?;
        recognize(Token::Tilde, scanner)?;
        Ok(Separator)
    }
}

// Apply to a separated list
fn main() -> ParseResult<()> {
    let data = b"1~~~2~~~3~~~4";
    let mut scanner = Scanner::new(data);
    // define the separated list using types parameters NumberData and Separator
    let result = SeparatedList::<_, NumberData, Separator>::accept(&mut scanner)?
        // the inner list can be extracted with the `data` attribute
        .data;
    println!("{:?}", result); // Ok([NumberData(1), NumberData(2), NumberData(3), NumberData(4)])
    Ok(())
}
```

**Caution**: The separate list expects an element after successfully accepting the separator.

```
1~~~2~~~3~~~4~~~
```

Will make fail the parse.

Data must be cleanup before using it.

```
1~~~2~~~3~~~4
```
