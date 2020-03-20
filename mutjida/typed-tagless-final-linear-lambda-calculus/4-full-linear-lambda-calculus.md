# 4 Full Linear Lambda Calculus

## Full Linear Lambda Calculus 

In this section, I will briefly introduce the rest of the LLC terms and types and then show how to extend the previous interpreter to incorporate full LLC. The presence of the additive unit (`T`) complicates algorithmic linearity checking, so I will need to add some more type level machinery; however, no new type level tricks will be needed to encode the elaborate typing rules required for efficient type checking.

### Functions

Before introducing new material, I'll quickly recap the LLC terms I've already used. I'll begin with linear variables and linear functions.

```
---------------------------------------- (lvar)
G  ;  D, x::A, D' \ D, _, D'  |-  x :: A


G ; x::A, Di \ _, Do  |-  m :: B
------------------------------------ (-<>I)
G ; Di \ Do  |-  \x -<> m :: A -<> B


G ; Di \ D  |- m :: A -<> B      G ; D \ Do  |- n :: A
------------------------------------------------------ (-<>E)
G ; Di \ Do  |-  m ^ n :: B
```

### Additives and Multiplicatives

In addition to linear and regular function types, full LLC also has conjunction and disjunction and their respective units. Unlike regular (classical or intuitionistic) logic, there are actually two flavors of conjunction in linear logic. 

The simultaneous (multiplicative) conjunction, `A * B`, requires both conjuncts be used at the same time.
```
G ; Di \ D  |-  m :: A      G ; D \ Do  |-  n :: B
-------------------------------------------------- (*I)
G ; Di \ Do  |-  <m,n> :: A * B


G ; Di \ D  |-  m :: A * B      G ; x::A, y::B, D \ _, _, Do  |-  n :: C
------------------------------------------------------------------------ (*E)
G ; Di \ Do  |-  let <x,y> = m in n :: C
```
To create a simultaneous conjunction, all the linear hypotheses needed for both conjuncts must be present since the elimination rule requires both conjuncts be used. The unit for simultaneous conjunction, `1`, contains no conjuncts, thus requires no linear hypotheses to create, but allows resource consumption when it is used.
```
----------------------- (1I)        
G ; D \ D  |-  <> :: 1


G ; Di \ D  |-  m :: 1      G ; D \ Do  |-  n :: A
-------------------------------------------------- (1E)
G ; Di \ Do  |-  let <> = m in n :: A
```

The alternative (additive) conjunction, `A & B`, requires that both conjuncts be able to be used, even though only one will be used.
```
G ; Di \ Do  |-  m :: A      G ; Di \ Do  |-  n :: B
---------------------------------------------------- (&I)
G ; Di \ Do  |-  (m,n) :: A & B


G ; Di \ Do  |-  m :: A & B            G ; Di \ Do  |-  m :: A & B
--------------------------- (&E1)      --------------------------- (&E2)
G ; Di \ Do  |-  pi1 m :: A            G ; Di \ Do  |-  pi2 m :: B
```
Note that the user of the additive conjunction gets to choose which conjunct to use, i.e. the choice is made in the elimination rule; and that the `&I` rule requires the two conjuncts use the same linear hypotheses. The unit for additive conjunction, `T`, contains no conjuncts, so it is always derivable (in any linear context), but there is no way to consume it (no elimination rule).
```
Do subset Di
------------------------ (TI)
G ; Di \ Do  |-  () :: T
```

Intuitionistic linear logic only has one (additive) disjunction, `+`, which contains one of two disjuncts where the creator chooses which, i.e. the choice is made in the introduction rule; therefore the `+E` rule must account for both possibilities.
```
G ; Di \ Do  |-  m :: A                    G ; Di \ Do  |-  m :: B
------------------------------- (+I1)      ------------------------------- (+I2)
G ; Di \ Do  |-  inl m :: A + B            G ; Di \ Do  |-  inr m :: A + B


                                G ; x::B, D \ _, Do  |-  n2 :: C
G ; Di \ D  |-  m :: A + B      G ; x::A, D \ _, Do  |-  n1 :: C    
---------------------------------------------------------------- (+E)
G ; Di \ Do  |-  case m of inl x -> n1 ; inr x -> n2 :: C
```
The unit for additive disjunction, `0`, has no introduction rule since there is nothing for the creator to choose; dually, the consumer of `0` can get anything out of it (if she somehow got a term of type `0`).
```
G ; Di \ D  |-  m :: 0   Do subset D
------------------------------------ (0E)
G ; Di \ Do  |-  abort m :: C
```

Finally, LLC has a modality to explicitly allow a term to be used non-linearly.
```
G ; \ |-  m :: A
----------------------- (!I)
G ; D \ D  |-  !m :: !A 


G ; Di \ D  |-  m :: !A      G, x::A ; D \ Do  |-  n :: B
--------------------------------------------------------- (!E)
G ; Di \ Do  |-  let !x = m in n :: B
```
In order to preserve linearity, `!A` can only be created if the `A` doesn't require any linear hypotheses. 

It is perhaps illuminating to note that regular function type, `->`, can be expressed in terms of `!` and `-<>`:
```
A -> B  ==  !A -<> B

\x -> m  ==  \y -<> let !x = y in m
```
This equivalence is justified by the following derivations (for simplicity I am not using the algorithmic typing rules) which show that the `->I` rule can be replaced by a combination of `-<>I` and `!E`
```
G, x::A ; D  |-  m :: B
---------------------------- (->I)
G ; D  |-  \x -> m :: A -> B


---------------------- (lvar)
G ; y::!A  |-  y :: !A            G, x::A ; D  |-  m :: B
--------------------------------------------------------- (!E)
G ; D, y::!A  |-  let !x = y in m :: B
--------------------------------------------- (-<>I)
G ; D  |-  \y -<> let !x = y in m :: !A -<> B
```
and that the `->E` rule can similarly be replaced by a combination of `-<>E` and `!I`
```
G ; D  |-  m :: A -> B     G ; |-  n :: A
----------------------------------------- (->E)
G ; D  |-  m n :: B

                             G ;  |-  n :: A
                             ----------------- (!I)
G ; D  |-  m :: !A -<> B     G ;  |-  !n :: !A
---------------------------------------------- (-<>E)
G ; D  |-  m ^ !n :: B
```
So regular functions are not strictly necessary in LLC, i.e. their presence adds no expressive power to the language. Nevertheless, they are convenient to have, especially when computing with LLC.

