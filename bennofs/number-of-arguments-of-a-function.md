# Number of arguments of a function

```active haskell
{-# LANGUAGE TypeFamilies #-}
{-# LANGUAGE TypeOperators #-}
{-# LANGUAGE DataKinds #-}
{-# LANGUAGE ScopedTypeVariables #-}
{-# LANGUAGE UndecidableInstances #-}
{-# LANGUAGE FlexibleContexts #-}

import GHC.TypeLits
import Data.Proxy

type family NumArgs a where
  NumArgs (a -> b) = NumArgs b + 1
  NumArgs a        = 0

numArgs :: forall a. (KnownNat (NumArgs a)) => a -> Integer
numArgs _ = natVal (Proxy :: Proxy (NumArgs a))

main :: IO ()
main = print $ numArgs ((+) :: Int -> Int -> Int)
```