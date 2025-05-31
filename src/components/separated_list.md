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
## Trailing separator

**Caution**: The separate list expects an element after successfully accepting the separator.

```
1~~~2~~~3~~~4~~~
```

Will make fail the parse.

Data must be cleanup before using it.

```
1~~~2~~~3~~~4
```

To avoid this, Elize exposes a function called `get_scanner_without_trailing_separator`.

```rust
# extern crate elyze;
/// Return a scanner without the trailing separator.
///
/// # Arguments
///
/// * `element` - The peekable element.
/// * `separator` - The peekable separator.
/// * `scanner` - The scanner.
///
/// # Returns
///
/// A `ParseResult` containing a `Scanner` without the trailing separator.
pub fn get_scanner_without_trailing_separator<'a, T, P1, P2>(
    element: P1,
    separator: P2,
    scanner: &Scanner<'a, T>,
) -> ParseResult<Scanner<'a, T>>
where
    P1: Peekable<'a, T> + PeekableImplementation<Type = DefaultPeekableImplementation>,
    P2: Peekable<'a, T> + PeekableImplementation<Type = DefaultPeekableImplementation>;
```

It takes two `Peekable` as arguments. The first is the `element` and the second is the `separator`. And a reference to a `Scanner` as third argument.

This function returns a `Scanner` truncated to not include the last separator.

### Example

We can create a visitor to use it.

```rust
# extern crate elyze;
use elyze::separated_list::get_scanner_without_trailing_separator;

#[derive(Debug)]
struct NumberList {
    data: Vec<usize>,
}

impl<'a> Visitor<'a, u8> for NumberList {
    fn accept(scanner: &mut Scanner<'a, u8>) -> ParseResult<Self> {
        // get the scanner without the trailing separator
        let mut data_scanner =
            get_scanner_without_trailing_separator(TokenNumber, Separator, &scanner)?;

        // accept the separated list and extract the data
        let data = SeparatedList::<u8, Number<usize>, Separator>::accept(&mut data_scanner)?
            .data
            .into_iter()
            .map(|x| x.0)
            .collect::<Vec<usize>>();
        
        // clean up the scanner because all data has been extracted
        scanner.bump_by(scanner.data().len());

        Ok(NumberList { data })
    }
}       
```

This cleanup allows us to handle all problematic cases.

```rust
# extern crate elyze;
fn main() -> ParseResult<()> {
    // list of elements separated by a separator
    let data = b"1~~~2~~~3~~~4";
    let mut scanner = Scanner::new(data);

    let result = NumberList::accept(&mut scanner)?;
    println!("{:?}", result); // NumberList { data: [1, 2, 3, 4] }

    // list of elements separated by a separator with trailing separator
    let data = b"1~~~2~~~3~~~4~~~";
    let mut scanner = Scanner::new(data);

    let result = NumberList::accept(&mut scanner)?;
    println!("{:?}", result); // NumberList { data: [1, 2, 3, 4] }

    // list of 1 element with trailing separator
    let data = b"1~~~";
    let mut scanner = Scanner::new(data);

    let result = NumberList::accept(&mut scanner)?;
    println!("{:?}", result); // NumberList { data: [1] }

    // list of 1 element without trailing separator
    let data = b"1";
    let mut scanner = Scanner::new(data);

    let result = NumberList::accept(&mut scanner)?;
    println!("{:?}", result); // NumberList { data: [1] }

    // list of 0 elements
    let data = b"";
    let mut scanner = Scanner::new(data);

    let result = NumberList::accept(&mut scanner)?;
    println!("{:?}", result); // NumberList { data: [] }

    // bad data
    let data = b"bad~~~";
    let mut scanner = Scanner::new(data);

    let result = NumberList::accept(&mut scanner);
    println!("{:?}", result); // Err(UnexpectedToken)

    Ok(())
}
```
