# Sum Types

    **Question**: What do all these languages have in common:    

* C
* C++
* C#
* Java
* Javascript
* Objective C
* Perl
* PHP
* Python
* Ruby

**Answer**: These languages all lack user-defined sum types

If you don't know what a sum type is then you are not alone, as many people who never stray from mainstream languages unknowingly miss out on this incredibly useful and fundamental language feature.

# Boolean Types

In Haskell, the simplest sum type is the `Bool` type:

```haskell
data Bool = False | True
```

This type definition says that a `Bool` can be one of two values: either `False` or `True`.  A "sum type" is any type that has multiple possible representations, and we use `|` to separate each representation.  We can use these representations in our functions as if they were ordinary values:

```active haskell
isZero :: Int -> Bool
isZero 0 = True
isZero _ = False

toBeOrNotToBe :: Bool -> String
toBeOrNotToBe True  = "Be"
toBeOrNotToBe False = "Not to be"

main = putStrLn (toBeOrNotToBe True)
```

Notice how `Bool` is not a built in type.  Rather, we define it within the Haskell language as an ordinary unprivileged type right <a href="http://hackage.haskell.org/packages/archive/base/latest/doc/html/Prelude.html#t:Bool">here</a>.  This already departs significantly from mainstream languages, which must either:

* interpret boolean values as integers, or
* add language support for booleans

`Bool` is not an `Int`, which means that we cannot accidentally use a `Bool` where we expect an `Int` or vice versa:

```active haskell
main = print (True + 1)  -- Type error!
```

We gain type safety when the language lets us define our own sum types rather than faking it with existing types like integers or strings.

Also, `Bool` has only two representations: `True` and `False`.  In contrast, if we define a `Bool` in terms of an integer, we typically have multiple representations for `True` (any non-zero number), and we must define and remember the mapping between integers and boolean values.

# Constructors

We can define our own types, so why not define a safe wrapper around division?

```active haskell
data DivisionResult = DivisionByZero | Success Double
    deriving (Show)

safeDivide :: Double -> Double -> DivisionResult
safeDivide x y =
    if y == 0
    then DivisionByZero
    else Success (x / y)

-- Try dividing by zero
main = print (safeDivide 4 2)
```

This time we add a small twist to our sum type!  If the division succeeds, we return the successful value.  If the division fails, we return no value.

Each alternative in a sum type is a constructor, and these constructors can hold values.  In the above example, `DivisionByZero` and `Success` are the two alternative constructors:

* `DivisionByZero` is an empty constructor that holds no values
* `Success` is a non-empty constructor that holds one value of type `Double`

We only return a value if we succeed, so we guarantee that anybody that consumes the result of our function cannot access an incorrect or meaningless value, either accidentally or intentionally, if our function fails.

# Pattern Matching

We use sum types by "pattern matching" on them.  For example, we might want to convert the result of safe division to a `String`:

```active haskell
data DivisionResult = DivisionByZero | Success Double
    deriving (Show)

safeDivide :: Double -> Double -> DivisionResult
safeDivide x y =
    if y == 0
    then DivisionByZero
    else Success (x / y)

-- show
divisionToString :: DivisionResult -> String
divisionToString r = case r of
    DivisionByZero -> "Division failed"
    Success x      -> show x

main = putStrLn $ divisionToString $ safeDivide 4 2
-- /show
```

The `case` syntax is the fundamental way to pattern match.  You provide it a value to inspect, and you define a result for each constructor that you might receive.  In the above example, we told `case` to inspect the `r` value, which had type `DivisionResult`.  This means that `r` could either be a `DivisionByZero` or a `Success` constructor with a value bound inside.

```haskell
divisionToString r = case r of
    {-hi-}DivisionByZero{-/hi-} -> "Division failed"
    {-hi-}Success x{-/hi-}      -> show x
```

`case` lets you match against non-empty constructors like `Success` and extract the value stored inside.  This ensures that the only way we can access the value is if division succeeded.  This protects against run-time failures where we try to access a non-existent result if division were to fail.  This differs from other languages which will provide a result no matter what, but then also supply an additional flag signifying whether or not that result is valid.

```active haskell
wrongDivide :: Double -> Double -> (Bool, Double)
wrongDivide x y = (y == 0, x / y)

-- Oops, we we were lazy and didn't check the boolean flag:
main = let (_, r) = wrongDivide 2 0 in print r
```

Also, pattern matching forces us to handle all possible alternatives.  We can only use a value of type `DivisionResult` if we handle both the success and failure scenarios.  Contrast this with some programming languages which use sentinel values of the same type as the result to signal failure, which can easily be ignored by the programmer or are indistinguishable from legitimate results:

```active haskell
wrongDivide :: Double -> Double -> Double
wrongDivide x y =
    if (y == 0)
    then 0
    else x / y

-- Does 0 mean it failed or computed 0?
main = print (wrongDivide 0 5)
```

