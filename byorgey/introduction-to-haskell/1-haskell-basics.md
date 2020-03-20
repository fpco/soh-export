# 1: Haskell Basics

# What is Haskell?

Haskell is a lazy, functional programming language created in the late 1980s by a committee of academics. There were a plethora of lazy functional languages around, everyone had their favorite, and it was hard to communicate ideas. So a bunch of people got together and designed a new language, taking some of the best ideas from existing languages (and a few new ideas of their own). Haskell was born.

So what is Haskell like? Haskell is:

## Functional

There is no precise, accepted meaning for the term “functional”. But when we say that Haskell is a functional language, we usually have in mind two things:

1. Functions are first-class, that is, functions are values which can be used in exactly the same ways as any other sort of value.
2. The meaning of Haskell programs is centered around evaluating expressions rather than executing instructions.

Taken together, these result in an entirely different way of thinking about programming. Much of this course will be spent exploring this way of thinking.

## Pure
Haskell expressions are always referentially transparent, that is:

- No mutation! Everything (variables, data structures...) is immutable.
- Expressions never have “side effects” (like updating global variables or printing to the screen).
- Calling the same function with the same arguments results in the same output every time.

This may sound crazy at this point. How is it even possible to get anything done without mutation or side
effects? Well, it certainly requires a shift in thinking (if you’re used to an imperative or object-oriented
paradigm). But once you’ve made the shift, there are a number of wonderful benefits:

- Equational reasoning and refactoring: In Haskell one can always “replace equals by equals”, just like you learned in algebra class.
- Parallelism: Evaluating expressions in parallel is easy when they are guaranteed not to affect one another.
- Fewer headaches: Simply put, unrestricted effects and action-at-a-distance makes for programs that are hard to debug, maintain, and reason about.

## Lazy

In Haskell, expressions are not evaluated until their results are actually needed. This is a simple decision
with far-reaching consequences, which we will explore throughout the semester. Some of the consequences
include:

- It is easy to define a new control structure just by defining a function.
- It is possible to define and work with infinite data structures.
- It enables a more compositional programming style (see wholemeal programming below).
- One major downside, however, is that reasoning about time and space usage becomes much more complicated!

## Statically Typed

Every Haskell expression has a type, and types are all checked at compile-time. Programs with type errors
will not even compile, much less run.

# Themes

Throughout this course, we will focus on three main themes.

## Types

Static type systems can seem annoying. In fact, in languages like C++ and Java, they are annoying. But this
isn’t because static type systems per se are annoying; it’s because C++ and Java’s type systems are
insufficiently expressive! This semester we’ll take a close look at Haskell’s type system, which

- Helps clarify thinking and express program structure

The first step in writing a Haskell program is usually to write down all the types. Because Haskell’s
type system is so expressive, this is a non-trivial design step and

- Serves as a form of documentation

Given an expressive type system, just looking at a function’s type tells you a lot about what the
function might do and how it can be used, even before you have read a single word of written
documentation.

- Turns run-time errors into compile-time errors

It’s much better to be able to fix errors up front than to just test a lot and hope for the best. “If it
compiles, it must be correct” is mostly facetious (it’s still quite possible to have errors in logic even in
a type-correct program), but it happens in Haskell much more than in other languages.

## Abstraction

“Don’t Repeat Yourself” is a mantra often heard in the world of programming. Also known as the “Abstraction
Principle”, the idea is that nothing should be duplicated: every idea, algorithm, and piece of data should occur
exactly once in your code. Taking similar pieces of code and factoring out their commonality is known as the
process of _abstraction_.

Haskell is very good at abstraction: features like parametric polymorphism, higher-order functions, and type
classes all aid in the fight against repetition. Our journey through Haskell this semester will in large part be a
journey from the specific to the abstract.

## Wholemeal programming

Another theme we will explore is _wholemeal programming_. A quote from Ralf Hinze:

