# Simple examples

### Simple application

``` active haskell
main = putStrLn "Hello, world!"
```

### System info

``` active haskell
import System.Info

main = do
    print os
    print arch
    print compilerName
    print compilerVersion
```

### System environment

``` active haskell
import System.Environment

main = do
    getArgs >>= print
    getProgName >>= print
    getEnvironment >>= print
```

### System environment for web application

``` active haskell web
import Happstack.Server.Env
import System.Environment

main = do
    environment <- getEnvironment
    simpleHTTP nullConf $ ok $ show environment
```
    
### Directories

``` active haskell
import System.Directory

main = do
    getCurrentDirectory >>= print 
    getHomeDirectory >>= print
    getUserDocumentsDirectory >>= print
```

### Current date and time

``` active haskell
import Data.Time

main = getCurrentTime >>= print
```

### Simple HTTP conduit

``` active haskell
import Network.HTTP.Conduit
import qualified Data.ByteString.Lazy as L

main = simpleHttp "http://www.winsoft.sk" >>= L.putStr
```

### Streaming HTTP conduit

``` active haskell
import Network.HTTP.Conduit
import Control.Monad.IO.Class (liftIO)

main = withManager $ \manager -> do
    request <- parseUrl "http://www.winsoft.sk"
    liftIO $ print request
    response <- httpLbs request manager
    liftIO $ print response
```

### Yesod version

``` active haskell
import Yesod

main = putStrLn yesodVersion
```

### Yesod application

``` active haskell web
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE TypeFamilies #-}
{-# LANGUAGE QuasiQuotes #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE MultiParamTypeClasses #-}

import Yesod

data WebApp = WebApp

instance Yesod WebApp

mkYesod "WebApp" [parseRoutes|
  / HomeR GET
|]

getHomeR = defaultLayout [whamlet|
  <div>Hello, world!
|]

main = warpEnv WebApp
```

### Snap application

``` active haskell web
{-# LANGUAGE OverloadedStrings #-}

import Snap.Http.Server.Env
import Snap.Core

main = httpServe defaultConfig $ writeBS "Hello, world!"
```

### Happstack application

``` active haskell web
import Happstack.Server.Env

main = simpleHTTP nullConf $ ok "Hello, world!"
```

### JavaScript minification

``` active haskell
{-# LANGUAGE OverloadedStrings #-}

import Text.Jasmine
import Data.ByteString.Lazy.Char8

main = print $ unpack $ minify "function test() { alert('Hello, world!'); }"
```

### Lists

``` active haskell
list = [1, 2, 3, 4, 5]

main = do
    print list

    print $ head list
    print $ tail list
    print $ last list
    print $ init list

    print $ list !! 3
    print $ elem 3 list

    print $ length list
    print $ null list
    print $ reverse list

    print $ take 2 list
    print $ drop 2 list

    print $ minimum list
    print $ maximum list

    print $ sum list
    print $ product list

    print [1..10]
    print ['A'..'Z']
    print [2,4..20]

    print $ take 10 $ cycle [1..4]
    print $ map (+1) list

    print $ filter (>3) list
    print $ all even list
    print $ any odd list

    print $ foldr (+) 0 list
    print $ foldr1 (+) list

    print $ foldl (+) 0 list
    print $ foldl1 (+) list

    print $ scanr (+) 0 list
    print $ scanr1 (+) list

    print $ scanl (+) 0 list
    print $ scanl1 (+) list

    print $ take 10 $ repeat 0
    print $ replicate 10 0
    print $ drop 3 list

    print $ ['a', 'b'] ++ ['c']
    print $ zip [1, 2, 3] ['a', 'b', 'c']
    print $ unzip [(1, 'a'), (2, 'b'), (3, 'c')]
    print $ zipWith (+) [1, 2, 3] [4, 5, 6]
    print $ [(x, y) | x <- [1..5], y <- ['a'..'e']]

    print $ words "Hello world"
    print $ unwords ["Hello", "world"] 
```

### Tuples

``` active haskell
tuple = (1, 2)

tuple3 = (1, 2, 3)

first (a, _, _) = a
second (_, b, _) = b
third (_, _, c) = c

main = do
    print tuple
    print $ fst tuple
    print $ snd tuple

    print tuple3
    print $ first tuple3
    print $ second tuple3
    print $ third tuple3
```

### Data.List

``` active haskell
import Data.List

main = do
    print $ intersperse '.' "Erik"
    print $ intercalate " " ["abc","efg","x"]
    print $ transpose ["abc","efg"]
    print $ subsequences "abc"
    print $ permutations "abc"

    print $ foldl' (+) 0 [1..1000000]
    print $ foldl1' (+) [1..1000000]

    print $ concat ["abc","efg"]
    print $ any (== 'a') "abcd"
    print $ all (== 'a') "abcd"
    print $ take 10 $ iterate (+1) 1
    print $ replicate 10 'x'
    print $ splitAt 3 "abcdefgh"
    print $ takeWhile (< 3) [1..]
    print $ span (< 3) [1..10]
    print $ break (> 3) [1..10]
    print $ stripPrefix "ab" "abcdefg"
    print $ isPrefixOf "ab" "abcdefg"
    print $ elem 'c' "abcdefg"
    print $ lookup 'c' [('a', 1), ('b', 2), ('c', 3)]
    print $ find (> 2) [1..]
    print $ partition (> 2) [1..10]
    print $ nub [1, 1, 3, 2, 1, 2, 4, 6]
    print $ sort [10, 9, 8, 7, 6, 5, 4, 3, 2, 1]
    print $ elemIndex 2 [1, 2, 3, 4, 2]
    print $ elemIndices 2 [1, 2, 3, 4, 2]
    print $ findIndex (< 3) [1, 2, 3, 4, 2]
    print $ findIndices (< 3) [1, 2, 3, 4, 2]
```

### Data.Char

``` active haskell
import Data.Char

main = do
    print $ isAlpha 'c'
    print $ isDigit '4'
    print $ toUpper 'a'
    print $ toLower 'E'
    print $ digitToInt '2'
    print $ intToDigit 9
    print $ intToDigit 12
    print $ ord('A')
    print $ chr(61)
```

### Data.Map

