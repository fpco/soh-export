# Imperative OOP Ceremony: do-Notation Pt. 2

## An Example: IO Actions

In the last tutorial we learned about Haskell's do-notation of Method Cascades that use a Programmable Fluent Interface, a.k.a. *Monads* in FP lingo.

As we can guess, since we defined the `main` function as a do-notation, `print` returns a monadic object. As we can observe, the evaluation of this monadic object yields an output to the console.

``` active haskell
main = do {
  print("Hello World!");
}
```

The type of the monadic object is `IO`, as it models IO actions like printing something to the screen.

As long as we're not binding another method to `print("Hello World!")` as in this simple example, we can skip the do-notation.

``` active haskell
main = print("Hello World!")
```

It is important to understand that `"Hello World!"` as a `String` and the `IO` object `print("Hello World")` (whose evaluation has the side-effect of `"Hello World!"` appearing on the screen) are two very different things. In order to make this difference more apparent it would be great to be able to access the inner value of the `IO` object, but `IO` objects are very shy; they have private constructors and accessors. But since they are monadic, we can bind another `print` to `print("Hello World!")` and print the inner value of `print("Hello World!")`. Instead of our own `bind` method (i.e. `print("Hello World!").bind(\(a) -> print(a))`) we shall use directly Haskell's bind operator `>>=`.

``` active haskell
main = print("Hello World!") >>= \(a) -> print(a)
```

As we can see, the actual inner value of the `IO` object `print("Hello World!")` isn't `"Hello World!"`, but "void", i.e. the empty tuple `()` (FP lingo: *Unit*). (The empty tuple `()` represents an output to something stateful like the real world.)

Let's bind `print` to an `IO` object that contains something more substantial. `getLine` provides us with an `IO` object that contains the `String` of an input from the command line. (Don't forget to press Enter.)

``` active haskell
main = getLine >>= \(a) -> print(a)
```

As we can see, `getLine` contains the `String` we have entered on the command line.

Using `IO`'s exported `return` factory function we can create our own `IO` objects and bind `print` to them. This is admittedly a very convoluted way of using the `print` function.

``` active haskell
main = return("Hello World!") >>= \(a) -> print(a)
```

## Return

As we can see in the previous example, `return` doesn't terminate the monadic expression, as it may make sense to bind another function to the monadic object that `return` constructs. (Remember that `return` is factory method.)

``` active haskell
main = do {
  a <- return("Hello World");
  print(a);
}
```

## >>=

Using the bind operator `>>=` may result in even terser code, especially when writing in pointfree style, e.g. when binding `print` without wrapping it up in `\(a) -> print(a)`.

``` active haskell
main = getLine >>= print
```

Cf. do-notation, where we must provide a bound variable.

``` active haskell
main = do {
  a <- getLine;
  print(a);
}
```

## >>

We may use a wildcard instead of `a` if we opt for not using it anyway.

``` active haskell
main = do {
  _ <- getLine;
  print("tldr;");
}
```

In this case, do-notation allows for ommiting `_ <-` alltogether.

``` active haskell
main = do {
  getLine;
  print("tldr;");
}
```

In order not to write `>>= \(_) ->` in normal Haskell syntax, the `Monad` interface provides an auxilary operator `>>`.

``` active haskell
main = getLine >> print("tldr;")
```

## {;}

Last but not least, do-notation doesn't require curly braces and trailing semicolons are unnecessary, too.

``` active haskell
main = do
  print("What's your name?")
  name <- getLine
  print("Hello " ++ name ++ "!")
```

Side note: In a more idiomatic Haskell style, we would also remove the parenthesis after `print`. As a substitute, we use `$`, the apply operator.

``` active haskell
main = do
  print "What's your name?"
  name <- getLine
  print $ "Hello " ++ name ++ "!"
```
