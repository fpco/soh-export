# Functor Buildup

NOTE: In progreess set of tutorials for my class on Safe Software at UCSB

Now that we have discussed data structures, and introduced the concept of type classes we want to use the concept of type classes to abstract and generalize our code.

Let's review some of the types we have seen so far. 

```haskell
data Maybe a = Just a | Nothing

data [a] = a :: [a] | [] -- isomorphic to `data List a = Cons a (List a) | Nil`

data Either e a = Left e 
                | Right a

data BinTree a = Branch (BinTree a) a (BinTree a)
               | Tip a
               | Empty

```

With `Maybe` we either have a value or we don't. Sometimes we want to apply a function to the value wrapped in the maybe producing a new value. We can do this a few ways:

Let's suppose that we have function `f` of type `Int -> Int`. Let's suppose we have a `Maybe Int` if we want to produce a new value from it we do

```haskell
let m = Just 4
    f = (+1)
in case m of
  Just v -> f v
```
If you are astute then you would notice that the above code snippet is partial and will fail on Nothing. How do we make this total? First let's make it a named function that takes arguments

```haskell
mapply f m = case m of
    Just v  -> f v
    Nothing -> error "What to do?"
```

What is our type signature here?

```haskell
mapply :: (Int -> Int) -> Maybe Int -> Int
```

Notice this isn't total how do we deal with the Nothing case? The easy right answer is to keep it inside of Maybe.

```haskell
mapply :: (Int -> Int) -> Maybe Int -> Maybe Int
mapply f m = case m of
    Just v  -> Just $ f v
    Nothing -> Nothing
```

Now if we look this seems more general, let's apply polymorphism here and generalize this signature, we don't even need to change the body.

```haskell
mapply :: (a -> a) -> Maybe a -> Maybe a
```

We can actually go a step further and allow the function's result type to be anything, as there is nothing restricting it to `a`.

```haskell
mapply :: (a -> b) -> Maybe a -> Maybe b
```

If one types mapply into ghci without a type signature we will see that is actually infers the same type:
```haskell
mapply :: (t -> a) -> Maybe t -> Maybe a
```

Now maybe this doesn't look too intersting on its own, but let's check out the other types. We've already seen a similar situation with Lists, when working with `map`.

```haskell
map :: (a -> b) -> [a] -> [a]
```

```haskell
eapply :: (a -> b) -> (Either e) a -> (Either e) b
eapply f e = case e of
    Left err -> Left err
    Right x  -> Right $ f e
```

```haskell
mapply :: (a -> b) -> Maybe a    -> Maybe b
eapply :: (a -> b) -> Either e a -> Either e b
map    :: (a -> b) -> ([]) a     -> ([]) b
tapply :: (a -> b) -> BinTree a  -> BinTree b
```

Much of abstraction in Math and Computer science is about patterns. In this case the type shape shows us that there is something common going on here. If we replace all the uncommon bits with a type variable we will see that. 

```haskell
mapply :: (a -> b) -> f a    -> f b
eapply :: (a -> b) -> f a    -> f b
map    :: (a -> b) -> f a    -> f b
tapply :: (a -> b) -> f a    -> f b
```

So we know that type classes allow us to introduce a type variables that can only be instantiated for some types. 

Haskell has a Functor type class that has signature:

```haskell
class Functor f where
    fmap :: (a -> b) -> f a -> f b
```

We can just instantiate this for all of our types that statisfy this interface, plus a few laws. These laws make sure that code behaves as expected and allows for 
```haskell
instance Functor ([]) where
    fmap = map

instance Functor Maybe where
    fmap _ Nothing  = Nothing
    fmap f (Just v) = Just $ f v

instance Functor (Either e) where
    fmap _ (Left e)  = Left e
    fmap f(Right v) = Right $ f v
```

There are also some surprising instances for `(,) e` and `(->) e`.
```haskell
instance Functor ((,) e) where
    fmap f (c, v) = (c, f v)

instance Functor ((->) e) where
    fmap f g = g . f
```