``` active haskell
import qualified Data.Map as Map

phoneBook = Map.fromList [(1234, "Erik"), (5678, "Patrik")]

main = do
    print phoneBook
    print $ Map.lookup 1234 phoneBook
    print $ (Map.empty :: Map.Map Int Int)
    print $ Map.singleton 3 5
    print $ Map.insert 1 "abc" Map.empty
    print $ Map.null phoneBook
    print $ Map.size phoneBook
    print $ Map.toList phoneBook
    print $ Map.keys phoneBook
    print $ Map.elems phoneBook
```

### Data.Set

``` active haskell
import qualified Data.Set as Set

set = Set.fromList "erik salaj"

main = do
    print set
    print $ Set.null set
    print $ Set.size set
    print $ Set.member 'a' set
```

### Data.Array

``` active haskell
import Data.Array

myArray = array (1, 3) [(1, "a"), (2, "b"), (3, "c")]

main = do
    print myArray
    print $ myArray ! 2
    print $ bounds myArray
    print $ indices myArray
    print $ elems myArray
    print $ assocs myArray
```

### Data.Complex

``` active haskell
import Data.Complex

number = 3 :+ 4

main = do
    print number
    print $ realPart number
    print $ imagPart number
	print $ polar number
	print $ magnitude number
	print $ phase number
	print $ conjugate number
```

### Data.HashSet

``` active haskell
import Prelude hiding (null, map, filter)
import Data.HashSet
import Data.Char

hashSet = fromList ['a', 'b', 'c']

main = do
    print $ hashSet

    print $ null hashSet
    print $ size hashSet

    print $ member 'a' hashSet
    print $ member 'e' hashSet

    print $ insert 'd' hashSet
    print $ delete 'b' hashSet

    print $ map (toUpper) hashSet
    print $ filter (> 'a') hashSet
```

### Data.HashMap

``` active haskell
import Prelude hiding (null, lookup, map, filter)
import Data.HashMap.Lazy
import Data.Char

hashMap = fromList [(1 :: Int, 'a'), (2, 'b'), (3, 'c')]

main = do
    print $ hashMap
    print $ keys hashMap
    print $ elems hashMap

    print $ null hashMap
    print $ size hashMap

    print $ member 1 hashMap
    print $ member 5 hashMap

    print $ lookup 1 hashMap
    print $ lookup 5 hashMap

    print $ hashMap ! 1
    print $ lookupDefault 'N' 5 hashMap

    print $ insert 4 'd' hashMap
    print $ delete 2 hashMap

    print $ map (toUpper) hashMap
    print $ filter (> 'a') hashMap
```

### Data.Graph

``` active haskell
import Data.Graph

graph = buildG (1, 6) [(1, 2), (1, 3), (2, 4), (5, 6)]

main = do
    print graph
    print $ vertices graph
    print $ edges graph
    print $ edges $ transposeG graph

    print $ outdegree graph
    print $ indegree graph

    print $ topSort graph
    print $ reachable graph 1

    print $ path graph 1 4
    print $ path graph 1 5

    print $ components graph
    print $ scc graph
    print $ bcc graph

    print $ dff graph
    print $ dfs graph [2]
```

### Show and read

``` active haskell
main = do
    print $ show 3
    print $ show [1, 2, 3]
    print $ show (1, False)

    print $ (read "34" :: Int)
    print $ (read "(1, False)" :: (Int, Bool))
```

### SQLite database

``` active haskell
{-# LANGUAGE OverloadedStrings #-}

import Database.Sqlite

printRows stmt = do
    row <- step stmt
    if row == Done then
        return ()
    else do
        col <- column stmt 0
        print col
        printRows stmt

main = do
    conn <- open "database.db"

    stmt <- prepare conn "DROP TABLE IF EXISTS MyTable;"
    step stmt
    finalize stmt

    stmt <- prepare conn "CREATE TABLE IF NOT EXISTS MyTable (Name VARCHAR(20));"
    step stmt
    finalize stmt

    stmt <- prepare conn "INSERT INTO MyTable(Name) VALUES('Erik');"
    step stmt
    finalize stmt

    stmt <- prepare conn "INSERT INTO MyTable(Name) VALUES('Patrik');"
    step stmt
    finalize stmt

    stmt <- prepare conn "SELECT * FROM MyTable;"
    printRows stmt
    finalize stmt

    close conn
```

### Files

``` active haskell
{-# START_FILE main.hs #-}
main = readFile "file.txt" >>= putStr

{-# START_FILE file.txt #-}
Hello, world!
```


``` active haskell
{-# START_FILE main.hs #-}
main = do
    contents <- readFile "file.txt"
    putStr contents

{-# START_FILE file.txt #-}
Hello, world!
```


``` active haskell
{-# START_FILE main.hs #-}
import System.IO

main = do
    handle <- openFile "file.txt" ReadMode
    contents <- hGetContents handle
    putStr contents
    hClose handle

{-# START_FILE file.txt #-}
Hello, world!
```


``` active haskell
{-# START_FILE main.hs #-}
import System.IO

main = withFile "file.txt" ReadMode $ \handle -> do
    contents <- hGetContents handle
    putStr contents

{-# START_FILE file.txt #-}
Hello, world!
```


``` active haskell
main = do
    writeFile "file.txt" "Hello, world!"
    readFile "file.txt" >>= print
```

### Random numbers

``` active haskell
import System.Random

main = (randomRIO (1, 100) :: IO Int) >>= print
```

### Base16 encoding

``` active haskell
{-# LANGUAGE OverloadedStrings #-}

import Data.ByteString.Base16
import Data.ByteString.Char8

main = do
    print $ unpack $ encode "Hello, world!"
    print $ decode "48656c6c6f2c20776f726c6421"
```

### Base64 encoding

``` active haskell
{-# LANGUAGE OverloadedStrings #-}

import Data.ByteString.Base64
import Data.ByteString.Char8

main = do
    print $ unpack $ encode "Hello, world!"
    print $ decode "SGVsbG8sIHdvcmxkIQ=="
```

### JSON

