# Monad Transformers and Static Effect Scoping

There's been [quite](http://edwinb.wordpress.com/2013/03/28/programming-and-reasoning-with-algebraic-effects-and-dependent-types/) a [bit](http://homepages.inf.ed.ac.uk/slindley/papers/handlers-draft-march2013.pdf) of [buzz](http://www.cs.indiana.edu/~sabry/papers/exteff.pdf) about algebraic effects and handlers lately, and of course, discontent with the status quo, monad transformers. One point made (among many) is that monad transformers essentially specify statically how effects interact, and that creating more dynamic seeming behavior can be confusing. This is true, although perhaps the latter part is debatable. In this article, I'd like to argue the thesis that the staticness can actually be a useful property, that certain ways of harnessing it can make the confusing constructions less so, and provide some hints that algebraic effects perhaps aren't so different from what we are already using.

### Delimited Continuations

Obviously the best place to start on this journey is delimited continuations. The core idea of continuations is that they allow one to capture up 'the rest of the program' as a first class value, to be jumped back to in some way at a later point. Delimited continuations allow one to place markers in the program, such that instead of capturing the entire 'rest of the program,' one only captures 'the rest of the program until we get to a marker.' This generally makes them a bit better behaved than traditional continuations.

One way of providing delimited continuations in Haskell is given by the CC-delcont package. It provides a monad for using delimited continuations, and in particular, I'd like to look at the pair of operations that are most common for the subject: shift and reset. A simplified version of their types look like the following:

``` haskell
reset :: (Prompt a -> CC a) -> CC a
shift :: Prompt b -> ((CC a -> CC b) -> CC b) -> CC a
```

`reset b` creates a delimited scope, and hands `b` a `Prompt` that can be used to refer to that scope. `shift p e`, captures the continuation inside the scope referred to by `p`, and hands it to `e` to use as it wishes. A simpler presentation would remove mention of the prompts, but they are actually essential in `CC` to get the types to work out. Quite a lot of examples get by with only a single prompt.

There are some important properties that the above operators follow. Basically, both `reset` and `shift` contain control effects to certain regions. If we forget about mismatched prompts, then no control effects can escape from a `reset`, nor can they escape from the body of a `shift`. It is also the case that the captured continuation passed to the body of `shift` is a pure function; it does not have any control effects. This is not true in `CC`, because a `p1` captured continuation can have `p2` effects (and actually, there are lower level operators that can subvert these properties). But, this is at least the intended behavior of `shift/reset`.

### An alternate implementation

There is, however, a way of working with delimited continuations using just the ordinary `Cont` type. My favorite such way is as follows:

``` haskell
reset :: Cont r r -> r
reset = (`runCont` id)

shift :: ((a -> r) -> r) -> Cont r a
shift = cont
```

The similarities between these types and the `CC` counterparts should be clear, hopefully. One thing to note is that the first argument to `Cont` is serving the same role as the type parameter of the prompt above; telling us what the result type of the continuation is for our delimited scope.

However, there is a huge difference between these types and the `CC` version: these types statically ensure the 'important properties' I mentioned above. `reset` obviously lets no control effects escape, because it is just returning a pure value. The continuation passed to the body of shift is obviously a pure function, and no control effects are allowed to escape from the body of shift, because the return type of the body is just `r`. Many presentations of this version of delimited continuations mask this by re-injecting things into `Cont`, but this is my favorite version because of how the types tell you all the relevant properties.

One can do the same construction with `ContT`, and a similar thing occurs:

``` haskell
reset :: Monad m => ContT r m r -> m r
reset = (`runContT` return)

shift :: ((a -> m r) -> m r) -> ContT r m a
shift = ContT
```

It is still obvious where control effects can occur with `shift` and `reset`, at least excluding any control effects supported by `m` (which is a way to have multiple prompts in this setup; except the prompt scoping is enforced statically---albeit with terrible de Bruijn indices---while in CC-delcont one could use prompts in invalid scopes). This delimiting of effects is achieved statically by moving in and out of the monad (transformer), and it seems to me to be a good fit for many `shift/reset` examples, as they are sometimes known as the static delimited control operators.

### Throw/catch confusion

Let's consider an example from the recent paper [Extensible Effects](http://www.cs.indiana.edu/~sabry/papers/exteff.pdf). The idea is the following: we want to have a nondeterministic computation with exceptions. If any one of the nondeterministic possibilities fails, the whole computation should fail. However, at a certain point, we want to handle some of the exceptions, and the handled exceptions should not interfere with the computation working or not.

