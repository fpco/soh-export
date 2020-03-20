# Representation, Operations, and Semantics

# Mathematical Discussion: Complex Numbers
Complex numbers are one of these concepts which keep popping up whether you are a mathematician, a physicist, an engineer, a computer scientist. 

Now, we are going to implement "the complex numbers" in various different ways. Hopefully, you will be very comfortable with them by the time we are done. Keep in mind that this is an example of a much bigger thing: how to take a mathematical idea, and turn it into a computational idea. 

# Cartesian Complex Numbers

``` active haskell

data CComplex = C {re::Double, im::Double} deriving (Show, Eq)
--this stands for the complex number (a +i b). You would create it like this: z = C a b
a = 1
b = 3
z = C a b

w = C 2 1

--You can add two CCNs
add (C a b) (C c d) = C (a+c) (b+d)
--could have been written like this: z1 `add` z2 = C (re z1 + re z2) (im z1 + im z2)

s1 = z `add` w

--You can multiply two CCNs
mult (C a b) (C c d) =C (a*c - b*d)  (b*c + a*d)

p1 = z `mult` w

--You can do other things as well, but that is enough for now.
newline = putStrLn ""

main = do  print  "a"
       	   print a
       	   newline
       	   print "b"
       	   print b
       	   newline
       	   print "z"
       	   print z
       	   newline
       	   print "w"
       	   print w
       	   newline
       	   print "z `add` w"
       	   print $ z `add` w
       	   newline
       	   print " z `mult` w"
       	   print $  z `mult` w

```
A little vocabulary:  the `data ...` statement is an implementation of the _Representation_. The `add ...` and `mult ...` statements are implementations of the _Operations_.

Here is what we see: the  `add` _operation_ is a _point-wise_ operation, which means that is does something to the real parts, and then does something to the imaginary parts. Notice that the order in which we compute these doesn't change the result.

As a rule of thumb, _point-wise_ operations are simple to understand, implement, and to _optimize_.

On the other hand, the `mult` operation is not point-wise. Granted, the implementation wasn't very hard, and it is clear that it is correct. However, the meaning of it is rather mysterious in this form. What does it do exactly. To answer this question, we turn to our second representation of the day: Polar Complex Numbers

# Polar Complex Numbers


``` active haskell

data PComplex = P {amp::Double, phase::Double} deriving (Show, Eq)
--this stands for the complex number (r * exp(i theta). You would create it like this: z = P r theta
r = 1
theta = 3
z = P r theta

w = P 2 1

--You can add two PCNs, but it is quite nasty. We will come back to it later.
add (P a b) (P c d) = error "Too Nasty For Now!"


--You can multiply two PCNs
mult (P a theta) (P b phi) =P (a*b)  (theta + phi)
--could have been written like this: z1 `mult` z2 = P (amp z1 * amp z2) (phase z1 + phase z2)

p1 = z `mult` w

--You can do other things as well, but that is enough for now.
newline = putStrLn ""

main = do  print  "r"
       	   print r
       	   newline
       	   print "theta"    
       	   print theta
       	   newline
       	   print "z"
       	   print z
       	   newline
       	   print "w"
       	   print w
       	   newline
       	   print " z `mult` w"
       	   print $  z `mult` w

```
We now see what `mult` does; it multiplies the amplitudes, and adds the phases. It is a point-wise operation.

# Cartesian and Polar

Alone, the polar representation is rather weak (addition is an ugly mess), but if you only want to multiply complex numbers, you are better off with Polars than Cartesians.

Ok, so we have these two ways of looking at the same thing. A natural question is "what is the relationship between these two representations?"