``` active haskell
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE DeriveGeneric #-}

import Data.Aeson
import Data.ByteString.Lazy.Char8
import GHC.Generics

data MyData = MyData { text :: String, number :: Int } deriving (Show, Generic)

instance FromJSON MyData
instance ToJSON MyData

myData = MyData "Hello" 123

main = do
    print myData
    print $ unpack $ encode myData
    print $ (decode "{ \"number\" : 123, \"text\" : \"Hello\" }" :: Maybe MyData)
```


``` active haskell
{-# LANGUAGE OverloadedStrings #-}

import Data.Aeson
import Control.Applicative
import Data.ByteString.Lazy.Char8 hiding (empty)

data MyData = MyData { text :: String, number :: Int } deriving Show

instance ToJSON MyData where
    toJSON (MyData text number) = object ["text" .= text, "number" .= number]

instance FromJSON MyData where
    parseJSON (Object v) = MyData <$> v .: "text" <*> v .: "number"
    parseJSON _          = empty

myData = MyData "Hello" 123

main = do
    print myData
    print $ unpack $ encode myData
    print $ (decode "{ \"number\" : 123, \"text\" : \"Hello\" }" :: Maybe MyData)
```

### Email validation

``` active haskell
{-# LANGUAGE OverloadedStrings #-}

import Text.Email.Validate

email = "eriksalaj@gmail.com"

main = do
    print $ isValid email
    print $ validate email
    print $ emailAddress email
    print $ canonicalizeEmail email

    let Just address = emailAddress email
    print $ localPart address
    print $ domainPart address
```

### Functor

``` active haskell
main = do
    print $ fmap (+ 1) Nothing
    print $ fmap (+ 1) $ Just 2

    print $ fmap (+ 1) [1, 2, 3]
    print $ fmap (* 2) (+ 5) 2 
```

### Applicative

``` active haskell
import Control.Applicative

main = do
    print $ (pure 1 :: Maybe Int)

    print $ Just (+ 1) <*> Nothing
    print $ Just (+ 1) <*> Just 2

    print $ [(+ 1), (* 2)] <*> []
    print $ [(+ 1), (* 2)] <*> [1, 2, 3]

    print $ Just 1 <* Just 2
    print $ Just 1 *> Just 2

    print $ (+ 1) <$> Nothing
    print $ (+ 1) <$> Just 2

    print $ 1 <$ Nothing
    print $ 1 <$ Just 2

    print $ Nothing <**> Just (+ 2)
    print $ Just 1 <**> Just (+ 2)

    print $ liftA (+ 1) Nothing
    print $ liftA (+ 1) $ Just 2

    print $ liftA2 (+) Nothing Nothing
    print $ liftA2 (+) (Just 1) (Just 2)

    print $ (+) <$> Just 1 <*> Nothing
    print $ (+) <$> Just 1 <*> Just 2
```

### Type class

``` active haskell
class MyClass a where
    myFunc :: a -> String

instance MyClass Bool where
    myFunc n = "Bool: " ++ show n

instance MyClass Char where
    myFunc n = "Char: " ++ show n

myShow :: MyClass a => a -> String
myShow n = myFunc n

main = do
    print $ myFunc True
    print $ myFunc 'a'

    print $ myShow True
    print $ myShow 'a'
```

### Record

``` active haskell
data Person = Person { firstName :: String, lastName :: String } deriving Show

person = Person "Erik" "Salaj"

main = do
    print person
    print Person { firstName = "Erik", lastName = "Salaj" }
    print $ firstName person
    print $ lastName person
```

### Threads

``` active haskell
import Control.Concurrent

main = do
    getNumCapabilities >>= print
    print rtsSupportsBoundThreads

    forkIO $ sequence_ $ replicate 3 $ do { print "Thread 1"; threadDelay 1 }
    forkIO $ sequence_ $ replicate 3 $ do { print "Thread 2"; threadDelay 1 }
    forkIO $ sequence_ $ replicate 3 $ do { print "Thread 3"; threadDelay 1 }
    threadDelay 10000
```

### CPU time

``` active haskell
import System.CPUTime

main = do
    print cpuTimePrecision
    getCPUTime >>= print
```

### External command

``` active haskell
import System.Process

main = do
    system "echo Hello, world!"

    system "uname -a"
    system "cat /proc/version"
    system "cat /proc/cpuinfo"
    system "lsb_release -a"

    system "ghc --version"
    system "cc --version"
    system "java -version"
    system "python --version"
```

### Trace

``` active haskell
import Debug.Trace

main = do
    print $ trace "Calling 1 + 1" (1 + 1)

    traceIO "Calling 1 + 1"
    print $ 1 + 1

    let x = 1
    print $ traceShow (x, x + x) (x + x)
```

### Sections

``` active haskell
main = do
    print $ (1 /) 2
    print $ (/ 1) 2
```

### C application

``` active haskell
{-# START_FILE main.hs #-}
import System.Process

main = do
    system "cc main.c"
    system "./a.out"

{-# START_FILE main.c #-}
#include <stdio.h>

int main() {
    printf("Hello, world!\n");
    return 0;
}
```

### Java application

``` active haskell
{-# START_FILE main.hs #-}
import System.Process

main = do
    system "javac Main.java"
    system "java Main"

{-# START_FILE Main.java #-}
public class Main {
    public static void main(String[] args) {
        System.out.println("Hello, world!");
    }
}
```

### Python application

``` active haskell
{-# START_FILE main.hs #-}
import System.Process

main = system "python hello.py"
{-# START_FILE hello.py #-}
print "Hello, world!"
```

### Unique values

``` active haskell
import Data.Unique

main = do
    unique <- newUnique
    print $ hashUnique unique

    unique <- newUnique
    print $ hashUnique unique

    unique <- newUnique
    print $ hashUnique unique
```

### Automatic testing

``` active haskell
import Test.QuickCheck

check x = x == (reverse . reverse) x

main = do
    print stdArgs
    sample (vector 3 :: Gen [Int])
    sample (orderedList :: Gen [Int])

    quickCheck (check :: [Int] -> Bool)
    verboseCheck (check :: [Int] -> Bool)
```

### UTF-8

``` active haskell
{-# LANGUAGE OverloadedStrings #-}

import qualified Data.ByteString.UTF8 as UTF8
import Data.ByteString.Char8

main = do
    print "áéí"
    print $ UTF8.fromString "áéí"
    print $ UTF8.length "\195\161\195\169\195\173"
    print $ UTF8.toString "\195\161\195\169\195\173"
```

