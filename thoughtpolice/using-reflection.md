# Reflecting values to types and back

_This week, on [/r/haskell](http://reddit.com/r/haskell)_:

"_I still want a `reflection` tutorial!_" - [Oliver Charles](http://www.reddit.com/r/haskell/comments/1l64oe/lensaeson_traversalprism/cbw5rmb)

# The configurations problem

Several years ago, Oleg Kiselyov and Chung-chieh Shan wrote a paper entitled "[Functional Pearl: Implicit configurations](http://www.cs.rutgers.edu/~ccshan/prepose/prepose.pdf)". It's actually a very fascinating paper and it's very readable too (I encourage tackling it if you're an intermediate Haskeller.) In the paper, they propose a solution the *configurations problem*:

> The conﬁgurations problem is to propagate run-time preferences
throughout a program, allowing multiple concurrent conﬁguration
sets to coexist safely under statically guaranteed separation...
> 
> In our approach, a term expression can refer to run-time
conﬁguration parameters as if they were compile-time constants
in global scope ... We can propagate any type of conﬁguration data -- numbers, strings, `IO` actions, polymorphic functions, closures, and abstract data types.

This may remind you of an extension our lovely GHC supports - `ImplicitParams` - which tries to make this sort of behavior lightweight. Unfortunately, `ImplicitParams` has a number of drawbacks, one being that they are 'infectuous' in areas like scoping (and they have other concerns too.) The solution in this case is to instead propogate values around *via types*, rather than the kind of dynamic binding which `ImplicitParams` gives us.

Well, our very own Edward Kmett of course packaged this up into a library, which he calls [reflection](http://hackage.haskell.org/package/reflection). This tutorial going to both show you *how* it works, and what it allows. **NB**: This writeup isn't really going to concern itself with Kiselyov's original presentation, and instead focuses on Eds from this point on. (On one hand, this implementation 'cheats' the premise by using unsafe features, where the original paper didn't - and as we will understand, makes our implementation significantly faster. On the other hand it's possibly the only Oleg paper I'm aware of which was reduced to, like, 7 lines of code.)

The `reflection` API can be boiled down to three very simple things:

```haskell
-- #1: A 'Proxy' value which we can tag with types.
data Proxy k = Proxy

-- #2: A class to 'reflect' values back into terms
class Reifies s a | s -> a where
  reflect :: proxy s -> a

-- #3: A function to reify values *into* types
-- 
-- NB: refiy's second argument has a rank2 type. The 's' type variable will
-- guarantee that two competing `Proxy s` values cannot unify with each other or be leaked.
reify :: forall a r. a -> (forall (s :: *). Reifies s a => Proxy s -> r) -> r
```

Here's an example of this:

```active haskell
module Main (main) where
import Data.Reflection

-- show
main :: IO ()
main = print example
  where
    example :: Int
    example = reify 10 $ \p -> (reflect p) + (reflect p)
-- /show
```

In this example, we reify the value 10 *over the enclosed lambda*. Inside the lambda, we can `reflect` the `p` value to get our `10 :: Int` back. But if we look at the type of `reify`, the lambda accepts a parameter of type `Proxy s`! And we never had any instances of `Reifies` for our types. So how does it know what value to return, given the `Proxy`? What's going on?

(Again, one important thing to note - although we won't mention it anymore - is the rank2 type of `reify`. This trick is similar to what we do to `ST`, to ensure we can't 'leak' a reference to a `Proxy s` and reuse it later.)

# A look under the hood

To understand what's going on, the easiest way to look at the source of `reify`, and see how it elaborates to GHC Core. As a warning, if you consider yourself a very purist Haskeller, the following definition is probably going to make you feel pretty uncomfortable.

Below is an implementation of the entire basic `reflection` API

```haskell
import Unsafe.Coerce

data Proxy k = Proxy

class Reifies s a | s -> a where
  reflect :: proxy s -> a

newtype Magic a r = Magic (forall (s :: *). Reifies s a => Proxy s -> r)

reify :: forall a r. a -> (forall (s :: *). Reifies s a => Proxy s -> r) -> r
reify a k = unsafeCoerce (Magic k :: Magic a r) (const a) Proxy
{-# INLINE reify #-}

```

Before we can fully understand this though, we need to understand what GHC does when faced with type classes during compilation.

## Type-class elaboration and the single-method dictionary

Before we continue, I'd recommend you go ahead and install [ghc-core](http://hackage.haskell.org/package/ghc-core) from Hackage, which will make life much easier. It makes the default output of Core much more readable (although we will clean it up even further with a few extra options.) Work along with the examples. Here, I'm using GHC 7.6.3, although the specifics haven't really changed much in the last few versions.

Consider the following program:

```haskell
module Main (main, test1, test2) where

main :: IO ()
main = return ()

class Foo a where
  bar :: a -> Int
  baz :: a -> Bool

instance Foo Int where
  bar x = x+1
  baz x = x==0

instance Foo Bool where
  bar True  = 0
  bar False = 1
  baz x     = x

lorem :: Foo a => a -> (Int, Bool)
lorem x = (bar x, baz x)
{-# NOINLINE lorem #-}

test1 = lorem (1::Int)
test2 = lorem True
```

When GHC compiles the definition of `lorem`, it needs to be able to dispatch calls to `bar` and `baz` depending on the argument type. In order to do this, we perform a process called *dictionary elaboration*, in which we essentially replace a *constraint* (like `Foo a`) with a compiler-generated dictionary, which contains the methods to call. So the compiler produces something like this:

```haskell
foo'Int'bar :: Int -> Int
foo'Int'bar x = x+1

foo'Int'baz :: Int -> Bool
foo'Int'baz x = x==0

foo'Bool'bar :: Bool -> Int
foo'Bool'bar True  = 0
foo'Bool'bar False = 1

foo'Bool'baz :: Bool -> Bool
foo'Bool'baz x     = x

data FooDict a =
  FooDict { fooDict'bar :: a -> Int
          , fooDict'baz :: a -> Bool
          }

lorem :: FooDict a -> a -> (Int, Bool)
lorem dict x = (fooDict'bar dict x, fooDict'baz dict x)

test1 = lorem (FooDict foo'Int'bar foo'Int'baz) 1
test2 = lorem (FooDict foo'Bool'bar foo'Bool'baz) True
```

As we can see, the type class is totally eliminated and instead replaced by a data type, containing our type class methods (note that `FooDict a` isomorphic to `(a -> Int, a -> Bool)`) The compiler then manually plumbs around the correct dictionaries at the call site of `lorem`.

Let's verify this using `ghc-core`. Stuff the above code into a file, and compile it like so:

```
ghc-core --no-asm --no-cast -- -dsuppress-var-kinds -dsuppress-type-applications -dsuppress-uniques Foo.hs
```

The arguments suppress a lot of the unnecessary output in this case, so the output looks slightly more like 'simple haskell' as opposed to 'compiler intermediate language.' But it's still pretty verbose.

We see output similar to the following:

```haskell
$fFooBool2 :: Int
$fFooBool2 = I# 1

lorem [InlPrag=NOINLINE]
  :: forall a. Foo a => a -> (Int, Bool)
lorem =
  \ (@ a) ($dFoo :: Foo a) (x :: a) ->
    (bar $dFoo x, baz $dFoo x)

test1 :: (Int, Bool)
test1 = lorem $fFooInt $fFooBool2

test2 :: (Int, Bool)
test2 = lorem $fFooBool True
```

(Quick tip: in System FC, the GHC Core language, *types* as well as values are explicit as arguments. The syntax `(@ a)` does not refer to a *value argument* - it instead says `lorem` is a function which takes a singular *type argument* called `a`, as well as two value arguments: a dictionary of type `Foo a` and a value of type `a`.)

So we see everything is as we expect. Both `test1` and `test2` pass a dictionary as the first argument to `lorem`, which dispatches on the `Foo a` dictionary to call the right functions. (Check the remainder of the output to see the rest, including how `$fFooInt` for example is actually a dictionary of `$fFooInt_$cbar` and `$fFooInt_$cbaz`)

But there is an important special case here: *if a type class only has one method, we simply pass around the method itself*. This is obvious - if a type class only contains a method `foo :: Bar => a -> Int`, there is no need to create a `data FooDict = FooDict (a -> Int)`, when instead we can just pass around the `(a -> Int)` on its own. Let's verify that with the following code:

```haskell
module Main (main, test1) where

main :: IO ()
main = return ()

class Foobar a where
  foobar :: a -> Int

instance Foobar Int where
  foobar x = x*x

ipsum :: Foobar a => a -> Int
ipsum a = (foobar a) + 10
{-# NOINLINE ipsum #-}

test1 = ipsum (10::Int)

```

Load it into `ghc-core`, and we see:

```haskell

$fFoobarInt_$cfoobar :: Int -> Int
$fFoobarInt_$cfoobar =
  \ (x :: Int) -> $fNumInt_$c* x x

ipsum :: forall a. Foobar a => a -> Int
ipsum =
  \ (@ a) ($dFoobar :: Foobar a) (a :: a) ->
    case ($dFoobar `cast` ...) a of _ { I# x ->
    I# (+# x 10)
    }

test1 :: Int
test1 =
  ipsum ($fFoobarInt_$cfoobar `cast` ...) test2


```

We see something different here: instead, `test1` passes *the function* `$fFoobarInt_$cfoobar` as an argument to `ipsum`, after coercing it using a `` `cast` `` expression. `ipsum` then uses it and applies it as a function to the argument `a` in the `case` expression. This shows we are correct: single-method 'dictionaries' for a type class are really just functions by themselves.

Side note: you may be wondering what the deal with `cast` is. It's a little too complex to go into here, but essentially, GHC Core has a notion of *coercion variables* which allow us to witness the equivalence *between two types*. Coercions allow us to establish equality between two types which are not syntactically equal - in the above case, we establish the fact that the dictionary type `Foobar Int` is *the same* as the type `Int -> Int`. If we made another instance, say `Foobar Bool`, and then somewhere else said `ipsum False`, we would get a coercion variable that establishes `Foobar Bool` as equivalent to `Bool -> Int`. `ipsum` also uses a coercion variable, more generally stating that `Foobar a` is equal to `a -> Int`. This is really about all you need to know.

While this trick is fairly obvious when you think about it, it is *critical* to the implementation of `reflection`, as we shall shortly see. (Maybe you already see where this is going...)

## From reify to reflect to nothing at all

We're about 80% of the way to understanding `reflection`. Only 80% left!

The core of it all is this:

```haskell
newtype Magic a r = Magic (forall (s :: *). Reifies s a => Proxy s -> r)

reify :: forall a r. a -> (forall (s :: *). Reifies s a => Proxy s -> r) -> r
reify a k = unsafeCoerce (Magic k :: Magic a r) (const a) Proxy
```

The first thing to note is that *a newtype has the same representation as its underlying type*. That means that in the above case, a value `Magic k` has the same runtime representation as `k` by itself. (GHC discharges such newtype-equivalences based on the same coercion variables we mentioned before. Newtypes are turned into coercions very early in the compilation process.)

Second, we must remember that *at the Core level, `k` takes two arguments, not one*. After elaboration, the first argument is the 'dictionary', but because `Reifies` only has one method, the first argument to `k` will be a function of type `proxy s -> a`. The second argument is of type `Proxy s`:

```
Given:

  k :: forall (s :: *). Reifies s a => Proxy s -> r
= { dictionary elaboration }
  k :: forall (s :: *). Reifies s a -> Proxy s -> r
= { Reifies is a single-method class; thus a dictionary is a fn }
  k :: forall (s :: *). (proxy s -> a) -> Proxy s -> r
```

So at runtime, `k` takes two arguments, not one. The trick here is that the *types* of `k` and `Magic k` are different at the Haskell level.

We add a 'layer' of `Magic` onto the type of `k`, and then `unsafeCoerce` it away. In doing so, we basically give `k` the final type above. By exploiting the fact `newtype` doesn't really exist at runtime, and `Reifies` actually becomes an argument, we can use `unsafeCoerce` to *expose the runtime representation of `k`*.

Remember: `unsafeCoerce` pays no attention to runtime representation. It just sidesteps the typechecker. In this case, however, we are coercing a value for which we are *certain* of the runtime representation. This lets us get away with 'slipping in' our own argument.

Let's go to our original example and step through it (with slight psuedo-syntax and hand-waving)

```
Given:

  reify 10 $ \p -> (reflect p) + (reflect p)
= { defn of reify }
  unsafeCoerce (Magic $ \p -> (reflect p) + (reflect p)) (const 10) Proxy
= { dictionary elaboration }
  unsafeCoerce (Magic $ \dReflect p -> (dReflect p) + (dReflect p)) (const 10) Proxy
= { unsafeCoerce Magic elimination }
  (\dReflect p -> (dReflect p) + (dReflect p)) (const 10) Proxy
= { beta reduction }
  (\p -> (const 10 p) + (const 10 p)) Proxy
= { beta reduction }
  (const 10 Proxy) + (const 10 Proxy)
= { defn of const }
  10 + 10
= { addition }
  20
```

More generally, we can use a similar hand-waving to establish the following: `reify a reflect = a`:

```
Given:

  reify a reflect
= { eta expand }
  reify a (\p -> reflect p)
= { defn of reify }
  unsafeCoerce (Magic $ \p -> reflect p) (const a) Proxy
= { dictionary elaboration }
  unsafeCoerce (Magic $ \dReflect p -> dReflect p) (const a) Proxy
= { unsafeCoerce Magic elimination }
  (\dReflect p -> dReflect p) (const a) Proxy
= { beta reduction }
  (\p -> const a p) Proxy
= { beta reduction }
  const a Proxy
= { defn of const }
  a
```

And that's about it! It should now also be clear why this implementation is so fast: it has almost no extra runtime overhead.

There are some important things to reiterate. By now, you should see why we never need an instance of `Reifies` for anything: because we create the dictionary ourselves, and use `const a` in its place. And thus `reflect p = const a p = a`, where `a` is the configuration you provided.

Second, it's important to note (as always!) that `unsafeCoerce`/`Magic` here really *don't* have a justifying rule to reason about them with. The above 'proof' is operational and a bit hand-wavy. You can't *really* apply many deductive reasoning steps like above to such operators. It all simply works because we `unsafeCoerce` a value with a guaranteed expectation of what it will look like at runtime. We could say that the 'unsafeCoerce Magic elimination' rule above is simply a truth to be assumed - that it will yield an appropriate function.

Third, you may still be wondering about the weird type `reflect :: proxy s -> a`. The reason for this will become clear, but in a nutshell, we do this because we want to unify the type variable `s` with the `s` at a call site, but we don't care about the actual `proxy` type itself - since `reflect` never actually uses its argument anyway.

# Dynamically constructing type-class instances

With our new power, it's possible to do something we couldn't before. We can create *dynamically create type class instances for a given type*. The trick is that we wrap our actual values into a `newtype`, and the instance of that `newtype` actually uses `reflect` to pull methods from the scope which `reify` creates. This allows you to pass in whatever dictionary you want to `reify`.

Here's an example. We'll create a type over which we can dynamically construct an `Ord` instance.

```active haskell
{-# LANGUAGE FlexibleContexts      #-}
{-# LANGUAGE MultiParamTypeClasses #-}
{-# LANGUAGE RankNTypes            #-}
{-# LANGUAGE UndecidableInstances  #-}
module Main where
import Data.Proxy
import Data.Reflection

-- show
-- Values of type 'a' in our dynamically constructed 'Ord' instance
newtype O a s  = O { runO :: a }
 
-- A dictionary describing an 'Ord' instance.
newtype Ord_ a = Ord_ { compare_ :: a -> a -> Ordering }
 
-- Utility
isEq :: Ordering -> Bool
isEq EQ = True
isEq _  = False
 
instance Reifies s (Ord_ a) => Eq (O a s) where
  a == b = isEq (compare_ (reflect a) (runO a) (runO b))
 
instance (Eq (O a s), Reifies s (Ord_ a)) => Ord (O a s) where
  compare a b = compare_ (reflect a) (runO a) (runO b)
 
-- Dynamically construct an 'Ord' instance out of a comparsion operator.
withOrd :: (a -> a -> Ordering) -> (forall s. Reifies s (Ord_ a) => O a s) -> a
withOrd f v = reify (Ord_ f) (runO . asProxyOf v)
  where
    asProxyOf :: f s -> Proxy s -> f s
    asProxyOf v _ = v

-- Regular ord instance
example1 :: Int
example1 = withOrd compare $ max (O 1) (O 2)

-- Backwards ord instance
example2 :: Int
example2 = withOrd (flip compare) $ max (O 1) (O 2)
 
main :: IO ()
main = print example1 >> print example2
-- /show
```

The above examples show we can compare these Ord-like values, but change out the actual dictionary at runtime!

Note in the instance declaration, we say `reflect a`, where `a :: O a s`. This goes back to what I said earlier about `reflect :: proxy s -> a` - we want the `s` type variables to unify, but we *don't* care that the outer type constructor is `O a :: * -> *`. In other words, `proxy` unifies with anything specifically because it's irrelevant as to what it is.

## But what about manual dictionaries?

The above might just seem like a parlor trick, although there is real value. Indeed - what about manual dictionaries, you ask yourself? We could feasibly just construct a type where the dictionary argument is *explicit* and we feed it in. Then we can shuffle it around in the associated type-class instance, and it works similarly to the API above. We construct a dictionary, and scope its use over some computation.

In the following example, `Mon1` is a monoid with a *manually managed* dictionary argument, while `Mon2` has its monoid instance managed by `reflection` instead.

```active haskell
{-# LANGUAGE FlexibleContexts      #-}
{-# LANGUAGE MultiParamTypeClasses #-}
{-# LANGUAGE RankNTypes            #-}
{-# LANGUAGE UndecidableInstances  #-}
module Main where
import Data.Proxy
import Data.Monoid
import Data.Reflection

-- show
-- Dynamically constructed monoid values.
data Monoid_ a = Monoid_ { mappend_ :: a -> a -> a, mempty_ :: a }

-- Monoids which explicitly hold their own dictionary
newtype Mon1 a = Mon1 (Monoid_ a -> a)
instance Monoid (Mon1 a) where
  mempty                  = Mon1 mempty_
  Mon1 f `mappend` Mon1 g = Mon1 $ \m -> mappend_ m (f m) (g m)
 
embed1 :: a -> Mon1 a
embed1 x = Mon1 (\_ -> x)
 
run1 :: (a -> a -> a) -> a -> Mon1 a -> a
run1 f z (Mon1 v) = v (Monoid_ f z)

-- Monoids with a reflection-managed dictionary.
newtype Mon2 a s = Mon2 { unMon2 :: a }
instance Reifies s (Monoid_ a) => Monoid (Mon2 a s) where
  mappend a b        = Mon2 $ mappend_ (reflect a) (unMon2 a) (unMon2 b)
  mempty = a where a = Mon2 $ mempty_ (reflect a)
 
embed2 :: a -> Mon2 a s
embed2 x = Mon2 x
 
run2 :: (a -> a -> a) -> a -> (forall s. Reifies s (Monoid_ a) => Mon2 a s) -> a
run2 f z v = reify (Monoid_ f z) (unMon2 . asProxyOf v)
  where
    asProxyOf :: f s -> Proxy s -> f s
    asProxyOf v _ = v
 
-- Examples
sixteen :: Monoid m => m -> m
sixteen one = eight <> eight
  where
    eight   = four <> four
    four    = two  <> two
    two     = one  <> one
{-# INLINE sixteen #-}
 
ex1 :: Int
ex1 = run1 (+) 0 (sixteen $ embed1 2)
 
ex2 :: Int
ex2 = run2 (+) 0 (sixteen $ embed2 2)

main :: IO ()
main = print ex1 >> print ex2
-- /show
```

The problem with this, though, is sharing. There is a huge difference between `ex1` and `ex2`.

Note the embedding of values into their monoid counterparts:

```haskell
embed1 :: a -> Mon1 a
embed1 x = Mon1 (\_ -> x)

embed2 :: a -> Mon2 a s
embed2 x = Mon2 x
```

Given these definitions, the definitions of `ex1` and `ex2` have very different runtime characteristics. In `ex1`, we *share the lambdas* in our computation, not the results. `ex2` however, does properly share the results.

We can confirm this with a quick look at the Core. `ex1`:

```haskell
ex1_x :: Int
ex1_x = I# 2

$wa1
  :: (Int -> Int -> Int)
     -> Int
$wa1 =
  \ (ww :: Int -> Int -> Int) ->
    ww
      (ww (ww ex1_x ex1_x) (ww ex1_x ex1_x))
      (ww (ww ex1_x ex1_x) (ww ex1_x ex1_x))

ex1 :: Int
ex1 =
  case $wa1 $fNumInt_$c+ of _ { I# x ->
  I# (+# x x)
  }

```

vs `ex2`:

```haskell
ex2_z :: Int
ex2_z = I# 0

ex2_x :: Monoid_ Int
ex2_x = Monoid_ $fNumInt_$c+ ex2_z

ex3 :: Any * -> Monoid_ Int
ex3 = \ _ -> ex2_x

ex2 :: Int
ex2 =
  ((\ (@ s)
      ($dReifies
         :: Data.Reflection.Reifies * s (Monoid_ Int)) ->
      let {
        eight [Dmd=Just L] :: Mon2 Int s

        eight =
          let {
            four [Dmd=Just L] :: Mon2 Int s

            four =
              let {
                two [Dmd=Just L] :: Mon2 Int s

                two =
                  case ($dReifies `cast` ...) (ex1_x `cast` ...)
                  of _ { Monoid_ ds _ ->
                  (ds ex1_x ex1_x) `cast` ...
                  } } in
              case ($dReifies `cast` ...) two of _ { Monoid_ ds _ ->
              (ds (two `cast` ...) (two `cast` ...)) `cast` ...
              } } in
          case ($dReifies `cast` ...) four of _ { Monoid_ ds _ ->
          (ds (four `cast` ...) (four `cast` ...)) `cast` ...
          } } in
      case ($dReifies `cast` ...) eight of _ { Monoid_ ds _ ->
      let {
        a1 [Dmd=Just S] :: Int

        a1 = ds (eight `cast` ...) (eight `cast` ...) } in
      \ _ -> a1
      })
   `cast` ...)
    ex3 (Data.Proxy.Proxy)
```

None of the work is shared in `ex1`, but all of it is in `ex2`.

And looking at `embed1`, this is rather obvious: we must recompute the lambda for every 'step'. This means that while `ex1` takes 16 reductions (one for every instance of `one`), `ex2` only takes 4 reductions thanks to the fact the results are shared. This sharing loss thus explodes pretty quickly.

(Incidentally, this exact problem is the reason Edward wrote `reflection` in the first place.)

# Turning up the magic to over 9000

Finally, we're going to do something even scarier. With our above examples of creating type class instances, one drawback is we have to manually 'annotate' our values by wrapping them in a `newtype`. The newtype with the phantom `s` parameter is what A) keeps instance resolution happy, and B) lets us have a modicum of safety by not leaking `Proxy` values. But that's kind of inconvenient - is there another way?

Of course there is. By the power of `ConstraintKinds`, `TypeFamilies` and `unsafeCoerce`, what could possibly stop us?

First, we'll build some machinery to help abstract out the common patterns of creating these dynamic instances. This will rely on Edward's other package known as [constraints](http://hackage.haskell.org/package/constraints). This helps reduce boilerplate and makes usage more consistent. If you're not already familiar with `constraints`, [I have a blog post about the general mechanism](http://neocontra.blogspot.com/2013/06/controlcategory-now-with-kind.html), concerning `:-` (called `|-` in my article) and `Sub`/`Dict`.

Let us first define `Lift`, which essentially tags a value of type `a` with an `s` phantom type (for the rank2 context,) and *also* a phantom for a *`Constraint` constructor*.

```haskell
newtype Lift (p :: * -> Constraint) (a :: *) (s :: *) = Lift { lower :: a }
```

`Lift p a s` describes a value of type `a`, with associated phantoms `s` and `p`. `p` in this case, is some type constructor which will produce a `Constraint` when given a type. We'll be applying `p` to `a` to yield a `Constraint` as we'll shortly see.

Now, we can define a type class which will tie a given class `Constraint` *to a representation of its dictionary*. We do this using `TypeFamilies`: for any given instance of the type class, we define a data constructor representing it.

```haskell
class ReifiableConstraint p where
  data Def (p :: * -> Constraint) (a :: *) :: *
  reifiedIns :: Reifies s (Def p a) :- p (Lift p a s)
```

We'll get back to `reifiedIns` in a minute.

`ReifiableConstraint` makes more sense when you see it being used:

```haskell
instance ReifiableConstraint Monoid where
  data Def Monoid a = Monoid { mappend_ :: a -> a -> a, mempty_ :: a }
  reifiedIns = Sub Dict
```

This means that we represent the `Monoid` constraint (a type constructor of kind `* -> Constraint`) as a dictionary, containing its `mappend` and `mempty` methods, like earlier. Hence, `ReifiableConstraint` associates a given `Constraint` with its 'reified form' as a value. Now we can tie `Def Monoid a` and `Lift Monoid a s` into an `Monoid` instance:


```haskell
instance Reifies s (Def Monoid a) => Monoid (Lift Monoid a s) where
  mappend a b        = Lift $ mappend_ (reflect a) (lower a) (lower b)
  mempty = a where a = Lift $ mempty_ (reflect a)
```

Finally, we define the general `with` utility. This can be used to create dynamic instances *for any `Lift`ed value which is tied to a `ReifiableConstraint`*.

```haskell
with :: Def p a -> (forall s. Reifies s (Def p a) => Lift p a s) -> a
with d v = reify d (lower . asProxyOf v)
  where
    asProxyOf :: f s -> Proxy s -> f s
    asProxyOf x _ = x
```

And now we can say:


```haskell
with (Monoid (+) 0) $ mempty <> Lift 10 <> Lift 12 -- evaluates to 22
with (Monoid (*) 1) $ mempty <> Lift 10 <> Lift 12 -- evaluates to 120
```

If we define instances for `Eq` and `Ord`, both `with` and `Lift` will work perfectly for them, too. If we put all the types together, we can sort of see how this plays out: given a `Lift p a s` value, over some constraint `p`, we derive the appropriate `Reifies` context with the associated `Def` value, and in the instance itself, we pull the methods from `Def` and use them, like we did earlier. This is mostly just generalizing out all of the type variables/type constructors we specialized initially.

So that's part one. We have some general machinery to ease implementing these special dictionaries and use them generically. Now we get to the scarier part, and we come back to `reifiedIns`. Its type is very important. It says:

> The type `Reifies s (Def p a)` entails `p (Lift p a s)`

So, in any context where `p (Lift p a s)` occurs as a `Constraint`, it is safe to replace it with the constraint `Reifies s (Def p a)`. This is because the latter constraint *implies* the former constraint. However, `Lift p a s` is really just a wrapper around a value of type `a`. So in a sense, `p a` should be able to imply `p (Lift p a s)` too. By the property of transitivity, this would mean the simple context `p a` could then imply `Reifies s (Def p a)`.

Using some unsafety, we can create this bridge:

```haskell
using :: forall p a. ReifiableConstraint p => Def p a -> (p a => a) -> a
using d m = reify d $ \(_ :: Proxy s) ->
  let replaceProof :: Reifies s (Def p a) :- p a
      replaceProof = trans proof reifiedIns
        where proof = unsafeCoerceConstraint :: p (Lift p a s) :- p a
  in m \\ replaceProof
```

Note carefully the definition here. It basically says, in step:

  * `reifiedIns` is a proof that the constraint `p (Lift p a s)` can be replaced by `Reifies s (Def p a)`
  * `proof` is an *unsafe proof* saying that the constraint `p a` can be replaced by `p (Lift p a s)`
  * By transitivity of these two proofs, we have proof `p a` may be replaced by `Reifies s (Def p a)`
  * Given the term `m` of type `a` appears in the context `p a`, and a proof that `Reifies s (Def p a) :- p a`, we can replace the context `p a => a` with `Reifies s (Def p a) => a`, which is exactly the type `reify` expects. Note we directly consume the `Proxy` with `ScopedTypeVariables` to bring the `s` type parameter into scope, so we can unify with it.

And with that magic, we can now say:

```haskell
using (Monoid (+) 0) $ mempty <> 10 <> 12 -- evaluates to 22
using (Monoid (*) 1) $ mempty <> 10 <> 12 -- evaluates to 120
```

# Conclusion

The code from the last section is below for completeness.

Many thanks to Edward for the discussions we had about these libraries, and the tips he gave me when approaching `reflection` from a pedagogical standpoint. And, of course, for writing the libraries themselves.

```active haskell
-- show
{-# LANGUAGE Rank2Types, FlexibleContexts, UndecidableInstances, TypeFamilies #-}
{-# LANGUAGE ConstraintKinds, KindSignatures, PolyKinds, TypeOperators #-}
{-# LANGUAGE FlexibleInstances, MultiParamTypeClasses, ScopedTypeVariables #-}
{-# LANGUAGE FunctionalDependencies #-}
module Main where
import Data.Proxy
import Data.Monoid
import Data.Reflection
import Data.Constraint
import Data.Constraint.Unsafe

--------------------------------------------------------------------------------
-- Lift/ReifiableConstraint machinery.
 
newtype Lift (p :: * -> Constraint) (a :: *) (s :: *) = Lift { lower :: a }
 
class ReifiableConstraint p where
  data Def (p :: * -> Constraint) (a :: *) :: *
  reifiedIns :: Reifies s (Def p a) :- p (Lift p a s)
 
with :: Def p a -> (forall s. Reifies s (Def p a) => Lift p a s) -> a
with d v = reify d (lower . asProxyOf v)
  where
    asProxyOf :: f s -> Proxy s -> f s
    asProxyOf x _ = x

--------------------------------------------------------------------------------
-- Kicking it up to over 9000

using :: forall p a. ReifiableConstraint p => Def p a -> (p a => a) -> a
using d m = reify d $ \(_ :: Proxy s) ->
  let replaceProof :: Reifies s (Def p a) :- p a
      replaceProof = trans proof reifiedIns
        where proof = unsafeCoerceConstraint :: p (Lift p a s) :- p a
  in m \\ replaceProof
 
--------------------------------------------------------------------------------
-- Examples of `ReifiableConstraint`

instance ReifiableConstraint Eq where
  data Def Eq a = Eq { eq_ :: a -> a -> Bool }
  reifiedIns = Sub Dict
 
instance Reifies s (Def Eq a) => Eq (Lift Eq a s) where
  a == b = eq_ (reflect a) (lower a) (lower b)
 
instance ReifiableConstraint Ord where
  data Def Ord a = Ord { compare_ :: a -> a -> Ordering }
  reifiedIns = Sub Dict
 
instance Reifies s (Def Ord a) => Eq (Lift Ord a s) where
  a == b = (compare a b == EQ)
 
instance Reifies s (Def Ord a) => Ord (Lift Ord a s) where
  compare a b = compare_ (reflect a) (lower a) (lower b)
 
instance ReifiableConstraint Monoid where
  data Def Monoid a = Monoid { mappend_ :: a -> a -> a, mempty_ :: a }
  reifiedIns = Sub Dict
 
instance Reifies s (Def Monoid a) => Monoid (Lift Monoid a s) where
  mappend a b        = Lift $ mappend_ (reflect a) (lower a) (lower b)
  mempty = a where a = Lift $ mempty_ (reflect a)
-- /show

main :: IO ()
main = return ()
```