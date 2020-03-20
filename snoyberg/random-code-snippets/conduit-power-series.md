# conduit power series

```haskell active
import Data.Conduit
import Data.Functor.Identity
import qualified Data.Conduit.List as CL
import qualified Data.Conduit.Util as CU

fact 0 = 1
fact i = i * fact (i - 1)

type PSeries = Source Identity Double

e :: PSeries
e =
    loop 0
  where
    loop i = do
        yield $ 1 / fromIntegral (fact i)
        loop $ i + 1
        
approx :: Double -> PSeries -> Double
approx value src =
    fst $ runIdentity $ src $= CL.isolate 100 $$ CL.fold go (0, 0)
  where
    go (accum, power) coef = (accum + coef * (value ^ power), power + 1)
    
addSeries :: PSeries -> PSeries -> PSeries
addSeries x y = CU.zip x y $= CL.map (\(x, y) -> x + y)

main = do
    print $ approx 1 e
    print $ approx 2 e
    print $ approx 1 $ addSeries e e
```