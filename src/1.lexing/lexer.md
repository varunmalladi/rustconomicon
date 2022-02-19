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
