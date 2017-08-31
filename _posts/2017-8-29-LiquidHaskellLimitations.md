---
layout: post
title: Strengths and Limitations of Liquid Haskell, and the Puzzle of `the`
published: false
---

This summer, I was lucky enough to be accepted to work with Liquid Haskell as
part of the Summer of Haskell project! This blog post is an explanation of one
of the more interesting puzzles involved, as well as what I learned of Liquid
Haskell, its strengths, and its limitations.

## What's Liquid Haskell?

Liquid Haskell is an external formal verification system for Haskell code. It
consists of a program, `liquid`, which reads in specifications written as
comments, generates a series of logical constraints, and then feeds the
generated constraints to an [SMT
solver](https://en.wikipedia.org/wiki/Satisfiability_modulo_theories), a
program capable of deciding whether or not the logical constraints are
"satisfiable" and non-contradicting. If the SMT solver is able to determine
that the constraints are satisfiable, then the program is "safe". Otherwise, a
"liquid type error" has occurred, and the program may produce undesirable
behavior when run. My Summer of Haskell project was to try to verify `base`,
the standard libraries of Haskell.

Liquid Haskell's type system is based around the idea of *refinement types*, a
system for "refining" types in a source language - here, Haskell - with logical
predicates. For example, take the `head` function as defined in Haskell's
`base` libraries, which proponents of Liquid Haskell would find highly unsafe:

```haskell
head :: [a] -> a
head (x:xs) = x
```

Note that if we call `head []`, our program will crash! Here come refinement types to the rescue:

```haskell
{-@ head :: {xs:[a] | len xs > 0} -> a @-}
head (x:xs) = x
```

There are a few things to note right off the bat, and the first is that the
Haskell type signature is gone! We've replaced it with a comment (which is fine
because Haskell can infer the correct type on its own for this simple case)
containing a "refined" type for `head`. The refined type is conceptually a
"subset" of all values `xs` of type `[a]` in Haskell - the subset where `len
xs`, the length of `xs`, is greater than zero. Liquid Haskell will then require
that any list passed into `head` have a length greater than zero, and if this
can't be proven from the environment, Liquid Haskell will throw a liquid type
error. Liquid Haskell will also (with the `--total` flag) attempt to check the
totality of functions in your program; that is, that your functions terminate
(do not loop forever or crash.)

We can express an extraordinary number of invariants with this system. For
example, we can express that `map` outputs a list with the same length as the
input list:

```haskell
{-@ map :: (a -> b) -> xs:[a] -> {ys:[a] | len xs == len ys} @-}
map f (x:xs) = f x : map f xs
map _ []     = []
```

We can express that indexing a list is only safe when the arguments are in bounds:

```haskell
{-@ (!!) :: xs:[a] -> {n:Int | 0 <= n && n < len xs} -> a @-}
(x:xs) !! n | n == 0    = x
(x:xs) !! n | otherwise = xs !! (n - 1)
```

Liquid Haskell is even capable of reasoning about the equality of Haskell types
by using "measures": Haskell functions lifted into the predicate logic used in
refinement types. In fact, the `head` function from before can be
straightforwardly lifted to a measure simply by adding the `{-@ measure head
@-}` annotation.

## How does it work?

Suppose that we look into the mind of Liquid Haskell checking our `map`
function. Liquid Haskell sees a recursive function with two cases. Let's look
at the second case first, the nil case:

```haskell
map _ [] = []
```

Our refined type for `map` is `(a -> b) -> xs:[a] -> {ys:[a] | len xs == len
ys}`. In this case, `xs` is `[]` and `ys` is `[]`. A single constraint needs to
be satisfied: `len xs == len ys`. Liquid Haskell knows that `len xs == 0`
because `len [] == 0`, by definition, and the same with `len ys`; thus, the
constraint is satisfied. On to the cons case:

```haskell
map f (x:xs) = f x : map f xs
```

Now, `xs = x:xs` and `ys = f x : map f xs`. Liquid Haskell knows the refined
type of `(:)`, which goes something like

```haskell
(:) :: a -> xs:[a] -> {ys:[a] | len ys = len xs + 1}
```

From that we can extrapolate that

```haskell
len (x:xs) = len xs + 1
```

