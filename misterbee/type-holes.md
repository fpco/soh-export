# Type Holes

[TypeHoles](https://ghc.haskell.org/trac/ghc/ticket/5910) demo.


Note that this program runs, and prints a type error when a hole `_` is encountered at runtime.


Code copied from [haskell.org/haskellwiki/GHC/TypeHoles](http://www.haskell.org/haskellwiki/GHC/TypeHoles)

To run in IDE, after opening in IDE,  go to Settings (hamburger icon in top-right) and set

 * Environment: GHC 7.8
 * Compiler flags: -fdefer-type-errors
 

``` active haskell
{-# LANGUAGE TypeHoles #-}
{-# LANGUAGE FlexibleContexts #-}
{-# LANGUAGE UndecidableInstances #-}

module FreeMonad where

-- Please go to Settings (hamburger icon in top-right) and set
-- Environment: GHC 7.8
-- Compiler flags: -fdefer-type-errors
 
data Free f a
  = Pure a
  | Free (f (Free f a))
 
instance Functor f => Monad (Free f) where
  return a     = Pure a
  Pure a >>= f = f a
  Free f >>= g = Free _ -- Hole!

instance Show (f (Free f a)) => Show (Free f a) where
  show (Free x) = show x

 
demo :: Free [] String
demo = Free [] >>= return


main :: IO ()
main =  print demo  -- Fails with type error at runtime, since `demo` tries to call `_`


```

