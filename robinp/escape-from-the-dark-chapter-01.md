# Escape from the dark - Chapter 01

## Signs of alert

> _You open your eyes to see the tunnel slowly being filled with smoke and dust.
> The flashing red lights make you remember your emergency
> training.
> You crawl through the debris and dust, reaching for the special kit.
> Opening it you find a pack of assorted cabels, a worn manual, and.. lemons?_

Welcome, I'm QECU, the Quantum Elevator Control Unit.

Congratulations! While drilling at level -523 of the Central Mine,
you managed to blow up the drilling heads just one day before the
quota completion deadline.

Thanks for your precise actions, there is a major power outage in the whole facility,
all the mineral stock is stuck between levels -420 to -523, the emergency tool storage
is flooded with water, and I'm forced to run off a [Lemon Battery](http://en.wikipedia.org/wiki/Lemon_battery).

The bad news is, that your only chance for survival is reprogramming the
Secondary Elevator Circuit and emerging to the exit at Level 2.
Of course the circuit is programmed using Haskell, a programming language you are unfamiliar with.

The good new is, that your getting out of the facility has a non-zero expected probability.
0.0000178%, to be precise. Also, my Lemon Batteries let me run
my digital clap synthesizer, the irony processor, and a
Haskell compiler, with which I might be able to introduce you to
programming. Fat chance.

## Your first lines

A crash course on computers: Computers... compute. Pretty surprising, isn't it?
This concludes the crash course on computers.

To compute, you need two things:

1. Values to act upon, such as the weight of a mineral container, 
   or the levels per second nominal travelling speed of a Quantum Elevator.
   Quantum Elevators are beautiful by the way. Much more reasonable than humans.
    
2. Functions to apply on values. For example, given as input the number
    of levels you have to crawl all the way up to the surface, and the nominal QE speed,
    using a function you can calculate the seconds it takes for you to reach the surface.
    Well, if the elevator would not be powered off, of course.

Here's your chance to declare a value yourself!
Type the following code on my touchpad, and send it for evaluation by pressing the Play button.

``` active haskell
-- show
lemons = 5
-- /show

main = print $ "There are " ++ show lemons ++ " lemons remaining."
```
The response acknowledges that you have just declared a value named `lemons`. By the way, it has a type of `Int` representing a signed integer number, and `5` as the actual value. If you want, you can put a type annotation on the value:


``` active haskell
-- show
lemons :: String
lemons = 5
-- /show

main = print $ "There are " ++ show lemons ++ " lemons remaining."
```

Ouch, something went wrong. Can you help me fix that type annotation above? I _accidentally_ entered `String` instead of `Int`. Thank you. Ok, in fact it was deliberate. I want to see if you can type. You can never know with humans.

### Some more about values

#### Names
                
When declaring a value, you bind a value to a name.
You can reference that value in your code, using the name you gave it.

#### Types

All values have a type - some sample values and type pairs:

``` haskell
meaning :: Int
meaning = 42

yourChance :: Float
yourChance = 0.000000178

energySource :: String
energySource = "lemons"

warningSign :: Char
warningSign = '!'
```

#### Type inference

Usually you don't have to specify the type of a value, since the compiler will automatically infer it for you. But if you wanted, you could do it - _We_ computers don't need explicit types, since we can infer them _ourselves_. Without help.

#### Immutability

A binding declared is final. That is, you cannot assign a
new value to the declared name. Try the following reassignment, and observe the results.

``` active haskell
-- show Won't compile. Think twice before you act.
message = "All systems normal"

message = "No wait, I mean all systems abnormal!"
-- /show
main = print message
```

Great! Now that you can declare values, you can get out to the surface finally! Well, no, that was a lie.
But cheer up, I estimate that you have completed objective 1 out of 15449, only 15448 to go!
