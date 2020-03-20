# Data Type Extensions

These extensions enhance the capabilites of Haskell’s algebraic data types.

# `DatatypeContexts`

**Available in:  GHC 7.0 and later**

The deprecated `DatatypeContexts` extension is officially part of the Haskell 98 and Haskell 2010 language standards, but is going to be removed in the next standard.  It allows you to constrain any and/or all type parameters in a `data` or `newtype` declaration, like so:

``` haskell
data (Eq v) => OneOrThreeRequiringEq v = ORE v | TRE v v v

newtype (Show a, Ord a) => RequiresShowAndOrdTagged a b = RSAOTagged b
```

Wherever a contextually-constrained data type appears, the relevant constraints are required to be present.  Such a data type does *not* carry around said constraints, but *requires them to be present (even if unused) at both construction and deconstruction* instead.  This behavior is generally undesirable, so data type contexts are widely considered a misfeature.

**Warning:**  *Do not use `DatatypeContexts` in new code!*  I only include it here to aid in understanding legacy Haskell codebases.  If you think you might want `DatatypeContexts`, try [`ExistentialQuantification`](https://www.schoolofhaskell.com/user/PthariensFlame/guide-to-ghc-extensions/data-type-extensions#existentialquantification-as-a-replacement-for-datatypecontexts) instead.

# `EmptyDataDecls`

**Available in:  All recent GHC versions**

The `EmptyDataDecls` extension is very simple:  it allows you to define data types with no constructors.  You *must* omit the `=` character when defining an empty data type.

The main use case for such a type is as a phantom parameter to some other type, but there are occasionally other uses of `EmptyDataDecls` as well.

Try it out!

``` active haskell
{-# LANGUAGE EmptyDataDecls #-}

data Empty

data EmptyWithPhantom x

type EmptyWithEmpty = EmptyWithPhantom Empty

main = putStrLn "No errors."
```

# `ExistentialQuantification`

**Available in:  All recent GHC versions**

The `ExistentialQuantification` extension deals with monomorphic data types that might contain multiple distinct types of contained values.  Existential types are especially useful for wrapping up similar things with distinct types in a container of monomorphic type.

For example, consider the following type class and instances:

``` haskell
class StringLike s where
    toString :: s -> String
    fromString :: String -> s

instance StringLike String where
    toString = id
    fromString = id

instance StringLike Text where
    toString = unpack
    fromString = pack
```

Suppose we want to collect a list of `StringLike` types, but they might be of different actual type.  We know ahead of time that we will only ever be using the operations of `StringLike` on the elements of the list.  We can use `ExistentialQuantification` to construct a monomorphic box *for all `StringLike` types*:

``` haskell
data Stringish = forall a. StringLike a => Stringish a

instance StringLike Stringish where
    toString (Stringish s) = toString s
    fromString = Stringish
```

Now we can make a list of `StringLike`s without the typechecker complaining!

``` haskell
[Stringish (pack "Hello as Text"), Stringish "Hello as String"]
```

Unfortunately, because of the extra power that `ExistentialQuantification` affords you, existentially-quantified declarations cannot have `deriving` clauses and cannot be `newtype`s.

**Warning:**  [*Be very careful with existential types!*](http://lukepalmer.wordpress.com/2010/01/24/haskell-antipattern-existential-typeclass/)  Try to search for other solutions before considering using them.

Try it out!

``` active haskell
{-# LANGUAGE ExistentialQuantification #-}

data AnyShowable = forall a. (Show a) => S a

instance Show AnyShowable where
    showsPrec p (S a) = showParen (p >= 10) $ ("S " ++) . showsPrec 11 a

main = print $ [S (3 :: Int), S (5.2 :: Double), S "Hello, world!",
                S (), S (S (S [S True, S EQ, S GT]))]
```

## [`ExistentialQuantification`](https://www.schoolofhaskell.com/user/PthariensFlame/guide-to-ghc-extensions/data-type-extensions#existentialquantification) as a Replacement for [`DatatypeContexts`](https://www.schoolofhaskell.com/user/PthariensFlame/guide-to-ghc-extensions/data-type-extensions#datatypecontexts)

You can use `ExistentialQuantification` to replace the deprecated extension `DatatypeContexts` by simply declaring the context on your constructor(s) rather than on your data type itself.  This keeps the same behavior for construction but switches deconstruction from *requiring* the declared context to *providing* the declared context, which is usually closer to what you want in such situations.

For example, this:

``` haskell
newtype (Eq x) => Equatable x = Eqt x
```

becomes this:

``` haskell
data Equatable x = (Eq x) => Eqt x
```

Notice that any `newtype`s must be transformed into `data`s, because `ExistentialQuantification` does not work on `newtype`s.

# `GADTSyntax`

**Available in:  GHC 7.2 and later**

The `GADTSyntax` extension enables a new syntax for `data` declarations.  For example, this declaration:

``` haskell
data OneOrBoth a b = OneL a | Both a b | OneR b
```

can also, and equivalently, be written as:


``` haskell
data OneOrBoth a b where
    OneL :: x      -> OneOrBoth x y
    Both :: v -> w -> OneOrBoth v w
    OneR ::      r -> OneOrBoth q r
```
As demonstrated above, there is no requirement that the type variables in the part of the declaration between `data` and `where` be related to the type variables after `where`, nor is there any requirement that the type variables used in any one constructor signature be related to the type variables used in any other constructor signature.

If the declaration has a `deriving` clause, it is put after the last constructor signature, as usual.

Record syntax also works with `GADTSyntax`, but it looks slightly different.  For example, this declaration:

``` haskell
data Vec2 n = Vec2 { xCoord :: n, yCoord :: n }
```

can also, and equivalently, be written as:

``` haskell
data Vec2 n where
    Vec2 :: { xCoord :: n, yCoord :: n} -> Vec2 n
```

Translating an existing `data` declaration, with record components or not, to the so-called “GADT-style syntax” that `GADTSyntax` introduces does not change the meaning of your data type declaration.

**NOTE:**  There is a way to avoid using superfluous type variables between `data` and `where`, but it requires another extension in tandem with `GADTSyntax` (namely, [`KindSignatures`]()).

There is one limitation on GADT-style syntax as enabled by `GADTSyntax`, however:  each constructor’s signature *must* end with the data type that you are declaring, as parametrized (if applicable) entirely by type variables, all of which much be distinct and unconstrained.  This requirement ensures that anything that you write in GADT-style syntax can be equivalently written in ordinary `data` syntax, and vice versa.

Try it out!

``` active haskell
{-# LANGUAGE GADTSyntax #-}

data MyData x y z where
    MyData    ::   x -> [y] -> (Maybe (z, z))   -> MyData x y z
    EmptyData :: {    isJiggeredUp :: Bool    } -> MyData u v w
  deriving (Show)

main = print $ [MyData "Hi!" [1 :: Integer, 3, 2] (Just (2 :: Int, 15)),
                EmptyData { isJiggeredUp = False }]
```

# `GADTs`

**Available in:  All recent GHC versions**

The `GADTs` extension (short for ***G**eneralized **A**lgebraic **D**ata **T**ype**s***) implicitly enables `GADTSyntax`, and then almost completely removes the single limitation discussed above.  When `GADTs` is enabled, the only requirement for a GADT-style `data` declaration is that each constructor signature (eventually) returns some well-kinded parametrization of the data type that you are declaring.

With `GADTs`, GADT-style syntax is no longer restricted to expressing ordinary data types:

- you can express data types that constrain some or all of their constructors’ type variables, possibly in different ways.
- you can express data types that “hide” a type variable, not letting it show up as a parameter.
- you can express data types that produce the type you’re declaring only at certain concrete or partially concrete parameter types.

The former two of these three features are actually instances of [`ExistentialQuantification`](https://www.schoolofhaskell.com/user/PthariensFlame/guide-to-ghc-extensions/data-type-extensions#existentialquantification), but you cannot use that extension’s syntax unless it is also enabled.  `GADTs` allows you to, in effect, use existential types without directly declaring them.

For example, this declaration:

``` haskell
data MyData x where
    MyData1 :: (Show a) =>        a   -> MyData a
    MyData2 :: (Read b) => String     -> MyData b
    MyData3 :: (Eq   v) => v  ->  [w] -> MyData w
```

is equivalent (with `ExistentialQuantification` enabled) to:

``` haskell
data MyData x =           (Show x) => MyData1        x
              |           (Read x) => MyData2 String
              | forall v. (Eq   v) => MyData3 v      [x]
```

Unfortunately, because of the extra power that `GADTs` affords you, GADT-style declarations that take advantage of this extra power cannot have `deriving` clauses, nor can they be `newtype`s.

Try it out!

``` active haskell
{-# LANGUAGE GADTs #-}

data AST ty where
    LitInt    :: Int                   -> AST Int
    LitString :: String                -> AST String
    OpApp     :: AST (a -> b) -> AST a -> AST b
    PlusOp    ::                          AST (Int -> Int -> Int)
    NegOp     ::                          AST (Int -> Int)
    LengthOp  ::                          AST (String -> Int)

evaluate                 :: AST ty -> ty
evaluate (LitInt    i  ) =  i
evaluate (LitString s  ) =  s
evaluate (OpApp     f v) =  evaluate f $ evaluate v
evaluate  PlusOp         =  (+)
evaluate  NegOp          =  negate
evaluate  LengthOp       =  length

main = print . evaluate $
 OpApp (
  OpApp PlusOp (
   OpApp NegOp (
    LitInt 17
   )
  )
 ) (
  OpApp LengthOp (
   LitString "Hello, world!"
  )
 )
```