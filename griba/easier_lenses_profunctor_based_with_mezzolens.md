# Easier lenses, Profunctor based, with the Mezzolens library

## Profunctors

A profunctor, -- type `(p a b)`, is a generalisation of a mapping between two domains, [contravariant](https://en.wikipedia.org/wiki/Covariance_and_contravariance_%28computer_science%29) on the first (the input `a`) and covariant on the second (the output `b`)

``` haskell
-- here we pass to `dimap` a contramap to the first type parameter of the profunctor (p b c)
-- and a map to the second one

class Profunctor p where
  dimap :: (a -> b) -> (c -> d) -> p b c -> p a d
  
```
So `dimap (contramap :: a -> b) (map :: c -> d)` offers a transformation from the input `a` to the output `d`, as function of, a transformation from `b` to `c`.

Applying `dimap (contramap :: a -> b) (map :: c -> d)` to an instance of `p b c` gives `p a d` : an instance of the same type `p` transforming the input `a` to the output `d`.

Some [instances](https://hackage.haskell.org/package/mezzolens/docs/Mezzolens-Profunctor.html#t:Profunctor) of profunctors are

* Functions: <code>Profunctor (->)</code>
* [*Kleisli*](http://hackage.haskell.org/package/base/docs/Control-Arrow.html#t:Kleisli) arrows: <code>Functor f => Profunctor (Kleisli f)</code>

Note: An [*Arrow*](https://www.haskell.org/arrows/) is a generalisation of a computation with an input and an unrelated output. Monads can be lifted to arrows with the [*Kleisli*](https://wiki.haskell.org/Arrow_tutorial#Kleisli_Arrows) datatype.

For a more detailed introduction check

* the video ["The Extended Functor Family" by George Wilson](https://www.youtube.com/watch?v=JZPXzJ5tp9w)

* the School of Haskell article ["I love profunctors. They're so easy"](https://www.schoolofhaskell.com/user/liyang/profunctors)

## Profunctor based lenses with Mezzolens

[Mezzolens](https://hackage.haskell.org/package/mezzolens "The Mezzolens package") is a library written by Russell O'Connor.

A lens is defined here as a means to transform a structure, as function of, a transformation on one of its components. So you may combine the focused component mapping and the lens, over the initial structure, giving a transformed structure or a query result.

It takes the shape of a function of a profunctor on the component, giving a profunctor on the structure.

To characterize a lens ([lens generation](#the-lens-generator)) you need a component selection function, and a structure update or query function from the image of the selected component transformation.

Since uncurrying is needed for lens generators, we add the *Strong* class lenses over pairs, defined below, as requirement.

``` haskell
-- given 'a' the type of the target component of an input structure of type 'ta'
-- and 'b' the type of a transformed 'a', from which you produce 'tb'

type Optical p ta tb a b = p a b -> p ta tb   

type Lens ta tb a b = forall p. Strong p => p a b -> p ta tb

-- where
class Profunctor p => Strong p where
  _1 :: p a b -> p (a, c) (b, c)
  _2 :: p a b -> p (c, a) (c, b)
  
-- simple lenses:
-- Lens' is the type of lenses that don't change the types of component and structure.  
type Lens' ta a = Lens ta ta a a 

```

The methods of a *Strong profunctor* can be seen as lenses that provide a transformation of a Pair, as function of, the transformation of one of its components.

Since a function is an instance of it, applying its lens to a function, gives as result a function over pairs: 

``` haskell
Prelude> import Mezzolens as M
Prelude M> import Mezzolens.Profunctor as MP
Prelude M MP> import Data.Function ((&))       -- (&): backwards application

Prelude M MP>:t _1 (f :: a -> b) 
:: (a, c) -> (b, c)

Prelude M MP>:t _2 (f :: a -> b) 
:: (c, a) -> (c, b)

-- focusing with a lens and the function (+2), gets the structure updated
Prelude M MP> (1, 2) & _1 (+2)  
(3, 2)

-- the function `M.get` with a lens, queries the focused component
Prelude M MP> (1, 2) & M.get _1
1

```

Lens generation:

``` haskell
-- from Mezzolens.Unchecked

-- lens :: (ta -> a) {- component selection -}
        -> (b -> ta -> tb) {- struct. transf. with component image -}
        -> Lens ta tb a b
-- ^ lens getter updater 

```

As lenses are functions, they can compose, giving you access to items deep in a complex structure.

``` haskell
{-| file test-mezzo.hs -}
{-# LANGUAGE PackageImports #-}

import "mezzolens" Mezzolens (Lens', (^.), get, set)
import "mezzolens" Mezzolens.Unchecked (lens)
import Data.Function ((&))       -- (&): backwards application

data Arc = Arc {_degree, _minute, _second :: Int} deriving (Show)
data Location = Location {_latitude, _longitude :: Arc} deriving (Show)

-- structure to play with
locBcn = Location (arcFromDecimal 41.399423) (arcFromDecimal 2.128037)

-- decimal to sexagesimal latitude/longitude
arcFromDecimal :: Double -> Arc
arcFromDecimal v = Arc intPart mins secs
  where
     (intPart, fraction) = properFraction v
     -- divide negative lat/lon fractions towards 0 with quotRem
     (mins, secs) = truncate (fraction * 3600) `quotRem` 60

-- lens generators 

degree, minute, second :: Lens' Arc Int
degree = lens _degree (\v arc -> arc {_degree = v})
minute = lens _minute (\v arc -> arc {_minute = v})
second = lens _second (\v arc -> arc {_second = v})

latitude :: Lens' Location Arc
latitude = lens _latitude (\v loc -> loc {_latitude = v})

-- composing lenses
lensDegreesFromLatitude, lensMinutesFromLatitude, lensSecondsFromLatitude :: Lens' Location Int
lensDegreesFromLatitude = latitude . degree
lensMinutesFromLatitude = latitude . minute
lensSecondsFromLatitude = latitude . second

-- get the focused component
degreeLat = locBcn & get lensDegreesFromLatitude

-- (^.) is an infix version of 'get'

-- update the focused component:
-- (+2) is a unary function (unary functions are instances of ''Profunctor'')
-- applying the lens to the Profunctor value (+2) will give us a profunctor of the same type (a function) on the structure

locTwoDegreesNordOfBcn = locBcn & lensDegreesFromLatitude (+2)

-- 'set' applies the lens to a constant function
-- set lens v = lens (const v)

```

With Ghci

``` haskell
$ ghci
GHCi, version 7.10.3: http://www.haskell.org/ghc/  :? for help
Prelude> :l test-mezzo
[1 of 1] Compiling Main             ( test-mezzo.hs, interpreted )
Ok, modules loaded: Main.

*Main> import Mezzolens as M

*Main M> :t M.get lensDegreesFromLatitude
M.get lensDegreesFromLatitude :: Location -> Int

*Main M> :t lensDegreesFromLatitude (+2)
lensDegreesFromLatitude (+2) :: Location -> Location

```

If you want a monadic transformation, wrap the monadic computation as a [*Kleisli arrow*](http://hackage.haskell.org/package/base/docs/Control-Arrow.html#t:Kleisli). Mezzolens brings an instance of Profunctor for Kleisli arrows.

Applying the lens to an *arrow* on the component returns an *arrow* on the structure, where you can get the monadic function inside with the Kleisli accessor *runKleisli*.

``` haskell
*Main M> import Control.Arrow as A

*Main M A> :{
*Main M A| let myMonadicCompoTransf :: Monad m => Int -> m Int
*Main M A|     myMonadicCompoTransf = return . (+2)
*Main M A| :}
*Main M A> :t runKleisli . lensDegreesFromLatitude . Kleisli $ myMonadicCompoTransf
runKleisli . lensDegreesFromLatitude . Kleisli $ myMonadicCompoTransf
  :: Monad m => Location -> m Location
```

But a monadic use is questionable unless you use [state lenses](https://hackage.haskell.org/package/mezzolens/docs/Mezzolens-State-Lazy.html) which I will not cover here.

### Prisms

Prisms are lenses where the component selection may not succeed, as they target parts of *sum types*.

Since the component selection results in an Either, we will require profunctors with the *Choice* class lenses over *Either*, needed in [prism generators](#the-prism-generator)

``` haskell
type Prism ta tb a b = forall p. Choice p => p a b -> p ta tb

-- where
class Profunctor p => Choice p where
  _Left :: p a b -> p (Either a c) (Either b c)
  _Right :: p a b -> p (Either c a) (Either c b)

```
A Choice profunctor is one that brings lenses methods that let you transform an Either variant content with the input profunctor.

``` haskell
-- a Prism generator from Mezzolens.Unchecked
prism :: (ta -> Either tb a) {- component selection match -}
      -> (b -> tb)           {- struct. build up from component image -}
      -> Prism ta tb a b
-- ^ prism match build
```

* the first *prism* generator parameter (component selection) should give either the component in case of success, or a result structure in case of fail.

* the second (final structure from component image) builds a sum type variant of *tb* from the image (type *b*) of the selected component

Example on the *head* of a List.

``` haskell
*Main> import Mezzolens as M
*Main M> import Mezzolens.Unchecked as MU
*Main M MU> :set -XLambdaCase

*Main M MU> :{
*Main M MU| let headMatch :: [a] -> Either [a] a
*Main M MU|     headMatch = \case
*Main M MU|                    x : _ -> Right x  -- target variant
*Main M MU|                    [] -> Left []     -- fail case giving a zero/empty structure

*Main M MU| let headBuild = \x -> [x]

*Main M MU| let _Head = MU.prism headMatch headBuild
*Main M MU| :}
            
-- Prism query (optional result)
*Main M MU> [locBcn] ^? (_Head . lensDegreesFromLatitude)
Just 41         

-- Update on a target variant
*Main M MU> [locBcn] & (_Head . lensDegreesFromLatitude) (+2)
[Location {_latitude = Arc {_degree = 43, _minute = 23, _second = 57}, _longitude = Arc {_degree = 2, _minute = 7, _second = 40}}]

-- Update on the non target variant
*Main M MU> [] & (_Head . lensDegreesFromLatitude) (+2)
[]
```


### Traverses and folds


The *Wandering* class and the function *gets*:

* *wander* offers a lens that let you traverse a *Traversable*, transforming its elements with the input profunctor.

* The *gets* function offers a means to get a fold of the [codomain](https://en.wikipedia.org/wiki/Codomain) (target set) of mapping the items focused by a traversal to a Monoid.

``` haskell
-- from Mezzolens.Profunctor
class (Strong p, Choice p) => Wandering p where
  wander :: Traversable f => p a b -> p (f a) (f b)
  
-- from Mezzolens
gets :: Monoid r => Fold ta tb a b -> (a -> r) -> ta -> r 

-- from Mezzolens.Optics
type Fold ta tb a b = forall p. (OutPhantom p, Wandering p) => Optical p ta tb a b
-- (OutPhantom p) means that the profunctor output type is coercible

```

with Ghci and the test code loaded

``` haskell
*Main> import Mezzolens as M
*Main M> import Mezzolens.Profunctor as MP

-- traverse a list, updating its elements

*Main M MP> [locBcn, locTwoDegreesNordOfBcn] & (MP.wander . lensDegreesFromLatitude) (+3)
[Location {_latitude = Arc {_degree = 44, _minute = 23, _second = 57}, _longitude = Arc {_degree = 2, _minute = 7, _second = 40}},Location {_latitude = Arc {_degree = 46, _minute = 23, _second = 57}, _longitude = Arc {_degree = 2, _minute = 7, _second = 40}}]


-- (toListOf): {toListOf lens = gets lens pure} it uses 'pure' to map the focused elements to a generic Applicative that when constraint to a List gives List singletons, folded with the List Monoid instance into a concatenation.

*Main M MP> [locBcn, locTwoDegreesNordOfBcn] & M.toListOf (MP.wander . lensDegreesFromLatitude) :: [Int]  
[41,43]

-- (^..) is an infix version of toListOf

*Main M MP> [locBcn, locTwoDegreesNordOfBcn] ^.. (MP.wander . lensDegreesFromLatitude) :: [Int]
[41,43]

-- (sumOf): maps the focused elements to the Monoid Sum and combine. See next section.

*Main M MP> [locBcn, locTwoDegreesNordOfBcn] & M.sumOf (MP.wander . lensDegreesFromLatitude)
84

```

### Understanding folds like *sumOf*

We take *Sum* as the *gets* mapping function, fetching the result with its  accessor *getSum*:

``` haskell
sumOf lens = getSum . gets lens Sum

gets :: Optical (SubStar (Constant r)) ta tb a b -> (a -> r) -> ta -> r
gets lens f = getConstant . h
 where
  Kleisli h = lens (Kleisli (Constant . f))
  
-- type SubStar = Kleisli      -- from Mezzolens.Profunctor

-- substituting ''h'' extracting it with ''runKleisli''

gets lens f = getConstant . (runKleisli (lens (Kleisli (Constant . f))))
  
```
    
using *wander* as *lens* in the *gets* expression, then reducing *wander* on a Kleisli arrow:


``` haskell
Prelude ...> let g f =  getConstant . (runKleisli (wander (Kleisli (Constant . f))))

-- from Mezzolens code:

instance Applicative f => Wandering (Kleisli f) where
  wander (Kleisli h) = Kleisli (traverse h)

-- since there is an Applicative instance for Constant 

Prelude ...> let g f = getConstant . (runKleisli (Kleisli (traverse (Constant . f))))

-- since {runKleisli . Kleisli == id}

Prelude ...> let g f = getConstant . (traverse (Constant . f))

Prelude ...> g Sum [1,2,3]
Sum {getSum = 6}
```

understanding *Constant* from Data.Functor.Constant

``` haskell
-- Constant has a phantom type parameter, so (Constant a) can be seen as a structure:
newtype Constant a b = Constant { getConstant :: a }

-- the (Constant a) instance for Applicative is defined only for 'a' Monoid
instance (Monoid a) => Applicative (Constant a) where
    pure _ = Constant mempty       -- applicative combinators will be ignored !!
    Constant x <*> Constant y = Constant (x `mappend` y)

Prelude ...> :t traverse
traverse
  :: (Applicative f, Traversable t) => (a -> f b) -> t a -> f (t b)
```

Since Constant is the Applicative type in the traversal, because in its instance *"pure"* ignores the combinator, whatever the traversable instance, you will get the Monoid composition instead, folding left to right as the container is traversed.


``` haskell
Prelude ...> :t traverse Constant
traverse Constant
  :: (Traversable t, Monoid a) => t a -> Constant a (t b)

-- you will not get (t b) inside Constant
-- but ''mempty'' from (''pure'' traversable_combinator), ''mappend''ed to all a's

Prelude ...> traverse Constant [Sum 1, Sum 2, Sum 3]
Constant (Sum {getSum = 6})
```
as summary:

``` haskell
Prelude ...> :t gets wander
gets wander :: (Traversable f, Monoid r) => (a -> r) -> f a -> r

Prelude ...> :t gets wander Sum
gets wander Sum :: (Num a, Traversable f) => f a -> Sum a

Prelude ...> gets wander Sum [1,2,3]
Sum {getSum = 6}
```

### Some generators

#### The *lens* generator

``` haskell
type Optical p ta tb a b = p a b -> p ta tb

type Lens ta tb a b = forall p. Strong p => Optical p ta tb a b

lens :: (ta -> a) {- getter: component selection -} 
     -> (b -> ta -> tb) {- setter: struct. transf. with component image -} 
     -> Lens ta tb a b

-- we need to find parameters for ''dimap''

-- let's obtain an output map for our lens, pairing the setter parameters
> :t uncurry setter
:: (b, ta) -> tb

-- a matching contramap changing the component woud be
-- :: ta -> (a, ta)

-- with (&&&) from Arrow -- it pairs the results of two arrows of the same input type

-- from Control.Arrow.Arrow class
-- (&&&) :: Arrow a => a b c -> a b c' -> a b (c, c')

> :t getter &&& id  -- since functions are instances of Arrow
:: ta -> (a, ta)

> :t dimap \ 
      (getter &&& id)  {- :: ta -> (a, ta) -} \
      (uncurry setter) {- :: (b, ta) -> tb -}
:: Profunctor p => p (a, ta) (b, ta) -> p ta tb

```

With `dimap (getter &&& id) (uncurry setter)` you have a lens from the structure to a pair.

Composing it with `_1` (the lens on the first item of a pair) we further put the focus on the component transformation `p a b`, adding the requirement that the profunctor should implement the class Strong where `_1` is defined.


``` haskell
import Control.Arrow ((&&&), (|||))

type Optical p ta tb a b = p a b -> p ta tb

type Lens ta tb a b = forall p. Strong p => Optical p ta tb a b

-- the generator:

lens :: (ta -> a)  {- component selection -}
     -> (b -> ta -> tb) {- structure transf. with component image -}
     -> Lens ta tb a b
lens getter setter = lensStructFromPair . _1
   where
      lensStructFromPair :: Profunctor p 
                         => p (a, ta) (b, ta) 
                         -> p ta tb
      lensStructFromPair = dimap (getter &&& id) (uncurry setter)

-- _1 is a lens on the first component of a pair
-- _1 :: Strong p => p a b -> p (a, c) (b, c)

```

#### The *prism* generator

Prisms target sum type variants

``` haskell

type Prism ta tb a b = forall p. Choice p => Optical p ta tb a b

prism :: (ta -> Either tb a) {- component selection match -} 
      -> (b -> tb)        {- build final struct. from component image -} 
      -> Prism ta tb a b

-- `match` should give an (Either tb a) which means a component `a` in case of success or a zero result structure of type `tb` in case of fail

-- `build` should wrap the transformed component `b` in a variant of the sum type structure of type `tb` 

-- we need to find parameters for ''dimap''

-- we take ''(match :: ta -> Either tb a)'' as the contramap operation

-- a matching map switching the comp. type with its image 
--    would be ''(Either tb b -> tb)''

-- we have ''(build :: b -> tb)''

-- (|||) from ArrowChoice, splices the input types of two arrows of the same result type in an Either

-- from Control.Arrow.ArrowChoice class
-- (|||) :: ArrowChoice a => a b d -> a c d -> a (Either b c) d

> :t id ||| build  -- since functions are instances of ArrowChoice
:: Either tb b -> tb

> :t dimap \
        match          {- :: ta -> Either tb a -} \
        (id ||| build) {- :: Either tb b -> tb -}
:: Profunctor p => p (Either tb a) (Either tb b) -> p ta tb

prism match build = lensStructFromEither . _Right
    where 
      lensStructFromEither :: Profunctor p 
                              => p (Either tb a) (Either tb b)
                              -> p ta tb
      lensStructFromEither = dimap match (id ||| build)

-- _Right is a lens on the second domain of an Either
-- _Right :: Choice p => p a b -> p (Either c a) (Either c b)
```
With `dimap match (id ||| build)` you have a lens from the Structure to an Either having the component as Right or a Left `tb` structure in case of mismatch.

Composing it with the lens *\_Right*, will make possible to focus on the component, adding the requirement that the profunctor should also implement the class Choice where *\_Right* is defined.


#### Isos

Iso's are lenses that convert a profunctor on structures, on one over an Isomorphic representation

``` haskell
type Iso ta tb a b = forall p. Profunctor p => Optical p ta tb a b

-- the generator:

iso :: (ta -> a) -> (b -> tb) -> Iso ta tb a b
iso = dimap

-- as example, the lens _swap converts a profunctor on pairs into a profunctor over its swapped type
_swap :: Iso (a,b) (c,d) (b,a) (d,c)
_swap = iso swap swap
```
