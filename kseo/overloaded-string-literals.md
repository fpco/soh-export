# Overloaded string literals

In Haskell, the type of a string literal `"Hello World"` is always `String`
which is defined as `[Char]` though there are other textual data types such as
`ByteString` and `Text`. To put it another way, string literals are monomorphic.

GHC provides a language extension called [OverloadedStrings][OverloadedStrings].
When enabled, literal strings have the type `IsString a => a`. `IsString` moudle
is defined in `Data.String` module of base package:

```haskell
class IsString a where
    fromString :: String -> a
```

`ByteString` and `Text` are examples of `IsString` instances, so you can declare
the type of string literals as `ByteString` or `Text` when `OverloadedStrings`
is enabled.

```haskell
{-# LANGUAGE OverloadedStrings #-}
a :: Text
a = "Hello World"

b :: ByteString
b = "Hello World"
```

Of course, `String` is also an instance of `IsString`. So you can declare the
type of a string literal as `String` as usual.

```haskell
c :: String
c = "Hello World"
```

[OverloadedStrings]: https://downloads.haskell.org/~ghc/7.0.4/docs/html/users_guide/type-class-extensions.html

