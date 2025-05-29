# Recognizing data

Matching data is important, but there are a lot of checks to do.

You first need to check if the start of the data matches the pattern.

Then, if it does, you get the subslice of the data that matches the pattern.

And most importantly, you need to bump the scanner to the end of the matched data. If you don't, your parser will 
never see the next data.

Let's take the previous example, where we want to find the word `hello`.

```rust
# extern crate elyze;
use elyze::matcher::Match;

// define a structure to implement the `Match` trait
struct Hello;

// implement the `Match` trait
impl Match<u8> for Hello {
    fn is_matching(&self, data: &[u8]) -> (bool, usize) {
        // define the pattern to match
        let pattern = b"hello";
        // check if the subslice of data matches the pattern
        (&data[..pattern.len()] == pattern, pattern.len())
    }

    fn size(&self) -> usize {
        5
    }
}

fn main() {
    let mut scanner = Scanner::new(b"hello world");
    let (found, size) = Hello.is_matching(scanner.remaining());
    if !found {
        println!("not found");
        return;
    }
    let data = &scanner.remaining()[..size];
    scanner.bump_by(size);
    println!("found: {:?}", String::from_utf8_lossy(data)); // found: "hello"
    print!("remaining: {:?}", String::from_utf8_lossy(scanner.remaining())); // remaining: " world"
}
```

## Recognizable trait

Because it is a common operation to recognize an object, Elyze provides the `Recognizable` trait.

Its role is to provide a unified way to recognize objects.

```rust
pub trait Recognizable<'a, T, V>: Match<T> {
    /// Try to recognize the object for the given scanner.
    ///
    /// # Type Parameters
    /// V - The type of the object to recognize
    ///
    /// # Arguments
    /// * `scanner` - The scanner to recognize the object for.
    ///
    /// # Returns
    /// * `Ok(Some(V))` if the object was recognized,
    /// * `Ok(None)` if the object was not recognized,
    /// * `Err(ParseError)` if an error occurred
    ///
    fn recognize(self, scanner: &mut Scanner<'a, T>) -> ParseResult<Option<V>>;

    /// Try to recognize the object for the given scanner.
    ///
    /// # Arguments
    /// * `scanner` - The scanner to recognize the object for.
    ///
    /// # Returns
    /// * `Ok(Some(&[T]))` if the object was recognized,
    /// * `Ok(None)` if the object was not recognized,
    /// * `Err(ParseError)` if an error occurred
    fn recognize_slice(self, scanner: &mut Scanner<'a, T>) -> ParseResult<Option<&'a [T]>>;
}
```

It defines to methods:
- `recognize(self, scanner: &mut Scanner<'a, T>) -> ParseResult<Option<V>>`
- `recognize_slice(self, scanner: &mut Scanner<'a, T>) -> ParseResult<Option<&'a [T]>>`

The input is both the mutable reference to the scanner, but their return type differs.

The `recognize` method returns the value of the object that was recognized. Whereas the `recognize_slice` method returns
a slice of the data that was recognized.

This distinction is done because sometimes, you don't want the structure itself but rather the data that encodes it.

For example, we want to get all bytes until we get the first space character.

```rust,compile_fail
# extern crate elyze;
use elyze::errors::{ParseError, ParseResult};
use elyze::matcher::Match;
use elyze::recognizer::Recognizable;
use elyze::scanner::Scanner;

// define a structure to implement the `Match` trait
struct UntilFirstSpace;

// implement the `Match` trait
impl Match<u8> for UntilFirstSpace {
    /// Check if the given data matches the pattern.
    fn is_matching(&self, data: &[u8]) -> (bool, usize) {
        let mut pos = 0;
        while pos < data.len() && data[pos] != b' ' {
            pos += 1;
        }
        (pos > 0, pos)
    }

    // The size of the object is unknown
    fn size(&self) -> usize {
        0
    }
}

// implement the `Recognizable` trait
impl<'a> Recognizable<'a, u8, UntilFirstSpace> for UntilFirstSpace {
    fn recognize(self, scanner: &mut Scanner<'a, u8>) -> ParseResult<Option<UntilFirstSpace>> {
        // check if the scanner has enough data
        if self.size() > scanner.remaining().len() {
            return Err(ParseError::UnexpectedEndOfInput);
        }

        let data = scanner.remaining();

        let (result, size) = self.is_matching(data);
        if !result {
            return Ok(None);
        }
        if !scanner.is_empty() {
            scanner.bump_by(size);
        }
        Ok(Some(self))
    }

    /// Try to recognize the object for the given scanner.
    /// Return the slice of elements that were recognized.
    fn recognize_slice(self, scanner: &mut Scanner<'a, u8>) -> ParseResult<Option<&'a [u8]>> {
        // Check if the scanner is empty
        if scanner.is_empty() {
            return Err(ParseError::UnexpectedEndOfInput);
        }

        let data = scanner.remaining();

        let (result, size) = self.is_matching(data);
        if !result {
            return Ok(None);
        }
        if !scanner.is_empty() {
            scanner.bump_by(size);
        }
        Ok(Some(&data[..size]))
    }
}

fn main() {
    let mut scanner = Scanner::new(b"hello world");
    let result = UntilFirstSpace
        .recognize_slice(&mut scanner)
        .expect("failed to parse");
    println!("{:?}", result.map(|s| String::from_utf8_lossy(s))); // Some("hello")

    let mut scanner = Scanner::new(b"loooooooooong string");
    let result = UntilFirstSpace
        .recognize_slice(&mut scanner)
        .expect("failed to parse");
    println!("{:?}", result.map(|s| String::from_utf8_lossy(s))); // Some("loooooooooong")
}
```

