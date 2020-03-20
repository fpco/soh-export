# Calculator

Console Version
---------------

How would we write a calculator program in Haskell?  A simple calculator should be able to perform the basic arithmetic operations on two numbers.

```active haskell
data Operator = Add | Subtract | Multiply | Divide
  deriving Read

eval l o r = case o of
  Add      -> l + r
  Subtract -> l - r
  Multiply -> l * r
  Divide   -> l / r

prompt txt = do
  putStrLn txt
  readLn

main = do
  l <- prompt "Left operand?"
  o <- prompt "Which operator?"
  r <- prompt "Right operand?"
  putStrLn $ "The result is " ++ show (eval l o r)
```

Note that since we used `deriving Read`, we get an instance of the `Read` typeclass.  That means that the <hoogle>read</hoogle> function now has a default implementation for our datatype `Operator`.  This default implementation expects the input to be formatted in the same way that <hoogle>show</hoogle> formats its output.  In this case, that means it expects one of "Add", "Subtract", "Multiply", or "Divide".

But something interesting is going on!  `prompt "Left operand"` and `prompt "Right operand"` are very similar expressions, but one reads numbers while the other reads operators.  The type of read is `Read a => String -> a`.  This means that `read` can be used on a `String`, to return any type `a`, that implements the `Read` typeclass.

```active haskell
main = do
  value <- readLn
  print (value :: Int)
```

Try changing the type of `value` to `String`, `[Int]`, `Bool`, etc.  If you remove it, there will be an "ambiguous type variable" exception - this is the compiler telling you that it needs more information in order to determine the type.  Without knowing the type, Haskell can't determine how to <hoogle>readLn</hoogle> (which uses `read`) or <hoogle>print</hoogle> (which uses `show`).


Yesod Version
-------------

Let's make our calculator program into a website!  Here's our pre-requisites:

```haskell
{-# LANGUAGE TypeFamilies, QuasiQuotes, MultiParamTypeClasses,
             TemplateHaskell, OverloadedStrings #-}
import Yesod
import Yesod.Form
import Control.Applicative
```

Nevermind the `LANGUAGE` stuff.  Beyond that, we're depending on Yesod and its forms.  The operators from `Control.Applicative` are needed to build forms.

Next, we write a simple one-page website:

```haskell
data Calculator = Calculator

instance Yesod Calculator

-- Necessary to tell Yesod how to render form-related error messages.
instance RenderMessage Calculator FormMessage where
    renderMessage _ _ = defaultFormMessage

mkYesod "Calculator" [parseRoutes|
/ HomeR GET
|]

main = warpEnv Calculator
```

The code for the form looks like this:
```haskell
data Calculation = Calculation Double Operator Double

form = renderDivs $ Calculation
    <$> areq doubleField               "Left operand"  Nothing
    <*> areq (selectField optionsEnum) "Operator"      Nothing
    <*> areq doubleField               "Right operand" Nothing
```

See the [Yesod Book](http://www.yesodweb.com/book/forms) for more information about forms.  In this example `areq` builds a required field.  `doubleField` and `selectField optionsEnum` specify different types of forms widgets (as well as a way of parsing the results).

The operators `(<$>)` and `(<*>)` from `Control.Applicative` allow us to treat the fields as if they yield values, and wrap up those results using a `Calculation` datatype.

Finally, we can define our page handler and use logic code that's similar to the console version above:

```active haskell web
{-# LANGUAGE TypeFamilies, QuasiQuotes, MultiParamTypeClasses,
             TemplateHaskell, OverloadedStrings #-}
import Yesod
import Yesod.Form
import Control.Applicative

data Calculator = Calculator

instance Yesod Calculator

instance RenderMessage Calculator FormMessage where
    renderMessage _ _ = defaultFormMessage

mkYesod "Calculator" [parseRoutes|
/ HomeR GET
|]

main = warpEnv Calculator

-- show
data Calculation = Calculation Double Operator Double

form = renderDivs $ Calculation
    <$> areq doubleField               "Left operand"  Nothing
    <*> areq (selectField optionsEnum) "Operator"      Nothing
    <*> areq doubleField               "Right operand" Nothing

getHomeR :: GHandler sub Calculator RepHtml
getHomeR = do
    ((result, widget), enctype) <- runFormGet form

    let resultText = "Result: " ++ case result of
          FormSuccess (Calculation l o r) -> show (eval l o r)
          _ -> ""

    defaultLayout $ do
        [whamlet|
            <h1> Calculator </h1>
            <form enctype=#{enctype}>
                ^{widget}
                <input #button type="submit" value="Calculate!">
            <h2> #{toHtml resultText}
          |]
        toWidget [cassius|
            body          { width: 15em; margin: 0em auto; }
            #button       { width: 10em; }
            input, select { width: 4em; margin: 0.5em; }
            .required     { text-align: right; }
          |]

data Operator = Add | Subtract | Multiply | Divide
  deriving (Eq, Enum, Bounded, Show)

eval l o r = case o of
    Add      -> l + r
    Subtract -> l - r
    Multiply -> l * r
    Divide   -> l / r
```

The logic code has one small difference - now, instead of `deriving Read`, we do `deriving (Eq, Enum, Bounded, Show)`.  This is because these instances are needed by `selectField optionsEnum`.  <hoogle>Eq</hoogle> is needed to determine which of the options was selected.  <hoogle>Bounded</hoogle> and <hoogle>Enum</hoogle> are used to enumerate all the possible values of `Operator`.  <hoogle>Show</hoogle> is used to determine the label used in the drop-down box entry.