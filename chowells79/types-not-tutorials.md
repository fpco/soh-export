# Programming With Types, Not Tutorials

# Introduction

I see a lot of people complain about the state of Haskell documentation.
Some of the complaints are reasonable.
After all, it took me a very long time to understand data families.
The documentation on the extension that enables them doesn't really say much about them.
But there are persistently recurring complaints that everything is too hard to understand because of all the math and the lack of tutorials.

Those complaints make very little sense to me.
The great thing about Haskell is that for many purposes, you don't need to understand the math or have a tutorial to use a new library.
You just need to be able to read type signatures.

# A Missing Library

I wanted to use this site to write up the story of my discovery of what data families can do.
I was excited about finding a use for a language feature I hadn't previously understood.
Data families fill an expressiveness gap in Haskell in a way I appreciate.
But I quickly ran into an obstacle.
The motivating example for my story was using an [operational](http://apfelmus.nfshost.com/articles/operational-monad.html) library to build a DSL with many different interpreters.
But while there are two different operational libraries available on Hackage, neither one is available here.

I decided to assemble a minimal library of my own to embed in my write-up.
I have some experience with the libraries on Hackage and I find both to be more complex than I really need or want.
I decided to use the approach from [`free-operational`](http://hackage.haskell.org/package/free-operational) to implement my version, because it leans more heavily on third-party libraries.
My hope was that those dependencies would reduce the amount of code I'd have to embed.

The hitch in my plan was that I barely knew anything at all about the dependencies.
I have the very slightest understanding of the [`free`](http://hackage.haskell.org/package/free) package.
I know that it's for building a monad out of a functor.
Somehow.
That's the end of what I know about `free`.
The other dependency, the [`kan-extensions`](http://hackage.haskell.org/package/kan-extensions) package?
I don't know anything about it.

I know they're both based on some heavy-duty math.
I don't care.
I'm not scared of math, but I haven't dug into category theory much, which is where both of those packages originate.
That's because so far, I haven't needed to.

# How To Not Math

I started by drawing inspiration from `free-operational` and sketching out my target API.

``` haskell
{-# LANGUAGE RankNTypes #-}

import Control.Applicative

data Program instr a
instance Functor (Program instr)
instance Applicative (Program instr)
instance Monad (Program instr)

singleton :: instr a -> Program instr a

interpret :: Monad m => (forall x. instr x -> m x) -> Program instr a -> m a
```

The sketch consists of one data type, three instances for that type, and two functions.
That's nice and small.

The documentation for `free-operational` suggested that the representation for `Program` is `newtype Program instr a = Program (toFree :: Free (Coyoneda instr) a)`.
Like I said, I vaguely know what [`Free`](http://hackage.haskell.org/package/free-4.5/docs/Control-Monad-Free.html#t:Free) is.
Don't ask me what a [`Coyoneda`](http://hackage.haskell.org/package/kan-extensions-4.0.1/docs/Data-Functor-Coyoneda.html#t:Coyoneda) is.
I don't know.
I don't care, at least for the purposes of writing a library.
But I'm not scared of math.
I'm happy to use its results when they work for me, even when I don't understand the underlying theory.

One thing I do know is that the documentation for `Free` suggests that [`F`](http://hackage.haskell.org/package/free-4.5/docs/Control-Monad-Free-Church.html#t:F) is a better choice of data type unless you plan on inspecting the tree built up by the `Monad` instance.
There's nothing in my target API that inspects the tree, so I decided to use `F` instead of `Free`.

``` haskell
{-# LANGUAGE RankNTypes, GeneralizedNewtypeDeriving #-}

import Control.Applicative

import Control.Monad.Free.Church
import Data.Functor.Coyoneda

newtype Program instr a = Program (F (Coyoneda instr) a)
                          deriving (Functor, Applicative, Monad)

singleton :: instr a -> Program instr a

interpret :: Monad m => (forall x. instr x -> m x) -> Program instr a -> m a
```

Thanks to `GeneralizedNewtypeDeriving`, I got the three instances I wanted for free via `F`.
I still don't know anything about `F`.
I just know that the docs say those instances exist, so I get to use them.

The rest of the work is writing the two functions in the API.
`singleton` is the easier of them, judging by the type signature, so it's what I worked on next.

`Program` is a `newtype` wrapper, so `singleton` starts by applying the wrapper.

``` haskell
singleton = Program . undefined
```

That `undefined` needed to have the type `instr a -> F (Coyoneda instr) a`.
This made the mystery how to create an `F`.
I looked at the documentation to see what functions return an `F`.
The first thing I found in [the docs](http://hackage.haskell.org/package/free-4.5/docs/Control-Monad-Free-Church.html) was [`toF`](http://hackage.haskell.org/package/free-4.5/docs/Control-Monad-Free-Church.html#v:toF).
The full type signature makes it clear this is for converting from `Free` to `F`, which is not what I wanted.

That's the only thing that returns an `F` value explicitly, so I had to look a little harder.
The next thing to do was look for something that *could* return `F`, and didn't take an `F` as an argument.
There's only one thing I found that fit that pattern, [`liftF`](http://hackage.haskell.org/package/free-4.5/docs/Control-Monad-Free-Church.html#v:liftF).

Before I was sure I had what I wanted in `liftF`, I had to make the type of `liftF` work.
I started with `(Functor f, MonadFree m f) => f a -> m a`.
For my purposes, `m` has to be `F (Coyoneda instr)`.
I checked to see if [`MonadFree`](http://hackage.haskell.org/package/free-4.5/docs/Control-Monad-Free-Church.html#t:MonadFree) has the necessary instance for that to happen.
I found `Functor f => MonadFree f (F f)` in the instance list, which was likely the instance I needed.
The functional dependency in the definition of `MonadFree` means that there can be only one choice of the first argument for any given second argument to `MonadFree`.
That means that instance must apply, and furthermore, it means that `f` is `Coyoneda instr`.
Progress!
I managed to move from needing to figure out how to create an `F` to needing to figure out how to create a `Coyoneda`.

``` haskell
singleton = Program . liftF . undefined
```

Given the types of `singleton` and `Program . liftF`, the `undefined` needed to have the type `instr a -> Coyoneda instr a`.
And a quick scan through [the docs](http://hackage.haskell.org/package/kan-extensions-4.0.1/docs/Data-Functor-Coyoneda.html) revealed [`liftCoyoneda`](http://hackage.haskell.org/package/kan-extensions-4.0.1/docs/Data-Functor-Coyoneda.html#v:liftCoyoneda), which is exactly the necessary type.

``` haskell
{-# LANGUAGE RankNTypes, GeneralizedNewtypeDeriving #-}

import Control.Applicative

import Control.Monad.Free.Church
import Data.Functor.Coyoneda

newtype Program instr a = Program (F (Coyoneda instr) a)
                          deriving (Functor, Applicative, Monad)

singleton :: instr a -> Program instr a
singleton = Program . liftF . liftCoyoneda

interpret :: Monad m => (forall x. instr x -> m x) -> Program instr a -> m a
```

I still don't know a thing about what `F` or `Coyoneda` mean.
But it isn't really slowing me down.

The last remaining piece of my target API is `interpret`.
This one looks a bit tougher, but how much of that is the type signature being more complex?
Nearly all of the complexity in that type signature is describing the polymorphic function which is the first argument.
I don't need to do anything with that argument other than apply it in the right place, so I can safely ignore that complexity.

To get my bearings, I started with a minimal skeleton:

``` haskell
interpret f (Program p) = undefined
```

I began to search through `F`'s docs again, looking for something that looked at least partially like `Monad m => F (Coyoneda instr) a -> m a`, or that could be a helpful building block for it.
There were three immediate candidates: [`fromF`](http://hackage.haskell.org/package/free-4.5/docs/Control-Monad-Free-Church.html#v:fromF), [`iterM`](http://hackage.haskell.org/package/free-4.5/docs/Control-Monad-Free-Church.html#v:iterM), and [`retract`](http://hackage.haskell.org/package/free-4.5/docs/Control-Monad-Free-Church.html#v:retract).

`fromF` involves a `MonadFree` constraint on the `m`, and I wanted any `Monad m`, so that's too restrictive.
`retract` has the limitation that the `m` it produces must be the same as in the input, which is also too restrictive.

So I came to the conclusion it must be `iterM`, somehow.
`iterM` also has another argument, probably a conversion function.
I have no idea where its type signature came from, or what that says about the internals of `F`.
But I didn't need to understand those things, I just needed to be able to supply arguments of the correct type.
The extra argument is also helpful in providing a way to use `f` in the call to `iterM`.
If I could fit it in there, it would mean that `interpret` is just a specially-crafted call to `iterM`, which would be convenient.

Putting together the known types, I had the following code:

``` active haskell
{-# LANGUAGE RankNTypes, GeneralizedNewtypeDeriving #-}

{-# LANGUAGE ScopedTypeVariables #-}

import Control.Applicative

import Control.Monad.Free.Church
import Data.Functor.Coyoneda

newtype Program instr a = Program (F (Coyoneda instr) a)
                          deriving (Functor, Applicative, Monad)

singleton :: instr a -> Program instr a
singleton = Program . liftF . liftCoyoneda

interpret :: forall m instr a. Monad m => (forall x. instr x -> m x) -> Program instr a -> m a
interpret f (Program p) = iterM eval p
  where
    eval :: Coyoneda instr (m a) -> m a
    eval = undefined


main = putStrLn "It must have type-checked successfully"
```

The `ScopedTypeVariables` extension was a temporary addition to help machine checking of my understanding of the types in use.
It allowed me to be sure that I've got the correct type signature on `eval` by running GHC against the code and verifying that it compiled.
I made the code block above active because that allows this site to compile and execute it too.

This felt close to done.
The types all seemed to fit together correctly.
There's just one that hadn't been filled in yet.
All that remained was implementing `eval`, knowing that it must use `f`.
Back to the docs for `Coyoneda`!

Fortunately, it's a small module.
I want something that will eliminate the `Coyoneda` type constructor, so I can immediately reject about half the module.
There are [`lowerCoyoneda`](http://hackage.haskell.org/package/kan-extensions-4.0.1/docs/Data-Functor-Coyoneda.html#v:lowerCoyoneda) and [`lowerM`](http://hackage.haskell.org/package/kan-extensions-4.0.1/docs/Data-Functor-Coyoneda.html#v:lowerM).
Neither of them quite work, though.
Each would introduce a new constraint on the type `instr`, which is otherwise unconstrained.
That's not acceptable, so I needed to look elsewhere.

The next step was looking at the instance list for `F`.
I needed something that didn't put any constraints on `instr`, which ruled out most of the instances.
Of the three remaining instances, none provide anything helpful.

There was one last place to look.
This was actually the part that took me the longest to remember when I was first implementing this.
There's one more way to interact with values of a data type than functions that operate on them.
You can pattern-match on the constructor that built them.

``` haskell

interpret :: forall m instr a. Monad m => (forall x. instr x -> m x) -> Program instr a -> m a
interpret f (Program p) = iterM eval p
  where
    eval :: Coyoneda instr (m a) -> m a
    eval (Coyoneda g i) = undefined
```

Unfortunately, I couldn't annotate `g` and `i` with type signatures to have GHC check them for me.
They involve an existential type variable that there's no way to specify in GHC's syntax, so I just have to describe the types informally here.

`g` is a function `b -> m a`, for some type `b`.
`i` is a value of type `instr b`, for the same unknown type `b`.
I needed to put those together with `f :: forall x. instr x -> m x` using the knowledge that `m` is an instance of `Monad` and come up with a value with the type `m a`.
Fortunately, that unknown type `b` fit together perfectly with `f`'s polymorphic type.
The first step had to be `f i :: m b`, because it's the only way to use `f`.
With `f i :: m b` and `g :: b -> m a`, I had `f i >>= g :: m a`, still without knowing what type `b` is.
And now the `b` is gone from the type.
Even more successfully, what I have left matches exactly the type `eval` is supposed to return.
This had to be it!

Of course, even in Haskell, compilation doesn't guarantee correctness.
I still have to verify that.
Here's a silly little test of it all together:

``` active haskell
{-# START_FILE main.hs #-}
{-# LANGUAGE GADTs #-}

import Control.Monad.MiniOperational

data Demo a where
    GetNumber :: Demo Int
    PrintSomethingLike :: Int -> Demo ()

getNumber :: Program Demo Int
getNumber = singleton GetNumber

printSomethingLike :: Int -> Program Demo ()
printSomethingLike = singleton . PrintSomethingLike

runDemo :: Program Demo a -> IO a
runDemo = interpret vaguelyEval
  where
    vaguelyEval :: Demo a -> IO a
    vaguelyEval (GetNumber) = return 7 -- the best number
    vaguelyEval (PrintSomethingLike x) = do
        putStrLn $ " ** " ++ show x ++ " **"

main :: IO ()
main = do
    x <- runDemo $ do
        a <- getNumber
        printSomethingLike $ a - 1
        printSomethingLike $ a + 1
        fmap ((,) a) getNumber
    print x


{-# START_FILE Control/Monad/MiniOperational.hs #-}
{-# LANGUAGE RankNTypes, GeneralizedNewtypeDeriving #-}

module Control.Monad.MiniOperational
    ( Program
    , singleton
    , interpret ) where

import Control.Applicative

import Control.Monad.Free.Church
import Data.Functor.Coyoneda

newtype Program instr a = Program (F (Coyoneda instr) a)
                          deriving (Functor, Applicative, Monad)

singleton :: instr a -> Program instr a
singleton = Program . liftF . liftCoyoneda

interpret :: Monad m => (forall x. instr x -> m x) -> Program instr a -> m a
interpret f (Program p) = iterM eval p
  where
    eval (Coyoneda g i) = f i >>= g
```

Not only does it compile, it works!

# Conclusion

I still don't really understand `free`.
I definitely don't have a clue what `kan-extensions` are in general, or what `Coyoneda` is specifically.
Those are math topics deep in the realm of category theory.
But I was able to use the libraries without the slightest clue about the math behind them.
I didn't need a tutorial.
I'm not even sure what tutorial could possibly have helped – unless you decide this article is a tutorial that should have been titled "How to Free Coyoneda".

I was able to use the absolutely fantastic documentation we call type signatures to quickly figure out how to assemble pieces I don't fully understand into an effective final result.
The whole process of doing this from scratch the first time only took about an hour, and that was including the decision of which implementation strategy I was going to use.
This is why most Haskell libraries don't have tutorials.
They already have better documentation than most tutorials could ever hope to be.

Of course, I'm not saying that libraries can't benefit from tutorials.
`lens` benefits greatly from its myriad tutorials.
That's because it has some of the most ridiculously abstract and flexible type signatures I've ever seen.
Flexibility makes it possible to combine more things in more ways, such that types provide less guidance, and tutorials fill in some of that guidance.
But a complaint that you can't use most libraries because they involve math you don't understand and don't have any tutorials?
That's one of the silliest things I've ever heard.
They have type signatures!
Look what I was able to do in this case with just some knowledge of types and an idea what my goal was.

I'm not special.
Anyone can learn to do this.
All it takes is for people to focus on what's present and how useful it really is.

##### Discussion

Discussion of this article can be found on [reddit](http://www.reddit.com/r/haskell/comments/1yvfmc/programming_with_types_not_tutorials/) and [hacker news](https://news.ycombinator.com/item?id=7298646).
