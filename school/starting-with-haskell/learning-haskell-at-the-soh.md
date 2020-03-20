# Learning Haskell at the SoH

The main page of the School of Haskell contains a list of tutorials, courses, and articles that are carefully picked by us. If you're new to Haskell, you might want to look into one of the beginner courses. We also have some in-depth courses and a changing selection of recent articles.

## Active Haskell

What is different about our tutorials and articles is that authors are able to embed active Haskell snippets in them. If you see a snippet like this:

``` active haskell
factorial 0 = 1
factorial n = n * factorial (n - 1)

main = print (factorial 4)
```

you may click on the **Run** button and the code will be executed right there on the spot. Try it! 

Being able to see the code running is fun, but real learning begins when you start modifying code. Try editing something inside the snippet. The safe edit would be to change the number 4 to something else. Try it and run the resulting program. The answer changes! If you make a mistake, the code won't compile and you will see a compiler error.

## Hidden Gems

Many authors include exercises in their content. They often provide a skeleton of the code you're supposed to modify as well as the solution to the exercise. But for those who want to go through the process of solving the exercise (a highly recommended practice), seeing the solution would spoil the pleasure. That's why solutions are usually hidden from view. You can reveal the hidden text by pressing the down arrow.

### Exercise

This definition of `factorial` doesn't work for negative numbers. Fix it so that a runtime error is emitted when `factorial` is called with an invalid argument.

``` active haskell
factorial 0 = 1
factorial n = n * factorial (n - 1)

main = print (factorial (-2)) -- will cause stack overflow
```
@@@ Show solution
``` active haskell
factorial 0 = 1
factorial n = 
   if n > 0 
     then n * factorial (n - 1)
   else
     error "Factorial of a negative number!"

main = print (factorial (-2))
```
@@@

## Active Web

Sometimes you will see web snippets, like this one, written in Yesod:

``` active haskell web
{-# LANGUAGE TypeFamilies, QuasiQuotes, MultiParamTypeClasses,
             TemplateHaskell, OverloadedStrings #-}
import Yesod

data MyWebSite = MyWebSite

instance Yesod MyWebSite

mkYesod "MyWebSite" [parseRoutes|
  / HomeR GET
|]

-- show
getHomeR = defaultLayout $ do
  [whamlet|
    <h2> Things To Do
    <ul>
      <li> Learn Haskell
      <li> Write a killer app
      <li> Create a startup
      <li> Go public
  |]
  toWidget [cassius|
    body
      background-color: #edf
  |]
-- /show
main = warpEnv MyWebSite
```

When you run it, you actually start a web server whose output is displayed in a frame under the snippet. Try it! As before, you can edit the snippet in place and try small modifications. 

The web frame that displays the web page you are serving has additional buttons that let you reload the site, view it in a separate window, or close the pane. 

## Your Content

At some point you will want to create your own little programs and run them. Or you might want to publish your own tutorials or articles. To do that, click the **Login** button in the NavBar. You can either login with *Persona* or *Google+*, or register an account for your username. It's very easy. Try it!