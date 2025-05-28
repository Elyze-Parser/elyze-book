# Match data

Parsing is a process that recognizes a pattern in the data.

If we come back with the eye analogy, when you are sweeping over the characters is the text, you will discover 
letter by letter, which word you are looking for, and so on.

If you take, for example, the list of characters `['h', 'e', 'l', 'l', 'o', ' ', w', 'o', 'r', 'l', 'd']` and you 
want to find the word `hello`.

You will see successively the letter `h`, then `e`, then `l`, then `l`, then `o`.

If all the characters match, you have found the word. Otherwise, it's something else.

## The `Match` trait

To materialize this idea, Elyze defines a `Match` trait.

```rust
pub trait Match<T> {
    /// Returns true if the data matches the pattern.
    ///
    /// # Arguments
    /// data - the data to match
    ///
    /// # Returns
    /// (found, number of matched characters)
    /// 
    /// (true, index) if the data matches the pattern,
    /// (false, index) otherwise
    fn matcher(&self, data: &[T]) -> (bool, usize);
}
```

The matcher method returns a tuple `(found, index)` where `found` is a boolean and `index` is the number of 
characters matched.

Like the [Scanner](./concepts/scanner.md) trait, the `Match` trait is generic over the type of the data it matches.

To come back to our example, we can create a struct that implements the `Match` trait for the substring `hello`.

```rust
# extern crate elyze;
use elyze::matcher::Match;
struct Hello;
impl Match<u8> for Hello {
    fn matcher(&self, data: &[u8]) -> (bool, usize) {
        (data == b"hello", 5)
    }
}

fn main() {
    let hello = Hello;
    assert_eq!(hello.matcher(b"hello"), (true, 5));
    assert_eq!(hello.matcher(b"world"), (false, 5));
}
```

## The `MatchSize` trait

Because the `Match` trait is generic over the type of the data it matches, it's sometimes necessary to know the 
size of the data to match.

To do this, Elyze defines a `MatchSize` trait.

```rust
pub trait MatchSize<T> {
    /// Returns the size of the data to match.
    fn size(&self) -> usize;
}
```

This trait is separated from the `Match` trait because some Elyze concepts only need the size but not the matching behavior.

```rust
# extern crate elyze;
use elyze::matcher::MatchSize;
struct Hello;
impl MatchSize<u8> for Hello {
    fn size(&self) -> usize {
        5
    }
}

fn main() {
    let hello = Hello;
    assert_eq!(hello.size(), 5);
}
```