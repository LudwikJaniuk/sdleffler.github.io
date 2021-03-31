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

# Procedural macros as compilers

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
based DSLs! And what I'd like to share here are some relatively simple bits and pieces of how ours
works and how you could use these techniques to make your own.

# Parsing with `syn`

If you've written proc macros before, you've almost certainly used `syn`. But I'm still going to
talk a little bit about it here, and how we use it.

While you can't do any sort of serious declarative grammar stuff with `syn`, you can still do some
fairly in-depth parsing, and one of its most powerful features is that you can parse your own
arbitrary macro grammar and then parse something like a full Rust type or statement or expression
using `syn`'s own builtin Rust syntax parsing features. For example, here's a snippet from the bit
of our parser which parses `call` statements:

```rust
// ...
        } else if lookahead.peek(kw::call) {
            // Ast::Call: call <type> or call <block>
            let call_span = input.parse::<kw::call>()?.span();
            let lookahead = input.lookahead1();

            let callee = if lookahead.peek(token::Brace) {
                input.parse::<Block>()?.0
            } else if kw::peek_any(input) {
                return Err(input.error("expected a block or a type, but found a keyword"));
            } else {
                let ty = input.parse::<Type>().map_err(|mut e| {
                    e.combine(lookahead.error());
                    e
                })?;

                let span = ty.span();

                Spanned {
                    inner: Syntax::Type(ty),
                    span,
                }
            };

            Ok(Spanned {
                inner: Syntax::Call(Box::new(callee)),
                span: call_span,
            })
        } else if lookahead.peek(kw::choose) {
// ...
```

We have a few custom keywords of our own, defined using `syn`'s custom keyword macros and kept in a
`kw` module inside our parser (`kw::peek_any` is our little shortcut function for checking if
something is one of our custom keywords.) We get these out of the way so that we can be sure we
don't accidentally parse one as an identifier inside a `syn::Type`, and then we can just go ahead
and parse a `Type` happily - which lets us get arbitrary Rust type syntax inside our proc macro,
basically for free! You can use this for lots of things where you would normally use a `tt` or
`expr` or `stmt` or such block inside a `macro_rules!` macro. This part of `syn` is invaluable to
writing proc macros for Rust.

## Proc macros make small syntax trees

Depending on your use case... you'll probably never have more than a couple thousand nodes in a
little custom AST for your macro language. So you can keep it real simple when designing the
representation. No need for an arena or string interning or such to keep it efficient; things are
pretty much guaranteed to be small enough that you can be elegant rather than just efficient.

# CFG representations can look a little intimidating to work with but are actually really easy

If you've ever looked into writing a compiler before, you've probably come across the idea of a
control flow graph, or "CFG". Or maybe you've hit one of its variants, like SSA (Single Static
Assignment) form or VSDGs (Value State Dependence Graph). And diagrams with these can also look a
bit intimidating to implement, at least in my opinion. So I do want to share how ours works, because
it was honestly really easy to get going, and immediately enabled some very thoroughly nice
reasoning.

