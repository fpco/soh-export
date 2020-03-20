# Haskell Fast & Hard (Part 1)

## Don't be afraid

Many book/articles about Haskell start by introducing some esoteric formula (quick sort, Fibonacci, etc...).
I will do the exact opposite.
At first I won't show you any Haskell super power.
I will start with similarities between Haskell and other programming languages.
Let's jump to the mandatory "Hello World".

``` active haskell
main = putStrLn "Hello World!"
```

Now, a program asking your name and replying "Hello" using the name you entered:

``` active haskell
main = do
    print "What is your name?"
    name <- getLine
    print ("Hello " ++ name ++ "!")
```

First, let us compare with a similar program in some imperative languages:

```
# Python
print "What is your name?"
name = raw_input()
print "Hello %s!" % name
```

```
# Ruby
puts "What is your name?"
name = gets.chomp
puts "Hello #{name}!"
```

```
// In C
#include <stdio.h>
int main (int argc, char **argv) {
    char name[666]; // <- An Evil Number!
    // What if my name is more than 665 character long?
    printf("What is your name?\n"); 
    scanf("%s", name);
    printf("Hello %s!\n", name);
    return 0;
}
```

The structure is the same, but there are some syntax differences.
A major part of this tutorial will be dedicated to explaining why.

In Haskell, there is a `main` function and every object has a type.
The type of `main` is `IO ()`.
This means, `main` will cause side effects.

Just remember that Haskell can look a lot like mainstream imperative languages.

#### Exercise

Modify the following code in order to ask for the name and the city and replying nicely.

``` active haskell
main = do
    putStrLn "What is your name?"
    name <- getLine
    putStrLn $ name ++ "! This is a very nice name."
```

@@@ Solution

``` active haskell
main = do
    putStrLn "What is your name?"
    name <- getLine
    putStrLn $ name ++ "! This is a very nice name."
    putStrLn "Where do you live?"
    city <- getLine
    putStrLn $ "Hey, I Love " ++ city ++ "!"
```

@@@

### Very basic Haskell

Before continuing you need to be warned about some essential properties of Haskell.

_Functional_

Haskell is a functional language.
If you have an imperative language background, you'll have to learn a lot of new things.
Hopefully many of these new concepts will help you to program even in imperative languages.

_Smart Static Typing_

Instead of being in your way like in `C`, `C++` or `Java`, the type system is here to help you.

_Purity_

Generally your functions won't modify anything in the outside world.
This means, it can't modify the value of a variable, can't get user input, can't write on the screen, can't launch a missile.
On the other hand, parallelism will be very easy to achieve.
Haskell makes it clear where effects occur and where you are pure.
Also, it will be far easier to reason about your program.
Most bugs will be prevented in the pure parts of your program.

Furthermore pure functions follow a fundamental law in Haskell:

 > Applying a function with the same parameters always returns the same value.

_Laziness_

Laziness by default is a very uncommon language design.
By default, Haskell evaluates something only when it is needed.
In consequence, it provides a very elegant way to manipulate infinite structures for example.

A last warning on how you should read Haskell code.
For me, it is like reading scientific papers.
Some parts are very clear, but when you see a formula, just focus and read slower.
Also, while learning Haskell, it _really_ doesn't matter much if you don't understand syntax details.
If you meet a `>>=`, `<$>`, `<-` or any other weird symbol, just ignore them and follows the flow of the code.

<h4 id="function-declaration">Function declaration</h4>

You might be used to declare functions like this:

In `C`:

```
int f(int x, int y) {
    return x*x + y*y;
}
```

In Javascript:

```
function f(x,y) {
    return x*x + y*y;
}
```

in Python:

```
def f(x,y):
    return x*x + y*y
```

in Ruby:

```
def f(x,y)
    x*x + y*y
end
```

In Scheme:

```
(define (f x y)
    (+ (* x x) (* y y)))
```

Finally, the Haskell way is:

``` haskell
f x y = x*x + y*y
```

Very clean. No parenthesis, no `def`.

Don't forget, Haskell uses functions and types a lot.
It is thus very easy to define them.
The syntax was particularly well thought for these objects.

#### Exercise

Declare correctly the function g(x,y)=x<sup>2</sup>-y<sup>2</sup>+x -y

``` active haskell
-- show Replace undefined by your definition
g = undefined
-- show Should display 6 and -8
main = do 
    print $ g 3 2
    print $ g 3 4
```

@@@ Solution
``` active haskell
-- show
g x y = x*x - y*y + x - y
-- /show
main = do 
    print $ g 3 2
    print $ g 3 4
```

@@@

<h4 id="a-type-example">A Type Example</h4>

The usual way is to declare the type of your function.
This is not mandatory.
The compiler is smart enough to discover it for you.

Let's play a little.

``` active haskell
-- We declare the type using ::
f :: Int -> Int -> Int
f x y = x*x + y*y

main = print (f 2 3)
```

Now try


``` active haskell
f :: Int -> Int -> Int
f x y = x*x + y*y

main = print (f 2.3 4.2)
```

The problem: `4.2` isn't an Int.

