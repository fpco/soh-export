# Method override

Related to: [http://stackoverflow.com/questions/22902419/yesod-put-and-delete-using-hidden-method-parameter/22903897](http://stackoverflow.com/questions/22902419/yesod-put-and-delete-using-hidden-method-parameter/22903897).

```haskell active web
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE QuasiQuotes       #-}
{-# LANGUAGE TemplateHaskell   #-}
{-# LANGUAGE TypeFamilies      #-}
import Yesod.Core

data App = App

mkYesod "App" [parseRoutes|
/ HomeR GET POST PUT
|]

instance Yesod App

getHomeR :: Handler Html
getHomeR = defaultLayout
    [whamlet|
        <form method=post action=@{HomeR}?_method=PUT>
            <button>This should be a PUT
    |]

postHomeR :: Handler Html
postHomeR = defaultLayout [whamlet|POST|]

putHomeR :: Handler Html
putHomeR = defaultLayout [whamlet|PUT|]

main :: IO ()
main = warpEnv App
```