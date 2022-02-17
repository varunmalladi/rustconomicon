# Cursor

Given our input, which we are assuming is coming in as a `&str`, how do we 
get a stream of tokens? The natural idea is to walk through this input 
character-by-character, and determine tokens this way. An iterator over characters
turns out to not be robust enough: we want to do other things, like look ahead 
without consuming the item, advance until a certain parameter condition is met, 
and remember how long the token we just consumed was. 

To provide this functionality, we define the `Cursor` struct. It is like an 
iterator over `char`s, but with more features. 

## Struct definition
Here is our working definition. We'll try to explain why we have defined it this 
way, but it will become clearer in hindsight as we continue with our program.
```
use std::str::Chars; // iterator over `char`s

pub struct Cursor<'a> {
    initial_len: usize,
    chars: Chars<'a>,
    prev: char,
}
```
- `initial_len` is going to help determine the length of the token we have just 
consumed. 
- `chars` is the iterator our struct builds on and provides extra functionality 
for.
- `prev` is the character from the iterator we have just consumed. Certain tokens
depend on the character that immediately preceded it, and this will help in that 
case.

We specify the lifetime `'a` because `Chars` needs it. And the reason `Chars` needs 
a lifetime is because it would like to know when/if it can drop its items. The 
`Cursor` will own the `Chars` and its items, so we can tell Rust that its safe to 
drop whenever `Cursor` is being dropped. 

##
