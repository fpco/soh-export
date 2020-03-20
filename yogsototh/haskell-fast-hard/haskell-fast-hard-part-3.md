# Haskell Fast & Hard  (Part 3)

## Hard Part

The hard part can now begin.

### Functional style

![Biomechanical Landscape by H.R. Giger](http://yannesposito.com/Scratch/img/blog/Haskell-the-Hard-Way/hr_giger_biomechanicallandscape_500.jpg)

In this section, I will give a short example of the impressive refactoring ability provided by Haskell.
We will select a problem and solve it using a standard imperative way.
Then I will make the code evolve.
The end result will be both more elegant and easier to adapt.

Let's solve the following problem:

> Given a list of integers, return the sum of the even numbers in the list.
>
> example:
> `[1,2,3,4,5] ⇒  2 + 4 ⇒  6`

To show differences between the functional and imperative approach,
I'll start by providing an imperative solution (in Javascript):

``` javascript
function evenSum(list) {
    var result = 0;
    for (var i=0; i< list.length ; i++) {
        if (list[i] % 2 ==0) {
            result += list[i];
        }
    }
    return result;
}
```

But, in Haskell we don't have mutable variables, nor for loop.
One solution to achieve the same result without loops is to use recursion.

> _Remark_:
> Recursion is generally perceived as slow in imperative languages.
> But it is generally not the case in functional programming.
> Most of the time Haskell will handle recursive functions efficiently.

Here is a `C` version of the recursive function.
Note that for simplicity, I assume the int list ends with the first `0` value.

``` c
int evenSum(int *list) {
    return accumSum(0,list);
}

int accumSum(int n, int *list) {
    int x;
    int *xs;
    if (*list == 0) { // if the list is empty
        return n;
    } else {
        x = list[0]; // let x be the first element of the list
        xs = list+1; // let xs be the list without x
        if ( 0 == (x%2) ) { // if x is even
            return accumSum(n+x, xs);
        } else {
            return accumSum(n, xs);
        }
    }
}
```

Keep this code in mind. We will translate it into Haskell.
But before, I need to introduce three simple but useful functions we will use:

``` haskell
even :: Integral a => a -> Bool
head :: [a] -> a
tail :: [a] -> [a]
```

`even` verifies if a number is even.

``` haskell
even :: Integral a => a -> Bool
even 3  ⇒ False
even 2  ⇒ True
```

`head` returns the first element of a list:

``` haskell
head :: [a] -> a
head [1,2,3] ⇒ 1
head []      ⇒ ERROR
```

`tail` returns all elements of a list, except the first:

``` haskell
tail :: [a] -> [a]
tail [1,2,3] ⇒ [2,3]
tail [3]     ⇒ []
tail []      ⇒ ERROR
```

Note that for any non empty list `l`,
`l ⇔ (head l):(tail l)`


The first Haskell solution.
The function `evenSum` returns the sum of all even numbers in a list:

``` active haskell
-- Version 1
evenSum :: [Integer] -> Integer

evenSum l = accumSum 0 l

accumSum n l = if l == []
                  then n
                  else let x = head l
                           xs = tail l
                       in if even x
                              then accumSum (n+x) xs
                              else accumSum n xs
main = print $ evenSum [1..10]                    
```

Here is an example of execution ; (I know I'm cheating. But I will talk about non-strict later) :

<pre>
*Main> evenSum [1..5]
accumSum 0 [1,2,3,4,5]
<span class="yellow">1 is odd</span>
accumSum 0 [2,3,4,5]
<span class="yellow">2 is even</span>
accumSum (0+2) [3,4,5]
<span class="yellow">3 is odd</span>
accumSum (0+2) [4,5]
<span class="yellow">4 is even</span>
accumSum (0+2+4) [5]
<span class="yellow">5 is odd</span>
accumSum (0+2+4) []
<span class="yellow">l == []</span>
0+2+4
0+6
6
</pre>

Coming from an imperative language all should seem right.
In reality many things can be improved.
First, we can generalize the type.

``` haskell
-- show
evenSum :: Integral a => [a] -> a
-- /show
main = do print $ evenSum [1..10]
```

Next, we can use sub functions using `where` or `let`.
This way our `accumSum` function won't pollute the global namespace.

``` active haskell
-- show
-- Version 2
evenSum :: Integral a => [a] -> a

evenSum l = accumSum 0 l
    {-hi-} where {-/hi-} accumSum n l =
            if l == []
                then n
                else let x = head l
                         xs = tail l
                     in if even x
                            then accumSum (n+x) xs
                            else accumSum n xs
-- /show
main = print $ evenSum [1..10]
```



Next, we can use pattern matching.

``` active haskell
-- show
-- Version 3
evenSum l = accumSum 0 l
    where
        accumSum {-hi-}n []{-/hi-} = n
        accumSum {-hi-}n (x:xs){-/hi-} =
             if even x
                then accumSum (n+x) xs
                else accumSum n xs
-- /show
main = print $ evenSum [1..10]
```

What is pattern matching?
Use values instead of general parameter names
(For the brave, a more complete explanation of pattern matching can be found [here](http://www.cs.auckland.ac.nz/references/haskell/haskell-intro-html/patterns.html)).

Instead of saying: `foo l = if l == [] then <x> else <y>`
You simply state:

``` haskell
foo [] =  <x>
foo l  =  <y>
```

But pattern matching goes even further.
It is also able to inspect the inner data of a complex value.
We can replace

``` haskell
foo l =  let x  = head l
             xs = tail l
         in if even x
             then foo (n+x) xs
             else foo n xs
```

with

``` haskell
foo (x:xs) = if even x
                 then foo (n+x) xs
                 else foo n xs
```

This is a very useful feature.
It makes our code both terser and easier to read.




In Haskell you can simplify function definition by η-reducing them.
For example, instead of writing:

``` haskell
f x = (some expresion) x
```

you can simply write

``` haskell
f = some expression
```

Exercise:

Simplify the function evenSum by η-reducing it.


``` active haskell
-- show
-- Version 3
evenSum {-hi-}l{-/hi-} = accumSum 0 {-hi-}l{-/hi-}
    where
        accumSum n [] = n
        accumSum n (x:xs) =
             if even x
                then accumSum (n+x) xs
                else accumSum n xs
-- /show
main = print $ evenSum [1..10]
```


@@@ Solution (Version 4)

We use this method to remove the `l`:

``` active haskell
-- show
-- Version 4
evenSum :: Integral a => [a] -> a

evenSum = accumSum 0
    where
        accumSum n [] = n
        accumSum n (x:xs) =
             if even x
                then accumSum (n+x) xs
                else accumSum n xs
-- /show
main = print $ evenSum [1..10]
```
@@@


#### Higher Order Functions

![Escher](http://yannesposito.com/Scratch/img/blog/Haskell-the-Hard-Way/escher_polygon.png)

To make things even better we should use higher order functions.
What are these beasts?
Higher order functions are functions taking functions as parameter.

Here are some examples:

``` haskell
filter :: (a -> Bool) -> [a] -> [a]
map :: (a -> b) -> [a] -> [b]
foldl :: (a -> b -> a) -> a -> [b] -> a
```

Let's proceed by small steps.

``` active haskell
-- show
-- Version 5
evenSum l = mysum 0 (filter even l)
    where
      mysum n [] = n
      mysum n (x:xs) = mysum (n+x) xs
-- /show
main = print $ evenSum [1..10]
```

where

``` haskell
filter even [1..10] ⇔  [2,4,6,8,10]
```

The function `filter` takes a function of type (`a -> Bool`) and a list of type `[a]`. It returns a list containing only elements for which the function returned `true`.

Our next step is to use another way to simulate a loop.
We will use the `foldl` function to accumulate a value.
The function `foldl` captures a general coding pattern:

<pre>
myfunc list = foo <span class="blue">initialValue</span> <span class="green">list</span>
    foo accumulated []     = accumulated
    foo tmpValue    (x:xs) = foo (<span class="yellow">binop</span> tmpValue x) xs
</pre>

Which can be replaced by:

<pre>
myfunc list = foldl <span class="yellow">binop</span> <span class="blue">initialValue</span> <span class="green">list</span>
</pre>

If you really want to know how the magic works.
Here is the definition of `foldl`.

``` haskell
foldl f z [] = z
foldl f z (x:xs) = foldl f (f z x) xs
```

``` haskell
foldl f z [x1,...,xn]
⇔  f (... (f (f z x1) x2) ...) xn
```

But as Haskell is lazy, it doesn't evaluate `(f z x)` and pushes it to the stack.
This is why we generally use `foldl'` instead of `foldl`;
`foldl'` is a _strict_ version of `foldl`.
If you don't understand what lazy and strict means,
don't worry, just follow the code as if `foldl` and `foldl'` where identical.

Now our new version of `evenSum` becomes:

``` active haskell
-- show
-- Version 6
-- foldl' isn't accessible by default
-- we need to import it from the module Data.List
import Data.List
evenSum l = foldl' mysum 0 (filter even l)
  where mysum acc value = acc + value
-- /show
main = print $ evenSum [1..10]
```

We can simplify by using directly a lambda notation.
This way we don't have to create the temporary name `mysum`.

``` active haskell
-- show
-- Version 7
-- Generally it is considered a good practice
-- to import only the necessary function(s)
import Data.List (foldl')
evenSum l = foldl' (\x y -> x+y) 0 (filter even l)
-- /show
main = print $ evenSum [1..10]
```

And of course, we note that

``` haskell
(\x y -> x+y) ⇔ (+)
```




Finally

``` active haskell
-- show
-- Version 8
import Data.List (foldl')
evenSum :: Integral a => [a] -> a
evenSum l = foldl' (+) 0 (filter even l)
-- /show
main = print $ evenSum [1..10]
```

`foldl'` isn't the easiest function to intuit.
If you are not used to it, you should study it a bit.

To help you understand what's going on here, a step by step evaluation:

<pre>
  <span class="yellow">evenSum [1,2,3,4]</span>
⇒ foldl' (+) 0 (<span class="yellow">filter even [1,2,3,4]</span>)
⇒ <span class="yellow">foldl' (+) 0 <span class="blue">[2,4]</span></span>
⇒ <span class="blue">foldl' (+) (<span class="yellow">0+2</span>) [4]</span>
⇒ <span class="yellow">foldl' (+) <span class="blue">2</span> [4]</span>
⇒ <span class="blue">foldl' (+) (<span class="yellow">2+4</span>) []</span>
⇒ <span class="yellow">foldl' (+) <span class="blue">6</span> []</span>
⇒ <span class="blue">6</span>
</pre>

##### Exercise

Rewrite the following program using `foldl'`

``` active haskell
import Data.List (foldl')
-- show prod [3,4,5] will return 3*4*5=60
prod :: [Integer] -> Integer
prod [] = 1
prod (x:xs) = x*prod xs

main = print $ prod [3,4,5] 
```
@@@ Solution
``` active haskell
import Data.List (foldl')
-- show
prod = foldl' (*) 1
-- /show
main = print $ prod [3,4,5] 
```
@@@

---

Another useful higher order function is `(.)`.
The `(.)` function corresponds to the mathematical composition.

``` haskell
(f . g . h) x ⇔  f ( g (h x))
```

We can take advantage of this operator to η-reduce our function:

``` haskell
-- show
-- Version 9
import Data.List (foldl')
evenSum :: Integral a => [a] -> a
evenSum = (foldl' (+) 0) . (filter even)
-- /show
main = do print $ evenSum [1..10]
```

Also, we could rename some parts to make it clearer:

``` haskell
-- show
-- Version 10
import Data.List (foldl')
sum' :: (Num a) => [a] -> a
sum' = foldl' (+) 0
evenSum :: Integral a => [a] -> a
evenSum = sum' . (filter even)
-- /show
main = do print $ evenSum [1..10]
```

It is time to discuss a bit.
What did we gain by using higher order functions?

At first, you can say it is terseness.
But in fact, it has more to do with better thinking.
Suppose we want to modify slightly our function.
We want to get the sum of all even square of element of the list.

```
[1,2,3,4] ▷ [1,4,9,16] ▷ [4,16] ▷ 20
```

Update the version 10 is extremely easy:

``` haskell
squareEvenSum = sum' . (filter even) . (map (^2))
squareEvenSum' = evenSum . (map (^2))
```

We just had to add another "transformation function"[^0216].

[^0216]: You should remark `squareEvenSum''` is more efficient that the two other versions. The order of `(.)` is important.

```
map (^2) [1,2,3,4] ⇔ [1,4,9,16]
```

The `map` function simply apply a function to all element of a list.

We didn't had to modify anything _inside_ the function definition.
It feels more modular.
But in addition you can think more mathematically about your function.
You can then use your function as any other one.
You can compose, map, fold, filter using your new function.

To modify version 1 is left as an exercise to the reader ☺.

If you believe we reached the end of generalization, then know you are very wrong.
For example, there is a way to not only use this function on lists but on any recursive type.
If you want to know how, I suggest you to read this quite fun article: [Functional Programming with Bananas, Lenses, Envelopes and Barbed Wire by Meijer, Fokkinga and Paterson](http://eprints.eemcs.utwente.nl/7281/0
1/db-utwente-40501F46.pdf). You could also just get a bit of the idea by viewing my [presentation about Category Theory](http://yogsototh.github.com/Category-Theory-Presentation).

This example should show you how great pure functional programming is.
Unfortunately, using pure functional programming isn't well suited to all usages.
Or at least such a language hasn't been found yet.

One of the great powers of Haskell is the ability to create DSLs
(Domain Specific Language)
making it easy to change the programming paradigm.

In fact, Haskell is also great when you want to write imperative style programming.
Understanding this was really hard for me when learning Haskell.
A lot of effort has been done to explain to you how much functional approach is superior.
Then when you start the imperative style of Haskell, it is hard to understand why and how.

But before talking about this Haskell super-power, we must talk about another
essential aspect of Haskell: _Types_.

[continue to next part](https://www.fpcomplete.com/school/haskell-fast-hard/haskell-fast-hard-part-4)

