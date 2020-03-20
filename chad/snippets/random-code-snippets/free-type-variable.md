# Free type variable

Answer to question [posed on the Yesod mailing list](https://groups.google.com/d/msg/yesodweb/nrOtbyOigCc/Tlcy3xWgtWsJ).

Sometimes GHC will complain about not knowing about a type variable. In the example below, the reason is that the behavior of the program depends on specifying that type variable. For example, in `someFunction`, the choice of monad will determine the name to be returned.

```haskell active
class Monad m => NamedMonad m where
    namedMonad :: m a -> String
instance NamedMonad IO where
    namedMonad _ = "IO"
instance NamedMonad Maybe where
    namedMonad _ = "Maybe"

someFunction :: NamedMonad m => (String, m Int)
someFunction =
    (name, res)
  where
    name = namedMonad res
    res = return $ length name

main :: IO ()
main = do
    -- Not ambiguous
    let (name1, val1) = someFunction
    val1' <- val1
    print (name1, val1')
    
    -- Very ambiguous, try commenting out
    putStrLn $ fst someFunction
    
    -- But this fixes is
    putStrLn $ fst (someFunction :: (String, Maybe Int))
```