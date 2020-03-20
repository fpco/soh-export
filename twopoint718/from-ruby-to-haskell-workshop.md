# From Ruby to Haskell Workshop

Project 1: sort
=============

Let's start out by trying to copy a command line utility that's well-known in UNIX-land, `sort`. If you aren't familiar with sort, it just takes the lines of a file and puts them in ASCIIbetical order. If I had a file that looked like:

    d
    a
    c
    b

and I ran sort on it, I would get:

    a
    b
    c
    d

There are a lot of other options that sort takes, but I'll make a couple of simplifying changes:

 * Sort only accepts lines on standard in (no files)
 * Sort prints directly on standard out

This is actually rather in keeping with the UNIX tradition of tools that do one simple thing well. If we want file I/O rather than STDIN/STDOUT, we can just use input redirection to wire that up. So we're not really giving anything up and we're making the interface really consistent (as you'll soon see).

Let's get started:

Enough syntax to be dangerous
--------------------------

We'll need to know a few things about haskell in order to do this project. We have to know about strings, simple I/O, lines, and how to sort things. In all the examples below, you can edit the text and then run it and play around.

### Strings

``` active haskell
myString = "Hello, World!"

main = putStrLn myString
```

I put a couple of extra things in there because otherwise it wouldn't have been very interesting to look at. I added the `main` function (which, like C, is the entry point for the program). I also used <hoogle>putStrLn</hoogle> to print out a string with a newline. I'll be using something like this in each example just to show you what's going on.

A string in Haskell is simply a list of characters:

``` haskell
"Hello, World" :: String
```

(click into "String" above to see the type signature)

### Lists

Lists are an ordered collection of the *same type* of thing. This may differ a bit from some other languages, like Ruby, where a list can be a mix of things.

``` active haskell
myNumbers = [1, 2, 3]

myStrings = ["from", "ruby", "to", "haskell"]

main = do
    print myStrings
    print myNumbers
```

(you can ignore `do` for right now, it allows me to run a bunch of actions one after the other). I used <hoogle>print</hoogle> here because it knows how to print things 

Notice that it is an error to mix types in a list:

``` active haskell
myFail = [26, "biggest number"]

main = print myFail
```

The last thing to know about lists is that you can *pattern match* on them. A list is recursively defined as a first element followed by "the rest". This is what it looks like:

``` haskell
data [] a = [] | a : [a]
```

This says that a list of some type `a` is either an empty list OR (the pipe, `|`) it is an `a` glued (with the "cons" function, <hoogle>:</hoogle>) onto the front of a list of type `a` (`[a]`). Pattern matching is where you show haskell each *case* you want to handle by giving a *series of definitions*:

``` active haskell
myCount []     = 0
myCount (x:xs) = 1 + myCount xs

main = print (myCount "chris" == 5)
```

Do you see how I matched each case? If the list is empty, the length is zero. Otherwise, if the list is made out of a head element glued onto a body, then the length is the 

#### Exercise 1

Fix the above value, `myFail` so that it works. What did you have to do? Why? (Extra credit: what does the error message mean?)

#### Solution 1

@@@
You'll have to change either 26 into a string or "biggest number" into a number. I think the first one is easier:

``` haskell
myFail = ["26", "biggest number"]
```

(unless of course you know the value of "biggest number", in that case, send me an email and tell me!)
@@@

### Sorting

Sort is easy: just hoogle it: <hoogle>sort</hoogle>. It is a library function in `Data.List`, a super-helpful collection of functions for working with lists. Take a look at the type signature:

``` haskell
Ord a => [a] -> [a]
```

Starting at the beginning, `Ord` is for things that can be put in an order. We start with a list of some type, `a`, that can be ordered and then we end up with a list of the same type that except that the elements are in order!


### Lines

Since we are sorting lines of input, how can we represent lines to make our job really easy?

#### Exercise 2

Fill in the code so that it compiles and runs (`undefined` is a function that will compile, but will fail if you try to actually use it):

``` active haskell
import Data.List (sort)

myLines = undefined

sortedLines :: [String]
sortedLines = sort myLines

main = print sortedLines
```

#### Solution 2

@@@
Use a list of strings (any strings will do):

``` haskell
myLines = ["foo", "bar", "baz"]
```
@@@

### Lines and Unlines

If you look carefully in the `Data.List` library, you'll find [lines](http://www.haskell.org/ghc/docs/latest/html/libraries/base/Data-List.html#v:lines) and [unlines](http://www.haskell.org/ghc/docs/latest/html/libraries/base/Data-List.html#v:unlines). Here's what they look like:

``` active haskell
myLines = lines "this\nhas\nnewlines"
main = print myLines
```

``` active haskell
myString = unlines ["this", "will", "have", "newlines"]
main = putStr myString
```

### A little bit of I/O

We almost have all the pieces we need, we can break on lines, sort those lines, and then join them back together. A sketch of this looks like this:


Do these steps:

1. input = getInputSomehow???
2. myLines = lines input
3. mySortedLines = sort myLines
4. mySortedString = unlines mySortedLines
5. putStr mySortedString

What would getting input look like? You may have heard Whenever we want to find Haskell functions, we can look on hoogle for the answer. Let's hoogle the type that we expect to need: <hoogle>:: IO String</hoogle>. `IO` is the special type in Haskell that means we need to talk to the outside world. Looking a bit <hoogle>getContents</hoogle> looks like a just what we wanted.

Next we need to use it to get input. For this we can use a bit of *do notation*.

``` active haskell
main = do
    userInput <- getContents
    putStrLn userInput
```

This should just echo the lines that you typed back to you.

#### Exercise 3

Print your input twice.

#### Solution 3

@@@
``` active haskell
main = do
    input <- getContents
    putStrLn input
    putStrLn input
```

Or something like that.
@@@

Putting it together
-----------------

We know everything we need. Let's translate our outline into code:

1. input = getInputSomehow???
2. myLines = lines input
3. mySortedLines = sort myLines
4. mySortedString = unlines mySortedLines
5. putStr mySortedString

``` active haskell
import Data.List (sort)

main = do
    input <- getContents
    let myLines = lines input
        mySortedLines = sort myLines
        mySortedString = unlines mySortedLines
    putStr mySortedString
```

That's it! Because of some limitations of this software, you may notice that you have no way to signal that the "input is done". But here's the code (with a minor change) working:

``` active haskell
{-# START_FILE input.txt #-}
this
is
a
text
file
{-# START_FILE Sort.hs #-}
import Data.List (sort)

main = do
    input <- readFile "input.txt"
    let myLines = lines input
        mySortedLines = sort myLines
        mySortedString = unlines mySortedLines
    putStr mySortedString
```

The only change was to replace <hoogle>getContents</hoogle> with <hoogle>readFile</hoogle>.

### Refactor

*Note* the next section gets a little bit more advanced (as refactoring always is) and I introduce some stuff that you don't really need. Follow along if you're interested :)

