# Recognizer

The `Recognizer` is meant to fix the problem when you are matching a pattern which can vary.

That's the case with numeric operators like `+` or `-` for example.

The `Recognizer` works by matching `Recognizable` one by one until a match is found. Il no pattern matches, it returns
a `None` value.

The `Recognizer` takes a mutable to the `Scanner` as a parameter.

Then a chain of `try_or` methods is used to add `Recognizable` objects to the `Recognizer`.

```rust
# extern crate elyze;
use elyze::recognizer::Recognizer;

# fn main() -> ParseResult<()> { 
Recognizer::new(scanner)
    .try_or(Operator::Add)?
    .try_or(Operator::Sub)?
    .finish();
# }
```

## Step by step

The `Recognizer` works in 3 steps:
1. Initializing with a scanner
2. Add a `Recognizable` to the `Recognizer`
3. Call the `finish` method to get the result


### Step 1: Initializing with a scanner

```rust
# extern crate elyze;
use elyze::recognizer::Recognizer;
use elyze::scanner::Scanner;

fn main() {
    let data = b"+";
    let mut scanner = Scanner::new(data);
    let result = Recognizer::<u8, Operator>::new(&mut scanner);
}
```

The `Recognizer` is defined by 

```rust
# extern crate elyze;
/// A `Recognizer` is a type that wraps a `Scanner` and holds a successfully
/// recognized value.
///
/// When a value is successfully recognized, the `Recognizer` stores the value in
/// its `data` field and returns itself. If a value is not recognized, the
/// `Recognizer` rewinds the scanner to the previous position and returns itself.
///
/// # Type Parameters
///
/// * `T` - The type of the data to scan.
/// * `U` - The type of the value to recognize.
/// * `'a` - The lifetime of the data to scan.
/// * `'b` - The lifetime of the `Scanner`.
pub struct Recognizer<'a, 'b, T, R> {
    data: Option<R>,
    scanner: &'b mut Scanner<'a, T>,
}
```

That's why you need to specify the type of T and R in its `new` call.

### Step 2: Add a `Recognizable` to the `Recognizer`

Once to the `Recognizer` is initialized, you can add one or more `Recognizable` to it.

To do so, the `Recognizer` provides the `try_or` method.

```rust
# extern crate elyze;
impl<'a, 'b, T, R: Recognizable<'a, T, R>> Recognizer<'a, 'b, T, R> {
    pub fn try_or(mut self, element: R) -> ParseResult<Self>;
}
```

This one takes a `R` object where `R` implements the `Recognizable` trait. And returns a `ParseResult` containing the 
`Recognizer` object.

You can add as many `Recognizable` as you want. But by rust limitations, th `R` must be the same at each call. That's why
using an enumeration of [tokens](tokens.html) is a good idea.

Here are my tokens:

```rust
#[derive(Debug)]
enum Operator {
    Add,
    Sub,
}
```

It implements `Match<u8>`

```rust
# extern crate elyze;

fn main() -> ParseResult<()> {
    let data = b"+";
    let mut scanner = Scanner::new(data);
    // Initialize the recognizer (type can be inferred)
    let recognizer = Recognizer::new(&mut scanner);
    // Try to apply the recognizer on the operator add, if it fails, return an error
    let recognizer_add = recognizer.try_or(Operator::Add)?;
    // Try to apply the recognizer on the operator sub, if it fails, return an error
    let recognizer_add_and_sub = recognizer_add.try_or(Operator::Sub)?;
    
    Ok(())
}
```

The `try_or` method works as follows:
- Checks the internal state, if internal state is `None`:
  - It calls the `recognize` method of the `Recognizable` object.
  - If the `recognize` method returns `Ok(Some(value))`, it sets the internal state to `Some(value)`.
  - If the `recognize` method returns `Ok(None)`
  - If an error occurs, it returns the error.
- If the internal state is `Some`, it returns the `Recognizer` as is.

The execution is immediate not lazy, you are not registering an operation but applying it.

### Step 3: Call the `finish` method to get the result

Once all the `Recognizable` have been added to the `Recognizer`, you can call the `finish` method to get the result.

```rust
# extern crate elyze;
impl<'a, 'b, T, R: Recognizable<'a, T, R>> Recognizer<'a, 'b, T, R> {
    pub fn finish(self) -> Option<R>;
}
```

This step can't fail. It returns the internal state of the `Recognizer`.

```rust
# extern crate elyze;
fn main() -> ParseResult<()> {
    let data = b"+";
    let mut scanner = Scanner::new(data);
    // Initialize the recognizer
    let recognizer = Recognizer::new(&mut scanner);
    // Try to apply the recognizer on the operator add, if it fails, return an error
    let recognizer_add = recognizer.try_or(Operator::Add)?;
    // Try to apply the recognizer on the operator sub, if it fails, return an error
    let recognizer_add_and_sub = recognizer_add.try_or(Operator::Sub)?;
    // Finish the recognizer
    let result = recognizer_add_and_sub.finish();
    dbg!(result); // Some(Operator::Add)
}
```

## Recognizer to Visitor

If you want to reuse the `Recognizer` object, you can convert it to a `Visitor` object.

```rust
# extern crate elyze;
#[derive(Debug)]
// Define a structure to implement the `Visitor` trait
struct OperatorData(Operator);

impl<'a> Visitor<'a, u8> for OperatorData {
  fn accept(scanner: &mut Scanner<'a, u8>) -> ParseResult<Self> {
    // Build and apply the recognizer
    let operator = Recognizer::new(scanner)
            .try_or(Operator::Add)?
            .try_or(Operator::Sub)?
            .finish()
            // If the recognizer fails, return an error
            .ok_or(ParseError::UnexpectedToken)?;

    Ok(OperatorData(operator))
  }
}
```

The behavior is now reusable.

```rust
# extern crate elyze;
fn main() -> ParseResult<()> {
    let data = b"+";
    let mut scanner = Scanner::new(data);
    // Initialize the recognizer
    let result = OperatorData::accept(&mut scanner)?.0;
    dbg!(result); // Operator::Add

    let data = b"-";
    let mut scanner = Scanner::new(data);
    // Initialize the recognizer
    let result = OperatorData::accept(&mut scanner)?.0;
    dbg!(result); // Operator::Sub

    let data = b"x";
    let mut scanner = Scanner::new(data);
    // Initialize the recognizer
    let result = OperatorData::accept(&mut scanner);
    dbg!(result); // Err(UnexpectedToken)

    Ok(())
}
```

## Recommendations

Because that the first matching is returned, you should add the most specific patterns first.

Example:

If you match "hell" and "hello". You should add "hello" first. Otherwise, the recognizer will always return "hell".