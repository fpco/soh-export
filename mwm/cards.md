# Dynamic typing is doomed

# Introduction

Ok, the title is a bit sensationalist. I don't think dynamic typing is really dead. Different tasks call for different language characterstics, and there will always be a place for dynamically typed languages. However, I believe the decades-long debate about whether dynamic typing or static typing is the better choice for general-purpose computing is pretty much over, and dynamic typing lost.

# The arguments

The core argument for static typing is that the compiler will catch a lot of errors for you, which saves you time. While a proper test suite will catch all or most of them in any case, and using static typing doesn't eliminate the need for such a suite, taking the _test_ step out of the _edit-load-test_ cycle is as much a win as taking the compile step out of the _edit-compile-load-test_ cycle.

The core argument for dynamic typing is that providing the compiler with type information is a waste of time. Having to type in the type name as many as three times in some cases interrupts and slows development.

Both of these statements are prima facei true, and the question is which provides the larger savings. I personally spent two decades using dynamically typed languages because - well I hate typing.

## The fallacy

The problem is that you only have to provide the compiler with lots of type information for languages using primitive type systems. Since most of the popular statically typed languages do that, people don't realize that this __is__ a fallacy.

If you instead use a language with a modern type system - or even a more conventional one and a modern type engine - you can avoid providing type information most of the time. Instead, the compiler can derive the type of variables based on the operations they participate in and the constants also used in those operations, allowing for variable use that is similar to dynamically typed languages.

It's not perfect, because some operations can work on multiple types, and the compiler may not have enough information to pick an ambiguous type. But those cases are the exception, not the rule.

## The irony

What's ironic here is that a statically typed language can use the type information to cut down on the typing needed in some cases. All languages provide default behaviors for types composed from more primitive types. Dynamically typed languages do this when comparing container types, since the comparison of those is done by comparing the contained elements in an order determined by the container. Statically typed languages can also do this for creation and for union types, because they have the information required to do so where dynamically typed languages don't.

So, while the argument against static typing is that the developer has to write more code, the reality is that a language with a modern type system can derive the type information, and use it to cut down on the amount of code the developer has to write!