``` active haskell
main = do print "(c,toPolar c, toCartesian $ toPolar c) "
          print $ (c,toPolar c, toCartesian $ toPolar c)
          newline
          print "(p,toCartesian p,toPolar $ toCartesian p)"
          print $ (p,toCartesian p,toPolar $ toCartesian p)
          
data CComplex = C {re::Double, im::Double} deriving (Show, Eq)
data PComplex = P {amp::Double, phase::Double} deriving (Show, Eq)

--from geometry, we see:
toPolar (C x y) = P (sqrt $ x**2 + y**2) (atan (y/x))

toCartesian (P a theta) = C (a* cos theta) (a* sin theta)



c = C 1 2.3
p = P 1 (pi/2) --can you see what mathematical object this is?


newline =putStrLn ""


```

Great, so we see we can go back and forth between our two representations. We can even go from polar to Cartesian to polar again and get the same number!

Now, it's time for a mixed representation: we want our polar and Cartesian complex numbers to be to forms of the same thing, like in mathematics.

``` active haskell
data Complex = C {re::Double, im::Double} 
    | P {amp::Double, phase::Double} deriving (Show, Eq)    
--you can read this as: a complex is either a C, in which case it has both a re and an im part, or it is a P, in which case it has both an amp and a phase.

toPolar (C x y) = P (sqrt $ x**2 + y**2) (atan (y/x))
toPolar alreadyPolar = alreadyPolar  -- if the evaluation arives here, it means it is not a cartesian. In our case, it must be a polar then. 

toCartesian (P a theta) = C (a* cos theta) (a* sin theta)
toCartesian alreadyCartesian = alreadyCartesian 

--fundamental definitions
addC (C a b) (C c d) = C (a+c) (b+d)
multP (P a theta) (P b phi) = P (a*b)  (theta + phi)

--full definitions derived from the fundamental ones
z1 `add` z2 = (toCartesian z1) `addC` (toCartesian z2)
z1 `mult` z2 = (toPolar z1) `multP` (toPolar z2)

main = do print $ (C 1 3) `add` (P 1 (pi/2))
          print $ (C 1 3) `mult` (P 1 (pi/2))    

--fundamental definitions (extra)
oppositeC (C a b) = C (-a) (-b)
inverseP (P r theta) = P (1/r) (-theta)

minusC z1 z2 = addC z1 (oppositeC z2) --this is dangerous if z1 or z2 are polars, but because it calls functions that fail on polars, it will fail too if you feed it polars. 

divideP z1 z2 = multP z1 (inverseP z2) 
```
Exercises: 1) implement the full definitions of opposite and inverse; 2) full definition of minus and divide. 3) (bonus) re-implement the fundamental operations in a way that is type-safe (will complain at compile time instead of at execution time). 4) (bonus) make Complex an instance of `Num` (this should be trivial if you know what `Num` is)

# Different Representations?
So far, we have seen three representations: C, P, and C|P. We saw that there were links between them. For the record, C|P is called the _type sum_ of C and P. The sum is associated with the intuitive notion of _either_: C|P is either C or P.

But let us go deeper. When you come right down to it, C is just a pair of `Double`, and so is P. They are both pairs of `Double` for which we have defined _operations_. For instance, to multiply two pairs of `Double` which form a P, we use `multP (P a theta) (P b phi) = P (a*b)  (theta + phi)`, whereas to multiply two pairs of `Double` which form a C, we use `multP (C a b) (C c d) =C (a*c - b*d)  (b*c + a*d)`. 
In fact, Ps and Cs have the same representation, but different operations. We will come back to this later.

Focusing on `C`, we said that it was a pair of `Double`. We can also write it like this: `C = Double & Double` (this is not valid haskell by the way), and we say that C is the _type product_ of Double and Double. In the same way, we write `P = Double & Double`

So, in general, a representation can be a _type sum of type products_, like `C|P =(Double&Double)|(Double&Double)`. The correct haskell for this is:

```haskell

data Type = Term1 Fact1 Fact2 Fact3 ... FactN | Term2 Fact(N+1) Fact(N+2) ...
data CP = C Double Double | P Double Double
```

We saw that the same representation  with different operations will define a different thing. Here is the third concept for today:

Supposing you have a representation r and a list of operations ops, the _semantics_ is a function which associates each element of r to a mathematical object in o and each element of ops to a function in fs.