> “Functional languages excel at wholemeal programming, a term coined by Geraint Jones.
Wholemeal programming means to think big: work with an entire list, rather than a sequence of
elements; develop a solution space, rather than an individual solution; imagine a graph, rather than a single path. The wholemeal approach often offers new insights or provides new
perspectives on a given problem. It is nicely complemented by the idea of projective
programming: first solve a more general problem, then extract the interesting bits and pieces by
transforming the general program into more specialised ones.”

For example, consider this pseudocode in a C/Java-ish sort of language:

```
lst = [2,3,5,7,11]

int total = 0;
for ( int i = 0; i < lst.length; i++) {
  total = total + 3 * lst[i];
}

print total;
```
This code suffers from what Richard Bird refers to as “indexitis”: it has to worry about the low-level details of
iterating over an array by keeping track of a current index. It also mixes together what can more usefully be
thought of as two separate operations: multiplying every item in a list by 3, and summing the results.

In Haskell, we can just write

```active haskell
lst = [2,3,5,7,11]

total = sum (map (3*) lst)

main = print total
```
In this course we’ll explore the shift in thinking represented by this way of programming, and examine how and why Haskell makes it possible.

# Declarations and Variables

Here is some Haskell code:
```active haskell 
x :: Int
x = 3
-- Note that comments are preceded by two hyphens
{- or enclosed
   in curly brace/hypens pairs. -}
-- x = 4

main = print x
```

Try uncommenting the line `x=4` above; you should see the error `Multiple declarations of 'x'`.

The above code declares a variable `x` with type `Int` (`::` is pronounced “has type”) and declares the value of `x` to be 3. Note that this will be the value of `x` forever (at least, in this particular program). The value of `x` cannot be changed later.

In Haskell, _variables are not mutable boxes_; they are just names for values!

Put another way, `=` does not denote _assignment_ like it does in many other languages. Instead, `=` denotes _definition_, like it does in mathematics. That is, `x = 4` should not be read as “`x` gets 4” or “assign 4 to `x`”, but as “`x` is defined to be 4”.

What do you think this code means?
```haskell
y :: Int
y = y + 1
```

@@@
Because `=` denotes _definition_ rather than _assignment_, this does **not** increment the value of `y`. Instead, this statement is taken as a _recursive_ definition; evaluation of `y` yields
```haskell
y = y + 1
  = (y + 1) + 1
  = ((y + 1) + 1) + 1
  =
  .
  .
  .
```
resulting in an endless loop.
@@@

# Basic Types

```haskell
-- Machine-sized integers
i :: Int
i = -78
```

`Int`s are guaranteed by the Haskell language standard to accommodate values at least up to ±2^29, but the
exact size depends on your architecture. For example, on many 64-bit architectures the range is ±2^63 . You can find the range on a particular machine by evaluating the following:
```active haskell
main = print 
-- show
  (minBound :: Int, maxBound :: Int)
-- /show
```

[Note that idiomatic Haskell uses `camelCase` for identifier names. If you don’t like it, tough luck.]

The `Integer` type, on the other hand, is limited only by the amount of memory on your machine.

```active haskell
-- Arbitrary-precision integers
n :: Integer
n = 1234567890987654321987340982334987349872349874534

reallyBig :: Integer
reallyBig = 2^(2^(2^(2^2)))

numDigits :: Int
numDigits = length (show reallyBig)

main = print numDigits
```

For floating-point numbers, there is `Double`:

```haskell
d1, d2 :: Double
d1 = 4.5387
d2 = 6.2831e-4
```

There is also a single-precision floating point type, `Float`, but it is not used much.

```haskell
-- Booleans
b1, b2 :: Bool
b1 = True
b2 = False

-- Unicode characters
c1, c2, c3 :: Char
c1 = 'x'
c2 = 'Ø'
c3 = 'ダ'

-- Strings are lists of characters with special syntax
s :: String
s = "Hello, Haskell!"
```

