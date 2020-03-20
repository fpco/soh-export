# 2. Playing with Routes and Links

In the <a href="yesod-1" target="_blank">previous tutorial</a> I described a single-file Yesod program that ran a very simple web site. I will continue adding new features to this program to illustrate various aspects of Yesod while at the same time teaching you elements of Haskell.

I'll be using the style of programming that's quite appropriate for a tutorial, but you should realize that an industrial web site requires much more structure. For instance, most of the HTML, CSS, and JavaScript code would be put in separate files rather than embedded directly in Haskell code. The Yesod framework supports this professional style of programming by providing a ready-made scaffolding for a web site, which you can fill with your custom code. We'll talk about scaffolding in the later installments of this tutorial. For now let's explore some of the customization points I talked about in the previous tutorial: adding routes, adding links between pages, and exploring the `whamlet` EDSL. For your convenience, here's our starting point from the previous tutorial:

``` active haskell web
{-# LANGUAGE TypeFamilies, QuasiQuotes, MultiParamTypeClasses,
             TemplateHaskell, OverloadedStrings #-}
import Yesod

data Piggies = Piggies

instance Yesod Piggies

mkYesod "Piggies" [parseRoutes|
  / HomeR GET
|]

getHomeR = defaultLayout [whamlet|
  Welcome to the Pigsty!
|]

main = warpEnv Piggies
```

##Routes

First thing that comes to mind is to enrich our web site by adding more pages and some kind of navigation between them. We'll create the "about" page by first adding a new route to our list of routes:

``` active haskell web
{-# LANGUAGE TypeFamilies, QuasiQuotes, MultiParamTypeClasses,
             TemplateHaskell, OverloadedStrings #-}
import Yesod

data Piggies = Piggies

instance Yesod Piggies

-- show This won't compile!
mkYesod "Piggies" [parseRoutes|
  /      HomeR  GET
  /about AboutR GET
|]
-- /show
getHomeR = defaultLayout [whamlet|
  Welcome to the Pigsty!
|]

main = warpEnv Piggies
```
Remember, routes are defined using a simple EDSL `parseRoutes`. You add a route by adding a line to the quasi-quoted argument to the (Template Haskell) function `mkYesod`. The first element in the line is the route itself -- `/about` in this case. If the URL of the web site is, say, `http://fpcomlete.com`, the URL of the about page will be 

``` haskell
http://fpcomlete.com{-hi-}/about{-/hi-}
```

The second element in the line is the resource name, `AboutR`, which we'll be using internally to address this page. Finally, `GET` is the HTTP request method that will be used to access this page. (Note: This is just a simple example of a static route. Later you'll see how to construct dynamic routes parameterized by integers, dates, names, and so on.)

An interesting exercise is to try to compile our program with just this one change. The error message you get is very telling:

```
Not in scope: `getAboutR'
```
Each route passed to `mkYesod` has to have a corresponding handler defined, and we are missing one for the About page. Let's stub it for now:

``` active haskell web
{-# LANGUAGE TypeFamilies, QuasiQuotes, MultiParamTypeClasses,
             TemplateHaskell, OverloadedStrings #-}
import Yesod

data Piggies = Piggies

instance Yesod Piggies

mkYesod "Piggies" [parseRoutes|
  /      HomeR  GET
  /about AboutR GET
|]

getHomeR = defaultLayout [whamlet|
  Welcome to the Pigsty!
|]

-- show This will compile!
getAboutR = defaultLayout [whamlet||]
-- /show

main = warpEnv Piggies
```
(Later I'll show you how to use `undefined` to stub function bodies -- we could have done it here but we would need a type signature for `getAboutR` and we haven't gotten that far yet.)

We already had a handler for the home route, but let's play with it a little:

``` active haskell web
{-# LANGUAGE TypeFamilies, QuasiQuotes, MultiParamTypeClasses,
             TemplateHaskell, OverloadedStrings #-}
import Yesod

data Piggies = Piggies

instance Yesod Piggies

mkYesod "Piggies" [parseRoutes|
  /      HomeR  GET
  /about AboutR GET
|]

-- show
getHomeR = defaultLayout [whamlet|
  Welcome to the Pigsty!
  <br>
  <a href=@{AboutR}>About Us.
|]
-- /show

getAboutR = defaultLayout [whamlet||]

main = warpEnv Piggies
```

You can see now that `whamlet` is really an enhanced version of HTML. You can insert HTML tags in it -- like the line break `<br>` or the anchor `<a ...>`. I used a few features of `whamlet` that make it more convenient than straight HTML (which, by the way, you can also use in Yesod -- more about it later).

One of the features is that, <a title="Haskell Syntax" href="http://fpcomplete.com/ten-things-you-should-know-about-haskell-syntax/" target="_blank">just like in Haskell</a>, whitespace is significant and the compiler pays attention to formatting. It means that if you open a tag on a separate line, you don't need to provide the closing tag. I used this feature to elide the closing `</a>` tag after "About Us." Granted, this is a minor convenience, but it removes a major source of annoyance: when you forget the closing tag or mismatch the opening and closing tags. Meaningful formatting goes even further: You may use indentation to nest your tags. For instance, you can create a paragraph block, complete with the embedded link:

``` active haskell web
{-# LANGUAGE TypeFamilies, QuasiQuotes, MultiParamTypeClasses,
             TemplateHaskell, OverloadedStrings #-}
import Yesod

data Piggies = Piggies

instance Yesod Piggies

mkYesod "Piggies" [parseRoutes|
  /      HomeR  GET
  /about AboutR GET
|]

-- show Go ahead, run it!
getHomeR = defaultLayout [whamlet|
    Welcome to the Pigsty!
    {-hi-}<p>{-/hi-}
        If you'd like to know more
        <a href=@{AboutR}>About Us
        we'll be happy to oblige.
|]
-- /show

getAboutR = defaultLayout [whamlet||]

main = warpEnv Piggies
```

The indentation trick works because of the nesting property of HTML tags. (It also works in formatting Haskell code because of the nesting property of scopes.)

##Type-Safe URLs

There's more interesting stuff going on here -- interpolation. That's when you insert in-place code in the middle of formatting, as in:

``` haskell 
<a href={-hi-}@{AboutR}{-/hi-}>About Us.
```
In regular HTML you would use a URL (whether absolute or relative) in a link tag. For instance:

``` haskell 
<a href={-hi-}"/about"{-/hi-}>About Us.
```
That works, but it introduces a fragile dependency. Essentially, once you decide on the URLs for your internal pages, you should never change them, because you're going to break internal links. Yes, there are utilities that can scan your web site and report broken links, but that's a crutch. What you really want is the web site with broken links not to compile. And that's what Yesod provides through type-safe URLs.

So how does this work? You've already seen one part of the solution: a level of indirection between routes and resources. The route for the about page, which is also its URL, is defined as `/about`. The resource corresponding to it is a Haskell data constructor, called `AboutR` (we already have one such data constructor, `HomeR`; `AboutR` is the second data constructor for the same type -- see "A Bit of Haskell" at the end).

In Yesod you define your routes once and then use their <em>resource names</em> everywhere. How is this different? Since each resource is not a string but a separate Haskell type constructor, if you change its name in any place you will get an undefined symbol error and the code will not compile. 

Let's try it. If I make a typo (can you spot it?):

``` active haskell web
{-# LANGUAGE TypeFamilies, QuasiQuotes, MultiParamTypeClasses,
             TemplateHaskell, OverloadedStrings #-}
import Yesod

data Piggies = Piggies

instance Yesod Piggies

mkYesod "Piggies" [parseRoutes|
  /      HomeR  GET
  /about AboutR GET
|]

getHomeR = defaultLayout [whamlet|
    Welcome to the Pigsty!
    {-hi-}<p>{-/hi-}
        If you'd like to know more
-- show See the compilation error you get
        <a href=@{{-hi-}AbuotR{-/hi-}}>About Us.
-- /show
        we'll be happy to oblige.
|]

getAboutR = defaultLayout [whamlet||]

main = warpEnv Piggies
```

I get the following error:

```
Not in scope: data constructor `AbuotR'
```
(The GHC compiler is careful not to say `AbuotR` is undefined, because it could be defined somewhere else. Instead it tells you that `AbuotR`'s definition is not in scope. If you come from C++, you need to get used to "not in scope" meaning "undefined".)

Type-safe URLs also help you detect errors in more complex cases, for instance when you're dealing with parameterized routes. In that case the type constructor for a resource takes an argument (an integer, a date, or a user name). If you call it with the wrong type of argument, the program will not compile.

You've probably figured it out by now, or remember from the previous tutorial, but let me repeat: The syntax for embedding routes inside whamlet is to enclose them with `@{` and `}`. We'll see other types of interpolation soon.

To finish this example, let's expand the stub handler for the new route. We already know the name of the handler from the error message we got in the beginning, but let me remind you that the name is constructed from a lowercase request type, `get`, followed by the resource name, `AboutR`:

``` active haskell web
{-# LANGUAGE TypeFamilies, QuasiQuotes, MultiParamTypeClasses,
             TemplateHaskell, OverloadedStrings #-}
import Yesod

data Piggies = Piggies

instance Yesod Piggies

mkYesod "Piggies" [parseRoutes|
  /      HomeR  GET
  /about AboutR GET
|]

getHomeR = defaultLayout [whamlet|
    Welcome to the Pigsty!
    <p>
        If you'd like to know more
        <a href=@{AboutR}>About Us
        we'll be happy to oblige.
|]

-- show
getAboutR = defaultLayout [whamlet|
  Enough about us!
  <br>
  <a href={-hi-}@{HomeR}{-/hi-}>Back Home.
|]
-- /show

main = warpEnv Piggies
```

This code is very similar to the root handler, except that it contains a link back to the root page instead. You should be getting the hang of it by now. Try adding more pages to our web site, cross-linking them, and playing with HTML tags (various levels of headings, lists, spans, etc.).

##Conclusion

The important lesson is that Yesod's creative use of Haskell's type system makes web applications more robust. We've just seen the example of type-safe URLs, and we'll see more such examples in the future.

##A Bit of Haskell

In this installment we talked about data constructors like `HomeR` or `AboutR` so it makes sense to learn a bit about defining data types in Haskell. We've already seen one example in the previous tutorial, that of the Yesod foundation type for our website:

``` haskell 
data Piggies = Piggies
```
A bit of syntax: There are three ways of defining types in Haskell:

- Using `data` you can define an arbitrary (algebraic) data type. This is the workhorse of the type system.
- Using `type` you can define an alias or a synonym for a type. This is equivalent to C++ `typedef`. For instance, a `String` is an alias for a list of characters:

``` haskell 
type String = [Char]
```
All functions that work on lists also work with `String`s.
- Using `newtype` you can create a new type from an existing one. For instance, if you have a function that takes, say, a first name and a last name, and you don't want the two to be confused, you create new types for them, complete with data constructors:

``` haskell 
newtype FirstName = FirstName String
newtype LastName  = LastName String
```
Now, if you pass a `String` or a `LastName` where `FirstName` is expected, you'll get an error. You could accomplish the same with a `data` definition, but `newtype` is quicker and more efficient.

For now, let's concentrate on `data`.

The `Piggies` definition is interpreted as follows: We are defining a type `Piggies` on the left hand side of the equal sign. The `Piggies` on the right hand side is a <em>data constructor</em>, which can be used to create values of type `Piggies`. These two names don't have to be the same. In fact, you can try changing the definition to:

``` haskell 
data Piggies = {-hi-}MakePiggies{-/hi-}
```
and see which parts of code require the type name and which require the type constructor.

The line:

``` haskell 
instance Yesod {-hi-}Piggies{-/hi-}
```
doesn't change. It states that the <em>type</em> `Piggies` is an instance of the <em>type class</em> `Yesod` (more about type classes in the future).

The (Template Haskell) function `mkYesod` also requires the name of the type (as string):

``` haskell 
mkYesod {-hi-}"Piggies"{-/hi-} [parseRoutes|
  /      HomeR  GET
  /about AboutR GET
|]
```
But the argument to `warpEnv` must be a value, so we need a data constructor to create it:

``` haskell 
main = warpEnv {-hi-}MakePiggies{-/hi-}
```
We've learned in this tutorial that resource names are data constructors. So what is the data type that they construct? The long answer would require the knowledge of Haskell type families (that's what the language pragma `TypeFamilies` at the top of the file is for), so I'll postpone it till much later. For now you can just imagine that there is a data type called `Route_Piggies` and its definition is:

``` haskell 
data Route_Piggies = HomeR | AboutR
```
Now we have two data constructors on the right hand side separated by a vertical bar (pronounced "or"). We use these data constructors to create values that are passed to `whamlet` through escape sequences `@{...}`. If you try to pass anything else, like a string or a widget, you'll get a really scary compiler error.

Another example of this type of data definition is the `Bool` type:

``` haskell 
data Bool = True | False
```
You read this definition as "Bool is either True of False." You are free to think of data types with constructors that take no arguments (and that's what we've seen so far) as corresponding to enumerations in other languages. We'll talk more about defining data types with non-trivial constructors and about recursive and parameterized data types in the future.

Important: The names of types and data constructors must start with a **capital letter**. This rule applies to buit-in types as well. Function and variable names always start with a **lower case** letter.

## Some Basic Data Types

- `Char`: Unicode character. A list of `Char` is `[Char]` (more about lists later).
- `Bool`: An enumeration of `True` and `False`.
- `Int`: Native integer. There are also fixed-width numeric types defined in modules `Data.Int` (signed) and `Data.Word` (unsigned).
- `Integer`: More expensive infinite precision integer. It lets you calculate things like factorials of large numbers. It is implemented using a highly optimized GMP library, so it's not as bad as it looks.
- `Double`: Native floating-point number.

## Acknowledgments

I'd like to thank Michael Snoyman and Andres Löh for reviewing my posts. I'm also grateful to many people who commented on my previous tutorials and discussed them on reddit. 

## Appendix

For your reference, here's the complete program:

``` active haskell web
{-# LANGUAGE TypeFamilies, QuasiQuotes, MultiParamTypeClasses,
             TemplateHaskell, OverloadedStrings #-}
import Yesod

data Piggies = Piggies

instance Yesod Piggies

mkYesod "Piggies" [parseRoutes|
  /      HomeR  GET
  /about AboutR GET
|]

getHomeR = defaultLayout [whamlet|
  Welcome to the Pigsty!
  <br>
  <a href=@{AboutR}>About Us.
|]

getAboutR = defaultLayout [whamlet|
  Enough about us!
  <br>
  <a href=@{HomeR}>Back Home.
|]

main = warpEnv Piggies
```

