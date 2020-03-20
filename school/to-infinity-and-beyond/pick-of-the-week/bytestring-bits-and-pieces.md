# ByteString Bits and Pieces

## ByteString Basics

[ByteString](http://hackage.haskell.org/package/bytestring) provides a more efficient alternative to Haskell's built-in `String` which can be used to store 8-bit character strings and also to handle binary data. It provides alternative versions of functions such as `readFile` and also equivalents of standard list manipulation functions:

``` active haskell
{-# START_FILE Main.hs #-}
import qualified Data.ByteString as B

main = do
    contents <- B.readFile "foo.txt"
    print $ B.reverse contents
{-# START_FILE foo.txt #-}

... em esreveR
```


### Characters or bytes?

Depending on the context, we may prefer to view the `ByteString` as being made up of a list of elements of type `Char` or of `Word8` (Haskell's standard representation of a byte). There's only one `ByteString` data structure for both, but the library exposes different functions depending on how we want to interpret the contents:

``` active haskell
import qualified Data.ByteString as B
import qualified Data.ByteString.Char8 as BC

bytestring = BC.pack "I'm a ByteString, not a [Char]"

bytes = B.unpack bytestring
chars = BC.unpack bytestring

main = do
    BC.putStrLn bytestring
    print $ head bytes
    print $ head chars
```

Here we've used the `pack` function to convert a `String` into a `ByteString` and then used two different `unpack` functions to get back both a list of `Char`s (the original `String`) and a list of `Word8`s. `Data.ByteString` provides the `Word8` functions while `Data.ByteString.Char8` provides the `Char` equivalents.

Of course we don't need to unpack the `ByteString` to a list to get the first element. We can just use the `head` functions provided by the library itself 

``` active haskell
import qualified Data.ByteString as B
import qualified Data.ByteString.Char8 as BC

bytestring = BC.pack "I'm a ByteString, not a [Char]"

main = do
    BC.putStrLn bytestring
    print $ B.head bytestring
    print $ BC.head bytestring
    
```

### ByteStrings and Unicode

ByeString character functions only work with ASCII text, hence the `Char8` in the package name. If you try and use unicode Strings it will mess up:

``` active haskell
import qualified Data.ByteString.Char8 as BC

hello      = "你好"
helloBytes = BC.pack hello

main = do
    putStrLn hello
    BC.putStrLn helloBytes
    print $ BC.length helloBytes
```
If you are working with unicode, you should use the [`Text`](http://hackage.haskell.org/package/text) package.



### Lazy ByteStrings

`ByteString` also has a lazy version, which is a better choice if you are processing large amounts of data and don't want to read it all into memory at once. Just import `Data.ByteString.Lazy` instead of `Data.ByteString`. Sometimes you will find libraries which use one type when you are using the other. For example, [Aeson](http://hackage.haskell.org/package/aeson) uses lazy ByteStrings, but you may only be dealing with small JSON snippets and want to write your own code using the strict version. You can convert between them easily enough if you have to:

``` active haskell
import qualified Data.ByteString as B
import qualified Data.ByteString.Lazy as BL
import qualified Data.ByteString.Char8 as BC
import qualified Data.ByteString.Lazy.Char8 as BLC

strict = BC.pack "I'm a strict ByteString (or am I)"
lazy = BLC.pack "I'm a lazy ByteString (or am I)"

strictToLazy = BL.fromChunks [strict]
lazyToStrict = B.concat $ BL.toChunks lazy

main = do
    BLC.putStrLn strictToLazy
    BC.putStrLn lazyToStrict

```

Newer versions of the library have `toStrict` and `fromStrict` functions in the `Data.ByteString.Lazy` module which you can use instead.

### The `OverloadedStrings` Language Extension

When you enter a string literal, Haskell will normally assume it is of type `String` (`[Char]`). This useful language extension allows us to have string literals interpreted as `ByteString`s, provided we import `Data.ByteString.Char8`:


``` active haskell
{-# LANGUAGE OverloadedStrings #-}

import Data.ByteString.Char8 ()
import qualified Data.ByteString as B

bytes = "I'm a ByteString, not a [Char]" :: B.ByteString

str   = "I'm just an ordinary [Char]"    :: String

main = do
  print bytes
  print str

```
As you can see here, we might have to add explicit types in some cases to let Haskell know which kind of string we want. In `ghci`, you can get the same behaviour by starting it using:

```
ghci -XOverloadedStrings
```


## ByteString binary data

Manipulating binary data is easy with `ByteString`. In fact, these notes are really a collection of bits and pieces I picked up along the way while doing the exercises for Coursera's [Cryptography I](https://www.coursera.org/course/crypto) and had to use `ByteString` for the first time.


### Hex and Base64 Encoding

Binary data is often encoded as hex or base64 to provide an ASCII text representation, so we need an easy way of decoding these to a `ByteString` containing the bare bytes. This is exactly what the [base16-bytestring](http://hackage.haskell.org/package/base16-bytestring) and [base64-bytestring](http://hackage.haskell.org/package/base64-bytestring) packages were written for.

Here's an example for base64:

``` active haskell
{-# LANGUAGE OverloadedStrings #-}

import Data.ByteString.Char8 ()
import qualified Data.ByteString as B

import Data.ByteString.Base64 (encode, decode)

Right bytes = decode "SSdtIGEgYmFzZTY0IGVuY29kZWQgQnl0ZVN0cmluZw=="

main = print bytes

```

And one for a hex-encoded string:

``` haskell
{-# LANGUAGE OverloadedStrings #-}

import Data.ByteString.Base16 (encode, decode)

bytes = fst $ decode "49276d2061206865782d656e636f6465642042797465537472696e6720286f722077617329"

main = print bytes
```

Unfortunately, base16-bytestring isn't available in Stackage yet, so we can't use active code here. 

### One-Time Pad

If you want to XOR one bytestring against another, to implement one-time pad encryption for example, you can use `zipWith`:

```active haskell
{-# LANGUAGE OverloadedStrings #-}

import Data.ByteString.Char8 ()
import qualified Data.ByteString as B
import Data.ByteString.Base64 (decode)
import Data.Bits (xor)

Right key = decode "kTSFoLQRrR+hWJlLjAwXqOH5Z3ZLDWray5mBgNK7lLuHdTwab8m/v96y"

encrypt = B.pack . B.zipWith xor key
decrypt = encrypt

main = do
    let encrypted = encrypt "I'm a secret message"
    print encrypted
    print $ decrypt encrypted
```

That's about it. You can view the full [package documentation](http://hackage.haskell.org/package/bytestring) to see what other functions are available.

