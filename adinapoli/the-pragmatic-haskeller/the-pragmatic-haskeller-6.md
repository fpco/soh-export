# Episode 6 - Wrapping Up

## Foreword

This is part of [The Pragmatic Haskeller](https://github.com/cakesolutions/the-pragmatic-haskeller) series.

Note: The code we are referring to is contained inside the folder **08-heist** of the Github repository.

### Wiring everything together

At the moment, we have *almost* everything in place: we have a JSON layer to marshall/unmarshall our
data structures, we have an API we can call if we want recipes, we can store our JSON as a BSON into
a MongoDB collection and we have a simple but effective DSL for describing recipes. Now it's time to
expose our mini language to the world, and obviously for a web application this means writing a bunch
of HTML pages. We'll do that using Snap's templating system, Heist. Even though you can use pretty
much every template language you want, Heist is the one which come shipped with Snap, so we'll stick
with that. But before diving into that, we need to fix the bug I was talking about last time, do
you remember?

### Using lens for accessing and modifying immutable data structures

If you followed carefully the [last episode](https://www.fpcomplete.com/user/adinapoli/the-pragmatic-haskeller/episode-5-a-simple-dsl),
you might have noticed that our parser was a bit naive. Let's try to run again the code and see the output:

``` haskell
Right (Recipe {recipeName = "Ciambellone", ingredients = [Ingredient {ingredientName = "Flour",
quantity = 250, measure = Just "gr"},Ingredient {ingredientName = "Sugar", quantity = 250, measure = Just "gr"},
Ingredient {ingredientName = "Sunflower Oil", quantity = 130, measure = Just "ml"},
Ingredient {ingredientName = "Water", quantity = 130, measure = Just "ml"},
Ingredient {ingredientName = "Eggs", quantity = 3, measure = Nothing}], steps = [
Step {stepName = "Mixing everything", order = 1, stepDuration = Nothing},
Step {stepName = "Cooking in oven at 200 degrees", order = 1,
stepDuration = Just (Duration {duration = 40, durationMeasure = "minutes"})}]})
```

Do you spot something wrong? Well, look at the order or our steps, it's always 1! This is because `order` is
not something we create as the result of the parsing, so in the parser we put a placeholder value to allow
our parser to finish. Now it's time to fix things using [lens(es)](http://hackage.haskell.org/package/lens).
Again, the goal of this episode is not learning you to use lenses (many have tried),
but how you can pragmatically use a tiny part of this fantastic package to get the job done.


``` active haskell

{-# START_FILE Main.hs #-}
{-# LANGUAGE TemplateHaskell #-}

module Main where

import Control.Lens
import Control.Applicative

-------------------------------------------------------------------------------
type Measure = String


-------------------------------------------------------------------------------
data Ingredient = Ingredient 
  { ingredientName :: String
  , quantity :: Int
  , measure :: Maybe Measure
  } deriving Show


-------------------------------------------------------------------------------
data Duration = Duration
  { duration :: Int
  , durationMeasure :: Measure
  } deriving (Eq, Show)


-- show
-- We need to "embue" our data type of magical lens powers.
-- The other data types remain the same.
-------------------------------------------------------------------------------
data Step = Step
  { _stepName :: String
  , _order :: Int
  , _stepDuration :: Maybe Duration
  } deriving (Eq, Show)

makeLenses ''Step

instance Ord Step where
    compare s1 s2 = compare (_order s1) (_order s2)

-------------------------------------------------------------------------------
data Recipe = Recipe
  { _recipeName :: String
  , _ingredients :: [Ingredient]
  , _steps :: [Step]
  } deriving Show

makeLenses ''Recipe


-- We can now succinctly correct the order
correctOrder :: Recipe -> Recipe
correctOrder r = r { _steps = newSteps (_steps r)}
  where newSteps s = zipWith (over order) (const <$> [1..length s]) s


buggedRecipe :: Recipe
buggedRecipe = Recipe "Ciambellone" [
    Ingredient "Flour" 250 (Just "gr"),
    Ingredient "Sugar" 250 (Just "gr")]
    [Step "Mixing everything" 1 Nothing
    ,Step "Cooking in oven at 200 degrees" 1 (Just (Duration 40 "minutes"))]

main :: IO ()
main = do
    print . show $ buggedRecipe
    print . show . correctOrder $ buggedRecipe
-- /show
```

Wow, that was short! The above snippet is quite laconic, so I think it's
worth an accurate explanation. What we want is to "enumerate" our steps,
so ideally for each step we want to generate ad assign an incremental
counter. Do do that, we need to actually generate the enumeration, which
is what, unsurprisingly, `[1 .. length s]` does. So far so good. Now we
also need a way to "map" this enumeration over our existing steps,
yielding another list of steps, but with the correct order. To do that,
let's first recall the signature of `zipWith`:

``` haskell
zipWith :: (a -> b -> c) -> [a] -> [b] -> [c]
```

So all we need to pass to our zipWith is a function and two lists: for
each `a` and `b` we read from the lists, `zipWith` will apply the
function to them, producing a value `c` which will be accumulated into
a list produce the final result! The list of `b` is obvious, it's our
old list of steps, but what about the function to apply and the first
list? The answer lies in the `over` function, which simplified
signature can be daunting at first:

``` haskell
over :: Setter s t a b -> (a -> b) -> s -> t
```

We can intuitively read the above as: "Given a `Setter` and a
function `f` from `a` to `b` along with a source `s`, over
apply `f` to the source, producing the target `t`. If you think
about that, it's all we need! We have the `Setter` (`order`) generated
for us for free when we called `makeLenses ''Step`, and we do have
the source, it will be a single `Step` record. But what about the
function? It's easy! All we need is something that, no matter what
we feed into, will yield a constant result.. well, apparently
`const` is what we need! So let's wire up everything together:

* We build a list of partially applied functions, such as:
  `[const 1, const 2, ....]`
* We zip this list together with `over order` and a `[Step]`.

Let's simulate the first two steps:

``` active haskell

{-# START_FILE Main.hs #-}
{-# LANGUAGE TemplateHaskell #-}

module Main where

import Control.Lens

-- We need to "embue" our data type of magical lens powers

-------------------------------------------------------------------------------
type Measure = String


-------------------------------------------------------------------------------
data Ingredient = Ingredient 
  { ingredientName :: String
  , quantity :: Int
  , measure :: Maybe Measure
  } deriving Show


-------------------------------------------------------------------------------
data Duration = Duration
  { duration :: Int
  , durationMeasure :: Measure
  } deriving (Eq, Show)


-------------------------------------------------------------------------------
data Step = Step
  { _stepName :: String
  , _order :: Int
  , _stepDuration :: Maybe Duration
  } deriving (Eq, Show)

makeLenses ''Step

instance Ord Step where
    compare s1 s2 = compare (_order s1) (_order s2)

-- show
-- correspond to the first invocation of zipWith:
main :: IO ()
main = do
    print . show $ over order (const 1) (Step "mix everything" 1 Nothing)
    print . show $ over order (const 2) (Step "cook in the oven" 1 Nothing)
-- /show
```

Fantastic, we have corrected our orders in a one liner!


### Writing our first heist template

Brilliant! Now we have our DSL ready, a way to convert the generated data
type from and to JSON and a way to serialise everything as BSON inside MongoDB.
What else we need? Well, we need user interaction! We're going to build a simple
Heist template to display a text area where the user can modify our DSL, and a
button to send the DSL to the server, where it will be parsed, validated and
if it yields a valid `Recipe`, converted to JSON and showed in a "success" page.
We'll simplify a bit, not writing into the DB, but just because I didn't want
to have too much data being written in my small Amazon Box. But you already saw
the episode where we were talking with Mongo, and you know that once we have our
Aeson `Object` converting it into BSON is a one liner.

Without further ado, let's write first the presentation layer, a.k.a our Heist
template. For who have no clue about what I'm talking about, Heist is just
another library in the Snap "toolbelt". Quoting from the documentation:

 _"Heist is a templating engine based loosely on ideas from the Lift Web Framework.
 It functions as a bridge between your web application and its views.
 Heist templates are HTML (or XML) fragments that are not required to have a single root element."_

So, at least for the front-end developer, Heist is nothing more than a template engine.
Armed with this knowledge, we can write this simple template:

``` html
 <!-- new_dsl.tpl -->
 <apply template="base">
 
     <div class="control-group span5">
       <form method="post" action="/recipe/new">
           <label class="control-label" for="dsl">Submit your recipe</label>
           <div class="controls">
             <textarea class="input-xlarge span5" name="dsl" id="dsl">
 <recipe/>
             </textarea>
             <br/>
             <button type="submit" class="span5 btn btn-primary">Save your recipe!</button>
             <div class="span5">
               <parsingError/>
             </div>
           </div>
       </form>
 
     </div>
 
     <div name="errorLine" id="errorLine" class="hidden"><editorCurrentLine/></div>
 
     <script>
       var editor = CodeMirror.fromTextArea(document.getElementById("dsl"), {
         lineNumbers: true,
         styleActiveLine: true,
         theme: "ambiance",
         mode: "recipe"
       });
       var cursorLine = parseInt(document.getElementById("errorLine").innerText)
       editor.setSize(500,350);
       editor.setCursor(cursorLine-1)
     </script>
 
 </apply>

```

Even if it's the first time you ever looked at an Heist template, it should
be familiar; it's just xml/html on steroid. The only unfamiliar bits are:

* `apply template="base"`, which, unsurprisingly, it's just a way to enforce
  reuse and composability, allowing us to reuse a template as "container" for
  our specific page. Inside `base.tpl` ((browse it!)[https://github.com/cakesolutions/the-pragmatic-haskeller/blob/master/08-heist/snaplets/heist/templates/base.tpl])
  you'll find a tag called `<apply-content/>` which has the purpose of doing exactly that.

* Some "unbound" tags such as `<recipe/>` or `<editorCurrentLine/>`: they are
  our "entrypoints" for *splices*, which will be discussed in a minute.

### Handling our route and writing a Splice

The last bit which will show you now it's probably the most important bit of
the entire series (no pressure!) because it wires together everything and allows
our web application to actually doing "something interesting". Obviously I can't
cover the entire app here and that's why there is a Github repository you can
clone to have a play. In this last paragraph we'll write a *handler* which will
produce a *splice*. Ok, but what's a *splice* exactly? Quoting from the Snap's
documentation: "A Splice takes the input node from the template and outputs a
list of nodes that get “spliced” back into the template. This lets you call
haskell functions from your templates, while ensuring that business logic does
not creep into the presentation layer". Sounds nice! So, in a nutshell, we'll
write our *splice* and instruct Snap which tag we want to bind to a particular
*splice*, and we're sorted! Snap will "splice back" the value where we
previously have the tag in our template. To enforce modularity and separation
of concerns, we'll create a file called `Routes.hs` inside the package `DSL`,
and finally we'll assemble its routes back to the main `Site.hs`. Ready for the code?

``` haskell
{-# START_FILE Routes.hs #-}
{-# LANGUAGE OverloadedStrings #-}

module Pragmatic.DSL.Routes where

import Control.Lens
import Data.Aeson.Encode.Pretty (encodePretty)
import Data.ByteString (ByteString)
import Data.ByteString.Lazy (toStrict)
import Data.Text (Text)
import Data.Text.Encoding (decodeUtf8)
import Pragmatic.DSL.Parser
import Pragmatic.JSON.Parser()
import Pragmatic.Server.Application
import Pragmatic.Types
import Snap
import Snap.Snaplet.Heist
import Text.Blaze.Html5
import Text.Blaze.Renderer.XmlHtml (renderHtmlNodes)
import Text.Parsec (parse)
import qualified Data.ByteString.Char8 as BC (unpack)
import qualified Data.Text as T
import qualified Heist.Interpreted as I
import qualified Text.Blaze.Html5 as H
import qualified Text.Blaze.Html5.Attributes as A

dslRoutes :: [(ByteString, AppHandler ())]
dslRoutes = [("/recipe/new", handleNewRecipe)]

recipe2json :: Recipe -> Text
recipe2json = decodeUtf8 . toStrict . encodePretty


correctOrder :: Recipe -> Recipe
correctOrder r = r { _steps = newSteps (_steps r)}
  where newSteps s = zipWith (over order) (const <$> [1..length s]) s


-- Can be refactored out
bootstrapAlert :: String -> String -> I.Splice AppHandler
bootstrapAlert alertType msg = return $ renderHtmlNodes innerHtml
  where innerHtml = H.div ! A.class_ (toValue ("alert alert-" ++ alertType)) $
                      toHtml msg

handleNewRecipe :: AppHandler ()
handleNewRecipe = method POST handleParsing
  where handleParsing = do
            dslSourceCode <- getPostParam "dsl"
            maybe (spliceError "Dsl can't be empty!" "")(\s ->
                  case parse recipe "" (BC.unpack s) of
                    Left e -> spliceError (show e) (T.strip . decodeUtf8 $ s)
                    -- Parsing succeeded, we render the template "new_recipe.tpl"
                    Right r -> let splices = [("json", I.textSplice $ recipe2json . correctOrder $ r)]
                                in renderWithSplices "new_recipe" splices) dslSourceCode

        spliceError e d = let splices = [("parsingError", bootstrapAlert "alert" e)
                                        ,("editorCurrentLine", findErrorLine e)
                                        ,("recipe", I.textSplice d)]
                             in renderWithSplices "new_dsl" splices 

        findErrorLine = I.textSplice . T.singleton . Prelude.head . snd . splitAt 6
```

Ok, it seems a lot of code in the first place, but don't worry, we'll go through it.
The only two functions worth explaining are `bootstrapAlert` and `handleNewRecipe`. Let's start
from the former. In a nutshell, this function generates, given a `msg` and an
`alertType`, the required HTML to display a [Bootstrap](http://twitter.github.io/bootstrap/) alert message.
It's doing that using [Blaze](http://hackage.haskell.org/package/blaze-html-0.6.1.1),
a fantastic library created by Jasper Van der Jeugt which allow us to
generate HTML directly from Haskell, with a extremely nice syntax.

As regards `handleNewRecipe`, the workflow is the following: if the method is a POST,
try to extract the content of the html element with id "dsl": if it's there (it's a `Just`),
then try to parse the content, reacting accordingly whether it's a valid recipe or not,
whereas if you got a `Nothing`, display an error to the user, informing that he can't
submit an empty DSL. The interesting stuff happen inside the "Right branch" of the `case`,
where we build an associative list which second argument of each tuple is a *splice*. You
might have notices some functions have been imported qualified with the prefix `I`, and
this is because recent Heist versions introduced "Compiled Splices" as opposed to
"Interpreted Splices", which we are using here. The discussion about "Interpreted" vs
"Compiled" is out the scope of this episode, so I'll add a link in the reference section
if you are interested in learning further.

It's still worth taking a look at how an interpreted Splice is defined, to understand
what we are doing here, and how the whole function typecheck. First of all, when we want
to build a Splice out from a simple `Text`, doing that is kid's stuff thanks to the
function `I.textSplice`:

``` haskell
I.textSplice :: Monad m => Text -> HeistT n m Template
```

In case we want to build something more complicated, like a list of Html nodes, things
are just slightly more complicated, and it's what we are doing in `bootstrapAlert`: we
are building a list of html nodes with:

``` haskell
renderHtmlNodes :: Text.Blaze.Html.Html -> [Text.XmlHtml.Common.Node]
```

and then putting this list inside the environment context calling `return`, and yielding
a `Splice AppHandler`. If we look at the implementation, an interpreted `Splice` is
defined as such:

``` haskell
type Splice m = HeistT m m [Node]
```

where, quoting from the documentation "The type parameter `m` is the runtime execution monad
(in a Snap application this will usually be `Handler` or `Snap`)". Aha! So this explain
why the code works (remember? our `AppHandler` is just a type synonym around
`Handler Pragmatic Pragmatic`)! It's because both `textSplice` and `bootstrapAlert` returns
an Heist transformer, where the "execution monad" will be bound to our `AppHandler`!

The last thing we need to do is to call the function `renderWithSplices` which has the
following type signature:

``` haskell
renderWithSplices :: HasHeist b => ByteString -> [(Text, SnapletISplice b)] -> Handler b v ()
```

Which basically takes a `ByteString` (the name of the *template* we want to target) and a list
of tuples to instruct Snap about the particular association between a tag `T` (in our template)
and a splice `S`: the result will be an `Handler b v ()`, exactly what we need (remember? A
route list in nothing more than a `[(ByteString, Handler b v ())]`, which associate for each
uri the correspondent `Handler` that handles it.

As you can see from the code, we call the `renderWithSplices` in two places: one is in case we
want to display parsing errors to the user (and even handle a basic form of validation, not
allowing empty text to be passed aroud for parsing), the other is when the parsing succeeded and
we want to show the user, on another page, the result of the parsing.

That's it! Now we can use our newly created route into the "main" one, defined inside `Site.hs`,
and finally call `http://localhost:8000/recipe/new` to see the outcome!

## Conclusions

During this months, I've showed you how you can build a basic Haskell web application to solve real world
problems. We have DB access, JSON parsing, a DSL and an API to talk to external services. A lot of business
out there have kick-started with a lot less! So I would humbly say "Mission accomplished". This series wasn't
meant to teach you everything or cover every possible case you might hit during your everyday job. We didn't
touch a lot of topics, one in particular being form validation. The goal of this series was to show you that
there is nothing intimidating about Haskell and, even if sometimes the community is accused of navel gazing,
there are lot of libraries and frameworks to get your everyday job done. Even though this tutorial spanned
across several months, I wrote the entire code for it in a couple of weeks, showing that even if you are
a intermediate Haskeller you can be quite productive with the language.

## External References

As usual, refer to the official documentations:

* [Heist](http://hackage.haskell.org/package/heist) documentation

* [Lens](http://lens.github.io/) documentation

* [Blog post](http://www.haskellforall.com/2013/05/program-imperatively-using-haskell.html) from Gabriel
  Gonzalez aka Tekmo which teaches us about "imperative programming" with lenses.

* [Explanation](http://snapframework.com/docs/tutorials/compiled-splices) about "Interpreted" vs "Compiled" Splices.

## The code

Grab the code [here](https://github.com/cakesolutions/the-pragmatic-haskeller/tree/master/08-dsl).
The example is self contained, just cabal-install it!

## What about other episodes?
You might notice that we jumped suddently from episode 5 to episode 8 and you might have been wondering
why. The answer is that episodes 6 and 7 didn't cover enough material for a separate tutorial, so I've
condensed everything in a final wrapping up tutorial. There is also an extra, unfinished episode called
`realtime` where I show a chart which updates itself in realtime calling asynchronously a server
endpoint. As an exercise, you might want to try to convert all the Javascript used into lovely Haskell
using, for example, [Fay](https://github.com/faylang/fay/wiki).

## Where to go from here

Well, you have two paths:

* Keeping exploring the Snap space, building your own `Snaplet` to solve a particular problem

* Try a bite of another web framework like `Yesod` or `Happstack`

Regardless on what you choose, you might also start exploring the fascinating world of FRP maybe
trying to integrate languages like `Elm` in your codebase. Good luck!