But this code won't compile, the rustc will complain that there is a conflict implementation.

```
error[E0119]: conflicting implementations of trait `Recognizable<'_, u8, UntilFirstSpace>` for type `UntilFirstSpace`
   |
25 | impl<'a> Recognizable<'a, u8, UntilFirstSpace> for UntilFirstSpace {
   | ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
   |
   = note: conflicting implementation in crate `elyze`:
           - impl<'a, T, M> Recognizable<'a, T, M> for M
             where M: elyze::matcher::Match<T>;
```

And effectively, Elyze has already done the work for you using a marvelous feature of the rust language called the
[blanket implementation](https://medium.com/@mikecode/rust-blanket-implementation-07cd8357071e).

This one says that all "things" implementing the `Match` trait also implement the `Recognizable` trait.

Here is it is the blanket implementation for the `Recognizable` trait against any `M` that implements the `Match` trait:

```rust,ignore
/// Recognize an object for the given scanner.
/// Return the recognized object.
impl<'a, T, M: Match<T>> Recognizable<'a, T, M> for M {
    fn recognize(self, scanner: &mut Scanner<'a, T>) -> ParseResult<Option<M>> {
        // check if the scanner has enough data
        if self.size() > scanner.remaining().len() {
            return Err(ParseError::UnexpectedEndOfInput);
        }

        let data = scanner.remaining();

        // check if the data matches the pattern
        let (result, size) = self.is_matching(data);
        if !result {
            return Ok(None);
        }
        
        // bump the scanner if it's not empty
        if !scanner.is_empty() {
            scanner.bump_by(size);
        }
        
        // return the object
        Ok(Some(self))
    }

    /// Try to recognize the object for the given scanner.
    /// Return the slice of elements that were recognized.
    fn recognize_slice(self, scanner: &mut Scanner<'a, T>) -> ParseResult<Option<&'a [T]>> {
        // Check if the scanner is empty
        if scanner.is_empty() {
            return Err(ParseError::UnexpectedEndOfInput);
        }

        let data = scanner.remaining();

        // check if the data matches the pattern
        let (result, size) = self.is_matching(data);
        if !result {
            return Ok(None);
        }
        
        // bump the scanner if it's not empty
        if !scanner.is_empty() {
            scanner.bump_by(size);
        }
        
        // return the slice of data that was recognized
        Ok(Some(&data[..size]))
    }
}
```

You must simplify your code into:

```rust
# extern crate elyze;
use elyze::matcher::Match;
use elyze::recognizer::Recognizable;
use elyze::scanner::Scanner;

// define a structure to implement the `Match` trait
struct UntilFirstSpace;

// implement the `Match` trait
impl Match<u8> for UntilFirstSpace {
    fn is_matching(&self, data: &[u8]) -> (bool, usize) {
        let mut pos = 0;
        while pos < data.len() && data[pos] != b' ' {
            pos += 1;
        }
        (pos > 0, pos)
    }

    // The size of the object is unknown
    fn size(&self) -> usize {
        0
    }
}

fn main() {
    let mut scanner = Scanner::new(b"hello world");
    let result = UntilFirstSpace
        .recognize_slice(&mut scanner)
        .expect("failed to parse");
    println!("{:?}", result.map(|s| String::from_utf8_lossy(s))); // Some("hello")

    let mut scanner = Scanner::new(b"loooooooooong string");
    let result = UntilFirstSpace
        .recognize_slice(&mut scanner)
        .expect("failed to parse");
    println!("{:?}", result.map(|s| String::from_utf8_lossy(s))); // Some("loooooooooong")
}
```

### Signature breakdown

Because function signatures are polymorphic, they become a bit more complicated.

```rust,ignore
impl<'a, T, M: Match<T>> Recognizable<'a, T, M> for M {
    pub fn recognize(self, scanner: &mut Scanner<'a, T>) -> ParseResult<Option<M>>
}
```
- `'a` type parameter is the lifetime of the data to parse.
- `T` type parameter is the type of the data to parse.
- `M` type parameter is the type of the object that we want to recognize.

We implement the `Recognizable` for:
- `'a` the lifetime of the data to parse.
- `T` the type of the data to parse.
- `M` the type of the object that we want to recognize.

And the return type is `ParseResult<Option<M>>`.

The recognition may fail, and even in case of success, the process couldn't be able to recognize the object.

The same goes for the `recognize_slice` function but differs in the return type which is `&'a [T]` instead of `M`.

```rust,ignore
impl<'a, T, M: Match<T>> Recognizable<'a, T, M> for M {
    pub fn recognize_slice(self, scanner: &mut Scanner<'a, T>) -> ParseResult<Option<&'a [T]>>
}
```

## Utility functions

With the actual toolbox, you're already able to write these lines:

```rust,ignore
fn main() {
    let mut scanner = Scanner::new(b"hello world");
    let data = Hello.recognize(&mut scanner).expect("failed to parse");

    if let Some(hello) = data {
        println!("found: {hello:?}"); // found: "Hello"
        print!(
            "remaining: {:?}",
            String::from_utf8_lossy(scanner.remaining())
        ); // remaining: " world"
    } else {
        println!("not found");
    }
}
```

That's great, but it's a bit verbose.

To help the readability, Elyze provides a few utility functions.

- `recognize`
- `recognize_slice`

Both are a thin wrapper around the `Recognizable` trait.

They basically call the `recognize` or `recognize_slice` function on the `Recognizable` trait, and transform the 
`None` variant to an `Err(ParseError::UnexpectedToken)`.

### recognize_slice

```rust
fn main() -> ParseResult<()> {
    let mut scanner = Scanner::new(b"hello world");
    let hello_string : &[u8] = recognize_slice(Hello, &mut scanner)?;

    println!("found: {}", String::from_utf8_lossy(hello_string)); // found: "hello"
    print!(
        "remaining: {:?}",
        String::from_utf8_lossy(scanner.remaining())
    ); // remaining: " world"

    Ok(())
}
```

The main benefit is the ability to use the `?` operator to handle errors. So you can chain recognize functions.

Example, recognize 3 successive "hello"s.

```rust
# extern crate elyze;
fn main() -> ParseResult<()> {
    let mut scanner = Scanner::new(b"hellohellohello world");
    // recognize the first "hello"
    recognize_slice(Hello, &mut scanner)?;
    // recognize the second "hello"
    recognize_slice(Hello, &mut scanner)?;
    // recognize the third "hello"
    recognize_slice(Hello, &mut scanner)?;

    Ok(())
}
```

Because `recognize_slice` is a thin wrapper around the `Recognizable` trait, it has the same type parameters as the
`Recognizable::recognize_slice` method's trait.

```rust,ignore
// the signature of the `recognize_slice` function
pub fn recognize_slice<'a, T, V, R>(
    recognizable: R,
    scanner: &mut Scanner<'a, T>,
) -> ParseResult<&'a [T]>
where
    R: Recognizable<'a, T, V>,
```

- `'a` type parameter is the lifetime of the data to parse.
- `T` type parameter is the type of the data to parse.
- `V` type parameter is the type of the object that we want to recognize.
- `R` type parameter is the type of the object that we want to recognize.

### recognize

Same as `recognize_slice`, the `recognize` function returns the object that was recognized.

```rust
# extern crate elyze;

fn main() -> ParseResult<()> {
    let mut scanner = Scanner::new(b"hello world");
    let hello : Hello = recognize(Hello, &mut scanner)?;

    println!("found: {hello:?}"); // found: "hello"
    print!(
        "remaining: {:?}",
        String::from_utf8_lossy(scanner.remaining())
    ); // remaining: " world"

    Ok(())
}
```

This time that's the `Hello` structure which is returned and not the `&[u8]` slice.

Its signature is quite the same as the `recognize_slice` function, but differs in the return type.

```rust,ignore
```rust,ignore
// the signature of the `recognize` function
pub fn recognize<'a, T, V, R>(
    recognizable: R,
    scanner: &mut Scanner<'a, T>,
) -> ParseResult<V>
where
    R: Recognizable<'a, T, V>,
```

`V` is the type of the object that was recognized.