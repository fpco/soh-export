# Lenses from Scratch

`Lens` gets a lot of flack for having crazy type signatures. This is a whirlwind derivation of the craziest of signatures from the ground up. It might be useful as a starting point for exploring more of the `lens` code and seeing "through" the sophisticated types.

## Getting and Setting

At the simplest, a `Lens` is a getter and a setter much like the getters and setters in an OO language. If you've not seen them before you might wonder what "getting" and "setting" even means in a pure language.

In Haskell, getting and setting refers to the relationship between a "large" structure and some "smaller" part of it. For instance, given a tuple you might want to get and set the left side of it.

``` haskell
get :: (a, b) -> a
get (part, _) = part

set :: a' -> (a, b) -> (a', b)
set part (_, b) = (part, b)
```

so `get` is an extraction function and `set x` is a transformation on the "larger" structure parameterized by the "smaller" structure.


## Early lenses

A `Lens` is always just a "getter" and a "setter" together. The most obvious implementation would be to tuple them.

```haskell
data Lens a b 
  = Lens { get :: a -> b
         , set :: b -> a -> a
         }
```

and indeed this is a perfectly valid `Lens` that has most of the common properties of the more clever derivations to come. We'll build our lens `l` that focus on the "smaller" part of some "larger" thing and then use the field selectors `get l` and `set l` just like before.

```haskell
-- | A *Lens* that focuses on the first element in a tuple.
_1 :: Lens (a, b) a
_1 = Lens (\(part, _) -> part) (\part (_, b) -> (part, b))

> :t get _1 
get _1 :: (a, b) -> a

> :t set _1
set _1 :: a -> (a, b) -> (a, b)
```

If you're really paying attention you'll notice that the "setter" on this lens is actually slightly less powerful than the one we wrote in the first section. We'll revisit that later and rectify it.

### Composition

If this were all that Lenses offered there might not be much of a point, but they can also be chained letting us build lenses which focus deep inside of the "larger" object from many simpler lenses together.

```haskell
-- | Lens composition
(>-) :: Lens a b -> Lens b c -> Lens a c
la >- lb = Lens (get lb . get la) $ \part whole ->
  set la (set lb part (get la whole)) whole
```

the setter is a little messy, but if you think of there being big (`a`), middle (`b`), and little (`c`) pieces then it's this process, given an original big part and a new little part

1. Get the original middle part from the original big part
2. Update the middle part with the new little part
3. Update the bit part with the new middle part

which isn't too challenging, though it is tedious. Part of the magic of lens composition is that it completely dispatches with all of this tedious boilerplate.

```haskell
> :t _1 >- _1 >- _1 >- _1 >- _1    -- useless, but if you didn't have pattern matching
                                   -- this would be very challenging to code
                                   -- ... we don't always have pattern matching,
                                   -- but can always have lenses
_1 >- _1 >- _1 >- _1 >- _1 :: Lens (((((c, b4), b3), b2), b1), b) c
```


## Generalizing to Store

The lenses up to this point are fully functional, but there's a lot more structure to them that we haven't expoloited. For instance, both the "getter" and the "setter" take the larger piece as an argument. We can thus combine those applications and "extract an argument from our tuple".

```haskell
type Lens a b = (a ->  b, b -> a -> a)   -- same as before
type Lens a b = (a ->  b, a -> b -> a)   -- flip the setter
type Lens a b =  a -> (b,      b -> a)   -- pull the parameter out
```

The intuition to guide this change is that now a `Lens` is a function that takes a large structure and returns to us two things, (1) a small piece from inside that large structure, and (2) a "holey" version of our original structure---it demands a small piece like the one it gave us in order to restore the original large structure. In fact, if we just put these pieces back together we'll get back our original object.

```haskell
lensToId :: Lens a b -> (a -> a)
lensToId f = \a -> let (piece, hole) = f a in hole piece
```

