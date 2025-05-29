# Visitor

The Elyze's keystone is the visitor pattern.

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