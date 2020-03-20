# Small Arithmetic AST Example

``` active haskell

data Exp = Number     Int
         | Add        Exp Exp
         | Subtract   Exp Exp
         | Multiply   Exp Exp
         | Divide     Exp Exp
  deriving (Show)

instance Num Exp where
    x + y = Add       x y
    x - y = Subtract  x y
    x * y = Multiply  x y
    
    fromInteger x = Number . fromIntegral $ x
    
main = do
    print (4 + 4 * 10 - 8  :: Exp)
```



    