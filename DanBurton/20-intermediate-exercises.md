# 20 intermediate exercises


#Instructions

Fill in the blanks (which are marked `error "todo"`) with a nontrivial implementation that typechecks. I recommend that you use the run button after implementing each one to make sure that it still typechecks. Let the types guide you! If you have trouble implementing a typeclass method, then try writing out the specialized type that it should have. For example, `furry` imlpemented for `[]` should have the type `(a -> b) -> [a] -> [b]`.

``` active haskell
-- show Fluffy instances
class Fluffy f where
  furry :: (a -> b) -> f a -> f b

-- Exercise 1
-- Relative Difficulty: 1
instance Fluffy [] where
  furry = error "todo"

-- Exercise 2
-- Relative Difficulty: 1
instance Fluffy Maybe where
  furry = error "todo"

-- Exercise 3
-- Relative Difficulty: 5
instance Fluffy ((->) t) where
  furry = error "todo"

newtype EitherLeft b a = EitherLeft (Either a b)
newtype EitherRight a b = EitherRight (Either a b)

-- Exercise 4
-- Relative Difficulty: 5
instance Fluffy (EitherLeft t) where
  furry = error "todo"

-- Exercise 5
-- Relative Difficulty: 5
instance Fluffy (EitherRight t) where
  furry = error "todo"
-- /show

main = putStrLn "It typechecks!"
```

``` active haskell
-- show Misty instances
class Misty m where
  banana :: (a -> m b) -> m a -> m b
  unicorn :: a -> m a

-- Exercise 7
-- Relative Difficulty: 2
instance Misty [] where
  banana = error "todo"
  unicorn = error "todo"

-- Exercise 8
-- Relative Difficulty: 2
instance Misty Maybe where
  banana = error "todo"
  unicorn = error "todo"

-- Exercise 9
-- Relative Difficulty: 6
instance Misty ((->) t) where
  banana = error "todo"
  unicorn = error "todo"

newtype EitherLeft b a  = EitherLeft (Either a b)
newtype EitherRight a b = EitherRight (Either a b)

-- Exercise 10
-- Relative Difficulty: 6
instance Misty (EitherLeft t) where
  banana = error "todo"
  unicorn = error "todo"

-- Exercise 11
-- Relative Difficulty: 6
instance Misty (EitherRight t) where
  banana = error "todo"
  unicorn = error "todo"

-- /show

main = putStrLn "It typechecks!"
```

``` active haskell
-- show Additional Misty functions
class Misty m where
  banana :: (a -> m b) -> m a -> m b
  unicorn :: a -> m a


-- Exercise 6
-- Relative Difficulty: 3
-- (use banana and/or unicorn)
furry' :: (Misty m) => (a -> b) -> m a -> m b
furry' = error "todo"

-- Exercise 12
-- Relative Difficulty: 3
jellybean :: (Misty m) => m (m a) -> m a
jellybean = error "todo"

-- Exercise 13
-- Relative Difficulty: 6
apple :: (Misty m) => m a -> m (a -> b) -> m b
apple = error "todo"

-- Exercise 14
-- Relative Difficulty: 6
moppy :: (Misty m) => [a] -> (a -> m b) -> m [b]
moppy = error "todo"

-- Exercise 15
-- Relative Difficulty: 6
-- (bonus: use moppy)
sausage :: (Misty m) => [m a] -> m [a]
sausage = error "todo"

-- Exercise 16
-- Relative Difficulty: 6
-- (bonus: use apple + furry')
banana2 :: (Misty m) => (a -> b -> c) -> m a -> m b -> m c
banana2 = error "todo"

-- Exercise 17
-- Relative Difficulty: 6
-- (bonus: use apple + banana2)
banana3 :: (Misty m) => (a -> b -> c -> d) -> m a -> m b -> m c -> m d
banana3 = error "todo"

-- Exercise 18
-- Relative Difficulty: 6
-- (bonus: use apple + banana3)
banana4 :: (Misty m) => (a -> b -> c -> d -> e) -> m a -> m b -> m c -> m d -> m e
banana4 = error "todo"
-- /show

main = putStrLn "It typechecks!"
```

``` active haskell
-- show State instances for Fluffy and Misty
class Misty m where
  banana :: (a -> m b) -> m a -> m b
  unicorn :: a -> m a

class Fluffy f where
  furry :: (a -> b) -> f a -> f b


newtype State s a = State {
  state :: (s -> (s, a))
}

-- Exercise 19
-- Relative Difficulty: 9
instance Fluffy (State s) where
  furry = error "todo"

-- Exercise 20
-- Relative Difficulty: 10
instance Misty (State s) where
  banana = error "todo"
  unicorn = error "todo"
-- /show

main = putStrLn "It typechecks!"
```
