# Haskell Fast & Hard (Part 5)

## Hell Difficulty Part

Congratulations for getting so far!
Now, some of the really hardcore stuff can start.

If you are like me, you should get the functional style.
You should also understand a bit more the advantages of laziness by default.
But you also don't really understand where to start in order to make a real
program.
And in particular:

- How do you deal with effects?
- Why is there a strange imperative-like notation for dealing with IO?

Be prepared, the answers might be complex.
But they all be very rewarding.


### Deal With IO

![Magritte, Carte blanche](http://yannesposito.com/Scratch/img/blog/Haskell-the-Hard-Way/magritte_carte_blanche.jpg)

---

Too long; didn't read:

A typical function doing `IO` looks a lot like an imperative program:

``` haskell
f :: IO a
f = do
  x <- action1
  action2 x
  y <- action3
  action4 x y
```

- To set a value to an object we use `<-` .
- The type of each line is `IO *`;
  in this example:

  - `action1     :: IO b`
  - `action2 x   :: IO ()`
  - `action3     :: IO c`
  - `action4 x y :: IO a`
  - `x :: b`, `y :: c`
- Few objects have the type `IO a`, this should help you choose.
  In particular you cannot use pure functions directly here.
  To use pure functions you could do `action2 (purefunction x)` for example.

---

In this section, I will explain how to use IO, not how it works.
You'll see how Haskell separates the pure from the impure parts of the program.

Don't stop because you're trying to understand the details of the syntax.
Answers will come in the next section.

What to achieve?

> Ask a user to enter a list of numbers.
> Print the sum of the numbers

``` active haskell
toList :: String -> [Integer]
toList input = read ("[" ++ input ++ "]")

main = do
  putStrLn "Enter a list of numbers (separated by comma):"
  input <- getLine
  print $ sum (toList input)
```

It should be straightforward to understand the behavior of this program.
Let's analyze the types in more detail.

```
putStrLn :: String -> IO ()
getLine  :: IO String
print    :: Show a => a -> IO ()
```

Or more interestingly, we note that each expression in the `do` block has a type of `IO a`.

<pre>
main = do
  putStrLn "Enter ... " :: <span class="high">IO ()</span>
  getLine               :: <span class="high">IO String</span>
  print Something       :: <span class="high">IO ()</span>
</pre>

We should also pay attention to the effect of the `<-` symbol.

```
do
 x <- something
```

If `something :: IO a` then `x :: a`.

Another important note about using `IO`.
All lines in a do block must be of one of the two forms:

```
action1             :: IO a
                    -- in this case, generally a = ()
```

or

```
value <- action2    -- where
                    -- bar z t :: IO b
                    -- value   :: b
```

These two kinds of line will correspond to two different ways of sequencing actions.
The meaning of this sentence should be clearer by the end of the next section.



Now let's see how this program behaves.
For example, what occur if the user enter something strange?
Try to write `foo` instead of a list of integer:


``` active haskell
toList :: String -> [Integer]
toList input = read ("[" ++ input ++ "]")

main = do
  putStrLn "Enter a list of numbers (separated by comma):"
  input <- getLine
  print $ sum (toList input)
```


Argh! An evil error message and a crash!
The first evolution will be to answer with a more friendly message.

In order to do this, we must detect that something went wrong.
Here is one way to do this.
Use the type `Maybe`.
It is a very common type in Haskell.

``` haskell
import Data.Maybe
```

What is this thing? `Maybe` is a type which takes one parameter.
Its definition is:

``` haskell
data Maybe a = Nothing | Just a
```

This is a nice way to tell there was an error while trying to create/compute
a value.
The `maybeRead` function is a great example of this.
This is a function similar to the function `read`[^1],
but if something goes wrong the returned value is `Nothing`.
If the value is right, it returns `Just <the value>`.
Don't try to understand too much of this function.
I use a lower level function than `read`; `reads`.

[^1]: Which itself is very similar to the javascript `eval` on a string containing JSON).

``` haskell
maybeRead :: Read a => String -> Maybe a
maybeRead s = case reads s of
                  [(x,"")]    -> Just x
                  _           -> Nothing
```

Now to be a bit more readable, we define a function which goes like this:
If the string has the wrong format, it will return `Nothing`.
Otherwise, for example for "1,2,3", it will return `Just [1,2,3]`.

``` haskell
getListFromString :: String -> Maybe [Integer]
getListFromString str = maybeRead $ "[" ++ str ++ "]"
```

We simply have to test the value in our main function.

