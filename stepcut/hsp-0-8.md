# HSP 0.8

HSP and hsx2hs are a pair of libraries which can be used to generate XML and HTML.

HSP 0.8 represents a major refactoring from previous versions. The essense, however, is still the same. Right now, this document just serves as a quick intro to using HSP 0.8 + hsx2hs for thoses already familiar with HSP. For more in-depth details on HSP, you can read [the HSP section of the Happstack Crashcourse](http://happstack.com/docs/crashcourse/Templates.html#hello-hsp). 

The HSP library contains types, classes, monads, and functions needed to create XML and HTML content.

The `hsx2hs` library contains a preprocessor and now also a QuasiQuoter which allow you to embed literal XML markup in your Haskell code. `hsx2hs` generates output which is compatible with HSP. While the HSP and hsx2hs libraries are designed to work in harmony, neither explicitly depends on the other. This means, for example, if you like `hsx2hs`, but not HSP, you can easily create your own library which is compatible with `hsx2hs` output.

However, here we will show how to use them together. First we will show how to use the external `hsx2hs` preprocessor:

``` haskell
{-# LANGUAGE OverloadedStrings #-}
{-# OPTIONS_GHC -F -pgmFhsx2hs #-}
module Main where

import           Data.Text.Lazy    (Text)
import qualified Data.Text.Lazy.IO as T
import HSP.HTML4        (renderAsHTML)
import HSP.Monad        (HSPT(unHSPT))
import HSP.XMLGenerator (Attr(..), XMLGen(..), XMLGenT, EmbedAsChild(..), EmbedAsAttr(..), unXMLGenT)
import HSP.XML          (XML)

html :: (Functor m, Monad m) => XMLGenT (HSPT XML m) XML
html = <p class="some">I haz a paragraph!</p>

main :: IO ()
main =
    do h <- unHSPT $ unXMLGenT $ html
       T.putStrLn $ renderAsHTML h
```

We start by enabling the `OverloadedStrings` pragma. HSP is now based around `Text`, so this will allow us to avoid having to call `pack` or `fromString` by hand for string literals.

The next line

    {-# OPTIONS_GHC -F -pgmFhsx2hs #-}

tells GHC to use the `hsx2hs` preprocessor. Note that in prior versions the preprocessor used to be called `trhsx`. When migrating old code, make sure you change that or you will get very confusing error messages.

Next we have the code that actually creates some XML using literal markup:

``` haskell
html :: (Functor m, Monad m) => XMLGenT (HSPT XML m) XML
html = <p class="some">I haz a paragraph!</p>
```

The `HSPT` monad is essentially `IdentityT` with an extra phantom type parameter. The `XMLGenT` monad is also essentially just the `IdentityT` monad and exists only to make the type-checker happy.

In many applications you will not use the `HSPT` monad directly. Instead you will add it to your application specific monad transformer stack -- or you will ignore `HSPT` entirely and create an instance of `XMLGen` for your application specific monad. 

in `main` we just unwrap the monads, and render the XML as HTML:

``` haskell
main :: IO ()
main =
    do h <- unHSPT $ unXMLGenT $ html
       T.putStrLn $ renderAsHTML h
```

The new `hsx2hs` allows us to avoid using an external preprocessor and instead use a `QuasiQuoter`. Here is the same app rewritten using the new `hsx` `QuasiQuoter`:

``` haskell
{-# LANGUAGE QuasiQuotes, OverloadedStrings #-}
module Main where

import           Data.Text.Lazy (Text)
import qualified Data.Text.Lazy.IO as T
import HSP.HTML4                   (renderAsHTML)
import HSP.Monad                   (HSPT(unHSPT))
import HSP.XML                     (XML)
import HSP.XMLGenerator            (Attr(..), XMLGenT, XMLGen(..), EmbedAsAttr(..), EmbedAsChild(..), unXMLGenT)
import Language.Haskell.HSX.QQ     (hsx)

html :: (Functor m, Monad m) => XMLGenT (HSPT XML m) XML
html = [hsx| <p class="some">I haz a paragraph!</p> |]

main :: IO ()
main =
    do h <- unHSPT $ unXMLGenT $ html
       T.putStrLn $ renderAsHTML h
```

We can now drop the crazy `OPTIONS_GHC` pragma and just add `QuasiQuotes` to the `LANGUAGE` pragma. We also import `hsx` from `Language.Haskell.HSX.QQ`.

We then use `hsx` in the html generation code:

``` haskell
html :: (Functor m, Monad m) => XMLGenT (HSPT XML m) XML
html = [hsx| <p class="some">I haz a paragraph!</p> |]
```

With the preprocessor, we could just put literal XML in the code with no special introduction. With the `QuasiQuoter` we have to explicitly delimit the XML portions.

Note that the type of the `html` function is the same, and that `main` is unaltered.

Feel free to use the preprocessor or the `hsx` QuasiQuoter. The preprocessor is nice because it does not require extra noise around the XML. It also outputs plain Haskell code -- which means it can be used with Fay and other compilers which do not support the QuasiQuotation extension.

However, it also strips all the comments from the code (which makes Haddock sad) and the line numbers in error messages are a bit fuzzy. It also requires that `hsx2hs` be in the user's path.

The QuasiQuoter version is requires a little extra noise around the XML, but generates better line numbers in error messages, leaves the Haddock comments alone, etc. Additionally, the [hsx| |] can actually make it easier for your text editor to do syntax highlighting.

Under the hood, both methods use the same code with some different glue code on top for making the transformation happen. We plan to continue supporting both methods indefinitely.






