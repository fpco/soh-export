# HTML crawling

Motivated by: [http://stackoverflow.com/questions/15658067/missing-characters-using-text-regex-pcre-to-parse-web-page-title](http://stackoverflow.com/questions/15658067/missing-characters-using-text-regex-pcre-to-parse-web-page-title)

```haskell active
{-# LANGUAGE OverloadedStrings #-}
import           Data.Monoid          (mconcat)
import           Network.HTTP.Conduit (simpleHttp)
import           Text.HTML.DOM        (parseLBS)
import           Text.XML.Cursor      (attributeIs, content, element,
                                       fromDocument, ($//), (&//), (>=>))

main = do
    lbs <- simpleHttp "http://www.ted.com/talks/francis_collins_we_need_better_drugs_now.html"
    let doc = parseLBS lbs
        cursor = fromDocument doc
    print $ mconcat $ cursor $// element "title" &// content
    print $ mconcat $ cursor $// element "span" >=> attributeIs "id" "altHeadline" &// content
```