### Type representations

``` active haskell
import Data.Typeable

main = do
    print $ typeOf 'a'
    print $ typeOf "Hello, world!"
    print $ typeOf putStrLn

    print $ (cast True :: Maybe Int)
    print $ (cast True :: Maybe Bool)
```

### Modules

``` active haskell
{-# START_FILE main.hs #-}
import Test

main = helloWorld

{-# START_FILE Test.hs #-}
module Test where

helloWorld = putStrLn "Hello, world!"
```

### Numeric

``` active haskell
import Numeric
import Data.Char

main = do
    print $ showInt 123 ""
    print $ showHex 123 ""
    print $ showOct 123 ""
    print $ showIntAtBase 2 intToDigit 123 ""

    print $ showFloat 123.456 ""
    print $ showEFloat (Just 2) 123.456 ""
    print $ showFFloat (Just 2) 123.456 ""
    print $ showGFloat (Just 2) 123.456 ""
    print $ floatToDigits 10 123.456
    print $ floatToDigits 16 123.456
```

### Data.Tree

``` active haskell
import Data.Tree

tree = Node "A" [Node "B" [], Node "C" [Node "D" [], Node "E" []]]

main = do
    print tree
    putStrLn $ drawTree tree
    putStrLn $ drawForest $ subForest tree

    print $ flatten tree
    print $ levels tree
```

### Binary serialization

``` active haskell
{-# LANGUAGE OverloadedStrings #-}

import Data.Serialize
import Data.Word
import Data.ByteString.Char8

main = do
    print $ encode (123 :: Word8)
    print $ (decode "{" :: Either String Word8)

    print $ encode (123 :: Word16)
    print $ (decode "\NUL{" :: Either String Word16)

    print $ encode 'a'
    print $ (decode "a" :: Either String Char)

    print $ encode ("abc" :: String)
    print $ (decode "\NUL\NUL\NUL\NUL\NUL\NUL\NUL\ETXabc" :: Either String String)
```

### Integral

``` active haskell
main = do
    print $ 123 `quot` 4
    print $ 123 `quot` (-4)

    print $ 123 `div` 4
    print $ 123 `div` (-4)

    print $ 123 `mod` 4
    print $ 123 `mod` (-4)

    print $ 123 `quotRem` 4
    print $ 123 `quotRem` (-4)

    print $ 123 `divMod` 4
    print $ 123 `divMod` (-4)
```

### Data.IORef

``` active haskell
import Data.IORef

main = do
    ref <- newIORef 0
    value <- readIORef ref
    print value

    writeIORef ref $ value + 1
    readIORef ref >>= print

    modifyIORef ref (+ 2)
    readIORef ref >>= print
```

### Transactions

``` active haskell
import Control.Concurrent.STM

type Account = TVar Integer

credit account amount = do
    current <- readTVar account
    writeTVar account (current + amount)

debit account amount = do
    current <- readTVar account
    writeTVar account (current - amount)

transfer from to amount =
    atomically $ do
        debit from amount
        credit to amount

main = do
    account1 <- atomically $ newTVar 10
    account2 <- atomically $ newTVar 20

    transfer account1 account2 5

    balance1 <- atomically $ readTVar account1
    balance2 <- atomically $ readTVar account2
    print balance1
    print balance2
```

### Data.Tuple

``` active haskell
import Data.Tuple

main = print $ swap (1, 2)
```

### Byte order

``` active haskell
import System.ByteOrder

main = print byteOrder
```

### Byte dump

``` active haskell
import Text.Bytedump

main = do
    print $ hexString 100
    print $ dumpRawS "Hello, world!"
    print $ dumpS "Hello, world!"
```

### UUID

``` active haskell
import Prelude hiding (null)
import Data.UUID
import Data.UUID.V1
import Data.UUID.V3 as V3
import Data.UUID.V4
import Data.UUID.V5 as V5

main = do
    print nil
    print $ null nil
    print $ toWords nil
    print $ fromWords 1 2 3 4
    print $ toString nil
    print $ fromString "00000000-0000-0000-0000-000000000000"

    uuid <- nextUUID
    print uuid

    random <- nextRandom
    print random

    print $ V3.namespaceDNS
    print $ V3.namespaceURL
    print $ V3.namespaceOID
    print $ V3.namespaceX500
    print $ V3.generateNamed V3.namespaceDNS [1, 2, 3]

    print $ V5.namespaceDNS
    print $ V5.namespaceURL
    print $ V5.namespaceOID
    print $ V5.namespaceX500
    print $ V5.generateNamed V5.namespaceDNS [1, 2, 3]
```

### CPU information

``` active haskell
import System.Arch
import System.Endian

main = do
    print $ getSystemArch
    print $ getSystemEndianness
    print $ toBE32 0xFF000000
```

### HostName

``` active haskell
import Network.HostName

main = getHostName >>= print
```

### SHA

``` active haskell
{-# LANGUAGE OverloadedStrings #-}

import Data.Digest.Pure.SHA
import Data.ByteString.Lazy.Char8

main = do
    print $ sha1 "Hello, world!"
    print $ sha224 "Hello, world!"
    print $ sha256 "Hello, world!"
    print $ sha384 "Hello, world!"
    print $ sha512 "Hello, world!"

    print $ hmacSha1 "key" "Hello, world!"
    print $ hmacSha224 "key" "Hello, world!"
    print $ hmacSha256 "key" "Hello, world!"
    print $ hmacSha384 "key" "Hello, world!"
    print $ hmacSha512 "key" "Hello, world!"
```

### MD5

``` active haskell
{-# LANGUAGE OverloadedStrings #-}

import Data.Digest.Pure.MD5
import Data.ByteString.Lazy.Char8

main = print $ md5 "Hello, world!"
```

### Punycode

``` active haskell
{-# LANGUAGE OverloadedStrings #-}

import Data.Text.Punycode
import Data.ByteString.Char8

main = do
    print $ encode "Slovenský jazyk"
    print $ decode "Slovensk jazyk-2sb"
```

### Dimensional

``` active haskell
import Numeric.Units.Dimensional.Prelude
import qualified Prelude

main = do
    print $ 1 *~ kilo meter
    print $ 1 *~ (kilo meter / hour)
    print $ 1 *~ newton
    print $ 1 *~ pascal
```

