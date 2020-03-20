# Haskell Basics #3

## Tuples and Lists

We have already encountered tuples prior,
albeit very briefly.
Now we will introduce them alongside their
cousin, lists.

Tuples are hereogenous, meaning that
their elements may be of different types.

``` active haskell
aTuple = (1, 2, 34)
anotherTuple = (1, "two", (3, "four"))
main = print $ (aTuple, anotherTuple)
```

The latter `anotherTuple` consists of
an integer, a string, and even another tuple.

In fact the value passed tot he print statement
is itself a tuple consisting of two tuples,
both of which are of different constituent types.

Lists, on the other hand, are homogenous,
menaing that all of their lements must
be of the same type.

``` active haskell
aList = [1, 2, 34]
anotherList = [1, "two", [3, "four"]]
main = print $ (aList, anotherList)
```

This fails to evaluate, because all elements need to be the same.

``` active haskell
aList = [1, 2, 34]
anotherList = ["one", "two", "thirty four"]
yetAnotherList = [(1, 1), (2, 3), (5, 8)]
main = print $ (aList, anotherList, yetAnotherList)
```

Now this works, because all of the values within the
lists are of the same type.

How do you tell a list apart from a tuple?

``` haskell
aListHasSquareBrackets = [1, 2, 3]
aTupleHasRoundParentheses = (1, 2, 3)
```

## List operations

Prepending an element to a list

``` active haskell
anElement = (-22)
anotherElement = 2
aList = [1, 2, 34]
yetAnotherList = anElement:anotherElement:aList
main = print $ (anElement, anotherElement, aList, yetAnotherList)
```

Prepending a list to another list

``` active haskell
anotherList = [(-22), 2]
aList = [1, 2, 34]
yetAnotherList = aList ++ anotherList
main = print $ (anotherList, aList, yetAnotherList)
```

String concatenation

``` active haskell
aStr = "Hello"
bStr = " World!"
concatenatedStr = aStr ++ bStr
main = print $ (aStr, bStr, concatenatedStr)
```

Turns out that strings are lists of characters: `[Char]`
So strings can be concatenated the same way lists are,
pretty cool, eh?

Accessing a list by index

``` active haskell
anotherList = [(-22), 2]
aList = [1, 2, 34]
yetAnotherList = aList ++ anotherList
thirdElement = yetAnotherList !! 2
main = print $ (yetAnotherList, thirdElement)
```

Note that, as with almost all other programming languages,
indices start at zero.

Other miscellaneous functions - explore the following:
 `length`, `head`, `last`, `tail`, `init`,

``` active haskell
m = [1..7]
--m = []

--x = length m
--x = head m
--x = tail m
--x = last m
--x = init m
--x = reverse m
--x = sum m
--x = product m
--x = minimum m
--x = null m
--x = maximum m

main = print $ (x)
```

Remove the `--` one line at a time to uncomment.
Be sure to try each function on both empty and non-empty lists.

Now it is time for an exercise!

Above, we have the function `maximum`.
I want you to define your own function that accomplishes the
exact same thing: return the element of the input list which
is the largest. We shall call it `listMax`

``` active haskell
listMax m
  -- your code goes here

aList = [(-3)..2]
bList = [1,(-17),3,(-3),(-10),0]
main = do
  print $ listMax aList
  print $ listMax bList
```

Clues:
@@@
  The basic building block you should build this function upon
  is `max`, which takes in two values, and evaluates to
  the larger of the two.
@@@
@@@
  Recall the `if .. then .. else` block from earlier?
  Use guards, it will  make things a lot easier.
@@@
@@@
  Recall the list functions that we just looked at.
  Functions that you may want to use `length`, `head`, and `tail`
@@@
@@@
  The trick to this is that `listMax` will need to call itself
  at some point.
  When a function calls itself, it is called recursion.
  This a key concept required to program in Haskell
  (and other functional programming languages), because
  iteration, AKA loops, are not allowed or frowned upon
  as they are an imperative programming technqiue.
@@@

Answer (not quite):
@@@
``` active haskell
listMax m
  | length m == 1 = head m
  | otherwise = max (head m) (listMax (tail m))

aList = [(-3)..2]
bList = [1,(-17),3,(-3),(-10),0]
main = do
  print $ listMax aList
  print $ listMax bList
```

Congratulations, on making it this far, for you have understood recursion.
If you have not done so already, expand the clues above, and have a read of them.

The key idea to formulating this solution is to decompose the problem.

In this case, one would think along the lines of:
"The largest element in any list,
is the largest element between its first element,
and the largest element of the rest of the list"

Thus, you express this in Haskell, as such:

``` haskell
listMax m = max (head m) (listMax (tail m))
```

What has happened here is that you have expressed the problem
using recursion, where the function calls itself on a smaller
portion of the original problem.

... and then you think, "What if the list has only got one element, as there will be no rest of the list?"
So you extend your expression above, to include:
"Except when the list has only got one element,
the largest element of this list is its solo element."

``` haskell
listMax m
  | length m == 1 = head m
  | otherwise = max (head m) (listMax (tail m))
```

What has happened here is that you have thought of a base case for the
recursion. Above, the function would call itself recursively,
on successively smaller sections of the problem. However, it does not
know when to stop - and what you have told it to do here is simply
to stop at the last element.

Great, now you have conquered recursion, including an understanding of base cases.
There is one more base case that you will need to add.

@@@

A clue to get you to the final answer:

@@@
``` haskell
cList = [] :: [Int]
```
You will need to modify `listMax` to be able to handle empty lists.
Why we need `:: [Int]` at the end of our definition:
- The Haskell compiler cannot infer the type of an empty list, as there are no values within it. Thus, when we attempt to pass it into a function containing `max` it will fail - because `max` needs to know the type of its input.
- The reason why it works for the other lists is because Haskell does `type inference`, which is, as the name suggests, inferring the types based on their contents. This is useful, because you do not need to declare all your types in the general case - just the ones which need to be explicitly stated.

``` haskell
  print $ cList
```

Add this print statement to your implementation to test it.
@@@

Answer:
@@@
``` active haskell
listMax m
  | null m = error "No maximum in empty list"
  | length m == 1 = head m
  | otherwise = max (head m) (listMax (tail m))

aList = [(-3)..2]
bList = [1,(-17),3,(-3),(-10),0]
cList = [] :: [Int]
main = do
  print $ listMax aList
  print $ listMax bList
  print $ listMax cList
```
@@@

Another exercise!

Now that you have got your your brain muscles flexing, time to try another - just to get used to the concept of recursion.

Write a `minList` function, which is exactly the same as `maxList`, except that it finds the smallest element in a list.

Answer:
@@@
``` active haskell
listMin m
  | null m = error "No minimum in empty list"
  | length m == 1 = head m
  | otherwise = min (head m) (listMin (tail m))

aList = [(-3)..2]
bList = [1,(-17),3,(-3),(-10),0]
cList = [] :: [Int]
main = do
  print $ listMin aList
  print $ listMin bList
  print $ listMin cList
```
This time around it should have been much easier.
In fact, all you really had to do, besides changing the name of the function, was to swap `max` for `min`!
@@@

Note that these implementations still leave a lot to be desired.
If you so wish, take a look at Haskell's own implementation of
`minimum` and `maximum`.
