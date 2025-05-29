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
    fn is_matching(&self, data: &[T]) -> (bool, usize);

    /// Returns the size of the data to match.
    fn size(&self) -> usize;
}
```

The matcher method returns a tuple `(found, index)` where `found` is a boolean and `index` is the number of 
characters matched.

Like the [Scanner](scanner.html) trait, the `Match` trait is generic over the type of the data it matches.

To come back to our example, we can create a struct that implements the `Match` trait for the substring `hello`.

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
    let hello = Hello;
    assert_eq!(hello.matcher(b"hello world"), (true, hello.size()));
    assert_eq!(hello.matcher(b"world is beautiful"), (false, 5));
}
```