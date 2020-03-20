# Encoding Objects

In Haskell, object-oriented programming is hardly adopted. In my humble opinion, this is because:

* A lot of patterns used in other OOP languages can be replaced by other features of Haskell: polymorphism, first-class functions, ExistentialQuantification, etc.
* Scarce use cases: few projects need to encapsulate states while handling various operations.
* Potential disgust for OOP.

The Motivation
----

Still, occasionally, we need objects.

When a method is invoked, an object performs an action, potentially modifying its hidden state.
The important property here is that objects have the same type as long as the interface (i.e. set of operations) are the same, even if the type of their states are different. Thus objects are extensible in terms of entities: you can add a new entity without changing any data types. This is not possible with sum types.

A method takes 0 or more arguments, and returns a result. Some streaming libraries like `pipes` can express this kind of interactivity, but only when it has exactly one operation. One reasonable approach would be to represent an interface as a GADT. We call these constructors "messages".

```haskell
data Hello x where
  Hello :: Hello ()
  Increment :: Int -> Hello Int
```

We want to translate `Hello a` into `m a` where `m` is a monad, but `forall a. Hello a -> m a` isn't stateful unless `m` utilises some extra machinery to maintain the state.
If `m` is weaker than `ST` or `IO`, we can't use references. If `m` is a state monad, the state is not hidden.

Mealy machines
----

A Mealy machine is a state machine that produces an output for each input. Objects take messages and returns a result; behaves exactly like a Mealy machine.

In Haskell, mealy machines can be encoded as a recursive type:

```haskell
newtype Mealy a b = Mealy { runMealy :: a -> (b, Mealy a b) }
```

`a` and `b` are monomorphic, so no more than one method can be handled.

Polymorphosis
----

Comparing `forall a. Hello a -> m a` and `Mealy`, it seems possible to combine these two, using `Rank2Types`:

```haskell
newtype Object f g = Object { runObject :: forall x. f x -> g (x, Object f g) }
```

This `hello` object can handle messages maintaining the internal state.

```haskell
hello :: Int -> Object Hello IO
hello n = Object $ \case -- LambdaCase extension
  Increment m -> return (n, hello (n + m))
  Hello -> putStrLn "Hello" >> return ((), hello n)
```

You can put `Object`s into a container. `MVar` can be used to use them like other OOPLs

`newMVar` instantiates an object. We can define a method invocation operator:

```haskell
(.-) :: MVar (Object f IO) -> f a -> IO a
v .- f = do
  obj <- takeMVar v
  (a, obj') <- restore (runObject obj f >>= evaluate) `onException` putMVar v obj
  putMVar v obj'
  return a
```

In case of exception, it will reset to the original object. This prevents the state from getting inconsistent and it seems like an improvement over other OOP implementations.

```haskell
> v <- newMVar (hello 0)
> v .- Hello
Hello
> v .- Increment 1
0
> v .- Increment 42
1
```

It works as an instance indeed.

Composition!?
----

`Object`s are composable as well as functions are (not to be confused with "composition" in OOP sense!).

```haskell
(@>>@) :: Functor h => Object f g -> Object g h -> Object f h
Object m @>>@ Object n = Object $ fmap (\((x, m'), n') -> (x, m' @>>@ n')) . n . m
```

With an identity object `echo = Object $ \f -> (\x -> (x, echo)) <$> f :: Functor f => Object f f`, they form a category of interfaces where morphisms are objects (it's double-confusing because the objects, in categorical sense, are interfaces...).

I admit the object composition isn't so important.

Mortals
----
Unlike typical objects, this object encoding can die. It's simple: the effect it produces may fail.

```haskell
newtype Mortal f g a = Mortal { unMortal :: Object f (EitherT a g) }
```

Mortals form a monad: `return` dies immediately, and the bind reincarnates an object, passing the final result.

```haskell
instance Monad m => Monad (Mortal f m) where
  return a = mortal $ const $ left a
  {-# INLINE return #-}
  m >>= k = mortal $ \f -> lift (runEitherT $ runMortal m f) >>= \r -> case r of
    Left a -> runMortal (k a) f
    Right (x, m') -> return (x, m' >>= k)
```

In game programming, typically the manager has to maintain the collection of objects. The ability to die by oneself is quite useful.

Conclusion
----

Objects can be thought of as Mealy machines of effects, and their Haskell encoding is actually straightforward. This novel building block might be too strong to use everyday, but I'm pretty sure there are places where this abstraction fits well, like game programming.

The library implementation is available on hackage: [objective](https://hackage.haskell.org/package/objective)