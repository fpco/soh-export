# Order of Evaluation

The order of evaluation in Haskell is determined by one and only one thing: __Data Dependence__.

An expression will only be evaluated when it is needed, and will not be evaluated if it is not needed.

##Pure Code

Consider the following code:
``` active haskell
f x = v2 + 4
  where v0 = undefined + 1
        v1 = 2 * x
        v2 = v1 / 3 + x

main = putStrLn $ show (f 10)
```
The order of the variable definitions in the where clause has no effect whatsoever on the evaluation of this function.
``` active haskell
f' x = v2 + 4
  where v2 = v1 / 3 + x
        v0 = undefined + 1
        v1 = 2 * x

main = putStrLn $ show (f' 10)
```
`v0` is never evaluated in either `f` or `f'` since it is not needed by the result value `v2 + 4`.

Now that we've shown what won't be evaluated, how do we go about seeing when things are evaluated?

### Detour: Tracing in Haskell
The standard Haskell libraries come with a module `Debug.Trace` which has a `trace` function
``` haskell
trace :: String -> a -> a
```
which will output the given string to `stderr` before returning its second argument.

However we can't use `trace` by itself to track when things are evaluated because `trace`, like everything else, is lazy. So evaluating `trace str x` will cause `str` to be output without forcing `x` to be evaluated.
``` active haskell
import Debug.Trace

main = putStrLn $ show $ trace "res" $ trace "1" 1 + trace "2" 2
```
As we can see, `res` is output before `1` and `2`.

However, we can write a function to force evaluation before tracing.
``` active haskell
import Debug.Trace

tr msg x = seq x $ trace msg x

main = putStrLn $ show $ tr "res" $ tr "1" 1 + tr "2" 2
```
Now, as expected, `res` is output after `1` and `2`.

### Order of Evaluation
We can now see explicitly when each term is evaluated.
``` active haskell
import Debug.Trace
tr msg x = seq x $ trace msg x

f' x = v2 + 4
  where v2 = tr "v2" $ v1 / 3 + x'
        v0 = tr "v0" $ undefined + 1
        v1 = tr "v1" $ 2 * x'
        x' = tr "x" x

main = putStrLn $ show (f' 10)
```
I've introduced `x'` so we can also see when `x` is evaluated.

`x` is needed by `v1` which is needed by `v2` which is needed by the resulting value which is printed. Therefore the order of evaluation is: `x`, `v1`, `v2`.

The same principle holds true for expressions anywhere, not just in `where`
clauses. 
``` active haskell
data E a b = L a | R b

keepRs [] = []
keepRs (R x : xs) = x : keepRs xs
keepRs (L _ : xs) = keepRs xs

g = sum . keepRs

main = putStrLn $ show (g [R 10, L undefined, R 20])
```
No expressions underneath a `L` constructor in the input list to `g` ever gets evaluated since `keepRs` never uses the expressions underneath `L` constructors.

To get a more detailed picture of how evaluation is working, we can have a traced version of the above.
``` active haskell
import Debug.Trace
tr msg x = seq x $ trace msg x

data E a b = L a | R b

keepRs [] = []
keepRs (R x : xs) = x : keepRs xs
keepRs (L _ : xs) = keepRs xs

g = sum . keepRs

main = putStrLn $ show (g [(tr "R_0" R) (tr "R_0's 10"  10), 
                           (tr "L"   L) (tr "undefined" undefined), 
                           (tr "R_1" R) (tr "R_1's 20"  20)
                          ]
                       )
```
Notice the outer constructors (`L` and `R`) of each list element are evaluated; they are needed by `keepRs` to figure out which clause to apply. However, only the arguments of the `R` constructors are evaluated when they are needed by `sum`.

One thing to mention is that bang patterns can be used to create strict data constructors which can force the evaluation of their arguments. For example, suppose we rewrite the previous code with a strict version of `E`:
``` active haskell
data E a b = L !a | R !b

keepRs [] = []
keepRs (R x : xs) = x : keepRs xs
keepRs (L _ : xs) = keepRs xs

g = sum . keepRs

main = putStrLn $ show (g [R 10, L undefined, R 20])
```
Now, even though `keepRs` doesn't use the arguments of `L`, they still get evaluated since `L` is strict.

##Monadic Code
Even in monadic code, data dependence is the only thing which determines if and when an expression gets evaluated. Although the order of monadic actions affects a program-- it determines the order in which the monadic operations are performed and in which variables are brought into scope-- it does not determine the order in which expressions get evaluated.

Consider the following monadic version of our first example, extended with some extra monadic actions.
``` active haskell
import Debug.Trace
tr msg x = seq x $ trace msg x

fM x = do
  x' <- return $ tr "x" x
  v0 <- return $ tr "v0" $ undefined + 1
  v1 <- return $ tr "v1" $ 2 * x'
  v2 <- return $ tr "v2" $ v1 / 3 + x'
  putStrLn "Enter an Int: "
  v3 <- fmap (\x -> tr "v3" $ v0 + 2 + read x) getLine
  return $ v2 + 4

main = putStrLn . show =<< fM 10
```
Even though the `getLine` is executed, there is no error during evaluation. `v0` does not get evaluated since it is only used in `v3`, but `v3` is not evaluated since it is not used by the result value. In fact, you can enter complete gibberish and there will still be no error.

To understand better what exactly monadic code causes to happen, we just need to look a little closer. `do` notation is just syntactic sugar and can be expanded into regular Haskell syntax using ` >>= `:

``` haskell
do x <- m
   f
```
where `f` is some code which depends upon `x` (i.e. in which `x` occurs unbound) gets translated to
``` haskell
m >>= (\x -> f) 
```
I have added parentheses to make the two arguments to ` >>= ` clear.

The type of ` >>= `
``` haskell
(>>=) :: Monad m => m a -> (a -> m b) -> m b
```
along with the left identity monad law
``` haskell
return x >>= f  ==  f x
```
tell us that ` >>= ` does not look at the expression it takes out of its monad and passes on to its second argument; consider the case where `x` is `undefined` and `f` is `const ()`.

The first argument of ` >>= ` is only evaluated enough to get a monadic expression, but anything under the constructor for the monad type is not evaluated. For example, if we are in the `Maybe` monad
``` haskell
data Maybe a = Nothing | Just a

instance Monad Maybe where
  return x = Just x

  (Just x) >>= f = f x
  Nothing  >>= _  = Nothing
```
` >>= `  will not cause anything under the `Just` constructor to be evaluated.