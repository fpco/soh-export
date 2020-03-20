# Untitled

```haskell active
{-# LANGUAGE OverloadedStrings #-}
import Text.XML.Cursor
import Text.HTML.DOM (sinkDoc)
import Network.HTTP.Client.Conduit
import Data.Conduit
import Control.Monad.IO.Class
import qualified Data.Text as T
import Data.Time
import System.Locale

main = withManager $ do
    withResponse ("http://hackage.haskell.org/package/conduit-1.1.0") $ \res -> do
        doc <- responseBody res $$ sinkDoc
        let uploadDate = fromDocument doc $// element "th" >=> hasContent "Upload date" >=> followingSibling &/ content
        liftIO $ print (parseTime defaultTimeLocale "%c" $ T.unpack $ T.concat uploadDate :: Maybe UTCTime)

hasContent t c =
    if T.concat (c $// content) == t
        then [c]
        else []
```