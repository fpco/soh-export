# Algebraic Data Types (ADTs) with Aeson

_This is tiny cheat sheet, which will help with decoding JSON data to Haskell's ADT._

# Simple ADT

Let's start with simple example:

``` haskell
data YesNo = Yes
           | No
    deriving Show
```

First of all let's think how we could express our Haskell values in JSON format?
According to [Aeson's documentation](http://hackage.haskell.org/package/aeson-0.8.0.0/docs/Data-Aeson.html):
 
> So stick to objects (e.g. maps in Haskell) or arrays (lists or vectors in Haskell):

To avoid possible decoding pitfalls we should stick with objects and arrays.
Here are possible _(or valid)_ respresentations:

```
For Yes value: { "value" : "yes" } { "value" : 1 } { "value" : 1.0 }
     No value: { "value" : "no" }  { "value" : 0 } { "value" : 0.0 }
```

Let's declare `FromJSON` instance for our `YesNo` data type:

``` haskell
instance FromJSON YesNo where

    -- parseJSON takes a Value, it could be one of follwing data constructors:
    -- Object, Array, String, Number, Bool or Null.
    -- First of all we expect an Object, it is defined as Object !Object,
    -- where second Object is just a type synonym for HashMap Text Value. In
    -- our case we should choose somehow our Haskell value constructor
    -- according to recieved value.
    -- So, `o` is actually a HashMap, and all we need is to lookup key "type"
    -- We should use strict Text for key:
    parseJSON (Object o) = case HML.lookup (pack "type") o of
        -- value of entity has type Value
        Just (String t) -> fromString (TL.unpack (TL.fromStrict t))
        Just (Number n) -> fromNum n
        -- Other cases are invalid
        _               -> empty
        where fromString :: String -> Parser YesNo
              fromString "yes" = pure Yes
              fromString "no"  = pure No
              fromString _     = empty
              fromNum n
                  | n == 1 || n == 1.0 = pure Yes
                  | n == 0 || n == 0.0 = pure No
                  | otherwise = empty
```

Let's also declare `ToJSON` instance for our Haskell data type:

``` haskell
instance ToJSON YesNo where
    toJSON Yes = object [ "value" .= String "yes" ]
    toJSON No  = object [ "value" .= String "no" ]
```

Now we can play with it and test it:

``` active haskell
{-# LANGUAGE OverloadedStrings #-}

import Data.Aeson
import Data.Aeson.Types                          ( Parser )

import Data.Text                                 ( Text, pack )
import qualified Data.Text.Lazy as TL            ( fromStrict, unpack )
import qualified Data.ByteString.Lazy as BSL     ( toChunks )
import qualified Data.ByteString.Lazy.Char8 as C ( fromChunks, unpack )

import qualified Data.HashMap.Lazy as HML        ( lookup )

import Control.Applicative                       ( empty, pure )


data YesNo = Yes
           | No
    deriving Show
    
instance FromJSON YesNo where
    parseJSON (Object o) = case HML.lookup "value" o of
        Just (String t) -> fromString (TL.unpack (TL.fromStrict t))
        Just (Number n) -> fromNum n
        _               -> empty
        where fromString :: String -> Parser YesNo
              fromString "yes" = pure Yes
              fromString "no"  = pure No
              fromString _     = empty
              fromNum n
                  | n == 1 || n == 1.0 = pure Yes
                  | n == 0 || n == 0.0 = pure No
                  | otherwise = empty
                  
instance ToJSON YesNo where
    toJSON Yes = object [ "value" .= String "yes" ]
    toJSON No  = object [ "value" .= String "no" ]

    
-- show
main = do
    let yesJ  = encode Yes
        yesJS = C.unpack $ C.fromChunks $ BSL.toChunks yesJ
    putStrLn "Encoded:"
    print yesJS
    putStrLn "Decoding:"
    print (decode yesJ :: Maybe Value)
    print (decode yesJ :: Maybe YesNo)
```

# Complex ADT

Now, let's imagine we are designing some kind of API. Assume that we store in database
pairs of person's name and his/her cash amount.

``` haskell
data Command = NotCommand
             | WrongArg      String
             | CommandCreate { name :: Text, value :: Double }
             | CommandUpdate { id   :: Int,  value :: Double }
             | CommandDelete { id   :: Int }
    deriving Show
```

Here are valid representations of out `Command` data type:

`Create` command example

```
{ "type" : "command",
  "name" : "create",
  "data" : {
     "name" : "Arthur",
     "value" : 100.0
  }
}
```

`Update` command example

```
{ "type" : "command",
  "name" : "update",
  "data" : {
     "id" : 1,
     "value" : 90.0
   }
}
```

`Delete` command example
```
{ "type" : "command",
  "name" : "delete",
  "data" : 1
}
```

So, we expect key `type`, which always should be `"command"`, to distinguish
commands we use key `name`, and each command have third mandatory key `data`,
which differs for each command.

We could declare following `FromJSON` instance for `Command`:

``` haskell
instance FromJSON Command where
    -- First of all we lookup for mandatory key `type`
    parseJSON (Object o) = case HML.lookup "type" o of
        Just (String "command") -> let dt = HML.lookup "data" o
                                   in case HML.lookup "name" o of
            -- Then we lookup for key `name`, to distinguish commands
            Just (String "create") -> createCmd dt
            Just (String "update") -> updateCmd dt
            Just (String "delete") -> CommandDelete <$> o .: "data"
            _                      -> unrecognizedCommand
        _ -> pure NotCommand
        where createCmd Nothing           = missingData
              createCmd (Just (Object d)) = CommandCreate <$> d .: "name" <*> d .: "value"
              createCmd _                 = incorrectData
              updateCmd Nothing           = missingData
              updateCmd (Just (Object d)) = CommandUpdate <$> d .: "id"   <*> d .: "value"
              updateCmd _                 = incorrectData

              missingData         = pure $ WrongArg "Missing mandatory `data` key."
              incorrectData       = pure $ WrongArg "Incorrect data received."
              unrecognizedCommand = pure $ WrongArg "Unrecognized command name."
    parseJSON _ = pure NotCommand
```

There is nothing special about `ToJSON` instance, so let's just omit its declaration and test our code!

``` active haskell
{-# LANGUAGE OverloadedStrings #-}

import Data.Aeson

import Data.Text                                 ( Text, pack )
import qualified Data.Text.Lazy as TL            ( fromStrict, unpack )

import qualified Data.HashMap.Lazy as HML        ( lookup )

import Control.Applicative                       ( empty, pure, (<$>), (<*>) )
import qualified Data.ByteString.Lazy.Char8 as BSCL


data Command = NotCommand
             | WrongArg      String
             | CommandCreate { name :: Text, value :: Double }
             | CommandUpdate { id   :: Int,  value :: Double }
             | CommandDelete { id   :: Int }
    deriving Show

instance FromJSON Command where
    parseJSON (Object o) = case HML.lookup "type" o of
        Just (String "command") -> let dt = HML.lookup "data" o
                                   in case HML.lookup "name" o of
            Just (String "create") -> createCmd dt
            Just (String "update") -> updateCmd dt
            Just (String "delete") -> CommandDelete <$> o .: "data"
            _                      -> unrecognizedCommand
        _ -> pure NotCommand
        where createCmd Nothing           = missingData
              createCmd (Just (Object d)) = CommandCreate <$> d .: "name" <*> d .: "value"
              createCmd _                 = incorrectData
              updateCmd Nothing           = missingData
              updateCmd (Just (Object d)) = CommandUpdate <$> d .: "id"   <*> d .: "value"
              updateCmd _                 = incorrectData

              missingData         = pure $ WrongArg "Missing mandatory `data` key."
              incorrectData       = pure $ WrongArg "Incorrect data received."
              unrecognizedCommand = pure $ WrongArg "Unrecognized command name."
    parseJSON _ = pure NotCommand

instance ToJSON Command where
    toJSON NotCommand = String "Not a command"
    toJSON (WrongArg t) = String (pack t)
    toJSON (CommandCreate n v) = object [ "type" .= String "command"
                                        , "name" .= String "create"
                                        , "data" .= object [ "name" .= String n
                                                           , "value" .= toJSON v
                                                           ]
                                        ]
    toJSON (CommandUpdate i v) = object [ "type" .= String "command"
                                        , "name" .= String "update"
                                        , "data" .= object [ "id" .= toJSON i
                                                           , "value" .= toJSON v
                                                           ]
                                        ]
    toJSON (CommandDelete i)   = object [ "type" .= String "command"
                                        , "name" .= String "delete"
                                        , "data" .= toJSON i
                                        ]
-- show
main = do
    let c = encode $ CommandCreate "Svetlana" 100.0
    print (decode c :: Maybe Command)
    print (decode "{\"type\":\"command\",\"name\":\"create\"}" :: Maybe Command)
    print (decode "{\"type\":\"command\",\"name\":\" reate\",\"data\":{\"name\":\"Svetlana\",\"value\":100.0}}" :: Maybe Command)
    print (decode "{\"type\":\"command\",\"name\":\"create\",\"data\":{\" ame\":\"Svetlana\",\"value\":100.0}}" :: Maybe Command)
```
