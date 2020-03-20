# I/O in Haskell for Java programmers

# Hello, world!

Let's go right ahead and get started with the classic "Hello, world!" program in Java:

## In Java

``` java
public static void main(String[] args) {
  System.out.println("Hello, world!");
}
```

## In Haskell

The code has very much the same shape in Haskell, but with slightly different notation, types, and prodecure naming and behavior:

``` haskell
module Main where {
  main :: IO ()
  main = do {
    putStrLn "Hello, world!";
  }
}
```

Haskell can infer the type signature of `main`, and can also insert the default `module` declaration for us, so for future examples I will omit these.

``` active haskell
main = do {
  putStrLn "Hello, world!";
}
```

## Notation and types

If you've seen Haskell before, you may be scratching your head. Haskell has whitespace-sensitive syntax, but I've chosen to use Haskell's optional curly braces and semicolons for familiarity to Java programmers.

You'll notice, however, that function application looks different. While in Java, you write `foo(bar, baz, qux)`, in Haskell you merely write `foo bar baz qux`. You can think of it as shifting the original paren to the left, and omitting the commas: `(foo bar baz qux)`. While I *could* have written `putStrLn("Hello, world!");`, this would be misleading because the comma-separated notation does not work for Haskell functions with multiple arguments. On this I will not compromise and give a Java-esque presentation, since using pre-defined Haskell functions with multiple arguments will be necessary.

In Haskell, the type signature is written separately from the definition. Types are written `identifier :: Type`, which reads "`identifier` has type `Type`". The `()` type is like Haskell's version of `void`. The Haskell type signature of `main` also explicitly notes that `IO` effects can happen during `main`. You don't need to worry much about this for now, though we would certainly hope that IO effects can happen in `main`, given that this is the whole point of the tutorial!

You may notice that in Haskell, we write `main = do { ... }` while in Java we simply write `main(...) { ... }`, without the `= do` part. That's just some magic notation that I want you to ignore for right now.

## The main method

There are a few differences between `main` in Haskell and `main` in Java.

### Modules vs classes

In Java, the `main` method is a public, static member of the enclosing class. This means that you do not need to create an object of that type in order to invoke the `main` method. If you "run" a Java file, then by default, that file's outermost class's `main` function is invoked.

In Haskell, the `main` method must be *exported* by a *module*. When you "run" a Haskell file, it invokes the files' outermost module's `main` function. In this regard, modules and classes fulfill a similar role, but they are different in essentially every other way imaginable. I won't discuss the details here; we have access to a runnable `main` method and that's all we really need to know about classes or modules for this tutorial.

### Program arguments

You'll notice that the Java version of `main` includes an array of Strings, typically named `args`, which are the textual arguments that the program is invoked with. Haskell's version of `main` does *not* mention `args` anywhere, but you can gain access to the program arguments by using `getArgs`.

``` haskell
import System.Environment (getArgs)

main = do {
  args <- getArgs;
  print args;
}
```

I'll explain imports and the left-facing arrow (`<-`) soon, though you probably already have an intuition for how they work.

## Printing a String

In Haskell, you can use `putStrLn` in place of `System.out.println`. The only catch is that in Haskell, there is no implicit invocation of `toString` as there is in Java. The Haskell equivalent of `toString` is `show`. (Actually, `show` is more like Python's `repr`, because it produces a string which, when read, can typically be parsed back into the data it represents.)

``` active haskell
main = do {
  putStrLn (show 3);
}
```

String concatenation in Java is done with `+`, which again implicitly calls `toString` on things. In Haskell, string concatenation is done with `++`, and you must perform explicit string conversions.

``` active haskell
main = do {
  putStrLn ("I have " ++ show 3 ++ " apples");
}
```

## Now you try

Try out Haskell's string printing, string concatenation, and coersion to strings. Just don't try to use variables yet; I'll teach you those soon.

``` active haskell
main = do {
  putStrLn ("edit me!");
}
```

# Echoing and looping

Now let's move on to something a little more interesting. Again, we'll start with a Java example and transliterate to Haskell.

## In Java

``` java
import java.util.Scanner
class Main {

    public static void main(String[] args) {
        Scanner scan = new Scanner(System.in);
        System.out.println("Please enter a line of text");
        String line = scan.nextLine();
        while (!line.equals("end")) {
            System.out.println(line);
            line = scan.nextLine();
            System.out.println("Please enter a line of text");
        }
        System.out.println("The end.");
    }
}
```

This simple Java program scans standard input one line at a time, and echoes the line back to standard output.

## Reading input in Haskell

Ok, let's start simple. We'll just read in one line and spit it back out.

``` active haskell
main = do {
  putStrLn "Please enter a line of text";
  line <- getLine;
  putStrLn line;
}
```

Here I used `getLine` and the left-facing arrow (`<-`). `getLine` eats up a line from standard input, and the left-facing arrow binds the identifier `line` to the result of running `getLine`. If you click on `getLine` in the code above, you'll see that it has type `IO String`, meaning that it is an `IO` action that produces a `String`. In Haskell, you assign the result of *actions* using the left-facing arrow.

### Aside

You can assign the result of a *pure computation* using `let`.

``` active haskell
main = do {
  putStrLn "Please enter your name";
  name <- getLine;
  let {greeting = "Hello, " ++ name ++ "!"};
  putStrLn greeting;
}
```

## Looping in Haskell

``` active haskell
import Control.Monad (unless)

main = do {
  putStrLn "Please enter a line of text";
  line <- getLine;
  if (line /= "end")
  then do {
      putStrLn line;
      main;
  }
  else do {
      putStrLn "The end.";
  }
}
```

In Haskell, you can "loop" via primitive recursion. In this example, I just invoked `main` again. You might worry about blowing the call stack by doing this. Don't. Haskell's "stack" is very different from Java's. This has to do with lazy evaluation, among other things. It is beyond the scope of this tutorial to explain why this Haskell code is efficient, so just trust me for now.

Work in progress! Suggestions welcome: danburton.email AT gmail

