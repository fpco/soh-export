# Adding security to MFlow

# Improving MFlow Security

[MFlow](http://hackage.haskell.org/package/MFlow) is a Web Framework that turns Web programing back into just ordinary programming by automating all the extra complexities.

## Provided Demo

While looking at implementing a service that I wanted to be secure, I looked into the way passwords were stored in MFlow, particularly the [User Widget](http://hackage.haskell.org/package/MFlow-0.4.5.4/docs/MFlow-Forms.html#v:userWidget).  There is a [demo](https://github.com/agocorona/MFlow/blob/master/Demos/LoginSample.hs):

``` active haskell web
import Data.Monoid
import MFlow.Wai.Blaze.Html.All

main= runNavigation "" $ transientNav loginSample

loginSample= do
    userRegister "user" "user"
    r <- page  $   p <<  "Please login with user/user"
               ++> userWidget Nothing userLogin
               <|> wlink "exit" << p << "or exit"
        
    if r == "exit" then return () else do
        user <- getCurrentUser
    
        r <- page  $   b <<  ("user logged as " <>  user)
                   ++> wlink True  << p <<  "logout"
                   <|> wlink False << p <<  "or exit"

        if r
          then do
             logout
             page  $ p << "logged out" ++> wlink () << "press here to exit"
          else return ()
```

## Salting and Key Stretching

We can do better. Let's see how it can be done. SoH gives access to [pwstore-fast](http://hackage.haskell.org/package/pwstore-fast-2.4.1) which will handle the low level details for us. At first I was not aware of this library and implemented it by hand, then this made it much simpler. Usage is simple:

``` active haskell
{-# LANGUAGE OverloadedStrings #-}
import Crypto.PasswordStore
import qualified Data.ByteString.Char8 as B
main = do
    passwordHash <- makePassword "ilovesushi" 14
    B.putStrLn passwordHash
    print $ verifyPassword "bananas" passwordHash
    print $ verifyPassword "ilovesushi" passwordHash
```

We see that we can create a properly salted and stretched hash.

## Update Demo to use Crypto.PasswordStore

Let's add a few lines to take advantage of the benefits of the salting and stretching. Because TCache uses a file backend for persistence, we want to ensure if our files (or DB, or key-value store, etc) are compromised, that the attacker does not get plaintext passwords.

``` active haskell web
import MFlow.Wai.Blaze.Html.All
import Data.Monoid
import Data.IORef
import Data.ByteString hiding (null,map,putStr)
import qualified Data.ByteString.Char8 as BC
import Crypto.PasswordStore

import MFlow.Wai(waiMessageFlow)
import Network.Wai.Handler.WarpTLS as TLS
import Network.Wai.Handler.Warp (defaultSettings,Settings,setPort)
import Control.Workflow (Workflow)
import Control.Monad(unless)
import Data.Maybe (fromMaybe)
import System.Environment(getArgs,getEnvironment)
import Data.Char(isNumber)
-- show
main= do
    {-hi-}setAuthMethod $ Auth (tCacheRegister' defaultHashStrength){-/hi-}
                          {-hi-}tCacheValidate'{-/hi-}
    runNavigation "" $ transientNav loginSample

{-hi-}type HashStrength = Int
defaultHashStrength = 14 :: HashStrength{-/hi-}

-- | Register an user/password
tCacheRegister' ::  HashStrength -> String -> String  -> IO (Maybe String)
tCacheRegister' strength user password  =  do
  {-hi-}salted_password <- makePassword (BC.pack password) strength{-/hi-}
  atomically $ do
    let newuser = User user {-hi-}(BC.unpack salted_password){-/hi-}
    withSTMResources [newuser] $ doit {-hi-}newuser{-/hi-}
  where
    doit {-hi-}u{-/hi-} [Just (User _ _)] = resources{toReturn= Just "user already exist"}
    doit {-hi-}u{-/hi-} [Nothing] = resources{toAdd= [u],toReturn= Nothing}

tCacheValidate' ::  UserStr -> PasswdStr -> IO (Maybe String)
tCacheValidate'  u p =
    let user= eUser{userName=u}
    in  atomically
     $ withSTMResources [user]
     $ \ mu -> case mu of
         [Nothing] -> resources{toReturn= err }
         [Just u@(User _ pass )] -> resources{toReturn =
               case {-hi-}verifyPassword (BC.pack p) (BC.pack pass){-/hi-} of
                 True -> Nothing
                 False -> err
               }
     where
     err= Just  "Username or password invalid"

loginSample= do
    userRegister "user" "user"
    r <- page $ p << "Please login with user/user"
               ++> userWidget Nothing userLogin
               <|> wlink "exit" << p << "or exit"

    if r == "exit" then return () else do
        user <- getCurrentUser

        r <- page $ b << ("user logged as " <> user)
                   ++> wlink True << p << "logout"
                   <|> wlink False << p << "or exit"

        if r
          then do
             logout
             page $ p << "logged out" ++> wlink () << "press here to exit"
          else return ()
-- /show
```

The highlighted portions show what we did. We override the default `Auth` using `setAuthMethod` and use custom register and validate functions. The `` tCacheRegister` `` and `` tCacheValidate` `` are straight from `MFlow.tCacheRegister` and `MFlow.tCacheValidate` other than the highlighted portions. A small patch into those functions would add this capability to the first demo shown above.

## Conclusions

That was pretty simple to change. I think something like that should be the default for the `userWidget`.

### Mistakes

Let me know if I made any mistakes, especially with crypto, any ByteString conversion issues, etc.