``` active haskell
import Data.Maybe
maybeRead :: Read a => String -> Maybe a
maybeRead s = case reads s of
                  [(x,"")]    -> Just x
                  _           -> Nothing
getListFromString :: String -> Maybe [Integer]
getListFromString str = maybeRead $ "[" ++ str ++ "]"
main :: IO ()
-- show
main = do
  putStrLn "Enter a list of numbers (separated by comma):"
  input <- getLine
  let maybeList = getListFromString input in
      case maybeList of
          Just l  -> print (sum l)
          Nothing -> error "Bad format. Good Bye."
-- /show
```

In case of error, we display a nice error message.

Note that the type of each expression in the main's do block remains of the form `IO a`.
The only strange construction is `error`.
I'll say `error msg` will simply take the needed type (here `IO ()`).

One very important thing to note is the type of all the functions defined so far.
There is only one function which contains `IO` in its type: `main`.
This means main is impure.
But main uses `getListFromString` which is pure.
It is then clear just by looking at declared types which functions are pure and
which are impure.

Why does purity matter?
I certainly forget many advantages, but the three main reasons are:

- It is far easier to think about pure code than impure one.
- Purity protects you from all the hard to reproduce bugs due to side effects.
- You can evaluate pure functions in any order or in parallel without risk.

This is why you should generally put as most code as possible inside pure functions.

Our next evolution will be to prompt the user again and again until she enters a valid answer.
We create a function which will ask the user for an list of integers
until the input is right.

``` haskell
askUser :: IO [Integer]
askUser = do
  putStrLn "Enter a list of numbers (separated by comma):"
  input <- getLine
  let maybeList = getListFromString input in
      case maybeList of
          Just l  -> return l
          Nothing -> askUser
```

This function is of type `IO [Integer]`.
Such a type means that we retrieved a value of type `[Integer]` through some IO actions.
Some people might explain while waving their hands:

> «This is an `[Integer]` inside an `IO`»

If you want to understand the details behind all of this, you'll have to read the next section.
But sincerely, if you just want to _use_ IO.
Just practice a little and remember to think about the type.

Finally our main function is quite simpler:

``` active haskell
import Data.Maybe

maybeRead :: Read a => String -> Maybe a
maybeRead s = case reads s of
                  [(x,"")]    -> Just x
                  _           -> Nothing
getListFromString :: String -> Maybe [Integer]
getListFromString str = maybeRead $ "[" ++ str ++ "]"
askUser :: IO [Integer]
askUser = do
  putStrLn "Enter a list of numbers (separated by comma):"
  input <- getLine
  let maybeList = getListFromString input in
      case maybeList of
          Just l  -> return l
          Nothing -> askUser
-- show
main :: IO ()
main = do
  list <- askUser
  print $ sum list
-- /show
```

We have finished with our introduction to `IO`.
This was quite fast. Here are the main things to remember:

- in the `do` bloc, each expression must have the type `IO a`.
  You are then limited in the number of expressions available.
  For example, `getLine`, `print`, `putStrLn`, etc...
- Try to externalize the pure functions as much as possible.
- the `IO a` type means: an IO _action_ which returns an element of type `a`.
  `IO` represents actions; under the hood, `IO a` is the type of a function.
  Read the next section if you are curious.

If you practice a bit, you should be able to _use_ `IO`.

> _Exercises_:
>
> - Make a program that sums all of its arguments. Hint: use the function `getArgs`.


### IO trick explained