Here it is for C:
``` haskell
C a b -> a + i b
addC -> +
multC -> *
...
```

where the rhs is in the mathematical domain, and the lhs is in the Computational domain.

For P:
``` haskell
P a b -> a * exp (i b)
addP -> +
multP -> *
...
```

For C|P
``` haskell
C a b -> a + i b
P a b -> a * exp (i b)
toPolar -> identity
toCartesian -> identity
add -> +
mult -> *
...
```

You might be thinking "well that was useless." but it wasn't, and to illustrate this, I will show a completely different representation: the matrix representation.

# Matrix Complex Numbers

``` active haskell
import Data.List
data C = C {re::Double, im::Double} deriving (Show, Eq)    

--normally you would use hmatrix for this, but the package isn't available at this time it seems.
--this is a really ugly hack, but it illustrates the idea.
type Mats a = [[a]]

addM = zipWith (zipWith (+)) --pointwise addition

--only for 2 by 2, but easy to generalise
multM m1 m2 = let m2' = transpose m2
                  mult i j = sum $ zipWith (*) (m1!!i) (m2'!!j)
              in [[mult 0 0, mult 0 1],[mult 1 0, mult 1 1]]

data M = M (Mats Double) deriving (Show,Eq)

add (M z1) (M z2) = M $ addM z1 z2
mult (M z1) (M z2) = M $ multM z1 z2

main = do print $ toCartesian $ (toMatrix (C 0 1)) `mult` (toMatrix (C 0 1))    
          print $ toCartesian $ (toMatrix (C 0 1)) `add` (toMatrix (C 0 1))              


toMatrix (C a b) = (M [[a,-b],[b,a]])
toCartesian (M [[a,b],_]) = C a (-b)

```

Exercises: 1) define the semantics of M. 2) implement the opposite and inverse for M. 3) implement C|P|M in the most economical (no redundancy) manner.


Notice that after we've implemented `toMatrix`, the addition and multiplication of complex numbers reduces to the addition and multiplication of matrices.

Furthermore, given `toMatrix`, we have `toMatrix (-z1) = - (toMatrix z1)` which means that viewing a matrix as a complex number, the opposite of that matrix corresponds to the opposite of the complex number. We also have `toMatrix (1/z1) = (toMatrix z1)^(-1)` which states that the matrix inverse corresponds to the complex number inverse.

Exercise: Go on Wikipedia, and see how to compute the inverse of a two by two matrix. For the opposite, you just have to multiply each component of the matrix by (-1). 1) implement the matrix inverse and opposite. 2) use this code in C|P|M to test the above statement that `toMatrix (-z1) = - (toMatrix z1)` and `toMatrix (1/z1) = (toMatrix z1)^(-1)`.

#C|P|M and different computational paths
By now, we have seen various ways to perform various computations. For instance, the mathematical idea of (z1 + z2) can be translated in the following manner:

``` haskell
--First way: Cartesian
z1 + z2 = addC z1 z2

--Second way: Matrix
z1 + z2 = addM z1 z2

```

These two ways are strongly related, and here is how:

``` haskell
z1 + z2 = toWhatever $ (toCartesian z1) `addC` (toCartesian z2) 
        = toWhatever $ (toMatrix z1) `addM` (toMatrix z2)    
```
Where `toWhatever` can be taken as `toCartesian`, `toPolar`, or `toMatrix`; the equation is still valid for each.

This is our first example of two different computational _paths_ that start with the same input and arrive at the same result.

Exercise: define two corresponding paths for multiplication.

These are paths to compute some non-trivial mathematical function (+), but there are also other equivalent paths:

``` haskell
toWhatever1 (toWhatever2 z1) = toWhatever1 z1
```

Granted this equality is not exact, but they would be exact given infinite precision in the `Double` arithmetic. In fact, the rhs is slightly better because there is always less computation involved on this path. On the other hand, the lhs is more likely to occur in human-written code (not that you would write this explicitly, but there are circumstances).

Here is one such circumstance:

