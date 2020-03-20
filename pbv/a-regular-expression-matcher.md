# A regular expression matcher

*Regular expressions* (regexps) are a both a theoretical model of
computation and a practical basis for language processing (e.g. the
Unix command-line tools `grep` and `awk`, scripting languages such as
`bash` and `perl` and the `lex` and `alex` compiler-building tools).

This tutorial we are going to build a simple regexp matcher in
Haskell. The key to expressing matching in an elegant and compositional
way is to use a combination of *algebraic datatypes* and programming
with *continuations*. 
Note, however, that the emphasis of this tutorial is on clarity, *not* optimization;
real-world code should rely on an industrial-strengh  
library such as `regexp-pcre`. 

This tutorial was inspired by Olivier Danvy's
[Defunctionalization at Work](http://www.brics.dk/RS/01/23)
(a BRICS technical report).

## Regular expressions

> I define UNIX as "30 definitions of regular expressions living under
> one roof." --- Don Knuth

Although practical tools adopt extended definitions of regular
expressions, we will consider only simple *regexps* built as follows:

- 0 that matches nothing (the empty language);
- 1 (or *epsilon*) that matches the empty string;
- a single character *c* matching itself;
- the union (+) of two *regexp*s;
- the concatenation (.) of two *regexp*s;
- the zero-or-more repetition (\*) of a *regexp* (also called *Kleene closure*).

Concatenation, union and repetition are standard in practical 
tools such as `grep`. Constants 0 and 1 are necessary to ensure 
good algebraic properties (every language recognized by an automaton
can be represented by a regexp) and sometimes ommitted in practice:
e.g. in `grep` the empty string
is represented by `^$` but there is no representation for the empty language. 

Instead of re-using some existing type to encode regexps (e.g. strings) we
begin by defining a custom recursive datatype.
This will make it easier to process regexps and define the matching algorithm.
```haskell
data Regexp = Zero                  -- empty
            | One                   -- epsilon
            | Lit Char              -- single character
            | Plus Regexp Regexp    -- union (+)
            | Cat  Regexp Regexp    -- concatenation (.)
            | Many Regexp           -- repetition (*)
```

Some examples of regexp together with a description of what they match:
```haskell
Lit 'a'                        -- an 'a'
Plus (Lit 'a') (Lit 'b')       -- an 'a' or a 'b'
Cat (Many (Lit 'a')) (Lit 'b') -- b, ab, aab, aaab, ...
```
Note that *any* value of the `Regexp` data type (except for non-terminating ones)
corresponds to a valid regular expression. This means
that ill-formed regexps are avoided simply by type checking!

However, writting complex regexps this way is verbose and
error-prone. We therefore introduce some shortcuts.

First, we define two infix operators for 
union and concatenation of regexps. 
We can also use some algebraic properties 
such as `0+e = e+0 = e` and `1.e = 1.e = e`
to do some automatic simplification.
We also define a "smart" constructor for repetition.

```haskell
infixl 6 <+>
infixl 7 <>

(<+>) :: Regexp -> Regexp -> Regexp
Zero <+> e = e
e <+> Zero = e
e1 <+> e2  = Plus e1 e2

(<>) :: Regexp -> Regexp -> Regexp
Zero <> _   = Zero
_ <> Zero   = Zero
One <> e    = e
e <> One    = e
e1 <> e2    = Cat e1 e2

many :: Regexp -> Regexp 
many Zero     = One
many One       = One
many (Many e)  = Many e
many e         = Many e
```

Second, we employ the `OverloadedStrings` GHC extension to automatically 
convert any string literal to a regexp.
For example, the string `"abc"` is 
converted to a concatenation of characters:
```haskell
Cat (Lit 'a') (Cat (Lit 'b') (Lit 'c'))
```
(*To see how this is achived open the full code window on the runnable 
example at the end.*)

Using these operators we can write regexps quite succintly:
``` haskell
"ab" 
"a"<+>"b"
"ab" <> many ("a"<+>"b")
```


## Matching

Our objective is to define a matching function, i.e. a function that
takes a regexp and a string and yields a boolean. 
```haskell
match :: Regexp -> String -> Bool
```

However, if we try to define the `match` function directly
by recursion on the `Regexp` data type we run into
problems. 
For example, to match the concatenation of two
regexps, we would have to split the input string:
```haskell
match (Cat e1 e2) cs = match e1 cs1 && match e2 cs2
    -- incomplete: missing some cs1, cs2 such that cs=cs1++cs2
```
The problem is that `match` does not yield how much of the input
string was matched.

One solution would be to have `match` yield a pair e.g. of
a boolean and a string.
But another more elegant solution is to define a "worker" function
taking an extra parameter called the *continuation* 
that specifies what to do with the non-matched part of
the string; in this case, the continuation is a function from 
strings to booleans (the result of matching). 
For redability we define a type synomym for continuations;
```haskell
type Cont = String -> Bool   -- type for continuations
```

We can now define the worker function `accept` by case-analysis on the regexp;
note that `accept` (unlike `match`) is higher-order because the 
continuation argument `k` is a function.
The top-level function `match` simply calls `accept` with a continuation that checks
for the empty string (i.e. the `null` function from the standard Prelude).
```haskell
accept :: Regexp -> String -> Cont -> Bool  -- worker function
accept Zero    cs      k = False
accept One     cs      k = k cs
accept (Lit c) (c':cs) k = c==c' && k cs
accept (Lit c) []      k = False
accept (Cat e1 e2) cs  k = accept e1 cs (\cs' -> accept e2 cs' k)
accept (Plus e1 e2) cs k = accept e1 cs k || accept e2 cs k
accept (Many e) cs k     = acceptMany e cs k
  where 
     acceptMany e cs k 
       = k cs || accept e cs (\cs' -> cs'/=cs && acceptMany e cs' k)
```

The case of `Zero` always fails while `One` success and applies the continuation
to the remaining string. The case for single character checks the start of the
string and applies the continuation. The case for union is trivial.
Concatenation is more interesting: we simply call `accept` for the first regexp
*e1* with a continuation that calls `accept` for *e2* (and then proceeds to
the original continuation).
Finally, we use an auxiliary definition `acceptMany` for matching the repetition.


The following example allows experimenting with the matcher;
try changing the string or the regexp and re-running the program.

```active haskell
{-# LANGUAGE OverloadedStrings #-}
import GHC.Exts (IsString(..))

data Regexp = Zero                  -- empty
            | One                   -- epsilon
            | Lit Char              -- single character
            | Plus Regexp Regexp    -- union (+)
            | Cat  Regexp Regexp    -- concatenation (.)
            | Many Regexp           -- repetition (*)
            deriving Show

infixl 6 <+>
infixl 7 <>

(<+>) :: Regexp -> Regexp -> Regexp
Zero <+> e = e
e <+> Zero = e
e1 <+> e2  = Plus e1 e2

(<>) :: Regexp -> Regexp -> Regexp
Zero <> _   = Zero
_ <> Zero   = Zero
One <> e    = e
e <> One    = e
e1 <> e2    = Cat e1 e2

many :: Regexp -> Regexp 
many Zero     = One
many One       = One
many (Many e)  = Many e
many e         = Many e

type Cont= String -> Bool

accept :: Regexp -> String -> Cont -> Bool  -- worker function
accept Zero    cs      k = False
accept One     cs      k = k cs
accept (Lit c) (c':cs) k = c==c' && k cs
accept (Lit c) []      k = False
accept (Cat e1 e2) cs  k = accept e1 cs (\cs' -> accept e2 cs' k)
accept (Plus e1 e2) cs k = accept e1 cs k || accept e2 cs k
accept (Many e) cs k     = acceptMany e cs k
  where 
     acceptMany e cs k 
       = k cs || accept e cs (\cs' -> cs'/=cs && acceptMany e cs' k)


match :: Regexp -> String -> Bool
match re s = accept re s null

instance IsString Regexp where
  fromString cs = foldr ((<>) . Lit) One cs

-- show
main = print (match ("ab" <> many "ba") "abbaba")
-- /show
```


## Exercises

1. Define auxiliary functions `plus` and `option`
   for the `+` and `?` operators in `grep`, i.e. one-or-more times 
   repetition and optional matching.
   @@@
   Here are one-line solutions that re-use the previous functions.
   ```haskell
   plus, option :: Regexp -> Regexp
   plus e = e <> many e
   option e = "" <+> e
   ```
   @@@

2. The function `acceptMany` for handling repetition includes a condition 
   `cs'/=cs` to check that characters are consumed in the recursive case. 
   However, checking list inequality requires traversing the full list
   in the worst-case. Find a way to avoid this inefficient check.
   @@@
   It is sufficient to pair each character with its *index*
   and replace the check that the inequality on indices rather than string.
   The type of continuations and the worker function need to be modified;
   the relevant change is the case of the repetition operator:
   ```haskell
   type Cont = [(Int,Char)] -> Bool   -- new type for continuations
   
   accept :: Regexp -> [(Int,Char)] -> Cont -> Bool
   accept (Many e) ics k  = acceptMany e ics k
      where 
        acceptMany e ics k 
           = k ics || accept e ics (\ics' -> fst ics'/=fst ics && acceptMany e ics' k)
   -- other cases are straightforward

   match :: Regexp -> String -> Bool
   match re cs = accept re (zip [0..] cs) null
   ```
   @@@