### Java parser

``` active haskell
{-# START_FILE main.hs #-}
import Language.Java.Lexer
import Language.Java.Parser
import Language.Java.Pretty

main = do
    source <- readFile "Main.java"
    print $ lexer source
    print $ parser compilationUnit source

    let result = parser compilationUnit source
    case result of
        Left error -> print error
        Right ast -> putStrLn $ prettyPrint ast
{-# START_FILE Main.java #-}
public class Main {
    public static void main(String[] args) {
        System.out.println("Hello, world!");
    }
}
```

### JavaScript parser

``` active haskell
{-# START_FILE main.hs #-}
import Language.JavaScript.Parser

main = do
    source <- readFile "Main.js"
    print $ parse source "Main.js"

    let result = parse source "Main.js"
    case result of
        Left error -> print error
        Right ast -> putStrLn $ renderToString ast
{-# START_FILE Main.js #-}
function test() {
    alert('Hello, world!');
}
```

### Crypto.Hash

``` active haskell
{-# LANGUAGE OverloadedStrings #-}

import Crypto.Hash.MD2 as MD2
import Crypto.Hash.MD4 as MD4
import Crypto.Hash.MD5 as MD5
import Crypto.Hash.RIPEMD160 as RIPEMD160
import Crypto.Hash.SHA1 as SHA1
import Crypto.Hash.SHA224 as SHA224
import Crypto.Hash.SHA256 as SHA256
import Crypto.Hash.SHA384 as SHA384
import Crypto.Hash.SHA512 as SHA512
import Crypto.Hash.Tiger as Tiger
import Crypto.Hash.Whirlpool as Whirlpool
import Data.ByteString.Base16

main = do
    print $ encode $ MD2.hash "Hello, world!"
    print $ encode $ MD4.hash "Hello, world!"
    print $ encode $ MD5.hash "Hello, world!"

    print $ encode $ RIPEMD160.hash "Hello, world!"

    print $ encode $ SHA1.hash "Hello, world!"
    print $ encode $ SHA224.hash "Hello, world!"
    print $ encode $ SHA256.hash "Hello, world!"
    print $ encode $ SHA384.hash "Hello, world!"
    print $ encode $ SHA512.hash "Hello, world!"

    print $ encode $ Tiger.hash "Hello, world!"

    print $ encode $ Whirlpool.hash "Hello, world!"
```

### Diff

``` active haskell
{-# START_FILE main.hs #-}
import Data.Algorithm.Diff
import Data.Algorithm.DiffOutput

main = do
    file1 <- readFile "file1.txt"
    file2 <- readFile "file2.txt"

    let
        lines1 = lines file1
        lines2 = lines file2
    print $ getDiff lines1 lines2
    print $ getGroupedDiff lines1 lines2
    putStrLn $ ppDiff $ getGroupedDiff lines1 lines2
{-# START_FILE file1.txt #-}
first line
second line
{-# START_FILE file2.txt #-}
first line
hello
third line
```

### Zlib

``` active haskell
{-# LANGUAGE OverloadedStrings #-}

import Codec.Compression.Zlib
import Data.ByteString.Lazy.Char8

main = do
    print $ compress "Hello, world!"
    print $ decompress "x\156\243H\205\201\201\215Q(\207/\202IQ\EOT\NUL ^\EOT\138"
```

### GZip

``` active haskell
{-# LANGUAGE OverloadedStrings #-}

import Codec.Compression.GZip
import Data.ByteString.Lazy.Char8

main = do
    print $ compress "Hello, world!"
    print $ decompress "\US\139\b\NUL\NUL\NUL\NUL\NUL\NUL\ETX\243H\205\201\201\215Q(\207/\202IQ\EOT\NUL\230\198\230\235\r\NUL\NUL\NUL"
```

### Linear Algebra

``` active haskell
import Linear

main = do
    print $ V0

    print $ V1 1
    print $ V1 1 + V1 2
    print $ V1 1 - V1 2
    print $ V1 1 * V1 2
    print $ V1 1 / V1 2

    print $ V2 1 2
    print $ V2 1 2 + V2 3 4
    print $ V2 1 2 - V2 3 4
    print $ V2 1 2 * V2 3 4
    print $ V2 1 2 / V2 3 4
    print $ perp $ V2 0 1

    print $ V3 1 2 3
    print $ V3 1 2 3 + V3 4 5 6
    print $ V3 1 2 3 - V3 4 5 6
    print $ V3 1 2 3 * V3 4 5 6
    print $ V3 1 2 3 / V3 4 5 6
    print $ cross (V3 1 2 3) (V3 4 5 6)

    print $ V4 1 2 3 4
    print $ V4 1 2 3 4 + V4 5 6 7 8
    print $ V4 1 2 3 4 - V4 5 6 7 8
    print $ V4 1 2 3 4 * V4 5 6 7 8
    print $ V4 1 2 3 4 / V4 5 6 7 8
    print $ vector $ V3 1 2 3
    print $ point $ V3 1 2 3

    print $ (zero :: V3 Double)
    print $ negated $ V3 1 2 3
    print $ V3 1 2 3 ^* V3 4 5 6
    print $ V3 1 2 3 *^ V3 4 5 6
    print $ V3 1 2 3 ^/ 2
    print $ sumV [V3 1 2 3, V3 4 5 6, V3 7 8 9]
    print $ (basis :: [V3 Int])
    print $ basisFor $ V3 1 2 3
    print $ kronecker $ V3 1 2 3
    print $ outer (V3 1 2 3) (V3 4 5 6)

    print $ nearZero (1e-10 :: Double)
    print $ nearZero (1e-15 :: Double)

    print $ trace $ V3 (V3 1 2 3) (V3 4 5 6) (V3 7 8 9)
    print $ diagonal $ V3 (V3 1 2 3) (V3 4 5 6) (V3 7 8 9)

    print $ dot (V3 1 2 3) (V3 4 5 6)
    print $ quadrance $ V3 1 2 3
    print $ qd (V3 1 2 3) (V3 4 5 6)
    print $ distance (V3 1 2 3) (V3 4 5 6)
    print $ norm $ V3 1 2 3
    print $ signorm $ V3 1 2 3
    print $ normalize (V3 1 2 3 :: V3 Double)
```