# GHCi

*GHCi* is an interactive Haskell REPL (Read-Eval-Print-Loop) that comes with GHC. At the GHCi prompt, you
can evaluate expressions, load Haskell files with `:load` (`:l`) (and reload them with `:reload` (`:r`)), ask for the type of an expression with `:type` (`:t`), and many other things (try `:?` for a list of commands). 

# Arithmetic
Try evaluating each of the following expressions in GHCi:

```haskell
ex01 = 3 + 2
ex02 = 19 - 27
ex03 = 2.35 * 8.6
ex04 = 8.7 / 3.1
ex05 = mod 19 3
ex06 = 19 `mod` 3
ex07 = 7 ^ 222
ex08 = (-3) * (-7)
```
Note how \`backticks\` make a function name into an infix operator. Note also that negative numbers must
often be surrounded by parentheses, to avoid having the negation sign parsed as subtraction. (Yes, this is ugly. I’m sorry.)
This, however, gives an error:

```active haskell
i = 30 :: Int
n = 10 :: Integer
main = print (i + n)
```

Addition is only between values of the same numeric type, and Haskell does not do implicit conversion. You must explicitly convert with:

- `fromIntegral`: converts from any integral type (`Int` or `Integer`) to any other numeric type.
- `round`, `floor`, `ceiling`: convert floating-point numbers to `Int` or `Integer`.

Now try this:

```active haskell
i = 30 :: Int
main = print (i / i)
```
This is an error since `/` performs floating-point division only. For integer division we can use `div`:

```active haskell
i = 30 :: Int
main = print (i `div` i, 12 `div` 5)
```

If you are used to other languages which do implicit conversion of numeric types, this can all seem rather prudish and annoying at first. However, I promise you’ll get used to it, and in time you may even come to appreciate it. Implicit numeric conversion encourages sloppy thinking about numeric code.

# Boolean Logic

As you would expect, Boolean values can be compared with `(&&)` (logical and), `(||)` (logical or), and `not`. For example,

```haskell
ex11 = True && False
ex12 = not (False || True)
```

Things can be compared for equality with `(==)` and `(/=)`, or compared for order using `(<)`, `(>)`, `(<=)`, and `(>=)`.

```haskell
ex13 = ('a' == 'a')
ex14 = (16 /= 3)
ex15 = (5 > 3) && ('p' <= 'q')
ex16 = "Haskell" > "C++"
```

Haskell also has `if` expressions: `if b then t else f` is an expression which evaluates to `t` if the Boolean expression `b` evaluates to `True`, and `f` if `b` evaluates to `False`. Notice that `if` expressions are very different than `if` statements. For example, with an `if` statement, the `else` part can be optional; an omitted `else` clause means "if the test evaluates to `False` then do nothing". With an `if` expression, on the other hand, the `else` part is required, since the `if` expression must result in some value.

# Defining Basic Functions
We can write functions on integers by cases.

```active haskell
-- Compute the sum if the integers from 1 to n.
sumtorial :: Integer -> Integer
sumtorial 0 = 0
sumtorial n = n + sumtorial (n - 1)

main = print (sumtorial 10)
```

Note that the syntax for the type of a function `sumtorial :: Integer -> Integer` says that `sumtorial` is a function which takes an `Integer` as input and yields another `Integer` as output.

Each clause is checked in order from top to bottom, and the first matching clause is chosen. For example, `sumtorial 0` evaluates to `0`, since the first clause is matched. `sumtorial 3` does not match the first clause (`3` is not `0`), so the second clause is tried. A variable like `n` matches anything, so the second clause matches and `sumtorial 3` evaluates to `3 + sumtorial (3-1)` (which can then be evaluated further).

Choices can also be made based on arbitrary Boolean expressions using _guards_. For example:
```active haskell
hailstone :: Integer -> Integer
hailstone n
  | n `mod` 2 == 0 = n `div` 2
  | otherwise      = 3 * n + 1
  
