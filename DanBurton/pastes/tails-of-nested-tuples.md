# "Tails" of nested tuples

``` active haskell
(&) :: a -> b -> (a, b)
(&) = (,)
infixr 9 &

fullTuple :: (Int, (Bool, (Char, ())))
fullTuple = 3 & True & 's' & ()

-- replacing the "tails" of fullTuple with undefined.
-- Notice how they all have the same type.

undefined4 :: (Int, (Bool, (Char, ())))
undefined4 = undefined

undefined3 :: (Int, (Bool, (Char, ())))
undefined3 = 3 & undefined

undefined2 :: (Int, (Bool, (Char, ())))
undefined2 = 3 & True & undefined

undefined1 :: (Int, (Bool, (Char, ())))
undefined1 = 3 & True & 's' & undefined

main = print fullTuple
```