# Part 1

## Introduction

This tutorial is the first in a series of 4 tutorials on how to build a file 
hosting service using Yesod. Site visitors will be
able to upload collections of files, and later download those same files to a
different computer. We'll touch on issues you will likely encounter when
writing any web app such as HTML generation, styling, maintaining application
state, and managing form data.

The main goal of the tutorials are to show you how to get started
writing web applications using Haskell and the Yesod Web Framework. Each entry
will build on the last as we write a complete application from start to
finish.

The Yesod Web Framework is a complete package for developing web
applications. Yesod projects execute as an HTTP server that can be self-hosted
or interface with an existing server using methods such as FastCGI. A unique
system of mapping URL paths to handler functions make it so that broken links
trigger a compile-time error. Several miniature domain-specific languages aid
in the making of persistence layers, routing systems, and templates. Fragments
of HTML, CSS, and JavaScript can be packaged into components called
widgets. These can be combined with each other efficiently so that
clients receive a single copy of everything they need.

Many of Yesod's features will be left out of discussion. One of these is
Yesod's built-in support for automated testing. Another is Yesod's ability to
run applications such that templates are automatically reloaded at
runtime.

## Conventions

As we go I will follow a set of conventions in how examples appear,
programming constructs are referred to, and new material is presented. Being
aware of these conventions from the beginning may help you follow along.

When I refer to the name of a function, data type, or class which is found in
within the libraries supported by FP Complete, it will often be hyperlinked to
documentation. Examples might be the <hoogle>lookup</hoogle> function from the
<hoogle>Data.List</hoogle> module. When I quote a short expression of code,
refer to a name defined within the project, or otherwise embed a programming
language fragment into the English prose, it will appear `like this`.

These tutorials are written using the FP School of Haskell learning site. 
Most of the provided code samples will be executable. These will have the usual FP 
School of Haskell "play" button in the upper-right hand corner. 
You are encouraged to experiment with these example applications, 
and to execute the modifications that you make. Over the course of this 
series the project will grow, so in each new section we will only
show files where changes have occurred. You will always be able to see the
hidden code by clicking on buttons which will appear in the upper-right hand
corner of the example.

Pay special attention to highlighted blocks within the examples. These are the
lines which have been changed or added. Comments represent lines of code as
they were in a previous example.

# Minimal Example

Let's start by writing a minimal web site that serves a single page of static
HTML. I'll walk you through the code line-by-line. Later on we will break this
application into several files, but everything will fit into the same pattern.

``` active haskell web
{-# START_FILE Main.hs #-}
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE QuasiQuotes #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE TypeFamilies #-}

module Main where

import Yesod

data App = App
instance Yesod App

mkYesod "App" [parseRoutes|
/ HomeR GET
|]

getHomeR :: Handler Html
getHomeR = defaultLayout $ do
    setTitle "File Processor"
    toWidget [whamlet|
<h2>Previously submitted files
|]

main :: IO ()
main = warpEnv App
```

## Language extensions

``` haskell
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE QuasiQuotes #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE TypeFamilies #-}
```

The first block of text is a collection of compiler pragmas declaring that
several language extensions will be used. We will briefly describe their
purpose here. See the GHC user guide for more information.

### OverloadedStrings

Overloaded strings allows us to use string literals where we would not
otherwise be able to. Take a look at the line reading, `setTitle "File
Processor"`. The <hoogle search="Yesod">setTitle</hoogle> action expects the
type of its argument to be <hoogle search="Text.Blaze.Html">Html</hoogle>, and
we've supplied a string literal. Standard Haskell would fix the type of our
string to `[Char]` or `String`, and report a type error.

Using overloaded strings is not required, but frees us from having to manually
convert string literals to the types we want. Without this extension we would
have needed to import the `Data.String` module and set the page title with,
`setTitle . fromString $ "File Processor"`.

We recommend trying this out as an exercise. Disable the OverloadedStrings
extension, and read the compiler error. Once you have done this,
change the line starting with "setTitle" so that the program will run. A
solution is included below.

@@@
``` active haskell web
{-# START_FILE Main.hs #-}
{-# LANGUAGE QuasiQuotes #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE TypeFamilies #-}

module Main where

import Data.String
import Yesod

data App = App
instance Yesod App

mkYesod "App" [parseRoutes|
/ HomeR GET
|]

getHomeR :: Handler Html
getHomeR = defaultLayout $ do
    setTitle . fromString $ "File Processor"
    toWidget [whamlet|
<h2>Previously submitted files
|]

main :: IO ()
main = warpEnv App
```
@@@

### TemplateHaskell and QuasiQuotes

The `TemplateHaskell` language extension enables compile-time
metaprogramming. Yesod uses this in a number of places to reduce and simplify
the amount of code we need to write. Much of our application's boilerplate
will be written for us.

