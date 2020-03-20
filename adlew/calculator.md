# Calculator

I have been thinking for a while about how to introduce friends of mine to haskell, people are often intrigued by all the impressive one liners out there such as the fibbonaci sequence etc. But I believe there is a gap between the one-liners and small practical applications.

This guide focuses on showing the simplicity and modularity of Haskell in a problem that very well suits functional programming.

So, let's get started. Below is a simple "Hello world" Haskell program.

``` active haskell
main = putStrLn "Hello world"
```

Simple and clean, let's move on to some serious number crunching!
``` active haskell
main = print (1 + 1)
```

Okay so we can do math, now how would we tell Haskell to evaluate a 
string such as "1 + 5 * 2"?
Naturally the first step is to parse the content of the string into a more readable form for the computer. 

Our first step is to separate all tokens so we know what we are working with. Luckily for us there allready is such a function
``` active haskell
main = print (words "1 + 5 * 2")
```
So, now that we have a way to structure the information we need a way to evaluate all the numbers and apply the correct mathematical function on them. Take for example 2 - 2 * 2, would we simply evaluate from left to right we would get (2 - 2) * 2 which is equal to 0 and incorrect. Instead we need to start with multiplication 2 - (2 * 2) = 2 - 4 = -2.

To achieve this we could create a register that stores how to prioritize and evaluate functions.

When we design the register we could really use a type system to verify that it contains what we just said it should. And luckily for us there is one, everything in Haskell is statically typed, in other words something that was a Int one moment ago can't suddenly be a String. By default Haskell infer the types for us but we can choose to do it explicitly.

here are some examples of types.
```
a list of strings = [String]

a integer = Int

a function that takes a integer and returns a string = Int -> String

a tuple of one integer and one string = (Int, String)
```

Our register need to have a String that contains the name of the operator and a function that takes two numbers and return a new one. The priority can be deduced by the order of the list.

In code this would look like this
``` haskell
type Operator = Double -> Double -> Double
type Entry = (String, Operator)
type Register = [Entry]
```
where "type" creates a type alias such that Operator = Double -> Double -> Double.
Now we can create a register like this.
``` haskell
operatorRegister :: Register -- The :: operator means 'is of type'
operatorRegister = [
                ("+", (+)),
                ("*", (*))
            ]
```
We are able to directly use the + function because in haskell all operators are normal functions.

Now that we have a way to gather the information and a neat operator register we can move on and start writing the calculator.

In Haskell when developing a function it can be a great ide to start by defining in the type system what we want the function to be able to do. Let's try this.
``` haskell
calculate :: String -> Double
calculate = undefined
```
In english this means that calculate is a function that takes a string and returns a decimal number and said function does not yet have any defined behavior. undefined can be very usefull while we figure out the structure of the program as the program still compiles and all types are checked. So we can be notified of any type errors before writing a single line of real code.

As there is no serious parsing going on we do not need a dedicated parse function but can simply use the words function mentioned before and therefore only need a function to evaluate the parsed data.

``` haskell
calculate :: String -> Double
-- operatorRegister is the register we defined before.
calculate = eval operatorRegister . words

eval :: Register -> [String] -> Double
eval = undefined
```
Here we introduce a new function, the dot. The dot in haskell means that the output of the right-hand function is sent to the left-hand function.

I'm now gona show you the code for the eval function but don't freak out we are gona go through it step by step.

``` haskell
eval :: Register -> [String] -> Double
eval _ [number] = read number
eval ((operator, function):rest) unparsed =
    case span (/=operator) unparsed of
        (_, []) -> eval rest unparsed
        (beforeOperator, afterOperator) -> 
            function
                (eval operatorRegister beforeOperator)
                (eval operatorRegister $ drop 1 afterOperator)
```
In Haskell you can pattern match function parameters which means the function can act differently depending on the input.
``` haskell
-- Called when only 1 element is unparsed
eval _ [number] = ... 
-- _ means we discard the value so it matches anything but we ignore the value.
-- [number] is matched if the parameter is a list with a single element, in which
-- case the number = the element.

-- Called otherwise.
eval ((operator, function):rest) unparsed = ...
-- ((operator, function):rest) means take the first element of the list and bind
-- the elements in the tuple operator and function then bind the rest of the list to rest.
```
The first eval entry is quite straightforward, it converts a String to a Double.
The second eval entry is a bit more tricky let's first look at the case .. of ... operator
``` active haskell
main = putStrLn $ boolToString False
-- the $ operator works the same way as parentheses
-- and main could also be written:
-- putStrLn (boolToString False)

boolToString :: Bool -> String
boolToString variable = case variable of
    True -> "It's true"
    False -> "It's a lie!"
```
The case .. of ... is simply a way to pattern match a variable.
Now let's get to the variable we pattern match: span (/=operator) unparsed.
``` active haskell
-- in this example span will keep on taking elements until the current element is 5 then
-- then it stops and returns a tuple with the content before the element on the left side
-- and the rest on the right side.
main = print $ span (/=5) [1..10]
```

