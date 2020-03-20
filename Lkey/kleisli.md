# Kleisli

#Monads

##Motivation/Meta

It seems to me that there is a lot of conflation of the following concepts

* what monads *are*
* what monads can do
* how a particular monad behaves. e.g. "like a box!"

The goal here is to explain only what makes a monad, in brief,
to a person who understands the following things about Haskell.

* function composition
* fmap
* data types

While introducing the concept of `join`

## A refresher

### Types and composition

We have two functions `f` and `g`.
Let's focus on their types.

```haskell
f :: (a -> b)
g :: (b -> c)
```

If we wish to compose our function we use `(.)`

```haskell
(.) :: (b -> c) -> (a -> b) -> (a -> c)

g . f :: (a -> c)
```

Composition with `(.)` happens *right to left*, which makes the types read backwards.
In practice getting used to this order is easy, but since we want to make type chaining explict in our examples, let's define *left to right* composition so the relationships later read more clearly.

```haskell
(|>) :: (a -> b) -> (b -> c) -> (a -> c)
(|>) = flip (.)

f |> g :: (a -> c)

-- now we can write
     f :: a -> b
     g ::      b -> c
f |> g :: a ->      c

```

Composition with `(|>)` has some simple, yet important characteristics.

First off, if we compose any other function with the identity function `id` the result is an unchanged function.
The same kind of thing that happens when you add 0 to a number, or multilpy a number by 1.

```haskell
f :: (a -> b)
id :: forall α.(α -> α)
id x = x                -- the definition is boring.

      f :: a -> b
     id ::      b -> b  -- α matching b
f |> id :: a ->      b

     id :: a -> a       -- α matching a
      f ::      a -> b
id |> f :: a ->      b

```

Next, we have the rules for chaining more than 2 compositions. The types work the way that (I hope) you expect.

```haskell
f :: a -> b
g ::      b -> c
h ::           c -> d

f |> g |> h :: a -> d
```

### Laws for values
Our types line up nicely, but at the value level we need a few constraints.
```haskell

-- composition with identity should change nothing at the type *or* value level.
 f |> id == f -- right identity
id |> f  == f -- left identity

-- parenthesis should never change the result of well defined composition.
f |> (g |> h) == (f |> g) |> h -- associativity

-- and since parens don't matter, we can write:
f |> g |> h
-- like we did above without violating any rules.
```

Fairly reasonable huh?

You probably already have an intuition if you have been writing haskell for while.

## A new kind of function

*If you skipped the refresher, then just know that are using left to right function composition `|>` below for illustration.

### A goal
Now let's introduce an wrinkle. A new kind of function that looks like this `a -> m b`.

I am going to call these functions **karrows**.

We are going to figure out how to compose **karrows** in a way that mirrors how we compose ordinary functions.
Keeping with the theme, let's call the "composition of **karrows**" **komposition**.

You can think of `m` as a stand in for `[b]`, or `Maybe b`, or what any useful Haskell data type you are familiar with.
We leave it as the more general `m` because we want the most general definition of **komposition**.

What did I mean by "mirroring composition"? Something that looks like this:

```haskell
f :: a -> m b
g :: b -> m c

f !> g :: a -> m c -- Komposition of f and g
```

If we try to use normal composition to define **komoposition**, `f` will feed its `m b` to `g` and `g` will promptly choke on it and refuse to compile. The types simply will not line up.

```haskell
f :: a -> m b
g ::      b -> m c -- blech, an m?!
```

Since `g` must operate on a `b`, we must devise a way to get the `g` to `b`.

The simplest answer would be to put `g` *inside* the `m`... and that is exactly what we are going to do.

So...

Law 1:
I declare that `m` must be *get insidable*. 

Let's call the key we use to get inside something `fmap`.

So are we done? Is requiring `m` to be a *get insidable* enough to define **komposition**?

Let's look again.

### Getting inside

```haskell
f :: a -> m b
g :: b -> m c

f :: a -> m b
g ::        b -> m c -- using fmap here
--    but ^ stays so our result m (m c)

-- a becomes m b 
-- b becomes m c
-- so the result becomes m (m c)

```
### Smashing doubles
So it is clear that getting inside only gets us part of the way there.

Now we have to contend with the double `m`s.

So let's declare another law to fix the problem.

Law 2:
I declare that `m` must not only be a *get insidable*, it must also be *Flattenable*.

I declare that whenever I find an `m (m c)`, I must be able to smash it down to an `m c`.

So, with (type-level) hammer in hand, we begin the end of our journey.

```haskell
f :: a -> m b
g ::        b -> m c
--            m (m c) -- as before
join ::       m (m c) -> m c

-- We can now proudly say that our newly minted
-- (!>) operation **Komposition** is complete, all we have to do is fit the pieces together
```

### Laws for values
At this point you might be thinking that everything that we declared to get what we want was a little too arbitrary, and it is.

We need a few rules to force what we have done to be really useful, like we did before with funciton composition.

Recall what we did with `id` in the refresher section.
What is `id` for **karrows**?
```haskell
id :: forall α.(α -> α)
id x = x
return::forall α.(α -> m α)
return x = ? -- depends on m I'm afraid, we will touch on this later.


-- Even though we cannot define a general `return` for `m`,
-- when it *is* properly defined it must follow rules like before.

f !> return == f  -- right identity
return !> f == f  --  left identity
```

Can we chain **komopositions**?

Sure we can, but the same caveats apply as before

```haskell
-- as far as the types are concerned
f :: a -> m b
g ::        b -> m c -- (join1)
h ::               c -> m d -- (join2)

-- and we can get
f !> g !> h :: a -> m d

-- but like before, the value level has a rule
f !> (g !> h) == (f !> g) !> h

-- and once again, because parens don't affect the outcome we get to write
f !> g !> h
-- without worrying
```

#### In Review
```haskell
-- laws for function composition
f |> id == f
id |> f == f
f |> (g |> h) == (f |> g) |> h

-- laws for karrow composition
f !> return == f
return !> f == f
f !> (g !> h) == (f !> g) !> h

```

#### komposition's form
Every `m`, if we are going to compose it, must have its own `return`, `join`, and `fmap`, 
and we can use them to desribe how we might actually make `(!>)`.

```haskell
f :: (a -> m b)
g :: (b -> m c)
(!>) :: (a -> m b) -> (b -> m c) -> (a -> m c)
f !> g = ?                 -- 
       = ? f               -- start with f
       = ? g . f           -- need to result of f into g
       = ? fmap g . f      -- types don't line up, so we use fmap
       = join . fmap g . f -- need to flatten the result
                           -- done
       
       -- switching things around for completeness
       = ?                   -- 
       = f ?                 -- f should be first
       = f |> g ?            -- need to result of f into g
       = f |> fmap g ?       -- types don't line up, so we use fmap
       = f |> fmap g |> join -- need to flatten the result
                             -- done

```

## What now?

What have we gained from this exercise? 

Well, assuming you understood the constraints required to make `m` **komposable**, and the laws for well-defined **komposition**, then you know what monads *are*.
`m` will always be a monad whenever these constraints are satisfied.

What you don't know is what particular monads look like, feel like to use, or the power that they give you to abstract. but you have all the tools you need to verify that something is indeed a monad. A very short checklist.



































