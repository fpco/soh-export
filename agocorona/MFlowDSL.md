# MFlow as a DSL for Web applications

# Introduction
This tutorial shows some examples of the use of [MFlow](http://hackage.haskell.org/package/MFlow) as a DSL for web applications rather than as a web framework as such made of heterogeneous stuff. This tutorial modify step by step a simple application, the canonical application of stateful web frameworks, that ask for two numbers and show the result.

This initial application is modified with different applicative and monadic combinators to create from a multipage application to a single page application, AJAX driven,  with autorefresh, push, runtime templating and so on.

The same code can be combined to create very different kinds of applications under the same paradigm. MFlow is essentially a set of applicative and monadic combinators that implement tracking and backtracking effects for navigation, logging for state persistence and an extension of formlets for the user interface. This set of examples do not exhaust the variety of use cases that the model solves.

For, example, route descriptions as monadic expressions can be seen in this more basic [MFlow Introduction](https://www.fpcomplete.com/user/agocorona/MFlow-tutoria). For more complex examples see the [MFlow demos site](http://mflowdemo.herokuapp.com)

*NOTE: SOH seems to run a single web snippet per user and session. It takes certain time until it reset itself. Otherwise, a previous executed example will stay running instead of the one  you are expecting.*

*NOTE2: Since the SOH uses HTTPS, the browser reject the load of third party scripts, such is JQuery, so look at the notes of each example.*

## The initial example: Sum application with three pages  
Two pages  ask for a number.  The third shows the result and a link. Yet, the code(almost) fit in a tweet. If the link is pressed, the first page appears again, since `runNavigation` runs the navigation in a loop. The back button works.

``` haskell active web
{-# LANGUAGE OverloadedStrings #-}

import MFlow.Wai.Blaze.Html.All

main= runNavigation "" . step $ do
       n  <- page $ getInt Nothing <** submitButton "first"
       n' <- page $ getInt Nothing <** submitButton "second"
       page $ p << ( n+n') ++> wlink () "click"

```

Note that although `getInt` is a form field that expect an Int, the `<form>` tag is not explicit. It is added automatically by MFlow when necessary. However in special cases. in pages with multiple forms, `wform` can do it explicitly.

Note also that `wlink` return a typed value, () in this case. and this is what the last `page` return to `runNavigation` as it expects. `page` present a page made of forms, formatting and links and return the result.

for a more detailed explanation of the latter, read the [MFlow Introduction](https://www.fpcomplete.com/user/agocorona/MFlow-tutoria) , for example.

The operator `<**` is like the applicative operator `<*` but ever execute and present both sides, no matter if the first succeed or fail.

The operators `++>` and `<++` append and prepend rendering markup, respectively, to an active element. In this case the  rendering library is blaze-html, given by the imported module.


There are also utilities for setting up the header and the footer of the page, as well as other details that will not be shown here. I refer to the links above, with more details and more complex examples. This tutorial is about the different kinds of applications that can be created adding different combinators.


The session in MFlow  is the monadic procedure in the `FlowM` monad. The session data is in normal procedure variables. That is a big advantage over MVC frameworks, where pages do not share variable scopes and the concept of flow is inexistent. The session state has to be stored in untyped collections accessed by a key string.


## Three pages, using setSessionData

But sometimes is necessary to "transport" data among procedures, and to pass variables among multiple procedures is tedious and inelegant. in Haskell, there is the state monad. The programmer is free to use it, but it is too technical for people who want to use MFlow as a DSL to get their application easy and fast. For this purpose exist `getSessionData` and `setSessionData`. 

``` haskell active web

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

`getSessionData`, in this case, look for a tuple of two Ints . if `setSessionData` did not store two integers previously, then `onNothing` will return (0,0) . That was not the case. computeSum can be called without parameters because the two integers are in the session state. The session state is navigation-wide.

These primitives can be used in both sides of `page`. If the user press the back button, the session state may backtrack (recover its previous value) or not depending on if you read it in one side or in the other. When going back, the only code executed is the code in the right of `page`. If `getSessionData` is there, it will retrieve the last state no matter if it is going back. This is the way to keep the user-defined state actualized to the last value when pressing the back button. Otherwise, if `getSessionData`is in the left of `page`, the session data will be the one that the page had when was presented previously. [For more details](http://haskell-web.blogspot.com.es/2013/04/more-on-session-management-in-mflow.html)

## Persistent session state
The same application using persistent session state.

`step` writes a log and uses it to recover the state. if the process times out or the server shuts dowm, the state is recovered. If the flow is interrupted after asking the first number, when restarted, the application will read the first number from the log and will present the second page to the user, not the first. 

This log/recovery mechanism is used also for stopping processes after a certain timeout and restarting them on demand, in order to reduce the server load. 

The timeout is set with `setTimeouts` .This example kill the process from memory after 10 seconds of inactivity. when the user send a new request  the process is restarted and the log is used to restore it at the page that he was when stopped. But after 60 seconds of inactivity the server erases the log that contain the user session state, so the process will restart from the beginning when called again. Verify that and change the timeouts if you wish. The second timeout is an `Integer` so the session state can be kept form months or years. 0 means forever in both cases. That is the default.

*Warning: do not execute this code until the previous example executed of this tutorial dies out. Better execute it in a separate browser (an icon in the top left of the execution box permits so) and wait until SOH say that the example was killed before trying the next example.*

``` haskell active web
{-# LANGUAGE OverloadedStrings #-}

import MFlow.Wai.Blaze.Html.All

main= runNavigation "" $ do
       setTimeouts 10 60
       n  <- step. page $ getInt Nothing <** submitButton "first"
       n' <- step. page $ getInt Nothing <** submitButton "second"
       
       step . page $ p << (n+n') ++> wlink () "click"

```

The `setSessionData` and `getSessionData` example can be transformed in a analogous way adding `step` before `page`. Then the session data managed by the sessiondata mechanism will persist as well beyond the timeout. 


## Two pages, using formlets

The first page ask for the two numbers with an applicative formlet that generate a 2-tuple, using the applicative operators and adding some line breaks. The second page show the result.

*Warning: do not execute this code until the previous example executed of this tutorial dies out. Better execute it in a separate browser (an icon in the top left of the execution box permits so) and wait until SOH say that the example was killed before trying the next example.*

``` haskell active web
{-# LANGUAGE OverloadedStrings #-}

import MFlow.Wai.Blaze.Html.All
import Control.Applicative

main= runNavigation "" . step $ do
       (n1,n2) <- page $ (,) <$> getInt Nothing <++ br
                             <*> getInt Nothing <++ br
                             <** submitButton "send"
                             
       page $ p << (n1+n2) ++> wlink () "click"

```


## Single page, refreshed in each iteration, monadic formlet. 
In this case, a single page is refreshed in each iteration but the content change according with the monadic procedure inside the page, and the user responses. The input boxes and the result appear sequentially. Each step in the monadic execution is a roundtrip to the server. So the server can do whatever with the intermediate result and change the presentation as a result of that. The second input has access to the previous response. After showing the result, changing the inputs again, change the result. 

The `do` sequence end up in a `wlink` which is a widget that renders a link tag with the content of the second parameter and return his first parameter when clicked. if the link is pressed the widget/formlet return its value `()` in this case. And that is what `page` returns. Since there are no more pages, `runNavigation` will execute the flow again and will present the page with no data. 



*Warning: do not execute this code until the previous execution of this tutorial dies out. Better execute it in a separate browser (an icon in the top left of the execution box permits so) and wait until SOH say that the example was killed before trying the next example.*

``` haskell active web
{-# LANGUAGE OverloadedStrings #-}

import MFlow.Wai.Blaze.Html.All

main= runNavigation "" .step . page . pageFlow "s" $ do
       n  <- getInt Nothing   <** submitButton "first"  <++ br
       n' <- getInt (Just n)  <** submitButton "second" <++ br
       p << (n+n') ++> wlink () "click to repeat"

```


## Single page, refreshed in each iteration, monadic formlet, factoring out the button
Now a single button is presented out of the monadic computation of the page flow. 

The combinator that compose both is `<**`. It is like the applicative `<*` but ever execute and present both sides, no matter if the first succeed or fail. The first  term is between commented parenthesis (really they are not necessary)

The operator `<**` return the result of this first parameter. But the first term end with `noWidget` which is a dumb, invisible widget that ever fails. So the navigation does not progress from that page on. Every invocation of the page with any parameter en up in the refreshing of the same page again and again with new sum results.

*Warning: do not execute this code until the previous example executed of this tutorial dies out. Better execute it in a separate browser (an icon in the top left of the execution box permits so) and wait until SOH say that the example was killed before trying the next example.*

``` haskell active web
{-# LANGUAGE OverloadedStrings #-}

import MFlow.Wai.Blaze.Html.All

main= runNavigation "" . step . page . pageFlow "s" $ 
--   (
     do
       n  <- "First "  ++> getInt Nothing   <++ br
       n' <- "Second " ++> getInt (Just n)  <++ br
       p  << (n+n') ++> noWidget
--   )
      <** br ++> pageFlow "button" (submitButton "submit")

```

## Single page, refreshed in each iteration, using `wcallback` instead of monadic composition

While monadic sequences present elements is sequence by adding new rendering below the one generated by previous lines of code, `wcallback` erase the previous rendering and present only the new rendering. 

To figure out how it works, press the back button and look at what happens. Each wcallback perform a rountrip to the server and generates new rendering.

Note that `wcallback` has more priority than `++>` so  the latter takes as parameter only the textboxes, not the thext before them. So only these fields are erased: the `getInt` fields is erased and is substituted by the result. To erase the text too, enclose the text and the input field together in a parenthes. (Try it)

*Warning: do not execute this code until the previous example executed of this tutorial dies out. Better execute it in a separate browser (an icon in the top left of the execution box permits so) and wait until SOH say that the example was killed before trying the next example.*

``` haskell active web

{-# LANGUAGE OverloadedStrings #-}

import MFlow.Wai.Blaze.Html.All

main= runNavigation "" .step . page . pageFlow "s" $ 
        "the first number is "  ++> getInt Nothing 
       `wcallback` \n  -> b << n 
                      ++> br 
                      ++> "the second number is " 
                      ++> getInt Nothing
       `wcallback` \n' -> b << n'
                      ++> br 
                      ++> "the sum is "
                      ++> b << (show $ n+n')
                      ++> br 
                      ++> wlink () "one more time"
       
```

You can also press the back button and see what happens.



Run this and see what happens:

*Warning: do not execute this code until the previous example executed of this tutorial dies out. Better execute it in a separate browser (an icon in the top left of the execution box permits so) and wait until SOH say that the example was killed before trying the next example.*

``` haskell active web

{-# LANGUAGE OverloadedStrings #-}

import MFlow.Wai.Blaze.Html.All

main= runNavigation "" . step . page . pageFlow "s" $ sums
 where

 getInt'= getInt Nothing <! [("size","5")]
 
 sums= do
      n <- wlink () "one more time"
           `wcallback` const ("the first number is: "  
            ++> getInt' 
                `wcallback` (\n  -> (((b << n) ++> return n) <++ br))) 
                      
                      
      n' <- "the second number is: "  
             ++> getInt' 
                `wcallback` \n  -> b << n ++> return n <++ br 
                     
      br ++> "the sum is "
         ++> b <<  (n+n' :: Int)
         ++> br 
         ++> return ()
         <++ br
      sums


```

`sums` is a loop 'within' a page, (in the `View` monad). The parentheses in the first statement are not necessary (except the outer one). they are there in order to illustrate about the priorities of the operators. The second statement is the same as the previous so they can be factorized.

Note that `return` here work as a widget that return its value inmediately.

The page is fully refreshed in each iteration. It is possible to avoid full page refreshes using `appendUpdate`, `autoRefresh` or `prependUpdate`. 

## Single page, monadic, autoRefreshing.

No page refresh occur in ths case. The form under `autoRefresh` is refreshed via AJAX (But see the note)

However if the first link is pressed, the navigation occur, since the link is outside of  `autoRefresh`. However,
Since `runNavigation` execute the navigation in a loop, the result is the presentation of the same page again.

*NOTE: SOH uses HTTP, so latest browsers reject the download of third party Javascript trough  HTTP such is the case with the JQuery code. To make the autorefreshing happens, permit mixed content in this page. (In future versions of MFlow this will be solved). To do so, click the shield on the right of the address bar in Chrome and Mozilla, or the message thar appears below in Explorer).*

If JQuery is not loaded, autoRefresh fall back to normal page refreshing and the result are shown as well.

*Warning: do not execute this code until the previous example executed of this tutorial dies out. Better execute it in a separate browser (an icon in the top left of the execution box permits so) and wait until SOH say that the example was killed before trying the next example.*

``` haskell active web
{-# LANGUAGE OverloadedStrings #-}

import MFlow.Wai.Blaze.Html.All

main= runNavigation "" .step . page $ 
     "Hi, I'm not refreshed, unless you "
     ++>  wlink () "click here" 
     <++ " in which case, navigation happens"
   
   <** (autoRefresh .  pageFlow "s" $ do
          n  <- "First "  ++> getInt Nothing   <++ br
          n' <- "Second " ++> getInt (Just n)  <++ br
          p  << (n+n') ++> noWidget
         <** br ++> pageFlow "button" (submitButton "submit"))

```
Using page flows with monadic or `wcallback` under `autoRefresh` permit highly dynamic and highly responsive applications since only the widget activated is refreshed in the browser.


## Single page, autoRefreshing. Using push to present the results
Now the results are  appended via a push widget that get every new sum generated by the autorefrehed monadic widget above. The monadic form writes the sum to a MVar. The push widget wait for the MVar to be filled and append the number to the list of results. the `push` primitive uses long polling (also called comet) to update its content in real time. In this case `Append` tell `push` to append each result to the formers ones.

*NOTE: SOH uses HTTP, so latest browsers reject the download of third party Javascript trough HTTP such is the case with the JQuery code. To make the autorefreshing happens, permit mixed content in this page. (In future versions of MFlow this has been solved). To do so, click the shield on the right of the address bar in Chrome and Mozilla, or the message thar appears below in Explorer).*

*NOTE2: In this case, not event this works. My browser keeps warning about insecure content. So I choose to make it inactive. try it in your own computer.*

``` haskell 
{-# LANGUAGE OverloadedStrings #-}

import MFlow.Wai.Blaze.Html.All
import System.IO.Unsafe
import Control.Concurrent.MVar


res = unsafePerformIO $ newEmptyMVar :: MVar Int  

main= runNavigation "" . step . page $ do
   "Hi, I' m not refreshed, unless you "
     ++>  wlink () "click here" 
     <++ " in which case, a navigation happens"
   
    <** (autoRefresh .  pageFlow "s" $ do
          n  <- "First  "  ++> getInt Nothing   <++ br
          n' <- "Second " ++> getInt (Just n)  <++ br
          liftIO $ putMVar res $ n + n' 
          noWidget 
         <** br ++> pageFlow "button" (submitButton "submit"))
         
    <** push Append 0  (do
            n <- liftIO  $ takeMVar res  
            b << (show n) ++> br ++> noWidget )

```



## Single page, only `dField` placeholders updated
*NOTE: This example does not work with the current (11/09/13) version of MFlow installed in The FP complete site.*

Here autoRefresh is not used, but no navigation happens: witerate creats a mask where only the content generated by  `dField`  is refreshed via implicit AJAX, instead of the whole page.  Here the `getInt` boxes are refreshed by `dField` in order to present their validation errors produced (when the entry is not an Int). But otherwise, the dFields for the text boxes can be eliminated (try it).  The sum result is also included in a dField that is refreshed in each iteration.

`dField` encloses the rendering of his widget parameter within a `span` tag that is updated in each interaction with the new rendering of the same widget. In this case less than a hundred bytes are interchanged between the server and the browser in each iteration and the page, scripts etc remain.



``` haskell 
{-# LANGUAGE OverloadedStrings #-}

import MFlow.Wai.Blaze.Html.All
import Data.Monoid

main= do
  userRegister "user" "user"
  
  runNavigation "" . step . page $ 
     wform wlogin **>
     ( pageFlow "s" . edTemplate "user" "template.html" .  witerate $ do
        n  <- dField(getInt Nothing) 
                <|> return 0 <++ "first"  <> br
                
        n' <- dField(getInt Nothing) 
                <|> return 0 <++ "second" <> br
                <** submitButton "OK"
      
        dField( p << (n+n') ++> noWidget))


```

Since witerate freezes the rendering of the first interaction and only the dField holes are updated, the rendering of the first interaction must be complete, so the incremental presentation of monadic rendering in the previous examples would not work. To prevent partial presentation, the two input lines return valid responses, either the user reseponse or 0. This is assured with the `<|>` operator so all the lines are executed. 

Alternatively, a simpler solution is to use the tuple example above with two pages, using applicative formlets:

``` haskell 
{-# LANGUAGE OverloadedStrings #-} 

import MFlow.Wai.Blaze.Html.All
import Data.Monoid

main= do
  userRegister "user" "user"
  
  runNavigation "" . step . page $ 
     wform wlogin **>
     ( pageFlow "s" . edTemplate "user" "template.html" .  witerate $ do
        (n,n') <- page $ (,) <$> dField(getInt Nothing) <++ br
                             <*> dField(getInt Nothing) <++ br
                             <** submitButton "send"
      
        dField( p << (n+n') ++> noWidget))


```


like `autoRefresh`, `witerate` and `dField` fall back to normal page refreshing when JavaScript is not available.

Suggested exercice: Guess what is necessary for forcing an autorefresh/navigation in this application. Yo can also add pages to the flow.

## Single page, only dField placeholders updated, editable template
*NOTE: This example does not work with the current (11/09/13) version of MFlow installed in The FP complete site.*

This example add `edTemplate` and user login management to the previous example. 

Since `witerate` and `dField` generate a mask with placeholders refreshed with AJAX, such mask can be modified, and the placeholders can be relocated, layout and extra links can be added and so on.  `edTemplate` allows the edition of the mask at runtime. in this way the programmer does not need to have to insert much detailed rendering at programming/test/integration time. Just add it at a latter time whenever it is convenient.

In this case the user *"user"* has the authorization to edit the template. This app register this user with `userRegister` at initialization time. There is also a login widget  `wlogin` that permits to login with this user/password and to logout to see how the wysiwyg editor is activated and deactivated depending on the user. The application works even when you are editing the layout!. The layout is stored in the `./texts` folder. Save your edition and the next time the application will use your layout for all the users. Adding skin functionalities and/or mobile versions is straighforward with `edTemplate`.

Take care not to erase the placeholders created by `dField`. If you want more precise edition beyond adding extra text and links, you can use the HTML edit option to add table layout or whatever.

In this case the applcation has two forms, one for logging and another for the application itself. That´s why `wform` must be used explicitly in wlogin since witerate add it automatically. This should be the case of wlogin. This will be fixed in next releases. 

``` haskell 
{-# LANGUAGE OverloadedStrings #-}

import MFlow.Wai.Blaze.Html.All
import Data.Monoid

main= do
  userRegister "user" "user"
  runNavigation "" . step . page $ 
     wform wlogin **>
     ( pageFlow "s" . edTemplate "user" "template.html" .  witerate $ do
       n  <- dField(getInt Nothing) 
                <|> return 0 <++ "first"  <> br
                
       n' <- dField(getInt Nothing) 
                <|> return 0 <++ "second" <> br
                <** submitButton "OK"
      
       dField( p << (n+n') ++> noWidget))

```

`edTemplate` can be used without `witerate`. It makes sense for textual pages with links or input fields where there is nothing dynamic to present. The page can be enriched with more content and layout even at exploitation time. wrap your page or element of page in a `edTemplate` tag and you will let the designers, editors and even javascript programmers to have freedom to evolve the presentation.

`edTemplate` also works well when JavaScript is disabled. No edition is possible in this case of course, but the rendering is right. 