``` haskell
f z1 z2 = toPolar $ z1 `add` z2

show' z1 = let z = toCartesian z1
           in  show (re z) ++ " + i " ++ show (im z)
           
main = putStrLn $ show' $ f z1 z2           
```

You see that the `main` will do something of the form `toW1 (toW2 x)`. Of course, you could have written `f` without the conversion, but suppose you wanted to write it this way (maybe it's used a lot in your program). 

In this case, one path is clearly _better_ than the other, so you could imagine a compiler that could systematically replace `toW1 (toW2 x)` by `toW1 x` everywhere in your code.

In more realistic programs, it isn't as easy to choose which path is better, but this example allows us to define what a _proper optimization_ is.

A _proper optimization_ is a transformation from one path to another for which the semantics of any program is conserved. and

A _monotonic optimization_ is a transformation from one path to a new path better no matter what.

As a rule, improper (meaning not proper) optimizations are quite useless in practice, and _monotonic optimizations_ are quite rare.

By the way, I have cooked up this terminology and do not expect it to be standard. 

In practice, we want proper and quasi-monotonic optimizations (they improve performance almost always), because those can be made by the compiler in a way that beats "by hand optimization".

However, in a real program, you have many different optimizations, often they are mutually exclusive, and so you (or the compiler) have to choose between many many many many paths, and the performance of each one can only be estimated. This is a very hard problem, and it will come back to haunt us in the future, but for now we will leave it be.

But before, I want to show you some non-trivial optimization that we can still hope to make automatically.

#Distributivity and Optimization
So far, the optimization we saw didn't even relate to the computing of (+) and (\*), which is silly because in the C representation (Cartesian), we didn't have all that extra machinery. In fact, the C representation was kind of awesome in terms of performance. 

First of all, the C|P|M representation has advantages too. For instance, we have a great way to compute (\*) (in P) that is better than what we had in C in terms of both performance and clarity of semantics. Also, we made a link with matrices. Suppose you have implemented this great library for matrices; then, you can use the C|P|M representation to test your new library. It is true that the matrix representation is always worst than the C representation in terms of performance, but in C|P|M, there is no need to ever use M (C|P is already complete, and so is C).

In general, understanding multiple representations tells you more about the nature of what you're trying to implement. For instance, now you have a way to relate matrices to complex numbers, so if you get complex numbers, then you get a little bit of matrices for free and vice versa.

But like I said before, if all you do with your complex numbers is multiplication and inverses, then P is _better_ than C, and that is a common phenomenon: different representations and semantics give rise to good and bad operations. C goes with (+), and P goes with (\*). You can still implement the other operation, but it is no longer necessary in C|P.

So here is a mathematical property that links (+) to (\*); this is where to look for optimizations in C|P.

``` haskell
--distributivity.
w * (z1 + z2) = w * z1 + w * z2
```

As you can see, this mathematical relationship gives rise to two computational paths.

Here are two possible scenarios:

``` haskell
-- w is a P
-- z1, z2 are Cs

w * (z1 + z2) = w * z1 + w * z2
w * (z1 `addC` z2) = w `multP` (toP z1) + w `multP` (toP z2)
w `multP` (toP (z1 `addC` z2)) = toC (w `multP` (toP z1)) `addC` toC (w `multP` (toP z2))

(lhs better than rhs)

--w, z1, z2 are Ps
w * (z1 + z2) = w * z1 + w * z2
w * (toC z1 `addC` toC z2) = w `multP` z1 + w `multP` z2
w `multP` toP (toC z1 `addC` toC z2) = toC (w `multP` z1) `addC` toC (w `multP` z2)

(lhs pretty much the same as rhs)
```
Exercise: work out another scenario (ex: all of them are C)

As you can see, the  optimization `rhs -> lhs` is much better in some cases, and not useful in other cases (can you find a case where it is worst?)

Notice that this optimization also exists in C, where it is always effective (always saves one operation) because there is only one case.

#Conclusion
There are two lessons to be learned:

First, the arithmetic of complex numbers. I hope you got that one.

Second, there are three concepts (Representation,Operations,Semantics) which allow us to go from a mathematical idea to a computational idea, and back again. Later, we will go from physical ideas to mathematical ones, and so on.

When implementing a mathematical idea, it doesn't make much sense to discuss the merits of some Representation without the associated operations and semantics. These three things should go together.

We went rather deep this time, and these concepts will come back time and again. However, we will come back to the surface, and stick to it most of the time. If you learned something today, then you are probably good to continue; to have an easy time too. I wanted to give you an idea of the depth which exists beneath our feet. 

#Summary

Given a mathematical idea, you can create a computational idea by specifying three things;

##Representation
The representation is the _thing_ you store while doing your computations. For instance, one can store the real and imaginary part of a complex number (C), or someone else might store the amplitude and the phase of a complex number (P), or yet another person will store an appropriate 2-by-2 matrix (M). These are three different representations. 

If there is a way to go from one representation to another, one can often introduce a mixed representation (C|P|M), and obtain many advantages (understanding, efficiency, validation by testing, etc.)

##Operation
The operations are defined as function on the representation, and there are (roughly speaking) two kinds of operations: 1) the ones which correspond to a non-trivial function in mathematics. 2) the ones which only live in the computational realm (often, those correspond to the identity function in mathematics).

