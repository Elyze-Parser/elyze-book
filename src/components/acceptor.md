# Acceptor

The `Accecptor` is quite the same thing as the `Recognizer` but instead of taking `Recognizable` objects, 
it takes `Visitor` objects.

Let's define two visitors.

```rust
# extern crate elyze;
#[derive(Default, Debug)]
struct OperatorPlus;
#[derive(Default, Debug)]
struct OperatorMinus;

impl Match<u8> for OperatorPlus {
    fn is_matching(&self, data: &[u8]) -> (bool, usize) {
        match_pattern(b"+", data)
    }
    fn size(&self) -> usize {
        1
    }
}
impl Match<u8> for OperatorMinus {
    fn is_matching(&self, data: &[u8]) -> (bool, usize) {
        match_pattern(b"-", data)
    }
    fn size(&self) -> usize {
        1
    }
}
```

And another more complex.

```rust
#[derive(Default)]
struct Hello;
#[derive(Default)]
struct Space;
#[derive(Default)]
struct World;

impl Match<u8> for Hello {
    fn is_matching(&self, data: &[u8]) -> (bool, usize) {
        (&data[..5] == b"hello", 5)
    }

    fn size(&self) -> usize {
        5
    }
}

impl Match<u8> for Space {
    fn is_matching(&self, data: &[u8]) -> (bool, usize) {
        (data[0] as char == ' ', 1)
    }

    fn size(&self) -> usize {
        1
    }
}

impl Match<u8> for World {
    fn is_matching(&self, data: &[u8]) -> (bool, usize) {
        (&data[..5] == b"world", 5)
    }

    fn size(&self) -> usize {
        5
    }
}

// define a structure to implement the `Visitor` trait
#[derive(Debug)]
struct HelloWorld;

impl<'a> Visitor<'a, u8> for HelloWorld {
    fn accept(scanner: &mut Scanner<'a, u8>) -> ParseResult<Self> {
        Hello::accept(scanner)?; // accept the word "hello"
        Space::accept(scanner)?; // accept the space character?; // recognize the space character
        World::accept(scanner)?; // accept the word "world"?; // recognize the word "world"
        // return the `HelloWorld` object
        Ok(HelloWorld)
    }
}
```

We have now, 3 visitors : `OperatorPlus`, `OperatorMinus` and `HelloWorld`.

Because all `Acceptor` result must be homogenous, we use an enumeration.

```rust
# extern crate elyze;
#[derive(Debug)]
enum Operator {
    Plus(OperatorPlus),
    Minus(OperatorMinus),
    HelloWorld(HelloWorld),
}
```

Then we can use it.

```rust
# extern crate elyze;
use elyze::acceptor::Acceptor;
fn main() -> ParseResult<()> {
    let data = b"+ 2";
    let mut scanner = Scanner::new(data);
    let accepted = Acceptor::new(&mut scanner)
        .try_or(Operator::Plus)?
        .try_or(Operator::HelloWorld)?
        .try_or(Operator::Minus)?
        .finish()
        .ok_or(ParseError::UnexpectedToken)?;

    println!("{:?}", accepted); // +

    let data = b"- 2";
    let mut scanner = Scanner::new(data);
    let accepted = Acceptor::new(&mut scanner)
        .try_or(Operator::Plus)?
        .try_or(Operator::HelloWorld)?
        .try_or(Operator::Minus)?
        .finish()
        .ok_or(ParseError::UnexpectedToken)?;

    println!("{:?}", accepted); // -

    let data = b"hello world 2";
    let mut scanner = Scanner::new(data);
    let accepted = Acceptor::new(&mut scanner)
        .try_or(Operator::Plus)?
        .try_or(Operator::HelloWorld)?
        .try_or(Operator::Minus)?
        .finish()
        .ok_or(ParseError::UnexpectedToken)?;

    println!("{:?}", accepted); // HelloWorld

    Ok(())
}
```

## Reusable Acceptor

Likewise the `Recognizer`, an `Acceptor` can be embedded in a `Visitor`.

```rust
# extern crate elyze;

impl<'a> Visitor<'a, u8> for Operator {
    fn accept(scanner: &mut Scanner<'a, u8>) -> ParseResult<Self> {
        Acceptor::new(scanner)
            .try_or(Operator::Plus)?
            .try_or(Operator::HelloWorld)?
            .try_or(Operator::Minus)?
            .finish()
            .ok_or(ParseError::UnexpectedToken)
    }
}
```

Which simplifies the code.

```rust
# extern crate elyze;
fn main() -> ParseResult<()> {
    let data = b"+ 2";
    let mut scanner = Scanner::new(data);
    let accepted = Operator::accept(&mut scanner)?;

    println!("{:?}", accepted); // +

    let data = b"- 2";
    let mut scanner = Scanner::new(data);
    let accepted = Operator::accept(&mut scanner)?;

    println!("{:?}", accepted); // -

    let data = b"hello world 2";
    let mut scanner = Scanner::new(data);
    let accepted = Operator::accept(&mut scanner)?;

    println!("{:?}", accepted); // HelloWorld

    Ok(())
}
```