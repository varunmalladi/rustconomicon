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

### 

This works fine if every 
token is represented by a single character. For example, if our input was `";,!"`, 
then we would first read `;` and know it's the token `Semicolon`, then read `,` and 
know it's `Comma`, then read `!` and know it's `Exclamation`. 

The issue is that not all tokens are just one character. For example, suppose we 
had the input `"    cat"`. We don't want to read each individual space as a 
`Whitespace`, just indicate that there is a whitespace with a single `Whitespace`
token. So read the first character, see it is a `Whitespace`, and then want to 
keep advancing until all that is left is `"cat"`. But this would mean stopping 
right before the letter `c`, which we could only do if we could "look ahead" one 
character.

In other words, we want to ability to *peek*, i.e. look ahead. In general, the 
point of saying all this is that we want more functionality than a simply iterator 
over the characters can provide. For instance, we want the ability to look ahead.  


 Suppose we represented words with a token called `Word`
(which is not accurate, just used here as an example). Then we might expect our 
tokens to be `Word`, `Whitespace`, `Word`. Now, we could read through 
character-by-character and say that a letter is the beginning of a word. Okay, but 
how do we know when the word ends? We would know, in this approach, when we read 
the space. So at that point our theoretical function would give us a token 
`Word`. Then it would start looking at the next character, which is now `c`. We
skipped over the 
