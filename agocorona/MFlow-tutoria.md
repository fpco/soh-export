# Introduction to MFlow

# Introduction to the MFlow Web framework

Here the basic elements of the MFlow frameworks are put in action in an example to show how easy is to create a demo app with 10+ pages.

If you don´t know Haskell and just whant to see a code example and how it works, go straight to the code example, look at it, see what happens and read what it produces. You will see that it is pretty intuitive. Once you learn the combination rules for the operators and the main primitives, you can start to play with your own applications.

## View monad
The first of the two monads is the `View` monad, that is what the widgets are made of. A widget has signature 

``` haskell
View rendering monad returnValue
```

Widgets are combined using monadic and applicative combinators to create  more reusable complex widgets, upto whole pages.

A page is just a complex widget. They are composed of formlet elements, that are the elementary particles that widgets are made of. A formlet is a form element or a link, that displays itself and validates the input entered. they are combined using monadic combinators, besides applicative combinators. But Widgets in MFlow also include HTML formatting. There are push widgets, Ajax-autorefreshed widgets, dynamic widgets, page flow widgets, widget containers etc. But this does not change that. These flavours of widgets are created with modifiers of ordinary widgets that are part of the EDSL in the `View` monad. This EDSL is a superset of the [applicative](http://learnyouahaskell.com/functors-applicative-functors-and-monoids) operators used for creating [formlets](http://www.haskell.org/haskellwiki/Web/Libraries/Formlets) in other web frameworks.

But don´t feel intimidated by these concepts. You don´t have to know the theory behind the applicative and monadic operators and formlets to create MFlow applications. Just use them. This tutorial is about how intuitive is to mix components for your needs and how nice and composable are the haskell operators for that purpose, without regard for how they work at the byte level.

## FlowM monad
The second is the `FlowM` monad, in which the navigation of the application is defined, as a sequence, with ordinary flow-control statements, much like in a console application. 

The `FlowM` monad manages the synchronization of the server with the browser request. if a request comes from a previous page, the `flowM` monad will backtrack (go back) to he code of the page that handle that request. if a link point to a page down in the navigation tree, the `FlowM` monad will reach that page and will execute his view code to satisfy the  user request. That mechanism of tracking and backtracking across the navigation is part of he magic of the FlowM monad that is not explained here.

the signature of a navigation is:

``` haskell
FlowM rendering monad returnValue
```

That means that a module that implement a navigation, let´s call it a 'flowlet' for example a cart check-out has this signature, is type safe and can be combined with other flowlets to create applications.

Usually flows or navigations, that is procedures in the `FlowM` monad are run with `runNavigation`. This latter expect a name for the flow, besides the monadic flow itself. `runNavigation` run the flow in a endless loop. when the flow ends, it is restarted again. About the `step` there are much to say, but at this moment let´s not enter in the details

What connect both monads is  `page` or  `ask`  (both are identical)

``` haskell
page :: View r m a -> FlowM r m a
```

That is, `page` get a widget, renders it in the web browser, iterates until it obtains a valid output of the kind expected and return it to the flow. 


## Example
Here in this example you will see the basic widgets and widget combinators. You will see how pages are composed and how different pages compose a flow.

Since there is a problem with running more than one web snippet in a single tutorial, I have made this example long, but hopefully self explanatory.

There is a more advanced tutorial [The MFlow DSL](https://www.fpcomplete.com/user/agocorona/MFlowDSL) that show  more advanced features like  persistent session data, session management, container widgets or utilities for single page development, like `autoRefresh`, `push` or `witerate`. Also I will add a tutorial for using data and databases.

But first the basics. Look at the code, run the example, see what happens and read the messages. I hope that you find it autoexplicative. Specially if you are fluent with applicative combinators, formlets and HTML. For more advanced examples look at the [MFlow demo web site](http://mflowdemo.herokuapp.com)

``` active haskell web




module Main where

import MFlow.Wai.Blaze.Html.All
import Data.Monoid


main = runNavigation "" . step $ do
    r <- page  $  h3 << "Basic example with inputs and links"
              ++> getString (Just "text input")
              <* submitButton "OK"
              
    page $   b << ("you typed: "++ r)
         ++> p << "wlink's return typed values to the flow"
         ++> p << "in MFlow a page is a widget made of widgets"
         ++> wlink () << p << "next"


    r <- page $   h3 << "Other basic elements"
              ++> p << " the operator (++>) prepend HTML to a widget"
              ++> p << "in this case, a field that expect an Int"
              ++> getInt Nothing
                     <! [("placeholder","This is an attribute for the getInt form")]
              <* submitButton "OK"
              <++ p << "the operator (<<) add text to a blaze-html tag in this case"
              <>  p << "If the OverloadedStrings extension is used, this is not necessary"
              <>  p << " Note that form tags are added automatically"

    page $ b << ("you entered: "++ show r) ++> wlink () << p << "next"


    r <- page  $   h3 << "Here the alternative operator (<|>) is used to choose between two options"
               ++> wlink True  << b << "True" <++ br
               <|> wlink False << b << "False"

    page $ b << ("you entered: "++ show r)  ++> wlink () << p << "next"

    -- |+|    <+>

    r <- page $ ul << h3 << "Operators"
            ++> li << " (<<<) embed a widget in a tag"
            ++> li << " (|*>) intersperse a widget within a list of others"
            ++> ul << li << "in this case, a link is interspersed within a list of input fields"
            ++> ((h1 <<< wlink "next" << b << "next")
            |*> [getString Nothing <![("placeholder","enter field "++ show i)] | i <- [1..3]])
            <** submitButton "OK"

    case r of
      (Just _, Nothing) -> return ()
      (Nothing, Just s) -> do
                page $ p << ("you entered " ++ s ++ " in some box") 
                   ++> wlink " next" << b << "next"
                return()

    page $ pageFlow "1" $ do
        r <- h3 << "Page flows run a monad within a page"
             ++> p << "note in the code that this sequence is within 'page'"
             ++> p << "The call \"pageFlow\" creates a set of unique identifiers and stores"
             ++> p << "the data entered during the steps of the pageflow"
             ++> wlink True  << b << "True" <++ br
             <|> wlink False << b << "False"

        p << "Until the first line is not validated, the second does not execute"
          ++> p << ("you entered: "++ show r)  ++> wlink () << p << "next"

    page $ pageFlow "2" $ do
        h2 << "Field validators" ++> wlink () << p << "next"

        r <- p << "this field expect a string less than 5 characters. Otherwise it present a alert message"
             ++> getString Nothing
                `validate`
                (\r -> if length r > 5 then return . Just $ script << "alert ('length must be lsst than five chars ')"
                                       else return Nothing)
             <* submitButton "OK"

        p << ("you entered: "++ show r) ++> wlink () << p << "next"


    page $ pageFlow "3" $ do
        h3 << "A select/option box" ++> wlink () << b << "click" <++ br
        r <- getSelect
                        (setSelectedOption "" (p << "select a option")   <|>
                         setOption "red"      (b << "red")               <|>
                         setOption "blue"     (b << "blue")              <|>
                         setOption "Green"    (b << "Green"))
                         <! [("onchange","this.form.submit()")]

        p << (r ++ " selected") ++> wlink () << p <<  "next"

    let colors= getCheckBoxes(  (setCheckBox False "Red"   <++ b <<  "red")
                             <> (setCheckBox False "Green" <++ b <<  "green")
                             <> (setCheckBox False "blue"  <++ b <<  "blue"))

    page $ pageFlow "4" $ do
        h2 << "checkboxes" ++> wlink () << b << "click" <++ br

        r <- colors <** submitButton "submit"

        p << (show r ++ " selected") ++> wlink () << p <<  " next"

   
    page $ pageFlow "5" (do
       h3 << "A mix of applicative and monadic operators can be used to create multiple pageflows within a page" 
        ++> b << "here two pageflows in a form are composed with an alternative operator"
        ++> wlink "" << p << "click here"
       (pageFlow "colors" (do
         r <- colors <++ br
         p << (show r ++ "selected") ++> noWidget <++ br)
        <|>
        pageFlow "input" (do
         r <- getString Nothing  <++ br
         p << (show r ++"entered") ++> noWidget <++ br)
        <* submitButton "OK" ))
     <|> br ++> wlink "next" << b << "back to the beginning"

    return ()

    

    
```
You can press the back button at any time and redo each example if you wish.

After this hopefully self explanatory example, let´s talk about how some functionalities of other web frameworks are carried out in MFlow.

## Routes
A route is expressed in MFlow as a case statement depending on the return value of a menu page. There is no special syntax for it. It is just part of the navigation. Like any monadic computation in Haskell, a navigation is type safe.

This navigation define three routes and a menu page:

``` haskell

data Option= Option1 | Option2

main = runNavigation "verb" . step $ do
    r <- page  $   h3 << "Here the alternative operator is used to choose between two options"
               ++> wlink Option1 << b << "Choose Option1" <++ br
               <|> wlink Option2 << b << "Choose Option2"

    case r of
       Option1 -> proc1
       Option2 -> proc2

```

URLs which start with `http://host/verb/option1/...`  will go straight to `proc1`. The web browser will not display the menu. That is the magic of the tracking mechanism of MFlow. As you would expect, URLs which star with `http://host/verb/option2/..` will go strigh to `proc2`. But the URL `http://host/verb` will go to the initial menu.  

The same result would be achieved using `waction` instead of the case statement.


``` haskell

wlink x << rendering `waction` proc
   
```

Would execute the navigation proc with the parameter `x` when the link is pressed. the whole expression is a widget that can be used instead of `wlink x` alone. The advantage is that the widget defines its own navigation, and permits the creation of pages that import widgets with complex behavious whose details the programmer of the application should not be aware of.  However Page flows and `wcallback` can do it with less power.

The tree of routes can be as complex as you wish simply adding additional layers of menus. It is like the traditional procedural console oriented business after all. But you can press the back button or enter any arbitrarily REST URL to reach any of the options on the tree, as long as the navigation to the destination page is made of links.

If instead of "verb" runNavigation receives an empty string, the verb "noscrip" will be assumed. Besides the main flow defined within `runNavigation` there may be other auxiliary flows for auxiliary purposes. Push widgets install themselves as independent flows. A widget that uses AJAX may need to install a process to serve the request. that is created dynamically with the `requires` call. that´s why the main flow need an identifier: to separate the routes in the main flow from the others in auxiliary processes.

In the last version of MFlow, the default flow is ever the one specified by `runNavigation` unless the verb "" is used, in which case the default flow is "noscript" for back compatibility.

Since `runNavigation` execute the flow in a loop, to create an stateless procedure simulating a MVC framework with a single route, it is simple a matter of criating a navigation with a single page.


## Session Management

the session in MFlow  is the monadic procedure in the `FlowM` monad. The session data is in normal procedure variables, as in any procedure. That is a big advantage over MVC frameworks, where pages does not share variable scopes and the concept of flow is inexistent. The session state has to be stored in untyped collections accessed by a key string.

This snippet are three pages , the two first ask for a number. The third sum them.  To see it running go to [The MFlow DSL](https://www.fpcomplete.com/user/agocorona/MFlowDSL)

``` haskell 

{-# LANGUAGE OverloadedStrings #-}

import MFlow.Wai.Blaze.Html.All

main= runNavigation "" . step $ do
       n  <- page $ getInt Nothing <** submitButton "first"
       n' <- page $ getInt Nothing <** submitButton "second"
       page $ p << ( n+n') ++> wlink () "click"
```

The two numbers are managed as in any other monadic procedure. However, since the FlowM monad backtrack, the session is managed as expected: If after showing the sum you go back and enter a new second number, the sum of the first and the new second will appear. And so on.

But sometimes is necessary to "transport" data among procedures, and to pass data as parameters among multiple functions can be tedious and inelegant. in Haskell, there is the state monad. The user is free to use it, but it is too technical for people that want just to use MFlow as a DSL to get their application easy and fast. For this purpose exist `getSessionData` and `setSessionData`. 

This example does the same than the previous example:

``` haskell 

{-# LANGUAGE OverloadedStrings #-}

import MFlow.Wai.Blaze.Html.All

main= runNavigation "" . step $ do
       n  <- page $ getInt Nothing <** submitButton "first"
       n' <- page $ getInt Nothing <** submitButton "second"
       setSessionData (n,n')
       computeSum

computeSum= do
       (n,n') <- getSessionData `onNothing` return (0:: Int,0 ::Int)
       page $ p << ( n+n') ++> wlink () "click"
```
To see it running, go to [The MFlow DSL](https://www.fpcomplete.com/user/agocorona/MFlowDSL)

`getSessionData`, in this case, look for a tuple of two Ints . if `setSessionData` did not store two integers previously, then onNothing will return (0,0) . That was not the case. computeSum can be called without parameters because the two integers are in the session state. The session state is navigation-wide.

These primitives can be used in both sides of `page`. If the user press the back button, the session state may backtrack or not depending on if you read it in one side or in the other. When going back, the only code executed is the code in the right of `page`. If `getSessionData` is there, it will retrieve the last state no matter if it is going back. This is the way to keep the user-defined state actualized to the last value when pressing the back button. Otherwise, if `getSessionData`is in the left of `page`, the session data will be the one that the page had when was presented previously. [For more details](http://haskell-web.blogspot.com.es/2013/04/more-on-session-management-in-mflow.html)
The session state can be made persistent. for example, if a timeout occurs or the server shut dowm, the, session state can be logged with `step`. For an explanation of this go th the next tutorial: [The MFlow DSL](https://www.fpcomplete.com/user/agocorona/MFlowDSL)