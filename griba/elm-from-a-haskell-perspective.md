# Elm from a Haskell perspective

[Elm](http://elm-lang.org/) has no genericity mechanisms for structures by now, like Haskell typeclasses, nor instance arguments.

Elm gives a parser error if you try to use type variables for structures like *(m a)* in a function.

Instead you have a few predefined, compiler bound, type variable categories.

(Examples tested with Elm version 0.18)

### Syntax

Check it [here](http://elm-lang.org/docs/syntax).

### Predefined stuff

Elm's predefined functions are at [Basics](http://package.elm-lang.org/packages/elm-lang/core/latest/Basics), but there are additional [default imports](https://github.com/elm/core#default-imports).

### TypeVarCategories

Elm has predefined type variable categories that have implicit functionality. You specify a type var. category by means of a prefix. Check the compiler function [*categorizeVar*](https://github.com/elm/compiler/blob/master/builder/src/Deps/Diff.hs#LC288)

+ type variables with [*"number"*](http://package.elm-lang.org/packages/elm-lang/core/latest/Basics#+) as prefix form a category that corresponds to Haskell's *Num* typeclass (a [ring](https://en.wikipedia.org/wiki/Ring_(mathematics)) structure), defines (+), (-), (\*), *abs*, *negate*, adding (^) and *clamp* to the pack. Check the link documentation for available instances.
+ [*"comparable"*](http://package.elm-lang.org/packages/elm-lang/core/latest/Basics#compare) type var category corresponds to Haskell's *Ord* functionality
+ [*"appendable"*](http://package.elm-lang.org/packages/elm-lang/core/latest/Basics#++) type var category corresponds to Haskell's *Semigroup* with (++) as right associative infix operator for *append*

To extend the functionality with new functions based on the predefined ones for the category, you only have to categorize the type variable of the parameter using the category prefix.

### Type differences

+ Elm's *Int* has the integer division as double slash [(//)](http://package.elm-lang.org/packages/elm-lang/core/latest/Basics#//) and *mod* as (%), and *rem* as *rem*. (/) is [only for Floats](http://package.elm-lang.org/packages/elm-lang/core/latest/Basics#/)

But there is a catch with (//). It does not return the integer part when dividing negatives. It behaves like haskell's *quot* truncating towards zero !! So it does not correspond to the [euclidean division](https://en.wikipedia.org/wiki/Euclidean_division) definition because it doesn't comply the rule stating that the remainder should be non&#x2011;negative.

```haskell
import Html as H
import List as L

-- (//) behaves like `quot`, not like `div`
intDblSlashIsQuot : Int -> Int -> Bool
intDblSlashIsQuot denom num = abs (num // denom) == abs (-num // denom)

checkProperty = intDblSlashIsQuot 3

main = H.text <| toString <| L.map checkProperty [7, -2, 4, -8]
```

+ Elm's *Float* (JavaScript's IEEE754 Number) corresponds to Haskell's *Double*
+ Parameters with the type category *number* allow Int and Float as actual parameter types.
+ Elm's data types use the clause keyword *type*. Type aliases are defined with "type alias".

+ Elm has a special uninhabited type [*Never*](http://package.elm-lang.org/packages/elm-lang/core/latest/Basics#Never) to be used as a Task error type parameter for tasks that cannot fail.

### Some structure equivalences

+ Elm's [*Maybe*](http://package.elm-lang.org/packages/elm-lang/core/latest/Maybe#Maybe) corresponds to Haskell's *Maybe*
+ Elm's [*Result*](http://package.elm-lang.org/packages/elm-lang/core/latest/Result) has an homomorphic corresponding in Haskell's *Either*

+ Squared bracket literals \[1,3,5\] denote Elm's [Lists](http://package.elm-lang.org/packages/elm-lang/core/latest/List), not Arrays.

```haskell
-- zips are missing but easy done
zip : List a -> List b -> List (a,b)
zip = List.map2 (,)

zip3 = List.map3 (,,)
zip4 = List.map4 (,,,)

```

+ Elm's random access sequence [Array](http://package.elm-lang.org/packages/elm-lang/core/latest/Array) is **immutable**, not like JS ones. A Haskell corresponding is Vector from [Data.Vector.Unboxed](https://hackage.haskell.org/package/vector/docs/Data-Vector-Unboxed.html)

+ Elm's linear access sequence [List](http://package.elm-lang.org/packages/elm-lang/core/latest/List)

+ Elm's mapping [*Dict*](http://package.elm-lang.org/packages/elm-lang/core/latest/Dict) structure (dictionary) requires *comparable* key types (ordered domains). This includes Int, Float, Time, Char, String, and tuples or lists of comparable types. Its corresponding in Haskell is [Data.Map.Strict](http://hackage.haskell.org/package/containers/docs/Data-Map-Strict.html) where stored values are forcedly evaluated.

+ Elm's [*Set*](http://package.elm-lang.org/packages/elm-lang/core/latest/Set) structure as a special case of Dict where the elements are the keys, has the same restriction.

+ Elm's [*Task*](http://package.elm-lang.org/packages/elm-lang/core/latest/Task) structure is for effect actions that may fail. It implements Haskell's [MonadError](http://hackage.haskell.org/package/mtl/docs/Control-Monad-Except.html#t:MonadError) throwing errors with *fail* that skips the subsequent actions until the *onError* monadic catcher is evaluated, and its type *(Task err a)* would correspond to *IO* lifted to a *MonadError* compliant transformer, like *([ExceptT](http://hackage.haskell.org/package/mtl/docs/Control-Monad-Except.html#t:ExceptT) err IO a)*.

+ Haskell range syntax is not supported in Elm. Specific functions are used: [List.range](http://package.elm-lang.org/packages/elm-lang/core/latest/List#range). But you can define a *dot-dot* operator (..)

```haskell
import Html as H
import List

(..) = List.range
infix 9 ..

main = H.text <| toString (1..7)
```

A Haskell style stepped range for Ints defining [enumFromThenTo](http://hackage.haskell.org/package/base/docs/Prelude.html#v:enumFromThenTo).

```haskell
import Html as H
import List as L
import Debug as D
import Char as Ch

-- tailrec
intEnumFromThenToTR : List Int -> Int -> Int -> Int -> List Int
intEnumFromThenToTR acc ini nxt top = 
   if ini == nxt 
   then D.log "next must be different than initial" []
   else if {- ini beyond top -} ini /= top && 
           compare ini top /= compare ini nxt 
        then L.reverse acc
        else intEnumFromThenToTR (ini :: acc) nxt (nxt + nxt - ini) top

intEnumFromThenTo = intEnumFromThenToTR []  -- using this seems to slow result yield

v0 = intEnumFromThenTo 9 7 1     -- [9,7,5,3,1]

v1 = intEnumFromThenToTR [] 9 7 1  -- not as elegant but maybe faster

-- refactored
intEnumFromThenTo2 : Int -> Int -> Int -> List Int
intEnumFromThenTo2 ini0 nxt0 top =
    if ini0 == nxt0 
    then D.log "next must be different than initial" []
    else let 
             step = nxt0 - ini0
             go acc ini nxt = 
               if {- ini beyond top -} ini /= top && 
                  compare ini top /= compare ini nxt 
               then L.reverse acc
               else go (ini :: acc) nxt (nxt + step)
          in go [] ini0 nxt0      

v2 = intEnumFromThenTo2 9 7 1

charEnumFromThenTo: Char -> Char -> Char -> List Char
charEnumFromThenTo ini0 nxt0 top = 
    if ini0 == nxt0 
    then D.log "next must be different than initial" []
    else L.map Ch.fromCode <| 
               intEnumFromThenTo2 (Ch.toCode ini0) (Ch.toCode nxt0) (Ch.toCode top)

v3 = charEnumFromThenTo 'Y' 'W' 'A'
main = H.text <| toString (v2, v3)
```

Using partial application seems to yield result later in Elm's *try-online* vs. specifying all function arguments at once. Check it also with the program at the page bottom.

### Functions

[Strict evaluation](https://groups.google.com/d/msg/elm-discuss/9XxV9L0zoA0/ZUU9RGKAthoJ).

Single pattern definitions.

No *where* clauses.

Elm's Prelude is described [here](https://github.com/elm/core#default-imports).

Elm's API search is [here](http://package.elm-lang.org/).

+ Haskell's *id* is called [*identity*](http://package.elm-lang.org/packages/elm-lang/core/latest/Basics#identity) in Elm.

+ Haskell's *const* is called [*always*](http://package.elm-lang.org/packages/elm-lang/core/latest/Basics#always) in Elm.

+ Haskell's [*flip*](http://package.elm-lang.org/packages/elm-lang/core/latest/Basics#flip), *curry* and *uncurry* are named equally in Elm.

+ Haskell's function application Data.Function.($) corresponds to Elm's [(<|)](http://package.elm-lang.org/packages/elm-lang/core/latest/Basics#%3C|)
+ Haskell's reverse application Data.Function.(&) corresponds to Elm's [(|>)](http://package.elm-lang.org/packages/elm-lang/core/latest/Basics#|%3C)

+ Haskell's right to left composition Control.Category.(<<<) corresponds to Elm's [(<<)](http://package.elm-lang.org/packages/elm-lang/core/latest/Basics#%3C%3C)
+ Haskell's left to right composition Control.Category.(>>>) corresponds to Elm's [(>>)](http://package.elm-lang.org/packages/elm-lang/core/5.1.1/Basics#%3E%3E)

+ Haskell's *show* corresponds to Elm's [*toString*](http://package.elm-lang.org/packages/elm-lang/core/latest/Basics#toString)

+ Haskell's Data.Tuple *fst*, *snd* are defined in Elm's Basics, but they are named *first*, *second* in Elm's [Tuple](http://package.elm-lang.org/packages/elm-lang/core/latest/Tuple)

+ Tuple constructors are defined for arity (<= 9), but the [Tuple](http://package.elm-lang.org/packages/elm-lang/core/latest/Tuple) module advises against it use, proposing records instead.

```haskell
import Html as H

v = (,,,,,,,,) 1 2 3 4 5 6 7 8 9 -- `_Tuple9` Ok, `_Tuple10` undefined

main = H.text (toString v)
```
+ [Functions arity](https://github.com/elm/compiler/blob/master/builder/src/Generate/Functions.hs) has a top of 9 arguments

### usual ADT functions


+ Functor: Haskell Data.Functor.fmap corresponds to Elm structures' [map](http://package.elm-lang.org/packages/elm-lang/core/latest/Maybe#map) function

+ Monad: Haskell's bind (>>=) corresponds to Elm structures' [*andThen*](http://package.elm-lang.org/packages/elm-lang/core/latest/Maybe#andThen) flipped.

+ Applicative: With Elm structures' ([map2](http://package.elm-lang.org/packages/elm-lang/core/latest/Maybe#map2), .., map5) you have the functionality that corresponds to Haskell's Control.Applicative (liftA2, ...) to combine the results of a string of actions as subsequent parameters.

For Task's *map{N}*, action side effects are serialized with *andThen* (Haskell's bind) so they rather correspond to Control.Monad.liftM{N} instead of Control.Applicative.liftA{N}.

#### Reductions, Traversals and Tail call optimization

>> The Elm compiler is able to do tail-call elimination on a function when any of the branches are a direct self-recursive call. See [Tail-call elimination](https://github.com/evancz/functional-programming-in-elm/blob/master/recursion/tail-call-elimination.md).

+ Lists
```haskell
-- from Elm's core library List
foldl : (a -> b -> b) -> b -> List a -> b
foldl func acc list =
  case list of
    []      -> acc
    x :: xs -> foldl func (func x acc) xs   -- tail recursive
    
foldr : (a -> b -> b) -> b -> List a -> b
foldr = Native.List.foldr     -- via JS list conversion toArray(xs)

-- map is based on foldr, so elements are traversed right to left
map : (a -> b) -> List a -> List b
map f xs = foldr (\x acc -> f x :: acc) [] xs

-- Task.sequence traverses left-to-right a list of Tasks giving a Task that returns the List of results. It uses map2 (like Applicative.liftA2 but serializing the side effects: Control.Monad.liftM2) to concatenate its results.
```

### Desugaring Haskell Do blocks

Simulation of a Haskell *do* block, nesting subsequent monadic functions to have all lambda argument variables in scope. Try it in Elm's [try online](http://elm-lang.org/try):

```
import Html as H exposing (Html)
import Maybe as M

-- Maybe.andThen : (a -> Maybe b) -> Maybe a -> Maybe b

-- simulation of a Haskell "do" block
monadic : Maybe Int
monadic = Just 1 
              |> M.andThen (\x -> Just 2
              |> M.andThen (\y -> let y2 = y * y     -- a `do block` let
                                  in Just (x + y2)
              |> M.andThen (\z -> Just (x + y2 + z)  -- y2 still in scope
                            )))
-- rewriting it with (>>=)
(>>=) m f = M.andThen f m
infixl 1 >>=

monadic2 : Maybe Int
monadic2 = Just 1 
              >>= (\x -> Just 2
              >>= (\y -> let y2 = y * y     -- a `do block` let
                                  in Just (x + y2)
              >>= (\z -> Just (x + y2 + z)  -- y2 still in scope
                   )))

main : Html String
main = H.text <| toString (monadic, monadic2)
```

### Deferring computations - Lazy parameters

See also [elm-lang/lazy](https://github.com/elm-lang/lazy). Lazyness memoisation is advised against. Have a look at the link's *"Pitfalls"* section regarding memory usage for memoization of lazy expressions and parameters.


```haskell
import Html as H

-- lazyness without memoization
type Lazy a = Lazy (() -> a)

force : Lazy a -> a
force (Lazy f) = f ()
--------------
-- use:
maybeOrElse : Maybe a -> Lazy a -> a
maybeOrElse mbX lzY = case mbX of
      Just x -> x
      Nothing -> force lzY

v = maybeOrElse Nothing <| Lazy (\_ -> sqrt 2)

main = H.text <| toString v
```

* [Lazyness with memoization](http://package.elm-lang.org/packages/elm-lang/lazy/latest)
* A [Stream library](http://package.elm-lang.org/packages/naddeoa/stream/latest)

### A static "Hello world" page

```haskell
import Html as H exposing (Html)

-- appendables (Semigroup) have (++) as infixr for append (See Basics)

main : Html msg
main = H.text <| "Hello " ++ "cruel world!"
```

### Pages without effects

Web programs without effects follow the so called *beginnerProgram* structure as [explained here](https://guide.elm-lang.org/architecture/user_input/buttons.html).

```haskell
Html.beginnerProgram : { model : model, 
                    view : model -> Html msg, 
                    update : msg -> model -> model } 
                    -> Program Never model msg
```

```haskell
main = Html.beginnerProgram { model = initialModel, view = view, update = update }
```

### Pages with effects

```haskell
Html.program : { init : (model, Cmd msg), 
            update : msg -> model -> (model, Cmd msg), 
            subscriptions : model -> Sub msg, 
            view : model -> Html msg } 
            -> Program Never model msg
```

Effect actions (tasks) are not evaluated directly but by sending a command to an *Effect manager* that will be handled differently based on their protocol processing state. 

### Effects, Effects managers and Message routing

Effect managers support protocol processing by handling its own message queue (*selfMsg* queue), and may send messages to the app main loop through the *appMsg* queue parameter of a *Router* object assigned to them.

You can interact with them sending *commands* and *subscriptions*  specific to the *effect manager* which finally will send messages routed to your app main loop, and handled by the *update* function.

See the guide on [Effects](https://guide.elm-lang.org/architecture/effects/) and the [Platform Router](http://package.elm-lang.org/packages/elm-lang/core/latest/Platform#ProcessId)

```haskell
-- from the Platform module:
{-| Task: Represents asynchronous effects that may fail. It is useful for stuff like HTTP.
-}
type Task err ok = Task

{-| Router: An effect manager has access to a “router” that routes messages between the main app and your individual effect manager.
-}
type Router appMsg selfMsg = Router


{-| sendToApp: The effects manager module will be able to send the router a message for the main loop of your app. This message will be handled by the overall `update` function, just like events from `Html`.
-}
sendToApp : Router appMsg a -> appMsg -> Task err ()

{-| sendToSelf: The effects manager module will be able to send the router a message for its own message queue. This message will
be routed to the `onSelfMsg` function, where you can update the state of your effect manager as necessary.
-}
sendToSelf : Router a selfMsg -> selfMsg -> Task err ()

```

The *(Cmd msg)* and *(Sub msg)* types are the types of requests for commands or subscriptions sent to the effect managers (see below) that may send app messages to be processed by the *update* function. 

Each effect manager exposes user entry points to build such requests.

```
-- from Platform.Cmd
{-| Cmd msg: the type of commands to Elm's effects

Every Cmd specifies (1) which effects you need access to and (2) the type of messages that will come back into your application.
-}
type Cmd msg

-- from Platform.Sub

{-| Sub msg: the type of commands to Elm's effects
Every Sub specifies (1) which effects you need access to and (2) the type of messages that will come back into your application.
-}

```

Elm has modules qualified by the *effect* keyword as effect managers. They handle Msgs from/to the run time system.

As example, the [Websocket Effect Manager](https://github.com/elm-lang/websocket/blob/master/src/WebSocket.elm) skeleton, expliciting which queue the message types are related:

```haskell
effect module WebSocket where { command = MyCmd, subscription = MySub } exposing
  ( send
  , listen
  , keepAlive
)
...

-- MyCmd and MySub are parameterized by an appMsg type to encode app. message constructors.

type MyCmd appMsg = Send String String

type MySub appMsg = Listen String (String -> appMsg) 
                 | KeepAlive String

-- | user entry calls to build command or subscription requests

send : String -> String -> Cmd appMsg
send url txt = command (Send url txt)

listen : String -> (String -> appMsg) -> Sub appMsg
listen url tagger = subscription (Listen url tagger)
-- ^ tagger: appMsg constructor to wrap the received String with

keepAlive : String -> Sub appMsg
keepAlive url = subscription (KeepAlive url)

-- | request processing automaton (input -> state -> state) callback

onEffects
  : Platform.Router appMsg SelfMsg
  -> List (MyCmd appMsg)     -- cmd requests
  -> List (MySub appMsg)     -- subscription requests
  -> State appMsg
  -> Task Never (State appMsg)
onEffects router cmds subs state = ...

-- | State: The msg param in the State type is the appMsg of the type of the current subscriptions

type alias State msg =
  { sockets : SocketsDict
  , queues : QueuesDict
  , subs : SubsDict msg
}

-- | selfMsgs (type changed from Msg to SelfMsg to avoid confusion)

type SelfMsg  
  = GoodOpen String WS.WebSocket
  | BadOpen String
  | Receive String String
  | Die String

-- | selfMsgs processing automaton callback

onSelfMsg 
  : Platform.Router appMsg SelfMsg 
  -> SelfMsg 
  -> State appMsg 
  -> Task Never (State appMsg)
onSelfMsg router selfMsg state = ...

-- | cmdMap called by Platform.Cmd.map

cmdMap : (a -> b) -> MyCmd a -> MyCmd b

-- | subMap called by Platform.Sub.map

subMap : (a -> b) -> MySub a -> MySub b
```

See [WebSockets client program](https://guide.elm-lang.org/architecture/effects/web_sockets.html) from the Elm's guide.

Other effect managers: Time (subscription only), Random (command only), other at [core library or Effects pkg list](http://package.elm-lang.org/)

## Running a Task

The type *(Task err a)* represents an action that may fail.

You may run a Task with the commands *perform* or *attempt*, that send a message to the appMsg queue upon finalisation, through the Task effect manager (See the [module](http://package.elm-lang.org/packages/elm-lang/core/latest/Task)'s source).

Within *Effect managers*, automaton callbacks are Tasks. You can also run tasks asynchronously through [Process.spawn](http://package.elm-lang.org/packages/elm-lang/core/latest/Process#spawn) and abort them with *Process.kill*.

The structure Task implements Haskell's Monad and [MonadError](http://hackage.haskell.org/package/mtl/docs/Control-Monad-Except.html#t:MonadError) functionality as well, for computations that can throw errors whose type is determined by the type of the monad, here *(Task err)*, where *throwError* bypasses subsequent actions until *catchError* is found.

+ *succeed* -> Haskell Monad's *return* implementation
+ *andThen* -> Haskell Monad's flipped *bind* impl.
+ *fail* -> Haskell MonadError's *throwError* 
+ *onError* -> Haskell MonadError's *catchError*
+ *map* -> Haskell Functor's *fmap*
+ *map2* -> Haskell Control.Monad's *liftM2*
+ *map3* -> Haskell Control.Monad's *liftM3*
+ *sequence* -> Haskell Control.Monad *sequence* for a List container

```haskell
effect module Task where { command = MyCmd } exposing
  ( Task                      -- the type
  , perform, attempt          -- cmds to run tasks
  , succeed, andThen          -- return, bind
  , map, map2, map3, map4, map5  -- functor, liftM{N}
  , sequence                     -- sequence a list of tasks
  , fail, onError, mapError   -- throwError, catchError
)

{-| Command to perform asynchronously a Task that cannot fail (err ~ Never) and send a msg to the appMsg queue
-}
perform : (a -> msg) -> Task Never a -> Cmd msg
perform toMessage task =
  command (Perform (map toMessage task))


{-| Command to attempt asynchronously a task that might fail and send a msg chosen by the resultToMessage function
-}
attempt : (Result err a -> msg) -> Task err a -> Cmd msg
attempt resultToMessage task =
  command (Perform (
    task
      |> andThen (succeed << resultToMessage << Ok)   -- Ok is a constructor of type Result
      |> onError (succeed << resultToMessage << Err)  -- Err is a constructor of type Result
))


{-| Simple task generator that succeeds immediately when run.
    A Haskell's Monad `return` implementation.
-}
succeed : a -> Task err a

{-| Tasks chaining.
    A flipped version of Haskell's Monad `bind`
-}
andThen : (a -> Task x b) -> Task x a -> Task x b

{-| fail gives a task that fails immediately when run.
    fail "file not found" : Task String a
    
    fail: Monadic exception thrower, MonadError's throwError
-}
fail : err -> Task err a

{- onError: Monadic exceptions catcher, MonadError's catchError
-}
onError : (x -> Task y a) -> Task x a -> Task y a

{- mapError: to wrap the error type into a wider union one with a constructor
-}
mapError : (x -> y) -> Task x a -> Task y a

{- Functor map
-}
map : (a -> b) -> Task x a -> Task x b
map f taskA = 
        taskA |> andThen (\a -> succeed (f a))

{- Results combination of serialized tasks: Haskell's liftM2
-}

map2 : (a -> b -> c) -> Task x a -> Task x b -> Task x c
map2 f2 taskA taskB = 
          taskA |> andThen (\a -> taskB
                |> andThen (\b -> succeed (f2 a b)
                          ))
...

{-| form a Task from a list of tasks returning the list of results.
-}
sequence : List (Task x a) -> Task x (List a)
sequence tasks =
  case tasks of
    [] -> succeed []
    task :: remainingTasks -> 
              map2 (::) task (sequence remainingTasks)
```

### Adding tipical control functions for the type "Task"

```haskell
module Task_Extra exposing (..)

import Task exposing (Task, succeed, andThen, map2)

-- lazyness without memoization
type Lazy a = Lazy (() -> a)

force : Lazy a -> a
force (Lazy f) = f ()

-- let's call traverse what in Haskell would be mapM or traverseM
traverse : (a -> Task err b) -> List a -> Task err (List b)
traverse f li = case li of
              [] -> succeed []
              x :: xs -> map2 (::) (f x) (traverse f xs)

-- side effects only
traverse_ : (a -> Task err ()) -> List a -> Task err ()
traverse_ f li = case li of
               [] -> succeed ()
               x :: xs -> f x |> andThen (\_ -> traverse_ f xs)

-- Haskell's monadic forM
for = flip traverse
for_ = flip traverse_

when : Bool -> Lazy (Task err ()) -> Task err ()
when cond lzTask = 
    if cond then force lzTask 
            else succeed ()

whenM : Lazy (Task err Bool) -> Lazy (Task err ()) -> Task err ()
whenM lzBoolTask lzTask = 
    force lzBoolTask |> andThen (\cond -> when cond lzTask)

-- replicate with Index
replicate : Int -> (Int -> Task err a) -> Task err (List a)
replicate n f = 
   if n <= 0 then succeed []
   else let go i =                 -- i: 0..n
                   if (i < n) then map2 (::) (f i) <| go (i+1)
                   else succeed []    
        in go 0   

-- side effects only
replicate_ : Int -> (Int -> Task err ()) -> Task err ()
replicate_ n f =                 
   if n <= 0 then succeed ()
   else let go i action =          -- i: 0..n
                   if (i < n) then go (i+1) (f i)   -- tail recursive
                   else succeed ()    
        in go 0 (succeed ())  

```

## Elm timing example: Comparing elapsed times of expressions with partial application vs complete number of arguments.

Check the [function application templates](https://github.com/elm/compiler/blob/master/builder/src/Generate/Functions.hs) and run this program in Elm's [try-online](http://elm-lang.org/try).

```haskell
import Html as H exposing (Html)
import Time as TM exposing (Time)
import Platform.Cmd
import Platform.Sub
import Task as TS exposing (Task)
import List as L
import Debug as D

-- (>>=) for Tasks
(>>=) m f = TS.andThen f m
infixl 1 >>=

type Lazy a = Lazy (() -> a)

force : Lazy a -> a
force (Lazy f) = f ()

type MyErr = MyErr String

type Msg = Start | Elapsed (Time, Time) | MsgErr String

type alias Model = {elapsed: Result String (Time, Time)}

intEnumFromThenToTR : List Int -> Int -> Int -> Int -> List Int
intEnumFromThenToTR acc ini nxt top = 
   if ini == nxt then D.log "next must be different than initial" []
   else if {- ini beyond top -} ini /= top && 
           compare ini top /= compare ini nxt 
   then L.reverse acc
   else intEnumFromThenToTR (ini :: acc) nxt (nxt + nxt - ini) top

intEnumFromThenTo = intEnumFromThenToTR []  -- using partial application seems to slow execution

v0 = intEnumFromThenTo 1003 1001 1   

v1 = intEnumFromThenToTR [] 1003 1001 1  -- maybe faster

-- evaluate the deferred lzA parameter N times
iterateM_ : Int -> Lazy a -> Task MyErr ()
iterateM_ n lzA = 
  case compare n 0 of 
    LT -> TS.fail <| MyErr "iterateM_: n must be positive" 
    EQ -> TS.succeed ()
    GT -> let -- tail recursive, evaluating the action as parameter
              go m action = 
                if m == 0 then TS.succeed ()
                else go (m-1) (TS.succeed (force lzA) 
                                   >>= (\_ -> TS.succeed ()))
          in go n (TS.succeed ())

-- time N evaluations of lzA
timeItN : Int -> Lazy a -> Task MyErr Time
timeItN n lzA = TM.now 
      >>= (\ tmIni -> iterateM_ n lzA
      >>= (\ _ -> TM.now
      >>= (\ tmFinal -> TS.succeed (tmFinal - tmIni)
      )))

timeV : List Int -> Task MyErr Time
timeV v = timeItN 1000 <| Lazy (\_ -> L.sum v)

targetsTimes : Task MyErr (Time, Time)
targetsTimes = timeV v0
           >>= (\ t1 -> timeV v1
           >>= (\ t2 -> TS.succeed (t1, t2)
           ))

timesResultToMsg : Result MyErr (Time, Time) -> Msg
timesResultToMsg res = 
     case res of
       Ok v -> Elapsed v
       Err (MyErr str) -> MsgErr str

view : Model -> Html msg
view model =
    case model.elapsed of
      Err str -> H.div [] [H.text <| "Error: " ++ str]
      Ok (t1, t2) -> -- arrange (t1, t2) in a List to map units once
                     let res = L.map TM.inMilliseconds [t1, t2]
                     in H.div [] [H.text <| toString res]

update : Msg -> Model -> (Model, Cmd Msg)
update msg mdl = case msg of
      Start -> (mdl, TS.attempt timesResultToMsg targetsTimes)
      Elapsed tm -> ({ mdl | elapsed = Ok tm}, Cmd.none)
      MsgErr str -> ({ mdl | elapsed = Err str}, Cmd.none)

init : (Model, Cmd Msg)
init = ({elapsed = Ok (0, 0)}, 
        TS.perform (\_ -> Start) (TS.succeed ())) -- send Start msg

main = H.program { init = init, update = update, 
                   view = view, subscriptions = (\_ -> Sub.none)}
```

## More info

+ [Cabal-like project file, JSON styled (elm-package.json)](https://guide.elm-lang.org/reuse/modules.html#building-projects-with-multiple-modules)
+ [A library project file](https://github.com/elm-lang/websocket/blob/master/elm.json)
+ [Installing packages](https://github.com/elm-lang/elm-package/blob/master/README.md)

+ [The Elm Book - An Introduction to Elm](https://guide.elm-lang.org/)
+ [Forms in Elm](https://medium.com/@l.mugnaini/forms-in-elm-validation-tutorial-and-examples-2339830055da)

## Other client side Model-View-Controller frameworks

[PureScript](http://taylor.fausak.me/static/pages/2015-10-22-better-know-a-language-purescript.html#1) offers alternatives for designing Web User Interfaces much nearer to the language power of Haskell and much lighter weight than GHCJS (no GHC RTS emulation).

See [Purescript client-side MVC frameworks](https://www.schoolofhaskell.com/user/griba/purescript-client-side-mvc-frameworks)
