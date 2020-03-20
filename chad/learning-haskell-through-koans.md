# Learning Haskell Through Koans

In the few years since Ruby Koans first came out, the approach has been mimicked in a wide variety of programming languages. Work on [Haskell Koans](https://github.com/roman/HaskellKoans) was started in January 2012 by Román González and Tatsuhiro Ujihisa.

The premise is simple: A _koan_ is a small snippet of almost-correct code, given for "meditation". Each koan is a kind of puzzle, and is a great way for users to learn more about a language.

Here's a simple example:

```active haskell
import Test.HUnit
check p = do
  assert p
  putStrLn "OK"
--show
result = fixMe

main = check (2 + 2 == result)
```

Running this as-is gives a compile error, since `fixMe` is undefined. But changing the code by replacing `fixMe` with `4` gives a reassuring `OK`.

A lot of programmers are "hands-on" learners, and would rather just try out a new tool and explore some possibilities, rather than starting with a thorough review of documentation or associated research papers. 

Building koans on School of Haskell is easy. Here's the markdown behind the above example:

    ```active haskell
    import Test.HUnit
    check p = do
      assert p
      putStrLn "OK"
    --show
    result = fixMe

    main = check (2 + 2 == result)
    ```
The code before `--show` is hidden, and has two parts:

1. First we `import Test.HUnit`, a Haskell unit-testing framework.
2. Next we define `check`, a thin wrapper around HUnit's `assert` command.

So in general, each koan can look like this:

    ```active haskell
    import Test.HUnit
    check p = do
      assert p
      putStrLn "OK"
    --show
    YOUR KOAN HERE
    ```

That's really all there is to it. Koans are a great fit with our Active Haskell, and we'd especially love to see how this approach can be used to introduce users to new libraries. Let us know what you think! 

Want to discuss this? Check out our [forums](http://forums.fpcomplete.com/post/Koans-6259430)!