Our CFG is built on a library called [`thunderdome`](https://docs.rs/thunderdome), which provides a
really nice and ergonomic arena collection type. It's great for implementing graph type structures
in Rust. If you've ever tried implementing a graph in Rust using just references or with
`std::rc::Rc`, you probably know that it's a total pain for your references to other graph nodes to
be difficult to dereference, often have to have `Weak` pointers to break cycles, or in the case of
references, be simply impossible to define without using unsafe code or a safe arena allocator like
[`typed_arena`](https://docs.rs/typed-arena). Thunderdome provides a simple interface with copyable
`Index`s used to access allocated values. With that in hand, our CFG consists of three main types:

```rust
/// A single entry for a node in the CFG.
#[derive(Debug, Clone)]
pub struct CfgNode {
    /// The variant of this node.
    pub expr: Ir,
    /// The continuation of the node, if present. Absence of a continuation indicates the `Done` or
    /// "halting" continuation.
    pub next: Option<Index>,
    /// The associated syntactic span of this node. If an error is emitted regarding this node, this
    /// span may be used to correctly map it to a location in the surface syntax.
    pub span: Span,
    /// "Machine-generated" nodes are allowed to be unreachable. This is used when automatically
    /// inserting continues when resolving scopes, so that if a continue is inserted which isn't
    /// actually reachable, an error isn't emitted.
    pub allow_unreachable: bool,
    /// Nodes keep a hashset of errors in order to deduplicate any errors which are emitted multiple
    /// times for the same node. Errors are traversed when code is emitted during
    /// [`Cfg::generate_target`].
    pub errors: HashSet<CompileError>,
}

/// The "kind" of a CFG node. CFG nodes have additional data stored in [`CfgNode`] which is always
/// the same types and fields for every node, so we separate into the `Ir` variant and `CfgNode`
/// wrapper/entry struct.
#[derive(Debug, Clone)]
pub enum Ir {
    /// Indicating receiving a value of some type.
    Recv(Type),
    /// Indicating sending a value of some type.
    Send(Type),
    /// `Call` nodes act somewhat like a call/cc, and run their body continuation in the same scope
    /// as they are called all the way to "Done" before running their own continuation.
    Call(Option<Index>),
    /// `Choose` nodes have a list of continuations which supersede their "next" pointer. Before
    /// scope resolution, these continuations may be extended by their "implicit" subsequent
    /// continuation, which is stored in the "next" pointer of the node. The scope resolution pass
    /// "lowers" this next pointer continuation into the arms of the `Choose`, and so after scope
    /// resolution all `Choose` nodes' next pointers should be `None`.
    Choose(Vec<Option<Index>>),
    /// Like `Choose`, `Offer` nodes have a list of choices, and after scope resolution have no
    /// continuation.
    Offer(Vec<Option<Index>>),
    /// `Split` nodes have a transmit-only half and a receive-only half. The nodes' semantics are
    /// similar to `Call`.
    Split {
        /// The transmit-only half.
        tx_only: Option<Index>,
        /// The receive-only half.
        rx_only: Option<Index>,
    },
    /// Early on, loops *may* have a body; however, after scope resolution, all loops should have
    /// their bodies be `Some`. So post scope resolution, this field may be unwrapped.
    Loop(Option<Index>),
    /// `Break` nodes directly reference the loop that they break. They can be considered as a
    /// direct reference to the "next" pointer of the referenced loop node.
    Break(Index),
    /// Like break nodes, `Continue` nodes directly reference the loop that they continue.
    /// Semantically they can be considered a reference to the body of the loop, but as they are a
    /// special construct in the target language, we don't resolve them that way.
    Continue(Index),
    /// A "directly injected" type.
    Type(Type),
    /// Emitted when we need to have a node to put errors on but need to not reason about its
    /// behavior.
    Error,
}

/// A cfg of CFG nodes acting as a context for a single compilation unit.
#[derive(Debug)]
pub struct Cfg {
    arena: Arena<CfgNode>,
}
```

This is the actual definition, with doc comments and all! It may look like a lot but it's the entire
language in one spot. The `Arena` in `Cfg` is a Thunderdome arena, and the `Index` types in `Ir` and
`CfgNode` are also from Thunderdome. Crucially, `Type` here is from `syn`. Thanks to `syn`, we can
directly embed parsed Rust type syntax into our IR, which makes things really easy later. Way better
than using strings that you have to parse again into tokens or something.

Most notably, one of the things that having a CFG gives you is the ability to attach additional
information to every node. In our case, we keep a set of errors associated with each node as well as
some other data about whether or not to emit unreachable code errors for a given node. The
`allow_unreachable` field is specifically there because we auto-insert `Ir::Continue` nodes. And, the
auto-insertion is a bit overenthusiastic and will sometimes insert unreachable `Continue` nodes (but
that's absolutely fine as long as we don't generate errors from them.)

So what does this actually look like in practice? Here's the code which produces a set of reachable
CFG nodes. We use this during dead code reporting, because we need to find the *boundary* of
reachable and unreachable code. We don't want to report lots of unreachable code errors which could
be summed up as just one, and generate way too much noise for the user to look through.

```rust
    /// Compute the set of nodes which are reachable from the root of the [`Cfg`], using the results
    /// of the [`FlowAnalysis`] to determine whether nodes can be passed through.
    fn analyze_reachability(&self, flow: &FlowAnalysis, root: Index) -> HashSet<Index> {
        let mut stack = vec![root];
        let mut reachable = HashSet::new();

        while let Some(node_index) = stack.pop() {
            if !reachable.insert(node_index) {
                continue;
            }

            let node = &self[node_index];

            match &node.expr {
                Ir::Loop(child) | Ir::Call(child) => {
                    stack.extend(child);

                    if flow.is_passable(node_index) {
                        stack.extend(node.next);
                    }
                }
                Ir::Split { tx_only, rx_only } => {
                    stack.extend(tx_only.iter().chain(rx_only));

                    if flow.is_passable(node_index) {
                        stack.extend(node.next);
                    }
                }
                Ir::Choose(choices) | Ir::Offer(choices) => {
                    stack.extend(choices.iter().filter_map(Option::as_ref));

                    assert!(node.next.is_none(), "at this point in the compiler, all continuations of \
                        `Choose` and `Offer` nodes should have been eliminated by the resolve_scopes pass.");
                }
                Ir::Send(_)
                | Ir::Recv(_)
                | Ir::Type(_)
                | Ir::Break(_)
                | Ir::Continue(_)
                | Ir::Error => {
                    if flow.is_passable(node_index) {
                        stack.extend(node.next);
                    }
                }
            }
        }

        reachable
    }
```

At this point, many of the passes and transforms and checks that need to be done are written as
depth or breadth-first traversals over the CFG. If you wanted to, you could write up a visitor
pattern but we did find that generally speaking we wouldn't get too much benefit from going through
that process. As long as your language is fairly simple there's no harm in writing it all out by
hand.

# Flow analysis via a kind of SAT-ish solver is kinda fun

In order to properly analyze some of this stuff, you need something a bit more in-depth than just a
plain traversal. Flow analysis is one of these things; in our case, there are two things we want to know about
nodes:

- Whether a node is "passable", meaning that execution *can* flow "through" it to the next node.
- Whether a node is "haltable", meaning that execution *can* halt if it passes through this node.

The haltability analysis is special to some of our semantics about the `call` node. The `call` node
is really a kind of `call/cc`, which allows us to implement "context-free session types", a strictly
more powerful form of session type. The rule for a `call` node to be passable is that the session
type inside the `call` must have the potential to halt, because control flow only continues to the
node after the `call` after the entirety of the inside of the `call` comes to a halt. We can
calculate the passability/haltability of all the nodes that matter using three different
constraints/predicates over the CFG:

```rust
/// Flow analysis works by using a constraint solver: we try to solve for whether every node in the
/// graph is "passable", and in the process of doing so, we also solve "haltable" and "breaks from
/// within some node or its children to some given parent loop node" constraints.
#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Hash)]
pub enum Constraint {
    /// A `Passable` constraint carries the information of which node we are trying to check is
    /// passable or not.
    Passable(Index),
    /// A `Haltable` constraint for checking whether or not a node is "haltable".
    Haltable(Index),
    /// A `BreakableTo` constraint for checking whether or not a loop can be "broken to" by a node
    /// within its body.
    BreakableTo {
        /// The node we want to check for a `Break`.
        breaks_from: Index,
        /// The loop we want to `Break` to.
        breaks_to: Index,
    },
}
```

These constraints can be traced to exactly where they originate in the conditions for each relevant
node to be "passable". The passable conditions for pretty much every node but `call`, `split`, and
`loop` can be defined in terms of only passability. But for `call` and `split`, we have to show that
the subroutines may halt, and for `loop`, we have to show that the inside of the loop contains a
`break` which targets the loop node itself. Which is where the `BreakableTo` constraint comes from.
Now that we have these constraint types, we can effectively express what constraints must be true
for some condition:

```rust
/// A conjunction of constraints: true if *all* are true.
pub type Conjunction = Vec<Constraint>;

/// A disjunction of conjunctions of constraints: true if *any* of the clauses contain constraints
/// *all of which* are true.
#[derive(Debug, Clone)]
pub struct Dnf(pub Vec<Conjunction>);

impl Dnf {
    /// The trivially false disjunction, unprovable no matter what.
    pub fn trivially_false() -> Self {
        Dnf(vec![])
    }

    /// The trivially true disjunction, provable with no evidence.
    pub fn trivially_true() -> Self {
        Dnf(vec![vec![]])
    }

    /// Tests if a conjunction is *trivially* true (i.e. requires no evidence to be true).
    pub fn is_trivially_true(&self) -> bool {
        self.0.iter().any(|c| c.is_empty())
    }

    /// Construct a disjunction from a single conjunction.
    pub fn only_if(conj: Conjunction) -> Self {
        Dnf(vec![conj])
    }
}
```

Where `Dnf` stands for Disjunctive Normal Form. So if you've ever done logic/SAT solvers, this is
probably starting to look really familiar!

We have one last little puzzle piece to place before I can explain how this solver works in its
entirety:

```rust
/// An implication in normal form: a set of preconditions which prove a consequence.
///
/// When we can satisfy the preconditions (listed in disjunctive normal form), we are allowed to
/// learn that the consequence is true and insert it into our set of knowledge, which we use to
/// prove further preconditions to be true.
#[derive(Debug)]
pub struct Implication {
    /// The preconditions to the implication.
    pub preconditions: Dnf,
    /// The consequent of the implication.
    pub consequent: Constraint,
}
```

So... how does the solver work? It's really simple! We keep a queue of implications to try. When we
run out of implications to prove, we've shown everything we've asked is true and we're done:

```rust
/// The current state of the flow analysis solver.
#[derive(Debug)]
pub struct Solver<'a> {
    /// The [`Cfg`] being solved for in this invocation of the analysis.
    cfg: &'a Cfg,
    /// The set of nodes which are currently proven to satisfy the `Passable` constraint.
    passable: HashSet<Index>,
    /// The set of nodes which are currently proven to satisfy the `Haltable` constraint.
    haltable: HashSet<Index>,
    /// The set of pairs of nodes which are currently proven to satisfy the `BreakableTo`
    /// constraint, in the order of `breaks_from` followed by `breaks_to`.
    breakable_to: HashSet<(Index, Index)>,
    /// The number of implications examined since the last progress was made. (If this exceeds the
    /// length of the implication queue, the solver cannot make any more progress.)
    progress: usize,
    /// The queue of [`Implication`]s yet to be proven.
    queue: VecDeque<Implication>,
    /// The set of all [`Constraint`]s which either have been proven or are in-progress.
    queried: HashSet<Constraint>,
}
```

We initialize the solver by asking it to prove a `Passable` constraint on every node (since that's
what we really care about; we could also ask it to show `Haltable` but we don't use that information
outside the flow analyzer, so instead we only calculate `Haltable` where it's necessary.) The "meat"
of the solver, where we keep all of the logic that defines what makes a node passable or not, lives
in a function called `preconditions` inside the `flow` module, and it's called in exactly one place,
the solver's `push` method:

```rust
    /// Push a constraint into the solver's state, returning whether or not it is currently known to
    /// be true.
    fn push(&mut self, constraint: Constraint) -> bool {
        if self.queried.insert(constraint) {
            let dnf = preconditions(self.cfg, constraint);

            if dnf.is_trivially_true() {
                // Optimization: if the precondition of an implication is trivially true, we don't
                // need to examine its conditions, we can just assert the truth of its consequent
                self.insert_fact(constraint);
            } else {
                self.queue.push_back(Implication {
                    preconditions: dnf,
                    consequent: constraint,
                });
            }

            true
        } else {
            false
        }
    }
```

The `preconditions` function takes a constraint (now the "consequent") and returns the `Dnf` corresponding to the necessary
constraints which must be true in order to prove the consequent (the "preconditions".) Some
preconditions are trivially true; if they are, we can just insert that constraint directly into the
knowledge base instead of pushing it onto our queue of implications we're trying to prove.

During solving, we check to see if we're stuck or not; we're stuck if we've gone through the whole
queue and made no progress and looped back around to the front again. This is checked using a
counter inside the solver which is incremented every time we pop an implication and set to zero
every time we make progress. When the counter is greater than the length of the queue, we're stuck
and there were some things we couldn't prove:

```rust
    /// Pop a new yet-to-be-proven [`Implication`] from the [`Solver`]'s queue. If no more progress
    /// can be made, or there are no more implications left, returns [`None`].
    fn pop(&mut self) -> Option<Implication> {
        if self.progress > self.queue.len() {
            None
        } else {
            self.progress += 1;
            self.queue.pop_front()
        }
    }

    ...

    /// Run the full flow analysis algorithm and return a solution.
    pub fn solve(mut self) -> FlowAnalysis {
        while let Some(implication) = self.pop() {
            let validity = self.try_prove(&implication.preconditions);

            if let Validity::Trivial = validity {
                self.insert_fact(implication.consequent);
            } else {
                self.queue.push_back(implication);
            }

            if let Validity::Trivial | Validity::Progress = validity {
                self.progress = 0;
            }
        }

        FlowAnalysis {
            passable: self.passable,
            haltable: self.haltable,
        }
    }
```

Once we pop an implication off the stack, we try to prove it, which is done by checking to see if
its preconditions are satisfied. Since the preconditions are in DNF form, this means we can check
each of the conjunctions inside the main disjunction, and if all of the constraints inside any one
of those conjunctions are true, then the precondition holds true. If this is *not* the case, then we
take *every* constraint in the DNF and push it onto the queue, ignoring constraints which we've
pushed before. If we proved the precondition trivially, or if we've inserted a constraint we haven't
seen before, then we've made progress; the constraint set has changed, and preconditions which were
previously not provable might now be provable. Otherwise, we've made no progress:

```rust
    /// Test if a constraint currently holds, given what we already know.
    fn constraint_holds(&self, constraint: &Constraint) -> bool {
        match constraint {
            Constraint::Passable(node) => self.passable.contains(node),
            Constraint::Haltable(node) => self.haltable.contains(node),
            Constraint::BreakableTo {
                breaks_from,
                breaks_to,
            } => self.breakable_to.contains(&(*breaks_from, *breaks_to)),
        }
    }

    /// Attempt to prove that a DNF holds. If it can not yet be shown to be true, insert its atomic
    /// constraints so they can be proven in the future, potentially.
    fn try_prove(&mut self, dnf: &Dnf) -> Validity {
        if dnf
            .0
            .iter()
            .any(|cj| cj.iter().all(|c| self.constraint_holds(c)))
        {
            Validity::Trivial
        } else {
            let mut progress = false;
            for &constraint in dnf.0.iter().flatten() {
                progress |= self.push(constraint);
            }

            if progress {
                Validity::Progress
            } else {
                Validity::NoProgress
            }
        }
    }
```

And... that's basically it! It looks like a lot but it's a machine performing logical analysis of a
simple CFG. I hope that someone can use this as a rough template or example for their own project,
maybe for another Rust proc macro.

# Wrapping up

These were the two most interesting parts of the proc macro project (or so I thought) for others to
see, and the bits I thought might be most useful to others. Again, the [full
README](https://github.com/boltlabs-inc/dialectic/tree/main/dialectic-compiler) for the macro
compiler crate is intended as a reference for contributing and is a good resource on how the whole
thing works. The macro compiler is also fully documented, so you can [look at it on
docs.rs](https://docs.rs/dialectic-compiler) if you want to know about specific bits or pieces. We
consider it to be pretty thoroughly commented, if we do say so ourselves.

If you have questions, you can contact me on Twitter at [@sleffy_](https://twitter.com/sleffy_),
through email at `shea@errno.com`, or on Discord as SLEFFY#6314. Happy to talk about it anytime!

- Shea
