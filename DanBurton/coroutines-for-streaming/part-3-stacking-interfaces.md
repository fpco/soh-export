# Part 3: Stacking Interfaces

# Specializing Coroutine to our purposes

Rather than explain right now why we're doing this, just stick with me while I specialize the previous Coroutine code to our use case, hardcoding the `Interface` into the type, and making `Producing` and `Consuming` newtypes. One benefit to this is that users will have clearer type errors. I've included the Coroutine type declarations to help illustrate that all I have done here is set the suspension functor to `Interface i o`.

```haskell
-- newtype Coroutine s m r
--   = Coroutine { resume :: m (CoroutineState s m r) }
newtype Producing o i m r
  = Producing { resume :: m (ProducerState o i m r) }

-- data CoroutineState s m r
--   = Run (s (Coroutine s m r))
--   | Done r
data ProducerState o i m r
  = Produced o (Consuming r m i o)
  | Done r

newtype Consuming r m i o
  = Consuming { provide :: i -> Producing o i m r }
```

Since nothing has really changed, the type class instances remain essentially the same as well. I'll provide my implementation here for clarity, though. Notice the similar recursion scheme for implementing `fmap` for `ProducerState o i`, and implementing `>>=` and `hoist` for `Producing o i`.

```haskell
instance (Functor m) => Functor (Producing o i m) where
   fmap f p = Producing $ fmap (fmap f) (resume p)

instance (Functor m) => Functor (ProducerState o i m) where
  fmap f (Done x) = Done (f x)
  fmap f (Produced o k) = Produced o $ Consuming (fmap f . provide k)

instance (Functor m, Monad m) => Applicative (Producing o i m) where
   pure = return
   (<*>) = ap

instance (Monad m) => Monad (Producing o i m) where
   return x = Producing $ return (Done x)
   p >>= f = Producing $ resume p >>= \s -> case s of
     Done x -> resume (f x)
     Produced o k ->
      return $ Produced o $ Consuming ((>>= f) . provide k)

instance MonadTrans (Producing o i) where
   lift = Producing . liftM Done

instance MFunctor (Producing o i) where
  hoist f = go where
    go p = Producing $ f $ liftM map' (resume p)
    map' (Done r) = Done r
    map' (Produced o k) = Produced o $ Consuming (go . provide k)
```

The main operations, `yield` and `$$`, remain the same, modulo newtype gymnastics.

```haskell
yield :: Monad m => o -> Producing o i m i
yield o = Producing $ return $ Produced o $ Consuming return

infixl 0 $$
($$) :: Monad m => Producing a b m r -> Consuming r m a b -> m r
producing $$ consuming = resume producing >>= \s -> case s of
  Done r -> return r
  Produced o k -> provide consuming o $$ k
```

Before we go further, I suppose we should play with what we've got.

# Play time

```active haskell
-- /show
import Control.Applicative
import Control.Monad
import Control.Monad.Morph
import Control.Monad.Trans.Class

newtype Producing o i m r
  = Producing { resume :: m (ProducerState o i m r) }

data ProducerState o i m r
  = Produced o (Consuming r m i o)
  | Done r

newtype Consuming r m i o
  = Consuming { provide :: i -> Producing o i m r }

instance (Functor m) => Functor (Producing o i m) where
   fmap f p = Producing $ fmap (fmap f) (resume p)

instance (Functor m) => Functor (ProducerState o i m) where
  fmap f (Done x) = Done (f x)
  fmap f (Produced o k) = Produced o $ Consuming (fmap f . provide k)

instance (Functor m, Monad m) => Applicative (Producing o i m) where
   pure = return
   (<*>) = ap

instance (Monad m) => Monad (Producing o i m) where
   return x = Producing $ return (Done x)
   p >>= f = Producing $ resume p >>= \s -> case s of
     Done x -> resume (f x)
     Produced o k ->
      return $ Produced o $ Consuming ((>>= f) . provide k)

instance MonadTrans (Producing o i) where
   lift = Producing . liftM Done

instance MFunctor (Producing o i) where
  hoist f = go where
    go p = Producing $ f $ liftM map' (resume p)
    map' (Done r) = Done r
    map' (Produced o k) = Produced o $ Consuming (go . provide k)

yield :: Monad m => o -> Producing o i m i
yield o = Producing $ return $ Produced o $ Consuming return

infixl 0 $$

($$) :: Monad m => Producing a b m r -> Consuming r m a b -> m r
producing $$ consuming = resume producing >>= \s -> case s of
  Done r -> return r
  Produced o k -> provide consuming o $$ k


-- show
example1 :: Producing String String IO ()
example1 = do
  name <- yield "What's your name? "
  lift $ putStrLn $ "Hello, " ++ name
  color <- yield "What's your favorite color? "
  lift $ putStrLn $ "I like " ++ color ++ ", too."

-- this comes in handy for defining Consumers
foreverK :: Monad m => (a -> m a) -> a -> m r
foreverK f = go where
  go a = f a >>= go

stdOutIn :: Consuming r IO String String
stdOutIn = Consuming $ foreverK $ \str -> do
  lift $ putStrLn str
  lift getLine >>= yield

stdInOut :: Producing String String IO r
stdInOut = provide stdOutIn ""

main = example1 $$ stdOutIn
```

