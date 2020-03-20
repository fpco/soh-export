# 12. The List Monad

## The Anatomy of a Monad

The goal of programming is to write programs that perform computations. When complexity reaches a certain level, we start decomposing larger computations into smaller ones. The quality of this decomposition is measured by how much coupling there is between the pieces, and how well we -- and the compiler -- can control and verify it. There is overwhelming evidence that hidden couplings are a major source of bugs in both single-threaded and (even more so!) multi-threaded code.

Pure functional programming reduces couplings to the very minimum -- it's just plugging the output of one function to the input of another. However, many traditional *notions of computation* are expressed in a pseudo-functional way: with procedures that take arguments and return results but also do some non-functional shenanigans. There is a surprisingly large class of such computations that can be turned into pure functions by just modifying their return types. 

A monad describes the way of transforming the return *type* of a particular kind of computation into a fancier *monadic type*. Functions that return a monadic type are called *monadic functions*. Each monad provides a mechanism for composing such monadic functions. 

As we have seen, the `do` notation simplifies the syntax of composing multiple monadic functions. 

## The List Monad

Let's illustrate this on a particular type of computation: the non-deterministic computation. It's the kind of computation that, instead of producing a single result, might produce many. 

### Non-Deterministic Computations

Think of a chess program that evaluates future moves. It has to anticipate the moves of a non-deterministic opponent. Only one such move will materialize, but all of them have to be taken into account in planning the strategy. The basic routine in such a program would likely be called `move`: it would take a state of the chessboard and return a new state after a move has been made. In order to make two moves, you would compose two such compuatations, etc. But what should `move` return? There are many possible moves in almost every situation. The result of `move` is non-deterministic.

A computation returning a non-deterministic result of type `a` is not a pure function, but it can be made into one by transforming its result type from `a` to a list of `a`. In essence, we create a function that returns all possible results at once. Our chess program would probably have many such non-deterministic functions as well as some regular ones. In order to be able to compose them, we define a monad. 

### Handcrafted List Monad

A monadic function returns a monadic type. Let's define the monadic type, `List a`, for non-deterministic computations. For the purpose of illustration, I'm not going to use the built-in list type, since it already is an instance of a monad and we would run into name conflicts. So here's our private version of the list:

``` haskell
data List a = Nil | Cons a (List a)
    deriving Show
```

Let's think about composing non-deterministic functions. How do we want to calculate the result of two chess moves? If the first move returns a list of options, we should apply the second move to each of the resulting positions in turn. We'll end up with a list of lists of options. If we want a composition of two monadic functions to be another monadic functions, we have to somehow turn this list of lists into a single list. We need to `join` them. With built-in lists we would just use the function `concat`:

``` haskell
concat :: [[a]] -> [a]
```

For our private implementation of `List` we have to code it ourselves:

``` active haskell
data List a = Nil | Cons a (List a)
    deriving Show

join :: List (List a) -> List a
join Nil = Nil
join (Cons xs xss) =  cat xs (join xss)

cat :: List a -> List a -> List a
cat Nil ys = ys
cat (Cons x xs) ys = Cons x (cat xs ys)

l1 = Cons 1 (Cons 2 Nil)
l2 = Cons 3 Nil

main = print $ join $ Cons l1 (Cons l2 Nil)
```

Now let's implement monadic bind. In the chess example, I said that we wanted to apply the second `move` to each of the options returned by the first `move`. We know how to do it with regular lists: we just apply `map` to it:
```
map :: (a -> b) -> [a] -> [b]
```

We can easily implement an analogous function for our `List`, but let's use this opportunity to talk about another important pattern: Type constructors that let you apply functions to their content. Then we'll come back to defining bind.

### Functor

This pattern is called a *functor* and is defined in Haskell as a type class with one function `fmap`

``` haskell
class Functor fr where
    fmap :: (a -> b) -> fr a -> fr b
```
Here, `fr` is a type constructor. `fmap` takes a function that transforms `a` into `b` and applies it to the type `fr a`, producing `fr b`. The intuition is that `fmap` *reaches inside* `f a` to transform its contents. A regular list constructor `[a]` is a `Functor` with `fmap` being just the Prelude `map`. Our own `List` is also a functor:

``` haskell
instance Functor List where
    fmap f Nil = Nil
    fmap f (Cons x xs) = Cons (f x) (fmap f xs)
```

It turns out that *every monad is a functor*, although, for historical reasons this important information is not encoded into the definition of the `Monad` typeclass. (The reverse though is not true: there are functors that are not monads.)

Let's verify this statement with some of the monads that we've seen so far. Here's the instance of `Functor` for `Maybe`:

