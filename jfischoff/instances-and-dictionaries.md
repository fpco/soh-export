# Instances and Dictionaries

### An Idealized Operational View of Type Class Dictionary Passing

When you make a class

```haskell
class Foo a where
    quux :: a -> Int
    muux :: a -> Bool
```

and then an instance. 

```haskell 
data Bar = Bar 

instance Foo Bar where
    quux = \Bar -> 1
    muux = \Bar -> True
```

There is a desugaring processes that occurs, removing type classes. 

For each class there is an associated record,

```haskell 
data FooD a = FooD          -- class Foo a where
    { quuxD :: a -> Int     --    quux :: a -> Int
    , muuxD :: a -> Bool    --    muux :: a -> Bool
    }
```

and for each instance there is associated value of that record.

```haskell
barFooInstanceDictionary :: FooD Bar  
barFooInstanceDictionary = FooD      -- instance Foo Bar where
    { quuxD = \Bar -> 1              --     quux = \Bar -> 1
    , muuxD = \Bar -> True           --     muux = \Bar -> True
    }
```

Functions that before had class contexts 

```haskell 
goo :: Foo a => a -> Int
goo a = quux a
```

are converted to functions that take an additional parameter. 

This parameter is called method 'dictionary'.

```haskell 
gooD :: FooD a -> a -> Int -- Foo became FooD and '=>' became '->'
gooD fooDictionary a = quuxD fooDictionary a -- the context is now a parameter!
```

The instance method `quux` has become a function `quuxD`, a record selector for `FooD`. 

`FooD` is a record of method implemenations for the `Foo` type class.

So if we write

```haskell
test :: Int
test = quux Bar
```

This will get desugared

```haskell
test :: Int
test = quuxD ? Bar
```

Somehow we need `barFooInstanceDictionary` to appear where the `?` is. 

We rely on type inference to determine that `quuxD` must have the type 

```haskell
quuxD :: FooD Bar -> Bar -> Int
```

and rely on the instance selection to persuade the compiler to pass `barFooInstanceDictionary` to `quuxD`:


```haskell
test :: Int
test = quuxD barFooInstanceDictionary Bar
```

This leads to an operational view of type classes:

> Type classes are a way to pass instance dictionaries implicitly. 

### A Simplification

In the typical case, the dictionaries are records of functions.

If we reduce slightly we get

```haskell
test :: Int
test = (\Bar -> 1) Bar
```

So in the end, our type class desugaring moved our method implementation in front of the argument so it could be applied.

Which is where the statement

<blockquote class="twitter-tweet" lang="en"><p>C++ implicitly passes an argument, &#39;this&#39;, to an instance method. <a href="https://twitter.com/search?q=%23Haskell&amp;src=hash">#Haskell</a>, implicitly passes an instance method to an argument.</p>&mdash; Haskell Tips (@HaskellTips) <a href="https://twitter.com/HaskellTips/statuses/429369349871124480">January 31, 2014</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

came from.