This means that if the right side is empty the element was not found. Now we can better understand the code:
``` haskell
case span (/=operator) unparsed of
    -- If the element is not found we call the function recursivly searching for the next
    -- operator in the register.
    (_, []) -> eval rest unparsed
    
    -- This is called if we do find a operator, we call the function from the register
    -- and recursivly calculate the rest of the unparsed content before we send it to
    -- the function.
    (beforeOperator, AfterOperator) ->
        function 
            (eval operatorRegister beforeOperator)
            (eval operatorRegister $ drop 1 afterOperator)
```

Now you should have an ide of how the program works and here is what we got so far.
(This program only supports + and * but feel free to add new operators.
``` active haskell
type Operator = Double -> Double -> Double
type Entry = (String, Operator)
type Register = [Entry]

operatorRegister :: Register
operatorRegister = [
                ("+", (+)),
                ("*", (*))
            ]
            
main = print $ calculate "2 * 5 + 5"
            
calculate :: String -> Double
calculate = eval operatorRegister . words
            
eval :: Register -> [String] -> Double
eval _ [number] = read number
eval ((operator, function):rest) unparsed =
    case span (/=operator) unparsed of
        (_, []) -> eval rest unparsed
        (beforeOperator, afterOperator) -> 
            function
                (eval operatorRegister beforeOperator)
                (eval operatorRegister $ drop 1 afterOperator)
```

The reason why this program works is because it keeps on desecting the unparsed content until there are only numbers left and then they parse the numbers and send them back to the operator function which in turn operate on them and send the content up until we are back at the top and the sum is finally returned.

A simple parse tree could look something like this:
Parsing begin...
2 * 5 + 5
(2 * 5) + (5)
( (2) * (5) ) + (5)
Parsing done, only numbers left.
(10) + (5)
15
Evaluation complete.

Now as you may or may not have realised this is a working program except for one thing, there is no error checking. If we would for example write "1 + ." or "1 +" there will be a mean error waiting for us.

In a programming language such as C you might fix this by adding some ifs or elses to make the program fail gracefully. In haskell though, we don't need any of that. Meet Maybe
``` active haskell
main = print $ evenSteven 4

evenSteven :: Int -> Maybe Int
evenSteven number = if even number then Just number else Nothing
```

Maybe, is a elegant way to create conditional returns in a way not easily replicated in a non-functional language. One advantage with Maybe is that it can also be used recursivly, making functions depend on each other so that if a subexpression return Nothing the current expression also returns Nothing.
``` active haskell
-- Import the <$> and <*> functions from the
-- Control.Applicatve librarie.
import Control.Applicative ((<$>), (<*>))

main = print $ maybeAdd 6 5

add :: Int -> Int -> Int
add a b = a + b

maybeAdd :: Int -> Int -> Maybe Int
-- The <$> and <*> provdies a way to "lift" a function
-- so it understands maybe.
maybeAdd a b = add <$> (evenSteven a) <*> (evenSteven b)

evenSteven :: Int -> Maybe Int
evenSteven number = if even number then Just number else Nothing
```

This is one of the fundamental strengths of Haskell, with only two operators we can make the code foolproof.

Let's update our calculator to use maybe to return a value.
``` active haskell
import Control.Applicative ((<$>), (<*>))

type Operator = Double -> Double -> Double
type Entry = (String, Operator)
type Register = [Entry]

modulu :: Double -> Double -> Double
modulu a b = fromIntegral $ mod (round a) (round b)

operatorRegister :: Register
operatorRegister = [
                ("-", (-)),
                ("+", (+)),
                ("/", (/)),
                ("*", (*)),
                ("%", modulu)
            ]
            
main = print $ calculate "3 * 2 + 5 % 2"
            
calculate :: String -> Maybe Double
calculate = eval operatorRegister . words
            
eval :: Register -> [String] -> Maybe Double
eval [] _ = Nothing -- No operator found.
eval _ [] = Nothing -- If a operator don't have anything to operate on.
eval _ [number] = Just $ read number
eval ((operator, function):rest) unparsed =
    case span (/=operator) unparsed of
        (_, []) -> eval rest unparsed
        (beforeOperator, afterOperator) -> 
            function
                <$> (eval operatorRegister beforeOperator)
                <*> (eval operatorRegister $ drop 1 afterOperator)
```

And that's it, we now have a simple working calculator, thanks for you time and good luck with Haskell!