``` haskell
instance Functor Maybe where
    fmap f Nothing = Nothing
    fmap f (Just x) = Just (f x)
```
You can see how `f` gets *under the skin* of `Maybe`.

This is the `Functor` instance for `State`:

``` haskell
instance Functor (State s) where
    fmap f act = state $ \st -> 
        let (x, st') = runState act st
        in (f x, st')
```

Here, the work is a little harder, because you have to create an action that, when the state becomes available, runs the original action and applies the function `f` to its result. State is changed from `st` to `st'` as a side effect of running the first action.

There are some simple functor laws that must be satisfied by every functor's `fmap`. One is that doint the `fmap`with the identity function doesn't change anything:
``` haskell
fmap id fra = fra
```
and the other is that `fmap` preserves function composition:
``` haskell
fmap (f . g) fra = fmap f (fmap g fra)
```

(`id` is the identity function, `id x = x`.)

## List Bind

As I said before, to bind two non-deterministic functions, we have to apply the second one to each element of the list returned by the first one and then collapse the results into a single list using `join`:

``` haskell
xs >>= k = join (fmap k xs)
```

The other monadic function `return` simply produces a one-element list:
``` haskell
return x = Cons x Nil
```

We can test this monad in action:

``` active haskell
data List a = Nil | Cons a (List a)

instance (Show a) => Show (List a) where
    show Nil = ""
    show (Cons x xs) = show x ++ ", " ++ show xs

instance Functor List where
    fmap f Nil = Nil
    fmap f (Cons x xs) = Cons (f x) (fmap f xs)

instance Monad List where
    return x = Cons x Nil
    xs >>= k = join $ fmap k xs

join :: List (List a) -> List a
join Nil = Nil
join (Cons xs xss) =  cat xs (join xss)

cat :: List a -> List a -> List a
cat Nil ys = ys
cat (Cons x xs) ys = Cons x (cat xs ys)

neighbors :: (Num a) => a -> a -> List a
neighbors x dx = Cons (x - dx) (Cons x (Cons (x + dx) Nil))


test = do
    x <- neighbors 0 100
    y <- neighbors x 1
    return y
    
main = print $ test
```

Here's an interesting tidbit: the definition of bind in terms of `fmap` and `join` works for every monad `m`:

``` haskell
ma >>= k = join $ fmap k ma
```
where `join` is defined to have the following signature:
``` haskell
join :: m (m a) -> m a
```
`join` converts the double application of the monadic type constructor into a single application -- it *flattens* it. In terms of lists, `join` converts a list of lists into a single list. 

The opposite is also true: you can define `join` in terms of bind. Remember, bind has a way of extracting a value from its first argument and applying the continuation to it. So if you start with a doubly wrapped argument, bind will extract a singly wrapped center. You can then pick `id` as the continuation, et voila! 

``` haskell
join mma = mma >>= id
```
It turns out that mathematicians prefer the definition of a monad as a functor with join and return, whereas programmers prefer to use the one with bind and return. As you can see, it doesn't really matter. 