main = print (hailstone 3)
```
Any number of guards can be associated with each clause of a function definition, each of which is a Boolean expression. If the clause’s patterns match, the guards are evaluated in order from top to bottom, and the first one which evaluates to `True` is chosen. If none of the guards evaluate to `True`, matching continues with the next clause.

For example, suppose we evaluate `hailstone 3`. First, `3` is matched against `n`, which succeeds (since a variable matches anything). Next, <code>(n \`mod\` 2 == 0)</code> is evaluated; it is `False` since `n = 3` does not result in a remainder of `0` when divided by `2`. `otherwise` is just a convenient synonym for `True`, so the second
guard is chosen, and the result of `hailstone 3` is thus `3 * 3 + 1 = 10`.

As a more complex (but more contrived) example:

```active haskell
foo :: Integer -> Integer
foo 0 = 16
foo 1
  | "Haskell" > "C++" = 3
  | otherwise         = 4
foo n
  | n < 0           = 0
  | n `mod` 17 == 2 = -43
  | otherwise       = n + 3
  
main = print [foo (-3), foo 0, foo 1, foo 36, foo 38]

```

As a final note about Boolean expressions and guards, suppose we wanted to abstract out the test of
evenness used in defining `hailstone`. A first attempt is shown below:

```active haskell
isEven :: Integer -> Bool
isEven n
  | n `mod` 2 == 0 = True
  | otherwise      = False
  
main = print [isEven 2, isEven 5]
```

This _works_, but is much too complicated. Can you see why?

@@@
The code returns `True` or `False`, depending whether <code>n \`mod\` 2 == 0</code> is `True` or `False`. It would have been much simpler to write it as

```haskell
isEven : Integer -> Bool
isEven n = n `mod` 2 == 0
```
@@@

# Pairs

We can pair things together like so:

```haskell
p :: (Int, Char)
p = (3, 'x')
```

Notice that the (x,y) notation is used both for the _type_ of a pair and a pair _value_.

The elements of a pair can be extracted again with _pattern matching_:

```active haskell
sumPair :: (Int, Int) -> Int
sumPair (x,y) = x + y

main = print (sumPair (3,4))
```

Haskell also has triples, quadruples, ... but you should never use them. As we’ll see in the next lesson, there are much better ways to package three or more pieces of information together.

# Using functions

To apply a function to some arguments, just list the arguments after the function, separated by spaces, like this:

```active haskell
f x y z = x + y + z
main = print (f 3 17 8)
```

The above example applies the function `f` to the three arguments `3`, `17`, and `8`.

Note that function application has higher precedence than any infix operators! So it would be incorrect to write `f 3 n+1 7` if you intend to pass `n+1` as the second argument to `f`, because this parses as `(f 3 n) + (1 7)`. Instead, one must write `f 3 (n+1) 7`.

# Constructing Lists

The simplest possible list is the empty list:

```haskell
emptyList = []
```

Other lists are built up from the empty list using the _cons_ operator, `(:)`. Cons takes an element and a list, and produces a new list with the element prepended to the front.

```haskell
a = 1 : []
b = 3 : (1 : [])
c = [2,3,4] == 2 : 3 : 4 : []
```
We can see that `[2,3,4]` notation is just convenient shorthand for `2 : 3 : 4 : []`. Note also that these are really singly-linked lists, and **not** arrays.

``` active haskell
-- /show
hailstone :: Integer -> Integer
hailstone n
  | n `mod` 2 == 0 = n `div` 2
  | otherwise      = 3 * n + 1
  
-- show
hailstoneSeq :: Integer -> [Integer]
hailstoneSeq 1 = [1]
hailstoneSeq n = n : hailstoneSeq (hailstone n)

