---
layout: post
title: Rust's Type System is Turing Complete
published: true
---

_(N.B. The world "fuck" appears multiple times in this post. I recommend that
the reader temporarily not consider "fuck" as profanity, as it isn't used that
way here.)_

Not so long ago, a certain someone [made a challenge on
the Rust subreddit](https://www.reddit.com/r/rust/comments/5y4x9r/challenge_rusts_type_system_is_not_turing_complete/)
asserting that while everyone likes to say that Rust's type system is Turing-complete,
no one actually seems to have a proof anywhere. In response to that post, I
constructed a proof in the form of [an implementation of Smallfuck -- a known 
Turing-complete language -- entirely in the Rust type system.](https://github.com/sdleffler/tarpit-rs)
This blog post is an attempt to explain the inner workings of that Smallfuck
implementation, as while as what this means for the Rust type system.

So what *is* Turing-completeness? Turing-completeness is a property of most
programming languages which states that they can simulate a universal Turing
machine. A related notion is Turing-equivalence. Turing-equivalent languages
can simulate *and* be simulated by Turing-machines -- so if you have any two
Turing-equivalent languages, it must be possible to translate any program written
for one into a program written for the other. Most if not all Turing-complete
systems are also known to be Turing-equivalent. [(I regret not being able to find
a better citation for this.)](https://en.wikipedia.org/wiki/Turing_completeness#Formal_definitions)

There are several important things to note about Turing-completeness. We know
about the [halting problem](https://en.wikipedia.org/wiki/Halting_problem). One
consequence of this is that if you have a Turing-complete language, which can
simulate any universal Turing machine, then it must be capable of infinite loops.
This is why it's useful to know whether or not Rust's type system is Turing-complete --
what that means is, if you're able to encode a Turing-complete language into Rust's
type system, then the process of checking a Rust program to ensure that it is
well-typed must be [undecidable](https://en.wikipedia.org/wiki/Undecidable_problem).
The typechecker must be capable of infinite loops.

How do we show that the Rust type system is Turing-complete? The most
straightforward way to do so (and in fact I don't know of any other ways) is
to implement a known Turing-complete language in it. If you can implement a
Turing-complete language in another language, then you can clearly simulate
any universal Turing machine in it -- by simulating it in the embedded language.

## Smallfuck: Useless and Useful

So, what's [Smallfuck](https://esolangs.org/wiki/Smallfuck)? Smallfuck is a
minimalist programming language which is known to be Turing-complete when memory
restrictions are lifted. I chose to implement it in the Rust type system over
other Turing-complete languages because of its simplicity. While [esoteric languages](https://esolangs.org/wiki/Main_Page)
are fairly useless for actually writing programs, they're great for proving
Turing-completeness.

Smallfuck is actually pretty close to a Turing machine itself. The original
specification for Smallfuck states that it runs in a machine with a
finite amount of memory. However, if we lift that restriction and allow it to
access a theoretically infinite array of memory, then Smallfuck becomes
Turing-complete. So here, we consider a variation of Smallfuck with infinite
memory. The Smallfuck machine consists of an infinite tape of memory
consisting of "cells" containing bits along with a pointer into that array of
cells.

```
                 Pointer |
                         v
...000001000111000001000001111...
```

Smallfuck programs are strings of five instructions:

```
< | Pointer decrement
> | Pointer increment
* | Flip current bit
[ | If current bit is 0, jump to the matching ]; else, go to the next instruction
] | Jump back to the matching [ instruction
```

This gives you the ability to select cells and make loops. Here's a simple example
program:

```
>*>*>*[*<]
```

This is a dead simple and totally useless program (as most programs are in Smallfuck,
thanks to its total lack of any sort of I/O whatsoever) which simply sets three
bits in a row to `1`, and then uses a loop to put them all back to `0`, ending
with the pointer in the same location it started. A visualization:

```
Instruction pointer
|               Memory pointer
v               v
>*>*>*[*<] | ...0...
```

The first instruction moves the pointer to the right. All cells default to `0`:

```
 v               v
>*>*>*[*<] | ...00...
```

The next instruction is a "flip current bit" instruction, so we flip the bit at
the pointer from `0` to `1`.

```
 v               v
>*>*>*[*<] | ...01...
```

This occurs three times. Let's skip ahead to the start of the loop:

```
      v            v
>*>*>*[*<] | ...0111...
```

Now we're at the beginning of a loop. The `[` instruction says, "if the current
bit is zero, jump to the matching `]`; else, go to the next instruction." The bit
at the pointer is `1`, so we go to the next instruction.

```
       v           v
>*>*>*[*<] | ...0111...
```

This flips the current bit back to zero; then, we move the memory pointer back
one location.

```
         v        v
>*>*>*[*<] | ...0110...
```

Now we're at the closing `]`. This is an unconditional jump back to the start
of the loop.

```
      v           v
>*>*>*[*<] | ...0110...
```

Now we branch again. Is the current cell zero? No? Then we continue:

```
       v          v
>*>*>*[*<] | ...0110...

        v         v
>*>*>*[*<] | ...0100...

         v       v
>*>*>*[*<] | ...0100...

      v          v
>*>*>*[*<] | ...0100...
```

After looping one last time, we end up here:

```
      v         v
>*>*>*[*<] | ...0000...
```

Which is exactly where we started: all cells back to zero, and with the pointer
in its starting location.

## Runtime Smallfuck in Rust

So what would a simple implementation of this look like in Rust?
I'll start by walking through the run-time implementation of Smallfuck that I
bundled with my type-level implementation in order to verify that the type-level
and run-time implementations coincided. We'll store Smallfuck programs as an AST,
like so:

```rust
enum Program {
    Empty,
    Left(Box<Program>),
    Right(Box<Program>),
    Flip(Box<Program>),
    Loop(Box<(Program, Program)>),
}
```

This isn't quite true to the representation of Smallfuck as a string, but it's
much easier to interpret. We also need a type to represent the state of a running
Smallfuck program:

```rust
struct State {
    ptr: u16,
    bits: [u8; (std::u16::MAX as usize + 1) / 8],
}
```

Although to be Turing-complete we technically need an infinite tape, this run-time
implementation is only for checking the semantics of the type-level version. A
finite tape is a perfectly fine approximation to make, for now. By making the
length of `bits` to be `(std::u16::MAX + 1) / 8`, we ensure that we have a bit for
every `u16` address. Now for some operations which we'll use in implementing our
interpreter:

```rust
impl State {
    fn get_bit(&self, at: u16) -> bool {
        self.bits[(at / 8) as usize] & (0x1 << (at & 0x7)) != 0
    }

    fn get_current_bit(&self) -> bool {
        self.get_bit(self.ptr)
    }

    fn flip_current_bit(&mut self) {
        self.bits[(self.ptr / 8) as usize] ^= 0x1 << (self.ptr & 0x7);
    }
}
```

This is some fairly standard bit manipulation. We store 8 bits into one cell in
our `bits` array, so to find out which cell a given bit goes to, we use
truncating division by 8, which if we're trying to be exceedingly clever results
in a bit-shift to the right by three places. We're essentially using this addressing
scheme:

```
Index into the bytes of `bits` --> 1100011010011 \ 001 <-- Which bit in the byte
```

So it's clear we can do things through nice little bit-masks. Shifting right by
three places chops off the "which bit" section, leaving just the index. `0x7`
in hexadecimal is `0b111` in binary. This makes its purpose quite
clear: taking `& 0x7` clears all bits in our pointer except for the last three,
which indicate which bit in the byte. Shifting `0x1` by that amount gets us a
`u8` with only a single bit set to `1`, letting us index our byte; and finally,
`!= 0` lets us quickly check whether or not the respective bit is set. The xor
operation in `flip_current_bit` simply flips the indexed bit.

Now that we have those primitives, let's move on to implementing our interpreter.
We'll do so by recursively calling a function which pattern-matches over `Program`:

```rust
impl Program {
    fn big_step(&self, state: &mut State) {
        use self::Program::*;

        match *self {
            Empty => unimplemented!(),
            Left(ref next) => unimplemented!(),
            Right(ref next) => unimplemented!(),
            Flip(ref next) => unimplemented!(),
            Loop(ref body_and_next) => unimplemented!(),
        }
    }
}
```

An `Empty` program modifies no state, so its implementation is very simple:

```rust
Empty => {},
```

`Left` and `Right` just increment/decrement the pointer by one. Since we've
chosen to use a wrapping tape in our simple implementation, we use `wrapping_add`
and `wrapping_sub`:

```rust
Left(ref next) => {
    state.ptr = state.ptr.wrapping_sub(1);
    next.big_step(state);
},
Right(ref next) => {
    state.ptr = state.ptr.wrapping_add(1);
    next.big_step(state);
},
```

`Flip` is very simple, since we wrote our convenient `flip_current_bit` function:

```rust
Flip(ref next) => {
    state.flip_current_bit();
    next.big_step(state);
},
```

And last but not least, `Loop`. We check the current bit. If it's `1`, then we
execute the body of the loop, and then execute the loop instruction again with
the updated state. If it's `0`, we move on to the next instruction:

```rust
Loop(ref body_and_next) => {
    let &(ref body, ref next) = body_and_next.as_ref();
    if state.get_current_bit() {
        body.big_step(state);
        self.big_step(state);
    } else {
        next.big_step(state);
    }
},
```

We have to do the `.as_ref()` dance with `body_and_next` because we boxed the tuple.
Our finished `.big_step()` function looks like this:

```rust
impl Program {
    fn big_step(&self, state: &mut State) {
        use self::Program::*;
    
        match *self {
            Empty => {},
            Left(ref next) => {
                state.ptr = state.ptr.wrapping_sub(1);
                next.big_step(state);
            },
            Right(ref next) => {
                state.ptr = state.ptr.wrapping_add(1);
                next.big_step(state);
            },
            Flip(ref next) => {
                state.flip_current_bit();
                next.big_step(state);
            },
            Loop(ref body_and_next) => {
                let &(ref body, ref next) = body_and_next.as_ref();
                if state.get_current_bit() {
                    body.big_step(state);
                    self.big_step(state);
                } else {
                    next.big_step(state);
                }
            },
        }
    }
}
```

We won't be implementing any sort of debug printing for our `State` since we
won't need it. When we use our run-time interpreter to check against our type-level
interpreter, we'll do so by iterating over the bits in the output of the type-level
interpreter and checking to ensure that the same bits are set in the run-time output.
Now, we're free to start on the fun stuff!

## Type-Level Smallfuck in Rust

Rust has a feature called *traits*. Traits are a way of doing compile-time static
dispatch, and can also be used to do runtime dispatch (although in practice that's
rarely useful.) I will assume here that the reader knows what traits are in Rust
and has used them a fair amount. To implement Smallfuck, we'll be relying on a
particular feature of traits known as *associated types*.

### Trait resolution and unification

In Rust, in order to call a trait method or resolve an associated type, the compiler
must go through a process called *trait resolution*. In order to resolve a trait,
the compiler has to look for an `impl` which *unifies* with the types involved.
[Unification](https://en.wikipedia.org/wiki/Unification_(computer_science)) is a
process for solving equations between types. If there is no solution, we say that
unification has failed. Here's an example:

```rust
trait Foo<B>: A {
   type Associated;
}


impl<B> Foo<B> for u16 {
    type Associated = bool;
}


impl Foo<u64> for u8 {
    type Associated = String;
}
```

This is a very contrived example, but it will serve a point. In order to resolve
a reference to an associated type (e.g. `<F as Foo<T>>::Associated`), the Rust
compiler has to search for an `impl` which matches `F` and `T` correctly. Let's
say that we have `F == u16` and `T == u64`. What if the compiler tries the
second impl, of `Foo<u64> for u8`? Then it will immediately find that `F` does
not match `u8` -- so, it's not the impl we're looking for. What if it tries the
first? Then we have `F == u16`, which is true, so all good. Now we have to match
`T == B`. And here's where the magic happens.

The Rust compiler *unifies*
`F` with `B`. Since `B` is a *type variable* -- not a concrete type, like `String`
or `u64` -- its value is free to be assigned. So now `B` is replaced with `u64`,
and going through with the substitution of `B => u64`, we now have an impl of
`Foo<u64> for u16` which is valid. Unification is a fairly simple procedure: it
takes in a type (term) which may have variables, and every time it runs into a place where
a variable that hasn't yet been assigned is matched against any other term, the
variable is assigned to that term (even if the other term is a variable!) Then,
any time that variable is come across while unifying the same term, it's replaced
with its assigned value, and instead unification tries to unify the assigned value
against the other term.

The output of unification is a "substitution" - a mapping of variables to terms,
such that when you take the terms you were trying to unify and replace all the
variables mentioned in the substitution with the terms they're mapped to, you end
up with two identical terms. Here's a standalone example of unification, trying
to unify `Foo<X, Bar<u16, Z>` with `Foo<Baz, Y>`:

```rust
Foo<X, Bar<u16, Z>> == Foo<Baz, Y>
```

First we check the heads of the terms - we get `Foo` vs. `Foo`, which checks out.
So we haven't failed to unify yet. If we had, say, `Foo == Bar`, then unification
would fail. Next, we decompose this into two subproblems. We know `Foo<A, B> == Foo<C, D>`
if and only if `A == C` and `B == D`:

```rust
X == Baz, Bar<u16, Z> == Y
```

We've now *solved* a variable -- `X` has a clearly defined value, `Baz`. So we can
now add to our substitution, `[X -> Baz]`. Then, we apply that substitution to the
second term, `Bar<u16, Z> == Y`. Since there are no occurrences of `X` in there,
nothing happens, and we're left with the same term. And then we have a solution
for `Y`. So our final substitution looks like `[X -> Baz, Y -> Bar<u16, Z>]`. Let's
observe what happens when we apply that to the original equation we wanted to check:

```rust
Foo<X, Bar<u16, Z>> == Foo<Baz, Y> [X -> Baz, Y -> Bar<u16, Z>]
```

Becomes:

```rust
Foo<Baz, Bar<u16, Z>> == Foo<Baz, Bar<u16, Z>>
```

Which is obviously true. The two terms are now equal! So what's an example of
unification failing? Let's try:

```rust
Foo<Bar, X> == Foo<X, Baz>
```

Which breaks down into two equations:

```rust
Bar == X, X == Baz
```

So we solve the first equation, obtaining the substitution `[X -> Bar]`. Applying
this to `X == Baz` yields `Bar == Baz`, which is clearly false. So our unification
has failed -- there isn't a solution.

Unification is such a useful process that there actually exists a programming
language, [Prolog](https://en.wikipedia.org/wiki/Prolog), which *describes programs*
through logical terms, and execution is unification of terms. So that seems suspicious.
If you have a Turing-complete language, Prolog, which is based off of unification,
then it seems straightforward that Rust's trait resolution -- which works by attempting
to unify trait impls until it finds the one which works -- might be Turing-complete
as well.

### Computing with traits

And now that we've got the boilerplate out of the way, let's look at how we actually
write this in Rust. We'll start our Smallfuck implementation now! We'll be using
a macro I wrote a few months ago called [`type_operators!`](https://crates.io/crates/type-operators)
which compiles a DSL into a collection of Rust struct definitions, trait
definitions, and trait impls. While I'll be explaining my implementation using
`type_operators!`, I'll also be showing what the `type_operators!` invocations
expand to. Let's get started!

We'll start simple. How do we represent the bits of our Smallfuck state in Rust
types?

```rust
type_operators! {
    [ZZ, ZZZ, ZZZZ, ZZZZZ, ZZZZZZ]
    
    concrete Bit => bool {
        F => false,
        T => true,
    }
}
```

There are several things to note here. The first is the weird list, `[ZZ, ZZZ, ZZZZ, ZZZZZ, ZZZZZZ]`.
That one's hard to explain, but it has to do with how Rust macros can't create unique
type names. So you have to get around this with a hack where you provide your own list.
Ignore this for now - it's not related to the actual implementation. But, `concrete Bit` is!

`type_operators!` takes two kinds of type-level pseudo-datatype definitions - `data`
and `concrete`. The difference lies in how `type_operators!` generates traits to ensure
you don't mismatch these type-level datatypes. The above example compiles to the
following definitions:


```rust
pub trait Bit {
    fn reify() -> bool;
}


pub struct T;
pub struct F;


impl Bit for T {
    fn reify() -> bool { true }
}

impl Bit for F {
    fn reify() -> bool { false }
}
```

Hopefully you can now see what's happening! `T` and `F` become unit structs
implementing `Bit`. `Bit` gives them a `reify()` function that lets you turn
the types `T` and `F` into the corresponding boolean representation. So you can
write `<T as Bit>::reify()` which will produce `true`. This is useful because
then as long as you have a type variable `B: Bit`, you can use `B::reify()` to
turn it into a boolean. I use this for turning the output of the Smallfuck interpreter
into values that I can check against the runtime implementation.

Hopefully that was fairly clear! Let's look another `concrete` definition we have:

```rust
concrete List => BitVec {
    Nil => BitVec::new(),
    Cons(B: Bit, L: List = Nil) => { let mut tail = L; tail.push(B); tail },
}
```

Now... we have some complexity. Let's dig through this slowly.

This is a type-level cons-list. The first things we get out of this is two struct
types, `Nil` and `Cons`:

```rust
pub struct Nil;
pub struct Cons<B: Bit, L: List = Nil>(PhantomData<(B, L)>);
```

So now we've got bits and lists of bits. We can construct a list `[T, F, F]`
as `Cons<T, Cons<F, Cons<F, Nil>>>`. We also get some traits on top of this:

```rust
pub trait List {
    fn reify() -> BitVec;
}


impl List for Nil {
    fn reify() -> BitVec { BitVec::new() }
}

impl<B: Bit, L: List> List for Cons<B, L> {
    fn reify() -> BitVec {
        let mut tail = <L as List>::reify();
        tail.push(<B as Bit>::reify());
        tail
    }
}
```

So one thing to note is that

```rust
Cons(B: Bit, L: List = Nil) => { let mut tail = L; tail.push(B); tail }
```

results in a sort of syntax sugar where `L` and `B` are automatically reified
and then bound into those variables. This is to get around some limitations of
macro hygiene, and to allow the user to actually use those values.

So hopefully now that you've got that all figured out, let's examine the last
two `concrete` definitions we'll be using:

```rust
concrete ProgramTy => Program {
    Empty => Program::Empty,
    Left(P: ProgramTy = Empty) => Program::Left(Box::new(P)),
    Right(P: ProgramTy = Empty) => Program::Right(Box::new(P)),
    Flip(P: ProgramTy = Empty) => Program::Flip(Box::new(P)),
    Loop(P: ProgramTy = Empty, Q: ProgramTy = Empty) => Program::Loop(Box::new((P, Q))),
}

concrete StateTy => StateTyOut {
    St(L: List, C: Bit, R: List) => {
        let mut bits = L;
        let loc = bits.len();
        bits.push(C);
        bits.extend(R.into_iter().rev());

        StateTyOut {
            loc: loc,
            bits: bits,
        }
    },
}
```

Hopefully you've got all the patterns figured out by now. Here's a complete listing
showing how all these definitions compile down to Rust structs, traits, and impls:

<script src="https://gist.github.com/sdleffler/46700a8add454bc23cdb399a766cb273.js"></script>

The `ProgramTy` is fairly straightforward, and hopefully you can now see why I
chose to encode the run-time encoding of Smallfuck programs as an AST -- to mirror
the type-level encoding as best as possible. The `St` type under the `StateTy`
trait is a [zipper list](https://en.wikipedia.org/wiki/Zipper_(data_structure))
representing simultaneously the pointer location and memory of the Smallfuck
interpreter. The `L` and `R` lists represent the memory to either side of `C`,
the current bit under the pointer.

Now we can look at the most interesting part -- the actual implementation of the
Smallfuck interpreter. It consists of one trait and a dozen or so impls. Here's
the `type_operators!` code:

```rust
(Run) Running(ProgramTy, StateTy): StateTy {
    forall (P: ProgramTy, C: Bit, R: List) {
        [(Left P), (St Nil C R)] => (# P (St Nil F (Cons C R)))
    }
    forall (P: ProgramTy, L: List, C: Bit) {
        [(Right P), (St L C Nil)] => (# P (St (Cons C L) F Nil))
    }
    forall (P: ProgramTy, L: List, C: Bit, N: Bit, R: List) {
        [(Left P), (St (Cons N L) C R)] => (# P (St L N (Cons C R)))
        [(Right P), (St L C (Cons N R))] => (# P (St (Cons C L) N R))
    }
    forall (P: ProgramTy, L: List, R: List) {
        [(Flip P), (St L F R)] => (# P (St L T R))
        [(Flip P), (St L T R)] => (# P (St L F R))
    }
    forall (P: ProgramTy, Q: ProgramTy, L: List, R: List) {
        [(Loop P Q), (St L F R)] => (# Q (St L F R))
        [(Loop P Q), (St L T R)] => (# (Loop P Q) (# P (St L T R)))
    }
    forall (S: StateTy) {
        [Empty, S] => S
    }
}
```

This compiles to a whole lotta stuff. The first thing is a trait. Here's where
that weird gensym list comes into play. Since the gensyms are only used here, they
can't collide with anything the user writes since nothing involving type variables
are actually spliced in. Since writing `ZZ`, `ZZZ` etc. is a pain, I'm going to
ignore what I actually put in that gensym list and just write nicely:

```rust
pub trait Running<S: StateTy>: ProgramTy {
    type Output: StateTy;
}

pub type Run<P: ProgramTy, S: StateTy> = <P as Running<S>>::Output;
```

So, `(Run) Running(ProgramTy, StateTy): StateTy` becomes a sort of type-level
function from two types implementing `ProgramTy` and `StateTy` respectively and
ouputting one type which implements `StateTy`. Now for the stuff inside. Let's
take a look at once simple definition:

```rust
forall (P: ProgramTy, C: Bit, R: List) {
    [(Left P), (St Nil C R)] => (# P (St Nil F (Cons C R)))
}
```

This compiles down to a trait impl, like so:

```rust
impl<P: ProgramTy, C: Bit, R: List> Running<St<Nil, C, R>> for Left<P>
    where P: Running<St<Nil, F, Cons<C, R>>> {
    type Output = <P as Running<St<Nil, F, Cons<C, R>>>>::Output;
}
```

The `(# P (St Nil F (Cons C R)))` means "recursively 'call' the type-level function
with the arguments `P (St Nil F (Cons C R))`." `type_operators!` uses a lisp-like
DSL for ease of parsing; `(A B C)` compiles to the type `A<B, C>`. The `#` "function"
is special in `type_operators!` because its usage must be tracked: whenever a call to
`#` is made, an additional constraint must be added to the `where` clause. You can see
how this works in the example above.

Now that the semantics of the type-level function definition are clarified, let's
look at how Smallfuck is defined. We have four different definitions which deal
with the `Left` and `Right` instructions:

```rust
forall (P: ProgramTy, C: Bit, R: List) {
    [(Left P), (St Nil C R)] => (# P (St Nil F (Cons C R)))
}
forall (P: ProgramTy, L: List, C: Bit) {
    [(Right P), (St L C Nil)] => (# P (St (Cons C L) F Nil))
}
forall (P: ProgramTy, L: List, C: Bit, N: Bit, R: List) {
    [(Left P), (St (Cons N L) C R)] => (# P (St L N (Cons C R)))
    [(Right P), (St L C (Cons N R))] => (# P (St (Cons C L) N R))
}
```

The first one defines what occurs when the pointer moves left, but the left-hand
cons-list in our zipper list is empty. We have to create a new `F` bit, and move
the current bit into the right-hand side cons-list. The left-hand cons list stays
`Nil`. The second definition here works equivalently, but in the case that the
pointer has to move right yet the right-hand cons-list is `Nil`. The third and
fourth definitions, respectively, deal with the cases where the left and right
cons-lists are `Cons<N, L>` and `Cons<N, R>` respectively. In this case, we can
pop the next value out of the cons-list and move it to be the current bit; and
then push the current bit back into the other cons-list. Here's a better visualization:

```
The zipper list is like this:
[...L...] C [...R...]
Where ...L... represents a list which may be Cons or Nil; we don't care. It's
opaque.

[...L..., LN] C [RN, ...R...]
Here is a list where the left-hand list is Cons<LN, L> and the right-hand list
is Cons<RN, R>.

The variables used here are meaningful: L = Left, R = Right, N = Next, C = Current.

Pointer moves left, left-hand side is Nil:
[] C [...R...] => [] 0 [C, ...R...]

Pointer moves right, right-hand side is Nil:
[...L...] C [] => [...L..., C] 0 []

Pointer moves left, left-hand side is Cons<N, L>:
[...L..., N] C [...R...] => [...L...] N [C, ...R...]

Pointer moves right, right-hand side is Cons<N, R>:
[...L...] C [N, ...R...] => [...L..., C] N [...R...]
```

So how does Rust actually "execute" these left and and right movements? Let's say
we have a current program `Left<P>` and a state `St<Nil, F, R>`. When we put these
into the `Run<P, S>` type synonym as `Run<Left<P>, St<Nil, F, R>>` we end up with
`<Left<P> as Running<St<Nil, F, R>>>::Output`. This causes Rust to look for impls
to unify.

Due to Rust's rules for writing traits and impls, there can only ever be at most
one valid impl for any given types Rust attempts to find an impl for. Rust finds
the impl that we compiled by-hand as an example earlier:

```rust
impl<P: ProgramTy, C: Bit, R: List> Running<St<Nil, C, R>> for Left<P>
    where P: Running<St<Nil, F, Cons<C, R>>> {
    type Output = <P as Running<St<Nil, F, Cons<C, R>>>>::Output;
}
```

Rust unifies `Left<P1>` from the trait with `Left<P2>` that we're asking it to
unify with. (The variables have the same letter, but they're technically different
so I'm naming them `P1` and `P2` here.) This works out fine; Rust unifies `P1`
with `P2` and unification succeeds. Then, Rust unifies `St<Nil, C, R1>` with
`St<Nil, F, R2>`. This succeeds; `Nil` is concrete, and `Nil == Nil`. The concrete
type `F` is assigned to the variable `C`. And finally, the variables `R1` and `R2`
are unified without complaints.

Now we have the substitution `[P1 -> P2, C -> F, R1 -> R2]` which is substituted
into the body of the impl. We end up with this:

```rust
impl Running<St<Nil, F, R2>> for Left<P2>
    where P: Running<St<Nil, F, Cons<F, R2>>> {
    type Output = <P as Running<St<Nil, F, Cons<F, R2>>>>::Output;
}
```

And so the "output type" of the "computation" is `<P as Running<St<Nil, F, Cons<F, R2>>>>::Output`.

Now that we've got that under our belts, let's look at the handling for the
`Flip` instruction in `Running`:

```rust
forall (P: ProgramTy, L: List, R: List) {
    [(Flip P), (St L F R)] => (# P (St L T R))
    [(Flip P), (St L T R)] => (# P (St L F R))
}
```

So this is fairly straightforward. The left and right lists aren't modified at
all. We recursively call `Run` on the program after the `Flip` instruction, and
change `F -> T` and `T -> F` by pattern-matching.

```rust
forall (P: ProgramTy, Q: ProgramTy, L: List, R: List) {
    [(Loop P Q), (St L F R)] => (# Q (St L F R))
    [(Loop P Q), (St L T R)] => (# (Loop P Q) (# P (St L T R)))
}
```

And here's the most complex instruction -- `Loop<P, Q>`. We check the current bit
by pattern-matching. In the case that it's `F`, which means `0` in terms of the
runtime representation, we recursively run the part of the program *after* the
loop, hence skipping the body of the loop. In the case that the current bit is
`T`, we make *two* recursive calls. The first produces a new state by running
the body of the loop. Then, we run `Loop<P, Q>` again with the new state. This
allows the program to branch off of the current state again.

And now we're totally finished with all the complicated bits! The last instruction
to handle is just `Empty`. All `Empty` does is return the current state unmodified,
and not recurse:

```rust
forall (S: StateTy) {
    [Empty, S] => S
}
```

Relax! You're finished with the type-level nastiness. Here's a code listing showing
the full expanded version of the `Running` trait and its impls:

<script src="https://gist.github.com/sdleffler/a432103278432140f84f6b17869f8b52.js"></script>

## Wrap-up: Testing and Conclusions

Along with the type-level implementation, I include two macros, `sf!` and `sf_test!`,
which allow for testing of the runtime implementation against the type-level implementation.
Here they are, in all their fairly simple glory:

```rust
// A Smallfuck state which is filled with `F` bits - a clean slate.
pub type Blank = St<Nil, F, Nil>;


// Convert nicely formatted Smallfuck into type-encoded Smallfuck.
macro_rules! sf {
    (< $($prog:tt)*) => { Left<sf!($($prog)*)> };
    (> $($prog:tt)*) => { Right<sf!($($prog)*)> };
    (* $($prog:tt)*) => { Flip<sf!($($prog)*)> };
    ([$($inside:tt)*] $($outside:tt)*) => { Loop<sf!($($inside)*), sf!($($outside)*)> };
    () => { Empty };
}


macro_rules! sf_test {
    ($($test_name:ident $prog:tt)*) => {
         $(
            #[test]
            fn $test_name() {
                let prog = <sf! $prog as ProgramTy>::reify();

                let typelevel_out = <Run<sf! $prog, Blank> as StateTy>::reify();
                let runtime_out = prog.run();

                println!("Program: {:?}", prog);
                println!("Type-level output: {:?}", typelevel_out);

                let offset = runtime_out.ptr.wrapping_sub(typelevel_out.loc as u16);

                for (i, b1) in typelevel_out.bits.into_iter().enumerate() {
                    let b2 = runtime_out.get_bit((i as u16).wrapping_add(offset));
                    println!("[{}] {} == {}",
                            i,
                            if b1 { "1" } else { "0" },
                            if b2 { "1" } else { "0" });
                    assert_eq!(b1, b2);
                }
            }
         )*
    }
}
```

I have to credit [durka](https://github.com/durka) for fixing my `sf!` macro, which
was originally near-useless. Thanks much!

Two tests are also included with the full source code of the runtime and type-level
implementations:

```rust
sf_test! {
    back_and_forth {
        > * > * > * > * < [ * < ]
    }
    forth_and_back {
        < * < * < * < * > [ * > ] > > >
    }
}
```

The tests check only the bits set by the type-level implementation relative to
the location of the pointer, but it's enough for me to be certain that the implementations
are correct. Smallfuck is so simple there's almost no room for error; nevertheless,
if anyone would like to contribute more tests, [I would welcome any pull requests.](https://github.com/sdleffler/tarpit-rs)

So, Rust's type system is Turing-complete. What does this mean?

Pretty much nothing. Honestly, it means pretty much nothing. Sure, the type system
can get into infinite loops, but we already have a recursion limit in the type checker
so that's nearly irrelevant. Sure, we can write things like Smallfuck in the type
system. Okay, that last one's kinda cool.

In most cases where the typechecker hits the recursion limit, something's wrong with
your program and it won't compile no matter what. Only if you're *really* mucking
with the type system -- like, say, writing seriously large Smallfuck programs with
it -- then maybe you'll hit the recursion limit.

Rust's type system is Turing-complete. Maybe this'll resolve some highly speculative
discussions between anyone looking to talk about extensions to Rust's type system --
but if you're just writing Rust, it's just a curiosity. Not much more. But pretty cool, no?

One last time: the full source code of this project is available here, on GitHub. https://github.com/sdleffler/tarpit-rs
