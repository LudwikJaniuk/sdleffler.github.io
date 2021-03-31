---
layout: post
title: Compiling Session Types With Procedural Macros in Rust
published: false
---

I've recently been doing some work for a company called Bolt Labs, mainly on a very interesting
library called Dialectic. Dialectic is a Rust crate which provides a form of "session typing", which
in this context is a way to write out a specification of a protocol for communication between two
parties, as a type. Practically speaking, what this means is that you can write a type for some
protocol and then the Rust compiler (and the Dialectic crate) will prevent you from performing
operations in it in the wrong order or at the wrong time. Strictly speaking, this sort of thing has
typically been restricted to research languages for quite a while, for a few very relevant reasons:

1. Your type system has to be strong enough and expressive enough to be capable of representing
   session types in some form.
2. Your type system must be capable of reasoning about *linear types.* A linear type *must* be used
   exactly once; you can't copy it around, or use it in multiple places in a function, and you can't
   just discard it. It *has* to be consumed by something.
3. Your language must be capable of expressing session types in such a way that they are humanly
   readable and writable. If you can't write out and read these specs yourself, there is little
   point in having them, and debugging becomes very difficult.

I'm pretty excited about Dialectic because it's an opportunity for research tech to get into
practical usage! And here's why:

1. Rust's type system is strong and expressive enough to represent session types. There are a couple
   of things we have to work around but it's way more than doable and set to be improved upon in
   future Rust releases.
2. Rust supports *affine* typing, which is kind of like linear typing. The main difference is that
   affine types can be discarded/left unused, and this is close enough for Dialectic to work well.
3. **Dialectic lets you write out session types in a very expressive domain specific language,
   implemented as a procedural Rust macro that generates types.**

So let's get to the point. Here's an example of a very simple session type written in Dialectic, for
a little protocol which counts how long your name is:

```rust
type Server = Session! {
    recv String; // Receive a `String`; and then...
    send String; // ... send one back!
};
```

And here's a server implementing this tiny protocol:

```rust
#[Transmitter(Tx ref for String)]
#[Receiver(Rx for String)]
async fn server<Tx, Rx>(chan: Chan<Server, Tx, Rx>) -> Result<(), Box<dyn Error>>
where
    Tx::Error: Error + Send,
    Rx::Error: Error + Send,
{
    let (name, chan) = chan.recv().await?;
    let greeting = format!(
        "Hello, {}! Your name is {} characters long.",
        name,
        name.chars().count()
    );
    let chan = chan.send(&greeting).await?;
    chan.close();
    Ok(())
}
```

So in maybe 15 lines of code we have a type-checked implementation of a protocol which, minus a
little boilerplate (asserting that we can transmit `String`s by reference and also receive
`String`s) is type-checked, asserting that it does in fact try to receive a string and then send a
string. And which also doesn't care about what backend it is run over; you can test this protocol
using an MPSC queue backend before running it over a TCP socket in production. Or anything else
which implements the necessary `Transmitter`/`Receiver` backend traits.

And best of all, Dialectic is *almost* zero overhead! Benchmarks let us do something like 10 million
ops per second using Dialectic and the MPSC queue backend, so any sort of actual network
communication backend will have no visible extra cost from using Dialectic. Rust makes this possible
via its "zero-cost abstractions" and async functions.

And, if you looked at that `Session!` macro invocation a couple code blocks back, you were probably
able to quickly figure out what it meant. So... what would it look like without the `Session!`
macro?

```rust
type Server = Recv<String, Send<String, Done>>;
```

...huh, well, you can still sorta read it. It's in a kind of continuation passing style, where the
last argument of a session type is the "next" action to take. Okay. Let's take a look at a more
complex example, for a protocol which lets you send a bunch of numbers and then sum or multiply them
all:

```rust
type Server = Session! {
    // While the client has more sets of numbers to tally...
    loop {
        // Offer the client the choice to keep going or end the session.
        offer {
            // If the client wants to keep going...
            0 => {
                recv Operation; // Let the client tell us whether they want to sum or multiply these numbers.
                call ServerTally; // And then do the `ServerTally` subprotocol/subroutine.
            }
            // If the client has no more to tally, we're done!
            1 => break,
        }
    }
};

type ServerTally = Session! {
    // Until the client is satisfied...
    loop {
        // Offer the client the choice to receive a new number to add/multiply or end
        // the tally and receive the total.
        offer {
            // If there's another number, receive it so we can add/multiply it to the total.
            0 => recv i64,
            // If there's not, send the total and break out of the tally loop.
            1 => {
                send i64;
                break;
            }
        }
    }
};
```

