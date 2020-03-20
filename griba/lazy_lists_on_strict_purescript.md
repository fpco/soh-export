# Strictness surprises in PureScript Lazy lists

## PureScript a Haskell-like language for a JavaScript substrate

PureScript is a Haskell like programming language with some features biaised towards its main compilation backend which is JavaScript.

Between them strictness by default, JavaScript primitive types and literals where rectangular brackets are for the type Array instead of List, and some other [differences](https://github.com/purescript/documentation/blob/master/language/Differences-from-Haskell.md).

```haskell
$ pulp repl

PSCi, version 0.11.7
Type :? for help
import Prelude

> :t [1,2,3]
Array Int

> :t true 
Boolean

> :t 1.5
Number 

> :t unit
Unit

-- ranges don't use brackets and are defined for Arrays and for Lists

> import Data.Array ((..)) 
> :t 1..5                  
Array Int

```
## Building Lazy Lists in PureScript from recursive definitions

There is a Data.Enum module that defines the finite sequences *enumFromTo* and *enumFromThenTo* but not the infinite ones *enumFrom* and *enumFromThen*, and looking for a solution I have encountered some problems.

The lists are defined in a non Prelude package called *purescript-lists*

There are strict list modules Data.List, Data.List.NonEmpty and lazy lists ones Data.List.Lazy and Data.List.Lazy.NonEmpty based on a *Lazy* type wrapper defined in *purescript-lazy* Data.Lazy module.

Trying to define a Lazy sequence with a stop point seems to work apparently without problems but, defining one on Int(s) without stop point, makes the stack overflow.

```haskell
> import Data.List.Lazy

> 
> :paste
… f :: Int -> List Int
… f x | x > 0 = x : f (x-1)   
…     | otherwise = nil
…

> show $ take 3 $ f 9
"fromStrict ((Cons 9 (Cons 8 (Cons 7 Nil))))"

> :paste
… g :: Int -> List Int
… g x = x : g (x+1)   
…

> show $ take 3 $ g 0

/home/gabi64/lleng/purescript/soh/lazy2/.psci_modules/node_modules/$PSCI/index.js:13
var g = function (x) {
                 ^
RangeError: Maximum call stack size exceeded

>

```
It seems that strictness is playing a dirty trick here, evaluating the second *cons* parameter beforehand against our interest, but strict evaluation is in the nature of PureScript.

But if we use the *cons* function definition expression with the Data.Lazy *defer* function, instead of going with the (:) operator, the evaluation of the recursive case will be effectively deferred.

```haskell

-- | cons as defined In Data.List.Lazy

cons :: forall a. a -> List a -> List a
cons x xs = List $ defer \_ -> Cons x xs  -- Cons is a constructor of the type Step

-- defining our function the same way

> import Data.Lazy

> h x = List $ defer \_ -> Cons x $ h (x+1)

> show $ take 3 $ h 0
"fromStrict ((Cons 0 (Cons 1 (Cons 2 Nil))))"

```
Now it works!

Let's try defining *enumFrom*

```haskell
> import Data.Maybe
> import Data.Enum

> :paste
… enumFrom :: forall a. Enum a => a -> List a
… enumFrom x = List $ defer \_ -> case succ x of
…                                   Just x' -> Cons x $ enumFrom x'
…                                   Nothing -> Cons x nil
… 
> show $ take 3 $ enumFrom 0
"fromStrict ((Cons 0 (Cons 1 (Cons 2 Nil))))"

> show $ take 3 $ enumFrom 'A'
"fromStrict ((Cons 'A' (Cons 'B' (Cons 'C' Nil))))"

```

Let's try defining *enumFromThen* that corresponds in haskell to a stepped Enum sequence specifying the second value:

(it requires the class *BoundedEnum* that defines *toEnum* and *fromEnum* but there is no instance for Ints)

```haskell
> import Data.Lazy
> import Data.Maybe
> import Data.Enum
> import Data.List.Lazy

> :paste
… enumFromThen :: forall a. BoundedEnum a => a -> a -> List a
… enumFromThen x y = List $ defer \_ -> 
…                let step = fromEnum y - fromEnum x
…                    mbZ = toEnum (fromEnum y + step) :: Maybe a 
…                in case mbZ of
…                         Just z -> Cons x $ enumFromThen y z
…                         Nothing -> Cons x nil
…                                  
> show $ take 3 $ enumFromThen 'E' 'C'
"fromStrict ((Cons 'E' (Cons 'C' (Cons 'A' Nil))))"

```

To add an instance for Ints, because [*orphan instances* are disallowed](https://github.com/purescript/documentation/blob/master/language/Differences-from-Haskell.md#orphan-instances) the only way is to define it in the EnumBounded class module making a custom version.

```haskell
-- added to a local copy of Data.Enum

instance boundedEnumInt :: BoundedEnum Int where
  cardinality = Cardinality (top - bottom + 1)  -- overflowed but unused unless you use Ints as components of compound BoundedEnum instances
  
  fromEnum = id
  toEnum = Just
```
Now the package manager *bower* complaints that my local "Data.Enum" module name collides with the installed *purescript-enums* one.

```bash
bower uninstall purescript-enums
```
The *psc-package* packaging system is a better option as it is based, à la *stackage*, on sets of libraries that are guaranteed compatible.

Adding a *psc-package* project file.

```bash
git clone https://github.com/purescript/psc-package
cd psc-package
stack install

cd path/to/my-project
psc-package init  # interactively build the project file

# to start a new psc-package project: pulp --psc-package init 

```
The project file *psc-package.json* after adding some dependencies (with `psc-package install pkgName`):

```json
{
  "name": "lazy4",
  "set": "psc-0.11.7",
  "source": "https://github.com/purescript/package-sets.git",
  "depends": [
    "monoid",
    "foldable-traversable",
    "console",
    "prelude",
    "lists",
    "strings",
    "lazy"
  ]
}

```
Testing for Int sequences:

```haskell
module Main where

import Prelude
import Control.Monad.Eff (Eff)
import Control.Monad.Eff.Console (CONSOLE, log)

import Data.List.Lazy (List(..), nil, take, Step(Cons))
import Data.Lazy (defer)
import Data.Enum (class BoundedEnum, class Enum, fromEnum, succ, toEnum)
import Data.Maybe (Maybe(..))

enumFrom :: forall a. Enum a => a -> List a
enumFrom x = List $ defer \_ -> case succ x of
                                  Just x' -> Cons x $ enumFrom x'
                                  Nothing -> Cons x nil

enumFromThen :: forall a. BoundedEnum a => a -> a -> List a
enumFromThen x y = List $ defer \_ -> 
               let step = fromEnum y - fromEnum x
                   mbZ = toEnum (fromEnum y + step) :: Maybe a  
               in case mbZ of
                        Just z -> Cons x $ enumFromThen y z
                        Nothing -> Cons x nil
                                 
main :: forall e. Eff (console :: CONSOLE | e) Unit

main = log $ show $ take 5 $ enumFromThen 1 3
```
Output:

```bash
$ pulp --psc-package run

Compiling Data.Enum
Compiling Main
* Build successful.
fromStrict ((Cons 1 (Cons 3 (Cons 5 (Cons 7 (Cons 9 Nil))))))
```

## Avoiding unneeded heavy computations with Lazy parameters

```haskell
$ pulp repl
PSCi, version 0.11.7
Type :? for help

import Prelude

> import Data.Maybe (Maybe(..))
> import Data.Lazy (Lazy, defer, force)
> import Math (sqrt)

> :paste
… heavyComp :: forall a. Boolean -> a {- default -} -> Lazy a -> a

… heavyComp false def _ = def
… heavyComp true _ x = force x
… ^D

> heavyComp false 1.41 $ defer \_ -> sqrt 2.0
1.41

> heavyComp true 1.41 $ defer \_ -> sqrt 2.0
1.4142135623730951

```




