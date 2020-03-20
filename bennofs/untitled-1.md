# Untitled

```active haskell
import Data.Type.Equality
import Data.Proxy

data Foo a = Foo a

instance Eq a where
  Foo a1 == Foo a2 = case testEquality (Proxy :: Proxy a) (Proxy :: Proxy Int) of
    Nothing -> a1 == a2
    Just Refl -> traceShow (a1,a2) $ a1 == a1

main :: IO ()
main = do
  print (Foo (1 :: Int) == Foo 2)
  print (Foo "test" == Foo "test")

```