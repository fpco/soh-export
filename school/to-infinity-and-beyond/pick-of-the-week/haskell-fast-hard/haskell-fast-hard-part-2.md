# Haskell Fast & Hard (Part 2)

## Essential Haskell

![Kandinsky Gugg](http://yannesposito.com/Scratch/img/blog/Haskell-the-Hard-Way/kandinsky_gugg.jpg)

I suggest you to skim this part.
Think of it like a reference.
Haskell has a lot of features.
Many informations are missing here.
Get back here if notation feels strange.

I use the `⇔` symbol to state that two expression are equivalent.
It is a meta notation, `⇔` does not exists in Haskell.
I will also use `⇒` to show what is the return of an expression.

### Notations

##### Arithmetic

```
3 + 2 * 6 / 3 ⇔ 3 + ((2*6)/3)
```

##### Logic

```
True || False ⇒ True
True && False ⇒ False
True == False ⇒ False
True /= False ⇒ True  (/=) is the operator for different
```

##### Powers

```
x^n     for n an integral (understand Int or Integer)
x**y    for y any kind of number (Float for example)
```

`Integer` have no limit except the capacity of your machine:

```
4^103
102844034832575377634685573909834406561420991602098741459288064
```

Yeah!
And also rational numbers FTW!
But you need to import the module `Data.Ratio`:

```
$ ghci
....
Prelude> :m Data.Ratio
Data.Ratio> (11 % 15) * (5 % 3)
11 % 9
```

##### Lists

```
[]                      ⇔ empty list
[1,2,3]                 ⇔ List of integral
["foo","bar","baz"]     ⇔ List of String
1:[2,3]                 ⇔ [1,2,3], (:) prepend one element
1:2:[]                  ⇔ [1,2]
[1,2] ++ [3,4]          ⇔ [1,2,3,4], (++) concatenate
[1,2,3] ++ ["foo"]      ⇔ ERROR String ≠ Integral
[1..4]                  ⇔ [1,2,3,4]
[1,3..10]               ⇔ [1,3,5,7,9]
[2,3,5,7,11..100]       ⇔ ERROR! I am not so smart!
[10,9..1]               ⇔ [10,9,8,7,6,5,4,3,2,1]
```

##### Strings

In Haskell strings are list of `Char`.

```
'a' :: Char
"a" :: [Char]
""  ⇔ []
"ab" ⇔ ['a','b'] ⇔  'a':"b" ⇔ 'a':['b'] ⇔ 'a':'b':[]
"abc" ⇔ "ab"++"c"
```

> _Remark_:
> In real code you shouldn't use list of char to represent text.
> You should mostly use `Data.Text` instead.
> If you want to represent stream of ASCII char, you should use `Data.ByteString`.

##### Tuples

The type of couple is `(a,b)`.
Elements in a tuple can have different type.

```
-- All these tuple are valid
(2,"foo")
(3,'a',[2,3])
((2,"a"),"c",3)

fst (x,y)       ⇒  x
snd (x,y)       ⇒  y

fst (x,y,z)     ⇒  ERROR: fst :: (a,b) -> a
snd (x,y,z)     ⇒  ERROR: snd :: (a,b) -> b
```

##### Deal with parentheses

To remove some parentheses you can use two functions: `($)` and `(.)`.

```
-- By default:
f g h x         ⇔  (((f g) h) x)

-- the $ replace parenthesis from the $
-- to the end of the expression
f g $ h x       ⇔  f g (h x) ⇔ (f g) (h x)
f $ g h x       ⇔  f (g h x) ⇔ f ((g h) x)
f $ g $ h x     ⇔  f (g (h x))

-- (.) the composition function
(f . g) x       ⇔  f (g x)
(f . g . h) x   ⇔  f (g (h x))
```

Exercise:

Define correctly the `selectWin`X functions using only `fst`, `snd` and `.`
in order to show `win` on each line.

``` active haskell
selectWin1 = undefined
main = do
  print $ selectWin1 (1,"win") -- should return "win"
```

``` active haskell
selectWin2 = undefined
main = do
  print $ selectWin2 (("win","no"),"not this one")
```

``` active haskell
selectWin3 = undefined
main = do
  print $ selectWin3 (1,("no",("win","almost")))
```

@@@ Solution

``` active haskell
selectWin1 = snd
selectWin2 = fst . fst
selectWin3 = fst . snd . snd
main = do
  putStrLn $ selectWin1 (1,"win")
  putStrLn $ selectWin2 (("win","no"),"not this one")
  putStrLn $ selectWin3 (1,("no",("win","almost")))
```
@@@


### Useful notations for functions

Just a reminder:

```
x :: Int            ⇔ x is of type Int
x :: a              ⇔ x can be of any type
x :: Num a => a     ⇔ x can be any type a
                      such that a belongs to Num type class
f :: a -> b         ⇔ f is a function from a to b
f :: a -> b -> c    ⇔ f is a function from a to (b→c)
f :: (a -> b) -> c  ⇔ f is a function from (a→b) to c
```

Defining the type of a function before its declaration isn't mandatory.
Haskell infers the most general type for you.
But it is considered a good practice to do so.

_Infix notation_

``` haskell
square :: Num a => a -> a
square x = x^2
```

Note `^` use infix notation.
For each infix operator there its associated prefix notation.
You just have to put it inside parenthesis.

``` haskell
square' x = (^) x 2

square'' x = (^2) x
```

We can remove `x` in the left and right side!
It's called η-reduction.

``` haskell
square''' = (^2)
```

Note we can declare function with `'` in their name.
Here:

> `square` ⇔  `square'` ⇔ `square''` ⇔ `square '''`

_Tests_

An implementation of the absolute function.

``` haskell
absolute :: (Ord a, Num a) => a -> a
absolute x = if x >= 0 then x else -x
```

Note: the `if .. then .. else` Haskell notation is more like the
`¤?¤:¤` C operator. You cannot forget the `else`.

Another equivalent version:

``` haskell
absolute' x
    | x >= 0 = x
    | otherwise = -x
```

> Notation warning: indentation is _important_ in Haskell.
> Like in Python, a bad indentation could break your code!

``` active haskell
square :: Num a => a -> a
square x = x^2
square' x = (^) x 2
square'' x = (^2) x
square''' = (^2)
absolute :: (Ord a, Num a) => a -> a
absolute x = if x >= 0 then x else -x
absolute' x
    | x >= 0 = x
    | otherwise = -x
-- show
main = do
      print $ square 10
      print $ square' 10
      print $ square'' 10
      print $ square''' 10
      print $ absolute 10
      print $ absolute (-10)
      print $ absolute' 10
      print $ absolute' (-10)
-- /show
```

Exercise:

Modify the following code in order to use only prefix notation:

``` active haskell
f x y = x*x + y*y
main = print $ f 2 4
```

@@@ Solution

``` active haskell
f x y = (+) ((*) x x) ((*) y y)
main = print $ f 2 4
```

@@@

[continue to next part](https://www.fpcomplete.com/school/haskell-fast-hard/haskell-fast-hard-part-3)