### Data.Matrix

``` active haskell
import Data.Matrix

m1 = matrix 3 4 $ \(r, c) -> 4 * (r - 1) + c
m2 = fromList 3 4 [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12]
m3 = fromLists [[1, 2, 3, 4], [5, 6, 7, 8], [9, 10, 11, 12]]

main = do
    print m1
    print m2
    print m3

    print $ zero 3 4
    print $ identity 3
    print $ permMatrix 3 1 2

    print $ nrows m1
    print $ ncols m1

    print $ getElem 2 3 m1
    print $ m1 ! (2, 3)
    print $ getRow 2 m1
    print $ getCol 3 m1
    print $ getDiag m1

    print $ setElem 13 (2, 3) m1
    print $ transpose m1
    print $ extendTo 0 4 8 m1
    print $ mapRow (\c x -> 2 * x) 3 m1

    print $ submatrix 2 3 1 2 m1
    print $ minorMatrix 1 2 m1
    print $ splitBlocks 2 3 m1

    print $ m1 <|> zero 3 2
    print $ m1 <-> zero 2 4

    print $ multStd m1 (identity 4)

    print $ scaleMatrix 2 m1
    print $ scaleRow 2 3 m1
    print $ combineRows 3 2 1 m1
    print $ switchRows 1 2 m1

    print $ luDecomp $ fromLists [[1.0, 2.0], [3.0, 4.0]]
    print $ trace m1
    print $ diagProd m1

    print $ detLaplace $ identity 3
    print $ detLU $ fromLists [[1.0, 2.0], [3.0, 4.0]]
```

### Towers of Hanoi

``` active haskell
data Tower = Tower1 | Tower2 | Tower3 deriving Show

move 0 from to temp = []
move n from to temp = move (n - 1) from temp to ++ [(from, to)] ++ move (n - 1) temp to from

main = print $ move 3 Tower1 Tower2 Tower3
```

### C parser

``` active haskell
{-# START_FILE main.hs #-}
import Language.C

main = do
    result <- parseCFilePre "test.c"
    case result of
        Left error -> print error
        Right ast -> do
            print ast
            print $ pretty ast
{-# START_FILE test.c #-}
int main() {
    printf("Hello, world!\n");
    return 0;
}
```

### Data.Either

``` active haskell
import Data.Either
import Prelude hiding (error)

type ErrorOrValue = Either String Int

error = Left "MyError" :: ErrorOrValue
value = Right 123 :: ErrorOrValue

main = do
    print error
    print value

    print $ isLeft error
    print $ isLeft value

    print $ isRight error
    print $ isRight value

    case error of
        Left x -> print $ "Error: " ++ x
        Right x -> print $ "Value: " ++ show x

    case value of
        Left x -> print $ "Error: " ++ x
        Right x -> print $ "Value: " ++ show x

    print $ either show (show.(+ 1)) error
    print $ either show (show.(+ 1)) value

    print $ either (\_ -> 0) id error
    print $ either (\_ -> 0) id value

    print $ lefts [error, value]
    print $ rights [error, value]
    print $ partitionEithers [error, value]
```

### Data.Maybe

``` active haskell
import Data.Maybe
import Data.Char

nothing = Nothing :: Maybe String
just = Just "Hello, world!" :: Maybe String

main = do
    print nothing
    print just

    print $ isNothing nothing
    print $ isNothing just

    print $ isJust nothing
    print $ isJust just

    case nothing of
        Nothing -> print "Nothing"
        Just x -> print x

    case just of
        Nothing -> print "Nothing"
        Just x -> print x

    print $ maybe "Default" (map toUpper) nothing
    print $ maybe "Default" (map toUpper) just

    print $ fromJust just

    print $ fromMaybe "Default" nothing
    print $ fromMaybe "Default" just

    print $ listToMaybe ([] :: [Int])
    print $ listToMaybe [1, 2, 3]

    print $ maybeToList nothing
    print $ maybeToList just

    print $ catMaybes [nothing, just]

    print $ mapMaybe (\_ -> (Nothing :: Maybe Int)) [1, 2, 3]
    print $ mapMaybe (\x -> Just x) [1, 2, 3]
```

### Fibonacci

``` active haskell
fibonacci = 0 : 1 : zipWith (+) fibonacci (tail fibonacci)

main = print $ take 20 fibonacci
```

``` active haskell
fib a b = a : fib b (a + b)

fibonacci = fib 0 1

main = print $ take 20 fibonacci
```

``` active haskell
fib n m a b
    | n == m = a
    | otherwise = fib n (m + 1) b (a + b)

fibonacci n = fib n 0 0 1

main = print [fibonacci n | n <- [0..19]]
```

``` active haskell
fib 0 a b = a
fib n a b = fib (n - 1) b (a + b)

fibonacci n = fib n 0 1

main = print [fibonacci n | n <- [0..19]]
```

### Coin changes

``` active haskell
changeCount 0 _ = 1
changeCount _ [] = 0
changeCount n (coin : coins)
    | n > 0 = changeCount (n - coin) (coin : coins) + changeCount n coins
    | otherwise = 0

main = print $ changeCount 10 [8, 5, 1]
```

``` active haskell
import Data.List
import Data.Ord

allChanges 0 _ = [[]]
allChanges _ [] = []
allChanges n (coin : coins)
    | n > 0 = map (coin :) (allChanges (n - coin) (coin : coins)) ++ allChanges n coins
    | otherwise = []

shortest = minimumBy $ comparing length

optimalChange n coins = shortest $ allChanges n coins

main = do
    print $ allChanges 10 [8, 5, 1]
    print $ optimalChange 10 [8, 5, 1]
```

``` active haskell
bestChange 0 _ = Just []
bestChange _ [] = Nothing
bestChange n (coin : coins)
    | n > 0 = shorter (fmap (coin :) (bestChange (n - coin) (coin : coins))) (bestChange n coins)
    | otherwise = Nothing

shorter Nothing Nothing = Nothing
shorter (Just a) Nothing = Just a
shorter Nothing (Just b) = Just b
shorter (Just a) (Just b) = if length a < length b then Just a else Just b

main = print $ bestChange 10 [8, 5, 1]
```

