# Imperative OOP Ceremony

Haskell as a functional language has three language barriers to programmers like me with an imperative object-oriented background: syntax, terminology and concepts. Luckily, we neither need to use FP lingo like "currying" nor best-practice Haskell syntax.


``` active haskell
import Prelude hiding ((.))
x.f = f(x)

main = do {
  print("Hello World!");
  "Hello World!".print;
}
```

We have tweaked Haskell's preloaded module `Prelude`, which comes with a bunch of useful functions such as `print` as well as a `.` operator of its own, which we had to re-implement in line 2 according to our own imperative/object-oriented needs. Click on the play button and see for yourself: `"Hello World!".print` has become a valid expression, too!

Since functions without parenthesis look awkward, let's define an additional `apply` function that makes our code look even more object-oriented.

``` active haskell
import Prelude hiding ((.))
x.f = f(x)
f `apply` x = f(x)

main = do {
  print("Hello World!");
  "Hello World!".apply(print);
}
```

Since `x.apply(f)` is not "native" Haskell, we can't use it in a function declaration. Instead, we had to use ``f `apply` x``, which is the same. In the next tutorial we will see why.