The solution, 
don't declare the type for `f`.
Haskell will infer the most general type for us:


``` active haskell
f x y = x*x + y*y

main = print (f 2.3 4.2)
```

It works! 
Great, we don't have to declare a new function for every single type.
For example, in `C`, you'll have to declare a function for `int`, for `float`, for `long`, for `double`, etc...

But, what type should we declare?
To discover the type Haskell has found for us, just launch ghci:

```
%ghci
GHCi, version 7.0.4: http://www.haskell.org/ghc/  :? for help
Loading package ghc-prim ... linking ... done.
Loading package integer-gmp ... linking ... done.
Loading package base ... linking ... done.
Loading package ffi-1.0 ... linking ... done.
Prelude>let f x y = x*x + y*y
Prelude>:type f
f :: Num a => a -> a -> a
```

Uh? What is this strange type?

~~~
Num a => a -> a -> a
~~~

First, let's focus on the right part `a -> a -> a`.
To understand it, just look at a list of progressive examples: 

<table><tr><th>The&nbsp;written&nbsp;type</th><th>Its meaning</th>
</tr><tr><td> <code>Int</code>    </td><td> the type <code>Int</code>
</td></tr><tr><td> <code>Int -> Int</code>    </td><td> the type function from <code>Int</code> to <code>Int</code>       
</td></tr><tr><td> <code>Float -> Int</code>  </td><td> the type function from <code>Float</code> to <code>Int</code>
</td></tr><tr><td> <code>a -> Int</code>      </td><td> the type function from any type to <code>Int</code>
</td></tr><tr><td> <code>a -> a</code>        </td><td> the type function from any type <code>a</code> to the same type <code>a</code>
</td></tr><tr><td> <code>a -> a -> a</code>   </td><td> the type function of two arguments of any type <code>a</code> to the same type <code>a</code>
</td></tr></table>

In the type `a -> a -> a`, the letter `a` is a _type variable_. 
It means `f` is a function with two arguments and both arguments and the result have the same type.
The type variable `a` could take many different type value.
For example `Int`, `Integer`, `Float`...

So instead of having a forced type like in `C` with declaring the function for `int`, `long`, `float`, `double`, etc... 
We declare only one function like in a dynamically typed language.

Generally `a` can be any type. 
For example a `String`, an `Int`, but also more complex types, like `Trees`, other functions, etc...
But here our type is prefixed with `Num a => `. 

`Num` is a _type class_.
A type class can be understood as a set of types.
`Num` contains only types which behave like numbers.
More precisely, `Num` is class containing types who implement a specific list of functions, and in particular `(+)` and `(*)`.

Type classes are a very powerful language construct.
We can do some incredibly powerful stuff with this.
More on this later.

Finally, `Num a => a -> a -> a` means:

Let `a` be a type belonging to the `Num` type class.
This is a function from type `a` to (`a -> a`).

Yes, strange. 
In fact, in Haskell no function really has two arguments.
Instead all functions have only one argument.
But we will note that taking two arguments is equivalent to taking one argument and returning a function taking the second argument as parameter.

More precisely `f 3 4` is equivalent to `(f 3) 4`. 
Note `f 3` is a function:

``` haskell
f :: Num a => a -> a -> a
g :: Num a => a -> a
g = f 3
g y ⇔ 3*3 + y*y
```

Another notation exists for functions. 
The lambda notation allows us to create functions without assigning them a name.
We call them anonymous function.
We could have written:

``` haskell
g = \y -> 3*3 + y*y
```

The `\` is used because it looks like `λ` and is ASCII.

If you are not used to functional programming your brain should start to heat up.
It is time to make a real application.

But just before that, we should verify the type system works as expected:


``` active haskell
f :: Num a => a -> a -> a
f x y = x*x + y*y

main = print (f 3 2.4)
```

It works, because, `3` is a valid representation both for Fractional numbers like Float and for Integer.
As `2.4` is a Fractional number, `3` is then interpreted as being also a Fractional number.

If we force our function to work with different types, it will fail:


``` active haskell
f :: Num a => a -> a -> a
f x y = x*x + y*y

x :: Int
x = 3
y :: Float
y = 2.4
main = print (f x y) -- won't work because type x ≠ type y
```

The compiler complains. 
The two parameters must have the same type.

If you believe it is a bad idea, and the compiler should make the transformation 
from a type to another for you, you should really watch this great (and funny) video:
[WAT](https://www.destroyallsoftware.com/talks/wat)

#### Exercises

What is the type of the following functions?

``` haskell
f x = x
h x = "Hello"
p a b c x = a*x*x + b*x + c 
```

@@@ Solutions
``` haskell
-- show The argument can be anything, even another function
f :: a -> a
f x = x 

-- show String is a better name for [Char]
h :: a -> String
h x = "Hello"

-- show Beware the 'a' in the type is a type, in the definition the symbol 'a' is a parameter
p :: (Num a) => a -> a -> a -> a -> a
p a b c x = a*x*x + b*x + c 

```
@@@

[continue to part 2](https://www.fpcomplete.com/school/haskell-fast-hard/haskell-fast-hard-part-2)