### Queens

``` active haskell
queens n = queens' n n

queens' n 0 = [[]]
queens' n k = [x:xs | xs <- queens' n (k - 1), x <- [1..n], isSafeColumn x xs, isSafeDiagonal x xs]

isSafeColumn x xs = not $ elem x xs

isSafeDiagonal x xs = all (\(a, b) -> abs(x - a) /= b) $ zip xs [1..]

showLine n k = replicate (k - 1) '.' ++ "X" ++ replicate (n - k) '.'

showSolution s = (mapM_ putStrLn [showLine (length s) k | k <- s]) >> putStrLn ""

main = mapM_ showSolution $ queens 4
```

### GHC options

``` active haskell
{-# OPTIONS_GHC -fwarn-missing-signatures #-}

main = putStrLn "Hello, world!"
```

### PackageImports

``` active haskell
{-# LANGUAGE PackageImports #-}

import "unordered-containers" Data.HashSet

main = print $ singleton 'a'
```

### Monoid

``` active haskell
import Data.Monoid

main = do
    print $ Sum 10
    print $ getSum $ Sum 10
    print $ (mempty :: Sum Int)
    print $ mappend (Sum 10) (Sum 20)
    print $ Sum 10 <> Sum 20
    print $ mconcat [Sum 10, Sum 20, Sum 30]

    print $ Product 10
    print $ getProduct $ Product 10
    print $ Product 10 <> Product 20

    print $ First (Just 10) <> First (Just 20)
    print $ First Nothing <> First (Just 20)

    print $ Last (Just 10) <> Last (Just 20)
    print $ Last Nothing <> Last (Just 20)

    print $ Any False <> Any False
    print $ Any False <> Any True

    print $ All False <> All True
    print $ All True <> All True

    print $ Dual (First (Just 10)) <> Dual (First (Just 20))
    print $ Dual (First (Just 20)) <> Dual (First (Just 10))

    print $ appEndo (Endo (+ 10)) 1
    print $ appEndo (Endo (+ 10) <> (Endo (+ 20))) 2
```

### Data.Vector

``` active haskell
import Prelude hiding (replicate, enumFromTo, enumFromThenTo, length, null)
import Data.Vector

main = do
    print $ (empty :: Vector Char)
    print $ singleton 'a'
    print $ replicate 10 'b'
    print $ generate 10 (* 2)
    print $ iterateN 10 (+ 1) 100
    print $ enumFromN 10 5
    print $ enumFromStepN 2 3 10
    print $ enumFromTo 10 20
    print $ enumFromThenTo 10 12 20

    print $ fromList [1..5]
    print $ fromListN 3 [1..5]
    print $ toList $ replicate 10 'c'

    let vector = fromList [1..10]

    print $ length vector
    print $ null vector
    print $ null $ fromList []

    print $ vector ! 0
    print $ vector !? 0
    print $ vector !? 10
```

### Cellular automaton

``` active haskell
rule ' ' ' ' ' ' = ' '
rule ' ' ' ' 'X' = 'X'
rule ' ' 'X' ' ' = ' '
rule ' ' 'X' 'X' = 'X'
rule 'X' ' ' ' ' = 'X'
rule 'X' ' ' 'X' = ' '
rule 'X' 'X' ' ' = 'X'
rule 'X' 'X' 'X' = ' '

start n = replicate n ' ' ++ "X" ++ replicate n ' '

next (a : b : c : rest) = rule a b c : next (b : c : rest)
next _ = " "

rows n = take n $ iterate (\x -> ' ' : next x) $ start n

main = mapM_ putStrLn $ rows 32
```

### Enum

``` active haskell
main = do
    print $ succ 'a'
    print $ pred 'b'
    print $ (toEnum 65 :: Char)
    print $ fromEnum 'B'

    print $ take 10 $ enumFrom 'a'
    print $ take 10 $ ['a'..]

    print $ take 10 $ enumFromThen 'a' 'c'
    print $ take 10 $ ['a', 'c'..]

    print $ enumFromTo 'a' 'e'
    print $ ['a'..'e']

    print $ enumFromThenTo 'a' 'c' 's'
    print $ ['a', 'c'..'s']
```

### SKI calculus

``` active haskell
data SKI = S | K | I | App SKI SKI | Var String deriving (Show, Eq)

eval (App I x) = eval x
eval (App (App K x) y) = eval x
eval (App (App (App S x) y) z) = eval (App (App x z) (App y z))
eval (App x y) =
    if (App x y) == app then
        app
    else
        eval app
    where
        app = App (eval x) (eval y)
eval x = x

main = do
    print $ eval $ App I $ Var "x"
    print $ eval $ App (App K $ App I $ Var "x") $ Var "y"
    print $ eval $ App (App (App S I) I) $ Var "x"
    print $ eval $ App (App I I) (Var "x")
```

### Sieve of Eratosthenes

``` active haskell
sieve (p : xs) = p : sieve [x | x <- xs, x `mod` p /= 0]

primes = sieve [2..]

main = print $ take 20 primes
```

### Bounded

``` active haskell
import Data.Int
import Data.Word

main = do
    print $ (minBound :: Bool)
    print $ (maxBound :: Bool)

    print $ (minBound :: Char)
    print $ (maxBound :: Char)

    print $ (minBound :: Int)
    print $ (maxBound :: Int)

    print $ (minBound :: Int8)
    print $ (maxBound :: Int8)

    print $ (minBound :: Word8)
    print $ (maxBound :: Word8)
```

### Data.Bits

``` active haskell
import Data.Bits

main = do
    print $ bitSizeMaybe (0 :: Int)
    print $ isSigned (0 :: Int)

    print $ (2 .&. 3 :: Int)
    print $ (2 .|. 3 :: Int)
    print $ (xor 2 3 :: Int)
    print $ complement (2 :: Int)

    print $ (bit 2 :: Int)
    print $ popCount (123 :: Int)
    print $ testBit (2 :: Int) 1
    print $ setBit (2 :: Int) 0
    print $ clearBit (2 :: Int) 1
    print $ complementBit (2 :: Int) 1

    print $ shift (2 :: Int) 10
    print $ shift (2 :: Int) (-10)
    print $ shiftL (2 :: Int) 10
    print $ shiftL (2 :: Int) (-10)
    print $ shiftR (2 :: Int) 10
    print $ shiftR (2 :: Int) (-10)

    print $ rotate (2 :: Int) 10
    print $ rotate (2 :: Int) (-10)
    print $ rotateL (2 :: Int) 10
    print $ rotateL (2 :: Int) (-10)
    print $ rotateR (2 :: Int) 10
    print $ rotateR (2 :: Int) (-10)
```

