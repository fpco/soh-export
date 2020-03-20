# Factories using QuickCheck

You can use <hoogle>derive</hoogle> package to automatically derive `QuickCheck`'s <hoogle>Arbitrary</hoogle> instances.

As this snippet also shows, it may be a wise idea to incorporate some `Positive` type to not have to re-write positive int generation for every field that's actually positive.

``` active haskell
{-# LANGUAGE OverloadedStrings, TemplateHaskell #-}

{-# OPTIONS_GHC -F -pgmFderive -optF-F #-}

import Control.Applicative
import Test.QuickCheck
import Test.QuickCheck (Positive(..))
import Test.QuickCheck.Gen (unGen)
import Data.Text (Text)
import Data.String (fromString, IsString)
import System.Random (getStdGen, StdGen)
import qualified Data.Text as T

data User = User { username :: Text
                 , age :: Int
                 , gender :: Maybe Gender
                 , email :: String
                 , balance :: Balance
                 , workAddress :: Maybe Text
                 , country :: Maybe Country
                 }
          deriving (Eq, Show)

data Gender = Male | Female
            deriving (Eq, Show)

data Balance = Balance Integer Currency
             deriving (Eq, Show)

data Currency = USFishes | BitBones | GoldenIPhones
              deriving (Eq, Show)

data Country = NowhereLand | SomewhereLand
             deriving (Eq, Show)

main :: IO ()
main = do
    g <- getStdGen
    -- let user = unGen userGen g 10
    let user = buildUser g
    putStrLn $ "User is: " ++ (show user)

buildUser :: StdGen -> User
buildUser g = unGen userGen g 20

userGen :: Gen User
userGen = do
    u <- arbitrary :: Gen User
    uid <- positiveIntGen
    age <- positiveIntGen
    return $ u { username = T.append "user_" $ strPositive uid
               , email = buildEmail $ getPositive uid
               , age = getPositive age }

buildEmail :: Int -> String
buildEmail uid = "email_" ++ (show uid) ++ "@example.com"

-- utilities

positiveIntGen :: Gen (Positive Int)
positiveIntGen = arbitrary

strPositive :: (Integral i, IsString s, Show i) => Positive i -> s
strPositive i = fromString . show $ getPositive i

-- I couldn't find this
instance Arbitrary Text where
    arbitrary = fromString <$> (arbitrary :: Gen String)

{-!
deriving instance Arbitrary User
deriving instance Arbitrary Gender
deriving instance Arbitrary Country
deriving instance Arbitrary Currency
deriving instance Arbitrary Balance
!-}

```
