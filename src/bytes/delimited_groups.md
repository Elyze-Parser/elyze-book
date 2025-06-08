# Delimited Groups

There is a special case of peeking when you want to get data embedded in a delimited group.

For example, you want to get the contents of a parentheses-delimited expression.

```
( 1 + 2 )
^
you're here
```

You want the inner data

```
 1 + 2 
```

You have to recognize the opening parentheses, then search for the closing parentheses. To do this, the peeking is the
best solution.

But you also have to deal with balanced expressions: sometimes you will have nested parentheses-delimited expressions.

```
( ( 1 + 2 ) + 3 )
^
you're here
```

If you stop at the first closing parenthesis, you will get `( ( 1 + 2 )`. So the inner expression will be: 

```( 1 + 2 ```.

That's because your group is unbalanced. There are more opening parentheses than closing parentheses.

To make it work, you've to keep track of the number of opening and closing parentheses.

A number is the perfect solution. Initially at 0, you increase it when you find an opening parenthesis, and decrease it
when you find a closing parenthesis.

The algorithm stops when the number is 0. Because we always match the opening parentheses as first bytes. The balancing
starts at 1.

```
( ( 1 + 2 ) + 3 )
^
b: 1
```

The next opening parentheses increments by 1 the balancing. Because balancing equals 2, the algorithm continues.

```
( ( 1 + 2 ) + 3 )
  ^
  b: 2
```

The next recognized element is a closing parentheses. The balancing is decreased by 1. The algorithm continues.

```
( ( 1 + 2 ) + 3 )
          ^
          b: 1
```

The next recognized element is also a closing parentheses. 
The balancing is decreased by 1. The algorithm stops because balancing is now 0.

```
( ( 1 + 2 ) + 3 )
                ^
                b: 0
```

The real slice of data is:

```
 ( 1 + 2 ) + 3 
```

## GroupKind

Elyze defines a `GroupKind` enumeration that implements the `Peekable` trait. 

This one allows peeking into a delimited group.

### Parentheses Group

One of the `GroupKind` is the `ParenthesesGroup`. Like explained in the introduction.

```rust
# extern crate elyze;
use elyze::bytes::components::groups::GroupKind;
use elyze::peek::peek;
use elyze::scanner::Scanner;

fn main() -> ParseResult<()> {
    let data = b"( 5 + 3 - ( 10 * 8 ) ) + 54";
    let mut tokenizer = Scanner::new(data);
    let result = peek(GroupKind::Parenthesis, &mut tokenizer)?;

    if let Some(peeked) = result {
        assert_eq!(peeked.peeked_slice(), b" 5 + 3 - ( 10 * 8 ) ");
    }
    Ok(())
}
```

It supports the character escaping of parentheses.

If your data are like this:

```
( ( 1 + 2 ) \) + 3 )
```

The escaped closing parenthesis `\)` will be ignored. And so the real the data will be correctly parsed. And includes
escaped characters.

```
 ( 1 + 2 ) \) + 3 
```

The escaping is done by the `\` character.

```rust
# extern crate elyze;
use elyze::bytes::components::groups::GroupKind;
use elyze::peek::peek;
use elyze::scanner::Scanner;

fn main() -> ParseResult<()> {
    let data = b"( 5 + 3 - \\( ( 10 * 8 \\)) \\)) + 54";
    let mut tokenizer = Scanner::new(data);
    let result = peek(GroupKind::Parenthesis, &mut tokenizer)?;

    if let Some(peeked) = result {
        assert_eq!(peeked.peeked_slice(), b" 5 + 3 - \\( ( 10 * 8 \\)) \\)");
    }
    Ok(())
}
```

### Quoted Groups

In addition, Elyze also supports quoted groups.

```rust
# extern crate elyze;
use elyze::bytes::components::groups::GroupKind;
use elyze::peek::peek;
use elyze::scanner::Scanner;

fn main() -> ParseResult<()> {
    let data = b"'hello world' data";
    let mut tokenizer = Scanner::new(data);
    let result = peek(GroupKind::Quotes, &mut tokenizer)?;

    if let Some(peeked) = result {
        assert_eq!(peeked.peeked_slice(), b"hello world");
    }
    Ok(())
}
```

Because quote groups use the same symbol for opening and closing the group, you can't detect nested groups. But you can
escape identifiers if you want.

```rust
# extern crate elyze;
use elyze::bytes::components::groups::GroupKind;
use elyze::peek::peek;
use elyze::scanner::Scanner;

fn main() -> ParseResult<()> {
    let data = "'I\\'m a quoted data' - 'yes me too'";
    let mut tokenizer = Scanner::new(data.as_bytes());
    let result = peek(GroupKind::Quotes, &mut tokenizer).expect("failed to parse");

    if let Some(peeked) = result {
        assert_eq!(peeked.peeked_slice(), b"I\\'m a quoted data");
    }
    Ok(())
}
```

The same can be done with double quotes.

```rust
# extern crate elyze;
use elyze::bytes::components::groups::GroupKind;
use elyze::peek::peek;
use elyze::scanner::Scanner;

fn main() -> ParseResult<()> {
    let data = "\"I'm a quoted data\" - \"yes me too\"";
    let mut tokenizer = Scanner::new(data.as_bytes());
    let result = peek(GroupKind::DoubleQuotes, &mut tokenizer).expect("failed to parse");

    if let Some(peeked) = result {
        assert_eq!(peeked.peeked_slice(), b"I'm a quoted data");
    }
    Ok(())
}
```