# 11. State Monad

The use of the `Either` monad helped us simplify error processing in the [last tutorial](https://www.fpcomplete.com/user/bartosz/basics-of-haskell/10_Error_Handling). I promised to show you how another monad, the state monad, can eliminate explicit symbol-table threading. But before I do that, let's have a short refresher on currying, since it's relevant to the construction of the state monad (there is actually some beautiful math behind the relationship of currying and the state monad).

There are two ways of encoding a two-argument function and, in Haskell, they are equivalent. One is to implement a function that takes two values:

``` haskell
fPair :: a -> b -> c
fPair x y = ...
```

The other is to implement a function that takes one argument and returns another function of one argument (parentheses added for emphasis):

``` haskell
fCurry :: a -> (b -> c)
fCurry x = \y -> ...
```

This might seem like a trivial transformation, but I'll show you how it can help us in coding the evaluator. 

## Curried Evaluator

Let me remind you what the signature of the function `evaluate` was -- to make things simpler, let's consider the version from before the introduction of the `Either` monad:

``` haskell
evaluate :: Tree -> SymTab -> (Double, SymTab)
```

I'm going to parenthesize it the way that highlights the currying interpretation:

``` haskell
evaluate :: Tree -> (SymTab -> (Double, SymTab))
```
Let's read this signature carefully: `evaluate` is a function that takes a `Tree` and returns a function, which takes a `SymTab` and returns a pair `(Double, SymTab)`. What if we take this reading to heart and rewrite `evaluate` so that it actually returns a function (a lambda). 

Let's start with the `UnaryNode` evaluator, which used to look like this:

``` haskell
evaluate (UnaryNode op tree) symTab =
    let (x, symTab') = evaluate tree symTab
    in case op of
         Plus  -> ( x, symTab')
         Minus -> (-x, symTab')
```
and let's try something like this:

``` haskell
evaluate :: Tree -> (SymTab -> (Double, SymTab))
evaluate (UnaryNode op tree) =
    \symTab -> 
      let (x, symTab') = {-hi-}evaluate{-/hi-} tree symTab --??
      in case op of
         Plus  -> ( x, symTab')
         Minus -> (-x, symTab')
```
You see what the problem is? In the new scheme, the inner call to `evaluate` will no longer return a pair `(x, symTab')` but a function `(SymTab -> (Double, SymTab))`. Let me call this function `act` for *action*. How can we extract `x` and `symTab'` from that action? By running it! We do have an argument `symTab` to pass to it -- it's the argument of the lambda:

``` haskell
evaluate :: Tree -> (SymTab -> (Double, SymTab))
evaluate (UnaryNode op tree) =
    \symTab -> 
        let act = evaluate tree 
            (x, symTab') = {-hi-}act symTab{-/hi-}
        in case op of
            Plus  -> ( x, symTab')
            Minus -> (-x, symTab')
```

What have just happened? We called the new `evaluate` only to immediately execute the resulting action? Then why even bother with the intermediate step? 

First of all, it's a neat idea that evaluation can be separated into two phases: one for creating a network of functions like `evaluate` calling each other but not actually evaluating the result; and another phase for excecuting this network, starting with a particular state -- the symbol table in this case. Obviously, if you provide a different starting symbol table, you will obtain a different final result. But the network of functions depends only on the original parse tree. 

The second reason is that this form brings us closer to our goal of abstracting away the tedium of symbol-table passing. Symbol table passing is what "actions" are supposed to do; `evaluate` should only construct the tracks for the symbol-table train.

Interestingly, this separation between creating an action and running it turned out to be quite useful in C++, as I showed in my old post [Monads in C++](http://bartoszmilewski.com/2011/07/11/monads-in-c/). There, the actions were constructed at compile time using an EDSL, and then executed at runtime. 

Going back to our program, we'll try follow the same procedure we used to derive the `Either` monad. The most important part of a monad is the bind function. Remember, bind is the glue that binds the output of one function to the input of another function -- the one we call a continuation. The signature of bind is determined by the definition of the `Monad` class. It has the form:
``` haskell
bind :: Blob a
     -> (a -> Blob b)
     -> Blob b
```
where `Blob` stands for the type constructor we are trying to monadize. In our case, this type constructor is of the form `(SymTab -> (a, SymTab))`, with the type parameter `a` nested inside the return type of an action. I'll call this function type the new `Evaluator`:

``` haskell
type Evaluator a = SymTab -> (a, SymTab)
```

We'll standardize it later using a `newtype` definition, which is required by `instantiate`, but for now let's just work with a type synonym. 

So here's what monadic bind should look like for our type (yes, it's exactly the same as for our `Either` monad, except that `Evaluator` now hides a function):

``` haskell
bindS :: Evaluator a
      -> (a -> Evaluator b)
      -> Evaluator b
```

The client of bind is supposed to pass an evaluator as the first argument and a continuation as the second. The continuation is a function that returns an evaluator. Let's look for this pattern in our implementation of `evaluate` of the `UnaryNode`:

``` haskell
evaluate :: Tree -> (SymTab -> (a, SymTab))
evaluate (UnaryNode op tree) =
    (\symTab -> 
          let act = evaluate tree 
              (x, symTab') = act symTab
          in case op of
              Plus  -> ( x, symTab')
              Minus -> (-x, symTab'))
```

We are looking for a piece of code that can be interpreted as "the rest of code." On first attempt we might think of the following lambda as our continuation:

``` haskell
\x' -> case op of
         Plus  -> ( x', symTab')
         Minus -> (-x', symTab'))
```
but it's the wrong type. Our continuation is supposed to be returning an `Evaluator`, not a pair `(Double, SymTab)`. How can we turn this value into an evaluator? That's what monadic `return` is supposed to do. Its signature, again, is determined by the `Monad` class (I'm calling it `returnS` for now to avoid name conflicts):

``` haskell
returnS :: a -> Evaluator a
```

The implementation is a no-brainer, really. We turn `x` into a function that returns this `x` with a side of `symTab`:

``` haskell
returnS x = \symTab -> (x, symTab)
```

So here's the candidate that fulfills all our requirements for a continuation:

``` haskell
\x' -> case op of
         Plus  -> return x'
         Minus -> return (-x')
```

This is indeed a fine monadic function (returning a value of the soon to be monadic `Evaluator`), and it fits the type signature of the continuation required by bind; except that we don't see it in the original code. We can't carve it out of the current implementation of `evaluate`. If we could only find a way to insert this `returnS` and then immediately cancel it. But how can one undo `returnS`? Well, how about exectuting its result? Check this out:

``` haskell
(returnsS x) symTab' = (\symTab -> (x, symTab)) symTab'
                     = (x, symTab')
```
When you execute a lambda, you simply replace it with its body and replace the formal parameter with the actual argument. Here, I replaced `symTab` (formal parameter, or *bound* variable) with `symTab'` (the argument). In general, the argument may be a whole expression. You just stick at every place the formal parameter appears in the body. (You have to be careful though not to introduce name conflicts.)

So here's the final rewrite:

``` active haskell
data Operator = Plus | Minus
data Tree = UnaryNode Operator Tree
type SymTab = ()
-- show
type Evaluator a = SymTab -> (a, SymTab)

returnS :: a -> Evaluator a
returnS x = \symTab -> (x, symTab)

evaluate :: Tree -> (SymTab -> (Double, SymTab))
evaluate (UnaryNode op tree) =
    \symTab -> 
        let act = evaluate tree 
            (x, symTab') = act symTab
            k = \x' -> case op of
                        Plus  -> returnS x'
                        Minus -> returnS (-x')
            act' = k x
        in
            act' symTab'

main = putStrLn "It type checks!"
```

If it type checks, it must be correct, right? To convince yourself that this indeed works, first apply `k` to `x` -- this will just replace `x'` with `x`. Then apply the resulting action to `symTab'` to cancel out the `returnS`s.

Let's continue with our program to define a new monad. To this end, we need to identify the pattern we've been looking for. We want to pick the implementation of `bindS` from `evaluate`. 

![](http://www.bartosz.com/images/Basics/BindState.png)


We can clearly see the two arguments to bind: one is `act`, the result of `evaluate tree`, and the other is the continuation `k`. The rest must be bind. Here it is, together with `returnS` and the new version of `evaluate`:


``` active haskell
data Operator = Plus | Minus
data Tree = UnaryNode Operator Tree
type SymTab = ()
-- show
type Evaluator a = SymTab -> (a, SymTab)

returnS :: a -> Evaluator a
returnS x = \symTab -> (x, symTab)

bindS :: Evaluator a
      -> (a -> Evaluator b)
      -> Evaluator b
bindS act k =
    \symTab ->
        let (x, symTab') = act symTab
            act' = k x
        in 
            act' symTab'

evaluate :: Tree -> (SymTab -> (Double, SymTab))
evaluate (UnaryNode op tree) =
    bindS (evaluate tree)
          (\x -> case op of
                   Plus  -> returnS x
                   Minus -> returnS (-x))

main = putStrLn "It type checks!"
```

## Symbol Table Monad

Let's formalize what we've done so far using an actual instance of the `Monad` typeclass. First, we need to encapsulate our evaluator type in a `newtype` declaration. This muddles things a little, but is necessary if we want to use it in an `instance` declaration. Here's a type that contains nothing but a function:

``` haskell
newtype Evaluator a = Ev (SymTab -> (a, SymTab))
```

And here are our return and bind functions in their cleaned up form:

``` haskell
instance Monad Evaluator where
    return x = Ev (\symTab -> (x, symTab))
    (Ev act) >>= k = Ev $ \symTab ->
        let (x, symTab') = act symTab
            (Ev act')    = k x
        in 
            act' symTab'
```

Now that the paperwork is done, we can start using the `do` notation. Here's our monadic `UnaryNode` evaluator:

``` haskell
evaluate (UnaryNode op tree) = do
    x <- evaluate tree 
    case op of
       Plus  -> return x
       Minus -> return (-x)
```

`SumNode` is even more spectacular:

``` haskell
evaluate (SumNode op left right) = do
    lft <- evaluate left
    rgt <- evaluate right
    case op of
       Plus  -> return (lft + rgt)
       Minus -> return (lft - rgt)
```

Compare it with the original:

``` haskell
evaluate (SumNode op left right) symTab = 
    let (lft, symTab')  = evaluate left symTab
        (rgt, symTab'') = evaluate right symTab'
    in
        case op of
          Plus  -> (lft + rgt, symTab'')
          Minus -> (lft - rgt, symTab'')
```

All references to the symbol table are magically gone. The code is not only cleaner, but also less error prone. In the original code there were way too many opportunities to use the wrong symbol table for the wrong call. That's all taken care of now.

There are only three places where you'll see explicit use of the symbol table: `lookUp`, `addSymbol`, and the main loop -- as it should be! I recommend studying the complete code for the calculculator listed at the end of this tutorial, with special attention to those functions.

Now you have seen with your own eyes that all this can be done with pure functions. We managed to manipulate state -- the symbol table -- in a purely functional way.

There is a popular misconception that you *must* use impure code to deal with mutable state, and that Haskell monads are impure. There are ways to introduce impurities in Haskell -- there's a bunch of functions whose names start with *unsafe* and there is `trace` for debugging, the `ST` monad (not to be confused with the `State` monad), all of which (carefully) let you inject impurity into your code. Sometimes it's done for debugging, sometimes for performance. In general, though, you can and should stick to the purely functional style.


## State Monad

What we have just done is to create our own version of a generic state monad. It was, hopefully, a good learning experience, but one that shouldn't be repeated when writing production code. So let's familiarize ourselves with the `Control.Monad.State` version of the state monad (strictly speaking the state monad is defined using a *monad transformer*, so the actual code in the library may look a bit different from what I present). State monad is defined by a new type `State`, which is parameterized by two type variables. The first one is used to represent the state (in our case that would be `SymTab`), and the second is the generic type parameter of every monad type constructor.

``` haskell
newtype State s a = State s -> (a, s)
```
`State` has one data constructor also called `State`. It takes a function as an argument. The interesting thing is that this constructor is not exported from the library so you can't pattern match on it. If you want to create a new monadic `State`, use the function `state`:


``` haskell
state :: (s -> (a, s)) -> State s a
```

Instead of extracting an action from `State`, which you can't do, and acting with it on some state, you call the function `runState` which does it for you:

``` haskell
runState :: State s a -> s -> (a, s)
```

The `Monad` instance declaration for `State` looks something like this:

``` haskell
instance Monad (State s) where
    return x = state (\st -> (x, st))
    act >>= k = state $ \st -> 
                          let (x, st') = runState act st
                          in runState (k x) st'
```

Notice that `State s` is not a type but a type constructor: it needs one more type variable to become a type. As I mentioned before, `Monad` class can only be instantiated with type constructors.

I've shown you how to extract the bind operator from state-threading code, but there is a more general derivation that's based on types. In Haskell you often see functions whose implementation is determined by their signatures. Sometimes it's determined uniquely, more often we pick the simplest non-trivial implementation that type checks. Here's the signature of `>>=` that is required by the `Monad` class as applied to `State s`:

``` haskell
(>>=) :: State s a -> (a -> State s b) -> State s b
act >>= k = ...
```

The first observation is that, in order to run the continuation `k`, we need a value of type `a`. The only source of such value could be the first argument, `act`, and the only way to retrieve it is to call `act` with some state. But we don't have any state yet. 

But notice that bind itself doesn't produce a value -- it produces a `State` object. How do you construct a `State`? By calling `state` with a function. Bind must therefore define a lambda of the signature `s -> (b, s)` and pass it to `state`. The outer shell of `>>=` must therefore have the form:

``` haskell
act >> k = state $ \st -> ...
```

Now, inside the lambda, we do have access to a state variable `st` and we can use it to run `act`.

``` haskell
act >> k = state $ \st ->
                       let (x, st') = runState act st
                       ...

```
Now we have `x` of type `a` so we can call the continuation `k`:
``` haskell
act >> k = state $ \st ->
                       let (x, st') = runState act st
                           act' = k x
                           ...

```
The continuation returns an action `act'` of the type `State s b`. Our lambda, though, must return a pair of the type `(b, s)`. The only way to generate a value of the type `b` is to run `act'` with some state. Here we have a choice: we can run it with the original `st` or with the new `st'`. The first choice would mean that the state never changes and, in fact, doesn't even have to be returned by the action. There is a perfectly good monad built on this assumption: it's called the reader monad (see the exercise at the end of this tutorial). But since here we are modeling mutable state, we choose to use `st'` to run `act'`:

``` haskell
act >> k = state $ \st ->
                       let (x, st') = runState act st
                           act' = k x
                       in runState act' st'

```

There is one more ingredient necessary to make the state monad usable: the ability to access and modify the state. There are two generic functions `get` and `put` that provide this functionality:

``` haskell
get :: State s s
get = state $ \st -> (st, st)

put :: s -> State s ()
put newState = state $ \_ -> ((), newState)
```

`get` returns the value of the state. `put` returns unit, but has a "side effect" of injecting new state into subsequent computations.


## What Is a Monad?


We've seen two seemingly disparate examples of a monad and I will show you some more in the next tutorial. What do they have in common, other than implementing the functions `return` and `>>=`? Why are these two functions so important? It's time for some deeper insights.

The basic premise of all programming is that you can decompose a complex computation into a set of simpler ones. 

The difference between various programming paradigms is in the mechanics of composing smaller computations into larger ones. For instance, in C you use a combination of functions and side effects. You call a function (procedure) whose effects can be:

1. Returning a value
2. Modifying an argument (when it's a reference)
3. Modifying global variables
4. Interacting with the external world

Some of the effects are visible in the signature of the function (types of input and output parameters), others are implicit. The compiler may help with flagging explicit mismatches, but it can't check the implicit ones. So when you're composing functions in C, you have to keep in mind all the hidden interactions between them. 

In OO programming, side effects are somewhat tamed with data hiding. Although arguments are mostly passed by reference, including the implicit `this` pointer, the things you can do to them are restricted by their interfaces. Still, hidden dependencies make composition fragile. This is especially painful when dealing with concurrency. 

The starting point of functional programming is that functions have no side effects whatsoever, so function composition is a straightforward matter of passing the results of one function as the input to the next. This is a great starting point from the point of composability. However, many of the traditional notions of computation don't have straightforward translations into pure functions. This has been a huge problem in the adoption of functional languages. 

Two things happened (not necessarily in that order) to change this situation: 

1. We learned how to translate most computations into functions.
2. We use monads to abstract the tedium of this translation.

I tried to emphasize the same two steps when introducing monads. 

First, I showed you how to translate *partial computations* into total functions. These functions encapsulate their results into `Maybe` or `Either` types. I also showed you how to deal with mutable state by passing it as an additional parameter into and out of a function. 

This is a very general pattern: Take a computation that takes input and produces output but does it in a non-functional way, and modify input and output types in such a way that the computation becomes functional. 

Next, I showed you a way to do the same thing by modifying only the return types of the computation. If the translation of a computation required adding *input parameters* to the original signature (passing the symbol table in, for instance), I used currying and turned the output type into a function type. (In Exercise 1 you'll use the same trick used to implement the reader monad.)

So this is lesson one: A computation can be turned into a function that encapsulates the originally non-functional bits into its modified (decorated, fancified, or whatever you call it) output data type.

The great thing about it is that now all this additional information is visible to the compiler and the type checker. There is even a name for this system in type theory: the *effect system*. A function signature may expose the *effects* of a function in addition to just turning input into output types. These effects are propagated when composing functions (as was the effect of modifying the symbol table, or being undefined for some values of arguments) and can be checked at compile time.

A potential shortcoming of this approach is that the composition of such fancified functions requires writing some boilerplated code. In the case of `Maybe`- or `Either`-returning functions, we have to pattern match the results and fork the execution. In case of action-returning functions, we need to run these actions, provide the additional parameters they need, and pass results to the next action. 

To our great relief, this highly repetitive (and error-prone) glue code can be abstracted into just two functions: `>>=` and `return` (optionally `>>` and `fail`). Now we can test our implementation of the glue code in one place, or still better, use the library code. And to make our lives even better, we have this wonderful syntactic sugar in the shape of the `do` notation.

But now, when you look at a do block, it looks very much like imperative code with hidden side effects. The `Either` monadic code looks like using functions that can throw exceptions. `State` monad code looks as if the state were a global mutable variable. You access it using `get` with no arguments, and you modify it by calling `put` that returns no value. So what have we gained in comparison to C?

*We* might not see the hidden effects, but the compiler does. It desugars every do block and type-checks it. The state might look like a global variable but it's not. Monadic bind makes sure that the state is threaded from function to function. It's never shared. If you make your Haskell code concurrent, there will be no data races. 


## Exercises

** Ex 1.** Define the reader monad. It's supposed to model computations that have access to some read-only environment. In imperative code such environment is often implemented as a global object. In functional languages we need to pass it as an argument to every function that might potentially need access to it. The reader monad hides this process.

``` active haskell
newtype Reader e a = Reader (e -> a)

reader :: (e -> a) -> Reader e a
reader f = undefined

runReader :: Reader e a -> e -> a
runReader = undefined

ask :: Reader e e
ask = reader (\e -> e)

instance Monad (Reader e) where
    ...

type Env = Reader String
-- curried version of
-- type Env a = Reader String a

test :: Env Int
test = do
    s <- ask
    return $ read s + 1

main = print $ runReader test "13"
```

@@@ Show solution
``` active haskell
newtype Reader e a = Reader (e -> a)

reader :: (e -> a) -> Reader e a
reader f = Reader f

runReader :: Reader e a -> e -> a
runReader (Reader act) env = act env

ask :: Reader e e
ask = reader (\e -> e)

instance Monad (Reader e) where
    return x = reader (\_ -> x)
    rd >>= k = reader $ \env ->
                           let x = runReader rd env
                               act' = k x
                           in runReader act' env

type Env = Reader String
-- curried version of
-- type Env a = Reader String a

test :: Env Int
test = do
    s <- ask
    return $ read s + 1

main = print $ runReader test "13"
```
@@@

**Ex 2.** Use the `State` monad from <hoogle search="Control.Monad.State">Control.Monad.State</hoogle> to re-implement the evaluator.

``` active haskell
import Data.Char
import qualified Data.Map as M
import Control.Monad.State

data Operator = Plus | Minus | Times | Div
    deriving (Show, Eq)

data Tree = SumNode Operator Tree Tree
          | ProdNode Operator Tree Tree
          | AssignNode String Tree
          | UnaryNode Operator Tree
          | NumNode Double
          | VarNode String
    deriving Show

type SymTab = M.Map String Double

type Evaluator a = State SymTab a

lookUp :: String -> Evaluator Double
lookUp str = do ...

addSymbol :: String -> Double -> Evaluator ()
addSymbol str val = do ...

evaluate :: Tree -> Evaluator Double

evaluate (SumNode op left right) = ...

evaluate (ProdNode op left right) = ...

evaluate (UnaryNode op tree) = ...

evaluate (NumNode x) = ...

evaluate (VarNode str) = ...

evaluate (AssignNode str tree) = ...

expr = AssignNode "x" (ProdNode Times (VarNode "pi") 
                                (ProdNode Times (NumNode 4) (NumNode 6)))

main = print $ runState (evaluate expr) (M.fromList [("pi", pi)])
```

@@@ Show solution

``` active haskell
import Data.Char
import qualified Data.Map as M
import Control.Monad.State

data Operator = Plus | Minus | Times | Div
    deriving (Show, Eq)

data Tree = SumNode Operator Tree Tree
          | ProdNode Operator Tree Tree
          | AssignNode String Tree
          | UnaryNode Operator Tree
          | NumNode Double
          | VarNode String
    deriving Show

type SymTab = M.Map String Double

type Evaluator a = State SymTab a

lookUp :: String -> Evaluator Double
lookUp str = do
    symTab <- get
    case M.lookup str symTab of
      Just v  -> return v
      Nothing -> error $ "Undefined variable " ++ str

addSymbol :: String -> Double -> Evaluator ()
addSymbol str val = do 
    symTab <- get
    put $ M.insert str val symTab
    return ()

evaluate :: Tree -> Evaluator Double

evaluate (SumNode op left right) = do
    lft <- evaluate left
    rgt <- evaluate right
    case op of
       Plus  -> return $ lft + rgt
       Minus -> return $ lft - rgt

evaluate (ProdNode op left right) = do
    lft <- evaluate left
    rgt <- evaluate right
    case op of
       Times -> return $ lft * rgt
       Div   -> return $ lft / rgt

evaluate (UnaryNode op tree) = do
    x <- evaluate tree 
    case op of
       Plus  -> return x
       Minus -> return (-x)

evaluate (NumNode x) = return x

evaluate (VarNode str) = lookUp str

evaluate (AssignNode str tree) = do
    v <- evaluate tree
    addSymbol str v
    return v

expr = AssignNode "x" (ProdNode Times (VarNode "pi") 
                                (ProdNode Times (NumNode 4) (NumNode 6)))

main = print $ runState (evaluate expr) (M.fromList [("pi", pi)])
```
@@@

## Calculator with the Symbol Table Monad

Here's the complete runnable version of the calculator that uses our Symbol Table Monad.

@@@ Show code
``` active haskell
import Data.Char
import qualified Data.Map as M

data Operator = Plus | Minus | Times | Div
    deriving (Show, Eq)

data Token = TokOp Operator
           | TokAssign
           | TokLParen
           | TokRParen
           | TokIdent String
           | TokNum Double
           | TokEnd
    deriving (Show, Eq)

operator :: Char -> Operator
operator c | c == '+' = Plus
           | c == '-' = Minus
           | c == '*' = Times
           | c == '/' = Div
 
tokenize :: String -> [Token]
tokenize [] = []
tokenize (c : cs) 
    | elem c "+-*/" = TokOp (operator c) : tokenize cs
    | c == '='  = TokAssign : tokenize cs
    | c == '('  = TokLParen : tokenize cs
    | c == ')'  = TokRParen : tokenize cs
    | isDigit c = number c cs
    | isAlpha c = identifier c cs
    | isSpace c = tokenize cs
    | otherwise = error $ "Cannot tokenize " ++ [c]

identifier :: Char -> String -> [Token]
identifier c cs = let (name, cs') = span isAlphaNum cs in
                  TokIdent (c:name) : tokenize cs'

number :: Char -> String -> [Token]
number c cs = 
   let (digs, cs') = span isDigit cs in
   TokNum (read (c : digs)) : tokenize cs'

---- parser ----

data Tree = SumNode Operator Tree Tree
          | ProdNode Operator Tree Tree
          | AssignNode String Tree
          | UnaryNode Operator Tree
          | NumNode Double
          | VarNode String
    deriving Show

lookAhead :: [Token] -> Token
lookAhead [] = TokEnd
lookAhead (t:ts) = t

accept :: [Token] -> [Token]
accept [] = error "Nothing to accept"
accept (t:ts) = ts

expression :: [Token] -> (Tree, [Token])
expression toks = 
   let (termTree, toks') = term toks
   in
      case lookAhead toks' of
         (TokOp op) | elem op [Plus, Minus] -> 
            let (exTree, toks'') = expression (accept toks') 
            in (SumNode op termTree exTree, toks'')
         TokAssign ->
            case termTree of
               VarNode str -> 
                  let (exTree, toks'') = expression (accept toks') 
                  in (AssignNode str exTree, toks'')
               _ -> error "Only variables can be assigned to"
         _ -> (termTree, toks')

term :: [Token] -> (Tree, [Token])
term toks = 
   let (facTree, toks') = factor toks
   in
      case lookAhead toks' of
         (TokOp op) | elem op [Times, Div] ->
            let (termTree, toks'') = term (accept toks') 
            in (ProdNode op facTree termTree, toks'')
         _ -> (facTree, toks')

factor :: [Token] -> (Tree, [Token])
factor toks = 
   case lookAhead toks of
      (TokNum x)     -> (NumNode x, accept toks)
      (TokIdent str) -> (VarNode str, accept toks)
      (TokOp op) | elem op [Plus, Minus] -> 
            let (facTree, toks') = factor (accept toks) 
            in (UnaryNode op facTree, toks')
      TokLParen      -> 
         let (expTree, toks') = expression (accept toks)
         in
            if lookAhead toks' /= TokRParen 
            then error "Missing right parenthesis"
            else (expTree, accept toks')
      _ -> error $ "Parse error on token: " ++ show toks

parse :: [Token] -> Tree
parse toks = let (tree, toks') = expression toks
             in
               if null toks' 
               then tree
               else error $ "Leftover tokens: " ++ show toks'

---- evaluator ----
-- show

type SymTab = M.Map String Double

newtype Evaluator a = Ev (SymTab -> (a, SymTab))

instance Monad Evaluator where
    (Ev act) >>= k = Ev $
        \symTab -> 
            let (x, symTab') = act symTab
                (Ev act') = k x
            in act' symTab'
    return x = Ev (\symTab -> (x, symTab))

lookUp :: String -> Evaluator Double
lookUp str = Ev $ \symTab ->
    case M.lookup str symTab of
      Just v  -> (v, symTab)
      Nothing -> error $ "Undefined variable " ++ str

addSymbol :: String -> Double -> Evaluator Double
addSymbol str val = Ev $ \symTab ->
    let symTab' = M.insert str val symTab
    in  (val, symTab')

evaluate :: Tree -> Evaluator Double

evaluate (SumNode op left right) = do
    lft <- evaluate left
    rgt <- evaluate right
    case op of
       Plus  -> return $ lft + rgt
       Minus -> return $ lft - rgt

evaluate (ProdNode op left right) = do
    lft <- evaluate left
    rgt <- evaluate right
    case op of
       Times -> return $ lft * rgt
       Div   -> return $ lft / rgt

evaluate (UnaryNode op tree) = do
    x <- evaluate tree 
    case op of
       Plus  -> return x
       Minus -> return (-x)

evaluate (NumNode x) = return x

evaluate (VarNode str) = lookUp str

evaluate (AssignNode str tree) = do
    v <- evaluate tree
    addSymbol str v

main = do
   loop (M.fromList [("pi", pi)])

loop symTab = do
   str <- getLine
   if null str
   then
      return ()
   else
      let toks = tokenize str
          tree = parse toks
          Ev act = evaluate tree
          (val, symTab') = act symTab
      in do
          print val
          loop symTab'
```
@@@