Any function calls made from the top level of a source file are examples of
Template Haskell in use. The line reading `mkYesod "App" ...` is one such
place. We will describe what this line does later on.

The `QuasiQuotes` language extension is an addition to Template Haskell which
allows us to embed domain specific languages directly into Haskell source
files. Yesod includes several of these small languages, and our first example
uses 2 of them. The extension gets its name from the `[|` and `|]` tokens that
surround it. I'll highlight this extension's usage below:

``` haskell
mkYesod "App" {-hi-}[parseRoutes|
/ HomeR GET
|]{-/hi-}

getHomeR :: Handler Html
getHomeR = defaultLayout $ do
    setTitle . fromString $ "File Processor"
    toWidget {-hi-}[whamlet|
<h2>Previously submitted files
|]{-/hi-}
```

### TypeFamilies

The `TypeFamilies` language extension is used internally by Yesod. We will not
be using it directly in this tutorial, but the extension will need to be
enabled in a few modules.

## Module and Import Declarations

``` haskell
module Main where

import Yesod
```

Our hello world application includes everything in a single file. When we
expand on it I will show you the recommended way to organize a Yesod
project. The Yesod module exports most of the basic functionality for
writing a Yesod application.

## Defining the Foundation Type

``` haskell
{-hi-}data App = App{-/hi-}
instance Yesod App
```

All Yesod sites contain a single "foundation" type. This type is used to hold
references to any state that the site needs to maintain. The type is
constructed once in the application's `main` function, and remains in memory
until the program terminates. Since our example has no state, the `App` constructor has no fields.

## Instantiating the Yesod Typeclass

``` haskell
instance Yesod App
```

Our `App` data type has been made an instance of the `Yesod` type class. We
can now supply an `App` to functions which accept instances of Yesod as an
argument.

The Yesod type class includes many methods. They all contain default
implemantations that can be overridden. By writing `instance Yesod App`,
we've declared that all of the default methods should be used.

## Automatically Generating Boilerplate.

``` haskell
mkYesod "App" [parseRoutes|/ HomeR GET|]
```

The <hoogle>mkYesod</hoogle> template haskell function generates
most of the boilerplate needed to tie our `App` data type and the
<hoogle>Yesod</hoogle> type class together into a complete application. The
first argument is the name of our data type as a literal string. The second
argument is a declaration of our site's routes. Everything between
"[parseRoutes|" and "|]" is not Haskell, but a miniature language for
declaring how URLs should map to handler actions. The string, "/ HomeR GET",
says that our site will listen for HTTP GET requests at the root path, and
will pass those requests to a handler function named getHomeR.

Yesod's <hoogle>mkYesod</hoogle> Template Haskell function
generates data types and type aliases that are used by the rest of our
application. One such type alias is `Handler`, which is used in handler
functions.

## Writing Route Handler Actions

``` haskell
getHomeR :: Handler Html
getHomeR = defaultLayout $ do
    setTitle "File Processor"
    toWidget [whamlet|
<h2>Previously submitted files
|]
```

Any route mentioned in our <hoogle>parseRoutes</hoogle>
quasi-quoted fragment must have an associated handler function. There is a
convention for how handler functions should be named. Recall that our route
definition was `[parseRoutes|/ HomeR GET|]`. The `HomeR` fragment becomes the
name for our home page if we ever want to generate a hyperlink back to
it. Handler actions always have names starting with the HTTP verb they handle
followed by the name of the route.

In the first line we see <hoogle>defaultLayout</hoogle> being
called. This is one of the Yesod type class's methods. It operates in a
special handler monad that the call to <hoogle>mkYesod</hoogle>
generated for us. The <hoogle>defaultLayout</hoogle> method takes
a "widget" action as an argument, and produces a block of HTML to be used in
the final HTTP response.

The next 3 lines make up our widget action. The call to
<hoogle>setTitle</hoogle> specifies what ends up in our final HTML's <code>&lt;title&gt;</code>
tag. Yesod uses a templating language called Hamlet to produce HTML. This is
what you see in the block starting with `toWidget [whamlet|` and ending in
`|]`. Hamlet supports a number of features that will be useful to us later
on. For now we're just generating static HTML that will end up in our `<body>`
tag.

## Starting the Server Process

``` haskell
main :: IO ()
main = warpEnv App
```

The `main` action is where we initialize configurations, open database
connections, or perform other up-front processes. Typically our foundation
data type, `App` in this case, will hold references to anything needed during
execution.

# A Closer Look at Hamlet

Earlier on We gave you a glimps of using Hamlet to generate static HTML. We want
to end this episode by showing you a few other features of Hamlet so you will
have something to experiment with.

## Nesting