Try building your own coroutines with the Producing monad, and hooking them together with `$$`. But remember, they must be in opposite states, and have compatible interfaces, as well as the same underlying monad, in order to connect.

# Two interfaces makes a Proxy

What happens when we put two interfaces on top of each other?

```haskell
Producing a b (Producing c d m) r
```

What *is* this? Well, it is a computation which can transfer control to one of two interfaces. The action `yield a` will surrender control to the outer interface, while `lift (yield c)` will surrender control to the inner interface. What happens when we connect such a thing's outer interface?

```haskell
p :: Producing a b (Producing c d m) r
c :: Consuming r   (Producing c d m) a b
p $$ c ::           Producing c d m r
```

Since the "inner monad" is `Producing c d m`, the "Consuming" counterpart must have the same inner monad, `Producing c d m`. Once connected, the two computations *merge* their use of the `c/d` interface, and become (to the outside world) one computation. The `a/b` interface becomes unobservable, or satisfied, or connected, or whatever you want to call it.

That's cool, but there is something obnoxious about it. What if I want to connect computations which don't necessarily suspend on the same underlying interfaces?

```haskell
p :: Producing a b m r
c :: Consuming r (Producing c d m) a b
p $$ c :: Type Error
```

Luckily, `hoist` and `lift` can help us insert the missing layer, so that the two can connect. After all, we can say that `p` communicates on the `c/d` interface, it just *happens* to do so zero times.

```haskell
insert0 = lift          -- add a new layer at depth 0 (the outermost layer)
insert1 = hoist insert0 -- add a new layer at depth 1
insert2 = hoist insert1 -- add a new layer at depth 2

p :: Producing a b m r
insert1 p :: Producing a b (t m) r -- t = any MonadTrans we want
c :: Consuming r (Producing c d m) a b

-- t becomes specialized to (Producing c d)
insert1 p $$ c :: Producing c d m r
```

What just happened? We took a computation over the `a/b` interface, *connected* it to a computation over *both* the `a/b` and `c/d` interfaces, and *transformed* it into just a computation over the `c/d` interface. My friends, we have stumbled onto the concept of a `Proxy`, and just implemented `$=`.

```haskell
newtype Proxy r m upI downI
  = Proxy { unProxy :: Consuming r (Producing (Fst downI) (Snd downI) m) (Fst upI) (Snd upI) }

type family Fst (xy :: (*,*)) :: *
type family Snd (xy :: (*,*)) :: *
type instance Fst '(x,y) = x
type instance Snd '(x,y) = y

($=) :: Monad m => Producing a b m r -> Proxy r m '(a,b) '(c,d) -> Producing c d m r
producing $= Proxy proxy = insert1 producing $$ proxy
```

Proxies have two interfaces, a "downstream" interface, and an "upstream" interface. We can connect a proxy to a `Producing` coroutine via the proxy's upstream interface (also its *outer* interface, which is in a state of `Consuming`). I gave `Proxy` a rather unsightly definition, which allows us to write each interface as a tuple. (This requires `DataKinds`, `KindSignatures`, and `TypeFamilies` language extensions.) The reason for this is so that we can (once ghc-7.8 is finished) write the Category instance for `Proxy`. We'll talk more about this later.

# Producing layers commute

On the topic of two interfaces, when thinking about it from the "enhanced language" perspective, it seems intuitive that `Producing a b (Producing c d m) r` is the same as `Producing c d (Producing a b m) r`. And it is!

```active haskell
-- show given this (puzzle pieces)
{-# LANGUAGE ScopedTypeVariables #-} -- this comes in handy
{-# LANGUAGE EmptyDataDecls #-}
{-# LANGUAGE KindSignatures #-}

import Control.Monad.Trans.Class (MonadTrans, lift)
import Control.Monad.Morph (MFunctor, hoist)

data Producing o i (m :: * -> *) r -- don't rely on the internals of this
instance (Monad m) => Monad (Producing o i m) where
instance MonadTrans (Producing o i) where
instance MFunctor (Producing o i) where

newtype Consuming r m i o
  = Consuming { provide :: i -> Producing o i m r }

infixl 0 $$
($$) :: Monad m => Producing a b m r -> Consuming r m a b -> m r
producing $$ consuming = undefined -- take this as a given

-- The proxy newtype was left out for simplicity
idProxy :: Monad m => Consuming r (Producing a b m) a b
idProxy = undefined -- take this as a given
-- bonus: implement this by also assuming yield as a given

insert0 :: Monad m =>
  m r -> Producing a b m r
insert0 = lift

insert1 :: (MFunctor t, Monad m) =>
  t m r -> t (Producing a b m) r
insert1 = hoist insert0

insert2 :: (MFunctor t, MFunctor t2, Monad m, Monad (t m)) =>
  t2 (t m) r -> t2 (t (Producing a b m)) r
insert2 = hoist insert1

-- show implement this (the puzzle)
commute :: forall a b c d m r. Monad m =>
  Producing a b (Producing c d m) r -> Producing c d (Producing a b m) r
commute p = p' $$ funnel where
  -- what types should p' and funnel have? (leverage scoped type variables)
  p' :: ()
  p' = undefined
  funnel :: ()
  funnel = undefined
  -- types hint: remember, $$ removes the outermost interface
  -- implementation hint: use insert0/1/2 with p and idProxy

-- show and see if it compiles. Type tetris is fun!
main = putStrLn "It compiles!"
```

