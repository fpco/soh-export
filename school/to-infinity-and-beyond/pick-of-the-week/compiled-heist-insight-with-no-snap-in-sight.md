# Compiled Heist insight, with no Snap in sight

#Introduction

**Heist** is a **XML/HTML** templating library which enforces a watertight separation between models and views. This makes very easy to create and manipulate the templates using standard XML/HTML tools.

While Heist is commonly used within the **Snap** web framework, it doesn't depend on it, and can be used in other contexts, for example for generating XML reports within a batch application. It this tutorial we will use Heist just by itself.

Heist comes in two flavors: **interpreted** and **compiled**. The tutorial with focus on compiled Heist.

Compiled Heist is efficient because it tries to perform the most amount of work possible at template load time. For example, imagine a very big template consisting of tens of thousands of HTML elements, into which a single small string value, possibly fetched from a database, is interpolated. 

With interpreted Heist, all the nodes in the template would have to be traversed each time the template is rendered. This is wasteful. Compiled Heist is smarter: most of the processing will be performed at load time. At runtime, what will be done is something like "fetch the value from the db, prepend this big precalculated bytesting to the value, append this other big precalculated bytesting to the value, ready".

The official introduction to compiled Heist can be found [here](http://snapframework.com/docs/tutorials/compiled-splices).

#Compiled splices

Top-level compiled splices are like global immutable symbols, and are declared and bound to specific XML tags when initializating the Heist engine.

For complex views, compiled splices can have sub-splices, defined with functions like `withLocalSplices`, `withSplices` and `manyWithSplices`. These sub-splices are not global but local to the enclosing splice.

#The runtime monad

The runtime monad is the monad in which "deferred" actions take place. Deferred actions are used to fetch those pieces of data that can only be known at template render time (as opposed to template load time). We can't just precompile everything!

Throughout this tutorial, the runtime monad for the splices will be plain `IO`. Hence the `Splice IO` and `RuntimeSplice IO` signatures. 

In Snap applications, the runtime monad is `Handler`, and this means that compiled splices will have all the `Snaplet` machinery at their disposal (remember that `Handler` is itself a monad transformer over the `Snap` monad.)

#Some key types

A `RuntimeSplice` is just a monad transformer over the runtime monad, that augments it with the ability to "write back" to the compiled parts of the splice. This is done with mutable references under the hood (the `Promise` datatype) but for most use cases the user doesn't need to be aware of it.

Runtime actions that return `Builder` values can be transformed into `DList (Chunk n)` values using the `yieldRuntime` function. A `DList (Chunk n)` value is an effectful list of sorts, that emits the contents of the rendered page while performing effects in the runtime monad.

And, since `Splice n` is just a type synonym for `HeistT n IO (DList (Chunk n))`, this means we can use `return` to build a simple compiled splice out of the effectful list. That splice will do all of its work at runtime.

#The simplest splice

This snippet shows how to initialize Heist using `heistInit`. We pass the function a value of type `HeistConfig`. `HeistConfig` is a monoid, so you can merge two configurations with ease. Here we specify two basic fields: the current directory as the source for templates, and a single top-level splice bound to the `foo` tag.

`(##)` is defined in module `Heist.SpliceAPI`. If offers a convenient syntax to define lists of tag-splice pairs using do-notation.

The snippet uses the very simple predefined splice called `C.runChildren`. What it does is to render the contents of the tag to which is bound, the `foo` tag in this case. So, it will elide the `foo` tag and nothing more. Try it!

``` active haskell
{-# START_FILE Main.hs #-}
{-# LANGUAGE OverloadedStrings  #-}
import Data.Monoid
import qualified Data.ByteString as B
import Blaze.ByteString.Builder
import Control.Monad
import Control.Applicative
import Control.Monad.Trans.Either
import Heist
import Heist.Compiled as C

-- show
splice :: Splice IO
splice = C.runChildren

main = do
    let heistConfig = mempty
            { 
              hcCompiledSplices = 
                    "foo" ## splice
            , hcTemplateLocations =
                    [loadTemplates "."]            
            }            
    heistState <- either (error "oops") id <$> 
         (runEitherT $ initHeist heistConfig)
    builder <- maybe (error "oops") fst $
         renderTemplate heistState "simple"
    toByteStringIO B.putStr builder
-- /show    
{-# START_FILE simple.tpl #-}
<body>
    <foo>
        <p>foo should disappear!</p>
    </foo>
</body>  
```

`C.runChildren` is frequently used in combination with more complex splice functions, because Heist often stores the "views" for a data structure in the contents of the tag to be spliced. Which is a rather good idea, where else to put them?

#Basic substitution

This is just like the previous example, but now we are substituting a value read at runtime instead of just rendering the contents of the tag. The runtime action reads from console, and it is promoted to a splice using `return $ C.yieldRuntimeText`.

``` active haskell
{-# START_FILE Main.hs #-}
{-# LANGUAGE OverloadedStrings  #-}
import Data.Monoid
import qualified Data.Text as T
import qualified Data.ByteString as B
import Blaze.ByteString.Builder
import Control.Monad
import Control.Monad.IO.Class
import Control.Applicative
import Control.Monad.Trans.Either
import Heist
import Heist.Compiled as C

-- show
runtime :: RuntimeSplice IO T.Text
runtime = liftIO $ do
    putStrLn "Write something:"
    T.pack <$> getLine

splice :: Splice IO
splice = return $ C.yieldRuntimeText $ runtime
-- /show

main = do
    let heistConfig = mempty
            { 
              hcCompiledSplices = 
                    "foo" ## splice
            , hcTemplateLocations =
                    [loadTemplates "."]            
            }            
    heistState <- either (error "oops") id <$> 
         (runEitherT $ initHeist heistConfig)
    builder <- maybe (error "oops") fst $
         renderTemplate heistState "simple"
    toByteStringIO B.putStr builder
    
{-# START_FILE simple.tpl #-}
<body>
    <foo>
        Your text here.
    </foo>
</body>  
```

#Splicing a list of values

What if the runtime action returns a list of values and we want to render then contiguously? We can use `deferMany` for that.

Notice that `deferMany` requires a function `RuntimeSplice n a -> Splice n` as the first argument, that tells it how to render each particular element of the list. But its type is too general for our purposes. We just want to render each Text as it comes, without any modification. So we turn to the `textSplice` and `pureSplice` functions to do the necessary lifting. These functions often go together.

``` active haskell
{-# START_FILE Main.hs #-}
{-# LANGUAGE OverloadedStrings  #-}
import Data.Monoid
import qualified Data.Text as T
import qualified Data.ByteString as B
import Blaze.ByteString.Builder
import Blaze.ByteString.Builder.Char.Utf8
import Control.Monad
import Control.Monad.IO.Class
import Control.Applicative
import Control.Monad.Trans.Either
import Heist
import Heist.Compiled as C

-- show
runtime :: RuntimeSplice IO [T.Text]
runtime = liftIO $ replicateM 3 $ do
    putStrLn "Write something:"
    T.pack <$> getLine

splice :: Splice IO
splice = C.deferMany 
    (C.pureSplice . C.textSplice $ id) 
    runtime
-- /show

main = do
    let heistConfig = mempty
            { 
              hcCompiledSplices = 
                    "foo" ## splice
            , hcTemplateLocations =
                    [loadTemplates "."]            
            }            
    heistState <- either (error "oops") id <$> 
         (runEitherT $ initHeist heistConfig)
    builder <- maybe (error "oops") fst $
         renderTemplate heistState "simple"
    toByteStringIO B.putStr builder
    
{-# START_FILE simple.tpl #-}
<body>
    <foo>
        Your texts here.
    </foo>
</body>  
```

#Splicing composite values

Now suppose we want to render a data structure that has fields, like a record. And we want it to be rendered using the contents of the enclosing tag (`person`) as the view. Each field will have its own sub-tag (`name` and `age`).

Function `C.withSplices` lets us build a composite splice out of a list of functions that know how to render each field. Notice the use of `C.runChildren` since we want to use the contents of `person` as the view.

He we again use `(##)` as a convenient notation to build the list of tag-function pairs. Notice how we transform the field accessors using `C.pureSplice` and `C.textSplice`. 

``` active haskell
{-# START_FILE Main.hs #-}
{-# LANGUAGE OverloadedStrings  #-}
import Data.Monoid
import qualified Data.Text as T
import qualified Data.ByteString as B
import Blaze.ByteString.Builder
import Blaze.ByteString.Builder.Char.Utf8
import Control.Monad
import Control.Monad.IO.Class
import Control.Applicative
import Control.Monad.Trans.Either
import Heist
import Heist.Compiled as C

-- show
data Person = Person {
            name :: T.Text
        ,   age :: Int
        }
    
runtime :: RuntimeSplice IO Person
runtime = return $ Person "Splicy Splicer" 33

splice :: Splice IO
splice = C.withSplices C.runChildren
                       splicefuncs 
                       runtime
    where              
    splicefuncs = do
        "name" ## (C.pureSplice . C.textSplice $ 
                        name)
        "age" ## (C.pureSplice . C.textSplice $ 
                        T.pack . show . age)        
-- /show

main = do
    let heistConfig = mempty
            { 
              hcCompiledSplices = 
                    "person" ## splice
            , hcTemplateLocations =
                    [loadTemplates "."]            
            }            
    heistState <- either (error "oops") id <$> 
         (runEitherT $ initHeist heistConfig)
    builder <- maybe (error "oops") fst $
         renderTemplate heistState "simple"
    toByteStringIO B.putStr builder
    
{-# START_FILE simple.tpl #-}
<body>
    <person>
        <div>
        <p><name>John Smith</name></p>
        <p><age>77</age></p>
        </div>
    </person>
</body>  
```

# Splicing a list of composite values

But what if we want to splice a list of records? Easy! Just use `C.manyWithSplices` instead of  `C.withSplices`.

``` active haskell
{-# START_FILE Main.hs #-}
{-# LANGUAGE OverloadedStrings  #-}
import Data.Monoid
import qualified Data.Text as T
import qualified Data.ByteString as B
import Blaze.ByteString.Builder
import Blaze.ByteString.Builder.Char.Utf8
import Control.Monad
import Control.Monad.IO.Class
import Control.Applicative
import Control.Monad.Trans.Either
import Heist
import Heist.Compiled as C

-- show
data Person = Person {
            name :: T.Text
        ,   age :: Int
        }
    
runtime :: RuntimeSplice IO [Person]
runtime = return [
      Person "Splicy Splicer" 33
    , Person "Hasty Hornet" 27       
    ]

splice :: Splice IO
splice = C.manyWithSplices C.runChildren
                           splicefuncs 
                           runtime
    where              
    splicefuncs = do
        "name" ## (C.pureSplice . C.textSplice $ 
                        name)
        "age" ## (C.pureSplice . C.textSplice $ 
                        T.pack . show . age)        
-- /show

main = do
    let heistConfig = mempty
            { 
              hcCompiledSplices = 
                    "person" ## splice
            , hcTemplateLocations =
                    [loadTemplates "."]            
            }            
    heistState <- either (error "oops") id <$> 
         (runEitherT $ initHeist heistConfig)
    builder <- maybe (error "oops") fst $
         renderTemplate heistState "simple"
    toByteStringIO B.putStr builder
    
{-# START_FILE simple.tpl #-}
<body>
    <person>
        <div>
        <p><name>John Smith</name></p>
        <p><age>77</age></p>
        </div>
    </person>
</body>  
```

#Splicing nested datatypes

Another common use case is having to render a record with complex subfields that have their own views. Here for example we are using a list of `Locations` for each person.

What we do is to define two composite splices and nest one into the other. The only tricky bit is that we can't use `C.pureSplice . C.textSplice` like we did with "atomic" fields, because `locSplice` already takes a `RuntimeSplice IO [Location]` and returns a `C.Splice IO`. But we can use `liftM locations` to extract the list of locations from the `RuntimeSplice IO Person` value, and now the types fit.

``` active haskell
{-# START_FILE Main.hs #-}
{-# LANGUAGE OverloadedStrings  #-}
import Data.Monoid
import qualified Data.Text as T
import qualified Data.ByteString as B
import Blaze.ByteString.Builder
import Blaze.ByteString.Builder.Char.Utf8
import Control.Monad
import Control.Monad.IO.Class
import Control.Applicative
import Control.Monad.Trans.Either
import Heist
import Heist.Compiled as C

-- show
data Person = Person {
            name :: T.Text
        ,   age :: Int
        ,   locations :: [Location]
        }
        
data Location = Location {
          street :: T.Text
       ,  number :: Int
       }
    
runtime :: RuntimeSplice IO Person
runtime = return $ Person "Splicy Splicer" 33 $
            [ Location "Barnaby Street" 34,
              Location "Juana de Vega" 89 ]  

splice :: Splice IO
splice = C.withSplices C.runChildren
                       splicefuncs 
                       runtime
    where              
    splicefuncs = do  
        "name" ## (C.pureSplice . C.textSplice $ 
                        name)
        "age" ## (C.pureSplice . C.textSplice $ 
                        T.pack . show . age)
        "location" ## (locSplice . 
                         liftM locations )                     
        
locSplice :: RuntimeSplice IO [Location] -> 
             C.Splice IO
locSplice runtime' = C.manyWithSplices C.runChildren 
                                       splicefuncs 
                                       runtime'
    where
    splicefuncs = do
        "street" ## (C.pureSplice . C.textSplice $ 
                            street)
        "number" ## (C.pureSplice . C.textSplice $ 
                            T.pack . show . number)                                  
-- /show

main = do
    let heistConfig = mempty
            { 
              hcCompiledSplices = 
                    "person" ## splice
            , hcTemplateLocations =
                    [loadTemplates "."]            
            }            
    heistState <- either (error "oops") id <$> 
         (runEitherT $ initHeist heistConfig)
    builder <- maybe (error "oops") fst $
         renderTemplate heistState "simple"
    toByteStringIO B.putStr builder
    
{-# START_FILE simple.tpl #-}
<body>
    <person>
    <div>
        <p><name>John Smith</name></p>
        <p><age>77</age></p>
        
        <location>
        <div class="loc">
            <p><street/></p>
            <p><number/></p>
        </div>
        </location>
    </div>
    </person>
</body>  
```

#Closing remarks

These examples cover some common Heist use cases. More complex use cases may require using the lower-level functions in module `Heist.Compiled.LowLevel`, but I bet those are infrequent.

Happy splicing!