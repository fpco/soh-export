# xml-conduit + lens

```active haskell
{-# START_FILE test.xml #-}
-- show
<foo>
    <bar>
        <baz>
            <bin/>
        </baz>
    </bar>
    <bar/>
</foo>
{-# START_FILE main.hs #-}
-- show
{-# LANGUAGE OverloadedStrings #-}
import Prelude hiding (readFile, writeFile)
import Text.XML
import Control.Lens.Zipper
import Control.Lens hiding (element)
import qualified Data.Map as Map
import Data.Text (Text)
import qualified Data.ByteString.Lazy.Char8 as L

main = do
    Document pro root' epi <- readFile def "test.xml"
    let root = NodeElement root'
    print $ root' ^. named "foo" . child -- . named "foo" -- . child -- . traverse . element "bar"
    print $ root' & named "foo" . child .~ []
    L.putStrLn $ renderLBS def $ (\e -> Document pro e epi)
        $ rezip
        $ zipper root'
        &/ "bar"
        &/ "baz"
        &.. focus . name .~ "RENAMED"
    --print $ downward (element "foo") $ zipper $ NodeElement root

infixl 1 &/
infixl 1 &..

x &/ name = x & fromWithin (child . each . element . named name)
x &.. f = x & upward & f

element :: Prism' Node Element
element =
    prism' NodeElement get
  where
    get (NodeElement e) = Just e
    get _ = Nothing

named :: Name -> Prism' Element Element
named name =
    prism' id get
  where
    get e@(Element name' _ _)
        | name == name' = Just e
        | otherwise = Nothing

child :: Lens' Element [Node]
child =
    lens get put
  where
    get (Element _ _ x) = x
    put :: Element -> [Node] -> Element
    put (Element a b _) c = Element a b c

name :: Lens' Element Name
name = lens (\(Element x _ _) -> x) (\(Element _ y z) x -> Element x y z)
```