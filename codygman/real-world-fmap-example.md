# Real world fmap example

[Question for context](https://news.ycombinator.com/item?id=9686845)

You start out with:

``` haskell active autorun
data Foo = Foo deriving Show
data Bar = Bar deriving Show
data Baz = Baz deriving Show

x :: Foo
x = Foo

y :: Foo -> Bar
y _ = Bar

z :: Bar -> Baz
z _ = Baz

main = print (z . y $ x)
```


Requirements change and you find out that x can fail, so you have to update it and all of it's callers to thread through the Maybe Monad like so:


``` haskell active autorun
data Foo = Foo deriving Show
data Bar = Bar deriving Show
data Baz = Baz deriving Show

x :: Maybe Foo
x = Just Foo

y :: Maybe Foo -> Maybe Bar
y _ = Just Bar

z :: Maybe Bar -> Maybe Baz
z _ = Just Baz

main = print (z . y $ x)
```

But wait... you don't have to update your functions to "thread through" the maybe monad! See below:

``` haskell active autorun
import Control.Applicative (<$>) -- must import explicitly before GHC 7.10
    
data Foo = Foo deriving Show
data Bar = Bar deriving Show
data Baz = Baz deriving Show

x :: Maybe Foo
x = Just Foo

y :: Foo -> Bar
y _ = Bar

z :: Bar -> Baz
z _ = Baz

main = print (z . y <$> x)
```

Let me remind you of the type of fmap:

``` haskell
fmap :: Functor f => (a -> b) -> f a -> f b
```

specialized to the case of Maybe's functor instance and our types:

```
fmap :: (Foo -> Bar) -> Maybe Foo -> Maybe Bar
```

fmap (aka <$>) eliminated the problem having to update all of those functions whereas in a less composable language updating all callers would have been unavoidable.

For more information checkout [Functors, applicatives, and monads in pictures](http://adit.io/posts/2013-04-17-functors,_applicatives,_and_monads_in_pictures.html). It was the first tutorial that made these concepts click for me. It didn't happen immediately, but after many cycles of experimenting, re-reading that tutorial, and reflection I finally understood these concepts.