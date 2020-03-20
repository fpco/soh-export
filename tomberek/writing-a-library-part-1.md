# Writing a library - part 1

We will create a library to implement a matching algorithm for a production system using [this paper](https://lirias.kuleuven.be/bitstream/123456789/259239/3/TKDESI-2009-03-0153.R1.pdf). We will document the thought process and translation of concepts in the paper into Haskell code.

A production system allows one to define many rules which describe the behavior of a system. The goal of this algorithm will be to efficiently "fire" the rules with changing conditions.

Starting with section 2 "A Core Rule Language" is composed of facts and rules.

> Each fact is a runtime instance of a class. A class has a unique name, and declares a number of fields, each with a type and an optional field name. For this article, standard boolean, string, integer, and float types suffice. Once asserted, the values of a fact’s fields can no longer be changed.

Translated to Haskell conventions:

``` active haskell
data Fact name fields = Fact name (HList fields) deriving Show

data Edge
type EdgeFact = Fact Edge (HList [Int,Int,Bool,String,Bool])

main = print $ Fact Edge (HCons 3 $ HCons 2 $ HCons True $ HCons "hi" $ HCons False HNil)
```