This works fine, but could we pull the logic of sorting out of the main function? Also there is an intermediate variable for each step; do we need that?

What about pulling out all the sorting functions. We can extract all the `let` lines because we know by their types that they have no side effects:

``` haskell
import Data.List (sort)

lines
sort
unlines
```

#### Exercise 4

Do this transformation. You'll (probably) need [let..in](http://zvon.org/other/haskell/Outputsyntax/letQexpressions_reference.html) syntax.

#### Solution 4

@@@
``` active haskell
{-# START_FILE input.txt #-}
this
is
a
text
file
{-# START_FILE Sort.hs #-}
import Data.List (sort)

sortLines input =
    let myLines = lines input
        mySortedLines = sort myLines
    in unlines mySortedLines

main = do
    input <- readFile "input.txt"
    putStr (sortLines input)
```
@@@

Next we can get rid of all the intermediate values. The primary trick here is to notice that the input of one function *exactly* matches up with the input to the next. And we can use *function composition*

``` haskell
sortLines input = (unlines . sort . lines) input
```

Lastly, notice that in `sortLines` it takes just one argument to the function chain. Using [equational reasoning](http://www.haskell.org/haskellwiki/Equational_reasoning_examples) we can simply eliminate it from both sides:

``` haskell
sortLines = unlines . sort . lines
```

putting it all together:

``` active haskell
{-# START_FILE input.txt #-}
this
is
a
text
file
{-# START_FILE Sort.hs #-}
import Data.List (sort)

sortLines = unlines . sort . lines

main = do
    input <- readFile "input.txt"
    putStr (sortLines input)
```

lastly, getting back to the standard input goal of our original project statement, we can use the library function <hoogle>interact</hoogle> which abstracts almost all the machinery of `main`. Interact takes a function from `String` to `String`, applies it to standard input, and then prints the result on standard out. Since `sortLines` is now just such a function, we can use it with interact:

``` active haskell
import Data.List (sort)

sortLines = unlines . sort . lines

main = interact sortLines
```

Unfortunately, we're back to the problem where it is hard to show in the browser because I don't know how to signal that I'm done typing input. But there you have it. A two or three-line (very simple) version of UNIX sort.

Project 2 Gravatar Downloader:
===========================

This project is a bit more advanced:

``` haskell
{-# LANGUAGE OverloadedStrings #-}

module Main where

import Data.Digest.OpenSSL.MD5 (md5sum)
import Network.Curl.Download (openURI)
import System.Directory (createDirectoryIfMissing)

import qualified Control.Monad.Parallel as MP
import qualified Data.ByteString.Char8 as B
import qualified Data.ByteString.Lazy.Char8 as BL

-- Data

type PairName = B.ByteString
type HashedPairName = B.ByteString
type Handle = B.ByteString
type Gravatar = (PairName, HashedPairName)

bendyworks, gravatarUrl, devPrefix :: B.ByteString
bendyworks = "@bendyworks.com"
gravatarUrl = "http://gravatar.com/avatar/"
devPrefix = "dev+"

subdir :: String
subdir = "./avatars/"

handles :: [Handle]
handles = [ "begriffs"
          , "bendycode"
          , "bigtiger"
          , "devn"
          , "joshuarh"
          , "listrophy"
          , "mathias"
          , "micfunk"
          , "missingdink"
          , "ryalnd"
          , "twopoint718"
          ]

avatarHashes :: [Gravatar]
avatarHashes = concatMap hashEmail allPairs
    where allPairs = pairs handles :: [(Handle, Handle)]

-- Main function

main = do
    createDirectoryIfMissing False subdir
    mapM downloadAvatar avatarHashes
    return ()

-- Support functions

hashEmail :: (Handle, Handle) -> [Gravatar]
hashEmail = map (\(a, b) -> (a, B.pack . md5sum $ b)) . formatEmail

twin x = (x, x)

-- I want to capture both permutations:
-- user1 + user2 and user2 + user1
formatEmail :: (Handle, Handle) -> [(PairName, PairName)]
formatEmail (p1, p2) = [twin $ fmt (p1, p2), twin $ fmt (p2, p1)]
    where fmt (a, b) = B.concat [devPrefix, a, "+", b, bendyworks]

avatarPath :: B.ByteString -> B.ByteString
avatarPath hash = B.concat [gravatarUrl, hash]

-- names are already sorted
pairs :: [Handle] -> [(Handle, Handle)]
pairs []     = []
pairs (x:xs) = [(x, a) | a <- xs] ++ pairs xs

downloadAvatar :: Gravatar -> IO ()
downloadAvatar (filename, hash) = do
    either_image <- openURI path
    putStrLn $ "Downloading: " ++ path
    case either_image of
      Left err -> putStrLn err
      Right image -> B.writeFile outputPath image
  where
    path = B.unpack $ avatarPath hash
    outputPath =  B.unpack $ B.concat [B.pack subdir, filename, extension]
    extension = ".jpg"
```

But I'll go through a few of the interesting points.

'pairs' function
---------------

This is where I started. I figured that I would need a list of all of the possible pairs of github handles that we use at Bendyworks. In the way that I design things, I always start with what data I need and then I try to figure out how to work backwards from that:

``` active haskell
pairs []     = []
pairs (x:xs) = [(x, a) | a <- xs] ++ pairs xs

main = putStrLn $ pairs [1, 2, 3, 4]
```

If you've worked with python you may get what's going on with the square brackets. This is a *list comprehension*, it describes how to build the list that I want. It says that for some list with a first element `x`, pair that element with each `a` (which is taken from the rest of the list, `xs`). We then recursively combine that with all of the pairs of the second element, and so on. Recursion! In the empty list, there are no pairs.

'main' function
--------------

``` haskell
main = do
    createDirectoryIfMissing False subdir
    mapM downloadAvatar avatarHashes
    return ()
```

What's cool about haskell programs, like C programs of yore, is that there's a simple `main` entry point to the program. Here is no different. When we start we create a directory called `avatars` and then we do some kind of map over all the avatar hashes, downloading each one. This already tells us everything that the program does! We just have to know about <hoogle>mapM</hoogle> and the `downloadAvatar` function.

'downloadAvatar' function
-----------------------

``` haskell
downloadAvatar :: Gravatar -> IO ()
downloadAvatar (filename, hash) = do
    either_image <- openURI path
    putStrLn $ "Downloading: " ++ path
    case either_image of
      Left err -> putStrLn err
      Right image -> B.writeFile outputPath image
  where
    path = B.unpack $ avatarPath hash
    outputPath =  B.unpack $ B.concat [B.pack subdir, filename, extension]
    extension = ".jpg"
```

This looks formidable, but it really simple. Since I'm using `do` notation, you can follow each line in the doc block sequentially. We <hoogle>openURI</hoogle> to read the page at the given url (`path` is described in the `where` clause). I print a little message saying that we're downloading it then, depending on how the `openURI` call turned out, `Left` is an error, and `Right` is the image data. If I got image data back, I write that to a file based upon the passed-in `filename`.

'Gravatar' type synonym
---------------------

I introduce a few *type synonyms* at the start of my program to organize the data that I'm using.

``` haskell
type PairName = B.ByteString
type HashedPairName = B.ByteString
type Handle = B.ByteString
type Gravatar = (PairName, HashedPairName)
```

This is a very lightweight way to provide some clarity and documentation. Haskell treats the thing on the left as being interchangeable with the thing on the right. This is like a `typedef`.

Where 'avatarHashes' comes from
----------------------------

Based upon the type signature, we know that `avatarHashes` has a type of `[Gravatar]`. Let's expand that:

``` haskell
[Gravatar] = [(PairName, HashedPairName)]
           = [(B.ByteString, B.ByteString)]
```

Two functions are used to generate those `Gravatar`s:

``` haskell
avatarHashes :: [Gravatar]
avatarHashes = concatMap hashEmail allPairs
    where allPairs = pairs handles :: [(Handle, Handle)]
```

The first is `allPairs` which generates, well, all the pairs of email addresses by calling `pairs` on the list of handles.

The rest of the functions are little helpers that I'll briefly discuss:

### hashEmail

``` haskell
hashEmail :: (Handle, Handle) -> [Gravatar]
hashEmail = map (\(a, b) -> (a, B.pack . md5sum $ b)) . formatEmail
```

This just converts a list of things like:

```
(dev+twopoint718+bigtiger@bendyworks.com, dev+twopoint718+bigtiger@bendyworks.com)
```

into a list of things like:

```
(dev+twopoint718+bigtiger@bendyworks.com, a40912d1ceb0e89380fb978b90936a7e)
```

### formatEmail

Some interesting dirty work is performed in `formatEmail`:

``` haskell
formatEmail :: (Handle, Handle) -> [(PairName, PairName)]
formatEmail (p1, p2) = [twin $ fmt (p1, p2), twin $ fmt (p2, p1)]
    where fmt (a, b) = B.concat [devPrefix, a, "+", b, bendyworks]
```

This glues together two handles into the special (and totally arbitrary) email address that we use at Bendyworks as the "author" of git commits. For each pair of handles, I generate two addresses, one where person 1 (`p1`) is first and another where person 2 (`p2`) is first. The other bit of plumbing is the tiny `twin` function:

``` haskell
twin x = (x, x)
```

which "doubles" anything into a two-tuple:

```
twin 1 == (1, 1)
```

This is because when I'm hashing email addresses, I want an unhashed "copy" to label the downloaded image with. The other simple reason is that because this is the input that `hashEmail` is expecting.

Wrapping up
----------

That should cover the how of this little program. I included in the imports the `Control.Monad.Parallel` (aliased as `MP`) package. This lets me change `main` into this:

``` haskell
main = do
    createDirectoryIfMissing False subdir
    MP.mapM downloadAvatar avatarHashes
    return ()
```

Note the `MP` qualified function name `mapM`. This lets me run all the `downloadAvatar` actions in parallel! Try this out. For me this is easily 4 times as fast (it went from about 8 seconds to about 2 on my computer).
