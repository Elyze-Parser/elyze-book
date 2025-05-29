# Visitor

The Elyze's keystone is the [visitor pattern](https://refactoring.guru/design-patterns/visitor).

It is materialized by a `Visitor` trait.

```rust
# extern crate elyze;
use elyze::errors::ParseResult;
use elyze::scanner::Scanner;

/// A visitor pattern.
///
/// # Type parameters
///
/// * `'a` - The lifetime of the data to visit.
/// * `T` - The type of the data to visit.
pub trait Visitor<'a, T>: Sized {
    /// Try to accept the `Scanner` and return the result of the visit.
    ///
    /// # Arguments
    ///
    /// * `scanner` - The scanner to accept.
    ///
    /// # Returns
    ///
    /// The result of the visit.
    fn accept(scanner: &mut Scanner<'a, T>) -> ParseResult<Self>;
}
```

This one defines a unique method `accept` that takes a `Scanner` as argument and returns a `ParseResult` of the visitor.

The visitor returns itself when it is accepted by scanner data.

The `Visitor` is generic over the type of the data to visit, and its lifetime corresponds 
to the lifetime of the data visited.

## Accepting data

To accept scanner data, the object must implement the `Visitor` trait.

Let's say that you have three [`Recognizable`](recognizing.html#recognizable-trait) objects:
- `Hello` : recognize the word "hello"
- `Space` : recognize the space character
- `World` : recognize the word "world"

You want that your `HelloWorld` structure recognize the sentence "hello world" and return a `HelloWorld` object.

You have to implement the `Visitor` trait for the `HelloWorld` structure.

```rust
# extern crate elyze;
use elyze::errors::ParseResult;
use elyze::matcher::Match;
use elyze::recognizer::Recognizable;
use elyze::scanner::Scanner;
use elyze::visitor::Visitor;

// define a structure to implement the `Visitor` trait
#[derive(Debug)]
struct HelloWorld;

// implement the `Visitor` trait
impl<'a> Visitor<'a, u8> for HelloWorld {
    fn accept(scanner: &mut Scanner<'a, u8>) -> ParseResult<Self> {
        recognize(Hello, scanner)?; // recognize the word "hello"
        recognize(Space, scanner)?; // recognize the space character
        recognize(World, scanner)?; // recognize the word "world"
        // return the `HelloWorld` object
        Ok(HelloWorld)
    }
}

fn main() {
    let data = b"hello world";
    let mut scanner = Scanner::new(data);
    // Use the accept method on HelloWorld
    let result = HelloWorld::accept(&mut scanner);
    println!("{:?}", result); // Ok(HelloWorld)
}
```