![Magritte, ceci n'est pas une pipe](http://yannesposito.com/Scratch/img/blog/Haskell-the-Hard-Way/magritte_pipe.jpg)

---

Here is a tldr for this section.

To separate pure and impure parts,
`main` is defined as a function
which modifies the state of the world

```
main :: World -> World
```

A function is guaranteed to have side effects only if it has this type.
But look at a typical main function:

```
main w0 =
    let (v1,w1) = action1 w0 in
    let (v2,w2) = action2 v1 w1 in
    let (v3,w3) = action3 v2 w2 in
    action4 v3 w3
```

We have a lot of temporary elements (here `w1`, `w2` and `w3`)
which must be passed on to the next action.

We create a function `bind` or `(>>=)`.
With `bind` we don't need temporary names anymore.

```
main =
  action1 >>= action2 >>= action3 >>= action4
```

Bonus: Haskell has syntactical sugar for us:

```
main = do
  v1 <- action1
  v2 <- action2 v1
  v3 <- action3 v2
  action4 v3
```

---

Why did we use this strange syntax, and what exactly is this `IO` type?
It looks a bit like magic.

For now let's just forget all about the pure parts of our program, and focus
on the impure parts:

``` haskell
askUser :: IO [Integer]
askUser = do
  putStrLn "Enter a list of numbers (separated by commas):"
  input <- getLine
  let maybeList = getListFromString input in
      case maybeList of
          Just l  -> return l
          Nothing -> askUser

main :: IO ()
main = do
  list <- askUser
  print $ sum list
```

First remark; it looks like an imperative structure.
Haskell is powerful enough to make impure code look imperative.
For example, if you wish you could create a `while` in Haskell.
In fact, for dealing with `IO`, imperative style is generally more appropriate.

But you should had noticed the notation is a bit unusual.
Here is why, in detail.

In an impure language, the state of the world can be seen as a huge hidden global variable.
This hidden variable is accessible by all functions of your language.
For example, you can read and write a file in any function.
The fact that a file exists or not can be seen as different states of the world.

For Haskell this state is not hidden.
It is explicitly said `main` is a function that _potentially_ changes the state of the world.
Its type is then something like:

``` haskell
main :: World -> World
```

Not all functions may have access to this variable.
Those which have access to this variable are impure.
Functions to which the world variable isn't provided are pure[2].

[2]: There are some _unsafe_ exceptions to this rule. But you shouldn't see such use on a real application except maybe for debugging purpose.

Haskell considers the state of the world as an input variable to `main`.
But the real type of main is closer to this one[3]:

[3]: For the curious the real type is `data IO a = IO {unIO :: State# RealWorld -> (# State# RealWorld, a #)}`. All the `#` as to do with optimisation and I swapped the fields in my example. But mostly, the idea is exactly the same.

``` haskell
main :: World -> ((),World)
```

The `()` type is the null type.
Nothing to see here.

Now let's rewrite our main function with this in mind:

``` haskell
main w0 =
    let (list,w1) = askUser w0 in
    let (x,w2) = print (sum list,w1) in
    x
```

First, we note that all functions which have side effects must have the type:

``` haskell
World -> (a,World)
```

Where `a` is the type of the result.
For example, a `getChar` function should have the type `World -> (Char,World)`.

Another thing to note is the trick to fix the order of evaluation.
In Haskell, in order to evaluate `f a b`, you have many choices:

- first eval `a` then `b` then `f a b`
- first eval `b` then `a` then `f a b`.
- eval `a` and `b` in parallel then `f a b`

This is true, because we should work in a pure language.

Now, if you look at the main function, it is clear you must eval the first
line before the second one since, to evaluate the second line you have
to get a parameter given by the evaluation of the first line.

Such trick works nicely.
The compiler will at each step provide a pointer to a new real world id.
Under the hood, `print` will evaluate as:

- print something on the screen
- modify the id of the world
- evaluate as `((),new world id)`.

Now, if you look at the style of the main function, it is clearly awkward.
Let's try to do the same to the askUser function:

``` haskell
askUser :: World -> ([Integer],World)
```

Before:

``` haskell
askUser :: IO [Integer]
askUser = do
  putStrLn "Enter a list of numbers:"
  input <- getLine
  let maybeList = getListFromString input in
      case maybeList of
          Just l  -> return l
          Nothing -> askUser
```

After:

``` haskell
askUser w0 =
    let (_,w1)     = putStrLn "Enter a list of numbers:" in
    let (input,w2) = getLine w1 in
    let (l,w3)     = case getListFromString input of
                      Just l   -> (l,w2)
                      Nothing  -> askUser w2
    in
        (l,w3)
```

This is similar, but awkward.
Look at all these temporary `w?` names.

The lesson, is, naive IO implementation in Pure functional languages is awkward!

Fortunately, there is a better way to handle this problem.
We see a pattern.
Each line is of the form:

``` haskell
let (y,w') = action x w in
```

Even if for some line the first `x` argument isn't needed.
The output type is a couple, `(answer, newWorldValue)`.
Each function `f` must have a type similar to:

``` haskell
f :: World -> (a,World)
```

Not only this, but we can also note that we always follow the same usage pattern:

``` haskell
let (y,w1) = action1 w0 in
let (z,w2) = action2 w1 in
let (t,w3) = action3 w2 in
...
```

Each action can take from 0 to n parameters.
And in particular, each action can take a parameter from the result of a line above.

For example, we could also have:

``` haskell
let (_,w1) = action1 x w0   in
let (z,w2) = action2 w1     in
let (_,w3) = action3 x z w2 in
...
```

And of course `actionN w :: (World) -> (a,World)`.

> IMPORTANT, there are only two important patterns to consider:
>
> ```
> let (x,w1) = action1 w0 in
> let (y,w2) = action2 x w1 in
> ```
>
> and
>
> ```
> let (_,w1) = action1 w0 in
> let (y,w2) = action2 w1 in
> ```

![Jocker pencil trick](http://yannesposito.com/Scratch/img/blog/Haskell-the-Hard-Way/jocker_pencil_trick.jpg)

Now, we will do a magic trick.
We will make the temporary world symbol "disappear".
We will `bind` the two lines.
Let's define the `bind` function.
Its type is quite intimidating at first:

``` haskell
bind :: (World -> (a,World))
        -> (a -> (World -> (b,World)))
        -> (World -> (b,World))
```

But remember that `(World -> (a,World))` is the type for an IO action.
Now let's rename it for clarity:

``` haskell
type IO a = World -> (a, World)
```

Some example of functions:

``` haskell
getLine :: IO String
print :: Show a => a -> IO ()
```

`getLine` is an IO action which takes a world as parameter and returns a couple `(String,World)`.
Which can be summarized as: `getLine` is of type `IO String`.
Which we also see as, an IO action which will return a String "embeded inside an IO".

The function `print` is also interesting.
It takes one argument which can be shown.
In fact it takes two arguments.
The first is the value to print and the other is the state of world.
It then returns a couple of type `((),World)`.
This means it changes the state of the world, but doesn't yield anymore data.

This type helps us simplify the type of `bind`:

``` haskell
bind :: IO a
        -> (a -> IO b)
        -> IO b
```

It says that `bind` takes two IO actions as parameter and return another IO action.

Now, remember the _important_ patterns. The first was:

``` haskell
let (x,w1) = action1 w0 in
let (y,w2) = action2 x w1 in
(y,w2)
```

Look at the types:

``` haskell
action1  :: IO a
action2  :: a -> IO b
(y,w2)   :: IO b
```

Doesn't it seem familiar?

``` haskell
(bind action1 action2) w0 =
    let (x, w1) = action1 w0
        (y, w2) = action2 x w1
    in  (y, w2)
```

The idea is to hide the World argument with this function. Let's go:
As an example imagine if we wanted to simulate:

``` haskell
let (line1,w1) = getLine w0 in
let ((),w2) = print line1 in
((),w2)
```

Now, using the bind function:

``` haskell
(res,w2) = (bind getLine (\l -> print l)) w0
```

As print is of type `(World -> ((),World))`, we know `res = ()` (null type).
If you didn't see what was magic here, let's try with three lines this time.

``` haskell
let (line1,w1) = getLine w0 in
let (line2,w2) = getLine w1 in
let ((),w3) = print (line1 ++ line2) in
((),w3)
```

Which is equivalent to:

``` haskell
(res,w3) = bind getLine (\line1 ->
             bind getLine (\line2 ->
               print (line1 ++ line2)))
```

Didn't you notice something?
Yes, no temporary World variables are used anywhere!
This is _MA_. _GIC_.

We can use a better notation.
Let's use `(>>=)` instead of `bind`.
`(>>=)` is an infix function like
`(+)`; reminder `3 + 4 ⇔ (+) 3 4`

``` haskell
(res,w3) = getLine >>=
           \line1 -> getLine >>=
           \line2 -> print (line1 ++ line2)
```

Ho Ho Ho! Happy Christmas Everyone!
Haskell has made syntactical sugar for us:

``` haskell
do
  x <- action1
  y <- action2
  z <- action3
  ...
```

Is replaced by:

``` haskell
action1 >>= \x ->
action2 >>= \y ->
action3 >>= \z ->
...
```

Note you can use `x` in `action2` and `x` and `y` in `action3`.

But what about the lines not using the `<-`?
Easy, another function `blindBind`:

``` haskell
blindBind :: IO a -> IO b -> IO b
blindBind action1 action2 w0 =
    bind action (\_ -> action2) w0
```

I didn't simplify this definition for clarity purpose.
Of course we can use a better notation, we'll use the `(>>)` operator.

And

``` haskell
do
    action1
    action2
    action3
```

Is transformed into

``` haskell
action1 >>
action2 >>
action3
```

Also, another function is quite useful.

``` haskell
putInIO :: a -> IO a
putInIO x = IO (\w -> (x,w))
```

This is the general way to put pure values inside the "IO context".
The general name for `putInIO` is `return`.
This is quite a bad name when you learn Haskell. `return` is very different from what you might be used to.


To finish, let's translate our example:

``` active haskell
import Data.Maybe

maybeRead :: Read a => String -> Maybe a
maybeRead s = case reads s of
                  [(x,"")]    -> Just x
                  _           -> Nothing
getListFromString :: String -> Maybe [Integer]
getListFromString str = maybeRead $ "[" ++ str ++ "]"
-- show
askUser :: IO [Integer]
askUser = do
  putStrLn "Enter a list of numbers (separated by commas):"
  input <- getLine
  let maybeList = getListFromString input in
      case maybeList of
          Just l  -> return l
          Nothing -> askUser

main :: IO ()
main = do
  list <- askUser
  print $ sum list
-- /show  
```

Is translated into:

``` active haskell
import Data.Maybe

maybeRead :: Read a => String -> Maybe a
maybeRead s = case reads s of
                  [(x,"")]    -> Just x
                  _           -> Nothing
getListFromString :: String -> Maybe [Integer]
getListFromString str = maybeRead $ "[" ++ str ++ "]"
-- show
askUser :: IO [Integer]
askUser =
    putStrLn "Enter a list of numbers (sep. by commas):" >>
    getLine >>= \input ->
    let maybeList = getListFromString input in
      case maybeList of
        Just l -> return l
        Nothing -> askUser

main :: IO ()
main = askUser >>=
  \list -> print $ sum list
-- /show  
```

You can compile this code to verify it keeps working.

Imagine what it would look like without the `(>>)` and `(>>=)`.



### Monads

![Dali, reve. It represents a weapon out of the mouth of a tiger, itself out of the mouth of another tiger, itself out of the mouth of a fish itself out of a grenade. I could have choosen a picture of the Human centipede as it is a very good representation of what a monad really is. But just to thing about it, I find this disgusting and that wasn't the purpose of this document.](http://yannesposito.com/Scratch/img/blog/Haskell-the-Hard-Way/dali_reve.jpg)

Now the secret can be revealed: `IO` is a _monad_.
Being a monad means you have access to some syntactical sugar with the `do` notation.
But mainly, you have access to a coding pattern which will ease the flow of your code.

> **Important remarks**:
>
> - Monad are not necessarily about effects!
>   There are a lot of _pure_ monads.
> - Monad are more about sequencing

For the Haskell language `Monad` is a type class.
To be an instance of this type class, you must provide the functions `(>>=)` and `return`.
The function `(>>)` will be derived from `(>>=)`.
Here is how the type class `Monad` is declared (mostly):

``` haskell
class Monad m  where
  (>>=) :: m a -> (a -> m b) -> m b
  return :: a -> m a

  (>>) :: m a -> m b -> m b
  f >> g = f >>= \_ -> g

  -- You should generally safely ignore this function
  -- which I believe exists for historical reason
  fail :: String -> m a
  fail = error
```

> Remarks:
>
> - the keyword `class` is not your friend.
>   A Haskell class is _not_ a class like in object model.
>   A Haskell class has a lot of similarities with Java interfaces.
>   A better word should have been `typeclass`.
>   That means a set of types.
>   For a type to belong to a class, all functions of the class must be provided for this type.
> - In this particular example of type class, the type `m` must be a type that takes an argument.
>   for example `IO a`, but also `Maybe a`, `[a]`, etc...
> - To be a useful monad, your function must obey some rules.
>   If your construction does not obey these rules strange things might happens:
>
>   ```
>   return a >>= k  ==  k a
>   m >>= return  ==  m
>   m >>= (\x -> k x >>= h)  ==  (m >>= k) >>= h
>   ```

#### Maybe is a monad

There are a lot of different types that are instance of `Monad`.
One of the easiest to describe is `Maybe`.
If you have a sequence of `Maybe` values, you can use monads to manipulate them.
It is particularly useful to remove very deep `if..then..else..` constructions.

Imagine a complex bank operation. You are eligible to gain about 700€ only
if you can afford to follow a list of operations without being negative.

``` active haskell
deposit  value account = account + value
withdraw value account = account - value

eligible :: (Num a,Ord a) => a -> Bool
eligible account =
  let account1 = deposit 100 account in
    if (account1 < 0)
    then False
    else
      let account2 = withdraw 200 account1 in
      if (account2 < 0)
      then False
      else
        let account3 = deposit 100 account2 in
        if (account3 < 0)
        then False
        else
          let account4 = withdraw 300 account3 in
          if (account4 < 0)
          then False
          else
            let account5 = deposit 1000 account4 in
            if (account5 < 0)
            then False
            else
              True

main = do
  print $ eligible 300 -- True
  print $ eligible 299 -- False
```



Now, let's make it better using Maybe and the fact that it is a Monad

``` active haskell
deposit :: (Num a) => a -> a -> Maybe a
deposit value account = Just (account + value)

withdraw :: (Num a,Ord a) => a -> a -> Maybe a
withdraw value account = if (account < value)
                         then Nothing
                         else Just (account - value)

eligible :: (Num a, Ord a) => a -> Maybe Bool
eligible account = do
  account1 <- deposit 100 account
  account2 <- withdraw 200 account1
  account3 <- deposit 100 account2
  account4 <- withdraw 300 account3
  account5 <- deposit 1000 account4
  Just True

main = do
  print $ eligible 300 -- Just True
  print $ eligible 299 -- Nothing
```



Not bad, but we can make it even better:

``` active haskell
deposit :: (Num a) => a -> a -> Maybe a
deposit value account = Just (account + value)

withdraw :: (Num a,Ord a) => a -> a -> Maybe a
withdraw value account = if (account < value)
                         then Nothing
                         else Just (account - value)

eligible :: (Num a, Ord a) => a -> Maybe Bool
eligible account =
  deposit 100 account >>=
  withdraw 200 >>=
  deposit 100  >>=
  withdraw 300 >>=
  deposit 1000 >>
  return True

main = do
  print $ eligible 300 -- Just True
  print $ eligible 299 -- Nothing
```

We have proven that Monads are a good way to make our code more elegant.
Note this idea of code organization, in particular for `Maybe` can be used
in most imperative language.
In fact, this is the kind of construction we make naturally.

> An important remark:
>
> The first element in the sequence being evaluated to `Nothing` will stop
> the complete evaluation.
> This means you don't execute all lines.
> You have this for free, thanks to laziness.

You could also replay these example with the definition of `(>>=)` for `Maybe`
in mind:

``` haskell
instance Monad Maybe where
    (>>=) :: Maybe a -> (a -> Maybe b) -> Maybe b
    Nothing  >>= _  = Nothing
    (Just x) >>= f  = f x

    return x = Just x
```

The `Maybe` monad proved to be useful while being a very simple example.
We saw the utility of the `IO` monad.
But now a cooler example, lists.



#### The list monad

![Golconde de Magritte](http://yannesposito.com/Scratch/img/blog/Haskell-the-Hard-Way/golconde.jpg)

The list monad helps us to simulate non deterministic computations.
Here we go:

``` active haskell
import Control.Monad (guard)

allCases = [1..10]

resolve :: [(Int,Int,Int)]
resolve = do
              x <- allCases
              y <- allCases
              z <- allCases
              guard $ 4*x + 2*y < z
              return (x,y,z)

main = do
  print resolve
```

MA. GIC.

For the list monad, there is also a syntactical sugar:

``` haskell
  print $ [ (x,y,z) | x <- allCases,
                      y <- allCases,
                      z <- allCases,
                      4*x + 2*y < z ]
```

I won't list all the monads, but there are many monads.
Using monads simplifies the manipulation of several notions in pure languages.
In particular, monad are very useful for:

- IO,
- non deterministic computation,
- generating pseudo random numbers,
- keeping configuration state,
- writing state,
- ...

If you have followed me until here, then you've done it!
You know monads (Well, you'll certainly need to practice a bit to get used to them and to understand when you can use them and create your own. But you already made a big step in this direction.)!

## Appendix

This section is not so much about learning Haskell.
It is just here to discuss some details further.


### More on Infinite Tree

In the section [Infinite Structures](#infinite-structures) we saw some simple
constructions.
Unfortunately we removed two properties from our tree:

1. no duplicate node value
2. well ordered tree

In this section we will try to keep the first property.
Concerning the second one, we must relax it but we'll discuss how to
keep it as much as possible.

Our first step is to create some pseudo-random number list:

``` haskell
shuffle = map (\x -> (x*3123) `mod` 4331) [1..]
```

Just as a reminder, here is the definition of `treeFromList`

``` haskell
treeFromList :: (Ord a) => [a] -> BinTree a
treeFromList []    = Empty
treeFromList (x:xs) = Node x (treeFromList (filter (<x) xs))
                             (treeFromList (filter (>x) xs))
```

and `treeTakeDepth`:

``` haskell
treeTakeDepth _ Empty = Empty
treeTakeDepth 0 _     = Empty
treeTakeDepth n (Node x left right) = let
          nl = treeTakeDepth (n-1) left
          nr = treeTakeDepth (n-1) right
          in
              Node x nl nr
```

See the result of:

``` active haskell
import Data.List
data BinTree a = Empty
                 | Node a (BinTree a) (BinTree a)
                  deriving (Eq,Ord)

-- declare BinTree a to be an instance of Show
instance (Show a) => Show (BinTree a) where
  -- will start by a '<' before the root
  -- and put a : a begining of line
  show t = "< " ++ replace '\n' "\n: " (treeshow "" t)
    where
    treeshow pref Empty = ""
    treeshow pref (Node x Empty Empty) =
                  (pshow pref x)

    treeshow pref (Node x left Empty) =
                  (pshow pref x) ++ "\n" ++
                  (showSon pref "`--" "   " left)

    treeshow pref (Node x Empty right) =
                  (pshow pref x) ++ "\n" ++
                  (showSon pref "`--" "   " right)

    treeshow pref (Node x left right) =
                  (pshow pref x) ++ "\n" ++
                  (showSon pref "|--" "|  " left) ++ "\n" ++
                  (showSon pref "`--" "   " right)

    -- show a tree using some prefixes to make it nice
    showSon pref before next t =
                  pref ++ before ++ treeshow (pref ++ next) t

    -- pshow replace "\n" by "\n"++pref
    pshow pref x = replace '\n' ("\n"++pref) (show x)

    -- replace on char by another string
    replace c new string =
      concatMap (change c new) string
      where
          change c new x
              | x == c = new
              | otherwise = x:[] -- "x"

shuffle = map (\x -> (x*3123) `mod` 4331) [1..]

treeFromList :: (Ord a) => [a] -> BinTree a
treeFromList []    = Empty
treeFromList (x:xs) = Node x (treeFromList (filter (<x) xs))
                             (treeFromList (filter (>x) xs))
treeTakeDepth _ Empty = Empty
treeTakeDepth 0 _     = Empty
treeTakeDepth n (Node x left right) = let
          nl = treeTakeDepth (n-1) left
          nr = treeTakeDepth (n-1) right
          in
              Node x nl nr
-- show
main = do
      putStrLn "take 10 shuffle"
      print $ take 10 shuffle
      putStrLn "\ntreeTakeDepth 4 (treeFromList shuffle)"
      print $ treeTakeDepth 4 (treeFromList shuffle)
-- /show
```

Yay! It ends!
Beware though, it will only work if you always have something to put into a branch.

For example

``` haskell
treeTakeDepth 4 (treeFromList [1..])
```

will loop forever.
Simply because it will try to access the head of `filter (<1) [2..]`.
But `filter` is not smart enought to understand that the result is the empty list.

Nonetheless, it is still a very cool example of what non strict programs have to offer.

Left as an exercise to the reader:

- Prove the existence of a number `n` so that `treeTakeDepth n (treeFromList shuffle)` will enter an infinite loop.
- Find an upper bound for `n`.
- Prove there is no `shuffle` list so that, for any depth, the program ends.

In order to resolve these problem we will modify slightly our
`treeFromList` and `shuffle` function.

A first problem, is the lack of infinite different number in our implementation of `shuffle`.
We generated only `4331` different numbers.
To resolve this we make a slightly better `shuffle` function.

``` haskell
shuffle = map rand [1..]
          where
              rand x = ((p x) `mod` (x+c)) - ((x+c) `div` 2)
              p x = m*x^2 + n*x + o -- some polynome
              m = 3123
              n = 31
              o = 7641
              c = 1237
```

This shuffle function has the property (hopefully) not to have an upper nor lower bound.
But having a better shuffle list isn't enough not to enter an infinite loop.

Generally, we cannot decide whether `filter (<x) xs` is empty.
Then to resolve this problem, I'll authorize some error in the creation of our binary tree.
This new version of code can create binary tree which don't have the following property for some of its nodes:

> Any element of the left (resp. right) branch must all be strictly inferior (resp. superior) to the label of the root.

Remark it will remains _mostly_ an ordered binary tree.
Furthermore, by construction, each node value is unique in the tree.

Here is our new version of `treeFromList`. We simply have replaced `filter` by `safefilter`.

``` haskell
treeFromList :: (Ord a, Show a) => [a] -> BinTree a
treeFromList []    = Empty
treeFromList (x:xs) = Node x left right
          where
              left = treeFromList $ safefilter (<x) xs
              right = treeFromList $ safefilter (>x) xs
```

This new function `safefilter` is almost equivalent to `filter` but don't enter infinite loop if the result is a finite list.
If it cannot find an element for which the test is true after 10000 consecutive steps, then it considers to be the end of the search.

``` haskell
safefilter :: (a -> Bool) -> [a] -> [a]
safefilter f l = safefilter' f l nbTry
  where
      nbTry = 10000
      safefilter' _ _ 0 = []
      safefilter' _ [] _ = []
      safefilter' f (x:xs) n =
                  if f x
                     then x : safefilter' f xs nbTry
                     else safefilter' f xs (n-1)
```

Now run the program and be happy:

``` active haskell

import Debug.Trace (trace)
import Data.List
data BinTree a = Empty
                 | Node a (BinTree a) (BinTree a)
                  deriving (Eq,Ord)
instance (Show a) => Show (BinTree a) where
  -- will start by a '<' before the root
  -- and put a : a begining of line
  show t = "< " ++ replace '\n' "\n: " (treeshow "" t)
    where
    treeshow pref Empty = ""
    treeshow pref (Node x Empty Empty) =
                  (pshow pref x)

    treeshow pref (Node x left Empty) =
                  (pshow pref x) ++ "\n" ++
                  (showSon pref "`--" "   " left)

    treeshow pref (Node x Empty right) =
                  (pshow pref x) ++ "\n" ++
                  (showSon pref "`--" "   " right)

    treeshow pref (Node x left right) =
                  (pshow pref x) ++ "\n" ++
                  (showSon pref "|--" "|  " left) ++ "\n" ++
                  (showSon pref "`--" "   " right)

    -- show a tree using some prefixes to make it nice
    showSon pref before next t =
                  pref ++ before ++ treeshow (pref ++ next) t

    -- pshow replace "\n" by "\n"++pref
    pshow pref x = replace '\n' ("\n"++pref) (" " ++ show x)

    -- replace on char by another string
    replace c new string =
      concatMap (change c new) string
      where
          change c new x
              | x == c = new
              | otherwise = x:[] -- "x"

treeTakeDepth _ Empty = Empty
treeTakeDepth 0 _     = Empty
treeTakeDepth n (Node x left right) = let
          nl = treeTakeDepth (n-1) left
          nr = treeTakeDepth (n-1) right
          in
              Node x nl nr
shuffle = map rand [1..]
          where
              rand x = ((p x) `mod` (x+c)) - ((x+c) `div` 2)
              p x = m*x^2 + n*x + o -- some polynome
              m = 3123
              n = 31
              o = 7641
              c = 1237
treeFromList :: (Ord a, Show a) => [a] -> BinTree a
treeFromList []    = Empty
treeFromList (x:xs) = Node x left right
          where
              left = treeFromList $ safefilter (<x) xs
              right = treeFromList $ safefilter (>x) xs
safefilter :: (a -> Bool) -> [a] -> [a]
safefilter f l = safefilter' f l nbTry
  where
      nbTry = 10000
      safefilter' _ _ 0 = []
      safefilter' _ [] _ = []
      safefilter' f (x:xs) n =
                  if f x
                     then x : safefilter' f xs nbTry
                     else safefilter' f xs (n-1)
-- show
main = do
      putStrLn "take 10 shuffle"
      print $ take 10 shuffle
      putStrLn "\ntreeTakeDepth 8 (treeFromList shuffle)"
      print $ treeTakeDepth 8 (treeFromList $ shuffle)
-- /show
```

You should realize the time to print each value is different.
This is because Haskell compute each value when it needs it.
And in this case, this is when asked to print it on the screen.

Impressively enough, try to replace the depth from `8` to `100`.
It will work without killing your RAM!
The flow and the memory management is done naturally by Haskell.

Left as an exercise to the reader:

- Even with large constant value for `deep` and `nbTry`, it seems to work nicely. But in the worst case, it can be exponential.
  Create a worst case list to give as parameter to `treeFromList`.
  _hint_: think about (`[0,-1,-1,....,-1,1,-1,...,-1,1,...]`).
- I first tried to implement `safefilter` as follow:
  <pre>
  safefilter' f l = if filter f (take 10000 l) == []
                    then []
                    else filter f l
  </pre>
  Explain why it doesn't work and can enter into an infinite loop.
- Suppose that `shuffle` is real random list with growing bounds.
  If you study a bit this structure, you'll discover that with probability 1,
  this structure is finite.
  Using the following code
  (suppose we could use `safefilter'` directly as if was not in the where of safefilter)
  find a definition of `f` such that with probability `1`,
  treeFromList' shuffle is infinite. And prove it.
  Disclaimer, this is only a conjecture.

``` haskell
treeFromList' []  n = Empty
treeFromList' (x:xs) n = Node x left right
    where
        left = treeFromList' (safefilter' (<x) xs (f n)
        right = treeFromList' (safefilter' (>x) xs (f n)
        f = ???
```


## Thanks

Thanks to [`/r/haskell`](http://reddit.com/r/haskell) and
[`/r/programming`](http://reddit.com/r/programming).
Your comment were most than welcome.

Particularly, I want to thank [Emm](https://github.com/Emm) a thousand times
for the time he spent on correcting my English.
Thank you man.
