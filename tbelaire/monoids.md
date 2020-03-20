# Monoids

Lets look at one of my favourite typeclasses, the humble `Monoid`.

A `Monoid` is a set of things (call them `a`), and a function `<> :: a -> a -> a` with the property that
``` haskell
(x <> y) <> z == x <> (y <> z)
```
for all the things in our set.  This is called accociativity.  You can immediately think of things that fit this, such as 
``` haskell
    (1 + 2) + 3 == 6 == 1 + (2 + 3)
```
Or how about
``` haskell
("A " ++ "Lazy ") ++ "Cat" == "A Lazy Cat" == "A " ++ ("Lazy " ++ "Cat")
```

Now, there's also one more thing that is required to be a `Monoid`, and that's an element like `0` or `""`, called `mempty` such that

``` haskell
x <> mempty == x == mempty <> x
```

So, you can see that `"" ++ "Cat" == "Cat"`, for example, which is how it got the name `mempty` (**M**onoid **empty**).  It is what is called an identity for `<>`.  It must have the property that:
``` haskell
mempty <> x = x = mempty <> x
```
You can see how this holds for lists.

```haskell
[] ++ xs == xs == xs ++ []

0  +  x  == x  == 0  +  x
```

Also, the `<>` function goes by the name `mappend` since it works like the `append` function on lists, but for `Monoids`.  The nice `<>` name is defined in `Data.Monoid`.



``` active haskell
-- show
infixr 6 <>

class Monoid a where
    (<>) :: a -> a -> a
    mempty :: a

instance Monoid [a] where
    a <> b = a ++ b
    mempty = []

instance Monoid Int where
    a <> b = a + b
    mempty = 0
    
{-
    Now, how about we make a function to "sum" a list.
    Or, thinking about it from the list point of view, concatenate a list.
-}
    

mconcat []     = mempty
mconcat (x:xs) = x <> (mconcat xs)

example1 :: Int  -- I need this here since my Monoid instance is just for Ints
example1 = mconcat [1,2,3]
example2 :: String
example2 = mconcat ["The ", "Happy ", "Bear"]
-- /show
main = do putStrLn $ "example1 -> " ++ show example1
          putStrLn $ "example2 -> " ++ show example2

```


Let's take a look at some of the other instances in there.

    Num a => Monoid (Product a)	 
    Num a => Monoid (Sum a)	 
    
Oh, well then, we seem to have overlooked something.  For numbers, not only does `+` and `0` work, but also `*` and `1`!  
``` haskell
(a*b)*c == a*(b*c)
1*x == x
```
Well, we can't have *two* instances for numbers.  Haskell won't like that, it wouldn't know what we mean when we say `mempty :: Int`.  So, what we do is make some new types.

``` active haskell
import Prelude hiding (sum, product)

infixr 6 <>

class Monoid a where
    (<>) :: a -> a -> a
    mempty :: a

instance Monoid [a] where
    a <> b = a ++ b
    mempty = []

mconcat []     = mempty
mconcat (x:xs) = x <> (mconcat xs)

-- show
data Product a = Product a    -- It would make sense to newtype these
data Sum     a = Sum     a    -- but I'm not talking about newtype here

getProduct (Product x) = x
getSum     (Sum     x) = x     -- You could get these for free with record notation

-- It also wouldn't hurt to derive a Num instance, but I won't here

instance (Num a) => Monoid (Product a) where
    mempty = Product 1
    (Product a) <> (Product b) = Product (a * b)

instance (Num a) => Monoid (Sum a) where
    mempty = Sum 0
    (Sum a) <> (Sum b) = Sum (a + b)

sum xs = unSum $ mconcat $ map Sum xs
product xs = unProduct $ mconcat $ map Product xs

test :: Int
test = 42
-- /show
main = do putStrLn $ "sum     [2,3,4] -> " ++ (show $ sum [2,3,4])
          putStrLn $ "product [2,3,4] -> " ++ (show $ product [2,3,4])
          putStrLn $ "test -> " ++ (show test)
```
At this point, I think you can read through [Data.Monoid](http://hackage.haskell.org/packages/archive/base/latest/doc/html/src/Data-Monoid.html)
and learn about more of the interesting instances such as `Any` and `All`, `First` and `Last`, and Lexographic `Ordering`.  (Note the use of `mappend` instead of `<>`, since that was how it was frist defined.  I used `<>` in my `Monoid` class, since it is easier to read)

Now, so far, I've only been working on lists, but that's not all we can work over.  Let's make an very simple tree.  
``` active haskell
{-# LANGUAGE DeriveFunctor #-}
import Data.Monoid

data Tree a = Leaf a | Branch (Tree a) (Tree a) deriving (Functor, Show, Eq)

fringe :: (Monoid a) => Tree a -> a
fringe (Branch l r) = (fringe l) <> (fringe r)
fringe (Leaf a)     = a

exTree =(Branch (Branch (Leaf 5) (Leaf 8))
                (Branch (Leaf 3)
                        (Branch (Leaf 7)
                                (Leaf 4))))
                                
main = do putStrLn $ "Sum :" ++ (show $ getSum $ fringe $ fmap Sum $ exTree)
          putStrLn $ "List: " ++ fringe (fmap (\x -> show x ++", ") exTree)
```

Have fun.
                        
