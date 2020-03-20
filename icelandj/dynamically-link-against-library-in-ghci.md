# Dynamically link against library in GHCi

This post reviews a small feature of GHCi version 7.10 in order to promote it:

> [1.5.2.3. GHCi](https://downloads.haskell.org/~ghc/7.10.1-rc3/docs/html/users_guide/release-7-10-1.html#idp5818560)
> 
> * It's now possible to use `:set -l{foo}` in `GHCi` to link against a foreign library after startup.

# Before 7.10

Before 7.10 library flags had to be passed as command-line arguments to `ghci`. If we wanted to call [`zlibVersion`](http://refspecs.linuxbase.org/LSB_3.1.1/LSB-Core-generic/LSB-Core-generic/zlib-zlibversion-1.html) from Haskell

> `#include <zlib.h>`
> 
> `const char * zlibVersion (void);`

we would first create a Haskell file

``` haskell
{-# START_FILE Version.hs #-}
import Foreign.C.String

foreign import ccall "zlibVersion"
  c_zlibVersion :: IO CString

zlibVersion :: IO String
zlibVersion = do
  version <- c_zlibVersion
  peekCString version
```

and then load it with `ghci`

``` haskell
$ ghci -l{-hi-}z{-/hi-} Version.hs
…
ghci> :t c_zlibVersion 
c_zlibVersion :: IO CString
ghci> :t zlibVersion 
zlibVersion :: IO String
ghci> zlibVersion
"1.2.3.4"
```

`z` is the shared library we use which will load the shared library `libz.so` ([further information](https://downloads.haskell.org/~ghc/7.10.1-rc3/docs/html/users_guide/ghci-invocation.html#idp6876624)).


Having to specify your libraries up front can be inconvenient if you want to load libraries mid-session or if your editor or IDE manages `ghci`'s arguments.

# Now

In GHCi 7.10 one can write:

``` haskell
ghci> :set -l{-hi-}z{-/hi-}
ghci> :load Version.hs
…
ghci> zlibVersion 
"1.2.3.4"

```

# Dynamically loading your own shared library

Quick example:

``` haskell
{-# START_FILE /tmp/succ.c #-}
#include <stdio.h>

int foo(int x)
{
  static int var = 0;

  var += x;

  printf("Hey %d\n", var);
  return x + 1;
}
```

compiled with

> `$ clang -fPIC -shared -o /tmp/libsucc.so /tmp/succ.c`

and invoked with

``` haskell
ghci> import Control.Monad
ghci> import Foreign.C.Types 
ghci> 
ghci> :set -l{-hi-}succ{-/hi-} -L{-hi-}/tmp{-/hi-}
ghci> foreign import ccall foo :: CInt -> IO CInt
ghci>
ghci> replicateM 5 (foo 10)
Hey 10
Hey 20
Hey 30
Hey 40
Hey 50
[11,11,11,11,11]
ghci> replicateM 4 (foo 3)
Hey 53
Hey 56
Hey 59
Hey 62
[4,4,4,4]
```

# Development

* **Ticket #1407** *[Add the ability to `:set -l{foo}` in `.ghci` files ](https://ghc.haskell.org/trac/ghc/ticket/1407)*
* **Phabricator D194** *[Add the ability to `:set -l{foo}` in `.ghci` files](https://phabricator.haskell.org/D194)*