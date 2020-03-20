# Template Haskell 101

Learning Template Haskell has been on my [Fun List](https://www.facebook.com/notes/kent-beck/the-fun-list/533672119998990) for quite a while, but for some reason I never got to it.
Recently however, while looking for a reasonable way to include multiline strings in Haskell programs, I stumbled upon, at first strange [advice](http://www.haskell.org/haskellwiki/Poor_man%27s_here_document#Quasiquoting): use Template Haskell.


So I put together this simple test program (mostly copying from the wikipage mentioned above):

``` active haskell
{-# START_FILE Str.hs #-}
module Str(str) where
 
import Language.Haskell.TH
import Language.Haskell.TH.Quote
 
str = QuasiQuoter { quoteExp = stringE }

{-# START_FILE Main.hs #-}
{-# LANGUAGE QuasiQuotes #-}
module Main where
import Str
 
longString = [str|This is a multiline string.
It's many lines long.
 
 
It contains embedded newlines. And Unicode:
 
Ἐν ἀρχῇ ἦν ὁ Λόγος
 
It ends here: |]
 
main = putStrLn longString
   
```

This sparked my curiosity (hey, I need to understand this), and finally gave me the excuse to learn TH. The QuasiQuote stuff (which is an extension of TH) I will get to later, first let's start with TH itself.

One [tutorial](http://www.hyperedsoftware.com/blog/entries/first-stab-th.html) advised exploring in GHCi (whih I love to do) so I started with that.

```
$ ghci -XTemplateHaskell

> :m +Language.Haskell.TH
> runQ [| \x -> 1 |]

LamE [VarP x_0] (LitE (IntegerL 1))

> :t it
it :: Exp

```

So we can (kind of) parse Haskell at runtime, which is good and well but what is more interesting is the other direction: we can construct a syntax tree and inject (aka ``splice'') it into our program

```
> runQ [| succ 1 |]
AppE (VarE GHC.Enum.succ) (LitE (IntegerL 1))
Prelude Language.Haskell.TH> $(return it)
2

> $(return (LitE (IntegerL 42)))
42

```

names are just a trifle more involved

```
> $( return (AppE (VarE (mkName "succ")) (LitE (IntegerL 1))))
2

```

So far we've been only constructing expressions, but other syntax elements, like patterns and declarations can be constructed too:

```
> runQ [d| p1 (a,b) = a |]
[FunD p1_0 [Clause [TupP [VarP a_1,VarP b_2]] (NormalB (VarE a_1)) []]]
```

For technical reasons, definitions cannot be used in the same module they are built, so we need at least two modules to demonstrate this in a real program:

``` active haskell
{-# START_FILE Build1.hs #-}
{-# LANGUAGE TemplateHaskell #-}
module Build1 where
import Language.Haskell.TH

build_p1 :: Q [Dec]
build_p1 = return
    [ FunD p1 
             [ Clause [TupP [VarP a,VarP b]] (NormalB (VarE a)) []
             ]
    ] where
       p1 = mkName "p1"
       a = mkName "a"
       b = mkName "b"
       
{-# START_FILE Declare1.hs #-}       
{-# LANGUAGE TemplateHaskell #-}
import Language.Haskell.TH
import Build1

$(build_p1)

main = print $ p1 (1,2)
```

I will not explain what `FunD` etc means, and instead refer you to the [documentation](http://hackage.haskell.org/packages/archive/template-haskell/2.8.0.0/doc/html/Language-Haskell-TH.html#g:11).

Building and transforming syntax trees for a language with bindings is usually a pain, because you have to keep remembering to avoid capture. Luckily, TH makes it easy with the function [newName](http://hackage.haskell.org/packages/archive/template-haskell/2.8.0.0/doc/html/Language-Haskell-TH.html#v:newName):

```
newName :: String -> Q Name
```

(which, by the way explains one of the reasons why [Q](http://hackage.haskell.org/packages/archive/template-haskell/2.8.0.0/doc/html/Language-Haskell-TH.html#t:Q) needs to be a monad).

So let us make our example capture-proof (even if it seems unnecessary right now). 
Note that `p1` is a global name and still needs to be named just that with `mkName`, while `a` and `b` can be any names, so we generate them with `newName`

``` active haskell
{-# START_FILE Build2.hs #-}
{-# LANGUAGE TemplateHaskell #-}
module Build2 where
import Language.Haskell.TH
--show
build_p1 :: Q [Dec]
build_p1 = do
  let p1 = mkName "p1"  
  a <- newName "a"
  b <- newName "b"
  return
    [ FunD p1 
             [ Clause [TupP [VarP a,VarP b]] (NormalB (VarE a)) []
             ]
    ]
--/show    

{-# START_FILE Declare2.hs #-}  
{-# LANGUAGE TemplateHaskell #-}
import Language.Haskell.TH
--show
import Build2

$(build_p1)

main = print $ p1 (1,2)

```

What we have done so far served little purpose (except purely educational), but let us tackle a real problem: define all projections for big (say 16-) tuples. Doing this manually would be obviously a major PITA, but Template Haskell can help.

Actually, let us start smaller and define both projections for pairs first. Extending this to any number is then a simple exercise.


First we might want abstract declaring a simple function, e.g.

```
simpleFun name pats rhs = FunD name [Clause pats (NormalB rhs) []]
```

Then, if I have a function such that `build_p n` builds nth definition, I can build all $n$ using <hoogle>mapM</hoogle>:

```
build_ps = mapM build_p [1,2]
```

``` active haskell
{-# START_FILE Build3.hs #-}
{-# LANGUAGE TemplateHaskell #-}
module Build3 where
import Language.Haskell.TH

simpleFun :: Name -> [Pat] -> Exp -> Dec
simpleFun name pats rhs = FunD name [Clause pats (NormalB rhs) []]

--show
build_ps = mapM build_p [1,2] where
    fname n = mkName $ "p2_" ++ show n
    argString k = "a" ++ show k
    argStrings = map argString [1,2]
    build_p n = do    
        argNames <- mapM newName argStrings
        let args = map VarP argNames
        return $ simpleFun (fname n) [TupP args] (VarE (argNames !! (n-1)))
--/show
{-# START_FILE Declare3.hs #-}  
{-# LANGUAGE TemplateHaskell #-}

import Language.Haskell.TH

import Build3
--show
$(build_ps)

main = mapM_ print 
  [ p2_1 (1,2)
  , p2_2 (1,2)
  ]
--/show

```

In the [next episode](https://www.fpcomplete.com/user/marcin/quasiquotation-101) we will look at quasiquotation.