main = print (hailstoneSeq 5)
```
We stop the hailstone sequence when we reach 1. The hailstone sequence for a general n consists of n
itself, followed by the hailstone sequence for `hailstone n`, that is, the number obtained by applying the hailstone transformation once to `n`.

# Functions on Lists

We can write functions on lists using _pattern matching_.

```active haskell
-- Compute the length of a list of Integers
intListLength :: [Integer] -> Integer
intListLength [] = 0
intListLength (x:xs) = 1 + intListLength xs

main = print (intListLength [1,2,3,4,5])
```

The first clause says that the length of an empty list is 0. The second clause says that if the input list looks like `(x:xs)`, that is, a first element xconsed onto a remaining list `xs`, then the length is one more than the length of `xs`.

Since we don’t use `x` at all we could also replace it by an underscore: `intListLength (_:xs) = 1 + intListLength xs`.

We can also use nested patterns:

```active haskell
sumEveryTwo :: [Integer] -> [Integer]
sumEveryTwo []         = [] -- Do nothing to the empty list
sumEveryTwo (x:[])     = [] -- Do nothing to lists with a single element
sumEveryTwo (x:(y:zs)) = (x + y) : sumEveryTwo zs

main = print (sumEveryTwo [1,2,3,4,5,6,7,8])
```

Note how the last clause matches a list starting with `x` and followed by... a list starting with `y` and followed by the list `zs`. We don’t actually need the extra parentheses, so `sumEveryTwo (x:y:zs) = ...` would be equivalent.

# Combining Functions

It’s good Haskell style to build up more complex functions by combining many simple ones.

```active haskell
hailstone :: Integer -> Integer
hailstone n
  | n `mod` 2 == 0 = n `div` 2
  | otherwise      = 3 * n + 1
  

hailstoneSeq :: Integer -> [Integer]
hailstoneSeq 1 = [1]
hailstoneSeq n = n : hailstoneSeq (hailstone n)

-- Compute the length of a list of Integers
intListLength :: [Integer] -> Integer
intListLength [] = 0
intListLength (x:xs) = 1 + intListLength xs

-- show
-- The number of hailstone steps needed to reach 1 from a given number
hailstoneLen :: Integer -> Integer
hailstoneLen n = intListLength (hailstoneSeq n) - 1

main = print (hailstoneLen 5)
```

This may seem inefficient to you: it generates the entire hailstone sequence first and then finds its length, which wastes lots of memory... doesn’t it? Actually, it doesn’t! Because of Haskell’s lazy evaluation, each element of the sequence is only generated as needed, so the sequence generation and list length calculation are interleaved. The whole computation uses only O(1) memory, no matter how long the sequence. [Actually, this is a tiny white lie, but explaining why (and how to fix it) will have to wait a few lessons.]

# A Word About Error Messages

Actually, six:

## Don't be scared of error messages!

GHC’s error messages can be rather long and (seemingly) scary. However, usually they’re long not because they are obscure, but because they contain a lot of useful information! Here’s an example:



```
Prelude> 'x' ++ "foo"

<interactive>:2:1:
    Couldn't match expected type `[Char]' with actual type `Char'
    In the first argument of `(++)', namely 'x'
    In the expression: 'x' ++ "foo"
    In an equation for `it': it = 'x' ++ "foo"
```

First we are told <code> Couldn't match expected type \`[Char]' with actual type \`Char' </code>. This means that something was expected to have a list type, but actually had type `Char`. What something? The next line tells us: it’s the first argument of `(++)` which is at fault, namely, `'x'`. The next lines go on to give us a bit more context.

Now we can see what the problem is: clearly `'x'` has type `Char`, as the first line said. Why would it be expected to have a list type? Well, because it is used as an argument to `(++)`, which takes a list as its first argument.

When you get a huge error message, resist your initial impulse to run away; take a deep breath; and read it carefully. You won’t necessarily understand the entire thing, but you will probably learn a lot, and you may just get enough information to figure out what the problem is.

