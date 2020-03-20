# Life after monads

After mastering monads, the next challenge we usually face is combining two (or more) separate monads.
In this tutorial I outline simple plumbing (when the monads to be fit together are not that different),
and two mainstream approaches for general plumbing.

## Simple plumbing

When the monads are similar in nature, we can convert one to the other in an ad-hoc, monad-specific way.
This mostly happens when chaining computations, some of which run in `Maybe`, others in `Either`.

Having the following functions,

```haskell
data MyError = Err Text | OtherErr

verboseFailing :: Int -> Either MyError Int
verboseFailing = ...

justFailing :: Int -> Maybe Int
justFailing = ...
```

we can compose them to either result a `Maybe` or an `Either`, depending on our intent.
Usually it is a good idea to go with the latter, but let's see both.

```haskell
import Control.Monad ((>=>))
import Control.Error.Util (note)

verboseComp :: Int -> Either MyError Int
verboseComp = verboseFailing >=> note OtherErr . justFailing
```

Here we take advantage of the <hoogle>note</hoogle> function from the `errors` package, but
you could write it yourself as well. The other way around:

```haskell
import Control.Error.Util (hush)

justFailingComp :: Int -> Maybe Int
justFailingComp = hush . verboseFailing >=> justFailing
```

This time the <hoogle>hush</hoogle> function was used to silence the error.
The `errors` package contains a lot of combinators that for example let us go from `Maybe` to `MaybeT` easily. Check it out.

### Plumbing Eithers with different error types.

Once we are talking about plumbing and `Eithers` - assume we have something like the following.

```haskell
data LowlevelError = ...
data InputError = ...

validated :: Text -> Either InputError Int
validated = ...

someComp :: Int -> Either LowlevelError Int
someComp = ...
```
How do we combine these methods?
An approach is to create a hierarchy of errors.

```haskell
data AppError = ELowlevel LowlevelError | EInput InputError

app s = do
    a <- EInput `fmapL` validated s
    ELowlevel `fmapL` someComp a
```

The <hoogle>fmapL</hoogle> is again from `errors`, and maps on the left.
Although we could make it nicer by introducing a flipped alias.

```haskell
wrapError = flip fmapL

app s = do
    a <- validated s `wrapError` EInput
    someComp a `wrapError` ELowlevel
```

## Combining wildly different monads  

What happens when we leave the safe realm of `Maybe` and `Either`?

```active haskell
import Control.Monad.Trans.Reader
import Control.Monad.Trans.State

more :: Reader Int Int
more = ask >>= return . (+1)

bang :: State String Int
bang = modify (++"!") >> get >>= return . length

-- | How to combine more and bang?
morePlusBang = undefined

main = do
    -- We can of course use them separately.
    print $ runReader more 42
    print $ runState (bang >> bang) "?"
```

Let me outline two mainstream approaches for plumbing different monads.

## Solution 1: Monadic interfaces

One solution is to program against monad interfaces from `mtl`,
instead of actual instances from `transformers`.

```active haskell
{-# START_FILE Lib.hs #-}
{-# LANGUAGE FlexibleContexts #-}
module Lib where
-- Notice we just import the monad interfaces,
-- not specific implementations.
import Control.Monad.Reader.Class
import Control.Monad.State.Class

-- | The interface of 'm' is specified using a typeclass now.
-- 'm' can be any data type, if it at least conforms to the
-- 'MonadReader' interface (which generalizes the various 'Reader's).
more :: (MonadReader Int m) => m Int
more = ask >>= return . (+1)

bang :: (MonadState String m) => m Int
bang = modify (++"!") >> get >>= return . length

-- | We need to propagate the merged requirements of the
-- parts we use.
morePlusBang :: (MonadReader Int m, MonadState String m) => m Int
morePlusBang = do
    a <- more
    b <- bang
    return $ a + b

{-# START_FILE Main.hs #-}
import Control.Monad.Trans.RWS
import Lib

main = do
    -- The type signature just fixes the unused 'w' monoid in RWS.
    print (runRWS (bang >> morePlusBang) 42 "?"  :: (Int,String,[()]))
```

Here we have chosen <hoogle>RWS</hoogle>, the Reader-Writer-State monad,
as it supports our requirements well.

In fact, you are free to choose any monad (stack) that conforms to the
requirements. For example

```haskell
main = do
    print $ runState (runReaderT (bang >> morePlusBang) 42) "?"
```

automatically infers the stack to be `ReaderT Int (State String) a`.

The pro is that now we can nicely compose the computations, as long as we are
able to find a single stack that fits all our requirements.

A cons is that the typeclasses might be more costy runtime-wise, as the
typeclass dictionaries need to be passed. Also, the signatures are a bit
more heavyweight due to the added constraints. Latter can be somewhat mitigated
by using the `ConstraintKinds` language extension, which lets us do
aliases like `type MyReqs m = (MonadState String m, MonadReader Int m)`.

### Baking the stack

It is a reported usage, that after fleshing out the exact requirements
for a given application stack, a specific stack is "baked" in.

First, a new monad is created, and all typeclass constraints are
replaced by using the specific monad.

```haskell
{-# LANGUAGE GeneralizedNewtypeDeriving #-}
newtype App a = App { runApp :: ReaderT Int (State String) a }
    deriving (Monad, MonadState, MonadReader)

more :: App Int
more = App $ ask >>= return . (+1)

bang :: App Int
bang = App $ modify (++"!") >> get >>= return . length

morePlusBang :: App Int
morePlusBang = do
    a <- more
    b <- bang
    return $ a + b
```

The pro seems to be that this helps the compiler better inline?
While a con is, that now all functions in the app have access to the full
stack, while previously functions could only access the part they had
business with (`Reader` for `more`, `State` for `bang`).

Actually if no new functionality is added, `App` could have been a simple
type alias instead of a newtype. A newtype makes more sense if we also provide
specialized operations on it, at least hiding the internals a bit.

## Solution 2: Transformer stacks

If we are not too picky, can just implement the functions via monad
transformers, and plumb them together explicitly when needed.

Note that in the code now we need to `lift` the inner monad to the outermost
level. If our stack were multiple levels deep, we might need `lift . lift`
and so on.

```active haskell
import Control.Monad.Trans.Class (lift)
import Control.Monad.Trans.Reader
import Control.Monad.Trans.State

more :: (Monad m) => ReaderT Int m Int
more = ask >>= return . (+1)

bang :: (Monad m) => StateT String m Int
bang = modify (++"!") >> get >>= return . length

morePlusBang :: ReaderT Int (State String) Int
morePlusBang = do
    a <- more
    b <- lift bang
    return $ a + b

main = print $ runState (runReaderT morePlusBang 42) "?"
```
The pro is that it is straightforward, and most of the time we don't need
to combine too many monads anyway. The cons is that we really need to have
a good grasp of monad transformers to be able to lift at will.
But the latter would be needed anyway. Also, our code is a bit less flexible,
since altering the stack has effect on functions using it. For example,
adding a new layer needs us to add some more lifting.

I suggest also taking a look at the `mmorph` package, which can be
useful if we have to work with monad stacks not under our control:
It lets us to swap (<hoogle>hoist</hoogle>) the inner monad layer in a transformer (among others).

## Outro

Hopefully this post gives some pointers to you. In the real world,
nothing is ultimately black and white, and you are encouraged to do
your own contemplation about the various approaches.