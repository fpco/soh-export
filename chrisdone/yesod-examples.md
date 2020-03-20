# Yesod examples

## Trivial rest API


``` active haskell web
{-# LANGUAGE ScopedTypeVariables #-}
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE QuasiQuotes       #-}
{-# LANGUAGE TemplateHaskell   #-}
{-# LANGUAGE TypeFamilies      #-}
import           Data.Text (Text)
import           Yesod
import qualified Data.Map as M

data App = App

mkYesod "App" [parseRoutes|
/ HomeR GET
|]

instance Yesod App

getHomeR :: Handler TypedContent
getHomeR = selectRep $ do
    provideRep $ return
        [shamlet|
            <p>Hello, my name is #{name} and I am #{age} years old.
        |]
    provideRep $ do
        value <- requireJsonBody
        let name :: Maybe Text = M.lookup ("name" :: Text) value
        return $ object
          [ "name" .= name
          , "age" .= age
          ]
  where
    name = "Michael" :: Text
    age = 28 :: Int

main :: IO ()
main = warpEnv App

```