# Lens exercise

Exercises as seen here:

[http://artyom.me/lens-over-tea-1](http://artyom.me/lens-over-tea-1)

```active haskell
{-# LANGUAGE RankNTypes #-}

type Lens s t a b = forall f. Functor f => (a -> f b) -> s -> f t
type Lens' s a = Lens s s a a


-- _1 :: Functor f => (a -> f b) -> (a, x) -> f (b, x)
_1 :: Lens (a, x) (b, x) a b
_1 f (a, x) = undefined

-- _2 :: Functor f => (a -> f b) -> (x, a) -> f (x, b)
_2 :: Lens (x, a) (x, b) a b
_2 f (x, a) = undefined

-- Make a lens out of a getter and a setter.
lens :: (s -> a) -> (s -> b -> t) -> Lens s t a b
lens get set f s = undefined

-- Combine 2 lenses to make a lens which works on Either.
choosing :: Lens s1 t1 a b -> Lens s2 t2 a b
         -> Lens (Either s1 s2) (Either t1 t2) a b
choosing = undefined

-- Modify the target of a Lens and return the result. (Bonus points if you
-- do it without lambdas and defining new functions.)
(<%~) :: Lens s t a b -> (a -> b) -> s -> (b, t)
(<%~) = undefined

-- Modify the target of a Lens, but return the old value.
(<<%~) :: Lens s t a b -> (a -> b) -> s -> (a, t)
(<<%~) = undefined

-- There's a () in every value. (No idea what this one is for, maybe will
-- find out later.)
united :: Lens' a ()
united = undefined

main :: IO ()
main = putStrLn "It compiled!"
```
