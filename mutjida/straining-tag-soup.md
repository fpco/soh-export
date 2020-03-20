# Straining Tag Soup

## Introduction
[Tagsoup](http://hackage.haskell.org/package/tagsoup-0.12.8) is a package for parsing strings of xml into a list of tag elements.
```haskell
-- | A single HTML element. A whole document is represented by a list of @Tag@.
--   There is no requirement for 'TagOpen' and 'TagClose' to match.
data Tag str =
     TagOpen str [Attribute str]  -- ^ An open tag with 'Attribute's in their original order
   | TagClose str                 -- ^ A closing tag
   | TagText str                  -- ^ A text node, guaranteed not to be the empty string
   | TagComment str               -- ^ A comment
   | TagWarning str               -- ^ Meta: A syntax error in the input file
   | TagPosition !Row !Column     -- ^ Meta: The position of a parsed element
     deriving (Show, Eq, Ord, Data, Typeable)
     
parseTags :: StringLike str => str -> [Tag str] 
```
There are functions for splitting a list of `Tag`s into sections whose first item matches a predicate; but there are no functions for searching/filtering the xml objects; there is module which converts a list of tags into a tree of tags, but it is provisional and has limited support for searching.

This tutorial will show how to create a simple, yet effective, library for searching/filtering a list of xml objects. I will first show a novel technique for transforming a list of `Tag`s into a list of objects using a generalization of `unfoldr`. Then I will create a small DSL for predicates over these objects; this DSL will make use of Applicatives.

## Objects

First I'll need a type to represent xml objects.
```haskell
data Object = Object {
      srcFile :: FilePath,
      kind :: String,
      attrs :: [Attribute String],
      text :: String,
      kids :: [Object]
    } 
  deriving (Eq, Show)
```
I include a source file field in anticipation of reading in xml from several files.

I will also have a few convenience functions for `Object`s.
```haskell
emptyObj = Object {
             srcFile = "",
             kind = "",
             attrs = [],
             text = "",
             kids = []
           }

addKid :: Object -> Object -> Object
addKid obj kid = obj{kids = kids obj ++ [kid]}

getAttr :: String -> Object -> Maybe String
getAttr nm = lookup nm . attrs

showObj :: Object -> String
showObj obj = "("++kind obj++
              ", attrs="++show (attrs obj)++
              ", text="++text obj++
              ", kids=["++intercalate "," (map showObj $ kids obj)++"])"
```

I will parse a list of `Tag`s into an `Object` and the remaining list of `Tag`s
I would like to structure this function in a manner similar to `unfoldr`
```haskell
unfoldr :: (b -> Maybe (a, b)) -> b -> [a]
```
where I have a function to produce the next `Object` and the rest of the stream, and I use a combinator like `unfoldr` to handle the recursion and produce all the `Object`s. However, I can not use `unfoldr`; since it doesn't return the rest of the stream when it is done, I can not use it to handle the recursive calls parse the child `Object`s. So I will use a generalization of `unfoldr`
```haskell
unfoldGen :: (a -> Either c (b, a)) -> a -> ([b], c)
unfoldGen f = either ([],) (\(v, x') -> first (v:) (unfoldGen f x')) . f
```
where `either` is from `Data.Either` and `first` is from `Control.Arrow`.

I can now use a function `nextObj` of type
```haskell
nextObj :: FilePath -> [Tag String] -> Either [Tag String] (Either Object String, [Tag String])
```
with `unfoldGen` to turn a list of `Tag`s into a list of `Object`s. The `Either Object String` is necessary in `nextObj` since there can be both text and objects inside an object.

```haskell
nextObj :: FilePath -> [Tag String] -> Either [Tag String] (Either Object String, [Tag String])
nextObj file [] = Left []
nextObj file (t:ts) = case t of
    TagOpen nm ats | nm /= "?xml" -> Right (Left obj, popTagClose ts')
      where (res, ts') = unfoldGen (nextObj file) ts
            obj = Object{srcFile = file, 
                         kind = nm, 
                         attrs = ats,
                         text = concat $ rights res,
                         kids = lefts res
                        }
            -- silently ignore orphaned tags
            popTagClose = drop 1 . dropWhile (not . isTagCloseName nm)

    TagClose _ -> Left (t:ts)

    TagText txt | all isSpace txt -> nextObj file ts -- ignore just whitespace text
                | otherwise -> Right (Right txt, ts)

    _ -> nextObj file ts -- ignore all other TagSoup tags
```

I can now write `tagsToObjs` to transform a list of `Tag`s to a list of `Object`s
```haskell
tagsToObjs :: FilePath -> [Tag String] -> [Object]
tagsToObjs file = lefts . fst . unfoldGen (nextObj file)
```

I can now put everything together to get a complete program.

```active haskell
--/show
{-# LANGUAGE TupleSections #-}
import Control.Applicative
import Control.Arrow
import Data.Char
import Data.Either
import Data.List
import Data.Maybe
import qualified Data.Text as T
import qualified Data.Text.IO as T
import Debug.Trace
import System.FilePath
import Text.HTML.TagSoup


data Object = Object {
      srcFile :: FilePath,
      kind :: String,
      attrs :: [Attribute String],
      text :: String,
      kids :: [Object]
    } 
  deriving (Eq, Show)

emptyObj = Object {
             srcFile = "",
             kind = "",
             attrs = [],
             text = "",
             kids = []
           }

addKid :: Object -> Object -> Object
addKid obj kid = obj{kids = kids obj ++ [kid]}

getAttr :: String -> Object -> Maybe String
getAttr nm = lookup nm . attrs

showObj :: Object -> String
showObj obj = "("++kind obj++
              ", attrs="++show (attrs obj)++
              ", text="++text obj++
              ", kids=["++intercalate "," (map showObj $ kids obj)++"])"


unfoldGen :: (a -> Either c (b, a)) -> a -> ([b], c)
unfoldGen f = either ([],) (\(v, x') -> first (v:) (unfoldGen f x')) . f

nextObj :: FilePath -> [Tag String] -> Either [Tag String] (Either Object String, [Tag String])
nextObj file [] = Left []
nextObj file (t:ts) = case t of
    TagOpen nm ats | nm /= "?xml" -> Right (Left obj, popTagClose ts')
      where (res, ts') = unfoldGen (nextObj file) ts
            obj = Object{srcFile = file, 
                         kind = nm, 
                         attrs = ats,
                         text = concat $ rights res,
                         kids = lefts res
                        }
            -- silently ignore orphaned tags
            popTagClose = drop 1 . dropWhile (not . isTagCloseName nm)

    TagClose _ -> Left (t:ts)

    TagText txt | all isSpace txt -> nextObj file ts -- ignore just whitespace text
                | otherwise -> Right (Right txt, ts)

    _ -> nextObj file ts -- ignore all other TagSoup tags

tagsToObjs :: FilePath -> [Tag String] -> [Object]
tagsToObjs file = lefts . fst . unfoldGen (nextObj file)

-- show Parse a file into a list of Objects. 
getFileObjs :: FilePath -> IO [Object]
getFileObjs file = tagsToObjs file . parseTags . T.unpack <$> T.readFile file

conv = "<conversation><hello comment='boring'></hello><goodbye>boa viagem</goodbye></conversation>"
test = "<song notes='at the end'><verse>eu vou embora</verse><chorus>boa viagem</chorus><verse>a deus</verse></song>"

main = do
  putStrLn $ "tags=\n" ++ show (parseTags $ test ++ conv)
  putStrLn ""
  mapM_ (putStrLn . ("obj=\n"++) . showObj) $ tagsToObjs "" $ parseTags $ conv ++ test 
  putStrLn "\nok"
```

## Predicates

Now that I can create a list of `Object`s, it'd be nice to search and filter the objects.

I'll start with the simplest way to get a list of sub-`Object`s satisfying a given predicate.
```haskell
findObjs :: (Object -> Bool) -> Object -> [Object]
findObjs p obj | p obj = obj : objs
               | otherwise = objs
  where objs = concatMap (findObjs p) (kids obj)
```
Now I can write pretty much any kind of search I want. 

Here are some basic handy predicates.
```haskell
anyObj = const True

kindIs k = (== k) . kind

textVal p = p . text

letVal getVal useVal obj = useVal (getVal obj) obj
```

Here are some basic building blocks for predicates on attributes.
```haskell
genAttr p = isJust . find p . attrs

-- Note: Attributes are just pairs in TagSoup
p ! q = genAttr (\(k,v) -> p k && q v)

attrName p = genAttr (p . fst)

attrVal p = genAttr (p . snd)
```

I can also define compound predicates.
```haskell
(<&&> :: (Object -> Bool) -> (Object -> Bool) -> Object -> Bool
p1 <&&> p2 = \obj -> p1 obj && p2 obj

(<||> :: (Object -> Bool) -> (Object -> Bool) -> Object -> Bool
p1 <||> p2 = \obj -> p1 obj || p2 obj
```
However, it is slightly inconvenient to have to use named variables when using compound predicates. So I will use the fact that `((->) a)` is an instance of `Applicative` to tidy up my predicates.
```haskell
type ObjPred = Object -> Bool

(<&&>) :: ObjPred -> ObjPred -> ObjPred
p1 <&&> p2 = (&&) <$> p1 <*> p2
infixl 6 <&&>

(<||>) :: ObjPred -> ObjPred -> ObjPred
p1 <||> p2 = (||) <$> p1 <*> p2
infixl 5 <||>
```

Note that I can compose with regular `Boolean` functions.
```haskell
not . (kindIs "a" <&&> not . attrName (== "bad") :: ObjPred
```

I will now add predicates on ancestors and descendants of a given object; i.e. I'd like to be able to find an object which has a descendant satisfying some property. In order to do this, I will have to have access to both parents and descendants of a given object. One way to accomplish this is to add a parent field to `Object`s. However, this will complicate object traversal and serialization (which I mightbe interested in). So, I will opt for adding the traversal history to the find function.

I will start by redefining `ObjPred` to take some state (the path to the current object).
```haskell
data ObjPredState = ObjPredState{curObj :: Object, prevObjs :: [Object]}

type ObjPred = ObjPredState -> Bool
```
and I will use a few convenience functions on `ObjPredState`.
```haskell
newObjPredState :: Object -> ObjPredState
newObjPredState obj = ObjPredState{curObj = obj, prevObjs = []}


nextObjPredStates :: ObjPredState -> [ObjPredState]
nextObjPredStates objPredSt = map modOPS (kids obj)
  where obj = curObj objPredSt
        modOPS o = ObjPredState{curObj = o, prevObjs = obj : prevObjs objPredSt}


prevObjPredState :: ObjPredState -> Maybe ObjPredState
prevObjPredState objPredSt = case prevObjs objPredSt of
                               [] -> Nothing
                               (x:xs) -> Just ObjPredState{curObj = x, prevObjs = xs}
```
Then I can redefine `findObjs` to keep track of the paths.
```haskell
findObjs' :: ObjPred -> Object -> [ObjPredState]
findObjs' p = go . newObjPredState where
  go opSt | p opSt = opSt : opSts
          | otherwise = opSts
    where opSts = concatMap go $ nextObjPredStates opSt
    
findObjs :: ObjPred -> Object -> [Object]
findObjs p = map curObj . findObjs' p
```
I have separated out `findObjs'` from `findObjs` since `findObjs'` is more basic and might be needed to write some predicates (i.e. those which require looking at the path to this object).

Now I can write the following predicates.
```haskell
hasParent :: ObjPred -> ObjPred
hasParent p = maybe False p . prevObjPredState

hasAncestor :: ObjPred -> ObjPred
hasAncestor p = maybe False checkParent . prevObjPredState
    where checkParent opSt = p opSt || hasAncestor p opSt

hasChild :: ObjPred -> ObjPred
hasChild p = any p . nextObjPredStates

hasDescendant :: ObjPred -> ObjPred
hasDescendant p opSt = or $ p opSt : map (hasDescendant p) (nextObjPredStates opSt)
```
And make the following convenience infix operators.
```haskell
x ~~> y = x <&&> hasChild y
infixl 7 ~~>

x *~~> y = x <&&> hasDescendant y
infixl 7 *~~>

x +~~> y = x <&&> hasChild (hasDescendant y)
infixl 7 *~~>

x ~/~> y = x <&&> not . hasChild y
infixl 7 ~/~>

x *~/~> y = x <&&> not . hasDescendant y
infixl 7 *~/~> 

x +~/~> y = x <&&> not . hasChild (hasDescendant y)
infixl 7 *~/~>
```
And of course the corresponding ones for ancestors can be defined.

Now I can write predicates such as
```haskell
kindIs "a" +~~> kindIs "b" +~/~> (== "name") ! ('&' `elem`)
```
to specify an `a` which has a proper descendant `b` which doesn't have a proper descendant with a `name` attribute whose value contains an `&`.


