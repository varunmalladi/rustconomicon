# Lexing

Our goal here is to take some input and return a stream of tokens somehow 
representing this input. Specifically, we'll be working towards the following
function:
```
pub fn tokenize(input: &'_ str) -> impl Iterator<Item = Token> + '_ {...}
```
As this suggests,
- Our input as a `&str`. This is so we can have some modularity in our code, 
making this code independent of how we actually end up receiving input from the 
user.
- Our output is an iterator over tokens, which themselves are represented by a 
`struct Token`. 

There are also some anonymous lifetime annotations, which we'll find are 
necessary in our subsequent discussions.

## Outline
We give here a basic sketch of our approach.

### Cursor
Given our input, which we are assuming is coming in as a `&str`, how do we 
get a stream of tokens? The natural idea is to walk through this input 
character-by-character, and determine tokens this way. An iterator over characters
turns out to not be robust enough: we want to do other things, like look ahead 
without consuming the item, advance until a certain parameter condition is met, 
and remember how long the token we just consumed was. 

To provide this functionality, we define the `Cursor` struct. It is like an 
iterator over `char`s, but with more features. 

### Lexer
This is where we'll define `tokenize()` and everything needed to make it make 
sense. This includes defining tokens.
