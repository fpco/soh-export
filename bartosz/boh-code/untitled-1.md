# 2. Function Syntax

Can you predict the result of the following code? Here `sq` is a function that squares its argument.

``` active haskell
sq b = b * b
main = print $
-- show
	sq 3+1 
-- /show
```

## Function Call

1.  I defined a function `pyth` that takes two numbers and returns the sum of their squares. Add parentheses to the code below to make it compile (don't get scared by unintellegible error messages):

    ``` active haskell
    pyth a b = a * a + b * b
    main = print $
    -- show
        pyth 3 * 2 pyth -1 8
    -- /show
    ```
    @@@ Show answer

    ``` active haskell
    pyth a b = a * a + b * b
    main = print $
    -- show
    	pyth (3 * 2) (pyth (-1) 8)
    -- /show
    ```
    @@@

2.  I also defined a function `pyth'` (yes, you can use apostrophes -- or 'primes', as they are called in mathematics -- in identifiers). `pyth'` takes a tuple of numbers, as in `(a, b)`, and returns the sum of their squares. Add parentheses and commas to code below to make it compile. You may reduce the number of parentheses if you take into account that the comma inside a tuple has lower precedence than arithmetic operators.

    ``` active haskell
    pyth' (a, b) = a * a + b * b
    main = print $
    -- show
        pyth' 3 * 2 pyth' -1 8
    -- /show
    ```
    @@@ Show answer


    ``` active haskell
    pyth' (a, b) = a * a + b * b
    main = print $
    -- show
        pyth' (3 * 2, pyth' (-1, 8))
    -- /show
    ```
    @@@
3.  The `print` function prints its argument, as long as it is convertible to a string. Numbers are convertible to strings. The code below works but looks more like Lisp than Haskell. Try to remove as many parentheses as you can using `$` signs (Hint: With some cleverness, you can get rid of them all).

    ``` active haskell
    pyth a b = a * a + b * b
    main = do
    -- show
        print (sqrt (pyth 3 ((-1) - 3)))
    -- /show
    ```

    @@@ Show answer

    ``` active haskell
    pyth a b = a * a + b * b
    main = 
    -- show
        print $ sqrt $ pyth 3 $ -1 -3
    -- /show
    ```
    @@@

## Function Definition

1.  Define a function `flop` that reverses a pair.
    ``` active haskell
    flop ? = ?
    main = print $ flop (1, "one")
    ```
    @@@ Show solution
    ``` active haskell
    flop (a, b) = (b, a)
    main = print $ flop (1, "one")
    ```
    @@@
2.  Define a function that takes two points (pairs) and returns a segment (a pair of points).
    ``` active haskell
    makeSegment ? = ?
    main = print $ makeSegment (1, 2) (3, 4)
    ```
    @@@ Show solution
    ``` active haskell
    makeSegment p1 p2 = (p1, p2)
    main = print $ makeSegment (1, 2) (3, 4)
    ```
    @@@
3.  Define a function that takes a segment and returns it's center point.
    ``` active haskell
    center ? = ?
    main = print $ center ((1,2), (3, 4))
    ```
    @@@ Show solution
    ``` active haskell
    center ((x, y), (x', y')) = ((x + x')/2, (y + y')/2)
    main = print $ center ((1,2), (3, 4))
    ```
    @@@

## Recursion

**Ex 1**. Print squares of numbers from 1 to 10.
``` active haskell
loop :: Int -> IO ()
loop n = undefined

main :: IO ()
main = loop 1
```

@@@ Show solution
``` active haskell
loop :: Int -> IO ()
loop n = do
    if n <= 10
    then do
        putStrLn (show (n * n))
        loop (n + 1)
    else
        return ()

main :: IO ()
main = loop 1
```
@@@

**Ex 2**. No exposition of recursion is complete without factorial. Use the following property: Factorial of n is n times the factorial of (n - 1), and the factorial of 0 is 1.

``` active haskell
fact :: Int -> Int
fact n = undefined

main = print (fact 20)
```

@@@ Show solution
``` active haskell
fact :: Int -> Int
fact n = if n > 0 then n * fact (n - 1) else 1

main = print (fact 20)
```
@@@