**Ex 1.** Define `join` for `Maybe` (don't be surprised how simple it is):
``` active haskell
join :: Maybe (Maybe a) -> Maybe a
join = undefined

test1, test2, test3 :: Maybe (Maybe String)
test1 = Nothing
test2 = Just Nothing
test3 = Just (Just "a little something")

main = do
    print $ join test1
    print $ join test2
    print $ join test3
```
@@@ Show solution 
``` active haskell
join :: Maybe (Maybe a) -> Maybe a
join Nothing = Nothing
join (Just mb) = mb

test1, test2, test3 :: Maybe (Maybe String)
test1 = Nothing
test2 = Just Nothing
test3 = Just (Just "a little something")

main = do
    print $ join test1
    print $ join test2
    print $ join test3
```

@@@

**Ex 2.** Define functions `listBind` and `listReturn` for regular lists in a way analogous to `>>=` and `return` for our private lists (again, it's pretty simple):
``` active haskell
import Data.Char

listBind :: ...
listBind = undefined

listReturn :: a -> [a]
listReturn = undefined

neighbors x = [x - 1, x, x + 1]

main = do
    print $ listBind [10, 20, 30] neighbors
    print $ listBind "string" (listReturn . ord)
```

@@@ Show solution 
``` active haskell
import Data.Char

listBind :: [a] -> (a -> [b]) -> [b]
listBind xs k = concat (map k xs)

listReturn :: a -> [a]
listReturn x = [x]

neighbors x = [x - 1, x, x + 1]

main = do
    print $ listBind [10, 20, 30] neighbors
    print $ listBind "string" (listReturn . ord)
```
@@@

Since we are exploring different ways of representing a monad, there's another one that stresses composability. On several occasions I described a monad as a means to compose functions whose return types are embellished. This can be implemented by *bind*ing the result of the first function to the second function. But it can also be done by composing two monadic functions using a special composition operator. 

Regular functions are composed right to left using the dot operator, as in `g . f` (pass the result of `f` to `g`). Here's the signature of dot:
``` haskell
(.) :: (b -> c) -> (a -> b) -> (a -> c)
```
It takes a function from `b` to `c` and a function from `a` to `b` and composes them into one function that goes straight from `a` to `c`.

Monadic functions can be composed likewise using the so called Kleisli operator, colloquially known as the *fish* operator: `g <=< f`. The fish is easily defined using bind:

``` haskell
(<=<) :: Monad m => (b -> m c) -> (a -> m b) -> (a -> m c)
g <=< f = \x -> f x >>= g
```
@@@ Explain!
First, I replaced regular functions in the signature of `(.)` with monadic functions. The fish applied to two such functions must produce another function, so it's natural to return a lambda. This lambda is supposed to take `x` of type `a`. The only thing we can do with such an `x` is to apply `f` to it -- it has the right signature. This gives us a result of the type `m b`. We have to somehow apply our `g` to it. That's what `>>=` is for. It has the right signature. 

It's amazing how sometimes you can derive the implementation of a generic function just from type signatures. It's like a jigsaw puzzle where pieces can only fit together one way. 
@@@

Here it is applied to the list monad:

``` active haskell
(<=<) :: Monad m => (b -> m c) -> (a -> m b) -> (a -> m c)
g <=< f = \x -> f x >>= g

f x = [x, x + 1]
g x = [x * x]

test = g <=< f

main = print $ test 7
```

**Ex 3.** Implement the other fish operator that composes from left to right:
``` active haskell
(>=>) :: Monad m => ...
f >=> g = undefined

f x = [x, x + 1]
g x = [x * x]

test = f >=> g

main = print $ test 7
```
@@@ Show solution
``` active haskell
(>=>) :: Monad m =>  (a -> m b) -> (b -> m c) -> (a -> m c)
f >=> g = \x -> f x >>= g

f x = [x, x + 1]
g x = [x * x]

test = f >=> g

main = print $ test 7
```
@@@

You can also start with the (right or left) fish and define bind in terms of it:

``` haskell
ma >>= k = (\x -> ma) >=> k
```
@@@ Explain!
This is another jigsaw puzzle. The signatures tell it all:
``` haskell
ma     :: m a
k      :: a -> m b
result :: m b
(>=>)  :: (a -> ma) -> (a -> mb) -> (a -> mb)
```
Notice that the lambda we used as the first argument to `>=>` ignores its argument `x`. A function that ignores its argument is called a constant function. You can create a constant function using `const` from Prelude:
``` haskell
-- const ma = \x -> ma
ma >>= k = const ma >=> k
```
@@@

As you can see, these formulations are equivalent. What's nice about the last one is that it's very easy to express monadic laws in it. Monadic laws (also called *axioms*) are additional conditions that must be fulfilled by every monad implementation in order to, for instance, make the `do` notation unambiguous. In terms of `return` and the fish operator, these laws simply state that the fish must be associative and that `return` is an identity of fish:

``` haskell
(f >=> g) >=> h = f >=> (g >=> h)
return >=> f = f
f >=> return = f
```
Notice an interesting thing: If you replace the fish with `*` and `return` with `1`, you get the laws of multiplication:
``` haskell
(a * b) * c = a * (b * c)
1 * a = a
a * 1 = a
```
Consider this a useful mnemonic, but there is actually a nice theory behind it. 

**Ex 4.** Express the fish operator for standard lists considering the non-deterministic function interpretation (the solution is totally unsurprising):

``` active haskell
import Data.Char

(>=>) :: ...
f >=> g = undefined

modCase c = [toLower c, toUpper c]
camelize = modCase >=> modCase

main = print $ fmap camelize "Hump"
```

@@@ Show solution
``` active haskell
import Data.Char

(>=>) :: (a -> [b]) -> (b -> [c]) -> (a -> [c])
f >=> g = \x -> concat (map g (f x))

modCase c = [toLower c, toUpper c]
camelize = modCase >=> modCase

main = print $ fmap camelize "Hump"
```
@@@


## List Monad and List Comprehension

As I mentioned before, the standard Prelude defines the instance of `Monad` for built-in lists. This lets us use the `do` notation to compose list operations. Here's a function `squares` that squares each element of a list:

``` active haskell
squares lst = do
    x <- lst
    return (x * x)

main = print $ squares [1, 2, 3]
```

We can desugar this code to see how it works internally:

``` haskell
squares lst = lst >>= \x -> return (x * x)
```

Let's expand `>>=` and `return`:

``` haskell
squares lst = 
    concat $ fmap k lst
  where
    k = \x -> [x * x]
```

Here, `fmap k` produces a list of one-element lists of squares. This list of lists is then squashed into a single list by `concat`. 

At a higher abstraction level, you may think of a `do` block as producing a list. The last `return` shows you how to generate an element of this list. The line `x <- lst` draws an element from `lst`.

Of course, `squares` can be implemented simply by using `fmap`:
``` haskell
squares = fmap sq
  where sq x = x * x
```

A more interesting example is when you draw from more than one list:

``` active haskell
pairs l1 l2 = do
    x <- l1
    y <- l2
    return (x, y)

main = print $ pairs [1, 2, 3] "abc"
```

There is a shortcut notation for `do` blocks that deals with lists called *list comprehension*. List comprehension is based on a mathematical notation for defining sets. For instance, our first example can be written as:

``` haskell
[x * x | x <- lst]
```

You read it as "a list of `x * x` where `x` is drawn from `lst`." Similarly, the second example reduces to:

``` active haskell
main = print $ [(x, y) | x <- [1..3], y <- "abc"]
```

The clauses to the right of the vertical bar (pronounced "where") are processed in order, just like lines in the `do` block. So, for instance, the second clause may depend on the result of the first one, as in this example that prints ordered pairs of integers:

``` active haskell
main = 
  print $ [(x, y) | x <- [1..4], y <- [x..4]]
```

Moreover, you may filter the elements of lists, which is not easy to accomplish using the `do` notation (there is a `guard` function for that, but it's a bit tricky so I won't explain it here). For instance, in this example we use a guard that only allows Pythagorean triples to pass through:

``` active haskell
triples =
  [(x, y, z) | z <- [1..]
             , x <- [1..z]
             , y <- [x..z]
             , x * x + y * y == z * z]

main = print $ take 4 triples
```

Notice that we are using an infinite list of `z`s (with no upper bound) so the resulting list is also infinite. However, since Haskell is lazy, the program will terminate after the first 4 results are printed. In an imperative language this list comprehension would probably be expressed as a deeply nested loop.

```
void printNTriples(int n) {
    int i = 0;
    for (int z = 1;; ++z) {
        for (int x = 1; x <= z; ++x) {
            for (int y = x; y <= z; ++y){
                if (x * x + y * y == z * z){
                    printf ("(%d, %d, %d)\n", x, y, z);
                    if (++i == n)
                        return;
                }
            }
        }
    }
}
```
Haskell's use of infinite lists or streams is a powerful idiom for structuring code. In C++ it's very hard to separate the algorithm for generating Pythagorean triples from the algorithm that prints the first n of them. For instance, in the above C++ code the control over the length of the result list happens at the innermost level of the loop. Alternatively, you could define your own lazy C++ iterator, but then you'd end up with another form of inversion of control in the implementation of `operator++` (you'd have to turn the loop inside out). If you're a C++ programmer you should try it.

In the next tutorial we'll go back to the symbolic calculator project and consolidate our monads.

## Exercises

**Ex 5.** Each card in a deck has a rank between 1 (Ace) and 13 (King) and a Suit (Club, Diamond, Heart, Spade). Write a list comprehension that generates all cards in a deck. Hint: You can encode suit as an enumeration deriving `Show` and `Enum`. The `Enum` type class will let you create ranges like `[Club .. Spade]` (put space before `..` or the parser will be confused).

``` active haskell
data Suit = ...

data Rank = Rank Int

instance Show Rank where
   ...

deck = ...

main = print deck
```
@@@ Show solution

``` active haskell
data Suit = Club | Diamond | Heart | Spade
    deriving (Show, Enum)

data Rank = Rank Int

instance Show Rank where
    show (Rank 1)  = "Ace"
    show (Rank 11) = "Jack"
    show (Rank 12) = "Queen"
    show (Rank 13) = "King"
    show (Rank i)  = show i

deck = [(Rank r, s) | s <- [Club .. Spade]
                    , r <- [1..13]]

main = print deck
```
@@@

**Ex 6.** What does this function do?
``` haskell
f [] = []
f (p:xs) = f [x | x <- xs, x < p] 
        ++   [p]
        ++ f [x | x <- xs, x >= p]
```
@@@ Show answer
It's a (very inefficient, but extremely pedagogical) implementation of quicksort.
``` active haskell
f [] = []
f (p:xs) = f [x | x <- xs, x < p] 
        ++   [p]
        ++ f [x | x <- xs, x >= p]

main = print $ f [2, 5, 1, 3, 4]
```
@@@
