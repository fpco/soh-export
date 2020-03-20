# SoH Markdown

Markdown is a system for embedding formatting information in a document in the most unobtrusive way. If you need more sophisticated formatting, you are free to embed HTML tags in Markdown as well, but that's rarely necessary.

There are many versions of Markdown, the School of Haskell uses some of extensions of the [GitHub Flavored Markdown](http://github.github.com/github-flavored-markdown/) (GFM), which is a popular extension of [basic Markdown](http://daringfireball.net/projects/markdown/syntax). I'll start with our extensions and then describe the supported extensions from GFM.

## SoH Extensions

### Active and Passive code

The most important extension is the ability to embed runnable and editable Haskell code in your content. The syntax for that is an extension of GFM _fenced code_ and it looks like this:

    ``` active haskell
    main = putStrLn "You can edit me and run!"
    ```
and is displayed like this:

``` active haskell
main = putStrLn "You can edit me and run!"
```

Make sure fenced code is always preceded by an **empty line** and that the three backticks start in the **first column**. The word `haskell` must be lower-case.

If you skip the word `active` you will get the standard GFM fenced code, as in:


    ``` haskell
    main = putStrLn "You can neither edit me nor run!"
    ```
which is displyed as:


``` haskell
main = putStrLn "You can neither edit me nor run!"
```

Notice though that even passive code has added SoH functionality: When you click on the function `putStrLn` inside the code snippet, we will display the type signature and the documentation for it. This works for all Haskell library functions.

If you just use the fence without specifying a language, the text will be displayed as pre-formatted, in accordance with GFM:


```
main = putStrLn "I'm not even syntax-highlighted"
```

### Hoogle links

You can also explicitly embed Hoogle links in the text, as in:

    ...higher order functions like <hoogle>map</hoogle>
    
When you click on such a link, a Hoogle documentation page for it will be displayed. Try it:

...higher order functions like <hoogle>map</hoogle>.

In the hoogle tag you may narrow the search to a particular library with `<hoogle search="Prelude.map">` or to a specified number of items with `<hoogle results="1">`. You may also specify the text that is shown when the user hovers over the link, `<hoogle title="Check the signature">`.

BTW, just like in GFM, you embed _inline code_ using backticks. If you write \`main\`, it will be displayed as `main`

### Show fragments

Sometimes you want to expose only a fragment of a complete program. Here's how you do it:


    ``` active haskell
    -- show
    sq x = x * x
    -- /show
    main = putStrLn $ "sq of 12 = " ++ (show $ sq 12)
    ```
Only the lines between `show` directives will be shown, but the whole program will be executed when run.

``` active haskell
-- show
sq x = x * x
-- /show
main = putStrLn $ "sq of 12 = " ++ (show $ sq 12)
```

You can show more than one fragment of code and you can add a comment after show, as in `-- show Print the result`

``` active haskell
-- show Type signature
sq :: Num a => a -> a
-- /show
sq x = x * x
-- show Print the result
main = putStrLn $ "sq of 12 = " ++ (show $ sq 12)
-- /show
```

### Multi-File Snippets

A snippet may combine several files. These files will be brought to the server, compiled and run. As you can see in the example below, you may define modules in separate files and import one into another. You may also add text or even binary files and images that have been uuencoded.

    ``` active haskell
    {-# START_FILE Upcase.hs #-}
    module Upcase (upcase) where
    import Data.Char
    upcase str = map toUpper str
    {-# START_FILE Main.hs #-}
    import Upcase
    main = do
        contents <- readFile "foo.txt"
        putStrLn $ upcase contents
    {-# START_FILE foo.txt #-}
    Two monads walk into a bar...   

    ```


``` active haskell
{-# START_FILE Upcase.hs #-}
module Upcase (upcase) where
import Data.Char
upcase str = map toUpper str
{-# START_FILE Main.hs #-}
import Upcase
main = do
    contents <- readFile "foo.txt"
    putStrLn $ upcase contents
{-# START_FILE foo.txt #-}
Two monads walk into a bar...   

```

### Highlights

To highlight a fragment of Haskell code enclose it between {-hi-} and {-/hi-}:


    ``` haskell
    map {-hi-}(\(x, y)-> (x + y, x - y)){-/hi-} $ zip [1, 2, 3] [3, 2, 1]
    ```


``` haskell
map {-hi-}(\(x, y)-> (x + y, x - y)){-/hi-} $ zip [1, 2, 3] [3, 2, 1]
```


### Hide solution


This feature is very useful when you embed exercises in your tutorial. You put the solution between triple at sign `@@@` fences:

    @@@
    ``` active haskell
    main = putStrLn (show 43)
    ```
    Note: It is not clear what the question was.
    @@@
The reader has to press the "Show" button to see it:

@@@
``` active haskell
main = putStrLn (show 43)
```
Note: It is not clear what the question was.
@@@


### Active Web

In our Yesod tutorials you might notice code samples that, when run, start a web server. The pages served by that server are displayed in a frame. This is accomplished by adding `web` to the active code markdown:

    ``` active haskell web

## Elements of GFM

### Paragraphs

An empty line is interpreted as a paragraph separator.

### Headings

Use one hash mark for level one headings, two for level two, and so on. For instance, the section title above is a level three heading:

    ### Headings

Headings are used to create a table of contents for your tutorial.

### Lists

Create ordered lists by preceding every item with a number, a period, and whitespace. Nested list are indented by four spaces.

    1.  No encounters
    2.  Encounters
        1. First kind
        2. Second kind

This is what they look like:

1.  No encounters
2.  Encounters
    1. First kind
    2. Second kind
    
For unordered lists use either a minus or a plus sign or an asterisk:

    - People who like lists
    - People who don't like lists
        - explode on seeing a list
        - close their eyes and sulk

Resulting in:

- People who like lists
- People who don't like lists
    - explode on seeing a list
    - close their eyes and sulk

### Block quotes

To quote a block of text, precede each line with the `>` sign. For instance:

    > Quo usque tandem abutere, Catilina, patientia nostra? 
    > Quam diu etiam furor iste tuus nos eludet? 
    > Quem ad finem sese effrenata iactabit audacia?

is displayed as:

> Quo usque tandem abutere, Catilina, patientia nostra? 
> Quam diu etiam furor iste tuus nos eludet? 
> Quem ad finem sese effrenata iactabit audacia?


### Links

This is the syntax for in-line links:

    Visit the [FP Complete web site](http://fpcomplete.com).
With the following result:

Visit the [FP Complete web site](http://fpcomplete.com).

### Images

You can embed external images using the link syntax with an exclamation mark in front:

```
![pączki](http://upload.wikimedia.org/wikipedia/commons/thumb/d/de/HomePaczki.jpg/320px-HomePaczki.jpg  "pączki")
```

![pączki](http://upload.wikimedia.org/wikipedia/commons/thumb/d/de/HomePaczki.jpg/320px-HomePaczki.jpg  "pączki")

or you can drag and drop your local images.

### Videos

You can also embed youtube videos. You'll need to get the *id* of the video from
[youtube](http://youtube.com/). This can be found by opening the context menu of the video and selecting `Copy video URL`. You'll get a URL similar to 'http://www.youtube.com/watch?v=ZHSBwlm5C8U'. The value of the id is the string after the `v=`. Just copy that into a `youtube` tag like so:

```
<youtube>ZHSBwlm5C8U</youtube>
```

To get this result:
<youtube>ZHSBwlm5C8U</youtube>

### Emphasis

To italicize text, surround it with single underscores (or asterisks), as in `_important stuff_`, which shows as _important stuff_. Double the underscore (or asterisks) to make it bold, as in `**We boldly go!**`, which shows as **We boldly go!**.

### Inline code

To embed code fragments in-line, surround them by backticks, as in \`sinkShip\`, which is displayed as `sinkShip`. 
