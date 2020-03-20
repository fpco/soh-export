# Hamlet escaping

```active haskell
{-# LANGUAGE QuasiQuotes #-}
import Text.Hamlet
import Text.Blaze.Html.Renderer.String

main = putStrLn $ renderHtml [shamlet|
$doctype 5
<p>Hello world! {{foo}}
|]
```