Hamlet's syntax is similar to HTML in that text is marked up with angle
bracketted tags. You may have noticed in the example given above that We did
not type a closing tag. This is because Hamlet uses whitespace for
nesting. As an example, here is how you would make an unordered list:

``` active haskell web
{-# START_FILE Main.hs #-}
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE QuasiQuotes #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE TypeFamilies #-}

module Main where

import Yesod

data App = App
instance Yesod App

mkYesod "App" [parseRoutes|
/ HomeR GET
|]

getHomeR :: Handler Html
getHomeR = defaultLayout $ do
    setTitle "File Processor"
    toWidget [whamlet|
<h2>Previously submitted files
{-hi-}<ul>
    <li>readme.txt
    <li>report.pdf
    <li>music.wav{-/hi-}
|]

main :: IO ()
main = warpEnv App
```

Try making a few modifications to get a feel for how nesting works. Any tag
starting at a later column than the one above it is considered a child
node. See how you might make an HTML table.

## Pulling Data from Surroundings

Blocks of Hamlet are able to include values taken from the surrounding
code. This is done using the hash curly brace syntax, `#{...}`. Imagine that
our handler action has pulled a few filenames from a database, and wants to
include it in the final HTML. Here is how you might do something like that.

``` active haskell web
{-# START_FILE Main.hs #-}
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE QuasiQuotes #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE TypeFamilies #-}

module Main where

import Yesod

data App = App
instance Yesod App

mkYesod "App" [parseRoutes|
/ HomeR GET
|]

-- show
getHomeR :: Handler Html
getHomeR = defaultLayout $ do
{-hi-}    let filename1 = "readme.txt" :: String
        filename2 = "report.pdf" :: String
        filename3 = "music.wav" :: String{-/hi-}
    setTitle "File Processor"
    toWidget [whamlet|
<h2>Previously submitted files
<ul>
$#    <li>readme.txt
$#    <li>report.pdf
$#    <li>music.wav
{-hi-}    <li>#{filename1}
    <li>#{filename2}
    <li>#{filename3}{-/hi-}
|]
-- /show

main :: IO ()
main = warpEnv App
```

Haskell's `let` syntax binds a name to a value. Any type that can be converted
to HTML can be used. I've specified that each filename is a `String`. Haskell
is often able to figure out which type to use, but in this case it is not. Try
removing ":: String" from one of the filenames to see what error you
get. It's helpful to become familiar with compiler errors.

## Looping Through a List

Later on we'll use this technique of referencing Haskell values from within a
Hamlet template to pull data from our site's global state. When that time
comes we'll want a way of looping through multiple values to produce a list of
any length. Hamlet has a few special constructs for this kind of thing. Here
is an example of looping through a list.

``` active haskell web
{-# START_FILE Main.hs #-}
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE QuasiQuotes #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE TypeFamilies #-}

module Main where

import Yesod

data App = App
instance Yesod App

mkYesod "App" [parseRoutes|
/ HomeR GET
|]

-- show
getHomeR :: Handler Html
getHomeR = defaultLayout $ do
--    let filename1 = "readme.txt" :: String
--        filename2 = "report.pdf" :: String
--        filename3 = "music.wav" :: String
{-hi-}    let filenames = ["readme.txt", "report.pdf", "music.wav"] :: [String]{-/hi-}
    setTitle "File Processor"
    toWidget [whamlet|
<h2>Previously submitted files
<ul>
$#    <li>#{filename1}
$#    <li>#{filename2}
$#    <li>#{filename3}
{-hi-}    $forall filename <- filenames
        <li>#{filename}{-/hi-}
|]
-- /show

main :: IO ()
main = warpEnv App
```

In the `$forall` command, "filename" is bound to each element of the
"filenames" list. Hamlet has several other special commands besides these.

## Conditionals

Here is how we might use the `null` command and an if-else construct for
special handling in the case of an empty list.

``` active haskell web
{-# START_FILE Main.hs #-}
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE QuasiQuotes #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE TypeFamilies #-}

module Main where

import Yesod

data App = App
instance Yesod App

mkYesod "App" [parseRoutes|
/ HomeR GET
|]

-- show
getHomeR :: Handler Html
getHomeR = defaultLayout $ do
    let filenames = [] :: [String]
    setTitle "File Processor"
    toWidget [whamlet|
<h2>Previously submitted files
$# <ul>
$#    $forall filename <- filenames
$#        <li>#{filename}
{-hi-}$if null filenames
    <p>No files have been uploaded yet.
$else
    <ul>
        $forall filename <- filenames
            <li>#{filename}{-/hi-}
|]
-- /show

main :: IO ()
main = warpEnv App
```

Try adding an entry or two into the `filenames` list to verify that the
`$else` branch is executed in that case.

# Summary

In this episode we looked at a minimal Yesod application, and saw how to
generate HTML using Hamlet templates. The next entry We will show you how to
organize a Yesod project.