# An example

 I realized this interesting facet of static typing while working on [project euler problem #54](http://projecteuler.net/problem=54)

Given that Python values cutting out boilerplate, I wonder if there's some way to leverage this as completely as Haskell does. The enum types aren't a problem, and a tuple for a card should also work mostly the same. The real trick is the union type used to represent a hand. In Python I'd create another enum for the types of the hands, which would be about the same length as the union type declaration in Haskell. However, where in Haskell I'm now done, Python would seem to require creating a wrapper type to provide the comparison function.

# A comparison

So, let's go through the types involved in both languages.

Note that I'm using the Enum class from as-yet-unreleased Python 3.4. There is no Enum class in earlier versions of Python, though there are  a number of third party modules available that are similar to this. Since this was chosen for Python 3.4, it's presumably the best of breed and will become the defacto standard.

The python types haven't been put into a program, and have only been tested informally.

## Suit

Haskell provides a simple declaration for the suit of a card:

``` haskell
data Suit = Spades | Hearts | Diamonds | Clubs deriving (Show, Eq)
```

A suit is either `Spades`, `Hearts`, `Diamonds` or `Clubs`. The last bit - `deriving Eq` - tells the compiler to automatically create the code so that instances of a Suit can be compared for equality.  It doesn't provide an ordering (that would be `Ord` as well as `Eq`), because most poker games - and the problem statement - don't rank suits. This type - and all the rest - derive from `Show` so we can just `print` them for debugging.

Python is equally succinct:

``` python
Suit = Enum('Suit', 'Spades Hearts Diamonds Clubs')
```

This creates the class `Suit` as a subclass of `Enum` with the four suits as members. The comparison behaviors come from the `Enum` class, not the `Suit` type, so there's no need to specify something like a `deriving` clause.

The oddity of using strings and having to specify the name twice (once to name the created class, and once as a variable to bind it to) comes from using the *functional* shorthand for `Enum`. While the bound variable and the class name don't have to be the same, it's idiomatic to do so, and doing otherwise could result in some confusion in tracebacks and other error messages. A `class` statement could be used, but that would require binding the members to values by hand.

## Rank

The Haskell for the rank of a card is only slightly longer:

``` haskell
data Rank = Two | Three | Four | Five | Six | Seven | Eight | Nine | Ten
     | Jack | Queen | King | Ace deriving (Show, Eq, Ord, Enum)
```

For rank, we want ordered comparison, and a sequence to the entries in the list, so we add `Ord` and `Enum` to the deriving clause to get those respective behaviors.

Python is also slightly longer:

``` python
Rank = IntEnum('Rank', 'Two Three Four Five Six Seven Eight Nine Ten '
                    'Jack Queen King Ace')
```

While the `Enum` class is already iterable, it doesn't have an ordering, so we use the subclass `IntEnum` which adds that.

## Card

The representation for a card in Haskell is as obvious as the `Suit` and `Rank` types:

``` haskell
data Card = Card Rank Suit deriving Show

instance Eq Card where
    Card r1 _ == Card r2 _ = r1 == r2

instance Ord Card where
    Card r1 _ <= Card r2 _ = r1 <= r2
```

Here Haskell gets the duplicate name. The first `Card` is the name of the type, the second the name of a constructor for instances of that type. Again, idiomatic usage is that they be the same if there's only one constructor, but it's not required.

This is the first type where we can't automatically derive the behavior we want. That's because the default would include the `suit` component in the comparison, whereas we want to ignore it. So we need to provide two instance definitions - one for `Eq` and one for `Ord` - that provide the desired behavior.

Python has a similar problem:

``` python
@functools.total_ordering
class Card(object):
    def __init__(self, rank, suit):
           self.rank = rank
           self.suit = suit
    def __eq__(self, other):
          return self.rank == other.rank
    def __lt__(self, other):
          return self.rank < other.rank
```

Haskell's default for comparisons uses the features of a `Card`. In a dynamic language, what features do an don't exist can change during execution. This makes them a poor choice for use in default behaviors. So the default behavior for comparison of objects in Python is to compare their identity. That is an implementation-defined value that is only guaranteed to be unique to this object during it's existence, and not change during the life of the object.

This means that by default, an object is only equal to itself. Still not what we want, so we provide an `__eq__` method to get the proper value.

The default for ordering, on the other hand, is useless. You can't depend on two objects having the same order relationship between runs. So any time we want to have an ordering on instances of a class, we have to write at least one method, in this case `__lt__`. The `@functools.total_ordering` decorator then takes care of providing the rest of the comparison methods for a total ordering, which is what we want.

Knowing the features also means that  a `Card` constructor must always accept a `Rank` and a `Suit` and return a `Card` so a default constructor can be created by the language.  Being able to attach features dynamically is normally leveraged in a creation method in dynamic languages, so we have to  write the `__init__` method to do that. To be fair, at least one dynamic language allows the creation of instances of such types which bind the values automatically, but that does seem to be the exception rather than the rule for dynamically typed languages.

## Hand

So,  we're ready for the actual hand type. In Haskell, that's only a little bit more complicated than the preceding:

``` haskell
data Hand = HighCard [Card]
          | PairOf Rank [Card]
          | TwoPair Rank Rank [Card]
          | ThreeOf Rank [Card]
          | Straight [Card]
          | Flush [Card]
          | FullHouse Rank Rank [Card]
          | FourOf Rank [Card]
          | StraightFlush [Card]
            deriving (Show, Eq, Ord)
          -- RoyalFlush isn't listed; it's just an Ace-high StraightFlush
```

Here, we can see why Haskell has both constructors and type names. The type is `Hand`, but the constructors are the types of poker hands: `FullHouse`, `Flush`, etc.  `Hand`, like most of the previous Haskell types, derives the `Eq` and `Ord` functions for the type, just like it does for any other container type.  This does what we want - `Hand` types listed first will be less than hand types listed later, so a `Flush` is greater than a `Straight`.  Two `Hand`s of the same type will be compared by the provided `Rank`s, if any, and then the lists of `Card`s in the hand will be compared if the `Rank`s are equal. The only requirement for correct behavior is that the `Hand`s be created correctly.

In Python, this is a bit harder - there is no union type for Python. So we're going to have to provide our own discriminator. That's:

``` python
Hand = IntEnum('Hand', 'HighCard PairOf TwoPair ThreeOf Straight Flush FullHouse FourOf StraightFlush')
```

And now there's lots of choices, but no obviously good ones. Writing a single class similar to `Card`, except you then have to decide how to handle the differences between initializing the different `Hand` types. You could default the two `Rank` elements so you can leave them off when not needed, but this exposes the internal implementation in the API. One way to deal with different initializations would be a subclass for each hand type, which might be idiomatic, but would be a bit long.

You can avoid having to write custom comparisons by using a tuple that starts with a `Hand` entry and comparing those. In that case, a function for each hand type that returns the appropriate tuple does the job, and provides a clean API, but it's a bit  repetitive:

``` python
def HighCard(cards):
    return Hand.HighCard, cards

def PairOf(rank, cards):
    return Hand.PairOf, rank, cards

def TwoPair(hiRank, loRank, cards):
    return Hand.TwoPair, hiRank, loRank, cards

def ThreeOf(rank, cards):
    return Hand.ThreeOf, rank, cards

def Straight(cards):
    return Hand.Straight, cards

def Flush(cards):
    return Hand.Flush, cards

def FullHouse(overRank, pairRank, cards):
    return Hand.FullHouse, overRank, pairRank, cards

def FourOf(rank, cards):
    return Hand.FourOf, rank, cards

def StraightFlush(cards):
    return Hand.StraightFlush, cards
```

You could drop the `Hand` enum and manage the values yourself, but that still leaves this version twice as long as the Haskell version.

# Conclusion

As I stated at the outset, the statically typed code has less boilerplate. But it's really a minimal difference. Even in the worst case, with no actual code beyond the type declaration and creation, the dynamically typed code is only about twice as long as the statically typed code. In this case, exposing some of the implementation details to the user could make them shorter in both languages, and much closer to the same length. Adding real code - for instance, to categorize a hand as the proper type then invoke the appropriate constructor - will make that less significant. 

The real difference was that - when I got the union type - the statically typed language still had an one obvious way to do things, that was succinct and provided an API I was happy with. With a dynamically typed language, there is no union type, since they don't make sense. After all, your variable can hold all the types, you just have to write the code that deals with the differences between them. But - well, you have to write that code. Which means figuring out the best way to deal with it. You also have to decide what the API is going to look like, and the examine the tradeoffs in it from using a simpler implementation. That took longer than writing all the rest of the code put together.

So, given that my statically typed language has slightly less typing and provides - in this case - an obvious way to do something that presents a problem in the dynamic language, it would seem that dynamic languages no longer have an advantage to offset having to wait for tests to run before you catch type errors. So they will eventually - and given the speed at which this industry adopts new languages, I probably won't live to see the day - dissappear. Or maybe both will disappear before then.


# Working example

This is the code I used for the solution to the project euler problem, modified to allow for interactive comparisons. A card is a single digit through 9, then the first letter of the rank (`23456789TJQKA`) followed by the first letter of the suit ('SDHC'). A hand is five cards. You enter two hands on a line and hit enter, then the code will figure out the hand types and print them back labelling the winner and loser.

``` active haskell
-- show
-- /show
import Data.Char (isSpace)
import Data.List (sortBy, group, delete)
import Control.Monad (forever)
-- show
data Suit = Spades | Hearts | Diamonds | Clubs deriving (Show, Eq)
-- /show
instance Read Suit where
  readsPrec _ [] = []
  readsPrec p (c:cs) | isSpace c = readsPrec p cs
                     | c == 'S' = [(Spades, cs)]
                     | c == 'H' = [(Hearts, cs)]
                     | c == 'D' = [(Diamonds, cs)]
                     | c == 'C' = [(Clubs, cs)]
                     | otherwise = []
-- show

data Rank = Two | Three | Four | Five | Six | Seven | Eight | Nine | Ten
     | Jack | Queen | King | Ace
     deriving (Show, Eq, Ord, Enum)
-- /show
instance Read Rank where
  readsPrec _ [] = []
  readsPrec p (c:cs) | isSpace c = readsPrec p cs
                     | c == 'A' = [(Ace, cs)]
                     | c == '2' = [(Two, cs)]
                     | c == '3' = [(Three, cs)]
                     | c == '4' = [(Four, cs)]
                     | c == '5' = [(Five, cs)]
                     | c == '6' = [(Six, cs)]
                     | c == '7' = [(Seven, cs)]
                     | c == '8' = [(Eight, cs)]
                     | c == '9' = [(Nine, cs)]
                     | c == 'T' = [(Ten, cs)]
                     | c == 'J' = [(Jack, cs)]
                     | c == 'Q' = [(Queen, cs)]
                     | c == 'K' = [(King, cs)]
                     | otherwise = []
-- show

data Card = Card Rank Suit deriving Show

-- Eq & Ord can't be derived, since poker ignores suits in hand rankings.
instance Eq Card where
  Card r1 _ == Card r2 _ = r1 == r2

instance Ord Card where
  Card r1 _ <= Card r2 _ = r1 <= r2
-- /show
instance Read Card where
  readsPrec p cs = do
    (r, cs1) <- readsPrec p cs
    (s, cs2) <- readsPrec p cs1
    return (Card r s, cs2)
-- show

data Hand = HighCard [Card]
          | PairOf Rank [Card]
          | TwoPair Rank Rank [Card]
          | ThreeOf Rank [Card]
          | Straight [Card]
          | Flush [Card]
          | FullHouse Rank Rank [Card]
          | FourOf Rank [Card]
          | StraightFlush [Card]
            deriving (Show, Eq, Ord)
          -- RoyalFlush isn't listed; it's just an Ace-high StraightFlush
-- /show            
instance Read Hand where
  readsPrec p cs = do
    (c1, cs1) <- readsPrec p cs
    (c2, cs2) <- readsPrec p cs1
    (c3, cs3) <- readsPrec p cs2
    (c4, cs4) <- readsPrec p cs3
    (c5, cs5) <- readsPrec p cs4
    return (makeHand [c1, c2, c3, c4, c5], cs5)

makeHand :: [Card] -> Hand
makeHand cards = 
  let sorted = sortBy (flip compare) cards
      grouped = sortBy (\a b -> compare (length b) (length a)) $ group sorted
      Card highOf _ = head . head $ grouped
      Card nextOf _ = head . head . tail $ grouped
      isStraight cards =
        all (\ (a, b) -> a `follows` b) (zip cards (tail cards)) where
            Card r1 _ `follows` Card r2 _ =
              r2 == if r1 > Two then pred r1 else Ace
  in
   (case length grouped of
       5 | all (\ (Card _ a, Card _ b) -> a == b) (zip cards (tail cards)) 
           -> if isStraight sorted then StraightFlush else Flush
         | isStraight sorted -> Straight
         | otherwise -> HighCard
       4 -> PairOf highOf
       3 -> if (length . head) grouped == 3 then ThreeOf highOf
            else TwoPair highOf nextOf
       2 -> if (length . head) grouped == 4 then FourOf highOf
            else FullHouse highOf nextOf)
     sorted


-- These should be generalized to deal with more than two hands per line.
data Deal = Deal {h1, h2 :: Hand}

instance Read Deal where
  readsPrec p s = do
            (h1, s1) <- readsPrec p s
            (h2, s2) <- readsPrec p s1
            return (Deal h1 h2, s2)


main = forever $ do
  line <- getLine
  let d = read line :: Deal
  let (w, l) = if h1 d > h2 d then (h1 d, h2 d) else (h2 d, h1 d)
  putStrLn $ "Winner: " ++ (show w)
  putStrLn $ "Loser: " ++ (show l)
```

The input code isn't very good - I used this as an excuse to learn how to write `Read` instances - and it does no error checking, so if you screw up you get a `no parse` error and it exits. It was sufficient for the problem, though.

There is a bug that isn't exercised by the project problem. I've left it in for the reader to find as an exercise.