More sophisticated mainstream languages will signal errors out of band using exceptions, but these are implemented as a language feature, rather than as an ordinary type.  Notice a common pattern: mainstream languages require language extensions to implement features that ordinary sum types would have provided.

# Syntactic Sugar

Haskell offers syntactic sugar for pattern matching where you can instead write multiple function definitions, one for each constructor alternative:

```active haskell
data DivisionResult = DivisionByZero | Success Double
    deriving (Show)

safeDivide :: Double -> Double -> DivisionResult
safeDivide x y =
    if y == 0
    then DivisionByZero
    else Success (x / y)

-- show
-- Old version
divisionToString :: DivisionResult -> String
divisionToString r = case r of
    DivisionByZero -> "Division failed"
    Success x      -> show x

-- New version
divisionToString2 :: DivisionResult -> String
divisionToString2 DivisionByZero = "Division failed"
divisionToString2 (Success x)    = show x

main = putStrLn $ divisionToString2 $ safeDivide 4 2
-- /show
```

The compiler desugars new version into a form identical to the old version.

We can use this syntactic sugar to implement an elegant version of the `not` function:

```active haskell
import Prelude hiding (not)

-- show
not :: Bool -> Bool
not False = True
not True  = False

main = print (not True)
-- /show
```

# Either

Haskell defines the `Either` type, which is also a sum type:

```active haskell
data Either a b = Left a | Right b
```

The `Either` type is the canonical sum type from which we can build all other sum types.  In fact, computer scientists often use `(+)` to refer to `Either`.

We call `Either` THE sum type because if the `a` type is inhabited by `M` possible values, and the `b` type is inhabited by `N` possible values, then `Either a b` is inhabited by `M + N` possible values.

Let's use a simple example to convince ourselves of this.  If the `Bool` type is inhabited by `2` values, and the `()` type is inhabited by `1` value, then the `Either Bool ()` type must be inhabited by `2 + 1 = 3` values:

```haskell
val1 :: Either Bool ()
val1 = Left False

val2 :: Either Bool ()
val2 = Left True

val3 :: Either Bool ()
val3 = Right ()
```

There are exactly three unique values that have type `Either Bool ()`: two values of type `Bool` wrapped within a `Left` constructor and one value of type `()` wrapped within a `Right` constructor, for a total of three values.

We can simulate any type with multiple constructors using `Either`.  For example, we could simulate `Bool` using `Either () ()`:

```haskell
type Bool = Either () ()

false :: Bool
false = Left ()

true :: Bool
true = Right ()
```

Note that the `()` value is inhabited by exactly one value: `()`, therefore `Either () ()` must have `1 + 1 = 2` possible values, just like `Bool`.

If we had a type with more than two constructors:

```haskell
data Trool = False | True | DontKnow
```

... we can still simulate it using `Either`:

```haskell
type Trool = (Either () (Either () ()))
-- i.e. Trool = 1 + (1 + 1) = 3

false :: Trool
false = Left ()

true :: Trool
true = Right (Left ())

dontKnow :: Trool
dontKnow = Right (Right ())
```

# Product Types

If we have sum types, then perhaps we also have product types, too.  A product type is just a tuple, or a constructor with more than one argument:

```haskell
-- A product of an Integer and  String
(4, "Hello") :: (Integer, String)

-- A data type that is a product of a Char, an Integer, and Bool
data Multiple = M Char Integer Bool
```

**Exercise:** Why do you think they are called product types?
<hidden>
**Answer**: A product type is inhabited by the product of the number of inhabitants of each field.
</hidden>

**Exercise:** What do you think the "canonical" product is?
<hidden>
**Answer**: The binary tuple: `(a, b)`.  If the `a` type is inhabited by `M` values and the `b` type is inhabited by `N` values, then `(a, b)` is inhabited by `M * N` values.
</hidden>

Almost every mainstream language has some sort of product type:

C:

    // A product of a double and a double
    struct point {
        double x;
        double y;
    };

Python:

    # float x
    # float y
    # A product of a float and a float
    (x, y)

Java:

    // The product of a double and a double
    class Point {
        double x;
        double y;
    }

In other words, mainstream languages are rich in product types, yet conspicuously deficient in sum types.  Notice that product types lack the ability to:

* create new primitive data types that are not numbers, bools, or strings
* represent a choice between two alternatives

# Conclusion

Many language features that you would normally expect a language author to implement are easily written using sum types.  In fact, languages that provide sum types tend to have smaller and more orthogonal feature sets since they provide users much greater flexibility to implement their own features.

Once you understand the usefulness of sum types, you will wonder why mainstream languages go to remarkable lengths in order to avoid implementing them.  Sum types are arguably the most striking feature missing from most languages.