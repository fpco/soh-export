# Web apps that write like console apps

# My history with the web
When the web first showed up, I was delighted. Here was a tool I could use to release cross-platform apps by releasing one app. Since I was working for a multi-platform software vendor, this was great - we had people with Macs, Windows machines, and most of the available Unix workstations. Now I could write an app once and they could all use it.

So I started automating some of the things we hadn't done before because we couldn't reach the entire audience or afford to alienate those we couldn't reach. Write an HTML page or two, the code to process the input, and then write out the results, and we're done. All fun, easy and productive.

Then something **evil** happened. Web templates. Suddenly, it wasn't about writing code any more. It was about writing templates, then writing code fragments to plug values into the holes. Worse yet, most template systems broke the better web text authoring tools, at least until those tools were taught about that template language. They had the same effect on web text processing tools. Writing for the web was no longer fun, easy or productive. So I stopped.

And every time I've looked at web application tools since, it seems there's been another level of complications added to paper over the problems with template systems. Routes. Adapters. Messy config files. A simple app might have more text in config files than in code. And this is seriously considered a good thing?

## Application types
Console applications aren't necessarily easy to write. But the logic at least flows through them in a straightforward way. You evaluate expressions, and some of those trigger user interactions. With a web template, you're never sure when the fragments that plug things in will get evaluated. Unless, of course, the web template system makes guarantees about that. Most don't. This makes the code fragile. Again, not fun, easy or productive.

Of course, a typical graphical desktop application has many of the same problems. You provide a user interface, and then connect code fragments to it that interface elements cause to run. It's a bit more predictable than the web interface, because the fragments are controlled by UI elements instead of plugging into a template. But it's still more painful than a console application.

## [MFlow](http://mflowdemo.herokuapp.com/)
I recently ran into MFlow, and for the first time in a long time found myself wanting to write a web application.  MFlow makes web applications write like console applications - you evaluate expressions that trigger user interactions. Except they happen in the browser, not a terminal window.

MFlow leverages Haskell's *do* syntax to make the web interactions happen at the right time. In particular, what's been called the "programmable semicolon" nature of that syntax.

# Example application
To show how this works, I wanted to use a simple application, so I chose a very simple game. It's known by a number of names, but the rules are easy. You start with a pile of matches, and alternate turns taking either one or two matches. The player that takes the last match wins. Which means a game - its complete state - can be represented by a single integer.

## Not production quality
Note that this code is **not** production quality code. I've left out any kind of error checking that would obscure the code, haven't done anything to make it pretty, and in general kept it as short and simple as possible. I have tried to keep it idiomatic, though.

# The game
The code below extends the `Game` state to have `Lost`/`Won`/`Illegal` indicators, the latter used when someone makes an illegal move. The functions just update a game with a move, finds the computers next move, provide an English description of a move, and of course tie those together to handle everything that happens between the human player making a move and being prompted for their next move. All completely independent of any actual interface code.

```haskell
module Game (Game (..), move, prompt) where

data Game = Illegal | Won | Lost | Game Int deriving (Show)
type Move = Int

-- Create a prompt for the current game and message.
prompt :: String -> Int -> String
prompt m l = m ++ "There are " ++ show l ++ " matches. How many do you take? "

-- Given a game and a move, provide a description for the move.
describeMove :: Game -> Move -> String
describeMove g m =
    case g of
        Won      -> "You won. "
        Lost     -> "I took " ++ show m ++ " and you lost. "
        Illegal  -> "You can only take 1 or 2 matches. Taking the last match wins the game. "
        (Game _) -> "I took " ++ show m ++ ". "

-- Given a game and a move, return the Game resulting from the Move.
makeMove :: Game -> Move -> Game
makeMove g@(Game l) m | m /= 1 && m /= 2 = Illegal
                      | m >= l           = Won
                      | otherwise        = Game $ l - m

-- Given a Game, find the best move for it
findMove :: Game -> Move
findMove (Game l) | l <= 2       = l
                  | rem l 3 /= 0 = rem l 3
                  | otherwise    = 1

-- Given a game and player move, calculate computer move and
-- return (message, new game)
move :: Game -> Move -> (Game, String)
move g m = case makeMove g m of
               Illegal -> (g, describeMove Illegal 1)
               Won     -> (Won, describeMove Won 1)
               Lost    -> (undefined, "Can't happen! ")
               g'      -> let m' = findMove g' in
                              case makeMove g' m' of
                                  Won  -> (Lost, describeMove Lost m')
                                  Lost -> (undefined, "Can't happen! ")
                                  g''  -> (g'', describeMove g'' m')
```

