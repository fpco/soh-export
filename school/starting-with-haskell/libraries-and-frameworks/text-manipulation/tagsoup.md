# HTML Parsing

HTML is the structured markup language used for pages on the World Wide Web. Given that it is structured, it's possible to extract information from them programmatically. Given that the schema describing HTML is viewed more as a suggestion than a rule, the parser needs to be very forgiving of errors.

For Haskell, one such parser is the `html-conduit` parser, part of the relatively light-weight `xml-conduit` package. This tutorial will walk you through the creation of a simple application: seeing how many hits we get from [bing.com](http://www.bing.com/) when we search for "School of Haskell".

## Fetching the Page

We're going to use `Network.HTTP.Conduit` to fetch the page and store it for later reference. This uses the function <hoogle search="Network.HTTP.Conduit.simpleHttp" results=1>simpleHttp</hoogle> to get the page:

``` active haskell
import Network.HTTP.Conduit (simpleHttp)
import qualified Data.ByteString.Lazy.Char8 as L

-- the URL we're going to search
url = "http://www.bing.com/search?q=school+of+haskell"

-- test
main = L.putStrLn . L.take 500 =<< simpleHttp url
```

## Finding the Data

Now that we have the page contents, we need to find the data we're interested in. Examing the page, we see that it's in a `span` tag, with the `id` of `count`.  The `html-conduit` package can parse the data for us. After doing so, we can use operators from the <hoogle search="Text.XML.Cursor">Text.XML.Cursor</hoogle> package to pick out the data we want.

`Text.XML.Cursor` provides operators inspired by the [XPath](http://www.w3.org/TR/xpath20/) language. If you are familiar with XPath expressions, these will come naturally. If not - well, they are still fairly straightforward. We extract the page as before, then use `parseLBS` to parse the lazy `ByteString` that it returns, and then `fromDocument` to create the `cursor`. The `$//` operator is similar to the `//` syntax of XPath, and selects all the nodes in the cursor that match the `findNodes` expression. The `&|` will apply the `fetchData` function to each node in turn, the resulting list being passed to `processData`.


The `findNodes` function uses `element "span"` to select the `span` tags. Then `>=>` composes that with the next selector `attributeIs "id" "count"`, which selects for - you guessed it - elements with an `id` attribute of `count`. Since `id` attributes are supposed to be unique, that should be our element. The node we want is actually the content of the text node that is a child of the node we found, so we use `child` to extract that node.

The `extractData` function uses the `content` function to extract the actual text from the node we found. Since `content` operates on a list of `Nodes`, `extractData` applies <hoogle search="Data.Text.concat">Data.Text.concat</hoogle> to turn the list of `Text`'s into a single `Text`.

Finally, we process that data - a list of the results of `extractData` - with `processData`. Since we want the text from the first element in the list we are passed, we use `head` before printing it. The resulting string has type `Text`, so <hoogle search="Data.Text.unpack">Data.Text.unpack</hoogle> turns it into a string for `putStrLn`.

``` active haskell
{-# LANGUAGE OverloadedStrings #-}

import Network.HTTP.Conduit (simpleHttp)
import qualified Data.Text as T
import Text.HTML.DOM (parseLBS)
import Text.XML.Cursor (Cursor, attributeIs, content, element, fromDocument, child,
                        ($//), (&|), (&//), (>=>))

-- The URL we're going to search
url = "http://www.bing.com/search?q=school+of+haskell"

-- The data we're going to search for
findNodes :: Cursor -> [Cursor]
findNodes = element "span" >=> attributeIs "id" "count" >=> child

-- Extract the data from each node in turn
extractData = T.concat . content

-- Process the list of data elements
processData =  putStrLn . T.unpack . T.concat

cursorFor :: String -> IO Cursor
cursorFor u = do
     page <- simpleHttp u
     return $ fromDocument $ parseLBS page

-- test
main = do
     cursor <- cursorFor url
     processData $ cursor $// findNodes &| extractData
```

Note that if you get no result, it probably means that bing has changed it's output, so the tutorial needs to be tweaked. If you get more than one result, it means the input HTML is invalid.

You can find the list of `Cursor` operators and functions along with their descriptions at <hoogle search="Text.XML.Cursor">Text.XML.Cursor.</hoogle>

## With a List

As a second example, let's extract the list of URL's from the search. These are simply `a` tags wrapped in `h3` tags.  So we change `findNodes` to find those tags, and `extractData` to fetch the `href` attribute. Finally, we process the resulting list by using `mapM_` to pass each string to <hoogle search="Data.Text.IO.putStrLn">Data.Text.IO.putStrLn<hoogle> to print each URL on a line, rather than using `unpack` to turn it into a string. This requires changing the imports a bit. In this case, rather than using a qualified import to avoid conflicts with the `Prelude`, we explicitly import it and hide the functions we want. All these changes are highlighted.

``` active haskell
{-# LANGUAGE OverloadedStrings #-}

import Network.HTTP.Conduit (simpleHttp)
{-hi-}import Prelude hiding (concat, putStrLn)
import Data.Text (concat)
import Data.Text.IO (putStrLn){-/hi-}
import Text.HTML.DOM (parseLBS)
import Text.XML.Cursor (Cursor, attribute, element, fromDocument, ($//), (&//), (&/), (&|))

-- The URL we're going to search
url = "http://www.bing.com/search?q=school+of+haskell"

-- The data we're going to search for
findNodes :: Cursor -> [Cursor]
findNodes = {-hi-}element "h3" &/ element "a"{-/hi-}

-- Extract the data from each node in turn
extractData = {-hi-}concat . attribute "href"{-/hi-}

-- Process the list of data elements
processData = {-hi-}mapM_ putStrLn{-/hi-}

cursorFor :: String -> IO Cursor
cursorFor u = do
     page <- simpleHttp u
     return $ fromDocument $ parseLBS page

main = do
     cursor <- cursorFor url 
     processData $ cursor $// findNodes &| extractData
```

# Error handling

This tutorial did not cover error handling. Given the nature of HTML, errors are common, and the html parser deals with that as well as it can. If you're using XML, then above tools will work - just use the appropriate parser from <hoogle search="Text.XML.Stream.Parse">xml-conduit</hoogle> and the tools described above. If you need to detect errors in your XML, you maight want to look at the [XML parsing with validation tutorial](https://www.fpcomplete.com/school/advanced-haskell-1/xml-parsing-with-validation).