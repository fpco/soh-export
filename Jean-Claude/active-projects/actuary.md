# Actuarial calculations

## Why use Haskell ?

Using Haskell is clearly unsual when associated to actuaries. Actuaries do not program that much, they rather use softwares like R or Matlab.
My goal here is to use Haskell to model actuarial concepts 

## Simple computations

The simplest thing that can be computed is the premium, given an insured value.
Here, we handle everything as Double.

``` active haskell
basePremium :: Double -> Double -> Double
basePremium baserate exposure
	-- manage exceptions (negative base rate)
	| baserate < 0  
		 = error "Negative rate"
	-- manage exceptions (negative insured value)
	| exposure < 0  
		 = error "Negative exposure"
	-- nominal case
	| otherwise
		= baserate * exposure


main :: IO ()
main = do
    print (basePremium 0.0005 1000000)
```