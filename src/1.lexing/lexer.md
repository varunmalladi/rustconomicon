# Lexer

## Tokens
Recall that at this point, the stream of tokens we are generating does 
not know its specific content. For example, a "string" token doesn't know
the actual string it was constructed from. 

One reason we might do this is that it is very wasteful to store a copy 
of the input in the tokens when it is already there in the file. Instead, 
we can just store the length of the token. Again, we receive a stream of
tokens, in order, so knowing the length of each token will allow us to 
walk to the input `&str` knowing which token is where.

Since we want each token to know its length (e.g. with `Cursor.len_consumed()`),
it is cleaner to have the following abstraction:
```rust
#[derive(Debug, PartialEq)]
pub struct Token {
    pub kind: TokenKind,
    pub len: usize,
}

impl Token {
    pub fn new(kind: TokenKind, len: usize) -> Token {
        return Token { kind, len };
    }
}
```

The actual "tokens" as we've been using them are going to be `TokenKind`s. At 
this point we put some essentials in (enough to tokenize a hello world program), 
and we can always add to this later on:
```rust
#[derive(Debug, PartialEq)]
pub enum TokenKind {
    /* Multi-char tokens */
    Whitespace,
    Identifier,                    // Includes keywords, ...
    Literal { kind: LiteralKind }, // Includes strings, ...

    /* Single-char tokens */
    Semi,       // ;
    OpenParen,  // (
    CloseParen, // )
    OpenBrace,  // {
    CloseBrace, // }
    Exclam,     // !

    /* Anythng else */
    Unknown,
}
```

We are abstracting away `Literal` because at a high level we want to view them as 
the same token (e.g. raw strings, double-quoted strings, etc.) but the specific 
way our compiler deals with them may differ. We could just put all the `LiteralKind`s
inside `TokenKind`, but in the spirit of grouping related things together we make it
its own `enum`:
```rust
#[derive(Debug, PartialEq)]
pub enum LiteralKind {
    Str { terminated: bool }, // e.g. "Hello, world!"
}
``` 

The `terminated` field basically tells us if there is an ending quote. A string in 
some code could last longer than a line, and if we just tokenize the code line-by-line,
we would have trouble dealing with this case.

## Cursor methods

Now we will add functionality to our `Cursor` that will enable us to obtain 
tokens from it. Right now, there are four kinds of tokens we need to deal 
with. The first kind are the single-character tokens, such as `Semi` and 
`CloseParen`. These are pretty easy to determine— just consume the character 
and see what it is. The other three are more difficult: `Identifier`, 
`Whitespace` and `Literal`. We will handle the more difficult cases in seperate
functions, which a larger, public function called `eat_token()` will call
as necessary.

First, we need a way to consume whitespace. If we are walking down our Cursor, 
just one whitespace is enough to indicate a `Whitespace` token, so that we may 
just keep advancing until we get a non-whitespace character:
```rust
impl Cursor<'_> {
    fn eat_whitespace(&mut self) -> TokenKind {
        self.adv_until(is_whitespace);
        return Whitespace;
    }
}

pub fn is_whitespace(c: char) -> bool {
    matches! {
        c,
        '\t'
        | '\n'
        | ' '
    }
}
```
(`eat_whitespace()` will be called by another `Cursor` method which will be 
public, so we don't need to make it public here.) Recall that `adv_until()`
took a function as an argument, and this is an instance of how we could use 
that.

Next, let's think about how to consume a `Literal`, which for us, at this point,
just means a string surrounded on both sides by a double quote. Once we consume 
a character and see that it is a double quote, we can determine that this marks 
the beginning of a `Literal` of kind `Str` and consume the remaining characters 
in the token as follows:
```rust
impl Cursor<'_> {
    fn eat_double_quote_str(&mut self) -> bool {
        loop {
            match self.adv() {
                // Reached end of input
                None => break,
                // Found terminating character
                Some('"') => {
                    return true;
                }
                // Nothing special, keep advancing
                Some(_) => (),
            }
        }

        // Couldn't find a terminating character
        return false;
    }
}
```
The return value here is whether the `LiteralKind` of `Str` is terminated.

Finally, we have identifiers. These are discussed in the Rust reference 
[here](https://doc.rust-lang.org/reference/identifiers.html). What is layed
out there is that identiers either start with an "XID start" follows by 
"XID continue"s or starts with an "_" followed by "XID continue"s. We handle
these cases separately:
```rust
pub fn is_id_start(c: char) -> bool {
    if c == '_' || unicode_xid::UnicodeXID::is_xid_start(c) {
        return true;
    }
    return false;
}

pub fn is_id_continue(c: char) -> bool {
    return unicode_xid::UnicodeXID::is_xid_continue(c);
}
```

Once we detect an id start, we can consume the rest of that token with:
```rust
impl Cursor<'_> {
    fn eat_id_continue(&mut self) -> TokenKind {
        self.adv_until(is_id_continue);
        return Identifier;
    }
}
```

Finally, piecing all these parts together, we can define:
```rust
impl Cursor<'_> {
    pub fn eat_token(&mut self) -> Token {
        let first_char = self.adv().unwrap();
        // Starting from the first character, try to determine what TokenKind
        // we have. Of course, one kind of character could indicate one of many
        // TokenKinds, and we deal with these in the most direct way.
        let token_kind = match first_char {
            /* Multi-character tokens */
            c if is_whitespace(c) => self.eat_whitespace(), // Whitespace
            c if is_id_start(c) => self.eat_id_continue(),  // Identifier

            /* Literals */
            '"' => {
                let is_terminated = self.eat_double_quote_str();
                let kind = Str {
                    terminated: is_terminated,
                };

                Literal { kind }
            }

            /* Single-character tokens (reserved characters) */
            ';' => Semi,
            '(' => OpenParen,
            ')' => CloseParen,
            '{' => OpenBrace,
            '}' => CloseBrace,
            '!' => Exclam,

            /* Anything else */
            _ => Unknown,
        };

        return Token::new(token_kind, self.len_consumed());
    }
}
```

## Tokenize
Given all of this, we are in a place to define `tokenize()`, which is 
what we were after in this lexing portion:
```rust
pub fn tokenize(input: &'_ str) -> impl Iterator<Item = Token> + '_ {
    let mut cursor = Cursor::new(input);
    std::iter::from_fn(move || {
        if cursor.is_empty() {
            None
        } else {
            cursor.reset_len_consumed();
            Some(cursor.eat_token())
        }
    })
}
```

We need the anonymous lifetimes `'_` because otherwise the iterator is given a 
`static` lifetime, which doens't make sense as its contents only live as long as 
`input`. 

This function first creates a cursor. The second line creates an iterator in the 
following way: each time an item is taken from the iterator, e.g. with `.next()`, 
the object defined in the second line calls the function it takes as an argument. 
This function in particular consumes the next token. The `move` keyword indicates
that this function should take ownership of the objects referenced within it, in 
this case it will take ownership of the `Cursor`, which makes sense since the caller 
of the `tokenize()` function will only every work with the iterator it returns, not 
directly will the caller know about the `Cursor` from which it was built.