and also that

```haskell
len (f x : map f xs) = len (map f xs) + 1
```

Thus, we know

```haskell
len (x:xs) = len (f x : map f xs)
```

if and only if

```haskell
len xs = len (map f xs)
```

We can then proceed *by induction* - while proving `map`, we also have the
refined type of `map` available to us! This is sound as long as `map` is a
terminating function (I'll get to termination in a bit.) By induction, we know

```haskell
len (map f xs) = len xs
```

Which allows us to prove the last requirement, and thus `map` is safe.

### Proving termination with Liquid Haskell

Liquid Haskell will additionally try to prove termination for your functions by
checking for recursive functions and adding constraints requiring that any
recursion be *well-founded*. For `map`, this translates into the requirement
that in the recursive call to `map f xs`, `xs` be strictly smaller than the
input list given to `map`, which is `x:xs` - clearly larger than `xs`.

## More expressive refinement types: typing `filter` with bounded, abstract refinements

Things get tricky when we try to express more complex types with Liquid
Haskell. Consider, for example, the type of the `filter` function, defined as
follows:

```haskell
filter :: (a -> Bool) -> [a] -> [a]
filter p (x:xs) | p x       = x : filter p xs
filter p (_:xs) | otherwise = filter p xs
filter _ []                 = []
```

Naively, one might refine the type of `filter` as:

```haskell
filter :: (a -> Bool) -> xs:[a] -> {ys:[a] | len ys <= len xs}
```

However, this leaves out some crucial information. Specifically, the fact that
`ys` has been filtered through the passed-in predicate. A more clever
refinement expresses the fact that all elements in `ys` satisfy the predicate,
as well as any constraints which elements of `xs` satisfy. This can be
expressed through the following scarily complex refinement type:

```haskell
filter :: forall <p :: a -> Bool, w :: a -> Bool -> Bool>.
              {y :: a, b :: {v:Bool<w y> | v} |- {v:a | v == y} <: a<p>}
              (x:a -> Bool<w x>) -> xs:[a] -> [a<p>]
```

Let's break this down. The first thing to note is the odd universal
quantification which starts the type; this is quantification *over refinement
predicates.* We have two logical predicates we're quantifying over; the first
predicates over values of type `a`, the type of the elements in the list we're
filtering. Then, the second quantifies over the combination of a list element
and a boolean. Let's try to decode what exactly these predicates are. The first
thing to notice is where the `w` predicate is mentioned - in the type of the
predicate function passed to filter:

```haskell
x:a -> Bool<w x>
```

This type - given to the predicate passed into the Haskell filter - says that
all booleans returned from the predicate will satisfy `w`, where `w` also knows
about the argument to the predicate. Here, the notation `Bool<w x>` is
equivalent to the notation `{v:Bool | w x v}`. So `w` somehow qualifies the
output of the Haskell predicate; what about `p`?

The first thing to note is that `p` shows up first of all (and arguably most
importantly) in the return type of the refined `filter` type:

```haskell
(x:a -> Bool<w x>) -> xs:[a] -> [a<p>]
```

From this we can see that `p` is the predicate qualifying over elements of the
output list. And now that we know that `w` qualifies the predicate and `p`
qualifies the outputs, we can inspect the component which connects the two:

```haskell
{y :: a, b :: {v:Bool<w y> | v} |- {v:a | v == y} <: a<p>}
```

This is notation soup. It can be carefully read as a logical implication,
specifying that:

For all `y` of type `a`, and `b` of type `Bool`;

**if** `b` and `w y b`, then the refinement type `{v:a | v == y}` is a subtype
of `a<p>`.

This can be broken down even further. We can note that a subtyping query in
Liquid Haskell is actually an implication over the refinement predicates on the
Haskell types; thus, we can read the consequent (the implied part) as:

`v == y` implies `p v`.

And then we can substitute `y` in for `v` to get `p y`. So we can now read the
statement as:

For all `y` of type `a` and `b` of type `Bool`; **if** `b` and `w y b`, then `p y`.

And finally we can rewrite as a series of implications:

For all `y` of type `a` and `b` of type `Bool`, `b` implies that `w y b`
implies `p y`.

### Why are bounded refinements necessary?

Liquid Haskell's refinement types belong to what we call a *decidable* type
system - one for which typechecking always terminates, returning a "yes" or a
"no" to the question of "is this program safe?". In order to preserve
decidability, Liquid Haskell uses as its base a decidable logic for its
refinements' predicate logic. Unfortunately, decidable logics are inherently
very inflexible; [first-order logics, in which quantifiers are allowed in any
position within logical expressions, are
undecidable](http://kilby.stanford.edu/~rvg/154/handouts/fol.html) and to
preserve decidability we must restrict ourselves to logics which are
"quantifier-free". Logics that are powerful enough to quantify over predicates
are called "second-order", logics that can quantify over predicates over
predicates are "third-order", and so on; and a logic which is powerful enough
to contain statements from any nth-order logic is called a "higher-order"
logic.

Suppose we return to the problem of filter, and consider what we're
really doing with bounded refinements: we're *accepting an argument to our
refinement which is itself a predicate.* In other words, we're looking at a
statement which appears inherently second-order; however, Liquid Haskell is
able to play some tricks and rewrite these second-order statements into its
quantifier-free logic, which is what makes bounded refinements so powerful.

## Verifying Haskell's `base` libraries with Liquid Haskell

As I mentioned before, my project this summer was to verify `base` with Liquid
Haskell by adding refinement type signatures in comments. Unfortunately, we
were unable to verify the whole of base, but we did find a number of
interesting bugs in Liquid Haskell, beginning with issues specific to special
privileges that `base` has in GHC and ending with a serious bug in Liquid
Haskell's handling of recursive functions. In the process, I found one
particular puzzle which involves no bugs, but fascinates me nonetheless - a
brief puzzle about a function in `base` called `the`.

## `GHC.Exts.the`

The `the` function takes a nonempty list, verifies that all elements of the
list are equal, and then returns the head of the list. It's defined as:

```haskell
the :: (Eq a) => [a] -> a
the (x:xs)
  | all (x ==) xs = x
  | otherwise     = errorWithoutStackTrace "GHC.Exts.the: non-identical elements"
the []            = errorWithoutStackTrace "GHC.Exts.the: empty list"
```

This provides for a small but not uninteresting refined type:

```haskell
the :: (Eq a) => {xs:[{x:a | x = head xs}] | len xs > 0} -> {x:a | x = head xs}
```

We indicate that all elements of the list are equal to the head of the list,
that the list has a length greater than zero, and that the return value is
equal to the head of the list.

The problem with verifying this is that `all` is a generic function defined
over instances of the `Foldable` typeclass. The type signature for `all` looks
like this:

```haskell
all :: (Foldable t) => (a -> Bool) -> t a -> Bool
```

Unfortunately, Liquid Haskell does not have any simple way to reason
inductively about the elements of a `Foldable` type. Thus one way to rewrite
this function is as follows:

```haskell
the :: (Eq a) => [a] -> a
the (x:xs)
  | null (filter (x /=) xs) = x
  | otherwise               = errorWithoutStackTrace "GHC.Exts.the: non-identical elements"
the []                      = errorWithoutStackTrace "GHC.Exts.the: empty list"
```

I chose (with the advice of my mentors) to rewrite it this way in an attempt to
keep modification of the code in `base` to a minimum. In any case, this is a
logically correct reimplementation of `the` in terms of `null` and `filter`
instead of `all`, which also shouldn't have any significant performance impact.

But Liquid Haskell cannot verify the refinement type we showed above. [Here's a
link to the online Liquid Haskell demo showing
this.](http://goto.ucsd.edu:8090/index.html#?demo=permalink%2F1504126167_3372.hs)

Why is this? While talking to my mentors, we found several possible reasons,
which we eliminated in quick succession:

1. `filter` is incorrectly typed, and Liquid Haskell isn't able to infer the necessary information from the call to it.
2. `null` is incorrectly typed, and although `filter` is working correctly, `null` is unable to use the information from `filter`.
3. Something weirder is going on with Liquid Haskell.

It turns out the problem was actually with `null` - but not with the refinement
type of `null`, because we have no way to correctly express the necessary
logical statement. Here's the problem:

- Suppose that an element of `xs` isn't equal to `x`. Then we have an element `{y:a | y /= x }`.
- We know that all elements in `xs` are subtypes of `{y:a | y = x}`. Thus, `filter` returns a list of the type `{y:a | y = x && y /= x}`, which is a subtype of `{y:a | false}`!
- We expect `null` to be able to infer that a list of `false` elements - elements which cannot possibly exist - must be empty, and thus `null` must return true.

However, the last step requires inductive reasoning - we must look into the
list and check its first element to bring its proof of falsehood into scope. We
can rewrite `the` in a way that typechecks with Liquid Haskell as follows:

```haskell
{-@ the :: (Eq a) => {xs:[{x:a | x = head xs}] | len xs > 0} -> {x:a | x = head xs} @-}
the (x:xs) = case filter (x /=) xs of
  []    -> x
  (_:_) -> errorWithoutStackTrace "GHC.Exts.the: non-identical elements"
the []     = errorWithoutStackTrace "GHC.Exts.the: empty list"
```

Can we make this work with `null`? We can, by expressing a theorem which
relates the falsehood of a list's elements to its length:

```haskell
{-@ falseNullTheorem :: xs:[{a | false}] -> {ys:[a] | xs = ys && len ys = 0} @-}
falseNullTheorem []     = []
falseNullTheorem (x:xs) = (x:xs)
```

Why does this work? In the last case - the "cons" case - Liquid Haskell is able
to infer the bound of `len ys = 0` because `x` cannot exist and carries with it
a proof of `false`. I've left in the "cons" case here for clarity, but in
actuality Liquid Haskell is smart enough to figure out how to prove `false` for
that case on its own. Anyway, we can rewrite `the` like so:

```haskell
{-@ the :: (Eq a) => {xs:[{x:a | x = head xs}] | len xs > 0} -> {x:a | x = head xs} @-}
the (x:xs)
  | null (falseNullTheorem (filter (x /=) xs)) = x
  | otherwise                                  = errorWithoutStackTrace "GHC.Exts.the: non-identical elements"
the []                                         = errorWithoutStackTrace "GHC.Exts.the: empty list"
```

And Liquid Haskell will happily prove this "safe", as it is.

## `the` and the inherent limitations of Liquid Haskell

The main reason I find `the` so interesting is that it showcases one of the
current main limitations of Liquid Haskell: the inability to do automatic
inductive reasoning. This makes sense, as Liquid Haskell's internal reasoning
is strictly in terms of quantifier-free logic. It may be possible to do this in
the future, though, using something along the lines of bounded refinement
types; perhaps with some sort of bounded invariant on lists. In the mean time,
however, this will have to remain one of the more interesting limitations of
Liquid types.

### Laziness and Liquid Haskell

Another problem with Liquid Haskell is an odd disconnect between Liquid Haskell
and Haskell: Liquid Haskell treats all types it refines as if they are strictly
evaluated. In other words, Liquid Haskell would treat the expression:

```haskell
head [0, undefined]
```

...as unsafe, even though `undefined` will never be evaluated and thus this
expression cannot ever crash your program. This created problems while
attempting to verify functions in `GHC.List` which were optimized for laziness.

At current Liquid Haskell doesn't have a convenient way to deal with lazy
values. Values known to be lazy can be turned into explicit thunks at the
Haskell level by representing them as functions of the type `() -> a`, which
can then be refined with useful Liquid Haskell types; however, this requires
modifying the actual Haskell type of the expression, and as such is a somewhat
undesirable fix.

## Why you should Liquify your Haskell

All in all, working with Liquid Haskell this summer has cemented my opinion
that refinement types are the most accessible form of formal verification
available today. Refinement types are automatically verified, can be kept
decidable, and aside than writing specifications, require no interaction from
the user. Refinements can be gradually added to Haskell code, and are far more
powerful than they appear above: Liquid Haskell incorporates not just logic of
equality and reasoning about integers (equalities and inequalities) but also a
logic of sets! This allows for simple reasoning about things such as the set of
items inside an abstract data structure and a surprising level of flexibility.

If you're looking to add a bit more safety to your Haskell program, I strongly
recommend you [try out Liquid Haskell
today](https://github.com/ucsd-progsys/liquidhaskell).
