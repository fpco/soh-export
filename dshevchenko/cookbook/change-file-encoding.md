# Change file encoding

We have some file in some encoding (for example, `UTF16BE`), ans we want to convert it to `UTF-8` encoding.

### Solution

We have to install packages [`iconv`](http://hackage.haskell.org/package/iconv) and [`bytestring`](http://hackage.haskell.org/package/bytestring). Now:

```haskell
module Main where

import Codec.Text.IConv (convert)
import Data.ByteString.Lazy as Lazy
import Data.ByteString as Strict

main :: IO ()
main = do
    let path = "/Users/dshevchenko/f4.cpp"
    -- Read a file lazily, not strictly...
    text <- Strict.readFile path
    let convertedText = convert "UTF-16BE" "UTF-8" (fromStrict text)
    -- Write in the same file...
    Lazy.writeFile path convertedText
```

That's all.

### Remark


`iconv` package is a wrapper around a Unix-utility `iconv`. So you can view a complete list of supported encodings, just run:

```bash
$ iconv -l
```