This intuition is already important---[the idea of separating structures into parts and the holes that those parts fit into shows up repeatedly](http://r6.ca/blog/20121209T182914Z.html)---but the process of doing that separation `a -> (b, b -> a)`, and the result `(b, b -> a)` itself, both have surprisingly rich structure. 

Without going into any details about what that structure "means", I'll describe it. To begin, that pair is often called `Store`...

```haskell
data Store b a = Store b (b -> a)
```

### Functors

Almost too obvious to mention, but `(b, b -> a)` forms a `Functor` in exactly the same way that `(->) e`, the `Reader` `Monad`, does.

```haskell
instance Functor (Store b) where
  fmap f (Store piece hole) = Store piece (f . hole)
```

### Comonads

Without any elaboration, you can think of a `Comonad` as the inversion of a `Monad`. `Monad`s `m` are defined by their operations `join :: m (m a) -> m a` and `return :: a -> m a`. `Comonads` (often denoted `w` as a visual pun) are defined by their operations `extract :: w a -> a` and `duplicate :: w a -> w (w a)`. Or, to drive home the correspondance even more

```
|   Monad        a  -> m a                  m (m a) -> m a
| Comonad              w a -> a      w a -> w (w a)

```

It turns out that `Store b` forms a `Comonad`.

```haskell
instance Comonad (Store b) where
  extract (Store piece hole) = hole piece    -- this is the second part of lensToId!
  duplicate (Store piece hole) = 
    Store piece (\newPiece -> Store newPiece hole)
```

### Coalgebras

The final interesting structure is the `Coalgebra`. This is a generalization of `unfoldr` for turning a "seed" into something with more structure. For instance, you can use `unfoldr` to create a list counting down from an integer.

```haskell
> unfoldr (\n -> if (n == 0) then Nothing else Just (n, n-1)) 10
[10,9,8,7,6,5,4,3,2,1]
```

If we just look at `unfoldr go` we see it has a type like `a -> [a]`, an "injection" function. Things like `unfoldr go` transform seed values into lists of values and are exactly what `Coalgebra`s are. These are tightly related to recursion because so long as the larger result structure is a `Functor` (and this is actually a requirement to calling `a -> f a` a `Coalgebra`) then we can repeatedly apply our coalgebra.

```haskell
>                                  unfoldr go                :: a ->   [a]
>       fmap (unfoldr go)  .       unfoldr go                :: a ->  [[a]]  -- this is like "duplicate"
> fmap (fmap (unfoldr go)) . fmap (unfoldr go) . unfoldr go  :: a -> [[[a]]] -- and duplicate again!
```

So, at this point since we're calling our `Lens a b = a -> Store b a` and `Store b` is a `Functor` we're allowed to call our `Lens` a `Coalgebra`.

```haskell
type Coalg f a = a -> f a
type Lens a b = Coalg (Store b) a          -- (thanks to Austin Seipp for pointing this out)
```

Which, all together, leads to the [fascinating](http://r6research.livejournal.com/23705.html) or [oblique](https://twitter.com/PLT_Borat/status/228009057670291456) notion that "Lenses are Costate Comonad Coalgebras". [1]


## What did we gain?

Alright, so now that we've seen that this rather tiny trick, pulling just one argument out of our trivial "pair of a getter and a setter" structure, has exposed a great deal of hidden structure in lenses does it make our lenses any more powerful? Our intuition any more clear?

Well, hopefully thinking about the `Store` `Comonad` as holding a "piece" and its "hole" as two parts of the "whole" as given new insight, but is it really worth all that jargon?

One way we get a lot of improvement is in writing the composition operator `(>-)`. Previously it wasn't exactly difficult to write, but demanded a small amount of mental gymnastics to write the combined "setter". What does it look like using the `Store` `Comonad`?

```haskell
(>-) :: Lens a b -> Lens b c -> Lens a c
(la >- lb) a = let Store partB holeBA = la a
                   Store partC holeCB = lb partB

                   holeCA             = holeBA . holeCB
               in  Store partC holeCA
```

Really, given the types available, this function almost writes itself. It also more clearly demonstrates the idea of splitting our `a` type into the two underlying pieces and their holes---the two layers of `Store`s---and then sewing them back together to form the resulting, composed `Store`.

Furthermore, if you read the [blog post by Russell O'Connor](http://r6research.livejournal.com/23705.html) I linked earlier, you'll might have noticed there's something genuinely beautiful going on here.


### Laws

Our inutitions so far are built upon our beliefs about how Lenses "should" behave, but we can be more specific. The notion of a lens has been studied for a long time by computer scientists and a set of formal laws which outline everyone's intution have been established. These are [Pierce's Laws of a "Very Well Behaved Lens"](http://dl.acm.org/citation.cfm?doid=1232420.1232424):

1. Setting something back that you just got out is a no-op

    set s (get s) == s

2. Getting something that you just set is exactly that thing

    get (set s v) == v

3. If you put something repeatedly, the last put wins

    get (set (set s v1) v2) == v2

Hopefully you'll find that your inutition about what "getting" and "setting" might mean is completely aligned with these laws. They're carefully designed to capture the entire behavior of getting and setting and nothing more.

What O'Connor discovered was that these laws, things thought to be the essence of a particularly practical notion of computer science, can be derived as a specific instance of much more general laws known to hold for *any* "Comonad Coalgebra". In particular, given some `l :: Lens b a = a -> Store b a` they are completely equivalent to

```haskell
extract . l == id            -- injecting with the coalgebra then extracting the comonad is `id`
                             --    (this is just breaking our whole into the "piece" and "hole"
                             --     then jamming them back together)
fmap l . l  == duplicate l   -- duplicate is the same as "repeated coalgebra injection"
                             --    (see the comments in the Coalgebra section)
```

Clearly these laws are less inuitive than the "Well Behaved Lens Laws", but they also are not much more than a very formal was of saying that Comonad Coalgebras are as boring as possible---they behave exactly the way you would expect them to.

And "Costate" Comonad Coalgebras [1] behave as boringly as possible, exactly as getters and setters.


## Can we go further?

This is already a lot of generality, but we haven't yet reached the representation that [`lens`](http://hackage.haskell.org/package/lens) uses today. There's one last step.

Right now, if someone were to use our "Comonad Coalgebra" lenses they would need an instance of the `Store` datatype. That means dragging around all of our lens library as a dependency to every library which offers a lenslike interface. Not the worst result, but it'd be nice if we could offer a "minimal" interface for this dependency. Something that eliminates the datatype `Store` while keeping all of its power around.

If you read [Edward Kmett's discussion of Store Comonadic Cellular Automata](https://www.fpcomplete.com/user/edwardk/cellular-automata) or [my followup exploration](https://www.fpcomplete.com/user/tel/pretext-by-experiments-and-guesses) you will have seen a very particular and interesting operation on `Store`s called `experiment`

```haskell
experiment :: Functor f => Store b a -> (b -> f b) -> f a
experiment (Store part hole) inj = hole <$> inj part
```

As Edward hinted and I tried to explore in much greater depth, `experiment` converts a `Store` into its "functional" form. Furthermore, this conversion is an isomorphism.

```
Store piece hole          ~=         experiment (Store piece hole)
Store b     a             ~=         (b -> f b) -> f a
```

Just for completeness, I'll write the other isomorphism which I previously called `guess`

```haskell
guess :: ( forall f . Functor f => (b -> f b) -> f a )   ->   Store b a
guess m = m $ \piece -> Store piece id
```

which uses the fact that `Store` is a `Functor`.

Since it's an isomorphism, we gain nothing of value by choosing to use `Store b a` or `(forall f. Functor f => (b -> f b) -> f a)`. With `Store` we have much simpler looking types, but with the second type (also called a [`Pretext`](http://hackage.haskell.org/packages/archive/lens/3.9.0.2/doc/html/Control-Lens-Internal-Context.html#t:Pretext)) we don't need to carry around our `Store` datatype.

If we expand out what `Lens b a` is when using `Pretexts` it looks like this

```haskell
type Lens a b = Functor f => a -> (b -> f b) -> f a
```

which, while much more confusing, can be written for any type using only `fmap`.

```haskell
_1 :: Lens (a, b) a
_1 (a, b) inj = (,b) <$> inj a
```

## Composing Pretexts

Lets revisit the classic example for the power of a Lens, how does it look to compose it? It's somewhat difficult in this format to think about how two lenses compose, but we can do it.

```haskell
(>-) :: Lens a b -> Lens b c -> Lens a c
(la >- lb) a = ...
```

—wait! Actually, there's a trick. If we rearrange our definition of a `Pretext` lens.


```haskell
type Lens a b = Functor f => (b -> f b) -> (a -> f a)
```

and maybe make another type alias

```haskell
type Coalg f x = x -> f x
```

then we can see this as a much simpler type

```haskell
type Lens a b = Functor f => Coalg f b -> Coalg f a
```

and in fact we immediately have composition using basic function composition!

```haskell
(>-) :: Lens a b -> Lens b c -> Lens a c
la >- lb = la . lb
```

This finally is exactly the type of a `Lens'` you might see in the [`lens`](hackage.haskell.org/package/lens) library. In this extended form you can compose lenses using only normal function composition `(.)` and you don't need any external dependencies to write a lens for a type in your code---you need only `fmap`.

It's worth noting that this formulation was perhaps first discovered by [Twan van Laarhoven under the name "CPS Functional Reference"](http://twanvl.nl/blog/haskell/cps-functional-references) though it arose from a somewhat different direction.


## But wait, what about s-t-a-b?

If you've used [`lens`](hackage.haskell.org/package/lens) a lot you probably caught a slight trick I played a moment ago—I only bothered to define what `Lens'`, a.k.a. `Simple Lens`, is. This whole article has focused entirely on two-parameter lenses while the ones in the library are *four* parameter lenses! Where does this extra complexity come from?

Well, let's try writing one of these lenses. I'll pick the same example I've carried throughout.

```haskell
_1 :: Lens (a,b) a
_1 inj (a, b) = (,b) <$> inj a
```

That's a pretty tight definition that seems to make sense with just two parameters. Watch what happens if we forget the type line and let GHCi infer things, though.

```haskell
> :t \inj (a, b) -> (,b) <$> inj a
\inj (a, b) -> (,b) <$> inj a
  :: Functor f => (a -> f b) -> (a, t) -> f (b, t)
```

We can "sort of" interpret that type as a Pretext lens, except that we need `a ~ b` and `(a, t) ~ (b, t)`, each implying the other. This extra freedom indicates something very powerful about getters and setters---setting has the ability to change the type of the whole! [Not completely, and not always](http://comonad.com/reader/2012/mirrored-lenses/), but the type of `Lens` we defined earlier completely wipes that possibility off the table.

This possibility is, in fact, the exact thing we enountered way back at the beginning of this article, write before the first "Composition" section.

We can fix it, though, by generalizing the type of our `Lens` to its final form that we know and love.

```haskell
type Lens s t a b = Functor f => (a -> f b) -> (s -> f t)
```

Quite obvious now isn't it? ;)


---

If this was an interesting read, I highly recommend Russell O'Connor's [*Functor is to Lens as Applicative is to Biplate*](http://arxiv.org/abs/1103.2841v1) which carries out this argument with much more elegance and generality while also making many more connections to related topics. He also proves the important steps which I'm simply omitting.

---

(Thanks to aseipp, drb226, alen_ribic, heisenbug, Jameshfisher, mutjida, and tailbalance for comments and corrections.)

[Comments on Reddit](http://www.reddit.com/r/haskell/comments/1l8ld7/lenses_from_scratch/)


[1] It's a common gag, as evidenced by PL Borat there, to call `Store` lenses "Costate Comonad Coalgebras". Above I walked through where "Coalgebra" and "Comonad" come from, but left "Costate" alone. It turns out there is a relationship between `Store` and `State` though it's not truly "Co-" relationship.

The "Co-" terminology comes from category theory where "Co-X" means "X with all of the arrows turned around". Category theory tends to encode all interesting properties of all things in the arrows, so this is a very important notion. Thus all the more reason to be precise in how `State` and `Store` are actually related.

Again in whirlwind fashion, one can think of all `Monad`s as arising from the composition of two `Functors` which share a special relationship known as "adjunction". For the `Store` monad, they are as follows

```
( b , b -> a )   ==  ( b ,  _ )   .   ( b -> _ )   $   a
```

If we reverse the order of these functions, we end up with `State` directly.

```
( b , b -> a )   ==  ( b ,  _ )   .   ( b -> _ )   $   a
                     ( b -> _ )   .   ( b ,  _ )   $   a    ==   ( b -> ( b ,  a ) )
```

and obviously visa versa. I don't know of a good name for "adjunction flipped cousins", which perhaps explains the longevity of the pithy, suggestive, but ultimately incorrect term "Costate Comonad".