And this is nicely readable and can be commented real nicely too! Then, let's see what it looks like
written out as types:

```rust
type Server = Loop<
    Offer<(
        Recv<Operation, Call<ServerTally, Continue<0>>>,
        Done,
    )>
>;

type ServerTally = Loop<
    Offer<(
        Recv<i64, Continue<0>>,
        Send<i64, Done>,
    )>
>;
```

Well. This is *significantly* less readable, doesn't look like Rust code at all, is harder to nicely
comment, and offers little explanation as to its purpose. I hope you can see how with larger
protocols, even if they're broken up into subroutines, writing them out as types can be far more
intimidating and write-only.

So how does this macro work? It's not trivial. The target language has "break-by-default" loops
which never continue unless you explicitly `Continue<N>`, which feels very un-Rust-like and can be
more difficult to work with, since the "next" operation of a loop is still inside the `Loop` type.
If we want to do things like provide Rust-like loop labels, break-to-label, and more, we have to do
some significant transformations on the syntax.

So now that you see our motivation, let's take a look at how the `Session!` macro compiler works.

# Procedural Macros as Compilers

In Rust, procedural macros (or "proc macros") are basically syntax transform machines which take in
arbitrary tokens and spit out arbitrary tokens. You are then free to parse the input syntax however
you like, and then spit out tokens with "spans" mapping them to regions of the original source for
the purpose of error reporting, whether for user errors or for Rust telling you that your macro is
emitting unparseable or bad syntax or type errors in your output.

While the `Session!` macro is a bit ridiculous for a procedural macro, it's not at all complex as
far as compilers go. It consists of three main representations and a couple of passes on each:

1. Input comes in as a Rust `TokenTree` and is parsed using the `syn` crate into an AST type, `Syntax`.
2. The `Syntax` type is transformed into a control flow graph (CFG) representation, inside the `Cfg`
   type.
3. Two passes are run on the `Cfg` type: scope resolution and dead code reporting. Scope resolution
   is responsible for transforming the CFG from having "implicit" continuations to having "explicit"
   continuation-passing-style (CPS) continuations. Dead code reporting looks for parts of the
   session type which will never be reached, maybe because of a loop that can never break or such.
   It is not a naive pass, but rather performs a kind of "true" control flow analysis, and can be
   relied on to catch *any* unreachable nodes which would otherwise silently disappear during
   compilation.
4. The `Cfg` is transformed into the `Target` type, which directly represents the syntax of the type
   language. During this transformation (`Cfg::generate_target`) explicit loop targets for nodes
   like `continue` and `break` statements are transformed into DeBruijn indices, indicating which
   "level" of loop they need to continue or break out of, and `break` statements are resolved into
   the nodes which they "jump" to.
5. The `Target` type is lowered to a Rust `TokenTree` and returned from the procedural macro.

There are a number of benefits from performing the transform like this:

- During parsing, we can produce informative syntax errors which correspond to relevant spans of the
  input.
- Analysis of the session type during the CFG phase lets us produce informative errors for users
  who are designing a protocol, and catch potential mistakes ahead of time. The goal with Rust and a
  library like this is to get as close as possible to complete confidence in your program once it
  compiles.
- Compiling a DSL like this allows us to have *much* more reasonable features for working with
  loops, like labeled breaks and continues. I think it's underrated how useful these things are when
  you need them, especially for what is essentially a spec language. Going first to a CFG
  representation before moving to a DeBruijn-indexed representation for loops is much easier to
  understand.
- There are a few weird errors coming from limitations of how the Rust type system/trait resolver
  works. We can catch these during an "unproductive loop" check while lowering to the target
  language, instead of letting the user see some huge infinite-recursion type errors.

If you want to see the actual implementation, the standalone session macro compiler is in [its own
crate.](https://github.com/boltlabs-inc/dialectic/tree/main/dialectic-compiler) The README there
also gives a comprehensive overview of the compiler's design in greater detail, including invariants
upheld between passes and possible errors emitted by each pass.

# So what's the point here?

Procedural macros in Rust are an *incredibly* powerful tool. "Regular" `macro_rules!` macros are
already an incredibly powerful tool in Rust, but proc macros take it to a whole new level, and let
you nicely report errors as well *and* test your macros rigorously. In our case, the proc macro
parser is quickcheck-tested to ensure that given a random generated AST, the act of repeatedly
printing and then re-parsing the syntax is idempotent. And we have a number of tests to ensure both
that the expected types are generated and also that the resulting types compile in the context of an
integration test.

I hope that this inspires you to make more complex, interesting, and thoroughly cool proc macro
based DSLs!