# Console interface
This being a very simple game, the interface is also simple. Just loop printing how many matches are left in the game and then get a move from the user. The code is below, and runnable at the [FP Complete School of Haskell](http://):

``` active haskell
{-# START_FILE Game.hs #-}
-- /show
module Game (Game (..), move, prompt) where

data Game = Illegal | Won | Lost | Game Int deriving (Show)
type Move = Int

-- Create a prompt for the current game and message.
prompt :: String -> Int -> String
prompt m l = m ++ "There are " ++ show l ++ " matches. How many do you take? "

-- Given a game and a move, provide a description for the move.
describeMove :: Game -> Move -> String
describeMove g m =
    case g of
        Won      -> "You won. "
        Lost     -> "I took " ++ show m ++ " and you lost. "
        Illegal  -> "You can only take 1 or 2 matches. Taking the last match wins the game. "
        (Game _) -> "I took " ++ show m ++ ". "

-- Given a game and a move, return the Game resulting from the Move.
makeMove :: Game -> Move -> Game
makeMove g@(Game l) m | m /= 1 && m /= 2 = Illegal
                      | m >= l           = Won
                      | otherwise        = Game $ l - m

-- Given a Game, find the best move for it
findMove :: Game -> Move
findMove (Game l) | l <= 2       = l
                  | rem l 3 /= 0 = rem l 3
                  | otherwise    = 1

-- Given a game and player move, calculate computer move and
-- return (message, new game)
move :: Game -> Move -> (Game, String)
move g m = case makeMove g m of
               Illegal -> (g, describeMove Illegal 1)
               Won     -> (Won, describeMove Won 1)
               Lost    -> (undefined, "Can't happen! ")
               g'      -> let m' = findMove g' in
                              case makeMove g' m' of
                                  Won  -> (Lost, describeMove Lost m')
                                  Lost -> (undefined, "Can't happen! ")
                                  g''  -> (g'', describeMove g'' m')
{-# START_FILE Main.hs #-}
-- show
module Main where

import Game

-- loop that actually plays the game.
play :: Game -> String -> IO String
play g@(Game l) m = do
    putStrLn $ prompt m l
    x <- fmap read getLine
    case move g x of
        g'@(Game _, _) -> uncurry play g'
        (_, m')        -> return m'

-- Main entry: play the game and announce the results
main :: IO ()
main = do
       m <- play (Game 20) "Hello. "
       putStrLn m
```

The `play` function has the obvious structure: we print (with `putStrLn`) the prompt for this move. Then read a line from the (with `getLine`) and use `read` to convert it to an integer. We run the `move` function from the `Game` module to get a message and new game after applying the user and computer moves. If that's still of the form `Game n`, then the game isn't over, so we loop and do it again. Otherwise, we return the message to `main`. `main` is likewise straightforward: run the `play` function with a greeting to get a string describing the results, then print the results.

# Web interface
If you write web apps - especially if you do it in Haskell - you might consider writing up this web app in your favorite framework. If you do it in Haskell, feel free to use my `Game` module! I haven't done it, because I probably couldn't escape claims of biasing the results. And besides, I'm lazy. If you do this, please provide us with a link to or a copy of your code so others can look at it!

Ok, the web application uses the same basic structure. The code is below, and [runnable](http://) at the FP Complete School of Haskell. Unfortunately, there seems to be a bug in the older version of MFlow there. While things work fine in the current version, they seem to loop improperly at th SoH.

``` active haskell web
{-# START_FILE Main.hs #-}
-- show
module Main where

import MFlow.Wai.Blaze.Html.All

import Game

-- loop that actually plays the game.
play :: Game -> String -> View Html IO String
play g@(Game l) m =
    (toHtml (prompt m l) ++> br ++> getInt Nothing <! [("autofocus", "1")])
    `wcallback` \x ->
        case move g x of
            g'@(Game _, _) -> uncurry play g'
            (_, m') -> return m'
-- Main entry: play the game and announce the results
main :: IO ()
main = runNavigation "" . step $ do
    m <- page $ play (Game 20) "Hello. "
    page $ toHtml m ++> wlink () << " Another game?"
{-# START_FILE Game.hs #-}
-- /show
module Game (Game (..), move, prompt) where

data Game = Illegal | Won | Lost | Game Int deriving (Show)
type Move = Int

-- Create a prompt for the current game and message.
prompt :: String -> Int -> String
prompt m l = m ++ "There are " ++ show l ++ " matches. How many do you take? "

-- Given a game and a move, provide a description for the move.
describeMove :: Game -> Move -> String
describeMove g m =
    case g of
        Won      -> "You won. "
        Lost     -> "I took " ++ show m ++ " and you lost. "
        Illegal  -> "You can only take 1 or 2 matches. Taking the last match wins the game. "
        (Game _) -> "I took " ++ show m ++ ". "

-- Given a game and a move, return the Game resulting from the Move.
makeMove :: Game -> Move -> Game
makeMove g@(Game l) m | m /= 1 && m /= 2 = Illegal
                      | m >= l           = Won
                      | otherwise        = Game $ l - m

-- Given a Game, find the best move for it
findMove :: Game -> Move
findMove (Game l) | l <= 2       = l
                  | rem l 3 /= 0 = rem l 3
                  | otherwise    = 1

-- Given a game and player move, calculate computer move and
-- return (message, new game)
move :: Game -> Move -> (Game, String)
move g m = case makeMove g m of
               Illegal -> (g, describeMove Illegal 1)
               Won     -> (Won, describeMove Won 1)
               Lost    -> (undefined, "Can't happen! ")
               g'      -> let m' = findMove g' in
                              case makeMove g' m' of
                                  Won  -> (Lost, describeMove Lost m')
                                  Lost -> (undefined, "Can't happen! ")
                                  g''  -> (g'', describeMove g'' m')
```

The `play` function looks a lot like `play` in the console version. The code to output HTML is a bit more complicated, because - well, we've got to produce a lot more text.  Wrap the prompt in HTML, provide a `br` tag. Instead of using `getLine` and `read` to get an integer, we use `getInt` and apply an `autofocus` attribute to the resulting tag. As a the final bit of IO, we use `wcallback` to extrract the integer rather than just extracting it directly, as that will erase the previous contents of the page. On the other hand, the rest of the function - not involving any user interaction - is identical between the two versions.

`main` is similar. We need a short expression to deal with running on the Web instead of in a console before the `do` code. The `play` function result is passed to a `page` so it runs in a web page. Likewise, the message gets translated to HTML, and we tack on a link to start over before handing that to `page` to display.

# Comparison
While the actual display code is a bit more complicated - we are dealing with a remote display that needs things wrapped in markup - the basic structure is still the same. `play` prompts the user, reads the result, and then loops or exits. `main` just invokes `play` and then displays its result, though the Web version has a link to play again added to it.

## Confession
While the code is pretty much idiomatic Haskell as is, I have made one change from what I'd write in order to enhance the similarity. The `do` in the console version desugars nicely, and I'd probably have used that version `main = play (Game 5) "Hello. " >>= putStrLn` if I weren't doing the comparison. The web version could be desugared, but would require an explicit lambda or an ugly transformation to point-free style, so I'd leave it as is.

# Programmable semicolons?
I mentioned "programmable semicolons". That's one of the characterizations of the `do` syntax for Haskell. Yes, there are no semicolons in this code. Like most modern languages, Haskell makes them optional at the end of a line.

For the console code, semicolons behave pretty much as you'd expect them to. For the web code, there's a pleasant surprise in store for you. You probably tried entering an illegal move - something not 1 or 2 - at some point, and noticed that you were told the rules of the game, and then prompted again. Both versions behave the same way, and that behavior is explicit in the code.

Did you try entering values that weren't valid integers? Say a `hello`? If not, do so in both now. The console version exits with an obtuse error message. I did tell you that the code wasn't production quality. The web version tells you the value isn't valid, and prompts you again. Part of that is `getInt` - it will fail to validate if the value you input doesn't `read` as an `Int` type. The other is the "programmable semicolon" behavior: if some expression fails, that step of the display gets run again instead of propagating the failure.

# Summary
I think this short demonstration illustrates nicely that MFlow allows for writing web applications with the same architecture as console applications, where that is appropriate. While setting up the display takes more work, I don't believe that can be fixed with anything that uses HTML for the display description.

On the other hand, dealing with user input is easier than a console, because all you get from a console is a string of text, whereas the MFlow framework can process it to a known type, and handles invalid input for you. In fact, if you just use `getTextBox` in a context where the inferred type is an instance of `Read`, it will work for any type. Some care must be taken if the inferred type is `Maybe a`, though, which `getInt` (and friends) avoids.

The bottom line is that it tends to balance out, and writing web apps is once again, easy, fun and productive.