(I'm going to make my own nondeterminism class here, as `MonadPlus` is overloaded to also cover exception-like things, so exceptions will eat non-determinism.)

``` active haskell
{-# LANGUAGE FlexibleContexts #-}
-- show
import Control.Monad.Identity
import Control.Monad.Error
import Control.Monad.List

class Monad m => MonadChoice m where
  choose :: [a] -> m a
-- /show
instance MonadChoice [] where choose = id
instance Monad m => MonadChoice (ListT m) where
  choose = ListT . return
instance (Error e, MonadChoice m) => MonadChoice (ErrorT e m) where
  choose = lift . choose
-- show
newtype TooBig = TooBig Integer deriving (Eq, Ord, Show, Read)
instance Error TooBig -- don't care

example :: (MonadError TooBig m) => m Integer -> m Integer
example m = do
  v <- m
  if v > 5
    then throwError (TooBig v)
    else return v

handle :: MonadError TooBig m => m Integer -> m Integer
handle m = m `catchError` \e ->
  case e of
    TooBig v | v <= 7 -> return v
    _ -> throwError e

wrong1 :: Either TooBig [Integer]
wrong1 = runIdentity . runErrorT . runListT
       . handle . example $ choose [2,3,5,7]

wrong2 :: [Either TooBig Integer]
wrong2 = runErrorT . handle . example $ choose [2,3,5,7,11]

-- /show
main = do print wrong1
          print wrong2
```

The above program demonstrates that no single, global choice of effect interaction produces the desired results. In `wrong1`, the failing 7 destroys all other computations, leaving it as the only case when we recover. In `wrong2`, we get the local recovery, but the still failing 11 does not cause the rest of the computations to die.

The proposal in the paper seems to be to make `catchError` behave in the way that we want, but leave its API the same as the mtl. However, I contend that this is actually a sub-optimal choice. The API of `catchError` is based on the notion that it is an operation in some indeterminate error-supporting monad. But, if the algebraic effects and handlers work tells us anything (or at least, if I understand it well enough), it's that handlers are actually not operations of the monad. `throw` is a generator of the algebra, but `catch` is a handler, and should probably eliminate from the set of effects, much like `reset` above. However, we can actually write such an operation using the mtl/transformers:

``` active haskell
{-# LANGUAGE FlexibleContexts #-}

import Control.Monad.Identity
import Control.Monad.Error
import Control.Monad.List

class Monad m => MonadChoice m where
  choose :: [a] -> m a

instance MonadChoice [] where choose = id
instance Monad m => MonadChoice (ListT m) where
  choose = ListT . return
instance (Error e, MonadChoice m) => MonadChoice (ErrorT e m) where
  choose = lift . choose

newtype TooBig = TooBig Integer deriving (Eq, Ord, Show, Read)
instance Error TooBig -- don't care

example :: MonadError TooBig m => m Integer -> m Integer
example m = do
  v <- m
  if v > 5
    then throwError (TooBig v)
    else return v

-- show
localCatch :: Monad m => ErrorT e m a -> (e -> m a) -> m a
localCatch m h = runErrorT m >>= \x -> case x of
  Left  e -> h e
  Right a -> return a

handle :: MonadError TooBig m => ErrorT TooBig m Integer -> m Integer
handle m = m `localCatch` \e ->
  case e of
    TooBig v | v <= 7 -> return v
    _ -> throwError e

right1 :: Either TooBig [Integer]
right1 = runIdentity . runErrorT . runListT
       . handle . example $ choose [2,3,5,7]

right2 :: Either TooBig [Integer]
right2 = runIdentity . runErrorT . runListT
       . handle . example $ choose [2,3,5,7,11]
-- /show

main = do print right1
          print right2
```

The code is identical, except we use `localCatch` instead of `catchError`. However, the type of `localCatch` makes clear that it delimits the scope in which (certain) error effects can occur, handling all of them. Our use of it in `handle` rethrows some of the exceptions, but it is not necessary to do so; exceptions are actually reintroduced as an effect in the handler, but in the `m` monad, so that we can make a later decision about how they should interact with the nondeterministic choice.

This is the purported confusing solution to the problem, using two exception transformers. And indeed, if we look at the final stack we used, it was `ErrorT e (ListT (Error e))`. However, at no point were we obligated to think about that. The exception generators simply programmed to the `MonadError e` specification, the first `ErrorT e` was used to introduce a local exception scope in which we could recover without clobbering other effects, and the second was used when we finally observed the computation, when we _did_ want errors to clobber the nondeterminism. And this is exactly what would have been going on with algebraic effects, where the local handler would pass through the choice effects, and reintroduce exceptions so that they could interact differently with choice than they did in the local scope.

If anything, this example seems to be pointing to the stock `catchError` being a bad operator, at least for this use case. The more handler-like `runErrorT`, or a combinator based on it, is appropriate, and gives effect scoping information in the types, much like `shift/reset`.

Finally, note that the choice of `runErrorT` in `localCatch` was somewhat arbitrary. Algebraic effects allow programs to write to one error algebra, but be used with many handlers. This is also true here; programs written with `throwError` incur a `MonadError` constraint, but that constraint can be locally satisfied by any concrete instance thereof, and the choice of instance corresponds to a choice of handler for the error algebra. Choosing an entire stack of transformers corresponds to handling all effects simultaneously in a particular way, but handling only a portion of the effects can be done in the mtl, by running a particular transformer and choosing to pass the remaining obligations to the underlying monad.

### Another example

It occurred to me when thinking about this that I'd used a similar technique while writing a [piece of a compiler](https://github.com/ermine-language/ermine/blob/master/src/Ermine/Inference/Discharge.hs). It is an implementation of a Haskell-like language, and the relevant code is in the type class solving machinery. The idea is that we wish to replace the type class obligations we've collected with simpler versions, based on class and instance declarations we've seen. The core loop looks something like (simplified from the actual code):

``` haskell
dischargesBySupers :: (Alternative m, Discharge m)
                   => Class -> [Class] -> m [Class]

dischargesByInstance :: (Alternative m, Discharge m)
                     => Class -> m [Class]

entails :: (Alternative m, Discharge m) => [Class] -> Class -> m [Class]
entails cs ob = ob `dischargesBySupers` cs
            <|> (dischargesByInstance ob >>= simplify cs)

simplify :: Discharge m => [Class] -> [Class] -> m [Class]
simpify cs obs = do
  x <- for obs $ \ob ->
    runMaybeT (entails cs ob) <&> fromMaybe (pure ob)
  pure $ join x
```

`dischargesBySupers` informs us if its first argument can be gotten by projection out of the classes in the list. `dischargesByInstance` tells us if there is an instance for the given class, and what obligations it requires. Both of these checks can fail, hence the `Alternative`. Then, `entails` checks if the list of class obligations is sufficient to entail a particular obligation, and gives what obligations that would add, again potentially failing. `simplify` reduces many obligations given a set of prerequisites. However, `simplify` is not generally allowed to fail, as it is the point at which we call into the solver; so, it creates a local scope in which failure is permitted, but handles failure by yielding the original value.

Note, though, that `enails` calls back into `simplify`, to ensure that the additional obligations returned are themselves fully simplified. In this case, we are using `simplify` in a context that can fail, but it still creates its own neseted effect scope. In general, there is no static determination on how many nested effect scopes are created, only static determination of which effects are handled in which scope.


### Conclusion

The above is not the only example of where we use local, static effect scopes in the compiler. For instance, we use a local scope to track sharing information in parts of the unifier. However, we do this by writing the effectful code using a type class interface, `MonadFoo` and 'handling' with a corresponding `runFooT`. Doing so, one gets something that looks a lot like the algebra+handlers way of doing effects (to my untrained eyes, at least), even with the boring old mtl. Static effect scoping needn't preclude local effect scoping.

### Bonus

I mentioned earlier that one could do multi-prompt continuations in the alternate style; here's how it looks:

``` active haskell
import Control.Monad.Cont
-- show
reset :: Monad m => ContT r m r -> m r
reset = (`runContT` return)

shift :: ((a -> m r) -> m r) -> ContT r m a
shift = ContT

example1 :: MonadIO m => m ()
example1 = reset $ do
  liftIO $ putStrLn "Example 1"
  reset $ do
    -- throw the continuation away
    shift $ \_ -> return ()
  liftIO $ putStrLn "Hi!"

example2 :: MonadIO m => m ()
example2 = reset $ do
  liftIO $ putStrLn "Example 2"
  reset $ do
    -- capture to the outer reset this time
    lift . shift $ \_ -> return ()
  liftIO $ putStrLn "Bye!"
-- /show
main = example1 >> example2
```

For each `lift`, we `shift` out one more layer. This, of course, is not a fantastic API.