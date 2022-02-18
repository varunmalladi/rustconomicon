# Cursor

Given our input, which we are assuming is coming in as a `&str`, how do we 
get a stream of tokens? The natural idea is to walk through this input 
character-by-character, and determine tokens this way. An iterator over characters
turns out to not be robust enough: we want to do other things, like look ahead 
without consuming the item, advance until a certain parameter condition is met, 
and remember how long the token we just consumed was. 

To provide this functionality, we define the `Cursor` struct. It is like an 
iterator over `char`s, but with more features. 

The `Cursor` doesn't remember what it has consumed— that is up to the caller to 
handle the return value whenever calling one of `Cursor`'s methods. 

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

## Instantiating
Our definition above suggests how we might instantiate one of these things given 
our input, which we are assuming is a `&str`:
```
pub fn new(input: &'a str) -> Cursor<'a> {
    return Cursor {
        initial_len: input.len(),
        chars: input.chars(),
        prev: '\0',
    };
}
```

## Interpreting length
We provide a method to know when there is nothing left to read:
```
pub fn is_empty(&self) -> bool {
    return self.chars.as_str().len() == 0;
}
```

We mentioned above that the struct field `initial_len` would be used to determine 
the length of a token. The idea is that we can determine, using methods from
`Chars`, the length remaining in the iterator. By subtracting this from `initial_len`
we can find out how much we have read. If we reset `initial_len` every time we 
read a token, we can determine the length of a token.

To get the length:
```
pub fn len_consumed(&self) -> usize {
    return self.initial_len - self.chars.as_str().len();
}
```

To reset `initial_len`:
```
pub fn reset_len_consumed(&mut self) {
    self.initial_len = self.chars.as_str().len();
}
```

## Advancing through
We first provide a basic function to advance just one character, which is 
essentially a wrapper around `Char.next()` that updates the other fields appropriately.
Maybe this function will be called, but it will be particularly useful as a 
building block.
```
pub fn adv(&mut self) -> Option<char> {
    let consumed_char = self.chars.next()?;

    self.prev = consumed_char;

    return Some(consumed_char);
}
```
Notice the `?` propogates a return value of `None` from `Chars.next()` to the caller of 
`adv()`, so that a value of `None` is returned from `adv()` when there are no 
more characters left in the iterator.

If we're thinking about using this to consume tokens, then you can imagine we will 
be advancing until we reach the end of a token. We don't want to have to consume a 
character belonging to the next token just to know when the current one has ended.
We want a function that will allow us to "look ahead" without consuming the next 
character.
```
pub fn peek(&self) -> char {
    let c: char;
    match self.chars.clone().next() {
        Some(ch) => {
            c = ch;
        }
        None => {
            c = EOF_CHAR;
        }
    }
    return c;
}
```
Note that this is not very efficient, as it involves making a copy of the iterator 
and then consuming the next character in the copy. "Peeking" functions are generally 
not efficient, as iterator data structures do not prioritize the efficiency of this 
functionality.
- **Why do we return end-of-file here and none above?**

We are almost ready to write the "advance until" function. The last business to 
deal with is figuring out how to let the caller of this function specify the 
"stop" condition. One option would be to let the user pass in a `char` as a 
parameter, so that we stop advancing right before that `char` is consumed. To
leave room for more complex conditions, however, we let the user pass in a 
function that takes in a `char` and returns a `bool`. The `bool` will tell us 
to stop advancing if it is false.
```
pub fn adv_until(&mut self, mut condition: impl FnMut(char) -> bool) {
    while condition(self.peek()) && !self.is_empty() {
        self.adv();
    }
}
```

As we've just discussed, `condition` is the function telling us when to stop advancing.
- **Why mutable function instead of regular function?**

We'll also stop advancing if there is nothing left to read.