@@@ My solution
```haskell
commute :: forall a b c d m r. Monad m =>
  Producing a b (Producing c d m) r -> Producing c d (Producing a b m) r
commute p = p' $$ funnel where
  p' :: Producing a b (Producing c d (Producing a b m)) r
  p' = insert2 p
  funnel :: Consuming r (Producing c d (Producing a b m)) a b
  funnel = Consuming (insert1 . provide idProxy)
```
@@@

Cool! With clever use of `insert1` and friends, we see that coroutine interface layers commute.

# More implementation

Now that we have `$$`, and `commute` at our disposal, we have the high-level tools we need to implement  `=$` and `=$=` as well. Go ahead, give it a shot!

```active haskell
-- show given this
{-# LANGUAGE ScopedTypeVariables #-} -- this comes in handy
{-# LANGUAGE EmptyDataDecls, KindSignatures #-}
{-# LANGUAGE DataKinds, TypeFamilies #-}

import Control.Monad.Trans.Class (MonadTrans, lift)
import Control.Monad.Morph (MFunctor, hoist)

data Producing o i (m :: * -> *) r -- don't rely on the internals of this
instance (Monad m) => Monad (Producing o i m) where
instance MonadTrans (Producing o i) where
instance MFunctor (Producing o i) where

newtype Consuming r m i o
  = Consuming { provide :: i -> Producing o i m r }

newtype Proxy r m upI downI
  = Proxy { unProxy :: Consuming r (Producing (Fst downI) (Snd downI) m) (Fst upI) (Snd upI) }

type family Fst (xy :: (*,*)) :: *
type family Snd (xy :: (*,*)) :: *
type instance Fst '(x,y) = x
type instance Snd '(x,y) = y

infixl 0 $$
($$) :: Monad m => Producing a b m r -> Consuming r m a b -> m r
producing $$ consuming = undefined -- take this as a given

commute :: Monad m => Producing a b (Producing c d m) r
                   -> Producing c d (Producing a b m) r
commute = undefined -- take this as a given

insert0 :: Monad m =>
  m r -> Producing a b m r
insert0 = lift

insert1 :: (MFunctor t, Monad m) =>
  t m r -> t (Producing a b m) r
insert1 = hoist insert0

insert2 :: (MFunctor t, MFunctor t2, Monad m, Monad (t m)) =>
  t2 (t m) r -> t2 (t (Producing a b m)) r
insert2 = hoist insert1

-- show implement these
(=$) :: forall a b c d m r. Monad m =>
  Proxy r m '(a,b) '(c,d) -> Consuming r m c d -> Consuming r m a b
Proxy proxy =$ consuming = Consuming $ \(a :: a) ->
  let
    p :: ()
    p = undefined
    c :: ()
    c = undefined
  in
    p $$ c

(=$=) :: forall a a' b b' c c' m r. Monad m =>
  Proxy r m '(a,a') '(b,b') -> Proxy r m '(b,b') '(c,c') -> Proxy r m '(a,a') '(c,c')
Proxy proxyl =$= Proxy proxyr = Proxy $ Consuming $ \(a :: a) ->
  let
    p :: ()
    p = undefined
    c :: ()
    c = undefined
  in
    p $$ c

-- show and see if it compiles.
main = putStrLn "It compiles!"
```

@@@ My implementation
```haskell
(=$) :: forall a b c d m r. Monad m =>
  Proxy r m '(a,b) '(c,d) -> Consuming r m c d -> Consuming r m a b
Proxy proxy =$ consuming = Consuming $ \(a :: a) ->
  let
    p :: Producing c d (Producing b a m) r
    p = commute (provide proxy a)
    c :: Consuming r (Producing b a m) c d
    c = Consuming (insert1 . provide consuming)
  in
    p $$ c

(=$=) :: forall a a' b b' c c' m r. Monad m =>
  Proxy r m '(a,a') '(b,b') -> Proxy r m '(b,b') '(c,c') -> Proxy r m '(a,a') '(c,c')
Proxy proxyl =$= Proxy proxyr = Proxy $ Consuming $ \(a :: a) ->
  let
    p :: Producing b b' (Producing a' a (Producing c c' m)) r
    p = insert2 (commute (provide proxyl a))
    c :: Consuming r (Producing a' a (Producing c c' m)) b b'
    c = Consuming $ insert1 . provide proxyr
  in
    p $$ c
```
@@@