### Math constants

``` active haskell
import Numeric.MathFunctions.Constants

main = do
    print m_epsilon
    print m_huge
    print m_tiny
    print m_max_exp
    print m_pos_inf
    print m_neg_inf
    print m_NaN

    print m_1_sqrt_2
    print m_2_sqrt_pi
    print m_ln_sqrt_2_pi
    print m_sqrt_2
    print m_sqrt_2_pi
    print m_eulerMascheroni
```

### Polynomial

``` active haskell
import Numeric.Polynomial
import Data.Vector

polynomial = fromList [1, 2, 3]

main = do
    print polynomial
    print $ evaluatePolynomial 10 polynomial
    print $ evaluateOddPolynomial 10 polynomial
    print $ evaluateEvenPolynomial 10 polynomial
```

### Statistics

``` active haskell
import Statistics.Sample
import Data.Vector
import qualified Statistics.Distribution as D
import Statistics.Distribution.Normal

sample = fromList [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
weightedSample = fromList [(1, 10), (2, 1), (3, 1), (4, 1), (5, 1)]

main = do
    print sample
    print weightedSample

    print $ range sample

    print $ mean sample
    print $ meanWeighted weightedSample
    print $ harmonicMean sample
    print $ geometricMean sample

    print $ variance sample
    print $ fastVariance sample
    print $ varianceWeighted weightedSample
    print $ varianceUnbiased sample
    print $ fastVarianceUnbiased sample
    print $ meanVariance sample
    print $ meanVarianceUnb sample

    print $ stdDev sample
    print $ fastStdDev sample

    print $ standard
    print $ normalDistr 10 5

    print $ D.cumulative standard 0
    print $ D.quantile standard 0.5
    print $ D.density standard 0
    print $ D.mean standard
    print $ D.variance standard
    print $ D.stdDev standard
```

### Collatz sequence

``` active haskell
collatz 1 = [1]
collatz n
    | n `mod` 2 == 0 = n : collatz (n `div` 2)
    | otherwise = n : collatz (3 * n + 1)
    
main = do
    print $ collatz 5
    print $ collatz 17
```

### Monad

``` active haskell
inc n = Just (n + 1)

add1 n = [n + 1]

main = do
    print $ Nothing >> (Just 0)
    print $ (Just 0) >> (Nothing :: Maybe Int)
    print $ (Just 0) >> Nothing >> (Just 1)
    print $ (Just 0) >> (Just 1) >> (Just 2)

    print $ Nothing >>= inc >>= inc >>= inc
    print $ (Just 0) >>= inc >>= inc >>= inc

    print $ [] >> [1, 2]
    print $ [1, 2] >> ([] :: [Int])
    print $ [1] >> [3, 4, 5]
    print $ [1, 2] >> [3, 4, 5]
    print $ [1, 2, 3] >> [3, 4, 5]

    print $ [] >>= add1 >>= add1 >>= add1
    print $ [1, 2, 3] >>= add1
    print $ [1, 2, 3] >>= add1 >>= add1
    print $ [1, 2, 3] >>= add1 >>= add1 >>= add1
```

### State monad

``` active haskell
import Control.Monad.State

inc :: State Int Int
inc = do
    n <- get
    put (n + 1)
    return n

incBy :: Int -> State Int Int
incBy x = do
    n <- get
    modify (+x)
    return n

main = do
    print $ evalState inc 1
    print $ execState inc 1
    print $ runState inc 1
    print $ runState (withState (+3) inc) 1
    print $ runState (mapState (\(a, s) -> (a + 3, s + 4)) inc) 1

    print $ runState (incBy 5) 10
```

### Reader monad

``` active haskell
import Control.Monad.Reader

data Environment = Environment { text1 :: String, text2 :: String }

getText :: Reader Environment String
getText = do
    text1 <- asks text1
    text2 <- asks text2
    return $ text1 ++ ", " ++ text2

main = print $ runReader getText $ Environment "Hello" "world!"
```

### Writer monad

``` active haskell
import Control.Monad.Writer

write :: Int -> Writer [Int] String
write n = do
    tell [1..n]
    return "Done"

main = do
    print $ runWriter $ write 10
    print $ execWriter $ write 10
```

### Church numerals

``` active haskell
import Prelude hiding(succ, exp)

zero s z = z
one s z = s z
two s z = s $ s z
three s z = s $ s $ s z
four s z = s $ s $ s $ s z

five s z = s $ four s z
six s z = s $ five s z
seven s z = s $ six s z

succ x s z = s $ x s z
add x y s z = x s $ y s z
mul x y s z = x (y s) z
exp x y s z = (y x) s z

main = do
    print $ zero (+1) 0
    print $ one (+1) 0
    print $ two (+1) 0
    print $ three (+1) 0
    print $ four (+1) 0
    print $ five (+1) 0
    print $ six (+1) 0
    print $ seven (+1) 0
    print $ succ seven (+1) 0
    print $ add four six (+1) 0
    print $ mul four six (+1) 0
    print $ exp two five (+1) 0

    print $ zero ('*':) ""
    print $ one ('*':) ""
    print $ two ('*':) ""
    print $ three ('*':) ""
    print $ four ('*':) ""
    print $ five ('*':) ""
    print $ six ('*':) ""
    print $ seven ('*':) ""
    print $ succ seven ('*':) ""
    print $ add four six ('*':) ""
    print $ mul four six ('*':) ""
    print $ exp two five ('*':) ""
```

### Version

``` active haskell
import Data.Version

main = print $ showVersion
    Version {
        versionBranch = [1, 2, 3, 4],
        versionTags = ["Tag1", "Tag2", "Tag3"]
    }
```

Visit [Winsoft](http://www.winsoft.sk) web site.