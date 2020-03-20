# Joseph Abrahamson's Basic Lensing tutorial digested and extended with action examples and links

---

# intro

## source

This is a digest of [Joseph Abrahamson](http://jspha.com/)'s [Basic Lensing FPComplete tutorial](https://www.fpcomplete.com/user/tel/basic-lensing).  I have

-   shown his examples in action

-   added links

-   verified examples via unit testing

see also:

-   [Benjamin C. Pierce](http://www.cis.upenn.edu/~bcpierce/) et. al. "Quotient Lenses" ([pdf](http://www.cis.upenn.edu/~bcpierce/papers/quotient-lenses.pdf))

-   [Edward Kmett](https://plus.google.com/u/0/113063331545548237308/about)'s [`Control.Lens`](http://hackage.haskell.org/package/lens)

-   Simon Peyton Jones [video](http://skillsmatter.com/podcast/scala/lenses-compositional-data-access-and-manipulation) on Lens at Skills Matter Oct 2013

-   <http://www.haskellforall.com/2013/05/program-imperatively-using-haskell.html>

-   <http://www.haskellforall.com/2012/01/haskell-for-mainstream-programmers_28.html>

## A lens "focuses" on a smaller part of a larger object.

    {-# LANGUAGE TemplateHaskell #-}
    
    module UserTelBasicLensing where
    
    import Control.Lens
    import Test.HUnit -- for unit testing examples
    
    data Arc      = Arc      { _degree   :: Int, _minute    :: Int, _second :: Int } deriving (Eq, Show)
    data Location = Location { _latitude :: Arc, _longitude :: Arc }                 deriving (Eq, Show)

Underscores in record names above are a `Control.Lens` convention for generating template haskell (TH).

The following is a TH splice. It generates lenses automatically based on record functions in `Location`:

    $(makeLenses ''Location)

Above creates two lenses:

    :t latitude
    -- latitude  :: Functor f => (Arc -> f Arc) -> Location -> f Location
    
    :t longitude
    -- longitude :: Functor f => (Arc -> f Arc) -> Location -> f Location

The [`type` of `Lens`](http://hackage.haskell.org/package/lens-3.9.2/docs/Control-Lens-Lens.html#t:Lens) is

    type Lens s t a b = forall f. Functor f => (a -> f b) -> s -> f t
    type Lens' s a = Lens s s a a

So the types above can be viewed as:

    -- latitude  :: Lens' Location Arc
    -- longitude :: Lens' Location Arc

---

# lenses used as getters/setters

A lens is a function that get be used to get (via `view`) or set (via `set`) a part of a data structure.

In this example case, `longitude` (likewise for `latitude`) can be
used to `view` or `set` that part of `Location` :

**`view`**

    :t view
    -- view :: Control.Monad.Reader.Class.MonadReader s m => Getting a s a -> m a
    
    :i Getting
    -- type Getting r s a = (a -> Accessor r a) -> s -> Accessor r s
    
    :i Accessor
    -- newtype Accessor r a = Accessor {runAccessor :: r}
    
    :t runAccessor
    -- runAccessor :: Accessor r a -> r
    
    :t view longitude
    -- view longitude :: Control.Monad.Reader.Class.MonadReader Location m => m Arc
    --   i.e.,:
    -- view longitude :: Location -> Arc

**`set`**

    :t set
    -- set :: ASetter s t a b -> b -> s -> t
    
    :i ASetter
    -- type ASetter s t a b = (a -> Mutator b) -> s -> Mutator t
    
    :i Mutator
    -- newtype Mutator a = Control.Lens.Internal.Setter.Mutator {Control.Lens.Internal.Setter.runMutator :: a}
    
    :t Control.Lens.Internal.Setter.runMutator
    -- Control.Lens.Internal.Setter.runMutator :: Mutator a -> a
    
    :t set longitude
    -- set longitude :: Arc -> Location -> Location

The following examples use unit tests (rather than GHCI input/output) to ensure correctness.

    t :: (Eq a) => (Show a) => String -> a -> a -> Test
    t testName actual expected  = TestCase $ assertEqual testName expected actual
    
    l = Location (Arc 1 2 3) (Arc 4 5 6)
    
    t01 = t "01"
          (view longitude l)
          (Arc 4 5 6)
    
    t02 = t "02"
          (set longitude (Arc 40 50 60) l)
          (Location (Arc 1 2 3) (Arc 40 50 60))
    
    t03 = t "03"
          l
          (Location (Arc 1 2 3) (Arc 4  5  6))

## getters/setters without lenses

Lenses are useful because, in *immutable* Haskell, to change nested
fields in a data structure you need to recreate all the objects
wrapped around the value that you are changing:

    getLongitudeR :: Location -> Arc
    getLongitudeR (Location { _longitude = lat }) = lat
    
    setLongitudeR :: Arc -> Location -> Location
    setLongitudeR lat loc = loc { _longitude = lat }
    
    t04 = t "04"
          (setLongitudeR (Arc 44 55 66) l)
          (Location (Arc 1 2 3) (Arc 44 55 66))

The lens version does this for you "automatically".

---

# another way to build lenses using `lens`

    :t lens
    -- lens :: Functor f => (s -> a) -> (s -> b -> t) -> (a -> f b) -> s -> f t
    --   i.e.,:
    -- lens :: (c -> a) -> (c -> a -> c) -> Lens' c a

The following are identical:

    :t lens getLongitudeR (flip setLongitudeR)
    -- lens getLongitudeR (flip setLongitudeR)      :: Functor f => (Arc -> f Arc) -> Location -> f Location
    
    :t lens (view longitude) (flip $ set longitude)
    -- lens (view longitude) (flip $ set longitude) :: Functor f => (Arc -> f Arc) -> Location -> f Location
    
    :t longitude
    -- longitude                                    :: Functor f => (Arc -> f Arc) -> Location -> f Location

Above shows a law of lenses: for all lenses, `l`:

    l == lens (view l) (flip $ set l)

---

# lens benefits

Benefits of wrapping getters/setters together:

-   export just the lenses instead of the record functions

-   use other kinds of combinators to operate on these lenses for affecting the "focal" record values

E.g., modification via combinator named `over`:

    {-# ANN modifyLongitude "HLint: ignore Redundant bracket" #-}
    modifyLongitude  :: (Arc -> Arc) -> (Location -> Location)
    modifyLongitude  f = longitude `over` f
    
    arcTimes11 :: Arc -> Arc
    arcTimes11 (Arc a b c) = Arc (a*11) (b*11) (c*11)
    
    longitudeTimes11 :: Location -> Location
    longitudeTimes11 = modifyLongitude arcTimes11
    
    t05 = t "05"
          (longitudeTimes11 l)
          (Location (Arc 1 2 3) (Arc 44 55 66))

`over` lifts given function between getter and setter to create a
function which modifies a part of the greater whole.

---

# composing lens via `(.)` to go deeper into structure

    $(makeLenses ''Arc)

    :t degree
    -- degree :: Functor f => (Int -> f Int) -> Arc -> f Arc
    
    :t minute
    -- minute :: Functor f => (Int -> f Int) -> Arc -> f Arc
    
    :t second
    -- second :: Functor f => (Int -> f Int) -> Arc -> f Arc

Now use `(.)` to get deeper inside `Location`:

    :t (.)
    -- (.) :: (b -> c) -> (a -> b) -> a -> c
    --   i.e.,:
    -- (.) :: Lens' a b -> Lens' b c -> Lens' a c
    
    :t longitude . degree
    -- longitude . degree :: Functor f => (Int -> f Int) -> Location -> f Location
    --   i.e.,:
    -- longitude . degree :: Lens' Location Int
    
    :t view (longitude . degree)
    -- view (longitude . degree) :: Control.Monad.Reader.Class.MonadReader Location m => m Int
    --   i.e.,:
    -- view (longitude . degree) :: Location -> Int
    
    :t set  (longitude . degree)
    -- set  (longitude . degree) :: Int -> Location -> Location

Using the above type signatures as a guide, we can get/set specific parts of `Location`:

    t06 = t "06"
          (view (longitude . degree) l)
          4
    
    t07 = t "07"
          (set  (longitude . degree) 202 l)
          (Location (Arc 1 2 3) (Arc 202 5 6))
    
    t08 = t "08"
          (view (longitude . second) l)
          6
    
    t09 = t "09"
          (set  (longitude . second) 202 l)
          (Location (Arc 1 2 3) (Arc 4 5 202))

## combining lenses as pairs or `Either`

pairs, i.e., **`(,)`**

    p :: Lens' (Location, Location) (Arc, Arc)
    p = latitude `alongside` longitude
    
    l10  = Location (Arc  10  20  30) (Arc  40  50  60)
    l100 = Location (Arc 100 200 300) (Arc 400 500 600)
    
    t10 = t "10"
          (view p (l10, l100))
          (Arc 10 20 30, Arc 400 500 600)
    
    t11 = t "11"
          (set p (Arc 111 222 333, Arc 444 555 666) (l10, l100))
          (Location (Arc 111 222 333) (Arc 40 50 60), Location (Arc 100 200 300) (Arc 444 555 666))

**`Either`**

    ei :: Lens' (Either Arc Arc) Int
    ei = choosing degree second
    
    a10  = Arc  10  20  30
    a100 = Arc 100 200 300
    
    t12 = t "12"
          (view ei (Left   a10))
          10
    t13 = t "13"
          (view ei (Right  a10))
          30
    t14 = t "14"
          (view ei (Left  a100))
          100
    t15 = t "15"
          (view ei (Right a100))
          300
    
    t16 = t "16"
          (set ei (-1) (Left   a10))
          (Left (Arc (-1) 20 30))
    t17 = t "17"
          (set ei (-1) (Right a100))
          (Right (Arc 100 200 (-1)))

---

# summary

lens abstraction

-   idea of holding on to a value that's focused on a smaller part of a larger type

-   algebra for combining (via pairs and eithers, products and coproducts), composing, and modifying these values

-   subsumes record syntax

-   minimizes book-keeping on getters and setters

Lens can do *lots* more.

Feedback/discussion at: <http://haroldcarr.com/posts/2013-10-06-intro-to-haskell-lenses.html>

---

# example accuracy

    main = runTestTT $ TestList[t01, t02, t03, t04, t05, t06, t07, t08, t09, t10, t11, t12, t13, t14, t15, t16, t17]

    main
    -- Counts {cases = 17, tried = 17, errors = 0, failures = 0}

Thanks for to [Gabriel Gonzalez](http://www.haskellforall.com/) for useful feedback incorporated before publishing.