It is futile to develop the representation and the operations separately---note that this statement is an empirical generalization which is often true in practice, but isn't true in general. 

In very simple cases, it is possible to have a _best_ representation in the sense that the operations you want to define are all clearly better (simpler, more efficient) in this representation than in any other you can think of. However, most cases have the following character: representation `C` makes the `(+)` operation optimal in every respect, but representation `P` makes the `(*)` operation optimal.

This fact (that one representation rarely cuts it) justifies mixed representations. However, a mixed representation is inherently more complex, and may introduce some inefficiency (going from C to P to C to P all the time kills the benefits of C over P and P over C)

If one chooses representation and operations well, the end product is often of much greater quality, and the extra time required in thinking about these things is _always_ regained through the development of sufficiently grand projects.  

##Semantics
Ok, so now you can do these funny things with 2-by-2 matrices, and somehow, these things "correspond" to the mathematical equations you wanted to compute. The semantics is a way to go from the computational realm back into the mathematical realm. In other words, given a program which does all these funny things to matrices, you use the semantics to figure out "what the hell you are computing". 

Any useful computational idea corresponds to a mathematical one. By defining the semantics explicitly, it is possible to think about your programs in the mathematical realm (which is much simpler). 

Also, this rule (define the semantics explicitly) gives you a general way to document your programs. In a world of highly optimized pieces of code written by someone else, it is vital to define the semantics explicitly.

Without this, it becomes harder to use highly optimized code written by someone else than to simply implement your own version (not optimized). This practice (rewriting everything yourself) is unsustainable as problems become bigger and bigger (this is cruelly true in computational physics).

The sad part of this is that _it doesn't have to be that way_. If someone defines the semantics of a piece of code, then it becomes as easy to use as any simpler code which has the same semantics. If you do not wish your work to fade into irrelevance the moment you stop working on it, please define the semantics of your code explicitly. Afterwards, you can optimize all you want (if the problem at hand requires such performance), but keep the semantics intact.


##Implementation
Once you have your computational idea, you need to write it explicitly in a programming language of your choice.

haskell (any purely functional language, really, but haskell happens to be the best in my opinion) has the advantage that it expresses computational ideas with extreme fidelity. This allows you to do something even better than to translate math into computational; with haskell, you can develop your computational idea by increments, and see how/if it works. This is not only a lot of fun, it allows you to come up with better ideas. Even if (due to some restraint) your final project doesn't end up in haskell, developing your computational ideas in haskell is by far the best.

In the "real world", the computational idea you end up using is often the one with the nicest implementation rather than the nicest structure (representation, operation, semantics). In haskell however, these two nice things (implementation and the idea itself) tend to coincide.