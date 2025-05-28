# Scanner

A parser is like an eye that sweeps over the data to find relevant elements.

To represent this eye, Elyze uses a structure called a [Scanner](https://docs.rs/elyze/#scanner).

It's a thin wrapper around the [std::io::Cursor](https://doc.rust-lang.org/std/io/struct.Cursor.html). It allows
 advancing or rewinding the cursor position as the parsing progresses.

```rust
# use std::io::Cursor;
/// Wrapper around a `Cursor`.
#[derive(Debug, PartialEq)]
pub struct Scanner<'a, T> {
    /// The internal cursor.
    cursor: Cursor<&'a [T]>,
}
```

The scanner is generic over the type of the data it scans. This implies that the scanner can be used to scan
data of any type. It's intended to be used over bytes. But you can use it over a slice of structure if you want to.

```rust
# extern crate elyze;
// import the scanner
use elyze::scanner::Scanner;

struct Foo;

fn main() {
    // create a scanner over a slice of arbitrary data
    let mut scanner = Scanner::new(&[Foo, Foo, Foo]);
}
```

And because a reference over a slice of data, no `Clone` nor `Copy` constraint is required.


The scanner is a reference to the data it scans.

## Manipulate the cursor

As your parse will progress, you want to make the scanner move too.

### Move forward

You can call the `bump_by` method to move forward the cursor
position in the slice of data embedded in the scanner by the number of elements you have consumed.

```rust,ignore
impl<'a, T> Scanner<'a, T> {
    /// Move the internal cursor forward by `n` positions.
    ///
    /// # Arguments
    ///
    /// * `n` - The number of positions to move the cursor forward.
    ///
    /// # Panics
    ///
    /// Panics if the internal cursor is moved past the end of the data.
    pub fn bump_by(&mut self, n: usize);
}
```

Let's take an example, you have a slice of bytes, and you want to move by 3 bytes the scanner.

```rust,ignore
let mut scanner = Scanner::new(&[0x01, 0x02, 0x03, 0x04, 0x05]);
scanner.bump_by(3);
```

The scanner will be at position 3. So it's now pointing to the byte `0x04`.

### Move backward

The `rewind` method moves backward the cursor. It's useful when your parser takes 
a wrong decision path, and you want to rewind the scanner to the previous state.

```rust,ignore
/// Move the internal cursor backward by `n` positions.
    ///
    /// # Arguments
    ///
    /// * `n` - The number of positions to move the cursor backward.
    ///
    /// # Panics
    ///
    /// Panics if the internal cursor is moved to a position before the start of the data.
    pub fn rewind(&mut self, n: usize);
}
```

Like in this example, you have a slice of bytes, you first move by 3 bytes the scanner, but it found that was not
possible to continue the parsing, but it is maybe possible to parse something else. So you have to go back to the
previous state.

```rust,ignore
let mut scanner = Scanner::new(&[0x01, 0x02, 0x03, 0x04, 0x05]);
scanner.bump_by(3);
// do something afterward but we want to go back to the previous state
scanner.rewind(3);
```

The scanner will be back at position 0. So it's pointing to the byte `0x01` again.

### Get the current position

It can be found that your parsing logic needs to know the current position of the scanner.

To get it, you can call the `current_position` method.

```rust,ignore
impl<'a, T> Scanner<'a, T> {
    /// Return the current position of the internal cursor.
    pub fn current_position(&self) -> usize
}
```

Example

```rust,ignore
let mut scanner = Scanner::new(&[0x01, 0x02, 0x03, 0x04, 0x05]);
assert_eq!(scanner.current_position(), 0);
scanner.bump_by(3);
assert_eq!(scanner.current_position(), 3);
```

### Move to a specific position

If you know exactly where you want to go, you can call the `jump_to` method.

Contrary to the `bump_by` and `rewind` methods, the `jump_to` methods which works on relative positions, the `jump_to` takes an
absolute position to move to.

```rust,ignore
impl<'a, T> Scanner<'a, T> {
    /// Move the internal cursor to the specified position.
    ///
    /// # Arguments
    ///
    /// * `n` - The position to move the cursor to.
    ///
    /// # Panics
    ///
    /// Panics if the internal cursor is moved past the end of the data.
    pub fn jump_to(&mut self, n: usize);
}
```

Like in this example, you have a slice of bytes, you first move by 3 bytes the scanner, and some treament force you to go to , so we jump to the absolute position 1.

```rust,ignore
let mut scanner = Scanner::new(&[0x01, 0x02, 0x03, 0x04, 0x05]);
// a previous operation bumped the scanner
scanner.bump_by(1);

// record the initial position
let initial_position = scanner.current_position();
assert_eq!(initial_position, 1);

// do something
scanner.bump_by(3);
scanner.rewind(2);
scanner.bump_by(1);

// rewind the state
scanner.jump_to(initial_position);
assert_eq!(scanner.current_position(), 1);

// the remaining data is back to the initial state
assert_eq!(scanner.remaining(), &[0x02, 0x03, 0x04, 0x05]);
```

## Manipulate the data

The sole cursor is not enough. You also need to be able to access the data within the scanner.

### Get the remaining data

That's the most useful method it gets the remaining data of the scanner. Remaining means all data after the current 
cursor position.

The `Scanner` exposes the `remaining` method to do that.

```rust,ignore
impl<'a, T> Scanner<'a, T> {
    /// Return the remaining data of the internal cursor.
    pub fn remaining(&self) -> &'a [T]
}
```

Because of the returning lifetime, the data are ensured to live as long as the scanner slice of data does. The scanner
can be dropped but data from `remaining` call won't.

```rust,ignore
fn process(mut scanner: Scanner<'_, u8>) -> &[u8] {
    // do something with the data
    // then bump the scanner
    scanner.bump_by(3);
    // return the remaining
    scanner.remaining()
}

fn main() {
    let data = b"hello world";
    let remaining = process(Scanner::new(data));
    assert_eq!(remaining, b"lo world");
}
```

### Get all the data

If you want a reference to the whole data, you can call the `data` method.

it can be useful to get data earlier in the parsing process.

```rust,ignore
impl<'a, T> Scanner<'a, T> {
    /// Return the whole data of the internal cursor.
    pub fn data(&self) -> &'a [T]
}
```

Same as `remaining`, it's safe to drop the scanner.

```rust,ignore
fn process(mut scanner: Scanner<'_, u8>) -> &[u8] {
    // do something with the data
    // then bump the scanner
    scanner.bump_by(3);
    // return the whole data
    scanner.data()
}

fn main() {
    let data = b"hello world";
    let remaining = process(Scanner::new(data));
    assert_eq!(remaining, b"hello world");
}
```

