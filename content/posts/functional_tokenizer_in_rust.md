+++ 
draft = false
date = 2023-05-22T11:36:08-06:00
title = "Functional Tokenizer in Rust"
description = ""
slug = ""
authors = ["Taylor Allred"]
tags = ["functional programming", "rust"]
categories = []
externalLink = ""
series = []
+++

Whenever people talk about functional programming in Rust, they usually talk about iterators and higher order functions. This is great and all, but I want to see how it would go to make an application with as little mutation as possible. 

For my stack-based language, I decided it was time for a re-write to add new features. To start things off, I made a new tokenizer. I could have gone with a tokenizer based on the one in [Crafting Interpreters](https://craftinginterpreters.com/scanning.html), but instead I decided to make one inspired by one I saw made in Elixir. The Elixir language makes elegant use of tail-call recursion and I wanted to see to what degree I could imitate that in Rust. 

I decided I would utilize the [`tailcall`](https://docs.rs/tailcall/latest/tailcall/) crate for the recursion, and [`im`](https://docs.rs/im/latest/im/) for storing the tokens in an immutable data structure. 

```rust
use im_rc::vector;
use im_rc::vector::*;
use tailcall::tailcall;
```

The tokens are represented as a struct with a `TokenType` and a `lexeme` string. 

```rust
#[derive(Debug, Clone)]
pub enum TokenType {
    Number,
    Identifier,
    LeftBracket,
    RightBracket,
    Dot,
    Comma,
    LeftArrow,
}

#[derive(Debug, Clone)]
pub struct Token {
    token_type: TokenType,
    lexeme: String,
}

impl Token {
    fn new(token_type: TokenType, lexeme: String) -> Self {
        Self {
            token_type,
            lexeme,
        }
    }
}
```

The main `tokenize` function takes a program as a string and then returns an immutable vector as output. The way it works is by having a main inner function to act a a helper function to get the recursion started. This `tokenize_inner` function gets called by `tokenize` immediately with the initial state (the empty vector). It then dispatches on the characters it comes across one-by-one to decide which token to add to the accumulating list of tokens and/or which recursive call to make. 

```rust
pub fn tokenize(program: String) -> Vector<Token> {
    #[tailcall]
    fn tokenize_inner(mut p: Peekable<Chars>, tokens: Vector<Token>) -> Vector<Token> {
        match p.next() {
            None => tokens,
            Some(curr) => match curr {
                '[' => tokenize_inner(
                    p,
                    tokens.conj(Token::new(TokenType::LeftBracket, String::from(curr))),
                ),
                ']' => tokenize_inner(
                    p,
                    tokens.conj(Token::new(TokenType::RightBracket, String::from(curr))),
                ),
                '.' => tokenize_inner(
                    p,
                    tokens.conj(Token::new(TokenType::Dot, String::from(curr))),
                ),
                ',' => tokenize_inner(
                    p,
                    tokens.conj(Token::new(TokenType::Comma, String::from(curr))),
                ),
                _ => tokenize_inner(p, tokens),
            },
        }
    }
		...
		tokenize_inner(program.chars().peekable(), vector![])
}
```

Here we’re using a peekable `Chars` iterator to get the next char. For single characters, we simply make a tail-recursive call to `tokenize_inner` with the `tokens` vector with a new token added to it. 

### Tangent: Immutable `Collection`s

The `im` crate’s data structures use the mutating idioms of standard Rust collections, so I decided to make my own library for adding elements to the vector in a Clojure-inspired functional way. This library basically extends the API of the `im` crate using a `Collection` trait. A `Collection` is a data structure that implements `conj` for adding elements, `disj` for removing elements, and a couple others that I’m not using yet. `conj` must take the collection by transferring ownership mutably, a value, and then returns the updated collection. 

```rust
use im_rc::vector::*;

pub trait Collection<T: Clone> {
    fn conj(self, value: T) -> Self;
}

impl<T: Clone> Collection<T> for Vector<T> {
    fn conj(mut self, value: T) -> Self {
        self.push_back(value);
        self
    }
}

impl Collection<char> for String {
    fn conj(mut self, value: char) -> Self {
        self.push(value);
        self
    } 
}
```

This trick of taking ownership mutably and then returning ownership to the caller is really useful for wrapping a “mutable” data structure (especially one that uses `&mut self` to perform updating functions) with what appears to be a purely functional API. To the caller, a value goes in and a value comes out but under the hood the value has been mutated. Rust makes this safe by invalidating the old value after the transfer of ownership. If we ever want to hold on to the old value, we can always use `clone` (which is very cheap for `im` ’s collection types). 

## Back to the tokenizer

The recursive tokenizer works great for single-character tokens, but what about ones with multiple characters? This turns out to be just as simple for those because the `tailcall` crate supports mutual recursion. For any multi-character token that we detect in `tokenize_inner` we can call another helper function to consume the token recursively and then call `tokenize_inner` again with the new token added to the accumulator once it’s done. 

### Tokenize identifier and number

If we see an alphabetic character, that means we should tokenize an identifier. At this point, we don’t know how big it’s going to be so we pass off control to `tokenize_identifier` and it will take care of it. `tokenize_identifier` peeks at the next value to see if it should consume it or be done. If it sees another alphabetic character, it can keep going so it adds the character to a growing string called `accumlated`. Once it sees a character it can’t use in an identifier, it calls `tokenize_inner` with the new identifiers token added to the `tokens` vector. 

```rust
#[tailcall]
fn tokenize_identifier(
    mut p: Peekable<Chars>,
    tokens: Vector<Token>,
    accumulated: String,
) -> Vector<Token> {
    match p.peek() {
        None => tokens.conj(Token::new(TokenType::Identifier, accumulated)),
        Some(nxt) => match nxt {
            c if c.is_alphabetic() => {
                let accumulated = accumulated.conj(p.next().unwrap());
                tokenize_identifier(p, tokens, accumulated)
            }
            _ => tokenize_inner(
                p,
                tokens.conj(Token::new(TokenType::Identifier, accumulated)),
            ),
        },
    }
}
```

It’s the same idea with `tokenize_number`, except this time we can have dots for floating point numbers. If a dot appears, we can consume it once, but if another one appears, we say that that is the end of the number token. 

```rust
#[tailcall]
fn tokenize_number(
    mut p: Peekable<Chars>,
    tokens: Vector<Token>,
    accumulated: String,
    dot_appeard: bool,
) -> Vector<Token> {
    match p.peek() {
        None => tokens.conj(Token::new(TokenType::Number, accumulated)),
        Some(nxt) => match nxt {
            c if c.is_ascii_digit() => {
                let accumulated = accumulated.conj(p.next().unwrap());
                tokenize_number(p, tokens, accumulated, dot_appeard)
            }
            '.' if !dot_appeard => {
                let accumulated = accumulated.conj(p.next().unwrap());
                tokenize_number(p, tokens, accumulated, true)
            }
            _ => tokenize_inner(p, tokens.conj(Token::new(TokenType::Number, accumulated))),
        },
    }
}
```

And then we add calls to `tokenize_identifier` and `tokenize_number` to make a working simple tokenizer. 

### Altogether Now

```rust
pub fn tokenize(program: String) -> Vector<Token> {
    #[tailcall]
    fn tokenize_inner(mut p: Peekable<Chars>, tokens: Vector<Token>) -> Vector<Token> {
        match p.next() {
            None => tokens,
            Some(curr) => match curr {
                '[' => tokenize_inner(
                    p,
                    tokens.conj(Token::new(TokenType::LeftBracket, String::from(curr))),
                ),
                ']' => tokenize_inner(
                    p,
                    tokens.conj(Token::new(TokenType::RightBracket, String::from(curr))),
                ),
                '.' => tokenize_inner(
                    p,
                    tokens.conj(Token::new(TokenType::Dot, String::from(curr))),
                ),
                ',' => tokenize_inner(
                    p,
                    tokens.conj(Token::new(TokenType::Comma, String::from(curr))),
                ),
                c if c.is_alphabetic() => tokenize_identifier(p, tokens, String::from(curr)),
                c if c.is_ascii_digit() => tokenize_number(p, tokens, String::from(curr), false),
                _ => tokenize_inner(p, tokens),
            },
        }
    }

    #[tailcall]
    fn tokenize_identifier(
        mut p: Peekable<Chars>,
        tokens: Vector<Token>,
        accumulated: String,
    ) -> Vector<Token> {
        match p.peek() {
            None => tokens.conj(Token::new(TokenType::Identifier, accumulated)),
            Some(nxt) => match nxt {
                c if c.is_alphabetic() => {
                    let accumulated = accumulated.conj(p.next().unwrap());
                    tokenize_identifier(p, tokens, accumulated)
                }
                _ => tokenize_inner(
                    p,
                    tokens.conj(Token::new(TokenType::Identifier, accumulated)),
                ),
            },
        }
    }

    #[tailcall]
    fn tokenize_number(
        mut p: Peekable<Chars>,
        tokens: Vector<Token>,
        accumulated: String,
        dot_appeard: bool,
    ) -> Vector<Token> {
        match p.peek() {
            None => tokens.conj(Token::new(TokenType::Number, accumulated)),
            Some(nxt) => match nxt {
                c if c.is_ascii_digit() => {
                    let accumulated = accumulated.conj(p.next().unwrap());
                    tokenize_number(p, tokens, accumulated, dot_appeard)
                }
                '.' if !dot_appeard => {
                    let accumulated = accumulated.conj(p.next().unwrap());
                    tokenize_number(p, tokens, accumulated, true)
                }
                _ => tokenize_inner(p, tokens.conj(Token::new(TokenType::Number, accumulated))),
            },
        }
    }

    tokenize_inner(program.chars().peekable(), vector![])
}
```

## Switching from Iterator to Linked List

This Rust code turned out surprisingly elegant and purely functional! Right? 

Unfortunately we had to make concessions for the use of `Iterator`. Iterators in Rust are stateful, and therefore have to be mutable because every call to `next()` changes the its state. This is why we had to keep doing `let accumulated = accumulated.conj(p.next().unwrap());` instead of just doing one recursive call with the new vector inline. We have to call `next()` before we pass `p` to the next call, otherwise we will get repeat characters. This is an artifact of using `peek`, which we had to use because `tokenize_identifier` and `tokenize_number` needed to *decide* whether or not to consume the character. If they couldn’t, they could always pass `p` off to `tokenize_inner` unchanged. 

There is clearly a better way to do this. That way is to do exactly what Elixir and Clojure do and use a sequence or linked list structure. Linked lists and recursion are best friends. They were made for each other. 

So let’s clean this implementation up by turning our iterator of characters into a linked list of characters and then we will have a purely functional data structure that has no state. 

 

### The `List` struct

To make a linked list, we will use an enum that has an `Empty` case and a `Cons` case, plus some methods to create lists and get the head or the tail from them. 

```rust
#[derive(Clone)]
pub enum List<T: Clone> {
    Empty,
    Cons(T, Box<List<T>>),
}

impl<T: Clone> List<T> {
    pub fn empty() -> Self {
        List::Empty
    }

    pub fn cons(self, value: T) -> Self {
        List::Cons(value, Box::new(self))
    }

    pub fn head(self) -> Option<T> {
        match self {
            List::Empty => None,
            List::Cons(value, _) => Some(value),
        }
    }

    pub fn tail(self) -> Self {
        match self {
            List::Empty => List::Empty,
            List::Cons(_, rest) => *rest,
        }
    }
}
```

(We will use a `Box` because it’s required for recursive data structures in Rust.) 

Then we can make a trait that can make any iterator into a `List`. We’ll call it `Sequence` after the protocol in Clojure. Then we’ll have the `Chars` iterator implement it. 

```rust
pub trait Sequence: Iterator + DoubleEndedIterator {
    fn seq(self) -> List<Self::Item>
    where
        Self::Item: Clone,
        Self: Sized,
    {
        self.rev().fold(List::Empty, |lst, value| lst.cons(value))
    }
}

impl<'a> Sequence for Chars<'a> {}
```

### Refactoring to use List

Now in our tokenizer, we can replace usage of `Peekable<Chars>` with `List<char>` and change the methods we are calling. We will replace `next` with `head`, and use `tail` to pass the remaining elements in recursive calls. (We will need to make use of `clone` to use `head` more than once in a function like we mentioned before, but this is a pretty trivial cost.)

```rust
pub fn tokenize(program: String) -> Vector<Token> {
    #[tailcall]
    fn tokenize_inner(p: List<char>, tokens: Vector<Token>) -> Vector<Token> {
        match p.clone().head() {
            None => tokens,
            Some(curr) => match curr {
                '[' => tokenize_inner(
                    p.tail(),
                    tokens.conj(Token::new(TokenType::LeftBracket, String::from(curr))),
                ),
                ']' => tokenize_inner(
                    p.tail(),
                    tokens.conj(Token::new(TokenType::RightBracket, String::from(curr))),
                ),
                '.' => tokenize_inner(
                    p.tail(),
                    tokens.conj(Token::new(TokenType::Dot, String::from(curr))),
                ),
                ',' => tokenize_inner(
                    p.tail(),
                    tokens.conj(Token::new(TokenType::Comma, String::from(curr))),
                ),
                '<' => match p.clone().tail().head() {
                    None => tokens,
                    Some('-') => tokenize_inner(
                        p.tail(),
                        tokens.conj(Token::new(TokenType::LeftArrow, String::from("<-"))),
                    ),
                    Some(_) => tokenize_inner(p.tail(), tokens),
                },
                c if c.is_alphabetic() => tokenize_identifier(p.tail(), tokens, String::from(curr)),
                c if c.is_ascii_digit() => {
                    tokenize_number(p.tail(), tokens, String::from(curr), false)
                }
                _ => tokenize_inner(p.tail(), tokens),
            },
        }
    }

    #[tailcall]
    fn tokenize_identifier(
        p: List<char>,
        tokens: Vector<Token>,
        accumulated: String,
    ) -> Vector<Token> {
        match p.clone().head() {
            None => tokens.conj(Token::new(TokenType::Identifier, accumulated)),
            Some(nxt) => match nxt {
                c if c.is_alphabetic() => {
                    tokenize_identifier(p.tail(), tokens, accumulated.conj(nxt))
                }
                _ => tokenize_inner(
                    p,
                    tokens.conj(Token::new(TokenType::Identifier, accumulated)),
                ),
            },
        }
    }

    #[tailcall]
    fn tokenize_number(
        p: List<char>,
        tokens: Vector<Token>,
        accumulated: String,
        dot_appeard: bool,
    ) -> Vector<Token> {
        match p.clone().head() {
            None => tokens.conj(Token::new(TokenType::Number, accumulated)),
            Some(nxt) => match nxt {
                c if c.is_ascii_digit() => {
                    tokenize_number(p.tail(), tokens, accumulated.conj(nxt), dot_appeard)
                }
                '.' if !dot_appeard => {
                    tokenize_number(p.tail(), tokens, accumulated.conj(nxt), true)
                }
                _ => tokenize_inner(p, tokens.conj(Token::new(TokenType::Number, accumulated))),
            },
        }
    }

    tokenize_inner(program.chars().seq(), vector![])
}
```

So how did `List` improve this? 

1. There is now no mutation in this implementation.
2. It is very explicit which *values* are being passed where (because of the lack of state).

What did it cost? 

- Immutable `Vector` has:
    - O(1) `clone`.
    - O(1) amortised, and O(log n) in the worst case for `push`/`pop`.
- Using linked lists didn’t change our Big-O complexity, but we did end up allocating more heap memory and linked lists are not as cache friendly as arrays. Perhaps there is a way to imitate the behavior of linked lists while maintaining the underlying structure.

## The takeaways

Functional programming is very much compatible in Rust if you utilize a couple of tricks.

- **Utilize the ownership/borrowing system to your advantage**:
    - Wrap mutating functions with functions that take ownership of the value, perform the mutation, and return the value.
    - Use transfer of ownership as much as possible rather than passing references. This seems counterproductive at first but works very nicely with a purely functional style.
    - Make use of clone when necessary. Ease the cost of cloning by using immutable data structures.
- **Use `tailcall` for recursive algorithms.** This gave us the elegance of recursion (as well as mutual recursion) without having to worry about the space cost on the stack.
- **Use linked lists with recursion** for a stateless sequence of values. Purely functional recursive data structures, and recursive algorithms go hand-in-hand.