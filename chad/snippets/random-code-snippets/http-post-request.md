# POST request

```haskell active web
{-# LANGUAGE TypeFamilies, QuasiQuotes, MultiParamTypeClasses,
             TemplateHaskell, OverloadedStrings #-}
import Yesod

data HelloWorld = HelloWorld

mkYesod "HelloWorld" [parseRoutes|
/ HomeR GET POST
|]

instance Yesod HelloWorld

getHomeR :: Handler RepHtml
getHomeR = defaultLayout [whamlet|
                             <form method=post>
                                 <button>POST
                         |]

postHomeR :: Handler RepHtml
postHomeR = defaultLayout [whamlet|This is a post|]

main :: IO ()
main = warpEnv HelloWorld
```